# Dev Report — CESM-849

**Task:** Fix WePOP Batch Print Wrong Customer Data and Unresponsive Print All Button (IBL020/IBL040)

---

## 2026-09-14

**1. Received bug report and initial analysis**
Bug report CESM-849 described 3 issues on WePOP (Barcode Label module):
- Bug 1: Batch printing ("Print All") in IBL020/IBL040 showed the wrong Customer on labels for customer ESTEC (e.g. "VO", "VJ", "VQ" instead of "ESTEC VIET NAM"), while single-record printing ("Print") was always correct.
- Bug 2: The ESTEC label layout had an extra "Shipment Qty" row not present in the customer-provided sample.
- Bug 3: The "Print All" button on IBL020 produced no response when clicked.

**2. Investigated the report dispatch chain via MCP oracle-shinwoo**
Identified the chain: `LG_RPT_IBL520` (single-print dispatcher, shared by IBL020+IBL040) and `LG_RPT_IBL520_1`/`LG_RPT_IBL540_1` (separate batch-print dispatchers for IBL020/IBL040) — both route by `customer LIKE '%SATO%'`, `order_type='00'`, or `TLG_PR_FACTORY_PK` to 4 leaf procedures: `LG_RPT_IBL522`/`523`/`524`/`526` (base, single-print) and `LG_RPT_IBL522_1`/`523_1`/`524_2`/`526_1` (batch-print, SHARED between the `_1`/`540_1` dispatchers).

Ran a 4-agent parallel workflow comparing the Customer expression and `TCO_BUSPARTNER` join between the "_1"/"_2" leaf procedures and their base counterparts — result: all joins were correct (`C.PK(+) = A.TCO_BUSPARTNER_PK`, resolved per package row), ruling out the originally suspected Customer-join bug.

