# Fresh Editor Plugin System Architecture

## Overview

The Fresh editor implements a sophisticated plugin system that runs TypeScript/JavaScript plugins in a dedicated thread using deno_core as the runtime engine. This architecture enables asynchronous operations, persistent state, and safe interaction with the editor through a comprehensive API.

## Core Architecture

### Three-Tier Design

```
┌─────────────────────────────────────────────────────┐
│           Editor (Main Thread - UI)                 │
│  - Terminal rendering                              │
│  - Key/mouse input handling                        │
│  - Buffer management                               │
└──────────────────┬──────────────────────────────────┘
                   │
         ┌─────────┴──────────┐
         │ Channel-based IPC  │
         └─────────┬──────────┘
                   │
┌──────────────────▼──────────────────────────────────┐
│      Plugin Thread (Dedicated OS Thread)            │
│  - Owns JsRuntime (deno_core)                       │
│  - Persistent Tokio runtime for async ops          │
│  - Plugin execution sandbox                        │
└──────────────────┬──────────────────────────────────┘
                   │
         ┌─────────┴──────────┐
         │ Shared Structures  │
         │ (Arc<RwLock<...>>) │
         └────────────────────┘
```

## Key Components

### 1. Plugin Loading (`src/services/plugins/thread.rs`)

**Plugin Discovery:**
- Scans two directories automatically:
  - `{exe_directory}/plugins/` - Bundled plugins
  - `{working_directory}/plugins/` - Project-local plugins
- All `.ts` files are loaded and transpiled on-the-fly
- Plugins can be disabled via `--no-plugins` flag

**Initialization Flow:**
```rust
let ts_plugin_manager = if enable_plugins {
    PluginThreadHandle::spawn(commands)?
} else {
    tracing::info!("Plugins disabled via --no-plugins flag");
    None
};
```

**Plugin Thread Creation:**
- Spawns dedicated OS thread
- Creates persistent Tokio async runtime
- Sets up bidirectional communication channels:
  - Request channel: Editor → Plugin thread
  - Command channel: Plugin thread → Editor

### 2. TypeScript Runtime (`src/services/plugins/runtime.rs`)

**Runtime Components:**
- Uses `deno_core::JsRuntime` for JavaScript execution
- Custom `TypeScriptModuleLoader` for `.ts` file support
- `deno_ast` for TypeScript transpilation
- Supports native async/await (major improvement over Lua)

**Module Loading:**
- Plugins execute as ES modules
- Top-level code runs immediately during load
- Must register commands/hooks during initialization

### 3. Safe API Layer (`src/services/plugins/api.rs`)

**EditorStateSnapshot:**
- Read-only snapshot of editor state
- Updated on each editor loop iteration
- Contains:
  - Active buffer and split IDs
  - Buffer metadata (path, length, modified state)
  - Cursor positions (primary and multi-cursor)
  - Viewport information
  - Text properties (for virtual buffers)
  - Clipboard content
  - Working directory

**Global Editor Object:**
- Exposed as `editor` global in TypeScript
- Provides 70+ methods for editor interaction
- All operations channeled through safe API

### 4. Command Handling (`src/app/plugin_commands.rs`)

**Command Pipeline:**
```
Plugin TypeScript Code
        ↓
  Editor API Call (e.g., insertText)
        ↓
  Deno Op (Rust binding)
        ↓
  PluginCommand enum
        ↓
  Channel Send to Main Thread
        ↓
  Command Handler
        ↓
  Buffer/State Update
        ↓
  Response (if needed)
        ↓
  Promise Resolution in Plugin
```

**Command Categories:**
- Buffer operations: `InsertText`, `DeleteRange`
- Visual decorations: `AddOverlay`, `AddVirtualText`, `SetLineIndicator`
- Split management: `FocusSplit`, `SetSplitBuffer`, `CloseSplit`
- Virtual buffers: `CreateVirtualBuffer`, `SetVirtualBufferContent`
- Process execution: `SpawnProcess`, `KillProcess`
- LSP integration: `SendLspRequest`

## Plugin API

### A. Buffer Operations

