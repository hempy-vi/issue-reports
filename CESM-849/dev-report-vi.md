# Dev Report — CESM-849

**Task:** Fix WePOP Batch Print Wrong Customer Data and Unresponsive Print All Button (IBL020/IBL040)

---

## 2026-09-14

**1. Nhận bug report và phân tích ban đầu**
Bug report CESM-849 mô tả 3 vấn đề trên WePOP (module Barcode Label):
- Bug 1: In hàng loạt ("Print All") ở IBL020/IBL040 hiển thị sai Customer trên label cho khách hàng ESTEC (ví dụ ra "VO", "VJ", "VQ"... thay vì "ESTEC VIỆT NAM"), trong khi in đơn lẻ ("Print") luôn đúng.
- Bug 2: Layout label ESTEC có thêm dòng "Shipment Qty" thừa so với mẫu khách cung cấp.
- Bug 3: Nút "Print All" ở IBL020 bấm không có phản hồi gì.

**2. Điều tra chain dispatch report qua MCP oracle-shinwoo**
Xác định chain: `LG_RPT_IBL520` (dispatcher Print đơn lẻ, dùng chung IBL020+IBL040) và `LG_RPT_IBL520_1`/`LG_RPT_IBL540_1` (dispatcher Print All riêng cho IBL020/IBL040) — cả 2 cùng định tuyến theo `customer LIKE '%SATO%'`, `order_type='00'`, hoặc `TLG_PR_FACTORY_PK` sang 4 leaf procedure: `LG_RPT_IBL522`/`523`/`524`/`526` (bản base) và `LG_RPT_IBL522_1`/`523_1`/`524_2`/`526_1` (bản batch, dùng CHUNG giữa 2 dispatcher `_1`/`540_1`).

Chạy workflow 4 agent song song so sánh biểu thức Customer + join `TCO_BUSPARTNER` giữa 4 leaf proc "_1"/"_2" và bản base — kết quả: join đều đúng (`C.PK(+) = A.TCO_BUSPARTNER_PK`, đọc theo package), không phải lỗi join Customer như nghi vấn ban đầu.

**3. Đọc lại toàn bộ (không chỉ đoạn Customer) 4 leaf proc "_1"/"_2" — phát hiện root cause thật**
Cả 4 proc `LG_RPT_IBL522_1`, `LG_RPT_IBL523_1`, `LG_RPT_IBL524_2`, `LG_RPT_IBL526_1` lọc bảng `TLG_PA_PACKAGES` chỉ bằng:
```sql
AND A.TABLE_NM = 'TLG_GD_PLAN_D'
AND A.TABLE_PK = Z.PK
```
(có comment `-- [FIX]` trong source, cho thấy đây là 1 thay đổi có chủ đích trước đó — dành cho IBL040, vì package của IBL040 link qua cột generic `TABLE_NM`/`TABLE_PK`).

Nhưng `LG_RPT_IBL520_1` (dispatcher Print All của **IBL020**) lại truyền vào các proc này 1 giá trị `TLG_SA_SALEORDER_D_PK` (PK dòng SO detail) — hoàn toàn khác kiểu với `TLG_GD_PLAN_D_PK` mà điều kiện trên mong đợi. Package sinh từ IBL020 (`LG_UPD_IBL020_V3`) được lưu trực tiếp vào cột `TLG_SA_SALEORDER_D_PK`, KHÔNG dùng `TABLE_NM`/`TABLE_PK` — xác nhận qua `LG_SEL_IBL020_BC`:
```sql
AND A.TLG_SA_SALEORDER_D_PK = P_TLG_SA_SALEORDER_D_PK
```
=> Khi bấm Print All ở IBL020: không có dòng `TLG_GD_PLAN_D` nào trùng số PK → cursor rỗng → im lặng không in gì (Bug 3). Hiếm khi PK trùng ngẫu nhiên với 1 dòng Plan thật của khách khác (PK là số nguyên dùng chung xuyên suốt schema) → hiện nhầm dữ liệu khách đó (Bug 1). **Bug 1 và Bug 3 ở IBL020 là hệ quả của cùng 1 lỗi.**

**4. Viết fix và áp dụng**
- SQL: khôi phục lại điều kiện lọc gốc song song (OR) với điều kiện hiện tại cho cả 4 proc "_1"/"_2":
```sql
AND ( (A.TABLE_NM = 'TLG_GD_PLAN_D' AND A.TABLE_PK = Z.PK)
   OR A.TLG_SA_SALEORDER_D_PK = Z.PK )
```
An toàn cho IBL040 (nhánh cũ giữ nguyên), khôi phục đúng cho IBL020.
- C# (`WEPOP MVC/Views/PrintPreview.cs`, hàm `Print()`): thêm nhánh hiện `XtraMessageBox` khi `dataTable` rỗng (trước đây thoát hàm im lặng — chính là nguyên nhân trực tiếp của "bấm Print All không thấy gì"), và thêm `XtraMessageBox.Show` vào catch khi 1 dòng dữ liệu có cột đầu (`labelKey|className|ver`) bị null/sai định dạng (trước chỉ ghi log file, không hiện gì cho user).

