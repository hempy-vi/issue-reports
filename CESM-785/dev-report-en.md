# Dev Report — CESM-785

**Task:** Fix Roll Location/Date Mismatch Between Warehouses HOQC301 and HDF102

---

## 2026-09-10

**1. Request received**
Roll barcode `D25091920979` (PO 202508-0343HSAEROW, Lot 005, Roll 030R, ITEM_PK 980529) had been transferred from warehouse HOQC301 to HDF102, but the data was inconsistent: at HOQC301 the roll still appeared but DR (Defect WH Outsource) confirm failed with an out-of-stock error; at HDF102 (the destination) the roll could not be found to select. The user requested correcting the product income date to 20/09/2025 and the actual warehouse to 2914/HDF102.

**2. Root cause of the zero-stock error**
Read the source of `SAMILV2.LG_PRO_FPIN00020_CONFIRM` (DR confirm procedure) and found that current stock is computed by function `LG_CHECK_STOCK` — SUM(INPUT_QTY) - SUM(OUTPUT_QTY) on ledger table `TLG_IN_STOCKTR`, filtered by warehouse/item/lot/po/roll/barcode.

Queried the ledger for this barcode:
```sql
SELECT PK, TR_DATE, TLG_IN_WAREHOUSE_PK, TLG_IN_WHLOC_PK, TLG_IT_ITEM_PK,
       LOT_NO, PO_NO, ROLL_ID, BARCODE, INPUT_QTY, OUTPUT_QTY,
       TRIN_TYPE, TROUT_TYPE, TABLE_NAME, TABLE_PK, DEL_IF, CRT_DT, CRT_BY
FROM TLG_IN_STOCKTR
WHERE BARCODE = 'D25091920979'
ORDER BY CRT_DT;
```
Result showed:
- Two Product Income documents dated 20/09/2025 (slips HOQC301-2509-06480, HOQC301-2509-06688) had been cancelled (DEL_IF = the row's own PK).
- The HOQC301→HDF102 transfer ran validly at 20/09/2025 10:14 (OUT at HOQC301, IN at HDF102, both 23.7).
- A replacement Product Income document (PK 5550351/83956907, slip HOQC301-2509-07071) was created on 21/09/2025 — after the transfer.

=> Active stock at HOQC301 = IN 23.7 (income #212204200) − OUT 23.7 (transfer) = **0**, exactly matching the `ORA-20999: Out Quantity is greater than stock quantity` error. This is a correct computation against the ledger, not a calculation bug.

**3. Root cause of the roll not appearing at HDF102**
Read the source of `SAMILV2.LG_SEL_BIPO00010` (the IV0701 Item Barcode Checking screen's procedure) and confirmed the displayed WH_ID/WH_NAME/Location come directly from:
```sql
LG_GET_WAREHOUSE_ID(A.TLG_IN_WAREHOUSE_PK)  WH_ID,
LG_GET_WAREHOUSE_NAME(A.TLG_IN_WAREHOUSE_PK) WH_NAME,
(SELECT MAX(L.LOC_ID || ' - ' || L.LOC_NAME) FROM TLG_IN_WHLOC L WHERE L.PK = A.TLG_IN_WHLOC_PK) LOCATION_NM
FROM TLG_PA_PACKAGES A
```
i.e. from table `TLG_PA_PACKAGES` (roll master), not recomputed from the ledger. Queried this table:
```sql
SELECT PK, ITEM_BC, TLG_IN_WAREHOUSE_PK, TLG_IN_WHLOC_PK, PROD_DATE, FIRST_STOCK_DATE,
       TLG_IN_WAREHOUSE_PK_BK, TLG_IN_WHLOC_PK_BK
FROM TLG_PA_PACKAGES WHERE ITEM_BC = 'D25091920979';
```
=> `TLG_IN_WAREHOUSE_PK`/`TLG_IN_WHLOC_PK` still = 1753/7646 (HOQC301) — never updated when the transfer ran, even though the ledger already recorded the roll as being at HDF102 (2914/12628). This is exactly why the roll still 'appeared' at HOQC301 (with zero stock) and 'could not be found' at HDF102 (despite having sufficient stock of 23.7 there).

Confirmed warehouse/location codes against master tables:
```sql
SELECT PK, WH_ID, WH_NAME FROM TLG_IN_WAREHOUSE WHERE PK IN (1753, 2914);
-- 1753 = HOQC301, 2914 = HDF102
SELECT PK, LOC_ID, LOC_NAME, TLG_IN_WH_PK FROM TLG_IN_WHLOC WHERE PK IN (7646, 12628);
-- 7646 = HOQC301, 12628 = HDF102_000 (HDF102 Common)
```

Also found that `TLG_PA_PACKAGES.PROD_DATE` is NULL system-wide (0 of 200,000 sampled rows populated) — not the field actually used for 'product income date'. The field actually in use is `TLG_PR_PROD_INCOME_M.PROD_DATE`/`INPUT_DATE` on the active income document (PK 5550351), currently 21/09/2025.

**4. Safety check before fixing**
```sql
SELECT PK, TR_DATE, CLOSE_YN, PR_ST_CLOSE_M_PK, FROM_CLOSE_PK
FROM TLG_IN_STOCKTR WHERE PK IN (212204200, 212166764, 212166765);
```
Both ledger rows to be modified have `CLOSE_YN='N'` and are not linked to any closing period (`FROM_CLOSE_PK` is NULL) — safe to modify the dates.

**5. Fix script prepared**
```sql
-- 1) Product income date: 20250921 -> 20250920
UPDATE SAMILV2.TLG_PR_PROD_INCOME_M
   SET PROD_DATE = '20250920', INPUT_DATE = '20250920',
       MOD_DT = SYSDATE, MOD_BY = 'DATA_FIX_CESM785'
 WHERE PK = 5550351 AND PROD_DATE = '20250921';

UPDATE SAMILV2.TLG_IN_STOCKTR
   SET TR_DATE = '20250920', MOD_DT = SYSDATE, MOD_BY = 'DATA_FIX_CESM785'
 WHERE PK = 212204200 AND TABLE_NAME = 'TLG_PR_PROD_INCOME_D'
   AND TABLE_PK = 83956907 AND TR_DATE = '20250921';

-- 2) Actual warehouse: HOQC301 (1753/7646) -> HDF102 (2914/12628)
UPDATE SAMILV2.TLG_PA_PACKAGES
   SET TLG_IN_WAREHOUSE_PK_BK = TLG_IN_WAREHOUSE_PK,
       TLG_IN_WHLOC_PK_BK     = TLG_IN_WHLOC_PK,
       TLG_IN_WAREHOUSE_PK    = 2914,
       TLG_IN_WHLOC_PK        = 12628,
       MOD_DT = SYSDATE, MOD_BY = 'DATA_FIX_CESM785'
 WHERE ITEM_BC = 'D25091920979'
   AND TLG_IN_WAREHOUSE_PK = 1753 AND TLG_IN_WHLOC_PK = 7646;

COMMIT;
```
The previous warehouse values are backed up into columns `TLG_IN_WAREHOUSE_PK_BK`/`TLG_IN_WHLOC_PK_BK` — following the existing convention on this table (original column comment: 'PQH USE IT FOR BACKUP WHEN UPDATE DATA'), together with a matching ROLLBACK statement in case of need.

**6. Operational note**
The old draft DR voucher `DR-HOQC301-20260910-169187` (still SAVED, not confirmed) had been created selecting warehouse HOQC301 because the roll picker was showing the roll there incorrectly at the time. After the fix, the roll will no longer appear at HOQC301, so the old draft should be discarded and a new DR voucher created selecting warehouse HDF102 instead.