```typescript
// Query buffer state
editor.getActiveBufferId() -> string
editor.getBufferInfo(bufferId) -> BufferInfo
editor.getBufferPath(bufferId) -> string
editor.getBufferLength(bufferId) -> number
editor.isBufferModified(bufferId) -> boolean

// Modify buffer content
editor.insertText(bufferId, position, text) -> void
editor.deleteRange(bufferId, start, end) -> void
```

### B. Cursor and Selection

```typescript
editor.getCursorPosition() -> number
editor.getAllCursorPositions() -> number[]
editor.getPrimaryCursor() -> { position: number, selection?: [number, number] }
editor.setBufferCursor(bufferId, position) -> void
editor.getCursorLine() -> number
```

### C. Visual Decorations (Overlays)

```typescript
// Syntax highlighting, error underlines, etc.
editor.addOverlay(options: {
  bufferId: string,
  range: [number, number],
  color: { r: number, g: number, b: number },
  namespace?: string
}) -> number  // Returns overlay handle

editor.removeOverlay(handle: number) -> void
editor.clearNamespace(namespace: string) -> void
editor.clearOverlaysInRange(bufferId, start, end) -> void
```

### D. Virtual Text and Lines

```typescript
// Inline annotations (like inline type hints)
editor.addVirtualText(options: {
  bufferId: string,
  position: number,
  text: string,
  color?: RGB,
  id?: string
}) -> void

// Full virtual lines (like git blame)
editor.addVirtualLine(options: {
  bufferId: string,
  afterLine: number,
  text: string,
  color?: RGB
}) -> void
```

### E. Line Indicators (Gutter)

```typescript
// Gutter symbols (breakpoints, git changes, etc.)
editor.setLineIndicator(options: {
  bufferId: string,
  line: number,
  text: string,
  color?: RGB,
  namespace?: string
}) -> void

editor.clearLineIndicators(namespace: string) -> void
```

### F. Virtual Buffers (Result Panels)

```typescript
// Create results panel with embedded metadata
await editor.createVirtualBufferInSplit(options: {
  title: string,
  content: string,
  textProperties?: Record<string, any>[],
  keybindings?: Record<string, string>
}) -> { bufferId: string, splitId: string }

// Query metadata at cursor
editor.getTextPropertiesAtCursor() -> any
```

### G. Commands and Modes

```typescript
// Register custom commands (invoked via :command)
editor.registerCommand(name: string, callback: string) -> void
editor.unregisterCommand(name: string) -> void

// Define buffer-specific keybindings
editor.defineMode(bufferId: string, keybindings: Record<string, string>) -> void
```

### H. Process Execution

```typescript
// Wait for process to complete
await editor.spawnProcess(
  command: string,
  args?: string[],
  cwd?: string
) -> { stdout: string, stderr: string, exitCode: number }

// Run background process
await editor.spawnBackgroundProcess(
  command: string,
  args?: string[],
  cwd?: string
) -> number  // Returns process ID

await editor.killProcess(processId: number) -> void
editor.isProcessRunning(processId: number) -> boolean
```

### I. File System

```typescript
await editor.readFile(path: string) -> string
await editor.writeFile(path: string, content: string) -> void
editor.fileExists(path: string) -> boolean
editor.fileStat(path: string) -> FileInfo
editor.readDir(path: string) -> DirEntry[]

// Path utilities
editor.pathJoin(...segments: string[]) -> string
editor.pathDirname(path: string) -> string
editor.pathBasename(path: string) -> string
editor.pathExtname(path: string) -> string
```

### J. LSP Integration

```typescript
// Send arbitrary LSP requests
await editor.sendLspRequest(
  language: string,
  method: string,
  params: any
) -> any

// Example: Switch between header/source in C++
await editor.sendLspRequest(
  "cpp",
  "textDocument/switchSourceHeader",
  { uri: currentFileUri }
)
```

### K. Prompts and Status

```typescript
editor.setStatus(message: string) -> void
editor.startPrompt(options: {
  prompt: string,
  defaultValue?: string
}) -> void
editor.setPromptSuggestions(suggestions: any[]) -> void
```

### L. Split Management

```typescript
editor.getActiveSplitId() -> string
editor.focusSplit(splitId: string) -> void
editor.setSplitBuffer(splitId: string, bufferId: string) -> void
editor.closeSplit(splitId: string) -> void
editor.setSplitRatio(splitId: string, ratio: number) -> void
editor.distributeSplitsEvenly() -> void
```

