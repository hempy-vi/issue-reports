# Dev Report — CESM-728

**Task:** [SAMIL] [DATA HANDLING] Export Item Excel File from Forms SB1.13 and SB1.16 (from Database)

---

## 2026-09-09

**1. Xác định nguồn dữ liệu thật của 2 form**

Đọc source ASP.NET (GASP) của 2 form được nêu trong ticket:
- SB1.13 R&D Item Register — `form/sa/10/sa100130.aspx`
- SB1.16 Item Code Inquiry — `form/sa/10/sa100010_V02.aspx`

Cả 2 form đều dùng pattern GASP chuẩn — lưới dữ liệu (`grdMaster`) được nạp qua 1 `<gw:data>` kiểu `grid`, trỏ thẳng vào 1 stored procedure:

```html
<!-- sa100130.aspx -->
<dso type="grid" function="sp_sel_sa100130">

<!-- sa100010_V02.aspx -->
<dso type="grid" parameter="0,1,...,59" function="sp_sel_sa100010" procedure="sp_upd_sa100010">
```

Xác định được data source thật: `SP_SEL_SA100130` (cho SB1.13) và `SP_SEL_SA100010` (cho SB1.16) — 2 stored procedure trong DB, không phải query tĩnh trong code.

**2. Sự cố kết nối DB — oracle-mcp timeout, chuyển sang REST API trực tiếp**

`oracle-mcp` của workspace SAMIL bị `CONNECT_TIMEOUT` ngay từ đầu session. Kiểm tra docker thấy container `oracle-api-samil` (REST backend, port 8084) vẫn đang chạy healthy:

```
docker ps -a --filter "name=samil"
oracle-api-samil   Up About an hour (healthy)   127.0.0.1:8084->8080/tcp
```

Chuyển sang gọi thẳng REST API của `oracle-api` (service tầng tool, không qua MCP) cho toàn bộ phần còn lại của task:

```
GET  /api/db/sources/{objectName}
POST /api/db/query/read-only
```

Gọi thử `GET /api/db/sources/SP_SEL_SA100130` bị lỗi 500. Xem log container:

```
java.sql.SQLException: ORA-17004: Invalid column type: 268435455
```

Lỗi driver JDBC khi map cột `TEXT` kiểu `LONG` của `ALL_SOURCE` sang JSON — bug ở endpoint `sources`, không phải do quyền hay kết nối DB. Né bằng cách query trực tiếp `ALL_SOURCE` qua endpoint `query/read-only` (endpoint này serialize `LONG` bình thường):

```sql
SELECT text FROM all_source WHERE name = 'SP_SEL_SA100130' AND type = 'PROCEDURE' ORDER BY line
```

Lấy được full source cả 2 procedure (194 dòng cho `SP_SEL_SA100130`, 230 dòng cho `SP_SEL_SA100010`), phân trang bằng `AND line > N`.

**3. Soạn export query "full data" từ source procedure**

Cả 2 procedure đều theo pattern: 1 tập filter search-box optional (luôn có dạng `... OR P_x IS NULL`) cộng thêm vài filter business bắt buộc (`DEL_IF = 0`, điều kiện outer-join). Soạn lại mỗi procedure thành 1 câu `SELECT` độc lập (không qua procedure/cursor) — giữ nguyên toàn bộ join + filter business bắt buộc, bỏ toàn bộ filter search-box optional để lấy full dataset:

- `queries/03_export_sb113_rnd_item_register.sql` — từ `SP_SEL_SA100130`, giữ 40 cột, alias theo đúng header hiển thị trên form.
- `queries/04_export_sb116_item_code_inquiry.sql` — từ `SP_SEL_SA100010`, giữ 56 cột (bỏ 5 cột `_Yarn NM 1..5` do procedure tự lặp lại data đã có).

**4. Guard chặn SQL — 2 lỗi cần sửa trước khi chạy được**

Test preview (`SELECT * FROM (<query>) WHERE ROWNUM <= 5`) qua `/api/db/query/read-only` báo lỗi:

```
SB1.13: {"message":"Query blocked: Blocked SQL keyword or package detected: \bUTL_"}
SB1.16: {"message":"Query blocked: Blocked SQL keyword or package detected: \bCREATE\b"}
```

