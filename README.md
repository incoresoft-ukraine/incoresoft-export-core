# Export Core

A JavaScript/TypeScript library for exporting video recordings from streaming servers. Supports chunked downloads, progress tracking, and multiple export formats (MKV, AVI).

## Features

- Automatic video splitting into hourly chunks for reliable downloads
- Progress tracking for individual chunks and overall export
- File System Access API support (saves directly to disk)
- Fallback to blob downloads for unsupported browsers
- Binary streaming protocol support for efficient data transfer
- Vue 3 composables and components included
- TypeScript support with full type definitions

## Installation

### Option 1: Standalone Demo (index.html)

Download `index.html` from the [Releases](https://github.com/incoresoft-ukraine/incoresoft-export-core/releases) page and open it directly in your browser. This is a fully self-contained demo application.

**Usage:**

1. Download `index.html`
2. Open it in a modern browser (Chrome, Edge, Firefox)
3. Fill in the export parameters:
   - **Server URL**: Base URL of your video server
   - **Auth Token**: Bearer token for API authentication
   - **Stream UUID**: UUID of the camera stream to export
   - **Format**: Choose MKV or AVI
   - **Time Range**: Select start and end date/time
4. Click "Export" to start downloading

### Option 2: Include in Your Project

Download the library files from [Releases](https://github.com/user/repo/releases):

- `incoresoft-export-core.umd.js` - For `<script>` tag usage
- `incoresoft-export-core.es.js` - For ES modules
- `style.css` - Required styles (optional, only if using Vue components)

#### Using Script Tag (UMD)

```html
<script src="path/to/incoresoft-export-core.umd.js"></script>
<script>
  const { createExportEngine, EExportFormat } = IncoresoftExportCore;

  const engine = createExportEngine({
    getExportLink: async (payload) => {
      const response = await fetch(`/api/export?stream=${payload.stream_uuid}&start=${payload.start_date}&end=${payload.end_date}`);
      const data = await response.json();
      return { link: data.link };
    },
    format: EExportFormat.MKV
  });
</script>
```

#### Using ES Modules

```javascript
import {
  createExportEngine,
  EExportFormat,
  EChunkStatus
} from './incoresoft-export-core.es.js';

const engine = createExportEngine({
  getExportLink: async (payload) => {
    return { link: 'https://...' };
  }
});
```

---

## API Reference

### Creating the Export Engine

```javascript
import { createExportEngine, EExportFormat } from '@incoresoft/export-core';

const engine = createExportEngine({
  // Required: Function to get download link from your server
  getExportLink: async (payload) => {
    // payload contains:
    // - stream_uuid: string
    // - format: 'avi' | 'mkv'
    // - start_date: number (timestamp in milliseconds)
    // - end_date: number (timestamp in milliseconds)
    // - with_audio: boolean

    const response = await fetch('/api/export-link', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': 'Bearer YOUR_TOKEN'
      },
      body: JSON.stringify(payload)
    });

    const data = await response.json();
    return { link: data.downloadUrl };
  },

  // Optional: Export format (default: MKV)
  format: EExportFormat.MKV,

  // Optional: Minimum chunk duration in ms (default: 120000 = 2 minutes)
  minChunkDuration: 60000,

  // Optional: Show browser warning when closing during export (default: true)
  enableBeforeUnloadWarning: true,

  // Optional: Callbacks
  onExportSuccess: () => {
    console.log('Export completed successfully!');
  },
  onChunkError: (errorCode) => {
    console.error('Chunk failed:', errorCode);
  }
});
```

### Adding Items to Export Queue

```javascript
// Add a single export item
engine.addItem({
  id: 'export-1',                              // Unique identifier
  camera_name: 'Front Door Camera',            // Display name
  camera_uuid: 'cam-uuid-123',                 // Camera UUID
  camera_id: 1,                                // Camera ID
  stream_uuid: 'stream-uuid-456',              // Stream UUID
  start_time: Date.now() - 3600000,            // Start time (1 hour ago)
  end_time: Date.now(),                        // End time (now)
  with_audio: true                             // Include audio track
});

// Add multiple items at once
engine.addItems([item1, item2, item3]);
```

### Setting Event Callbacks

```javascript
engine.setCallbacks({
  // Called when a chunk's status changes
  onChunkStatusChange: (chunkId, status, error, errorCode) => {
    console.log(`Chunk ${chunkId}: ${status}`);
    // status: 'ready' | 'waiting' | 'in-process' | 'downloaded' | 'errored'
  },

  // Called during chunk download with progress percentage
  onChunkProgress: (chunkId, progress) => {
    console.log(`Chunk ${chunkId}: ${progress}%`);
  },

  // Called when all chunks are processed
  onExportComplete: () => {
    console.log('Export finished!');
  },

  // Called when export is cancelled by user
  onExportCancelled: () => {
    console.log('Export was cancelled');
  },

  // Called when items list changes
  onItemsChange: () => {
    const items = engine.getItems();
    console.log('Items updated:', items.length);
  }
});
```

### Starting Export

```javascript
// Start export for all chunks
await engine.startExport();

// Start export for a single chunk
await engine.startChunkExport('chunk-id');

// Start export for specific chunks
await engine.startChunksExport(['chunk-1', 'chunk-2']);
```

### Cancelling Export

```javascript
// Cancel immediately
engine.cancel();

// Cancel with confirmation callback
const cancelled = await engine.cancelWithConfirmation();
```

### Managing Export Queue

```javascript
// Get all items
const items = engine.getItems();

// Get item by ID
const item = engine.getItemById('export-1');

// Remove item
engine.removeItem('export-1');

// Remove with confirmation
const removed = await engine.removeItemWithConfirmation('export-1');

// Clear all items
engine.clearAll();

// Clear with confirmation
const cleared = await engine.clearAllWithConfirmation();
```

### Working with Chunks

```javascript
// Get all chunks (flattened from all items)
const allChunks = engine.getAllChunks();

// Get chunks for specific item
const itemChunks = engine.getChunksForItem('export-1');

// Get current state
const state = engine.getState();
// Returns: { isRunning, currentChunkId, progress, format }
```

---

## Enums and Constants

### Chunk Status

```javascript
import { EChunkStatus } from '@incoresoft/export-core';

EChunkStatus.READY        // Initial state, ready to export
EChunkStatus.WAITING      // In queue, waiting to be processed
EChunkStatus.IN_PROCESS   // Currently downloading
EChunkStatus.DOWNLOADED   // Successfully completed
EChunkStatus.ERRORED      // Download failed
```

### Export Format

```javascript
import { EExportFormat } from '@incoresoft/export-core';

EExportFormat.MKV  // Matroska format
EExportFormat.AVI  // AVI format
```

### Error Codes

```javascript
import { EExportErrorCode } from '@incoresoft/export-core';

EExportErrorCode.EXPORT_IN_PROGRESS    // Export already running
EExportErrorCode.ITEM_NOT_FOUND        // Item ID not found
EExportErrorCode.CHUNK_NOT_FOUND       // Chunk ID not found
EExportErrorCode.DOWNLOAD_FAILED       // Download failed
EExportErrorCode.INVALID_TIME_RANGE    // Invalid time range
EExportErrorCode.LINK_REQUEST_FAILED   // API request failed
EExportErrorCode.FILE_SYSTEM_ERROR     // File system access error
EExportErrorCode.NETWORK_ERROR         // Network error
EExportErrorCode.CANCELLED             // Export was cancelled
```

---

## Vue 3 Integration

The library includes Vue 3 composables and components.

### Using the Composable

```vue
<script setup>
import { useExportEngine, provideExportEngine } from '@incoresoft/export-core';

const {
  engine,
  items,
  chunks,
  isExporting,
  progress,
  addItem,
  startExport,
  cancel
} = useExportEngine({
  getExportLink: async (payload) => {
    // Your implementation
    return { link: '...' };
  }
});

// Provide to child components
provideExportEngine(engine);
</script>
```

### Using Components

```vue
<template>
  <ExportItemList :items="items" @remove="removeItem" />
  <ExportChunkList :chunks="chunks" />
  <ExportFormatSelect v-model="format" />
</template>

<script setup>
import {
  ExportItemList,
  ExportChunkList,
  ExportFormatSelect
} from '@incoresoft/export-core';
import '@incoresoft/export-core/style.css';
</script>
```

---

## Utility Functions

```javascript
import {
  calculateChunks,
  flattenChunks,
  formatDuration,
  generateFilename,
  getTotalDuration,
  areAllChunksCompleted,
  DEFAULT_MIN_CHUNK_DURATION
} from '@incoresoft/export-core';

// Format duration (ms to "HH:MM:SS")
const formatted = formatDuration(3661000);
// Result: "01:01:01"

// Generate export filename
const filename = generateFilename(
  'Camera 1',           // Camera name
  1699900000000,        // Start timestamp
  1699903600000,        // End timestamp
  'mkv'                 // Format
);
// Result: "Camera_1_2023-11-13_12-00-00_to_13-00-00.mkv"
```

---

## Complete Example

```javascript
import { createExportEngine, EExportFormat } from '@incoresoft/export-core';

// 1. Create engine with your API integration
const engine = createExportEngine({
  getExportLink: async (payload) => {
    const params = new URLSearchParams({
      format: payload.format,
      start_date: payload.start_date.toString(),
      end_date: payload.end_date.toString(),
      with_audio: payload.with_audio.toString()
    });

    const response = await fetch(
      `https://your-server.com/api/streams/${payload.stream_uuid}/export?${params}`,
      {
        headers: { 'Authorization': 'Bearer YOUR_TOKEN' }
      }
    );

    const data = await response.json();
    return { link: data.link };
  },
  format: EExportFormat.MKV,
  onExportSuccess: () => alert('Export completed!'),
  onChunkError: (error) => console.error('Error:', error)
});

