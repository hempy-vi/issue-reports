# Dev Report — CESM-895

**Task:** Clarify How Form SS2.5.1 Data Is Populated (User-Loaded vs Auto-Updated) with Examples

---

## 2026-09-14

**1. Xác định bảng backend đứng sau form SS.2.5.1 (Margin Table After Cost)**

Tìm các bảng liên quan 'margin' trong schema SAMILV2, phát hiện có cả `SA_MARGIN_TABLE` và `SA_MARGIN_TABLE_V2` (cấu trúc cột gần như giống hệt nhau). Query order mẫu `202608-0025HNW036` trên cả 2 bảng, xác nhận data chỉ tồn tại ở `SA_MARGIN_TABLE` (PK=202717), không có ở `SA_MARGIN_TABLE_V2`.

```sql
SELECT 'SA_MARGIN_TABLE' SRC, PK, ORDER_NO, ITEM, CRT_BY, CRT_DT, MOD_BY, MOD_DT
FROM SA_MARGIN_TABLE WHERE ORDER_NO = '202608-0025HNW036'
UNION ALL
SELECT 'SA_MARGIN_TABLE_V2', PK, ORDER_NO, ITEM, CRT_BY, CRT_DT, MOD_BY, MOD_DT
FROM SA_MARGIN_TABLE_V2 WHERE ORDER_NO = '202608-0025HNW036';
```

**2. Tra CRT_BY/MOD_BY cho order mẫu + nhiều order khác, map ra tên thật**

Kết quả: `CRT_BY = 'thanh'`, `MOD_BY = 'thanh'`, cách nhau 5 giây (đúng hành vi bấm Save 1 lần trên form). Join thêm bảng user `TCO_BSUSER_XXX` (USER_ID/USER_NAME) để map login ra tên thật: `thanh` = NGUYỄN TRẦN THANH THANH.

Group by `CRT_BY` trên toàn bảng `SA_MARGIN_TABLE`: phần lớn là login user thật (jenny, hien, hnsthuan, sxoan, dony0905...), riêng `CRT_BY = 'convert'` có 13,880 dòng — là data migrate 1 lần từ hệ thống cũ (không phải batch job định kỳ).

**3. Loại trừ khả năng tự động cập nhật ngầm**

```sql
SELECT trigger_name FROM all_triggers WHERE table_name = 'SA_MARGIN_TABLE';  -- 0 dòng
SELECT job_name FROM all_scheduler_jobs WHERE UPPER(job_name) LIKE '%MARGIN%';  -- 0 dòng
```
Không có trigger, không có Oracle Scheduler job nào ghi vào `SA_MARGIN_TABLE`.

**4. Truy nguồn procedure INSERT/UPDATE thật sự chạy khi bấm Save**

```sql
SELECT DISTINCT owner, name FROM all_source WHERE UPPER(text) LIKE '%INSERT INTO SA_MARGIN_TABLE%';
```
Ra `SP_UPD_SA400190` (đang dùng), `SP_UPD_SA400190_V2`, và 2 procedure code cũ/chết (`SP_UPD_SA400050`, `STOMFRSTOMSO0016_U_02`, không còn form nào gọi tới, sửa lần cuối 2020). `SP_UPD_SA400190` nhận `P_CRT_BY` làm **tham số đầu vào** (không tự sinh), dùng để set `CRT_BY`/`MOD_BY` — xác nhận giá trị này do code phía web (session user đang login) truyền vào lúc gọi procedure, không phải logic ngầm tự chạy.

**5. Câu hỏi mở rộng: SS.2.5 và SS.2.5.1 có cùng nguồn dữ liệu không?**

User phản ánh: nhân viên khẳng định 100% chỉ dùng form SS.2.5 (`sa400190.aspx`), chưa từng đụng SS.2.5.1 (`sa400190_v2.aspx`), nhưng lại thấy chênh lệch số 'Final quantity' giữa 2 form cho cùng order (10,759 vs 8,069.3).

