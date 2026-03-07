# Stage 1: Introduction

## Why Pumpkin?

Pumpkin is a Minecraft server implementation written entirely in Rust. If you've been developing plugins for Bukkit, Spigot, or Paper, you're used to working with Java — a garbage-collected, object-oriented language running on the JVM. Pumpkin takes a different approach: it's compiled to native code, uses Rust's ownership model for memory safety, and leverages async I/O for high performance.

### Key Differences at a Glance

| Aspect | Java (Bukkit/Spigot/Paper) | Rust (Pumpkin) |
|--------|---------------------------|----------------|
| **Language** | Java | Rust |
| **Runtime** | JVM with garbage collection | Native binary, no GC |
| **Plugin format** | `.jar` files (bytecode) | `.so` / `.dll` / `.dylib` (native libraries) |
| **Concurrency** | Threads + synchronized blocks | `async`/`await` with Tokio runtime |
| **Memory management** | Garbage collector | Ownership & borrowing (compile-time) |
| **Plugin API** | Interface-based (`JavaPlugin`) | Trait-based (`Plugin`) |
| **Event system** | Annotations (`@EventHandler`) | Trait implementations (`EventHandler<E>`) |
| **Commands** | `onCommand()` or frameworks | `CommandTree` builder pattern |
| **Configuration** | `plugin.yml` + `config.yml` | `Cargo.toml` metadata + TOML configs |
| **Hot reload** | Supported (with caveats) | Supported on Linux/macOS, limited on Windows |

---

## The Pumpkin Plugin Architecture

In Java, your plugin is a `.jar` file that gets loaded by the server's class loader. In Pumpkin, your plugin is a **native dynamic library** (`.so` on Linux, `.dll` on Windows, `.dylib` on macOS) loaded via the `libloading` crate.

### How It Works

1. You write a Rust library crate with `crate-type = ["cdylib"]`
2. You implement the `Plugin` trait and annotate your struct with `#[plugin_impl]`
3. The macro generates the required exported symbols:
   - `PUMPKIN_API_VERSION` — for compatibility checking
   - `METADATA` — your plugin's name, version, authors, and description
   - `plugin()` — a factory function that creates your plugin instance
4. You compile and drop the resulting library into the server's `plugins/` directory
5. On startup, Pumpkin's `NativePluginLoader` discovers and loads your plugin

### Java Comparison

```java
// Java: plugin.yml defines metadata, class extends JavaPlugin
public class MyPlugin extends JavaPlugin {
    @Override
    public void onEnable() {
        getLogger().info("Plugin enabled!");
    }

    @Override
    public void onDisable() {
        getLogger().info("Plugin disabled!");
    }
}
```

```rust
// Rust: Cargo.toml defines metadata, struct implements Plugin trait
use pumpkin::plugin::api::context::Context;
use pumpkin_api_macros::{plugin_impl, plugin_method};
use std::sync::Arc;

#[plugin_impl]
pub struct MyPlugin;

impl MyPlugin {
    #[plugin_method]
    pub fn on_load(&mut self, server: Arc<Context>) -> Result<(), String> {
        server.log("Plugin enabled!");
        Ok(())
    }

    #[plugin_method]
    pub fn on_unload(&mut self, server: Arc<Context>) -> Result<(), String> {
        server.log("Plugin disabled!");
        Ok(())
    }
}
```

---

## Rust Concepts You'll Need

If you're coming from Java, here are the key Rust concepts to get comfortable with:

### Ownership & Borrowing (Instead of GC)

In Java, the garbage collector handles memory. In Rust, the compiler enforces ownership rules:

```rust
// Owned value — only one owner at a time
let player_name = String::from("Steve");

// Borrowing — temporary read access
fn greet(name: &str) {
    println!("Hello, {name}!");
}
greet(&player_name); // Borrow, don't move
```

### `Arc<T>` — Shared Ownership (Like Java References)

In Pumpkin's plugin system, you'll frequently see `Arc<T>` (Atomic Reference Counted). This is the closest Rust equivalent to a Java object reference shared between threads:

```rust
use std::sync::Arc;

// Similar to Java: Player player = getPlayer("Steve");
let player: Arc<Player> = get_player("Steve");

// Multiple references can exist simultaneously
let player_clone = player.clone(); // Cheap reference count increment
```

### `async`/`await` — Asynchronous Programming

Pumpkin uses Tokio for async I/O. If you've used CompletableFuture in Java, the concept is similar but more ergonomic:

```java
// Java (CompletableFuture)
CompletableFuture.supplyAsync(() -> {
    return database.lookup(playerId);
}).thenAccept(result -> {
    player.sendMessage(result);
});
```

```rust
// Rust (async/await)
let result = database.lookup(player_id).await;
player.send_message(result).await;
```

### Traits (Instead of Interfaces)

Rust traits are similar to Java interfaces, but more powerful:

```java
// Java
public interface Plugin {
    void onEnable();
    void onDisable();
}
```

```rust
// Rust
pub trait Plugin: Send + Sync + 'static {
    fn on_load(&mut self, server: Arc<Context>) -> PluginFuture<'_, Result<(), String>>;
    fn on_unload(&mut self, server: Arc<Context>) -> PluginFuture<'_, Result<(), String>>;
}
```

The `Send + Sync + 'static` bounds ensure your plugin is safe to use across threads — something Java handles implicitly through the JVM.

---

## Project Structure Comparison

### Java Plugin (Maven)

```
my-plugin/
├── pom.xml                    # Build configuration
├── src/main/
│   ├── java/com/example/
│   │   └── MyPlugin.java     # Plugin class
│   └── resources/
│       └── plugin.yml         # Plugin metadata
└── target/
    └── my-plugin-1.0.jar      # Compiled plugin
```

### Pumpkin Plugin (Cargo)

```
my-plugin/
├── Cargo.toml                 # Build config + plugin metadata
├── src/
│   └── lib.rs                 # Plugin implementation
└── target/release/
    └── libmy_plugin.so        # Compiled plugin (Linux)
```

---

## What's Next?

In [Stage 2: Getting Started](02-getting-started.md), you'll set up your development environment and create your first Pumpkin plugin from scratch.

---

[← Back to Index](README.md) | [Next: Getting Started →](02-getting-started.md)
