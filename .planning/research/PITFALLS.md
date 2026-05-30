# Pitfalls Research — Cowork Sync Desktop App

**Domain:** Cross-device desktop file sync (macOS + Windows)
**Researched:** 2026-05-30
**Overall confidence:** HIGH — findings verified against official docs, real-world bug reports, and production sync app post-mortems

---

## Data Loss Risks

### CRITICAL: Sync Loop Overwrites Good Data

**What goes wrong:** When the sync app receives a remote change event, downloads the file, and writes it locally, the local file-system watcher fires and treats the just-downloaded file as a local edit — queuing it for re-upload. On multi-device setups this creates a ping-pong loop where the newest change is continuously overwritten by the second-newest, destroying the winning version.

**Why it happens:** Bidirectional sync writers forget to mute watcher events during their own writes. Every write produces a FS event. Without a "we caused this" flag, the event handler cannot distinguish user edits from sync-engine writes.

**Consequences:** Data silently regresses. The user sees "synced" but has lost their last N edits. Google Drive Desktop has shipped this bug in production (issue thread still open as of late 2025).

**Warning signs:** Two devices oscillating file modification timestamps; upload counter incrementing immediately after a download completes; file content differs between devices after both report "up to date."

**Prevention:**
1. Maintain a `pendingLocalWrites: Set<string>` of paths currently being written by the sync engine.
2. On receiving an FS event, skip upload if the path is in that set; remove the path after the write + a short settle window.
3. Use a unique in-progress temp filename (e.g., `.cowork-sync-XXXX.tmp`) and rename atomically — watchers see a RENAME event which is easier to filter.
4. Log a `source: 'sync-engine' | 'user-edit'` tag on every file operation.

**Phase:** Core sync engine (first working sync milestone).

---

### CRITICAL: Last-Write-Wins With Clock Skew Silently Drops Changes

**What goes wrong:** Using wall-clock timestamps to resolve conflicts means the machine with the faster clock always wins, silently discarding changes from the other device. In a multi-device scenario, clock drift of 100–500 ms across geographic regions is enough to consistently prefer one machine, causing the other device's changes to be invisible.

**Why it happens:** Physical timestamps are not causally reliable in distributed systems. LWW resolves conflicts but the cost is data loss — whichever write loses is gone.

**Consequences:** User edits from Device B are always overwritten by Device A (or vice versa). No error shown. Appears to sync correctly.

**Warning signs:** One device's edits consistently fail to appear on the other; timestamps between devices differ by more than 1 second when both are on the same LAN.

**Prevention:**
1. Do not use local `Date.now()` as the authoritative conflict clock. Use server-assigned timestamps (backend receives the write and stamps it) as the tie-breaker.
2. Treat any conflict (same file modified on two devices between sync events) as requiring an explicit resolution strategy: keep-both (create a `.conflict` copy), last-server-write-wins (server clock only), or manual merge — pick one and document it.
3. Version files with a server-issued monotonic sequence number, not a client timestamp.

**Phase:** Conflict resolution design (before any multi-device testing begins).

---

### CRITICAL: Partial Write / Crash Mid-Transfer Corrupts Files

**What goes wrong:** The sync engine downloads a new version of a file and begins overwriting the local copy in-place. If the app crashes, the OS suspends the machine, or the write buffer is not flushed, the destination file is left truncated or corrupted. Cowork opens a half-written file and either crashes or silently reads garbage.

**Why it happens:** Direct overwrite without atomic replacement. No intermediate state tracking.

**Consequences:** User's Cowork data is corrupted. Recovery requires the user to manually restore from a previous sync revision, if one exists.

**Warning signs:** File size on disk smaller than expected; file modification time is in the past; sync log shows a download completed but the process died before rename.

**Prevention:**
1. Always write to a temp file first: `file.cowork.tmp` (same directory, same filesystem partition — required for atomic rename to work).
2. After write completes, call `fsync()` / `FlushFileBuffers()` on the temp file to ensure the OS flushes the write to disk.
3. Perform an atomic rename (`fs.rename` on both platforms, which is atomic on POSIX and near-atomic on NTFS) to swap in the new version.
4. On startup, scan for any orphaned `.cowork.tmp` files and either complete or discard them based on a persisted sync journal.

