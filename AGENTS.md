# ExileApi Autonomous Engineering Agent System

> Inspired by the `reverse-skill` framework (AI-powered routing + On-demand toolchain + Self-evolving knowledge base)

Welcome to the **ExileApi Agent Router**. This system equips AI agents with deterministic routing, verified playbooks, game memory reverse engineering knowledge, and an evolving bug-fix registry for developing, debugging, and maintaining **ExileApi** and its plugins (`AutoExile`, `ShopAutoBuyer`, `ExchangeCollector`, etc.).

---

## 🎯 Core Operating Principles

1. **Deterministic Routing over Guesswork**:
   Never guess terminal commands or memory structures. Follow the **[MASTER-ROUTING.md](./MASTER-ROUTING.md)** ladder (R0–R8) to select the appropriate domain skill.

2. **PoE Memory Safety First**:
   Always validate `entity.IsValid`, check component nullability (`GetComponent<Life>()`), and use safe memory dereferences to avoid crashing ExileCore or the game process.

3. **Continuous Build Verification**:
   Every code modification must be verified using `dotnet build` on the respective `.csproj` file with `0 Error(s)`.

4. **Self-Evolving Knowledge Base**:
   After fixing a complex bug or discovering a game structure change, record the root cause and solution in **[knowledge-base/FIELD_JOURNAL.md](./knowledge-base/FIELD_JOURNAL.md)** so future sessions never repeat the issue.

---

## 🧭 Master Routing Overview

| Route | Domain / Task Category | Skill Reference |
|---|---|---|
| **R0** | Scope Check & Task Classification | `rules/01-routing-contract.md` |
| **R1** | C# Build & Compilation Diagnostics | `skills/csharp-build-diagnostics/SKILL.md` |
| **R2** | Game Memory Offsets & Reversing | `skills/poe-memory-offset-reversing/SKILL.md` |
| **R3** | Bot Navigation & Stuck Recovery | `skills/bot-navigation-stuck-fixer/SKILL.md` |
| **R4** | Boss Encounter Logic & Timing | `skills/boss-encounter-tuning/SKILL.md` |
| **R5** | Single-File Executable Packaging | `skills/single-file-packager-deployment/SKILL.md` |
| **R6** | Trade Buyer & Stash Automation | `skills/trade-buyer-diagnostics/SKILL.md` |
| **R7** | Self-Evolving Knowledge Logging | `skills/self-evolving-knowledge-base/SKILL.md` |
| **R8** | GGPK File Patching (PoeRedux) | `skills/ggpk-file-patching/SKILL.md` |

---

## 📂 System Directory Structure

```
.agents/
├── AGENTS.md                  # Main entrypoint & instructions
├── RULES.md                   # Global execution rules & safety constraints
├── MASTER-ROUTING.md          # Deterministic decision ladder (R0-R7)
├── scripts/
│   ├── refresh-tool-index.ps1 # Toolchain validation script
│   └── verify-build-health.ps1# Build verification script
├── rules/
│   ├── 01-routing-contract.md
│   ├── 02-memory-safety-rules.md
│   ├── 03-ui-and-threading-rules.md
│   └── 04-packaging-rules.md
├── skills/
│   ├── poe-memory-offset-reversing/SKILL.md
│   ├── csharp-build-diagnostics/SKILL.md
│   ├── bot-navigation-stuck-fixer/SKILL.md
│   ├── boss-encounter-tuning/SKILL.md
│   ├── single-file-packager-deployment/SKILL.md
│   ├── trade-buyer-diagnostics/SKILL.md
│   ├── ggpk-file-patching/SKILL.md
│   └── self-evolving-knowledge-base/SKILL.md
└── knowledge-base/
    ├── FIELD_JOURNAL.md
    └── COMMON_PATTERNS.md
```
