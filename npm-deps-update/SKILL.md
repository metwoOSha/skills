---
name: npm-deps-update
description: Two-phase npm dependency maintenance: scan for vulnerabilities and outdated packages, write a review report to docs/, then apply only approved updates with tests. Use for npm audit, outdated deps, fix vulnerabilities.
---

# npm dependency check & update

Two phases, always separated by a human review:

1. **Check** (default, `/npm-deps-update` or `/npm-deps-update check`): scan, write a report
   to `docs/dependencies/`, stop. Changes nothing in the project except writing the report.
2. **Apply** (`/npm-deps-update apply`, or the user says the report is approved): read the
   reviewed report, apply only rows marked `update`, verify after every batch, record results.

Never go from Check to Apply in one run, even if everything looks safe. The review is the point.

## Files

```
docs/dependencies/
  policy.md                 standing rules the team keeps ("react: stay on 18.x"); read on every run
  YYYY-MM-DD-report.md      one report per check run
```

Create `docs/dependencies/` and an empty `policy.md` (template below) if missing.
If the project keeps docs elsewhere (e.g. `doc/`, `documentation/`), use that and say so.

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
- Non-zero exit codes from `npm audit` and `npm outdated` are expected (they mean "found
  something"). Parse the JSON, don't treat them as failures.

## Phase 1: Check

Track:

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

- Package manager from the lockfile: `package-lock.json` → npm, `pnpm-lock.yaml` → pnpm,
  `yarn.lock` → yarn (check `.yarnrc.yml` for Berry vs Classic). Commands below are npm;
  use the equivalents (`pnpm audit --json`, `pnpm outdated --format json`,
  `yarn npm audit --json`; overrides → `pnpm.overrides` / yarn `resolutions`).
- Workspaces / monorepo: if `workspaces` is set, scan with `--workspaces` (npm) or `-r` (pnpm)
  and record which workspace each package belongs to.
- Node and npm versions vs `engines` and `.nvmrc`. A mismatch goes in the report header.
- Verification scripts from `package.json`: typecheck, lint, test, build. Record the exact
  commands; Phase 2 runs them.
- Version style in `package.json` (`^`, `~`, exact) and `save-exact` in `.npmrc`. Updates must keep it.

### 2. Read policy.md

Each rule applies to the matching packages in this run: pre-fill their Decision as `skip`
(or cap the target), and put them in the "Held back by policy" section with the rule's reason.
A held-back package that has a known vulnerability is flagged ⚠ at the top of the report.
Rules past their `Review by` date are listed under "Policy rules to revisit".

### 3. Scan vulnerabilities

`npm audit --json` (and `npm audit signatures` if npm ≥ 9). For each entry in `vulnerabilities`:

- severity, advisory title + URL/GHSA id (from `via` objects), vulnerable range
- direct or transitive (`isDirect`); for transitive, the chain from a direct dependency
  (`npm ls <pkg>` shows it)
- prod or dev dependency (dev-only issues are lower priority, still listed)
- fix path, from `fixAvailable`:
  - `true` → `npm audit fix` handles it within ranges
  - object with `isSemVerMajor: false` → bump the named direct dependency
  - object with `isSemVerMajor: true` → needs a major of the named package (treat as a major update)
  - `false` → no upstream fix; candidate for `overrides` if a patched version of the vulnerable
    package exists and is range-compatible, otherwise "Not fixable now"

Optional deeper check: for High/Critical, grep whether the vulnerable function/API is actually
used and note "reachable / not reachable" in the row.

### 4. Scan outdated

`npm outdated --json` (add `--long` for dependency type). For each package: current, wanted,
latest, update type (patch / minor / major from semver), prod or dev.

Group packages that must move together and give them one row group, e.g. `react` + `react-dom` +
`@types/react` + `@types/react-dom`; `next` + `eslint-config-next`; one scope family
(`@tanstack/*`, `@testing-library/*`, `@storybook/*`); `typescript` + `@typescript-eslint/*`.
Check peer dependency ranges between them.

### 5. Research majors and risky fixes

For every major update (and any vulnerability fix that requires one):

- read the official changelog / release notes / migration guide (web search the exact versions;
  prefer the package's GitHub releases and docs over blogs). Record URLs.
- list the breaking changes, then grep the codebase for the affected APIs. Report only the
  ones that touch this code, with file paths and an effort estimate (S / M / L).
- check peer ranges: does anything else in the tree require the old major?

For patch/minor, skim release notes only for packages in the critical path (framework, router,
state, data fetching, auth, build tooling) and note anything surprising.

### 6. Write the report and stop

Write `docs/dependencies/YYYY-MM-DD-report.md` from the template below.

Default decisions:

| Case | Default |
|---|---|
| Vulnerability, non-breaking fix | `update` |
| Patch / minor | `update` |
| Major (including a vuln fix that needs one) | `review` |
| Held back by policy | `skip` |
| No fix available | `n/a` |

`review` means nothing happens until a human changes it to `update` or `skip`.

Then tell the user, briefly: path to the report, counts (vulns by severity; patch / minor /
major), the ⚠ items, and how to review: edit the Decision column, or say it in chat
("skip react", "update next only to 15.3"). Stop.

## Review (between phases)

If the user gives decisions in chat, write them into the report's Decision/Note columns yourself,
so the file stays the source of truth. When they skip a package, ask once:
**only this time, or add a rule to policy.md?** If a rule: add it with their reason (ask if none),
today's date, and a review date if they give one.

If they skip a package that has a vulnerability, say plainly that the vulnerability stays open,
and record that in the Note.

When the user says it's approved, set the report status to `APPROVED`.

## Phase 2: Apply

Track:

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

- Use the most recent report. Status must be `APPROVED`, or the user confirms now.
  Any row still `review` → ask, don't guess.
- Working tree clean (`git status`). If not, stop and ask.
- Create a branch: `chore/deps-YYYY-MM-DD`.

### 2. Baseline

Clean install (`npm ci`), then the verification commands from step 1 of Phase 1.
If anything already fails, stop: report what fails before any update. Otherwise
every later failure would be ambiguous.

### 3–6. Batches

Order: `npm audit fix` (no `--force`) → overrides for transitive vulns → all approved
patch/minor in one batch (split by group if it fails) → each approved major (or major group)
as its own batch.

Install exact target versions from the report (`npm install pkg@x.y.z ...`, `-D` for dev deps),
keeping the project's version prefix style. For overrides, add the entry to `package.json`
`overrides` and confirm with `npm ls <pkg>` that every copy resolved to the patched version.

For majors: apply the migration edits from the report's research, nothing more.

After **every** batch: install, then typecheck, lint, test, build.

- Pass → commit on the branch: `chore(deps): <batch summary>`.
- Fail → try one focused fix if the cause is obvious and within the approved migration. Still
  failing → revert the batch (`git checkout -- package.json package-lock.json` + `npm ci`, or
  revert the edits), mark those rows `failed` with the error summary, continue with the next batch.
  If a patch/minor batch fails, bisect by group to find the culprit rather than dropping the whole batch.

### 7. Final scan and results

Re-run `npm audit --json` and `npm outdated --json`. Fill the report's "Apply results" section:
what was updated (old → new), what failed and why, vulnerabilities before vs after, code edits
made, remaining open items. Set status to `APPLIED` (or `PARTIALLY APPLIED`).

Tell the user: branch name, counts before/after, failures, and anything needing a human
(failed majors, unfixable vulns). Offer to push / open a PR; don't do it unasked.

## Templates

### policy.md

```markdown
# Dependency policy

Rules read on every dependency check. Add a reason for every rule.

| Package | Rule | Reason | Added | Review by |
|---|---|---|---|---|
| react, react-dom | stay on 18.x | waiting for library X to support 19 | 2026-11-16 | 2027-02-01 |
```

Rule forms: `skip` (no updates at all), `stay on <major>.x`, `max <version>`,
`ignore advisory <GHSA-id>` (accepted risk, reason required).

### YYYY-MM-DD-report.md

```markdown
# Dependency report — YYYY-MM-DD

**Status:** AWAITING REVIEW   <!-- AWAITING REVIEW → APPROVED → APPLIED / PARTIALLY APPLIED -->
**Project:** <name> · <package manager + version> · Node <version> · branch <current>
**Verification:** `<typecheck>` · `<lint>` · `<test>` · `<build>`

## Summary
- Vulnerabilities: C critical, H high, M moderate, L low (P prod / D dev)
- Outdated: N (patch a, minor b, major c)
- ⚠ Needs attention: <held-back or unfixable packages with vulnerabilities>

## How to review
Edit **Decision** in each row: `update`, `skip`, or `update to <version>`. Add a reason to Note
when skipping. `review` rows must be decided before apply. Or tell Claude in chat.

## 1. Vulnerabilities
| # | Package | Severity | Advisory | Installed | Fixed in | Direct / via | Prod/Dev | Fix method | Reachable | Decision | Note |
|---|---|---|---|---|---|---|---|---|---|---|---|

## 2. Updates — patch / minor
| # | Package (group) | Prod/Dev | Current | Target | Type | Decision | Note |
|---|---|---|---|---|---|---|---|

## 3. Updates — major
| # | Package (group) | Current | Target | Breaking changes affecting us | Effort | Decision | Note |
|---|---|---|---|---|---|---|---|

Per major: changelog / migration links and affected files.

## 4. Held back by policy
| Package | Rule | Latest | Open vulnerabilities |
|---|---|---|---|

## 5. Not fixable now
| Package | Advisory | Why | Suggested action |
|---|---|---|---|

## 6. Policy rules to revisit

## 7. Apply results
_(filled in by the apply phase)_
```
