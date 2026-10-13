# Field Journal & Bug Post-Mortem Registry (FIELD_JOURNAL.md)

This log preserves real bug fixes, memory behaviors, and architectural solutions across development sessions.

### [2026-08-31] Escape Menu Popup Prevention & Full Inventory Space Handling
- **Target Component**: `Plugins/Source/ShopAutoBuyer/Core/Services/PurchaseExecutor.cs`, `InventorySpaceChecker.cs`, `StashDepositService.cs`, `ShopAutoBuyer.cs`
- **Symptom**: When player inventory became 100% full, the bot pressed Escape repeatedly, opening the Pause / Escape Menu (`RESUME GAME`, `OPTIONS`...), which blocked all in-game mouse and keyboard clicks.
- **Root Cause**:
  1. `InventorySpaceChecker.HasSpaceForItem` relied on `InventoryPanel.IsVisible` (which is false when Shop UI is open), causing it to falsely report free space and attempt Ctrl+Click buy on a full inventory.
  2. `PurchaseExecutor.finally`, `ShopAutoBuyer.Tick()`, and `StashDepositService` each sent `Keys.Escape` without checking if the previous Escape had already closed the UI. The 2nd Escape immediately opened the Game Pause Menu.
- **Fix Applied**:
  1. `InventorySpaceChecker` now checks `ServerInventory.InventSlotItem` grid coordinates directly, accurately detecting 0 free slots.
  2. Added `MouseHelper.IsEscapeMenuOpen(gc)` and `MouseHelper.CloseEscapeMenuIfOpen(gc)` to auto-detect and close the Escape Menu immediately.
  3. Removed redundant `Escape` presses in `PurchaseExecutor.finally`, `ShopAutoBuyer.cs`, and `StashDepositService.cs`.
- **Prevention Rule**: Never fire unverified `Keys.Escape` in multiple consecutive layers. Always verify `IsEscapeMenuOpen` and current UI state before sending keypresses.

---

### [2026-08-31] Teleport Lock During Stash Deposit & Crafting
- **Target Component**: `Plugins/Source/ShopAutoBuyer/Core/Services/PoeLiveTradeCoordinator.cs` & `ShopAutoBuyer.cs`
- **Symptom**: While the bot was in Hideout crafting items or depositing them into Guild Stash, WebSocket live search detected a trade and immediately triggered `/hideout {sellerChar}` (or Direct Travel token), interrupting the stash interaction.
- **Root Cause**: `HandleItemEvaluationAndTravelAsync` loop in `PoeLiveTradeCoordinator` checked `gc.IsLoading` and free slots, but neglected `_isBusyProvider()` (`_stashDepositService.IsDepositing`).
- **Fix Applied**:
  1. Added `_isBusyProvider()` into the pre-travel wait loop: strictly pauses travel until deposit/crafting coroutines complete.
  2. Added safety guard in `ShopAutoBuyer.cs` `_onSellerTargeted` callback to drop teleport commands whenever `_stashDepositService.IsDepositing == true`.
- **Prevention Rule**: All async travel coordinators must check `_isBusyProvider()` / `IsDepositing` before sending chat commands or triggering party teleports.

---

### [2026-08-31] Automated 2-Currency Batch Crafting Before Guild Stash Deposit
- **Target Component**: `Plugins/Source/ShopAutoBuyer/Core/Services/StashDepositService.cs` & `ShopAutoBuyerSettings.cs`
- **Symptom**: Items bought in bulk were deposited directly into Guild Stash in raw uncrafted states without applying required crafting currencies.
- **Root Cause**: Need a sequential 2-stage batch crafting routine to apply Currency #1 (Slot 5) across ALL items in inventory first, then apply Currency #2 (Slot 26) across ALL items before opening Guild Stash.
- **Fix Applied**:
  1. Resolved DevTree slot paths `PathFromRoot: (OpenLeftPanel/StashElement)49->2->0->0->1->1->0->0->1->[SlotIndex]->1` for Slot 5 and Slot 26.
  2. Implemented `ExecuteAutoCraftRoutine()`: opens Personal Stash, switches to `curr` tab, applies Shift+RightClick on Slot 5 -> Left-Click all inventory items, then Shift+RightClick on Slot 26 -> Left-Click all inventory items.
  3. Seamlessly integrated before opening Guild Stash in `ExecuteDepositCoroutine()`. Added hotkey `F9` and HUD button `[🔨 TEST AUTO CRAFT (F9)]`.
