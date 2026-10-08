<!-- GSD:project-start source:PROJECT.md -->

## Project

**Cowork Tools**

A desktop application (macOS and Windows) that syncs Claude Cowork data — projects, files, settings, and conversation history — across a user's devices via a cloud backend. Users install the app, log in with their Claude/Anthropic account, authorize access to their Cowork data, and from that point all Cowork data is automatically kept in sync across every device where the app is installed. Built initially for personal/internal use with the potential to share publicly.

**Core Value:** A user's Cowork work is never lost when switching devices — install the app and everything is there.

### Constraints

- **Compatibility**: Must read/write Cowork's local data format exactly as Cowork expects it — no corruption or schema drift
- **Platform**: macOS and Windows desktop only for v1
- **Auth**: Must integrate with Claude/Anthropic account system; no building a standalone credential store
- **Data integrity**: Sync must be safe — data loss from a bad sync is worse than no sync at all
<!-- GSD:project-end -->

<!-- MANUAL:repo-reality (not GSD-managed; safe to edit) -->

## Repository Reality (Current State)

> The sections below this one describe the **planned** Cowork _sync desktop app_
> (Tauri/Rust/Supabase). That app has **no code yet** (`.planning/STATE.md`: Phase 1,
> 0% complete). What actually lives in this repo today is a set of **Claude Cowork
> plugins**. Start here.

### What's in the repo

| Path                        | What it is                                                                                                                         |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `.claude-plugin/marketplace.json` | Marketplace manifest at the repo root; lists each plugin with `"source": "./<plugin>"`.                                     |
| `deep-research/`            | Cowork plugin, skill `deep-research`. 8-phase research pipeline; uses Firecrawl/Exa connectors when enabled, else Cowork's built-in web search/fetch. |
| `prompt-engineer/`          | Cowork plugin, skill `prompt-engineer`. Prompt-engineering workflow + three `references/` docs. No tools or connectors needed. |
| `llm-prompt-engineer/` | Cowork plugin, skill `llm-prompt-engineer`. Third-party (Jeffallan/claude-skills, MIT; keep `LICENSE`). Only local change: skill `name`. |
| `.planning/`                | GSD planning docs for the (unbuilt) sync app. Source of the GSD-managed sections below.                                            |
| `README.md`                 | User-facing description of the plugins and install steps.                                                                                 |

### Plugin anatomy

```
<plugin>/
├── .claude-plugin/
│   └── plugin.json        # name, version, description, keywords
└── skills/<skill-name>/
    ├── SKILL.md           # the pipeline definition (the actual logic)
    └── reference/         # methodology.md, quality-gates.md, SETUP.md, provenance
```

### Conventions for plugin work

- **One marketplace, at the repo root.** A new plugin gets its own top-level folder and
  an entry in `.claude-plugin/marketplace.json`; skill names must not collide.
- **Methodology lives in `reference/`**, not inline in SKILL.md — keep SKILL.md the
  orchestration layer and push detail (red-team rules, quality gates) into reference docs.
- **Red-team discipline is the differentiator**: a claim ships only if cited to a
  fetched source and surviving adversarial verification (3-persona / 2-of-3 kill rule).
  Preserve this when editing the pipeline.
- No build step / test suite yet — plugins are Markdown + JSON. "Validation" = the
  `.claude-plugin/*.json` parse and the SKILL.md pipeline runs end-to-end.
- A PostToolUse hook auto-runs Prettier on `.md`/`.json` after edits — expect
reformatting; don't fight it.
<!-- /MANUAL:repo-reality -->

<!-- GSD:stack-start source:research/STACK.md -->

## Technology Stack

## Recommended Stack

### Desktop Framework

