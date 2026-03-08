# Stage 3: Plugin Lifecycle

## How Plugins Start and Stop

Every Pumpkin plugin has a simple lifecycle, very similar to Bukkit:

1. **`on_load`** — Called when your plugin starts (like `onEnable()` in Bukkit)
2. **Running** — Your plugin responds to events and commands
3. **`on_unload`** — Called when the server stops or your plugin is unloaded (like `onDisable()`)

Both methods receive a `Context` object — your plugin's gateway to the server. Think of it as a combination of `this.getServer()` and `this` from Bukkit, all in one.

### Java Comparison

| Java (`JavaPlugin`) | Pumpkin |
|---------------------|---------|
| `onEnable()` | `on_load(server)` |
| `onDisable()` | `on_unload(server)` |
| `getServer()` | `server.server` |
| `getDataFolder()` | `server.get_data_folder()` |
| `getLogger()` | `server.log(...)` |

---

## Plugin Loading Sequence

When the Pumpkin server starts, here's what happens with your plugin:

```
1. Discovery      → Server scans the plugins/ directory for library files
2. Validation     → Checks that your plugin was built for the current API version
3. Loading        → Opens your library and reads its metadata
4. Initialization → Creates an instance of your plugin struct
5. on_load()      → Your code runs — register events, commands, etc.
6. Running        → Your plugin responds to game events
7. on_unload()    → Server shuts down or plugin is manually unloaded
8. Cleanup        → Library is closed
```

If your plugin fails to load (compilation error, API mismatch, etc.), the server continues running without it and logs the error.

---

## The Context: Your Server Connection

The `Context` object is passed to your `on_load` and `on_unload` methods. It provides everything you need to interact with the server:

- **`server.log("message")`** — Log a message (auto-prefixed with your plugin name)
- **`server.get_data_folder()`** — Get your plugin's data directory
- **`server.get_player_by_name("Steve")`** — Find an online player
- **`server.register_event(...)`** — Listen for game events
- **`server.register_command(...)`** — Add a custom command
- **`server.register_permission(...)`** — Register a permission node
- **`server.server`** — Access the underlying server instance for advanced operations

### Storing the Context

In Bukkit, you always have access to the server through `this` (your plugin instance). In Pumpkin, the `Context` is given to you in `on_load`, so you'll want to save it if you need it later:

```rust
#[plugin_impl]
pub struct MyPlugin {
    context: Option<Arc<Context>>,
}

impl MyPlugin {
    #[plugin_method]
    pub fn on_load(&mut self, server: Arc<Context>) -> Result<(), String> {
        self.context = Some(server.clone());
        server.log("Plugin loaded!");
        Ok(())
    }

    #[plugin_method]
    pub fn on_unload(&mut self, server: Arc<Context>) -> Result<(), String> {
        self.context = None;  // Clean up the reference
        server.log("Plugin unloaded!");
        Ok(())
    }
}
```

> **New to Rust?** The `Option<Arc<Context>>` means "this might or might not hold a Context." `Option` is Rust's way of saying "this could be empty" — like a nullable field in Java, but the compiler forces you to check before using it. `Arc` means it's a shared reference that's safe to use across threads.

#### Java Comparison

```java
// Java — the server is always available through 'this'
public class MyPlugin extends JavaPlugin {
    @Override
    public void onEnable() {
        getServer(); // Always available
    }
}
```

In Pumpkin, you receive the Context explicitly. This is a common Rust pattern: being explicit about what you have access to, rather than relying on hidden global state.

---

## What You Can Do with Context

### Data Folder

```rust
// Creates plugins/my-plugin/ if it doesn't exist and returns the path
let data_folder = server.get_data_folder();

// Save a config file to your plugin's folder
let config_path = data_folder.join("config.toml");
std::fs::write(&config_path, "setting = true").map_err(|e| e.to_string())?;
```

### Finding Players

```rust
// Look up an online player by name
if let Some(player) = server.get_player_by_name("Steve") {
    server.log(format!("Found player: {}", player.gameprofile.name));
}
```

#### Java Comparison

```java
// Java
Player player = Bukkit.getPlayer("Steve");
if (player != null) {
    getLogger().info("Found player: " + player.getName());
}
```

Notice how similar these are! The main difference is `if let Some(player)` instead of `if (player != null)` — Rust uses `Option` instead of null.

### Registering Events (Preview)

```rust
// Register a handler for player join events
server.register_event::<PlayerJoinEvent, _>(
    Arc::new(MyJoinHandler),
    EventPriority::Normal,
    false,  // non-blocking (read-only)
).await;
```

We'll cover events in detail in [Stage 4](04-events.md).

### Registering Commands (Preview)

```rust
// Register a custom command with a permission requirement
server.register_command(my_command_tree(), "my.command.permission").await;
```

We'll cover commands in detail in [Stage 5](05-commands.md).

---

## Managing Plugin State

Your plugin struct can hold data, just like fields in a Java class. Here's a plugin that tracks player statistics:

```rust
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::RwLock;

#[plugin_impl]
pub struct StatsPlugin {
    context: Option<Arc<Context>>,
    player_kills: Arc<RwLock<HashMap<String, u32>>>,
}

impl StatsPlugin {
    #[plugin_method]
    pub fn on_load(&mut self, server: Arc<Context>) -> Result<(), String> {
        self.context = Some(server.clone());
        self.player_kills = Arc::new(RwLock::new(HashMap::new()));
        server.log("Stats plugin loaded!");
        Ok(())
    }
}
```

> **New to Rust?** The `Arc<RwLock<HashMap<String, u32>>>` might look intimidating, but here's what each layer does:
> - `HashMap<String, u32>` — A map from player names to kill counts (like Java's `HashMap`)
> - `RwLock<...>` — Allows multiple readers OR one writer at a time (like Java's `ReadWriteLock`)
> - `Arc<...>` — Lets you share this data safely between your plugin and its event handlers
>
> Together, they're the Rust equivalent of Java's `ConcurrentHashMap` — but with compile-time guarantees that you'll never have a data race.

### Java Comparison

```java
// Java — thread safety is your responsibility
public class StatsPlugin extends JavaPlugin {
    private final ConcurrentHashMap<String, Integer> playerKills = new ConcurrentHashMap<>();
}
```

In Java, choosing the wrong collection type (e.g., `HashMap` instead of `ConcurrentHashMap`) can cause hard-to-find bugs. In Rust, the compiler catches this for you — if you try to share a regular `HashMap` between threads, it won't compile.

---

## Waiting for Other Plugins

If your plugin depends on another plugin, you can wait for it to finish loading:

```rust
#[plugin_method]
pub fn on_load(&mut self, server: Arc<Context>) -> Result<(), String> {
    // Wait for a dependency plugin to be available
    server.plugin_manager
        .wait_for_plugin("economy-plugin")
        .await
        .map_err(|e| format!("Dependency not found: {e}"))?;

    server.log("Economy plugin is ready, proceeding with setup...");
    Ok(())
}
```

You can also check if a specific plugin is active without waiting:

```rust
let is_active = server.plugin_manager.is_plugin_active("economy-plugin").await;
if is_active {
    server.log("Economy integration enabled!");
}
```

#### Java Comparison

```java
// Java
if (getServer().getPluginManager().getPlugin("Vault") != null) {
    getLogger().info("Vault integration enabled!");
}
```

---

## Listing Plugins

```rust
// Get all currently loaded plugins
let plugins = server.plugin_manager.active_plugins().await;
for plugin in &plugins {
    server.log(format!("Active: {} v{}", plugin.name, plugin.version));
}

// Check for failed plugins
let failed = server.plugin_manager.get_failed_plugins().await;
for (name, error) in &failed {
    server.log(format!("Failed: {} — {}", name, error));
}
```

---

## Complete Example: A Welcome Plugin

Here's a complete plugin that counts how many times each player has joined:

```rust
use std::sync::Arc;
use std::collections::HashMap;
use tokio::sync::RwLock;

use pumpkin::plugin::api::context::Context;
use pumpkin::plugin::api::events::player::PlayerJoinEvent;
use pumpkin::plugin::api::events::EventPriority;
use pumpkin::plugin::EventHandler;
use pumpkin::server::Server;
use pumpkin_api_macros::{plugin_impl, plugin_method};

#[plugin_impl]
pub struct WelcomePlugin {
    context: Option<Arc<Context>>,
    join_count: Arc<RwLock<HashMap<String, u32>>>,
}

// This struct handles the join event — we'll explain this pattern
// in detail in Stage 4.
struct JoinHandler {
    join_count: Arc<RwLock<HashMap<String, u32>>>,
}

impl EventHandler<PlayerJoinEvent> for JoinHandler {
    fn handle<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a PlayerJoinEvent,
    ) -> futures::future::BoxFuture<'a, ()> {
        Box::pin(async move {
            let player_name = event.player.gameprofile.name.clone();
            let mut counts = self.join_count.write().await;
            let count = counts.entry(player_name).or_insert(0);
            *count += 1;
        })
    }
}

impl WelcomePlugin {
    #[plugin_method]
    pub fn on_load(&mut self, server: Arc<Context>) -> Result<(), String> {
        self.context = Some(server.clone());
        self.join_count = Arc::new(RwLock::new(HashMap::new()));

        // Register the join event handler
        let handler = Arc::new(JoinHandler {
            join_count: self.join_count.clone(),
        });

        server.register_event::<PlayerJoinEvent, _>(
            handler,
            EventPriority::Normal,
            false,
        ).await;

        server.log("WelcomePlugin loaded!");
        Ok(())
    }

    #[plugin_method]
    pub fn on_unload(&mut self, server: Arc<Context>) -> Result<(), String> {
        let counts = self.join_count.read().await;
        server.log(format!("Total unique players seen: {}", counts.len()));
        Ok(())
    }
}
```

Don't worry if the event handler syntax looks complex — we'll break it down step by step in the next chapter!

---

## What's Next?

In [Stage 4: Events](04-events.md), we'll learn how to listen for game events — player joins, chat messages, block breaks, and more. This is where plugins really come to life!

---

[← Getting Started](02-getting-started.md) | [Back to Index](README.md) | [Next: Events →](04-events.md)