## Event System

### Available Hooks

**File Operations:**
- `before_file_open` - Before file is opened
- `after_file_open` - After file successfully opens
- `before_file_save` - Before saving to disk
- `after_file_save` - After successful save
- `buffer_closed` - Buffer is closed

**Text Editing:**
- `before_insert` - Before text insertion
- `after_insert` - After text inserted (with line info)
- `before_delete` - Before text deletion
- `after_delete` - After text deleted (with line info)

**Cursor and Selection:**
- `cursor_moved` - Cursor position changed
- `buffer_activated` - Buffer became active
- `buffer_deactivated` - Buffer lost focus

**Commands:**
- `pre_command` - Before action executes
- `post_command` - After action executes

**Rendering:**
- `render_start` - Before render pass (good for clearing overlays)
- `render_line` - Per-visible-line during render
- `lines_changed` - Batched efficient hook for line processing

**Prompts:**
- `prompt_changed` - User typed in prompt
- `prompt_confirmed` - User pressed Enter
- `prompt_cancelled` - User pressed Escape
- `prompt_selection_changed` - Selection moved in suggestions

**System:**
- `editor_initialized` - Editor startup complete
- `idle` - No input for N milliseconds

### Hook Registration

```typescript
// 1. Define handler as global function
globalThis.onFileSave = (data: { path: string, bufferId: string }) => {
  editor.setStatus(`Saved: ${data.path}`);
};

// 2. Register with event system
editor.on("after_file_save", "onFileSave");

// 3. Unregister when done
editor.off("after_file_save", "onFileSave");
```

## Plugin Examples

### Hello World Plugin

**Location:** `plugins/examples/hello_world.ts`

```typescript
// Register a simple command
globalThis.helloCommand = () => {
  const bufferId = editor.getActiveBufferId();
  const path = editor.getBufferPath(bufferId);
  const length = editor.getBufferLength(bufferId);

  editor.setStatus(`Hello! Editing ${path} (${length} bytes)`);

  // Highlight first 10 characters
  editor.addOverlay({
    bufferId,
    range: [0, 10],
    color: { r: 255, g: 100, b: 100 },
    namespace: "hello"
  });
};

editor.registerCommand("hello", "helloCommand");
```

### Async Process Plugin

**Location:** `plugins/examples/async_demo.ts`

```typescript
globalThis.gitStatusCommand = async () => {
  const bufferId = editor.getActiveBufferId();

  try {
    const result = await editor.spawnProcess("git", ["status"]);
    editor.setStatus(result.stdout.split('\n')[0]);
  } catch (error) {
    editor.setStatus(`Error: ${error.message}`);
  }
};

editor.registerCommand("git-status", "gitStatusCommand");
```

### Git Grep Plugin

**Location:** `plugins/git_grep.ts`

```typescript
// Interactive search with live results
globalThis.startGitGrep = () => {
  editor.startPrompt({ prompt: "Git grep: " });
};

globalThis.onPromptChange = async (data: { text: string }) => {
  if (!data.text) return;

  const result = await editor.spawnProcess("git", ["grep", "-n", data.text]);

  const suggestions = result.stdout
    .split('\n')
    .filter(line => line)
    .map(line => {
      const [file, lineNum, ...rest] = line.split(':');
      return {
        display: line,
        file,
        line: parseInt(lineNum)
      };
    });

  editor.setPromptSuggestions(suggestions);
};

globalThis.onPromptConfirm = (data: { selection: any }) => {
  if (data.selection) {
    editor.openFile(data.selection.file, data.selection.line, 0);
  }
};

editor.registerCommand("git-grep", "startGitGrep");
editor.on("prompt_changed", "onPromptChange");
editor.on("prompt_confirmed", "onPromptConfirm");
```

## Plugin Utility Libraries

**Location:** `plugins/lib/`

### PanelManager

Manages result panel lifecycle:

```typescript
import { PanelManager } from "./lib/panel_manager.ts";

const panel = new PanelManager("My Results", "initial content");
await panel.open();
panel.update("new content");
panel.toggle();  // Close if open, open if closed
```

