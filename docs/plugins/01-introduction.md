# Stage 1: Introduction

## Why Pumpkin?

If you've been developing Minecraft plugins with Bukkit, Spigot, or Paper, you already know how rewarding it is to extend the game. Now imagine doing that with a server that runs **significantly faster**, uses **less memory**, and catches bugs **before your plugin even runs**. That's what Pumpkin offers.

Pumpkin is a Minecraft server built from scratch in Rust — a modern language designed for performance and reliability. Don't worry if you've never written Rust before: Pumpkin's plugin system is designed to be approachable, and this guide will teach you everything you need along the way.

### What Makes Pumpkin Different?

Here's a quick comparison of what changes when you move from Java to Pumpkin:

| What you're used to (Java) | What's new (Pumpkin/Rust) |
|----------------------------|---------------------------|
| Plugins are `.jar` files | Plugins are native libraries (`.so`/`.dll`/`.dylib`) |
| JVM runs your code | Your code compiles directly to machine code — no runtime overhead |
| Garbage collector manages memory | Rust's compiler checks memory safety at compile time — no GC pauses |
| `plugin.yml` describes your plugin | `Cargo.toml` describes your plugin (Rust's build file) |
| `extends JavaPlugin` | `#[plugin_impl]` macro on your struct |
| `@EventHandler` annotations | Implement handler traits for specific events |
| `onCommand()` with manual parsing | `CommandTree` builder pattern with typed arguments |

### How Does Rust Benefit Plugin Development?

You might be wondering: "Why switch from Java?" Here are some real benefits:

1. **No garbage collector stalls.** In Java, the GC can cause lag spikes — especially on busy servers. Rust doesn't have a GC at all, so your server runs smoothly even under heavy load.

2. **Bugs caught at compile time.** Many errors that would crash a Java plugin at runtime (null pointers, data races, type mismatches) are caught by the Rust compiler before your plugin even loads. If it compiles, it's far less likely to crash.

3. **True concurrency without fear.** In Java, shared mutable state across threads is a common source of bugs. Rust makes data races *impossible* at the language level. The compiler simply won't allow unsafe concurrent access.

4. **Native performance.** Your plugin runs as compiled machine code, not interpreted bytecode. This means faster event handling, faster commands, and lower resource usage.

5. **Smaller memory footprint.** No JVM means the server and your plugins use significantly less RAM — great for hosting multiple servers or running on modest hardware.

---

## How Pumpkin Plugins Work

The basic idea is the same as Java plugins: you write code, the server loads it, and your plugin reacts to game events and commands. The packaging is a bit different:

1. You create a Rust project and write your plugin code
2. You add the `#[plugin_impl]` macro to your main struct — this generates all the boilerplate for you
3. You compile it into a native library (Cargo, the Rust build tool, handles this)
4. You drop the compiled file into the server's `plugins/` folder
5. Pumpkin loads it on startup, just like Bukkit loads `.jar` files

### Side-by-Side: Hello World

Here's what a minimal plugin looks like in both ecosystems:

```java
// Java (Bukkit)
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
// Rust (Pumpkin)
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

The structure is very similar! The `#[plugin_impl]` and `#[plugin_method]` macros handle the behind-the-scenes work — you just write your logic.

---

## Rust Concepts for Java Developers

Don't worry about mastering Rust before you start. Here's a gentle introduction to the concepts you'll encounter most often in plugin development:

### Variables and Types

Rust is statically typed like Java, but with type inference — you often don't need to write types explicitly:

```java
// Java
String name = "Steve";
int health = 20;
final double speed = 0.1;  // Can't be reassigned
```

```rust
// Rust
let name = "Steve";       // Type inferred as &str
let health = 20;          // Type inferred as i32
let speed = 0.1;          // Variables are immutable by default!
let mut counter = 0;      // Use 'mut' to make a variable mutable
counter += 1;
```

> **Key difference:** In Rust, variables are **immutable by default**. You need `mut` to make them changeable. This is the opposite of Java, where everything is mutable unless you use `final`. This helps prevent accidental changes to your data.

### `Option<T>` Instead of `null`

One of Rust's best features: there's no `null`! Instead, when a value might not exist, Rust uses `Option<T>`:

```java
// Java — can return null, might crash with NullPointerException
Player player = Bukkit.getPlayer("Steve");
if (player != null) {
    player.sendMessage("Hello!");
}
```

```rust
// Rust — the compiler forces you to handle the "not found" case
if let Some(player) = server.get_player_by_name("Steve") {
    player.send_system_message(&TextComponent::text("Hello!")).await;
}
// No null, no NullPointerException, ever!
```

The compiler won't let you forget to check for `None` — this alone eliminates one of the most common sources of Java plugin crashes.

### `Result<T, E>` Instead of Exceptions

Similarly, Rust doesn't have exceptions. Functions that can fail return a `Result`:

```java
// Java — might throw an exception you forgot to catch
try {
    config.save(file);
} catch (IOException e) {
    getLogger().warning("Failed to save: " + e.getMessage());
}
```

```rust
// Rust — the compiler ensures you handle the error
match std::fs::write(&config_path, content) {
    Ok(()) => server.log("Config saved!"),
    Err(e) => server.log(format!("Failed to save: {e}")),
}

// Or use the ? operator to propagate errors (common in plugin methods)
std::fs::write(&config_path, content).map_err(|e| e.to_string())?;
```

### Shared References with `Arc<T>`

In Java, objects are passed by reference and the garbage collector handles cleanup. In Rust, when something needs to be shared between different parts of your plugin, you'll use `Arc<T>` (Atomic Reference Counted):

```java
// Java — objects are shared by reference automatically
Player player = getPlayer("Steve");
someOtherMethod(player); // Both variables point to the same player
```

```rust
// Rust — Arc lets you share data safely between different parts of your code
let player = server.get_player_by_name("Steve");
let player_clone = player.clone(); // Creates another reference (cheap operation)
// Both 'player' and 'player_clone' point to the same player object
```

You don't need to deeply understand `Arc` right now — just know that when you see `Arc<Player>` or `Arc<Context>`, it's Rust's way of sharing data safely.

### `async`/`await` — Doing Things Without Blocking

Pumpkin handles many operations asynchronously (sending messages, querying data, etc.). If you've used `CompletableFuture` in Java, `async`/`await` in Rust is similar but much cleaner:

```java
// Java — chained callbacks
CompletableFuture.supplyAsync(() -> database.lookup(playerId))
    .thenAccept(result -> player.sendMessage(result));
```

```rust
// Rust — reads like normal sequential code
let result = database.lookup(player_id).await;
player.send_message(result).await;
```

The `await` keyword pauses the current task until the operation completes, without blocking the whole server. You'll see `.await` throughout Pumpkin's API — it just means "wait for this to finish."

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

Notice how much simpler the Pumpkin structure is — no separate `plugin.yml`, no deep directory nesting. Your `Cargo.toml` handles both build configuration and plugin metadata, and all your code lives in `src/`.

---

## What's Next?

Ready to get your hands dirty? In [Stage 2: Getting Started](02-getting-started.md), you'll install Rust, create your first plugin project, and see it running on a Pumpkin server.

---

[← Back to Index](README.md) | [Next: Getting Started →](02-getting-started.md)
