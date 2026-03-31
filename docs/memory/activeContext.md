# Active Context — Excalidraw

> Cross-ref: [domain-glossary.md](../product/domain-glossary.md) · [architecture.md](../technical/architecture.md) · [code-vs-docs.md](../technical/code-vs-docs.md) · [dev-setup.md](../technical/dev-setup.md)

> Current development focus, recent changes, and immediate concerns.
> Last updated: 2026-03-29

---

## 1. Current Release

- **Version:** 0.18.0 (released 2025-03-11)
- All core packages (`common`, `math`, `element`, `excalidraw`) at **0.18.0**
- `@excalidraw/utils` at **0.1.2** (independent versioning)
- Next unreleased changes tracked in `CHANGELOG.md` under `## Unreleased`

---

## 2. Unreleased API Changes (Active Work)

Source: `packages/excalidraw/CHANGELOG.md` — `## Unreleased` section.

### Breaking

- **Renamed** `excalidrawAPI` prop → `onExcalidrawAPI`
  - Now called on mount (not constructor), and with `null` on unmount
  - Source: `packages/excalidraw/index.tsx`, `packages/excalidraw/components/App.tsx`

### New Features in Progress

- **`ExcalidrawAPI.isDestroyed` flag** — guards against stale API calls after unmount
  - Source: `App.tsx` — `componentWillUnmount` sets `isDestroyed: true`

- **Lifecycle props**: `onMount`, `onInitialize`, `onUnmount`
  - Events also accessible via `api.onEvent("editor:mount" | "editor:initialize" | "editor:unmount")`
  - Source: `App.tsx` — `componentDidMount`, first `componentDidUpdate`, `componentWillUnmount`

- **`<ExcalidrawAPIProvider>`** + hooks: `useExcalidrawAPI()`, `useAppStateValue()`, `useOnExcalidrawStateChange()`
  - Imperative: `api.onStateChange(prop, callback)`, `api.onEvent(name, callback)`
  - Source: `packages/excalidraw/index.tsx`

- **`onExport` prop** — async export handler with progress/abort support
  - Receives `AbortSignal`, can yield `{ type: "progress", message }` updates
  - Source: `packages/excalidraw/components/App.tsx`, `actionExport.tsx`

---

## 3. Active Feature Areas

### 3.1 Elbow Arrows & Flowcharts
- Elbow arrow routing engine in `packages/element/src/elbowArrow.ts`
- Flowchart creation logic in `packages/element/src/flowchart.ts`
- Tests: `packages/element/tests/elbowArrow.test.tsx`, `flowchart.test.tsx`

### 3.2 Image Cropping
- In-place crop editor for image elements
- Action: `actionCropEditor` in `packages/excalidraw/actions/actionCropEditor.tsx`
- Tests: `packages/element/tests/cropElement.test.tsx`

### 3.3 Element Linking & Deep Links
- Link elements to each other or external URLs
- Action: `actionElementLink` in `packages/excalidraw/actions/actionElementLink.ts`
- Dialog: `packages/excalidraw/components/ElementLinkDialog.tsx`
- URL fragment: `#elementLink=<id>` scrolls to element

### 3.4 Lasso Selection
- Alternative to box selection; toggled via `preferredSelectionTool`
- Trail rendering: `packages/excalidraw/lasso/`
- Action: `toggleLassoTool` in action types

### 3.5 Shape Conversion Popup
- Convert between rectangle ↔ diamond ↔ ellipse in-place
- Component: `packages/excalidraw/components/ConvertElementTypePopup.tsx`
- Action: `toggleShapeSwitch`, `togglePolygon`

### 3.6 Scene Search
- Find elements by text content on canvas
- Component: `packages/excalidraw/components/SearchMenu.tsx`
- Action: `actionToggleSearchMenu` (`Ctrl+F`)

---

## 4. Known Technical Debt

Source: `docs/technical/code-vs-docs.md` — verified against source.

### Architectural FIXMEs
| Issue | Location |
|-------|----------|
| `isSomeElementSelected` uses closure-based mutable cache | `packages/element/src/selection.ts:138` |
| `pick` duplicated in `colors.ts` due to circular dep | `packages/common/src/colors.ts:116` |
| `UIOptions` defaults merged in render body, breaking memo | `packages/excalidraw/index.tsx:105` |
| Theme change detection via `onChangeEmitter` workaround | `packages/excalidraw/wysiwyg/textWysiwyg.tsx:964` |
| Static canvas type leaks interactive-only state fields | `packages/excalidraw/types.ts:189–191` |

### Known Bugs in Tests
| Issue | Location |
|-------|----------|
| Z-index ordering wrong for frame children brought forward | `packages/element/tests/zindex.test.tsx:1322` |
| `getElementsCompletelyInFrame()` fails in tests, works in browser | `packages/element/tests/frame.test.tsx:336` |
| Labeled arrow version bump test is flaky | `packages/excalidraw/wysiwyg/textWysiwyg.test.tsx:335` |
| Frame children incorrectly selected after duplication | `packages/excalidraw/actions/actionDuplicateSelection.test.tsx:488` |
| Curves outside minMax points → wrong bounding boxes (6 skipped) | `packages/excalidraw/tests/flip.test.tsx:479–589` |

### Planned Migrations (TODOs)
- Remove `GlobalCoord`/`LocalCoord` types once codebase fully uses Point tuples (`packages/math/src/types.ts`)
- Formalize global↔local coordinate translation (`packages/math/src/point.ts:169`)
- Move `editingGroupId`, `selectedElementIds`, `frameToHighlight` to interactive canvas only (`packages/excalidraw/types.ts`)

---

## 5. Development Environment

- **Node:** ≥18.0.0 · **TypeScript:** 5.9.3 · **Yarn:** 1.22.22 (classic)
- **Dev server:** `yarn start` → Vite on port 3000
- **Tests:** `yarn test:app` → Vitest (watch mode) · 103 test files across all packages
- **Lint/Format:** `yarn fix` · Husky pre-commit hooks
- **Build order:** `common` → `math` → `element` → `excalidraw` → `excalidraw-app`

---

## 6. Collaboration Infrastructure

- WebSocket server URL configured via `VITE_APP_WS_SERVER_URL`
- Firebase config via `VITE_APP_FIREBASE_CONFIG`
- Sentry error tracking (disable with `VITE_APP_DISABLE_SENTRY=true`)
- Analytics via `VITE_APP_ENABLE_TRACKING`

---

## 7. Memory Bank Status

| File | Status | Description |
|------|--------|-------------|
| `projectbrief.md` | ✅ Complete | Project overview, features, audience |
| `productContext.md` | ✅ Complete | UX patterns, scenarios, interaction model |
| `systemPatterns.md` | ✅ Complete | Architecture, state, rendering, collab |
| `techContext.md` | ✅ Complete | Stack, tooling, commands, env vars |
| `decisionLog.md` | ✅ Complete | App.tsx decisions, lifecycle, API surface |
| `activeContext.md` | ✅ Current file | Current focus, debt, active work |
| `progress.md` | ✅ Complete | Feature & package progress tracking |

