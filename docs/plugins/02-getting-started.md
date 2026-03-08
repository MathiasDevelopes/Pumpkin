# Stage 2: Getting Started

## Setting Up Your Development Environment

### Prerequisites

1. **Install Rust** (minimum version 1.94):
   ```bash
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
   ```

   > **Coming from Java?** Rust's installer sets up everything you need in one step. There's no separate JDK, no JAVA_HOME environment variable, and no version managers to worry about. The `rustup` tool handles everything.

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

### What is Cargo?

If you've used Maven or Gradle in Java, **Cargo** is Rust's equivalent — but it does even more. It's your:

- **Build tool** — compiles your code (`cargo build`)
- **Package manager** — downloads dependencies automatically
- **Test runner** — runs your tests (`cargo test`)
- **Project scaffolder** — creates new projects (`cargo new`)

All from a single command-line tool. No XML configuration files, no plugin repositories to configure.

---

## Creating Your First Plugin

### Step 1: Create the Project

```bash
cargo new --lib my_first_plugin
cd my_first_plugin
```

This creates a library project (not a standalone program), similar to creating a new Maven project for a Bukkit plugin.

### Step 2: Configure `Cargo.toml`

Open `Cargo.toml` — this is your project's configuration file, similar to Java's `pom.xml` or `build.gradle` combined with `plugin.yml`. Replace its contents with:

```toml
[package]
name = "my-first-plugin"
version = "0.1.0"
authors = ["Your Name"]
description = "My first Pumpkin plugin"
edition = "2024"

[lib]
crate-type = ["cdylib"]  # Compile as a native library the server can load

[dependencies]
pumpkin = { path = "../Pumpkin/pumpkin" }        # Core server API
pumpkin-api-macros = { path = "../Pumpkin/pumpkin-api-macros" }  # Plugin macros
```

A few things to note:

- The `[package]` section is like your `plugin.yml` — it defines your plugin's name, version, and description. Pumpkin reads this metadata automatically.
- `crate-type = ["cdylib"]` tells Rust to compile your code into a dynamic library that Pumpkin can load at runtime, similar to how the JVM loads `.jar` files.
- The `[dependencies]` section is like Maven dependencies — Cargo downloads and links them automatically.

#### Java Comparison

In Java, you'd have a separate `plugin.yml` for metadata:

```yaml
# Java: plugin.yml
name: MyFirstPlugin
version: 1.0.0
main: com.example.MyFirstPlugin
author: Your Name
description: My first plugin
```

In Pumpkin, `Cargo.toml` serves both purposes. Less files, less configuration.

### Step 3: Write the Plugin

Replace `src/lib.rs` with:

```rust
use std::sync::Arc;

use pumpkin::plugin::api::context::Context;
use pumpkin_api_macros::{plugin_impl, plugin_method};

// The #[plugin_impl] macro sets up everything the server needs
// to load your plugin — metadata, version checks, and initialization.
// Think of it as the Rust equivalent of "extends JavaPlugin".
#[plugin_impl]
pub struct MyFirstPlugin;

impl MyFirstPlugin {
    // #[plugin_method] marks this as a plugin lifecycle method.
    // on_load is called when the server starts — like onEnable() in Bukkit.
    #[plugin_method]
    pub fn on_load(&mut self, server: Arc<Context>) -> Result<(), String> {
        server.log("Hello from MyFirstPlugin! 🎃");
        Ok(())
    }

    // on_unload is called when the server stops — like onDisable() in Bukkit.
    #[plugin_method]
    pub fn on_unload(&mut self, server: Arc<Context>) -> Result<(), String> {
        server.log("Goodbye from MyFirstPlugin!");
        Ok(())
    }
}
```

Let's break down the new Rust syntax:

- **`use` statements** — Like Java `import` statements. They bring types into scope.
- **`pub struct MyFirstPlugin;`** — Declares your plugin. A struct in Rust is similar to a Java class, but without inheritance. The `pub` makes it visible outside this file.
- **`impl MyFirstPlugin`** — This is where you define methods on your struct, similar to writing methods inside a Java class body.
- **`&mut self`** — This means the method can modify the plugin's data. In Java terms, it's like a regular instance method. (You'll learn more about `&self` vs `&mut self` later.)
- **`Arc<Context>`** — The server context, passed to you on load. `Arc` means it's a shared reference that's safe to use across threads. Don't worry about the details — just use it like you'd use `this.getServer()` in Bukkit.
- **`Result<(), String>`** — The method returns either success (`Ok(())`) or an error (`Err("message")`). This is how Rust handles errors instead of throwing exceptions.

### Step 4: Build the Plugin

```bash
cargo build --release
```

Your compiled plugin will be at:
- **Linux:** `target/release/libmy_first_plugin.so`
- **macOS:** `target/release/libmy_first_plugin.dylib`
- **Windows:** `target/release/my_first_plugin.dll`

> **Tip:** The first build downloads dependencies and takes a bit longer. Subsequent builds are much faster since Cargo caches everything.

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

🎉 Congratulations — you've just built and loaded your first Pumpkin plugin!

---

## What Do the Macros Do?

You might be curious about what `#[plugin_impl]` and `#[plugin_method]` actually do behind the scenes. The short answer: they generate the boilerplate code that Pumpkin needs to load your plugin.

- **`#[plugin_impl]`** reads your `Cargo.toml` and generates the plugin metadata (name, version, authors, description) plus a factory function that the server calls to create your plugin instance. Without this macro, you'd need to write several `#[no_mangle]` functions manually.

- **`#[plugin_method]`** wraps your method so it works with Pumpkin's async system. You write normal-looking code, and the macro makes it compatible with the server's async runtime.

You don't need to understand the generated code to use these macros effectively. If you're curious, you can explore the details later in [Stage 9: Advanced Topics](09-advanced-topics.md).

---

## Plugin Data Folder

Every plugin gets its own data directory, just like `JavaPlugin.getDataFolder()`:

```rust
#[plugin_method]
pub fn on_load(&mut self, server: Arc<Context>) -> Result<(), String> {
    // Creates plugins/my-first-plugin/ if it doesn't exist
    let data_folder = server.get_data_folder();
    server.log(format!("Data folder: {}", data_folder.display()));
    Ok(())
}
```

This is where you'd store configuration files, player data, or any other files your plugin needs.

---

## Logging

Pumpkin provides a built-in logging system for plugins:

```rust
#[plugin_method]
pub fn on_load(&mut self, server: Arc<Context>) -> Result<(), String> {
    // Initialize the logger (optional, enables structured logging)
    server.init_log();

    // Simple log message (automatically prefixed with your plugin name)
    server.log("Plugin loaded successfully!");
    server.log(format!("Running version {}", env!("CARGO_PKG_VERSION")));
    Ok(())
}
```

#### Java Comparison

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

In [Stage 3: Plugin Lifecycle](03-plugin-lifecycle.md), we'll explore what you can do with the `Context` API — finding players, registering events, managing plugin state, and more.

---

[← Introduction](01-introduction.md) | [Back to Index](README.md) | [Next: Plugin Lifecycle →](03-plugin-lifecycle.md)
