---
name: npm-deps-update
description: "Two-phase npm dependency maintenance: scan for vulnerabilities and outdated packages, write a review report to docs/, then apply only approved updates with tests. Use for npm audit, outdated deps, fix vulnerabilities."
---

# npm dependency check & update

Two phases, always separated by a human review:

1. **Check** (default, `/npm-deps-update` or `/npm-deps-update check`): scan, write a report
   to `docs/dependencies/`, stop. Changes nothing in the project except writing the report.
2. **Apply** (`/npm-deps-update apply`, or the user says the report is approved): read the
   reviewed report, apply only rows marked `update`, verify after every batch, record results.

Never go from Check to Apply in one run, even if everything looks safe. The review is the point.

Paths to `assets/` and `references/` below are relative to this SKILL.md.

## Files in the project

```
docs/dependencies/
  policy.md                 standing rules the team keeps ("react: stay on 18.x"); read on every run
  YYYY-MM-DD-report.md      one report per check run
```

Create `docs/dependencies/` if missing, and `policy.md` from
[assets/templates/policy.md](assets/templates/policy.md). If the project keeps docs elsewhere
(`doc/`, `documentation/`), use that and say so.

## Hard rules

- Never `npm audit fix --force`. It silently installs major versions.
- Never `--legacy-peer-deps` / `--force` on install unless the project already sets it in `.npmrc`.
  A peer conflict is a finding for the report, not something to bypass.
- Never delete or regenerate the lockfile from scratch. Lockfile changes come only from installs.
- Never apply anything not marked `update` in the reviewed report.
- Never update a package that `policy.md` holds back, even if the report row says `update`:
  stop and ask which one wins.
- Never push, open a PR, publish, or touch deploy config unless the user asks.
- Code edits only for a major update the user approved, and only what its migration requires.
  List every such edit in the report.
- Non-zero exit codes from `npm audit` and `npm outdated` are expected. Parse the JSON.

Commands, JSON fields and pnpm/yarn equivalents: [references/npm-commands.md](references/npm-commands.md).

## Phase 1: Check

```
Dependency check:
- [ ] 1. Detect environment
- [ ] 2. Read policy.md
- [ ] 3. Scan vulnerabilities
- [ ] 4. Scan outdated
- [ ] 5. Research majors and risky fixes
- [ ] 6. Write report, stop
```

### 1. Detect environment

Record in the report header:
- package manager (from the lockfile) and its version; workspaces, if any
- Node version vs `engines` / `.nvmrc` (flag a mismatch)
- verification commands from `package.json`: typecheck, lint, test, build (Phase 2 runs exactly these)
- version style in `package.json` (`^`, `~`, exact) and `save-exact` in `.npmrc`; updates keep it

### 2. Read policy.md

Each rule pre-fills its packages' Decision as `skip` (or caps the target) and lists them under
"Held back by policy" with the reason. A held-back package with a known vulnerability is
flagged ⚠ in the summary. Rules past `Review by` go under "Policy rules to revisit".

### 3. Scan vulnerabilities

Run the audit (plus `npm audit signatures` on npm ≥ 9). Per advisory record: severity,
advisory id + URL, installed and fixed versions, direct or transitive (with the chain),
prod or dev, and the fix method derived from `fixAvailable` (see references). For
High/Critical, optionally grep whether the vulnerable API is actually used and mark
"reachable / not reachable".

### 4. Scan outdated

Per package: current, wanted, latest, update type (patch / minor / major), prod or dev.
Group packages that must move together (see references) and check peer ranges between them.

### 5. Research majors and risky fixes

For every major update, including vulnerability fixes that need one:
- read the official changelog / release notes / migration guide for the exact versions
  (package's GitHub releases and docs over blogs); record URLs
- grep the codebase for the APIs the breaking changes touch; report only those that affect
  this code, with file paths and effort S / M / L
- check whether anything else in the tree requires the old major

For patch/minor, skim release notes only for critical-path packages (framework, router,
state, data fetching, auth, build tooling).

### 6. Write the report and stop

Copy [assets/templates/report.md](assets/templates/report.md) to
`docs/dependencies/YYYY-MM-DD-report.md` and fill it. Default decisions:

| Case | Default |
|---|---|
| Vulnerability, non-breaking fix | `update` |
| Patch / minor | `update` |
| Major (including a vuln fix that needs one) | `review` |
| Held back by policy | `skip` |
| No fix available | `n/a` |

`review` means nothing happens until a human changes it to `update` or `skip`.

Tell the user briefly: report path, counts (vulns by severity; patch / minor / major),
the ⚠ items, and how to review: edit the Decision column, or say it in chat
("skip react", "update next only to 15.3"). Stop.

## Review (between phases)

- Decisions given in chat → write them into the report's Decision/Note columns yourself;
  the file is the source of truth.
- When a package is skipped, ask once: **only this time, or add a rule to policy.md?**
  For a rule, record the reason (ask if none), today's date and a review date if given.
- Skipping a package with a vulnerability → say plainly that it stays open; note it in the row.
- User approves → set report status to `APPROVED`.

## Phase 2: Apply

```
Dependency apply:
- [ ] 1. Preconditions
- [ ] 2. Baseline
- [ ] 3. Batch: audit fix
- [ ] 4. Batch: overrides
- [ ] 5. Batch: patch/minor
- [ ] 6. Batches: each approved major
- [ ] 7. Final scan + results
```

### 1. Preconditions

- Latest report has status `APPROVED`, or the user confirms now. Any row still `review` → ask.
- Working tree clean (`git status`); otherwise stop and ask.
- Create branch `chore/deps-YYYY-MM-DD`.

### 2. Baseline

Clean install, then the verification commands. If anything already fails, stop and report it:
otherwise every later failure is ambiguous.

### 3–6. Batches

Order: audit fix (no `--force`) → overrides for transitive vulns → all approved patch/minor
in one batch → each approved major (or major group) as its own batch.

- Install the exact target versions from the report, keeping the project's prefix style.
- Overrides: confirm every copy resolved to the patched version.
- Majors: apply only the migration edits identified in the report.

After **every** batch: install, typecheck, lint, test, build.

- Pass → commit on the branch: `chore(deps): <batch summary>`.
- Fail → one focused fix if the cause is obvious and within the approved migration. Still
  failing → revert the batch, mark its rows `failed` with the error summary, continue.
  A failing patch/minor batch is bisected by group to find the culprit.

### 7. Final scan and results

Re-run audit and outdated. Fill "Apply results" in the report: updated (old → new), failed
and why, vulnerabilities before vs after, code edits made, remaining open items. Set status
to `APPLIED` or `PARTIALLY APPLIED`.

Tell the user: branch, counts before/after, failures, what needs a human. Offer to push /
open a PR; don't do it unasked.
