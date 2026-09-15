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

## 2026-09-15

**1. Identified the form and located the correct Profit 2 formula**

User asked a follow-up: form **SS.2.5.2 "Margin Table Packages"** (`sa400190_v3.aspx`, same family as SS.2.5/SS.2.5.1) showed 'Profit 2' as **-9,239.61** (about -73 percent) for order `202607-0457WHTX`, while the regular 'Profit' row above it was a normal positive value (+3,423.04, +27 percent). Read the dso: `function="sp_sel_sa400190_v3"`, reading through view `SA_MG_TABLE`. Read the entire DECODE block carefully this time (learning from an earlier mistake of misreading dead code) to get the true live formula:

```sql
PROFIT_AMT_USD2 = ROUND(
    NVL(NEGO_AMT,0)
  - (NVL(COMMISSION_AMT_1,0)+NVL(COMMISSION_AMT_2,0)+NVL(COMMISSION_AMT_3,0)
   + NVL(PAYMENT_CONDITION_AMT_1,0)+...+NVL(PAYMENT_CONDITION_AMT_6,0)
   + NVL(FINISH1_AMT,0)+NVL(FINISH2_AMT,0)+NVL(FINISH3_AMT,0)
   + NVL(FINISH5_AMT,0)+NVL(FINISH6_AMT,0)   -- FINISH4_AMT is commented out, not counted
   + NVL(DYEING_AMT,0)+NVL(KNITTING_AMT,0)
   + NVL(PREDYE_AMT,0)+NVL(PREDYE_AMT2,0)
   + NVL(YARN_TOTAL_AMT,0)+NVL(CHARGE_AMT,0))
  - NVL(CLAIM_AMT,0)
, 2)
```
`NEGO_AMT` comes from the 'Nego Information' tab (`SA_IE_NEGO_D`), `CHARGE_AMT` from the 'Export Charge' tab (`TSA_EXP_CHGR_ITEM_DTL`), `CLAIM_AMT` from the 'Claim Note' tab (`SA_CLAIM_NOTE_ENTRY_D`, counting only claims with confirmed `STATUS='Y'`).

**2. Queried the view directly for this order and found NEGO_AMT = NULL**

```sql
SELECT TOTAL_COST_USD, PROFIT_AMT_USD2, NEGO_AMT, CHARGE_AMT, CLAIM_AMT
FROM SA_MG_TABLE WHERE SA_SALE_ORDER_PK = 151779;
```
→ `NEGO_AMT`/`CHARGE_AMT`/`CLAIM_AMT` were all NULL, and `PROFIT_AMT_USD2` exactly equaled `-TOTAL_COST_USD` - confirming NEGO_AMT was being treated as 0.

**3. Traced why NEGO_AMT was NULL despite real Nego data existing**

Queried `SA_IE_NEGO_D` directly by `SA_SALE_ORDER_PK=151779` and found one real row worth **$6,429.78**. Read the view's join logic carefully: `NEGO_AMT` links back to a sale order via a TEXT match `Z.ORDER_NO||Z.SUB_NO = SA_NEGO_SPLIT_QTY.PO_NO` (not via a direct PK relationship):
```sql
SELECT so.PO_NO, x.PO_NO AS SPLIT_PO_NO
FROM SA_SALE_ORDER so
JOIN SA_IE_NEGO_D n ON n.SA_SALE_ORDER_PK = so.PK
JOIN SA_NEGO_SPLIT_QTY x ON x.PK = n.SA_NEGO_SPLIT_QTY_PK
WHERE so.PK = 151779;
```
→ `SPLIT_PO_NO = '202409-0442HDELTA'` (the Order No/Sub No of a completely different, unrelated order, not this order's real PO No `BD19579`) - a wrongly formatted field value, preventing the view from linking the Nego amount back to the correct order.

**4. Confirmed the correct value and checked safety before applying the fix**

Sampled 20 other `SA_NEGO_SPLIT_QTY.PO_NO` values across the table and confirmed the established convention is indeed `ORDER_NO+SUB_NO` format (e.g. `202608-0373V037V`), so the correct value is `'202607-0457WHTX'` (not the real PO No `BD19579` as first guessed). Verified that `SA_NEGO_SPLIT_QTY.PK=221375` is referenced by exactly one Nego entry, and that the unrelated order it was mismatched to (`202409-0442HDELTA`, PK=114831) has its own separate Nego record (which is itself similarly broken, pointing to a third order) - so the fix carries no side effect on other orders' data.

**5. Applied the fix and verified the result**

```sql
UPDATE SA_NEGO_SPLIT_QTY
SET PO_NO = '202607-0457WHTX'
WHERE PK = 221375 AND DEL_IF = 0 AND PO_NO = '202409-0442HDELTA';
COMMIT;
```
Re-queried the view afterward: `NEGO_AMT = 6,429.78` (now correct), `PROFIT_AMT_USD2 = -2,158.7` (= 6,429.78 minus the 8,588.48 Total Cost at that moment) - the formula now runs correctly against real data, no longer showing a false negative caused by missing data.

**6. Broader scan uncovered a recurring system-wide pattern (flagged separately, not addressed here)**

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
→ **23,865 of 234,036 rows (about 10 percent)** system-wide have the same kind of PO_NO mismatch. Grouping by `CRT_BY`/month showed the issue spread continuously over more than a year and a half (2023-05 through 2024-11+) across a handful of specific users (ntrang, nnhung, nvan, nnphuong), not concentrated in one batch event - suggesting a recurring data-entry pattern (such as a 'copy from previous order' feature that fails to refresh the reference field) rather than a one-time incident. The exact mechanism/screen was not pinpointed (would require separate investigation) - flagged back to the user/IT for a decision, out of scope for this immediate fix.
