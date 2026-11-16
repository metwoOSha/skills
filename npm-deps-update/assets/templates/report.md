# Dependency report — YYYY-MM-DD

**Status:** AWAITING REVIEW   <!-- AWAITING REVIEW → APPROVED → APPLIED / PARTIALLY APPLIED -->
**Project:** <name> · <package manager + version> · Node <version>
**Scanned:** branch <current> · base <base branch> @ <commit> · behind base: <n> commits
**Verification:** `<typecheck>` · `<lint>` · `<test>` · `<build>`

## Summary

- Vulnerabilities: C critical, H high, M moderate, L low (P prod / D dev)
- Outdated: N (patch a, minor b, major c)
- ⚠ Needs attention: <held-back or unfixable packages with vulnerabilities>

## Since last report

<!-- Leave out on the first run. -->

**Previous:** [YYYY-MM-DD-report.md](YYYY-MM-DD-report.md) · status <status> · base <commit>

- Vulnerabilities: <n> new, <n> resolved, <n> still open (of which <n> accepted earlier)
- Needs a decision now: <n> rows (new packages, new target versions, unblocked, failed last time)
- Carried over unchanged: <n> rows (`skip` at the same target)
- Applied last time and outdated again: <list or "none">
- Blockers resolved since: <list or "none">

## How to review

Edit **Decision** in each row: `update`, `skip`, or `update to <version>`. Add a reason to Note
when skipping. `review` rows must be decided before apply. Or tell Claude in chat.
Rows marked "carried over" repeat an earlier decision and need no action unless you want to
change it.

## Blockers

Conditions that hold back several rows at once. Resolve these first; the rows they block stay
`review` until then.

| Blocker | Rows it blocks | What resolves it |
|---|---|---|

## 1. Vulnerabilities

| # | Package | Severity | Advisory | Installed | Fixed in | Direct / via | Prod/Dev | Fix method | Reachable | Decision | Note |
|---|---|---|---|---|---|---|---|---|---|---|---|

## 2. Updates — patch / minor

| # | Package (group) | Prod/Dev | Current | Target | Type | Decision | Note |
|---|---|---|---|---|---|---|---|

## 3. Updates — major

Includes real majors, `0.x` minor bumps, downgrade fixes and packages shared with other projects.

| # | Package (group) | Current | Target | Kind (major / 0.x / downgrade / shared) | Breaking changes affecting us (or "not researched: <reason>") | Effort | Decision | Note |
|---|---|---|---|---|---|---|---|---|

### Migration notes

<!-- Per major: changelog / migration guide links, affected files, required code edits. -->

## 4. Held back by policy

| Package | Rule | Latest | Open vulnerabilities |
|---|---|---|---|

## 5. Not fixable now

| Package | Advisory | Why | Suggested action |
|---|---|---|---|

## 6. Shared with other projects

| Package | Shared with / how | Constraint | Impact of updating |
|---|---|---|---|

## 7. Dependency hygiene

| Package | Finding (dev dep used at runtime / unused / duplicated) | Where | Suggested action |
|---|---|---|---|

## 8. Suggested policy rules

Accept or reject each; accepted rules are copied to `policy.md`.

| Package | Proposed rule | Reason | Accept? |
|---|---|---|---|

## 9. Policy rules to revisit

## 10. Apply results

_(filled in by the apply phase)_

- Branch:
- Vulnerabilities before → after:
- Updated (old → new):
- Failed (and why):
- Code edits made:
- Overrides added (remove when upstream fixes):
- Still open:
