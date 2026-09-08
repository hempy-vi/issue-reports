# Dev Report — CESM-418

**Task:** [DONGIL][Requirement] Enforce 100% USOT3137 Allocation Rule for All Finished Goods Lots in Monthly Lot Tracking (August 2026)

---

## 2026-09-08

**1. Xác định đối tượng cần sửa và dữ liệu tham chiếu**

Truy vết yêu cầu tới procedure `LG_PRO_DAILY_LOT_TRACKING_JOB` (schema DONGIL). Procedure không nhận tham số, luôn tự lấy tháng làm việc từ `SYSDATE`:

```sql
L_MONTH VARCHAR2(6):=to_char(sysdate, 'yyyymm');
...
L_PROCESS_TYPE VARCHAR2(10):='DD';/*==MM: Monthly closing; DD: Daily Closing=*/
```

`L_PROCESS_TYPE` bị hardcode `'DD'` và không bao giờ được gán lại — nhánh `'MM'` (dòng ~108-282) là dead code đối với DONGIL, xác nhận qua chính comment của dev gốc tại dòng 108: `--ko chay, cua dongil, can sua lai`.

Các bảng liên quan:
- `TLG_LOT_TRACKING_MAT` — chi tiết phân bổ nguyên liệu theo từng lot; đây chính là dữ liệu hiển thị ở tab "Material Detail" của màn PM0407.
- `TLG_LOT_TRACKING_TEMP` — bảng tạm nội bộ, chỉ dùng trong phạm vi 1 lần chạy job.
- `TLG_WI_LINE_OP_CONS` — tỷ lệ pha trộn nguyên liệu cố định của Work Instruction (master data, job này không đụng vào).

Mã nguyên liệu đã tra được:
- `USOT3137` = `TLG_IT_ITEM.PK 122`
- `BRAMID` = `PK 121`
- Cả 2 (+ AUS, BRA0001, BRAEAGLE, BRAM 36, USOT3136, USPIMA3346 — tổng 8 mã) đều thuộc `TLG_IT_ITEMGRP_PK = 27` (nhóm nguyên liệu bông thô).

**2. Xác nhận cơ chế mang số dư qua tháng bằng dữ liệu thật**

Dòng 284 xoá mềm toàn bộ dòng `TLG_LOT_TRACKING_MAT` của tháng hiện tại mỗi lần chạy, rồi build lại từ đầu (khớp với "tự tính lại toàn bộ tháng hiện tại mỗi đêm").

Dòng 296-355: 1 câu `WITH` (`TBL_IN_DATA`/`TBL_OUT_DATA`) tính số dư cuối kỳ (`QTY_END_MAT`) của tháng trước theo (mã nguyên liệu, lot nguyên liệu, lot pha trộn, dòng WI), rồi insert 1 dòng `'OPEN'` mới cho tháng hiện tại — **giữ nguyên `TLG_IT_ITEM_PK` gốc**:

```sql
SELECT L_TIN_STOCKTR_PK, CUR.SLIP_NO, 'I60', CUR.QTY_END_MAT, ...
CUR.TLG_IT_ITEM_PK, CUR.LOT_MAT, ... L_MONTH, 'OPEN'
```

Đã kiểm chứng với `MIX_LOT='MIXEDF1-260627-02'`, mã BRAMID (121): số dư OPEN giữ nguyên không đổi qua STD_YM 202607 (256,083 kg), 202608 (206,538 kg), 202609 (24,825 kg) — mang sang y nguyên, mã không bao giờ đổi.

**3. Xác nhận cơ chế phân bổ nguyên liệu ("ration")**

Dòng 613-655 (comment: "CAL RATION ITEM OF MIX_LOT"): cursor `L_CUR` tính `NEED_QTY` cho từng mã nguyên liệu theo tỷ lệ `RATE` lấy từ `TLG_WI_LINE_OP_CONS.CONS_QTY` — tỷ lệ pha trộn gốc của WI (vd 60% BRAMID / 40% USOT3137), cố định từ lúc lập WI, job này không ghi vào đó. Cursor `STOCK_OVER` (631-637) rút tồn khớp từ `TLG_LOT_TRACKING_TEMP` và insert dòng phân bổ (`STOCK_TYPE='MAPPING'`, `TROUT_TYPE='O60'`) vào lot thành phẩm.

1 nhánh phụ (dòng 791-862, chạy khi tồn còn lại của lot pha trộn nhỏ hơn nhu cầu của lot thành phẩm) rút thẳng từ `TEMP` không qua bước chia theo RATE — nhưng vì `TEMP` chỉ tổng hợp lại đúng những gì thật sự có trong `TLG_LOT_TRACKING_MAT` (đã được chuẩn hoá sẵn một khi fix ở bước mang số dư qua tháng có hiệu lực), nhánh này không cần sửa riêng.

