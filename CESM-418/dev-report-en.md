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

**Open items still awaiting user/business confirmation:**
1. Whether keeping `LOT_NO` unchanged after a material-code swap is acceptable, or a different lot reference is needed.
2. Whether the resulting "orphaned" BRAMID book balance (never consumed by this job again) has any downstream inventory/costing impact.
3. **[URGENT, not yet done]** Need to manually call `EXEC LG_PRO_DAILY_LOT_TRACKING('202608', 'DD');` to fix August before the September 10 closing — the patch does not retroactively fix existing data.
4. Confirm the patch's permanent, unconditional scope (all 8 codes in item group 27, no time limit) is really the intended long-term behavior.
5. Whether the unrelated carry-over `OUTER JOIN` bug (found in step 5) should be filed as its own separate ticket.
6. **[NEW]** The MM branch currently has no effect for DONGIL (nothing calls it) — is it worth keeping the patch on this branch, or is it purely defensive for other sites sharing the same procedure?
7. **[NEW]** Whether to add the defensive CASE WHEN at the `TLG_LOT_TRACKING_TEMP`-build step (Agent 2's recommendation, not mandatory).
8. **[NEW, needs follow-up]** Re-confirm after tonight's nightly run (Sep 8→9): does September's data actually self-heal to 100% USOT3137 as designed.
