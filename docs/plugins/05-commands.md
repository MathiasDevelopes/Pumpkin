# Stage 5: Commands

## The Command System

Pumpkin uses a tree-based command system that's different from Bukkit's `onCommand()` approach. Instead of parsing arguments manually, you build a `CommandTree` that declaratively defines your command's structure, arguments, and execution logic.

### Java vs Pumpkin Command Comparison

```java
// Java (Bukkit) — manual argument parsing
public class HealCommand implements CommandExecutor {
    @Override
    public boolean onCommand(CommandSender sender, Command cmd, String label, String[] args) {
        if (args.length == 0) {
            if (sender instanceof Player player) {
                player.setHealth(20.0);
                sender.sendMessage("You have been healed!");
            }
        } else {
            Player target = Bukkit.getPlayer(args[0]);
            if (target != null) {
                target.setHealth(20.0);
                sender.sendMessage("Healed " + target.getName());
            }
        }
        return true;
    }
}
```

```rust
// Rust (Pumpkin) — declarative tree builder
use pumpkin::command::CommandExecutor;
use pumpkin::command::args::ConsumedArgs;
use pumpkin::command::tree::CommandTree;
use pumpkin::command::tree::builder::argument;
use pumpkin::command::args::players::PlayersArgumentConsumer;

pub fn init_command_tree() -> CommandTree {
    CommandTree::new(["heal"], "Heal yourself or another player")
        .then(
            require(|sender| sender.is_player())
                .execute(HealSelfExecutor)
        )
        .then(
            argument("target", PlayersArgumentConsumer)
                .execute(HealTargetExecutor)
        )
}
```

---

## Building a `CommandTree`

The `CommandTree` builder API has four node types:

### 1. **Literal Nodes** — Fixed text arguments

```rust
use pumpkin::command::tree::builder::literal;

// /mycommand reload
// /mycommand status
CommandTree::new(["mycommand"], "My custom command")
    .then(literal("reload").execute(ReloadExecutor))
    .then(literal("status").execute(StatusExecutor))
```

#### Java Comparison

```java
// Java — manual string matching
if (args[0].equalsIgnoreCase("reload")) { ... }
else if (args[0].equalsIgnoreCase("status")) { ... }
```

### 2. **Argument Nodes** — Dynamic typed arguments

```rust
use pumpkin::command::tree::builder::argument;
use pumpkin::command::args::players::PlayersArgumentConsumer;

// /tp <target>
CommandTree::new(["tp"], "Teleport to a player")
    .then(argument("target", PlayersArgumentConsumer).execute(TpExecutor))
```

### 3. **Require Nodes** — Permission/condition gates

```rust
use pumpkin::command::tree::builder::require;

// Only players can use this command
CommandTree::new(["fly"], "Toggle flight")
    .then(
        require(|sender| sender.is_player())
            .execute(FlyExecutor)
    )
```

### 4. **Execute Leaves** — Terminal actions

```rust
// Execute at the root level (no arguments needed)
CommandTree::new(["ping"], "Check server latency")
    .execute(PingExecutor)
```

---

## The `CommandExecutor` Trait

Every command action implements this trait:

```rust
pub trait CommandExecutor: Sync + Send {
    fn execute<'a>(
        &'a self,
        sender: &'a CommandSender,
        server: &'a Server,
        args: &'a ConsumedArgs<'a>,
    ) -> CommandResult<'a>;
}

// CommandResult is a pinned future returning Result<i32, CommandError>
pub type CommandResult<'a> = Pin<Box<
    dyn Future<Output = Result<i32, CommandError>> + Send + 'a
>>;
```

The return value `i32` is the "success count" — similar to Minecraft's command success count used by command blocks. Return `1` for success, `0` for no-op.

---

## Argument Types

Pumpkin provides many built-in argument consumers:

| Consumer | Parses | Result `Arg` Variant |
|----------|--------|----------------------|
| `PlayersArgumentConsumer` | Player selector (`@a`, `@p`, name) | `Arg::Players(Vec<Arc<Player>>)` |
| `MsgArgConsumer` | Chat message (rest of input) | `Arg::Msg(String)` |
| `GamemodeArgumentConsumer` | Gamemode name/number | `Arg::GameMode(GameMode)` |
| `BlockPosArgumentConsumer` | Block coordinates (`x y z`) | `Arg::BlockPos(BlockPos)` |
| `Position3DArgumentConsumer` | 3D position (decimal) | `Arg::Pos3D(Vector3<f64>)` |
| `BoolArgumentConsumer` | `true`/`false` | `Arg::Bool(bool)` |
| `BoundedNumArgumentConsumer` | Numbers with range | `Arg::Num(Result<Number, NotInBounds>)` |
| `SimpleArgConsumer` | Raw string token | `Arg::Simple(&str)` |
| `ResourceLocationConsumer` | `namespace:path` | `Arg::ResourceLocation(&str)` |
| `ItemArgumentConsumer` | Item type | `Arg::Item(&str)` |
| `TimeArgumentConsumer` | Time value (`1d`, `5s`) | `Arg::Time(i32)` |

