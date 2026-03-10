# Stage 8: Permissions

## The Permission System

Pumpkin has a permission system that combines Minecraft's operator levels with custom permission nodes. If you've used Bukkit's permission system or permission plugins like LuckPerms, the concepts will be familiar.

### Operator Levels vs Custom Permissions

Pumpkin supports both:

1. **Operator Levels** (`PermissionLvl`) — The vanilla Minecraft permission system (levels 0–4)
2. **Custom Permission Nodes** — String-based permissions like `myplugin:commands.heal`

---

## Operator Levels

Minecraft has a built-in system of operator (OP) levels from 0 to 4. Each level grants progressively more server access:

- **Level 0** — Default player (no special permissions)
- **Level 1** — Can bypass spawn protection
- **Level 2** — Can use gameplay commands (`/gamemode`, `/tp`, etc.)
- **Level 3** — Can use multiplayer management commands (`/ban`, `/kick`, etc.)
- **Level 4** — Can use server management commands (`/stop`, `/save-all`, etc.)

### Checking Operator Level

```rust
// In a command executor — check if the sender has OP level 2+
if sender.has_permission_lvl(PermissionLvl::Two) {
    // Has OP level 2 or higher
}

// On a player reference
if player.has_permission_lvl(PermissionLvl::Three) {
    // Player is OP level 3+
}
```

### In Command Trees (Require Nodes)

```rust
use pumpkin::command::tree::builder::require;

// Only OP level 2+ can use this command
CommandTree::new(["heal"], "Heal a player")
    .then(
        require(|sender| sender.has_permission_lvl(PermissionLvl::Two))
            .execute(HealExecutor)
    )
```

#### Java Comparison

```java
// Java
if (player.isOp()) { ... }
if (player.hasPermission("minecraft.command.gamemode")) { ... }
```

---

## Custom Permissions

For more granular control, Pumpkin supports custom permission nodes through the `PermissionManager`.

### Registering Permissions

Register your custom permissions during plugin load:

```rust
use pumpkin_util::permission::{Permission, PermissionDefault};

#[plugin_method]
pub fn on_load(&mut self, server: Arc<Context>) -> Result<(), String> {
    // Register a simple permission
    server.register_permission(
        Permission::new("myplugin:admin", "Access admin features", PermissionDefault::Deny)
    ).await.map_err(|e| format!("Failed to register permission: {e}"))?;

    // Register command-specific permissions
    server.register_permission(
        Permission::new("myplugin:commands.heal", "Allow using /heal command", PermissionDefault::Deny)
    ).await.map_err(|e| format!("Failed to register permission: {e}"))?;

    server.register_permission(
        Permission::new("myplugin:commands.warp", "Allow using /warp command", PermissionDefault::Allow)
    ).await.map_err(|e| format!("Failed to register permission: {e}"))?;

    Ok(())
}
```

#### Java Comparison

```yaml
# Java: plugin.yml
permissions:
  myplugin.admin:
    description: Access admin features
    default: false
  myplugin.commands.heal:
    description: Allow using /heal command
    default: op
  myplugin.commands.warp:
    description: Allow using /warp command
    default: true
```

### Checking Permissions

```rust
// Check via the Context
let has_perm = server.player_has_permission(
    &player.gameprofile.id,
    "myplugin:admin"
).await;

// In a CommandSender context
let has_perm = sender.has_permission(server, "myplugin:commands.heal").await;
```

### In Command Registrations

When registering a command, you specify the required permission:

```rust
// The second argument is the permission node required
server.register_command(
    init_heal_tree(),
    "myplugin:commands.heal",
).await;
```

### In Require Nodes

```rust
CommandTree::new(["admin"], "Admin commands")
    .then(
        require(|sender| {
            // Check operator level as a baseline
            sender.has_permission_lvl(PermissionLvl::Two)
        })
        .execute(AdminExecutor)
    )
```

---

## Permission Check Events

The `PlayerPermissionCheckEvent` fires whenever a permission is checked, allowing you to dynamically grant or deny permissions:

```rust
use pumpkin::plugin::api::events::player::player_permission_check::PlayerPermissionCheckEvent;

struct DynamicPermissions;

impl EventHandler<PlayerPermissionCheckEvent> for DynamicPermissions {
    fn handle_blocking<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a mut PlayerPermissionCheckEvent,
    ) -> BoxFuture<'a, ()> {
        Box::pin(async move {
            // Dynamically grant permissions based on custom logic
            // For example: VIP players get extra permissions
            if is_vip(&event.player) && event.permission.starts_with("myplugin:vip.") {
                event.result = true;
            }
        })
    }
}
```

#### Java Comparison

```java
// Java (Vault-style)
@EventHandler
public void onPermissionCheck(PermissionCheckEvent event) {
    if (isVip(event.getPlayer()) && event.getPermission().startsWith("myplugin.vip.")) {
        event.setResult(PermissionResult.ALLOW);
    }
}
```

---

## Practical Examples

### Example 1: Rank-Based Permissions

```rust
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::RwLock;

struct RankManager {
    // UUID -> rank name
    player_ranks: Arc<RwLock<HashMap<uuid::Uuid, String>>>,
    // rank name -> list of permissions
    rank_permissions: HashMap<String, Vec<String>>,
}

impl RankManager {
    fn new() -> Self {
        let mut rank_permissions = HashMap::new();
        rank_permissions.insert("default".into(), vec![
            "myplugin.commands.warp".into(),
            "myplugin.commands.spawn".into(),
        ]);
        rank_permissions.insert("vip".into(), vec![
            "myplugin.commands.warp".into(),
            "myplugin.commands.spawn".into(),
            "myplugin.commands.fly".into(),
            "myplugin.vip.coloredchat".into(),
        ]);
        rank_permissions.insert("admin".into(), vec![
            "myplugin.*".into(), // Wildcard — all permissions
        ]);

        Self {
            player_ranks: Arc::new(RwLock::new(HashMap::new())),
            rank_permissions,
        }
    }

    fn has_permission(&self, rank: &str, permission: &str) -> bool {
        if let Some(perms) = self.rank_permissions.get(rank) {
            perms.iter().any(|p| {
                p == permission || p.ends_with(".*") && permission.starts_with(
                    &p[..p.len() - 2]
                )
            })
        } else {
            false
        }
    }
}
```

### Example 2: Permission-Gated Features in Commands

```rust
struct AdminCommandExecutor;

impl CommandExecutor for AdminCommandExecutor {
    fn execute<'a>(
        &'a self,
        sender: &'a CommandSender,
        server: &'a pumpkin::server::Server,
        _args: &'a ConsumedArgs<'a>,
    ) -> pumpkin::command::CommandResult<'a> {
        Box::pin(async move {
            // Check custom permission
            if !sender.has_permission(server, "myplugin:admin").await {
                return Err(CommandError::PermissionDenied);
            }

            // Or check operator level
            if !sender.has_permission_lvl(PermissionLvl::Three) {
                sender.send_message(
                    TextComponent::text("§cYou need OP level 3 for this!")
                ).await;
                return Err(CommandError::PermissionDenied);
            }

            // Execute admin action...
            sender.send_message(
                TextComponent::text("§aAdmin action executed!")
            ).await;
            Ok(1)
        })
    }
}
```

---

## What's Next?

In [Stage 9: Advanced Topics](09-advanced-topics.md), you'll learn about services for inter-plugin communication, custom plugin loaders, and async programming patterns.

---

[← Blocks & World](07-blocks-and-world.md) | [Back to Index](README.md) | [Next: Advanced Topics →](09-advanced-topics.md)
