# Stage 9: Advanced Topics

> **Note:** This stage covers more advanced patterns. If you're new to Rust, you may want to build a few simpler plugins first (using events and commands from Stages 4–5) before diving in here. These topics are here for when you need them.

## Services: Inter-Plugin Communication

Pumpkin's service system allows plugins to expose functionality to other plugins, similar to Bukkit's `ServicesManager`. This is how you build plugin APIs — for example, an economy plugin can expose a service that other plugins use for transactions.

### Registering a Service

Services implement the `Payload` trait (the same base as events), giving them type-safe cross-plugin communication:

```rust
use pumpkin::plugin::api::events::Payload;
use pumpkin_macros::Event;

// Define your service API
#[derive(Event, Clone)]
pub struct EconomyService {
    // Internal state — other plugins interact through methods
}

impl EconomyService {
    pub fn get_balance(&self, player_uuid: &uuid::Uuid) -> f64 {
        // Look up balance...
        100.0
    }

    pub fn transfer(&self, from: &uuid::Uuid, to: &uuid::Uuid, amount: f64) -> bool {
        // Transfer money...
        true
    }
}

// Register during on_load
#[plugin_method]
pub fn on_load(&mut self, server: Arc<Context>) -> Result<(), String> {
    let service = Arc::new(EconomyService { /* ... */ });
    server.register_service("economy", service).await;
    server.log("Economy service registered!");
    Ok(())
}
```

### Consuming a Service

Other plugins can look up and use registered services:

```rust
#[plugin_method]
pub fn on_load(&mut self, server: Arc<Context>) -> Result<(), String> {
    // Wait for the economy plugin to load
    server.plugin_manager.wait_for_plugin("economy-plugin").await
        .map_err(|e| format!("Economy plugin not found: {e}"))?;

    // Get the economy service
    if let Some(economy) = server.get_service::<EconomyService>("economy").await {
        let balance = economy.get_balance(&some_uuid);
        server.log(format!("Player balance: {balance}"));
    }

    Ok(())
}
```

#### Java Comparison

```java
// Java (Bukkit Services API)
// Provider
getServer().getServicesManager().register(
    Economy.class, new MyEconomy(), this, ServicePriority.Normal
);

// Consumer
RegisteredServiceProvider<Economy> rsp =
    getServer().getServicesManager().getRegistration(Economy.class);
Economy economy = rsp.getProvider();
double balance = economy.getBalance(player);
```

---

## Custom Plugin Loaders

Pumpkin's plugin system is extensible — you can register custom loaders for plugins written in languages other than Rust (e.g., Lua, JavaScript, WASM).

A custom loader needs to tell Pumpkin:
- **What files it can handle** (e.g., `.lua` files)
- **How to load them** (parse the file, create a Plugin wrapper)
- **How to unload them** (clean up resources)

### Implementing a Custom Loader

Here's a conceptual example of a Lua plugin loader:

```rust
use pumpkin::plugin::loader::{PluginLoader, PluginLoadFuture, PluginUnloadFuture, LoaderError};
use pumpkin::plugin::api::mod::{Plugin, PluginMetadata};

struct LuaPluginLoader;

impl PluginLoader for LuaPluginLoader {
    fn can_load(&self, path: &Path) -> bool {
        path.extension().is_some_and(|ext| ext == "lua")
    }

    fn load<'a>(&'a self, path: &'a Path) -> PluginLoadFuture<'a> {
        Box::pin(async move {
            // 1. Read the Lua file
            let source = std::fs::read_to_string(path)
                .map_err(|e| LoaderError::LibraryLoad(e.to_string()))?;

            // 2. Parse metadata from comments or config
            let metadata = PluginMetadata {
                name: "lua-plugin",
                version: "1.0.0",
                authors: "Author",
                description: "A Lua plugin",
            };

            // 3. Create a Plugin wrapper
            let plugin = Box::new(LuaPlugin::new(source))
                as Box<dyn Plugin>;

            // 4. Return the plugin with any loader-specific data
            let loader_data: Box<dyn Any + Send + Sync> = Box::new(());

            Ok((plugin, metadata, loader_data))
        })
    }

    fn can_unload(&self) -> bool {
        true
    }

    fn unload(&self, _data: Box<dyn Any + Send + Sync>) -> PluginUnloadFuture<'_> {
        Box::pin(async move { Ok(()) })
    }
}
```

### Registering a Custom Loader

Register your loader in a "meta-plugin" that bootstraps the new loader:

```rust
#[plugin_method]
pub fn on_load(&mut self, server: Arc<Context>) -> Result<(), String> {
    let loader = Arc::new(LuaPluginLoader);
    let loaded_new = server.register_plugin_loader(loader).await;

    if loaded_new {
        server.log("Lua plugin loader registered! New plugins discovered.");
    } else {
        server.log("Lua plugin loader registered. No new plugins found.");
    }

    Ok(())
}
```

