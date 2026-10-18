---
name: boss-encounter-tuning
description: >-
  Playbook for tuning boss fight state machines, delays, flask sequences, pre-cast attacks,
  and encounter lifecycle in AutoExile (Exarch, Eater, Maven, Feared, King, Oshabi, Saresh).
---

# Boss Encounter Tuning & Phase Machine Playbook

## 🔍 When to Use This Skill
- Tuning boss spawn delays, invulnerability periods, and flask activation timing.
- Adjusting right-click rapid attack intervals or pre-casting coordinates.
- Fixing boss death detection, loot sweep timeouts, or instant hideout exit.

---

## 📋 Boss Encounter Matrix

| Boss Encounter | Arena Center | Delay Strategy | Flask / Attack Timing |
|---|---|---|---|
| **The Searing Exarch** | `(253, 253)` | Wait **2.0s** after arrival | Press 1-2-3-4-5 and attack after 2s invulnerability descent |
| **The Eater of Worlds** | `(379, 282)` | **0.0s delay / -0.5s early** | Walk 0.5s $\rightarrow$ Blink $\rightarrow$ At $\le 40\text{g}$ press 1-2-3-4-5 & attack immediately |
| **The Maven** | `(291, 230)` | Dynamic Phase Machine | Brain blast escape $\rightarrow$ Memory game edge wait $\rightarrow$ Flappy bird DPS |
| **The Feared** | Dynamic | Multi-Boss Orbit | Target highest threat unique $\rightarrow$ Orbit & burst |
| **King of the Mists** | `(120, 120)` | Ritual / Maze | Maze solver $\rightarrow$ Totem dodge $\rightarrow$ DPS |

---

## 🛠️ Key Implementation Patterns

### 1. Rapid Right-Click Attack Interval
```csharp
if ((DateTime.Now - _lastCastTime).TotalMilliseconds >= 120)
{
    _lastCastTime = DateTime.Now;
    BotInput.RapidRightClickAt(targetScreenPos);
}
```

### 2. Full 5 Flask Press
```csharp
private static async Task PressAllFiveFlasksAsync()
{
    Keys[] keys = [Keys.D1, Keys.D2, Keys.D3, Keys.D4, Keys.D5];
    foreach (var k in keys)
    {
        Input.KeyDown(k);
        await Task.Delay(30);
        Input.KeyUp(k);
        await Task.Delay(60);
    }
}
```

### 3. Safe Death Cleanup
```csharp
BotInput.ReleaseRightClick();
BotInput.ReleaseAllKeys();
_isLooting = true;
Status = $"{Name} đã chết — Hoàn thành!";
return BossEncounterResult.Complete;
```

---

## 🐍 Bản Python đọc RAM (exileapi-rebuild `tools/boss_loop.py` + `tools/boss_cycle.py`) — 2026-10-17/18

Chu trình Invitation (Searing Exarch), máy trạng thái vô tận — kế hoạch đầy đủ ở repo `docs/BOSS_LOOP.md`:
`DETECT → HIDEOUT → OPEN_MAP → ENTER → FIGHT → LOOT → HOME`, `RESTOCK` (túi hết → rương guild), `RECOVER` khi lệch.

| Bước | Điều đã kiểm trên game (3.29.3) |
|---|---|
| Mở Map Device | click NHÃN "Map Device" (lớp chữ VT_WORLD_LABEL) — game tự đi tới; ngoài màn hình: click về phía thiết bị, dò nhãn mỗi 0.1s, hiện là click ngay |
| Đặt Invitation | chuột phải 1 lần; thành công = nút `activate` hiện TRONG màn hình (journal R16). Activate chỉ hiện khi thiết bị có món |
| Vào portal | nhãn portal = tên khu vực ("Absence of Patience and Wisdom"); có portal cũ → chờ entity portal MỚI (R17) |
| Boss | entity `…/CleansingMonsters/CleansingBoss@84`, tâm arena (252,252), vào arena tại (252,151) |
| Boss chết | Life cur = 0 qua component sinh tồn (R14) — KHÔNG dùng chữ "Map Complete" / entity biến mất / Envoy |
| Về hideout | phím người dùng (`` ` ``) → dự phòng portal thoát; arena CHẶN `/hideout` (R15) |
| Nhặt đồ | quy tắc AutoExile (COMMON_PATTERNS §10); chờ nhãn đầu ≤ 3s |
| Dừng | chỉ khi hết Invitation cả túi lẫn rương guild; End / chuột thật = dừng hẳn |

Chưa giải quyết (2026-10-18): bot chưa tự giết được boss — chuột phải từ ~28 ô, 7s máu vẫn 100% (cần phím skill +
flask của người dùng; Exarch có giai đoạn intro bất tử); click Map Device trúng portal khi 4 portal vây quanh.
