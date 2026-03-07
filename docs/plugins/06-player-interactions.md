# Stage 6: Player Interactions

## Working with Players

In Bukkit, you interact with players through the `Player` interface. In Pumpkin, players are represented by `Arc<Player>` — a thread-safe reference-counted pointer to the player object.

### Getting Player References

```rust
// From the Context (by name)
if let Some(player) = server.get_player_by_name("Steve") {
    // Use player...
}

// From an event
impl EventHandler<PlayerJoinEvent> for MyHandler {
    fn handle<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a PlayerJoinEvent,
    ) -> BoxFuture<'a, ()> {
        Box::pin(async move {
            let player = &event.player; // Arc<Player>
            let name = &player.gameprofile.name;
        })
    }
}

// From a CommandSender
if let Some(player) = sender.as_player() {
    // player is Arc<Player>
}
```

---

## Sending Messages

### System Messages

```rust
use pumpkin_util::text::TextComponent;

// Send a simple text message
player.send_system_message(
    &TextComponent::text("Hello, world!")
).await;

// Colored message (§ formatting codes work)
player.send_system_message(
    &TextComponent::text("§aThis is green! §cThis is red!")
).await;
```

#### Java Comparison

```java
// Java
player.sendMessage("§aThis is green! §cThis is red!");
player.sendMessage(Component.text("Hello!").color(NamedTextColor.GREEN));
```

### Action Bar Messages

```rust
// Send a message to the action bar (above hotbar)
player.send_actionbar_message(
    &TextComponent::text("§6+5 Gold")
).await;
```

#### Java Comparison

```java
// Java (Paper)
player.sendActionBar(Component.text("+5 Gold").color(NamedTextColor.GOLD));
```

### Broadcasting to All Players

```rust
// Broadcast to all online players via the server
server.server.broadcast_message(
    &TextComponent::text("§eServer announcement!"),
    &TextComponent::text("Server"),
    pumpkin_util::text::SayCommand::Say,
    None,
).await;
```

#### Java Comparison

```java
// Java
Bukkit.broadcastMessage("§eServer announcement!");
```

---

## Player Information

### Game Profile

```rust
// Player name
let name = &player.gameprofile.name;

// Player UUID
let uuid = &player.gameprofile.id;
```

### Position and World

```rust
// Get player's position
let position = player.living_entity.entity.pos.load();
let x = position.x;
let y = position.y;
let z = position.z;

// Get player's world
let world = player.living_entity.entity.world.clone();
```

#### Java Comparison

```java
// Java
Location loc = player.getLocation();
double x = loc.getX();
World world = player.getWorld();
```

### Gamemode

```rust
// Get current gamemode
let gamemode = player.gamemode.load();

// Set gamemode
use pumpkin_util::GameMode;
player.set_gamemode(GameMode::Creative).await;
player.set_gamemode(GameMode::Survival).await;
player.set_gamemode(GameMode::Spectator).await;
player.set_gamemode(GameMode::Adventure).await;
```

#### Java Comparison

```java
// Java
player.setGameMode(GameMode.CREATIVE);
GameMode mode = player.getGameMode();
```

---

## Teleportation

### Teleporting a Player

```rust
use pumpkin_util::math::vector3::Vector3;

// Teleport to coordinates
let position = Vector3::new(100.0, 64.0, 200.0);
player.teleport(position).await;
```

#### Java Comparison

```java
// Java
player.teleport(new Location(world, 100, 64, 200));
```

### Listening to Teleport Events

```rust
use pumpkin::plugin::api::events::player::PlayerTeleportEvent;

struct TeleportHandler;

impl EventHandler<PlayerTeleportEvent> for TeleportHandler {
    fn handle<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a PlayerTeleportEvent,
    ) -> BoxFuture<'a, ()> {
        Box::pin(async move {
            println!(
                "{} teleported from {:?} to {:?}",
                event.player.gameprofile.name,
                event.from,
                event.to,
            );
        })
    }
}
```

---

## Kicking Players

```rust
use pumpkin_util::text::TextComponent;

// Kick with a reason
player.kick(TextComponent::text("You have been kicked!")).await;

// Kick with colored reason
player.kick(
    TextComponent::text("§cBanned: §fCheating is not allowed")
).await;
```

#### Java Comparison

```java
// Java
player.kick(Component.text("You have been kicked!"));
```

---

## Player Events Quick Reference

Here are the most commonly used player events with practical examples:

### Handling Player Join