**3. Read the full source (not just the Customer expression) of the 4 "_1"/"_2" leaf procedures — found the real root cause**
All 4 procedures `LG_RPT_IBL522_1`, `LG_RPT_IBL523_1`, `LG_RPT_IBL524_2`, `LG_RPT_IBL526_1` filtered the `TLG_PA_PACKAGES` table using only:
```sql
AND A.TABLE_NM = 'TLG_GD_PLAN_D'
AND A.TABLE_PK = Z.PK
```
(marked with a `-- [FIX]` comment in the source, indicating a deliberate prior change — made for IBL040, since IBL040's packages are linked via the generic `TABLE_NM`/`TABLE_PK` columns).

However, `LG_RPT_IBL520_1` (the **IBL020** batch-print dispatcher) passes these procedures a `TLG_SA_SALEORDER_D_PK` value (an SO-detail-row PK) — an entirely different key type from the `TLG_GD_PLAN_D_PK` the filter expects. Packages created from IBL020 (`LG_UPD_IBL020_V3`) are stored directly in the `TLG_SA_SALEORDER_D_PK` column, NOT via `TABLE_NM`/`TABLE_PK` — confirmed via `LG_SEL_IBL020_BC`:
```sql
AND A.TLG_SA_SALEORDER_D_PK = P_TLG_SA_SALEORDER_D_PK
```
=> When "Print All" is clicked on IBL020: no `TLG_GD_PLAN_D` row shares that PK number → the cursor returns empty → silently nothing prints (Bug 3). In the rare case where the PK number happens to coincidentally match a real Plan row belonging to a different customer (PKs are integers shared across the whole schema), the batch print shows that unrelated customer's data instead (Bug 1). **Bug 1 and Bug 3 on IBL020 are consequences of the same defect.**

**4. Implemented and applied the fix**
- SQL: restored the original filter condition alongside the current one (OR logic) in all 4 "_1"/"_2" procedures:
```sql
AND ( (A.TABLE_NM = 'TLG_GD_PLAN_D' AND A.TABLE_PK = Z.PK)
   OR A.TLG_SA_SALEORDER_D_PK = Z.PK )
```
Safe for IBL040 (existing branch untouched), and restores correct behavior for IBL020.
- C# (`WEPOP MVC/Views/PrintPreview.cs`, `Print()` method): added an `else` branch showing an `XtraMessageBox` when `dataTable` is empty (previously the method silently returned — the direct cause of "clicking Print All shows nothing"), and added `XtraMessageBox.Show` to the per-row catch block for when a data row's first column (`labelKey|className|ver`) is null or malformed (previously only logged to a file, with no user-visible feedback).

**5. Incident: the entire label-printing chain became INVALID**
The user attempted to self-"restore" the base procedures by pasting the raw SQL text block from the original bug report (which still contained plain-English section labels such as "search master"/"search detail"/"print detail (đúng)") directly into a SQL execution tool, and hit:
```
[Error] Compilation (2: 2): PLS-00103: Encountered the symbol " " ...
```
Checking `all_objects`/`all_errors` revealed **10 procedures in the label-printing chain** (`LG_RPT_IBL520`, `LG_RPT_IBL520_1`, `LG_RPT_IBL522`, `LG_RPT_IBL522_1`, `LG_RPT_IBL523`, `LG_RPT_IBL523_1`, `LG_RPT_IBL524`, `LG_RPT_IBL524_2`, `LG_RPT_IBL526`, `LG_RPT_IBL526_1`, `LG_RPT_IBL540_1`) were now **INVALID** with genuine compile errors — affecting BOTH IBL020 and IBL040, for EVERY customer, not just ESTEC.

**6. Root cause diagnosis and recovery**
Using `DUMP(text,1016)` on `all_source` confirmed the stored source contained literal **non-breaking-space characters (U+00A0, hex `c2 a0`)** interleaved with regular spaces in the indentation — invisible in any normal viewer, but rejected by the PL/SQL parser, exactly matching the `PLS-00103` error. `ALTER PROCEDURE ... COMPILE` did not fix it (the stored source itself was genuinely corrupted, not a stale cache issue).

Re-typed (not copy-pasted) all 11 procedures from scratch, verified with `grep` that zero NBSP characters were present, and ran `CREATE OR REPLACE` to restore them — including the fix from step 4 for the 4 "_1"/"_2" procedures in the same pass. After running: confirmed all 11 procedures `VALID`.

**7. Found 2 more affected procedures (outside the print chain)**
The IBL040 screen reported `PLS-00905: object SWPROD.LG_SEL_IBL040 is invalid` on search. Checking confirmed `LG_SEL_IBL040` and `LG_SEL_IBL040_BC` were also NBSP-contaminated by the same faulty script run (these 2 procedures were part of the original pasted content but outside the label-printing chain, so were missed in step 6). Re-typed cleanly (verified 0 NBSP) and restored — a full scan of `all_objects` for `INVALID` status with today's modification date confirmed no other procedures were affected.

**8. Prepared architectural improvement (not yet executed)**
Per the user's request: fully separate the 4 "_1"/"_2" procedures currently SHARED between IBL020 and IBL040 (the root cause of the entire incident) — create 4 new procedures dedicated to IBL020 (`LG_RPT_IBL522_2`, `LG_RPT_IBL523_2`, `LG_RPT_IBL524_3`, `LG_RPT_IBL526_2`, using only the `TLG_SA_SALEORDER_D_PK` condition), update `LG_RPT_IBL520_1` to call the new procedures, then revert the 4 original procedures back to only the `TABLE_NM`/`TABLE_PK` condition (removing the OR branch) — so IBL020 and IBL040 no longer share any leaf procedure. SQL prepared but not yet run (not urgent, since the system is stable with the fix from steps 4-6).

**Remaining open items (not addressed in this session):**
- Bug 2 (Shipment Qty layout + field ordering for ESTEC on the `IBL526` template — a template shared by many other customers, requiring proper scoping by Customer before editing).
- Bug 1 specific to IBL040: a single Plan row can accumulate multiple barcode-creation batches for different "OTHER CUSTOMER" entries over time (since `ITEM_BC` is not unique) — Print All returns all of them, including stale labels for other customers — requires a data-handling decision.
- Step 8 (splitting the IBL020/IBL040 procedures) — SQL prepared, not yet executed.
