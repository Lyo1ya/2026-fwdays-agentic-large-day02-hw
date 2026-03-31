# Product Context — Excalidraw

> UX patterns, user-facing features, and key scenarios.
> Cross-ref: [domain-glossary.md](../product/domain-glossary.md) · [architecture.md](../technical/architecture.md) · [code-vs-docs.md](../technical/code-vs-docs.md) · [dev-setup.md](../technical/dev-setup.md)

---

## 1. Target Users

| Persona | Need |
|---------|------|
| **Casual sketcher** | Quick, informal diagrams without learning a complex tool |
| **Developer** | Embed `<Excalidraw>` React component into their own app via `@excalidraw/excalidraw` |
| **Team collaborator** | Real-time whiteboard with end-to-end encryption |

---

## 2. Canvas & Viewport

- Infinite 2D canvas rendered on two stacked `<canvas>` layers (`StaticCanvas` + `InteractiveCanvas`) plus an optional `NewElementCanvas` for in-progress shapes
- Coordinate system: **viewport ↔ scene** conversion via `viewportCoordsToSceneCoords` / `sceneCoordsToViewportCoords`
- Zoom: `Ctrl+wheel`, pinch-to-zoom (multi-touch + Safari gesture events), `Ctrl+0` reset, `Ctrl++`/`Ctrl+-`
- Scroll: drag with Hand tool (`H`), scrollbars, two-finger pan
- Grid mode (`Ctrl+'`): snaps elements to grid; object-snap mode (`Alt+S`): snaps to other elements

---

## 3. Toolbar & Tools

Defined in `packages/excalidraw/components/shapes.tsx` — `SHAPES` array:

| Tool | Shortcut | Notes |
|------|----------|-------|
| Hand (pan) | `H` | Drag-to-scroll, no element creation |
| Selection / Lasso | `V` / `1` | Box or lasso select; toggled via `preferredSelectionTool` |
| Rectangle | `R` / `2` | Fillable shape |
| Diamond | `D` / `3` | Fillable shape |
| Ellipse | `O` / `4` | Fillable shape |
| Arrow | `A` / `5` | Bindable to shapes |
| Line | `L` / `6` | Multi-point linear element |
| Freedraw | `P`/`X` / `7` | Freehand pencil strokes |
| Text | `T` / `8` | Auto-resize text element |
| Image | `9` | Embed raster images with compression |
| Eraser | `E` / `0` | Trail-based deletion |
| Laser pointer | `K` | Ephemeral pointer for presentations (not in toolbar) |
| Frame | `F` | Groups elements visually |

- **Tool lock** (`Q`): keeps active tool after placing an element (vs. auto-switch back to selection)
- **Pen mode**: stylus-only drawing, ignores palm/touch
- **Shape switch popup** (`ConvertElementTypePopup`): convert between rectangle ↔ diamond ↔ ellipse in-place

---

## 4. UI Layout

Composed in `LayerUI.tsx`:

| Zone | Contents |
|------|----------|
| **Top-left** | Main menu (hamburger): file ops, theme, settings, links |
| **Top-center** | Shape toolbar (`ShapesSwitcher`), lock & pen-mode buttons |
| **Top-right** | `UserList` (collaborator avatars), sidebar toggle |
| **Right sidebar** | `DefaultSidebar` — library panel, custom sidebar tabs |
| **Bottom** | `Footer` — zoom controls, undo/redo, help button |
| **Center overlay** | `WelcomeScreen` (shown on empty canvas) with hints for menu, toolbar, help |
| **Floating** | `ContextMenu`, `Toast`, `HintViewer`, `FollowMode` badge |

---

## 5. Key Dialogs & Panels

| Component | Trigger | Purpose |
|-----------|---------|---------|
| `CommandPalette` | `Ctrl+/` or `Ctrl+Shift+P` | Fuzzy-search all actions, tools, library items |
| `HelpDialog` | `?` | Keyboard shortcut reference |
| `ImageExportDialog` | `Ctrl+Shift+E` | Export to PNG, SVG, or copy PNG to clipboard; options for background, scale, padding, embed-scene |
| `JSONExportDialog` | `Ctrl+S` | Save scene as `.excalidraw` JSON file |
| `TTDDialog` | Menu → Text to Diagram | Convert Mermaid syntax or AI text prompt to diagram |
| `SearchMenu` | `Ctrl+F` | Find elements on canvas by text content |
| `Stats` | `Alt+/` | Scene & element stats (dimensions, angle, count) |
| `ColorPicker` | Property panel | Stroke, fill, background color selection + eye-dropper |
| `ElementLinkDialog` | `Ctrl+K` | Add hyperlink or link-to-element |
| `LibraryMenu` | Sidebar | Browse, insert, publish reusable element sets |
| `PasteChartDialog` | Paste CSV/TSV | Convert tabular data into chart elements |
| `ErrorDialog` | On error | Display error message with dismiss |

---

## 6. User Scenarios

