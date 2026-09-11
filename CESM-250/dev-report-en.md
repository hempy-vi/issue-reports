# Dev Report — CESM-250

**Task:** WePOP Issue Barcode Label (Stock In Request): Label Modify

---

## 2026-09-11

**1. Identify the controls to modify in report IBL530**
Request: (1) make PO No display larger, (2) rename the 'Roll' caption to 'LOCATION', on report `IBL530` (DevExpress XtraReport, under project `PROJECT\IBL LABEL`). `IBL530.cs` only contains code-behind (`setDataLabel`, `Detail_BeforePrint`) and has no UI content — the actual layout lives in `IBL530.Designer.cs` (`InitializeComponent()`).

Identified the correct controls by variable name and data binding:
- `xrLabel6` = 'PO No' caption (not changed).
- `xrLabel7` = the control showing the actual PO No value (bound to `[Table1].[Column2]`, font `Segoe UI Semibold, 10F`) — this is the control to enlarge.
- `xrLabel11` = 'Roll' caption (inside `xrPanel6`) — text changed to 'LOCATION'.
- `xrLabel16` = the control showing the actual Roll value (bound to `[Table1].[Column6]`) — left unchanged.

**2. Reviewed the font auto-scale mechanism before editing**
Read `WEPOP MVC\Controllers\WePanel.cs`, functions `BeforePrintDetail`/`BeforePrintPanel`/`BeforePrintControl` and `AutoscaleControlText`:

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

Any `XRLabel` without `Tag = "noResizeFont"` automatically shrinks its font (width-only, never grows) to fit the box on print. Raising the base Designer font is therefore the correct approach — the code will still shrink long values back down without breaking the layout.

**3. Applied the change in `IBL530.Designer.cs`**
```csharp
// before
this.xrLabel7.Font = new DevExpress.Drawing.DXFont("Segoe UI Semibold", 10F);
...
this.xrLabel11.Text = "Roll";

// after
this.xrLabel7.Font = new DevExpress.Drawing.DXFont("Segoe UI Semibold", 14F);
...
this.xrLabel11.Text = "LOCATION";
```

**4. Build verification**
`dotnet build "IBL LABEL.csproj"` — the `WEPOP_MVC` dependency built cleanly (both changes compiled fine). The `IBL LABEL.csproj` itself failed with MSB3822/MSB3823 (Non-string resources / GenerateResourceUsePreserializedResources) coming from `IBL550.resx` (embedded PNG images of a different report) — a build-environment issue (dotnet SDK 10 CLI against an old-style `.resx` targeting .NET Framework), unrelated to the two IBL530 changes and pre-existing. Needs a final build via full Visual Studio/MSBuild to verify.

**5. Investigated a stray tab character in PO No (found while verifying the printed output)**
User reported that `exec LG_RPT_IBL530('2831358,2831358','vng-241',:p_rtn_cur)` returned a PO No value with an extra trailing tab character.

Fetched the source via MCP `get_oracle_source('LG_RPT_IBL530')`: a standalone procedure, owner `SWPROD`, PO No line:
```sql
trim(A.PURCHASE_PO_NO) REF_PO_NO,
```

Checked the source data:
```sql
SELECT A.PK, A.PURCHASE_PO_NO,
       LENGTH(A.PURCHASE_PO_NO) LEN_RAW,
       LENGTH(TRIM(A.PURCHASE_PO_NO)) LEN_TRIM,
       INSTR(A.PURCHASE_PO_NO, CHR(9)) TAB_POS,
       DUMP(SUBSTR(A.PURCHASE_PO_NO, -1)) LAST_CHAR_DUMP
FROM TLG_PA_PACKAGES A
WHERE A.PURCHASE_PO_NO LIKE '%2606-0007%';
```
Result: PK=2831358 (and many other PKs in the same 'AD' supplier batch) had `PURCHASE_PO_NO = '260609-SW-AD-2606-0007' + CHR(9)`. `TAB_POS=23` (right at the end of the string), `LEN_RAW = LEN_TRIM = 23` — proving Oracle's default `TRIM()` **does not** strip tab characters (spaces only), so the tab was present in the source data itself and passed straight through the procedure into the output.

**6. Confirmed the fix direction with a readonly query**
```sql
SELECT trim(A.PURCHASE_PO_NO) AS OLD_TRIM,
       trim(replace(A.PURCHASE_PO_NO, chr(9), '')) AS NEW_TRIM,
       LENGTH(trim(A.PURCHASE_PO_NO)) OLD_LEN,
       LENGTH(trim(replace(A.PURCHASE_PO_NO, chr(9), ''))) NEW_LEN
FROM TLG_PA_PACKAGES A WHERE A.PK = 2831358;
```
Result: `OLD_TRIM` = `'260609-SW-AD-2606-0007<TAB>'` (23 chars) → `NEW_TRIM` = `'260609-SW-AD-2606-0007'` (22 chars, tab gone). Confirms `replace(A.PURCHASE_PO_NO, chr(9), '')` applied before `trim()` handles it correctly.

**7. Prepared the fix script for the stored procedure**
Only the single `REF_PO_NO` line in `LG_RPT_IBL530` changes:
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
The MCP DB tool only has read-only access, so this DDL was not executed — the script was handed over to run on an environment with proper `CREATE OR REPLACE PROCEDURE` privileges.
