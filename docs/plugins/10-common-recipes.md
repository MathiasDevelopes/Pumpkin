# Stage 10: Common Recipes

This chapter provides ready-to-use Pumpkin plugin patterns translated from popular Java (Bukkit/Spigot/Paper) examples. Each recipe shows the Java version side-by-side with the Pumpkin equivalent, so you can see exactly how to port your existing plugin ideas.

> **Tip:** These recipes combine concepts from earlier stages. If something looks unfamiliar, check the referenced stage for a detailed explanation.

---

## Recipe 1: Announcements (Scheduled Broadcasts)

### Java

```java
public class AnnouncerPlugin extends JavaPlugin {
    private int taskId;
    private final List<String> messages = List.of(
        "§6Visit our website: example.com",
        "§aRemember to vote daily!",
        "§bJoin our Discord: discord.gg/example"
    );
    private int index = 0;

    @Override
    public void onEnable() {
        taskId = Bukkit.getScheduler().scheduleSyncRepeatingTask(this, () -> {
            Bukkit.broadcastMessage(messages.get(index));
            index = (index + 1) % messages.size();
        }, 0L, 6000L); // Every 5 minutes
    }

    @Override
    public void onDisable() {
        Bukkit.getScheduler().cancelTask(taskId);
    }
}
```

### Pumpkin

In Rust, we use Tokio's async tasks instead of Bukkit's scheduler. The `tokio::select!` macro lets us wait for either the timer or a shutdown signal — whichever comes first:

```rust
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};
use tokio::sync::Notify;

use pumpkin::plugin::api::context::Context;
use pumpkin::server::Server;
use pumpkin_api_macros::{plugin_impl, plugin_method};
use pumpkin_util::text::TextComponent;

#[plugin_impl]
pub struct AnnouncerPlugin {
    shutdown: Arc<Notify>,
}

impl AnnouncerPlugin {
    #[plugin_method]
    pub fn on_load(&mut self, server: Arc<Context>) -> Result<(), String> {
        self.shutdown = Arc::new(Notify::new());

        let messages = vec![
            "§6Visit our website: example.com".to_string(),
            "§aRemember to vote daily!".to_string(),
            "§bJoin our Discord: discord.gg/example".to_string(),
        ];

        let server_ref = server.server.clone();
        let shutdown = self.shutdown.clone();
        let index = Arc::new(AtomicUsize::new(0));

        tokio::spawn(async move {
            loop {
                tokio::select! {
                    _ = tokio::time::sleep(tokio::time::Duration::from_secs(300)) => {
                        let i = index.fetch_add(1, Ordering::Relaxed) % messages.len();
                        server_ref.broadcast_message(
                            &TextComponent::text(&messages[i]),
                            &TextComponent::text("Server"),
                            pumpkin_util::text::SayCommand::Say,
                            None,
                        ).await;
                    }
                    _ = shutdown.notified() => break,
                }
            }
        });

        server.log("Announcer started!");
        Ok(())
    }

    #[plugin_method]
    pub fn on_unload(&mut self, server: Arc<Context>) -> Result<(), String> {
        self.shutdown.notify_one();
        server.log("Announcer stopped!");
        Ok(())
    }
}
```

---

## Recipe 2: Welcome Kit (Give Items on First Join)

### Java

```java
@EventHandler
public void onJoin(PlayerJoinEvent event) {
    Player player = event.getPlayer();
    if (!player.hasPlayedBefore()) {
        player.getInventory().addItem(
            new ItemStack(Material.DIAMOND_SWORD, 1),
            new ItemStack(Material.BREAD, 16),
            new ItemStack(Material.OAK_LOG, 32)
        );
        player.sendMessage("§6Welcome! Here's your starter kit!");
    }
}
```

### Pumpkin

