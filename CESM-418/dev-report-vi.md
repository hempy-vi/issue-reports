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

**15. Kiểm tra an toàn trước khi chạy tay xử lý tháng 8 (trước khi tự chạy)**

User hỏi tháng 8 đã trải qua monthly closing chưa, và đề xuất tự chạy tay `EXEC LG_PRO_DAILY_LOT_TRACKING('202608','MM')` rồi `('202609','DD')`.

Query bảng `TLG_CL_CLOSING_MAT_DETAIL` (bảng snapshot đóng sổ) xác nhận tháng 8 đã có snapshot (551 dòng, `PHASE_NAME='WIP1'`, tạo lúc 2026-09-05 08:20-08:41, `SUM_BEGIN=119,277.13` khớp đúng số dư cuối tháng 7 trên ảnh PM0407 gốc). Phát hiện bất thường: **toàn bộ 551 dòng snapshot đều mang mã `USOT3137`, không còn dòng BRAMID nào** — trong khi `TLG_LOT_TRACKING_MAT` (bảng Lot Tracking thật) vẫn còn nguyên 1,093 dòng BRAMID cho tháng 8. Chưa rõ đây là do rule đã tự đúng ở nguồn khác hay do snapshot bị thiếu dữ liệu.

Chạy 1 workflow 4-agent điều tra song song (`snapshot-origin`, `coverage-compare`, `mm-dry-run`, `approval-lock-check`) trước khi cho phép chạy MM. Kết quả:

1. **Nguồn gốc snapshot**: `TLG_CL_CLOSING_MAT_DETAIL` được ghi bởi `LG_PRO_KBPR00300_7` (+ bản `_V2_7`) — 1 pipeline đóng sổ **hoàn toàn khác**, không liên quan gì tới `LG_PRO_DAILY_LOT_TRACKING`, không đọc `TLG_LOT_TRACKING_MAT`. Nguồn của nó là `TLG_CL_CLOSING_MAT_D` (bảng master đóng sổ riêng) + `TLG_ST_TRANSFER_D/M` (giao dịch chuyển kho MIX thật, `STATUS=3`) để tính tỷ lệ, cộng với carry-forward từ snapshot tháng trước của chính nó **chỉ khi `MAT_END_QTY>0`**. 65 lot BRAMID CÓ mặt trong snapshot tháng 7 (`SUM_OUT=505,088.0088` khớp gần đúng `OUTPUT_QTY` thật) nhưng `SUM_END=0` tại đó — theo đúng logic riêng của pipeline này, chúng "đã dùng hết" cuối tháng 7 nên không carry sang tháng 8 (không phải bug ở bước carry của pipeline đó).
2. **Đối chiếu độ phủ**: snapshot 551 dòng tháng 8 chỉ có **90 LOT_NO riêng biệt** (từ `MIXEDF1-260729-01` đến `260831-04`, cutoff cứng đúng ngày 2026-07-29). `TLG_LOT_TRACKING_MAT` tháng 8 có **237 mixing lot** (65 BRAMID + 172 USOT3137). **147 mixing lot (mọi lot tạo từ 260627 đến 260728, gồm cả 65 lot BRAMID lẫn 82 lot USOT3137 thuần) hoàn toàn không có mặt trong snapshot dưới bất kỳ mã nào** — tổng snapshot chỉ bằng 47.6% IN / 77.7% OUT so với Lot Tracking thật.
3. **Chạy thử (SELECT-only) 3 cursor của nhánh MM cho '202608'**: xác nhận sẽ KHÔNG lỗi exception (`SELECT INTO` chỉ dùng `MAX()`, luôn trả 1 dòng dù JOIN miss — không phải `NO_DATA_FOUND` như giả định ban đầu) nhưng sẽ tạo ra **1,054 dòng thay vì 3,546 dòng hiện có** — vì bước đầu tiên `UPDATE ... SET DEL_IF=PK WHERE STD_YM LIKE '202608%'` xoá mềm toàn bộ rồi chỉ dựng lại từ 90 LOT_NO có trong snapshot. **65 lot BRAMID (505,088kg) sẽ biến mất thật, không phải đổi mã.**
4. **Trạng thái khoá/duyệt**: tháng 8 WIP1 hiện đã Approved (`TLG_CL_CLOSING_WIP_M`, `CONFIRM_DT=2026-09-05 08:57:38`, giống trạng thái tháng 7). `LG_PRO_DAILY_LOT_TRACKING` (toàn bộ 916 dòng, cả DD lẫn MM) hoàn toàn không đọc/ghi bảng `TLG_CL_CLOSING_WIP_M` — chạy lại không đụng/phá cờ Approved, nhưng cũng không có gì chặn chạy lại. Insert `TLG_LOT_TRACKING_HIS` đã bão hoà cho 202608 (62/62 dòng, có guard `NOT EXISTS`) nên chạy lại không tạo trùng lặp lịch sử đóng sổ.

**Kết luận: `EXEC LG_PRO_DAILY_LOT_TRACKING('202608','MM')` không được chạy** — nhánh MM của DONGIL vẫn đúng như comment cũ của dev ("ko chay, cua dongil, can sua lai"): phụ thuộc vào 1 bảng snapshot của pipeline đóng sổ khác, hiện chỉ phủ 90/237 mixing lot của tháng 8. Đã soạn sẵn phương án dự phòng rủi ro thấp `results/fix_august_direct_update_OPTION_B.sql` — UPDATE trực tiếp 1,093 dòng BRAMID sang USOT3137 (không đổi số lượng).

User bổ sung ngữ cảnh nghiệp vụ quan trọng: mọi bảng/store có tên chứa `LOT_TRACKING` thuộc 1 module độc lập, an toàn để xoá/sửa/chạy lại toàn bộ; về nghiệp vụ, tháng đã monthly closing bắt buộc phải có data Lot Tracking monthly chạy được cho tháng đó. Với thông tin này, khuyến nghị cuối: dùng **chính thủ tục đã patch, gọi với `P_TYPE='DD'`** cho cả 2 tháng (`EXEC LG_PRO_DAILY_LOT_TRACKING('202608','DD')` rồi `('202609','DD')`) thay vì `'MM'` — nhãn `'DD'`/`'MM'` chỉ là tên cơ chế tính (daily-mechanism vs monthly-snapshot-mechanism) trong code, không phải yêu cầu bắt buộc dùng `'MM'` cho tháng đã đóng sổ; nhánh DD mới là cơ chế đã audit đầy đủ (xem bước 14), tính từ dữ liệu nguồn thật cho toàn bộ 237 mixing lot.