**4. Soạn bản patch tối giản — 2 điểm sửa (v1)**

1. Khai báo 4 biến local mới + 1 `SELECT INTO` tra `PK`/`TLG_IT_ITEMGRP_PK`/`ITEM_CODE`/`ITEM_NAME` của `USOT3137` ngay đầu procedure.
2. Trong `TBL_IN_DATA`/`TBL_OUT_DATA` (CTE mang số dư qua tháng, dòng ~297-312): bọc cột mã/tên nguyên liệu bằng `CASE WHEN <cùng nhóm nguyên liệu, khác PK> THEN <USOT3137> ELSE <giá trị gốc>` — mọi thứ phía sau (JOIN, GROUP BY, câu INSERT cuối ở dòng 347-355) không cần đụng tới vì đã tự nhận giá trị đã chuẩn hoá qua alias có sẵn.
3. Trong cursor `L_CUR` (phân bổ, dòng 616-624): thêm 1 lớp `CASE WHEN` tương đương để gộp phần RATE của mọi mã bông khác vào USOT3137 trước khi tính `NEED_QTY` — nếu không làm vậy, phần RATE vốn gán cho BRAMID sẽ đi tìm tồn BRAMID trong `TEMP` (giờ đã rỗng sau khi fix bước mang số dư qua tháng) và âm thầm phân bổ thiếu.

Không đổi bất kỳ số lượng nào (`INPUT_QTY`/`OUTPUT_QTY`/`NEED_QTY`) ở đâu cả — chỉ đổi nhãn mã nguyên liệu. `LOT_NO` cố tình giữ nguyên không đổi.

**5. Chạy 1 workflow kiểm chứng độc lập — phát hiện 2 lỗ hổng chặn ở bản v1**

Khởi chạy 1 agent tái suy luận độc lập (không thấy bản patch đã soạn) cộng thêm 2 reviewer phản biện (đúng/an toàn kỹ thuật, và rủi ro nghiệp vụ/kế toán).

Phát hiện:
- **Lỗ hổng chặn**: bản v1 bỏ sót hoàn toàn 1 đường ghi thứ 3 vào `TLG_LOT_TRACKING_MAT` — khối "chuyển kho vào kho có `PROCESS_TYPE='MIXED'`" (dòng 386-414), insert `TLG_IT_ITEM_PK = D.TR_ITEM_PK` (đúng mã nguyên liệu thật được chuyển) mà không chuẩn hoá gì cả. Kiểm chứng bằng dữ liệu thật: BRAMID đi qua đúng đường này 3,776 lần / 4,322,567 kg — nguồn phát sinh nguyên liệu không phải USOT3137 LỚN NHẤT, lớn hơn cả khối lượng mang qua tháng.
- Xác nhận 2 điểm sửa CTE mang số dư qua tháng và cursor phân bổ đã soạn đúng về mặt số học/JOIN, và đúng phạm vi chỉ áp dụng cho nhóm nguyên liệu 27.
- **Không chặn, ngoài phạm vi**: phát hiện 1 lỗi có sẵn trong hệ thống (không liên quan tới ticket này) — `OUTER JOIN` trong CTE mang số dư qua tháng so khớp `SLIP_NO`/`TR_DATE` của 1 dòng số dư mở đầu kỳ với 1 dòng sản xuất của lot thành phẩm, nhưng đây là 2 chuỗi đánh số hoàn toàn không liên quan nên không bao giờ khớp; hệ quả là `OUTPUT` không bao giờ thực sự trừ vào số dư mang qua (`QTY_END_MAT = SUM(IN) - 0`, luôn luôn). Xác nhận bằng dữ liệu: số dư OPEN của BRAMID trên `MIXEDF1-260627-02` giữ nguyên đúng 3,546.51871 kg suốt 3 tháng liên tiếp dù có ghi nhận tiêu thụ khớp mỗi tháng. Đã đánh dấu để báo cáo thành ticket riêng, không sửa ở đây.
- Xác nhận thêm 1 vướng mắc vận hành: vì job không nhận tham số và luôn tự lấy `L_MONTH` từ `SYSDATE`, nếu chạy vào ngày 2026-09-10 (sau khi `SYSDATE` đã sang tháng 9) sẽ tính lại cho tháng 9, KHÔNG đụng tới sổ sách tháng 8 đã đóng.

**6. Vá v1 → v2 (bổ sung điểm sửa thứ 3 còn thiếu)**