Đọc dso trong 2 file `.aspx` (đã xác nhận URL thật qua screenshot browser):
```
sa400190.aspx     -> procedure="sp_upd_sa400190"     -> ghi SA_MARGIN_TABLE
sa400190_v2.aspx  -> procedure="sp_upd_sa400190_v2"  -> theo code phải ghi SA_MARGIN_TABLE_V2
```
Verify thực tế bằng 2 order khác (đang mở live trên máy user) → cả 2 đều thực sự ghi vào `SA_MARGIN_TABLE` (bảng cũ), không phải `_V2` — nghĩa là **SS.2.5 và SS.2.5.1 đọc/ghi chung 1 dòng dữ liệu** (cùng PK, cùng `SA_SALE_ORDER_PK`). Do đó `CRT_BY`/`MOD_BY` phản ánh đúng người dùng SS.2.5, dù họ chưa từng mở SS.2.5.1 — SS.2.5.1 chỉ là màn hình hiển thị/tính thêm trên cùng dữ liệu đó.

**6. Điều tra nguyên nhân chênh lệch số 'Final quantity' (10,759 vs 8,069.3)**

So sánh procedure SELECT của 2 form:
```sql
-- SP_SEL_SA400190 (SS.2.5): đọc thẳng cột đã lưu
A.PACKING_QTY   -- = 10,759 (khớp DB)

-- SP_SEL_SA400190_V2 (SS.2.5.1), qua view SA_MG_TABLE:
DECODE(
    ORDER_TYPE
  , 1, DECODE(UNIT_PRICE, 'KG', QC_WEIGHT_OK, QC_LENGTH_OK)
  , DECODE(UNIT_PRICE_P, 'KG', G.FG_WEIGHT, G.FG_LENGHT)
) AS PACKING_QTY
```
Order này có `ORDER_TYPE = '1'`, `UNIT = 'KG'` (query xác nhận trên `SA_SALE_ORDER`) → rơi vào nhánh lấy `QC_WEIGHT_OK` từ bảng `TABLE_MG_2` (`DATA_TYPE = 'QC'`) = **8,069.3** — khớp chính xác với kết quả user tự EXEC trực tiếp procedure để đối chiếu:
```sql
exec sp_sel_sa400190('202717', :p_rtn_value);      -- PACKING_QTY = 10759
exec sp_sel_sa400190_v2('202717', :p_rtn_value);   -- PACKING_QTY = 8069.3
```

**7. Truy `QC_WEIGHT_OK` về tận nguồn gốc — dữ liệu QC thật, không phải nhập tay**

`TABLE_MG_2` được tổng hợp bởi procedure `JOB_INSERT_MARGIN_TABLE` (không tìm thấy job/scheduler/trigger/nút nào gọi nó tự động — có thể do DBA/IT chạy tay, chưa xác định được tần suất chính xác). Truy ngược chuỗi join tới bảng chi tiết từng cuộn vải QC:
```
SA_SALE_ORDER -> SA_PROCESSING_ORDER -> SA_ORDER_PRODUCTION ->
SA_ORDER_PROD_COLOR -> SA_PROCESSING_CARD -> SA_QC_M -> SA_QC_D
```
Query trực tiếp cho order (Sale Order PK=152315):
```sql
SELECT qd.ROLL_DECISION, COUNT(*) ROLL_CNT, SUM(qd.ROLL_WEIGHT) TOTAL_WEIGHT
FROM ... (chuỗi join trên) WHERE e.PK = 152315 AND qd.DEL_IF = 0
GROUP BY qd.ROLL_DECISION;
```
Kết quả: 431 cuộn decision='Q' (đạt) tổng cân = **8,069.3 kg** — khớp tuyệt đối. Người nhập (`CRT_BY` trên `SA_QC_D`) là QC inspector **DUONG THI NHU QUYNH**, các dòng tạo từ 2026-09-03 10:40 đến 11:57 (mỗi cuộn cách nhau ~2-3 phút, đúng nhịp cân/quét thủ công), hoàn toàn độc lập với người làm margin costing (`thanh`, làm ngày 2026-08-03).

**8. Giải thích thêm chênh lệch với report QC khác (SD.9.3.1 Total View New Table / SD.8.4_V6 F.G Inquiry-Package)**

