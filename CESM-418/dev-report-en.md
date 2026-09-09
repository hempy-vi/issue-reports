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

**10. BC added a "no code change required" conclusion — re-verified, found incorrect**

User sent a PM0407 screenshot (WIP1/CM00P1, Biz Center DI_F1-Factory 1, July 2026, Material Detail tab) plus a BC-added note on the ticket: "Data Analysis: Upon checking the actual data for July 2026, the remaining WIP raw material inventory consists strictly of US Cotton (USOT3137). There are no mixed non-USOT3137 materials (e.g., BRAMID) carried over. Conclusion: ... no system code modification is required for this task."

Re-verified via a direct query rather than accepting it at face value. First confirmed `BIZ_CENTER=3402` is the ONLY biz center with lot-tracking data anywhere in the system (no other site/factory ambiguity) — matches "DI_F1-Factory 1" exactly. Then queried the exact scope the BC had checked:

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

Result:
```
202607  USOT3137   92 mixing lots   704,381.12 kg
202607  BRAMID     65 mixing lots   505,088.01 kg
202608  USOT3137  172 mixing lots 1,775,525.71 kg
202608  BRAMID     65 mixing lots   505,088.01 kg   (carried over unchanged)
```

BRAMID accounts for ~42% of total weight within the exact scope the BC checked for July — directly contradicting the "no mixed non-USOT3137 materials" conclusion. Identified why the BC missed it: the Mixing Lots visible on-screen (`MIXEDF1-260728` through `MIXEDF1-260731`, created in late July) are genuinely 100% USOT3137 — but older mixing lots (`MIXEDF1-260627-xx`, created in late June) still carry BRAMID, further down the Material Detail grid (which has a scrollbar). Conclusion: **disagreed** with "no system code modification is required" — reported back to the user with specific figures, kept the recommended patch.

**11. Discovered a parallel procedure — the patch turned out to already be live in production**

User shared an example from site KYUNGBANG, where `LG_PRO_DAILY_LOT_TRACKING_JOB` is just a thin wrapper calling `LG_PRO_DAILY_LOT_TRACKING(L_MONTH, 'DD')` (a separate procedure with parameters `P_YYYYMM`, `P_TYPE`), and asked for the same for DONGIL.

Before implementing, checked and found DONGIL **also already has** `LG_PRO_DAILY_LOT_TRACKING(P_YYYYMM, P_TYPE default 'MM')` — 1346 lines, near-identical to `_JOB` (only 3 differing lines: the signature, `L_MONTH:=p_yyyymm`, `L_PROCESS_TYPE:=P_TYPE` instead of hardcoded). A direct SQL diff between the two objects confirmed this:

```sql
SELECT A.LINE, A.TEXT AS JOB_TEXT, B.TEXT AS PARAM_TEXT
FROM (SELECT LINE, TEXT FROM ALL_SOURCE WHERE NAME='LG_PRO_DAILY_LOT_TRACKING_JOB' AND TYPE='PROCEDURE') A
FULL OUTER JOIN (SELECT LINE, TEXT FROM ALL_SOURCE WHERE NAME='LG_PRO_DAILY_LOT_TRACKING' AND TYPE='PROCEDURE') B
  ON A.LINE = B.LINE
WHERE NVL(A.TEXT,'~') <> NVL(B.TEXT,'~')
ORDER BY A.LINE
```

The diff also incidentally revealed: `LG_PRO_DAILY_LOT_TRACKING_JOB` in the DB already contained the exact content of the 5-point USOT3137 patch (the `-- [PATCH USOT3137] Edit 1...` comments showed up directly in `ALL_SOURCE`) — asked the user and **confirmed they had already compiled the v2 patch for real onto production** (with a `LG_PRO_DAILY_LOT_TRACKING_JOB_BK` backup taken beforehand).

Warned the user of a risk: literally following the original ask and turning `_JOB` into a wrapper calling `LG_PRO_DAILY_LOT_TRACKING` directly (the parameterized version, **not yet** patched) would disable the just-applied USOT3137 rule — `CREATE OR REPLACE` would overwrite the patch inside `_JOB`, and the real logic would shift to run inside `LG_PRO_DAILY_LOT_TRACKING`, which was still unpatched.

**12. Per user's instruction, moved all logic + the patch into LG_PRO_DAILY_LOT_TRACKING**

User clarified: `LG_PRO_DAILY_LOT_TRACKING` would become the main processing procedure, with the patch applied to **both** its `'MM'` and `'DD'` branches; `_JOB` would become nothing but a wrapper calling into it, matching KYUNGBANG's style.

Read through the entire `'MM'` branch (original lines 108-282) for the first time (previously treated as dead code, never analyzed in depth): its structure is simpler than the `'DD'` branch — 3 near-identical loops (no ration/rate mechanism like DD), copying directly from the closing snapshot table `TLG_CL_CLOSING_MAT_DETAIL` (columns `MAT_BEGIN_QTY`/`MAT_IN_QTY`/`MAT_OUT_QTY` mapping to `STOCK_TYPE='OPEN'/'DAILY'/'MAPPING'` respectively), all 3 pulling `A.MAT_ITEM_PK` straight — already joined to `TLG_IT_ITEM I`, so each just needed one CASE WHEN added to its SELECT list:

```sql
-- Before:
,A.MAT_ITEM_PK TLG_IT_ITEM_PK,
-- After (Edit M1/M2/M3, applied to all 3 loops via replace_all):
,CASE WHEN I.TLG_IT_ITEMGRP_PK = L_USOT_ITEMGRP_PK AND I.PK <> L_USOT_ITEM_PK
      THEN L_USOT_ITEM_PK ELSE A.MAT_ITEM_PK END TLG_IT_ITEM_PK,
```