**16. User chạy `DD('202608')` rồi `DD('202609')` — phát hiện lỗi mới: mất dữ liệu ở DAILY/transfer-in**

Verify sau khi chạy: nhóm nguyên liệu 27 (cotton) tháng 8/9 giờ chỉ còn USOT3137 (đúng mục tiêu chính của patch), nhưng khi so khối lượng trước/sau theo `STOCK_TYPE`:
- `OPEN` tháng 8: 837 dòng / 1,209,469.13kg — khớp chính xác số liệu trước khi chạy (OK).
- **`DAILY` tháng 8: 484 dòng/1,071,144.59kg (trước) → chỉ còn 12 dòng/26,823.08kg (sau) — mất ~97.5%.**
- `MAPPING` tháng 8: 2,225 dòng/1,400,461.90kg (trước) → 1,588 dòng/1,087,663.36kg (sau) — mất ~22% (nghi ngờ ban đầu là hệ quả dây chuyền, xem bước 19 để biết kết luận cuối).

Xác nhận nguồn dữ liệu chuyển kho thật (`TLG_ST_TRANSFER_D/M`, `STATUS=3`, `TR_DATE LIKE '202608%'`, `W.PROCESS_TYPE='MIXED'`) vẫn đầy đủ (~1,084,642.69kg) — không mất ở nguồn, nên lỗi nằm ở logic ghi, không phải mất dữ liệu nguồn.

**Root cause xác minh bằng ví dụ cụ thể** (`SLIP_NO='TR26-0624'`, `TLG_WI_LINE_M_PK=8360`): cursor transfer-in (dòng ~413-442 nguồn) có điều kiện chống trùng:

```sql
AND NOT EXISTS (SELECT 1 FROM TLG_LOT_TRACKING_MAT Z WHERE Z.DEL_IF = 0
  AND Z.TLG_WI_LINE_M_PK = L.PK AND Z.TRIN_TYPE = 'I60' AND Z.SLIP_NO = M.SLIP_NO)
```

Điều kiện này **không lọc theo `STD_YM`, không phân biệt `STOCK_TYPE`**. `TRIN_TYPE='I60'` được set chung bởi cả block insert DAILY (dòng 455) lẫn block carry-over insert OPEN cho tháng sau (dòng 380). Vì tháng 9 đã chạy nightly liên tục (tới 2026-09-07 23:30) và carry-over cùng `SLIP_NO`/`WI_LINE` này sang OPEN của 202609, khi chạy lại `DD('202608')` hôm nay, `NOT EXISTS` thấy "đã có" (dòng OPEN của 202609) nên bỏ qua insert DAILY thật của tháng 8. Xác nhận trực tiếp trong DB: các dòng soft-delete của `SLIP_NO='TR26-0624'`/`WI_LINE=8360` cho STD_YM='202609' có `CRT_DT` mỗi đêm liên tục từ 2026-08-05 đến 2026-09-07, chứng minh cơ chế carry-over-reuse-SLIP_NO này đã tồn tại lâu.

**Lỗi có sẵn từ trước, KHÔNG do patch USOT3137 gây ra** (patch Edit 5 chỉ sửa SELECT/GROUP BY ở vùng này, không đụng `NOT EXISTS`) — vô hại bấy lâu vì `_JOB` luôn chạy tuần tự đúng tháng hiện tại, chưa từng có kịch bản chạy lại 1 tháng đã qua trong khi tháng sau đã có data. Toàn bộ 484 dòng DAILY gốc của tháng 8 vẫn còn nguyên trong DB (soft-delete qua `DEL_IF`, không mất thật) — có thể khôi phục.

**17. Rà soát toàn bộ 916 dòng procedure tìm mọi guard cùng dạng lỗi**

Chạy 1 workflow điều tra: quét toàn bộ NOT EXISTS/NOT IN/MERGE/UNIQUE trong procedure — chỉ có đúng **4 guard constructs** trong toàn bộ 916 dòng:
- Dòng 98 (preamble chung, marker ngày `TLG_LOT_TRACKING_HIS`) — AN TOÀN, khớp theo `YYYYMMDD` đầy đủ.
- Dòng 438 (DD, cursor transfer-in DAILY) — NGUY HIỂM, root cause đã xác nhận ở bước 16.
- Dòng 535 (DD, cursor PROD-income → `TLG_LOT_TRACKING_PROD`) — **NGUY HIỂM, cùng dạng lỗi, chưa từng trigger thật** (`TLG_LOT_TRACKING_PROD.STD_YM` tồn tại và được ghi bởi chính INSERT này ở dòng 492/497, nhưng guard không kiểm tra nó).
- Dòng 567 (DD, cursor MAPPING/ration `CUR`) — AN TOÀN, có `A.STOCK_DATE = Z.TR_DATE` (so khớp ngày đầy đủ, mịn hơn tháng); guard này cũng bảo vệ luôn 2 INSERT O60 phía sau trong cùng vòng lặp (dòng 693, 837) vì kế thừa filter của `CUR`.

Xác nhận khối OPEN carry-over (dòng 313-394) **không có guard nào cả** — insert vô điều kiện sau khi đã xoá mềm theo tháng — khớp đúng với kết quả thực nghiệm (không đổi trước/sau). Xác nhận nhánh MM (dòng 122-299) không có guard nào — miễn nhiễm hoàn toàn với lớp lỗi này bất kể thứ tự chạy.

**18. Thiết kế và verify bản vá cho 2 guard nguy hiểm**

Fix tối thiểu — thêm điều kiện lọc vào WHERE có sẵn, không đổi cấu trúc:

