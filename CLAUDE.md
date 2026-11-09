# CLAUDE.md

## What this is

`@sudobility/sider_ui` — a **headless, presentation-only** React library of
shared Sider dashboard pages + components, consumed by `sider_app` and
`sider_extension`. No router, no data client, no auth, no CSS of its own; the
host injects all of it via two context providers. ESM only (`"type": "module"`).

## Stack

- React 19, TypeScript 6, Vitest 4. Package manager: **Bun** (`bun.lock`).
- Build: `tsc` → `dist/` (no bundler). Published to **GitHub Packages**.
- Peer deps (host-provided): `@sudobility/{components,design,sider_client,sider_types,types}`, `@tanstack/react-query`, `@heroicons/react`, `react`, `react-dom`.

## Commands (verified)

```bash
bun install          # needs read:packages token for @sudobility scope (see .npmrc)
bun run typecheck    # tsc --noEmit over all of src
bun run test:unit    # vitest run
bun run build        # tsc -p tsconfig.build.json + scripts/fix-esm-imports.mjs
bun run verify       # typecheck + test:unit — the CI/pre-publish gate
```

Run `bun run verify` before considering a change done.

- **No `lint` script.** `eslint.config.js` exists but is not scripted and does
  not run under the pinned ESLint 10 / typescript-eslint versions (throws). Do
  not add or document a lint command as working without fixing that first.
- Formatting: Prettier (`.prettierrc.json`: single quotes, semicolons, width 100).

## Architecture

- Everything public is re-exported through `src/index.ts` (a barrel). New
  exports MUST be added there.
- **Two required providers** wrap all pages:
  - `SiderUiProvider` (`src/context/config.tsx`) → `networkClient`, Firebase
    `token`, `apiUrl`, optional `SeoHead`. Read via `useSiderApi()`.
  - `RoutingProvider` (`src/context/routing.tsx`) → host router binding:
    `params`, `pathname`, `navigate`, language, and `routes` (host-owned
    app-relative URL builders). Read via `useSiderRouting()`/`useRoutes()`/
    `useRouteParams()`. Hooks throw if used outside their provider.
- Data fetching is delegated to `@sudobility/sider_client` React Query hooks
  (`useMe`, `useMyCaptureBatches`, `useMyTools`, …); the host must also supply a
  `@tanstack/react-query` provider. This library never calls the API directly.
- `src/pages/*` compose `src/components/*` (DataTable, JsonViewer, states,
  DashboardSidebar) with those hooks.

## Conventions / gotchas

- **Styling = Tailwind utilities on design-token CSS vars** (`text-foreground`,
  `bg-card`, `border-border`, `bg-destructive/10`, …) from `@sudobility/design`.
  The package emits **no CSS**. Hosts must add
  `./node_modules/@sudobility/sider_ui/dist/**/*.js` to their Tailwind content
  globs. Reuse existing token classes; don't hardcode colors.
- `tsconfig.json` has `noEmit: true`; builds go through `tsconfig.build.json`.
  Tests are excluded from the build.
- **Do not remove `scripts/fix-esm-imports.mjs`.** `moduleResolution: "bundler"`
  lets `tsc` emit extensionless relative imports; this post-build step rewrites
  them to `./x/index.js` so Node's native ESM loader can resolve the package.
- `@sudobility`-scoped installs need GitHub Packages auth (`.npmrc` →
  `npm.pkg.github.com`); missing token = unresolved deps.
- CI (`.github/workflows/ci-cd.yml`) uses the reusable
  `johnqh/workflows` unified-cicd on `main`/`develop`; publishes to GitHub
  Packages when `package.json` version bumps.
- License: BUSL-1.1.
