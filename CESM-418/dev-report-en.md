# Dev Report — CESM-418

**Task:** [DONGIL][Requirement] Enforce 100% USOT3137 Allocation Rule for All Finished Goods Lots in Monthly Lot Tracking (August 2026)

---

## 2026-09-08

**1. Identified the target object and reference data**

Traced the requirement to procedure `LG_PRO_DAILY_LOT_TRACKING_JOB` (schema DONGIL). It takes no parameters and always derives the working month from `SYSDATE`:

```sql
L_MONTH VARCHAR2(6):=to_char(sysdate, 'yyyymm');
...
L_PROCESS_TYPE VARCHAR2(10):='DD';/*==MM: Monthly closing; DD: Daily Closing=*/
```

`L_PROCESS_TYPE` is hardcoded `'DD'` and never reassigned — the `'MM'` branch (lines ~108-282) is dead code for DONGIL, confirmed by the original developer's own comment at line 108: `--ko chay, cua dongil, can sua lai` ("does not run, for dongil, needs fixing").

Relevant tables:
- `TLG_LOT_TRACKING_MAT` — per-lot material allocation detail; this is exactly what the PM0407 "Material Detail" tab displays.
- `TLG_LOT_TRACKING_TEMP` — internal staging table used only within a single run of the job.
- `TLG_WI_LINE_OP_CONS` — the Work Instruction's frozen material-blend ratio (master data, not touched by this job).

Item codes resolved:
- `USOT3137` = `TLG_IT_ITEM.PK 122`
- `BRAMID` = `PK 121`
- Both (+ AUS, BRA0001, BRAEAGLE, BRAM 36, USOT3136, USPIMA3346 — 8 codes total) belong to `TLG_IT_ITEMGRP_PK = 27` (the raw-cotton item group).

**2. Confirmed the carry-over mechanism with real data**

Line 284 soft-deletes the entire current month's `TLG_LOT_TRACKING_MAT` rows on every run, then rebuilds from scratch (matches "recalculates the entire current month every night").

Lines 296-355: a `WITH` query (`TBL_IN_DATA`/`TBL_OUT_DATA`) computes prior month's ending balance (`QTY_END_MAT`) per (material item, material lot, mixing lot, WI line), then inserts one new `'OPEN'` row for the current month — **keeping the original `TLG_IT_ITEM_PK` unchanged**:

```sql
SELECT L_TIN_STOCKTR_PK, CUR.SLIP_NO, 'I60', CUR.QTY_END_MAT, ...
CUR.TLG_IT_ITEM_PK, CUR.LOT_MAT, ... L_MONTH, 'OPEN'
```

Verified against `MIX_LOT='MIXEDF1-260627-02'`, item BRAMID (121): OPEN balance persists unchanged across STD_YM 202607 (256,083 kg), 202608 (206,538 kg), 202609 (24,825 kg) — carried forward verbatim, code never changes.

**3. Confirmed the material-allocation ("ration") mechanism**

Lines 613-655 (comment: "CAL RATION ITEM OF MIX_LOT"): cursor `L_CUR` computes `NEED_QTY` per material item using a `RATE` percentage sourced from `TLG_WI_LINE_OP_CONS.CONS_QTY` — the WI's original blend ratio (e.g. 60% BRAMID / 40% USOT3137), fixed at WI-authoring time, not written by this job. Cursor `STOCK_OVER` (631-637) draws matching stock from `TLG_LOT_TRACKING_TEMP` and inserts the allocation row (`STOCK_TYPE='MAPPING'`, `TROUT_TYPE='O60'`) against the finished lot.

A secondary branch (lines 791-862, taken when the mixing lot's remaining stock is less than the finished lot's need) drains `TEMP` directly without going through the RATE split — but since `TEMP` only aggregates what's actually in `TLG_LOT_TRACKING_MAT` (already normalized once the carry-over fix is in place), this branch needs no separate change.

**4. Drafted a minimal-footprint patch — 2 edit points (v1)**

1. Declared 4 new locals + one `SELECT INTO` resolving `USOT3137`'s `PK`/`TLG_IT_ITEMGRP_PK`/`ITEM_CODE`/`ITEM_NAME` at procedure start.
2. In `TBL_IN_DATA`/`TBL_OUT_DATA` (carry-over CTE, lines ~297-312): wrapped the item code/name columns in `CASE WHEN <same item group, different PK> THEN <USOT3137> ELSE <original>` — everything downstream (JOIN, GROUP BY, the final INSERT at 347-355) is left untouched since it inherits the normalized value through existing aliases.
3. In cursor `L_CUR` (ration, lines 616-624): added an equivalent `CASE WHEN` layer that consolidates the RATE share of every other cotton code into USOT3137 before computing `NEED_QTY` — without this, the RATE share originally assigned to BRAMID would look for BRAMID stock in `TEMP` (now empty after the carry-over fix) and silently under-allocate.

No quantity (`INPUT_QTY`/`OUTPUT_QTY`/`NEED_QTY`) is changed anywhere — only the material-code label. `LOT_NO` is deliberately left unchanged.

**5. Ran an independent verification workflow — found 2 blocking gaps in v1**