**Phase:** Core download path (Phase 1 sync). Startup cleanup routine (Phase 2 reliability).

---

### Unlinking Account While Offline Drops Unsynced Changes

**What goes wrong:** User signs out of the app while Device A has local changes that have not yet reached the backend. The local state is cleared. When the user signs back in on Device A, the restore pull populates from the last synced revision — which is missing the offline changes.

**Consequences:** Silent data loss that appears only when the user notices missing work, potentially days later.

**Warning signs:** Sign-out attempted while `pendingUploads > 0` in the sync queue.

**Prevention:**
1. Block sign-out (or show a blocking modal) when the upload queue is non-empty.
2. Drain the upload queue to completion before clearing local credentials.
3. If drain fails (no network), present an explicit warning: "You have N unsynced changes. Sign out anyway? This data will be lost."

**Phase:** Auth / sign-out flow.

---

## Performance Traps

### Watching Too Many Files — inotify / ReadDirectoryChangesW Resource Exhaustion

**What goes wrong:** Recursively watching a large directory tree consumes one kernel-level watch descriptor per directory (inotify on Linux, though not directly applicable here) or fills event buffers (ReadDirectoryChangesW on Windows). For Windows, the API's internal buffer is fixed at allocation time — if bulk file operations (e.g., Cowork importing a large project) outpace the drain rate, the buffer overflows and `lpBytesReturned` returns 0, silently discarding the entire pending batch.

**Why it happens:** ReadDirectoryChangesW allocates a fixed-size notification buffer per directory handle. Rapid bulk writes fill it faster than the drain loop reads it.

**Consequences:** Sync misses events. Files are modified but never uploaded. The user sees "up to date" with a stale backend.

**Warning signs:** On Windows, watching for `lpBytesReturned == 0` after the API returns TRUE indicates an overflow. Sync state diverges from local disk state without any error logged.

**Prevention:**
1. Allocate a large buffer (the maximum safe size for local NTFS is well above 64 KB; note the 64 KB hard limit applies only to network drives — not local volumes).
2. Implement a fallback full-scan ("reconciliation sweep") that runs on startup and after any detected overflow, comparing actual disk state against the last-known sync state.
3. Limit the watch scope: watch only the specific Cowork data directory, not the user's entire home folder.
4. Exclude temp/cache subdirectories within Cowork's data folder from the watch scope.

**Phase:** File watcher implementation (Phase 1). Overflow detection (Phase 2).

---

### Event Storm on Startup / Large Import — Hammering the Upload API

**What goes wrong:** The first time the app starts, or after a large import into Cowork, hundreds or thousands of FS events arrive in a short burst. Without throttling, the sync engine queues a separate upload API call per file, which: (a) exhausts connection pool, (b) triggers rate-limiting HTTP 429 responses from the backend, (c) causes the retry storm to compound the load.

**Why it happens:** Naively mapping one FS event to one immediate upload with no batching or concurrency cap.

**Consequences:** Slow startup sync, 429 errors, potential backend overload. If retry logic uses fixed-delay retries instead of exponential backoff + jitter, the storm repeats.

**Prevention:**
1. Debounce FS events: coalesce events for the same path within a 200–500 ms window before enqueueing.
2. Process the upload queue with bounded concurrency (e.g., 3–5 simultaneous uploads).
3. Implement exponential backoff with full jitter on 429 / 503 responses.
4. On the first run or after detecting a "new device restore" scenario, use a bulk-upload or batch-diff API endpoint rather than per-file uploads.

**Phase:** Upload queue design (Phase 1). Rate limiting and backoff (Phase 1, not deferrable).

---

### Large File Handling — Monolithic Upload Failures

**What goes wrong:** Cowork conversation history or project exports may grow into multi-MB or multi-GB files. A single monolithic HTTP PUT that fails halfway through wastes all bandwidth, must restart from byte 0, and may time out before completing on slow connections.

**Why it happens:** Not implementing resumable / chunked upload from the start.

**Consequences:** Large files never sync successfully on poor connections. Sync appears broken for power users with large histories.

**Warning signs:** Upload timeouts in logs; file size thresholds correlating with failure rate; user reports that sync "always fails at 80%."

