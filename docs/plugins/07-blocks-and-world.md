# Stage 7: Blocks & World

## Block Events

Pumpkin provides several block-related events for monitoring and controlling block interactions. If you've worked with Bukkit's `BlockListener`, these will feel familiar.

### Block Break Event

The `BlockBreakEvent` fires when a block is about to be broken:

```rust
use pumpkin::plugin::api::events::block::block_break::BlockBreakEvent;

struct BreakHandler;

impl EventHandler<BlockBreakEvent> for BreakHandler {
    fn handle_blocking<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a mut BlockBreakEvent,
    ) -> BoxFuture<'a, ()> {
        Box::pin(async move {
            // Access event fields
            let block = event.block;           // &'static Block — the block being broken
            let pos = &event.block_position;   // BlockPos — position in the world
            let exp = event.exp;               // u32 — XP to drop
            let drops = event.drop;            // bool — whether items drop

            // Optional: the player who broke it (None for non-player sources)
            if let Some(player) = &event.player {
                println!("{} broke {} at {:?}",
                    player.gameprofile.name,
                    block.name,
                    pos
                );
            }

            // Modify XP drops
            event.exp = 0;

            // Prevent item drops
            event.drop = false;

            // Or cancel entirely
            // event.set_cancelled(true);
        })
    }
}
```

#### Java Comparison

```java
@EventHandler
public void onBlockBreak(BlockBreakEvent event) {
    Block block = event.getBlock();
    Player player = event.getPlayer();
    event.setExpToDrop(0);
    event.setDropItems(false);
    // event.setCancelled(true);
}
```

### Block Place Event

The `BlockPlaceEvent` fires when a block is about to be placed:

```rust
use pumpkin::plugin::api::events::block::block_place::BlockPlaceEvent;

struct PlaceHandler;

impl EventHandler<BlockPlaceEvent> for PlaceHandler {
    fn handle_blocking<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a mut BlockPlaceEvent,
    ) -> BoxFuture<'a, ()> {
        Box::pin(async move {
            let player = &event.player;              // Arc<Player>
            let placed = event.block_placed;         // &'static Block — block being placed
            let against = event.block_placed_against;// &'static Block — adjacent block
            let pos = &event.block_position;         // BlockPos
            let can_build = event.can_build;         // bool

            println!("{} placed {} against {} at {:?}",
                player.gameprofile.name,
                placed.name,
                against.name,
                pos
            );
        })
    }
}
```

#### Java Comparison

```java
@EventHandler
public void onBlockPlace(BlockPlaceEvent event) {
    Block placed = event.getBlockPlaced();
    Block against = event.getBlockAgainst();
    Player player = event.getPlayer();
}
```

### Block Burn Event

Fires when a block is destroyed by fire:

```rust
use pumpkin::plugin::api::events::block::block_burn::BlockBurnEvent;

struct BurnHandler;

impl EventHandler<BlockBurnEvent> for BurnHandler {
    fn handle_blocking<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a mut BlockBurnEvent,
    ) -> BoxFuture<'a, ()> {
        Box::pin(async move {
            // Prevent fire spread/damage
            event.set_cancelled(true);
        })
    }
}
```

### Block Grow Event

Fires when a crop or tree grows:

```rust
use pumpkin::plugin::api::events::block::block_grow::BlockGrowEvent;

struct GrowHandler;

impl EventHandler<BlockGrowEvent> for GrowHandler {
    fn handle<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a BlockGrowEvent,
    ) -> BoxFuture<'a, ()> {
        Box::pin(async move {
            println!("Block grew at {:?}", event.block_pos);
        })
    }
}
```

### Block Redstone Event

Fires when a block's redstone signal changes:

```rust
use pumpkin::plugin::api::events::block::block_redstone::BlockRedstoneEvent;

struct RedstoneHandler;

impl EventHandler<BlockRedstoneEvent> for RedstoneHandler {
    fn handle<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a BlockRedstoneEvent,
    ) -> BoxFuture<'a, ()> {
        Box::pin(async move {
            println!(
                "Redstone changed at {:?}: {} -> {}",
                event.block_pos,
                event.old_current,
                event.new_current
            );
        })
    }
}
```

---

## World Events

### Chunk Load Event

Fires when a chunk is loaded into memory:

```rust
use pumpkin::plugin::api::events::world::chunk_load::ChunkLoad;

struct ChunkLoadHandler;

impl EventHandler<ChunkLoad> for ChunkLoadHandler {
    fn handle<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a ChunkLoad,
    ) -> BoxFuture<'a, ()> {
        Box::pin(async move {
            // Access chunk coordinates through the chunk data
            let chunk = event.chunk.read().await;
            println!("Chunk loaded at ({}, {})", chunk.x, chunk.z);
        })
    }
}
```

### Chunk Save Event

Fires when a chunk is saved to disk:

```rust
use pumpkin::plugin::api::events::world::chunk_save::ChunkSave;

struct ChunkSaveHandler;

impl EventHandler<ChunkSave> for ChunkSaveHandler {
    fn handle<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a ChunkSave,
    ) -> BoxFuture<'a, ()> {
        Box::pin(async move {
            let chunk = event.chunk.read().await;
            println!("Chunk saved at ({}, {})", chunk.x, chunk.z);
        })
    }
}
```

