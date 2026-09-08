# BC Report — CESM-418

**Task:** [DONGIL][Requirement] Enforce 100% USOT3137 Allocation Rule for All Finished Goods Lots in Monthly Lot Tracking (August 2026)

---

## 2026-09-08

**1. Identified the correct job and related data**

Identified this as the nightly lot-tracking job that automatically recalculates all data for the
current month every night (takes no parameters, always uses the current system month). Identified
the related screen (PM0407 - Material Detail tab) and the 8 raw-cotton material codes currently in
use in the system, including the approved code (USOT3137) and unapproved codes (e.g. BRAMID).

**2. Confirmed the "carry-over across months" issue with real data**

Checking real data showed: every night when the job runs, it wipes and recalculates all data for
the current month, then carries forward the unused material balance from the previous month into
the current month — but keeps the OLD material code unchanged (even when that code is unapproved).
Specific verification: one mixing lot still has over 200 tons of unapproved material sitting on the
books, and this exact figure repeats unchanged across 3 consecutive months without ever decreasing
or being replaced.

**3. Confirmed the material-allocation mechanism for finished-goods lots**

Confirmed the job allocates material to each finished-goods lot according to the blend ratio fixed
at production-planning time (Work Instruction) — e.g. a plan may fix 60% material A / 40% material
B, and this ratio never changes over time.

**4. Drafted an initial fix (2 points)**

Drafted a fix: at the 2 points identified (carry-over across months, and ratio-based allocation),
automatically replace every unapproved material code with the approved code (USOT3137), keeping the
total quantity unchanged — no quantity is altered, only the material-code label.

**5. Independent cross-check — found 1 more important gap that had been missed**

Ran an independent cross-check round (one person re-derived the entire job from scratch without
seeing the drafted fix, plus two separate reviewers challenging it from a technical angle and from a
business/accounting-risk angle). Results:
- Found: besides the 2 points already fixed, there was a 3rd point missed entirely — where material
  actually transferred into the mixing warehouse gets recorded. Checking real data showed this is
  actually the LARGEST source of unapproved material (over 4,300 tons flowed through this path),
  larger than the carry-over volume. This point has now been added to the fix.
- Confirmed the 2 original fix points are correct and correctly scoped (only affects the raw-cotton
  material group, does not touch other materials).
- Found an additional pre-existing system issue (unrelated to this request): the material balance
  carried over across months is never reduced by what was actually consumed — one specific lot's
  balance stays exactly unchanged for 3 consecutive months despite real consumption being recorded
  each month. This issue affects ALL material types (not specific to this USOT3137 case), so it is
  recommended to report it as a separate request rather than bundling it into this fix.
- Confirmed an additional operational issue: because the job always calculates based on the current
  system month (the month cannot be chosen), if the job runs on 2026-09-10 as scheduled, the system
  will already be in September at that point, so the job will recalculate September, NOT the
  already-closed August ledger — a separate manual step is needed to correctly re-run it for August.

**6. Added the missing fix point**

Added the fix for the 3rd point found in step 5 (material transferred into the mixing warehouse),
following the same principle: only change the material-code label, never the quantity.

**7. At request, prepared a complete, ready-to-apply fix for the DBA**

Combined the job's full original content with all 3 fix points into one complete file, with each
change clearly marked so the reviewer can cross-check easily.

**8. Handled a compile error when test-running the prepared file**

The user test-ran the file using an Oracle admin tool (Toad) and reported a syntax error. Carefully
checked using a separate comparison tool, confirming the 3 fix points were not the cause at all. To
be completely certain, also created a "control" version — the original content with nothing fixed
at all — for the user to test in parallel: the control version failed with the exact same error,
proving the issue was in the copying/reconstruction of the original content, not in the fixed
content.

Root cause identified: when the original job content was copied, it stopped just short of
capturing the very last closing line (a nested block in the middle had all its old code commented
out, making it easy to mistake for the end of the file, but there was actually one more real closing
line further down).

**9. Fixed**

Added back the one missing closing line to both the fixed version and the control version, re-checked
with the dedicated tool and confirmed no structural errors remain. Sent back to the user to try
running again.

---

**Open items still awaiting confirmation from the user/DONGIL side before this goes to production:**
1. Whether the original lot number is kept unchanged when the material code is swapped.
2. Whether the resulting "orphaned" unapproved-material balance (which will never be reduced again
   on this job's books) has any impact on how DONGIL tracks inventory/costing per material type.
3. Confirm the process for re-running August correctly matches DONGIL/consultant's expectations.
4. Confirm the permanent, unlimited-time scope (applying to the entire raw-cotton material group) is
   really the intended long-term behavior.
5. Whether the secondary issue found in step 5 (balance not reduced by consumption) should be
   reported as a separate request/ticket.
