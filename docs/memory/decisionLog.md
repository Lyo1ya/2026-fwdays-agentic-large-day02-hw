# App.tsx Decision Log

> Cross-ref: [domain-glossary.md](../product/domain-glossary.md) · [architecture.md](../technical/architecture.md) · [code-vs-docs.md](../technical/code-vs-docs.md) · [dev-setup.md](../technical/dev-setup.md)

> File: `packages/excalidraw/components/App.tsx` (~12 800 lines, class component)

---

## 1. Architecture Decisions

### 1.1 Class component over functional

`App` extends `React.Component<AppProps, AppState>`. Key reasons:

- Stable `this` reference required by `ActionManager`, `Store`, `History`, event handlers, and the imperative API object – all of which capture `this` at construction time.
- `withBatchedUpdates` / `withBatchedUpdatesThrottled` wrappers rely on class-instance methods.
- Instance fields for high-frequency mutable state (pointer coords, gesture map) avoid React scheduler overhead.

### 1.2 Elements live outside React state

Canvas elements are owned by the `Scene` class (`this.scene`), **not** `AppState`. This avoids VDOM diffing for a potentially large element array on every pointer-move. React only re-renders when `Scene` triggers `triggerRender` → `setState({})` (or `scene.triggerUpdate()` for forced canvas repaints).

### 1.3 Dual rendering pipeline

| Canvas | Purpose |
|---|---|
| `StaticCanvas` | Non-interactive background (shapes, fills) |
| `InteractiveCanvas` | Handles pointer events, selection, resize handles |
| `NewElementCanvas` | In-progress newly drawn element |

Rendering is driven by `Renderer.getRenderableElements()` inside `render()`, which reads `scene.getSceneNonce()` as a cache key.

### 1.4 Context tree

Eight+ React contexts are provided in `render()` for granular downstream subscriptions, avoiding prop-drilling and minimising re-render scope:

| Context | Value |
|---|---|
| `AppContext` | `this` (full class instance) |
| `AppPropsContext` | `this.props` |
| `EditorInterfaceContext` | `this.editorInterface` (formFactor, desktopUIMode) |
| `ExcalidrawContainerContext` | container ref + id |
| `ExcalidrawElementsContext` | `scene.getNonDeletedElements()` |
| `ExcalidrawAppStateContext` | `this.state` |
| `ExcalidrawSetAppStateContext` | `this.setAppState` |
| `ExcalidrawActionManagerContext` | `this.actionManager` |
| `ExcalidrawAPIContext` | `this.api` |

---

## 2. State Management

### 2.1 AppState (React class state)

`AppState` (~60 fields) holds all **UI and view state** that drives React renders. Updated via `this.setState()`. Key categories:

| Category | Fields |
|---|---|
| Viewport | `scrollX`, `scrollY`, `zoom`, `width`, `height`, `offsetTop`, `offsetLeft` |
| Tool | `activeTool`, `preferredSelectionTool`, `penMode`, `isBindingEnabled`, `bindMode` |
| Selection | `selectedElementIds`, `previousSelectedElementIds`, `selectedGroupIds`, `editingGroupId`, `selectedLinearElement` |
| Active UI | `contextMenu`, `openDialog`, `openMenu`, `openPopup`, `openSidebar`, `toast`, `errorMessage` |
| In-progress elements | `newElement`, `resizingElement`, `multiElement`, `selectionElement` |
| Collaboration | `collaborators`, `userToFollow`, `followedBy` |
| Flags | `isLoading`, `viewModeEnabled`, `zenModeEnabled`, `gridModeEnabled`, `objectsSnapModeEnabled` |
| Appearance | `theme`, `viewBackgroundColor`, `currentItem*` style props |
| Export | `exportBackground`, `exportWithDarkMode`, `exportEmbedScene`, `fileHandle` |

### 2.2 Scene (element store)

`this.scene: Scene` owns the canonical element array. Updates via:

- `scene.replaceAllElements(elements)` – full replacement (undo/redo, paste, init)
- `scene.insertElements(elements)` – append
- `scene.mutateElement(element, updates)` – patch single element
- `scene.triggerUpdate()` – invalidate without element change (forces canvas repaint)

