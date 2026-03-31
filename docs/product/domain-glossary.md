# Excalidraw Domain Glossary

Project-specific terminology as defined and used in the codebase.

---

## ExcalidrawElement

Union type of all drawing primitives on the canvas. Every element is JSON-serializable, contains no computed data, and is designed to be shared between peers.

Discriminated by the `type` field into: `rectangle`, `diamond`, `ellipse`, `text`, `line`, `arrow`, `freedraw`, `image`, `frame`, `magicframe`, `iframe`, `embeddable`, `selection`.

All variants extend `_ExcalidrawElementBase` which carries `id`, `x`, `y`, `width`, `height`, `angle`, style props, `version`, `versionNonce`, `index` (fractional), `isDeleted`, `groupIds`, `frameId`, `boundElements`, `updated`, `link`, `locked`.

**Key files:**
- `packages/element/src/types.ts` — type definitions (`_ExcalidrawElementBase`, all variant types, `ExcalidrawElement` union)
- `packages/element/src/mutateElement.ts` — `mutateElement()`, `newElementWith()`
- `packages/element/src/typeChecks.ts` — runtime type guards (`isTextElement`, `isArrowElement`, etc.)

---

## Scene

Class that owns the canonical array of all `OrderedExcalidrawElement`s in the editor. Provides derived views: non-deleted elements, element maps by id, frames, selected elements cache. Maintains a `sceneNonce` (random integer regenerated on every mutation) used as a render cache-invalidation key.

Primary mutation method is `replaceAllElements()`, which performs an immutable swap and syncs fractional indices via `syncInvalidIndices` / `syncMovedIndices`.

**Key files:**
- `packages/element/src/Scene.ts` — `Scene` class definition
- `packages/excalidraw/components/App.tsx` — `this.scene` instance, consumed throughout render and event handling

---

## AppState

Interface with 100+ fields describing the entire editor UI state: active tool, viewport (scroll/zoom), selection, editing state, current item defaults, collaborator map, UI panels, export settings, theme, grid, snapping, and more.

Defaults provided by `getDefaultAppState()`. Stored as React component state in the `App` class (`this.state`).

`ObservedAppState` is a narrowed subset tracked by `Store` for undo/redo diffing — only fields affecting the canvas or user-visible state.

**Key files:**
- `packages/excalidraw/types.ts` — `AppState` interface (line ~272), `ObservedAppState`, `StaticCanvasAppState`, `InteractiveCanvasAppState`
- `packages/excalidraw/appState.ts` — `getDefaultAppState()`
- `packages/excalidraw/components/App.tsx` — `this.state: AppState`

---

## Action

Interface for a discrete user-triggered command (e.g. copy, paste, undo, changeStrokeColor). Each action has a `name` (from `ActionName` union of ~60 names), a `perform()` function that receives `(elements, appState, formData, app)` and returns an `ActionResult`, an optional `keyTest()` for keyboard shortcuts, an optional `PanelComponent` for toolbar UI, and optional `predicate()` / `trackEvent`.

`ActionResult` is `{ elements?, appState?, files?, captureUpdate } | false`. The `captureUpdate` field (`CaptureUpdateAction.IMMEDIATELY | EVENTUALLY | NEVER`) controls undo/redo behavior.

**Key files:**
- `packages/excalidraw/actions/types.ts` — `Action` interface, `ActionResult`, `ActionName` union, `ActionSource`
- `packages/excalidraw/actions/manager.tsx` — `ActionManager` class
- `packages/excalidraw/actions/` — individual action implementations (e.g. `actionCopy.ts`, `actionHistory.tsx`)

---

## ActionManager

Registry-based command dispatcher. Holds a `Record<ActionName, Action>` map. Actions are registered via `registerAction()`. Dispatches through `handleKeyDown()` (keyboard shortcut matching with priority) or `executeAction()` (programmatic). The `updater` callback (`syncActionResult` in App) applies the `ActionResult` to React state and the Scene.

Also provides `renderAction()` which renders an action's `PanelComponent` for the toolbar/properties panel.

**Key files:**
- `packages/excalidraw/actions/manager.tsx` — `ActionManager` class
- `packages/excalidraw/components/App.tsx` — `this.actionManager` instance, `syncActionResult` method

---

## Store

Class that captures observed changes to elements and appState, compares `StoreSnapshot`s, produces `StoreChange` + `StoreDelta`, and emits `StoreIncrement` events. Orchestrates macro/micro action scheduling.