```sql
-- Guard dòng 438 (DAILY transfer-in) — Before:
AND NOT EXISTS (SELECT 1 FROM TLG_LOT_TRACKING_MAT Z WHERE Z.DEL_IF = 0
  AND Z.TLG_WI_LINE_M_PK = L.PK AND Z.TRIN_TYPE = 'I60' AND Z.SLIP_NO = M.SLIP_NO)
-- After:
AND NOT EXISTS (SELECT 1 FROM TLG_LOT_TRACKING_MAT Z WHERE Z.DEL_IF = 0
  AND Z.TLG_WI_LINE_M_PK = L.PK AND Z.TRIN_TYPE = 'I60' AND Z.SLIP_NO = M.SLIP_NO
  AND Z.STD_YM = L_MONTH AND Z.STOCK_TYPE = 'DAILY')

-- Guard dòng 535 (PROD-income) — Before:
AND NOT EXISTS (SELECT 1 FROM TLG_LOT_TRACKING_PROD Z WHERE Z.DEL_IF = 0
  AND Z.STOCK_NO = M.SLIP_NO AND Z.TLG_IT_ITEM_PK_PROD = D.ITEM_PK
  AND Z.TLG_IT_ITEM_PK_MIX = C.CHILD_PK AND D.LOT_NO = Z.LOT AND STOCK_TYPE = 'PROD')
-- After:
AND NOT EXISTS (SELECT 1 FROM TLG_LOT_TRACKING_PROD Z WHERE Z.DEL_IF = 0
  AND Z.STOCK_NO = M.SLIP_NO AND Z.TLG_IT_ITEM_PK_PROD = D.ITEM_PK
  AND Z.TLG_IT_ITEM_PK_MIX = C.CHILD_PK AND D.LOT_NO = Z.LOT AND STOCK_TYPE = 'PROD'
  AND Z.STD_YM = L_MONTH)
```

Guard dòng 438 cần cả `STD_YM` lẫn `STOCK_TYPE` vì khối OPEN carry-over cùng tháng cũng ghi `TRIN_TYPE='I60'`/`STD_YM=L_MONTH` (chỉ khác `STOCK_TYPE='OPEN'`) — nếu chỉ lọc `STD_YM` vẫn có thể va nhầm vào chính dòng OPEN cùng tháng. Verify tính an toàn cho cả 2 kịch bản: (1) chạy lại đúng 1 tháng nhiều lần liên tiếp — vẫn idempotent vì `UPDATE ... SET DEL_IF=PK WHERE STD_YM LIKE L_MONTH||'%'` đã xoá mềm dòng active của tháng đó TRƯỚC khi 2 guard này chạy; (2) chạy lại 1 tháng quá khứ khi tháng sau đã có dữ liệu — nay được cho phép đúng vì dòng của tháng sau mang `STD_YM` khác.

Re-verify độc lập cả 2 điểm sửa với live `ALL_SOURCE` (không tin lại transcription cũ) — khớp chính xác từng ký tự, không có drift. Ghép `results/LG_PRO_DAILY_LOT_TRACKING_PATCHED_v3_guardfix.sql` (full `CREATE OR REPLACE`, áp patch qua Edit tool exact-match — cả 2 lần thành công ngay lần đầu) và `results/LG_PRO_DAILY_LOT_TRACKING_live_before_guardfix_backup.sql` (đối chiếu). Verify cấu trúc: đếm `BEGIN`/`END`/`IF`/`LOOP`/`CASE` khớp tuyệt đối giữa 2 file (27/59/12/28/17), `diff` chỉ hiện đúng 2 vùng thay đổi dự kiến (2 comment + 2 mệnh đề AND), thân bài dài thêm đúng 2 dòng (917→919).

**19. Điều tra vai trò của `TLG_LOT_TRACKING_GD_M/GD_D` (đứng sau màn hình melt060/melt070) trong kế hoạch khôi phục**

Trước khi quyết định phương án khôi phục, kiểm tra xem 2 bảng "Goods-Delivery Lot Tracking" này có cần nằm trong phạm vi xoá+chạy-lại không (chúng đứng sau màn hình theo dõi xuất xứ nguyên liệu cho hàng đã giao). Tìm và đọc source `LG_SEL_MELT060_01` (dựng cây bên trái, gọi `LG_PRO_LOT_TRACKING_GD_M` làm bước đầu tiên để "rebuild-on-open" mỗi khi user search theo khoảng ngày), `LG_PRO_LOT_TRACKING_GD_M`/`LG_PRO_LOT_TRACKING_GD_D` (2 procedure RIÊNG BIỆT, khác hoàn toàn `LG_PRO_DAILY_LOT_TRACKING`).

Phát hiện chính:
- `LG_PRO_DAILY_LOT_TRACKING` **không tự INSERT** `MAT_ITEM`/`MAT_LOT`/`MAT_KG` vào `TLG_LOT_TRACKING_GD_D` — việc này do `LG_PRO_LOT_TRACKING_GD_D` làm, đọc trực tiếp từ `TLG_LOT_TRACKING_MAT WHERE TROUT_TYPE='O60'` (chính bảng bị bug BRAMID/USOT3137) — nên `GD_D.MAT_ITEM` liên quan trực tiếp tới bug đang sửa.
- `LG_PRO_DAILY_LOT_TRACKING` có tự dọn `GD_M/GD_D` (dòng 62-79) nhưng CHỈ xoá mềm `LEVEL_TYPE='LOT'`, và row có `MAIL_YN='Y'` (đã gửi mail/morningmate) sẽ KHÔNG bao giờ bị đụng tới (không lọc theo tháng) — đây chỉ là bước dọn dẹp, không tự INSERT lại gì; việc rebuild thật sự chỉ xảy ra khi ai đó mở lại màn hình melt060/melt070 cho đúng khoảng ngày.
- Phát hiện phụ: `TLG_LOT_TRACKING_GD_M.DELI_DATE` luôn rỗng (bug riêng trong `LG_PRO_LOT_TRACKING_GD_M` dòng 30: cursor hardcode `'' AS OUT_DATE` thay vì `M.OUT_DATE`) — không thể dùng cột này để lọc DELETE theo tháng, phải join qua `TLG_GD_OUTGO_M.OUT_DATE` bằng `TLG_GD_OUTGO_M_PK`.
- Thực nghiệm: kiểm tra trực tiếp cho khoảng 2026-08-01 đến 09-08 — toàn bộ 927/927 dòng GD_M `LEVEL_TYPE='LOT'` liên quan đã bị xoá mềm (`DEL_IF<>0`), tất cả đều `MAIL_YN='N'` — đã bị quét sạch như hệ quả phụ của chính 2 lần `EXEC DD` chạy tay ở bước 16. Không cần DELETE tay thêm cho `GD_M/GD_D` lần này, nhưng đây chỉ là may mắn cho đúng lần này (chưa có row `MAIL_YN='Y'` nào trong khoảng), không nên dựa vào cơ chế tự dọn này cho các lần sau.

**20. User quyết định hướng khôi phục: DELETE sạch tháng 8+9 rồi chạy lại DD**