### 2.3 Store + History (undo/redo)

```
pointer action → scene/state mutation
    → componentDidUpdate → store.commit(elementsMap, appState)
        → onDurableIncrementEmitter → history.record(delta)
```

- `Store` snapshots observed state, computes diffs (`StoreDelta`), emits increments.
- `History` records durable increments into undo/redo stacks.
- `CaptureUpdateAction.IMMEDIATELY` | `EVENTUALLY` | `NEVER` controls whether an update is captured.
- `resetStore()` / `resetHistory()` called on scene reset and initialisation.

### 2.4 Jotai atoms (UI-only micro-state)

Bypasses React class state for transient UI state not needing `AppState` serialisation:

| Atom | Purpose |
|---|---|
| `activeEyeDropperAtom` | Eye-dropper tool activation |
| `convertElementTypePopupAtom` | Shape-switch popup state |
| `isSidebarDockedAtom` | Sidebar dock preference |
| `searchItemInFocusAtom` | Search panel focused item |
| `activeConfirmDialogAtom` | Confirm dialog type |

Updated via `this.updateEditorAtom(atom, value)` which calls `editorJotaiStore.set(atom)` then `triggerRender()`.

### 2.5 Module-level mutable variables

High-frequency or singleton state stored as module-level `let` to skip React scheduler:

```ts
let isHoldingSpace, isPanning, isDraggingScrollBar
let didTapTwice, tappedTwiceTimer, firstTapPosition
let IS_PLAIN_PASTE, IS_PLAIN_PASTE_TIMER, PLAIN_PASTE_TOAST_SHOWN
let currentScrollBars, touchTimeout, invalidateContextMenu
const gesture: Gesture          // pointer map + pinch state
const YOUTUBE_VIDEO_STATES      // youtube player state per element id
```

### 2.6 Instance properties (non-React state)

Mutated directly, trigger manual re-renders where necessary:

| Property | Type | Notes |
|---|---|---|
| `files` | `BinaryFiles` | Image/asset binary data |
| `imageCache` | `Map<FileId, …>` | Decoded image bitmaps |
| `iFrameRefs` | `Map<id, HTMLIFrameElement>` | DOM refs to embeds |
| `embedsValidationStatus` | `Map<id, boolean>` | Embed URL validation result |
| `initializedEmbeds` | `Set<id>` | Embeds inserted to DOM (perf) |
| `visibleElements` | `readonly ExcalidrawElement[]` | Computed each render |
| `magicGenerations` | `Map<id, MagicGenerationData>` | AI generation results |
| `editorInterface` | `EditorInterface` | formFactor, desktopUIMode, isTouchScreen |

### 2.7 State update entry points

| Method | Batched | Use case |
|---|---|---|
| `syncActionResult()` | ✅ `withBatchedUpdates` | All `ActionManager.executeAction()` results |
| `updateScene()` | ✅ `withBatchedUpdates` | Imperative API (host apps, collab) |
| `setAppState()` | — | Delegating `setState` for child components |
| `updateEditorAtom()` | — (manual `triggerRender`) | Jotai atoms |
| `triggerRender(force?)` | — | Force repaint or empty setState re-render |
| `applyDeltas()` | — | Collab delta application |

---

## 3. Lifecycle

### 3.1 `constructor`

1. Initialise `AppState` from `props` + `getDefaultAppState()`
2. `refreshEditorInterface()` → compute formFactor/desktopUIMode
3. Create: `Library`, `ActionManager`, `Scene`, `canvas`, `rough.canvas`, `Renderer`, `Store`, `History`, `Fonts`
4. Register all actions + undo/redo actions
5. Pre-create `this.api` (needed if internal APIs call it before mount; recreated in `componentDidMount` to handle StrictMode double-invoke)

### 3.2 `componentDidMount`

