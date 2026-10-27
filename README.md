# 🤖 Agent Skill

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Platform](https://img.shields.io/badge/Platform-Windows%20%7C%20.NET%2010-blue.svg)](https://dotnet.microsoft.com/)
[![Framework](https://img.shields.io/badge/Architecture-reverse--skill-magenta.svg)](https://github.com/zhaoxuya520/reverse-skill)
[![Compatibility](https://img.shields.io/badge/AI%20Clients-Claude%20Code%20%7C%20Cursor%20%7C%20Cline%20%7C%20Antigravity-green.svg)](#supported-ai-clients)

> **Agent Skill** (trước đây *ExileApi Agent Skill Router Pack*)  
> AI-powered routing + On-demand toolchain bootstrapping + Self-evolving knowledge base  
> Designed for **ExileApi**, **AutoExile**, **ShopAutoBuyer**, **ExchangeCollector**, and game memory reverse engineering.

---

## 🌟 Architecture Overview

Inspired by the design principles of [zhaoxuya520/reverse-skill](https://github.com/zhaoxuya520/reverse-skill), this skill router provides AI agents with deterministic decision-making, verified game memory reading playbooks, and a self-evolving field journal to eliminate hallucination, prevent regressions, and automate complex bot development workflows.

```
User Prompt (Bug / Feature / Reversing)
    │
    ▼
[MASTER-ROUTING.md] (Deterministic Classifier)
    │
    ├── [R1] Build & Compilation Diagnostics ────────► skills/csharp-build-diagnostics/
    ├── [R2] Game Memory Offsets & Reversing ────────► skills/poe-memory-offset-reversing/
    ├── [R3] Bot Navigation & Stuck Recovery ────────► skills/bot-navigation-stuck-fixer/
    ├── [R4] Boss Encounter State Tuning ────────────► skills/boss-encounter-tuning/
    ├── [R5] Single-File Packager & Release ─────────► skills/single-file-packager-deployment/
    ├── [R6] Trade Buyer & Stash Automation ─────────► skills/trade-buyer-diagnostics/
    └── [R7] Self-Evolving Knowledge Logging ────────► skills/self-evolving-knowledge-base/
```

---

## 📁 Repository Structure

```
agent-skill/
├── AGENTS.md                  # Main entrypoint and agent prompt context
├── RULES.md                   # Global memory safety & execution constraints
├── MASTER-ROUTING.md          # Deterministic R0-R7 routing ladder
│
├── rules/                     # Core behavioral policies
│   ├── 01-routing-contract.md
│   ├── 02-memory-safety-rules.md
│   ├── 03-ui-and-threading-rules.md
│   └── 04-packaging-rules.md
│
├── skills/                    # Domain-specific playbooks
│   ├── poe-memory-offset-reversing/SKILL.md
│   ├── csharp-build-diagnostics/SKILL.md
│   ├── bot-navigation-stuck-fixer/SKILL.md
│   ├── boss-encounter-tuning/SKILL.md
│   ├── single-file-packager-deployment/SKILL.md
│   ├── trade-buyer-diagnostics/SKILL.md
│   └── self-evolving-knowledge-base/SKILL.md
│
├── knowledge-base/            # Self-evolving experience store
│   ├── FIELD_JOURNAL.md       # Root-cause bug post-mortems & prevention rules
│   └── COMMON_PATTERNS.md     # Production memory, input, & rendering patterns
│
└── scripts/                   # Automated environment checkers
    ├── refresh-tool-index.ps1 # Toolchain health check
    └── verify-build-health.ps1# 5-project zero-error build verification
```

---

## 🧭 Skills Catalog

| Skill Name | Description | Key Focus |
|---|---|---|
| **`poe-memory-offset-reversing`** | Game memory structure & offset analysis | DevTree, GameOffsets, IngameState, Components |
| **`csharp-build-diagnostics`** | .NET 10 & MSBuild error resolution | CS warnings, SDK compatibility, assembly refs |
| **`bot-navigation-stuck-fixer`** | Pathfinding & stuck recovery | A* grid mapping, walk-before-blink, collision |
| **`boss-encounter-tuning`** | Boss state machines & encounter tuning | Exarch, Eater, Maven, Feared, King timing |
| **`single-file-packager-deployment`** | Packaging standalone executables | AutoBoss_AllInOne.exe, AutoBuyer_AllInOne.exe |
| **`trade-buyer-diagnostics`** | Shop, trade, & stash deposit automation | ShopAutoBuyer, Faustus Exchange, WebSockets |
| **`frappe-large-list-query`** | Frappe/MariaDB list for 10k–100k+ rows, ~20 rows read per page | Numeric codes + lookup doctype, sort_rank, done_seq, keyset, cache `expires=True` |
| **`self-evolving-knowledge-base`** | Experience logging & regression prevention | FIELD_JOURNAL.md post-mortem registry |

---

## 💻 Quick Start & Installation

### 1. For Claude Code / Cursor / Cline
Copy this folder into your project root as `.agents/`:
```bash
# In your ExileApi workspace root:
git clone https://github.com/nlttien/agent-skill.git .agents
```

### 2. Validate Environment Health
Run the built-in PowerShell health check script:
```powershell
powershell -ExecutionPolicy Bypass -File .agents/scripts/refresh-tool-index.ps1
```

### 3. Verify Entire Build Health
```powershell
powershell -ExecutionPolicy Bypass -File .agents/scripts/verify-build-health.ps1
```

---

## 📜 License
This project is licensed under the [MIT License](LICENSE).