User chỉ đạo xoá toàn bộ data tháng 8+9 trên mọi bảng `LOT_TRACKING` liên quan, sau đó chạy lại DD. Rà soát toàn bộ 9 bảng có tên chứa `LOT_TRACKING`: chỉ **3 bảng** có cột `STD_YM` VÀ thực sự được `LG_PRO_DAILY_LOT_TRACKING` ghi (xác nhận qua chính block `TRUNCATE` comment sẵn ở đầu source, dòng 5-9): `TLG_LOT_TRACKING_MAT` (92,630 dòng thg8+9, tính cả các generation đã soft-delete), `TLG_LOT_TRACKING_PROD` (13,096 dòng), `TLG_LOT_TRACKING_HIS` (122 dòng). `TLG_LOT_TRACKING_GD_M/GD_D` (đã điều tra ở bước 19 — không cần xoá tay), `TLG_LOT_TRACKING_TEMP` (không có cột tháng nào, tự rebuild mỗi lần chạy), `TLG_GD_LOT_TRACKING_D/M`/`TLG_LOT_TRACKING_PROD_BC` (tên giống nhưng không được procedure này đụng tới) — không nằm trong phạm vi.

Soạn `results/recovery_delete_and_rerun_202608_202609.sql`: `DELETE` (hard delete) cả 3 bảng `WHERE STD_YM IN ('202608','202609')`, verify đã sạch, rồi `EXEC LG_PRO_DAILY_LOT_TRACKING('202608','DD')` → `('202609','DD')`, rồi verify kết quả cuối. Khuyến nghị kèm theo: vẫn nên compile patch guard-fix (bước 18) TRƯỚC khi chạy script này — xoá sạch tháng 9 giải quyết được sự cố hôm nay (không còn dòng nào để guard lỗi va phải), nhưng bug gốc vẫn tồn tại nếu không vá, sẽ tái phát nếu sau này chạy lại 1 tháng cũ bất kỳ trong khi tháng sau đã có dữ liệu.

**21. User compile patch guard-fix + chạy recovery script — verify kết quả**

Verify: DAILY tháng 8 giờ **489 dòng/1,084,642.69kg — khớp chính xác** với tổng nguồn thật (`TLG_ST_TRANSFER_D/M`) → xác nhận guard-fix hoạt động đúng, bug gốc đã hết. `OPEN` 837 dòng/1,209,469.13kg (không đổi). Toàn bộ nhóm cotton (group 27) chỉ còn USOT3137, không còn BRAMID — đúng mục tiêu chính CESM-418.

Phát hiện thêm 1 điểm cần làm rõ: `MAPPING` (ration) tháng 8 = 1,588 dòng/1,087,663.36kg — thấp hơn ~313K kg so với con số gốc trước sự cố hôm nay (2,225 dòng/1,400,461.90kg). Điểm mấu chốt: con số 1,588/1,087,663.36kg này **giống hệt** lần chạy hỏng lúc trước (khi DAILY chỉ có 12 dòng) — chứng minh khoản thiếu hụt MAPPING không liên quan tới bug guard vừa sửa (nếu liên quan, phải phục hồi tăng theo DAILY, nhưng không đổi 1kg nào).

Launch workflow điều tra riêng (3 agent song song) — cả 3 fail vì chạm giới hạn phiên làm việc (session limit), không phải lỗi logic. Chuyển sang điều tra thủ công trực tiếp:
- `TLG_PR_PROD_INCOME_M/D` tháng 8, `STATUS=3`: 832 chứng từ / 3,257,790.53kg, `MAX_MOD=2026-09-05 08:57:35` — trùng khớp gần như tuyệt đối với thời điểm đóng sổ WIP1 tháng 8 được Approve (`CONFIRM_DT=2026-09-05 08:57:38`) — cho thấy các chứng từ này bị chạm vào như 1 phần của chính quy trình đóng sổ, không phải dấu hiệu sửa/xoá bất thường. `STATUS=1` (chưa duyệt) chỉ 6 chứng từ/4,642.92kg — quá nhỏ để giải thích khoảng thiếu 313K kg.
- Đọc trực tiếp source cơ chế ration (dòng ~613-800): xác nhận ration cho 1 (`SLIP_NO`, `WI_LINE`, `ITEM`) chỉ được rút tồn từ đúng `TLG_LOT_TRACKING_TEMP` rows khớp chính xác `WI_LINE_M_PK` đó (dòng 670-671), không rút tự do từ tổng tồn kho toàn tháng. Nghĩa là dù tổng cung (`OPEN+DAILY = 2,294,111.82kg` hôm nay, thậm chí cao hơn historical ~2,280,613.72kg) rất dư dả, vẫn có thể xảy ra thiếu hụt cục bộ nếu tồn kho phân bổ không đúng theo từng `WI_LINE` cụ thể cần.

**Kết luận**: khoảng thiếu 313K kg ở MAPPING nhiều khả năng là biểu hiện của bug carry-over đã biết từ trước (bước 5: "JOIN sai... balance carry-over không bao giờ được trừ theo tiêu thụ thực") kết hợp với tính chất "path-dependent" của thuật toán ration (chạy lại 1 lần duy nhất từ đầu cho ra kết quả khác với chuỗi nhiều lần chạy tuần tự mỗi đêm trong suốt tháng 8 thực tế) — KHÔNG phải lỗi mới do patch hôm nay gây ra (bằng chứng: giống hệt cả ở lần chạy hỏng lẫn lần chạy đã sửa). Không vi phạm mục tiêu chính CESM-418. Cần báo cho business/DBA như 1 phát hiện riêng, không chặn việc đóng sổ 10/09.

**22. Phát hiện regression thật trên màn hình melt070 (Deli Lot Tracking V3) — do chính patch CESM-418 gây ra**

Business (qua chat nội bộ) báo màn hình melt070 (`/me/lt/melt070`) hiện nhiều dòng "Origin: none" + "0 files" cho tháng 8.

Đọc source `LG_SEL_MELT070_02`: Origin/file được tra bằng JOIN `TLG_KB_COTTON_INCOME_D` (ghi nhận nhập kho — giữ nguyên mã nguyên liệu GỐC, không bị patch đụng tới) với `TLG_LOT_TRACKING_GD_D` (lấy mã nguyên liệu từ `TLG_LOT_TRACKING_MAT` — đã bị patch đổi BRAMID→USOT3137) theo **cả mã nguyên liệu lẫn `LOT_NO`** (2 điểm: dòng ~61 trong CTE `TBL_LOT_INFO`, dòng ~204 ở LEFT JOIN chính):