**Prevention:**
1. For files above a configurable threshold (e.g., 5 MB), use a chunked / multipart upload protocol with server-side upload session IDs.
2. Persist in-progress chunk progress to disk so interrupted uploads resume at the correct byte offset.
3. Use content-addressable chunk hashing (SHA-256 per chunk) to skip chunks that already exist on the server.

**Phase:** Upload implementation. Implement naive single-upload first, add chunking as a second iteration once the threshold is understood.

---

### Polling Fallback CPU Burn

**What goes wrong:** FSEvents on macOS can fall back to polling if the watch fails to initialize (sandbox permission missing, volume type unsupported). Polling stat()s every file on a schedule. On a directory with thousands of small files, this creates constant CPU and disk I/O.

**Warning signs:** High CPU usage with no active sync; `fseventsd` consuming abnormal CPU in Activity Monitor; regular 0.5–1 second CPU spikes.

**Prevention:**
1. Explicitly check that the watcher was initialized using the FSEvents backend, not the polling backend; log a warning and surface it in the status indicator if polling is active.
2. Request the necessary entitlements / sandbox exceptions during app startup and report a clear error if they are denied.

**Phase:** File watcher implementation.

---

## Auth & Security Mistakes

### CRITICAL: Storing OAuth Tokens in Plaintext / localStorage

**What goes wrong:** The token refresh token is persisted to `localStorage`, a config file in `~/.config`, or printed to logs. Any process on the machine can read it. Malware or a shared user account leads to full account takeover.

**Why it happens:** Web developer muscle memory; convenience during development.

**Consequences:** Refresh token theft = persistent account access. Claude/Anthropic account compromise.

**Warning signs:** Token visible in any file readable without elevated privileges; token appearing in crash reports or log files.

