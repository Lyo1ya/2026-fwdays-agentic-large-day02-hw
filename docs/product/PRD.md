# Product Requirements Document — Excalidraw

> Reverse-engineered from source code (v0.18.0). Last verified: 2026-03-29.

---

## 1. Product Purpose

Excalidraw is a browser-based virtual whiteboard for creating diagrams with a
hand-drawn (sketchy) aesthetic. It ships in two forms:

1. **Standalone web app** (`excalidraw-app/`) — hosted on excalidraw.com
2. **Embeddable React library** (`@excalidraw/excalidraw` npm package) — for
   third-party integrations

Core value: instant, collaborative, end-to-end encrypted diagramming that
requires zero setup.

---

## 2. Target Audience

| Segment | Need |
|---------|------|
| **Casual users** | Quick, informal diagrams without design skills |
| **Developers** | Embed the React component into their own apps |
| **Teams** | Real-time collaborative whiteboarding |
| **Educators / presenters** | Visual explanations with hand-drawn feel |

---

## 3. Key Features

### 3.1 Drawing & Editing

- **Shapes**: rectangle, diamond, ellipse, arrow, line, freedraw, text
  (`ExcalidrawElement` union — `packages/element/src/types.ts`)
- **Image embedding**: supports PNG, JPEG, SVG, GIF, WebP, AVIF, BMP, ICO, JFIF
  (`IMAGE_MIME_TYPES` — `packages/common/src/constants.ts:237`)
- **Frames**: grouping containers (`ExcalidrawFrameElement`), including
  AI-powered "magic frames" (`ExcalidrawMagicFrameElement`)
- **Embeddables**: YouTube, Figma, and arbitrary iframes
  (`ExcalidrawEmbeddableElement`, `ExcalidrawIframeElement`)
- **Text**: auto-resize, 10 font families (Excalifont, Virgil, Helvetica,
  Cascadia, Nunito, Lilita One, Comic Shanns, Liberation Sans, Assistant)
  (`FONT_FAMILY` — `packages/common/src/constants.ts:130`)
- **Hand-drawn aesthetic**: Rough.js with deterministic seed per element
- **Freehand drawing**: pressure-sensitive via `perfect-freehand` library
- **Element locking**: `locked` property prevents accidental edits
- **Fractional indexing**: z-order via `fractional-indexing` library

### 3.2 Manipulation

- **~60 registered actions** (`ActionName` union — `packages/excalidraw/actions/types.ts`):
  copy, paste, undo, redo, delete, duplicate, align, distribute, flip,
  group/ungroup, z-index reorder, style changes, zoom, select all, etc.
- **Command palette** (`CommandPalette` component)
- **Keyboard shortcuts** (`actions/shortcuts.ts`)
- **Shape switching** (`actionToggleShapeSwitch.tsx`)
- **Linear element editing** with point manipulation (`actionLinearEditor.tsx`)
- **Bound text** on shapes (`actionBoundText.tsx`)
- **Element links / hyperlinks** (`actionElementLink.ts`, `actionLink.tsx`)
- **Crop editor** for images (`actionCropEditor.tsx`)
- **Object & midpoint snapping** (`actionToggleObjectsSnapMode.tsx`,
  `actionToggleMidpointSnapping.tsx`)

### 3.3 Export & Import

- **Export formats**: PNG, SVG, JSON, clipboard (PNG and SVG)
  (`EXPORT_IMAGE_TYPES` — `packages/common/src/constants.ts:279`)
- **Export scales**: 1×, 2×, 3× (`EXPORT_SCALES` constant)
- **Scene embedding**: scene data can be embedded in exported PNG/SVG files
  for round-trip import (via `png-chunk-*` and SVG metadata)
- **Import**: `.excalidraw` JSON, `.excalidraw.png`, `.excalidraw.svg` files
- **Library system**: reusable element sets, import/export, add-to-library
  (`actionAddToLibrary.ts`, `LibraryMenu.tsx`)

### 3.4 Collaboration

- **Real-time sync** via Socket.IO (`socket.io-client`)
  - Scene updates (`WS_SUBTYPES.UPDATE`)
  - Mouse location broadcasting (`WS_SUBTYPES.MOUSE_LOCATION`)
  - Idle status sync (`WS_SUBTYPES.IDLE_STATUS`)
  - Cursor sync at ~30fps (`CURSOR_SYNC_TIMEOUT = 33ms`)
- **End-to-end encryption**: AES-GCM 128-bit
  (`packages/excalidraw/data/encryption.ts`, `ENCRYPTION_KEY_BITS = 128`)
- **Conflict resolution**: higher `version` wins; equal versions → lower
  `versionNonce` wins; actively-edited local elements never overwritten
  (`packages/excalidraw/data/reconcile.ts`)
- **Full scene sync** every 20s (`SYNC_FULL_SCENE_INTERVAL_MS = 20000`)
- **Follow mode**: follow another user's viewport (`FollowMode/` component)

### 3.5 Persistence

| Layer | Technology | Key constants |
|-------|-----------|---------------|
| Local elements | `localStorage` | `SAVE_TO_LOCAL_STORAGE_TIMEOUT = 300ms` |
| Local files | IndexedDB (`idb-keyval`) | `FILE_CACHE_MAX_AGE_SEC = 31536000` (1 year) |
| Remote scene | Firebase Firestore | Room-based documents |
| Remote files | Firebase Storage | `FILE_UPLOAD_MAX_BYTES = 4 MiB` |
| Cross-tab sync | `localStorage` events | `SYNC_BROWSER_TABS_TIMEOUT = 50ms` |