- **Prevention Rule**: Always perform AutoCraft in Personal Stash (`curr` tab) first before switching to Guild Stash, and hold Shift to minimize mouse movements during batch item clicks.

---

### [2026-08-31] Eater of Worlds Movement & Timing Optimization
- **Target Component**: `Plugins/Source/AutoExile/Modes/BossEncounters/EaterEncounter.cs`
- **Symptom**: Bot was instantly blinking on the 1st tick upon entering zone, and had a 2.0s wait delay upon reaching the boss arena.
- **Root Cause**:
  1. Blink cooldown was initialized to `DateTime.MinValue`, causing instant blink at spawn before player character settled.
  2. Boss phase 2 had a `2.0s` wait timer copied from Exarch, whereas Eater of Worlds emerges immediately without the long invulnerability descent of Exarch.
- **Fix Applied**:
  1. Added a 500ms walk check (`walkElapsedMs >= 500`) before triggering `Q` (Blink) upon map entry.
  2. Removed post-arrival delay completely: as soon as player reaches $\le 40\text{g}$ from Eater, it triggers 5 flasks and immediately rapid-attacks the center spawn position to hit the boss upon emergence (-0.5s early).
- **Prevention Rule**: Searing Exarch requires a 2.0s wait delay due to sky descent animation; Eater of Worlds must attack immediately without wait delay.

---

### [2026-08-31] SingleFilePackager Stale DLL Extraction Bug
- **Target Component**: `SingleFilePackager/Program.cs` & `SingleFilePackager_Boss/Program.cs`
- **Symptom**: After rebuilding `AutoExile.dll` and running `BUILD_BOSS_PACKAGE.ps1`, launching `AutoBoss_AllInOne.exe` did not run the newest code changes.
- **Root Cause**: Unpacker in `Program.cs` had `if (!File.Exists(destPath) || new FileInfo(destPath).Length != entry.Length)`, which skipped extracting the DLL into `%LOCALAPPDATA%\PoE_AutoBoss_Engine` if the file size remained unchanged.
- **Fix Applied**: Replaced conditional check with unconditional extraction `try { entry.ExtractToFile(destPath, true); } catch { }` so every launch forces the newest payload. Also synchronized DLL copying in build scripts to both `Plugins/Compiled/<PluginName>/` and `Plugins/Compiled/<PluginName>.dll`.
- **Prevention Rule**: Always use `overwrite = true` for self-extracting payload archives in single-file distributions.

---

### [2026-08-31] AutoCraft & Stash Navigation Stalling Fix (Off-Screen Stash & Missing Guild Stash Fallback)
- **Target Component**: `Plugins/Source/ShopAutoBuyer/Core/Services/StashDepositService.cs` & `ShopAutoBuyer.cs`
- **Symptom**: When inventory was full, the bot stood still in Hideout without crafting or depositing, logging `[WARN] [CẢNH BÁO] Không thể mở cửa sổ rương GUILD STASH` repeatedly in a fast restart loop.
- **Root Cause**:
  1. `FindStashEntity(true)` returned `null` when no Guild Stash entity was placed in Hideout (only Personal Stash existed), causing `EnsureStashOpenRoutine` to fail all 15 attempts.
  2. When Stash / Guild Stash was off-screen (e.g. character standing near Faustus), `Camera.WorldToScreen` returned coordinates outside the window viewport, resulting in no mouse clicks and zero character movement.
  3. `ShopAutoBuyer.Tick()` restarted `StartDepositCoroutine()` on every frame when deposit failed, creating an infinite 0ms restart loop that locked the travel coordinator in `Trang thai: Đang Craft đồ / Cất rương Guild — Tạm dừng Travel...`.
- **Fix Applied**:
  1. **Automatic Fallback**: If `FindStashEntity(true)` returns null, it immediately falls back to `FindStashEntity(false)` (Personal Stash) with an alert log, preventing hard lock.
  2. **Off-Screen Navigation**: Clamped `Camera.WorldToScreen` coordinates within viewport bounds `[150, Width-150]` when clicking 3D stash entities, commanding the character to walk towards off-screen stashes until they become visible and open.
  3. **Deposit Throttling**: Added a 2.5s retry throttle (`_lastDepositAttemptTime`) in `ShopAutoBuyer.cs` to prevent infinite per-frame restart loops on failed deposits.