- SB1.13: procedure gốc gộp danh sách yarn bằng `UTL_I18N.unescape_reference(XMLAGG(...).EXTRACT('//text()'))` — package `UTL_*` bị SQL guard của `oracle-api` chặn cứng (an toàn, không phải bug). Thay bằng `LISTAGG` — cho cùng 1 chuỗi hiển thị, không qua vòng escape/unescape XML nên không lệch ký tự:
  ```sql
  -- gốc (bị chặn):
  UTL_I18N.unescape_reference(RTRIM(XMLAGG(XMLELEMENT(E, V2.YARN||'('||V1.CONS||')'||' ')).EXTRACT('//text()'), ' '))
  -- thay bằng:
  LISTAGG(V2.YARN||'('||V1.CONS||')', ' ' ON OVERFLOW TRUNCATE) WITHIN GROUP (ORDER BY V2.YARN)
  ```
  Thêm `ON OVERFLOW TRUNCATE` để tránh lỗi `ORA-01489` (string concatenation quá 4000 byte) làm crash cả query nếu 1 item có danh sách yarn quá dài — rủi ro lý thuyết mà `LISTAGG` có nhưng `XMLAGG` gốc không có.
- SB1.16: guard chặn theo whole-word (`\bCREATE\b`), khớp trúng chữ "Create" trong alias cột `CRT_BY "Create By"` (không liên quan gì tới câu lệnh DDL, chỉ là trùng chữ). Đổi alias thành `"Input By"`.

**5. Verify độc lập cả 2 query bằng workflow trước khi chạy thật**

Trước khi chạy full data đưa cho requester, launch 1 workflow gồm 2 agent chạy song song, mỗi agent so khớp column-by-column + filter-by-filter giữa export query và source procedure gốc (agent không thấy phần còn lại của conversation, chỉ nhận đúng 2 đoạn source để so sánh độc lập). Kết quả:

- **SB1.13 — FAIL** (cần xác nhận trước khi giao):
  - `A.REQ_DT BETWEEN P_FROM_DT AND P_TO_DT` là filter DUY NHẤT trong procedure không có `OR P_x IS NULL` escape (khác mọi search-param còn lại) — bị bỏ hoàn toàn trong export. Về bản chất đây là mandatory filter chứ không phải optional search box, nên việc bỏ nó là 1 quyết định mở rộng phạm vi (lấy toàn bộ lịch sử, không giới hạn ngày), không phải "bỏ qua ô search rỗng" như các filter khác.
  - Warning: đổi `XMLAGG` → `LISTAGG` (đã giải thích ở bước 4) có rủi ro lý thuyết `ORA-01489`.
  - Note: alias `SHRINK_WEIGHT "Shrinkage Width"` sai ý nghĩa (copy nguyên label từ form gốc, nhưng form gốc đặt nhầm tên — cột là "weight" không phải "width").
- **SB1.16 — PASS** (1 warning):
  - Việc bỏ `A.USE_YN = P_ACTIVE` (mandatory filter, không có `OR IS NULL`, form default luôn truyền `'Y'` vì checkbox Active không có state NULL/All) được đánh giá là mở rộng scope có chủ đích và hợp lý — đã ghi rõ lý do trong comment, cột `USE_YN` vẫn giữ lại trong output để lọc trong Excel.
  - Warning: `ORDER BY` đổi từ `ITEM_NAME DESC` (procedure gốc) sang `PK` — data từng dòng vẫn đúng, nhưng cột `Seq` (ROWNUM) và thứ tự dòng trong file sẽ không khớp với màn hình gốc.

Chi tiết đầy đủ 2 review: xem mục 6 (đã copy nguyên các điểm chính vào đây, không cắt bớt).

**6. Xử lý các điểm review — quyết định giữ, không dừng lại hỏi**