User đối chiếu thêm 1 report QC khác, thấy số `QC Weight = 8,645.1` / `Good Weight = 8,542.4` — không khớp 8,069.3. Xác nhận `TABLE_MG_2` có 2 dòng riêng biệt theo `DATA_TYPE`: `'QC'` (kiểm ngay sau sản xuất, ra 8,069.3/8,191.4) và `'OQC'` (Outgoing QC — kiểm lại trước khi đóng gói/xuất hàng, ra 8,542.4/8,645.1), tính từ 2 nguồn/công thức khác nhau trong `JOB_INSERT_MARGIN_TABLE`. Report `SD.8.4_V6 F.G Inquiry-Package` (chữ 'Package' = giai đoạn đóng gói) hiển thị đúng loại `OQC`, còn SS.2.5.1 (order này) lấy loại `QC` — 2 report đang cho xem 2 mốc kiểm QC khác nhau của cùng 1 lô hàng, không phải sai lệch dữ liệu.

## 2026-09-15

**1. Xác định form và tìm đúng công thức Profit 2**

User hỏi tiếp: form **SS.2.5.2 "Margin Table Packages"** (`sa400190_v3.aspx`, cùng họ với SS.2.5/SS.2.5.1) hiện dòng "Profit 2" ra **-9,239.61** (~-73%) cho order `202607-0457WHTX`, trong khi "Profit" (dòng trên) vẫn dương bình thường (+3,423.04, +27%). Đọc dso: `function="sp_sel_sa400190_v3"`, đọc qua view `SA_MG_TABLE`. Đọc kỹ toàn bộ khối DECODE (rút kinh nghiệm lần trước từng nhầm dead code) để lấy đúng công thức sống:

```sql
PROFIT_AMT_USD2 = ROUND(
    NVL(NEGO_AMT,0)
  - (NVL(COMMISSION_AMT_1,0)+NVL(COMMISSION_AMT_2,0)+NVL(COMMISSION_AMT_3,0)
   + NVL(PAYMENT_CONDITION_AMT_1,0)+...+NVL(PAYMENT_CONDITION_AMT_6,0)
   + NVL(FINISH1_AMT,0)+NVL(FINISH2_AMT,0)+NVL(FINISH3_AMT,0)
   + NVL(FINISH5_AMT,0)+NVL(FINISH6_AMT,0)   -- FINISH4_AMT bi comment, khong tinh
   + NVL(DYEING_AMT,0)+NVL(KNITTING_AMT,0)
   + NVL(PREDYE_AMT,0)+NVL(PREDYE_AMT2,0)
   + NVL(YARN_TOTAL_AMT,0)+NVL(CHARGE_AMT,0))
  - NVL(CLAIM_AMT,0)
, 2)
```
`NEGO_AMT` từ tab "Nego Information" (`SA_IE_NEGO_D`), `CHARGE_AMT` từ tab "Export Charge" (`TSA_EXP_CHGR_ITEM_DTL`), `CLAIM_AMT` từ tab "Claim Note" (`SA_CLAIM_NOTE_ENTRY_D`, chỉ tính claim đã confirm `STATUS='Y'`).

**2. Query view trực tiếp cho order, phát hiện NEGO_AMT = NULL**

```sql
SELECT TOTAL_COST_USD, PROFIT_AMT_USD2, NEGO_AMT, CHARGE_AMT, CLAIM_AMT
FROM SA_MG_TABLE WHERE SA_SALE_ORDER_PK = 151779;
```
→ `NEGO_AMT`/`CHARGE_AMT`/`CLAIM_AMT` đều NULL, `PROFIT_AMT_USD2` đúng bằng `-TOTAL_COST_USD` — xác nhận NEGO_AMT bị tính = 0.

**3. Truy nguyên nhân NEGO_AMT=NULL dù data Nego có thật**

