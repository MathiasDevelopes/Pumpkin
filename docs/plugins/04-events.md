# Stage 4: Events

## The Event System

Events are the heart of Minecraft plugin development. When a player joins, breaks a block, or sends a chat message, the server fires an event — and your plugin can react to it.

Pumpkin's event system works very similarly to Bukkit's: you create a handler, register it, and your code runs whenever the event fires. The syntax is a bit different, but the concepts are the same.

### Java vs Pumpkin: At a Glance

```java
// Java (Bukkit) — annotate a method
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
// Rust (Pumpkin) — implement a trait on a struct
struct MyJoinHandler;

impl EventHandler<PlayerJoinEvent> for MyJoinHandler {
    fn handle_blocking<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a mut PlayerJoinEvent,
    ) -> BoxFuture<'a, ()> {
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
    true, // blocking — we want to modify the event
).await;
```

The biggest difference: in Java you annotate a *method*, in Rust you implement a *trait* on a *struct*. The struct is your handler, and the trait method is where your logic goes.

> **New to Rust?** The `<'a>` (called a "lifetime") and `BoxFuture` syntax might look scary. Don't worry about understanding them deeply right now — this is boilerplate that follows the same pattern every time. Think of it as the Rust equivalent of `@EventHandler public void ...`. Focus on the code inside `Box::pin(async move { ... })` — that's where your logic lives.

---

## Two Types of Handlers

Pumpkin has a concept that doesn't exist in Bukkit: **blocking** vs **non-blocking** handlers.

### Non-Blocking Handlers (Read-Only)

Use these when you just want to *observe* an event without changing it — like logging, analytics, or sending notifications:

```rust
// Non-blocking: just log the event
struct LogHandler;

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

// Register as non-blocking
server.register_event::<PlayerJoinEvent, _>(
    Arc::new(LogHandler),
    EventPriority::Normal,
    false,  // non-blocking
).await;
```

The key method is `handle` — notice that `event` is `&'a PlayerJoinEvent` (a read-only reference). Multiple non-blocking handlers run **in parallel** for better performance.

### Blocking Handlers (Can Modify)

Use these when you need to *change* the event — like modifying a chat message, cancelling block placement, or changing a join message:

```rust
// Blocking: modify the join message
struct MessageHandler;

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

// Register as blocking
server.register_event::<PlayerJoinEvent, _>(
    Arc::new(MessageHandler),
    EventPriority::Normal,
    true,  // blocking
).await;
```

The key method is `handle_blocking` — notice `event` is `&'a mut PlayerJoinEvent` (a mutable reference). Blocking handlers run **sequentially**, before non-blocking ones.

| Handler Type | Method | Can Modify Event? | Execution |
|-------------|--------|:-----------------:|-----------|
| Non-blocking | `handle` | No (read-only) | Parallel — fast! |
| Blocking | `handle_blocking` | Yes | Sequential — safe! |

#### Why This Matters

In Bukkit, all handlers run sequentially and can always modify the event. This is simple, but it means event handling can't take advantage of multiple CPU cores. Pumpkin's split design means read-only operations (logging, analytics) run in parallel, while modifications run safely one at a time. This gives you better server performance for free.

---

## Event Priority

Just like Bukkit, you can control the order handlers execute:

- **Highest** — Executes first
- **High**
- **Normal** — Default, good for most plugins
- **Low**
- **Lowest** — Executes last, gets the final say

> **Tip:** Use `Highest` or `High` for protection plugins that need to cancel events early. Use `Normal` for most plugins. Use `Low` or `Lowest` if you want to see the final state of the event after other plugins have modified it.

```rust
// High priority — runs early, good for protection plugins
server.register_event::<BlockBreakEvent, _>(
    Arc::new(ProtectionHandler),
    EventPriority::High,
    true,
).await;

// Normal priority — default for most plugins
server.register_event::<BlockBreakEvent, _>(
    Arc::new(LoggingHandler),
    EventPriority::Normal,
    false,
).await;
```

---

## Cancelling Events

Many events can be **cancelled** to prevent their default behavior. This works just like `event.setCancelled(true)` in Bukkit:

```rust
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
@EventHandler
public void onBlockBreak(BlockBreakEvent event) {
    if (isProtectedArea(event.getBlock().getLocation())) {
        event.setCancelled(true);
    }
}
```

You can also check if another plugin already cancelled an event:

```rust
if event.cancelled() {
    return; // Another plugin already cancelled this
}
```

> **Note:** Only blocking handlers can cancel events, since cancellation requires modifying the event.

---

## Available Events

Here's a reference of all events you can listen to:

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

## Practical Examples

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

> **Rust tip:** In Rust, a single struct can implement the same trait for different type parameters. Here, `JoinLeaveHandler` implements `EventHandler<PlayerJoinEvent>` *and* `EventHandler<PlayerLeaveEvent>`. This is like having a single Java class that handles multiple event types — but type-safe.

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
                    return;
                }
            }
        })
    }
}

// Registration — the struct holds its configuration
let filter = Arc::new(ChatFilter {
    blocked_words: vec!["spam".into(), "badword".into()],
});
server.register_event::<PlayerChatEvent, _>(
    filter,
    EventPriority::High,
    true,
).await;
```

> **Pattern:** Notice how the handler struct holds data (`blocked_words`). This is how you give your handlers configuration or shared state — similar to passing data to a Bukkit listener through constructor parameters.

### Example 3: Movement Tracker

```rust
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

This handler uses `handle` (non-blocking) because it only reads the event data. It doesn't need to modify or cancel anything, so it can run in parallel with other handlers.

### Example 4: Prevent XP from Block Breaking

```rust
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
            // Or prevent item drops entirely:
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
    // Register join handler (blocking — modifies the message)
    server.register_event::<PlayerJoinEvent, _>(
        Arc::new(JoinHandler),
        EventPriority::Normal,
        true,
    ).await;

    // Register chat handler (blocking — can cancel messages)
    server.register_event::<PlayerChatEvent, _>(
        Arc::new(ChatHandler),
        EventPriority::High,
        true,
    ).await;

    // Register block handler (non-blocking — just logging)
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
