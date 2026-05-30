# Stack Research — Cowork Sync Desktop App

**Researched:** 2026-05-30
**Overall confidence:** MEDIUM-HIGH (desktop framework HIGH, auth MEDIUM, backend MEDIUM, Cowork paths MEDIUM)

---

## Recommended Stack

### Desktop Framework

**Recommendation: Tauri 2.11.2**

Use Tauri v2 (current stable: 2.11.2, released Oct 2024). This is a background tray/menu-bar daemon with a small popup UI — exactly the use case where Tauri's architecture wins decisively over Electron.

**Why Tauri over Electron:**
- Installer size: <10 MB vs Electron's 100–150 MB. A sync daemon does not need Chromium bundled.
- Memory at idle: ~30–50 MB vs Electron's 150–300 MB. This app runs continuously in the background; Electron's memory footprint is unjustifiable for a tray icon.
- Native WebView (WKWebView on macOS, WebView2 on Windows) is sufficient for a simple status popup — no need for a full Chromium instance.
- Tauri v2 has first-class tray/menu-bar support via `TrayIconBuilder`. Setting `LSUIElement: true` in `tauri.conf.json` hides the app from the macOS Dock and Cmd+Tab — the correct behavior for a background daemon.
- The file-watching logic lives in Rust (see below), so there is no "we need Node.js for the backend" reason to pick Electron.

**Why not Electron:**
- Bundles Chromium + Node.js per app. For a background sync daemon this is waste, not benefit.
- No advantage for this use case; Electron's larger ecosystem only matters for feature-heavy UIs.

**Why not native Swift/C++:**
- macOS-only. The requirement is macOS + Windows from a single codebase.

**Tauri v2 tray configuration (macOS):**
```json
{
  "bundle": {
    "macOS": {
      "infoPlist": {
        "LSUIElement": true
      }
    }
  }
}
```

**Confidence: HIGH** — Tauri 2.0 stable shipped October 2024, 2.11.x is the active patch series as of May 2026. Ecosystem is mature enough for a tray-only app. Large apps (1Password, Hoppscotch) have validated the approach.

---

### File System Watching

**Recommendation: Tauri's `@tauri-apps/plugin-fs` (v2.4.5) with `watch`/`watchImmediate` — backed by the Rust `notify` crate**

Because the app is built on Tauri, the correct watcher is the one Tauri provides natively: `@tauri-apps/plugin-fs` exposes `watch()` and `watchImmediate()` from JavaScript, which delegates to the `notify` crate in Rust. This means:

- On macOS: uses FSEvents (kernel-level, low overhead, recursive).
- On Windows: uses ReadDirectoryChangesW (kernel-level, low overhead, recursive).
- No polling. No extra process. No dependencies outside the Tauri plugin system.
- Debouncing is required — FSEvents and ReadDirectoryChangesW both fire duplicate events. Build debounce in from day one (recommend 300–500 ms settle window before triggering a sync).

**Do not use chokidar** in the Tauri context. Chokidar v5 (Nov 2025) is ESM-only, requires Node.js v20+, and is the right choice for a pure Node.js or Electron app. In Tauri, pulling in a Node.js file watcher adds a redundant abstraction layer over what the Rust `notify` crate already handles natively and more efficiently.

**`@parcel/watcher`** is an excellent alternative if the project ever pivots to Electron — it has better Windows performance than chokidar and optional Watchman backend. Not relevant for Tauri.

**Implementation note from field reports:** Use `std::thread::spawn` (not Tokio async) for the blocking watcher loop in Rust to avoid blocking the main runtime. Filter aggressively on file extensions and paths to avoid reacting to temp files, lock files, and `.DS_Store` churn.

**Confidence: HIGH** — `@tauri-apps/plugin-fs` v2.4.5 is the current official Tauri plugin; `notify` crate is the de facto Rust fs watching library (6+ years production history).

---

### Sync Backend

**Recommendation: Supabase (managed) for auth + database + realtime + storage**

**Architecture:**
```
Desktop app
  └─ Tauri frontend (TypeScript + Rust)
       ├─ Auth: Supabase Auth (JWT, refresh tokens)
       ├─ File metadata: Supabase Postgres (projects, paths, hashes, sync state)
       ├─ Real-time change notifications: Supabase Realtime (WebSocket)
       └─ File blobs: Supabase Storage (S3-compatible, CDN-backed)
```

