---
name: frappe-large-list-query
description: >-
  Thiết kế doctype + API danh sách Frappe/MariaDB cho bảng lớn (10k–100k+ dòng) mà mỗi lần tải chỉ đọc ~20 dòng:
  mã hoá giá trị lặp thành số nhỏ (0/1, 1..n) + bảng danh mục giải nghĩa, cột thứ tự tính sẵn có index,
  nhóm "chưa xong" lên trước, đánh số thứ tự 1,2,3… cho nhóm đã xong để nhảy trang số, con trỏ (keyset),
  cache đúng cách, đo số dòng DB thực đọc. Dùng khi làm trang danh sách/KPI/phân trang/tìm kiếm trên Frappe.
---

# Danh sách lớn trên Frappe: mỗi lần chỉ đọc ~20 dòng

Đo thật 2026-10-27 (Frappe/ERPNext v15.121.5 + MariaDB 10.6, podman local, gunicorn 2 worker × 4 thread, 50.000 đơn):
bản cuối **~7 ms/trang ở mọi trang (1 → 2.500)**, 40 người dùng đồng thời p95 ~300 ms, ~230–290 req/s, 0 lỗi.
Chi tiết từng bước + số liệu: `knowledge-base/FIELD_JOURNAL.md` [W29].

## 🔍 Khi nào dùng
- Doctype sẽ có hàng chục nghìn dòng trở lên và có trang danh sách / tab / thẻ KPI / phân trang số / tìm kiếm.
- Trang danh sách chậm, deep page chậm, KPI chậm, cache "không ăn".
- Câu hỏi "Frappe có chịu nổi N dòng không" — trả lời bằng số đo, không phỏng đoán.

## ✅ Quy tắc bắt buộc

1. **Giá trị lặp → số nhỏ + bảng danh mục giải nghĩa** (quy tắc owner chốt 2026-10-27).
   - Cờ hai giá trị (active, đã xong, VIP…) → `Check` (TINYINT 0/1). Ý nghĩa 0/1 ghi ở description của field.
   - Trạng thái / loại có nhiều giá trị → cột `*_code` (Int nhỏ: 1,2,3…) + **doctype danh mục**
     (`autoname: field:code`; field `code` Int unique, `label`, `is_done`/nhóm, `color`, `sort`).
     Đơn chỉ lưu số; nhãn/màu lấy từ danh mục (cache trong `frappe.cache`, xoá cache khi sửa danh mục).
   - Thuộc tính suy ra từ trạng thái (vd `is_done`) **lưu thành cột riêng**, set trong `validate()` — để vào index.
   - Lý do chính không phải tiết kiệm dung lượng mà là **index đi theo thứ tự**: `status IN ('A','B','C','D')`
     làm MariaDB gom 4 khoảng rồi sắp xếp lại (đọc 315/400 dòng); `is_done = 0` trên index `(is_done, sort_rank, name)`
     đọc thẳng 20 dòng.
2. **Cột thứ tự tính sẵn có index** (`sort_rank` BIGINT, set ở `validate()`), không `ORDER BY` biểu thức.
   `ORDER BY x IS NULL, x` hay `CASE …` → MariaDB không dùng index → sắp xếp cả bảng.
   Mẫu "chưa xong lên trước, deadline gần nhất trước; đã xong mới nhất trước":
   ```
   chưa xong: UNIX_TIMESTAMP(deadline)  (không deadline → 9_000_000_000, cuối nhóm)
   đã xong : 10_000_000_000 + (10_000_000_000 − UNIX_TIMESTAMP(completed_at))
   ```
3. **Nhóm đã xong đánh số thứ tự 1,2,3…** (`done_seq` Int có index, gán MAX+1 khi vào nhóm đã xong).
   Trang số P: `k = (P−1)·20 − số_đơn_chưa_xong` → `WHERE done_seq <= MAX − k ORDER BY done_seq DESC LIMIT 20` → đọc 21 dòng
   dù là trang 2.500. Nhóm chưa xong (vài trăm) lấy bằng OFFSET trong nhóm — rẻ.
   Đơn mở lại → để lỗ số (trang thiếu vài dòng chấp nhận được) hoặc đánh lại số ban đêm.
4. **Không OFFSET sâu trên cả bảng** (OFFSET 49.950 = đọc 50.001 dòng, ~200 ms). Dùng `done_seq` (mục 3)
   hoặc con trỏ keyset cho "tải thêm".
5. **Con trỏ keyset phải viết dạng range**, KHÔNG dùng so sánh bộ:
   ```sql
   -- SAI: MariaDB không dùng index → đọc ~50k dòng
   WHERE (sort_rank, name) > (%(r)s, %(n)s)
   -- ĐÚNG: đọc đúng 20 dòng
   WHERE sort_rank >= %(r)s AND (sort_rank > %(r)s OR name > %(n)s) ORDER BY sort_rank, name LIMIT 20
   ```
6. **KPI / đếm tab chỉ quét tập nhỏ**: điều kiện "đang mở" (index) hoặc `completed_at >= CURDATE()` (index);
   tổng toàn bảng + KPI cache 10–15 s (xoá cache khi đổi trạng thái). Không `SUM(CASE…)` trên cả 50k dòng mỗi lần tải.
7. **Cache có hạn trong Frappe v15: đọc phải truyền `expires=True`.**
   `frappe.cache.get_value(k)` khi trượt sẽ ghi `None` vào `frappe.local.cache`; `set_value(k, v, expires_in_sec=…)`
   chỉ ghi Redis, không sửa local cache → mọi lần đọc sau trong cùng tiến trình vẫn `None` → cache không bao giờ trúng.
   ```python
   v = frappe.cache.get_value(KEY, expires=True)
   if v is None:
       v = compute()
       frappe.cache.set_value(KEY, v, expires_in_sec=15)
   ```
8. **Tìm kiếm**: khớp đúng mã đơn / mã sàn trước (`name = q OR external_id = q`, index → 3 dòng). Không khớp mới
   `LIKE '%q%'` trên nhiều trường: bỏ index thứ tự (`IGNORE INDEX (sort_rank)`) để quét tuần tự 1 lần và lấy tổng bằng
   `COUNT(*) OVER ()` trong cùng câu (200→100 ms ở 50k). Cần nhanh hơn ở 100k+ → FULLTEXT / cột tìm kiếm riêng.
9. **1 API trả cả rows + tổng + KPI + đếm tab**; lấy đúng cột cần; tính trạng thái hiển thị (deadline…) ở server cho 20 dòng.
   Tránh: `frappe.get_doc` trong vòng lặp (50 dòng = 44 ms, 6× chậm hơn 1 câu SQL), tải hết về FE
   (`get_all(limit=0)` 50k = 2,6 s / 47 MB), `fields=["*"]`.
10. **Mọi cột khai báo trong JSON doctype**, không `ALTER TABLE` lúc chạy (site mới tạo từ repo sẽ thiếu cột — W28).
    Index: `search_index` cho cột đơn; index kép tạo bằng `frappe.db.add_index(dt, [cols], name)` trong `after_migrate`
    hoặc patch. DDL trong code: `frappe.db.sql_ddl` / `add_index` (lệnh `ALTER` qua `frappe.db.sql` →
    `ImplicitCommitError`).

## 📏 Đo số dòng DB thực đọc (kiểm chứng "chỉ quét 20")
```python
def rows_read(fn):
    snap = lambda: sum(int(v) for _, v in frappe.db.sql("SHOW SESSION STATUS LIKE 'Handler_read%%'"))
    a = snap(); fn(); return snap() - a
```
Đo từng câu con riêng — một câu đếm lẫn vào (vd cache không trúng) sẽ làm mọi trang cùng đọc thêm ~N dòng.
`FLUSH STATUS` cần quyền RELOAD (user site không có) → dùng hiệu trước/sau.
Tải HTTP: script Python `urllib` + thread, đăng nhập `/api/method/login`, đo p50/p95/p99 ở 1/5/10/20/40 người.

## 🧭 Frappe làm được gì (trả lời owner 2026-10-27)
| Ý tưởng | Frappe | Cách làm |
|---|---|---|
| Cờ 0/1 | ✅ sẵn | `Check` = TINYINT 0/1 |
| Mã số nhỏ + bảng giải nghĩa | ✅ được, có lưu ý | Field `Int` + doctype danh mục (`autoname field:code`). Frappe **không tạo khoá ngoại DB** (kể cả Link) → kiểm mã hợp lệ trong `validate()`. Link field luôn lưu chuỗi `name` (varchar); muốn số thật thì dùng Int + danh mục như trên. Desk hiện số thô → dùng formatter / trang SPA hiện nhãn |
| Cột tính sẵn + index kép | ✅ | field ẩn set ở `validate()`; `frappe.db.add_index` |
| Chưa xong lên trước, chỉ đọc 20 dòng | ✅ | `sort_rank` + index `(is_done, sort_rank, name)` |
| Trang số 1,2,3… không OFFSET sâu | ✅ | `done_seq` (mục 3) |
| Con trỏ keyset | ✅ (tự viết) | `frappe.get_list` không có sẵn → API whitelisted + `frappe.db.sql` / `frappe.qb` |
| Cache KPI | ✅ | `frappe.cache` (Redis) + `expires=True` |
| Window function, IGNORE INDEX | ✅ | MariaDB 10.6 qua `frappe.db.sql` |
| Realtime / job định kỳ | ✅ | `frappe.publish_realtime`, `scheduler_events` |