```rust
use std::sync::Arc;
use std::collections::HashSet;
use tokio::sync::RwLock;

use pumpkin::plugin::api::events::player::PlayerJoinEvent;
use pumpkin::plugin::EventHandler;
use pumpkin::server::Server;
use pumpkin_util::text::TextComponent;

struct WelcomeKitHandler {
    // Track players who have already received the kit
    // In production, persist this to disk
    seen_players: Arc<RwLock<HashSet<uuid::Uuid>>>,
}

impl EventHandler<PlayerJoinEvent> for WelcomeKitHandler {
    fn handle<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a PlayerJoinEvent,
    ) -> futures::future::BoxFuture<'a, ()> {
        Box::pin(async move {
            let uuid = event.player.gameprofile.id;
            let mut seen = self.seen_players.write().await;

            if seen.insert(uuid) {
                // First time joining!
                event.player.send_system_message(
                    &TextComponent::text("§6Welcome! Enjoy your stay on the server!")
                ).await;

                // Note: Item giving depends on inventory API availability.
                // Check the current Pumpkin API for inventory manipulation methods.
            }
        })
    }
}
```

---

## Recipe 3: AFK Detection

### Java

```java
public class AfkPlugin extends JavaPlugin implements Listener {
    private final Map<UUID, Long> lastActivity = new ConcurrentHashMap<>();
    private final Set<UUID> afkPlayers = ConcurrentHashMap.newKeySet();

    @EventHandler
    public void onMove(PlayerMoveEvent event) {
        UUID uuid = event.getPlayer().getUniqueId();
        lastActivity.put(uuid, System.currentTimeMillis());
        if (afkPlayers.remove(uuid)) {
            Bukkit.broadcastMessage(event.getPlayer().getName() + " is no longer AFK");
        }
    }

    @EventHandler
    public void onChat(AsyncPlayerChatEvent event) {
        lastActivity.put(event.getPlayer().getUniqueId(), System.currentTimeMillis());
    }
}
```

### Pumpkin

```rust
use std::sync::Arc;
use std::collections::{HashMap, HashSet};
use std::time::Instant;
use tokio::sync::RwLock;

use pumpkin::plugin::api::events::player::{PlayerMoveEvent, PlayerChatEvent, PlayerLeaveEvent};
use pumpkin::plugin::api::events::EventPriority;
use pumpkin::plugin::EventHandler;
use pumpkin::server::Server;
use pumpkin_util::text::TextComponent;

struct AfkState {
    last_activity: HashMap<uuid::Uuid, Instant>,
    afk_players: HashSet<uuid::Uuid>,
}

struct MoveActivityHandler {
    state: Arc<RwLock<AfkState>>,
}

impl EventHandler<PlayerMoveEvent> for MoveActivityHandler {
    fn handle<'a>(
        &'a self,
        server: &'a Arc<Server>,
        event: &'a PlayerMoveEvent,
    ) -> futures::future::BoxFuture<'a, ()> {
        Box::pin(async move {
            let uuid = event.player.gameprofile.id;
            let name = &event.player.gameprofile.name;
            let mut state = self.state.write().await;

            state.last_activity.insert(uuid, Instant::now());

            if state.afk_players.remove(&uuid) {
                server.broadcast_message(
                    &TextComponent::text(format!("§7{name} is no longer AFK")),
                    &TextComponent::text("Server"),
                    pumpkin_util::text::SayCommand::Say,
                    None,
                ).await;
            }
        })
    }
}

struct ChatActivityHandler {
    state: Arc<RwLock<AfkState>>,
}

impl EventHandler<PlayerChatEvent> for ChatActivityHandler {
    fn handle<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a PlayerChatEvent,
    ) -> futures::future::BoxFuture<'a, ()> {
        Box::pin(async move {
            let uuid = event.player.gameprofile.id;
            let mut state = self.state.write().await;
            state.last_activity.insert(uuid, Instant::now());
        })
    }
}

struct LeaveCleanupHandler {
    state: Arc<RwLock<AfkState>>,
}

impl EventHandler<PlayerLeaveEvent> for LeaveCleanupHandler {
    fn handle<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a PlayerLeaveEvent,
    ) -> futures::future::BoxFuture<'a, ()> {
        Box::pin(async move {
            let uuid = event.player.gameprofile.id;
            let mut state = self.state.write().await;
            state.last_activity.remove(&uuid);
            state.afk_players.remove(&uuid);
        })
    }
}
```

