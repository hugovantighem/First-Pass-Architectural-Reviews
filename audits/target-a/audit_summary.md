# First-Pass Architectural Diagnostic — Target-A (Condensed)

**Target:** github.com/[organization]/[project] · **Branch/commit:** master · `aeb78e5` · **Date:** 2026-05-29
**Scope:** security, reliability, concurrency, transactional, architecture · **Method:** multi-agent audit + cross-falsification (read-only)

## 1. Verdict
**Production readiness: FAIL**

- **Critical blockers:** CRON-1 — unrecovered panic in any Go-registered cron job terminates the entire process
- **Estimated fix effort:** ~1–2 engineering days (CRON-1 itself is a one-liner; backup/shutdown coupling is the bulk of remaining work)
- **Risk profile:** No exploitable remote-code-execution or privilege-escalation for anonymous users; the dominant risk is operational fragility — process crash via cron, partial backup archives on shutdown, and unprotected APIs on fresh installs. Secrets at rest lack encryption by default.
- **Operational impact if unaddressed:** crash · integrity risk · degraded UX · data leak (secrets plaintext)

## 2. Findings table (substantive only: FAIL · WARN)
| # | Status | Severity | State | Component | Title | File:line |
|---|--------|----------|-------|-----------|-------|-----------|
| 1 | FAIL | HIGH | ACTIVE | Cron scheduler | `go j.Run()` — unrecovered panic crashes process | `pkg/scheduler/cron.go:225`, `pkg/scheduler/job.go:23` |
| 2 | WARN | MEDIUM | ACTIVE | Settings / API | Rate limiting disabled by default on all fresh installs | `pkg/core/config.go:178` |
| 3 | WARN | MEDIUM | ACTIVE | Settings / secrets | SMTP · S3 · OAuth2 credentials stored plaintext in SQLite | `pkg/core/config.go:279` |
| 4 | WARN | MEDIUM | ACTIVE | Backup subsystem | Backup lock `Has()`+`Set()` non-atomic — TOCTOU race | `pkg/core/backup.go:45,49` |
| 5 | WARN | MEDIUM | ACTIVE | Backup subsystem | `backupCreate` blocks HTTP handler with `context.Background()` | `pkg/api/backup_handler.go:30` |
| 6 | WARN | MEDIUM | ACTIVE | HTTP server | Graceful-shutdown 1-second drain; error silently discarded | `pkg/api/server.go:176,181` |
| 7 | WARN | MEDIUM | ACTIVE | Scripting plugin | JS VM exposes `exec.Command` / `os.Exit` / `os.RemoveAll` — no sandbox | `pkg/scripting/binds.go:806` |
| 8 | WARN | MEDIUM | LATENT | Core DB helpers | `DeleteTable`/`DeleteView`/`SaveView` — doc-only SQLi guard (armed when caller passes user-controlled input) | `pkg/core/app.go:238` |

State: ACTIVE = reachable on a current code path · LATENT = real but not yet armed (note what arms it)

## 3. Focus — Finding #1 & #4 (deep dive)

---

**#1 — `go j.Run()` — unrecovered panic crashes process**
**FAIL · HIGH · ACTIVE · Confidence: HIGH**
Evidence: `pkg/scheduler/cron.go:225` — `go j.Run()`; `pkg/scheduler/job.go:23–26` — `j.fn()` called with no `defer recover()`.
Reproduction scenario: Register any Go cron job that triggers a nil-pointer dereference or explicit `panic()`. The raw goroutine has no recovery handler; Go's runtime terminates the entire process.
Counter-evidence considered: The scripting cron path (`pkg/scripting/binds.go:108–120`) wraps `executor.RunProgram` and converts JS exceptions to `error` — it does **not** trigger this path. The risk is scoped to Go-extension consumers and any future built-in cron job that panics. `routine.FireAndForget` (with `recover`) already exists at `pkg/routine/routine.go:13` — it is the established codebase pattern and is simply not applied here.
Recommendation: Replace `go j.Run()` with `routine.FireAndForget(j.Run)` at `cron.go:225`, or add `defer func() { recover() }()` as the first statement of `Job.Run()` at `job.go:23`.

---

**#4 — Backup lock `Has()`+`Set()` non-atomic — TOCTOU race**
**WARN · MEDIUM · ACTIVE · Confidence: HIGH**
Evidence: `pkg/core/backup.go:45` — `app.Store().Has(StoreKeyActiveBackup)`; `:49` — `app.Store().Set(StoreKeyActiveBackup, ...)`. `pkg/store/store.go:82` — `Has` acquires then releases `RLock`; `:140` — `Set` acquires a separate `Lock`. No `SetIfAbsent` primitive exists.
Reproduction scenario: Two simultaneous `POST /api/backups` requests pass the `Has()` guard before either calls `Set()`. Both enter `CreateBackup` concurrently, producing two archive files — double disk/S3 consumption — and potential `SQLITE_BUSY` contention under concurrent WAL writes.
Counter-evidence considered: `backupRestore` has the same pattern; both need fixing together. No `SetIfAbsent` exists in `store.Store` today.
Recommendation: Add `SetIfAbsent(key, value) bool` to `pkg/store/store.go` using a single mutex acquisition, and use it as the guard in both `CreateBackup` and `RestoreBackup`.

---

_First-pass diagnostic — non-exhaustive; confirm each finding before action._
_Full report (findings #1–#8 + appendices) available on request._

---

💡 **Want to test our Deep Architectural Review Copilot on your own codebase?**
We are currently looking for 5 engineers or technical founders for our calibration phase.
- **The deal:** We run a full audit on a project of your choice (a microservice, a side-project, or a public OSS repo for full confidentiality).
- **In exchange:** We ask for an honest, written critique of the report's quality and relevance.

Interested? Send me a direct message or a connection request to reserve your slot.