Fetched the entire verbatim source of `LG_PRO_DAILY_LOT_TRACKING` (SEPARATELY, not reusing `_JOB`'s, to avoid drift) via `ALL_SOURCE` — confirmed the body is identical to `_JOB`'s original (pre-patch) body, differing only in the first 3 lines.

Assembled `LG_PRO_DAILY_LOT_TRACKING_PATCHED.sql` — a full `CREATE OR REPLACE` with **8 patch points**: Edit 1/2 (4 new local variables + one `SELECT INTO` resolving `USOT3137`, running before either the MM or DD branch), Edit M1/M2/M3 (MM branch, new), Edit 3 (carry-over CTE, DD), Edit 5 (stock-transfer-in cursor, DD), Edit 4 (ration cursor, DD) — verified with a dedicated block-balance checker script (tracks `BEGIN`/`CASE`/`IF`/`LOOP`/`END` nesting depth, string/comment-aware): final stack depth 0, zero mismatches, parens balanced (250 open = 250 close).

Assembled `LG_PRO_DAILY_LOT_TRACKING_JOB_WRAPPER.sql` — the thin wrapper for `_JOB`, matching the KYUNGBANG pattern exactly:

```sql
CREATE OR REPLACE PROCEDURE LG_PRO_DAILY_LOT_TRACKING_JOB
IS
    L_MONTH VARCHAR2(6):=to_char(sysdate, 'yyyymm');
BEGIN
    LG_PRO_DAILY_LOT_TRACKING(L_MONTH, 'DD');
END;
/
```

Noted the mandatory ordering when applying: run the main procedure file FIRST, the wrapper SECOND — otherwise `_JOB` would temporarily call into the still-unpatched `LG_PRO_DAILY_LOT_TRACKING`. Important side benefit: re-running for a specific month (e.g. August) now just needs a direct parameterized call, `EXEC LG_PRO_DAILY_LOT_TRACKING('202608', 'DD');`, instead of the old runbook's temporary hardcoded `L_MONTH` edit.

**13. Confirmed both objects compiled successfully in production**

Re-checked `ALL_OBJECTS`: both `LG_PRO_DAILY_LOT_TRACKING` (last_ddl_time 13:39:03) and `LG_PRO_DAILY_LOT_TRACKING_JOB` (13:39:24 — correct order, main first then wrapper, 21 seconds apart) show `STATUS=VALID`. Re-read `ALL_SOURCE` to confirm `_JOB` is now genuinely the 6-line wrapper, and `LG_PRO_DAILY_LOT_TRACKING` carries all 9 patch markers (`Edit 1`, `Edit 2`, `Edit M1/M2/M3` ×3, `Edit 3` ×2, `Edit 5`, `Edit 4`).

**14. Final independent audit (3-agent adversarial workflow, run in parallel)**

Before treating this issue as code-complete, ran one last comprehensive audit workflow directly against the now-live procedure:

- **Agent 1 (MM branch)**: confirmed all 3 loops are correctly patched (the `CASE WHEN` correctly references alias `I` = `TLG_IT_ITEM`, already joined in each cursor's own FROM clause; downstream `CUR.TLG_IT_ITEM_PK` in the INSERT statements resolves correctly per standard PL/SQL). **Key finding**: the MM branch currently has NO real-world effect in DONGIL production — querying `ALL_DEPENDENCIES` (`referenced_name='LG_PRO_DAILY_LOT_TRACKING'`) returns exactly one dependent, `_JOB` (hardcoded `'DD'`), and `DBMS_JOB` (the only active scheduled job) also only calls `_JOB`. No other object in the schema calls it with `'MM'`.
- **Agent 2 (completeness sweep)**: scanned all 916 active lines, confirmed exactly **10 of 10 `INSERT INTO TLG_LOT_TRACKING_MAT`** points that write `TLG_IT_ITEM_PK` are normalized (7 directly via CASE WHEN, 3 downstream ration-insert points normalized transitively through the staging table `TLG_LOT_TRACKING_TEMP`, which itself is built straight from an already-clean `TLG_LOT_TRACKING_MAT`). Flagged one minor architectural weak point (not a current gap): the TEMP-build step and the 3 downstream ration inserts have no CASE WHEN of their own — they rely entirely (transitively) on upstream sources already being clean, a "single point of failure" if some future MAPPING-writing path ever forgets to normalize. Recommended adding a defensive CASE WHEN at the TEMP-build step, not urgent.
- **Agent 3 (live data state)**: confirmed August data (`STD_YM='202608'`) has **not been touched by the patch at all** — still 1,093 BRAMID rows / 505,088.00899 kg exactly as before (65 mixing lots, 9 physical lots, all created in a single month-end closing batch at 23:30-23:31 on Aug 31). Reconfirmed: since `_JOB` always derives `L_MONTH` from `SYSDATE`, running it on any day before October rolls in will always resolve `L_MONTH='202609'` — it never automatically touches August. Also found: September (the current month) still has live BRAMID (294 OPEN rows + 160 MAPPING rows) because the most recent nightly run (2026-09-07 23:30) happened BEFORE the patch was compiled (2026-09-08 13:39) — by design, tonight's nightly run (Sep 8→9) should self-heal it (the job soft-deletes and rebuilds the entire current month's MAT rows every night), but this has not yet been observed in practice.

Audit conclusion: the code is correct and complete, and is live in production, but two operational steps are still outstanding before the September 10 closing: (1) manually call `EXEC LG_PRO_DAILY_LOT_TRACKING('202608', 'DD');` to fix August, (2) confirm tonight's nightly run actually self-heals September as designed.

---

**15. Safety check before manually running the August fix (before executing anything)**

User asked whether August had already gone through monthly closing, and proposed manually running `EXEC LG_PRO_DAILY_LOT_TRACKING('202608','MM')` followed by `('202609','DD')`.

Queried table `TLG_CL_CLOSING_MAT_DETAIL` (the closing snapshot table) — confirmed August already has a snapshot (551 rows, `PHASE_NAME='WIP1'`, created 2026-09-05 08:20-08:41, `SUM_BEGIN=119,277.13` exactly matching July's ending balance shown in the original PM0407 screenshot). Found an anomaly: **all 551 snapshot rows carried item code `USOT3137`, with zero BRAMID rows** — while `TLG_LOT_TRACKING_MAT` (the real Lot Tracking table) still had all 1,093 BRAMID rows for August. Unclear whether this meant the rule was already correct at a different source, or the snapshot was simply missing data.

Ran a 4-agent parallel investigation workflow (`snapshot-origin`, `coverage-compare`, `mm-dry-run`, `approval-lock-check`) before allowing MM to run. Findings:

1. **Snapshot origin**: `TLG_CL_CLOSING_MAT_DETAIL` is written by `LG_PRO_KBPR00300_7` (+ its `_V2_7` twin) — a **completely separate** closing pipeline, unrelated to `LG_PRO_DAILY_LOT_TRACKING`, and it never reads `TLG_LOT_TRACKING_MAT`. Its sources are `TLG_CL_CLOSING_MAT_D` (a separate master closing table) + `TLG_ST_TRANSFER_D/M` (real warehouse transfer transactions into the MIX warehouse, `STATUS=3`) for ratio computation, plus its own prior-month snapshot carried forward **only when `MAT_END_QTY>0`**. All 65 BRAMID lots ARE present in July's snapshot (`SUM_OUT=505,088.0088`, closely matching the real `OUTPUT_QTY`) but `SUM_END=0` there — per this pipeline's own logic they were "fully consumed" by end of July, so they correctly did not carry into August (not a bug in that pipeline's own carry-forward step).
2. **Coverage comparison**: the 551-row August snapshot covers only **90 distinct LOT_NO values** (ranging `MIXEDF1-260729-01` to `260831-04`, a hard cutoff at 2026-07-29). `TLG_LOT_TRACKING_MAT` for August has **237 mixing lots** (65 BRAMID + 172 USOT3137). **147 mixing lots (every lot created 260627 through 260728, including all 65 BRAMID lots plus 82 pure-USOT3137 lots) are entirely absent from the snapshot under any code** — the snapshot's total is only 47.6% of true IN / 77.7% of true OUT.
3. **Dry-ran (SELECT-only) the MM branch's 3 cursors for '202608'**: confirmed it would NOT raise an exception (the `SELECT INTO` uses only `MAX()` aggregates, which always return one row even on a join miss — not the `NO_DATA_FOUND` originally assumed) but would produce **1,054 rows instead of the 3,546 currently present** — because the first step, `UPDATE ... SET DEL_IF=PK WHERE STD_YM LIKE '202608%'`, soft-deletes everything and then rebuilds only from the 90 LOT_NOs present in the snapshot. **The 65 BRAMID lots (505,088 kg) would genuinely vanish, not be relabeled.**
4. **Approval/lock status**: August WIP1 is already Approved (`TLG_CL_CLOSING_WIP_M`, `CONFIRM_DT=2026-09-05 08:57:38`, same state as July). `LG_PRO_DAILY_LOT_TRACKING` (all 916 lines, both DD and MM) never reads or writes `TLG_CL_CLOSING_WIP_M` at all — re-running does not touch or corrupt the Approved flag, but nothing blocks a re-run either. The `TLG_LOT_TRACKING_HIS` insert is already saturated for 202608 (62/62 rows, guarded by a `NOT EXISTS`) so re-running creates no duplicate closing-history rows.

**Conclusion: `EXEC LG_PRO_DAILY_LOT_TRACKING('202608','MM')` must NOT be run** — DONGIL's MM branch is exactly as the old developer comment says ("does not run, for dongil, needs fixing"): it depends on a snapshot table from an unrelated closing pipeline that currently covers only 90/237 of August's mixing lots. Prepared a lower-risk fallback, `results/fix_august_direct_update_OPTION_B.sql` — a direct UPDATE of the 1,093 BRAMID rows to USOT3137 (no quantity change).

User added important business context: every table/procedure whose name contains `LOT_TRACKING` belongs to one independent module, safe to fully clear/modify/rebuild; and from a business standpoint, a month that has already gone through monthly closing must have working Lot Tracking monthly data for that month. Given this, final recommendation: use the **already-patched procedure, called with `P_TYPE='DD'`** for both months (`EXEC LG_PRO_DAILY_LOT_TRACKING('202608','DD')` then `('202609','DD')`) instead of `'MM'` — the `'DD'`/`'MM'` labels are just internal computation-mechanism names (daily-mechanism vs monthly-snapshot-mechanism), not a hard requirement to use `'MM'` for an already-closed month; the DD branch is the one that was fully audited (see step 14), computing from real source data across all 237 mixing lots.

**16. User ran `DD('202608')` then `DD('202609')` — discovered a new bug: data loss in DAILY/transfer-in**

Post-run verification: raw-cotton item group 27 for August/September now shows only USOT3137 (the patch's core goal, achieved), but comparing quantities before/after by `STOCK_TYPE`:
- `OPEN` for August: 837 rows / 1,209,469.13 kg — exactly matches the pre-run figure (OK).
- **`DAILY` for August: 484 rows / 1,071,144.59 kg (before) → only 12 rows / 26,823.08 kg (after) — a ~97.5% loss.**
- `MAPPING` for August: 2,225 rows / 1,400,461.90 kg (before) → 1,588 rows / 1,087,663.36 kg (after) — a ~22% drop (initially suspected to be a cascading effect; see step 19 for the final conclusion).

Confirmed the true warehouse-transfer source data (`TLG_ST_TRANSFER_D/M`, `STATUS=3`, `TR_DATE LIKE '202608%'`, `W.PROCESS_TYPE='MIXED'`) remained fully intact (~1,084,642.69 kg) — nothing was lost at the source, so the defect had to be in the write logic, not missing source data.

**Root cause confirmed with a concrete example** (`SLIP_NO='TR26-0624'`, `TLG_WI_LINE_M_PK=8360`): the transfer-in cursor (source lines ~413-442) has a duplicate-prevention condition:

```sql
AND NOT EXISTS (SELECT 1 FROM TLG_LOT_TRACKING_MAT Z WHERE Z.DEL_IF = 0
  AND Z.TLG_WI_LINE_M_PK = L.PK AND Z.TRIN_TYPE = 'I60' AND Z.SLIP_NO = M.SLIP_NO)
```

This condition **does not filter by `STD_YM` and does not distinguish `STOCK_TYPE`**. `TRIN_TYPE='I60'` is set by both the DAILY insert block (line 455) AND the OPEN carry-over insert block for the following month (line 380). Because September had already been running nightly continuously (through 2026-09-07 23:30) and its carry-over reused the same `SLIP_NO`/`WI_LINE` into a September OPEN row, when `DD('202608')` was re-run, the `NOT EXISTS` found "already present" (the September OPEN row) and skipped inserting August's real DAILY row. Confirmed directly in the DB: the soft-deleted rows for `SLIP_NO='TR26-0624'`/`WI_LINE=8360` under STD_YM='202609' show a `CRT_DT` every single night from 2026-08-05 through 2026-09-07, proving this carry-over-reuses-SLIP_NO mechanism has existed for a long time.

**This is a pre-existing bug, NOT caused by the USOT3137 patch** (Edit 5 only touched this cursor's SELECT/GROUP BY, never its `NOT EXISTS`) — it was harmless until now because `_JOB` always ran sequentially for the current month, and there had never before been a scenario of re-running a past month while a later month already had data. All 484 of August's original DAILY rows are still physically intact in the DB (soft-deleted via `DEL_IF`, not truly lost) — recoverable.

**17. Swept the full 916-line procedure for every guard sharing the same bug class**

Ran an investigation workflow: scanned the entire procedure for every NOT EXISTS/NOT IN/MERGE/UNIQUE construct — found exactly **4 guard constructs** in all 916 lines:
- Line 98 (shared preamble, `TLG_LOT_TRACKING_HIS` day-marker) — SAFE, matches a full `YYYYMMDD` date.
- Line 438 (DD, DAILY transfer-in cursor) — DANGEROUS, confirmed root cause in step 16.
- Line 535 (DD, PROD-income cursor → `TLG_LOT_TRACKING_PROD`) — **DANGEROUS, same defect shape, never yet actually triggered** (`TLG_LOT_TRACKING_PROD.STD_YM` exists and is written by this same INSERT at line 492/497, but the guard never checks it).
- Line 567 (DD, MAPPING/ration cursor `CUR`) — SAFE, includes `A.STOCK_DATE = Z.TR_DATE` (full calendar-date match, finer-grained than month); this guard also protects the two O60 INSERTs later in the same loop (lines 693, 837) since they inherit `CUR`'s filter.

Confirmed the OPEN carry-over block (lines 313-394) has **no guard at all** — it inserts unconditionally after the month-scoped soft-delete — matching the empirical result (unchanged before/after). Confirmed the MM branch (lines 122-299) has zero guards — structurally immune to this entire bug class regardless of run order.

**18. Designed and verified the fix for the 2 dangerous guards**

Minimal fix — adding a filter condition to an existing WHERE clause, no structural change:

```sql
-- Guard at line 438 (DAILY transfer-in) — Before:
AND NOT EXISTS (SELECT 1 FROM TLG_LOT_TRACKING_MAT Z WHERE Z.DEL_IF = 0
  AND Z.TLG_WI_LINE_M_PK = L.PK AND Z.TRIN_TYPE = 'I60' AND Z.SLIP_NO = M.SLIP_NO)
-- After:
AND NOT EXISTS (SELECT 1 FROM TLG_LOT_TRACKING_MAT Z WHERE Z.DEL_IF = 0
  AND Z.TLG_WI_LINE_M_PK = L.PK AND Z.TRIN_TYPE = 'I60' AND Z.SLIP_NO = M.SLIP_NO
  AND Z.STD_YM = L_MONTH AND Z.STOCK_TYPE = 'DAILY')

-- Guard at line 535 (PROD-income) — Before:
AND NOT EXISTS (SELECT 1 FROM TLG_LOT_TRACKING_PROD Z WHERE Z.DEL_IF = 0
  AND Z.STOCK_NO = M.SLIP_NO AND Z.TLG_IT_ITEM_PK_PROD = D.ITEM_PK
  AND Z.TLG_IT_ITEM_PK_MIX = C.CHILD_PK AND D.LOT_NO = Z.LOT AND STOCK_TYPE = 'PROD')
-- After:
AND NOT EXISTS (SELECT 1 FROM TLG_LOT_TRACKING_PROD Z WHERE Z.DEL_IF = 0
  AND Z.STOCK_NO = M.SLIP_NO AND Z.TLG_IT_ITEM_PK_PROD = D.ITEM_PK
  AND Z.TLG_IT_ITEM_PK_MIX = C.CHILD_PK AND D.LOT_NO = Z.LOT AND STOCK_TYPE = 'PROD'
  AND Z.STD_YM = L_MONTH)
```

The line-438 guard needs both `STD_YM` and `STOCK_TYPE` because the same-month OPEN carry-over block also writes `TRIN_TYPE='I60'`/`STD_YM=L_MONTH` (differing only in `STOCK_TYPE='OPEN'`) — filtering on `STD_YM` alone could still wrongly collide with that same month's own OPEN row. Verified safety for both scenarios: (1) re-running the same month multiple times in a row stays idempotent, since `UPDATE ... SET DEL_IF=PK WHERE STD_YM LIKE L_MONTH||'%'` already soft-deletes that month's active rows BEFORE either guard runs; (2) re-running a past month while a later month already has data is now correctly allowed, since the later month's rows carry a different `STD_YM`.

Independently re-verified both edit points against live `ALL_SOURCE` (not trusting the earlier transcription) — matched character-for-character, no drift. Assembled `results/LG_PRO_DAILY_LOT_TRACKING_PATCHED_v3_guardfix.sql` (full `CREATE OR REPLACE`, edits applied via Edit-tool exact-string-match — both succeeded on the first try) and `results/LG_PRO_DAILY_LOT_TRACKING_live_before_guardfix_backup.sql` (reference copy). Verified structure: `BEGIN`/`END`/`IF`/`LOOP`/`CASE` counts matched exactly between the two files (27/59/12/28/17), a `diff` showed only the 2 intended change regions (2 comments + 2 added AND clauses), body grew by exactly 2 lines (917→919).

**19. Investigated whether `TLG_LOT_TRACKING_GD_M/GD_D` (backing the melt060/melt070 screens) needed to be part of the recovery**

Before deciding on the recovery approach, checked whether these two "Goods-Delivery Lot Tracking" tables (which back the raw-material-origin tracking screen for shipped goods) needed to be in scope for delete+rerun. Found and read `LG_SEL_MELT060_01` (builds the left-side tree, calls `LG_PRO_LOT_TRACKING_GD_M` as its first step to "rebuild-on-open" whenever a user searches a date range), and `LG_PRO_LOT_TRACKING_GD_M`/`LG_PRO_LOT_TRACKING_GD_D` (two SEPARATE procedures, entirely distinct from `LG_PRO_DAILY_LOT_TRACKING`).

Key findings:
- `LG_PRO_DAILY_LOT_TRACKING` does **not itself insert** `MAT_ITEM`/`MAT_LOT`/`MAT_KG` into `TLG_LOT_TRACKING_GD_D` — that is done by `LG_PRO_LOT_TRACKING_GD_D`, which reads directly from `TLG_LOT_TRACKING_MAT WHERE TROUT_TYPE='O60'` (the same table affected by the BRAMID/USOT3137 bug) — so `GD_D.MAT_ITEM` is directly tied to the bug being fixed.
- `LG_PRO_DAILY_LOT_TRACKING` does have its own cleanup of `GD_M/GD_D` (lines 62-79), but it ONLY soft-deletes `LEVEL_TYPE='LOT'` rows, and a row with `MAIL_YN='Y'` (already emailed/notified via morningmate) is NEVER touched by it (no month filter) — this is only a cleanup step, it never re-inserts anything; the actual rebuild only happens when someone reopens the melt060/melt070 screen for that date range.
- Secondary finding: `TLG_LOT_TRACKING_GD_M.DELI_DATE` is always blank (a separate bug in `LG_PRO_LOT_TRACKING_GD_M` at line 30: its cursor hardcodes `'' AS OUT_DATE` instead of `M.OUT_DATE`) — this column cannot be used to scope a month-based DELETE; the correct scope requires joining through `TLG_GD_OUTGO_M.OUT_DATE` via `TLG_GD_OUTGO_M_PK`.
- Empirically: checked the 2026-08-01 through 09-08 range directly — all 927/927 relevant GD_M `LEVEL_TYPE='LOT'` rows were already soft-deleted (`DEL_IF<>0`), all `MAIL_YN='N'` — already swept as a side effect of the two `EXEC DD` runs from step 16. No manual DELETE was needed for `GD_M/GD_D` this time, but this was purely lucky for this specific instance (no `MAIL_YN='Y'` rows happened to exist in the range) — this self-cleanup mechanism should not be relied upon for future recoveries.

**20. User's recovery decision: DELETE August+September data cleanly, then rerun DD**

User directed deleting all August+September data across every relevant `LOT_TRACKING` table, then rerunning DD. Swept all 9 tables whose name contains `LOT_TRACKING`: only **3 tables** have a `STD_YM` column AND are actually written by `LG_PRO_DAILY_LOT_TRACKING` (confirmed via the procedure's own commented-out `TRUNCATE` block at the top of its source, lines 5-9): `TLG_LOT_TRACKING_MAT` (92,630 rows for Aug+Sep, including all soft-deleted generations), `TLG_LOT_TRACKING_PROD` (13,096 rows), `TLG_LOT_TRACKING_HIS` (122 rows). `TLG_LOT_TRACKING_GD_M/GD_D` (already investigated in step 19 — no manual delete needed), `TLG_LOT_TRACKING_TEMP` (no month column at all, fully rebuilt on every run), `TLG_GD_LOT_TRACKING_D/M`/`TLG_LOT_TRACKING_PROD_BC` (similarly-named but not touched by this procedure) — out of scope.

Assembled `results/recovery_delete_and_rerun_202608_202609.sql`: hard `DELETE` on all 3 tables `WHERE STD_YM IN ('202608','202609')`, verify clean, then `EXEC LG_PRO_DAILY_LOT_TRACKING('202608','DD')` → `('202609','DD')`, then verify the final result. Accompanying recommendation: still compile the guard-fix patch (step 18) BEFORE running this script — clearing September resolves today's specific incident (no rows left for the buggy guard to collide with), but the root-cause bug still exists unpatched and would recur if a past month is ever re-run again while a later month already has data.

**21. User compiled the guard-fix patch + ran the recovery script — verified results**

Verified: August's DAILY is now **489 rows / 1,084,642.69 kg — an exact match** with the true source total (`TLG_ST_TRANSFER_D/M`) → confirms the guard fix works correctly, root cause resolved. `OPEN` 837 rows / 1,209,469.13 kg (unchanged). All of raw-cotton item group 27 now shows only USOT3137, no BRAMID — matching CESM-418's core goal.

Found one more thing needing clarification: August's `MAPPING` (ration output) = 1,588 rows / 1,087,663.36 kg — about 313K kg lower than the pre-incident baseline (2,225 rows / 1,400,461.90 kg). Critical detail: this exact figure (1,588 rows / 1,087,663.36 kg) is **identical** to the earlier BROKEN run (when DAILY had only 12 rows) — proving the MAPPING shortfall is unrelated to the guard bug just fixed (if it were related, it should have recovered/increased along with DAILY, but it didn't move by a single kg).

Launched a separate 3-agent investigation workflow — all 3 agents failed due to hitting the session usage limit, not a logic error. Switched to direct manual investigation:
- `TLG_PR_PROD_INCOME_M/D` for August, `STATUS=3`: 832 documents / 3,257,790.53 kg, `MAX_MOD=2026-09-05 08:57:35` — nearly exactly coincides with when August's WIP1 closing was Approved (`CONFIRM_DT=2026-09-05 08:57:38`) — suggesting these records were touched as part of the closing process itself, not evidence of an unusual edit/deletion. `STATUS=1` (not yet approved) is only 6 documents / 4,642.92 kg — far too small to explain the 313K kg gap.
- Read the ration mechanism's source directly (lines ~613-800): confirmed ration for a given (`SLIP_NO`, `WI_LINE`, `ITEM`) can only draw stock from `TLG_LOT_TRACKING_TEMP` rows matching that exact `WI_LINE_M_PK` (lines 670-671), not freely from the month's total stock. This means that even though aggregate supply (`OPEN+DAILY = 2,294,111.82 kg` today, even higher than the historical ~2,280,613.72 kg) is abundant, a local shortfall can still occur if stock isn't distributed correctly across the specific WI lines that need it.

**Conclusion**: the ~313K kg MAPPING shortfall is most likely a manifestation of the already-known pre-existing carry-over bug (step 5: "the wrong JOIN... means the carried-forward balance is never actually reduced by real consumption") combined with the "path-dependent" nature of the ration algorithm (a single fresh recompute produces a different result than the sequence of nightly recomputations that actually happened throughout August) — NOT a new bug introduced by today's patch (evidence: identical in both the broken and the fixed run). Does not violate CESM-418's core goal. Should be reported to business/DBA as a separate finding, does not block the September 10 closing.

**22. Discovered a real regression on the melt070 screen (Deli Lot Tracking V3) — directly caused by the CESM-418 patch**

Business (via internal chat) reported the melt070 screen (`/me/lt/melt070`) showing many rows with "Origin: none" + "0 files" for August.

Read the source of `LG_SEL_MELT070_02`: Origin/file data is looked up by joining `TLG_KB_COTTON_INCOME_D` (the raw-material receiving record — keeps the ORIGINAL item code, untouched by the patch) with `TLG_LOT_TRACKING_GD_D` (which gets its item code from `TLG_LOT_TRACKING_MAT` — already relabeled BRAMID→USOT3137 by the patch) matching on **both the item code AND `LOT_NO`** (two spots: line ~61 in CTE `TBL_LOT_INFO`, line ~204 in the main LEFT JOIN):

```sql
-- CTE TBL_LOT_INFO, line ~61 (before):
AND DI.TLG_IT_ITEM_PK = X1.TLG_IT_ITEM_PK_MAT
AND DI.LOT_NO = TRIM(X1.MAT_LOT)
-- Main LEFT JOIN, line ~204 (before):
AND L.TLG_IT_ITEM_PK_MAT (+) = Z2.TLG_IT_ITEM_PK_MAT
AND L.MAT_LOT (+) = TRIM(Z2.MAT_LOT)
```

Once the codes no longer match (121 BRAMID ≠ 122 USOT3137), the join returns NULL → Origin=NULL. The fallback heuristic (`ITEM_CODE LIKE 'USA%'`) doesn't rescue it either, since `'USOT3137'` doesn't match the `'USA%'` pattern (it starts with "USO", not "USA").

**Verified with real data**: all 4 sample `LOT_NO`s shown as "none/0 files" in the screenshot (`203/1076/26-01`, `303C/665/26-01`, `402A/660/26-01`, `402C/219/26-01`) were indeed **purchased and recorded as BRAMID** in `TLG_KB_COTTON_INCOME_D` — 100% confirming this is a direct consequence of the CESM-418 patch.

This screen has `AUSTRALIA/BRAZIL/USA_GIN_CODE` buttons — it serves raw-cotton origin certification for customs/export purposes, where Origin must reflect the true physical origin, not the internally normalized accounting label. Presented two options to the user: (A) fix melt070's join to key on `LOT_NO` alone (keeps the USOT3137 accounting record intact while displaying the true origin), (B) leave as-is. **User chose (A).**

Verified safety before fixing: confirmed **0 `LOT_NO`s in `TLG_KB_COTTON_INCOME_D` are associated with more than one distinct item code**, and **0 `LOT_NO`s are associated with more than one PO_DOC/Origin** — removing the item-code match condition from the join is completely safe, with no risk of fan-out/duplicate rows. Assembled `results/LG_SEL_MELT070_02_PATCHED_origin_join_fix.sql` — a full `CREATE OR REPLACE`, 2 edits tagged `-- [PATCH MELT070-ORIGIN-FIX]`, removing only the item-code match condition and keeping `LOT_NO` as the join key.

**23. Swept the full schema — found 5 more procedures with the same bug**

User asked to also check the "store popup" (the procedure behind the file-viewing popup opened from the "X files" link on the grid). Found `LG_SEL_MELT060_02_FILES` with the exact same defect. Swept the whole schema (every object referencing both `TLG_KB_COTTON_INCOME_D` AND `TLG_LOT_TRACKING_GD_D`) and found a total of **6 procedures** with this bug:

| Procedure | Screen/function | Bug points |
|---|---|---|
| `LG_SEL_MELT070_02` | melt070 main grid | 2 (fixed in step 22) |
| `LG_SEL_MELT060_02_FILES` | File-viewing popup | 1 |
| `LG_SEL_MELT021_02` | melt021 screen | 2 |
| `LG_SEL_MELT060_02` | melt060 main grid (V2 of melt070) | 2 |
| `LG_SEL_MO00010` | File popup variant (uses an `IN` subquery instead of `EXISTS`) | 1 |
| `LG_SEL_MELT060_SEND_FLOW_MAIL` | Sends the customer-facing traceability-report email | 4 (2 duplicated 'FLOW'/'MAIL' blocks, 2 points each) |

Pulled each procedure's full source and applied the exact same fix principle confirmed in step 22 (remove the item-code match condition, keep `LOT_NO` only). Re-verified via grep across all 6 files: every item-code condition was correctly removed, every `LOT_NO` condition remained intact. Assembled the remaining 5 patch files (`LG_SEL_MELT060_02_FILES_PATCHED_origin_join_fix.sql`, `LG_SEL_MELT021_02_PATCHED_origin_join_fix.sql`, `LG_SEL_MELT060_02_PATCHED_origin_join_fix.sql`, `LG_SEL_MO00010_PATCHED_origin_join_fix.sql`, `LG_SEL_MELT060_SEND_FLOW_MAIL_PATCHED_origin_join_fix.sql`) — plus the one from step 22, all 6 are now ready to compile. Notably, `LG_SEL_MELT060_SEND_FLOW_MAIL` sends the traceability-report email directly to customers — if left unpatched, emails would carry missing/incorrect origin information for exactly the lots the patch relabeled.

---

**Open items still awaiting user/business confirmation (as of end of 2026-09-08):**
1. Whether keeping `LOT_NO` unchanged after a material-code swap is acceptable, or a different lot reference is needed.
2. Whether the resulting "orphaned" BRAMID book balance (never consumed by this job again) has any downstream inventory/costing impact.
3. Confirm the patch's permanent, unconditional scope (all 8 codes in item group 27, no time limit) is really the intended long-term behavior.
4. Whether the unrelated carry-over `OUTER JOIN` bug (found in step 5) should be filed as its own separate ticket.
5. The MM branch currently has no effect for DONGIL (nothing calls it, and its own snapshot data source is also incomplete — see step 15) — is it worth keeping the patch on this branch, or is it purely defensive for other sites sharing the same procedure?
6. Whether to add the defensive CASE WHEN at the `TLG_LOT_TRACKING_TEMP`-build step (Agent 2's recommendation in step 14, not mandatory).
7. **[NEW]** The ~313K kg MAPPING gap (step 21, suspected related to the already-known pre-existing carry-over bug) — does this need further dedicated investigation to precisely quantify the impact, or is it acceptable to treat this as a consequence of the already-known bug and address it together when that bug is reported?
8. **[NEW]** After compiling the 6 patch files in step 23 — should other screens/procedures (outside the `TLG_KB_COTTON_INCOME_D`+`TLG_LOT_TRACKING_GD_D` scope) also be swept for potential impact from the item-code relabeling, or are these 6 procedures the complete scope?

---

## 2026-09-09

**1. Independently verifying business's (Le Tien's) claim of "100% US since August" via the Production/warehouse-transfer module**

Business (Le Tien) responded via internal chat, asserting "since August we've been using 100% Origin US" (citing Lot No 2608, whose Mat_item column shows USOT3137). Asked to prove this independently, with progressively tighter constraints: (1) without going through `TLG_LOT_TRACKING_GD_D`/`GD_M`, (2) without going through ANY table named `LOT_TRACKING`.

Built a chain entirely within the Production/warehouse-transfer module: `TLG_WI_LINE_M.SLIP_NO` (= the real MIX_LOT code; note the `MIX_LOT_NO` column on the same table is always NULL/unused, verified directly) → `TLG_WI_LINE_M.TLG_ST_TRANSFER_REQ_M_PK` → `TLG_ST_TRANSFER_REQ_M/D` → `TLG_ST_TRANSFER_D/M` (real warehouse-transfer vouchers, `STATUS=3`) → `TLG_IN_WAREHOUSE` (filtered `PROCESS_TYPE='MIXED'`):

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

For 3 sample mixing lots (`MIXEDF1-260627-02`, `MIXEDF1-260701-01`, `MIXEDF1-260721-03`) → 28 rows, both BRAMID and USOT3137 have real warehouse-transfer vouchers dated exactly to each lot's creation date. Expanded to the full June-July 2026 range (found and fixed a missing-parentheses `AND`/`OR` precedence bug in the first run, which had let the `DEL_IF`/`STATUS`/`PROCESS_TYPE` filters be bypassed for one month's branch): **USOT3137 165 mixing lots / 1,100,141.43kg; BRAMID 138 mixing lots / 1,091,235.31kg** — both have real transfer vouchers, not fake/erroneous data. => Business's "100% US since August" claim is not accurate at the historical-data layer.

**2. Preparing (not executing) a DELETE script for `TLG_LOT_TRACKING_GD_D`/`GD_M`, scoped to August+September 2026**

Per user request, drafted `results/delete_gd_m_gd_d_aug_sep_2026.sql`: hard DELETE `TLG_LOT_TRACKING_GD_D` (16,283 rows) first, then `TLG_LOT_TRACKING_GD_M` (1,413 rows), filtered by `WHERE TLG_GD_OUTGO_M.OUT_DATE BETWEEN '20260801' AND '20260930'`. The scope (only these 2 months, not all-time history) was reconfirmed with the user before drafting.

**3. Re-verifying OPEN balance after the guard-fix recovery (run on 2026-09-08) — confirming allocation is already 100% USOT3137**

```sql
SELECT M.STD_YM, I.ITEM_CODE, M.STOCK_TYPE, ROUND(SUM(M.INPUT_QTY),2) IN_QTY
FROM TLG_LOT_TRACKING_MAT M JOIN TLG_IT_ITEM I ON I.PK = M.TLG_IT_ITEM_PK
WHERE M.DEL_IF = 0 AND I.ITEM_CODE IN ('BRAMID','USOT3137')
AND M.STD_YM IN ('202607','202608','202609') AND M.STOCK_TYPE = 'OPEN'
GROUP BY M.STD_YM, I.ITEM_CODE, M.STOCK_TYPE
```

Result: 202607 (pre-patch) still shows BRAMID 69,272.57kg alongside USOT3137 45,395.22kg; **202608 and 202609 (post patch+recovery) show only USOT3137** (1,209,469.13kg and 2,294,111.82kg), **0 BRAMID rows**. Broader re-verification: `TLG_LOT_TRACKING_MAT`/`TLG_LOT_TRACKING_PROD` for STD_YM 202608/202609, across every `STOCK_TYPE`, referencing BRAMID (PK=121) → **0 rows**. Confirms the CESM-418 patch is working exactly as designed at the allocation layer.

**4. User ran the GD_M/GD_D delete script — found a false assumption baked into the script itself; only a partial rebuild occurred**

After the user ran the script from step 2, only **390/1,413 (GD_M)** and **600/16,283 (GD_D)** rows came back — not a full rebuild as the script's own comment had claimed. Re-read the full source of `LG_PRO_LOT_TRACKING_GD_M`/`LG_PRO_LOT_TRACKING_GD_D`:

- `LG_PRO_LOT_TRACKING_GD_M(P_DT_FROM, P_DT_TO, P_SEL_OPTION, P_TXT_SEARCHLIST, P_SEL_INTERFACE_YN, P_PARTNER, P_PARAM2, P_PARAM3, P_PARAM4, P_LANG, P_CRT_BY)` — only `P_DT_FROM`/`P_DT_TO` actually affect the filter/insert logic (the main cursor filters `M.OUT_DATE BETWEEN P_DT_FROM AND P_DT_TO`); 7 of the remaining 11 parameters appear in no WHERE/SELECT condition at all, and only `P_CRT_BY` is used, as the `MOD_BY` audit value. At the end it calls `LG_PRO_LOT_TRACKING_GD_D(P_DT_FROM, P_DT_TO)` with the same date range.
- => The "rebuild-on-open" mechanism **only rebuilds the exact date range passed in** when the UI calls it (matching whatever the user searched on-screen), and does NOT automatically cover the whole month as originally assumed in the delete script's comment.

Drafted `results/force_full_rebuild_gd_m_gd_d_aug_sep_2026.sql`:
```sql
EXEC LG_PRO_LOT_TRACKING_GD_M('20260801','20260930', NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, 'RECOVERY-CESM418');
```
Verified it's safe to run: `LG_PRO_LOT_TRACKING_GD_M` uses a `SELECT INTO`/`EXCEPTION` existence-check pattern before inserting each CUST/PO_NO/DELI/IT/LOT node — idempotent; `LG_PRO_LOT_TRACKING_GD_D` only processes GD_M rows where `NVL(MAP_QTY,0) < NVL(QTY,0)` — already-fully-mapped rows are skipped automatically on rerun.

**5. Business sent a melt070 screenshot (Lot 2608RN) showing several Origin=none/0-files rows; investigated root cause**

Read the full source of `LG_SEL_MELT070_02` — Origin/files are computed from CTE `TBL_LOT_INFO`: INNER JOIN `TLG_KB_COTTON_INCOME_D` (DI) → `TLG_PO_DOC_D0` (D1) → `TLG_PO_DOC_M` (M1) → `TES_FILE` (Z1), filtered by:
```sql
AND EXISTS (SELECT 1 FROM TLG_LOT_TRACKING_GD_D X1 WHERE X1.DEL_IF = 0
  AND EXISTS (SELECT 1 FROM TABLE(SPLIT(P_PARAM1,';')) WHERE COLUMN_VALUE = X1.TLG_LOT_TRACKING_GD_m_PK)
  AND DI.LOT_NO = TRIM(X1.MAT_LOT))
```
then OUTER JOINed via `L.MAT_LOT(+) = TRIM(Z2.MAT_LOT)` into the main query (the Option A patch from 09-08 — removing the item-code match condition, keeping only `LOT_NO` — was already applied here).

Traced the 4 "none" mat_lots directly (`203/1076/26-01`, `303C/665/26-01`, `402A/660/26-01`, `402C/219/26-01`):
```sql
SELECT LOT_NO, TLG_PO_DOC_M_PK, TLG_IT_ITEM_PK, DEL_IF FROM TLG_KB_COTTON_INCOME_D
WHERE LOT_NO LIKE '203/1076/26%' OR LOT_NO LIKE '303C/665/26%' OR ...
```
→ all 4 have `TLG_IT_ITEM_PK=121` (BRAMID) in the original purchase record (the mat_lot correctly showing "USA/7 files" — `301A/1584/26-01` — has item=122, genuine USOT3137, never relabeled). Re-ran the full INNER JOIN chain for `LOT_NO='203/1076/26-01'` alone, bypassing the EXISTS/P_PARAM1 condition entirely:
```sql
-- (chain DI->D1->M1->Z1, WHERE DI.LOT_NO = '203/1076/26-01', no EXISTS)
```
→ **still returns data**: `ORIGIN='BRA'`, a total of **7 files** (TYPE01/02/03/05/06/07 with 1 file each, TYPE07 with 2). Cross-checked `TLG_LOT_TRACKING_GD_D.MAT_LOT` for lot VP2608RN's entry for this mat_lot: stored value is exactly `"203/1076/26-01"` (14 characters, an exact match to `TLG_KB_COTTON_INCOME_D.LOT_NO`, no formatting/whitespace drift).

=> **Conclusion: the source data (`TLG_KB_COTTON_INCOME_D`+`TLG_PO_DOC_M/D0`+`TES_FILE`) is fully intact — NOT lost or corrupted by the GD_M/GD_D delete+rebuild step.** "None/0 files" is a SEPARATE bug in `LG_SEL_MELT070_02`'s own `EXISTS`/`P_PARAM1` condition (which restricts by the list of GD_M.PKs currently in the UI's search results) — the exact mechanism causing the mismatch isn't 100% pinned down yet (would need the real `P_PARAM1` value from the browser to verify further), but it is definitively not data loss.

**6. Business pushed back with real physical-warehouse evidence (IV0401 W/H Stock Checking screen, `lg_sel_bisc00020`)**

Business cited: `exec lg_sel_bisc00020('20260722','20260831','10','428','','','N','ENG','Y',:p_rtn_value)`, asserting BRAMID has been fully depleted in the warehouse since 2026-07-22. Read the full source of `lg_sel_bisc00020` — it uses `TLG_SA_STOCK_CLOSING_M/D` (real period-end stock-balance table) + `TLG_IN_STOCKTR` (real inbound/outbound transactions), **going through no LOT_TRACKING table whatsoever**.

Independently recomputed for warehouse PK=428 (M011-COTTON W/H Fac1, matching `WH_TYPE='10'`):
```sql
-- BEGIN_QTY(7/22) = closing_end_qty(most recent closing before 7/22)
--                 + SUM(IN_QTY-OUT_QTY from TLG_IN_STOCKTR, TR_DATE between the closing and 7/22)
```
→ matched the business's screenshot numbers EXACTLY (Begin Qty USOT3137 = 1,974,477.71, Total In = 713,165.00, Total Out = 1,444,729.04). For BRAMID at M011 specifically: Begin Qty(7/22) = 1,649,051.5 + (-1,649,051.5) = **0.00kg**; Total In/Out (7/22→now) = **0/0**. Extended the check to the only other cotton raw-material warehouse (M001-IQC Cotton W/H Fac1): also **~0 (slightly negative, effectively unavailable)**, 0 transactions since 7/22. => **Business is entirely correct about the physical-warehouse state.**

**7. Cross-referenced against the Production/warehouse-transfer chain (step 1) to pin down the exact cutoff date**

```sql
SELECT L.SLIP_NO MIX_LOT, MAX(M.TR_DATE) LAST_TR_DATE, ROUND(SUM(D.TR_QTY),2) TONG_KG_BRAMID
FROM TLG_WI_LINE_M L JOIN TLG_ST_TRANSFER_REQ_M R ON ... JOIN TLG_IN_WAREHOUSE W ON ...
WHERE ... AND W.PROCESS_TYPE='MIXED' AND D.TR_ITEM_PK=121
GROUP BY L.SLIP_NO ORDER BY LAST_TR_DATE DESC
```
→ The last mixing lot that genuinely used BRAMID was **2026-07-21** (`MIXEDF1-260721-01/02/03`, ~21,730kg total) — no mixing lot after that date used BRAMID. This matches exactly with the warehouse reporting depletion from 7/22 (step 6). **Confirmed cutoff: from 2026-07-22 onward, every mixing lot is genuinely 100% USOT3137; mixing lots from 2026-07-21 and earlier genuinely contain BRAMID — a fixed physical fact that no system operation can reverse.**

**8. Synthesized the business explanation for the customer-facing team — distinguishing MIX time from DELIVERY time**

Final conclusion, communicated back to business: the "still shows Brazil in August" data reflects two distinct points in time within the same chain — (a) MIX (raw-material blending, a one-time event, which genuinely stopped on 7/21) versus (b) DELI (finished-goods delivery to the customer, spread over subsequent weeks, potentially into August-September). A lot delivered in August that was produced from a batch mixed with BRAMID before 7/21 genuinely still contains Brazilian cotton — this is not a new mix, and not a data error.

Also clarified an important implication for the next business decision: the item code now showing "USOT3137" for these lots (even after CESM-418's normalization) **does not reflect the true raw material actually used** — the original purchase records for these lots (verified in step 5) are still genuine Brazil purchase records, with complete supporting files. If fully accurate-to-reality data is wanted, lots mixed before 7/22 should display Brazil again (both item and Origin); lots mixed from 7/22 onward (step 7) are genuinely, verifiably 100% USOT3137 and need no correction.

---

**Open items still awaiting user/business confirmation (added 2026-09-09):**
9. **[NEW]** Whether to override the Origin display to USA (the drafted V2 patch for `LG_SEL_MELT070_02`/`MELT021_02`/`MELT060_02`) for lots confirmed to have genuine, fully-attached Brazil purchase certificates — this is a compliance/origin-certification decision, not merely a display choice. `LG_SEL_MELT060_SEND_FLOW_MAIL` (sends the traceability email directly to customers) still has its V2 fix uncompiled (`_PENDING_confirmation.sql`), pending exactly this decision.
10. **[NEW]** Whether the item/Origin for lots mixed BEFORE 2026-07-22 should be corrected back to show Brazil (instead of the normalized USOT3137) to match physical reality — or kept as the fully-normalized 100% USOT3137 CESM-418 originally required (accepting this as an internal accounting label, not the true physical origin).
11. **[NEW]** The separate bug in `LG_SEL_MELT070_02`'s `EXISTS`/`P_PARAM1` condition (step 5, which hides Origin/files even though the source data is intact) — should this be investigated further to pin down the exact mechanism and patched, independent of the business decisions in items 9/10?
12. **[NEW]** Confirmed the delete scope for `results/delete_gd_m_gd_d_aug_sep_2026.sql` is August+September 2026 only — this script and `results/force_full_rebuild_gd_m_gd_d_aug_sep_2026.sql` (step 4) are both still awaiting the user to run them via Toad.