- Installer size: <10 MB vs Electron's 100–150 MB. A sync daemon does not need Chromium bundled.
- Memory at idle: ~30–50 MB vs Electron's 150–300 MB. This app runs continuously in the background; Electron's memory footprint is unjustifiable for a tray icon.
- Native WebView (WKWebView on macOS, WebView2 on Windows) is sufficient for a simple status popup — no need for a full Chromium instance.
- Tauri v2 has first-class tray/menu-bar support via `TrayIconBuilder`. Setting `LSUIElement: true` in `tauri.conf.json` hides the app from the macOS Dock and Cmd+Tab — the correct behavior for a background daemon.
- The file-watching logic lives in Rust (see below), so there is no "we need Node.js for the backend" reason to pick Electron.
- Bundles Chromium + Node.js per app. For a background sync daemon this is waste, not benefit.
- No advantage for this use case; Electron's larger ecosystem only matters for feature-heavy UIs.
- macOS-only. The requirement is macOS + Windows from a single codebase.

### File System Watching

- On macOS: uses FSEvents (kernel-level, low overhead, recursive).
- On Windows: uses ReadDirectoryChangesW (kernel-level, low overhead, recursive).
- No polling. No extra process. No dependencies outside the Tauri plugin system.
- Debouncing is required — FSEvents and ReadDirectoryChangesW both fire duplicate events. Build debounce in from day one (recommend 300–500 ms settle window before triggering a sync).

### Sync Backend

- This is an internal/personal tool for one user (per PROJECT.md). The Supabase free tier covers 500 MB storage, 2 GB bandwidth, and unlimited auth — enough to run indefinitely at personal scale.
- Supabase bundles auth, Postgres, realtime WebSockets, and file storage in one product. Cloudflare's equivalent requires assembling Workers + D1 + R2 + Durable Objects — four products with separate billing, documentation, and failure modes. Supabase is right-sized for this use case.
- Supabase Realtime provides WebSocket subscriptions to Postgres table changes out of the box. When Device A syncs, Device B's desktop app receives a push notification via the realtime channel and pulls the delta — no polling, no custom WebSocket server to maintain.
- Supabase Storage integrates with Row Level Security (RLS) so the same JWT used for auth gates file access with zero extra middleware.
- `sync_files` table: `(id, user_id, device_id, path, content_hash, size, modified_at, synced_at, blob_storage_key)`
- File content stored in Supabase Storage bucket (`cowork-files/{user_id}/{hash}`)
- Content-addressed storage (keyed by hash) gives deduplication and idempotent re-upload for free
- Last-write-wins conflict resolution keyed on `modified_at` for v1 — sufficient for the single-user case described in PROJECT.md
- Unnecessary complexity for a personal sync tool. Supabase Realtime handles fan-out to multiple devices. A custom server adds infrastructure to maintain.

### Auth

