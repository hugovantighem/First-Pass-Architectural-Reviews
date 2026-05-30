# AI Architectural Diagnostic — Public Benchmarks

This repository publishes first-pass architectural diagnostics produced by a multi-agent system on real-world open-source projects.

**The goal:** measure whether a multi-agent LLM pipeline can surface findings comparable to what a senior engineer would identify in 1–2 days of initial exploration — in under an hour — and calibrate that claim transparently, in public.

This is a benchmark. Projects are anonymised — the goal is to assess the system's reasoning quality, not to publish vulnerability disclosures. Findings are reproducible: the stack, size, and file:line references are sufficient to locate and verify each one independently.

---

## 💡 Want to test this on your own codebase?

We are currently looking for **5 engineers or technical founders** for our calibration phase.

**The deal:** We run a full audit on a project of your choice — a microservice, a side-project, or a public OSS repo for full confidentiality.
**In exchange:** An honest, written critique of the report's quality and relevance.

→ [Reach out on LinkedIn](https://www.linkedin.com/in/hugo-vantighem-9669a14b)

---

## Benchmarks

| # | Stack | Stars | LOC (approx) | Findings | Report |
|---|-------|-------|-------------|----------|--------|
| Target-A | Go + SQLite | 50k+ ⭐ | ~90k | 8 (1 HIGH, 7 MEDIUM) | [full](audits/target-a/audit_full.md) · [summary](audits/target-a/audit_summary.md) |

---

## Scorecard — Target-A

| Metric | Value |
|--------|-------|
| Findings surfaced | 8 |
| False positives | 0 |
| HIGH severity | 1 |
| MEDIUM severity | 7 |
| Confidence level | HIGH across all 8 |
| Falsifier demotions | 1 (JS sandbox: HIGH → MEDIUM) |
| Watchlist (narrow-window hypothesis) | 1 (CRON-DEADLOCK) |
| Hygiene items (LOW) | 6 |

---

## Methodology

Each audit runs a multi-agent pipeline covering: codebase exploration, architecture, reliability, concurrency, transactional integrity, and security — followed by an adversarial falsification pass that challenges each finding before synthesis.

Each finding includes:
- Severity level (HIGH / MEDIUM / LOW)
- Confidence level (HIGH / MEDIUM / LOW)
- File:line reference (validated against the actual repository)
- Reproduction path or trigger condition
- Counter-evidence section (the best argument *against* the finding)

---

## What these reports are

- First-pass diagnostics — not exhaustive audits
- Starting points for deeper investigation — not production-ready security assessments
- Structured observations with explicit uncertainty — not definitive verdicts

## What these reports are not

- Replacements for human code review
- Penetration test results
- Security certifications of any kind

---

## Independent critique strongly encouraged

Readers are invited to:

- Manually validate findings against the source code (file:line references are exact)
- Share the report with Claude / GPT-4o / Gemini / DeepSeek and ask them to critique the findings, challenge the confidence levels, or estimate what a senior engineer would charge to produce a comparable analysis
- Challenge incorrect reasoning in the Issues tab
- Submit corrections or counter-analyses as PRs

**Critical feedback is more useful than positive feedback.** If a finding is wrong, say so — and say why.

---

## Current calibration status

This system is in active calibration. Known limitations:

- False positive rate varies by codebase complexity and stack familiarity
- Severity calibration is imperfect — may over- or under-weight findings without a demonstrated exploit path
- Coverage is not exhaustive — the system may miss subsystems that were not explored

---

## Contact

Hugo Vantighem — [LinkedIn](https://www.linkedin.com/in/hugo-vantighem-9669a14b)

If you work on one of these projects and want to discuss findings, correct errors, or run the system on a private codebase — reach out directly.
