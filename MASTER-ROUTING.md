# Master Routing Ladder (MASTER-ROUTING.md)

When receiving a user prompt, classify the request through this deterministic decision ladder (R0 → R7). Do not execute tools until the route is established.

---

### 🪜 Routing Decision Tree

```
User Prompt
    │
    ├── [R1] Build failure / C# compiler error / CS8602 / MSBuild? ─────────► skills/csharp-build-diagnostics/
    │
    ├── [R2] Game update / Memory offset / Struct breakdown / DevTree? ────► skills/poe-memory-offset-reversing/
    │
    ├── [R3] Bot stuck / Pathfinding / Navigation / Blink failure? ─────────► skills/bot-navigation-stuck-fixer/
    │
    ├── [R4] Boss fight timing / Flask delay / Phase transition? ───────────► skills/boss-encounter-tuning/
    │
    ├── [R5] Build exe / SingleFilePackager / Payload zip / Dist missing? ──► skills/single-file-packager-deployment/
    │
    ├── [R6] Shop buyer / Currency trade / Stash deposit / WebSocket? ──────► skills/trade-buyer-diagnostics/
    │
    ├── [R8] Patch Content.ggpk / PoeRedux / màn hình đen / treo sau patch? ─► skills/ggpk-file-patching/
    │
    ├── [R9] Frappe danh sách lớn / phân trang / KPI chậm / doctype 10k+ dòng? ► skills/frappe-large-list-query/
    │
    └── [R7] Unknown edge case / Bug resolved / Lesson learned? ────────────► skills/self-evolving-knowledge-base/
```

---

### Detailed Route Specifications

#### `[R1] C# Build Diagnostics`
- **Triggers**: `dotnet build` errors, `error CS...`, missing references, .NET 10 compilation warnings.
- **Skill**: `.agents/skills/csharp-build-diagnostics/SKILL.md`

#### `[R2] PoE Memory & Offsets Reversing`
- **Triggers**: Entity components returning null, `GameOffsets.dll` mismatch, DevTree path changes, IngameState structures.
- **Skill**: `.agents/skills/poe-memory-offset-reversing/SKILL.md`

#### `[R3] Bot Navigation & Stuck Recovery`
- **Triggers**: Player spinning, getting stuck on terrain, failing to blink over obstacles, navigation timeout.
- **Skill**: `.agents/skills/bot-navigation-stuck-fixer/SKILL.md`

#### `[R4] Boss Encounter Tuning`
- **Triggers**: Eater, Exarch, Maven, Feared, King, Oshabi logic, pre-cast attacks, flask activation timing, invulnerability windows.
- **Skill**: `.agents/skills/boss-encounter-tuning/SKILL.md`

#### `[R5] Single-File Packager & Deployment`
- **Triggers**: Building `.exe` files into `dist/`, payload packaging, DLL synchronization, AppData caching issues.
- **Skill**: `.agents/skills/single-file-packager-deployment/SKILL.md`

#### `[R8] GGPK File Patching (PoeRedux)`
- **Triggers**: sửa file game trong Content.ggpk, PoeRedux patch/restore/repair, màn hình đen, treo khi vào khu vực sau patch.
- **Skill**: `.agents/skills/ggpk-file-patching/SKILL.md`

#### `[R6] Trade Buyer & Stash Automation`
- **Triggers**: ShopAutoBuyer window recognition, Currency Exchange collector, StashDepositService item clicks, Trade API rate limits.
- **Skill**: `.agents/skills/trade-buyer-diagnostics/SKILL.md`

#### `[R9] Frappe Large List Query`
- **Triggers**: trang danh sách / tab / KPI / phân trang / tìm kiếm trên doctype Frappe nhiều dòng; "Frappe có chịu nổi N dòng không"; deep page chậm; cache Frappe không trúng; thiết kế mã trạng thái 0/1, 1..n + bảng danh mục.
- **Skill**: `.agents/skills/frappe-large-list-query/SKILL.md`

#### `[R7] Self-Evolving Knowledge Base`
- **Triggers**: Logging a fixed bug, recording a newly discovered offset pattern, updating `FIELD_JOURNAL.md`.
- **Skill**: `.agents/skills/self-evolving-knowledge-base/SKILL.md`
