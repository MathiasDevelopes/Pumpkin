# Stage 2: Getting Started

## Setting Up Your Development Environment

### Prerequisites

1. **Install Rust** (minimum version 1.94):
   ```bash
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
   ```

2. **Verify your installation:**
   ```bash
   rustc --version   # Should be >= 1.94
   cargo --version
   ```

3. **Clone and build the Pumpkin server** (needed for development):
   ```bash
   git clone https://github.com/Snowiiii/Pumpkin.git
   cd Pumpkin
   cargo build --release
   ```

> **Java developers:** Think of `cargo` as a combination of Maven/Gradle and `javac`. It manages dependencies, compiles code, and runs tests — all in one tool.

---

## Creating Your First Plugin

### Step 1: Create the Project

```bash
cargo new --lib my_first_plugin
cd my_first_plugin
```

This creates a library crate (not a binary), similar to creating a new Maven project for a plugin.

### Step 2: Configure `Cargo.toml`

Replace the contents of `Cargo.toml` with:

```toml
[package]
name = "my-first-plugin"
version = "0.1.0"
authors = ["Your Name"]
description = "My first Pumpkin plugin"
edition = "2024"

[lib]
crate-type = ["cdylib"]  # Compile as a dynamic library

[dependencies]
pumpkin = { path = "../Pumpkin/pumpkin" }        # Core server API
pumpkin-api-macros = { path = "../Pumpkin/pumpkin-api-macros" }  # Plugin macros
```

> **Important:** The `crate-type = ["cdylib"]` tells Cargo to produce a C-compatible dynamic library (`.so`/`.dll`/`.dylib`). This is what the Pumpkin server loads at runtime.

#### Java Comparison

In Java, your `plugin.yml` defines the metadata:

```yaml
# Java: plugin.yml
name: MyFirstPlugin
version: 1.0.0
main: com.example.MyFirstPlugin
author: Your Name
description: My first plugin
```

In Pumpkin, `Cargo.toml` serves this purpose — the `#[plugin_impl]` macro automatically reads the `[package]` fields and exports them as your plugin's metadata.

### Step 3: Write the Plugin

Replace `src/lib.rs` with:

```rust
use std::sync::Arc;

use pumpkin::plugin::api::context::Context;
use pumpkin_api_macros::{plugin_impl, plugin_method};

/// The main plugin struct. Annotate with #[plugin_impl] to generate
/// all the required boilerplate (metadata, API version, factory function).
#[plugin_impl]
pub struct MyFirstPlugin;

impl MyFirstPlugin {
    /// Called when the plugin is loaded by the server.
    /// Similar to JavaPlugin.onEnable()
    #[plugin_method]
    pub fn on_load(&mut self, server: Arc<Context>) -> Result<(), String> {
        server.log("Hello from MyFirstPlugin! 🎃");
        Ok(())
    }

    /// Called when the plugin is unloaded.
    /// Similar to JavaPlugin.onDisable()
    #[plugin_method]
    pub fn on_unload(&mut self, server: Arc<Context>) -> Result<(), String> {
        server.log("Goodbye from MyFirstPlugin!");
        Ok(())
    }
}
```

### Step 4: Build the Plugin

```bash
cargo build --release
```

Your compiled plugin will be at:
- **Linux:** `target/release/libmy_first_plugin.so`
- **macOS:** `target/release/libmy_first_plugin.dylib`
- **Windows:** `target/release/my_first_plugin.dll`

### Step 5: Install and Run

Copy the compiled library to the Pumpkin server's `plugins/` directory:

```bash
# Linux example
cp target/release/libmy_first_plugin.so /path/to/pumpkin-server/plugins/

# Start the server
cd /path/to/pumpkin-server
cargo run --release
```

You should see your plugin's message in the server console:

```
[MyFirstPlugin] Hello from MyFirstPlugin! 🎃
```

---

## Understanding `#[plugin_impl]`

