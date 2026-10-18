# Common Architectural Patterns in ExileApi (COMMON_PATTERNS.md)

This reference documents proven patterns for memory access, navigation, combat, and ImGui rendering.

---

## 1. Safe Entity Search
```csharp
private Entity? FindBoss(GameController gc)
{
    if (gc.Entities == null) return null;

    foreach (var entity in gc.Entities)
    {
        if (entity == null || !entity.IsValid || entity.Id == gc.Player?.Id) continue;

        var path = entity.Path ?? string.Empty;
        var renderName = entity.RenderName ?? string.Empty;

        if (path.Contains("ConsumeBoss", StringComparison.OrdinalIgnoreCase) ||
            renderName.Contains("Eater of Worlds", StringComparison.OrdinalIgnoreCase))
        {
            var life = entity.GetComponent<Life>();
            if (life != null && life.CurHP > 0 && !entity.IsDead)
            {
                return entity;
            }
        }
    }
    return null;
}
```

---

## 2. World-to-Screen Absolute OS Input
```csharp
var cam = gc.IngameState.Camera;
var windowRect = gc.Window.GetWindowRectangle();
var screenPos = cam.WorldToScreen(worldPos3D);
var absPos = new Vector2(windowRect.X + screenPos.X, windowRect.Y + screenPos.Y);
BotInput.RapidRightClickAt(absPos);
```

---

## 3. Shift + RightClick Batch Currency Crafting
```csharp
// 1. Prime currency on cursor via Shift + RightClick
MouseHelper.FastDirectMove(currencyScreenPos);
Input.KeyDown(Keys.LShiftKey);
Input.RightDown();
Input.RightUp();

// 2. Left-click every target item in player inventory while holding Shift
foreach (var item in inventoryItems)
{
    MouseHelper.FastDirectMove(item.ScreenPos);
    Input.LeftDown();
    Input.LeftUp();
    Thread.Sleep(craftDelayMs);
}
Input.KeyUp(Keys.LShiftKey);
```

---

## 4. DevTree Currency Slot Traversal Pattern
```csharp
// Path: (OpenLeftPanel/StashElement)49->2->0->0->1->1->0->0->1->[SlotIndex]->1
int[] prefixPath = { 49, 2, 0, 0, 1, 1, 0, 0, 1 };
Element? elem = stashElement;
foreach (var idx in prefixPath)
{
    if (elem != null && elem.IsValid && idx < elem.Children.Count)
        elem = elem.Children[idx];
}
var slotElement = elem?.Children[slotIndex]?.Children[1] ?? elem?.Children[slotIndex];
```

---

# Phần B — Bộ công cụ Python đọc RAM (exileapi-rebuild, Linux / Wine) — 2026-10-17/18

## 5. Tìm tiến trình game qua wrapper (umu-run / Proton)
```python
argv0 = open(f'/proc/{pid}/cmdline','rb').read().split(b'\0')[0].decode().lower()
if argv0.endswith(('pathofexile.exe', 'pathofexile_x64.exe', 'pathofexilesteam.exe')):
    return pid          # tiến trình wine thật; wrapper có argv0 = /usr/bin/python3
```

## 6. Đọc Life / ES / Mana của entity (component sinh tồn, bản 3.29.3)
```python
VT_LIFE = 0x1435a7340                       # vtable component — đổi theo patch, đo lại bằng R14
LIFE_MAX, LIFE_CUR, ES_MAX, ES_CUR = 0x1a4, 0x1a8, 0x23c, 0x240
def life_of(p, ent):
    a, b = q(p, ent + 0x10), q(p, ent + 0x18)          # vector con trỏ component
    for i in range((b - a) // 8):
        c = q(p, a + 8 * i)
        if q(p, c) == VT_LIFE:
            mx, cur = struct.unpack('<ii', p.read(c + LIFE_MAX, 8))
            return (mx, cur) if mx > 0 else None
# boss chết  ⇔  cur == 0   (entity xác vẫn còn — đừng dùng "biến mất")
```
Hiệu chỉnh offset: lấy 1 con số người dùng thấy (ES/Life trong bảng Character) → quét component nhân vật tìm cặp
(max, cur) bằng số đó → so cùng offset trên entity đã biết sống/chết.

## 7. Tìm element UI theo chữ — CHỈ bản đang trong màn hình
```python
def visible_text(p, org, *texts):
    want = {t.lower() for t in texts}
    for e, t in ui_texts(p):                    # ui_texts đã lọc cờ hiển thị (+0x1e9 & 8)
        if t.lower() in want:
            x, y = cell_center(p, e, org[3])
            if 0 <= x <= org[2] and 0 <= y <= org[3]:   # bỏ bản ẩn/sót có toạ độ âm
                return e
```

## 8. Máy trạng thái bot + thang tự phục hồi (boss_cycle.py)
```python
EXPECT = {'HIDEOUT': ('hideout', 20), 'FIGHT': ('arena', None), ...}   # vị trí bắt buộc + ngân sách giây
def check(self):                      # gọi đầu mỗi bước
    need, limit = EXPECT[self.state]
    if limit and time.time() - self.state_t > limit: raise Deviation('quá giờ')
    if need and self.where() != need: raise Deviation(f'cần {need}')
# vòng chính: Deviation/lỗi bước → recover(): bậc 1 đóng bảng+hồi sinh, 2 về hideout, 3 chờ game (PID mới → dựng lại);
# lỗi lặp > N/15' → nghỉ 1→2→5→10 phút (KHÔNG dừng); người dùng (End/chuột/SIGTERM) → dừng hẳn, không phục hồi;
# ResetRequested (SIGUSR1) → bỏ bước, về hideout, xoá bộ đếm.
```
`where()` = no-game / loading (chưa có nhân vật) / dead (chữ Resurrect) / hideout (entity MappingDevice) /
arena (entity boss) / other.

