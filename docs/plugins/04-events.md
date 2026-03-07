# Stage 4: Events

## The Event System

Pumpkin's event system is inspired by the Bukkit event system but adapted for Rust's type system and async runtime. If you've used `@EventHandler` in Java, you'll find the concepts familiar.

### Java vs Pumpkin Event Comparison

```java
// Java (Bukkit)
public class MyListener implements Listener {
    @EventHandler(priority = EventPriority.NORMAL)
    public void onPlayerJoin(PlayerJoinEvent event) {
        event.setJoinMessage("Welcome, " + event.getPlayer().getName() + "!");
    }
}

// Registration:
getServer().getPluginManager().registerEvents(new MyListener(), this);
```

```rust
// Rust (Pumpkin)
use std::sync::Arc;
use pumpkin::plugin::EventHandler;
use pumpkin::plugin::api::events::player::PlayerJoinEvent;
use pumpkin::server::Server;

struct MyJoinHandler;

impl EventHandler<PlayerJoinEvent> for MyJoinHandler {
    fn handle_blocking<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a mut PlayerJoinEvent,
    ) -> futures::future::BoxFuture<'a, ()> {
        Box::pin(async move {
            event.join_message = TextComponent::text(
                format!("Welcome, {}!", event.player.gameprofile.name)
            );
        })
    }
}

// Registration (inside on_load):
server.register_event::<PlayerJoinEvent, _>(
    Arc::new(MyJoinHandler),
    EventPriority::Normal,
    true, // blocking — we need to modify the event
).await;
```

---

## Core Concepts

### The `Payload` Trait

Every event in Pumpkin implements the `Payload` trait, which provides type identification and downcasting:

```rust
pub trait Payload: Send + Sync {
    fn get_name_static() -> &'static str where Self: Sized;
    fn get_name(&self) -> &'static str;
    fn as_any(&self) -> &dyn Any;
    fn as_any_mut(&mut self) -> &mut dyn Any;
}
```

Events use name-based identification instead of `TypeId` to support safe cross-compilation boundary downcasting. You don't need to implement this manually — the `#[derive(Event)]` macro handles it.

### The `EventHandler<E>` Trait

This is what you implement to react to events:

```rust
pub trait EventHandler<E: Payload>: Send + Sync {
    /// Non-blocking handler — read-only access to the event
    fn handle<'a>(
        &'a self,
        server: &'a Arc<Server>,
        event: &'a E,
    ) -> BoxFuture<'a, ()> {
        Box::pin(async {})
    }

    /// Blocking handler — mutable access, can modify the event
    fn handle_blocking<'a>(
        &'a self,
        server: &'a Arc<Server>,
        event: &'a mut E,
    ) -> BoxFuture<'a, ()> {
        Box::pin(async {})
    }
}
```

### Blocking vs Non-Blocking Handlers

This is a key distinction that doesn't exist in Bukkit:

| Type | Access | Execution | Use When |
|------|--------|-----------|----------|
| **Non-blocking** (`handle`) | Read-only (`&E`) | Parallel with other non-blocking handlers | Logging, analytics, notifications |
| **Blocking** (`handle_blocking`) | Mutable (`&mut E`) | Sequential, before non-blocking handlers | Modifying event data, cancelling events |

```rust
// Non-blocking: just log the event (read-only)
impl EventHandler<PlayerJoinEvent> for LogHandler {
    fn handle<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a PlayerJoinEvent,
    ) -> BoxFuture<'a, ()> {
        Box::pin(async move {
            println!("Player joined: {}", event.player.gameprofile.name);
        })
    }
}

// Blocking: modify the join message (mutable)
impl EventHandler<PlayerJoinEvent> for MessageHandler {
    fn handle_blocking<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a mut PlayerJoinEvent,
    ) -> BoxFuture<'a, ()> {
        Box::pin(async move {
            event.join_message = TextComponent::text("A hero has arrived!");
        })
    }
}
```

#### Java Comparison

In Bukkit, all handlers are effectively "blocking" — they run sequentially and can always modify the event. Pumpkin's split design allows better concurrency: read-only handlers can run in parallel, improving performance.

---

## Event Priority

Priorities control the order handlers execute:

```rust
pub enum EventPriority {
    Highest,  // Executes first
    High,
    Normal,   // Default
    Low,
    Lowest,   // Executes last — gets final say
}
```

