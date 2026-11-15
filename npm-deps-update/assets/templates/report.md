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

### Migration notes

<!-- Per major: changelog / migration guide links, affected files, required code edits. -->

## 4. Held back by policy

| Package | Rule | Latest | Open vulnerabilities |
|---|---|---|---|

## 5. Not fixable now

| Package | Advisory | Why | Suggested action |
|---|---|---|---|

## 6. Policy rules to revisit

## 7. Apply results

_(filled in by the apply phase)_

- Branch:
- Vulnerabilities before → after:
- Updated (old → new):
- Failed (and why):
- Code edits made:
- Overrides added (remove when upstream fixes):
- Still open:
