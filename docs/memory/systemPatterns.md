# System Patterns — Excalidraw

> Cross-ref: [domain-glossary.md](../product/domain-glossary.md) · [architecture.md](../technical/architecture.md) · [code-vs-docs.md](../technical/code-vs-docs.md) · [dev-setup.md](../technical/dev-setup.md)

## High-Level Architecture

```
┌───────────────────────────────────────────────┐
│              excalidraw-app                   │
│  (standalone app: collab, Firebase, routing)  │
├───────────────────────────────────────────────┤
│         @excalidraw/excalidraw                │
│  (React component, actions, renderer, UI)     │
├──────────┬──────────┬────────────┬────────────┤
│ @excali  │ @excali  │ @excalidraw│ @excalidraw│
│ draw/    │ draw/    │ /math      │ /utils     │
│ element  │ common   │            │            │
└──────────┴──────────┴────────────┴────────────┘
```

**Dependency flow (bottom-up):**
`common` → `math` → `element` → `excalidraw` → `excalidraw-app`

Build order follows the same chain; packages must be built in this order.

## Package Responsibilities

### `@excalidraw/common`
- Shared constants (`APP_NAME`, `THEME`, `MIME_TYPES`, encryption key bits)
- Utility functions (`debounce`, `throttleRAF`, `resolvablePromise`, `cloneJSON`)
- Event bus (`appEventBus`)
- Branded/utility types (`Mutable`, `ValueOf`, `ResolvablePromise`)
- Color helpers, key constants, URL utils

### `@excalidraw/math`
- 2D geometry primitives using **branded tuple types**
- Core types: `GlobalPoint`, `LocalPoint`, `Radians`, `Degrees`, `InclusiveRange`
- Modules: `point`, `vector`, `segment`, `line`, `curve`, `ellipse`, `polygon`, `rectangle`, `triangle`, `angle`, `range`
- **Convention:** Points are `[x, y]` tuples, not `{ x, y }` objects

### `@excalidraw/element`
- Element model: creation (`newElement`), mutation (`mutateElement`, `newElementWith`)
- Element types: shapes, text, images, arrows, frames, embeddables
- Hit testing (`collision`, `resizeTest`)
- Bounds calculation, alignment, distribution
- Fractional indexing for z-order (`fractionalIndex`)
- Arrow routing including elbow arrows (`elbowArrow`)
- Scene management (`Scene.ts`)
- Element store with versioning (`store.ts`)
- Binding logic (arrow ↔ shape connections)
- Visual debug overlay (`visualdebug.ts`)

### `@excalidraw/excalidraw`
- Main `<Excalidraw>` React component + imperative API
- Action system (40+ actions: clipboard, align, flip, export, etc.)
- Renderer (interactive canvas + static canvas + SVG export)
- Data layer: JSON serialization, encryption, compression, restore
- i18n (multi-language via locale JSON files)
- Fonts (custom WOFF2: Virgil, Cascadia, Assistant)
- UI components: toolbar, sidebar, dialogs, color picker, command palette
- History (undo/redo)
- Hooks-based architecture with Jotai state

### `@excalidraw/utils`
- Standalone utilities for export & conversion
- No React dependency required

## State Management

### Jotai (Atomic State)
- Two Jotai stores coexist:
  - **`editorJotaiStore`** — internal to `@excalidraw/excalidraw`, scoped per editor instance via `EditorJotaiProvider`
  - **`appJotaiStore`** — app-level store in `excalidraw-app` for collaboration state
- Key atoms:
  - `collabAPIAtom` — collaboration API handle
  - `isCollaboratingAtom` — collaboration mode flag
  - `isOfflineAtom` — connectivity status

### AppState
- Central `AppState` interface in `packages/excalidraw/types.ts`
- Managed inside the main `<App>` component
- Serialised/restored via `restoreAppState()`

### Element State
- Elements stored as ordered arrays (`OrderedExcalidrawElement[]`)
- Each element has a version number, bumped on mutation
- Scene versions tracked for sync conflict resolution
- `reconcileElements()` merges local + remote changes

## Rendering Architecture

- **Dual-canvas approach:**
  - `staticScene` — renders non-interactive elements (exported drawings)
  - `interactiveScene` — renders selection handles, cursors, collaborative pointers
