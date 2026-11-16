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

A second Check on the same day: overwrite today's report only while its status is
`AWAITING REVIEW`. If it is `APPROVED` or later, it holds decisions; write
`YYYY-MM-DD-2-report.md` instead and say so.

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
- [ ] 2. Read policy.md (+ 2b previous report)
- [ ] 3. Scan vulnerabilities
- [ ] 4. Scan outdated (+ 4b cross-project and hygiene checks)
- [ ] 5. Research majors and risky fixes
- [ ] 6. Write report, stop
```

### 1. Detect environment

**Branch freshness first.** `git fetch`, find the base branch (the one the team merges into:
`dev`, `develop`, `main`; ask if unclear) and compare: `git rev-list --count HEAD..origin/<base>`.
If the current branch is behind, or `package.json` / the lockfile differ from the base, stop and
ask: update the branch, or scan `origin/<base>` instead. A scan of a stale branch reports
problems that are already fixed.

Record in the report header:
- branch, base branch, and the base commit the scan ran against
- package manager (from the lockfile) and its version; workspaces, if any
- Node version vs `engines` / `.nvmrc` (flag a mismatch)
- verification commands from `package.json`: typecheck, lint, test, build (Phase 2 runs exactly these)
- version style in `package.json` (`^`, `~`, exact) and `save-exact` in `.npmrc`; updates keep it

### 2. Read policy.md

Each rule pre-fills its packages' Decision as `skip` (or caps the target) and lists them under
"Held back by policy" with the reason. A held-back package with a known vulnerability is
flagged ⚠ in the summary. Rules past `Review by` go under "Policy rules to revisit".

### 2b. Read the previous report

Runs are meant to repeat, so a check should only ask a human about what changed. Find the
latest earlier report in `docs/dependencies/` (by date in the file name). None → first run,
skip this step and leave "Since last report" out.

From it take: date, status, base commit, every row's package, target and Decision, the
Blockers table, and Apply results if any. Then, after steps 3–4, compare (rules below) and fill
"Since last report". Carrying over a decision is not applying it: carried-over rows are still
listed in their table so the report stays complete.

| Previous row | Now | Default Decision |
|---|---|---|
| `skip` | same target version | `skip`, Note "carried over from <date>" |
| `skip` | newer target available | the normal default (table in step 6), Note "skipped <date> at <old target>" |
| `skip` on a vulnerability | still open | `skip` carried over, and still ⚠ in the summary: an accepted risk is not a fixed one |
| `update` in an `APPLIED` report | still outdated | the normal default, Note "applied <date> but outdated again" |
| `failed` | any | `review`, Note "failed <date>: <reason>" |
| held by a blocker | blocker resolved | the normal default, Note "unblocked: <what changed>" |
| anything in a report that never reached `APPROVED` | — | nothing carries over; that report was not reviewed |

A carried-over decision never outranks `policy.md`, and a carried-over `skip` never hides a new
vulnerability on the same package: that vulnerability gets its own row with the normal default.

### 3. Scan vulnerabilities

Run the audit (plus `npm audit signatures` on npm ≥ 9). Per advisory record: severity,
advisory id + URL, installed and fixed versions, direct or transitive (with the chain),
prod or dev, and the fix method derived from `fixAvailable` (see references).
**Always compare `fixAvailable.version` with the installed version of that package:** npm
sometimes "fixes" by going back to an older release. A lower version is a `downgrade` fix, goes
to the Major table as `review`, and the row says what would be lost. For
High/Critical, optionally grep whether the vulnerable API is actually used and mark
"reachable / not reachable".

### 4. Scan outdated

Per package: current, wanted, latest, update type (patch / minor / major), prod or dev.
Group packages that must move together (see references) and check peer ranges between them.

Update type follows semver's breaking rules, not the digit that changed:
- `0.y.z`: a minor bump (`0.4.x → 0.5.0`) is **breaking** → classify as major
- `0.0.z`: any bump is breaking → classify as major
- pre-releases (`-beta`, `-rc`) are never targets unless already installed

### 4b. Cross-project and hygiene checks

- **Shared with other projects.** Find dependencies that tie this repo to others: internal
  packages (org scope such as `@company/*`, `git+`, `file:`, `link:`, `workspace:` specs),
  this repo's own `peerDependencies` if it is a library (consumers must accept the new range),
  and packages the team keeps on one version across repos (check for shared configs, guard
  scripts, or ask). Mark these rows "shared: <where>" and default them to `review`.
  If sibling repos are not discoverable, ask the user once which repos share dependencies.
- **devDependencies used at runtime.** Grep runtime source (not tests, stories, configs, scripts,
  test helpers) for imports of devDependencies. Each hit is a hygiene finding: it should probably
  be a dependency (matters for servers, libraries and anything installed with `--omit=dev`).
  For a library with a build, confirm against the build output: a package that ends up in `dist`
  ships at runtime whatever its source file is called. Recipe and exclusions in references.
- **Unused or duplicated.** Same package in both dependencies and devDependencies; packages
  declared but never used. A plain text search over-reports unused packages: rule out the
  implicit uses listed in references before reporting one, or use `knip` / `depcheck` if the
  project already has it. Report only; never remove without a decision.

### 5. Research majors and risky fixes

Research is the expensive step; spend it only where an update can actually happen. Skip the
changelog reading for a major that is already decided by something else, and say why in its row
("not researched: <reason>"):
- held back by `policy.md`
- unused (§ hygiene): the decision is removal, not an update
- blocked by a prerequisite the project does not meet yet (Node version, a shared package that
  must move in other repos first); research it in the run after the blocker is resolved
- a downgrade fix

For every other major update, including vulnerability fixes that need one:
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
| Patch / minor (real, per the semver rules above) | `update` |
| Major, `0.x` minor, downgrade fix, or shared with other projects | `review` |
| Held back by policy | `skip` |
| No fix available | `n/a` |

`review` means nothing happens until a human changes it to `update` or `skip`.

When one condition blocks several rows (a Node version the targets require, a package that must
move in other repos first, a browser baseline), write it once under "Blockers" with the rows it
blocks, instead of repeating it in every Note.

When the scan shows a lasting reason to hold something back (a major blocked by another
library, a version pinned for another repo, an accepted advisory), add it to
"Suggested policy rules" in the report. Suggestions are never written to `policy.md` until
the user accepts them.

Tell the user briefly: report path, counts (vulns by severity; patch / minor / major),
what changed since the last report (new / resolved vulnerabilities, rows needing a decision vs
carried over), the blockers and ⚠ items, and how to review: edit the Decision column, or say it in chat
("skip react", "update next only to 15.3"). Stop.

## Review (between phases)

- Decisions given in chat → write them into the report's Decision/Note columns yourself;
  the file is the source of truth.
- When a package is skipped, ask once: **only this time, or add a rule to policy.md?**
  For a rule, record the reason (ask if none), today's date and a review date if given.
- Skipping a package with a vulnerability → say plainly that it stays open; note it in the row.
- Accepted "Suggested policy rules" → copy them to `policy.md`; mark rejected ones in the report.
- User approves → set report status to `APPROVED`.
- `policy.md` and reports are meant to be committed (the policy is shared team state; reports
  are the history of decisions). Commit them with the dependency branch in Phase 2, or on their
  own if nothing gets applied. Never commit them into an unrelated feature branch; ask if the
  current branch is not a dependency branch.

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
- Base branch still matches the commit the report was made against. If it moved and touched
  `package.json` or the lockfile, re-run Check first.
- Create branch `chore/deps-YYYY-MM-DD` from the up-to-date base (or the name the team's
  ticket convention requires).

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
