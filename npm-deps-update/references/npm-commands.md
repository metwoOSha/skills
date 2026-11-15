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
| `{ isSemVerMajor: true }` | major of `<name>` → goes to the Major table as `review` |
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
Update type comes from comparing `current` with the target (`latest`, or `wanted` if capped).
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

## Peer dependency conflicts

`ERESOLVE` on install means the target breaks a peer range. Do not bypass it. Find which
package holds the old peer range (`npm explain <pkg>`), check whether a newer version of that
package widens it, and either add it to the group or record the conflict in the report.