- Rough.js generates SVG-like hand-drawn paths on HTML5 Canvas
- `Renderer.ts` in `scene/` orchestrates render scheduling
- SVG export via `staticSvgScene.ts`
- Animation frame handler for smooth updates (`animation-frame-handler.ts`)

## Collaboration Architecture

```
┌──────────┐  WebSocket  ┌──────────────┐  Firestore  ┌──────────┐
│  Client  │ ◄──────────►│  WS Server   │◄───────────►│ Firebase │
│ (Portal) │  (encrypted) │ (relay only) │  (encrypted) │ Database │
└──────────┘              └──────────────┘              └──────────┘
```

- **Portal** (`Portal.tsx`): manages WebSocket connection via `socket.io-client`
- **Collab** (`Collab.tsx`): orchestrates collaboration lifecycle (PureComponent)
- **End-to-end encryption:** AES-GCM 128-bit; key never sent to server, stays in URL fragment
- **Sync protocol:**
  - `SCENE_INIT` — full scene broadcast on room join
  - `SCENE_UPDATE` — incremental element updates
  - `MOUSE_LOCATION` — cursor position (volatile broadcast, ~30fps)
  - `IDLE_STATUS` — user idle state
- **Element versioning:** `broadcastedElementVersions` Map prevents redundant broadcasts
- **Conflict resolution:** `reconcileElements()` merges by element version
- **File sync:** images uploaded to Firebase Storage with AES-GCM encryption
- **Tab sync:** `tabSync.ts` coordinates state across browser tabs via `BroadcastChannel`

## Data Persistence

| Storage | Purpose |
|---------|---------|
| **Firebase Firestore** | Collaborative room scene data (encrypted) |
| **Firebase Storage** | Binary files / images (encrypted) |
| **localStorage** | Local scene, app state, theme, username |
| **IndexedDB** | Library items, TTD chat history (`idb-keyval`) |

## Action System

- Actions defined in `packages/excalidraw/actions/`
- Each action: `{ name, perform, trackEvent, keyTest?, PanelComponent? }`
- Registered via `register.ts`, managed by `manager.tsx`
- Actions can modify both elements and app state
- `CaptureUpdateAction` enum controls undo/redo capture granularity

## Key Design Patterns

### Branded Types
- Math types use TypeScript branded primitives: `Radians`, `Degrees`, `GlobalPoint`, `LocalPoint`
- Prevents accidental mixing of coordinate systems at compile time

### Immutable Element Updates
- `newElementWith(element, updates)` — returns a new element with bumped version
- `mutateElement()` — in-place mutation (used sparingly, during active editing)

### Fractional Indexing
- Z-order uses `fractional-indexing` library
- Allows inserting elements between existing ones without reindexing

### Event-Driven Communication
- `appEventBus` for cross-component events
- `emitter.ts` for typed pub/sub

### Encryption
- `encryptData()` / `decryptData()` — AES-GCM via Web Crypto API
- `generateEncryptionKey()` — produces exportable JWK key
- Room key stored only in URL hash (never sent to server)

### Data Serialization Pipeline
```
Elements → JSON.stringify → compress (pako) → encrypt (AES-GCM) → Firestore
Firestore → decrypt → decompress → JSON.parse → restoreElements() → Elements
```

### PWA Support
- Service worker via `vite-plugin-pwa`
- Manifest icons in `public/`
- Offline-capable with local persistence fallback

## Testing Patterns

- Tests co-located with source in `__tests__/` or `*.test.ts(x)` files
- `setupTests.ts` at root: mocks canvas, throttleRAF, provides polyfills
- `jsdom` environment for all tests
- Helpers in `packages/excalidraw/tests/helpers/` for UI interaction simulation
- Coverage thresholds enforced (lines 60%, branches 70%, functions 63%)

## Build & Release Pipeline

1. **Package build order:** `common` → `math` → `element` → `excalidraw`
2. **Package build tool:** esbuild (via `scripts/buildPackage.js`)
3. **App build:** Vite
4. **Type generation:** `tsc` (separate step per package)
5. **Release:** `scripts/release.js` — publishes to npm with tag (test/next/latest)
6. **Docker:** multi-stage — Node 18 build, Nginx 1.27-alpine serve

## Details
For detailed architecture → see docs/technical/architecture.md
For domain glossary → see docs/product/domain-glossary.md
