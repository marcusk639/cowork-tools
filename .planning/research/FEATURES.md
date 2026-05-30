# Features Research — Cowork Sync Desktop App

**Domain:** Cross-device desktop sync for structured app data (not generic file sync)
**Researched:** 2026-05-30
**Confidence:** MEDIUM-HIGH — patterns verified across Dropbox, OneDrive, iCloud, Obsidian Sync, Sync.com documentation and user community feedback

---

## Table Stakes

Features users take for granted. Their absence triggers churn or distrust faster than any bug.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Automatic background sync | Every sync tool does this; users will not manually trigger sync | Low-Med | Triggered by file-system events + lifecycle hooks (Cowork open/close, app open/close) |
| Launch at login / run as daemon | Sync apps that require manual launch feel broken; users expect it to "just work" on any boot | Low | macOS: LaunchAgent; Windows: Task Scheduler or startup registry entry |
| System tray / menu bar presence | The canonical control surface for a background sync app; absence means no way to check status | Low | macOS menu bar top-right; Windows system tray bottom-right; platform conventions differ (color icon vs monochrome) |
| Status indicator (idle / syncing / error) | Users need to know the app is alive and current; the mental model breaks without this | Low | Three states minimum: synced (green check), syncing (spinner/arrows), error (red X or yellow warning) |
| Initial device setup pulls all data down | The core promise of the app — "install and everything is there" | Med | Must handle large initial payloads; show deterministic progress (bytes or file count), not a spinner |
| Error surfacing | Silent failures are worse than visible errors; users must know when sync is broken | Med | Distinguish transient errors (retry silently) from persistent errors (surface with action) |
| Safe sync — no data corruption or loss | Users trust sync apps with irreplaceable work; a single data-loss incident destroys trust permanently | High | Write-to-temp-then-atomic-rename pattern; validate writes before removing source; do not overwrite with older version |
| Resume interrupted sync | Network drops, sleeps, and reboots happen; the app must pick up where it left off | Med | Checkpoint progress; use delta/incremental sync, not full re-upload |
| Account login (Anthropic/Claude account) | The app's identity and access model depends on this; there is no fallback without it | Med | OAuth flow with token refresh; secure storage of credentials (Keychain / Windows Credential Manager) |
| Settings persistence | Users configure the app once and expect it to stay configured across restarts | Low | Stored locally; survives app update |

---

## Differentiators