Query trực tiếp `SA_IE_NEGO_D` theo `SA_SALE_ORDER_PK=151779` → có 1 dòng thật, **$6,429.78**. Đọc kỹ join trong view: `NEGO_AMT` link về order qua so khớp TEXT `Z.ORDER_NO||Z.SUB_NO = SA_NEGO_SPLIT_QTY.PO_NO` (không phải qua PK trực tiếp):
```sql
SELECT so.PO_NO, x.PO_NO AS SPLIT_PO_NO
FROM SA_SALE_ORDER so
JOIN SA_IE_NEGO_D n ON n.SA_SALE_ORDER_PK = so.PK
JOIN SA_NEGO_SPLIT_QTY x ON x.PK = n.SA_NEGO_SPLIT_QTY_PK
WHERE so.PK = 151779;
```
→ `SPLIT_PO_NO = '202409-0442HDELTA'` (Order No/Sub No của 1 order KHÁC hoàn toàn, không phải PO No thật `BD19579` của order này) — sai định dạng field, khiến view không link được Nego về đúng order.

**4. Xác minh giá trị đúng + kiểm tra an toàn trước khi sửa**

Sample 20 dòng `SA_NEGO_SPLIT_QTY.PO_NO` khác trong bảng → xác nhận đúng quy ước phải là `ORDER_NO+SUB_NO` (vd `202608-0373V037V`), nên giá trị đúng là `'202607-0457WHTX'` (không phải PO No thật `BD19579` như suy đoán ban đầu). Kiểm tra record `SA_NEGO_SPLIT_QTY.PK=221375` chỉ có đúng 1 dòng Nego tham chiếu tới, và order đang "vô tình" nhận nhầm (`202409-0442HDELTA`, PK=114831) có Nego riêng của nó (đang lỗi tương tự, trỏ sang order thứ 3) → sửa không ảnh hưởng dữ liệu order khác.

**5. Chạy fix và verify**

```sql
UPDATE SA_NEGO_SPLIT_QTY
SET PO_NO = '202607-0457WHTX'
WHERE PK = 221375 AND DEL_IF = 0 AND PO_NO = '202409-0442HDELTA';
COMMIT;
```
Verify lại view sau khi chạy: `NEGO_AMT = 6,429.78` (đã lên đúng), `PROFIT_AMT_USD2 = -2,158.7` (= 6,429.78 − 8,588.48 Total Cost tại thời điểm đó) — công thức chạy đúng với data thật, hết còn "âm giả" do thiếu data.

**6. Quét diện rộng phát hiện lỗi hệ thống lặp lại (báo riêng, chưa xử lý)**

```sql
SELECT COUNT(*) TOTAL_LINKED,
  SUM(CASE WHEN UPPER(x.PO_NO)=UPPER(so.ORDER_NO||so.SUB_NO) THEN 1 ELSE 0 END) MATCHED,
  SUM(CASE WHEN x.PO_NO IS NULL THEN 1 ELSE 0 END) PO_NO_NULL,
  SUM(CASE WHEN x.PO_NO IS NOT NULL AND UPPER(x.PO_NO)!=UPPER(so.ORDER_NO||so.SUB_NO) THEN 1 ELSE 0 END) MISMATCHED
FROM SA_IE_NEGO_D n
JOIN SA_SALE_ORDER so ON so.PK=n.SA_SALE_ORDER_PK
LEFT JOIN SA_NEGO_SPLIT_QTY x ON x.PK=n.SA_NEGO_SPLIT_QTY_PK AND x.DEL_IF=0
WHERE n.DEL_IF=0;
```
→ **23,865/234,036 dòng (~10%)** bị lệch PO_NO tương tự trên toàn hệ thống. Group theo `CRT_BY`/tháng cho thấy lỗi trải đều liên tục hơn 1 năm rưỡi (2023-05 → 2024-11+) bởi vài user cụ thể (ntrang, nnhung, nvan, nnphuong), không tập trung 1 đợt — nghi ngờ do 1 tính năng "copy từ order trước" ở màn hình nhập Nego Split Qty không cập nhật lại field tham chiếu. Chưa xác định chính xác cơ chế/màn hình gây lỗi (cần điều tra riêng) — báo lại cho user/IT quyết định, KHÔNG xử lý trong phạm vi fix lần này.