Emits two types of increments:
- `DurableIncrement` — contains a `StoreDelta`, pushed to `History` for undo/redo
- `EphemeralIncrement` — change-only (no delta), for external subscribers (e.g. collaboration sync)

**Key files:**
- `packages/element/src/store.ts` — `Store`, `StoreSnapshot`, `StoreChange`, `StoreDelta`, `StoreIncrement`, `DurableIncrement`, `EphemeralIncrement`, `CaptureUpdateAction`

---

## StoreSnapshot

Immutable snapshot of `SceneElementsMap` + `ObservedAppState` at a point in time. Compared against the previous snapshot to detect changes via element version hashing and appState hashing. Produces `StoreChange` (what changed) and `StoreDelta` (reversible diff for undo).

**Key files:**
- `packages/element/src/store.ts` — `StoreSnapshot` class (line ~643)

---

## StoreDelta

Reversible diff between two `StoreSnapshot`s. Contains `ElementsDelta` and `AppStateDelta`. Can be applied forward or backward to replay/undo changes. Extended by `HistoryDelta` for the undo/redo stack.

**Key files:**
- `packages/element/src/store.ts` — `StoreDelta` class (line ~497)
- `packages/element/src/delta.ts` — `ElementsDelta`, `AppStateDelta`, `Delta`

---

## CaptureUpdateAction

Enum-like const object controlling how a state change is captured for undo/redo:
- `IMMEDIATELY` — delta captured and pushed to undo stack right away (most local updates)
- `EVENTUALLY` — deferred, merged into the next `IMMEDIATELY` commit (async multi-step operations)
- `NEVER` — no undo entry (remote updates, initialization)

Returned in every `ActionResult` as `captureUpdate`.

**Key files:**
- `packages/element/src/store.ts` — `CaptureUpdateAction` definition

---

## History

Manages undo/redo stacks of `HistoryDelta` entries. Subscribes to `Store.onDurableIncrementEmitter` to receive deltas. `HistoryDelta.applyTo()` replays a delta onto elements and appState, excluding `version`/`versionNonce` so each undo/redo generates fresh version values for collaboration.

**Key files:**
- `packages/excalidraw/history.ts` — `History` class, `HistoryDelta` class

---

## Tool / ToolType

String literal union representing the currently active drawing/interaction tool. Values: `selection`, `lasso`, `rectangle`, `diamond`, `ellipse`, `arrow`, `line`, `freedraw`, `text`, `image`, `eraser`, `hand`, `frame`, `magicframe`, `embeddable`, `laser`, or `custom`.

Stored in `appState.activeTool` along with metadata (`locked`, `lastActiveTool`, `fromSelection`).

**Key files:**
- `packages/excalidraw/types.ts` — `ToolType`, `ActiveTool`, `ElementOrToolType`
- `packages/excalidraw/components/App.tsx` — tool switching logic, pointer event handling per active tool

---

## Library / LibraryItem

Reusable collections of elements that users can save and insert. A `LibraryItem` is `{ id, status, elements, created, name? }` where `status` is `"published" | "unpublished"` and `elements` is a `NonDeleted<ExcalidrawElement>[]`.

The `Library` class manages persistence via `LibraryPersistenceAdapter`, uses a Jotai atom (`libraryItemsAtom`) for reactive state, and emits `LibraryUpdate` events (added/deleted/updated items).

**Key files:**
- `packages/excalidraw/types.ts` — `LibraryItem`, `LibraryItems`, `LibraryItemsSource`
- `packages/excalidraw/data/library.ts` — `Library` class, `LibraryPersistenceAdapter` interface, `mergeLibraryItems()`

---

## Collaboration / Collab

Real-time multi-user editing. The `Collab` class (a `PureComponent`) manages the collaboration lifecycle: connecting to a room, broadcasting scene updates, receiving remote changes, and tracking collaborator presence (cursors, idle state).

Uses `Portal` for encrypted Socket.IO transport. Incoming remote elements are merged via `reconcileElements()`.

**Key files:**
- `excalidraw-app/collab/Collab.tsx` — `Collab` class, collaboration lifecycle
- `excalidraw-app/collab/Portal.tsx` — `Portal` class, Socket.IO transport, encryption
- `packages/excalidraw/data/reconcile.ts` — `reconcileElements()`, `shouldDiscardRemoteElement()`

---

## Portal

Transport layer for collaboration. Wraps a Socket.IO socket connection, manages `roomId` and `roomKey`, tracks `broadcastedElementVersions` to avoid redundant sends. Encrypts data before emitting and decrypts on receive.