- **Prevention Rule**: Always implement coordinate clamping for 3D interactable entities to allow natural character pathfinding towards off-screen targets, and always provide graceful fallbacks between Guild Stash and Personal Stash.

---

## [2026-10-10] Acc Manager — Web app (Vue 3 + Frappe) session lessons

### [W1] Trang SPA trắng sau refactor: TDZ trong `<script setup>`
- **Target**: `Account-manager/src/views/BoosterAccountsView.vue` (repo GegeTeamCode/Account-manager)
- **Symptom**: Vào `/booster-accounts` content trống tuyệt đối (sidebar cha vẫn render). Lint 0 warning, build xanh, test pass.
- **Root Cause**: Lời gọi `useBoosterMailSync({ items, ... })` nằm ở dòng 334 nhưng `const items = ref([])` khai báo ở dòng 419. `<script setup>` chạy tuần tự → `ReferenceError: Cannot access 'items' before initialization` → component crash ngay setup, không có error boundary.
- **Fix Applied**: Di chuyển lời gọi composable xuống ngay sau khai báo `items`. Quét lại mọi composable call còn lại theo thứ tự khai báo (commit `e05ce00`).
- **Prevention Rule**: (1) Khi tách composable khỏi view, đặt lời gọi LUÔN sau các ref nó nhận vào. (2) Lint/build KHÔNG bắt lỗi runtime TDZ — sau mỗi đợt refactor view phải tự mở từng route bằng browser thật để verify, không chỉ tin gate.