// 2. Set up progress tracking
engine.setCallbacks({
  onChunkProgress: (chunkId, progress) => {
    document.getElementById('progress').textContent = `${progress}%`;
  },
  onChunkStatusChange: (chunkId, status) => {
    console.log(`Chunk ${chunkId} is now ${status}`);
  },
  onExportComplete: () => {
    document.getElementById('status').textContent = 'Done!';
  }
});

// 3. Add items to export
engine.addItem({
  id: 'video-1',
  camera_name: 'Lobby Camera',
  camera_uuid: 'abc-123',
  camera_id: 1,
  stream_uuid: 'stream-xyz',
  start_time: new Date('2024-01-15T10:00:00').getTime(),
  end_time: new Date('2024-01-15T12:00:00').getTime(),
  with_audio: true
});

// 4. Start export
document.getElementById('exportBtn').onclick = async () => {
  try {
    await engine.startExport();
  } catch (error) {
    console.error('Export failed:', error);
  }
};

// 5. Cancel if needed
document.getElementById('cancelBtn').onclick = () => {
  engine.cancel();
};
```

---

## Browser Support

| Browser | Support Level |
|---------|--------------|
| Chrome 86+ | Full (File System Access API) |
| Edge 86+ | Full (File System Access API) |
| Firefox | Fallback (blob downloads) |
| Safari | Fallback (blob downloads) |

---

## How It Works

1. **Chunking**: Long recordings are automatically split into 1-hour chunks for reliable downloads
2. **Queue Processing**: Chunks are processed sequentially to prevent server overload
3. **Progress Tracking**: Real-time progress updates for each chunk and overall export
4. **File Saving**: Uses File System Access API when available (direct disk write), falls back to blob downloads
5. **Error Recovery**: Failed chunks can be retried individually without re-downloading successful ones

---

## License

MIT