### 6.1 Solo Drawing

1. Open app → `WelcomeScreen` with "Load file" and "Help" shortcuts
2. Pick a tool (keyboard shortcut or toolbar click)
3. Click/drag on canvas to create element
4. Select element → property panel shows stroke, fill, font, opacity, layers
5. Undo/redo (`Ctrl+Z` / `Ctrl+Shift+Z`) backed by `History` + `Store` delta system
6. Export: `Ctrl+Shift+E` → PNG/SVG with optional embedded scene data; `Ctrl+S` → `.excalidraw` JSON
7. Copy to clipboard: `Shift+Alt+C` copies selection as PNG

### 6.2 Real-time Collaboration

1. Click "Live collaboration" → `generateCollaborationLinkData()` creates `roomId` + `roomKey`
2. URL fragment `#room=<roomId>,<roomKey>` — key never sent to server
3. WebSocket via Socket.IO (`Portal.tsx`): messages encrypted with AES-GCM 128-bit
4. Sync protocol:
   - `SCENE_INIT` — full scene on room join
   - `SCENE_UPDATE` — incremental element diffs, merged via `reconcileElements()`
   - `MOUSE_LOCATION` — remote cursor positions (volatile, ~30 fps)
   - `IDLE_STATUS` — away/active state
   - `USER_VISIBLE_SCENE_BOUNDS` — viewport for follow mode
5. Persistence: Firestore for scene data, Firebase Storage for binary files (images)
6. **Follow mode**: click collaborator avatar → viewport syncs to their view (`FollowMode.tsx`)
7. `UserList` shows avatars, in-call/speaking/muted status indicators

### 6.3 Embedding in Third-party Apps

1. Install `@excalidraw/excalidraw` npm package
2. Render `<Excalidraw initialData={...} onChange={...} />` — full React component
3. Access `ExcalidrawImperativeAPI` via `onExcalidrawAPI` callback:
   - `updateScene()`, `getSceneElements()`, `setActiveTool()`, `scrollToContent()`
   - `addFiles()`, `resetScene()`, `toggleSidebar()`
   - Event subscriptions: `onChange`, `onPointerDown`, `onPointerUp`, `onScrollChange`
4. Customization points: `renderTopLeftUI`, `renderTopRightUI`, `UIOptions`, custom `MainMenu`, custom `WelcomeScreen`
5. Working examples: `examples/with-nextjs/`, `examples/with-script-in-browser/`

### 6.4 Library & Reuse

1. Select elements → "Add to library" (or via command palette)
2. Library items stored locally (IndexedDB via `LocalData`)
3. Insert from library panel in sidebar — items rendered as SVG thumbnails
4. Publish library for sharing; import `.excalidrawlib` files

### 6.5 Advanced Workflows

- **Mermaid → Excalidraw**: paste Mermaid syntax in TTD dialog → auto-converts to diagram elements
- **Text to Diagram (AI)**: describe diagram in natural language → AI generates elements (requires `onTextSubmit` handler)
- **Flowchart creation**: arrow auto-binding to shapes, elbow connectors, midpoint snapping
- **Frames**: group elements into named frames (`F` key); frame-scoped export
- **Element locking** (`Ctrl+Shift+L`): prevent accidental edits
- **Image cropping**: in-place crop editor for image elements
- **Deep linking**: `#elementLink=<id>` in URL scrolls to specific element

---

## 7. Interaction Model

### Pointer Pipeline

```
onPointerDown → hitTest / tool dispatch
    → onPointerMove (drag: create, resize, move, draw)
        → onPointerUp (finalize element, commit to Store)
```

- **Multi-touch**: pinch-to-zoom via `gesture` map tracking active pointers
- **Long-press** (touch): opens context menu after `touchTimeout`
- **Double-tap**: text editing or shape conversion
- **Alt+drag**: duplicate element

### Keyboard

- Tool shortcuts: single key (`R`, `D`, `O`, etc.)
- Action shortcuts: modifier combos (`Ctrl+G` group, `Ctrl+D` duplicate)
- `ActionManager.handleKeyDown()` dispatches with priority ordering
- View-mode restricts available actions to read-only subset

### Modes

| Mode | Toggle | Effect |
|------|--------|--------|
| **View mode** | `Alt+R` | Read-only, no editing, restricted actions |
| **Zen mode** | `Alt+Z` | Hides UI chrome for distraction-free viewing |
| **Grid mode** | `Ctrl+'` | Shows grid, snaps to grid |
| **Snap mode** | `Alt+S` | Snaps to other element edges/centers |

---

## 8. Theming & Accessibility

- Light / dark theme toggle (`Shift+Alt+D`) — persisted, affects canvas and UI
- Hand-drawn aesthetic via Rough.js rendering
- Multi-language support: 40+ locales via Crowdin (`packages/excalidraw/locales/`)
- RTL layout support
- PWA: installable, offline-capable via service worker
- Keyboard-navigable command palette and dialogs

