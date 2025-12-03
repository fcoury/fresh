# Fresh Editor - Deno Integration Deep Dive

## Overview

Fresh uses **deno_core** (Deno's embeddable JavaScript/TypeScript runtime) as the execution engine for its plugin system. This provides modern JavaScript features, native async/await support, and TypeScript transpilation without requiring an external compiler.

## Core Integration Architecture

### Runtime Initialization

**Location:** `src/services/plugins/runtime.rs:2721-2743`

```rust
pub fn with_state_and_responses(
    state_snapshot: Arc<RwLock<EditorStateSnapshot>>,
    command_sender: std::sync::mpsc::Sender<PluginCommand>,
    pending_responses: PendingResponses,
) -> Result<Self> {
    // 1. Initialize V8 platform (once per process)
    crate::v8_init::init();

    // 2. Create event handlers registry
    let event_handlers = Rc::new(RefCell::new(HashMap::new()));

    // 3. Create runtime state
    let runtime_state = Rc::new(RefCell::new(TsRuntimeState {
        state_snapshot,
        command_sender,
        event_handlers: event_handlers.clone(),
        pending_responses: Arc::clone(&pending_responses),
        next_request_id: Rc::new(RefCell::new(1)),
        background_processes: Rc::new(RefCell::new(HashMap::new())),
        next_process_id: Rc::new(RefCell::new(1)),
    }));

    // 4. Create JsRuntime with custom extension
    let mut js_runtime = JsRuntime::new(RuntimeOptions {
        module_loader: Some(Rc::new(TypeScriptModuleLoader)),
        extensions: vec![fresh_runtime::init()],
        ..Default::default()
    });

    // 5. Store runtime state in OpState
    js_runtime.op_state().borrow_mut().put(runtime_state);

    Ok(Self {
        js_runtime,
        event_handlers,
        pending_responses,
    })
}
```

### V8 Platform Setup

**Location:** `src/v8_init.rs`

```rust
use std::sync::Once;

static INIT: Once = Once::new();

pub fn init() {
    INIT.call_once(|| {
        // Initialize V8 platform (once per process)
        deno_core::JsRuntime::init_platform(None, false);
    });
}
```

The V8 platform is initialized once per process to avoid multiple initialization errors.

## TypeScript Transpilation

### Custom Module Loader

**Location:** `src/services/plugins/runtime.rs:62-138`

```rust
struct TypeScriptModuleLoader;

impl deno_core::ModuleLoader for TypeScriptModuleLoader {
    fn resolve(
        &self,
        specifier: &str,
        referrer: &str,
        _kind: ResolutionKind
    ) -> Result<ModuleSpecifier, ModuleLoaderError>
    {
        deno_core::resolve_import(specifier, referrer)
            .map_err(|e| JsErrorBox::generic(e.to_string()))
    }

    fn load(
        &self,
        module_specifier: &ModuleSpecifier,
        _maybe_referrer: Option<&ModuleSpecifier>,
        _is_dynamic: bool,
        _requested_module_type: RequestedModuleType,
    ) -> ModuleLoadResponse {
        let specifier = module_specifier.clone();

        // Read file from filesystem
        let path = match specifier.to_file_path() {
            Ok(p) => p,
            Err(_) => return ModuleLoadResponse::Sync(Err(
                JsErrorBox::generic(format!("Invalid file path: {}", specifier))
            )),
        };

        let code = match std::fs::read_to_string(&path) {
            Ok(c) => c,
            Err(e) => return ModuleLoadResponse::Sync(Err(
                JsErrorBox::generic(format!("Failed to read file: {}", e))
            )),
        };

        // Check file extension and transpile .ts files
        let (code, module_type) = match path.extension().and_then(|s| s.to_str()) {
            Some("ts") => {
                match transpile_typescript(&code, &specifier) {
                    Ok(transpiled) => (transpiled, ModuleType::JavaScript),
                    Err(e) => return ModuleLoadResponse::Sync(Err(e)),
                }
            }
            _ => (code, ModuleType::JavaScript),
        };

        let module_source = ModuleSource::new(
            module_type,
            ModuleSourceCode::String(code.into()),
            &specifier,
            None,
        );

        ModuleLoadResponse::Sync(Ok(module_source))
    }
}
```

### TypeScript Transpilation Function

```rust
fn transpile_typescript(
    source: &str,
    specifier: &ModuleSpecifier
) -> Result<String, JsErrorBox>
{
    use deno_ast::{parse_module, ParseParams, MediaType, TranspileOptions, EmitOptions};

    // Parse TypeScript source
    let parsed = parse_module(ParseParams {
        specifier: specifier.clone(),
        text: source.into(),
        media_type: MediaType::TypeScript,
        capture_tokens: false,
        scope_analysis: false,
        maybe_syntax: None,
    }).map_err(|e| JsErrorBox::generic(format!("Parse error: {}", e)))?;

    // Transpile to JavaScript
    let transpiled = parsed.transpile(
        &TranspileOptions::default(),
        &Default::default(),
        &EmitOptions::default(),
    ).map_err(|e| JsErrorBox::generic(format!("Transpile error: {}", e)))?;

    Ok(transpiled.into_source().text)
}
```

**Key Features:**
- Uses `deno_ast` crate for parsing and transpilation
- Handles both `.ts` and `.js` files
- No external TypeScript compiler needed
- Transpilation happens at load time
- Errors are properly propagated to JavaScript

## Deno Ops (Rust-to-JavaScript Bindings)

### Extension Definition

**Location:** `src/services/plugins/runtime.rs:2598-2680`

Fresh defines a custom Deno extension with **82 ops**:

```rust
#[op2]
use deno_core::{extension, op2, OpState, JsBuffer};

extension!(
    fresh_runtime,
    ops = [
        // Status & Logging (2 ops)
        op_fresh_set_status,
        op_fresh_debug,

        // Clipboard (1 op)
        op_fresh_set_clipboard,

        // Buffer Operations - Read (8 ops)
        op_fresh_get_active_buffer_id,
        op_fresh_get_cursor_position,
        op_fresh_get_buffer_path,
        op_fresh_get_buffer_length,
        op_fresh_is_buffer_modified,
        op_fresh_get_buffer_saved_diff,
        op_fresh_get_buffer_info,
        op_fresh_list_buffers,

        // Buffer Operations - Write (2 ops)
        op_fresh_insert_text,
        op_fresh_delete_range,

        // Overlay/Decoration (5 ops)
        op_fresh_add_overlay,
        op_fresh_remove_overlay,
        op_fresh_clear_namespace,
        op_fresh_clear_overlays_in_range,
        op_fresh_clear_all_overlays,

        // Virtual Text (8 ops)
        op_fresh_add_virtual_text,
        op_fresh_remove_virtual_text,
        op_fresh_remove_virtual_texts_by_prefix,
        op_fresh_clear_virtual_texts,
        op_fresh_add_virtual_line,
        op_fresh_remove_virtual_line,
        op_fresh_clear_virtual_lines,
        op_fresh_clear_virtual_text_namespace,

        // View Transforms (2 ops)
        op_fresh_submit_view_transform,
        op_fresh_clear_view_transform,

        // Line Indicators (4 ops)
        op_fresh_set_line_indicator,
        op_fresh_clear_line_indicators,
        op_fresh_set_line_numbers,
        op_fresh_refresh_lines,

        // Cursor Operations (7 ops)
        op_fresh_insert_at_cursor,
        op_fresh_get_cursor_line,
        op_fresh_get_all_cursor_positions,
        op_fresh_get_primary_cursor,
        op_fresh_get_all_cursors,
        op_fresh_get_viewport,
        op_fresh_set_buffer_cursor,

        // Commands (2 ops)
        op_fresh_register_command,
        op_fresh_unregister_command,

        // File Operations (2 ops)
        op_fresh_open_file,
        op_fresh_open_file_in_split,

        // Split Management (6 ops)
        op_fresh_get_active_split_id,
        op_fresh_focus_split,
        op_fresh_set_split_buffer,
        op_fresh_close_split,
        op_fresh_set_split_ratio,
        op_fresh_distribute_splits_evenly,

        // Process Execution (4 ops - async)
        op_fresh_spawn_process,
        op_fresh_spawn_background_process,
        op_fresh_kill_process,
        op_fresh_is_process_running,

        // LSP (1 op - async)
        op_fresh_send_lsp_request,

        // File System (5 ops - some async)
        op_fresh_read_file,
        op_fresh_write_file,
        op_fresh_file_exists,
        op_fresh_file_stat,
        op_fresh_read_dir,

        // Path Utilities (5 ops)
        op_fresh_path_join,
        op_fresh_path_dirname,
        op_fresh_path_basename,
        op_fresh_path_extname,
        op_fresh_path_is_absolute,

        // Environment (2 ops)
        op_fresh_get_env,
        op_fresh_get_cwd,

        // Event System (3 ops)
        op_fresh_on,
        op_fresh_off,
        op_fresh_get_handlers,

        // Prompts (2 ops)
        op_fresh_start_prompt,
        op_fresh_set_prompt_suggestions,

        // Virtual Buffers (8 ops - some async)
        op_fresh_create_virtual_buffer_in_split,
        op_fresh_create_virtual_buffer_in_existing_split,
        op_fresh_create_virtual_buffer,
        op_fresh_define_mode,
        op_fresh_show_buffer,
        op_fresh_close_buffer,
        op_fresh_set_virtual_buffer_content,
        op_fresh_get_text_properties_at_cursor,

        // Layout (1 op)
        op_fresh_set_layout_hints,
    ],
);
```

### Op Implementation Examples

#### Synchronous Op (Fast Path)

```rust
#[op2(fast)]
fn op_fresh_get_active_buffer_id(state: &mut OpState) -> Result<u64, JsErrorBox> {
    let runtime_state = state.borrow::<Rc<RefCell<TsRuntimeState>>>();
    let runtime_state = runtime_state.borrow();
    let snapshot = runtime_state.state_snapshot.read().unwrap();

    Ok(snapshot.active_buffer_id)
}
```

**Fast Ops:**
- Use `#[op2(fast)]` for maximum performance
- No serialization overhead
- Return simple types (numbers, booleans)

#### Synchronous Op with Serialization

```rust
#[op2]
#[serde]
fn op_fresh_get_buffer_info(
    state: &mut OpState,
    #[bigint] buffer_id: u64
) -> Result<Option<BufferInfo>, JsErrorBox>
{
    let runtime_state = state.borrow::<Rc<RefCell<TsRuntimeState>>>();
    let runtime_state = runtime_state.borrow();
    let snapshot = runtime_state.state_snapshot.read().unwrap();

    Ok(snapshot.buffers.get(&buffer_id).cloned())
}
```

**Serialized Ops:**
- Use `#[serde]` attribute for return value
- Can return complex types (structs, enums)
- Automatic JSON serialization

#### Asynchronous Op

```rust
#[op2(async)]
#[serde]
async fn op_fresh_spawn_process(
    #[string] command: String,
    #[serde] args: Vec<String>,
    #[string] cwd: Option<String>,
) -> Result<SpawnResult, JsErrorBox>
{
    use tokio::process::Command;

    // Check for tokio runtime context
    if tokio::runtime::Handle::try_current().is_err() {
        return Err(JsErrorBox::generic(
            "spawnProcess requires an async runtime context (tokio)"
        ));
    }

    let mut cmd = Command::new(&command);
    cmd.args(&args);

    if let Some(dir) = cwd {
        cmd.current_dir(dir);
    }

    let output = cmd.output().await.map_err(|e|
        JsErrorBox::generic(format!("Failed to spawn process: {}", e))
    )?;

    Ok(SpawnResult {
        stdout: String::from_utf8_lossy(&output.stdout).to_string(),
        stderr: String::from_utf8_lossy(&output.stderr).to_string(),
        exit_code: output.status.code(),
    })
}

#[derive(serde::Serialize)]
struct SpawnResult {
    stdout: String,
    stderr: String,
    exit_code: Option<i32>,
}
```

**Async Ops:**
- Use `#[op2(async)]` attribute
- Return `Future` (automatically handled by deno_core)
- Integrate with Tokio runtime
- JavaScript sees these as Promises

### Op Decorators Reference

```rust
// Fast synchronous op (no serialization)
#[op2(fast)]
fn op_name(state: &mut OpState, x: i32) -> i32 { }

// Synchronous op with serialization
#[op2]
#[serde]
fn op_name(state: &mut OpState, #[serde] data: MyStruct) -> Result<ReturnType, JsErrorBox> { }

// Asynchronous op
#[op2(async)]
async fn op_name(#[string] text: String) -> Result<String, JsErrorBox> { }

// Parameter decorators:
#[string]         // String parameter
#[serde]          // JSON-serializable parameter
#[bigint]         // BigInt/u64 parameter
#[buffer]         // Binary buffer
```

## Global Editor API Bootstrap

### JavaScript API Wrapper

**Location:** `src/services/plugins/runtime.rs:2748-3040`

After creating the JsRuntime, Fresh executes a bootstrap script that creates the global `editor` object:

```javascript
const core = Deno.core;

const editor = {
    // Status and Logging
    setStatus(message) {
        core.ops.op_fresh_set_status(message);
    },
    debug(message) {
        core.ops.op_fresh_debug(message);
    },

    // Clipboard
    copyToClipboard(text) {
        core.ops.op_fresh_set_clipboard(text);
    },

    // Buffer Queries
    getActiveBufferId() {
        return core.ops.op_fresh_get_active_buffer_id();
    },
    getCursorPosition() {
        return core.ops.op_fresh_get_cursor_position();
    },
    getBufferPath(bufferId) {
        return core.ops.op_fresh_get_buffer_path(bufferId);
    },
    getBufferLength(bufferId) {
        return core.ops.op_fresh_get_buffer_length(bufferId);
    },
    isBufferModified(bufferId) {
        return core.ops.op_fresh_is_buffer_modified(bufferId);
    },
    getBufferInfo(bufferId) {
        return core.ops.op_fresh_get_buffer_info(bufferId);
    },

    // Buffer Mutations
    insertText(bufferId, position, text) {
        return core.ops.op_fresh_insert_text(bufferId, position, text);
    },
    deleteRange(bufferId, start, end) {
        return core.ops.op_fresh_delete_range(bufferId, start, end);
    },

    // Async Operations (return Promises)
    async spawnProcess(command, args = [], cwd = null) {
        return await core.ops.op_fresh_spawn_process(command, args, cwd);
    },
    async spawnBackgroundProcess(command, args = [], cwd = null) {
        return await core.ops.op_fresh_spawn_background_process(command, args, cwd);
    },
    async killProcess(processId) {
        return await core.ops.op_fresh_kill_process(processId);
    },

    // File System (async)
    async readFile(path) {
        return await core.ops.op_fresh_read_file(path);
    },
    async writeFile(path, content) {
        return await core.ops.op_fresh_write_file(path, content);
    },

    // LSP Integration (async)
    async sendLspRequest(language, method, params) {
        return await core.ops.op_fresh_send_lsp_request(language, method, params);
    },

    // Event System
    on(eventName, handlerName) {
        return core.ops.op_fresh_on(eventName, handlerName);
    },
    off(eventName, handlerName) {
        return core.ops.op_fresh_off(eventName, handlerName);
    },

    // ... 60+ more methods
};

globalThis.editor = editor;

// Event dispatcher for hook system
globalThis.__eventDispatcher = async function(handlerName, eventData) {
    const handler = globalThis[handlerName];
    if (typeof handler === 'function') {
        try {
            const result = handler(eventData);
            // Handle both sync and async handlers
            const finalResult = (result instanceof Promise) ? await result : result;
            return finalResult !== false;
        } catch (error) {
            console.error(`Error in handler ${handlerName}:`, error);
            return false;
        }
    }
    return true;
};
```

This bootstrap code runs once during runtime initialization.

## Module Loading System

### Loading Plugins

**Location:** `src/services/plugins/runtime.rs:3115-3167`

```rust
pub async fn load_module(&mut self, path: &str) -> Result<()> {
    self.load_module_with_source(path, "").await
}

pub async fn load_module_with_source(
    &mut self,
    path: &str,
    plugin_source: &str
) -> Result<()> {
    // Set plugin source as global for command registration context
    let set_source = format!(
        "globalThis.__PLUGIN_SOURCE__ = {};",
        if plugin_source.is_empty() {
            "null"
        } else {
            format!("\"{}\"", plugin_source)
        }
    );

    self.js_runtime.execute_script("<set_plugin_source>", set_source)?;

    // Resolve file path to module specifier
    let module_specifier = deno_core::resolve_path(
        path,
        &std::env::current_dir()?
    )?;

    // Load as side ES module (allows multiple modules without conflicts)
    let mod_id = self.js_runtime
        .load_side_es_module(&module_specifier)
        .await?;

    // Evaluate the module
    let result = self.js_runtime.mod_evaluate(mod_id);

    // Run event loop to process any async initialization
    self.js_runtime.run_event_loop(Default::default()).await?;

    // Wait for module evaluation to complete
    result.await?;

    // Clear plugin source context
    let clear_source = "globalThis.__PLUGIN_SOURCE__ = null;".to_string();
    self.js_runtime.execute_script("<clear_plugin_source>", clear_source)?;

    Ok(())
}
```

**Key Points:**
- Uses `load_side_es_module` to allow multiple plugins
- Sets global context (`__PLUGIN_SOURCE__`) for tracking
- Runs event loop after loading for async initialization
- Cleans up context after module loads

### Event Loop Management

```rust
pub async fn execute_script(
    &mut self,
    name: &'static str,
    code: &str
) -> Result<()> {
    // Execute synchronous code
    self.js_runtime.execute_script(name, code)?;

    // Process all pending async operations (Promises, timers, etc.)
    self.js_runtime.run_event_loop(Default::default()).await?;

    Ok(())
}
```

The event loop must be explicitly run after executing code to process:
- Promise resolutions
- Async op completions
- Microtasks
- Any other V8 async operations

## Async Communication Patterns

### Request-Response Pattern

**Location:** `src/services/plugins/runtime.rs:2007-2063`

For operations that need editor responses (like creating buffers), Fresh uses a request-response pattern:

```rust
#[op2(async)]
#[serde]
async fn op_fresh_create_virtual_buffer_in_split(
    state: Rc<RefCell<OpState>>,
    #[string] title: String,
    #[string] content: String,
    // ... other parameters
) -> Result<CreateVirtualBufferResult, JsErrorBox>
{
    let (command_sender, pending_responses, request_id) = {
        let state_borrow = state.borrow();
        let runtime_state = state_borrow.borrow::<Rc<RefCell<TsRuntimeState>>>();
        let runtime_state = runtime_state.borrow();

        let request_id = {
            let mut next_id = runtime_state.next_request_id.borrow_mut();
            let id = *next_id;
            *next_id += 1;
            id
        };

        (
            runtime_state.command_sender.clone(),
            Arc::clone(&runtime_state.pending_responses),
            request_id
        )
    };

    // Create oneshot channel for response
    let (tx, rx) = tokio::sync::oneshot::channel();

    // Store sender in pending responses map
    {
        let mut pending = pending_responses.lock().unwrap();
        pending.insert(request_id, tx);
    }

    // Send command with request ID
    command_sender.send(PluginCommand::CreateVirtualBufferInSplit {
        title,
        content,
        // ... other fields
        request_id: Some(request_id),
    }).map_err(|e| JsErrorBox::generic(format!("Failed to send command: {}", e)))?;

    // Wait for response from editor
    let response = rx.await.map_err(|e|
        JsErrorBox::generic(format!("Response channel closed: {}", e))
    )?;

    // Parse response
    match response {
        PluginResponse::CreateVirtualBuffer { buffer_id, split_id } => {
            Ok(CreateVirtualBufferResult {
                buffer_id: buffer_id.to_string(),
                split_id: split_id.to_string(),
            })
        }
        _ => Err(JsErrorBox::generic("Unexpected response type")),
    }
}
```

### Delivering Responses

**Location:** `src/services/plugins/runtime.rs:3041-3055`

The editor calls this method to deliver responses back to waiting ops:

```rust
pub fn deliver_response(&self, request_id: u64, response: PluginResponse) -> Result<()> {
    let mut pending = self.pending_responses.lock().unwrap();

    if let Some(sender) = pending.remove(&request_id) {
        sender.send(response).map_err(|_|
            anyhow::anyhow!("Failed to deliver response: receiver dropped")
        )?;
    } else {
        tracing::warn!("No pending response for request ID {}", request_id);
    }

    Ok(())
}
```

## Shared Runtime State

### State Structure

**Location:** `src/services/plugins/runtime.rs:141-163`

```rust
struct TsRuntimeState {
    /// Read-only snapshot of editor state (updated each frame)
    state_snapshot: Arc<RwLock<EditorStateSnapshot>>,

    /// Command sender for write operations
    command_sender: std::sync::mpsc::Sender<PluginCommand>,

    /// Event handlers: event_name -> Vec<handler_function_name>
    event_handlers: Rc<RefCell<HashMap<String, Vec<String>>>>,

    /// Pending async response senders
    pending_responses: Arc<Mutex<HashMap<u64, tokio::sync::oneshot::Sender<PluginResponse>>>>,

    /// Next request ID counter
    next_request_id: Rc<RefCell<u64>>,

    /// Background processes: process_id -> Child
    background_processes: Rc<RefCell<HashMap<u64, tokio::process::Child>>>,

    /// Next process ID counter
    next_process_id: Rc<RefCell<u64>>,
}
```

This state is stored in the V8 `OpState` and accessible from all ops:

```rust
fn op_example(state: &mut OpState) -> Result<...> {
    let runtime_state = state.borrow::<Rc<RefCell<TsRuntimeState>>>();
    let runtime_state = runtime_state.borrow();

    // Access shared state
    let snapshot = runtime_state.state_snapshot.read().unwrap();
    // ...
}
```

## Event Emission System

### Emitting Events to Plugins

**Location:** `src/services/plugins/runtime.rs:3057-3113`

```rust
pub async fn emit(&mut self, event_name: &str, event_data: &str) -> Result<bool> {
    // Get handlers for this event
    let handler_names = {
        let handlers = self.event_handlers.borrow();
        handlers.get(event_name).cloned().unwrap_or_default()
    };

    if handler_names.is_empty() {
        return Ok(true);
    }

    // Call each registered handler
    for handler_name in &handler_names {
        let script = format!(
            "__eventDispatcher({}, {})",
            serde_json::to_string(handler_name)?,
            event_data
        );

        // Execute the handler
        let result = self.js_runtime.execute_script("<emit>", script);

        if let Err(e) = result {
            tracing::error!("Error executing handler {}: {}", handler_name, e);
            continue;
        }

        // Run event loop to process any async work in the handler
        self.js_runtime.run_event_loop(Default::default()).await?;
    }

    Ok(true)
}
```

**Plugin Side:**
```typescript
// Register handler
globalThis.myHandler = async (data) => {
    await editor.spawnProcess("git", ["status"]);
    editor.setStatus("Done!");
};

editor.on("after_file_save", "myHandler");
```

**Editor Side:**
```rust
// Emit event with JSON data
runtime.emit("after_file_save", r#"{"path": "/foo/bar.rs", "bufferId": "123"}"#).await?;
```

## What Fresh Doesn't Use

Fresh creates a **minimal Deno runtime** with only what's needed:

### Not Included:
- ❌ Standard Deno APIs (`Deno.readFile`, `Deno.writeFile`)
- ❌ Web APIs (`fetch`, `WebSocket`, `localStorage`)
- ❌ Deno permissions system
- ❌ npm package imports
- ❌ TypeScript type checking (only transpilation)
- ❌ Deno standard library
- ❌ Web Workers
- ❌ Dynamic imports (can be added if needed)

### Only Included:
- ✅ `deno_core` runtime
- ✅ TypeScript transpilation via `deno_ast`
- ✅ Custom `fresh_runtime` extension (82 ops)
- ✅ ES module support
- ✅ async/await with Promises
- ✅ V8 engine
- ✅ Event loop

This minimal approach:
1. **Reduces attack surface** - No network access, no arbitrary file I/O
2. **Improves performance** - Less code to load and execute
3. **Simplifies mental model** - Only Fresh APIs available
4. **Maintains control** - All operations go through typed ops

## Performance Considerations

### Fast Ops for Hot Paths

```rust
// Hot path: Called every cursor movement
#[op2(fast)]
fn op_fresh_get_cursor_position(state: &mut OpState) -> u64 {
    // No serialization overhead
    // Direct memory access
    // Optimized by V8
}
```

### Batch Operations

```rust
// Clear all overlays in namespace at once
#[op2]
fn op_fresh_clear_namespace(state: &mut OpState, #[string] namespace: String) {
    // Single command instead of N remove_overlay calls
}
```

### Async for I/O

```rust
// Don't block the plugin thread on I/O
#[op2(async)]
async fn op_fresh_read_file(#[string] path: String) -> Result<String> {
    tokio::fs::read_to_string(path).await
}
```

## Error Handling

### Rust → JavaScript Errors

```rust
fn op_example() -> Result<String, JsErrorBox> {
    // Rust errors automatically become JavaScript exceptions
    Err(JsErrorBox::generic("Something went wrong"))
}
```

**JavaScript sees:**
```typescript
try {
    core.ops.op_example();
} catch (error) {
    console.log(error.message); // "Something went wrong"
}
```

### JavaScript → Rust Errors

```typescript
globalThis.myHandler = () => {
    throw new Error("Plugin error");
};
```

**Rust catches:**
```rust
if let Err(e) = runtime.execute_script("<test>", "myHandler()") {
    println!("JavaScript error: {}", e);
}
```

## Complete Flow Example

### Plugin Code (TypeScript)

```typescript
// plugins/my_plugin.ts
globalThis.myCommand = async () => {
    const bufferId = editor.getActiveBufferId();
    const path = editor.getBufferPath(bufferId);

    const result = await editor.spawnProcess("git", ["log", "--oneline", "-10"]);

    const { bufferId: resultBufferId } = await editor.createVirtualBufferInSplit({
        title: "Git Log",
        content: result.stdout,
    });

    editor.setStatus("Git log displayed");
};

editor.registerCommand("git-log", "myCommand");
```

### Execution Flow

1. **User types `:git-log`**
2. **Editor looks up command** → finds `myCommand`
3. **Editor calls** → `runtime.execute_script("myCommand()")`
4. **JavaScript executes:**
   - Calls `editor.getActiveBufferId()` → `op_fresh_get_active_buffer_id()`
   - Calls `editor.getBufferPath()` → `op_fresh_get_buffer_path()`
   - Calls `await editor.spawnProcess()` → `op_fresh_spawn_process()` (async)
     - Creates Tokio task
     - Spawns git process
     - Returns Promise
     - Promise resolves with output
   - Calls `await editor.createVirtualBufferInSplit()` → `op_fresh_create_virtual_buffer_in_split()` (async)
     - Generates request ID: 42
     - Creates oneshot channel
     - Stores sender in `pending_responses[42]`
     - Sends `PluginCommand::CreateVirtualBufferInSplit { request_id: 42 }`
     - Awaits response
5. **Main editor thread:**
   - Receives command from channel
   - Creates virtual buffer
   - Calls `runtime.deliver_response(42, response)`
6. **Plugin thread:**
   - Oneshot receiver gets response
   - Promise resolves
   - JavaScript continues
   - Calls `editor.setStatus()` → `op_fresh_set_status()`
7. **Event loop runs**
8. **Command completes**

## Summary

Fresh's Deno integration provides:

1. **Modern JavaScript Runtime** - V8 engine via deno_core
2. **TypeScript Support** - Automatic transpilation via deno_ast
3. **82 Custom Ops** - Comprehensive editor API
4. **Native Async/Await** - Tokio integration for true async
5. **Minimal Attack Surface** - Only Fresh APIs, no standard Deno APIs
6. **Type-Safe Communication** - Structured commands and responses
7. **Event-Driven Architecture** - Hook system for editor events
8. **Persistent Runtime** - Single runtime shared across all plugins
9. **Process Isolation** - Plugins run in dedicated thread

This represents a production-ready, secure, and performant plugin runtime that successfully addresses the limitations of previous Lua-based systems while maintaining simplicity and safety.