### Chunk Send Event

Fires when chunk data is sent to a player (cancellable):

```rust
use pumpkin::plugin::api::events::world::chunk_send::ChunkSend;

struct ChunkSendHandler;

impl EventHandler<ChunkSend> for ChunkSendHandler {
    fn handle_blocking<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a mut ChunkSend,
    ) -> BoxFuture<'a, ()> {
        Box::pin(async move {
            // Prevent sending specific chunks (e.g., hidden areas)
            // event.set_cancelled(true);
        })
    }
}
```

### Spawn Change Event

Fires when the world spawn point changes:

```rust
use pumpkin::plugin::api::events::world::spawn_change::SpawnChangeEvent;

struct SpawnHandler;

impl EventHandler<SpawnChangeEvent> for SpawnHandler {
    fn handle<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a SpawnChangeEvent,
    ) -> BoxFuture<'a, ()> {
        Box::pin(async move {
            println!("World spawn changed to {:?}", event.new_position);
        })
    }
}
```

---

## Practical Examples

### Example 1: Logging Block Changes

A simple block logging plugin that tracks who placed and broke blocks:

```rust
use std::sync::Arc;
use std::fs::OpenOptions;
use std::io::Write;
use chrono::Local;

use pumpkin::plugin::api::events::block::block_break::BlockBreakEvent;
use pumpkin::plugin::api::events::block::block_place::BlockPlaceEvent;
use pumpkin::plugin::api::events::EventPriority;
use pumpkin::plugin::EventHandler;
use pumpkin::server::Server;

struct BlockLogger {
    log_path: std::path::PathBuf,
}

impl EventHandler<BlockBreakEvent> for BlockLogger {
    fn handle<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a BlockBreakEvent,
    ) -> futures::future::BoxFuture<'a, ()> {
        Box::pin(async move {
            let player_name = event.player.as_ref()
                .map(|p| p.gameprofile.name.clone())
                .unwrap_or_else(|| "Unknown".to_string());

            let entry = format!(
                "[BREAK] {} broke {} at {:?}\n",
                player_name,
                event.block.name,
                event.block_position
            );

            if let Ok(mut file) = OpenOptions::new()
                .create(true)
                .append(true)
                .open(&self.log_path)
            {
                let _ = file.write_all(entry.as_bytes());
            }
        })
    }
}

impl EventHandler<BlockPlaceEvent> for BlockLogger {
    fn handle<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a BlockPlaceEvent,
    ) -> futures::future::BoxFuture<'a, ()> {
        Box::pin(async move {
            let entry = format!(
                "[PLACE] {} placed {} at {:?}\n",
                event.player.gameprofile.name,
                event.block_placed.name,
                event.block_position
            );

            if let Ok(mut file) = OpenOptions::new()
                .create(true)
                .append(true)
                .open(&self.log_path)
            {
                let _ = file.write_all(entry.as_bytes());
            }
        })
    }
}
```

#### Java Comparison

```java
// Java (CoreProtect-style logging)
@EventHandler
public void onBlockBreak(BlockBreakEvent event) {
    logAction("BREAK", event.getPlayer(), event.getBlock());
}

@EventHandler
public void onBlockPlace(BlockPlaceEvent event) {
    logAction("PLACE", event.getPlayer(), event.getBlockPlaced());
}
```

### Example 2: Restricted Blocks

Prevent specific blocks from being placed:

```rust
struct RestrictedBlocks {
    banned_blocks: Vec<String>,
}

impl EventHandler<BlockPlaceEvent> for RestrictedBlocks {
    fn handle_blocking<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a mut BlockPlaceEvent,
    ) -> futures::future::BoxFuture<'a, ()> {
        Box::pin(async move {
            if self.banned_blocks.iter().any(|b| b == event.block_placed.name) {
                let player = event.player.clone();
                let block_name = event.block_placed.name;
                event.set_cancelled(true);
                player.send_system_message(
                    &pumpkin_util::text::TextComponent::text(
                        format!("§cYou cannot place {block_name}!")
                    )
                ).await;
            }
        })
    }
}

// Registration:
let handler = Arc::new(RestrictedBlocks {
    banned_blocks: vec!["tnt".into(), "bedrock".into(), "barrier".into()],
});
server.register_event::<BlockPlaceEvent, _>(handler, EventPriority::High, true).await;
```

### Example 3: Double-Drop Mining

Make certain blocks drop double items:

```rust
struct DoubleDropHandler;

impl EventHandler<BlockBreakEvent> for DoubleDropHandler {
    fn handle_blocking<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a mut BlockBreakEvent,
    ) -> futures::future::BoxFuture<'a, ()> {
        Box::pin(async move {
            // Double the XP for ore blocks
            if event.block.name.contains("ore") {
                event.exp *= 2;
            }
        })
    }
}
```

---

## What's Next?

In [Stage 8: Permissions](08-permissions.md), you'll learn how to create and check custom permissions for your plugin.

---

[← Player Interactions](06-player-interactions.md) | [Back to Index](README.md) | [Next: Permissions →](08-permissions.md)