**Prevention:**
1. Store the refresh token exclusively in the OS credential store:
   - macOS: Keychain Services (via Tauri's `stronghold` plugin or a Rust `keyring` crate)
   - Windows: Windows Credential Manager (`CRED_TYPE_GENERIC` via `CredWrite`)
2. Never log token values. Log token presence/absence and expiry time only.
3. Use Tauri v2's `stronghold` plugin which wraps IOTA Stronghold (a memory-safe secret vault). Store is encrypted at rest with a user-derived key.
4. On macOS, be aware of the 4096-byte line buffer limit in the security CLI — use the API directly, not shell invocations, to avoid silent token truncation (a known Claude Code bug: anthropics/claude-code#28901).

**Phase:** Auth implementation (Phase 1, before any token is persisted anywhere).

---

### CRITICAL: Hardcoded client_secret in the Distributed Binary

**What goes wrong:** Developer registers an OAuth client, receives a `client_id` and `client_secret`, and embeds both in the app bundle. The binary ships to users. Strings extraction (`strings ./app.exe`) reveals the secret in seconds.

**Why it happens:** Treating a desktop app like a server-side app where secrets can be kept private.

**Consequences:** Anyone can impersonate the app's OAuth client, obtain tokens on behalf of users, or abuse the Anthropic API quota.

**Prevention:**
1. Desktop apps are public clients. Do not use `client_secret`. Use PKCE (Proof Key for Code Exchange) — the `code_verifier` / `code_challenge` pair ensures only the app instance that initiated the auth request can exchange the code, even without a secret.
2. If the Anthropic OAuth flow requires a client secret that cannot be avoided, route the token exchange through a thin backend proxy (a Cloudflare Worker or lightweight server endpoint) that holds the secret server-side and never exposes it in the binary.

**Phase:** Auth design (must be decided before any OAuth code is written).

---

### Token Refresh Race Condition — Multiple Simultaneous Refreshes

**What goes wrong:** The app makes two API calls concurrently. Both detect the access token is expired. Both trigger a refresh simultaneously. One refresh invalidates the other's refresh token (single-use refresh token semantics). One call fails with 401, and the app logs the user out or enters a retry loop.

**Why it happens:** No mutex or in-flight refresh deduplication.

**Prevention:**
1. Implement a token refresh lock: if a refresh is in progress, queue additional API calls and resolve them once the single refresh completes.
2. After a refresh completes, broadcast the new token to all queued callers.
3. Add a `lastRefreshedAt` guard so a refresh is not re-triggered if it succeeded within the last 30 seconds.

**Phase:** HTTP client / auth layer (Phase 1).

---

### Redirect URI Interception on macOS

**What goes wrong:** Custom URL scheme handlers (e.g., `coworksync://oauth/callback`) can be registered by any app on macOS. A malicious app that registers the same scheme first will receive the authorization code.

**Prevention:**
1. Prefer `localhost` redirect URIs (`http://127.0.0.1:{random-port}/callback`) over custom URL schemes — only the process that opened the port can receive the response.
2. PKCE mitigates the risk of code interception even if the URI is hijacked — the intercepted code is useless without the `code_verifier`.

**Phase:** Auth implementation.

---

## Platform-Specific Traps

### macOS: FSEvents Coalesces Events Across Time

**What goes wrong:** FSEvents batches and coalesces change notifications. If two files in the same directory change within the coalescing window, a single event is delivered reporting the directory changed — not which files. The app must then stat the entire directory to find what changed. If the latency parameter is set too aggressively (too high), changes feel delayed; too low, and the coalescing benefit is lost.

**Consequences:** The sync engine uploads the wrong file, misses a change, or processes stale events in the wrong order.

**Prevention:**
1. Use FSEvents with `kFSEventStreamCreateFlagFileEvents` to get file-level (not directory-level) granularity.
2. Still debounce: even file-level events are delivered in batches and may include multiple events for the same path.
3. After any FSEvents hiccup (watcher restart, sleep/wake), run a reconciliation diff between the persisted sync state and the actual directory to catch missed events.

**Phase:** File watcher implementation.

---

### macOS: Sandbox Entitlements Block the Watch Directory

**What goes wrong:** A sandboxed macOS app can only access files within its own container (`~/Library/Containers/<bundle-id>/`) plus paths explicitly granted by the user via `NSOpenPanel` or hardened runtime entitlements. Cowork's data directory is almost certainly outside the sandbox container. Without the correct entitlements, `FSEventStreamCreate` silently succeeds but delivers no events for the off-limits path.

**Warning signs:** Watcher appears to initialize without error; no events ever arrive for the Cowork data directory; adding a test file manually never triggers a callback.

**Prevention:**
1. Request the `com.apple.security.files.user-selected.read-write` entitlement and present an `NSOpenPanel` on first run to let the user grant access to the Cowork data directory explicitly.
2. Persist the security-scoped bookmark (not just the path string) returned by the panel — this is the only mechanism that survives app restarts in a sandboxed context.
3. Consider distributing as a non-sandboxed app (using a Developer ID certificate, not via the Mac App Store) for v1 to simplify the access model, but note that this requires hardened runtime and notarization.

**Phase:** macOS packaging / first-run flow (Phase 1 for the entitlement request; Phase 2 for hardening).

---

### macOS: Notarization Is Mandatory, Not Optional

**What goes wrong:** Distributing an un-notarized macOS binary causes Gatekeeper to block launch with "cannot be opened because the developer cannot be verified." macOS Sequoia removed the user-facing override command (`spctl --master-disable` no longer works); enterprise users must use MDM. For personal/internal use, this is still a friction blocker.

**Requirements:**
1. Apple Developer Program membership ($99/year).
2. Code signature with a Developer ID Application certificate.
3. Notarization via `notarytool` (not the deprecated `altool`).
4. Hardened Runtime entitlements enabled.

**Phase:** Build/distribution pipeline (not day one, but must be set up before any macOS user can run the app).

---

### Windows: ReadDirectoryChangesW Buffer Overflow Drops Events Silently

*(Detailed above under Performance Traps — cross-referenced here for platform-specific context.)*

**Additional Windows-specific note:** The overflow returns `TRUE` with `lpBytesReturned == 0` — there is no error code. Naive implementations treat this as "no events" rather than "events were lost." Add explicit detection: after any call that returns 0 bytes, trigger a full reconciliation scan.

**Phase:** File watcher implementation.

---

### Windows: File Locking Blocks Reads During App Usage

**What goes wrong:** Windows applications frequently hold exclusive locks on files they have open. Cowork itself may hold a write lock on its database or history files while the user is actively using it. The sync engine's read attempt returns `ERROR_SHARING_VIOLATION`. Unlike macOS (which uses advisory locks), Windows enforces mandatory file locks at the kernel level.

**Consequences:** Sync silently skips a file that is in use. The backend receives a stale version. After Cowork closes, the file is synced — but the backend revision history shows a gap.

**Warning signs:** Sync log shows `ERROR_SHARING_VIOLATION` for specific files; sync only succeeds after the Cowork app closes.

**Prevention:**
1. Retry file reads with exponential backoff on `ERROR_SHARING_VIOLATION` (up to a configurable max-wait, e.g., 30 seconds).
2. Treat a persistent lock as a deferred-sync condition: mark the file for re-attempt on the next lifecycle trigger (Cowork close event).
3. Use `CreateFile` with `FILE_SHARE_READ | FILE_SHARE_WRITE` share flags where possible to open for reading without blocking the owning process.

**Phase:** File read / upload path. Windows-specific retry layer (Phase 1).

---

### Windows: SmartScreen Reputation Build-Up

**What goes wrong:** A newly signed Windows executable (even with a valid OV Authenticode certificate) is flagged by Microsoft SmartScreen as "unrecognized app" and requires user click-through to run. EV (Extended Validation) certificates get immediate reputation; OV certificates require time (weeks to months of installs) to accumulate reputation.

**Additional note:** Since June 2023, OV certificates must reside on a Hardware Security Module (HSM). Azure Key Vault is the most accessible option for individual developers.

**Prevention:**
1. Budget for an EV certificate if any non-technical users will install the app.
2. Submit the binary to Microsoft's malware analysis portal for manual reputation seeding.
3. For internal/personal use (PROJECT.md scope), an OV certificate with the known SmartScreen friction is acceptable.

**Phase:** Windows distribution pipeline.

---

## Sync Logic Bugs

### Missing Debounce — Uploading In-Progress Writes

**What goes wrong:** The file watcher fires the moment a write begins. The sync engine reads the file immediately, uploading a partial or empty version. The application finishes writing the full content milliseconds later, but the watcher event for the completed write may be coalesced or not arrive if the OS considers the path unchanged.

**Why it happens:** Many editors (and Cowork itself) use a write pattern that generates 3–5 FS events per save: create temp file → write → rename to final. Without debouncing, each intermediate event triggers an upload attempt.

**Warning signs:** Uploaded file on backend is consistently smaller than expected; upload happens before the modified timestamp stabilizes; multiple upload attempts per single user save.

**Prevention:**
1. Debounce all FS events with a 200–500 ms quiet-period window per path before starting any read/upload operation.
2. After debounce, verify the file is not still growing: compare size at debounce trigger vs. size 100 ms later before reading.
3. Filter known temporary file patterns from the watch: `*.tmp`, `~$*`, `.~lock.*`, `*.swp`, `.DS_Store`, `desktop.ini`, `Thumbs.db`.

**Phase:** File watcher + upload pipeline (Phase 1).

---

### No Idempotency Key — Duplicate Uploads on Retry

**What goes wrong:** The sync engine uploads a file, the request completes on the server, but the network drops before the 200 response reaches the client. The client retries. The server receives the same file twice and creates a duplicate revision or, worse, creates a conflict with itself.

**Prevention:**
1. Generate a deterministic idempotency key per upload attempt: `SHA-256(device-id + file-path + file-content-hash)` or use a UUID persisted alongside the in-flight state.
2. The backend de-duplicates on this key and returns the original 200 if it has already processed the request.
3. Mark the local sync record as `IN_FLIGHT` before upload; on startup, re-examine all `IN_FLIGHT` records and query the backend to determine their actual status before re-uploading.

**Phase:** Upload pipeline + backend API design (Phase 1 — must be built in together).

---

### Conflict Detection Using Only Timestamps Is Fragile

**What goes wrong:** The conflict detection logic compares `local_mtime` against `server_last_modified`. If the user's machine is in a different timezone or the filesystem reports mtime at 1-second granularity (FAT32 on some USB drives, or HFS+ on older macOS), two genuinely different versions appear identical, or the conflict is missed entirely.

**Prevention:**
1. Use content-hash (SHA-256 or SHA-1 of file bytes) as the primary equality check, not mtime.
2. Keep mtime only as a fast first-pass filter; always confirm with hash before concluding "no conflict."
3. Store the last-synced content hash alongside each file's sync record so the comparison is hash vs. hash, not timestamp vs. timestamp.

**Phase:** Conflict detection logic (Phase 1 design).

---

### Initial Pull on New Device — Clobbers Newer Local Files

**What goes wrong:** User installs the sync app on a device that already has some Cowork data (e.g., manually copied). The initial restore pull overwrites all local files with whatever is on the backend, regardless of which is newer.

**Prevention:**
1. During initial restore, compare each file's content hash against the backend revision's hash.
2. If local and remote differ, apply the same conflict resolution policy as the running sync engine (e.g., keep-both) rather than blindly overwriting.
3. Provide a first-run UI choice: "Merge with existing local data" vs. "Replace with cloud backup."

**Phase:** Initial device setup / restore flow.

---

## Observability Gaps

### No Correlation ID Across Client → Backend

**What goes wrong:** A sync failure shows up on the backend as a 500 error. The client log shows a generic "upload failed." There is no shared identifier that ties the client-side event to the backend-side event, making root-cause analysis require cross-referencing timestamps from two separate systems.

**Prevention:**
1. Generate a `sync_session_id` (UUID) at app startup and a `request_id` per upload/download operation.
2. Include both as HTTP headers (`X-Sync-Session-Id`, `X-Request-Id`) on every API call.
3. Backend logs and client logs both record these IDs — now a single grep finds the full trace.

**Phase:** HTTP client setup (Phase 1 — build this in before the first API call).

---

### No Structured Sync Event Log

**What goes wrong:** The only observable state is the UI status badge ("syncing" / "up to date"). When a user reports "my file isn't syncing," there is no queryable event history to diagnose what the sync engine saw and did.

**Prevention build:**
Log the following as structured JSON events (not human-readable strings) to a rolling log file:

| Event | Fields |
|-------|--------|
| `fs.event` | path, event_type (create/modify/delete/rename), source (user/sync-engine) |
| `upload.queued` | path, content_hash, file_size_bytes |
| `upload.started` | path, request_id, attempt_number |
| `upload.completed` | path, request_id, duration_ms, bytes_transferred |
| `upload.failed` | path, request_id, http_status, error_message, will_retry |
| `download.started` | path, request_id, remote_version |
| `download.completed` | path, duration_ms |
| `conflict.detected` | path, local_hash, remote_hash, resolution |
| `watcher.overflow` | platform, directory, action_taken (reconcile) |
| `auth.token_refreshed` | success, expires_in |
| `auth.token_refresh_failed` | error, retry_in |

Rotate logs at 10 MB; keep last 5 rotations. Never log token values.

**Phase:** Logging infrastructure (Phase 1, before sync code is written — retroactively adding structured logging is painful).

---

### No File-Level Sync State Persistence

**What goes wrong:** The sync engine tracks what has been synced in memory only. On restart, it has no idea what was uploaded before the crash. It either re-uploads everything (wasteful) or assumes everything is current (misses post-crash changes).

**Prevention:**
1. Persist a sync database (SQLite is the standard choice for this) mapping `file_path → { last_synced_hash, last_synced_at, status: pending|in_flight|synced|conflict }`.
2. On startup, reconcile this database against both the local filesystem and the backend manifest.
3. Never trust in-memory state across process restarts.

**Phase:** Sync state design (Phase 1 — the database schema needs to exist before the first file is synced).

---

### Status Indicator Does Not Surface Specific Errors

**What goes wrong:** The app shows a red dot for "error" but gives no detail. User cannot tell if it is a network issue, an auth issue, a file permission issue, or a platform-level watcher failure. Support requests arrive without actionable information.

**Prevention:**
1. The status indicator must distinguish: `syncing`, `up_to_date`, `auth_error` (requires re-login), `network_error` (retrying), `file_error` (specific file blocked — show which), `watcher_error` (file watching degraded).
2. Each error state should link to a log excerpt or a human-readable explanation.
3. Provide a "Copy diagnostics" button that exports the last 100 log lines as a redacted (no token values) text blob.

**Phase:** Status UI (Phase 1 for basic states; Phase 2 for detailed error surfaces).

---

## Phase Mapping Summary

| Pitfall | Phase Priority | Deferrable? |
|---------|---------------|-------------|
| Sync loop overwrites | Phase 1 (core sync) | No — catastrophic if deferred |
| LWW clock skew | Phase 1 (design) | No — requires backend API contract |
| Partial write / atomic rename | Phase 1 (download path) | No |
| Hardcoded client_secret | Phase 1 (auth design) | No |
| Token stored insecurely | Phase 1 (auth) | No |
| Debounce / in-progress reads | Phase 1 (watcher) | No |
| Idempotency key on upload | Phase 1 (upload) | No |
| Structured sync log | Phase 1 (infra) | No — painful to add later |
| Sync state SQLite DB | Phase 1 (design) | No |
| Token refresh race condition | Phase 1 (HTTP client) | No |
| ReadDirectoryChangesW overflow | Phase 1 (Windows watcher) | No |
| Windows file locking retry | Phase 1 (Windows read path) | Partial — basic retry only |
| FSEvents coalescing handling | Phase 1 (macOS watcher) | No |
| macOS sandbox entitlements | Phase 1 (macOS packaging) | No — app cannot watch directory without it |
| Upload rate limiting / backoff | Phase 1 (upload queue) | No |
| Large file chunked upload | Phase 2 | Yes — start with 5 MB threshold check |
| Correlation IDs | Phase 1 (HTTP client) | No |
| macOS notarization | Phase 1-2 (distribution) | Yes for dev iteration; No for first user |
| Windows SmartScreen / EV cert | Phase 2 (distribution) | Yes for internal use |
| Initial pull conflict on new device | Phase 2 (restore flow) | Acceptable for v1 with warning UI |
| Status indicator error detail | Phase 2 (UI) | Yes — basic states in Phase 1 |

---

## Sources

- ReadDirectoryChangesW buffer overflow behavior: https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-readdirectorychangesw and https://qualapps.blogspot.com/2010/05/understanding-readdirectorychangesw_19.html
- Internxt Drive Desktop sync loop bug (production): https://github.com/internxt/drive-desktop/issues/738
- FSEvents documentation: https://developer.apple.com/documentation/coreservices/file_system_events
- macOS sandbox and file access: https://bdash.net.nz/posts/sandboxing-on-macos/
- macOS notarization requirements: https://developer.apple.com/documentation/security/notarizing-macos-software-before-distribution
- Windows Tauri code signing: https://v2.tauri.app/distribute/sign/windows/
- Clock skew in distributed systems: https://systemdr.substack.com/p/the-clock-skew-conflict-when-time
- LWW vs CRDTs conflict resolution: https://dzone.com/articles/conflict-resolution-using-last-write-wins-vs-crdts
- Desktop app secrets and PKCE: https://developers.onelogin.com/api-authorization/using-the-appauth-pkce-to-authenticate-to-your-electron-application
- Trail of Bits — insecure credential storage in desktop apps: https://blog.trailofbits.com/2025/04/30/insecure-credential-storage-plagues-mcp/
- macOS Keychain CLI 4096-byte truncation bug: https://github.com/anthropics/claude-code/issues/28901
- Tauri stronghold plugin docs: https://v2.tauri.app/plugin/stronghold
- Atomic write / temp file pattern: https://tech-champion.com/data-science/stop-silent-data-loss-checksum-atomic-writes-temp-file-patterns/
- File watcher debounce and coalescing: https://medium.com/@impactarchitecture/file-watchers-lie-debounce-throttle-and-coalescing-in-build-loops-8d91cb29f712
- Offline-first sync idempotency: https://dev.to/salazarismo/the-hidden-problems-of-offline-first-sync-idempotency-retry-storms-and-dead-letters-1no8
- API rate limiting exponential backoff with jitter: https://oneuptime.com/blog/post/2026-02-17-how-to-handle-api-rate-limiting-and-implement-exponential-backoff-in-gcp/view
- OneDrive file-in-use behavior: https://learn.microsoft.com/en-us/answers/questions/1285490/how-does-onedrive-sync-app-handle-an-opened-locked
- Workato bi-directional sync loop prevention: https://www.workato.com/product-hub/how-to-prevent-infinite-loops-in-bi-directional-data-syncs/