**Key files:**
- `excalidraw-app/collab/Portal.tsx` — `Portal` class

---

## Collaborator

Read-only type describing a remote user in a collaboration session: `pointer` position, `button` state, `selectedElementIds`, `username`, `userState` (active/idle/away), `color`, `avatarUrl`, `socketId`.

Stored in `appState.collaborators` as `Map<SocketId, Collaborator>`.

**Key files:**
- `packages/excalidraw/types.ts` — `Collaborator` type, `CollaboratorPointer`, `SocketId`

---

## reconcileElements

Function that merges local and remote element arrays during collaboration. Conflict resolution rules:
1. Higher `version` wins
2. Equal versions → lower `versionNonce` wins (deterministic tiebreak)
3. Local element being edited (`editingTextElement`, `resizingElement`, `newElement`) is never overwritten

After merge, elements are reordered by fractional index and invalid indices are synced.

**Key files:**
- `packages/excalidraw/data/reconcile.ts` — `reconcileElements()`, `shouldDiscardRemoteElement()`

---

## FractionalIndex

Branded string type (`string & { _brand: "franctionalIndex" }`). Defines element ordering using the `fractional-indexing` library. Enables inserting elements between existing ones without reindexing the entire array — critical for multiplayer reconciliation and undo/redo.

Synced by `syncMovedIndices()` and `syncInvalidIndices()` on every scene update.

**Key files:**
- `packages/element/src/types.ts` — `FractionalIndex` type
- `packages/element/src/fractionalIndex.ts` — `syncInvalidIndices()`, `syncMovedIndices()`, `orderByFractionalIndex()`

---

## Renderer

Class that prepares elements for canvas rendering. `getRenderableElements()` is memoized — filters non-deleted elements, excludes the currently-edited text element, then filters to elements within the viewport via `isElementInViewport()`. Returns `RenderableElementsMap` + `visibleElements` array.

**Key files:**
- `packages/excalidraw/scene/Renderer.ts` — `Renderer` class

---

## ShapeCache

Static class with a `WeakMap<ExcalidrawElement, { shape, theme }>` cache. Generates rough.js `Drawable` shapes via `RoughGenerator` for geometric elements, or `perfect-freehand` strokes for freedraw. Shapes are cached per element object reference and invalidated on mutation (via `ShapeCache.delete()`).

**Key files:**
- `packages/element/src/shape.ts` — `ShapeCache` class, `generateElementShape()`

---

## StaticCanvas / InteractiveCanvas

Two overlapping `<canvas>` elements in the editor:
- **StaticCanvas** (bottom) — renders all drawing elements, grid, background. Uses `renderStaticScene()`.
- **InteractiveCanvas** (top, transparent) — renders selection outlines, transform handles, collaborator cursors, snap lines. Uses `renderInteractiveScene()`. Handles all pointer events.

This separation avoids re-rendering the static scene when only selection/interaction state changes.

**Key files:**
- `packages/excalidraw/components/canvases/StaticCanvas.tsx`
- `packages/excalidraw/components/canvases/InteractiveCanvas.tsx`
- `packages/excalidraw/renderer/staticScene.ts` — `renderStaticScene()`
- `packages/excalidraw/renderer/interactiveScene.ts` — `renderInteractiveScene()`

---

## App

The central class component (`React.Component<AppProps, AppState>`, ~12 800 lines). Owns `scene`, `store`, `renderer`, canvas refs, `actionManager`, `history`, image cache, and all pointer/keyboard event handlers. Intentionally a class (not function) component to support the imperative API surface and mutable instance state.

**Key files:**
- `packages/excalidraw/components/App.tsx` — `App` class (line ~617)

---

## ExcalidrawImperativeAPI

Public API interface exposed to host applications via the `onExcalidrawAPI` prop. Provides methods: `updateScene`, `getSceneElements`, `getAppState`, `scrollToContent`, `updateLibrary`, `setActiveTool`, `onChange`, `onIncrement`, `resetScene`, `addFiles`, `history.clear`, and more.

**Key files:**
- `packages/excalidraw/types.ts` — `ExcalidrawImperativeAPI` interface (line ~917)
- `packages/excalidraw/index.tsx` — `Excalidraw` component wiring

---

## Emitter

Generic pub/sub class used throughout the codebase. `on()` subscribes, returns an unsubscribe callback. `trigger()` invokes all subscribers synchronously. Used by `Store` (increment emitters), `Library` (update emitter), `App` (pointer events), and more.

