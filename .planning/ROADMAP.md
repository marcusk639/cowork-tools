# Roadmap: Cowork Sync Desktop App

**Milestone:** v1 — Ship a working cross-device sync daemon
**Mode:** mvp
**Granularity:** standard
**Coverage:** 19/19 v1 requirements mapped

---

## Phases

- [ ] **Phase 1: Foundation** — Auth + Data Contract (Anthropic OAuth PKCE validated, tokens in OS keychain, Supabase project scaffolded, minimal backend API spec agreed)
- [ ] **Phase 2: Backend Core** — Sync Store (monotonic version counter, content-addressed blob storage, 409 conflict detection, Realtime push, tested in isolation)
- [ ] **Phase 3: Client Core** — Sync Engine + File Watcher (Tauri app, cross-platform watcher, SQLite sync DB, push/pull flows with all correctness invariants in place)
- [ ] **Phase 4: Resilience** — Offline + Reconnection (durable pending-ops queue, pull-then-push reconnection, exponential backoff, Windows overflow detection)
- [ ] **Phase 5: Packaging + UX Polish** — Tray UI, onboarding wizard, initial restore, code signing, launch-at-login, OS notifications

---

## Phase Details

### Phase 1: Foundation
**Goal**: A user can authenticate with their Anthropic account and have their token stored securely, and the backend API contract is defined so all downstream work builds against a real spec.
**Mode:** mvp
**Depends on**: Nothing (first phase)
**Requirements**: AUTH-01, AUTH-02, AUTH-03, AUTH-04, AUTH-05
**Success Criteria** (what must be TRUE):
  1. User can open the app and complete OAuth PKCE login with their Anthropic account; the access token lands in macOS Keychain or Windows Credential Manager — never on disk in plaintext
  2. If Anthropic OAuth is unavailable, user can complete login via Google OAuth and reach the same authenticated state
  3. If Anthropic OAuth is unavailable, user can complete login via Apple OAuth and reach the same authenticated state
  4. App restarts without prompting for login again — the session is fully restored from the OS keychain
  5. A minimal backend (Supabase project + OpenAPI spec with PUT /files, GET /changes, GET /health) is live and the auth token issued in criterion 1 is accepted by it
**Plans**: TBD

### Phase 2: Backend Core
**Goal**: The cloud backend correctly stores file blobs, assigns monotonic version numbers, detects conflicts, and pushes change notifications — so the client sync engine in Phase 3 can be built against a correct and tested server.
**Mode:** mvp
**Depends on**: Phase 1
**Requirements**: SYNC-06
**Success Criteria** (what must be TRUE):
  1. Uploading a file twice with a stale version number returns HTTP 409 (Conflict) — never silently overwrites the newer server version
  2. Each accepted upload increments a per-user monotonic version counter that is returned to the client; no two accepted uploads share the same version number
  3. A second client subscribed to Supabase Realtime receives a push notification within 5 seconds of a successful upload from the first client
  4. Content-addressed blob storage deduplicates identical file content (same SHA-256 hash → same stored object)
**Plans**: TBD

### Phase 3: Client Core
**Goal**: The Tauri desktop app watches Cowork data directories, pushes local changes to the backend, and pulls remote changes to disk — correctly and without sync loops, partial writes, or data corruption — on both macOS and Windows.
**Mode:** mvp
**Depends on**: Phase 2
**Requirements**: SYNC-01, SYNC-02, SYNC-03, SYNC-04, SYNC-05
**Success Criteria** (what must be TRUE):
  1. Saving a file in ~/Documents/Claude/ (macOS) or the Windows equivalent triggers an upload within 2 seconds, with no manual action required
  2. A change pushed from Device A appears at the correct Cowork path on Device B within 10 seconds, written atomically (temp file → fsync → rename) with no intermediate corrupted state visible to Cowork
  3. The app syncs automatically when Cowork opens or closes (lifecycle trigger), and when the sync app itself opens or closes
  4. After the app writes a downloaded file to disk, that file is not re-queued for upload — the write-lock flag prevents the sync loop in all tested scenarios
  5. On macOS, the file watcher operates correctly without triggering sandbox permission errors for ~/Documents/Claude/, ~/Library/Application Support/Claude/, and ~/.claude/
