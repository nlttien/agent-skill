---
name: ggpk-file-patching
description: >-
  Patch file game Path of Exile trong Content.ggpk (PoeRedux / LibGGPK3): sửa .ot/.pet/.datc64/.hlsl,
  backup có mã băm, restore an toàn, khôi phục từ CDN, chẩn đoán màn hình đen / treo khi vào khu vực.
---

# Patch Content.ggpk an toàn (PoeRedux, Linux native)

## 🔍 Khi nào dùng
- Viết / sửa patch PoeRedux (zoom camera, giảm hiệu ứng, icon minimap, tắt sương mù…).
- Game lỗi sau khi patch: màn hình đen, treo khi vào hideout/map, hộp thoại lỗi component, disconnect.
- Cần khôi phục file game không có backup.

## ✅ Quy tắc bắt buộc
1. **Không ghi khi có tiến trình khác mở Content.ggpk** (game, wineserver, rsync…) — kiểm `/proc/*/fd` NGAY TRONG hàm ghi,
   kể cả script thử nghiệm (journal R19).
2. Backup bản gốc + mã băm gốc/đã patch; restore chỉ ghi file đang là bản patch; backup hỏng → đổi tên, không xoá (R10).
3. Patch theo hệ số tính từ bản gốc (backup) → áp lại không cộng dồn. Sau patch đọc lại xác nhận từng file.
4. **Không thêm component dùng chung client/server vào base class quái** (`metadata/monsters/monster.ot`): server không
   gửi dữ liệu component đó → lỗi "Component … does not exist" / treo ở hideout (minion kế thừa) / disconnect (R11).
   Khối `client { X }` chỉ bổ sung cho component X đã có.
5. Chỉ dùng kỹ thuật game hoặc patch khác đã chạy thật; rỗng khối mesh làm load cực chậm; muốn "không vẽ" dùng graph
   `invisible.fxgraph` của game (R12).
6. Thử 1 patch gameplay mới mỗi lần, qua town → hideout → map.

## 🛠️ Công cụ (PoeRedux CLI `poeredux`)
`list` · `apply <ggpk> <patch…>` (tự đọc lại xác nhận) · `status` (patched / original / game cập nhật) ·
`restore <ggpk> [file | thư-mục/]` · `repair <ggpk> [--dry-run]` (so index CDN, tải bundle gốc) · `fix-hideout` ·
`ls / cat [--raw] / grep` (đọc file game để viết patch).

## 🔬 Chẩn đoán nhanh
| Triệu chứng | Nghi |
|---|---|
| Màn hình đen | shader bị thay `return (float4)0;` (Black Screen patch) → `repair` |
| Hộp thoại "Component X does not exist … base class Metadata/Monsters/Monster" | component chỉ thêm trong `client { }` |
| Vào town được, hideout/map treo (log dừng sau nạp khu vực) | sửa base class quái → minion / quái kế thừa |
| Patch "xong" mà game không đổi | đoạn cần sửa không còn (file đã patch từ trước / game đổi) → so mã băm |

Định dạng file & mẫu code: `knowledge-base/COMMON_PATTERNS.md` Phần C.
