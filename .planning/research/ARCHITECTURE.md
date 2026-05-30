# Architecture Research — Cowork Sync Desktop App

**Researched:** 2026-05-30
**Confidence:** MEDIUM-HIGH (patterns drawn from well-documented production systems: Dropbox, Nextcloud, Syncthing; supplemented with RFC 8252 and offline-first literature)

---

## System Overview

The system follows a **hub-and-spoke client-server model**: each desktop client is a spoke that reads/writes local Cowork data and communicates exclusively with a central cloud backend (the hub). Clients never talk to each other directly — all sync flows through the server.

```
[Device A: macOS]              [Cloud Backend]              [Device B: Windows]
  File Watcher                    REST API                    File Watcher
  Sync Engine    ──────push──→   Sync Store   ←──pull──────  Sync Engine
  Local DB                       Metadata DB                 Local DB
  Auth (keychain)                Auth Service                Auth (credential store)
```

Data always flows: local filesystem → sync engine → cloud backend → other devices' sync engines → their local filesystems. The backend is the single source of truth for the current synced state.

---

## Major Components

### Client Side

#### 1. File Watcher
**Responsibility:** Detect filesystem changes in the Cowork data directory (creates, modifies, deletes, renames).
**Inputs:** OS filesystem events from the Cowork data directory.
**Outputs:** Change events delivered to the Sync Engine.
**Implementation:** On macOS, use `FSEvents` (via Tauri's `tauri-plugin-fs-watch` or the underlying `notify` Rust crate). On Windows, use `ReadDirectoryChangesW` (same abstraction). Chokidar v5 (Node/Electron) or Rust's `notify` crate (Tauri) both provide cross-platform unified APIs over these OS primitives.
**Boundary:** Knows nothing about cloud state. Its only job is "something changed here."

#### 2. Sync Engine
**Responsibility:** The coordinator. Receives file change events, decides what to push; receives server notifications, decides what to write locally. Owns conflict resolution logic.
**Inputs:** File Watcher events (local changes); poll/push responses from backend (remote changes); reconnection events from Network Monitor.
**Outputs:** Upload payloads to the API Client; write instructions to the File Writer; status updates to the UI Status Layer.
**Key behaviors:**
- Debounces rapid file events (e.g., 300ms quiet window) before treating a save as a complete change
- Compares local file hash against last-known-synced hash to skip no-op events
- Manages the Pending Operations Queue
- Applies remote changes to local disk without re-triggering the file watcher (use a write lock / in-progress flag)

#### 3. Local Metadata DB (SQLite)
**Responsibility:** The client's memory of sync state. Persists across restarts.
**Schema (minimum):**
```sql
files (
  path TEXT PRIMARY KEY,        -- relative path from Cowork root
  content_hash TEXT,            -- SHA-256 of last synced content
  server_version INTEGER,       -- version number from server
  sync_status TEXT,             -- 'synced' | 'pending_upload' | 'pending_download' | 'conflict'
  modified_at INTEGER,          -- unix ms of last local modification
  synced_at INTEGER             -- unix ms of last successful sync
)

pending_ops (
  id TEXT PRIMARY KEY,
  op_type TEXT,                 -- 'upload' | 'delete' | 'rename'
  file_path TEXT,
  payload BLOB,
  created_at INTEGER,
  retry_count INTEGER DEFAULT 0,
  status TEXT DEFAULT 'pending' -- 'pending' | 'in_flight' | 'failed'
)
```

#### 4. API Client
**Responsibility:** All HTTP communication with the backend. Handles auth headers, retry on transient failures, and response parsing.
**Inputs:** Structured requests from Sync Engine.
**Outputs:** Structured responses (success, conflict, error) back to Sync Engine.
**Boundary:** Knows nothing about files — only about API contracts.

#### 5. Network Monitor
**Responsibility:** Detect online/offline state transitions and notify the Sync Engine to pause or resume.
**Implementation:** Poll a known endpoint (e.g., `GET /health`) with a 10s interval rather than relying on `navigator.onLine` (unreliable). Debounce flapping with a 5s confirmation window before declaring online.

#### 6. Auth Manager
**Responsibility:** Manages the OAuth token lifecycle — initiating PKCE login, receiving the authorization callback, storing tokens in OS secure storage, refreshing access tokens before expiry.
**Inputs:** Login trigger from UI; token expiry signals from API Client (401 responses).
**Outputs:** Valid access tokens to API Client on demand.
**Boundary:** The only component that touches the OS keychain / credential store.

#### 7. UI Status Layer
**Responsibility:** System tray icon and minimal status UI — syncing, up to date, error, offline. Receives status events from Sync Engine; does not drive any sync logic.

---

### Server Side

#### 1. Sync API (REST)
**Responsibility:** Accepts file uploads, serves file downloads, manages version metadata, routes change notifications.
**Key endpoints:**
- `PUT /files/{path}` — upload a file change with current hash + new content
- `GET /files/{path}?since_version={n}` — pull a file at or since a version
- `GET /changes?since_version={n}` — list all changed paths since a version (for reconnection catch-up)
- `DELETE /files/{path}` — record a deletion
- `GET /health` — used by client Network Monitor

#### 2. Metadata Store
**Responsibility:** The authoritative record of every file's current version, hash, and change history.
**Schema (minimum):**
```sql
files (
  user_id TEXT,
  path TEXT,
  version INTEGER,             -- monotonically increasing per user
  content_hash TEXT,
  size_bytes INTEGER,
  updated_at INTEGER,
  deleted BOOLEAN DEFAULT FALSE,
  PRIMARY KEY (user_id, path)
)

file_versions (
  user_id TEXT,
  path TEXT,
  version INTEGER,
  content_hash TEXT,
  stored_at INTEGER,
  PRIMARY KEY (user_id, path, version)
)
```

#### 3. Blob Store
**Responsibility:** Store actual file content, keyed by content hash. Deduplicates identical content across versions.
**Implementation:** S3-compatible object storage (AWS S3, Cloudflare R2, or MinIO for self-hosted). Key pattern: `{user_id}/{content_hash}`. Because the key is the content hash, uploading the same content twice is idempotent.

#### 4. Auth Service
**Responsibility:** Validate incoming OAuth tokens from Anthropic/Google/Apple. Issue short-lived session tokens or validate JWT claims. Not built from scratch — delegate to the upstream OAuth provider.

---

## Data Flow

### Push (local change → cloud)

```
1. OS emits FS event for modified file
2. File Watcher delivers event to Sync Engine
3. Sync Engine debounces (waits 300ms for quiet)
4. Sync Engine computes SHA-256 of file content
5. Compare hash against Local DB last-synced hash
   → If same: no-op, skip
   → If different: continue
6. Sync Engine writes pending_op to Local DB (status: 'pending')
7. Sync Engine calls API Client: PUT /files/{path}
   with body: { content_hash, content, client_version: last_known_server_version }
8. Server validates:
   a. Auth token valid
   b. client_version == current server version → accept, increment server version
   c. client_version < current server version → CONFLICT, return 409 with server's version
9. On 200: Sync Engine marks file 'synced' in Local DB, updates server_version
10. On 409: Sync Engine marks file 'conflict', surfaces to UI, applies conflict resolution policy
11. On network error: Sync Engine increments retry_count, exponential backoff (2^n seconds, max 5min)
```

### Pull (cloud change → local)

```
1. On reconnect or periodic poll (30s), Sync Engine calls GET /changes?since_version={last_version}
2. Server returns list of {path, version, content_hash, op_type}
3. Sync Engine compares each entry against Local DB:
   → If local hash == server hash: already synced, skip
   → If local status == 'pending_upload': potential conflict, apply policy
   → Otherwise: queue download
4. For each download: GET /files/{path}?version={n}
5. Sync Engine sets internal write-lock flag (prevents File Watcher from re-queuing)
6. File Writer writes content to disk
7. Sync Engine updates Local DB (content_hash, server_version, sync_status: 'synced')
8. Write-lock released
```

### Initial install / full restore

```
1. User installs app, authenticates
2. Sync Engine calls GET /changes?since_version=0
3. Server returns full manifest of all files
4. Client downloads and writes all files to Cowork data directory
5. Local DB populated with current state — future syncs are incremental
```

---

## Delta Sync Strategy

**Recommendation: Hash-gated full-file transfer for v1.**

For the Cowork use case (JSON/text workspace files, conversation history, settings — not large binary assets), full-file transfer per change is appropriate and significantly simpler to implement correctly:

- Compute SHA-256 on the client before upload
- Send the full file content in the request body
- Server stores content in blob store keyed by hash (automatic deduplication)
- Clients skip download if their local hash already matches the server hash

**When to add block-level delta sync (v2+):** Only if profiling reveals that large files (e.g., conversation history that grows continuously) are causing excessive bandwidth. At that point, use rolling-hash chunking (rsync algorithm) or content-defined chunking (Rabin fingerprinting). This complexity is not justified for v1.

**Conflict resolution policy for v1:** Last-writer-wins using the server's monotonic version counter as the arbiter. On a 409, the client fetches the server version, presents a "conflict" status in the UI, and — for v1 — the server version wins automatically. A conflict copy of the local version is preserved with a `.conflict.{timestamp}` suffix so no data is ever destroyed.

---

## Auth Architecture

**Standard: RFC 8252 + PKCE + loopback redirect.**

The correct OAuth flow for a desktop native app with no embedded secret:

```
1. User clicks "Log In"
2. Auth Manager generates:
   - code_verifier: 32 bytes cryptographically random, base64url-encoded (43-128 chars)
   - code_challenge: BASE64URL(SHA256(code_verifier))
3. Auth Manager opens system browser to:
   {auth_endpoint}?
     response_type=code
     &client_id={public_client_id}
     &redirect_uri=http://127.0.0.1:{random_port}/callback
     &code_challenge={code_challenge}
     &code_challenge_method=S256
     &scope=openid profile
4. Auth Manager starts a local HTTP server on that random port (ephemeral, single-use)
5. User authenticates in browser; provider redirects to http://127.0.0.1:{port}/callback?code={code}
6. Local HTTP server receives the code, shuts itself down
7. Auth Manager POSTs to token endpoint:
   code={code}&code_verifier={verifier}&grant_type=authorization_code&...
8. Receives access_token (short-lived, e.g., 1h) + refresh_token (long-lived)
9. code_verifier is zeroed out / discarded
```

**Token storage:**
- macOS: Keychain Services (`SecItemAdd` / `SecItemCopyMatching`) — use the `keyring` crate (Rust/Tauri) or `keytar` (Electron)
- Windows: Windows Credential Manager (Credential Locker) — same `keyring` / `keytar` abstractions cover both
- Never store tokens in plaintext config files, localStorage, or app databases

**Token refresh:**
- API Client checks token expiry before each request (subtract 60s buffer)
- On expiry or 401: Auth Manager uses refresh_token to obtain a new access_token silently
- On refresh failure (refresh_token expired/revoked): force re-login via browser flow
- Rotate refresh tokens on each use if the provider supports it (reduces stolen-token window)

**Google/Apple OAuth:** Same PKCE flow, different authorization endpoints. Treat as interchangeable at the Auth Manager layer — abstract behind an `AuthProvider` interface so the provider can be swapped.

---

## Offline & Reconnection

**Core principle: local-first. The app is always usable offline; sync is a background concern.**

### Offline behavior
- File Watcher continues running; Sync Engine continues detecting changes
- Changes are written to `pending_ops` in Local DB with status `'pending'`
- API Client calls are not attempted; Network Monitor has declared offline
- UI status shows "Offline — changes queued"
- No data is lost; the queue is durable (SQLite survives process restarts)

### Reconnection
```
1. Network Monitor detects online (confirmed with /health ping)
2. Sync Engine notified: "online"
3. Sync Engine executes reconnection sequence:
   a. GET /changes?since_version={last_known_version}  ← pull remote changes first
   b. Apply remote changes locally (handle any conflicts with queued local ops)
   c. Process pending_ops queue in FIFO order
   d. On success: clear from queue, update Local DB
   e. On per-item failure: increment retry_count, schedule exponential backoff
4. UI transitions to "Syncing..." then "Up to date"
```

### Retry strategy
- Exponential backoff: 2^retry_count seconds (1s, 2s, 4s, 8s… up to 5 minutes)
- Maximum 10 retries per operation; after that, mark `'failed'` and alert UI
- Network-level errors (timeout, 5xx): retry with backoff
- Client errors (400, 409 conflict): do not retry automatically — require specific handling

### Conflict during reconnection
When the queue contains a local `UPDATE` to file A, and the pull reveals the server also has a newer version of file A:
1. Server version wins (last-writer-wins for v1)
2. Local pending change is preserved as `.conflict.{timestamp}` copy
3. Sync status for that file shows "Conflict — local copy saved"

---

## Suggested Build Order

**Verdict: Data model + backend API first, then client.**

The rationale: the client's correctness depends entirely on the sync protocol contract. Building the client first against a stub or without a real backend risks designing a protocol that doesn't handle edge cases (conflicts, reconnection, version gaps) — leading to rewrites. Define the contract first, then build both sides against it.

### Phase 1 — Foundation: Auth + Data Model
Build before anything else. Everything depends on it.
- Define the file metadata schema (Local DB + server DB)
- Define the sync API contract (OpenAPI spec or equivalent)
- Implement OAuth PKCE flow end-to-end (browser → loopback → token → keychain)
- Stand up minimal backend: auth middleware, `PUT /files`, `GET /changes`
- **Test:** Can a token be obtained and stored? Can a file be uploaded and retrieved?

### Phase 2 — Backend Core: Sync Store
Build server-side sync robustly before client complexity.
- Metadata store with monotonic version counter per user
- Blob store integration (S3/R2)
- Conflict detection (version mismatch → 409)
- `GET /changes?since_version` endpoint
- **Test:** Upload two conflicting versions from two simulated clients; verify server returns correct 409 and stores correct winning version.

### Phase 3 — Client Core: Sync Engine + File Watcher
Now build the client against the real backend.
- File Watcher (cross-platform: macOS FSEvents + Windows ReadDirectoryChangesW)
- Local SQLite metadata DB
- Sync Engine: push flow (watch → hash → upload → update local DB)
- Sync Engine: pull flow (poll /changes → download → write to disk with write-lock)
- **Test:** Modify a file on Device A; confirm it appears on Device B via the real backend.

### Phase 4 — Resilience: Offline + Reconnection
Add durability and error handling.
- Pending ops queue in Local DB
- Network Monitor
- Reconnection sequence (pull-then-push with conflict handling)
- Exponential backoff and retry logic
- **Test:** Modify files while offline; reconnect; confirm changes reach the server.

### Phase 5 — Packaging + UI Polish
- System tray UI with status indicator
- Initial restore flow (fresh install pulls full state)
- macOS + Windows packaging (code signing, auto-update)
- **Test:** Full install-on-new-device scenario end-to-end.

### Dependency diagram
```
Auth + Data Model
      ↓
  Backend Core  ────────────────────────┐
      ↓                                 ↓
 Client Core (push)         Client Core (pull)
      ↓                                 ↓
        Resilience (offline + reconnect)
                    ↓
           Packaging + UI Polish
```

---

## Key Architectural Decisions (Opinionated)

| Decision | Recommendation | Rationale |
|----------|---------------|-----------|
| Sync topology | Client-server (hub-and-spoke) | Simplest correct model; no peer-to-peer NAT traversal complexity |
| Server as truth | Server version wins on conflict, local copy preserved | Data safety: never silently discard user data |
| Delta sync v1 | Full-file with hash-gate | Correct and simple; defer chunking until profiling proves it's needed |
| Conflict resolution v1 | Last-writer-wins (server version wins) | Avoids CRDT complexity; Cowork files are not collaboratively edited |
| Desktop framework | Tauri (Rust backend + WebView UI) | ~10x smaller binaries, lower memory, native FS/keychain APIs; Electron viable if TypeScript-only stack is required |
| File watching | `notify` crate (Tauri/Rust) or `chokidar` v5 (Electron) | Both abstract FSEvents + ReadDirectoryChangesW; `notify` preferred with Tauri |
| Token storage | OS keychain via `keyring` crate | RFC 8252 requirement; never plaintext |
| Local state DB | SQLite via `rusqlite` (Tauri) or `better-sqlite3` (Electron) | Reliable, queryable, survives crashes; supports the pending ops queue |
| Reconnection strategy | Pull-then-push | Prevents overwriting newer server state with stale local ops |

---

## Sources

- [Dropbox/Google Drive System Design — Narendra Gowda (Medium)](https://medium.com/@narengowda/system-design-dropbox-or-google-drive-8fd5da0ce55b)
- [Nextcloud Desktop Client Architecture Docs](https://docs.nextcloud.com/desktop/latest/architecture.html)
- [RFC 8252 — OAuth 2.0 for Native Apps](https://www.rfc-editor.org/rfc/rfc8252.html)
- [Auth0 — Authorization Code Flow with PKCE](https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow-with-pkce)
- [Building an Offline-First App with Sync Engine — DEV Community](https://dev.to/daliskafroyan/builing-an-offline-first-app-with-build-from-scratch-sync-engine-4a5e)
- [Delta Sync & Merkle Trees — System Design Sandbox](https://www.systemdesignsandbox.com/learn/delta-sync)
- [Syncthing — Understanding Synchronization](https://docs.syncthing.net/users/syncing)
- [Tauri 2.0 Official Docs](https://v2.tauri.app/)
- [Chokidar v5 GitHub](https://github.com/paulmillr/chokidar)
- [Offline-First Architecture — Jusuf Topic (Medium)](https://medium.com/@jusuftopic/offline-first-architecture-designing-for-reality-not-just-the-cloud-e5fd18e50a79)
