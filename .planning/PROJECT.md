# Cowork Tools

## What This Is

A desktop application (macOS and Windows) that syncs Claude Cowork data — projects, files, settings, and conversation history — across a user's devices via a cloud backend. Users install the app, log in with their Claude/Anthropic account, authorize access to their Cowork data, and from that point all Cowork data is automatically kept in sync across every device where the app is installed. Built initially for personal/internal use with the potential to share publicly.

## Core Value

A user's Cowork work is never lost when switching devices — install the app and everything is there.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] User can install the desktop app on macOS and Windows
- [ ] User can log in with their Claude/Anthropic account (and optionally Google/Apple OAuth)
- [ ] User can authorize the app to access their local Cowork data
- [ ] App syncs Cowork data (projects, files, settings, history) to a cloud backend in real time
- [ ] App syncs automatically when Cowork opens or closes
- [ ] App syncs automatically when the sync app itself opens or closes
- [ ] On a new device, installing the app and logging in pulls all previously synced data to the expected Cowork locations
- [ ] App provides a basic status indicator (syncing / up to date / error)

### Out of Scope

- Mobile (iOS, Android) — desktop only for v1; users who need mobile access aren't the target yet
- Linux — macOS and Windows cover the initial audience
- User-provided storage (iCloud, Dropbox, S3) — we build and own the backend for simplicity and control
- Public release / user management — internal/personal use first; multi-user and distribution come later

## Context

- Claude Cowork is Anthropic's collaborative workspace feature; its exact local storage format and path need to be researched before implementation
- Authentication involves Claude/Anthropic accounts; the exact OAuth flow and available APIs need investigation
- The sync backend needs to handle conflict resolution when data is modified on multiple devices before syncing
- "Real-time" sync likely means watching the Cowork data directory for file system events and pushing diffs to the backend
- The project is building from scratch — no existing codebase

## Constraints

- **Compatibility**: Must read/write Cowork's local data format exactly as Cowork expects it — no corruption or schema drift
- **Platform**: macOS and Windows desktop only for v1
- **Auth**: Must integrate with Claude/Anthropic account system; no building a standalone credential store
- **Data integrity**: Sync must be safe — data loss from a bad sync is worse than no sync at all

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Build our own sync backend | User-owned storage adds complexity and variability; we control the data contract | — Pending |
| macOS + Windows only | Covers the primary audience; Linux adds packaging complexity with little initial return | — Pending |
| Auth via Claude/Anthropic account + social OAuth | Cowork already authenticates this way; social OAuth (Google/Apple) is the fallback if Anthropic's OAuth isn't accessible externally | — Pending |
| Real-time sync + lifecycle triggers | Always-on sync is the most seamless UX; lifecycle triggers ensure consistency on open/close | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-05-30 after initialization*
