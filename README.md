# @sudobility/sider_ui

Shared Sider dashboard UI: a headless React component/page library consumed by
both **sider_app** and **sider_extension**. It ships the dashboard pages
(overview, capture batches, sites, tools) plus the shared data components and
context providers they depend on.

The package is **presentation only**. It has no router, no data client, no auth,
and no CSS of its own — the host application injects all of those through two
context providers (see [Integration](#integration)).

- **License:** BUSL-1.1
- **Module format:** ESM only (`"type": "module"`)
- **Published to:** GitHub Packages (`@sudobility` scope)

## Requirements

- [Bun](https://bun.sh) (the repo is pinned via `bun.lock`)
- Access to the private `@sudobility` scope on GitHub Packages. `.npmrc` maps
  `@sudobility:registry=https://npm.pkg.github.com`, so `bun install` needs a
  GitHub token with `read:packages` (e.g. via `~/.npmrc` or the
  `GH_PACKAGES_READ_TOKEN` env the CI uses). Without it the `@sudobility/*`
  dependencies will not resolve.

## Install

```bash
bun install
```

## Scripts

| Command                | What it does                                                        |
| ---------------------- | ------------------------------------------------------------------- |
| `bun run build`        | Compile `src` → `dist` with `tsc` (`tsconfig.build.json`), then run `scripts/fix-esm-imports.mjs` to make the ESM output Node-resolvable. |
| `bun run build:watch`  | Incremental build in watch mode (no import-fix step).               |
| `bun run clean`        | Remove `dist`.                                                      |
| `bun run typecheck`    | `tsc --noEmit` against the whole `src` tree.                        |
| `bun run test:unit`    | Run the Vitest suite (`vitest run --passWithNoTests`).             |
| `bun run verify`       | `typecheck` + `test:unit` — the pre-publish/CI gate.                |
| `bun run prepublishOnly` | `clean` + `build`; runs automatically on `bun publish`.           |

There is no `lint` script; an `eslint.config.js` exists but is not wired to a
script (and does not run under the currently pinned ESLint/`typescript-eslint`
versions). Formatting is handled by Prettier (`.prettierrc.json`).

## Build output

`dist/index.js` (`main`) and `dist/index.d.ts` (`types`); only `dist/**/*` is
published. `tsconfig.json` itself has `noEmit: true` — always build through
`tsconfig.build.json` (what `bun run build` does). The `fix-esm-imports.mjs`
post-build step rewrites extensionless/directory relative imports (allowed by
`moduleResolution: "bundler"`) into fully-qualified `./x/index.js` paths so
Node's native ESM loader can resolve them; do not remove it.

## Integration

The library renders nothing useful until the host wraps it in both providers:

```tsx
import { SiderUiProvider, RoutingProvider, DashboardOverview } from '@sudobility/sider_ui';

<SiderUiProvider networkClient={client} token={firebaseIdToken} apiUrl="http://localhost:8090">
  <RoutingProvider
    params={params}
    pathname={pathname}
    navigate={navigate}
    currentLanguage={lang}
    routes={routes} // host-owned URL builders (home(), site(id), …)
  >
    <DashboardOverview />
  </RoutingProvider>
</SiderUiProvider>;
```

- **`SiderUiProvider`** (`src/context/config.tsx`) supplies the API transport:
  `networkClient` (`@sudobility/types`), the Firebase `token`, the `apiUrl`, and
  an optional `SeoHead` component. Pages read it via `useSiderApi()`.
- **`RoutingProvider`** (`src/context/routing.tsx`) supplies the router binding
  from the host's framework: `params`, `pathname`, `navigate`, language, and a
  `routes` object of app-relative URL builders. Consumed via `useSiderRouting()`,
  `useRoutes()`, `useRouteParams()`.

Data fetching is delegated to `@sudobility/sider_client` React Query hooks
(`useMe`, `useMyCaptureBatches`, `useMyTools`, …), so the host must also provide
a `@tanstack/react-query` `QueryClientProvider`.

### Styling

Components use Tailwind utility classes against design-token CSS variables
(`text-foreground`, `bg-card`, `border-border`, `bg-destructive/10`, …) from
`@sudobility/design`. The package ships **no compiled CSS**. The host must:

1. Provide the Tailwind setup and design tokens.
2. Add this package to its Tailwind `content` globs:
   `'./node_modules/@sudobility/sider_ui/dist/**/*.js'`

## Source layout

```
src/
  index.ts                     barrel: re-exports everything public
  context/
    config.tsx                 SiderUiProvider, useSiderApi, SEOHead
    routing.tsx                RoutingProvider, useSiderRouting/useRoutes/…
  components/
    states/index.tsx           LoadingState, EmptyState, ErrorState, StatusBadge
    data/DataTable.tsx         generic table
    data/JsonViewer.tsx        JSON display
    data/secretSegments.ts     splits `{{secret:…}}` placeholders (+ .test.ts)
    dashboard/DashboardSidebar.tsx
  pages/                       DashboardOverview, CaptureBatches(+Detail),
                               MyTools, Sites, SiteDetail, ToolDetail
```

Anything exported must be added to `src/index.ts`.

## CI / publishing

`.github/workflows/ci-cd.yml` calls the reusable
`johnqh/workflows/.github/workflows/unified-cicd.yml` on pushes/PRs to `main` and
`develop`, publishing to **GitHub Packages** (`publish-target: github`) when the
version in `package.json` is new.
