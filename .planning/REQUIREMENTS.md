# Requirements: Cowork Tools

**Defined:** 2026-05-30
**Core Value:** A user's Cowork work is never lost when switching devices — install the app and everything is there.

## v1 Requirements

### Authentication

- [ ] **AUTH-01**: User can log in with their Claude/Anthropic account via OAuth PKCE flow
- [ ] **AUTH-02**: User can log in with Google OAuth if Anthropic OAuth is unavailable
- [ ] **AUTH-03**: User can log in with Apple OAuth if Anthropic OAuth is unavailable
- [ ] **AUTH-04**: Auth tokens are stored in the OS keychain (macOS Keychain / Windows Credential Manager), never in plaintext
- [ ] **AUTH-05**: User session persists across app restarts without re-authentication

### Sync Core

- [ ] **SYNC-01**: App detects changes in Cowork data directories (~/Documents/Claude/, ~/.claude/, ~/Library/Application Support/Claude/ on macOS; equivalents on Windows) and syncs them to the cloud backend in real time
- [ ] **SYNC-02**: App syncs when Cowork opens or closes (lifecycle trigger)
- [ ] **SYNC-03**: App syncs when the sync app itself opens or closes (lifecycle trigger)
- [ ] **SYNC-04**: File writes are atomic — write to temp file, fsync, then rename — never in-place
- [ ] **SYNC-05**: A write-lock flag prevents the app from re-uploading files it just downloaded (no sync loops)
- [ ] **SYNC-06**: Conflict resolution uses server-assigned monotonic version numbers, not client timestamps
- [ ] **SYNC-07**: Pending changes are queued in a local SQLite database and survive app restarts
- [ ] **SYNC-08**: Queued changes are pushed to the backend automatically on reconnect (pull-then-push sequence)

### Device Restore

- [ ] **REST-01**: On a new device, after login the app downloads all synced data to the correct Cowork paths
- [ ] **REST-02**: Initial restore shows deterministic progress so the user knows how long to wait

### Status & Feedback

- [ ] **STAT-01**: App displays a system tray / menu bar icon with at least three states: synced, syncing, error
- [ ] **STAT-02**: App sends an OS notification when sync fails or a conflict is detected

### Distribution

- [ ] **DIST-01**: App launches at login automatically (macOS LaunchAgent, Windows startup registry)
- [ ] **DIST-02**: First-run onboarding wizard walks the user through: login → authorize data access → initial sync with progress → ready

## v2 Requirements

### Status & Feedback

- **STAT-03**: Sync activity log showing the last N sync events, accessible from the tray menu
- **STAT-04**: Pause/resume sync toggle in the tray menu

### Conflict Transparency

- **CONF-01**: Per-category conflict strategy — LWW + conflict copy for opaque files, field-level merge for settings, append-only union for conversation history
- **CONF-02**: Conflict notification explains what was resolved and how

### Distribution

- **DIST-03**: Auto-update via tauri-plugin-updater + GitHub Releases

### Sync

- **SYNC-09**: Selective sync — user can choose which Cowork projects to sync
- **SYNC-10**: Bandwidth throttle controls

## Out of Scope

| Feature | Reason |
|---------|--------|
| Mobile (iOS, Android) | Desktop-first for v1; mobile adds separate platform complexity |
| Linux desktop | macOS and Windows cover the initial audience |
| User-provided storage (iCloud, Dropbox, S3) | We own the backend for simplicity and data contract control |
| Public release / user management | Personal/internal first; multi-user distribution comes later |
| Version history / file restore UI | Adds significant UI complexity; the sync backend can log history without exposing it |
| Web app | This is a native desktop daemon, not a browser tool |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| AUTH-01 | Phase 1 | Pending |
| AUTH-02 | Phase 1 | Pending |
| AUTH-03 | Phase 1 | Pending |
| AUTH-04 | Phase 1 | Pending |
| AUTH-05 | Phase 1 | Pending |
| SYNC-06 | Phase 2 | Pending |
| SYNC-01 | Phase 3 | Pending |
| SYNC-02 | Phase 3 | Pending |
| SYNC-03 | Phase 3 | Pending |
| SYNC-04 | Phase 3 | Pending |
| SYNC-05 | Phase 3 | Pending |
| SYNC-07 | Phase 4 | Pending |
| SYNC-08 | Phase 4 | Pending |
| REST-01 | Phase 5 | Pending |
| REST-02 | Phase 5 | Pending |
| STAT-01 | Phase 5 | Pending |
| STAT-02 | Phase 5 | Pending |
| DIST-01 | Phase 5 | Pending |
| DIST-02 | Phase 5 | Pending |

**Coverage:**
- v1 requirements: 19 total
- Mapped to phases: 19
- Unmapped: 0 ✓

---
*Requirements defined: 2026-05-30*
*Last updated: 2026-05-30 after roadmap creation (traceability populated)*