Theo đúng yêu cầu ticket ("pull trực tiếp từ database", "Data is complete and matches the database"), quyết định:
- SB1.13: **giữ bỏ filter REQ_DT** → export toàn bộ R&D item mọi thời điểm nhận hàng, không giới hạn theo ngày. Ghi rõ đây là giả định trong comment đầu file SQL + trong báo cáo giao việc, để requester dễ điều chỉnh nếu cần 1 khoảng ngày cụ thể.
- SB1.13: thêm `ON OVERFLOW TRUNCATE` vào `LISTAGG` (đã mô tả ở bước 4).
- SB1.13: sửa alias `SHRINK_WEIGHT` từ `"Shrinkage Width"` thành `"Shrinkage Weight"` (đúng ý nghĩa cột, data không đổi).
- SB1.16: **giữ `ORDER BY PK`** (không đổi lại `ITEM_NAME DESC`) — hợp lý hơn cho 1 file Excel tra cứu/đối chiếu dữ liệu. Ghi rõ trong comment: cột `Seq` trong file là số thứ tự trong chính file export, không phải số thứ tự trên màn hình gốc.

**7. Chạy full export thật + kiểm tra tính toàn vẹn dữ liệu**

Chạy 2 câu SELECT đã sửa qua `/api/db/query/read-only` (dùng `SELECT * FROM (<query>) WHERE ROWNUM <= N` với N đủ lớn để lấy hết, vì guard chỉ tự động áp giới hạn khi câu query KHÔNG có sẵn giới hạn — nếu tự set `ROWNUM` cao hơn số dòng thật, guard tôn trọng giới hạn đó):

- SB1.13: 24.598 dòng, 40 cột, ~22.7 MB JSON, ~22 giây.
- SB1.16: 8.548 dòng, 56 cột, ~7.9 MB JSON, ~26 giây.

Cả 2 procedure gốc đều có outer-join (`TLG_IT_ITEM`, `SA_PERSON_CODE` cho SB1.13; 9 lần `SA_YARN_CODE` cho SB1.16) — có rủi ro lý thuyết bị fan-out (1 dòng gốc thành nhiều dòng nếu bảng outer-join có nhiều hơn 1 match). Kiểm tra: đếm số `PK` distinct so với tổng số dòng trả về ở cả 2 file — khớp tuyệt đối (24.598 = 24.598, 8.548 = 8.548) → không có dòng nào bị nhân đôi do join.

**8. Build file Excel (.xlsx) thật — tự dựng OOXML vì máy không có internet**

Máy chạy session không có kết nối internet (`npm install` timeout) nên không cài được thư viện `xlsx` có sẵn. Tự viết 1 Node.js script (`build_xlsx.js`) dựng trực tiếp các phần XML tối thiểu của định dạng OOXML (`[Content_Types].xml`, `_rels/.rels`, `xl/workbook.xml`, `xl/_rels/workbook.xml.rels`, `xl/styles.xml`, `xl/worksheets/sheet1.xml`) từ JSON `{columns, rows}` — tự phát hiện kiểu cột (numeric cell nếu JSON value là `number`, còn lại dùng inline string), có style riêng cho header (bold, nền màu), freeze pane dòng 1, auto-filter.