```sql
-- CTE TBL_LOT_INFO, dòng ~61 (before):
AND DI.TLG_IT_ITEM_PK = X1.TLG_IT_ITEM_PK_MAT
AND DI.LOT_NO = TRIM(X1.MAT_LOT)
-- LEFT JOIN chính, dòng ~204 (before):
AND L.TLG_IT_ITEM_PK_MAT (+) = Z2.TLG_IT_ITEM_PK_MAT
AND L.MAT_LOT (+) = TRIM(Z2.MAT_LOT)
```

Khi mã không còn khớp (121 BRAMID ≠ 122 USOT3137), join trả NULL → Origin=NULL. Fallback heuristic (`ITEM_CODE LIKE 'USA%'`) cũng không cứu được vì `'USOT3137'` không khớp pattern `'USA%'` (bắt đầu bằng "USO" chứ không phải "USA").

**Verify bằng data thật**: cả 4 mẫu `LOT_NO` hiện "none/0 files" trên ảnh chụp màn hình (`203/1076/26-01`, `303C/665/26-01`, `402A/660/26-01`, `402C/219/26-01`) đều **thực sự được mua/ghi nhận là BRAMID** trong `TLG_KB_COTTON_INCOME_D` — xác nhận 100% đây là hệ quả trực tiếp từ patch CESM-418.

Màn hình này có nút `AUSTRALIA/BRAZIL/USA_GIN_CODE` — phục vụ chứng nhận xuất xứ bông cho hải quan/xuất khẩu, nơi Origin phải phản ánh đúng xuất xứ vật lý thật, không phải nhãn kế toán nội bộ đã chuẩn hoá. Đã báo 2 hướng cho user: (A) sửa JOIN của melt070 chỉ khớp theo `LOT_NO` (giữ nguyên sổ sách USOT3137 nhưng hiện đúng xuất xứ thật), (B) giữ nguyên hiện trạng. **User chọn (A).**

Verify an toàn trước khi sửa: xác nhận **0 `LOT_NO` nào trong `TLG_KB_COTTON_INCOME_D` gắn với >1 mã nguyên liệu khác nhau**, và **0 `LOT_NO` nào gắn với >1 PO_DOC/Origin khác nhau** — bỏ điều kiện so khớp mã nguyên liệu khỏi join hoàn toàn an toàn, không tạo fan-out/trùng dòng. Soạn `results/LG_SEL_MELT070_02_PATCHED_origin_join_fix.sql` — `CREATE OR REPLACE` đầy đủ, 2 điểm sửa đánh dấu `-- [PATCH MELT070-ORIGIN-FIX]`, chỉ bỏ đúng điều kiện so khớp mã nguyên liệu, giữ nguyên `LOT_NO` làm khoá join.

**23. Rà soát toàn schema — phát hiện thêm 5 procedure khác cùng lỗi**

User yêu cầu kiểm tra thêm "store popup" (procedure đứng sau popup xem file mở từ nút "X files" trên lưới). Tìm ra `LG_SEL_MELT060_02_FILES` dính chính xác cùng lỗi. Rà soát toàn schema (mọi object cùng dùng chung `TLG_KB_COTTON_INCOME_D` VÀ `TLG_LOT_TRACKING_GD_D`) cho ra tổng cộng **6 procedure** dính lỗi:

| Procedure | Màn hình/chức năng | Số điểm lỗi |
|---|---|---|
| `LG_SEL_MELT070_02` | Grid chính melt070 | 2 (đã vá bước 22) |
| `LG_SEL_MELT060_02_FILES` | Popup xem file | 1 |
| `LG_SEL_MELT021_02` | Màn hình melt021 | 2 |
| `LG_SEL_MELT060_02` | Grid chính melt060 (V2 của melt070) | 2 |
| `LG_SEL_MO00010` | Popup file (biến thể, dùng `IN` subquery thay vì `EXISTS`) | 1 |
| `LG_SEL_MELT060_SEND_FLOW_MAIL` | Gửi mail traceability report cho khách hàng | 4 (2 khối trùng lặp nhánh `'FLOW'`/`'MAIL'`, mỗi khối 2 điểm) |

Lấy full source từng procedure, áp dụng đúng 1 nguyên tắc sửa đã xác nhận ở bước 22 (bỏ điều kiện so khớp mã nguyên liệu, chỉ giữ `LOT_NO`). Verify lại bằng grep trên cả 6 file: toàn bộ điều kiện mã nguyên liệu đã bỏ đúng chỗ, toàn bộ điều kiện `LOT_NO` vẫn còn nguyên. Đã soạn đủ 5 file patch còn lại (`LG_SEL_MELT060_02_FILES_PATCHED_origin_join_fix.sql`, `LG_SEL_MELT021_02_PATCHED_origin_join_fix.sql`, `LG_SEL_MELT060_02_PATCHED_origin_join_fix.sql`, `LG_SEL_MO00010_PATCHED_origin_join_fix.sql`, `LG_SEL_MELT060_SEND_FLOW_MAIL_PATCHED_origin_join_fix.sql`) — cộng với file đã soạn ở bước 22, tổng 6 file sẵn sàng compile. Đáng chú ý `LG_SEL_MELT060_SEND_FLOW_MAIL` là procedure gửi email traceability report trực tiếp cho khách hàng — nếu không vá, email gửi ra sẽ thiếu/sai thông tin xuất xứ cho đúng những lot đã bị đổi mã.

---