> **Important:** In Pumpkin, `Highest` runs **first** and `Lowest` runs **last**. This matches Bukkit's behavior where `LOWEST` has the "last word" on event modifications.

### Registration with Priority

```rust
// High priority — runs early, good for protection plugins
server.register_event::<BlockBreakEvent, _>(
    Arc::new(ProtectionHandler),
    EventPriority::High,
    true, // blocking
).await;

// Normal priority — default for most plugins
server.register_event::<BlockBreakEvent, _>(
    Arc::new(LoggingHandler),
    EventPriority::Normal,
    false, // non-blocking
).await;
```

---

## Cancellable Events

Many events can be cancelled to prevent their default behavior, just like in Bukkit:

```rust
pub trait Cancellable: Send + Sync {
    fn cancelled(&self) -> bool;
    fn set_cancelled(&mut self, cancelled: bool);
}
```

### Cancelling an Event

```rust
use pumpkin::plugin::api::events::Cancellable;

struct AntiGriefHandler;

impl EventHandler<BlockBreakEvent> for AntiGriefHandler {
    fn handle_blocking<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a mut BlockBreakEvent,
    ) -> BoxFuture<'a, ()> {
        Box::pin(async move {
            // Cancel block breaking in protected areas
            if is_protected_area(&event.block_position) {
                event.set_cancelled(true);
            }
        })
    }
}
```

#### Java Comparison

```java
// Java
@EventHandler
public void onBlockBreak(BlockBreakEvent event) {
    if (isProtectedArea(event.getBlock().getLocation())) {
        event.setCancelled(true);
    }
}
```

### Checking if an Event was Cancelled

```rust
impl EventHandler<PlayerChatEvent> for ChatHandler {
    fn handle_blocking<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a mut PlayerChatEvent,
    ) -> BoxFuture<'a, ()> {
        Box::pin(async move {
            if event.cancelled() {
                return; // Another plugin already cancelled this
            }
            // Process the chat message
        })
    }
}
```

---

## Available Events

### Player Events

| Event | Cancellable | Description |
|-------|:-----------:|-------------|
| `PlayerJoinEvent` | ✅ | Player joins the server |
| `PlayerLeaveEvent` | ✅ | Player leaves the server |
| `PlayerChatEvent` | ✅ | Player sends a chat message |
| `PlayerMoveEvent` | ✅ | Player changes position |
| `PlayerTeleportEvent` | ✅ | Player teleports |
| `PlayerLoginEvent` | ✅ | Player attempts to log in |
| `PlayerCommandSendEvent` | ✅ | Player sends a command |
| `PlayerGamemodeChangeEvent` | ✅ | Player's gamemode changes |
| `PlayerInteractEvent` | ✅ | Player interacts with a block |
| `PlayerInteractEntityEvent` | ✅ | Player interacts with an entity |
| `PlayerPermissionCheckEvent` | ✅ | Permission check occurs |
| `PlayerChangeWorldEvent` | — | Player changes worlds |
| `PlayerExpChangeEvent` | — | Player's experience changes |
| `PlayerItemHeldEvent` | ✅ | Player changes held item slot |
| `PlayerChangedMainHandEvent` | — | Player switches main hand |
| `PlayerCustomPayloadEvent` | ✅ | Custom plugin channel message |
| `EggThrowEvent` | — | Player throws an egg |
| `PlayerFishEvent` | ✅ | Player fishes |

### Block Events

| Event | Cancellable | Description |
|-------|:-----------:|-------------|
| `BlockBreakEvent` | ✅ | Block is broken |
| `BlockPlaceEvent` | ✅ | Block is placed |
| `BlockBurnEvent` | ✅ | Block burns |
| `BlockGrowEvent` | ✅ | Block grows (crops, trees) |
| `BlockCanBuildEvent` | — | Check if block can be built at location |
| `BlockRedstoneEvent` | — | Redstone signal changes |

### World Events

| Event | Cancellable | Description |
|-------|:-----------:|-------------|
| `ChunkLoadEvent` | — | Chunk is loaded |
| `ChunkSaveEvent` | — | Chunk is saved |
| `ChunkSendEvent` | ✅ | Chunk data sent to player |
| `SpawnChangeEvent` | — | World spawn point changes |