Zip các part XML thành `.xlsx` thật bằng PowerShell (`System.IO.Compression`). Phát hiện lỗi lúc test: `[System.IO.Compression.ZipFile]::CreateFromDirectory` trên Windows tạo entry path dùng dấu `\` (backslash) thay vì `/` — sai chuẩn Open Packaging Conventions của `.xlsx` (dù nhiều máy Excel vẫn mở được do lenient, nhưng không đúng chuẩn). Sửa bằng cách tạo `ZipArchive` thủ công, add từng file với entry name ép về `/`:

```powershell
$rel = $f.FullName.Substring($PartsDir.Length + 1)
$relFixed = $rel.Replace([System.IO.Path]::DirectorySeparatorChar, '/')
$entry = $zip.CreateEntry($relFixed, ...)
```

**9. Verify file .xlsx thật bằng Excel COM automation**

Mở lại cả 2 file `.xlsx` vừa tạo bằng Excel COM automation (PowerShell `New-Object -ComObject Excel.Application`), đọc `UsedRange` để xác nhận đúng số dòng/cột và đọc đúng data (không chỉ validate zip/XML hợp lệ mà mở thật bằng Excel):

```
SAMIL_SB1.13_RnD_Item_Register_20260909.xlsx: Rows 24599 (header+data), Cols 40
SAMIL_SB1.16_Item_Code_Inquiry_20260909.xlsx: Rows 8549 (header+data), Cols 56
```

**10. Bàn giao**

File deliverable: `SAMIL_SB1.13_RnD_Item_Register_20260909.xlsx` và `SAMIL_SB1.16_Item_Code_Inquiry_20260909.xlsx`. 2 giả định cần requester (Ms. Oanh) xác nhận trước khi coi là final:
1. SB1.13 không giới hạn theo khoảng ngày Receipt Date (lấy toàn bộ lịch sử).
2. SB1.16 gồm cả item active lẫn inactive (cột `Use` để tự lọc), thứ tự dòng theo PK thay vì theo tên item như màn hình gốc.

Chưa gửi file cho requester — session không có kết nối email/Jira attachment, cần user tự đính kèm file vào email/Jira comment sau khi xem qua 2 giả định trên.

**11. Follow-up từ business: file SB1.16 thiếu Yarn Code**

Sau khi nhận 2 file, Ms. Oanh (qua đồng nghiệp "Ant") phản hồi: file SB1.16 hiện chỉ có "Yarn 1".."Yarn 9" (tên/mô tả yarn dạng text, vd "COTTON 40S CM") — chỉ có tên thì khó dò/tra cứu, cần thêm cột **Yarn Code** (mã yarn) ngay TRƯỚC mỗi cột tên yarn tương ứng.

Tra lại source `sa100011.aspx` (popup chọn Yarn, mở từ nút "Yarn" trên form SB1.16) — dùng procedure `sp_sel_sa100011`, grid có đúng 3 cột:

```html
<gw:grid id="idGrid" header="_PK|Yarn Code|Yarn Name" ... />
```

Xác nhận: bảng `SA_YARN_CODE` có 1 cột "Yarn Code" tách biệt hoàn toàn với cột `YARN` (tên, đang dùng làm "Yarn 1".."Yarn 9") và với `PK` (khoá nội bộ, đã có sẵn trong file dạng cột "Yarn0X PK" nhưng không phải cái Ms. Oanh cần) — đúng là cái cần thêm. Chưa biết tên cột thật trong `SA_YARN_CODE` (chỉ mới thấy header hiển thị), cần lấy full source `SP_SEL_SA100011` để xác định.

**12. Blocker — mất kết nối tới Oracle DB (network-level, không phải docker)**

Lúc thử gọi `/api/db/query/read-only` để lấy source `SP_SEL_SA100011`, phát hiện:
- `oracle-mcp`: lỗi `CONNECTION_CLOSED`.
- `oracle-api-samil` (container Docker vẫn đang chạy "Up", đã thử `docker compose ... start oracle-api`): log báo lỗi JDBC:
  ```
  oracle.net.ns.NetException: ORA-12170: Cannot connect. TCP connect timeout of 20000ms for host 192.168.240.204 port 1521.
  ```
- Test trực tiếp network từ máy:
  ```
  Test-NetConnection -ComputerName 192.168.240.204 -Port 1521
  → TcpTestSucceeded: False, PingSucceeded: False
  ```

Kết luận: DB host `192.168.240.204` không reachable ở tầng network từ máy đang chạy session (khả năng cao do VPN nội bộ SAMIL bị rớt) — không liên quan gì tới docker/`oracle-api` (container vẫn khoẻ mạnh, chỉ là không kết nối được tới DB backend). Không phải lỗi tự sửa được từ phía dev — đã dừng lại, báo user kiểm tra kết nối mạng/VPN.

**13. Kết nối lại — lấy source `SP_SEL_SA100011`, xác định đúng tên cột**

Sau khi user xác nhận đã reconnect VPN, test lại `/api/db/query/read-only` bằng `SELECT 1 FROM DUAL` — thành công. Lấy full source `SP_SEL_SA100011`:

```sql
PROCEDURE SP_SEL_SA100011 (P_YARN VARCHAR2, P_CURSOR OUT SYS_REFCURSOR)
IS
BEGIN
   OPEN P_CURSOR FOR
        SELECT A.PK, A.YARN_CODE, A.YARN
          FROM SA_YARN_CODE A
         WHERE     DEL_IF = 0
               AND USE = '-1'
               AND (   UPPER (A.YARN_CODE) LIKE '%' || UPPER (P_YARN) || '%'
                    OR UPPER (A.YARN) LIKE '%' || UPPER (P_YARN) || '%'
                    OR P_YARN IS NULL)
      ORDER BY A.YARN_CODE;