### NavigationController

Handles list selection with visual highlighting:

```typescript
import { NavigationController } from "./lib/navigation_controller.ts";

const nav = new NavigationController(bufferId, items);
nav.moveDown();
nav.moveUp();
const selected = nav.getSelected();
```

### VirtualBufferFactory

Simplified virtual buffer creation:

```typescript
import { VirtualBufferFactory } from "./lib/virtual_buffer_factory.ts";

const buffer = await VirtualBufferFactory.createInNewSplit({
  title: "Results",
  content: "...",
  keybindings: { "Enter": "selectItem" }
});
```

## Architectural Decisions

### Why a Dedicated Thread?

1. **Persistent State:** JavaScript runtime stays alive across commands
2. **Async Support:** Tokio runtime enables true async/await
3. **UI Responsiveness:** Long-running operations don't block rendering
4. **Memory Isolation:** V8 heap separate from main editor

### Why TypeScript?

1. **Type Safety:** Catch errors at development time
2. **IDE Support:** Autocomplete, go-to-definition, refactoring
3. **Developer Familiarity:** Web developers already know TypeScript
4. **Modern Syntax:** async/await, destructuring, arrow functions

### Why deno_core?

1. **Embeddable:** Can be used as a library in Rust applications
2. **Modern JS:** Full ES2024 support with async/await
3. **TypeScript Native:** Built-in transpilation via deno_ast
4. **Performance:** V8 engine is highly optimized
5. **Safety:** Sandboxed execution with custom ops only

### API Safety Model

1. **No Direct Access:** Plugins can't access internal editor structures
2. **Command-Based:** All mutations go through typed command enum
3. **Read-Only Snapshots:** State queries use immutable snapshots
4. **Channel Isolation:** Thread boundaries enforce separation

## File Organization

```
src/
├── services/plugins/
│   ├── mod.rs              # Module organization
│   ├── thread.rs           # Plugin thread management
│   ├── runtime.rs          # TypeScript runtime (3,300+ lines)
│   ├── api.rs              # Safe editor API definitions
│   ├── hooks.rs            # Event subscription system
│   ├── event_hooks.rs      # Event hook implementations
│   └── process.rs          # Process spawning functionality
├── app/
│   ├── plugin_commands.rs  # Command handlers (1,000+ lines)
│   └── script_control.rs   # Script testing mode
└── v8_init.rs              # V8 platform initialization

plugins/
├── git_grep.ts             # Git search integration
├── diagnostics.ts          # LSP error display
├── markdown.ts             # Markdown editing
├── examples/
│   ├── hello_world.ts      # Basic example
│   └── async_demo.ts       # Async operations example
└── lib/
    ├── panel_manager.ts    # Result panel utilities
    ├── navigation_controller.ts
    └── virtual_buffer_factory.ts
```

## Performance Considerations

### Efficient Communication

- Commands batched when possible
- State snapshots reused across multiple queries
- Overlay operations use namespaces for bulk removal

### Render Hooks

- `render_line` called per visible line (not entire buffer)
- `lines_changed` provides batched updates
- Overlays cached by renderer

### Async Best Practices

- Use `spawnBackgroundProcess` for long-running tasks
- Process results in chunks for large outputs
- Cancel operations when buffer closed

## Security

### Sandboxing

- No access to filesystem except through API
- No network access
- No arbitrary Deno APIs
- Only Fresh-provided ops available

### Plugin Isolation

- Plugins can't directly call other plugins
- Global scope isolated per execution
- No shared mutable state between plugins

## Summary

The Fresh editor's plugin system represents a production-ready architecture that:

1. Runs TypeScript plugins in a **dedicated thread** with deno_core
2. Provides **70+ API methods** for comprehensive editor control
3. Supports **native async/await** for asynchronous operations
4. Enables **event-driven development** via 20+ hooks
5. Maintains **UI responsiveness** through channel-based IPC
6. Ensures **safety** through API boundaries and command validation
7. Offers **rich visual capabilities** with overlays, virtual text, and custom rendering
8. Includes **production plugins** for git, LSP, markdown, and more

The system successfully solves the limitations of previous Lua-based architectures while maintaining simplicity and performance.