**Why Supabase over Cloudflare Workers + R2 + Durable Objects:**
- This is an internal/personal tool for one user (per PROJECT.md). The Supabase free tier covers 500 MB storage, 2 GB bandwidth, and unlimited auth — enough to run indefinitely at personal scale.
- Supabase bundles auth, Postgres, realtime WebSockets, and file storage in one product. Cloudflare's equivalent requires assembling Workers + D1 + R2 + Durable Objects — four products with separate billing, documentation, and failure modes. Supabase is right-sized for this use case.
- Supabase Realtime provides WebSocket subscriptions to Postgres table changes out of the box. When Device A syncs, Device B's desktop app receives a push notification via the realtime channel and pulls the delta — no polling, no custom WebSocket server to maintain.
- Supabase Storage integrates with Row Level Security (RLS) so the same JWT used for auth gates file access with zero extra middleware.

**Sync data model (high level):**
- `sync_files` table: `(id, user_id, device_id, path, content_hash, size, modified_at, synced_at, blob_storage_key)`
- File content stored in Supabase Storage bucket (`cowork-files/{user_id}/{hash}`)
- Content-addressed storage (keyed by hash) gives deduplication and idempotent re-upload for free
- Last-write-wins conflict resolution keyed on `modified_at` for v1 — sufficient for the single-user case described in PROJECT.md

**Why not build a custom WebSocket server:**
- Unnecessary complexity for a personal sync tool. Supabase Realtime handles fan-out to multiple devices. A custom server adds infrastructure to maintain.