When you register a new loader, Pumpkin automatically re-scans the `plugins/` directory for files the new loader can handle.

---

## Async Programming Patterns

Pumpkin is built on Tokio, so plugins can use all async patterns available in the Rust async ecosystem.

### Spawning Background Tasks

```rust
#[plugin_method]
pub fn on_load(&mut self, server: Arc<Context>) -> Result<(), String> {
    let server_clone = server.server.clone();

    // Spawn a background task
    tokio::spawn(async move {
        loop {
            tokio::time::sleep(tokio::time::Duration::from_secs(300)).await;
            // Auto-save player data every 5 minutes
            // save_all_data(&server_clone).await;
        }
    });

    server.log("Background auto-save task started!");
    Ok(())
}
```

#### Java Comparison

```java
// Java (Bukkit scheduler)
Bukkit.getScheduler().runTaskTimerAsynchronously(this, () -> {
    saveAllData();
}, 0L, 6000L); // 6000 ticks = 5 minutes
```

### Timeouts

```rust
use tokio::time::{timeout, Duration};

// Wait for an operation with a timeout
match timeout(Duration::from_secs(5), some_async_operation()).await {
    Ok(result) => { /* Operation completed */ }
    Err(_) => { /* Timed out */ }
}
```

### Channels for Communication

Channels allow different parts of your plugin to communicate asynchronously — similar to Java's `BlockingQueue` but designed for async code:

```rust
use tokio::sync::mpsc;

// Create a channel for event communication
let (tx, mut rx) = mpsc::channel::<String>(100);

// Producer (in an event handler)
let tx_clone = tx.clone();
// ... inside handler: tx_clone.send("player joined".to_string()).await;

// Consumer (in a background task)
tokio::spawn(async move {
    while let Some(msg) = rx.recv().await {
        println!("Event: {msg}");
    }
});
```

---

## Configuration Files

Pumpkin uses TOML for configuration (instead of YAML). The `serde` library handles converting between Rust structs and config files — you define your config as a struct, and serde does the rest:

```rust
use serde::{Deserialize, Serialize};

// The #[derive(Serialize, Deserialize)] macros automatically generate
// code to convert this struct to/from TOML. It's like Jackson in Java,
// but happens at compile time instead of runtime.
#[derive(Serialize, Deserialize)]
struct PluginConfig {
    welcome_message: String,
    max_homes: u32,
    features: FeatureConfig,
}

#[derive(Serialize, Deserialize)]
struct FeatureConfig {
    enable_fly: bool,
    enable_warps: bool,
}

impl Default for PluginConfig {
    fn default() -> Self {
        Self {
            welcome_message: "Welcome to the server!".to_string(),
            max_homes: 3,
            features: FeatureConfig {
                enable_fly: true,
                enable_warps: true,
            },
        }
    }
}

// Loading/saving config
fn load_config(data_folder: &std::path::Path) -> PluginConfig {
    let config_path = data_folder.join("config.toml");

    if config_path.exists() {
        let content = std::fs::read_to_string(&config_path)
            .expect("Failed to read config");
        toml::from_str(&content)
            .expect("Failed to parse config")
    } else {
        let config = PluginConfig::default();
        let content = toml::to_string_pretty(&config)
            .expect("Failed to serialize config");
        std::fs::write(&config_path, content)
            .expect("Failed to write default config");
        config
    }
}
```

#### Java Comparison

```java
// Java
saveDefaultConfig();
String welcomeMsg = getConfig().getString("welcome-message", "Welcome!");
int maxHomes = getConfig().getInt("max-homes", 3);
```

---

## Plugin Dependencies

### Specifying Dependencies in `Cargo.toml`

```toml
[dependencies]
pumpkin = { path = "../Pumpkin/pumpkin" }
pumpkin-api-macros = { path = "../Pumpkin/pumpkin-api-macros" }
pumpkin-util = { path = "../Pumpkin/pumpkin-util" }
pumpkin-macros = { path = "../Pumpkin/pumpkin-macros" }

# Third-party crates
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
toml = "0.8"
uuid = { version = "1", features = ["v4"] }
```

### Soft Dependencies (Optional Integrations)

```rust
#[plugin_method]
pub fn on_load(&mut self, server: Arc<Context>) -> Result<(), String> {
    // Check if optional dependency is available
    if server.plugin_manager.is_plugin_active("economy-plugin").await {
        server.log("Economy integration enabled!");
        // Set up economy hooks...
    } else {
        server.log("Running without economy support.");
    }

    Ok(())
}
```

---

## What's Next?

In [Stage 10: Common Recipes](10-common-recipes.md), you'll find ready-to-use translations of popular Java plugin patterns into Pumpkin's Rust API.

---

[← Permissions](08-permissions.md) | [Back to Index](README.md) | [Next: Common Recipes →](10-common-recipes.md)
