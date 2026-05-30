# First-Pass Architectural Diagnostic — Target-A

**Target:** github.com/[organization]/[project] · **Branch/commit:** master · `aeb78e5` · **Date:** 2026-05-29
**Scope:** security, reliability, concurrency, transactional, architecture · **Method:** multi-agent audit + cross-falsification (read-only)

## 1. Verdict
**Production readiness: FAIL**

- **Critical blockers:** CRON-1 — unrecovered panic in any Go-registered cron job terminates the entire process
- **Estimated fix effort:** ~1–2 engineering days (CRON-1 itself is a one-liner; backup/shutdown coupling is the bulk of remaining work)
- **Risk profile:** No exploitable remote-code-execution or privilege-escalation for anonymous users; the dominant risk is operational fragility — process crash via cron, partial backup archives on shutdown, and unprotected APIs on fresh installs. Secrets at rest lack encryption by default.
- **Operational impact if unaddressed:** crash · integrity risk · degraded UX · data leak (secrets plaintext)

## 2. Fix plan (patch-sets ordered by urgency)

### Patch-set A — Cron crash guard
Findings: #1
**Why together:** Single independent change; no coupling.
**Estimated effort:** 30 min

### Patch-set B — Backup / shutdown integrity
Findings: #5, #6
**Why together:** Moving `CreateBackup` to a background goroutine (fixing #5) is only safe if the shutdown drain window is long enough to let that goroutine finish; shipping one without the other leaves partial archives on disk or in S3.
**Estimated effort:** 4–6 h

### Patch-set C — Rate-limit default
Findings: #2
**Why together:** Single independent change.
**Estimated effort:** 15 min (one boolean + smoke test)

### Patch-set D — Secrets-at-rest warning
Findings: #3
**Why together:** Single independent change.
**Estimated effort:** 2–4 h

### Patch-set E — Backup concurrency guard
Findings: #4
**Why together:** Single independent change.
**Estimated effort:** 2–3 h

### Patch-set F — DDL identifier sanitizer
Findings: #8
**Why together:** Single independent change; ships before any Go-extension surface expands.
**Estimated effort:** 1–2 h

### Patch-set G — JS VM OS-exposure documentation / scoping
Findings: #7
**Why together:** Single independent change; meaningful sandbox requires JS runtime API investigation.
**Estimated effort:** 1 day (meaningful sandboxing) or 1 h (prominent doc/startup warning)

## 3. Findings table (substantive only: FAIL · WARN)
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

## 4. Detailed findings

---

**#1 — `go j.Run()` — unrecovered panic crashes process**
**FAIL · HIGH · ACTIVE · Confidence: HIGH**
Evidence: `pkg/scheduler/cron.go:225` — `go j.Run()`; `pkg/scheduler/job.go:23–26` — `j.fn()` called with no `defer recover()`.
Reproduction scenario: Register any Go cron job that triggers a nil-pointer dereference or explicit `panic()`. The raw goroutine has no recovery handler; Go's runtime terminates the entire process.
Counter-evidence considered: The scripting cron path (`pkg/scripting/binds.go:108–120`) wraps `executor.RunProgram` and converts JS exceptions to `error` — it does **not** trigger this path. The risk is scoped to Go-extension consumers and any future built-in cron job that panics. `routine.FireAndForget` (with `recover`) already exists at `pkg/routine/routine.go:13` — it is the established codebase pattern and is simply not applied here.
Recommendation: Replace `go j.Run()` with `routine.FireAndForget(j.Run)` at `cron.go:225`, or add `defer func() { recover() }()` as the first statement of `Job.Run()` at `job.go:23`.

---

**#2 — Rate limiting disabled by default on all fresh installs**
**WARN · MEDIUM · ACTIVE · Confidence: HIGH**
Evidence: `pkg/core/config.go:178` — `Enabled: false`; inline comment `// @todo once tested enough enable by default` confirms the intent. Default `Rules` include `*:auth` (2 req/3s) and `/api/` (300 req/10s) but are entirely inert while `Enabled` is false.
Reproduction scenario: Deploy Project-01 from a fresh binary without editing settings. All auth and API endpoints accept unlimited request rates from all sources.
Counter-evidence considered: Operators can enable rate limiting via the Admin UI. The risk is that every new deployment starts unprotected until an operator actively changes a setting, which many will not do.
Recommendation: Flip `Enabled: false` → `true` in the default `RateLimitsConfig`. Gate behind a release note given the prior `@todo` comment.

---

**#3 — SMTP · S3 · OAuth2 credentials stored plaintext in SQLite**
**WARN · MEDIUM · ACTIVE · Confidence: HIGH**
Evidence: `pkg/core/config.go:279–280` — credential fields serialised into `_params` table with no encryption when `APP_ENCRYPTION_KEY` is unset (the default).
Reproduction scenario: Any party with read access to `app_data/data.db` (backup exfiltration, misconfigured S3 public read on the backup bucket, host compromise) obtains all configured SMTP/S3/OAuth2 secrets in cleartext.
Counter-evidence considered: `APP_ENCRYPTION_KEY` fully encrypts these fields when set. The gap is that the zero-config default offers no warning that secrets are at risk — operators who set credentials without reading the encryption docs are silently unprotected.
Recommendation: Emit a startup `Logger().Warn` when `APP_ENCRYPTION_KEY` is absent and any credential field (SMTP password, S3 secret, OAuth2 client secret) is non-empty. The fix is additive and does not alter the encryption path.

---

**#4 — Backup lock `Has()`+`Set()` non-atomic — TOCTOU race**
**WARN · MEDIUM · ACTIVE · Confidence: HIGH**
Evidence: `pkg/core/backup.go:45` — `app.Store().Has(StoreKeyActiveBackup)`; `:49` — `app.Store().Set(StoreKeyActiveBackup, ...)`. `pkg/store/store.go:82` — `Has` acquires then releases `RLock`; `:140` — `Set` acquires a separate `Lock`. No `SetIfAbsent` primitive exists.
Reproduction scenario: Two simultaneous `POST /api/backups` requests pass the `Has()` guard before either calls `Set()`. Both enter `CreateBackup` concurrently, producing two archive files — double disk/S3 consumption — and potential `SQLITE_BUSY` contention under concurrent WAL writes.
Counter-evidence considered: `backupRestore` has the same pattern; both need fixing together. No `SetIfAbsent` exists in `store.Store` today.
Recommendation: Add `SetIfAbsent(key, value) bool` to `pkg/store/store.go` using a single mutex acquisition, and use it as the guard in both `CreateBackup` and `RestoreBackup`.

---

**#5 — `backupCreate` blocks HTTP handler goroutine with `context.Background()`**
**WARN · MEDIUM · ACTIVE · Confidence: HIGH**
Evidence: `pkg/api/backup_handler.go:30` — `app.CreateBackup(context.Background(), name)` called directly in the HTTP handler; `context.Background()` ignores client disconnects and server shutdown signals.
Reproduction scenario: A large database triggers a multi-minute backup. The handler goroutine is tied up for the entire duration. On shutdown, the backup is abandoned mid-zip, leaving a partial archive in `app_temp/` (local) or an incomplete multipart upload (S3, accruing storage charges until lifecycle cleanup).
Counter-evidence considered: `backupRestore` fires a background goroutine and responds immediately — this is the established pattern in the same file. `backupCreate` was not updated to match.
Recommendation: Mirror `backupRestore`: launch `CreateBackup` in a goroutine, respond `204` immediately, let the shutdown drain window (see #6) cover the goroutine's lifetime.

---

**#6 — Graceful-shutdown 1-second drain; error silently discarded**
**WARN · MEDIUM · ACTIVE · Confidence: HIGH**
Evidence: `pkg/api/server.go:176` — `ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)`; `:181` — `_ = server.Shutdown(ctx)` (error discarded).
Reproduction scenario: Any in-flight long request (backup, large file upload, slow query) is abandoned after 1 s. With fix #5 applied, background backup goroutines started just before shutdown will be orphaned. The discarded error means operators never see a log line confirming whether shutdown was clean.
Counter-evidence considered: 1 s is intentionally short for dev convenience, but no environment variable or config knob exposes it for production tuning. The error discard is inconsistent with the rest of the server.go error-handling style.
Recommendation: Raise the default to 30 s (or make it configurable via `ServeConfig`); propagate the `Shutdown` error to the caller log.

---

**#7 — JS VM exposes `exec.Command`, `os.Exit`, `os.RemoveAll`, `os.Getenv` — no sandbox**
**WARN · MEDIUM · ACTIVE · Confidence: HIGH**
Evidence: `pkg/scripting/binds.go:806–830` — `BindOS` registers `exec.Command`, `os.Exit`, `os.RemoveAll`, `os.Getenv`, and related syscall wrappers directly into every JS runtime.
Reproduction scenario: An operator deploys a JS hook (`.js` file in `app_hooks/`) that calls `$os.exec("rm", ["-rf", "/"])` or exfiltrates `$os.getenv("APP_ENCRYPTION_KEY")`. Any supply-chain compromise of the hooks directory achieves full host-level control.
Counter-evidence considered: Documentation states hooks run with process-level privileges. The attack vector is operator-controlled or supply-chain, not an unauthenticated API user. The Falsifier demoted this from HIGH to MEDIUM on those grounds; the concern is valid for multi-operator or managed-hosting scenarios.
Recommendation: At minimum, emit a prominent startup warning when `app_hooks/` contains `.js` files, stating that hooks run with full OS privileges. For managed hosting, evaluate JS runtime interrupt/capability restrictions or a separate subprocess model.

---

**#8 — `DeleteTable`/`DeleteView`/`SaveView` — doc-only SQLi guard**
**WARN · MEDIUM · LATENT · ACTIVE (armed when a Go extension or future built-in passes user-controlled input to these methods) · Confidence: HIGH**
Evidence: `pkg/core/app.go:238–258` — `DeleteTable`, `DeleteView`, `SaveView` interpolate the table/view name directly into SQL with a `dangerousTableName` parameter name as the only guard. No runtime identifier sanitization or quoting exists.
Reproduction scenario: A Go extension calling `app.DeleteTable(r.PathParam("table"))` is immediately exploitable via SQL injection. All current internal callers use schema-validated names, so no live exploit exists today.
Counter-evidence considered: The naming convention is a developer hint, not enforcement. The surface expands as the extension ecosystem grows.
Recommendation: Add a compile-time or runtime identifier sanitizer (allow-list `[A-Za-z0-9_]`, max length) inside each method — not at call sites — so the invariant is impossible to violate by future callers.

---

## 5. Method & limits
Agents run: Explorer (cartography), Security analyst, Reliability analyst, Concurrency analyst, Transactional analyst, Architecture analyst, Falsifier (existence-gate + demolition + runtime-trace + severity recalibration), Aggregator.
**Coverage verified (sound):** JWT two-step verification, token/password log leakage, CORS `AllowCredentials`, OAuth2 CSRF state, OTP/MFA bcrypt timing, SQL endpoint access control, impersonation privilege scoping, bcrypt cost 10, auth-origins session fixation, file upload/download path traversal, installer endpoint lifecycle, external-auth pre-hijacking, DB retry logic (bounded, non-transactional), migration atomicity (double-transaction wrap), mailer error propagation, S3 multipart abort on context cancel, restore operation intentional panic on double-failure, logger double-flush serialisation, JS VM pool per-item mutex, hook dispatch snapshot-under-RLock, watcher goroutine termination, nested `RunInTransaction` tx reuse, collection import atomicity, DDL in schema-sync participates in parent tx, entity ID validation uses concurrent DB correctly, layer boundary (`pkg/core/ → pkg/api/` import absent), hook system fully typed generics, settings hot-reload mutex-protected.
Reminder: this is a first pass (diagnostic, non-exhaustive) — confirm each finding before action.

## Appendix A — Watchlist / Hypotheses
CRON-DEADLOCK (`pkg/scheduler/cron.go:155–204`) — `Cron.Stop()` holds `RWMutex.Lock` while blocking on unbuffered `tickerDone` send; receiver goroutine spawned only after `runDue()` which acquires `RLock` — narrow but genuine deadlock window; impact if triggered is permanent cron non-functionality. Demoted from main findings because trigger window is ~3 instructions and requires specific scheduler timing; buffer `tickerDone` (capacity 1) and release `mux.Unlock()` before the send to eliminate.

## Appendix B — Hygiene & posture (LOW)
- **SQL-1** (`pkg/api/sql_handler.go:88–95`) — `/api/sql` write-detection skips queries prefixed with `--comment`, `/* comment */`, or `WITH`; routes write to `ConcurrentDB` instead of `NonconcurrentDB`. Superuser-only; no privilege escalation.
- **SQL-2** (`pkg/core/db_builder.go:155`) — ORM `dualDBBuilder` routes all `WITH`-prefixed queries to `concurrentDB`; write-CTEs from non-transactional ORM callers land on the wrong pool. Code comment acknowledges the gap.
- **WAL-SILENT** (`pkg/core/backup.go:87–88`) — WAL checkpoint error discarded before archive; a silent checkpoint failure means the backup may not include the latest committed data.
- **UPLOAD-CLOSE** (`pkg/storage/filesystem.go:251–256`) — `UploadFile`/`UploadMultipart` discard `w.Close()` error after `ReadFrom` failure; `Upload()` in the same file uses `errors.Join` correctly — apply the same pattern.
- **BATCH-1** (`pkg/api/batch_handler.go:192–233`) — inner goroutine continues executing against a rolled-back `txApp` after timeout; `sql.ErrTxDone` prevents writes; no data corruption, but hook side-effects (email, external calls) may still fire.
- **GOJA-PINNED** (`go.mod`) — JS runtime dependency pinned to commit pseudo-versions (`v0.0.0-`); no semantic versioning contract makes security patch uptake manual and invisible to tooling.
- **EXT-API** (`pkg/api/extensions.go`) — extension struct literals used as API contract with no version field; future field additions silently break third-party consumers.

---

_First-pass diagnostic — non-exhaustive; confirm each finding before action._

---

💡 **Want to test our Deep Architectural Review Copilot on your own codebase?**
We are currently looking for 5 engineers or technical founders for our calibration phase.
- **The deal:** We run a full audit on a project of your choice (a microservice, a side-project, or a public OSS repo for full confidentiality).
- **In exchange:** We ask for an honest, written critique of the report's quality and relevance.

Interested? Send me a direct message or a connection request to reserve your slot.
