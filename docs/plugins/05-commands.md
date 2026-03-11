# Stage 5: Commands

## The Command System

In Bukkit, you'd typically override `onCommand()` and manually parse the arguments from a `String[]` array. Pumpkin takes a different approach: you build a **command tree** that describes your command's structure upfront, and the server handles argument parsing, validation, and tab completion for you.

### Java vs Pumpkin: At a Glance

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
// Rust (Pumpkin) — declare the structure, then define the behavior
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

The tree approach means Pumpkin automatically knows what arguments are valid, can generate tab completions, and shows proper usage messages when commands are used incorrectly. No manual parsing needed.

---

## Building a Command Tree

The command tree builder has a few simple building blocks:

### Literal Nodes — Fixed text

These are subcommands — exact words the player types:

```rust
// Creates: /mycommand reload
//          /mycommand status
CommandTree::new(["mycommand"], "My custom command")
    .then(literal("reload").execute(ReloadExecutor))
    .then(literal("status").execute(StatusExecutor))
```

#### Java Comparison

```java
// Java — you'd manually check args[0]
if (args[0].equalsIgnoreCase("reload")) { ... }
else if (args[0].equalsIgnoreCase("status")) { ... }
```

### Argument Nodes — Dynamic typed values

These accept player input and parse it into the right type:

```rust
// Creates: /tp <target>
// The PlayersArgumentConsumer automatically handles player names and selectors (@a, @p)
CommandTree::new(["tp"], "Teleport to a player")
    .then(argument("target", PlayersArgumentConsumer).execute(TpExecutor))
```

### Require Nodes — Permission/condition gates

These add conditions that must be true before the command continues:

```rust
// Only players can use this command (not console, not command blocks)
CommandTree::new(["fly"], "Toggle flight")
    .then(
        require(|sender| sender.is_player())
            .execute(FlyExecutor)
    )
```

### Execute — The action itself

This is where your actual command logic goes. You can execute at the root level (no arguments) or at any point in the tree:

```rust
// No arguments needed — just run the command
CommandTree::new(["ping"], "Check server latency")
    .execute(PingExecutor)
```

---

## Writing Command Executors

Each command action is a struct that implements `CommandExecutor`. The execute method receives the sender, the server, and any parsed arguments:

```rust
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
            Ok(1)  // Return 1 for success
        })
    }
}
```

> **About the return value:** `Ok(1)` means "success" (the command did something). `Ok(0)` means "no-op" (nothing happened). This is used by command blocks in Minecraft to determine if a command succeeded.

### Checking the Sender Type

Commands can come from players, the console, command blocks, or RCON. You can check who sent it:

```rust
// Check if the sender is a player
if let Some(player) = sender.as_player() {
    // 'player' is the Player object
}

// Check type without getting the player
sender.is_player();   // true if a player
sender.is_console();  // true if the console

// Send a message to any sender type
sender.send_message(TextComponent::text("Hello!")).await;

// Check permissions
sender.has_permission_lvl(PermissionLvl::Two);
```

---

## Argument Types

Pumpkin provides built-in argument parsers for common types:

| Argument Consumer | What It Parses | Example Input |
|-------------------|---------------|---------------|
| `PlayersArgumentConsumer` | Player names/selectors | `Steve`, `@a`, `@p` |
| `MsgArgConsumer` | Rest of input as a message | `Hello world!` |
| `GamemodeArgumentConsumer` | Gamemode | `creative`, `survival` |
| `BlockPosArgumentConsumer` | Block coordinates | `100 64 200` |
| `Position3DArgumentConsumer` | Decimal coordinates | `100.5 64.0 200.5` |
| `BoolArgumentConsumer` | Boolean | `true`, `false` |
| `BoundedNumArgumentConsumer` | Number with range | `42` |
| `SimpleArgConsumer` | Raw text token | `my_warp_name` |
| `ItemArgumentConsumer` | Item type | `diamond_sword` |
| `TimeArgumentConsumer` | Time duration | `1d`, `5s`, `30t` |

### Extracting Arguments

When your executor runs, arguments have already been parsed. You extract them by name:

```rust
impl CommandExecutor for MyExecutor {
    fn execute<'a>(
        &'a self,
        sender: &'a CommandSender,
        _server: &'a pumpkin::server::Server,
        args: &'a ConsumedArgs<'a>,
    ) -> pumpkin::command::CommandResult<'a> {
        Box::pin(async move {
            // Get a player argument by name
            let Some(Arg::Players(targets)) = args.get("target") else {
                return Err(CommandError::InvalidConsumption(Some("target".into())));
            };

            // Get a message argument
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

> **Rust tip:** The `let Some(...) = ... else { return ... }` pattern is called a "let-else" statement. It's like an if-null-return check in Java: if the argument isn't found, return an error; otherwise, extract the value and continue.

---

## Complete Command Examples

### Example 1: Simple `/ping` Command

The simplest possible command — no arguments, just a response:

```rust
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

### Example 2: `/msg <player> <message>`

A command with two arguments — a target player and a message:

```rust
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

> **Tip:** `CommandTree::new(["msg", "whisper", "w"], ...)` registers the command under multiple names — so `/msg`, `/whisper`, and `/w` all work. This is like registering aliases in `plugin.yml`.

### Example 3: `/warp <set|tp|list>` — Subcommands

A command with subcommands using literal nodes:

```rust
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
//   /warp list         — Anyone (console, players, etc.)
```

---

## Registering and Unregistering Commands

### Registering

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

### Unregistering

```rust
#[plugin_method]
pub fn on_unload(&mut self, server: Arc<Context>) -> Result<(), String> {
    server.unregister_command("warp").await;
    server.log("Commands unregistered!");
    Ok(())
}
```

---

## Error Handling in Commands

If something goes wrong in your command, you can return different types of errors:

```rust
// Argument not found or invalid
return Err(CommandError::InvalidConsumption(Some("target".into())));

// A require() condition wasn't met
return Err(CommandError::InvalidRequirement);

// No permission
return Err(CommandError::PermissionDenied);

// Custom error with a message shown to the player
return Err(CommandError::CommandFailed(
    TextComponent::text("§cThis command can only be used by players!")
));
```

---

## What's Next?

In [Stage 6: Player Interactions](06-player-interactions.md), you'll learn how to work with players — sending messages, teleporting, changing gamemodes, and more.

---

[← Events](04-events.md) | [Back to Index](README.md) | [Next: Player Interactions →](06-player-interactions.md)