```rust
struct WelcomeHandler {
    motd: String,
}

impl EventHandler<PlayerJoinEvent> for WelcomeHandler {
    fn handle<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a PlayerJoinEvent,
    ) -> BoxFuture<'a, ()> {
        Box::pin(async move {
            let name = &event.player.gameprofile.name;

            // Send a welcome message to the joining player
            event.player.send_system_message(
                &TextComponent::text(format!("§6Welcome to the server, §e{name}§6!"))
            ).await;

            event.player.send_system_message(
                &TextComponent::text(format!("§7{}", self.motd))
            ).await;
        })
    }
}
```

### Handling Chat

```rust
struct ChatFormatter;

impl EventHandler<PlayerChatEvent> for ChatFormatter {
    fn handle_blocking<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a mut PlayerChatEvent,
    ) -> BoxFuture<'a, ()> {
        Box::pin(async move {
            let name = &event.player.gameprofile.name;

            // Prefix all messages with a rank
            event.message = format!("[Member] {name}: {}", event.message);
        })
    }
}
```

### Preventing Movement (Freeze)

```rust
struct FreezeHandler {
    frozen_players: Arc<RwLock<HashSet<uuid::Uuid>>>,
}

impl EventHandler<PlayerMoveEvent> for FreezeHandler {
    fn handle_blocking<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a mut PlayerMoveEvent,
    ) -> BoxFuture<'a, ()> {
        Box::pin(async move {
            let uuid = event.player.gameprofile.id;
            let frozen = self.frozen_players.read().await;
            if frozen.contains(&uuid) {
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
public void onMove(PlayerMoveEvent event) {
    if (frozenPlayers.contains(event.getPlayer().getUniqueId())) {
        event.setCancelled(true);
    }
}
```

---

## Complete Example: Spawn Protection Plugin

This example combines player events, block events, and commands to create a spawn protection system:

```rust
use std::sync::Arc;
use std::collections::HashSet;
use tokio::sync::RwLock;

use pumpkin::plugin::api::context::Context;
use pumpkin::plugin::api::events::block::{BlockBreakEvent, BlockPlaceEvent};
use pumpkin::plugin::api::events::EventPriority;
use pumpkin::plugin::EventHandler;
use pumpkin::server::Server;
use pumpkin_api_macros::{plugin_impl, plugin_method};
use pumpkin_util::math::position::BlockPos;

const SPAWN_RADIUS: f64 = 50.0;

fn is_near_spawn(pos: &BlockPos) -> bool {
    let dx = pos.0.x as f64;
    let dz = pos.0.z as f64;
    (dx * dx + dz * dz).sqrt() <= SPAWN_RADIUS
}

struct BreakProtection;
struct PlaceProtection;

impl EventHandler<BlockBreakEvent> for BreakProtection {
    fn handle_blocking<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a mut BlockBreakEvent,
    ) -> futures::future::BoxFuture<'a, ()> {
        Box::pin(async move {
            if is_near_spawn(&event.block_position) {
                if let Some(player) = &event.player {
                    // Allow OPs to break blocks
                    if !player.has_permission_lvl(pumpkin_util::PermissionLvl::Two) {
                        event.set_cancelled(true);
                        player.send_system_message(
                            &pumpkin_util::text::TextComponent::text(
                                "§cYou cannot break blocks near spawn!"
                            )
                        ).await;
                    }
                }
            }
        })
    }
}

impl EventHandler<BlockPlaceEvent> for PlaceProtection {
    fn handle_blocking<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a mut BlockPlaceEvent,
    ) -> futures::future::BoxFuture<'a, ()> {
        Box::pin(async move {
            if is_near_spawn(&event.block_position) {
                if !event.player.has_permission_lvl(pumpkin_util::PermissionLvl::Two) {
                    event.set_cancelled(true);
                    event.player.send_system_message(
                        &pumpkin_util::text::TextComponent::text(
                            "§cYou cannot place blocks near spawn!"
                        )
                    ).await;
                }
            }
        })
    }
}

#[plugin_impl]
pub struct SpawnProtectionPlugin;

impl SpawnProtectionPlugin {
    #[plugin_method]
    pub fn on_load(&mut self, server: Arc<Context>) -> Result<(), String> {
        server.register_event::<BlockBreakEvent, _>(
            Arc::new(BreakProtection),
            EventPriority::High,
            true,
        ).await;

        server.register_event::<BlockPlaceEvent, _>(
            Arc::new(PlaceProtection),
            EventPriority::High,
            true,
        ).await;

        server.log(format!(
            "Spawn protection enabled (radius: {} blocks)",
            SPAWN_RADIUS
        ));
        Ok(())
    }
}
```

---

## What's Next?

In [Stage 7: Blocks & World](07-blocks-and-world.md), we'll explore block and world events in more detail, including chunk management and world interaction.

---

[← Commands](05-commands.md) | [Back to Index](README.md) | [Next: Blocks & World →](07-blocks-and-world.md)
