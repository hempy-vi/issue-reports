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

**10. BC bổ sung kết luận "không cần sửa code" — kiểm chứng lại, phát hiện sai**

User gửi ảnh chụp màn PM0407 (WIP1/CM00P1, Biz Center DI_F1-Factory 1, tháng 07/2026, tab Material Detail) kèm nội dung BC bổ sung vào ticket: "Data Analysis: Upon checking the actual data for July 2026, the remaining WIP raw material inventory consists strictly of US Cotton (USOT3137). There are no mixed non-USOT3137 materials (e.g., BRAMID) carried over. Conclusion: ... no system code modification is required for this task."

Kiểm chứng lại bằng query trực tiếp thay vì chấp nhận luôn. Trước tiên xác nhận `BIZ_CENTER=3402` là biz center DUY NHẤT có dữ liệu lot tracking trong toàn hệ thống (không có site/factory khác gây nhầm lẫn) — khớp chính xác "DI_F1-Factory 1". Sau đó query đúng phạm vi BC đã kiểm tra:

```sql
SELECT M.STD_YM, I.ITEM_CODE AS MAT_ITEM, COUNT(DISTINCT M.MIX_LOT) MIX_LOTS,
       ROUND(SUM(M.INPUT_QTY),2) TOTAL_IN, ROUND(SUM(M.OUTPUT_QTY),2) TOTAL_OUT
FROM TLG_LOT_TRACKING_MAT M
JOIN TLG_IT_ITEM I ON I.PK = M.TLG_IT_ITEM_PK
JOIN TLG_IT_ITEM I2 ON I2.PK = M.TLG_IT_ITEM_PK_MIX
WHERE I2.ITEM_CODE = 'CM00P1' AND M.BIZ_CENTER = 3402
  AND M.STD_YM IN ('202607','202608') AND M.DEL_IF = 0
GROUP BY M.STD_YM, I.ITEM_CODE ORDER BY M.STD_YM, TOTAL_IN DESC
```

Kết quả:
```
202607  USOT3137   92 mixing lot   704,381.12 kg
202607  BRAMID     65 mixing lot   505,088.01 kg
202608  USOT3137  172 mixing lot 1,775,525.71 kg
202608  BRAMID     65 mixing lot   505,088.01 kg   (carry-over nguyên vẹn, không đổi)
```

BRAMID chiếm ~42% tổng khối lượng trong đúng phạm vi tháng 7 mà BC đã kiểm tra — mâu thuẫn trực tiếp với kết luận "no mixed non-USOT3137 materials". Xác định lý do BC không nhìn thấy: các Mixing Lot hiển thị trên UI (`MIXEDF1-260728` đến `MIXEDF1-260731`, tạo cuối tháng 7) đã đúng 100% USOT3137 thật — nhưng các mixing lot cũ hơn (`MIXEDF1-260627-xx`, tạo cuối tháng 6) vẫn còn BRAMID, nằm ở phần cần cuộn thêm trong lưới Material Detail (grid có scrollbar). Kết luận: **không đồng ý** với "no system code modification is required" — báo lại cho user kèm số liệu cụ thể, giữ nguyên khuyến nghị patch.

**11. Phát hiện procedure song song — patch hoá ra đã được apply thật lên production**

User đưa ví dụ site KYUNGBANG, nơi `LG_PRO_DAILY_LOT_TRACKING_JOB` chỉ là 1 wrapper mỏng gọi `LG_PRO_DAILY_LOT_TRACKING(L_MONTH, 'DD')` (1 procedure riêng có tham số `P_YYYYMM`, `P_TYPE`), yêu cầu làm giống vậy cho DONGIL.

Trước khi thực hiện, kiểm tra thì phát hiện DONGIL **cũng đã có sẵn** `LG_PRO_DAILY_LOT_TRACKING(P_YYYYMM, P_TYPE default 'MM')` — 1346 dòng, nội dung gần như song song với `_JOB` (chỉ khác 3 dòng: signature, `L_MONTH:=p_yyyymm`, `L_PROCESS_TYPE:=P_TYPE` thay vì hardcode). Diff trực tiếp qua SQL giữa 2 object xác nhận điều này:

