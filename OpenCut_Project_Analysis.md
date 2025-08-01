# OpenCut Project Structure Summary

## Overview
OpenCut is a **free, open-source video editor** built for web, desktop, and mobile. It's essentially a browser-based alternative to CapCut, focusing on privacy (files stay on device), simplicity, and removing paywalls.

## Architecture

### **Monorepo Structure (Turbo + Bun)**
```
opencut/
├── apps/
│   └── web/              # Main Next.js application
├── packages/
│   ├── auth/             # Authentication package
│   └── db/               # Database package (Drizzle + PostgreSQL)
└── [config files]       # Turbo, Biome, Docker, etc.
```

### **Main Web Application (`apps/web/`)**
```
src/
├── app/                  # Next.js app router
│   └── editor/[project_id]/  # Dynamic editor route
├── components/
│   ├── editor/           # Core editor components
│   │   ├── timeline/     # Timeline implementation
│   │   ├── media-panel/  # Media library
│   │   ├── properties-panel/  # Element properties
│   │   └── preview-panel.tsx  # Video preview & rendering
│   └── ui/               # Reusable UI components
├── stores/               # Zustand state management
├── hooks/                # Custom React hooks
├── lib/                  # Utilities & services
└── types/                # TypeScript definitions
```

## **Editor Frontend Architecture**

### **1. State Management (Zustand Stores)**

#### **Timeline Store** (`timeline-store.ts`) - 🎯 **Core Timeline Logic**
- **1,552 lines** - The heart of the editor
- **Multi-track management**: Video, audio, text tracks with proper ordering
- **Element management**: Add, move, split, trim, delete elements
- **Drag & drop**: Complex drag state management with snapping
- **Ripple editing**: Move/delete elements with automatic gap filling
- **History system**: Undo/redo with state snapshots
- **Snapping system**: Element-to-element and playhead snapping
- **Real-time persistence**: Auto-saves to IndexedDB/OPFS

#### **Other Key Stores**:
- **Playback Store**: Timeline playback with frame-accurate seeking
- **Media Store**: File management, thumbnails, metadata
- **Project Store**: Project lifecycle, bookmarks, settings
- **Editor Store**: Canvas size, global editor state

### **2. Timeline Implementation** 🎬

#### **Core Timeline Components**:

**`Timeline` (main component)**:
- Resizable panel layout with rulers and scroll sync
- Zoom controls (0.25x to 10x) with mouse wheel
- Selection box for multi-element selection
- Keyboard shortcuts integration
- Drag & drop from media panel

**`TimelineTrack`**:
- Individual track rendering with drag zones
- Drop validation (text→text tracks, media→media/audio tracks)
- Overlap detection and prevention
- Track-specific context menus

**`TimelineElement`**:
- Individual element rendering with resize handles
- Trimming with visual feedback
- Context menus for split/delete/copy operations
- Real-time duration and position updates

**`TimelinePlayhead`**:
- Frame-accurate playhead positioning
- Click-to-seek functionality
- Snap-to-frame behavior based on project FPS

#### **Timeline Constants & Mechanics**:
```typescript
TIMELINE_CONSTANTS = {
  PIXELS_PER_SECOND: 50,        // Base timeline scale
  ELEMENT_MIN_WIDTH: 80,        // Minimum element width
  DEFAULT_TEXT_DURATION: 5,     // Default text duration
  ZOOM_LEVELS: [0.25, 0.5, 1, 1.5, 2, 3, 4]
}

TRACK_HEIGHTS = {
  media: 65px,    // Video tracks
  text: 25px,     // Text overlays
  audio: 50px     // Audio tracks
}
```

### **3. Advanced Timeline Features**

#### **Snapping System** (`use-timeline-snapping.ts`):
- **Element-to-element snapping**: Edges align automatically
- **Playhead snapping**: Elements snap to current time
- **Configurable threshold**: 10px snap distance
- **Multi-edge detection**: Start/end edge prioritization

#### **Element Resizing** (`use-timeline-element-resize.ts`):
- **Trimming**: Non-destructive start/end trimming
- **Duration adjustment**: Visual feedback during resize
- **Constrained resizing**: Respect minimum widths and overlaps

#### **Selection System** (`use-selection-box.ts`):
- **Marquee selection**: Click-drag rectangular selection
- **Multi-element operations**: Bulk delete, move, copy
- **Keyboard modifiers**: Ctrl/Cmd for additive selection

### **4. Media Processing Pipeline**

#### **File Handling** (`media-processing.ts`):
```typescript
// Supports: Video (MP4, WebM), Audio (MP3, WAV), Images (JPG, PNG)
processMediaFiles() -> {
  - Generate thumbnails (FFmpeg fallback)
  - Extract metadata (duration, dimensions, FPS)
  - Create blob URLs for playback
  - Store in IndexedDB/OPFS
}
```

#### **FFmpeg Integration** (`ffmpeg-utils.ts`):
- **Browser-based FFmpeg**: `@ffmpeg/ffmpeg` for video processing
- **Thumbnail generation**: Extract video frames
- **Metadata extraction**: Duration, resolution, frame rate
- **Export preparation**: Future rendering pipeline

### **5. Preview Panel & Rendering**

#### **Real-time Preview** (`preview-panel.tsx` - 969 lines):
- **Canvas-based rendering**: Composite video, audio, text layers
- **Playback synchronization**: Frame-accurate timeline sync
- **Text overlay rendering**: Font families, positioning, animations
- **Aspect ratio management**: Responsive preview scaling
- **Fullscreen mode**: Expanded preview with escape key

