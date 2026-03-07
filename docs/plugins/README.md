# Pumpkin Plugin Development Guide

Welcome to the official Pumpkin Plugin Development Guide! This tutorial series is designed for Minecraft plugin developers — especially those transitioning from the Java ecosystem (Bukkit/Spigot/Paper) — who want to build plugins for the [Pumpkin](https://github.com/Snowiiii/Pumpkin) Minecraft server.

Pumpkin is a high-performance Minecraft server written in Rust. Its plugin system uses **native dynamic libraries** instead of JVM bytecode, giving you the full power and speed of compiled Rust while keeping a familiar plugin development experience.

---

## 📚 Tutorial Stages

| Stage | Title | Description |
|-------|-------|-------------|
| [01](01-introduction.md) | **Introduction** | Why Pumpkin? Rust vs Java comparison, ecosystem overview |
| [02](02-getting-started.md) | **Getting Started** | Setting up your environment, creating your first plugin |
| [03](03-plugin-lifecycle.md) | **Plugin Lifecycle** | The `Plugin` trait, `on_load`/`on_unload`, and the `Context` API |
| [04](04-events.md) | **Events** | Event system, handlers, priorities, cancellation |
| [05](05-commands.md) | **Commands** | Building commands with `CommandTree`, arguments, and permissions |
| [06](06-player-interactions.md) | **Player Interactions** | Working with players: messages, teleportation, gamemodes |
| [07](07-blocks-and-world.md) | **Blocks & World** | Block events, world events, chunk management |
| [08](08-permissions.md) | **Permissions** | Registering and checking permissions |
| [09](09-advanced-topics.md) | **Advanced Topics** | Services, custom loaders, async patterns, inter-plugin communication |
| [10](10-common-recipes.md) | **Common Recipes** | Java-to-Pumpkin translations of popular plugin patterns |

---

## 🔧 Prerequisites

- **Rust** (edition 2024, minimum version 1.94) — [Install Rust](https://rustup.rs/)
- Basic familiarity with Rust syntax (ownership, borrowing, traits, async/await)
- Experience with Minecraft plugin development (Bukkit/Spigot/Paper) is helpful but not required

## 🎃 Quick Links

- [Pumpkin Repository](https://github.com/Snowiiii/Pumpkin)
- [Pumpkin Documentation](https://pumpkinmc.org/)
- [Pumpkin Discord](https://discord.gg/pumpkinmc)

---

> **Note:** Pumpkin is under active development. The plugin API (currently version 2) may change between releases. Always check for API version compatibility when updating.
