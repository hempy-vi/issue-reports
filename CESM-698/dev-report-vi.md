# Dev Report — CESM-698

**Task:** [SAMIL] [BUG] POP Shows Inconsistent Information Between Inner and Outer Screens (Order 202606-0420SAWMT)

---

## 2026-09-09

**1. Xác định 2 màn hình trong ticket**

Từ 3 screenshot đính kèm ticket, xác định đây là 2 form Windows desktop (C#/WinForms) thuộc module **QC_DYE_V2_MAIN** (`D:\WINDOWS APP PROJECT\windows-app\SAMIL\QC_DYE_V2_MAIN\ScanSystem\ScanSystem`):
- `frmQC0012.cs` — màn nhập liệu thẻ công đoạn (nhận vải mộc từ xưởng dệt), hiển thị `265 SQM`.
- `frmGC0011.cs` — màn kiểm vải theo cuộn, hiển thị `250 SQM`.

Loại trừ module `QC_PRINT_V2` (dùng SP tiền tố khác `sp_sel_pop_p_qc0012_*`) và `Processing_cardV2`/`ITEM_LABEL` (bản cũ không `_v02`, `ITEM_LABEL` đã xác nhận là legacy không dùng nữa).

Đọc source 2 form:
- `frmQC0012.cs:73` — `lbSQM.Text` lấy cột `sqm` từ procedure **`SP_SEL_POP_QC0012_PC_V02`** (gọi theo PK thẻ công đoạn `prod_card_pk`).
- `frmQC0012.cs:206-298` (nút Save, `sp_upd_pop_qc0012_v02`) — xác nhận `sqm`/`weight` KHÔNG nằm trong payload lưu, đây là field chỉ hiển thị (read-only).
- `frmGC0011.cs:189` — `lbSqm.Text` lấy cột `sqm` từ **`SP_SEL_POP_QC0011_V02`** (gọi theo PK record QC gắn với cuộn vải cụ thể).
- `frmGC0011.cs:1552-1581` — xác nhận `lbSqm` (250) được dùng làm ngưỡng validate ±15% cho 3 số đo G/M thực tế nhập tay — theo thiết kế `lbSqm` PHẢI là G/M2 yêu cầu của order (spec), không phải số đo riêng của cuộn.

Đối chiếu screenshot màn order gốc GASP (`sa1000031.aspx`) — required weight của order = `250` G/m² — khớp `frmGC0011` (`250`), lệch với `frmQC0012` (`265`).

**2. Sự cố kết nối DB — VPN down, chuyển sang REST API sau khi VPN được nối lại**

MCP `oracle-samil` timeout khi khởi động session. Kiểm tra:
```
docker logs oracle-api-samil --tail 40
→ ORA-12170: Cannot connect. TCP connect timeout of 20000ms for host 192.168.240.204 port 1521
```
Test trực tiếp từ host (`Test-NetConnection 192.168.240.204 -Port 1521`, `/dev/tcp`) đều fail — xác nhận đây là vấn đề VPN/network tới mạng nội bộ SAMIL, không phải lỗi docker container. Báo user kiểm tra lại VPN.

Sau khi user kết nối lại VPN: `docker restart oracle-api-samil` → healthy, `Test-NetConnection` OK. MCP `oracle-samil` vẫn không tự reconnect trong session — chuyển sang gọi thẳng REST API của `oracle-api-samil` (`http://127.0.0.1:8084`, xem `platform/oracle-api/README.md`) bằng `curl`.

Phát hiện phụ: endpoint `/api/db/objects` và `/api/db/sources/{name}` của `oracle-api` có bug sẵn có — thiếu param `owner`/`type` sẽ crash `ORA-17004: Invalid column type` (JDBC không suy được kiểu cho `setNull` khi param optional = null) → phải luôn truyền đủ `?owner=SAMILV2&type=PROCEDURE`. Dùng `POST /api/db/query/read-only` (SELECT trực tiếp vào `all_objects`/`all_source`) thay cho endpoint `/api/db/objects` bị lỗi. (Bug thuộc `platform/oracle-api`, ngoài phạm vi ticket, chỉ ghi nhận không sửa.)

**3. Root cause — xác nhận bằng PL/SQL thật + data thật**

Lấy full source PL/SQL của cả 2 procedure qua `GET /api/db/sources/{name}?owner=SAMILV2&type=PROCEDURE`. Cả 2 dùng chung 1 công thức GSM:
```sql
round(DECODE(unit,'MTS',0.9144,1)
  * NVL((<unit_gravity> / SUBSTR(REPLACE(<a_demission>,'"'),-2) / 0.02323), 0), 2)
```
nhưng lấy `<unit_gravity>` từ 2 nguồn khác nhau:
- `SP_SEL_POP_QC0011_V02` (đúng): `C.A_UNIT_GRAVITY` — bảng `sa_order_production` (alias c) — giá trị spec **hiện tại/live** của order.
- `SP_SEL_POP_QC0012_PC_V02` (sai): `a.unit_gravity` — bảng `sa_processing_card` (alias a) — cột VARCHAR2 lưu bản copy 1 lần duy nhất tại thời điểm tạo thẻ công đoạn, **không bao giờ được refresh**.

Order `202606-0420SAWMT` có ghi chú revision "09/07 REV 기존 70/72" 250GSM 418GYD >>>66/68" 250GSM 395GYD" — spec bị sửa lại ngày 07/09 từ weight 418 G/YD → 395 G/YD (cùng target 250GSM). Thẻ công đoạn tạo trước lần sửa đó vẫn kẹt giá trị `unit_gravity='418'` cũ.

Verify bằng data thật (join `sa_processing_card`/`sa_order_production` cho PO `202606-0420SAWMT`):
```sql
SELECT a.pk card_pk, a.prod_card_no, a.lot, a.unit_gravity card_unit_gravity,
       b.pk order_pk, b.po_no, b.a_unit_gravity order_unit_gravity, b.a_demission, b.unit,
       round(DECODE(b.unit,'MTS',0.9144,1) * NVL((a.unit_gravity / SUBSTR(REPLACE(b.a_demission,'"'),-2) / 0.02323),0),2) sqm_using_card_gravity,
       round(DECODE(b.unit,'MTS',0.9144,1) * NVL((b.a_unit_gravity / SUBSTR(REPLACE(b.a_demission,'"'),-2) / 0.02323),0),2) sqm_using_order_gravity
FROM sa_processing_card a, sa_order_production b
WHERE a.sa_order_production_pk = b.pk AND a.del_if=0 AND b.del_if=0 AND b.po_no='202606-0420SAWMT'
```
Kết quả (20 thẻ công đoạn): LOT 001 (`260905-020`, thẻ dùng trong screenshot) có `card_unit_gravity='418'` (stale) vs `order_unit_gravity=395` (live) → `sqm_using_card_gravity = 264.62` (làm tròn hiển thị **265**, khớp y hệt screenshot 2) vs `sqm_using_order_gravity = 250.06` (hiển thị **250**, khớp screenshot 3). 17/20 thẻ công đoạn của order này bị stale tương tự, chỉ LOT 002-004 đã có giá trị đúng.

**4. Audit mở rộng — kiểm tra cùng lỗi trên các procedure khác**

Tìm tất cả procedure trong schema SAMILV2 dùng chung hằng số `0.02323` (dấu hiệu công thức GSM này):
```sql
SELECT owner, name, type, COUNT(*) hit_lines FROM all_source
WHERE UPPER(text) LIKE '%0.02323%' GROUP BY owner, name, type ORDER BY owner, name
```
→ 35 procedure. Lọc còn 10 procedure thuộc đúng họ POP-QC (loại `SP_RPT_*`/`SP_SEL_SA*` — report/sales, khác domain) để audit từng cái: đọc full PL/SQL + grep client code xem screen nào gọi + đánh giá dùng snapshot có phải bug hay intentional (vd màn reprint tag phải giữ giá trị lịch sử, không phải bug).

Kết quả:
- **Có bug, cùng pattern** — `SP_SEL_POP_QC0012_PC` (bản cũ, không `_V02`): dùng bởi `frmQC0012.cs` trong module **`Processing_cardV2`** (build artifact 2026-05-29 → đang active deploy thật). 2 chỗ dùng `a.unit_gravity` y hệt (dòng ~66, ~130).
- **Không phải bug** — `SP_SEL_POP_QC0011` (đã đúng, chỉ dùng `sa_processing_card` để join, không lấy giá trị từ đó); toàn bộ họ `SP_SEL_POP_P_QC00xx` của module `QC_PRINT_V2` (dùng bảng khác `sa_order_production_p`/`sa_prod_card_printting`, bảng card của họ này không có cột unit_gravity riêng nên về kiến trúc không thể dính bug dạng này); `SP_SEL_POP_QC0018N_D`/`SP_SEL_POP_QC0018_D` (sqm tính từ số đo tay thật của roll, không liên quan unit_gravity).
- **Có cùng lỗi logic nhưng dead code** — `SP_SEL_POP_QC0012_PC_V02_TEST`: WHERE clause hard-code 2 `prod_card_no` cụ thể, bỏ qua tham số đầu vào; grep toàn bộ 2 client app (GASP + Windows POP, ~11K file) ra 0 nơi gọi → orphaned/scratch procedure của dev, không ảnh hưởng người dùng thật.
- Phát hiện phụ không liên quan: `SP_SEL_POP_P_QC0012_PO` thiếu join predicate giữa `sa_order_production_p` và `sa_order_prod_color_p` (cross-join tiềm ẩn) — nhưng procedure này cũng là dead code (0 caller), dormant, ghi nhận riêng ngoài phạm vi ticket.

**5. Fix**

Soạn 2 script `CREATE OR REPLACE PROCEDURE` (sinh tự động bằng cách thay đúng 2 chỗ `a.unit_gravity` → `b.a_unit_gravity` trên source gốc fetch từ `ALL_SOURCE`, đã verify chỉ đúng 2 match/file, không đụng chỗ nào khác):
- `SP_SEL_POP_QC0012_PC_V02` (module QC_DYE_V2_MAIN — đúng màn hình trong ticket)
- `SP_SEL_POP_QC0012_PC` (module Processing_cardV2 — phát hiện thêm qua audit)

Kèm 2 script rollback tương ứng (= nguyên bản source cũ). Không cần sửa data cũ (`sa_processing_card.unit_gravity` bị stale) — fix chỉ sửa công thức đọc, sau khi deploy các thẻ công đoạn cũ tự động hiển thị đúng theo `sa_order_production.a_unit_gravity` hiện tại.

Sau khi 2 script được áp dụng vào DB, user xác nhận đã hiểu và đóng ticket.
