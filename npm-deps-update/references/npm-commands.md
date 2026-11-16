# Commands and data reference

## Package manager detection

| Lockfile | Manager | Notes |
|---|---|---|
| `package-lock.json` | npm | |
| `pnpm-lock.yaml` | pnpm | |
| `yarn.lock` + `.yarnrc.yml` | Yarn Berry (2+) | |
| `yarn.lock` without `.yarnrc.yml` | Yarn Classic (1.x) | |

More than one lockfile → report it as a finding and ask which one is authoritative.

## Command equivalents

| Purpose | npm | pnpm | Yarn Berry | Yarn Classic |
|---|---|---|---|---|
| Clean install | `npm ci` | `pnpm install --frozen-lockfile` | `yarn install --immutable` | `yarn install --frozen-lockfile` |
| Vulnerabilities | `npm audit --json` | `pnpm audit --json` | `yarn npm audit --all --recursive --json` | `yarn audit --json` |
| Outdated | `npm outdated --json --long` | `pnpm outdated --format json` | no built-in: compare each direct dep with `yarn npm info <pkg> --fields version --json` | `yarn outdated --json` |
| Safe auto-fix | `npm audit fix` | `pnpm audit --fix` (writes overrides) | none, use resolutions | none, use resolutions |
| Install exact version | `npm install pkg@x.y.z` (`-D` for dev) | `pnpm add pkg@x.y.z` (`-D`) | `yarn add pkg@x.y.z` (`-D`) | `yarn add pkg@x.y.z` (`-D`) |
| Why is it installed | `npm ls <pkg>` / `npm explain <pkg>` | `pnpm why <pkg>` | `yarn why <pkg>` | `yarn why <pkg>` |
| Force a transitive version | `"overrides"` in package.json | `"pnpm": { "overrides": {} }` | `"resolutions"` | `"resolutions"` |
| Workspaces | `--workspaces` | `-r` | `--recursive` / `-R` | `--cwd` per workspace |

Never run interactive commands; they hang the agent.

## `npm audit --json` (npm 7+)

```jsonc
{
  "vulnerabilities": {
    "<package>": {
      "name": "<package>",
      "severity": "low | moderate | high | critical",
      "isDirect": true,
      "via": [ /* string = comes through that package; object = the advisory itself */
        { "source": 1234, "name": "<package>", "title": "...", "url": "https://github.com/advisories/GHSA-...",
          "severity": "high", "range": "<1.2.3" }
      ],
      "effects": ["<packages that depend on this one>"],
      "range": "<1.2.3",
      "fixAvailable": true | false | { "name": "<direct dep>", "version": "x.y.z", "isSemVerMajor": false }
    }
  },
  "metadata": { "vulnerabilities": { "low": 0, "moderate": 0, "high": 0, "critical": 0, "total": 0 } }
}
```

Fix method from `fixAvailable`:

| Value | Fix method in report |
|---|---|
| `true` | `audit fix` (within existing ranges) |
| `{ isSemVerMajor: false }` | bump `<name>` to `<version>` |
| `{ isSemVerMajor: true }` | compare `version` with the installed `<name>` first: **higher** → major update, Major table, `review`; **lower** → `downgrade` fix (npm found no fixed newer release and offers an older one), Major table, `review`, Note says which features/fixes the downgrade drops |
| `false` | `override` if a patched version of the vulnerable package exists and satisfies its dependents' ranges; otherwise "Not fixable now" |

Exit code is non-zero when vulnerabilities exist. That is not an error.

## `npm outdated --json --long`

```jsonc
{
  "<package>": {
    "current": "1.2.3",   // installed
    "wanted": "1.4.0",    // max satisfying the package.json range
    "latest": "2.0.0",    // dist-tag latest
    "type": "dependencies | devDependencies",
    "location": "node_modules/<package>",
    "dependent": "<workspace or root>"
  }
}
```

`current` missing → package not installed; run a clean install first.
Update type comes from comparing `current` with the target (`latest`, or `wanted` if capped):

| Change | Type |
|---|---|
| `1.2.3 → 1.2.4` | patch |
| `1.2.3 → 1.3.0` | minor |
| `1.2.3 → 2.0.0` | major |
| `0.4.2 → 0.4.3` | patch |
| `0.4.2 → 0.5.0` | **major** (0.x minor is breaking) |
| `0.0.3 → 0.0.4` | **major** (0.0.x any change is breaking) |
| target lower than current | **downgrade** |

