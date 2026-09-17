---
name: security-sweep
description: Investigate whether a project (or every repo on the machine) is exposed to known and actively-exploited dependency vulnerabilities. Combines a deterministic per-ecosystem audit (npm audit, govulncheck, pip-audit, osv-scanner) with live web research into recent/active supply-chain incidents, then checks each project's lockfiles directly — the leading-indicator check that `npm audit` misses. Use when the user asks to "investigate security", "check for vulnerable dependencies", "are we affected by <a named CVE / package compromise>", "scan the projects for supply-chain issues", or wants a recurring security sweep.
---

# Security sweep

Answer one question well: **are these projects exposed to a dependency
vulnerability that matters right now?** "Matters right now" means either
catalogued-and-reachable, or an active incident in the wild that hasn't
fully propagated into the audit databases yet.

The value over a plain `npm audit` is the **research layer**. Advisory
databases lag disclosure, and malicious-package incidents are often
handled by unpublishing (no queryable advisory at all). So this sweep
pairs the deterministic tools with live research into what's actively
being exploited, and checks the lockfile *directly* for those names —
the fastest reliable signal, which no audit tool gives you.

Read-only by default: this investigates and reports. Only propose fixes;
don't apply version bumps without the user's go-ahead (they can break
builds).

## When to use
- "Investigate security" / "check for vulnerable dependencies" / "run a
  security sweep."
- "Are we affected by \<a named CVE, GHSA, or package-compromise story\>?"
  — the user read about something and wants to know their exposure.
- A periodic check (compose with the `schedule` skill for a recurring
  cloud sweep — see the end).

## Step 1 — Scope

Decide what to sweep. Default to the **current project**; offer a
multi-repo sweep if the user's phrasing is broad ("our projects",
"everything", "the fleet").

- Current project: the repo you're in.
- Multi-repo: enumerate the sibling repos next to the current one — i.e.
  the parent of the current git root (commonly `~/projects/*/` or
  `~/src/*/`). Confirm the directory with the user before walking it
  rather than assuming; a wrong guess wastes a long scan. For each repo,
  detect ecosystems by manifest (below), sweep in turn, report per
  project.

## Step 2 — Detect ecosystems and locate lockfiles

Per project, an ecosystem is present if its lockfile is:

| Ecosystem | Manifest / lockfile | Where it usually sits |
|-----------|---------------------|-----------------------|
| npm       | `package-lock.json` | `frontend/` or repo root |
| Go        | `go.mod` / `go.sum` | `backend/` or repo root |
| Python    | `poetry.lock`, `requirements.txt`, `uv.lock` | repo root |
| Swift/SPM | `Package.resolved`  | `ios/` or an `.xcodeproj` |

The lockfile — not the manifest — is what you scan: it pins the exact
transitive versions actually installed.

## Step 3 — Deterministic baseline (per ecosystem)

Run the ecosystem's own tooling. These catch everything already in the
advisory databases. Prefer tools with **reachability analysis** (they
report only vulns your code can actually reach) where available.

- **npm**: `cd <dir> && npm audit --omit=dev --audit-level=high` (what
  ships to users) and a plain `npm audit` (all, informational). No
  reachability — over-reports; weight prod findings.
- **Go**: `govulncheck ./...` — *does* reachability analysis (only flags
  vulns in called code); install once with
  `go install golang.org/x/vuln/cmd/govulncheck@latest`. The gold
  standard of these tools.
- **Python**: `pip-audit` (or `pip-audit -r requirements.txt`).
- **Any ecosystem, or when a tool is missing**: `osv-scanner` against the
  lockfile, run via container so there's nothing to install:
  `<runtime> run --rm -v "$PWD:/src:ro" ghcr.io/google/osv-scanner:latest scan --lockfile=/src/<lockfile>`.
  Broadest database; the fallback for Swift/SPM, which has thin
  first-party tooling.

Capture what each reports. Distinguish production/reachable from
dev/build-time — a high in a build tool that never ships is not the same
finding as one in a runtime dependency.

## Step 4 — Research the active threat landscape (the value-add)

This is what a bare audit can't do. Use WebSearch / WebFetch to find what
is *currently* active, then bring names back to Step 5.

- Recent supply-chain incidents in the project's ecosystems: search for
  recent npm/PyPI/Go compromise and malware campaigns; check the GitHub
  Advisory Database (github.com/advisories) and OSV (osv.dev) for
  recently-published entries touching packages you saw in Step 2.
- If the user named a specific CVE / GHSA / incident, fetch the
  authoritative advisory and extract the **exact affected package name(s)
  and version range(s)** — that's what you'll match against the lockfile.
- Prefer primary sources (the advisory, the maintainer's post, the
  registry's security notice) over aggregator headlines; note when
  something is unconfirmed.

Keep it current: date every claim, and say what your research window was
(e.g. "active incidents disclosed in the last ~8 weeks").

## Step 5 — Cross-reference names against the lockfile *directly*

For every package surfaced in Step 4, do not wait for the audit tool —
check the pinned tree yourself. This is the leading-indicator step:

```sh
# npm
cd <dir> && npm ls <package>
grep -n '"<package>"' package-lock.json
# Go
grep -n '<module-path>' go.sum
# Python
grep -in '<package>' poetry.lock requirements.txt uv.lock 2>/dev/null
# Swift
grep -rn '<package>' --include=Package.resolved .
```

Then compare the pinned version against the advisory's affected range.
Three outcomes per name: **exposed** (present and in range), **clear**
(absent, or present but outside the range), **uncertain** (transitive
version you couldn't fully resolve — dig further or flag it).

## Step 6 — Report

Per project, lead with the verdict, then the evidence:

- **Exposed** — package, pinned version, advisory/incident, whether it's
  production-reachable, and a concrete remediation (targeted version bump;
  never a blind `npm audit fix`). If a compromised version may have been
  *installed*, note its postinstall may already have run — the biggest
  reason an `ignore-scripts` default matters.
- **Clear** — the researched active threats checked and not present.
  Say what you checked, so "clear" means something.
- **Uncertain / not covered** — ecosystems with thin tooling (Swift),
  unresolved transitives, or research that couldn't confirm a range.
  Name the gap rather than implying full coverage.

Do not imply you found everything. State the window and the ecosystems
actually swept.

## Composing

- **Recurring sweep**: hand this skill's procedure to the `schedule`
  skill for a weekly cloud run across the repos, so a fresh incident is
  caught within days rather than whenever someone next thinks to look.
- **Before a release or a push**: run a scoped single-project sweep. It
  pairs naturally with a `make audit` / `make secret-scan` pre-push step
  if the project has one.

## Notes

- Audit tools are *lagging* indicators; the news and this skill's Step 4
  are the *leading* ones. When those disagree — research says a package is
  compromised but the tools are silent — trust the direct lockfile check
  and treat it as live.
- `npm audit` reports advisory severity, not *your* exploitability (no
  reachability analysis) — it over-reports. `govulncheck` is the opposite
  and the model to prefer where an ecosystem offers it.
- This skill answers exposure. Remediation (version bumps, pinning,
  `npm rebuild` for a scoped script exception) is a separate,
  user-approved step — surface the fix, don't apply it unprompted.
