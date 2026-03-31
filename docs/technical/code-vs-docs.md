# Code vs Docs — Undocumented Behavior & Technical Debt

This document catalogs behavior present in the codebase but absent from `/docs`.
Each entry is verified against source code with file references.

---

## 1. Implicit State Machines

### 1.1 `activeTool` State Machine

The `activeTool` field in `AppState` follows complex, undocumented transition rules
involving `lastActiveTool`, `lastActiveToolBeforeEraser`, and `locked` mode.

- **Eraser button temporarily overrides active tool**, restoring the previous tool on pointer-up:
  - `packages/excalidraw/components/App.tsx:7627–7660` — On `POINTER_BUTTON.ERASER`, saves current tool to `lastActiveToolBeforeEraser`, switches to eraser, then restores on `POINTER_UP`.
- **Finalize action** resets tool differently depending on `locked` and `freedraw` state:
  - `packages/excalidraw/actions/actionFinalize.tsx:254–330` — If eraser was active, restores `lastActiveTool`; otherwise falls back to preferred selection tool. Locked tools are preserved.
- **Restore silently forces `cursorButton: "up"` and resets `activeTool`** to `"selection"` if the persisted tool type is not in `AllowedExcalidrawActiveTools`:
  - `packages/excalidraw/data/restore.ts:939–956`

### 1.2 `cursorButton` Implicit State

`cursorButton` (`"up"` | `"down"`) acts as an implicit two-state machine that is silently forced to `"up"` during restore, regardless of the supplied value:

- `packages/excalidraw/data/restore.ts:941` — `cursorButton: localAppState?.cursorButton || "up"`
- Tests explicitly verify this coercion: `packages/excalidraw/tests/data/restore.test.ts:580–608`

### 1.3 Collaboration Room Initialization State Machine

Room initialization in `Collab.tsx` has an undocumented multi-phase state machine:
1. Socket opens with websocket+polling transports
2. `connect_error` triggers `fallbackInitializationHandler`
3. Existing room → `resetScene()`; new room → mutate image elements to `"pending"`
4. A timeout fallback fires `initializeRoom` if no `init-room` event arrives

- `excalidraw-app/collab/Collab.tsx:490–560`

---

## 2. Non-obvious Side Effects

### 2.1 `isSomeElementSelected` — Closure-based Mutable Cache

`isSomeElementSelected` is a module-level IIFE with hidden mutable closure state (`lastElements`, `lastSelectedElementIds`, `isSelected`). It caches results between calls and requires manual `clearCache()`.

- `packages/element/src/selection.ts:138–172`
- Comment: `// FIXME move this into the editor instance to keep utility methods stateless`

### 2.2 `ShapeCache` — Static Class with Global Mutable State

`ShapeCache` uses a static `WeakMap` cache and a static `RoughGenerator` instance. Deleting a shape also clears the `elementWithCanvasCache` as an undocumented side effect.

- `packages/element/src/shape.ts:81–112`
- `packages/element/src/shape.ts:105–107` — `delete` method also clears `elementWithCanvasCache`
- `packages/element/src/shape.ts:140` — `generateElementShape` also deletes `elementWithCanvasCache`

### 2.3 `SnapCache` — Static Singleton with Implicit Lifecycle

`SnapCache` holds mutable static state (`referenceSnapPoints`, `visibleGaps`) populated lazily via `maybeCacheReferenceSnapPoints` during pointer interactions and cleared via `destroy()`.

- `packages/excalidraw/snapping.ts:122–149`
- Populated in `packages/excalidraw/components/App.tsx:9382–9400`

### 2.4 `FirebaseSceneVersionCache` — Hidden Version Tracking

`FirebaseSceneVersionCache` is a static `WeakMap<Socket, number>` that silently caches scene versions. `isSavedToFirebase` returns `true` when no room exists to avoid blocking unload — a non-obvious fallback.

- `excalidraw-app/data/firebase.ts:118–142`

### 2.5 `restoreAppState` Silent Coercions

`restoreAppState` silently overrides several fields after the generic merge loop:
- Forces `cursorButton` to `"up"`
- Resets `penDetected` conditionally
- Forces `activeTool.lastActiveTool` to `null`
- Migrates `zoom` from number to `{ value }` object format

- `packages/excalidraw/data/restore.ts:939–965`

### 2.6 Text WYSIWYG Theme Side Effect

The text editor subscribes to `onChangeEmitter` to detect theme changes (comparing to a `LAST_THEME` variable) and re-applies styles. This is a workaround for the Store not emitting `appState.theme` updates.

- `packages/excalidraw/wysiwyg/textWysiwyg.tsx:964–969`
- Comment: `// FIXME after we start emitting updates from Store for appState.theme`

### 2.7 `pointTranslate` — Misleading API for Element Coordinates

`pointTranslate` has a `WARNING` comment noting it does NOT handle element rotation. Most callers use it for global↔local coordinate translation but must handle rotation separately.

- `packages/math/src/point.ts:157–175`
- Comment: `// TODO 99% of use is translating between global and local coords, which need to be formalized`

---

## 3. Initialization Order Dependencies

### 3.1 `pick` in `colors.ts` — Circular Dependency Workaround

A `pick` utility is duplicated in `colors.ts` instead of being imported from `utils.ts` due to a circular dependency between the packages.

