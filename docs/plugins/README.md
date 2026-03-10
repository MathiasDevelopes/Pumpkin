# Pumpkin Plugin Development Guide

Welcome to the official Pumpkin Plugin Development Guide! 🎃

This tutorial series is designed for Minecraft plugin developers — especially those coming from the Java ecosystem (Bukkit/Spigot/Paper) — who want to build plugins for the [Pumpkin](https://github.com/Pumpkin-MC/Pumpkin) Minecraft server.

**New to Rust?** That's perfectly fine! This guide walks you through everything step by step, introducing Rust concepts as they come up and explaining how they relate to what you already know from Java. You don't need to be a Rust expert to get started — the Pumpkin macros handle much of the complexity for you.

Pumpkin is a high-performance Minecraft server written in Rust. Its plugin system is designed to feel familiar to Java developers while giving you the benefits of Rust: blazing-fast performance, memory safety without a garbage collector, and compile-time guarantees that eliminate entire classes of bugs.

---

## 📚 Tutorial Stages

| Stage | Title | Description |
|-------|-------|-------------|
| [01](01-introduction.md) | **Introduction** | Why Pumpkin? How Rust benefits your plugins, key concepts explained gently |
| [02](02-getting-started.md) | **Getting Started** | Setting up your environment, creating your first plugin |
| [03](03-plugin-lifecycle.md) | **Plugin Lifecycle** | How plugins start, run, and stop — plus the Context API |
| [04](04-events.md) | **Events** | Reacting to game events: joins, chat, blocks, and more |
| [05](05-commands.md) | **Commands** | Building custom commands with arguments and permissions |
| [06](06-player-interactions.md) | **Player Interactions** | Working with players: messages, teleportation, gamemodes |
| [07](07-blocks-and-world.md) | **Blocks & World** | Block events, world events, chunk management |
| [08](08-permissions.md) | **Permissions** | Registering and checking permissions |
| [09](09-advanced-topics.md) | **Advanced Topics** | Services, custom loaders, async patterns, inter-plugin communication |
| [10](10-common-recipes.md) | **Common Recipes** | Java-to-Pumpkin translations of popular plugin patterns |

---

## 🔧 Prerequisites

- **Rust** (edition 2024, minimum version 1.94) — [Install Rust](https://rustup.rs/)
- Some programming experience (Java, or any other language)
- Experience with Minecraft plugin development (Bukkit/Spigot/Paper) is helpful but not required

> **Tip:** If you've never written Rust before, we recommend keeping [The Rust Book](https://doc.rust-lang.org/book/) open as a reference. But don't worry — this guide explains every Rust concept as it appears.

## 🎃 Quick Links

- [Pumpkin Repository](https://github.com/Pumpkin-MC/Pumpkin)
- [Pumpkin Documentation](https://pumpkinmc.org/)
- [Pumpkin Discord](https://discord.gg/pumpkinmc)
- [The Rust Book](https://doc.rust-lang.org/book/) — Learn Rust from scratch
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/) — Learn through hands-on examples

---

> **Note:** Pumpkin is under active development. The plugin API (currently version 2) may change between releases. Always check for API version compatibility when updating.