**5. Sự cố phát sinh: toàn bộ chuỗi in label bị INVALID**
User thử tự "restore" các proc base bằng cách dán nguyên khối text SQL từ bug report gốc (còn lẫn các dòng nhãn tiếng Anh thuần "search master"/"search detail"/"print detail (đúng)") thẳng vào tool chạy SQL, gặp lỗi:
```
[Error] Compilation (2: 2): PLS-00103: Encountered the symbol " " ...
```
Kiểm tra `all_objects`/`all_errors`: **10 proc trong chuỗi in label** (`LG_RPT_IBL520`, `LG_RPT_IBL520_1`, `LG_RPT_IBL522`, `LG_RPT_IBL522_1`, `LG_RPT_IBL523`, `LG_RPT_IBL523_1`, `LG_RPT_IBL524`, `LG_RPT_IBL524_2`, `LG_RPT_IBL526`, `LG_RPT_IBL526_1`, `LG_RPT_IBL540_1`) đang **INVALID** với lỗi compile thật — ảnh hưởng CẢ IBL020 và IBL040, MỌI khách hàng, không chỉ ESTEC.

**6. Chẩn đoán và khôi phục**
Dùng `DUMP(text,1016)` trên `all_source` xác nhận source đang lưu chứa ký tự **non-breaking space (U+00A0, hex `c2 a0`)** xen lẫn với space thường trong phần thụt lề — vô hình khi xem bình thường nhưng PL/SQL không chấp nhận, đúng khớp lỗi PLS-00103. `ALTER PROCEDURE ... COMPILE` không khắc phục được (source lưu thật đã bị nhiễm, không phải cache cũ).

Viết lại (gõ tay, không copy-paste) toàn bộ 11 proc, tự kiểm tra bằng `grep` xác nhận 0 ký tự NBSP, chạy `CREATE OR REPLACE` khôi phục — gồm luôn fix ở bước 4 cho 4 proc "_1"/"_2". Sau khi chạy: xác nhận cả 11 proc `VALID`.

**7. Phát hiện thêm 2 proc bị ảnh hưởng (ngoài chuỗi in label)**
Màn IBL040 báo lỗi `PLS-00905: object SWPROD.LG_SEL_IBL040 is invalid` khi search. Kiểm tra: `LG_SEL_IBL040` và `LG_SEL_IBL040_BC` cũng bị nhiễm NBSP từ đúng lần chạy script lỗi trên (2 proc này nằm trong nội dung gốc user dán nhưng ngoài phạm vi chuỗi in label nên chưa được phát hiện ở bước 6). Viết lại sạch (gõ tay, xác nhận 0 NBSP), khôi phục — quét lại toàn bộ `all_objects` với điều kiện `INVALID` + sửa trong ngày, xác nhận không còn proc nào khác bị ảnh hưởng.

**8. Chuẩn bị bước cải tiến kiến trúc (chưa thực thi)**
Theo đề xuất của user: tách hẳn 4 proc "_1"/"_2" hiện đang dùng CHUNG giữa IBL020 và IBL040 (nguồn gốc của toàn bộ sự cố) — tạo mới 4 proc riêng cho IBL020 (`LG_RPT_IBL522_2`, `LG_RPT_IBL523_2`, `LG_RPT_IBL524_3`, `LG_RPT_IBL526_2`, chỉ dùng điều kiện `TLG_SA_SALEORDER_D_PK`), sửa `LG_RPT_IBL520_1` gọi sang proc mới, sau đó revert 4 proc cũ về lại đúng 1 điều kiện `TABLE_NM`/`TABLE_PK` (bỏ nhánh OR) — để từ đó IBL020 và IBL040 không còn share bất kỳ leaf procedure nào. Đã viết sẵn SQL, chưa chạy (không gấp, hệ thống đang ổn định với fix bước 4-6).

**Việc còn tồn (chưa xử lý trong phiên này):**
- Bug 2 (layout Shipment Qty + thứ tự field cho ESTEC trên template `IBL526` — template dùng chung nhiều khách hàng khác, cần scope đúng theo Customer trước khi sửa).
- Bug 1 riêng cho IBL040: 1 dòng Plan có thể có nhiều đợt tạo barcode cho nhiều "OTHER CUSTOMER" khác nhau theo thời gian (do `ITEM_BC` không unique) — Print All lấy tất cả, kể cả nhãn cũ của khách khác — cần quyết định hướng xử lý dữ liệu.
- Bước 8 (tách proc IBL020/IBL040) — SQL đã viết, chưa chạy.