### 3.6 UI Modes

- **Light / Dark theme** (`THEME.LIGHT`, `THEME.DARK`)
- **View mode** — read-only canvas (`actionToggleViewMode.tsx`)
- **Zen mode** — minimal UI (`actionToggleZenMode.tsx`)
- **Grid mode** — snap to grid (`DEFAULT_GRID_SIZE = 20`, `DEFAULT_GRID_STEP = 5`)
- **Stats panel** — element/scene statistics (`actionToggleStats.tsx`)

### 3.7 PWA & Offline

- Progressive Web App support (`vite-plugin-pwa`)
- Service worker for offline caching (`public/service-worker.js`)
- Installable (`labels.installPWA` in App.tsx)

### 3.8 Integrations

- **Mermaid-to-Excalidraw** conversion (`TTDDialog/` component)
- **Diagram-to-Code** AI plugin (`DiagramToCodePlugin/` component)
- **Embeddable React component** with imperative API (`ExcalidrawImperativeAPI`)
- **Eye dropper** for color picking (`EyeDropper.tsx`)
- **Paste chart** from spreadsheet data (`PasteChartDialog.tsx`)

### 3.9 i18n

- Multi-language support via locale JSON files (`packages/excalidraw/locales/`)
- Language detection and selection (`excalidraw-app/app-language/`)

---

## 4. Technical Architecture Summary

### 4.1 Monorepo Structure

```
excalidraw-app/           → Standalone web app (Vite + React 19)
packages/
  excalidraw/             → Core React component library (v0.18.0)
  element/                → Element model, Scene, Store
  math/                   → 2D geometry (branded tuple types)
  common/                 → Shared constants, utilities, Emitter
  utils/                  → Export/conversion helpers
```

Build order: `common → math → element → excalidraw → utils`

### 4.2 Rendering

- **Dual-canvas architecture**: StaticCanvas (shapes) + InteractiveCanvas
  (selection, cursors, handles)
- **Per-element offscreen canvas caching** via `WeakMap`
- **Memoized render pipeline**: skips re-render if `sceneNonce` unchanged
- **App component**: class component (~12,800 lines) for imperative API
  and performance-critical mutable instance state

### 4.3 State Management

- **AppState**: 100+ field interface for all UI state
- **Scene**: canonical element array with fractional index ordering
- **Store**: snapshot diffing, produces `DurableIncrement` (→ History for
  undo/redo) and `EphemeralIncrement` (→ collab sync)
- **Jotai**: two isolated atom stores (editor-level, app-level)

---

## 5. Technical Limitations & Constraints

### 5.1 Performance

- `App` is a single class component (~12,800 lines) — large bundle, hard to
  code-split
- Canvas rendering is CPU-bound; no WebGL/GPU acceleration
- Per-element canvas caching trades memory for render speed
- Zoom range: 0.1× to 30× (`MIN_ZOOM`, `MAX_ZOOM`)

### 5.2 File & Data Limits

- Max image file size: **4 MiB** (`MAX_ALLOWED_FILE_BYTES`)
- Default max image dimension: **1440px** (`DEFAULT_MAX_IMAGE_WIDTH_OR_HEIGHT`)
- SVG export decimal precision: 2 digits (`MAX_DECIMALS_FOR_SVG_EXPORT`)
- No server-side rendering — browser-only (Web Crypto API, Canvas API)

### 5.3 Collaboration

- Encryption key is in the URL fragment — anyone with the link can decrypt
- No access control / permissions on shared rooms
- Conflict resolution is last-write-wins by version counter
- Deleted elements retained for 24h (`DELETED_ELEMENT_TIMEOUT = 86400000ms`)
- Full scene re-sync every 20s can be bandwidth-heavy for large scenes

### 5.4 Browser Requirements

- Requires `crypto.subtle` (Web Crypto API) — no HTTP-only support
- Canvas 2D context required — no fallback rendering
- Node.js ≥ 18 for development (`engines` in `package.json`)
- IndexedDB required for local file caching

### 5.5 Embedding

- React 19 peer dependency
- Jotai used internally — potential version conflicts with host apps
- CSS modules — style isolation depends on build tool configuration
- No SSR support — client-side only

### 5.6 Data Format

- Scene format is JSON (`application/vnd.excalidraw+json`)
- No schema versioning / migration system for saved files
- `isDeleted` soft-delete pattern means element arrays grow unboundedly
  in long-lived documents

---

## 6. Deployment Targets

| Target | Stack |
|--------|-------|
| **Vercel** | Primary hosting for excalidraw.com |
| **Docker** | Nginx-served static build (`Dockerfile` + `docker-compose.yml`) |
| **npm** | `@excalidraw/excalidraw`, `@excalidraw/math`, `@excalidraw/element`, `@excalidraw/common`, `@excalidraw/utils` |

---

## 7. Monitoring & Error Handling

- **Sentry** integration for error tracking (`excalidraw-app/sentry.ts`)
- **Analytics** event tracking (`@excalidraw/excalidraw/analytics`)
- **Error boundaries** via `ErrorDialog` component
- **Overwrite confirmation** dialogs for destructive actions

