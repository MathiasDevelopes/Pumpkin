# Stage 3: Plugin Lifecycle

## The `Plugin` Trait

At the core of every Pumpkin plugin is the `Plugin` trait. If you've worked with Bukkit, think of it as the equivalent of extending `JavaPlugin`:

```rust
pub trait Plugin: Send + Sync + 'static {
    fn on_load(&mut self, server: Arc<Context>) -> PluginFuture<'_, Result<(), String>>;
    fn on_unload(&mut self, server: Arc<Context>) -> PluginFuture<'_, Result<(), String>>;
}
```

Both methods have default implementations that return `Ok(())`, so you only need to override what you actually use.

### Java Comparison

| Java (`JavaPlugin`) | Pumpkin (`Plugin` trait) |
|---------------------|--------------------------|
| `onLoad()` | — (no equivalent, metadata is static) |
| `onEnable()` | `on_load(&mut self, server: Arc<Context>)` |
| `onDisable()` | `on_unload(&mut self, server: Arc<Context>)` |
| `getServer()` | `server.server` (via `Context`) |
| `getDataFolder()` | `server.get_data_folder()` |
| `getLogger()` | `server.log(...)` |

---

## Plugin Loading Sequence

When the Pumpkin server starts, plugins go through these stages:

```
1. Discovery    → Server scans the plugins/ directory
2. Validation   → API version check (PUMPKIN_API_VERSION)
3. Loading      → Dynamic library loaded, metadata extracted
4. Initialization → plugin() factory function called
5. on_load()    → Your plugin receives the Context
6. Running      → Plugin responds to events and commands
7. on_unload()  → Server shuts down or plugin is manually unloaded
8. Cleanup      → Library unloaded (except on Windows)
```

### Plugin States

Plugins can be in one of these states:

```rust
pub enum PluginState {
    Loading,          // Plugin is being loaded
    Loaded,           // Successfully loaded and running
    Failed(String),   // Failed to load (with error message)
}
```

---

## The `Context` API

The `Context` struct is your plugin's gateway to the server. It's passed to `on_load` and provides access to everything you need:

```rust
pub struct Context {
    pub server: Arc<Server>,                              // The server instance
    pub handlers: Arc<RwLock<HandlerMap>>,                 // Event handler registry
    pub plugin_manager: Arc<PluginManager>,                // Plugin management
    pub permission_manager: Arc<RwLock<PermissionManager>>,// Permission system
    pub logger: Arc<OnceLock<LoggerOption>>,               // Logging
}
```

### Storing the Context

Since `on_load` is the only place you receive the `Context`, you'll typically store it for later use:

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
        self.context = None;
        server.log("Plugin unloaded!");
        Ok(())
    }
}
```

#### Java Comparison

```java
// Java — the instance is always available via 'this'
public class MyPlugin extends JavaPlugin {
    @Override
    public void onEnable() {
        getServer(); // Always available
    }
}
```

In Pumpkin, you don't have implicit access to the server — you receive it explicitly through the `Context`. This is a Rust pattern: explicit is better than implicit.

---

## Core Context Methods

### Data Folder

```rust
// Creates plugins/my-plugin/ if it doesn't exist and returns the path
let data_folder = server.get_data_folder();

// Save a config file
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

### Registering Events

```rust
use pumpkin::plugin::api::events::player::PlayerJoinEvent;
use pumpkin::plugin::EventHandler;

server.register_event::<PlayerJoinEvent, _>(
    Arc::new(MyJoinHandler),
    EventPriority::Normal,
    false, // non-blocking (read-only access)
).await;
```

We'll cover events in detail in [Stage 4](04-events.md).

### Registering Commands

```rust
server.register_command(my_command_tree(), "my.command.permission").await;
```

We'll cover commands in detail in [Stage 5](05-commands.md).

---

## Managing Plugin State

Pumpkin plugins can hold mutable state in their struct fields. Since Rust enforces thread safety at compile time, you'll use standard synchronization primitives:

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

### Java Comparison

```java
// Java — thread safety is your responsibility
public class StatsPlugin extends JavaPlugin {
    private final ConcurrentHashMap<String, Integer> playerKills = new ConcurrentHashMap<>();

    @Override
    public void onEnable() {
        // No compile-time thread safety guarantees
    }
}
```

In Rust, the compiler refuses to compile code that could have data races. The `Arc<RwLock<T>>` pattern is Pumpkin's equivalent of Java's `ConcurrentHashMap` or `synchronized` blocks — but enforced at compile time.

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

You can also check if a specific plugin is active:

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

## Complete Example: A Stateful Plugin

Here's a complete example bringing everything together:

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

---

## What's Next?

In [Stage 4: Events](04-events.md), we'll take a deep dive into Pumpkin's event system — how to listen for events, modify them, cancel them, and understand handler priorities.

---

[← Getting Started](02-getting-started.md) | [Back to Index](README.md) | [Next: Events →](04-events.md)