#### **Background System**:
- **Color backgrounds**: Solid colors with opacity
- **Blur backgrounds**: Configurable blur intensity
- **Project-level settings**: Persistent background preferences

### **6. Storage & Persistence**

#### **Multi-tier Storage** (`storage-service.ts`):
```typescript
// Project metadata: IndexedDB
// Media files: OPFS (Origin Private File System)
// Timeline data: IndexedDB (per-project)

Storage Architecture:
├── video-editor-projects          # Project metadata
├── video-editor-media-{projectId} # Media metadata per project  
├── video-editor-timelines-{projectId} # Timeline state per project
└── media-files-{projectId}        # Actual media files (OPFS)
```

#### **Auto-save System**:
- **Debounced saves**: 100ms delay after changes
- **Incremental updates**: Only save changed data
- **Error handling**: Graceful fallbacks for storage failures

### **7. Keyboard Shortcuts & UX**

#### **Keybinding System** (`keybindings-store.ts`):
```typescript
// Playback controls
Space: Play/Pause
←/→: Frame by frame navigation
J/K/L: Rewind/Pause/Fast forward

// Timeline operations  
Ctrl+Z/Y: Undo/Redo
Delete: Remove selected elements
C: Split at playhead
Ctrl+D: Duplicate elements

// View controls
+/-: Zoom in/out
Ctrl+0: Reset zoom
```

## **How the Timeline Works** 🔧

### **Data Flow**:
1. **Media Import** → `media-store` → **File processing** → **Thumbnail generation**
2. **Drag to Timeline** → **Track validation** → **Element creation** → `timeline-store`
3. **Element Manipulation** → **Drag/resize/trim** → **State updates** → **Auto-save**
4. **Playback** → `playback-store` → **Preview rendering** → **Element synchronization**

### **Track Management**:
- **Track Ordering**: Text → Video → Audio (automatic sorting)
- **Main Track**: Guaranteed primary video track
- **Track Types**: Strict typing prevents incompatible drops

### **Element Lifecycle**:
```typescript
Element Creation → Timeline Placement → Real-time Updates → Persistence
     ↓                    ↓                   ↓              ↓
Media Processing → Track Validation → Preview Rendering → IndexedDB
```

### **Performance Optimizations**:
- **RequestAnimationFrame**: Smooth playback updates
- **Canvas rendering**: Hardware-accelerated preview
- **Lazy loading**: Media thumbnails on demand
- **Debounced operations**: Reduced state updates
- **OPFS storage**: Fast file access

## **Key Technical Decisions**

1. **Zustand over Redux**: Simpler state management for complex editor state
2. **IndexedDB + OPFS**: Client-side persistence without server dependencies  
3. **FFmpeg.wasm**: Browser-based video processing
4. **Canvas rendering**: Frame-accurate preview compositing
5. **Monorepo structure**: Shared packages for auth and database
6. **Next.js App Router**: Modern React patterns with SSR support

## **Key Files & Components**

### **Timeline Core**:
- `apps/web/src/stores/timeline-store.ts` (1,552 lines) - Master timeline state management
- `apps/web/src/components/editor/timeline/index.tsx` (1,207 lines) - Main timeline UI
- `apps/web/src/components/editor/timeline/timeline-track.tsx` (1,152 lines) - Track rendering
- `apps/web/src/types/timeline.ts` (166 lines) - Timeline type definitions

### **Editor Interface**:
- `apps/web/src/app/editor/[project_id]/page.tsx` - Main editor layout
- `apps/web/src/components/editor/preview-panel.tsx` (969 lines) - Video preview
- `apps/web/src/stores/playback-store.ts` (157 lines) - Playback control

### **Media & Storage**:
- `apps/web/src/lib/storage/storage-service.ts` (284 lines) - Data persistence
- `apps/web/src/lib/media-processing.ts` (102 lines) - File processing
- `apps/web/src/stores/media-store.ts` (328 lines) - Media management

### **Advanced Features**:
- `apps/web/src/hooks/use-timeline-snapping.ts` (179 lines) - Snapping system
- `apps/web/src/hooks/use-selection-box.ts` (200 lines) - Multi-selection
- `apps/web/src/hooks/use-timeline-element-resize.ts` (227 lines) - Element resizing

## **Dependencies**

### **Core Framework**:
- **Next.js 15.4.5**: React framework with app router
- **React 18.2.0**: UI library
- **TypeScript 5.8.3**: Type safety

### **State Management**:
- **Zustand 5.0.2**: Lightweight state management
- **Framer Motion 11.13.1**: Animations

### **Video Processing**:
- **@ffmpeg/ffmpeg 0.12.15**: Browser-based video processing
- **WaveSurfer.js 7.9.8**: Audio waveform visualization

### **UI Framework**:
- **Tailwind CSS 4.1.11**: Styling
- **Radix UI**: Accessible components
- **React Resizable Panels 2.1.7**: Resizable layout

### **Build Tools**:
- **Bun 1.2.18**: Package manager and runtime
- **Turbo 2.5.4**: Monorepo build system
- **Biome 2.1.2**: Linting and formatting

The timeline implementation is particularly sophisticated, handling complex video editing operations entirely in the browser while maintaining professional-grade features like ripple editing, multi-track management, and frame-accurate control.