The `#[plugin_impl]` macro does a lot of heavy lifting. Here's what it generates behind the scenes:

```rust
// What you write:
#[plugin_impl]
pub struct MyFirstPlugin;

// What the macro generates:
#[no_mangle]
pub static METADATA: PluginMetadata<'static> = PluginMetadata {
    name: "my-first-plugin",        // From Cargo.toml [package].name
    version: "0.1.0",               // From Cargo.toml [package].version
    authors: "Your Name",           // From Cargo.toml [package].authors
    description: "My first plugin", // From Cargo.toml [package].description
};

#[no_mangle]
pub static PUMPKIN_API_VERSION: u32 = 2; // Current API version

#[no_mangle]
pub fn plugin() -> Box<dyn Plugin> {
    Box::new(MyFirstPlugin)
}

impl Plugin for MyFirstPlugin {
    fn on_load(&mut self, server: Arc<Context>) -> PluginFuture<'_, Result<(), String>> {
        // Your on_load code, wrapped in async
    }
    fn on_unload(&mut self, server: Arc<Context>) -> PluginFuture<'_, Result<(), String>> {
        // Your on_unload code, wrapped in async
    }
}
```

This is analogous to how Bukkit reads your `plugin.yml` and uses reflection to instantiate your main class — except Pumpkin does it at the binary level through exported symbols.

---

## Understanding `#[plugin_method]`

The `#[plugin_method]` attribute wraps your method to return a `PluginFuture`. This lets you write straightforward synchronous-looking code that's actually async under the hood:

```rust
// What you write:
#[plugin_method]
pub fn on_load(&mut self, server: Arc<Context>) -> Result<(), String> {
    server.log("Hello!");
    Ok(())
}

// What it becomes:
pub fn on_load(&mut self, server: Arc<Context>) -> PluginFuture<'_, Result<(), String>> {
    Box::pin(async move {
        server.log("Hello!");
        Ok(())
    })
}
```

---

## Plugin Data Folder

Every plugin gets its own data directory, similar to `JavaPlugin.getDataFolder()`:

```rust
#[plugin_method]
pub fn on_load(&mut self, server: Arc<Context>) -> Result<(), String> {
    // Creates plugins/my-first-plugin/ if it doesn't exist
    let data_folder = server.get_data_folder();
    server.log(format!("Data folder: {}", data_folder.display()));
    Ok(())
}
```

---

## Logging

Pumpkin provides a built-in logging system for plugins:

```rust
#[plugin_method]
pub fn on_load(&mut self, server: Arc<Context>) -> Result<(), String> {
    // Initialize the logger (optional, enables structured logging)
    server.init_log();

    // Simple log message (prefixed with plugin name)
    server.log("Plugin loaded successfully!");
    server.log(format!("Running version {}", env!("CARGO_PKG_VERSION")));
    Ok(())
}
```

### Java Comparison

```java
// Java
getLogger().info("Plugin loaded successfully!");
getLogger().info("Running version " + getDescription().getVersion());
```

---

## Common Build Issues

### "API version mismatch"

```
Plugin API version mismatch (plugin 1, server 2)
```

This means your plugin was compiled against a different version of the Pumpkin API. Rebuild your plugin against the current server version.

### "Missing plugin metadata"

Make sure your `Cargo.toml` has `name`, `version`, `authors`, and `description` fields in the `[package]` section.

### "crate-type must be cdylib"

Ensure your `Cargo.toml` contains:

```toml
[lib]
crate-type = ["cdylib"]
```

---

## What's Next?

In [Stage 3: Plugin Lifecycle](03-plugin-lifecycle.md), we'll dive deeper into the `Plugin` trait, the `Context` API, and how to manage your plugin's state.

---

[← Introduction](01-introduction.md) | [Back to Index](README.md) | [Next: Plugin Lifecycle →](03-plugin-lifecycle.md)