**Các điểm còn cần user/phía nghiệp vụ xác nhận (tính tới hết 2026-09-08):**
1. Việc giữ nguyên `LOT_NO` sau khi đổi mã nguyên liệu có chấp nhận được không, hay cần 1 số tham chiếu lot khác.
2. Số dư sổ sách BRAMID bị "mồ côi" sau đó (không bao giờ được job này tiêu thụ nữa) có ảnh hưởng gì tới tồn kho/giá thành phía sau hay không.
3. Xác nhận phạm vi áp dụng vĩnh viễn, không giới hạn thời gian (toàn bộ 8 mã trong nhóm nguyên liệu 27) có đúng là hành vi mong muốn lâu dài hay không.
4. Có nên báo cáo riêng lỗi `OUTER JOIN` không liên quan phát hiện ở bước 5 thành 1 ticket khác hay không.
5. Nhánh MM hiện không có tác dụng cho DONGIL (không object nào gọi tới, và bản thân nguồn dữ liệu snapshot của nó cũng thiếu — xem bước 15) — có cần thiết phải duy trì patch trên nhánh này không, hay chỉ để phòng hờ cho site khác dùng chung procedure?
6. Có muốn bổ sung 1 lớp CASE WHEN phòng thủ tại bước build `TLG_LOT_TRACKING_TEMP` (khuyến nghị của Agent 2 ở bước 14, không bắt buộc) hay không?
7. **[MỚI]** Khoảng chênh lệch ~313K kg ở MAPPING (bước 21, nghi liên quan bug carry-over cũ đã biết) — có cần điều tra sâu thêm để định lượng chính xác tác động, hay chấp nhận đây là hệ quả của bug đã biết và xử lý chung khi báo cáo bug đó?
8. **[MỚI]** Sau khi compile 6 file patch ở bước 23 — có cần rà soát thêm các màn hình/procedure KHÁC (ngoài phạm vi `TLG_KB_COTTON_INCOME_D`+`TLG_LOT_TRACKING_GD_D`) có thể cũng bị ảnh hưởng bởi việc đổi mã nguyên liệu không, hay 6 procedure này đã là toàn bộ phạm vi?

---

## 2026-09-09

**1. Xác minh độc lập tuyên bố "100% US từ tháng 8" của BC (Le Tien) qua module Production/kho vận**

BC (Le Tien) phản hồi qua chat nội bộ, khẳng định "từ tháng 8 dùng 100% Origin US" (dẫn chứng Lot No 2608, cột Mat_item hiện USOT3137). Yêu cầu chứng minh độc lập, lần lượt siết chặt điều kiện: (1) không qua `TLG_LOT_TRACKING_GD_D`/`GD_M`, (2) không qua BẤT KỲ bảng nào tên `LOT_TRACKING`.

Dựng chain hoàn toàn thuộc module Production/kho vận: `TLG_WI_LINE_M.SLIP_NO` (= mã MIX_LOT thật; lưu ý cột `MIX_LOT_NO` trên cùng bảng luôn NULL/không dùng, đã verify trực tiếp) → `TLG_WI_LINE_M.TLG_ST_TRANSFER_REQ_M_PK` → `TLG_ST_TRANSFER_REQ_M/D` → `TLG_ST_TRANSFER_D/M` (chứng từ chuyển kho thật, `STATUS=3`) → `TLG_IN_WAREHOUSE` (lọc `PROCESS_TYPE='MIXED'`):

```sql
SELECT L.SLIP_NO MIX_LOT, M.TR_DATE, M.SLIP_NO TRANSFER_SLIP_NO,
       D.TR_ITEM_PK, I.ITEM_CODE, ROUND(D.TR_QTY,2) TR_QTY
FROM TLG_WI_LINE_M L
JOIN TLG_ST_TRANSFER_REQ_M R ON R.PK = L.TLG_ST_TRANSFER_REQ_M_PK AND R.DEL_IF = 0
JOIN TLG_ST_TRANSFER_REQ_D RD ON RD.TLG_ST_TRANSFER_REQ_M_PK = R.PK AND RD.DEL_IF = 0
JOIN TLG_ST_TRANSFER_D D ON D.TLG_ST_TRANSFER_REQ_D_PK = RD.PK AND D.DEL_IF = 0
JOIN TLG_ST_TRANSFER_M M ON M.PK = D.TLG_ST_TRANSFER_M_PK AND M.DEL_IF = 0
JOIN TLG_IN_WAREHOUSE W ON W.PK = M.IN_WH_PK AND W.DEL_IF = 0
JOIN TLG_IT_ITEM I ON I.PK = D.TR_ITEM_PK AND I.DEL_IF = 0
WHERE L.DEL_IF = 0 AND M.STATUS = 3 AND L.STATUS = 3 AND W.PROCESS_TYPE = 'MIXED'
```

Với 3 mixing lot mẫu (`MIXEDF1-260627-02`, `MIXEDF1-260701-01`, `MIXEDF1-260721-03`) → 28 dòng, cả BRAMID lẫn USOT3137 đều có chứng từ chuyển kho thật vào đúng ngày tạo lot. Mở rộng cho toàn bộ tháng 6-7/2026 (phát hiện và sửa lỗi ưu tiên `AND`/`OR` thiếu ngoặc ở lần chạy đầu — khiến điều kiện lọc `DEL_IF`/`STATUS`/`PROCESS_TYPE` bị bỏ qua cho 1 nhánh tháng): **USOT3137 165 mixing lot / 1,100,141.43kg; BRAMID 138 mixing lot / 1,091,235.31kg** — cả 2 đều có chứng từ chuyển kho thật, không phải dữ liệu ảo/lỗi. => Tuyên bố "100% US từ tháng 8" của BC không đúng ở tầng dữ liệu lịch sử.

**2. Chuẩn bị (không tự chạy) script xoá `TLG_LOT_TRACKING_GD_D`/`GD_M` phạm vi tháng 8+9/2026**

Theo yêu cầu user, soạn `results/delete_gd_m_gd_d_aug_sep_2026.sql`: hard DELETE `TLG_LOT_TRACKING_GD_D` (16,283 dòng) trước, rồi `TLG_LOT_TRACKING_GD_M` (1,413 dòng), lọc `WHERE TLG_GD_OUTGO_M.OUT_DATE BETWEEN '20260801' AND '20260930'`. Phạm vi (chỉ 2 tháng, không phải toàn bộ lịch sử) đã được xác nhận lại với user trước khi soạn.

**3. Verify lại OPEN balance sau guard-fix recovery (đã chạy ở 2026-09-08) — xác nhận allocation đã 100% USOT3137**

```sql
SELECT M.STD_YM, I.ITEM_CODE, M.STOCK_TYPE, ROUND(SUM(M.INPUT_QTY),2) IN_QTY
FROM TLG_LOT_TRACKING_MAT M JOIN TLG_IT_ITEM I ON I.PK = M.TLG_IT_ITEM_PK
WHERE M.DEL_IF = 0 AND I.ITEM_CODE IN ('BRAMID','USOT3137')
AND M.STD_YM IN ('202607','202608','202609') AND M.STOCK_TYPE = 'OPEN'
GROUP BY M.STD_YM, I.ITEM_CODE, M.STOCK_TYPE
```