---

## Recipe 4: Custom Chat Format

### Java

```java
@EventHandler(priority = EventPriority.HIGH)
public void onChat(AsyncPlayerChatEvent event) {
    String rank = getRank(event.getPlayer());
    event.setFormat("§7[" + rank + "§7] §f%s§7: §f%s");
}
```

### Pumpkin

```rust
struct ChatFormatHandler;

impl EventHandler<PlayerChatEvent> for ChatFormatHandler {
    fn handle_blocking<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a mut PlayerChatEvent,
    ) -> futures::future::BoxFuture<'a, ()> {
        Box::pin(async move {
            let name = &event.player.gameprofile.name;
            let rank = get_rank(&event.player).await; // Your rank lookup
            event.message = format!("§7[{rank}§7] §f{name}§7: §f{}", event.message);
        })
    }
}

// Register as blocking with HIGH priority
server.register_event::<PlayerChatEvent, _>(
    Arc::new(ChatFormatHandler),
    EventPriority::High,
    true,
).await;
```

---

## Recipe 5: Command Cooldowns

### Java

```java
private final Map<UUID, Long> cooldowns = new HashMap<>();
private static final long COOLDOWN_MS = 30000; // 30 seconds

@Override
public boolean onCommand(CommandSender sender, Command cmd, String label, String[] args) {
    if (sender instanceof Player player) {
        long now = System.currentTimeMillis();
        Long lastUse = cooldowns.get(player.getUniqueId());
        if (lastUse != null && now - lastUse < COOLDOWN_MS) {
            long remaining = (COOLDOWN_MS - (now - lastUse)) / 1000;
            player.sendMessage("§cPlease wait " + remaining + "s before using this again!");
            return true;
        }
        cooldowns.put(player.getUniqueId(), now);
        // Execute command...
    }
    return true;
}
```

### Pumpkin

```rust
use std::collections::HashMap;
use std::sync::Arc;
use std::time::{Duration, Instant};
use tokio::sync::RwLock;

use pumpkin::command::{CommandExecutor, CommandSender, CommandError};
use pumpkin::command::args::ConsumedArgs;
use pumpkin_util::text::TextComponent;

struct CooldownExecutor {
    cooldowns: Arc<RwLock<HashMap<uuid::Uuid, Instant>>>,
    cooldown_duration: Duration,
}

impl CommandExecutor for CooldownExecutor {
    fn execute<'a>(
        &'a self,
        sender: &'a CommandSender,
        _server: &'a pumpkin::server::Server,
        _args: &'a ConsumedArgs<'a>,
    ) -> pumpkin::command::CommandResult<'a> {
        Box::pin(async move {
            let Some(player) = sender.as_player() else {
                return Err(CommandError::InvalidRequirement);
            };

            let uuid = player.gameprofile.id;
            let now = Instant::now();

            // Check cooldown
            {
                let cooldowns = self.cooldowns.read().await;
                if let Some(last_use) = cooldowns.get(&uuid) {
                    let elapsed = now.duration_since(*last_use);
                    if elapsed < self.cooldown_duration {
                        let remaining = (self.cooldown_duration - elapsed).as_secs();
                        sender.send_message(TextComponent::text(
                            format!("§cPlease wait {remaining}s before using this again!")
                        )).await;
                        return Ok(0);
                    }
                }
            }

            // Set cooldown
            {
                let mut cooldowns = self.cooldowns.write().await;
                cooldowns.insert(uuid, now);
            }

            // Execute the actual command logic
            sender.send_message(TextComponent::text("§aCommand executed!")).await;
            Ok(1)
        })
    }
}
```

---

## Recipe 6: Server Rules Command

### Java