Features that create trust and satisfaction beyond what users assume exists. These are where the product earns loyalty.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Per-entity conflict resolution (not just last-write-wins) | Generic LWW loses work; projects, settings, and conversation history have different merge semantics | High | Projects and settings: field-level merge. Conversation history: append-only (newest device's additions always additive). Files: LWW with conflict copy preserved. Requires understanding Cowork data schema first. |
| Conflict transparency — "here is what was different" | Users who encounter a conflict and see no explanation lose trust; a log or brief notification of what was resolved and how is rare but extremely valued | Med | Show: which device, what time, what data type, which version won. Keep conflict log browsable. |
| Sync health log (recent activity) | Dropbox-style activity pane showing last N events (uploaded, downloaded, conflicted) gives users confidence that sync is working | Low-Med | Accessible from tray/menu bar; at minimum: timestamp, data category, direction (up/down), result |
| Offline-aware operation | App should behave gracefully when offline — queue changes locally, sync when connection resumes — not error-state out | Med | Local write queue; reconnection detection; deduplication of queued vs. already-synced changes |
| Deterministic initial sync progress | "Setting up your devices" with a real progress indicator (files remaining or % of data) dramatically reduces anxiety on first use | Low | Show count or percentage; estimate time remaining; allow background use of host app while sync completes |
| Pause sync | Power users on metered connections or slow laptops need to pause sync without quitting | Low | Single tray menu item; resume on same selection; state persists across restarts |
| Notifications only on errors and conflicts | Excessive notifications train users to ignore them; silence on routine sync, surface only errors and conflicts | Low | Use OS notification system for errors/conflicts; no sound or banner for normal sync |
| Clear onboarding explaining what the app does and what it will access | Sync apps that silently start writing files feel like malware; explicit authorization builds trust | Low-Med | Show what data categories will be synced; require explicit "authorize access" step; confirm after first successful sync |

---

## Anti-Features (Don't Build in v1)

These are explicitly excluded. Each has a reason and a "what to do instead."

| Anti-Feature | Why Exclude | What to Do Instead |
|--------------|-------------|-------------------|
| Selective sync (choose which projects/files to sync) | Adds UI complexity, edge cases (partially-synced state is hard to reason about), and support burden | Sync everything in scope for v1; add selective sync only when users demonstrate they need it |
| Bandwidth throttle controls | Adds settings surface complexity; the user population (personal/internal) is small and unlikely to hit bandwidth contention | Let the OS handle it; implement polite background-priority uploads instead (low-priority queuing) |
| Version history / file restore | Powerful safety net but requires backend storage design, versioning API, retention policy, and a restore UI — out of scope for MVP | Implement conflict-copy preservation (keep old version as a `.conflict` file locally) as a lightweight substitute |
| Sharing / multi-user sync | Project scope is personal/internal; adding sharing requires ACL design, invitation flows, permission management | Single-user sync only; revisit post-launch |
| User-provided storage backends (iCloud, S3, Dropbox) | Adds integration complexity and removes control of the data contract | Own the backend; this is explicitly documented as out of scope in PROJECT.md |
| Real-time collaborative editing | Cowork is not a collaborative editor in this context; live co-editing requires CRDT or OT, far beyond sync scope | Sync after edits are complete; detect concurrent edits and resolve via conflict strategy |
| Mobile sync (iOS, Android) | Out of scope per PROJECT.md; adds platform surface, MDM complexity, and mobile-specific storage access challenges | Desktop-only for v1 |
| Admin dashboard / usage analytics | Internal/personal use tool; no operators, no user management | Not needed; add if the app moves to public multi-user distribution |
| Offline-first architecture with full local replica | Adds significant complexity; the app's value is sync, not offline independence | Require network for first sync; cache last-known state locally for resilience, but don't design for prolonged offline use |

---

## Conflict Resolution Patterns

How similar tools handle the problem of two devices modifying the same data before either syncs.

### The Core Strategies (and Their Trade-offs)

**Last-Write-Wins (LWW)**
The version with the later timestamp is kept; the other is discarded. Simple to implement, zero user interaction required. Risk: silently destroys work when both writes are valid. Appropriate only when data is append-only or when the user is unlikely to edit the same item simultaneously on two devices.

**Conflict Copy Preservation (Dropbox Model)**
Both versions are kept. The original retains its name; the second version is saved as `filename (conflicted copy from Device B, 2026-05-29).ext`. No data is lost, but the user must manually reconcile. This is the safest strategy for files where merging is not possible. Dropbox's biggest user complaint is that conflicted copies accumulate silently — users find dozens of them later and don't know which to keep.

**Field-Level Merge (Structured Data)**
For JSON/structured data (settings, project metadata), compare fields independently. If Device A changed `theme` and Device B changed `fontSize`, both changes can be applied without conflict. Only flag a conflict when the same field was changed on both devices to different values. This is the right strategy for Cowork settings.

**Append-Only (Event/Log Data)**
Conversation history is naturally append-only: Device A's conversations and Device B's conversations are both valid and both should be preserved. Merge by union, deduplicated by a stable identifier (conversation ID or timestamp+device hash). No conflict in the traditional sense.

### Recommended Strategy for Cowork Data

| Data Category | Strategy | Rationale |
|---------------|----------|-----------|
| Settings | Field-level merge; LWW on same-field conflict | Settings changes are rare; LWW on field level has negligible data loss risk |
| Project metadata | Field-level merge; LWW on same-field conflict | Projects are structured; different fields can be merged safely |
| Files within projects | LWW with conflict copy preserved locally | Files are opaque blobs; cannot merge; preserve both, surface notice |
| Conversation history | Append-only union by conversation ID | History is additive; both devices' conversations are valid |

**Key insight from research:** Never silently discard data. If the strategy loses data, tell the user what was discarded, when, and why. Obsidian Sync and Dropbox both earn criticism when conflicts are resolved silently.

---

## Status and Feedback Patterns

What users expect to see from a background sync process, based on behavior of Dropbox, OneDrive, and Obsidian Sync.

### Visual States (Minimum Required)

| State | Icon Convention | When Shown |
|-------|----------------|-----------|
| Synced / Up to date | Green checkmark or static cloud | All changes pushed and pulled; nothing pending |
| Syncing | Animated arrows or spinner | Upload or download in progress |
| Paused | Static icon, no animation, "Paused" label | User manually paused |
| Error | Red X or yellow warning triangle | Persistent error requiring user action |
| Offline / Waiting | Grey or muted icon | No network connection; will retry |

Platform conventions matter: macOS menu bar icons should be monochrome (SF Symbols or template images) and respect Dark Mode. Windows system tray icons are typically color.

### Tray / Menu Bar Menu (Click on Icon)

Expected items, in order:

1. Status summary line ("Up to date" / "Syncing 3 items..." / "Error: see details")
2. Last synced timestamp ("Last sync: 2 minutes ago")
3. Separator
4. Recent activity (last 5 events inline, or "View activity log" link)
5. Separator
6. Pause Sync / Resume Sync toggle
7. Open Cowork (shortcut to the main app)
8. Separator
9. Preferences
10. Quit

### Notification Strategy

Notify via OS notification system (macOS Notification Center / Windows Action Center) for:
- Persistent sync errors (with a call-to-action: "Open to fix")
- Conflict detected and preserved (brief, non-urgent)
- First successful sync on a new device ("Everything is ready")

Do NOT notify for:
- Routine sync cycles completing
- Individual file uploads/downloads
- App starting or stopping

**Rationale:** Excessive notifications are the top complaint in Dropbox community forums. Users disable notifications entirely when trained to ignore them, which means they then miss real errors.

### Activity Log

Show in a scrollable pane accessible from the tray menu:
- Timestamp
- Data category (Settings, Projects, Files, History)
- Direction (uploaded / downloaded / conflict resolved)
- Result (success / failed / conflict)

Retain last 100 events. No need for export or persistent history in v1.

---

## Onboarding Patterns

How similar tools handle first-run setup, and what this app should do.

### Phase 1: Installation and First Launch

**What users expect:**
- Standard installer (`.dmg` on macOS, `.exe` on Windows); no manual path manipulation
- App launches immediately after install with a welcome/setup wizard
- Not: silently starting in the background with no UI

**Pattern from research:** Dropbox opens a browser-based sign-in on first launch, then returns to the desktop app with authorization complete. Obsidian Sync uses an in-app account modal. The key is: never make users leave to a browser for a critical step without explaining why.

### Phase 2: Authentication

Steps:
1. Show "Sign in with your Claude / Anthropic account" button prominently
2. Open OAuth flow (browser redirect or embedded webview — webview preferred to avoid context switch)
3. On callback, store token securely (Keychain / Windows Credential Manager)
4. Confirm identity visually ("Signed in as [email]") before proceeding

**Critical:** Prime users before requesting permissions. Show a screen explaining "This app will access your Cowork data at [path]" BEFORE the OS file-access permission dialog appears. Cold permission dialogs that appear without context are the top onboarding friction point.

### Phase 3: Data Authorization and Initial Sync

Steps:
1. Show what data categories will be synced (Projects, Files, Settings, Conversation History)
2. Show the local path(s) that will be monitored
3. Explicit "Start Sync" button — do not begin reading files without user confirmation
4. Show initial sync progress with a real indicator:
   - "Scanning your Cowork data..." (fast, usually seconds)
   - "Uploading X items..." with count or percentage
   - Do NOT use an indeterminate spinner for a step that takes minutes
5. On completion: "Everything is synced. The app will keep your data up to date in the background."

**Pattern from research:** Google's Open Health Stack guidelines specifically call out that "initial sync should be a distinct step" with guidance on timing and a deterministic progress view. Apps that use a spinner for a multi-minute process cause users to abandon setup or force-quit.

### Phase 4: Subsequent Device Setup (The Core Value Moment)

This is the scenario the entire app exists for: user installs on a second machine.

1. Same install + login flow as above
2. After authentication, app detects prior sync state in cloud
3. Show: "We found your Cowork data. Downloading X items to get you set up." with progress
4. On completion: "Done — your projects, settings, and history are ready."
5. Open Cowork (or prompt user to open it)

**Key:** The transition from "empty machine" to "fully populated Cowork" should feel like a reveal, not a process. Make the completion state visible and celebratory. This is the moment that validates the app's existence.

### What NOT to Do in Onboarding

- Do not ask users to configure sync frequency, bandwidth limits, or conflict preferences in onboarding — defer to defaults
- Do not show technical paths or file system details unless explicitly in a "details" expander
- Do not leave users on a blank tray icon with no indication of what happened
- Do not ask users to "restart for changes to take effect" — handle this in the installer

---

## Feature Dependencies

```
Account Login
  └─> Initial Sync (requires auth token)
        └─> Conflict Resolution (requires knowing what exists server-side)
        └─> Activity Log (begins populating on first sync)

File System Watcher
  └─> Background Sync (change detection is how sync is triggered)
        └─> Status Indicator (syncing state comes from watcher events)
        └─> Conflict Detection (requires comparing local change with server state)

Status Indicator
  └─> Tray / Menu Bar Icon (the indicator IS the icon)
        └─> Activity Log (accessible from tray menu)
        └─> Pause/Resume (accessible from tray menu)

Error Detection
  └─> OS Notification (only meaningful if errors are surfaced)
  └─> Activity Log (errors appear in log)
```

---

## MVP Recommendation

Build in this priority order:

**Must ship (table stakes):**
1. Account login via Anthropic OAuth
2. Launch at login + tray icon with three states (synced / syncing / error)
3. File system watcher → upload delta to backend on change
4. Pull from backend on Cowork open/close and app open
5. Initial sync with deterministic progress display
6. Onboarding wizard (auth → authorize → initial sync → done)
7. Per-category conflict strategy (LWW for files, field merge for settings, append for history)
8. Error surfacing via tray + OS notification

**Defer (differentiators, phase 2):**
- Activity log (tray-accessible)
- Offline queue with reconnection sync
- Conflict transparency notifications
- Pause/resume

**Do not build in v1:**
- Version history, selective sync, bandwidth controls, sharing, mobile

---

## Sources

- [Dropbox conflicted copy documentation](https://help.dropbox.com/organize/conflicted-copy)
- [OneDrive status icons reference](https://support.microsoft.com/en-us/office/what-do-the-onedrive-icons-mean-11143026-8000-44f8-aaa9-67c985aa49b3)
- [Sync.com desktop application overview](https://help.sync.com/hc/en-us/articles/38275586765075-The-Sync-desktop-application)
- [Conflict resolution strategies — Mobterest Studio / Medium](https://mobterest.medium.com/conflict-resolution-strategies-in-data-synchronization-2a10be5b82bc)
- [Offline sync developer guide — daily.dev](https://daily.dev/blog/offline-file-sync-developer-guide-2024/)
- [Google Open Health Stack — Design guidelines for offline and sync](https://developers.google.com/open-health-stack/design/offline-sync-guideline)
- [Dropbox new user onboarding — Product Onboarding](https://productonboarding.com/examples/dropbox-new-user-onboarding)
- [Obsidian Review 2026 — Cloudwards](https://www.cloudwards.net/obsidian-review/)
- [Obsidian Review — The Business Dive](https://thebusinessdive.com/obsidian-review)
- [Silent sync failure scenario — Inkdrop forum](https://forum.inkdrop.app/t/silent-sync-failure-behind-corporate-proxy-data-loss-scenario/2206)
- [UX patterns for loading — Pencil & Paper](https://www.pencilandpaper.io/articles/ux-pattern-analysis-loading-feedback)
- [Handling large model downloads UX — SitePoint](https://www.sitepoint.com/ux-patterns-large-model-downloads/)
- [macOS autostart: LaunchAgent vs LaunchDaemon — Eclectic Light Company](https://eclecticlight.co/2018/05/22/running-at-startup-when-to-use-a-login-item-or-a-launchagent-launchdaemon/)
