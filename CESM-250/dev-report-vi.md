# Dev Report — CESM-250

**Task:** WePOP Issue Barcode Label (Stock In Request): Label Modify

---

## 2026-09-11

**1. Xác định control cần sửa trong report IBL530**
Yêu cầu: (1) PO No hiển thị to hơn, (2) đổi caption 'Roll' thành 'LOCATION', trên report `IBL530` (DevExpress XtraReport, thuộc project `PROJECT\IBL LABEL`). File `IBL530.cs` chỉ chứa code-behind (`setDataLabel`, `Detail_BeforePrint`), không liên quan UI — layout thật nằm trong `IBL530.Designer.cs` (`InitializeComponent()`).

Xác định đúng control qua tên biến và binding:
- `xrLabel6` = caption 'PO No' (không đổi).
- `xrLabel7` = control hiển thị giá trị PO No thật (binding `[Table1].[Column2]`, font `Segoe UI Semibold, 10F`) — đây là control cần to hơn.
- `xrLabel11` = caption 'Roll' (trong `xrPanel6`) — đổi text thành 'LOCATION'.
- `xrLabel16` = control hiển thị giá trị Roll thật (binding `[Table1].[Column6]`) — giữ nguyên.

**2. Kiểm tra cơ chế auto-scale font trước khi sửa**
Đọc `WEPOP MVC\Controllers\WePanel.cs`, hàm `BeforePrintDetail`/`BeforePrintPanel`/`BeforePrintControl` và `AutoscaleControlText`:

```csharp
private static void AutoscaleControlText(XRLabel label)
{
    label.Text = Regex.Replace(label.Text, @"\r\n?|\n", " ");
    while (MeasureTextWidth(label) > (label.SizeF.Width - padding.Left - padding.Right))
    {
        label.Font = new DXFont(label.Font.Name, label.Font.Size - 0.1f, label.Font.Style);
    }
}
```

Các `XRLabel` không có `Tag = "noResizeFont"` sẽ tự co font (chỉ theo chiều rộng, không bao giờ phóng to) để vừa khung khi in. Do đó tăng font gốc trong Designer là hướng đúng — nếu nội dung dài, code tự co lại, không vỡ layout.

**3. Áp dụng thay đổi tại `IBL530.Designer.cs`**
```csharp
// truoc
this.xrLabel7.Font = new DevExpress.Drawing.DXFont("Segoe UI Semibold", 10F);
...
this.xrLabel11.Text = "Roll";

// sau
this.xrLabel7.Font = new DevExpress.Drawing.DXFont("Segoe UI Semibold", 14F);
...
this.xrLabel11.Text = "LOCATION";
```

**4. Build kiểm tra**
`dotnet build "IBL LABEL.csproj"` — dependency `WEPOP_MVC` build sạch (2 thay đổi trên biên dịch OK). Project `IBL LABEL.csproj` báo lỗi MSB3822/MSB3823 (Non-string resources / GenerateResourceUsePreserializedResources) từ `IBL550.resx` (ảnh PNG nhúng của report khác) — lỗi môi trường build (dotnet SDK 10 CLI với `.resx` kiểu cũ target .NET Framework), không liên quan tới 2 thay đổi trên IBL530, pre-existing. Cần build lại bằng Visual Studio/MSBuild đầy đủ để verify cuối.

**5. Điều tra ký tự tab thừa trong PO No (phát sinh khi kiểm tra bản in thật)**
User báo `exec LG_RPT_IBL530('2831358,2831358','vng-241',:p_rtn_cur)` trả về giá trị PO No dính thêm 1 ký tự tab ở cuối.

Lấy source qua MCP `get_oracle_source('LG_RPT_IBL530')`: proc đứng độc lập, owner `SWPROD`, dòng lấy PO No:
```sql
trim(A.PURCHASE_PO_NO) REF_PO_NO,
```

