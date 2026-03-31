# Tech Context — Excalidraw

> Cross-ref: [domain-glossary.md](../product/domain-glossary.md) · [architecture.md](../technical/architecture.md) · [code-vs-docs.md](../technical/code-vs-docs.md) · [dev-setup.md](../technical/dev-setup.md)

## Runtime & Language

| Item | Version / Detail |
|------|-----------------|
| **Node.js** | ≥ 18.0.0 |
| **TypeScript** | 5.9.3 |
| **Package manager** | Yarn 1.22.22 (classic) |
| **Module system** | ESM (`"type": "module"` in packages) |
| **TS target** | ESNext |
| **TS module resolution** | Node |
| **Strict mode** | Enabled |

## Core Framework & Libraries

| Library | Version | Purpose |
|---------|---------|---------|
| React | 19.0.0 | UI rendering |
| React DOM | 19.0.0 | DOM bindings |
| Jotai | 2.11.0 | Atomic state management |
| jotai-scope | 0.7.2 | Scoped Jotai stores |
| Rough.js | 4.6.4 | Hand-drawn rendering |
| perfect-freehand | 1.2.0 | Freehand stroke smoothing |
| clsx | 1.1.1 | CSS class merging |
| nanoid | 3.3.3 | ID generation |
| pako | 2.0.3 | Compression (deflate/inflate) |
| fractional-indexing | 3.2.0 | Fractional z-index ordering |
| lodash.throttle | 4.1.1 | Throttling |
| lodash.debounce | 4.0.8 | Debouncing |
| radix-ui | 1.4.3 | Accessible UI primitives |

## Backend / Collaboration

| Library | Version | Purpose |
|---------|---------|---------|
| Firebase | 11.3.1 | Firestore (persistence) + Storage (files) |
| socket.io-client | 4.7.2 | Real-time WebSocket transport |
| idb-keyval | 6.0.3 | IndexedDB key-value storage |

## Build Tooling

| Tool | Version | Purpose |
|------|---------|---------|
| Vite | 5.0.12 | Dev server & bundler |
| @vitejs/plugin-react | 3.1.0 | React Fast Refresh |
| vite-plugin-svgr | 4.2.0 | SVG as React components |
| vite-plugin-pwa | 0.21.1 | PWA / service worker |
| vite-plugin-checker | 0.7.2 | Type-check overlay |
| vite-plugin-ejs | 1.7.0 | HTML templating |
| vite-plugin-html | 3.2.2 | HTML transform |
| esbuild | 0.19.10 | Package ESM builds |
| sass | 1.51.0 | SCSS compilation |

## Testing

| Tool | Version | Purpose |
|------|---------|---------|
| Vitest | 3.0.6 | Test runner (vitest globals) |
| @vitest/coverage-v8 | 3.0.7 | Code coverage |
| @testing-library/react | 16.2.0 | Component testing |
| @testing-library/jest-dom | 6.6.3 | DOM matchers |
| jsdom | 22.1.0 | Browser environment |
| vitest-canvas-mock | 0.3.3 | Canvas API mock |
| fake-indexeddb | 3.1.7 | IndexedDB mock |

### Coverage Thresholds

```
lines:      60%
branches:   70%
functions:  63%
statements: 60%
```

## Linting & Formatting

| Tool | Detail |
|------|--------|
| ESLint | `eslint-config-react-app` based, `@excalidraw/eslint-config` |
| Prettier | `@excalidraw/prettier-config` |
| Husky | 7.0.4 — Git hooks |
| lint-staged | 12.3.7 — Pre-commit checks |

## Deployment

| Target | Stack |
|--------|-------|
| **Vercel** | Static site, custom headers (`vercel.json`) |
| **Docker** | Multi-stage: Node 18 build → Nginx 1.27-alpine serve |
| **npm** | Published via `scripts/release.js` (tags: test / next / latest) |

## Key CLI Commands

### Development
```bash
yarn start                  # Dev server on port 3000
yarn start:production       # Build + serve locally on port 5001
yarn start:example          # Build packages + run browser example
```

### Building
```bash
yarn build                  # Build app (Vite) + version stamp
yarn build:app:docker       # Build without Sentry
yarn build:packages         # Build all library packages (common → math → element → excalidraw)
yarn build:excalidraw       # Build excalidraw package only
```

### Testing
```bash
yarn test:app               # Vitest (watch mode)
yarn test:all               # Typecheck + lint + format + tests
yarn test:code              # ESLint
yarn test:typecheck         # tsc --noEmit
yarn test:coverage          # Vitest with coverage
```

### Maintenance
```bash
yarn fix                    # Auto-fix formatting + lint
yarn clean-install          # rm node_modules + fresh install
yarn rm:build               # Remove all build artifacts
yarn release                # Publish packages to npm
```

## Environment Variables (selected)

| Variable | Purpose |
|----------|---------|
| `VITE_APP_FIREBASE_CONFIG` | Firebase project config (JSON) |
| `VITE_APP_GIT_SHA` | Git commit SHA for Sentry |
| `VITE_APP_DISABLE_SENTRY` | Disable error tracking |
| `VITE_APP_ENABLE_TRACKING` | Enable analytics |
| `VITE_APP_PORT` | Dev server port (default 3000) |

## Workspace Structure (Yarn Workspaces)

```
root (excalidraw-monorepo)
├── excalidraw-app        — standalone web app
├── packages/common       — @excalidraw/common   0.18.0
├── packages/math         — @excalidraw/math     0.18.0
├── packages/element      — @excalidraw/element  0.18.0
├── packages/excalidraw   — @excalidraw/excalidraw 0.18.0
├── packages/utils        — @excalidraw/utils    0.1.2
├── examples/with-nextjs
└── examples/with-script-in-browser
```

Path aliases are configured in `tsconfig.json` and mirrored in Vite configs:
- `@excalidraw/common` → `packages/common/src/index.ts`
- `@excalidraw/element` → `packages/element/src/index.ts`
- `@excalidraw/excalidraw` → `packages/excalidraw/index.tsx`
- `@excalidraw/math` → `packages/math/src/index.ts`
- `@excalidraw/utils` → `packages/utils/src/index.ts`

## Details
For detailed architecture → see docs/technical/architecture.md
For domain glossary → see docs/product/domain-glossary.md