```
1. Recreate this.api (StrictMode-safe)
2. Expose window.h debug handles (test/dev only)
3. Subscribe: store.onDurableIncrementEmitter → history.record()
4. Subscribe: store.onStoreIncrementEmitter → props.onIncrement (if set)
5. Subscribe: scene.onUpdate → triggerRender
6. addEventListeners()
7. autoFocus container (if props.autoFocus)
8. ResizeObserver on container → refreshEditorInterface + updateDOMRect
9. Handle Web Share Target (launchQueue / web-share-target cache)
10. updateDOMRect(initializeScene) → async scene initialisation
11. Brave browser text-measure error check
12. Emit "editor:mount" lifecycle event
13. Call props.onMount, props.onExcalidrawAPI(api)
```

### 3.3 `initializeScene` (async, called from `componentDidMount`)

```
1. Register PWA launchQueue handler
2. setState({ isLoading: true })
3. Await props.initialData (function or promise)
4. Load library items from initialData
5. restoreElements() + restoreAppState()
6. Reconcile activeTool, preferredSelectionTool, openSidebar
7. resetStore() + resetHistory()
8. syncActionResult({ elements, appState, files, captureUpdate: NEVER })
9. clearImageShapeCache()
10. fonts.loadSceneFonts()
11. scrollToContent if URL has elementLink
```

### 3.4 `componentDidUpdate`

Runs after every render. Key responsibilities:

1. **First non-loading render**: emit `editor:initialize`, call `props.onInitialize`
2. `appStateObserver.flush(prevState)` – notify targeted state change subscribers
3. `updateEmbeddables()` – validate embed URLs, GC iframe refs
4. Sync `exportWithDarkMode` with current theme
5. Welcome screen auto-show when elements cleared
6. Collaborator leave detection → `maybeUnfollowRemoteUser()`
7. Fire `props.onScrollChange` / `onScrollChangeEmitter` on viewport change
8. Fire `onUserFollowEmitter` on follow/unfollow
9. Tool state reconciliation: auto-switch eraser→selection when elements selected; update eraser cursor on theme change
10. Hyperlink popup cleanup on tool change
11. `updateLanguage()` on `props.langCode` change
12. Re-`addEventListeners()` on viewMode toggle
13. Theme class toggle on container DOM element
14. Auto-finalize linear element editing if deselected
15. Guard deleted `editingTextElement`
16. **`store.commit(elementsMap, state)`** – always called; Store decides if snapshot changed
17. **`props.onChange(elements, state, files)`** + `onChangeEmitter.trigger()` – if not loading

### 3.5 `componentWillUnmount`

```
1. Invalidate API (isDestroyed = true, stub out get*/subscribe methods)
2. Emit "editor:unmount"
3. Call props.onUnmount, props.onExcalidrawAPI(null)
4. Clear launchQueue handler
5. Destroy: renderer, scene (and re-create empty), fonts, image cache
6. Disconnect resizeObserver
7. this.unmounted = true
8. removeEventListeners() (via onRemoveEventListenersEmitter)
9. library.destroy()
10. Stop trails: laserTrails, eraserTrail
11. Clear emitters: onChange, storeIncrement, durableIncrement, appStateObserver, editorLifecycleEvents
12. Destroy caches: ShapeCache, SnapCache
13. clearTimeout(touchTimeout)
14. Clear memoised selector caches: isSomeElementSelected, selectGroupsForSelectedElements
15. Reset document overscrollBehaviorX
```

---

## 4. Side Effects

### 4.1 DOM event listeners

Registered in `addEventListeners()` and cleaned up via `onRemoveEventListenersEmitter` (all unsubscribers stored, fired on `removeEventListeners()`). Re-registered on `viewModeEnabled` toggle.

| Scope | Events | Mode |
|---|---|---|
| `document` | `keydown`, `keyup` | view + edit (keydown only if `handleKeyboardGlobally`) |
| `document` | `pointerup`, `copy`, `pointermove` | view + edit |
| `document.fonts` | `loadingdone` | view + edit – font reload triggers re-render |
| `document` | `gesturestart/change/end` | view + edit – Safari pinch zoom |
| `window` | `message` | view + edit – YouTube/Vimeo postMessage |
| `window` | `focus` | view + edit – clean up missed pointerup, force repaint |
| `document` | `paste`, `cut`, `fullscreenchange` | edit only |
| `window` | `resize`, `unload`, `blur` | edit only |
| Container | `wheel` | both (passive: false) |
| Container | `dragover`, `drop` | edit only |
| Nearest scroll container | `scroll` | edit only, if `props.detectScroll` |