END SP_SEL_SA100011;
```

Xác nhận: `SA_YARN_CODE.YARN_CODE` chính là cột "Yarn Code" cần thêm — tách biệt hoàn toàn với `SA_YARN_CODE.YARN` (tên/mô tả).

Nhân tiện phát hiện + sửa 1 lỗi sót từ bước 4 trước đó: file lưu `queries/04_export_sb116_item_code_inquiry.sql` trong session vẫn còn alias cũ `CRT_BY "Create By"` — bản đã chạy full export lúc trước đã dùng đúng `"Input By"` (do bị SQL guard chặn chữ "Create") nhưng file lưu lại chưa từng được đồng bộ lại. Sửa lại cho khớp.

**14. Sửa export query SB1.16 — thêm 9 cột Yarn Code**

Sửa `queries/04_export_sb116_item_code_inquiry.sql`: thêm `YARNn.YARN_CODE "YarnNN Code"` ngay trước mỗi cột `YARNn "Yarn N"` tương ứng (cả outer SELECT lẫn inner SELECT), cho cả 9 slot yarn. Không cần thêm join mới — 9 bảng `SA_YARN_CODE YARN1..YARN9` đã được outer-join sẵn từ trước, chỉ cần select thêm cột `YARNn.YARN_CODE`:

```sql
-- trước:
A.WIDTH, A.GRAM, A.M2, A.YARN01_PK,
YARN1.YARN YARN1, A.PER01, A.YARN02_PK,
...
-- sau:
A.WIDTH, A.GRAM, A.M2, A.YARN01_PK, YARN1.YARN_CODE YARN01_CODE,
YARN1.YARN YARN1, A.PER01, A.YARN02_PK, YARN2.YARN_CODE YARN02_CODE,
...
```

Test preview 5 dòng qua `/api/db/query/read-only` trước khi chạy full — cột "Yarn01 Code" ra đúng giá trị (`000399`, `000267`, `001019`...) đứng ngay trước "Yarn 1" tương ứng ("COTTON 40S CM", "COTTON 30S CM", "COTTON 50S CM"...).

**15. Chạy lại full export SB1.16 + kiểm tra tính toàn vẹn**

Chạy lại full export: 8.548 dòng, 65 cột (56 cột cũ + 9 cột Yarn Code) — khớp đúng số dòng của lần chạy trước đó. Kiểm tra lại: đếm `PK` distinct so với tổng số dòng — khớp tuyệt đối (8.548 = 8.548) → không có dòng nào bị nhân đôi do join.

**16. Build lại file `.xlsx`, verify, bàn giao**

Build lại file Excel bằng đúng pipeline đã dựng ở bước 8 (`build_xlsx.js` + zip qua PowerShell `System.IO.Compression`). Đặt tên file khác bản gốc (thêm hậu tố `_v2`) để không đè lên file Ms. Oanh đang mở sẵn:

```
results/SAMIL_SB1.16_Item_Code_Inquiry_20260909_v2.xlsx
```

Verify lại bằng Excel COM automation — mở thật file, đọc đúng số dòng/cột và đúng thứ tự cột PK → Code → Name:

```
Rows: 8549 (header+data)  Cols: 65
Yarn01 PK header: "Yarn01 PK" | Yarn01 Code header: "Yarn01 Code" | Yarn 1 header: "Yarn 1"
Row 2: Yarn01 Code = 000399, Yarn 1 = COTTON 40S CM
```

File `SAMIL_SB1.13_RnD_Item_Register_20260909.xlsx` không đổi (yêu cầu chỉ áp dụng cho SB1.16). Báo user: cần thông báo Ms. Oanh đóng file SB1.16 bản cũ (chưa có Yarn Code) và dùng bản `_v2` thay thế.