Kết quả: 202607 (trước patch) vẫn còn BRAMID 69,272.57kg song song USOT3137 45,395.22kg; **202608 và 202609 (sau patch+recovery) chỉ còn USOT3137** (1,209,469.13kg và 2,294,111.82kg), **0 dòng BRAMID**. Re-verify diện rộng: `TLG_LOT_TRACKING_MAT`/`TLG_LOT_TRACKING_PROD` cho STD_YM 202608/202609, mọi `STOCK_TYPE`, tham chiếu tới BRAMID (PK=121) → **0 dòng**. Xác nhận patch CESM-418 đã hoạt động đúng như thiết kế ở tầng allocation.

**4. User chạy script xoá GD_M/GD_D — phát hiện giả định sai trong chính script, chỉ rebuild được 1 phần**

Sau khi user chạy script ở bước 2, số dòng còn lại chỉ **390/1,413 (GD_M)** và **600/16,283 (GD_D)** — không tự dựng lại đủ như ghi chú trong file. Đọc lại full source `LG_PRO_LOT_TRACKING_GD_M`/`LG_PRO_LOT_TRACKING_GD_D`:

- `LG_PRO_LOT_TRACKING_GD_M(P_DT_FROM, P_DT_TO, P_SEL_OPTION, P_TXT_SEARCHLIST, P_SEL_INTERFACE_YN, P_PARTNER, P_PARAM2, P_PARAM3, P_PARAM4, P_LANG, P_CRT_BY)` — chỉ `P_DT_FROM`/`P_DT_TO` thực sự ảnh hưởng logic lọc/insert (cursor chính lọc `M.OUT_DATE BETWEEN P_DT_FROM AND P_DT_TO`); 7/11 tham số còn lại không xuất hiện trong bất kỳ điều kiện WHERE/SELECT nào, chỉ `P_CRT_BY` được dùng làm giá trị audit `MOD_BY`. Cuối procedure gọi `LG_PRO_LOT_TRACKING_GD_D(P_DT_FROM, P_DT_TO)` cùng khoảng ngày.
- => Cơ chế "rebuild-on-open" **CHỈ dựng lại đúng khoảng ngày được truyền vào** khi UI gọi (theo đúng search của user trên màn hình), KHÔNG tự động phủ toàn bộ tháng như giả định ban đầu trong ghi chú của script xoá.

Soạn `results/force_full_rebuild_gd_m_gd_d_aug_sep_2026.sql`:
```sql
EXEC LG_PRO_LOT_TRACKING_GD_M('20260801','20260930', NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, 'RECOVERY-CESM418');
```
Verify an toàn chạy lại: `LG_PRO_LOT_TRACKING_GD_M` dùng pattern `SELECT INTO`/`EXCEPTION` (kiểm tra tồn tại trước khi insert từng node CUST/PO_NO/DELI/IT/LOT) — idempotent; `LG_PRO_LOT_TRACKING_GD_D` chỉ xử lý dòng GD_M còn `NVL(MAP_QTY,0) < NVL(QTY,0)` — dòng đã map đủ sẽ tự bỏ qua khi rerun.

**5. BC gửi screenshot melt070 (Lot 2608RN) — nhiều dòng Origin=none/0 files; điều tra root cause**

Đọc full source `LG_SEL_MELT070_02` — Origin/files tính từ CTE `TBL_LOT_INFO`: INNER JOIN `TLG_KB_COTTON_INCOME_D` (DI) → `TLG_PO_DOC_D0` (D1) → `TLG_PO_DOC_M` (M1) → `TES_FILE` (Z1), lọc bởi:
```sql
AND EXISTS (SELECT 1 FROM TLG_LOT_TRACKING_GD_D X1 WHERE X1.DEL_IF = 0
  AND EXISTS (SELECT 1 FROM TABLE(SPLIT(P_PARAM1,';')) WHERE COLUMN_VALUE = X1.TLG_LOT_TRACKING_GD_m_PK)
  AND DI.LOT_NO = TRIM(X1.MAT_LOT))
```
rồi OUTER JOIN `L.MAT_LOT(+) = TRIM(Z2.MAT_LOT)` vào query chính (đã có patch Option A từ 09-08: bỏ điều kiện so khớp mã nguyên liệu, chỉ giữ `LOT_NO`).

Trace trực tiếp 4 mat_lot bị "none" (`203/1076/26-01`, `303C/665/26-01`, `402A/660/26-01`, `402C/219/26-01`):
```sql
SELECT LOT_NO, TLG_PO_DOC_M_PK, TLG_IT_ITEM_PK, DEL_IF FROM TLG_KB_COTTON_INCOME_D
WHERE LOT_NO LIKE '203/1076/26%' OR LOT_NO LIKE '303C/665/26%' OR ...
```
→ cả 4 đều có `TLG_IT_ITEM_PK=121` (BRAMID) trong chứng từ gốc (mat_lot hiện đúng "USA/7 files" — `301A/1584/26-01` — có item=122, USOT3137 thật, không relabel). Chạy lại riêng chain INNER JOIN đầy đủ cho `LOT_NO='203/1076/26-01'`, KHÔNG qua điều kiện EXISTS/P_PARAM1:
```sql
-- (chain DI->D1->M1->Z1, WHERE DI.LOT_NO = '203/1076/26-01', không có EXISTS)
```
→ **vẫn ra kết quả**: `ORIGIN='BRA'`, tổng **7 files** (TYPE01/02/03/05/06/07 mỗi loại 1 file, TYPE07 2 file). Đối chiếu `TLG_LOT_TRACKING_GD_D.MAT_LOT` cho lot VP2608RN, mat_lot này: giá trị lưu đúng `"203/1076/26-01"` (14 ký tự, khớp tuyệt đối với `TLG_KB_COTTON_INCOME_D.LOT_NO`, không lệch định dạng/khoảng trắng).

=> **Kết luận: dữ liệu gốc (`TLG_KB_COTTON_INCOME_D`+`TLG_PO_DOC_M/D0`+`TES_FILE`) hoàn toàn nguyên vẹn, KHÔNG bị mất/hỏng bởi bước xoá+rebuild GD_M/GD_D.** "None/0 files" là 1 bug RIÊNG trong chính điều kiện `EXISTS`/`P_PARAM1` của `LG_SEL_MELT070_02` (giới hạn theo danh sách GD_M.PK đang được search trên UI) — chưa xác định 100% cơ chế chính xác gây loại nhầm (cần `P_PARAM1` thật từ browser để verify tiếp), nhưng chắc chắn không phải mất dữ liệu.

**6. BC phản hồi bằng chứng kho vật lý thật (màn hình IV0401 W/H Stock Checking, `lg_sel_bisc00020`)**

