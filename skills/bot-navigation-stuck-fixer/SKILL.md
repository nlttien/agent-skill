---
name: bot-navigation-stuck-fixer
description: >-
  Troubleshooting and fixing bot pathfinding, navigation stuck conditions, A* grid mapping,
  and movement skill / blink teleportation in AutoExile.
---

# Bot Navigation & Stuck Recovery Playbook

## 🔍 When to Use This Skill
- The bot gets stuck running into walls, doorways, or arena obstacles.
- Blink key (`Q`, `Flame Dash`, `Frostblink`, `Leap Slam`) fails to trigger or triggers in the wrong direction.
- Player character oscillates back and forth rapidly without advancing.

---

## 🛠️ Diagnostics & Solutions

### 1. Navigation System Architecture
- **`NavigationSystem.cs`**: Handles high-level target setting, pathfinding, and point-to-point movement.
- **`Pathfinding.cs`**: Implements grid-to-world, world-to-screen transforms, and line-of-sight checks.
- **`BotInput.cs`**: Issues raw mouse movement and key presses.

### 2. Walk-Before-Blink Pattern
When entering a new zone or encounter (e.g. Eater of Worlds):
```csharp
// Always allow character to settle / walk for 500ms before triggering Blink skills:
var walkElapsedMs = (DateTime.Now - _phaseStartTime).TotalMilliseconds;
if (walkElapsedMs >= 500 && distToTarget > 45f && (DateTime.Now - _lastBlinkTime).TotalMilliseconds >= 1200)
{
    _lastBlinkTime = DateTime.Now;
    var screenPos = Pathfinding.GridToScreen(gc, targetGridPos);
    var windowRect = gc.Window.GetWindowRectangle();
    var absPos = new Vector2(windowRect.X + screenPos.X, windowRect.Y + screenPos.Y);
    BotInput.ForceCursorPressKey(absPos, blinkKey);
}
```

### 3. Stuck Watchdog & Recovery
- If `playerGrid` does not change by more than $5\text{g}$ over $1.5\text{s}$ while `IsNavigating == true`:
  1. Trigger immediate `BotInput.StopMovement()`.
  2. Perform randomized local reposition step ($\pm 15\text{g}$).
  3. Re-calculate path with relaxed collision flags.

---

## 🛰️ Phát hiện KẸT khi không nhìn màn hình (điều khiển từ xa nhiều máy) — kế hoạch 2026-10-18

Chi tiết: repo exileapi-rebuild `docs/STUCK.md`. Ý chính cho agent:
1. **Không tin bot tự khai** — tiến trình canh chừng riêng đọc RAM game, so với trạng thái bot ghi.
2. **Kẹt = không có tiến triển** (không xong vòng / không đổi vị trí / máu boss không giảm), không phải "không log".
3. Mã lý do: `BOT_DEAD`, `BOT_FROZEN` (>90s không cập nhật, trừ lúc ngủ có hẹn), `NO_PROGRESS` (>3× thời gian vòng),
   `STATE_TOO_LONG`, `STATE_MISMATCH` (khai FIGHT mà RAM = town), `NOT_MOVING` (>15s khi đang đi), `NO_DAMAGE`
   (>20s trong tầm), `DEATH_LOOP`, `ERROR_LOOP`, `UI_BLOCKED`, `NO_FOCUS` (cửa sổ game không được chọn → phím không
   tới game), `LOADING_STUCK`, `DISCONNECTED` (`Client.txt` "Abnormal disconnect"), `GAME_DEAD`.
4. Mỗi sự cố: mã + số liệu RAM + ảnh cửa sổ game (bằng chứng cho quản trị) → tự xử lý theo mã (Reset / bật lại /
   focus) → 3 lần cùng mã / 30' không hết → đỏ, cần người.

Thang phục hồi đã làm trong `boss_cycle.py`: bậc 1 đóng bảng (Space) + hồi sinh → bậc 2 về hideout → bậc 3 chờ game
(PID mới thì dựng lại bộ đọc RAM); lỗi lặp → nghỉ tăng dần, không dừng.