```java
public boolean onCommand(CommandSender sender, Command cmd, String label, String[] args) {
    sender.sendMessage("§6=== Server Rules ===");
    sender.sendMessage("§f1. Be respectful");
    sender.sendMessage("§f2. No griefing");
    sender.sendMessage("§f3. No cheating");
    sender.sendMessage("§f4. Have fun!");
    return true;
}
```

### Pumpkin

```rust
struct RulesExecutor;

impl CommandExecutor for RulesExecutor {
    fn execute<'a>(
        &'a self,
        sender: &'a CommandSender,
        _server: &'a pumpkin::server::Server,
        _args: &'a ConsumedArgs<'a>,
    ) -> pumpkin::command::CommandResult<'a> {
        Box::pin(async move {
            let rules = [
                "§6=== Server Rules ===",
                "§f1. Be respectful",
                "§f2. No griefing",
                "§f3. No cheating",
                "§f4. Have fun!",
            ];

            for rule in &rules {
                sender.send_message(TextComponent::text(*rule)).await;
            }

            Ok(1)
        })
    }
}

pub fn init_command_tree() -> CommandTree {
    CommandTree::new(["rules"], "Display server rules")
        .execute(RulesExecutor)
}
```

---

## Recipe 7: Join Counter with Persistent Storage

### Pumpkin (with File-Based Persistence)

```rust
use std::sync::Arc;
use std::collections::HashMap;
use tokio::sync::RwLock;
use serde::{Deserialize, Serialize};

use pumpkin::plugin::api::context::Context;
use pumpkin::plugin::api::events::player::PlayerJoinEvent;
use pumpkin::plugin::api::events::EventPriority;
use pumpkin::plugin::EventHandler;
use pumpkin::server::Server;
use pumpkin_api_macros::{plugin_impl, plugin_method};
use pumpkin_util::text::TextComponent;

#[derive(Serialize, Deserialize, Default)]
struct JoinData {
    counts: HashMap<String, u32>,  // UUID string -> join count
}

#[plugin_impl]
pub struct JoinCounterPlugin {
    context: Option<Arc<Context>>,
    data: Arc<RwLock<JoinData>>,
}

struct JoinCountHandler {
    data: Arc<RwLock<JoinData>>,
    data_path: std::path::PathBuf,
}

impl EventHandler<PlayerJoinEvent> for JoinCountHandler {
    fn handle<'a>(
        &'a self,
        _server: &'a Arc<Server>,
        event: &'a PlayerJoinEvent,
    ) -> futures::future::BoxFuture<'a, ()> {
        Box::pin(async move {
            let uuid = event.player.gameprofile.id.to_string();
            let name = &event.player.gameprofile.name;

            let count = {
                let mut data = self.data.write().await;
                let count = data.counts.entry(uuid).or_insert(0);
                *count += 1;
                *count
            };

            // Notify the player
            event.player.send_system_message(&TextComponent::text(
                format!("§7Welcome back, §e{name}§7! This is join §6#{count}§7.")
            )).await;

            // Persist to disk
            let data = self.data.read().await;
            if let Ok(content) = toml::to_string_pretty(&*data) {
                let _ = std::fs::write(&self.data_path, content);
            }
        })
    }
}

impl JoinCounterPlugin {
    #[plugin_method]
    pub fn on_load(&mut self, server: Arc<Context>) -> Result<(), String> {
        let data_folder = server.get_data_folder();
        let data_path = data_folder.join("joins.toml");

        // Load existing data
        let data = if data_path.exists() {
            let content = std::fs::read_to_string(&data_path)
                .map_err(|e| e.to_string())?;
            toml::from_str(&content).unwrap_or_default()
        } else {
            JoinData::default()
        };

        self.data = Arc::new(RwLock::new(data));
        self.context = Some(server.clone());

        server.register_event::<PlayerJoinEvent, _>(
            Arc::new(JoinCountHandler {
                data: self.data.clone(),
                data_path,
            }),
            EventPriority::Normal,
            false,
        ).await;

        server.log("Join counter loaded!");
        Ok(())
    }
}
```

