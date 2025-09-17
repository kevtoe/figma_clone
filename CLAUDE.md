# Figma Clone Architecture Documentation

## Overview

This is a real-time collaborative design tool built with Next.js, TypeScript, Fabric.js, and Liveblocks. It implements core Figma-like functionality including real-time collaboration, canvas manipulation, comments system, and multi-user cursors.

## Technology Stack

- **Framework**: Next.js 14 with TypeScript
- **Canvas Library**: Fabric.js 5.3.0 - powers all shape manipulation and drawing
- **Real-time Collaboration**: Liveblocks - handles presence, storage, and comments
- **UI Components**: Radix UI primitives with Tailwind CSS
- **State Management**: Built-in React hooks + Liveblocks reactive storage

## Development Commands

```bash
npm run dev    # Start development server (SSR disabled for canvas)
npm run build  # Production build
npm run start  # Start production server
npm run lint   # ESLint with Prettier
```

## Environment Setup

Required environment variable:
```
NEXT_PUBLIC_LIVEBLOCKS_PUBLIC_KEY=your_liveblocks_public_key
```

Get your key from [Liveblocks Dashboard](https://liveblocks.io).

## Core Architecture

### Real-time Collaboration Layer

The app is built around Liveblocks' real-time infrastructure defined in `/liveblocks.config.ts`:

```typescript
type Storage = {
  canvasObjects: LiveMap<string, any>; // Synced shapes across users
};

type ThreadMetadata = {
  resolved: boolean;
  zIndex: number;
  x: number;         // Comment position on canvas
  y: number;
  time?: number;
};
```

**Key Liveblocks Integration Points:**
- `Room.tsx` - Wraps the app with `RoomProvider` and `LiveMap` for canvas objects
- `useStorage()` - Reactive subscription to shared canvas state
- `useMutation()` - Optimistic updates for shape modifications
- `useThreads()` - Comment system with spatial positioning

### Canvas Architecture

The canvas system is built on Fabric.js with extensive real-time synchronization:

**Core Canvas Files:**
- `/lib/canvas.ts` - Canvas initialization, event handlers, rendering logic
- `/lib/shapes.ts` - Shape creation functions (rectangle, circle, triangle, line, text, images)
- `/app/App.tsx` - Main canvas component orchestrating all interactions

**Canvas Event Flow:**
1. **Mouse Events** → Fabric.js handlers → Local shape manipulation
2. **Shape Changes** → `syncShapeInStorage` mutation → Liveblocks storage
3. **Storage Updates** → `useStorage` subscription → `renderCanvas` → Fabric.js re-render

**Critical Canvas Patterns:**
```typescript
// Real-time shape synchronization
const syncShapeInStorage = useMutation(({ storage }, object) => {
  const shapeData = object.toJSON();
  shapeData.objectId = objectId;
  const canvasObjects = storage.get("canvasObjects");
  canvasObjects.set(objectId, shapeData);
}, []);

// Canvas re-rendering on storage changes
useEffect(() => {
  renderCanvas({ fabricRef, canvasObjects, activeObjectRef });
}, [canvasObjects]);
```

### Real-time Presence System

**Live Cursors** (`/components/cursor/`)
- Each user's cursor position tracked via `useMyPresence` and `updateMyPresence`
- Cursor colors assigned cyclically from `COLORS` array based on `connectionId`
- Cursor chat activated with "/" key, reactions with "e" key

**Presence States** (`/types/type.ts`):
```typescript
enum CursorMode {
  Hidden,
  Chat,           // "/" to activate
  ReactionSelector, // "e" to activate  
  Reaction        // Mouse down to broadcast
}
```

### Comments System Architecture

The comments system overlays spatial comments on the canvas:

**Comment Components** (`/components/comments/`):
- `NewThread.tsx` - Handles comment placement workflow
- `CommentsOverlay.tsx` - Renders positioned comment threads  
- `PinnedThread.tsx` - Individual comment thread UI
- `PinnedComposer.tsx` - Comment creation/editing interface

**Comment Placement Flow:**
1. Click comment tool → `creatingCommentState: "placing"`
2. Click canvas position → `creatingCommentState: "placed"` + show composer
3. Submit comment → `createThread()` with x/y coordinates
4. Comments render absolutely positioned via CSS transforms

**Z-Index Management:**
Comments use a z-index system for layering. `useMaxZIndex` hook finds the highest z-index, and clicking comments promotes them to the top.

### Component Architecture

**Layout Structure:**
```
App.tsx (main canvas orchestrator)
├── Navbar.tsx (tools, active users)
├── LeftSidebar.tsx (shapes list)  
├── Live.tsx (canvas + overlays)
│   ├── <canvas> (Fabric.js)
│   ├── LiveCursors (other users)
│   ├── CursorChat (chat overlay)
│   ├── FlyingReactions (emoji animations)
│   └── Comments (spatial comments)
└── RightSidebar.tsx (shape properties)
```

**State Management Pattern:**
- Canvas state: Fabric.js objects + Liveblocks LiveMap
- UI state: Local React state for active tools, selections, etc.
- Collaborative state: Liveblocks presence, storage, and comments

### Key Integration Patterns

**Canvas-Storage Sync:**
Every Fabric.js object gets a UUID (`objectId`) for Liveblocks mapping. Changes flow:
Fabric.js object → `toJSON()` → LiveMap → Other users → Fabric.js reconstruction

**Undo/Redo System:**
Liveblocks provides automatic undo/redo for all mutations. Keyboard shortcuts (Cmd+Z, Cmd+Y) trigger `useUndo()` and `useRedo()`.

**Image Upload:**
File → FileReader → Base64 → `fabric.Image.fromURL()` → Canvas → Storage sync

**Context Menu Integration:**
Right-click on canvas shows context menu with shortcuts for chat, reactions, undo/redo.

## File Organization

```
/app/
  ├── App.tsx           # Main canvas application
  ├── Room.tsx          # Liveblocks room wrapper
  ├── layout.tsx        # Next.js layout with Room provider
  └── page.tsx          # SSR-disabled App component

/components/
  ├── Live.tsx          # Canvas container with overlays
  ├── Navbar.tsx        # Top toolbar
  ├── LeftSidebar.tsx   # Shapes list  
  ├── RightSidebar.tsx  # Properties panel
  ├── comments/         # Comment system components
  ├── cursor/           # Live cursor components
  ├── reaction/         # Reaction system
  └── users/            # Active users display

/lib/
  ├── canvas.ts         # Core canvas logic
  ├── shapes.ts         # Shape creation functions
  ├── key-events.ts     # Keyboard shortcuts
  └── useMaxZIndex.ts   # Z-index management

/types/type.ts          # TypeScript definitions
/constants/index.ts     # UI constants and configs
/liveblocks.config.ts   # Real-time configuration
```

## Performance Considerations

**Canvas Optimization:**
- Canvas events throttled at 16ms (60fps) via Liveblocks throttle
- Shape selection preserved across re-renders via `activeObjectRef`
- Canvas disposal on unmount prevents memory leaks

**Real-time Optimization:**
- Liveblocks handles optimistic updates and conflict resolution
- Presence updates batched automatically
- Storage mutations are atomic and consistently ordered

**SSR Handling:**
Canvas requires client-side rendering due to DOM dependencies. `page.tsx` uses `dynamic()` with `ssr: false`.

## Unique Architecture Decisions

1. **Canvas as Collaboration Medium**: Fabric.js objects serialized to JSON for real-time sync
2. **Spatial Comments**: Comments positioned with canvas coordinates, not DOM coordinates  
3. **Presence-Driven UI**: Cursor modes drive different interaction states
4. **Layered Architecture**: Canvas → Storage → Real-time → UI layers cleanly separated
5. **Event-Driven Sync**: Canvas events trigger storage mutations, storage changes trigger canvas updates

This architecture enables real-time collaborative design with smooth performance and conflict-free concurrent editing.