### Extracting Arguments

```rust
impl CommandExecutor for MyExecutor {
    fn execute<'a>(
        &'a self,
        sender: &'a CommandSender,
        server: &'a Server,
        args: &'a ConsumedArgs<'a>,
    ) -> CommandResult<'a> {
        Box::pin(async move {
            // Extract a player argument
            let Some(Arg::Players(targets)) = args.get("target") else {
                return Err(CommandError::InvalidConsumption(Some("target".into())));
            };

            // Extract a message argument
            let Some(Arg::Msg(message)) = args.get("message") else {
                return Err(CommandError::InvalidConsumption(Some("message".into())));
            };

            // Use the arguments
            for target in targets {
                target.send_system_message(
                    &TextComponent::text(message.clone())
                ).await;
            }

            Ok(targets.len() as i32)
        })
    }
}
```

---

## The `CommandSender`

The sender can be a player, console, command block, or RCON:

```rust
pub enum CommandSender {
    Rcon(Arc<tokio::sync::Mutex<Vec<String>>>),
    Console,
    Player(Arc<Player>),
    CommandBlock(Arc<CommandBlockEntity>, Arc<World>),
    Dummy,
}
```

### Useful `CommandSender` Methods

```rust
// Check sender type
sender.is_player();    // true if Player variant
sender.is_console();   // true if Console variant

// Get player (returns None for non-player senders)
if let Some(player) = sender.as_player() {
    // Use player...
}

// Send a message to the sender (works for all types)
sender.send_message(TextComponent::text("Hello!")).await;

// Check permissions
sender.has_permission_lvl(PermissionLvl::Two); // op level check
sender.has_permission(server, "my.permission").await; // custom permission
```

---

## Complete Command Examples

### Example 1: Simple `/ping` Command

```rust
use std::sync::Arc;
use pumpkin::command::{CommandExecutor, CommandSender, CommandError};
use pumpkin::command::args::ConsumedArgs;
use pumpkin::command::tree::CommandTree;
use pumpkin_util::text::TextComponent;

struct PingExecutor;

impl CommandExecutor for PingExecutor {
    fn execute<'a>(
        &'a self,
        sender: &'a CommandSender,
        _server: &'a pumpkin::server::Server,
        _args: &'a ConsumedArgs<'a>,
    ) -> pumpkin::command::CommandResult<'a> {
        Box::pin(async move {
            sender.send_message(TextComponent::text("§aPong!")).await;
            Ok(1)
        })
    }
}

pub fn init_command_tree() -> CommandTree {
    CommandTree::new(["ping"], "Check if the server is alive")
        .execute(PingExecutor)
}

// In on_load:
// server.register_command(init_command_tree(), "my.plugin.ping").await;
```

#### Java Equivalent

```java
public boolean onCommand(CommandSender sender, Command cmd, String label, String[] args) {
    sender.sendMessage("§aPong!");
    return true;
}
```

### Example 2: `/msg <player> <message>` Command

```rust
use pumpkin::command::args::players::PlayersArgumentConsumer;
use pumpkin::command::args::message::MsgArgConsumer;
use pumpkin::command::args::Arg;
use pumpkin::command::tree::builder::argument;

struct MsgExecutor;

impl CommandExecutor for MsgExecutor {
    fn execute<'a>(
        &'a self,
        sender: &'a CommandSender,
        _server: &'a pumpkin::server::Server,
        args: &'a ConsumedArgs<'a>,
    ) -> pumpkin::command::CommandResult<'a> {
        Box::pin(async move {
            let Some(Arg::Players(targets)) = args.get("target") else {
                return Err(CommandError::InvalidConsumption(Some("target".into())));
            };
            let Some(Arg::Msg(message)) = args.get("message") else {
                return Err(CommandError::InvalidConsumption(Some("message".into())));
            };

            for target in targets {
                target.send_system_message(
                    &TextComponent::text(format!("§7[§dWhisper§7] §f{message}"))
                ).await;
            }

            sender.send_message(
                TextComponent::text(format!("§7Message sent to {} player(s)", targets.len()))
            ).await;

            Ok(targets.len() as i32)
        })
    }
}

pub fn init_command_tree() -> CommandTree {
    CommandTree::new(["msg", "whisper", "w"], "Send a private message")
        .then(
            argument("target", PlayersArgumentConsumer)
                .then(argument("message", MsgArgConsumer).execute(MsgExecutor))
        )
}
```