```sql
SELECT A.LINE, A.TEXT AS JOB_TEXT, B.TEXT AS PARAM_TEXT
FROM (SELECT LINE, TEXT FROM ALL_SOURCE WHERE NAME='LG_PRO_DAILY_LOT_TRACKING_JOB' AND TYPE='PROCEDURE') A
FULL OUTER JOIN (SELECT LINE, TEXT FROM ALL_SOURCE WHERE NAME='LG_PRO_DAILY_LOT_TRACKING' AND TYPE='PROCEDURE') B
  ON A.LINE = B.LINE
WHERE NVL(A.TEXT,'~') <> NVL(B.TEXT,'~')
ORDER BY A.LINE
```

Diff cũng vô tình phát hiện: `LG_PRO_DAILY_LOT_TRACKING_JOB` trong DB hiện đang chứa đúng nội dung 5 điểm patch USOT3137 (comment `-- [PATCH USOT3137] Edit 1...` xuất hiện ngay trong `ALL_SOURCE`) — hỏi lại và **user xác nhận đã tự compile bản patch v2 thật lên production rồi** (có tạo 1 bản backup `LG_PRO_DAILY_LOT_TRACKING_JOB_BK` trước khi ghi đè).

Cảnh báo rủi ro cho user: nếu tạo `_JOB` thành wrapper gọi thẳng `LG_PRO_DAILY_LOT_TRACKING` (bản có tham số, **chưa** patch) đúng y yêu cầu ban đầu, sẽ vô hiệu hoá rule USOT3137 vừa apply (vì `CREATE OR REPLACE` sẽ ghi đè mất patch trong `_JOB`, và logic thật chuyển sang chạy ở `LG_PRO_DAILY_LOT_TRACKING` — nơi chưa patch).

**12. Theo yêu cầu user, chuyển toàn bộ logic + patch sang LG_PRO_DAILY_LOT_TRACKING**

User làm rõ: `LG_PRO_DAILY_LOT_TRACKING` sẽ là procedure xử lý chính, áp dụng patch cho **cả 2 nhánh** `'MM'` và `'DD'`; `_JOB` chỉ còn là wrapper gọi vào đó, theo đúng style KYUNGBANG.

Đọc lại toàn bộ nhánh `'MM'` (dòng 108-282 gốc) lần đầu tiên (trước giờ coi là dead code, chưa từng phân tích kỹ): cấu trúc đơn giản hơn nhánh `'DD'` — 3 vòng lặp giống hệt nhau (không qua cơ chế ration/rate như DD), copy trực tiếp từ bảng snapshot đóng sổ `TLG_CL_CLOSING_MAT_DETAIL` (cột `MAT_BEGIN_QTY`/`MAT_IN_QTY`/`MAT_OUT_QTY` tương ứng `STOCK_TYPE='OPEN'/'DAILY'/'MAPPING'`), cả 3 đều lấy thẳng `A.MAT_ITEM_PK` — đã JOIN sẵn tới `TLG_IT_ITEM I`, chỉ cần thêm CASE WHEN vào 1 cột SELECT mỗi vòng:

```sql
-- Trước:
,A.MAT_ITEM_PK TLG_IT_ITEM_PK,
-- Sau (Edit M1/M2/M3, áp cho cả 3 vòng lặp bằng replace_all):
,CASE WHEN I.TLG_IT_ITEMGRP_PK = L_USOT_ITEMGRP_PK AND I.PK <> L_USOT_ITEM_PK
      THEN L_USOT_ITEM_PK ELSE A.MAT_ITEM_PK END TLG_IT_ITEM_PK,
```

Lấy verbatim toàn bộ source `LG_PRO_DAILY_LOT_TRACKING` (RIÊNG, không tái dùng của `_JOB`, để tránh sai lệch) qua `ALL_SOURCE` — xác nhận thân bài giống hệt `_JOB` bản gốc (trước patch), chỉ khác 3 dòng đầu.