BC dẫn chứng: `exec lg_sel_bisc00020('20260722','20260831','10','428','','','N','ENG','Y',:p_rtn_value)`, khẳng định BRAMID đã hết sạch tại kho từ 22/7/2026. Đọc full source `lg_sel_bisc00020` — dùng `TLG_SA_STOCK_CLOSING_M/D` (bảng cân đối đóng kỳ thật) + `TLG_IN_STOCKTR` (giao dịch nhập/xuất kho thật), **hoàn toàn không qua bất kỳ bảng LOT_TRACKING nào**.

Tính lại độc lập cho warehouse PK=428 (M011-COTTON W/H Fac1, khớp `WH_TYPE='10'`):
```sql
-- BEGIN_QTY(22/7) = closing_end_qty(lan dong ky gan nhat truoc 22/7)
--                 + SUM(IN_QTY-OUT_QTY tu TLG_IN_STOCKTR, TR_DATE giua lan dong ky va 22/7)
```
→ khớp CHÍNH XÁC 100% với số liệu trên screenshot của BC (Begin Qty USOT3137 = 1,974,477.71, Total In = 713,165.00, Total Out = 1,444,729.04). Riêng BRAMID tại M011: Begin Qty(22/7) = 1,649,051.5 + (-1,649,051.5) = **0.00kg**; Total In/Out (22/7→hiện tại) = **0/0**. Mở rộng kiểm tra cả kho M001-IQC Cotton W/H Fac1 (kho nguyên liệu cotton thứ 2 duy nhất còn lại): cũng **~0 (âm nhẹ, không đủ dùng)**, 0 giao dịch sau 22/7. => **BC hoàn toàn đúng về hiện trạng kho vật lý.**

**7. Đối chiếu lại với chain Production/kho vận (bước 1) để tìm chính xác ngày cutoff**

```sql
SELECT L.SLIP_NO MIX_LOT, MAX(M.TR_DATE) LAST_TR_DATE, ROUND(SUM(D.TR_QTY),2) TONG_KG_BRAMID
FROM TLG_WI_LINE_M L JOIN TLG_ST_TRANSFER_REQ_M R ON ... JOIN TLG_IN_WAREHOUSE W ON ...
WHERE ... AND W.PROCESS_TYPE='MIXED' AND D.TR_ITEM_PK=121
GROUP BY L.SLIP_NO ORDER BY LAST_TR_DATE DESC
```
→ Lô mix BRAMID cuối cùng thật sự là **21/7/2026** (`MIXEDF1-260721-01/02/03`, tổng ~21,730kg) — không lô nào sau đó dùng BRAMID. Khớp hoàn toàn với kho báo hết từ 22/7 (bước 6). **Chốt mốc: từ 22/7/2026 trở đi, mọi lô mix genuinely 100% USOT3137 thật; các lô mix từ 21/7/2026 trở về trước có pha BRAMID thật — sự thật vật lý cố định, không thể đảo ngược bằng thao tác hệ thống.**

**8. Tổng hợp giải thích nghiệp vụ cho BC — phân biệt thời điểm MIX vs thời điểm DELIVERY**

Kết luận cuối, đã trao đổi lại với BC: dữ liệu "còn Brazil ở tháng 8" phản ánh đúng 2 thời điểm khác nhau trong 1 chuỗi — (a) MIX (trộn nguyên liệu, xảy ra 1 lần, dừng hẳn từ 21/7) khác với (b) DELI (giao hàng thành phẩm cho khách, rải rác nhiều tuần sau, có thể tới tháng 8-9). Lô hàng giao trong tháng 8 nếu được sản xuất từ mẻ đã mix bằng BRAMID trước 21/7 thì đúng là còn Brazil thật — không phải mix mới, không phải lỗi dữ liệu.

Đồng thời làm rõ với BC 1 hệ luỵ quan trọng cho quyết định nghiệp vụ tiếp theo: item hiện "USOT3137" cho các lô này (kể cả sau khi item-code được chuẩn hoá bởi CESM-418) **không phản ánh đúng thực tế đã dùng nguyên liệu gì** — chứng từ mua hàng gốc của các lô này (đã verify ở bước 5) vẫn là chứng từ Brazil thật, có file đầy đủ. Nếu muốn dữ liệu đúng 100% thực tế, các lô mix trước 22/7 phải hiển thị lại đúng là Brazil (cả item lẫn Origin); còn các lô mix từ 22/7 trở đi (bước 7) thì chắc chắn genuinely 100% USOT3137, không cần chỉnh sửa gì.

---

**Các điểm còn cần user/phía nghiệp vụ xác nhận (bổ sung 2026-09-09):**
9. **[MỚI]** Có nên ghi đè hiển thị Origin=USA (patch V2 đã soạn, `LG_SEL_MELT070_02`/`MELT021_02`/`MELT060_02`) cho các lô đã xác nhận có chứng từ Brazil THẬT đính kèm đầy đủ hay không — đây là quyết định compliance/chứng nhận xuất xứ, không chỉ là lựa chọn hiển thị. `LG_SEL_MELT060_SEND_FLOW_MAIL` (gửi email traceability trực tiếp cho khách hàng) vẫn đang giữ nguyên chưa compile bản V2 (`_PENDING_confirmation.sql`), chờ đúng quyết định này.
10. **[MỚI]** Có nên sửa lại để item/Origin của các lô mix TRƯỚC 22/7/2026 hiển thị đúng lại là Brazil (thay vì USOT3137 đã chuẩn hoá) để khớp đúng thực tế vật lý — hay giữ nguyên chuẩn hoá 100% USOT3137 như CESM-418 yêu cầu ban đầu (chấp nhận đây là nhãn kế toán nội bộ, không phải xuất xứ vật lý thật)?
11. **[MỚI]** Bug riêng trong điều kiện `EXISTS`/`P_PARAM1` của `LG_SEL_MELT070_02` (bước 5, làm ẩn Origin/files dù dữ liệu gốc còn nguyên) — cần điều tra sâu thêm cơ chế chính xác và vá lại hay không (độc lập với quyết định nghiệp vụ ở mục 9/10)?
12. **[MỚI]** Đã xác nhận scope xoá cho `results/delete_gd_m_gd_d_aug_sep_2026.sql` là chỉ tháng 8+9/2026 — script này và `results/force_full_rebuild_gd_m_gd_d_aug_sep_2026.sql` (bước 4) vẫn đang chờ user tự chạy qua Toad.