### [W2] Modal không bao giờ mở: sai API component ModalWrapper
- **Target**: `ModalWrapper.vue` + các modal trong BoosterAccountsView
- **Symptom**: Bấm nút mở modal không có gì xảy ra ("không click vào được"), kéo dài nhiều ngày.
- **Root Cause**: Template dùng `:show="x"`, `:title`, `@close` — ModalWrapper chỉ có `modelValue` (v-model). `v-if="modelValue"` luôn falsy → modal không render. Code cũ "đoán" API không tồn tại.
- **Fix Applied**: Chuyển hết sang `v-model` + thêm prop `title` thật cho ModalWrapper (render header chuẩn h3 + ✕ khi không có slot #header). Quét toàn app `:show=` sai — không còn.
- **Prevention Rule**: Trước khi dùng component, đọc `defineProps/defineEmits` của nó — không đoán API theo tên props quen thuộc. Pattern chuẩn modal của repo: `v-model` bật/tắt + slot `#header` hoặc prop `title`.

### [W3] Frappe `rename_doc` không nhận `ignore_permissions` + doctype phải bật `allow_rename`
- **Target**: `gege_custom/api/steam_account.py`, `api/booster_account.py`, doctype Steam Account / Booster Account
- **Symptom**: Sửa tên account (FE modal) báo `TypeError: rename_doc() got an unexpected keyword argument 'ignore_permissions'`; vá xong lộ tiếp `ValidationError: not allowed to be renamed`.
- **Root Cause**: Frappe v15 `frappe.rename_doc(doctype, old, new, force, merge, *, ignore_if_exists, show_alert, rebuild_search)` — không có `ignore_permissions`; và guard `if not (force or ignore_permissions) and not meta.allow_rename: throw` đòi doctype bật `allow_rename: 1`.
- **Fix Applied**: Bỏ keyword `ignore_permissions`; bật `"allow_rename": 1` trong JSON của Steam Account + Booster Account rồi `bench migrate`. Test live rename + revert OK (commit `2ee6cb3`, `8b226cb`).
- **Prevention Rule**: Đổi tên doc (rename docname) cần 2 điều: API gọi đúng signature bản Frappe đang dùng (`inspect.signature` để check) + doctype có `allow_rename: 1`. Đặt `show_alert=False` khi gọi từ API nội bộ.

### [W4] `bench console` qua SSH nuốt block indent — viết script flat
- **Symptom**: Pipe script python có khối `if`/vòng lặp indent vào `bench --site X console` qua ssh: IPython in lẫn lộn `...:`, block không chạy, thỉnh thoảng tự exit.
- **Root Cause**: IPython đọc stdin theo line-mode, xử lý continuation line qua pipe tệ.
- **Fix Applied**: Script test viết FLAT — mỗi dòng 1 lệnh, không indent block (dùng biến trung gian thay if). Grep output để lọc: chuỗi prefix `In [n]:` làm grep anchor `^` trượt — grep không anchor.
- **Prevention Rule**: Vẫn theo bài học cũ "file本地 → scp → chạy": với bench console, script phải flat-lines; thao tác file phức tạp thì viết `python3 /tmp/patch.py` riêng, KHÔNG nhét qua console.

### [W5] Port code giữa 2 repo: lấy kết quả cuối + tôn trọng các strip của repo đích
- **Symptom**: Port 12 commit từ Custom-UI-GeGe-Currency sang Account-manager; commit đầu push thiếu phần strip platform Steam/Client (phải fixup); mất 3 lần build vì boundary cắt hỏng template.
- **Root Cause**: (1) `git add` sót file đã stage-một-phần (trạng thái MM). (2) Cắt template theo dòng mà không hiểu cấu trúc slot (PaginatedListLayout có `#header` và `#filters` là 2 slot riêng — `#filters` có sticky riêng, KHÔNG được gộp vào 1 component). (3) Dedent `[8:]` ăn mất dòng có indent < 8 spaces làm mất tag đóng.
- **Fix Applied**: Port theo "kết quả cuối" (`git checkout <commit> -- <files>`) thay vì cherry-pick từng commit; không dedent (để prettier format); verify cặp `<template>` mở/đóng bằng đếm opens vs closes; cắt đúng ranh giới slot.
- **Prevention Rule**: Khi port nhiều commit giữa repo: mang thẳng bản cuối, áp lại bằng tay các thay đổi có chủ đích của repo đích (như strip platform); `git add` xong kiểm tra `git status` không còn MM; template surgery thì dùng compiler-sfc parse để tìm lỗi vị trí thay vì đoán.

### [W6] Quy hoạch repo Acc Manager (10-2026)
- Repo chính FE từ 2026-10-10: **GegeTeamCode/Account-manager** (đã đủ 4 tab: Booster Accounts / Outlook Mail / Aion 2 / Steam Accounts). Server .113 `/home/frappe/trader-ui` đã trỏ origin sang repo này, chạy branch `port/booster-session` (PR #1 chờ merge; sau merge đổi sang `main`).
- Repo cũ `nlttien/Custom-UI-GeGe-Currency` = archive, không làm việc nữa.
- BE vẫn repo `GegeTeamCode/frappe-erp15-gegecurrency` branch `feat/booster-account-mgmt`, sửa trực tiếp trên .113 rồi commit push; thay đổi doctype → `bench migrate` + `supervisorctl restart`.
- Người phụ trách Aion 2: field `assigned_to` (Link User) trên character; set 1 lần cập nhật toàn bộ character cùng email (như PC/Player/Gearup); FE badge 🛡️ tím trên header nhóm.
- Component pattern của repo: `components/booster/` — modal dùng `v-model` + props dữ liệu + emit `changed`/`saved` về view refresh; logic mức dòng ở `composables/useBoosterRow.js`, mail ở `useBoosterMailSync.js`.
- Chống mìn tương lai: mỗi lần refactor xong PHẢI browser-verify từng route; PR nhỏ theo chặng, gate `lint → build → prettier → format:check → test` trước mỗi push.

### [W7] Self-verify bằng browser thật trước khi bàn giao (quy trình chuẩn từ 2026-10-10)
- **Vấn đề**: Gate (lint/build/test) không bắt lỗi runtime (TDZ [W1], API component sai [W2]) → người dùng thành người test đầu tiên.
- **Quy trình đã chứng minh** (khoảng 5 phút, chạy được ngay):
  1. **User test trên .113**: bench console (script flat) — `agent-test-user (tên/mật khẩu lưu trong .ai本地, không ghi repo công khai), roles `Game Currency Admin` + `Booster`; set password qua `u.new_password = ...; u.save()`. Verify bằng `curl POST /api/method/login`.
  2. **SSH tunnel**: `ssh -N -L 8080:127.0.0.1:80 frappe@192.168.2.113` chạy background (không dùng `&` trong lệnh thường — bị chết theo shell) → app truy cập được tại `http://localhost:8080`.
  3. **Browser skill (browser-use:control-browser)**: mở tab → login form (fill 2 textbox theo placeholder + nút "🚀 Bắt đầu") → `goto` từng route → `domSnapshot()` đo độ dài + marker từ khóa → thử thao tác thật (VD: select người phụ trách) → đối chiếu DB bằng bench console.
- **Chuẩn đỗ route**: snapshot có nội dung thực (marker/domain data hiển thị), KHÔNG chỉ navbar. Lưu ý: snapshot ngắn chưa chắc lỗi — có thể chỉ là dữ liệu ít (Aion2 1 account test = 1.6K ký tự hợp lệ); đối chiếu marker + heading.
- **Prevention Rule**: sau mỗi deploy FE lên .113, LUÔN chạy quy trình này qua các route vừa đụng trước khi báo user "xong".

### [W8] frappe.rename_doc chậm ~2s trên DB ERPNext lớn — rename SQL trực tiếp khi không có Link field
- **Symptom**: Đổi tên Steam Account qua API mất ~2–2.3s (user thấy "rất tốn thời gian"), dù đã tắt `rebuild_search` + bỏ double-save.
- **Root Cause**: Profile console tách bước: get_doc 66ms, save 0.4s, **frappe.rename_doc 2.2s** — nó quét link fields trên toàn bộ database (hàng trăm bảng ERPNext) để cập nhật Link trỏ tới doc, kể cả khi không bảng nào Link tới.
- **Fix Applied**: Verify `get_link_fields("Steam Account") == []` → rename bằng 3 câu SQL trực tiếp: UPDATE name trong bảng chính, UPDATE `parent` trong child table (`Steam Account Game`), UPDATE `name` trong `__Auth` (password field), rồi `frappe.clear_cache(doctype=...)`. Kết quả: **2.3s → 0.09–0.29s**, games + mật khẩu decay nguyên vẹn (đã verify).
- **Prevention Rule**: (1) Trước khi dùng `frappe.rename_doc`, check `get_link_fields(doctype)` — nếu rỗng, SQL trực tiếp nhanh gấp ~20 lần. (2) Thiết kế doctype mới: cân nhắc autoname docname thay vì đặt docname theo tên người dùng nhập — đổi "tên hiển thị" không nên đồng nghĩa đổi khoá chính. (3) Khi điều tra chậm: profile từng bước trong bench console (`time.time()` quanh mỗi call) thay vì đo cả HTTP.

### [W9] Bẫy đa triển khai (multi-deployment): "fix rồi mà vẫn vậy" = user đang chạy app ở MÁY KHÁC
- **Symptom**: Fix + browser-verify end-to-end trên .113 xong xuôi, user vẫn báo "vẫn vậy" 3 lần liên tiếp.
- **Root Cause**: App chạy **nhiều nơi trong LAN**: .113 (dev tôi deploy) và **.228:80 prod + .228:5173 vite dev** (build khác, không có fix, dữ liệu thật nằm ở BE .228). User test trên .228 — mọi fix trên .113 vô nghĩa với họ.
- **Detection**: khi user báo lỗi UI nhưng browser-verify của tôi pass — probe LAN (`curl -m 2` các IP:80/5173/8080 quen thuộc), so sánh build hash `assets/index-*.js` giữa các nơi.
- **Prevention Rule**: (1) Đầu phiên làm việc, hỏi/chốt URL app user đang dùng (địa chỉ + port). (2) Sau khi fix, confirm build hash user đang tải (F12 Network) trùng bản mới. (3) Ghi danh sách deployments vào AGENTS.md của repo.

### [W10] Steam Guard code: pattern 5 ký tự phải bắt buộc có CHỮ SỐ
- **Symptom**: Trích mã Steam Guard từ mail thật → trả về chữ "email" làm mã (false-positive; mail Steam có cụm "Steam Guard code" + từ 5 ký tự liền).
- **Root Cause**: Regex `[A-Z0-9]{5}` + IGNORECASE match mọi từ 5 chữ cái; mail Steam nhiều từ tiếng Anh 5 ký tự ("email", "click"...).
- **Fix Applied**: Lọc match phải `any(ch.isdigit())` (mã Steam Guard luôn chứa số) cho cả nhánh label lẫn standalone-line. Verify mail thật: hết false-positive; khi không có mail Steam mới → báo "không tìm thấy" + show nội dung mail gần nhất.
- **Prevention Rule**: Regex trích mã OTP phải có ràng buộc hỗn hợp chữ+số (hoặc format chặt), đừng tin độ dài một mình; test với mail thật trước khi bàn giao.

### [W11] Hai phiên agent song song trên cùng repo: file bị ghi đè giữa chừng
- **Symptom**: Debug edits vừa viết biến mất (working tree clean dù chưa commit); file rơi vào trạng thái lai (mảnh code mới + block cũ, mất định nghĩa hàm nhưng còn lời gọi) — build vẫn xanh vì mảnh lai tình cờ hợp lệ, runtime sai.
- **Root Cause**: Một phiên agent khác chạy đồng thời trên cùng thư mục làm việc, rebase/checkout/ghi file đè lên edits của phiên này.
- **Detection**: (1) `git status` clean dù vừa sửa → nghi ngay ghi đè. (2) Edit no-op: `str.replace` cũ KHÔNG assert count vẫn in OK — mọi python patch PHẢI `assert count == 1` trước/sau replace. (3) So marker trong file thật (grep/sed) thay vì tin rằng lần sửa trước đã ăn.
- **Prevention Rule**: (1) Mọi patch bằng python phải assert số lần match. (2) Khi nghi song song: `git reflog` + `git log` so commit không phải của mình + chốt với user "một repo một phiên tại một thời điểm". (3) Deploy .113 phải luôn theo chu trình commit→push→fetch→reset→build — build trên cây bẩn là build ảo.

### [W12] Lệch timezone naive datetime làm sai lệch logic chu kỳ hồi phục (+30 NL)
- **Symptom**: Khi cập nhật Năng lượng trong Aion 2 (ví dụ nhập 0), sau khi lưu giá trị hiển thị tự động nhảy lên thành 30 (nhập 630 nhảy thành 660).
- **Root Cause**:
  1. `sync_aion2_resources` gọi `frappe.utils.get_datetime(int(timestamp_ms) / 1000)` truyền số timestamp float/int -> `frappe.utils.get_datetime` chỉ nhận string date hoặc datetime object nên trả về `None`, khiến `resource_timestamp` lưu thành `NULL` trong DB.
  2. Khi serialize dữ liệu ra FE (`_serialize`), `doc.resource_timestamp` bị `None` nên fallback sang `frappe.utils.now_datetime().timestamp() * 1000`. Nhưng `now_datetime()` là naive UTC datetime, khi gọi `.timestamp()` trong Python trên máy chủ có OS timezone `Asia/Ho_Chi_Minh (UTC+7)`, Python hiểu đó là local time và lùi 7 tiếng về quá khứ (-7h).
  3. FE có logic đếm chu kỳ hồi phục 3h: 7h trôi qua tương đương `floor(7/3) = 2 chu kỳ`. Với VIP hồi 15 NL/chu kỳ, 2 x 15 = 30 NL! Khiến giá trị vừa lưu lập tức bị FE tính dồn thêm 30 điểm.
- **Fix Applied**:
  - Backend: Dùng `datetime.datetime.utcfromtimestamp(float(timestamp_ms) / 1000)` để parse epoch ms chuẩn xác vào DB.
  - `_to_ms`: Gán `tzinfo=datetime.timezone.utc` cho naive datetime trước khi gọi `.timestamp() * 1000`.
  - Fallback timestamp: Dùng `int(time.time() * 1000)` thay cho naive `now_datetime().timestamp()`.
  - FE: Nhập bao nhiêu lưu bấy nhiêu (`Math.max(0, Math.min(val, max))`).
- **Prevention Rule**: (1) Tuyệt đối không gọi `.timestamp()` trên naive datetime trong Python khi server chạy khác múi giờ UTC. Luôn gán `tzinfo=timezone.utc` hoặc dùng timezone-aware. (2) `frappe.utils.get_datetime()` không parse được số epoch float/int — phải dùng `fromtimestamp` / `utcfromtimestamp`.

### [W13] Clipboard API trên Insecure Context (HTTP / IP LAN) bị chặn gây lỗi "Copy thất bại"
- **Symptom**: Nút copy trên app chạy qua mạng LAN (http://192.168.2.113) bị báo "Copy thất bại" khi người dùng click vào icon sao chép (Steam login, mật khẩu, text...).
- **Root Cause**: `navigator.clipboard.writeText` theo tiêu chuẩn W3C chỉ được phép hoạt động trong Secure Context (HTTPS hoặc localhost). Khi web app chạy trên HTTP thuần qua IP LAN nội bộ (`http://192.168.x.x`), trình duyệt (Chrome, Edge, Firefox) chặn `navigator.clipboard` (bị undefined hoặc ném `NotAllowedError` / `SecurityError`). Việc gọi trực tiếp `await navigator.clipboard.writeText()` mà không có fallback khiến luồng nhảy vào `catch` và báo "Copy thất bại".
- **Fix Applied**: Tạo utility `copyToClipboard` kiểm tra `window.isSecureContext` trước khi gọi `navigator.clipboard`, và luôn có fallback qua `textarea` ẩn + `document.execCommand('copy')` để đảm bảo sao chép thành công 100% trên cả HTTP LAN lẫn HTTPS.
- **Prevention Rule**: Không bao giờ gọi trực tiếp `navigator.clipboard.writeText` mà không có fallback `document.execCommand('copy')` trong các ứng dụng nội bộ chạy trên HTTP / IP LAN.

### [W14] Lỗi đè chồng nút (Overlap) trong Flexbox: Cấm dùng shrink không kiểm soát trên flex item chứa ruột cố định
- **Target**: Vue3 / TailwindCSS (Toolbar / Action Bar đa nút).
- **Symptom**: Khi hiển thị thanh công cụ gồm 2 khối (Khối trái và Khối phải) dùng \`justify-between\`, các nút của Khối trái (\`🛡️ Người phụ trách\`, \`🎮 Steam\`) bị văng ra ngoài và **NẰM ĐÈ CHỒNG LÊN** các nút của Khối phải (\`🐹 Shugo\`, \`🏰 Daily\`), chữ nọ xọ chữ kia làm giao diện nát bét (như ảnh thực tế của người dùng).
- **Root Cause**:
  1. Đặt \`shrink\` (\`flex-shrink: 1\`) trên container Khối trái, trong khi Khối phải đặt \`shrink-0\`.
  2. Khi không đủ chiều rộng, Flexbox ép hộp chứa của Khối trái co lại (ví dụ từ 250px xuống 60px). NHƯNG các phần tử con bên trong Khối trái lại có kích thước cố định (\`shrink-0\` hoặc không thể co).
  3. Vì Khối trái không có \`overflow: hidden\`, ruột của nó bị văng ra khỏi hộp chứa và **phủ đè lên Khối phải** bên cạnh!
- **Fix Applied**:
  1. Loại bỏ hoàn toàn class \`shrink\` trên container Khối trái — đặt cả Khối trái và Khối phải là **\`shrink-0\`** trong container **\`min-w-max\`** có **\`overflow-x-auto scrollbar-hide\`**.
  2. Khi cả 2 khối đều là \`shrink-0\`, trình duyệt buộc phải xếp chúng cạnh nhau theo thứ tự vật lý, **TUYỆT ĐỐI KHÔNG BAO GIỜ BỊ ĐÈ CHỒNG LÊN NHAU**.
  3. Cắt gọt text thừa của các nút phụ (chữ "Shugo:", "Daily:", "Kinah" không cần thiết) để thanh công cụ luôn siêu gọn gàng (~650px), vừa khít màn hình trên đúng 1 hàng duy nhất mà không bị xuống dòng.
- **Prevention Rule**: Tuyệt đối không dùng \`flex-shrink: 1\` (\`shrink\`) trên flex item nếu các phần tử con của nó không thể co lại tương ứng (vì sẽ gây lỗi tràn đè chồng lên flex item kế bên). Luôn dùng \`shrink-0\` kết hợp \`min-w-max\` và \`overflow-x-auto\`.
