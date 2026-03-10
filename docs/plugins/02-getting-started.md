# Stage 2: Getting Started

## Setting Up Your Development Environment

### Prerequisites

1. **Install Rust** (minimum version 1.94):
   ```bash
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
   ```

   > **Coming from Java?** Rust's installer sets up everything you need in one step. There's no separate JDK, no JAVA_HOME environment variable, and no version managers to worry about. The `rustup` tool handles everything.

2. **Verify your installation:**
   ```bash
   rustc --version   # Should be >= 1.94
   cargo --version
   ```

3. **Clone and build the Pumpkin server** (needed for development):
   ```bash
   git clone https://github.com/Pumpkin-MC/Pumpkin.git
   cd Pumpkin
   cargo build --release
   ```

### What is Cargo?

If you've used Maven or Gradle in Java, **Cargo** is Rust's equivalent — but it does even more. It's your:

- **Build tool** — compiles your code (`cargo build`)
- **Package manager** — downloads dependencies automatically
- **Test runner** — runs your tests (`cargo test`)
- **Project scaffolder** — creates new projects (`cargo new`)

All from a single command-line tool. No XML configuration files, no plugin repositories to configure.

---

## Creating Your First Plugin

### Step 1: Create the Project

```bash
cargo new --lib my_first_plugin
cd my_first_plugin
```

This creates a library project (not a standalone program), similar to creating a new Maven project for a Bukkit plugin.

### Step 2: Configure `Cargo.toml`

Open `Cargo.toml` — this is your project's configuration file, similar to Java's `pom.xml` or `build.gradle` combined with `plugin.yml`. Replace its contents with:

```toml
[package]
name = "my-first-plugin"
version = "0.1.0"
authors = ["Your Name"]
description = "My first Pumpkin plugin"
edition = "2024"

[lib]
crate-type = ["cdylib"]  # Compile as a native library the server can load

[dependencies]
pumpkin = { git = "https://github.com/Pumpkin-MC/Pumpkin.git", branch = "master", package = "pumpkin" }            # Core server API
pumpkin-api-macros = { git = "https://github.com/Pumpkin-MC/Pumpkin.git", branch = "master", package = "pumpkin-api-macros" } # Plugin macros
pumpkin-util = { git = "https://github.com/Pumpkin-MC/Pumpkin.git", branch = "master", package = "pumpkin-util" }       # Utility helpers (text, math, types)
```

A few things to note:

- The `[package]` section is like your `plugin.yml` — it defines your plugin's name, version, and description. Pumpkin reads this metadata automatically.
- `crate-type = ["cdylib"]` tells Rust to compile your code into a dynamic library that Pumpkin can load at runtime, similar to how the JVM loads `.jar` files.
- The `[dependencies]` section is like Maven dependencies — Cargo downloads and links them automatically. The `git = "..."` syntax tells Cargo to fetch the dependency directly from GitHub's latest master branch.
- **`pumpkin-util`** provides helpful types you'll use constantly in plugin development: `TextComponent` for formatted messages, `GameMode`, `Difficulty`, `BlockPos`, `Vector3`, math helpers, and more.

#### Java Comparison

In Java, you'd have a separate `plugin.yml` for metadata:

```yaml
# Java: plugin.yml
name: MyFirstPlugin
version: 1.0.0
main: com.example.MyFirstPlugin
author: Your Name
description: My first plugin
```

In Pumpkin, `Cargo.toml` serves both purposes. Less files, less configuration.

### Step 3: Write the Plugin

Replace `src/lib.rs` with:

```rust
use std::sync::Arc;

use pumpkin::plugin::api::context::Context;
use pumpkin_api_macros::{plugin_impl, plugin_method};

// The #[plugin_impl] macro sets up everything the server needs
// to load your plugin — metadata, version checks, and initialization.
// Think of it as the Rust equivalent of "extends JavaPlugin".
#[plugin_impl]
pub struct MyFirstPlugin;

impl MyFirstPlugin {
    // Every plugin struct needs a new() constructor — the server calls
    // this to create your plugin instance when it loads.
    pub fn new() -> Self {
        Self
    }

    // #[plugin_method] marks this as a plugin lifecycle method.
    // on_load is called when the server starts — like onEnable() in Bukkit.
    #[plugin_method]
    pub fn on_load(&mut self, server: Arc<Context>) -> Result<(), String> {
        server.log("Hello from MyFirstPlugin! 🎃");
        Ok(())
    }

    // on_unload is called when the server stops — like onDisable() in Bukkit.
    #[plugin_method]
    pub fn on_unload(&mut self, server: Arc<Context>) -> Result<(), String> {
        server.log("Goodbye from MyFirstPlugin!");
        Ok(())
    }
}
```

Let's break down the new Rust syntax:

- **`use` statements** — Like Java `import` statements. They bring types into scope.
- **`pub struct MyFirstPlugin;`** — Declares your plugin. A struct in Rust is similar to a Java class, but without inheritance. The `pub` makes it visible outside this file.
- **`pub fn new() -> Self`** — A constructor that the server calls to create your plugin. Every plugin must have this. `Self` is shorthand for the struct's own type.
- **`impl MyFirstPlugin`** — This is where you define methods on your struct, similar to writing methods inside a Java class body.
- **`&mut self`** — This means the method can modify the plugin's data. In Java terms, it's like a regular instance method. (You'll learn more about `&self` vs `&mut self` later.)
- **`Arc<Context>`** — The server context, passed to you on load. `Arc` means it's a shared reference that's safe to use across threads. Don't worry about the details — just use it like you'd use `this.getServer()` in Bukkit.
- **`Result<(), String>`** — The method returns either success (`Ok(())`) or an error (`Err("message")`). This is how Rust handles errors instead of throwing exceptions.

### Step 4: Build the Plugin

```bash
cargo build --release
```

Your compiled plugin will be at:
- **Linux:** `target/release/libmy_first_plugin.so`
- **macOS:** `target/release/libmy_first_plugin.dylib`
- **Windows:** `target/release/my_first_plugin.dll`

> **Tip:** The first build downloads dependencies and takes a bit longer. Subsequent builds are much faster since Cargo caches everything.

### Step 5: Install and Run

Copy the compiled library to the Pumpkin server's `plugins/` directory:

```bash
# Linux example
cp target/release/libmy_first_plugin.so /path/to/pumpkin-server/plugins/

# Start the server
cd /path/to/pumpkin-server
cargo run --release
```

You should see your plugin's message in the server console:

```
[MyFirstPlugin] Hello from MyFirstPlugin! 🎃
```

🎉 Congratulations — you've just built and loaded your first Pumpkin plugin!

---

## What Do the Macros Do?

You might be curious about what `#[plugin_impl]` and `#[plugin_method]` actually do behind the scenes. The short answer: they generate the boilerplate code that Pumpkin needs to load your plugin.

- **`#[plugin_impl]`** reads your `Cargo.toml` and generates the plugin metadata (name, version, authors, description) plus a factory function that the server calls to create your plugin instance. Without this macro, you'd need to write several `#[no_mangle]` functions manually.

- **`#[plugin_method]`** wraps your method so it works with Pumpkin's async system. You write normal-looking code, and the macro makes it compatible with the server's async runtime.

You don't need to understand the generated code to use these macros effectively. If you're curious, you can explore the details later in [Stage 9: Advanced Topics](09-advanced-topics.md).

---

## Plugin Data Folder

Every plugin gets its own data directory, just like `JavaPlugin.getDataFolder()`:

```rust
#[plugin_method]
pub fn on_load(&mut self, server: Arc<Context>) -> Result<(), String> {
    // Creates plugins/my-first-plugin/ if it doesn't exist
    let data_folder = server.get_data_folder();
    server.log(format!("Data folder: {}", data_folder.display()));
    Ok(())
}
```

This is where you'd store configuration files, player data, or any other files your plugin needs.

---

## Logging

Pumpkin provides a built-in logging system for plugins:

```rust
#[plugin_method]
pub fn on_load(&mut self, server: Arc<Context>) -> Result<(), String> {
    // Initialize the logger (optional, enables structured logging)
    server.init_log();

    // Simple log message (automatically prefixed with your plugin name)
    server.log("Plugin loaded successfully!");
    server.log(format!("Running version {}", env!("CARGO_PKG_VERSION")));
    Ok(())
}
```

#### Java Comparison

```java
// Java
getLogger().info("Plugin loaded successfully!");
getLogger().info("Running version " + getDescription().getVersion());
```

---

## The `pumpkin-util` Crate — Your Utility Toolkit

The `pumpkin-util` crate is a collection of helper types and functions that you'll use throughout your plugin development. Think of it as the Pumpkin equivalent of Apache Commons or Guava in the Java world — common building blocks that save you from reinventing the wheel.

Here's a quick overview of what's inside:

| Module | What It Provides | Java Equivalent |
|--------|-----------------|-----------------|
| `text::TextComponent` | Rich formatted messages (colors, bold, click events, hover tooltips) | `ChatComponent` / Adventure API |
| `GameMode` | Survival, Creative, Adventure, Spectator enum | `GameMode` enum |
| `Difficulty` | Peaceful, Easy, Normal, Hard enum | `Difficulty` enum |
| `PermissionLvl` | Operator permission levels (0–4) | `isOp()` / permission levels |
| `math::position::BlockPos` | Block coordinates (integer x, y, z) | `BlockPos` / `Location.toBlockLocation()` |
| `math::vector3::Vector3` | 3D vectors (used for positions, velocities) | `Vector` class |
| `math::experience` | XP calculation helpers (points to level, level to points) | Manual XP calculations |
| `text::color` | Named colors, RGB colors, hex colors | `ChatColor` |
| `text::click::ClickEvent` | Click actions (open URL, run command, copy to clipboard) | `ClickEvent` |
| `text::hover::HoverEvent` | Hover tooltips (show text, show item) | `HoverEvent` |

### TextComponent — Building Rich Messages

`TextComponent` is the type you'll use most often. It builds the formatted messages that players see in chat, action bars, titles, and more:

```rust
use pumpkin_util::text::TextComponent;
use pumpkin_util::text::color::NamedColor;

// Simple text message
let msg = TextComponent::text("Hello, world!");

// Colored and styled text — uses builder pattern (chain method calls)
let fancy = TextComponent::text("Welcome!")
    .color_named(NamedColor::Gold)
    .bold();

// Combine multiple components with add_child
let combined = TextComponent::text("Hello ")
    .add_child(
        TextComponent::text("Steve")
            .color_named(NamedColor::Green)
            .bold()
    )
    .add_child(TextComponent::text("!"));

// Rainbow text!
let rainbow = TextComponent::text("This is rainbow text!").rainbow();

// Gradient text between colors
let gradient = TextComponent::text("Smooth gradient")
    .gradient(&[
        pumpkin_util::text::color::RGBColor::new(255, 0, 0),
        pumpkin_util::text::color::RGBColor::new(0, 0, 255),
    ]);
```

#### Java Comparison

```java
// Java (Adventure API)
Component msg = Component.text("Welcome!")
    .color(NamedTextColor.GOLD)
    .decorate(TextDecoration.BOLD);

// Java (Legacy)
String msg = ChatColor.GOLD + "" + ChatColor.BOLD + "Welcome!";
```

> **Rust tip:** The builder pattern used here (`.color_named(...).bold()`) works because each method returns `self`, so you can chain calls. This is the same idea as Java's builder pattern, but Rust calls it "method chaining."

### Clickable and Hoverable Text

You can make text interactive — just like Adventure API's click/hover events:

```rust
use pumpkin_util::text::TextComponent;
use pumpkin_util::text::click::ClickEvent;
use pumpkin_util::text::hover::HoverEvent;

// Clickable link
let link = TextComponent::text("§bClick here to visit our website!")
    .click_event(ClickEvent::OpenUrl {
        url: "https://pumpkinmc.org".into(),
    });

// Text that runs a command when clicked
let cmd = TextComponent::text("§a[Click to Heal]")
    .click_event(ClickEvent::RunCommand {
        command: "/heal".into(),
    });

// Text with a hover tooltip
let hover = TextComponent::text("§eHover over me!")
    .hover_event(HoverEvent::show_text(
        TextComponent::text("Secret tooltip text!")
    ));
```

### GameMode and Difficulty

Common enums for game state, directly usable in your plugin logic:

```rust
use pumpkin_util::GameMode;
use pumpkin_util::Difficulty;

// Check a player's game mode
if player_gamemode == GameMode::Creative {
    // Player is in creative mode
}

// Parse from a string (useful for commands)
let mode: GameMode = "survival".parse().unwrap();
let diff: Difficulty = "hard".parse().unwrap();

// Display name
let name = GameMode::Creative.to_str(); // "Creative"
```

### BlockPos and Vector3 — Working with Coordinates

These types are used everywhere for positions and coordinates:

```rust
use pumpkin_util::math::position::BlockPos;
use pumpkin_util::math::vector3::Vector3;

// Create a block position
let pos = BlockPos::new(100, 64, -200);

// Iterate over all blocks in a region (like WorldEdit selections)
for block_pos in BlockPos::iterate(
    BlockPos::new(0, 60, 0),
    BlockPos::new(10, 65, 10),
) {
    // Process each block position in the 11×6×11 area
}

// Convert between block positions and floating-point positions
let float_pos: Vector3<f64> = pos.to_f64();
let centered: Vector3<f64> = pos.to_centered_f64(); // Center of block (adds 0.5)

// Create from floating-point (rounds down)
let block = BlockPos::floored(100.7, 64.2, -200.9);
// Result: BlockPos(100, 64, -201)

// Get chunk coordinates from block position
let chunk_pos = pos.chunk_position(); // Which chunk this block is in
```

#### Java Comparison

```java
// Java
Location loc = new Location(world, 100, 64, -200);
Block block = loc.getBlock();
int chunkX = loc.getBlockX() >> 4;
```

### Experience Helpers

Handy functions for XP calculations — no need to look up the formulas:

```rust
use pumpkin_util::math::experience;

// How many XP points needed to progress within level 15?
let points_needed = experience::points_in_level(15); // 37

// Total XP points to reach level 30 from zero
let total = experience::points_to_level(30); // 1395

// Convert total XP to level + remaining points
let (level, remaining) = experience::total_to_level_and_points(1000);
// level = 26, remaining = some points into level 26

// Calculate level progress bar (0.0 to 1.0)
let progress = experience::progress_in_level(remaining, level);
```

We'll use these utilities throughout the tutorial — especially `TextComponent` in the events, commands, and player interaction stages.

---

## Common Build Issues

### "API version mismatch"

```
Plugin API version mismatch (plugin 1, server 2)
```

This means your plugin was compiled against a different version of the Pumpkin API. Rebuild your plugin against the current server version.

### "Missing plugin metadata"

Make sure your `Cargo.toml` has `name`, `version`, `authors`, and `description` fields in the `[package]` section.

### "crate-type must be cdylib"

Ensure your `Cargo.toml` contains:

```toml
[lib]
crate-type = ["cdylib"]
```

---

## What's Next?

In [Stage 3: Plugin Lifecycle](03-plugin-lifecycle.md), we'll explore what you can do with the `Context` API — finding players, registering events, managing plugin state, and more.

---

[← Introduction](01-introduction.md) | [Back to Index](README.md) | [Next: Plugin Lifecycle →](03-plugin-lifecycle.md)