**Cloudflare Workers + R2 is the right upgrade path** if this scales to many users and egress costs become a concern (R2 has zero egress fees vs Supabase's metered egress). The migration path is clean because the architecture above separates concerns.

**Supabase JS client:** `@supabase/supabase-js` v2.x (current as of May 2026).

**Confidence: MEDIUM** — Supabase Realtime and Storage are production-grade. The specific sync data model is a design decision, not a proven template. Conflict resolution strategy needs validation once Cowork's actual file format is confirmed.

---

### Auth

**Recommendation: Anthropic OAuth 2.0 with PKCE (primary) + Supabase Auth custom provider (session management)**

**Two-layer auth strategy:**

**Layer 1 — Identity: Anthropic/Claude OAuth 2.0 + PKCE**
The Anthropic OAuth flow used by Claude Code is documented and functional:
- Authorization endpoint: `https://claude.ai/oauth/authorize`
- Token endpoint: `https://console.anthropic.com/v1/oauth/token`
- Client ID (Claude Code's): `9d1c250a-e61b-44d9-88ed-5944d1962f5e`
- Scopes: `org:create_api_key user:profile user:inference`
- PKCE method: S256
- Redirect: loopback on ephemeral port (`http://localhost:{port}/callback`)

The Rust crate `anthropic-auth` (crates.io) provides an async PKCE implementation wrapping these exact endpoints, making Tauri integration straightforward.

**Layer 2 — Session management: Supabase Auth custom JWT**
After obtaining the Anthropic OAuth token, exchange it for a Supabase session using a custom auth function (Supabase Edge Function verifies the Anthropic token and mints a Supabase JWT). This gives the desktop client a Supabase JWT for all subsequent database and storage operations, with automatic refresh handled by `@supabase/supabase-js`.

**Why this split:**
- Anthropic OAuth proves identity and grants access to the user's Cowork data.
- Supabase Auth handles session lifecycle (refresh tokens, expiry, RLS enforcement on database/storage).
- Avoids building your own session store.

**Google/Apple OAuth fallback:** Supabase Auth supports Google and Apple as built-in providers. These can be offered as fallbacks if the Anthropic OAuth integration hits undocumented API restrictions.

**Critical caveat:** The Anthropic OAuth endpoints reverse-engineered from Claude Code (client ID `9d1c250a-e61b-44d9-88ed-5944d1962f5e`) are not documented in Anthropic's public API docs. They are inferred from Claude Code's behavior. This is the highest-risk item in the stack — Anthropic could change or restrict these endpoints without notice. **This must be validated in Phase 1 before any other work depends on it.**

**Confidence: MEDIUM** — OAuth endpoints confirmed by community reverse-engineering and existing OSS implementations (`anthropic-auth` crate). Not confirmed by official Anthropic API documentation. Risk: undocumented API. Mitigation: build the auth abstraction layer so Google/Apple OAuth can replace Anthropic OAuth if needed.

---

### Packaging and Auto-Update

**Recommendation: Tauri's built-in bundler + `tauri-plugin-updater` + GitHub Releases as update server**

**Packaging:**
Tauri's bundler produces platform-native installers with a single `tauri build` command:
- macOS: `.dmg` + `.app` bundle, notarized via Apple Developer account ($99/yr)
- Windows: `.msi` (NSIS installer) + `.exe`, signed via Authenticode

No additional tool (electron-builder, etc.) is needed — this is built into Tauri.

**Auto-update:**
`tauri-plugin-updater` checks a JSON endpoint for new versions and downloads/applies updates in the background. The simplest hosting is a GitHub Release with a `latest.json` manifest (update-server pattern). No update server to maintain.

```json
// latest.json (served from GitHub Releases or static host)
{
  "version": "1.2.0",
  "platforms": {
    "darwin-aarch64": { "url": "...", "signature": "..." },
    "windows-x86_64": { "url": "...", "signature": "..." }
  }
}
```

Updates must be signed with a Tauri updater key pair (generated once, private key stored in CI secrets). This prevents update hijacking.

**CI/CD:** GitHub Actions with `tauri-action` handles cross-platform builds. macOS arm64 + x86_64 universal binary, Windows x86_64. The 2-part guide at DEV Community covers the full pipeline including code signing secrets.

**Code signing requirements:**
- macOS: Apple Developer account, Developer ID certificate, notarization via `xcrun notarytool`
- Windows: Authenticode certificate (OV cert avoids SmartScreen warnings on first run; EV cert eliminates them entirely but requires physical hardware key)

**Confidence: HIGH** — Tauri's bundler and updater are core, documented, battle-tested features of v2. Code signing requirements are standard Apple/Microsoft requirements, not Tauri-specific risks.

---

## What NOT to Use

| Technology | Reason |
|------------|--------|
| **Electron** | Ships 100–150 MB Chromium runtime for what is essentially a tray icon + small popup. Memory footprint (150–300 MB idle) is unjustifiable for a background daemon. No upside for this use case. |
| **chokidar (Node.js)** | Right tool for Node.js/Electron apps, wrong tool for Tauri. The Rust `notify` crate does the same thing natively without bridging through Node. Chokidar v5 is also ESM-only, adding a module format constraint. |
| **Cloudflare Workers + R2 + Durable Objects** | Correct architecture at scale; over-engineered for a personal/internal tool. Assembling four separate Cloudflare products (Workers, D1, R2, Durable Objects) adds operational complexity that Supabase eliminates for free at this scale. Revisit if the tool goes public. |
| **Custom WebSocket server** | Supabase Realtime provides WebSocket-based change notifications built on Postgres. Building a custom server adds infrastructure with no benefit. |
| **SQLite for sync state** | Tempting as a local cache, but introduces a secondary consistency surface. The sync state should live in Supabase Postgres (source of truth) with the desktop app querying it directly. If offline support is needed later, add local SQLite then. |
| **iCloud Drive / Dropbox as sync backend** | Explicitly out of scope per PROJECT.md. These add platform variability and remove control over the data contract. |
| **electron-builder** | Packaging tool for Electron. Irrelevant once Tauri is chosen. |

---

## Cowork Local Storage

**Confidence: MEDIUM** — Paths confirmed by GitHub issues and community reports. File format (JSONL) confirmed for Claude Code session history. Cowork-specific artifact format is less documented.

### Known Paths

| Platform | Path | Contents |
|----------|------|----------|
| macOS | `~/Documents/Claude/` | Cowork working directory — artifacts, scheduled task outputs (hardcoded, no config option as of May 2026) |
| macOS | `~/Documents/Claude/Projects/` | Cowork project folders (per "Projects" feature added 2025/2026) |
| macOS | `~/.claude/` | Claude Code config — settings, session history, MCP config |
| macOS | `~/.claude/projects/` | Session JSONL files, one directory per project (path URL-encoded, slashes → dashes) |
| macOS | `~/Library/Application Support/Claude/` | App support data — scheduled task config (`[session-id]/scheduled-tasks.json`) |
| Windows | `C:\Users\{username}\.claude\` | Equivalent of `~/.claude/` on macOS |
| Windows | `C:\Users\{username}\Documents\Claude\` | Equivalent of `~/Documents/Claude/` on macOS |

**Path override:** `CLAUDE_CONFIG_DIR` environment variable redirects the `~/.claude/` path if set.

### File Format

- Session history: `.jsonl` (JSON Lines), one event per line
- Session naming: `{session-id}.jsonl` inside `~/.claude/projects/{encoded-path}/`
- Scheduled tasks: `scheduled-tasks.json` (standard JSON)
- Project artifacts: files in `~/Documents/Claude/Projects/{project-name}/` — exact schema TBD by Cowork version

### Critical Warning

The `~/Documents/Claude/` path is **hardcoded** in Cowork as of the research date (GitHub issue #57177 is open requesting configurability). The sync app must watch and write to this exact path — it cannot be relocated without the user modifying Cowork internals. Any sync operation that writes to this directory must be safe against Cowork running concurrently; JSONL append-only semantics help here, but concurrent writes to the same session file must be avoided.

### Gaps (Requires Investigation)

- What is the exact schema of project artifact files inside `~/Documents/Claude/Projects/`? This is Cowork-specific and not fully documented.
- Does Cowork lock files while writing (file locks)? The sync app needs to detect and skip locked files.
- Is there a Cowork settings/preferences file that also needs syncing? (`~/Library/Application Support/Claude/` contents beyond scheduled tasks.)
- Windows path for `~/Library/Application Support/Claude/` equivalent — likely `%APPDATA%\Claude\` but unconfirmed.

---

## Open Questions

1. **Anthropic OAuth accessibility** — Are the OAuth endpoints used by Claude Code (`/oauth/authorize`, `/v1/oauth/token`) accessible to third-party desktop apps, or will Anthropic restrict them? The client ID `9d1c250a-e61b-44d9-88ed-5944d1962f5e` belongs to Claude Code — a new OAuth client registration may be required. **Must validate before auth implementation begins.**

2. **Cowork file locking** — Does Claude Cowork hold POSIX advisory locks or Windows share locks on JSONL session files while active? The sync app must handle this gracefully (skip locked files, retry on unlock).

3. **Cowork artifact format versioning** — Is there a version marker in artifact files so the sync app can refuse to sync if the format has changed (preventing corruption on older Cowork installs)?

4. **Conflict resolution strategy** — Last-write-wins by `modified_at` is proposed for v1. This is safe for the single-user multi-device case but fails if the same Cowork session is active on two devices simultaneously. Does Cowork prevent concurrent sessions? Needs confirmation.

5. **Windows `%APPDATA%` path** — Confirm that `%APPDATA%\Claude\` is the Windows equivalent of `~/Library/Application Support/Claude/` for scheduled task config.

6. **Supabase Storage size limits** — Cowork project artifacts could include large files (code outputs, generated images). Supabase Storage's free tier is 500 MB; need to assess typical project sizes and whether a paid tier or per-file size cap is needed.

---

## Installation Reference

```bash
# Tauri CLI
cargo install tauri-cli

# Create project
npm create tauri-app@latest cowork-sync -- --template vanilla-ts

# Tauri plugins
cargo add tauri-plugin-fs tauri-plugin-shell tauri-plugin-updater
npm install @tauri-apps/plugin-fs @tauri-apps/plugin-updater

# Supabase client
npm install @supabase/supabase-js

# Anthropic auth (Rust)
# In Cargo.toml:
# anthropic-auth = { version = "0.x", features = ["async"] }
```

---

## Sources

- Tauri v2 releases: https://v2.tauri.app/release/ (latest: 2.11.2)
- Tauri system tray docs: https://v2.tauri.app/learn/system-tray/
- Tauri menubar app guide (undocumented gotchas): https://dev.to/hiyoyok/building-a-menubar-app-with-tauri-v2-what-nobody-tells-you-9a2
- Tauri v2 code signing guide: https://dev.to/tomtomdu73/ship-your-tauri-v2-app-like-a-pro-code-signing-for-macos-and-windows-part-12-3o9n
- Tauri updater plugin: https://v2.tauri.app/plugin/updater/
- Tauri plugin-fs (v2.4.5): https://v2.tauri.app/plugin/file-system/
- notify-rs (Rust fs watcher): https://github.com/notify-rs/notify
- File watching with notify-rs for sync apps: https://dev.to/hiyoyok/file-watching-in-rust-with-notify-rs-hot-folders-for-a-sync-app-32d8
- Chokidar v5 (ESM-only, Nov 2025): https://github.com/paulmillr/chokidar
- @parcel/watcher comparison: https://github.com/vitejs/vite/issues/13593
- Tauri vs Electron (DoltHub, Nov 2025): https://www.dolthub.com/blog/2025-11-13-electron-vs-tauri/
- Supabase Realtime: https://supabase.com/realtime
- Supabase vs Cloudflare R2 pricing: https://www.buildmvpfast.com/api-costs/cloud-storage
- Cloudflare Durable Objects WebSockets: https://developers.cloudflare.com/durable-objects/best-practices/websockets/
- Anthropic OAuth PKCE gist: https://gist.github.com/cedws/3a24b2c7569bb610e24aa90dd217d9f2
- anthropic-auth Rust crate: https://crates.io/crates/anthropic-auth
- Cowork workspace base path issue (#57177): https://github.com/anthropics/claude-code/issues/57177
- Cowork projects storage: https://claudelab.net/en/articles/cowork/cowork-projects-integration-guide
- Claude directory docs: https://code.claude.com/docs/en/claude-directory
