# Dev Report — CESM-785

**Task:** Fix Roll Location/Date Mismatch Between Warehouses HOQC301 and HDF102

---

## 2026-09-10

**1. Tiếp nhận yêu cầu**
Roll barcode `D25091920979` (PO 202508-0343HSAEROW, Lot 005, Roll 030R, ITEM_PK 980529) đã được chuyển kho từ HOQC301 sang HDF102, nhưng dữ liệu bị lệch: vào kho HOQC301 vẫn thấy roll nhưng làm DR (Defect WH Outsource) bị báo hết tồn kho; vào kho HDF102 (kho đích) thì không có roll để chọn. User yêu cầu cập lại ngày product income = 20/09/2025 và kho thực tế = 2914/HDF102.

**2. Xác định nguyên nhân lỗi tồn kho = 0**
Đọc source `SAMILV2.LG_PRO_FPIN00020_CONFIRM` (procedure confirm DR), phát hiện tồn kho được tính bởi hàm `LG_CHECK_STOCK` — SUM(INPUT_QTY) - SUM(OUTPUT_QTY) trên bảng ledger `TLG_IN_STOCKTR`, lọc theo WH/item/lot/po/roll/barcode.

Truy vấn ledger cho barcode này:
```sql
SELECT PK, TR_DATE, TLG_IN_WAREHOUSE_PK, TLG_IN_WHLOC_PK, TLG_IT_ITEM_PK,
       LOT_NO, PO_NO, ROLL_ID, BARCODE, INPUT_QTY, OUTPUT_QTY,
       TRIN_TYPE, TROUT_TYPE, TABLE_NAME, TABLE_PK, DEL_IF, CRT_DT, CRT_BY
FROM TLG_IN_STOCKTR
WHERE BARCODE = 'D25091920979'
ORDER BY CRT_DT;
```
Kết quả cho thấy:
- 2 chứng từ Product Income tạo ngày 20/09/2025 (slip HOQC301-2509-06480, HOQC301-2509-06688) đã bị hủy (DEL_IF = chính PK của dòng).
- Transfer HOQC301→HDF102 chạy hợp lệ lúc 20/09/2025 10:14 (OUT tại HOQC301, IN tại HDF102, đều 23.7).
- Một chứng từ Product Income thay thế (PK 5550351/83956907, slip HOQC301-2509-07071) được tạo NGÀY 21/09/2025 — sau thời điểm transfer.

