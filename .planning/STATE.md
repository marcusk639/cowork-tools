# Project State: Cowork Sync Desktop App

## Project Reference

**Core Value:** A user's Cowork work is never lost when switching devices — install the app and everything is there.
**Current Focus:** Phase 1 — Foundation (Auth + Data Contract)
**Mode:** mvp

---

## Current Position

| Field | Value |
|-------|-------|
| Phase | 1 — Foundation |
| Plan | None (planning not yet started) |
| Status | Not started |
| Phase Goal | Auth + Data Contract validated |

**Progress:**
```
Phase 1 [          ] Not started
Phase 2 [          ] Not started
Phase 3 [          ] Not started
Phase 4 [          ] Not started
Phase 5 [          ] Not started

Overall: 0% — 0/5 phases complete
```

---

## Performance Metrics

| Metric | Value |
|--------|-------|
| Phases total | 5 |
| Phases complete | 0 |
| Requirements total (v1) | 19 |
| Requirements complete | 0 |
| Plans written | 0 |
| Plans complete | 0 |

---

## Accumulated Context

### Key Decisions Made

| Decision | Rationale | Phase |
|----------|-----------|-------|
| Tauri 2.x over Electron | ~10 MB installer vs ~150 MB; native tray support; Rust file watching via notify crate | Pre-phase |
| Supabase for backend | Managed auth + Postgres + Realtime + Storage; free tier covers personal scale | Pre-phase |
| Anthropic OAuth PKCE as primary, Google/Apple as fallback | Cowork already authenticates this way; fallback if Anthropic endpoints are restricted | Pre-phase |
| Hub-and-spoke (no P2P) | Backend is single source of truth; simpler conflict model | Pre-phase |
| Server-assigned monotonic version numbers | Avoids clock-skew LWW data loss pitfall | Pre-phase |
| Backend-first build order | Client sync engine must be built against real API contract, not stubs | Pre-phase |
| SQLite for local sync state | Durable pending-ops queue survives restarts; no external dependency | Pre-phase |

### Critical Risks to Track

| Risk | Severity | Mitigation | Phase |
|------|----------|------------|-------|
| Anthropic OAuth endpoints undocumented (reverse-engineered from Claude Code) | HIGH | Validate in Phase 1 before any auth code merges; fallback to Google/Apple is designed in | 1 |
| macOS sandbox silently blocks ~/Documents/Claude/ watcher | HIGH | Request entitlement + persist security-scoped bookmark; test in Phase 3 | 3 |
| Sync loop (app re-uploads its own downloads) | HIGH | Write-lock pendingLocalWrites set + filter RENAME events | 3 |
| Partial write corrupts Cowork files on crash | HIGH | Atomic rename via temp file + fsync from day one | 3 |
| Windows ReadDirectoryChangesW buffer overflow silently misses events | MEDIUM | Overflow detection + reconciliation sweep | 4 |
| Supabase Storage free tier (500 MB) may be insufficient | MEDIUM | Assess project sizes before Phase 2 completes | 2 |
| Cowork file locking behavior on Windows unknown | MEDIUM | Empirical testing in Phase 3 | 3 |

### Open Questions

| Question | Blocking | Phase |
|----------|----------|-------|
| Does Anthropic OAuth allow a separate client registration, or is reuse of Claude Code's client ID acceptable? | AUTH-01 | 1 |
| Does Claude Cowork hold POSIX advisory or Windows mandatory locks on JSONL session files while active? | SYNC-04 | 3 |
| Exact schema of ~/Documents/Claude/Projects/ artifact files — needs reverse-engineering from a live Cowork instance | SYNC-01 | 3 |
| Is %APPDATA%\Claude\ confirmed as the Windows equivalent of ~/Library/Application Support/Claude/? | SYNC-01 | 3 |
| Is Supabase free tier (500 MB) adequate for typical Cowork project sizes? | SYNC-07 | 2 |

### Todos

| Todo | Phase |
|------|-------|
| Validate Anthropic OAuth endpoints before any auth code is merged | 1 |
| Research macOS sandbox entitlement flow + security-scoped bookmark persistence during Phase 3 planning | 3 |
| Confirm Windows Cowork data paths empirically (not from community sources) | 3 |
| Assess Supabase Storage budget against typical Cowork project sizes | 2 |

### Blockers

None currently.

---

## Session Continuity

**Last session:** 2026-05-30 — Roadmap created (5 phases, 19/19 requirements mapped)
**Next action:** Run `/gsd-plan-phase 1` to plan Phase 1: Foundation

### Phase Transition Notes

No transitions yet.

---

*State initialized: 2026-05-30*
*Last updated: 2026-05-30 after roadmap creation*