### Example 3: `/warp <set|tp> <name>` — Subcommands with Literals

```rust
use pumpkin::command::args::simple::SimpleArgConsumer;
use pumpkin::command::tree::builder::{literal, argument, require};

struct WarpSetExecutor;
struct WarpTpExecutor;
struct WarpListExecutor;

impl CommandExecutor for WarpSetExecutor {
    fn execute<'a>(
        &'a self,
        sender: &'a CommandSender,
        _server: &'a pumpkin::server::Server,
        args: &'a ConsumedArgs<'a>,
    ) -> pumpkin::command::CommandResult<'a> {
        Box::pin(async move {
            let Some(Arg::Simple(name)) = args.get("name") else {
                return Err(CommandError::InvalidConsumption(Some("name".into())));
            };

            let Some(player) = sender.as_player() else {
                return Err(CommandError::InvalidRequirement);
            };

            // Save warp location (you'd store this in your plugin state)
            sender.send_message(
                TextComponent::text(format!("§aWarp '{name}' has been set!"))
            ).await;

            Ok(1)
        })
    }
}

impl CommandExecutor for WarpTpExecutor {
    fn execute<'a>(
        &'a self,
        sender: &'a CommandSender,
        _server: &'a pumpkin::server::Server,
        args: &'a ConsumedArgs<'a>,
    ) -> pumpkin::command::CommandResult<'a> {
        Box::pin(async move {
            let Some(Arg::Simple(name)) = args.get("name") else {
                return Err(CommandError::InvalidConsumption(Some("name".into())));
            };

            // Look up and teleport to warp (from your plugin state)
            sender.send_message(
                TextComponent::text(format!("§aTeleported to warp '{name}'!"))
            ).await;

            Ok(1)
        })
    }
}

impl CommandExecutor for WarpListExecutor {
    fn execute<'a>(
        &'a self,
        sender: &'a CommandSender,
        _server: &'a pumpkin::server::Server,
        _args: &'a ConsumedArgs<'a>,
    ) -> pumpkin::command::CommandResult<'a> {
        Box::pin(async move {
            sender.send_message(
                TextComponent::text("§6Available warps: §fspawn, shop, arena")
            ).await;
            Ok(1)
        })
    }
}

pub fn init_command_tree() -> CommandTree {
    CommandTree::new(["warp"], "Warp management")
        .then(
            literal("set")
                .then(
                    require(|sender| sender.is_player())
                        .then(argument("name", SimpleArgConsumer).execute(WarpSetExecutor))
                )
        )
        .then(
            literal("tp")
                .then(
                    require(|sender| sender.is_player())
                        .then(argument("name", SimpleArgConsumer).execute(WarpTpExecutor))
                )
        )
        .then(
            literal("list").execute(WarpListExecutor)
        )
}

// This creates:
//   /warp set <name>   — Players only
//   /warp tp <name>    — Players only
//   /warp list         — Anyone
```

---

## Registering Commands in Your Plugin

```rust
#[plugin_method]
pub fn on_load(&mut self, server: Arc<Context>) -> Result<(), String> {
    // Register with a permission node
    server.register_command(
        init_command_tree(),
        "myplugin.warp",
    ).await;

    server.log("Commands registered!");
    Ok(())
}
```

### Unregistering Commands

```rust
#[plugin_method]
pub fn on_unload(&mut self, server: Arc<Context>) -> Result<(), String> {
    server.unregister_command("warp").await;
    server.log("Commands unregistered!");
    Ok(())
}
```

---

## Error Handling

Commands use `CommandError` for error reporting:

```rust
pub enum CommandError {
    InvalidConsumption(Option<String>), // Argument parsing failed
    InvalidRequirement,                 // require() predicate failed
    PermissionDenied,                   // No permission
    CommandFailed(TextComponent),       // Custom error message
}
```

### Returning Custom Error Messages

```rust
impl CommandExecutor for MyExecutor {
    fn execute<'a>(
        &'a self,
        sender: &'a CommandSender,
        _server: &'a pumpkin::server::Server,
        _args: &'a ConsumedArgs<'a>,
    ) -> pumpkin::command::CommandResult<'a> {
        Box::pin(async move {
            let Some(player) = sender.as_player() else {
                return Err(CommandError::CommandFailed(
                    TextComponent::text("§cThis command can only be used by players!")
                ));
            };

            // Command logic...
            Ok(1)
        })
    }
}
```

---

## What's Next?

In [Stage 6: Player Interactions](06-player-interactions.md), you'll learn how to work with players — sending messages, teleporting, changing gamemodes, and more.

---

[← Events](04-events.md) | [Back to Index](README.md) | [Next: Player Interactions →](06-player-interactions.md)