Note: `^0.4.2` already only allows `0.4.x`, so `wanted` never crosses a 0.x minor; `latest` can.
Exit code 1 when anything is outdated. That is not an error.

## Overrides (npm)

```json
{
  "overrides": {
    "vulnerable-pkg": "1.2.4",
    "parent-pkg": { "vulnerable-pkg": "1.2.4" }
  }
}
```

Prefer the scoped form (`parent-pkg > vulnerable-pkg`) when only one path is affected.
After install, `npm ls vulnerable-pkg` must show only patched versions. Note each override in
the report so it can be removed once the parent package ships the fix.

## Packages that move together

Update as one group, one row, one batch:

- `react`, `react-dom`, `@types/react`, `@types/react-dom`
- `next`, `eslint-config-next`, `@next/*`
- `typescript`, `@typescript-eslint/*`, `typescript-eslint`
- `eslint` and its plugins/configs
- one scope family: `@tanstack/*`, `@testing-library/*`, `@storybook/*`, `@nestjs/*`, `@radix-ui/*`
- `jest`, `ts-jest`, `@types/jest`, `babel-jest`; `vitest`, `@vitest/*`
- `prisma`, `@prisma/client`

Also group any packages whose peer ranges pin each other (check `peerDependencies`).

## Cross-project dependencies

Signals that a dependency is shared with other repos:

- spec is `git+...`, `github:...`, `file:...`, `link:...`, `workspace:...`
- package is under the organisation's scope (look at `name` of this repo and its private
  registry in `.npmrc`)
- this repo is a library: its `peerDependencies` ranges are a contract with consumers;
  widening or bumping them affects every consumer
- guard / check scripts in the repo or CI that assert versions (grep CI config and `scripts/`
  for package names and version strings)
- comments or docs saying "keep in sync with ..."

## devDependencies used at runtime

```bash
# list devDependencies
node -e "console.log(Object.keys(require('./package.json').devDependencies||{}).join('\n'))"
# then for each name, search runtime source only
grep -rlE "from ['\"]<name>(/|['\"])|require\(['\"]<name>" src app lib pages --include=*.{ts,tsx,js,jsx,mjs,cjs}
```

Exclude `*.test.*`, `*.spec.*`, `*.stories.*`, `__tests__/`, `__mocks__/`, config files and
`scripts/`. Type-only imports (`import type`) are fine as dev deps.

Test helpers live next to runtime code under names the patterns above miss
(`*Fixtures.ts`, `test-utils.ts`, `setupTests.ts`, `*.mock.ts`). Treat a hit as a test helper,
not runtime code, when the file itself imports a test runner (`vitest`, `jest`,
`@testing-library/*`) or its name contains `fixture`, `test-util`, `mock` or `setup`.

For a library with a build, confirm against the output instead of trusting file names: a
devDependency that is not in the bundler's `external` list and is imported by any built entry
ends up inside `dist`. After a build, `grep -rl "<distinctive export or string of the package>" dist`
settles it; so does the bundler's manifest or stats file when the project emits one.

## Unused dependencies

Declared but never imported is a candidate, not a finding. A text search for the package name
misses these uses, so rule them out first:

| Used without its name in an import | Where to look |
|---|---|
| ESLint plugins and configs by short name (`plugin:react-hooks/…`, `'prettier'`, `storybook`) | ESLint config |
| Tools a framework loads implicitly (`sass` / `less` for style files, `@vitest/coverage-*` for `--coverage`, `postcss` plugins, Babel presets) | build / test config, style file extensions |
| CLIs run from scripts (`tsc` → `typescript`, `changeset` → `@changesets/cli`) | `package.json` scripts, CI config, git hooks |
| Peer dependencies of another installed package (`@testing-library/dom` for `@testing-library/react`) | `peerDependencies` of the other direct deps |
| `@types/*` packages | the matching runtime package, `tsconfig` `types` |
| Storybook addons, Vite/Webpack plugins by string | `.storybook/main.*`, bundler config |

If the project already has `knip` or `depcheck`, prefer its result and still check the table
for anything it flags. Do not install either one just for the check.

## Peer dependency conflicts

`ERESOLVE` on install means the target breaks a peer range. Do not bypass it. Find which
package holds the old peer range (`npm explain <pkg>`), check whether a newer version of that
package widens it, and either add it to the group or record the conflict in the report.
