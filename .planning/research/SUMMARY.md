# Project Research Summary

**Project:** Cowork Sync Desktop App
**Domain:** Cross-device desktop file sync for structured app data (macOS + Windows)
**Researched:** 2026-05-30
**Confidence:** MEDIUM-HIGH

## Executive Summary

This is a background-daemon sync app — a system tray / menu bar process that watches Claude Cowork's local data directory, pushes changes to a cloud backend, and pulls them down on other devices. The canonical architecture is hub-and-spoke client-server (not peer-to-peer): every client speaks exclusively to a central backend which is the single source of truth. The recommended implementation stack is Tauri 2.x (Rust + WebView, ~10 MB installer) over Electron (~150 MB), Supabase for managed auth + Postgres + Realtime + Storage, and Anthropic OAuth 2.0 with PKCE for identity. The backend should be designed and tested first; the client sync engine is built against the real API contract, not a stub.

The single highest-risk item in the entire stack is the Anthropic OAuth integration. The OAuth endpoints used by Claude Code are not in Anthropic's public API docs — they are reverse-engineered from Claude Code's behavior. If Anthropic restricts these endpoints or requires a separate client registration, the auth layer must fall back to Google/Apple OAuth via Supabase's built-in providers. This must be validated before any other auth work begins. Everything else in the stack is well-documented and low-risk.

The critical execution risks are all in the sync engine, not the UI: sync loops (the app re-uploading its own downloads), partial-write corruption (crashing mid-download leaves a corrupted file), and clock-skew data loss (LWW based on client timestamps silently drops edits from the slower-clocked device). All three must be designed out in Phase 1 before any multi-device testing. The architectural mitigations are well-understood — write-lock flags, atomic rename via temp file, server-assigned monotonic version numbers — but none are optional: each is a precondition for the sync engine being safe to run.

## Key Findings

### Recommended Stack

Tauri 2.11.2 is the correct framework for a background tray daemon. Its installer is under 10 MB, idle memory is 30–50 MB, and it provides first-class tray/menu bar support via `TrayIconBuilder`. Setting `LSUIElement: true` hides the app from the macOS Dock and Cmd+Tab switcher. File watching is handled by `@tauri-apps/plugin-fs` (backed by the `notify` Rust crate), which uses FSEvents on macOS and ReadDirectoryChangesW on Windows with no polling and no external dependencies. Supabase handles the entire cloud backend at personal scale — auth, Postgres, Realtime WebSocket push, and Storage — for free on the free tier. The Cloudflare Workers + R2 + Durable Objects stack is the right upgrade path if this goes public but is over-engineered for personal use.