=> Tồn kho active tại HOQC301 = IN 23.7 (income #212204200) − OUT 23.7 (transfer) = **0** — khớp đúng với lỗi `ORA-20999: Out Quantity is greater than stock quantity`. Đây là kết quả TÍNH ĐÚNG theo ledger, không phải bug tính toán.

**3. Xác định nguyên nhân roll không xuất hiện ở HDF102**
Đọc source `SAMILV2.LG_SEL_BIPO00010` (procedure của màn hình IV0701 Item Barcode Checking) — xác nhận WH_ID/WH_NAME/Location hiển thị được lấy trực tiếp từ:
```sql
LG_GET_WAREHOUSE_ID(A.TLG_IN_WAREHOUSE_PK)  WH_ID,
LG_GET_WAREHOUSE_NAME(A.TLG_IN_WAREHOUSE_PK) WH_NAME,
(SELECT MAX(L.LOC_ID || ' - ' || L.LOC_NAME) FROM TLG_IN_WHLOC L WHERE L.PK = A.TLG_IN_WHLOC_PK) LOCATION_NM
FROM TLG_PA_PACKAGES A
```
tức là từ bảng `TLG_PA_PACKAGES` (roll master), KHÔNG tính lại từ ledger. Truy vấn bảng này:
```sql
SELECT PK, ITEM_BC, TLG_IN_WAREHOUSE_PK, TLG_IN_WHLOC_PK, PROD_DATE, FIRST_STOCK_DATE,
       TLG_IN_WAREHOUSE_PK_BK, TLG_IN_WHLOC_PK_BK
FROM TLG_PA_PACKAGES WHERE ITEM_BC = 'D25091920979';
```
=> `TLG_IN_WAREHOUSE_PK`/`TLG_IN_WHLOC_PK` vẫn = 1753/7646 (HOQC301) — chưa được cập nhật khi transfer chạy, dù ledger đã ghi nhận roll đã ở HDF102 (2914/12628). Đây chính là nguyên nhân roll "vẫn thấy" ở HOQC301 (tồn=0) và "không có để chọn" ở HDF102 (dù tồn đủ 23.7).

Xác nhận mã kho/vị trí qua bảng master:
```sql
SELECT PK, WH_ID, WH_NAME FROM TLG_IN_WAREHOUSE WHERE PK IN (1753, 2914);
-- 1753 = HOQC301, 2914 = HDF102
SELECT PK, LOC_ID, LOC_NAME, TLG_IN_WH_PK FROM TLG_IN_WHLOC WHERE PK IN (7646, 12628);
-- 7646 = HOQC301, 12628 = HDF102_000 (HDF102 Common)
```

Đồng thời phát hiện field `TLG_PA_PACKAGES.PROD_DATE` luôn NULL toàn hệ thống (0/200.000 dòng có giá trị) — không phải field "ngày product income" đang dùng. Field đang dùng thật là `TLG_PR_PROD_INCOME_M.PROD_DATE`/`INPUT_DATE` của chứng từ income active (PK 5550351), hiện = 21/09/2025.

**4. Kiểm tra an toàn trước khi sửa**
```sql
SELECT PK, TR_DATE, CLOSE_YN, PR_ST_CLOSE_M_PK, FROM_CLOSE_PK
FROM TLG_IN_STOCKTR WHERE PK IN (212204200, 212166764, 212166765);
```
Cả 2 dòng ledger sẽ bị sửa đều có `CLOSE_YN='N'`, không gắn kỳ đóng sổ nào (`FROM_CLOSE_PK` NULL) → an toàn để sửa ngày.

**5. Soạn script fix**
```sql
-- 1) Product income date: 20250921 -> 20250920
UPDATE SAMILV2.TLG_PR_PROD_INCOME_M
   SET PROD_DATE = '20250920', INPUT_DATE = '20250920',
       MOD_DT = SYSDATE, MOD_BY = 'DATA_FIX_CESM785'
 WHERE PK = 5550351 AND PROD_DATE = '20250921';

UPDATE SAMILV2.TLG_IN_STOCKTR
   SET TR_DATE = '20250920', MOD_DT = SYSDATE, MOD_BY = 'DATA_FIX_CESM785'
 WHERE PK = 212204200 AND TABLE_NAME = 'TLG_PR_PROD_INCOME_D'
   AND TABLE_PK = 83956907 AND TR_DATE = '20250921';

-- 2) Kho thực tế: HOQC301 (1753/7646) -> HDF102 (2914/12628)
UPDATE SAMILV2.TLG_PA_PACKAGES
   SET TLG_IN_WAREHOUSE_PK_BK = TLG_IN_WAREHOUSE_PK,
       TLG_IN_WHLOC_PK_BK     = TLG_IN_WHLOC_PK,
       TLG_IN_WAREHOUSE_PK    = 2914,
       TLG_IN_WHLOC_PK        = 12628,
       MOD_DT = SYSDATE, MOD_BY = 'DATA_FIX_CESM785'
 WHERE ITEM_BC = 'D25091920979'
   AND TLG_IN_WAREHOUSE_PK = 1753 AND TLG_IN_WHLOC_PK = 7646;

COMMIT;
```
Backup giá trị kho cũ được lưu vào cột `TLG_IN_WAREHOUSE_PK_BK`/`TLG_IN_WHLOC_PK_BK` — đúng convention có sẵn trên bảng (comment gốc: "PQH USE IT FOR BACKUP WHEN UPDATE DATA"), kèm câu lệnh ROLLBACK tương ứng nếu cần lùi lại.

**6. Ghi chú vận hành**
Phiếu DR nháp cũ `DR-HOQC301-20260910-169187` (đang SAVED, chưa confirm) được tạo chọn kho HOQC301 do lúc đó bị hiển thị nhầm — sau khi fix, roll sẽ không còn xuất hiện ở HOQC301 nữa, cần hủy phiếu nháp cũ và tạo lại phiếu DR mới chọn kho HDF102.