- Authorization endpoint: `https://claude.ai/oauth/authorize`
- Token endpoint: `https://console.anthropic.com/v1/oauth/token`
- Client ID (Claude Code's): `9d1c250a-e61b-44d9-88ed-5944d1962f5e`
- Scopes: `org:create_api_key user:profile user:inference`
- PKCE method: S256
- Redirect: loopback on ephemeral port (`http://localhost:{port}/callback`)
- Anthropic OAuth proves identity and grants access to the user's Cowork data.
- Supabase Auth handles session lifecycle (refresh tokens, expiry, RLS enforcement on database/storage).
- Avoids building your own session store.

### Packaging and Auto-Update

- macOS: `.dmg` + `.app` bundle, notarized via Apple Developer account ($99/yr)
- Windows: `.msi` (NSIS installer) + `.exe`, signed via Authenticode
- macOS: Apple Developer account, Developer ID certificate, notarization via `xcrun notarytool`
- Windows: Authenticode certificate (OV cert avoids SmartScreen warnings on first run; EV cert eliminates them entirely but requires physical hardware key)

## What NOT to Use

| Technology                                    | Reason                                                                                                                                                                                                                                                                     |
| --------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Electron**                                  | Ships 100–150 MB Chromium runtime for what is essentially a tray icon + small popup. Memory footprint (150–300 MB idle) is unjustifiable for a background daemon. No upside for this use case.                                                                             |
| **chokidar (Node.js)**                        | Right tool for Node.js/Electron apps, wrong tool for Tauri. The Rust `notify` crate does the same thing natively without bridging through Node. Chokidar v5 is also ESM-only, adding a module format constraint.                                                           |
| **Cloudflare Workers + R2 + Durable Objects** | Correct architecture at scale; over-engineered for a personal/internal tool. Assembling four separate Cloudflare products (Workers, D1, R2, Durable Objects) adds operational complexity that Supabase eliminates for free at this scale. Revisit if the tool goes public. |
| **Custom WebSocket server**                   | Supabase Realtime provides WebSocket-based change notifications built on Postgres. Building a custom server adds infrastructure with no benefit.                                                                                                                           |
| **SQLite for sync state**                     | Tempting as a local cache, but introduces a secondary consistency surface. The sync state should live in Supabase Postgres (source of truth) with the desktop app querying it directly. If offline support is needed later, add local SQLite then.                         |
| **iCloud Drive / Dropbox as sync backend**    | Explicitly out of scope per PROJECT.md. These add platform variability and remove control over the data contract.                                                                                                                                                          |
| **electron-builder**                          | Packaging tool for Electron. Irrelevant once Tauri is chosen.                                                                                                                                                                                                              |

## Cowork Local Storage

### Known Paths

| Platform | Path                                    | Contents                                                                                                  |
| -------- | --------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| macOS    | `~/Documents/Claude/`                   | Cowork working directory — artifacts, scheduled task outputs (hardcoded, no config option as of May 2026) |
| macOS    | `~/Documents/Claude/Projects/`          | Cowork project folders (per "Projects" feature added 2025/2026)                                           |
| macOS    | `~/.claude/`                            | Claude Code config — settings, session history, MCP config                                                |
| macOS    | `~/.claude/projects/`                   | Session JSONL files, one directory per project (path URL-encoded, slashes → dashes)                       |
| macOS    | `~/Library/Application Support/Claude/` | App support data — scheduled task config (`[session-id]/scheduled-tasks.json`)                            |
| Windows  | `C:\Users\{username}\.claude\`          | Equivalent of `~/.claude/` on macOS                                                                       |
| Windows  | `C:\Users\{username}\Documents\Claude\` | Equivalent of `~/Documents/Claude/` on macOS                                                              |

### File Format

- Session history: `.jsonl` (JSON Lines), one event per line
- Session naming: `{session-id}.jsonl` inside `~/.claude/projects/{encoded-path}/`
- Scheduled tasks: `scheduled-tasks.json` (standard JSON)
- Project artifacts: files in `~/Documents/Claude/Projects/{project-name}/` — exact schema TBD by Cowork version

### Critical Warning

### Gaps (Requires Investigation)

- What is the exact schema of project artifact files inside `~/Documents/Claude/Projects/`? This is Cowork-specific and not fully documented.
- Does Cowork lock files while writing (file locks)? The sync app needs to detect and skip locked files.
- Is there a Cowork settings/preferences file that also needs syncing? (`~/Library/Application Support/Claude/` contents beyond scheduled tasks.)
- Windows path for `~/Library/Application Support/Claude/` equivalent — likely `%APPDATA%\Claude\` but unconfirmed.

## Open Questions

## Installation Reference

# Tauri CLI

# Create project

# Tauri plugins

# Supabase client

# Anthropic auth (Rust)

# In Cargo.toml:

# anthropic-auth = { version = "0.x", features = ["async"] }

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
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->

## Conventions

Conventions not yet established. Will populate as patterns emerge during development.

<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->

## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.

<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->

## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, `.github/skills/`, or `.codex/skills/` with a `SKILL.md` index file.

<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->

## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:

- `/gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd-debug` for investigation and bug fixing
- `/gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.

<!-- GSD:workflow-end -->

<!-- GSD:profile-start -->

## Developer Profile

> Profile not yet configured. Run `/gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.

<!-- GSD:profile-end -->