**Plans**: TBD
**UI hint**: yes

### Phase 4: Resilience
**Goal**: The sync app handles network loss and reconnection gracefully — pending changes survive app restarts and are reliably delivered once connectivity is restored, in the correct order.
**Mode:** mvp
**Depends on**: Phase 3
**Requirements**: SYNC-07, SYNC-08
**Success Criteria** (what must be TRUE):
  1. Changes made while offline are written to the local SQLite pending-ops queue and survive an app restart; they are delivered to the backend in full after reconnection
  2. On reconnect, the app completes a pull of remote changes before flushing the local queue — a queued local op never blindly overwrites a newer server version received in the same reconnection cycle
  3. A transient network error triggers exponential backoff (2^n seconds, max 5 min) and automatic retry; the user does not need to take any action
  4. On Windows, ReadDirectoryChangesW buffer overflow triggers a full reconciliation sweep rather than silently missing events
**Plans**: TBD

### Phase 5: Packaging + UX Polish
**Goal**: A real user can install the app on a new device, complete onboarding, have all their Cowork data restored to the right locations with visible progress, and trust the app to run silently in the background from that point on.
**Mode:** mvp
**Depends on**: Phase 4
**Requirements**: REST-01, REST-02, STAT-01, STAT-02, DIST-01, DIST-02
**Success Criteria** (what must be TRUE):
  1. Installing the app on a new device and logging in downloads all previously synced Cowork data to the correct paths; the user does not need to manually specify locations
  2. During initial restore, the app displays deterministic progress (files remaining or percentage) so the user knows it is working and can estimate completion time
  3. The system tray / menu bar icon reflects at least three distinct visual states — synced, syncing, and error — and updates within 2 seconds of a state change
  4. When a sync failure or conflict is detected, the OS delivers a notification to the user; silent failures do not occur
  5. The app is present in the macOS login items / Windows startup entries after installation and launches automatically on the next login without user configuration
  6. A first-run onboarding wizard guides the user through: login → authorize data access → initial sync with progress → ready state — without requiring external documentation
**Plans**: TBD
**UI hint**: yes

---

## Progress

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation | 0/? | Not started | - |
| 2. Backend Core | 0/? | Not started | - |
| 3. Client Core | 0/? | Not started | - |
| 4. Resilience | 0/? | Not started | - |
| 5. Packaging + UX Polish | 0/? | Not started | - |

---

## Coverage

| Requirement | Phase | Notes |
|-------------|-------|-------|
| AUTH-01 | Phase 1 | Anthropic OAuth PKCE flow |
| AUTH-02 | Phase 1 | Google OAuth fallback |
| AUTH-03 | Phase 1 | Apple OAuth fallback |
| AUTH-04 | Phase 1 | OS keychain token storage |
| AUTH-05 | Phase 1 | Session persistence across restarts |
| SYNC-06 | Phase 2 | Server-assigned monotonic version numbers (backend) |
| SYNC-01 | Phase 3 | File watcher, real-time push |
| SYNC-02 | Phase 3 | Cowork open/close lifecycle trigger |
| SYNC-03 | Phase 3 | Sync app open/close lifecycle trigger |
| SYNC-04 | Phase 3 | Atomic writes (temp → fsync → rename) |
| SYNC-05 | Phase 3 | Write-lock flag prevents sync loops |
| SYNC-07 | Phase 4 | SQLite pending-ops queue, survives restarts |
| SYNC-08 | Phase 4 | Pull-then-push reconnection |
| REST-01 | Phase 5 | New device restore to correct paths |
| REST-02 | Phase 5 | Deterministic restore progress display |
| STAT-01 | Phase 5 | Tray icon with synced/syncing/error states |
| STAT-02 | Phase 5 | OS notification on sync failure or conflict |
| DIST-01 | Phase 5 | Launch at login (LaunchAgent / startup registry) |
| DIST-02 | Phase 5 | First-run onboarding wizard |

**Total mapped: 19/19** ✓

---

*Roadmap created: 2026-05-30*
*Last updated: 2026-05-30 after initial creation*