---

## Quick Reference: Java → Pumpkin Cheat Sheet

| Java (Bukkit/Spigot/Paper) | Pumpkin (Rust) |
|-----------------------------|----------------|
| `extends JavaPlugin` | `#[plugin_impl] pub struct MyPlugin` |
| `onEnable()` | `on_load(&mut self, server: Arc<Context>)` |
| `onDisable()` | `on_unload(&mut self, server: Arc<Context>)` |
| `getServer()` | `server.server` |
| `getDataFolder()` | `server.get_data_folder()` |
| `getLogger().info(msg)` | `server.log(msg)` |
| `@EventHandler` | `impl EventHandler<E> for H` |
| `event.setCancelled(true)` | `event.set_cancelled(true)` |
| `event.isCancelled()` | `event.cancelled()` |
| `Bukkit.broadcastMessage(msg)` | `server.broadcast_message(...)` |
| `player.sendMessage(msg)` | `player.send_system_message(&text).await` |
| `player.getName()` | `player.gameprofile.name` |
| `player.getUniqueId()` | `player.gameprofile.id` |
| `player.getLocation()` | `player.living_entity.entity.pos.load()` |
| `player.teleport(loc)` | `player.teleport(pos).await` |
| `player.setGameMode(mode)` | `player.set_gamemode(mode).await` |
| `player.kick(reason)` | `player.kick(text).await` |
| `Bukkit.getPlayer(name)` | `server.get_player_by_name(name)` |
| `Bukkit.getScheduler().runTask(...)` | `tokio::spawn(async { ... })` |
| `plugin.yml` | `Cargo.toml` + `#[plugin_impl]` |
| `config.yml` (YAML) | `config.toml` (TOML) with `serde` |
| `ServicesManager` | `server.register_service()` / `server.get_service()` |
| `PluginManager.getPlugin(name)` | `server.plugin_manager.is_plugin_active(name)` |

---

## What's Next?

Congratulations! 🎃 You've completed the Pumpkin Plugin Development Guide.

You now have the knowledge to build plugins ranging from simple utilities to complex systems. Remember:

- **Start small** — Begin with a `/ping` command or a join message, then add complexity gradually
- **Let the compiler help you** — Rust's compiler error messages are detailed and helpful. When something doesn't compile, read the error carefully — it usually tells you exactly how to fix it
- **Don't fight the borrow checker** — If the compiler is stopping you from doing something, there's usually a good reason. It's protecting you from bugs that would be painful to debug at runtime
- **It gets easier** — The `Arc`, `RwLock`, and lifetime syntax might feel verbose at first, but it quickly becomes muscle memory. After a few plugins, you'll write it without thinking
- **You don't need to learn all of Rust** — You can build great plugins knowing just the concepts covered in this guide. Learn more as you need it

### Resources

- [Pumpkin Repository](https://github.com/Snowiiii/Pumpkin) — Source code and issues
- [Pumpkin Documentation](https://pumpkinmc.org/) — Official docs
- [Pumpkin Discord](https://discord.gg/pumpkinmc) — Community and support
- [The Rust Book](https://doc.rust-lang.org/book/) — The official Rust tutorial (free)
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/) — Learn through hands-on examples
- [Tokio Tutorial](https://tokio.rs/tokio/tutorial) — Learn async Rust (Pumpkin's async runtime)
- [Rustlings](https://github.com/rust-lang/rustlings) — Small exercises to practice Rust basics

### Need Help?

1. **Read the Rust compiler errors** — They're usually very clear and suggest fixes
2. **Check the Pumpkin source code** — The codebase is well-organized and the existing commands/plugins serve as great examples
3. **Ask in the Discord** — The community is active and welcoming
4. **Refer back to this guide** — Use the [cheat sheet above](#quick-reference-java--pumpkin-cheat-sheet) for quick Java→Rust translations

---

[← Advanced Topics](09-advanced-topics.md) | [Back to Index](README.md)