The Cowork data paths are confirmed: `~/Documents/Claude/` (artifacts/projects), `~/.claude/` (config/session history), `~/Library/Application Support/Claude/` (scheduled tasks) on macOS; Windows equivalents use `Documents\Claude\`, `\.claude\`, and `%APPDATA%\Claude\`. The `~/Documents/Claude/` path is hardcoded in Cowork (GitHub issue #57177 open for configurability) — the sync app must target this exact path.

**Core technologies:**
- **Tauri 2.11.2**: desktop framework — ~10 MB installer, native tray support, Rust file watching, no Chromium overhead
- **`@tauri-apps/plugin-fs` (notify crate)**: file watching — kernel-level FSEvents/ReadDirectoryChangesW, no polling
- **Supabase**: backend (auth + Postgres + Realtime + Storage) — free tier covers personal scale, eliminates custom server
- **Anthropic OAuth 2.0 + PKCE**: identity — matches Cowork's own auth; fallback to Google/Apple via Supabase if restricted
- **`keyring` Rust crate**: token storage — wraps macOS Keychain and Windows Credential Manager
- **SQLite (`rusqlite`)**: local sync state — pending ops queue, file hash cache, survives restarts
- **`tauri-plugin-updater` + GitHub Releases**: auto-update — no update server to maintain
- **GitHub Actions + `tauri-action`**: CI/CD — cross-platform builds, code signing

### Expected Features

**Must have (table stakes):**
- Automatic background sync triggered by filesystem events and Cowork open/close lifecycle hooks
- Launch at login / run as daemon (macOS LaunchAgent, Windows startup registry)
- System tray / menu bar presence with three states: synced, syncing, error
- Initial device setup that pulls all data to the correct Cowork paths with deterministic progress display
- Account login via Anthropic OAuth with tokens stored in OS keychain — never plaintext
- Atomic safe writes (write-to-temp, fsync, atomic rename) — data loss from sync destroys trust permanently
- Resume interrupted sync via durable pending-ops queue in SQLite
- Error surfacing via tray menu and OS notifications (not silent failures)
- Settings persistence across restarts

**Should have (differentiators):**
- Per-data-category conflict strategy: LWW with conflict copy for files, field-level merge for settings, append-only union for conversation history
- Conflict transparency notification showing what was resolved and how
- Sync health log (last N events accessible from tray menu)
- Offline-aware operation: queue changes locally, sync on reconnect
- Pause/resume sync toggle in tray menu

**Defer (v2+):**
- Selective sync (choose which projects to sync)
- Version history / file restore UI
- Bandwidth throttle controls
- Sharing / multi-user sync
- Mobile (iOS, Android)

### Architecture Approach

The system is hub-and-spoke client-server: clients push/pull exclusively through the cloud backend; no peer-to-peer. The backend holds the monotonic version counter that resolves all conflicts — never the client clock. The recommended build order is backend API contract first, then client sync engine against the real backend. Client-side, the key components are: File Watcher (OS events only, knows nothing about cloud state), Sync Engine (coordinator: hash, debounce, queue, conflict resolution), Local SQLite DB (pending ops + sync state, survives restarts), API Client (HTTP only, no file knowledge), Auth Manager (sole toucher of OS keychain), Network Monitor (online/offline transitions via /health polling), and UI Status Layer (receives state, drives no logic). The pull-then-push reconnection sequence is critical: always fetch remote changes before flushing the local queue to avoid overwriting newer server state.

**Major components:**
1. **File Watcher** — detects filesystem changes in Cowork data dir; knows nothing about cloud state
2. **Sync Engine** — coordinator: debounces events, computes hashes, manages pending-ops queue, owns conflict resolution
3. **Local SQLite DB** — durable sync state: `files` table (path, hash, server_version, status) + `pending_ops` queue
4. **API Client** — all HTTP to backend; handles auth headers, idempotency keys, exponential backoff
5. **Auth Manager** — sole owner of OAuth PKCE flow and OS keychain/credential store
6. **Network Monitor** — online/offline detection via /health polling with 5s flap-debounce
7. **Supabase Backend** — auth, Postgres metadata, Realtime WebSocket push, Storage blobs (content-addressed by SHA-256)

### Critical Pitfalls

1. **Sync loop overwrites good data** — When the engine writes a downloaded file, the file watcher fires and re-queues it for upload, creating a ping-pong that silently regresses content. Prevention: maintain a `pendingLocalWrites: Set<string>` flag; skip upload if the path is in the set; use `.cowork-sync-XXXX.tmp` + atomic rename so the RENAME event is easy to filter.

2. **LWW clock skew silently drops edits** — Using `Date.now()` as the conflict arbiter means the faster-clocked device always wins. 100–500 ms drift across devices is enough to consistently discard one machine's edits. Prevention: use server-assigned monotonic version numbers, never client timestamps, as the tie-breaker.

3. **Partial write / crash mid-download corrupts Cowork files** — Writing a downloaded file in-place and crashing mid-write leaves a truncated or corrupted file that Cowork then opens. Prevention: always write to `file.cowork.tmp` (same partition), call `fsync()`, then atomic rename. On startup, scan for orphaned `.tmp` files and clean up.

4. **OAuth token stored in plaintext** — Web developer muscle memory stores the refresh token in `localStorage`, a config file, or app logs. Any process on the machine can read it. Prevention: use the `keyring` crate (Keychain on macOS, Credential Manager on Windows) from day one; never log token values.

5. **macOS sandbox entitlements silently block the watch directory** — A sandboxed macOS app cannot watch `~/Documents/Claude/` without explicit user grant via `NSOpenPanel` and a persisted security-scoped bookmark. The watcher initializes without error but delivers no events. Prevention: request `com.apple.security.files.user-selected.read-write`, present the directory picker on first run, persist the security-scoped bookmark (not just the path string) so it survives restarts.

## Implications for Roadmap

The architecture research is explicit: define the backend API contract before building the client. The pitfalls research is equally clear: the sync engine correctness invariants (write-lock flag, atomic rename, server-version LWW, debounce, idempotency keys, SQLite state DB) must all be in place before the first multi-device test — none are safely deferrable. This drives a 5-phase structure matching the ARCHITECTURE.md build order recommendation.

### Phase 1: Foundation — Auth + Data Contract

**Rationale:** Everything else depends on a working auth token and an agreed API contract. The Anthropic OAuth risk must be resolved here before any downstream work assumes it. The SQLite schema, API spec, and backend scaffolding must exist before the sync engine is written.
**Delivers:** Working OAuth PKCE login → token in OS keychain; Supabase project with auth, Postgres schema, and Storage bucket; OpenAPI spec for sync endpoints; minimal backend (`PUT /files`, `GET /changes`, `GET /health`).
**Addresses:** Account login (table stakes), token security (critical pitfall), Anthropic OAuth validation (highest-risk open question).
**Avoids:** Building the client against stubs that don't exercise conflict and reconnection edge cases.
**Research flag:** Needs `--research-phase` — Anthropic OAuth endpoints are undocumented; must validate client ID reuse vs. new registration before implementation.

### Phase 2: Backend Core — Sync Store

**Rationale:** The server-side conflict model (monotonic version counter, 409 on version mismatch, content-addressed blob storage) must be correct and tested before the client sync engine consumes it.
**Delivers:** Full Supabase backend: metadata store with monotonic version counter per user, blob store integration, conflict detection returning 409, `GET /changes?since_version` endpoint, Supabase Realtime push for cross-device notifications.
**Implements:** Metadata Store, Blob Store, Auth Service components from ARCHITECTURE.md.
**Avoids:** Clock-skew LWW pitfall (server version wins, not client timestamp).

### Phase 3: Client Core — Sync Engine + File Watcher

**Rationale:** The client is built against the real backend. All correctness invariants go in here — this is the most dangerous phase if the pitfalls are ignored.
**Delivers:** Cross-platform file watcher (FSEvents + ReadDirectoryChangesW via notify crate), SQLite sync state DB, Sync Engine push flow (watch → debounce → hash → upload → update DB), Sync Engine pull flow (poll /changes → download → atomic write → update DB), write-lock flag preventing sync loops, idempotency keys on all uploads, structured sync event log.
**Addresses:** Background sync, safe sync, resume interrupted sync (all table stakes).
**Avoids:** Sync loop overwrite, partial write corruption, LWW clock skew, missing debounce, duplicate uploads on retry — all from PITFALLS.md Phase 1 non-deferrable list.
**Research flag:** macOS sandbox entitlement flow + security-scoped bookmark persistence need implementation-level research during phase planning.

### Phase 4: Resilience — Offline + Reconnection

**Rationale:** Network reliability is a precondition for user trust. The pull-then-push reconnection sequence and durable pending-ops queue distinguish a robust sync app from a fragile one.
**Delivers:** Network Monitor (online/offline via /health polling, 5s flap-debounce), durable pending-ops queue processing on reconnect, pull-then-push reconnection sequence with conflict handling for queued local ops, exponential backoff (2^n seconds, max 5 min, max 10 retries), Windows file-locking retry layer, Windows ReadDirectoryChangesW overflow detection + reconciliation sweep.
**Addresses:** Resume interrupted sync (table stakes), offline-aware operation (differentiator).
**Avoids:** Sign-out-while-offline data loss, Windows ReadDirectoryChangesW silent overflow.

### Phase 5: Packaging + UX Polish

**Rationale:** Ship when the sync engine is proven correct. Polish and distribution are the final gate before first real user.
**Delivers:** System tray / menu bar UI with 5 states (synced, syncing, paused, error, offline), onboarding wizard (auth → authorize → initial sync with progress → done), initial restore flow for new-device setup, macOS notarization + hardened runtime, Windows Authenticode signing, `tauri-plugin-updater` + GitHub Releases auto-update, launch-at-login (macOS LaunchAgent, Windows startup registry), per-category conflict strategy surfaced in UI, OS notifications for errors and conflicts only.
**Addresses:** All remaining table stakes: tray presence, status indicator, launch at login, deterministic initial sync progress, error surfacing, onboarding, settings persistence.
**Avoids:** macOS Gatekeeper notarization block, Windows SmartScreen friction.

### Phase Ordering Rationale

- Auth and API contract before everything because the sync engine cannot be correctly designed without knowing what version semantics the server will enforce.
- Backend before client because the 409 conflict model, monotonic version counter, and `GET /changes` endpoint shape the entire client pull flow — building the client first leads to rewrites.
- All PITFALLS.md Phase 1 non-deferrable items (sync loop, atomic write, LWW, debounce, idempotency, SQLite state, token storage, token refresh race) are structurally required in Phases 1–3.
- Resilience (Phase 4) is separated from core sync (Phase 3) because offline behavior adds significant complexity and is safely testable only once the happy path works end-to-end.
- Packaging last because Tauri's bundler, code signing, and auto-update are well-documented and do not affect sync correctness.

### Research Flags

Phases likely needing `--research-phase` during planning:
- **Phase 1:** Anthropic OAuth endpoint accessibility and client registration requirements — undocumented, highest-risk item in the stack; must be validated before any auth code is merged.
- **Phase 3:** macOS sandbox entitlement flow + security-scoped bookmark persistence for `~/Documents/Claude/` — implementation-level detail with known gotchas that can block the watcher silently.

Phases with standard patterns (skip research-phase):
- **Phase 2:** Supabase Postgres + Storage + Realtime are well-documented; monotonic version counter and content-addressed blob storage are standard patterns with extensive prior art.
- **Phase 4:** Exponential backoff, pending-ops queue, pull-then-push reconnection are documented patterns with reference implementations in Nextcloud, Syncthing.
- **Phase 5:** Tauri bundler, `tauri-plugin-updater`, macOS `notarytool`, and Windows Authenticode signing are all officially documented with step-by-step guides.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | MEDIUM-HIGH | Tauri/Supabase/notify crate are HIGH. Anthropic OAuth is MEDIUM — endpoints reverse-engineered, not officially documented. |
| Features | MEDIUM-HIGH | Patterns verified across Dropbox, OneDrive, Obsidian Sync docs and user feedback. Feature set is conventional for sync apps. |
| Architecture | MEDIUM-HIGH | Hub-and-spoke client-server is the canonical pattern; component boundaries drawn from Nextcloud, Dropbox, Syncthing architecture docs. |
| Pitfalls | HIGH | All major pitfalls sourced from official API docs, production bug reports, and post-mortems. ReadDirectoryChangesW overflow behavior from official Windows docs. |

**Overall confidence:** MEDIUM-HIGH

### Gaps to Address

- **Anthropic OAuth accessibility:** The client ID `9d1c250a-e61b-44d9-88ed-5944d1962f5e` belongs to Claude Code — a separate registration may be required. The fallback (Google/Apple OAuth via Supabase) must be designed in from the start. Validate in Phase 1 before any auth code is merged.

- **Cowork file locking behavior:** Does Claude Cowork hold POSIX advisory locks or Windows mandatory locks on JSONL session files while active? The Windows file-locking retry layer depends on the answer. Determine via empirical testing in Phase 3.

- **Cowork artifact file schema:** The exact schema of project artifact files inside `~/Documents/Claude/Projects/` is not fully documented. Must be reverse-engineered from a running Cowork instance before the sync engine can correctly categorize and hash these files. Address in Phase 3 planning.

- **Windows `%APPDATA%\Claude\` confirmation:** Unconfirmed as the Windows equivalent of `~/Library/Application Support/Claude/`. Confirm via testing before Phase 3.

- **Supabase Storage size budget:** Free tier is 500 MB. Cowork conversation histories can grow large. Assess typical project sizes and whether a per-file cap or paid tier is needed before Phase 2 completes.

- **Concurrent session safety:** LWW with server-version arbiter is safe for single-user multi-device when only one device is active. If Cowork can run on two devices simultaneously editing the same session file, confirm that the conflict-copy strategy is adequate. Validate during Phase 3 multi-device testing.

## Sources

### Primary (HIGH confidence)
- Tauri v2 official docs (v2.tauri.app) — framework, tray, plugin-fs, updater, code signing
- Supabase official docs — auth, Realtime, Storage, RLS
- RFC 8252 — OAuth 2.0 for Native Apps (PKCE loopback flow)
- Apple Developer docs — FSEvents, sandbox entitlements, notarization requirements
- Microsoft Docs — ReadDirectoryChangesW, Windows Credential Manager, Authenticode

### Secondary (MEDIUM confidence)
- Anthropic OAuth PKCE gist (cedws) — OAuth endpoint discovery, client ID
- `anthropic-auth` crate (crates.io) — PKCE implementation wrapping Anthropic endpoints
- Nextcloud Desktop Client Architecture Docs — component boundary patterns
- Syncthing architecture docs — delta sync strategy
- Dropbox / OneDrive community documentation — conflict copy and status indicator conventions
- DEV Community: Tauri menubar app gotchas, file watching with notify-rs
- GitHub issue #57177 (anthropics/claude-code) — confirmed hardcoded `~/Documents/Claude/` path

### Tertiary (LOW confidence, needs validation)
- claudelab.net — Cowork projects storage paths (community-sourced)
- code.claude.com/docs — Claude directory structure (may lag actual Cowork behavior)

---
*Research completed: 2026-05-30*
*Ready for roadmap: yes*