Soạn `LG_PRO_DAILY_LOT_TRACKING_PATCHED.sql` — full `CREATE OR REPLACE` với **8 điểm patch**: Edit 1/2 (khai báo 4 biến local + 1 `SELECT INTO` tra `USOT3137` chạy trước cả nhánh MM lẫn DD), Edit M1/M2/M3 (nhánh MM, mới), Edit 3 (carry-over CTE, DD), Edit 5 (transfer-in cursor, DD), Edit 4 (ration cursor, DD) — verify bằng script kiểm tra cân bằng riêng (đếm độ lồng `BEGIN`/`CASE`/`IF`/`LOOP`/`END`, bỏ qua string/comment): độ sâu stack cuối = 0, 0 lệch, số ngoặc cân bằng (250 mở = 250 đóng).

Soạn `LG_PRO_DAILY_LOT_TRACKING_JOB_WRAPPER.sql` — wrapper mỏng cho `_JOB`, đúng mẫu KYUNGBANG:

```sql
CREATE OR REPLACE PROCEDURE LG_PRO_DAILY_LOT_TRACKING_JOB
IS
    L_MONTH VARCHAR2(6):=to_char(sysdate, 'yyyymm');
BEGIN
    LG_PRO_DAILY_LOT_TRACKING(L_MONTH, 'DD');
END;
/
```

Ghi chú thứ tự bắt buộc khi áp dụng: chạy file procedure chính TRƯỚC, wrapper SAU — nếu ngược lại, `_JOB` sẽ tạm thời gọi vào bản `LG_PRO_DAILY_LOT_TRACKING` chưa patch. Lợi ích phụ quan trọng: từ giờ chạy lại cho 1 tháng cụ thể (vd tháng 8) chỉ cần gọi trực tiếp `EXEC LG_PRO_DAILY_LOT_TRACKING('202608', 'DD');` với tham số, không cần sửa tạm `L_MONTH` hardcode nữa như phương án Runbook cũ.

**13. Xác nhận cả 2 object đã compile thành công trên production**

Kiểm tra lại `ALL_OBJECTS`: cả `LG_PRO_DAILY_LOT_TRACKING` (last_ddl_time 13:39:03) và `LG_PRO_DAILY_LOT_TRACKING_JOB` (13:39:24 — đúng thứ tự main trước, wrapper sau, cách nhau 21 giây) đều `STATUS=VALID`. Đọc lại `ALL_SOURCE` xác nhận `_JOB` giờ đúng là wrapper 6 dòng, và `LG_PRO_DAILY_LOT_TRACKING` có đủ 9 marker patch (`Edit 1`, `Edit 2`, `Edit M1/M2/M3` ×3, `Edit 3` ×2, `Edit 5`, `Edit 4`).

**14. Tổng duyệt cuối cùng (workflow 3 agent phản biện độc lập, chạy song song)**

Trước khi coi issue này hoàn tất về mặt code, chạy 1 workflow rà soát toàn diện lần cuối trên chính procedure đã live:

- **Agent 1 (nhánh MM)**: xác nhận cả 3 vòng lặp đã patch đúng kỹ thuật (`CASE WHEN` lấy đúng alias `I` = `TLG_IT_ITEM` join sẵn trong from-clause của từng cursor; downstream `CUR.TLG_IT_ITEM_PK` trong câu INSERT resolve đúng chuẩn PL/SQL). **Phát hiện quan trọng**: nhánh MM hiện KHÔNG có tác dụng thực tế trong production DONGIL — truy vấn `ALL_DEPENDENCIES` (`referenced_name='LG_PRO_DAILY_LOT_TRACKING'`) chỉ trả về đúng 1 dòng phụ thuộc là `_JOB` (hardcode `'DD'`), và `DBMS_JOB` (job đang active duy nhất) cũng chỉ gọi `_JOB`. Không object nào khác trong schema gọi với `'MM'`.
- **Agent 2 (rà soát đầy đủ)**: quét toàn bộ 916 dòng active, xác nhận đúng **10/10 điểm `INSERT INTO TLG_LOT_TRACKING_MAT`** ghi `TLG_IT_ITEM_PK` đều đã chuẩn hoá (7 điểm trực tiếp qua CASE WHEN, 3 điểm ration cuối cùng chuẩn hoá gián tiếp qua bảng tạm `TLG_LOT_TRACKING_TEMP` — vốn build straight từ `TLG_LOT_TRACKING_MAT` đã sạch). Phát hiện 1 điểm yếu kiến trúc nhẹ (chưa phải lỗ hổng thực tế): bước build `TEMP` và 3 điểm ration cuối không có CASE WHEN riêng của mình, hoàn toàn phụ thuộc (transitively) vào các nguồn upstream đã sạch — là 1 "single point of failure" nếu sau này có 1 đường ghi `MAPPING` mới quên chuẩn hoá. Khuyến nghị thêm 1 lớp CASE WHEN phòng thủ tại bước build TEMP, không bắt buộc ngay.
- **Agent 3 (trạng thái dữ liệu thực tế)**: xác nhận dữ liệu tháng 8 (`STD_YM='202608'`) **hoàn toàn chưa được patch chạm tới** — vẫn còn 1,093 dòng BRAMID / 505,088.00899 kg y nguyên (65 mixing lot, 9 lot vật lý, tất cả tạo cùng 1 batch đóng sổ cuối tháng 8 lúc 23:30-23:31 ngày 31/08). Xác nhận lại: vì `_JOB` luôn lấy `L_MONTH` từ `SYSDATE`, chạy vào bất kỳ ngày nào trước khi sang tháng 10 sẽ luôn tính `L_MONTH='202609'` — không bao giờ tự động đụng tới tháng 8. Cũng phát hiện: tháng 9 (tháng hiện tại) vẫn còn BRAMID sống (294 dòng OPEN + 160 dòng MAPPING) vì lần chạy nightly gần nhất (đêm 07/09 23:30) xảy ra TRƯỚC khi patch compile (08/09 13:39) — về lý thuyết nightly đêm nay (08/09 → 09/09) sẽ tự làm sạch (job soft-delete + build lại toàn bộ MAT của tháng hiện tại mỗi đêm), nhưng chưa được quan sát thực tế.

Kết luận tổng duyệt: code đã đúng và đầy đủ, đã live trên production, nhưng còn 2 việc vận hành cấp bách cần làm trước ngày đóng sổ 10/09: (1) gọi tay `EXEC LG_PRO_DAILY_LOT_TRACKING('202608', 'DD');` để xử lý tháng 8, (2) xác nhận lại kết quả nightly đêm nay có tự làm sạch tháng 9 đúng như thiết kế hay không.

---

**Các điểm còn cần user/phía nghiệp vụ xác nhận:**
1. Việc giữ nguyên `LOT_NO` sau khi đổi mã nguyên liệu có chấp nhận được không, hay cần 1 số tham chiếu lot khác.
2. Số dư sổ sách BRAMID bị "mồ côi" sau đó (không bao giờ được job này tiêu thụ nữa) có ảnh hưởng gì tới tồn kho/giá thành phía sau hay không.
3. **[CẤP BÁCH, chưa thực hiện]** Cần gọi tay `EXEC LG_PRO_DAILY_LOT_TRACKING('202608', 'DD');` để xử lý tháng 8 trước ngày đóng sổ 10/09 — patch không tự sửa dữ liệu cũ.
4. Xác nhận phạm vi áp dụng vĩnh viễn, không giới hạn thời gian (toàn bộ 8 mã trong nhóm nguyên liệu 27) có đúng là hành vi mong muốn lâu dài hay không.
5. Có nên báo cáo riêng lỗi `OUTER JOIN` không liên quan phát hiện ở bước 5 thành 1 ticket khác hay không.
6. **[MỚI]** Nhánh MM hiện không có tác dụng cho DONGIL (không object nào gọi tới) — có cần thiết phải duy trì patch trên nhánh này không, hay chỉ để phòng hờ cho site khác dùng chung procedure?
7. **[MỚI]** Có muốn bổ sung 1 lớp CASE WHEN phòng thủ tại bước build `TLG_LOT_TRACKING_TEMP` (khuyến nghị của Agent 2, không bắt buộc) hay không?
8. **[MỚI, cần theo dõi]** Xác nhận lại sau nightly đêm nay (08/09 → 09/09): dữ liệu tháng 9 có tự làm sạch hết BRAMID như thiết kế hay không.
