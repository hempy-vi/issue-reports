# Dev Report — CESM-895

**Task:** Clarify How Form SS2.5.1 Data Is Populated (User-Loaded vs Auto-Updated) with Examples

---

## 2026-09-14

**1. Identify the backend table behind form SS.2.5.1 (Margin Table After Cost)**

Searched for margin-related tables in the SAMILV2 schema and found both `SA_MARGIN_TABLE` and `SA_MARGIN_TABLE_V2` (near-identical column structure). Queried the sample order `202608-0025HNW036` against both tables and confirmed the data only exists in `SA_MARGIN_TABLE` (PK=202717), not in `SA_MARGIN_TABLE_V2`.

```sql
SELECT 'SA_MARGIN_TABLE' SRC, PK, ORDER_NO, ITEM, CRT_BY, CRT_DT, MOD_BY, MOD_DT
FROM SA_MARGIN_TABLE WHERE ORDER_NO = '202608-0025HNW036'
UNION ALL
SELECT 'SA_MARGIN_TABLE_V2', PK, ORDER_NO, ITEM, CRT_BY, CRT_DT, MOD_BY, MOD_DT
FROM SA_MARGIN_TABLE_V2 WHERE ORDER_NO = '202608-0025HNW036';
```

**2. Check CRT_BY/MOD_BY for the sample order and several other orders, map logins to real names**

Result: `CRT_BY = 'thanh'`, `MOD_BY = 'thanh'`, 5 seconds apart (consistent with a single Save click on the form). Joined against the user table `TCO_BSUSER_XXX` (USER_ID/USER_NAME) to map the login to a real name: `thanh` = NGUYEN TRAN THANH THANH.

Grouped `CRT_BY` across the whole `SA_MARGIN_TABLE`: most values are real user logins (jenny, hien, hnsthuan, sxoan, dony0905...), except `CRT_BY = 'convert'` with 13,880 rows - a one-time data migration from the legacy system (not a recurring batch job).

**3. Ruled out automatic background updates**

```sql
SELECT trigger_name FROM all_triggers WHERE table_name = 'SA_MARGIN_TABLE';  -- 0 rows
SELECT job_name FROM all_scheduler_jobs WHERE UPPER(job_name) LIKE '%MARGIN%';  -- 0 rows
```
No trigger and no Oracle Scheduler job writes to `SA_MARGIN_TABLE`.

**4. Traced the actual INSERT/UPDATE procedure invoked on Save**

```sql
SELECT DISTINCT owner, name FROM all_source WHERE UPPER(text) LIKE '%INSERT INTO SA_MARGIN_TABLE%';
```
Returned `SP_UPD_SA400190` (currently used), `SP_UPD_SA400190_V2`, and two dead/legacy procedures (`SP_UPD_SA400050`, `STOMFRSTOMSO0016_U_02`, no longer referenced by any form, last modified in 2020). `SP_UPD_SA400190` receives `P_CRT_BY` as an **input parameter** (not self-generated), used to set `CRT_BY`/`MOD_BY` - confirming this value is passed in by the web-layer code (the logged-in session user) when the procedure is called, not derived by any background logic.

**5. Follow-up question: do SS.2.5 and SS.2.5.1 share the same data source?**

User reported: staff insisted they 100 percent only use form SS.2.5 (`sa400190.aspx`) and never touch SS.2.5.1 (`sa400190_v2.aspx`), yet noticed a discrepancy in the 'Final quantity' figure between the two forms for the same order (10,759 vs 8,069.3).