Thêm Edit 5: trong SELECT list của cursor chuyển kho (dòng 390) và mệnh đề `GROUP BY` tương ứng (dòng 411-413), áp dụng cùng công thức chuẩn hoá `CASE WHEN I.TLG_IT_ITEMGRP_PK = L_USOT_ITEMGRP_PK AND I.PK <> L_USOT_ITEM_PK THEN L_USOT_ITEM_PK ELSE D.TR_ITEM_PK END` (join tới `TLG_IT_ITEM I` đã có sẵn từ dòng 405, không cần thêm join mới).

**7. Theo yêu cầu user, ghép thành 1 bản `CREATE OR REPLACE PROCEDURE` hoàn chỉnh**

Lấy nguyên văn source của procedure qua `ALL_SOURCE` (`SELECT line, text FROM all_source WHERE name='LG_PRO_DAILY_LOT_TRACKING_JOB' AND type='PROCEDURE' ... ORDER BY line`, chia nhiều đợt phân trang để không vượt giới hạn số dòng của tool read-only), dựng lại toàn bộ nội dung vào 1 file local, rồi áp cả 5 điểm sửa bằng cách thay thế khớp chuỗi tuyệt đối (mỗi lần thay thế báo lỗi ngay nếu chuỗi "trước khi sửa" không khớp y nguyên — dùng như 1 lớp kiểm tra tự động không bị lệch do gõ lại thủ công). Mỗi điểm sửa được đánh dấu `-- [PATCH USOT3137]` ngay trong code để DBA dễ review. Output: `LG_PRO_DAILY_LOT_TRACKING_JOB_PATCHED.sql`.

**8. User báo lỗi compile (Toad for Oracle)**

```
PLS-00103: Encountered the symbol "end-of-file" when expecting one of the following:
( begin case declare end exception exit for goto if loop mod
null pragma raise return select update while with
```
tại dòng cuối cùng của file.

Chạy 1 công cụ kiểm tra cân bằng block PL/SQL độc lập (script PowerShell tự viết, theo dõi độ lồng `BEGIN`/`CASE`/`IF`/`LOOP`/`END`, có nhận diện string/comment) trên file đã patch — khớp y hệt hình dạng lồng nhau của bản gốc chưa sửa, nên 5 điểm sửa không phải nguyên nhân.

Để cô lập nguyên nhân, tạo thêm 1 **file đối chứng**: nội dung gốc chưa sửa gì, chỉ bọc thêm `CREATE OR REPLACE` / `/` (0 patch). File đối chứng CŨNG lỗi y hệt — chứng minh lỗi nằm ở bước trích xuất/dựng lại source, không phải ở 5 điểm sửa.

**Nguyên nhân gốc**: cấu trúc thật của procedure có 3 lớp — 1 `BEGIN` ngoài cùng (dòng gốc 3, ngay sau `IS`) bọc 1 block `DECLARE ... BEGIN ... END;` lồng bên trong (dòng gốc 17-880), tiếp theo là ~465 dòng code cũ đã comment hết hoàn toàn (dòng 881-1345, không có tác dụng), rồi tới đúng **1 dòng `END;` chưa comment ở dòng 1346** đóng lại `BEGIN` ngoài cùng đó. Bước trích xuất đã dừng lại ở dòng 880 (dòng `END;` đóng block bên trong), nên bị thiếu mất đúng dòng `END;` ngoài cùng đó ở cuối.

**9. Đã sửa xong**

Bổ sung lại dòng `END;` còn thiếu trước dấu `/` ở cuối, vào cả `LG_PRO_DAILY_LOT_TRACKING_JOB_PATCHED.sql` lẫn file đối chứng, chạy lại công cụ kiểm tra cân bằng block: độ sâu stack cuối cùng = 0, không còn lệch ở cả 2 file. Đã gửi lại cho user compile lại trong Toad.

---

**Các điểm còn cần user/phía nghiệp vụ xác nhận trước khi đưa vào chạy chính thức:**
1. Việc giữ nguyên `LOT_NO` sau khi đổi mã nguyên liệu có chấp nhận được không, hay cần 1 số tham chiếu lot khác.
2. Số dư sổ sách BRAMID bị "mồ côi" sau đó (không bao giờ được job này tiêu thụ nữa) có ảnh hưởng gì tới tồn kho/giá thành phía sau hay không.
3. Xác nhận quy trình chạy lại cho tháng 8 (tạm thời hardcode `L_MONTH:='202608'`, chạy tay 1 lần, rồi revert lại) có khớp với dự tính của DONGIL/consultant.
4. Xác nhận phạm vi áp dụng vĩnh viễn, không giới hạn thời gian (toàn bộ 8 mã trong nhóm nguyên liệu 27) có đúng là hành vi mong muốn lâu dài hay không.
5. Có nên báo cáo riêng lỗi `OUTER JOIN` không liên quan phát hiện ở bước 5 thành 1 ticket khác hay không.