- `packages/common/src/colors.ts:116` — `// FIXME can't put to utils.ts rn because of circular dependency`

### 3.2 `UIOptions` Normalization in `<Excalidraw>`

`UIOptions` defaults are merged inside the render body of the `<Excalidraw>` component, which causes the memo resolver to compare different object references each render.

- `packages/excalidraw/index.tsx:105–116`
- Comment: `// FIXME normalize/set defaults in parent component so that the memo resolver compares the same values`

### 3.3 Static Canvas AppState Props Leak Interactive State

Three `AppState` fields in `_CommonCanvasAppState` are marked as belonging to interactive canvas but currently live in the shared type used by static canvas too:

- `packages/excalidraw/types.ts:189–191`
  - `editingGroupId` — `// TODO: move to interactive canvas if possible`
  - `selectedElementIds` — `// TODO: move to interactive canvas if possible`
  - `frameToHighlight` — `// TODO: move to interactive canvas if possible`

### 3.4 WASM Binary Must Be Embedded at Build Time

The woff2 loader requires the WASM binary to be available inline (no URL-based fetching), creating an implicit build-time dependency.

- `packages/excalidraw/subset/woff2/woff2-loader.ts:30` — `// TODO: consider adding support for fetching the wasm from an URL (external CDN, data URL, etc.)`

---

## 4. HACK / FIXME / TODO Markers

### 4.1 HACK

| Location | Description |
|---|---|
| `packages/excalidraw/components/App.tsx:7126` | Disables transform handles for linear elements on mobile — no proper solution exists yet |

### 4.2 FIXME — Architectural Issues

| Location | Description |
|---|---|
| `packages/element/src/selection.ts:138` | `isSomeElementSelected` uses closure state; should move to editor instance |
| `packages/common/src/colors.ts:116` | `pick` duplicated due to circular dependency |
| `packages/excalidraw/index.tsx:105` | `UIOptions` defaults merged in render body, breaking memoization |
| `packages/excalidraw/actions/actionCanvas.tsx:73` | `PanelComponent` for canvas background should move to `DefaultItems.tsx` |
| `packages/excalidraw/components/App.tsx:8758` | Bare `// FIXME` on text-bindable container lookup |
| `packages/excalidraw/wysiwyg/textWysiwyg.tsx:964` | Theme change detection via `onChangeEmitter` is a workaround |
| `packages/excalidraw/components/EyeDropper.tsx:105` | Color preview offset doesn't swap when outside viewport |
| `packages/excalidraw/components/LayerUI.tsx:111–113` | Export/SaveAsImage visibility should be tested inside menu items themselves |
| `packages/element/src/textMeasurements.ts:31` | `getApproxMinLineWidth` misnamed, should be `getApproxMinContainerWidth` |
| `packages/element/src/textMeasurements.ts:98` | `getApproxMinLineHeight` misnamed, should be `getApproxMinContainerHeight` |
| `packages/utils/tests/export.test.ts:94` | `exportToSvg` no longer filters deleted elements — skipped test |

### 4.3 FIXME — Known Bugs

| Location | Description |
|---|---|
| `packages/element/tests/zindex.test.tsx:1322–1325` | Z-index ordering incorrect for frame children brought forward |
| `packages/element/tests/frame.test.tsx:336` | `getElementsCompletelyInFrame()` fails in tests but works in browser |
| `packages/excalidraw/wysiwyg/textWysiwyg.test.tsx:335` | Labeled arrow version bump test is flaky — "No one knows why" |
| `packages/excalidraw/actions/actionDuplicateSelection.test.tsx:488–494` | Frame children incorrectly selected after duplication (`selectGroupsForSelectedElements` bug) |

### 4.4 TODO — Planned Migrations

| Location | Description |
|---|---|
| `packages/math/src/types.ts:42,60` | Remove `GlobalCoord`/`LocalCoord` once codebase migrates to Point tuples |
| `packages/math/src/point.ts:26,30` | Remove `pointFrom` overloads after tuple migration |
| `packages/math/src/point.ts:169` | Formalize global↔local coordinate translation |
| `packages/excalidraw/types.ts:189–191` | Move `editingGroupId`, `selectedElementIds`, `frameToHighlight` to interactive canvas |
| `packages/excalidraw/types.ts:631` | Design better API before v0.18.0 |
| `packages/excalidraw/snapping.ts:44` | `VISIBLE_GAPS_LIMIT_PER_AXIS` — increase/remove after optimization |
| `packages/utils/src/shape.ts:361` | Replace with final rounded rectangle code |

### 4.5 TODO — Known Bugs / Debt in Tests

| Location | Description |
|---|---|
| `packages/excalidraw/tests/selection.test.tsx:250,273` | Memory leak if `pointerUp` is not triggered |
| `packages/excalidraw/tests/flip.test.tsx:479–589` | Curves outside minMax points produce wrong bounding boxes (6 skipped tests) |
| `packages/excalidraw/tests/history.test.tsx:2369` | #7348 — `groupIds` order not guaranteed after undo/redo postprocessing |
| `packages/excalidraw/tests/history.test.tsx:3596,3699` | #7348 — Rebinding in history to prevent data-integrity issues; may remove later |
| `packages/excalidraw/tests/history.test.tsx:4372` | #7348 — Empty undo/redo when container text dimensions change |