Read the dso declarations in both `.aspx` files (live URL confirmed via a browser screenshot):
```
sa400190.aspx     -> procedure="sp_upd_sa400190"     -> writes to SA_MARGIN_TABLE
sa400190_v2.aspx  -> procedure="sp_upd_sa400190_v2"  -> by code should write to SA_MARGIN_TABLE_V2
```
Verified empirically with two other orders (opened live on the user's machine) - both actually write to `SA_MARGIN_TABLE` (the old table), not `_V2` - meaning **SS.2.5 and SS.2.5.1 read/write the exact same row** (same PK, same `SA_SALE_ORDER_PK`). Therefore `CRT_BY`/`MOD_BY` correctly reflect the SS.2.5 user even though they never opened SS.2.5.1 - SS.2.5.1 is merely a display/computed screen over the same underlying data.

**6. Investigated the root cause of the 'Final quantity' mismatch (10,759 vs 8,069.3)**

Compared the SELECT procedures behind both forms:
```sql
-- SP_SEL_SA400190 (SS.2.5): reads the stored column directly
A.PACKING_QTY   -- = 10,759 (matches DB)

-- SP_SEL_SA400190_V2 (SS.2.5.1), via view SA_MG_TABLE:
DECODE(
    ORDER_TYPE
  , 1, DECODE(UNIT_PRICE, 'KG', QC_WEIGHT_OK, QC_LENGTH_OK)
  , DECODE(UNIT_PRICE_P, 'KG', G.FG_WEIGHT, G.FG_LENGHT)
) AS PACKING_QTY
```
This order has `ORDER_TYPE = '1'` and `UNIT = 'KG'` (confirmed via `SA_SALE_ORDER`) - which falls into the branch pulling `QC_WEIGHT_OK` from table `TABLE_MG_2` (`DATA_TYPE = 'QC'`) = **8,069.3** - exactly matching the result the user obtained by directly executing both procedures for cross-check:
```sql
exec sp_sel_sa400190('202717', :p_rtn_value);      -- PACKING_QTY = 10759
exec sp_sel_sa400190_v2('202717', :p_rtn_value);   -- PACKING_QTY = 8069.3
```

**7. Traced `QC_WEIGHT_OK` back to its true origin - real QC data, not manual entry**

`TABLE_MG_2` is refreshed by procedure `JOB_INSERT_MARGIN_TABLE` (no job/scheduler/trigger/button found that calls it automatically - it may be run manually by DBA/IT; exact refresh cadence not determined). Traced the join chain back to the roll-level QC detail table:
```
SA_SALE_ORDER -> SA_PROCESSING_ORDER -> SA_ORDER_PRODUCTION ->
SA_ORDER_PROD_COLOR -> SA_PROCESSING_CARD -> SA_QC_M -> SA_QC_D
```
Queried directly for this order (Sale Order PK=152315):
```sql
SELECT qd.ROLL_DECISION, COUNT(*) ROLL_CNT, SUM(qd.ROLL_WEIGHT) TOTAL_WEIGHT
FROM ... (join chain above) WHERE e.PK = 152315 AND qd.DEL_IF = 0
GROUP BY qd.ROLL_DECISION;
```
Result: 431 rolls with decision='Q' (passed) sum to **8,069.3 kg** - an exact match. The person who entered this data (`CRT_BY` on `SA_QC_D`) is QC inspector **DUONG THI NHU QUYNH**, with rows created between 2026-09-03 10:40 and 11:57 (about 2-3 minutes apart per roll, consistent with manual scanning/weighing), completely independent of the margin costing user (`thanh`, who worked on 2026-08-03).

**8. Explained a further discrepancy against another QC report (SD.9.3.1 Total View New Table / SD.8.4_V6 F.G Inquiry-Package)**

User cross-checked against another QC report and found `QC Weight = 8,645.1` / `Good Weight = 8,542.4` - not matching 8,069.3. Confirmed `TABLE_MG_2` keeps two separate rows by `DATA_TYPE`: `'QC'` (in-process check right after production, giving 8,069.3/8,191.4) and `'OQC'` (Outgoing QC - final check before packing/shipment, giving 8,542.4/8,645.1), computed from two different sources/formulas inside `JOB_INSERT_MARGIN_TABLE`. Report `SD.8.4_V6 F.G Inquiry-Package` (the word 'Package' indicates the packing stage) correctly displays the `OQC` figure, while SS.2.5.1 (for this order) uses the `QC` figure - the two reports are showing two different QC checkpoints of the same production lot, not a data inconsistency.