### 4.2 ResizeObserver

On container element. Fires `refreshEditorInterface()` + `updateDOMRect()`. Updates `editorInterface` (formFactor, canFitSidebar, isLandscape) and recalculates container offsets.

### 4.3 Async image pipeline

```
scene change → renderInteractiveSceneCallback → scheduleImageRefresh (throttled)
                                                      ↓
                                            addNewImagesToImageCache (async)
                                                      ↓
                                            updateImageCache → ShapeCache.delete
                                                      ↓
                                            scene.triggerUpdate → repaint
```

`addFiles()` (imperative API) follows the same path, plus `clearImageShapeCache()`.

### 4.4 Timers

| Timer | Purpose | Cleared in |
|---|---|---|
| `tappedTwiceTimer` | Double-tap detection (`TAP_TWICE_TIMEOUT`) | `App.resetTapTwice()` |
| `touchTimeout` | Long-press context menu | `resetContextMenuTimer()`, unmount |
| `IS_PLAIN_PASTE_TIMER` | Plain-paste mode window | paste handler |
| `bindModeHandler` | Delayed bind mode (`BIND_MODE_TIMEOUT`) | `resetDelayedBindMode()` |
| `cancelInProgressAnimation` | Scroll-to-content easing | next `scrollToContent` call |

### 4.5 AnimationFrameHandler (rAF-based)

Drives continuous canvas overlays:
- `laserTrails` – laser pointer trail
- `eraserTrail` – eraser stroke highlight
- `lassoTrail` – lasso selection path

Stopped in `componentWillUnmount`.

### 4.6 postMessage (embeds)

`onWindowMessage` handles `window.message` from `player.vimeo.com` and `www.youtube.com`. Tracks per-element YouTube player state in `YOUTUBE_VIDEO_STATES` map. `handleIframeLikeCenterClick` dispatches play/pause commands back to iframes.

### 4.7 Emitters (pub/sub)

| Emitter | Subscribers |
|---|---|
| `onChangeEmitter` | `api.onChange()`, collab |
| `onPointerDownEmitter` | `api.onPointerDown()` |
| `onPointerUpEmitter` | `api.onPointerUp()` |
| `onScrollChangeEmitter` | `api.onScrollChange()` |
| `onUserFollowEmitter` | `api.onUserFollow()` |
| `store.onStoreIncrementEmitter` | `api.onIncrement()`, collab |
| `store.onDurableIncrementEmitter` | `history.record()` |
| `missingPointerEventCleanupEmitter` | pointer-up cleanup |
| `onRemoveEventListenersEmitter` | event listener teardown |

### 4.8 AppEventBus (lifecycle)

`editorLifecycleEvents` with replay-last / once-only semantics:

| Event | Fired | Cardinality |
|---|---|---|
| `editor:mount` | end of `componentDidMount` | once, replay-last |
| `editor:initialize` | first `componentDidUpdate` after `isLoading=false` | once, replay-last |
| `editor:unmount` | start of `componentWillUnmount` | once, replay-last |

---

## 5. Imperative API (`ExcalidrawImperativeAPI`)

Constructed by `createExcalidrawAPI()`. Exposed via `props.onExcalidrawAPI` and `ExcalidrawAPIContext`. Key surface:

| Method | Delegates to |
|---|---|
| `updateScene()` | `this.updateScene` |
| `applyDeltas()` | `this.applyDeltas` |
| `mutateElement()` | `this.scene.mutateElement` |
| `addFiles()` | `this.addFiles` |
| `resetScene()` | `this.resetScene` |
| `scrollToContent()` | `this.scrollToContent` |
| `setActiveTool()` | `this.setActiveTool` |
| `toggleSidebar()` | `this.toggleSidebar` |
| `history.clear()` | `this.resetHistory` |
| `onChange/onPointerDown/…` | emitter `.on()` subscriptions |
| `onEvent` | `editorLifecycleEvents.on` |

API object reference is recreated in `componentDidMount` (StrictMode), and invalidated (stubbed) in `componentWillUnmount` with `isDestroyed: true`.