Kiểm tra dữ liệu gốc:
```sql
SELECT A.PK, A.PURCHASE_PO_NO,
       LENGTH(A.PURCHASE_PO_NO) LEN_RAW,
       LENGTH(TRIM(A.PURCHASE_PO_NO)) LEN_TRIM,
       INSTR(A.PURCHASE_PO_NO, CHR(9)) TAB_POS,
       DUMP(SUBSTR(A.PURCHASE_PO_NO, -1)) LAST_CHAR_DUMP
FROM TLG_PA_PACKAGES A
WHERE A.PURCHASE_PO_NO LIKE '%2606-0007%';
```
Kết quả: PK=2831358 (và nhiều PK cùng batch nhà cung cấp AD) có `PURCHASE_PO_NO = '260609-SW-AD-2606-0007' + CHR(9)`. `TAB_POS=23` (dính ngay cuối chuỗi), `LEN_RAW = LEN_TRIM = 23` — chứng minh `TRIM()` mặc định của Oracle **không** loại được ký tự tab (chỉ loại khoảng trắng), nên tab tồn tại ngay từ dữ liệu gốc, đi thẳng qua proc ra output.

**6. Xác nhận hướng fix bằng readonly query**
```sql
SELECT trim(A.PURCHASE_PO_NO) AS OLD_TRIM,
       trim(replace(A.PURCHASE_PO_NO, chr(9), '')) AS NEW_TRIM,
       LENGTH(trim(A.PURCHASE_PO_NO)) OLD_LEN,
       LENGTH(trim(replace(A.PURCHASE_PO_NO, chr(9), ''))) NEW_LEN
FROM TLG_PA_PACKAGES A WHERE A.PK = 2831358;
```
Kết quả: `OLD_TRIM` = `'260609-SW-AD-2606-0007<TAB>'` (23 ký tự) → `NEW_TRIM` = `'260609-SW-AD-2606-0007'` (22 ký tự, hết tab). Xác nhận `replace(A.PURCHASE_PO_NO, chr(9), '')` trước khi `trim()` xử lý đúng.

**7. Soạn sẵn script fix cho stored procedure**
Chỉ đổi đúng 1 dòng `REF_PO_NO` trong `LG_RPT_IBL530`:
```sql
CREATE OR REPLACE PROCEDURE SWPROD.LG_RPT_IBL530 (P_PKS_EXECKEY       VARCHAR2,
                                           P_CRT_BY            VARCHAR2,
                                           P_RTN_CUR       OUT SYS_REFCURSOR)
IS
BEGIN
    OPEN P_RTN_CUR FOR
          SELECT POP_LABEL ('IBL530'),
                 nvl(C.SHORT_NM, C.PARTNER_NAME),
                 trim(replace(A.PURCHASE_PO_NO, chr(9), '')) REF_PO_NO,
                 B.ITEM_CODE || ' - ' || B.ITEM_NAME ITEM,
                 B.SPEC04_NM LENGTH_IF,
                 B.SPEC03_NM SIZE_IF,
                 A.ROLL_NO,
                 trim(A.LOT_NO) LOT_NO,
                 A.QTY || ' ' || B.UOM,
                 A.ITEM_BC
            FROM TLG_PA_PACKAGES A, TLG_IT_ITEM B, TCO_BUSPARTNER C, (SELECT distinct COLUMN_VALUE     PK
                                FROM TABLE (
                                         SPLIT (
                                             REPLACE (P_PKS_EXECKEY,
                                                      CHR (39),
                                                      ''),
                                             ','))
                               WHERE COLUMN_VALUE IS NOT NULL) z
           WHERE     A.DEL_IF = 0
                 AND B.DEL_IF = 0
                 AND C.DEL_IF(+) = 0
                 AND B.PK = A.TLG_IT_ITEM_PK
                 AND C.PK(+) = A.TCO_BUSPARTNER_PK
                 AND (   A.EXECKEY = z.PK
                      OR A.PK = z.PK)
        ORDER BY A.PK DESC;
END;
/
```
MCP DB tool chỉ có quyền read-only, chưa chạy DDL này — script được bàn giao để chạy trên môi trường có quyền `CREATE OR REPLACE PROCEDURE` phù hợp.