Spawned an independent re-derivation agent (blind to the drafted patch) plus two adversarial reviewers (correctness/safety, and business/accounting risk).

Findings:
- **Blocking gap**: v1 completely missed a 3rd write path into `TLG_LOT_TRACKING_MAT` — a "stock transfer-in to a `PROCESS_TYPE='MIXED'` warehouse" block (lines 386-414), which inserts `TLG_IT_ITEM_PK = D.TR_ITEM_PK` (the real transferred item) with no normalization at all. Verified against real data: BRAMID flowed through this exact path 3,776 times / 4,322,567 kg — the single largest source of non-USOT3137 material, larger than the carry-over volume.
- Confirmed the already-drafted carry-over CTE and ration-cursor edits are arithmetically/JOIN-correct and correctly scoped to item group 27 only.
- **Non-blocking, out of scope**: found a pre-existing bug (unrelated to this ticket) — the carry-over CTE's `OUTER JOIN` matches `SLIP_NO`/`TR_DATE` of an opening-balance row against a finished-lot production row, but these are two structurally unrelated numbering series that never match; net effect is that `OUTPUT` never actually reduces the carried-forward balance (`QTY_END_MAT = SUM(IN) - 0`, always). Confirmed with data: BRAMID's OPEN balance on `MIXEDF1-260627-02` stayed at exactly 3,546.51871 kg across three consecutive months despite matching consumption being recorded each month. Flagged for a separate ticket, not touched here.
- Confirmed an operational gap: since the job has no parameters and always derives `L_MONTH` from `SYSDATE`, running it on 2026-09-10 (after SYSDATE has rolled into September) will recompute September, not touch August's already-closed ledger.

**6. Patched v1 → v2 (added the missing 3rd edit point)**

Added Edit 5: in the transfer-in cursor's SELECT list (line 390) and its matching `GROUP BY` clause (lines 411-413), applied the same `CASE WHEN I.TLG_IT_ITEMGRP_PK = L_USOT_ITEMGRP_PK AND I.PK <> L_USOT_ITEM_PK THEN L_USOT_ITEM_PK ELSE D.TR_ITEM_PK END` normalization (the join to `TLG_IT_ITEM I` already existed at line 405, no new join needed).

**7. At user's request, assembled a full `CREATE OR REPLACE PROCEDURE`**

Pulled the procedure's verbatim source via `ALL_SOURCE` (paginated `SELECT line, text FROM all_source WHERE name='LG_PRO_DAILY_LOT_TRACKING_JOB' AND type='PROCEDURE' ... ORDER BY line`, in ranges to stay under the read-only tool's row cap), reconstructed the body in a local file, then applied all 5 edits via exact-string-match replacement (each replacement fails loudly if the "before" text doesn't match verbatim — used as a built-in check that no manual retyping drift occurred). Each edit is tagged `-- [PATCH USOT3137]` inline for the DBA's review. Output: `LG_PRO_DAILY_LOT_TRACKING_JOB_PATCHED.sql`.

**8. User reported a compile failure (Toad for Oracle)**

```
PLS-00103: Encountered the symbol "end-of-file" when expecting one of the following:
( begin case declare end exception exit for goto if loop mod
null pragma raise return select update while with
```
at the file's last line.

Ran an independent PL/SQL block-balance checker (custom PowerShell script tracking `BEGIN`/`CASE`/`IF`/`LOOP`/`END` nesting, string/comment-aware) against the patched file — matched the exact same nesting shape as the untouched original, so the 5 edits were not the cause.

To isolate the cause, produced a **control file**: the unmodified original body wrapped only in `CREATE OR REPLACE` / `/` (zero patches). The control file failed with the identical error — proving the bug was in the source-extraction/reconstruction step, not in the 5 edits.

**Root cause**: the procedure's true structure is 3-layered — an outer `BEGIN` (original line 3, immediately after `IS`) wraps a nested `DECLARE ... BEGIN ... END;` block (original lines 17-880), followed by ~465 lines of fully commented-out legacy code (lines 881-1345, inert), followed by exactly **one un-commented `END;` at line 1346** that closes the *outer* `BEGIN`. The extraction had stopped at line 880 (the inner block's own closing `END;`), so it was missing that final outer `END;`.

**9. Fixed**

Appended the missing `END;` before the trailing `/` in both `LG_PRO_DAILY_LOT_TRACKING_JOB_PATCHED.sql` and the control file, and re-ran the block-balance checker: final stack depth 0, zero mismatches, for both files. Sent back to the user to re-compile in Toad.

---

**Open items still awaiting user/business confirmation before this goes to production:**
1. Whether keeping `LOT_NO` unchanged after a material-code swap is acceptable, or a different lot reference is needed.
2. Whether the resulting "orphaned" BRAMID book balance (never consumed by this job again) has any downstream inventory/costing impact.
3. Confirm the runbook for re-running August (temporarily hardcoding `L_MONTH:='202608'`, running once manually, then reverting) matches DONGIL/consultant's intended process.
4. Confirm the patch's permanent, unconditional scope (all 8 codes in item group 27, no time limit) is really the intended long-term behavior.
5. Whether the unrelated carry-over `OUTER JOIN` bug (found in step 5) should be filed as its own separate ticket.
