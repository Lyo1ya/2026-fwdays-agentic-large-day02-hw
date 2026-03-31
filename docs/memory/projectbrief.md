# Project Brief — Excalidraw

> Cross-ref: [domain-glossary.md](../product/domain-glossary.md) · [architecture.md](../technical/architecture.md) · [code-vs-docs.md](../technical/code-vs-docs.md) · [dev-setup.md](../technical/dev-setup.md)

## Overview

Excalidraw is an open-source, browser-based **virtual whiteboard** for sketching diagrams with a hand-drawn feel. It is published as both a standalone web application (`excalidraw-app/`) and a reusable **React component library** (`@excalidraw/excalidraw` npm package).

- **License:** MIT (Copyright © 2020 Excalidraw)
- **Repository name:** `excalidraw-monorepo`
- **Current package version:** 0.18.0

## Core Value Proposition

- Instant, collaborative, end-to-end encrypted diagramming in the browser
- Embeddable React component for third-party integrations
- Hand-drawn (sketchy) aesthetic powered by Rough.js
- Real-time multi-user collaboration via WebSocket + Firebase

## Key Features

### Drawing & Editing
- Freehand drawing, shapes (rectangles, ellipses, diamonds, arrows, lines)
- Text elements with auto-resize
- Image embedding with compression
- Library of reusable element sets
- Undo/redo with full history tracking
- Export to PNG, SVG, JSON, clipboard
- Import from JSON / Excalidraw files

### Collaboration
- Real-time collaborative editing with WebSocket rooms
- End-to-end encryption (AES-GCM 128-bit) — server never sees plaintext
- Cursor & idle-state synchronisation across users
- Firebase Firestore for persistence, Firebase Storage for binary files
- Shareable collaboration links with room ID + key in URL fragment

### Developer API
- `@excalidraw/excalidraw` — drop-in React component with imperative API
- `@excalidraw/math` — 2D math primitives (points, vectors, curves, polygons)
- `@excalidraw/element` — element creation, mutation, type-checking
- `@excalidraw/common` — shared constants, utilities, event bus
- `@excalidraw/utils` — helper utilities (export, conversion)

### Extras
- Progressive Web App (PWA) with offline support
- Command palette, keyboard shortcuts
- Mermaid-to-Excalidraw conversion
- "Diagram to Code" plugin (AI-powered)
- Multi-language support (i18n with Crowdin)
- Theme support (light / dark)

## Target Audience

1. **End users** — anyone who needs quick, informal diagrams
2. **Developers** — embedding the React component into their own apps
3. **Teams** — real-time whiteboard collaboration

## Deployment Targets

| Target | Details |
|--------|---------|
| **Vercel** | Primary hosting for excalidraw.com |
| **Docker** | `Dockerfile` + `docker-compose.yml`, Nginx-served static build |
| **npm** | Published packages: `@excalidraw/excalidraw`, `@excalidraw/math`, `@excalidraw/element`, `@excalidraw/common`, `@excalidraw/utils` |

## Repository Layout (high level)

```
excalidraw-app/     — Standalone web app (entry point)
packages/
  excalidraw/       — Core React component library
  element/          — Element model & operations
  math/             — 2D geometry primitives
  common/           — Shared utilities & constants
  utils/            — Export & conversion helpers
examples/           — Integration examples (Next.js, Vite)
firebase-project/   — Firestore rules, indexes
scripts/            — Build, release, locale tooling
public/             — Static assets (fonts, icons)
```

## Details
For detailed architecture → see docs/technical/architecture.md
For domain glossary → see docs/product/domain-glossary.md