## 9. Dừng khẩn cấp cắt ngang cả sleep
```python
# luồng evdev nghe bàn phím thật (bỏ 'exileapi-*' và 'macro'); gặp phím End:
_GUARD['hit'] = 'DỪNG KHẨN CẤP (phím End)'
os.kill(os.getpid(), signal.SIGINT)          # KHÔNG dùng _thread.interrupt_main() — không cắt time.sleep
```

## 10. Nhặt đồ theo AutoExile (LootSystem / InteractionSystem / BossMode)
1. Chỉ nhãn đồ đang hiện (tôn trọng loot filter), sắp theo gần nhân vật; quét lại mỗi lượt (AutoExile 500ms).
2. Nhãn đè nhãn khác → bấm phím di chuyển 1 lần về phía nhãn (đổi góc nhìn, nhãn tách) rồi mới click.
3. Món sát portal → bỏ (click có thể vào portal).
4. Đọc lại vị trí nhãn ngay lúc click; thành công = món biến mất; ≤ 5 click/món, cách 300ms.
5. Hỏng → chặn 5s → 15s → 30s; đã nhặt → chặn 30s (nhãn chớp lại); biến mất trước click → 0.5s rồi 120s.
6. Có WorldItem trong 30 ô mà không nhãn nào hiện → Z tắt rồi bật nhãn (cooldown 5s).
7. Sweep boss: AutoExile mặc định 5s; đồ rơi ngay khi boss chết → chờ nhãn đầu ≤ 3s.

## 11. Điều khiển bot từ overlay / web (botctl)
- PID bot: quét `/proc/*/cmdline` theo tên script (không tin pid trong file trạng thái cũ).
- start: `subprocess.Popen(..., start_new_session=True, stdin=DEVNULL)`; 1 bot điều khiển game (flock `game.lock`).
- reset: SIGUSR1 → chờ sự kiện "reset" trong trạng thái ≤ 15s → không có thì TERM → KILL → start.
- Kẹt (nhìn từ ngoài) = trạng thái không cập nhật > 90s VÀ không có `sleep_until` trong tương lai.
- Web: `ThreadingHTTPServer`, localhost không cần token, `--lan` bắt `X-Token`; CORS mở để 1 trang gom nhiều máy.

---

# Phần C — Patch file game Content.ggpk (PoeRedux, .NET 10 + LibGGPK3) — 2026-10-16/17

## 12. Mở GGPK & sửa file trong bundle
```csharp
using var ggpk = new BundledGGPK(path, false);   // LibGGPK3.LibBundledGGPK3 2.7.5
ggpk.Index.ParsePaths();
var tree = ggpk.Index.BuildTree(true);
record.Write(newBytes);                          // ghi vào bundle-đang-ghi (MemoryStream) — rẻ
ggpk.Index.Save();                               // lưu index sau cả loạt
```
Linux: cần `liboo2core.so` (lấy từ release linux-x64 của LibGGPK3 cùng phiên bản) đặt cạnh file chạy — Oodle native.
Đọc hàng loạt nhanh: `Index.Extract(files, handler)` / `SortByBundle` (mỗi bundle giải nén 1 lần).

## 13. File văn bản trong game
- `.ot/.otc/.ao/.epk/.pet/.trl/.mat`: UTF-16 LE có BOM `FF FE` (một số UTF-8) — giữ nguyên BOM khi ghi lại.
- `.pet` (hạt) 3 kiểu `particles_per_second`: bảng 7 số; đường cong `[dims] n (t v… "Interp")×n`;
  đường cong không ngoặc `n (t v Linear)×n`; GPU emitter JSON (`"emit_burst"`, `"particles_count"`).
  Nhân hệ số: chỉ nhân các số ngay trước token nội suy (Linear/Smooth/Flat), giữ t và header.
- Bảng `.datc64`: `u32 rows` + rows × rowSize + vùng biến đổi bắt đầu bằng `BB BB BB BB BB BB BB BB`;
  rowSize = (vị trí magic − 4) / rows; cột chuỗi = u64 offset tính từ đầu magic (UTF-16, kết thúc `00 00`).
  Vd `minimapicons`: [0] Id · [8] MinimapIconRadius i32 · [12] LargemapIconRadius i32.

## 14. Backup an toàn cho patch
- Lưu bản gốc + **mã băm bản gốc và bản đã patch** (manifest). Restore chỉ ghi lại file đang = bản đã patch;
  khác cả hai = game đã cập nhật → bỏ qua (không ghi đè bản gốc cũ). Zip hỏng → đổi tên, KHÔNG xoá.
- Patch theo hệ số luôn tính từ bản gốc trong backup (`ReadOriginal`) → áp lại / đổi hệ số không cộng dồn.
- Sau patch đọc lại xác nhận từng file đổi (hash ≠ gốc). "Không exception" ≠ "đã đổi".

## 15. Khôi phục không cần backup (CDN)
`logs/Client.txt` dòng `Web root: https://patch.poecdn.com/<bản>/` → `ServerBundleFactory` tải `_.index.bin` gốc →
so (bundle, offset, size) từng file với index trong máy → file lệch: tải bundle gốc, ghi đè. Cài chuẩn khớp CDN 100%.
