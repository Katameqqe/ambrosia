# Zero Trust ARPG — Project Proposal

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [What is Zero Trust in the Context of Games?](#2-what-is-zero-trust-in-the-context-of-games)
3. [Why Zero Trust Instead of Anti-Cheat?](#3-why-zero-trust-instead-of-anti-cheat)
4. [Architecture: How It Works](#4-architecture-how-it-works)
5. [Game Design](#5-game-design)
   - [Core Loop](#51-core-loop)
   - [Player Stats](#52-player-stats)
   - [Equipment System](#53-equipment-system)
   - [Item System — Unique Drops Only](#54-item-system--unique-drops-only)
   - [Classes](#55-classes)
   - [Passive Tree](#56-passive-tree)
   - [Skill System](#57-skill-system)
6. [Scope & Simplifications](#6-scope--simplifications)
7. [Summary](#7-summary)

---

## 1. Project Overview

This project combines two areas of interest: **game development** and **network security architecture**. The goal is to implement a simple Action RPG (ARPG) — inspired by games like *Path of Exile*, *Diablo*, and *Last Epoch* — built entirely on a **Zero Trust server authority model**.

The game itself is intentionally kept simple in scope. It serves as a **proof-of-concept and demonstration environment** for the Zero Trust principle applied to real-time multiplayer gaming — showing how this approach eliminates entire categories of cheating at the architectural level, rather than detecting cheats after they occur.

---

## 2. What is Zero Trust in the Context of Games?

**Zero Trust** is a security philosophy originally developed for enterprise networks. Its core principle is:

> *"Never trust, always verify."*

In traditional networks, devices inside a corporate firewall were implicitly trusted. Zero Trust eliminates this assumption — every request must be authenticated and validated, regardless of origin.

### Applied to Games

In a game context, Zero Trust means:

> **The server never trusts any data sent by the client. The client cannot make authoritative claims about the game state.**

Instead of the client saying what *happened*, it only reports **raw player intent** — what the player *did* (pressed a key, clicked a position). The server then:

1. Receives the raw input
2. Validates it against the current game state (physics, cooldowns, mana, line-of-sight, etc.)
3. Executes the logic
4. Broadcasts the result back to all relevant clients

The client is purely a **display terminal**. It shows what the server says is happening.

### Message Design Example

| ❌ Trusted Client (Classic) | ✅ Zero Trust (This Project) |
|---|---|
| `player_move { x: 450, y: 200, speed: 100 }` | `input { action: "move", mouse_pos: { x: 450, y: 200 } }` |
| `player_cast { spell: "fireball", damage: 500 }` | `input { action: "primary_skill", target_pos: { x: 380, y: 140 } }` |
| `player_pickup { item_id: "sword_legendary" }` | `input { action: "interact", object_id: "ground_item_042" }` |

The server validates everything: Is the target reachable? Does the player have enough mana? Is the cooldown expired? Is the object within interaction range? The client cannot fake any of these outcomes.

---

## 3. Why Zero Trust Instead of Anti-Cheat?

### The Problem with Traditional Anti-Cheat

Most commercial games rely on **anti-cheat software** (e.g., BattlEye, Easy Anti-Cheat, VAC) running on the client machine. This approach has fundamental limitations:

| Problem | Description |
|---|---|
| **Arms race** | Every anti-cheat is eventually bypassed; cheaters update their tools, developers patch, repeat. |
| **Kernel-level invasion** | Modern anti-cheat runs at ring-0 (kernel level), raising serious privacy and security concerns for players. |
| **False positives** | Legitimate players get banned due to software conflicts or heuristic errors. |
| **Reactive, not preventive** | Anti-cheat detects cheating *after* it is occurring, often *after* damage is done. |
| **Client-side trust** | If the client sends `my_damage = 9999`, anti-cheat might not catch a cleverly injected value. |

### Why Zero Trust Solves This Architecturally

Zero Trust does not *detect* cheating — it makes most forms of cheating **structurally impossible**:

| Cheat Type | Anti-Cheat Approach | Zero Trust Approach |
|---|---|---|
| **Speed hack** | Detect unusual movement speed pattern | Client cannot set speed; server owns movement |
| **Teleport hack** | Flag impossible position delta | Client never sends position; server computes it |
| **Damage hack** | Monitor damage values for anomalies | Client never sends damage; server computes it |
| **Item duplication** | Track item IDs across clients | Item state lives only on server |
| **God mode / invincibility** | Look for HP never decreasing | HP exists only on server |
| **Skill cooldown bypass** | Compare timestamps | Cooldown is enforced server-side |

The client is effectively a **thin renderer**. A cheater can modify what they *see* locally, but they cannot modify what the *server computes*.

This does not eliminate 100% of cheating (e.g., scripted bots can still send valid-looking inputs at superhuman speed), but it eliminates the entire class of **state manipulation cheats**, which are among the most game-breaking.

---

## 4. Architecture: How It Works

```
┌──────────────────────────────────────────────────────────────┐
│                          CLIENT                              │
│                                                              │
│  Renders world state received from server                    │
│  Captures player input (keyboard, mouse)                     │
│  Sends only: raw inputs + client timestamp                   │
│                                                              │
│  ❌ Does NOT: compute damage, movement, loot, skill effects  │
└──────────────────────────┬───────────────────────────────────┘
                           │  Input events only
                           │  e.g. { action: "move", pos: {x,y} }
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                    AUTHORITATIVE SERVER                       │
│                                                              │
│  ✅ Owns ALL game state (positions, HP, mana, items, etc.)   │
│  ✅ Validates every input against current state              │
│  ✅ Runs physics, combat, loot, and AI logic                 │
│  ✅ Broadcasts state updates to connected clients            │
│                                                              │
│  Input pipeline:                                             │
│    receive_input → validate → simulate → broadcast_delta     │
└──────────────────────────────────────────────────────────────┘
```

### Server Responsibilities

- **Movement**: Receives click/key input, computes valid target position, moves the player at the server-defined speed, and sends back `entity_move { id, from, to, speed, arrival_time }`.
- **Combat**: Receives `skill_use` input, checks mana/cooldown/range, rolls crits and elemental damage on the server, applies resistances, and sends back `damage_event { attacker, target, amount, type }`.
- **Loot**: On monster death, server rolls the drop table and spawns items into the world. Client cannot request specific items or manipulate drop rolls.
- **Instance Management**: Server creates and owns each instance. Players request to enter; the server authorizes it.

---

## 5. Game Design

The game is a simple ARPG with a hub area and instanced combat zones. The design borrows concepts from *Path of Exile* (passive tree, unique items), *Last Epoch* (skill modifier trees), and classic ARPG combat.

### 5.1 Core Loop

```
Hub Area  →  Create / Join Instance  →  Kill Monsters  →  Collect Loot  →  Return to Hub
    ↑                                                                              │
    └──────────────────── Upgrade gear, skills, passives ◄─────────────────────────┘
```

- The **Hub** is a shared social space. Players can trade, inspect characters, and form groups.
- An **Instance** is a private zone created by one player. One player can run it solo; additional players may join if the creator allows it.
- Instances have an **infinite wave or floor structure** — monsters respawn endlessly, scaling in difficulty. The goal is to push as deep as possible for better loot.

### 5.2 Player Stats

| Stat | Description |
|---|---|
| **Health (HP)** | Hit points. Reaching 0 = death (respawn at hub). |
| **Mana** | Resource consumed by skills. Regenerates over time. |
| **Armour** | Reduces incoming physical damage. |
| **Evasion** | Chance to completely avoid a hit. |
| **Fire / Cold / Lightning Resistance** | Reduces respective elemental damage taken (capped at 75% base). |
| **Critical Strike Chance** | Probability of dealing a critical hit. |
| **Critical Strike Multiplier** | Damage multiplier applied on a critical hit. |

All of these values are computed and stored exclusively on the server.

### 5.3 Equipment System

Each character has **6 equipment slots**:

| Slot | Description |
|---|---|
| **Armour** | Full-body armour set (not split into head/chest/legs/etc.) |
| **Main Hand Weapon** | Primary weapon (sword, staff, bow, etc.) |
| **Off Hand Weapon** | Secondary weapon or shield |
| **Ring** | Ring slot (one ring) |
| **Amulet** | Amulet slot |
| **Belt** | Belt slot |

Equipping items is a server-validated action. The server checks class requirements and slot availability before applying.

### 5.4 Item System — Unique Drops Only

To keep the project scope manageable while still providing meaningful itemization:

- Monsters drop only **Unique items** — every item is hand-designed with a fixed, predetermined list of modifiers.
- There are no **prefixes or suffixes**, no random affix rolling. Each modifier is a complete, standalone stat (e.g., `+25% Fire Resistance`, `Skills cost 15% less Mana`).
- This eliminates the need for a complex item generation system while still giving players meaningful upgrade decisions.
- Every unique item has a **name, flavor text, and fixed mod list**, similar to Unique items in *Path of Exile*.

Example unique item:

```
[Amulet] The Ember Knot
  — +18% Fire Resistance
  — +12% Critical Strike Chance
  — Skills deal 8% more damage per 100 Mana spent
  — "From ash it was forged; to ash it returns."
```

### 5.5 Classes

There are three playable classes, each with a different **base stat spread** and **skill pool**:

| Class | Focus | Base Strengths |
|---|---|---|
| **Ranger** | Ranged / Evasion | High Evasion, Dexterity-based, Bow/Wand skills |
| **Melee** | Close-range / Armour | High Armour, Strength-based, Sword/Axe/Mace skills |
| **Mage** | Spells / Mana | High Mana, Intelligence-based, Elemental skills |

Class determines starting position in the passive tree and which skills are natively available. Items can be class-restricted or universal.

### 5.6 Passive Tree

The passive tree is inspired by *Path of Exile* — a **shared, connected tree** where all three classes start at different positions and traverse toward (or overlap with) other class zones.

- The tree is **small** by design (project scope): approximately 100–150 nodes.
- Nodes grant **small, cumulative stat bonuses** (e.g., `+5 HP`, `+3% Evasion`, `+2% Crit Chance`).
- **Notable nodes** at key intersections grant larger or more unique bonuses (e.g., `Your critical strikes also apply a 1-second Chill`).
- The tree is static (no runtime randomness). All calculations are resolved on the server when a player respecs or levels up.

![Passive Tree Datagram](/datagrams/passive-tree.png)

### 5.7 Skill System

Skills work similarly to *Last Epoch's* skill modifier system:

- Each class has access to a set of **base skills** (e.g., Mage: Fireball, Ice Bolt, Lightning Arc).
- Each skill has its own **small modifier tree** with 10–20 nodes.
- Nodes are unlocked by spending points earned by leveling the specific skill (use it more → level it up).
- Modifier nodes change or augment the skill's behavior: `Fireball now pierces enemies`, `Ice Bolt applies 20% Chill on hit`, `Lightning Arc chains to 2 additional targets`.
- Support modifiers are entirely cosmetic/mechanical — they do **not** affect the server's core damage formula inputs directly. They change which formula the server applies.

All skill effects, modifiers, cooldowns, and mana costs are validated and executed on the server.

---

## 6. Scope & Simplifications

This is a thesis/engineering project, not a commercial title. The following deliberate simplifications keep the project achievable:

| Area | Simplification |
|---|---|
| World | One hub + one instance type (no varied biomes or dungeon layouts in v1) |
| Monsters | Small set of enemy types (3–5), each with simple AI patterns |
| Items | Unique-only drops, no crafting system |
| Equipment | 6 slots total, no sub-slots (no helmet/chest/gloves split) |
| Multiplayer | Instances support a small player count (e.g., 1–4); no massive open-world server |
| Passive Tree | ~100–150 nodes, not the 1400+ of full PoE |
| Skills | 3–5 skills per class; each skill tree has ~10–20 nodes |
| Visuals | Functionality-first; art is not the focus |

---

## 7. Summary

| Pillar | What This Project Demonstrates |
|---|---|
| **Zero Trust Security** | A working game where the server owns all game state and the client is a display terminal — making state-manipulation cheats structurally impossible |
| **Authoritative Server Design** | Clean separation between input (client) and simulation (server), with delta-state broadcasting |
| **ARPG Systems** | A functional, if small, implementation of core ARPG mechanics: stats, equipment, skills, passive tree, and loot |
| **Practical Security vs. Anti-Cheat** | A direct comparison showing why architecture is more robust than detection |

> **The core thesis:** Anti-cheat software fights a never-ending arms race against cheaters. Zero Trust architecture removes the battlefield entirely — if the client never computes the outcome, it cannot lie about it.