### Server Events

| Event | Cancellable | Description |
|-------|:-----------:|-------------|
| `ServerCommandEvent` | ✅ | Console command is executed |
| `ServerBroadcastEvent` | — | Server broadcasts a message |

---

## Event Examples

### Example 1: Custom Join/Leave Messages

```rust
struct JoinLeaveHandler;

impl EventHandler<PlayerJoinEvent> for JoinLeaveHandler {
    fn handle_blocking<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a mut PlayerJoinEvent,
    ) -> BoxFuture<'a, ()> {
        Box::pin(async move {
            let name = &event.player.gameprofile.name;
            event.join_message = TextComponent::text(
                format!("§a+ §f{name} joined the game")
            );
        })
    }
}

impl EventHandler<PlayerLeaveEvent> for JoinLeaveHandler {
    fn handle_blocking<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a mut PlayerLeaveEvent,
    ) -> BoxFuture<'a, ()> {
        Box::pin(async move {
            let name = &event.player.gameprofile.name;
            event.leave_message = TextComponent::text(
                format!("§c- §f{name} left the game")
            );
        })
    }
}
```

### Example 2: Chat Filter

```rust
struct ChatFilter {
    blocked_words: Vec<String>,
}

impl EventHandler<PlayerChatEvent> for ChatFilter {
    fn handle_blocking<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a mut PlayerChatEvent,
    ) -> BoxFuture<'a, ()> {
        Box::pin(async move {
            let msg_lower = event.message.to_lowercase();
            for word in &self.blocked_words {
                if msg_lower.contains(word) {
                    event.set_cancelled(true);
                    // Optionally notify the player
                    return;
                }
            }
        })
    }
}

// Registration
let filter = Arc::new(ChatFilter {
    blocked_words: vec!["spam".into(), "badword".into()],
});
server.register_event::<PlayerChatEvent, _>(
    filter,
    EventPriority::High,
    true,
).await;
```

### Example 3: Movement Tracker

```rust
use pumpkin::plugin::api::events::player::PlayerMoveEvent;

struct MovementTracker;

impl EventHandler<PlayerMoveEvent> for MovementTracker {
    fn handle<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a PlayerMoveEvent,
    ) -> BoxFuture<'a, ()> {
        Box::pin(async move {
            let dx = event.to.x - event.from.x;
            let dy = event.to.y - event.from.y;
            let dz = event.to.z - event.from.z;
            let distance = (dx * dx + dy * dy + dz * dz).sqrt();

            if distance > 10.0 {
                println!(
                    "Suspicious movement by {}: {:.2} blocks",
                    event.player.gameprofile.name, distance
                );
            }
        })
    }
}
```

### Example 4: Block Break with XP Drop Prevention

```rust
use pumpkin::plugin::api::events::block::BlockBreakEvent;

struct NoXpHandler;

impl EventHandler<BlockBreakEvent> for NoXpHandler {
    fn handle_blocking<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a mut BlockBreakEvent,
    ) -> BoxFuture<'a, ()> {
        Box::pin(async move {
            // Prevent XP from dropping when blocks are broken
            event.exp = 0;
            // Or prevent item drops
            // event.drop = false;
        })
    }
}
```

---

## Registering Multiple Events

You can register as many event handlers as you need in `on_load`:

```rust
#[plugin_method]
pub fn on_load(&mut self, server: Arc<Context>) -> Result<(), String> {
    // Register join handler
    server.register_event::<PlayerJoinEvent, _>(
        Arc::new(JoinHandler),
        EventPriority::Normal,
        true,
    ).await;

    // Register chat handler
    server.register_event::<PlayerChatEvent, _>(
        Arc::new(ChatHandler),
        EventPriority::High,
        true,
    ).await;

    // Register block handler (non-blocking, just logging)
    server.register_event::<BlockBreakEvent, _>(
        Arc::new(BlockLogHandler),
        EventPriority::Normal,
        false,
    ).await;

    server.log("All event handlers registered!");
    Ok(())
}
```

---

## What's Next?

In [Stage 5: Commands](05-commands.md), you'll learn how to create custom commands with arguments, tab completion, and permissions.

---

[← Plugin Lifecycle](03-plugin-lifecycle.md) | [Back to Index](README.md) | [Next: Commands →](05-commands.md)