**Key files:**
- `packages/common/src/emitter.ts` — `Emitter<T>` class

---

## BinaryFileData

Type for image/file attachments: `{ id: FileId, mimeType, dataURL: DataURL, created, lastRetrieved?, version? }`. Files are stored separately from elements — elements reference them via `fileId`. Local cache uses IndexedDB (`idb-keyval`), remote storage uses Firebase Storage.

**Key files:**
- `packages/excalidraw/types.ts` — `BinaryFileData`, `BinaryFiles`, `DataURL`, `FileId`
- `excalidraw-app/data/LocalData.ts` — local persistence
- `excalidraw-app/data/FileManager.ts` — upload/download management

---

## Binding / FixedPointBinding

Mechanism connecting arrows to bindable elements (rectangles, diamonds, ellipses, text, images, frames). A `FixedPointBinding` stores `elementId`, `fixedPoint` (normalized [0–1] coordinates on the bound element), and `mode` (`"inside" | "orbit" | "skip"`).

Arrows have `startBinding` and `endBinding` fields. When a bound element moves, the arrow endpoint recalculates its position from the `fixedPoint` ratio.

**Key files:**
- `packages/element/src/types.ts` — `FixedPointBinding`, `BindMode`, `BoundElement`
- `packages/element/src/binding.ts` — binding logic (~2 900 lines)

---

## Frame

Container element (`type: "frame"` or `type: "magicframe"`) that groups and clips child elements. Elements inside a frame have `frameId` set to the frame's `id`. Frame rendering clips children to the frame boundary. `magicframe` is a special variant for AI code generation.

**Key files:**
- `packages/element/src/types.ts` — `ExcalidrawFrameElement`, `ExcalidrawMagicFrameElement`, `ExcalidrawFrameLikeElement`
- `packages/element/src/frame.ts` — frame containment logic, clipping

---

## LinearElementEditor

Editor state class for interactively editing linear elements (lines and arrows). Tracks `selectedPointsIndices`, `isDragging`, `hoverPointIndex`, `segmentMidPointHoveredCoords`, and binding focus points. Created when a user enters point-editing mode on a linear element.

Stored in `appState.selectedLinearElement`.

**Key files:**
- `packages/element/src/linearElementEditor.ts` — `LinearElementEditor` class (~2 500 lines)

---

## Zoom / NormalizedZoomValue

`Zoom` is `{ value: NormalizedZoomValue }` where `NormalizedZoomValue` is a branded number. Represents the current viewport zoom level. Stored in `appState.zoom`. Used throughout coordinate transforms between scene space and viewport space.

**Key files:**
- `packages/excalidraw/types.ts` — `Zoom`, `NormalizedZoomValue`

---

## GlobalPoint / LocalPoint

Branded tuple types (`[x: number, y: number]`) from `@excalidraw/math`. `GlobalPoint` represents positions in world/canvas/scene space. `LocalPoint` represents positions in element-local space (e.g. arrow points relative to element origin). The branding prevents accidental mixing at compile time.

**Key files:**
- `packages/math/src/types.ts` — `GlobalPoint`, `LocalPoint`

---

## SnapLine

Visual guide rendered on the interactive canvas when object snapping is active. Three variants: `PointSnapLine` (alignment guides between element edges/centers), `GapSnapLine` (equal spacing guides), `PointerSnapLine` (guides relative to pointer). Stored in `appState.snapLines`.

**Key files:**
- `packages/excalidraw/snapping.ts` — `SnapLine`, `PointSnapLine`, `GapSnapLine`, `PointerSnapLine`, snapping logic (~1 400 lines)

---

## SceneData

Type used to batch-update the editor via `ExcalidrawImperativeAPI.updateScene()`. Contains optional `elements`, `appState`, `collaborators`, and `captureUpdate`. This is the primary API for host apps to programmatically modify the scene.

**Key files:**
- `packages/excalidraw/types.ts` — `SceneData`

---

## mutateElement / newElementWith

Two mutation strategies for elements:
- `mutateElement(element, elementsMap, updates)` — **in-place mutation**. Increments `version`, regenerates `versionNonce`, updates `updated` timestamp, invalidates `ShapeCache`. Does NOT trigger React re-render — use `scene.mutateElement()` for that.
- `newElementWith(element, updates)` — **immutable clone**. Returns a new object with spread updates and fresh version. Used when immutability is required (e.g. in Store diffing).

**Key files:**
- `packages/element/src/mutateElement.ts` — `mutateElement()`, `newElementWith()`

