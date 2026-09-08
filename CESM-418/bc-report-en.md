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

**10. BC added a "system change not required" conclusion — re-verified, found not accurate**

The user sent a PM0407 screenshot (July 2026, Factory 1) along with a conclusion the BC had added
to the ticket, stating that real July 2026 data had been checked and the WIP material inventory
consisted entirely of approved US Cotton (USOT3137), with no mixed/unapproved material (e.g.
BRAMID) being carried over — and therefore no system change was needed.

Rather than accepting this at face value, re-checked by querying the exact same scope the BC had
looked at (same factory, same finished-goods item, same two months of July and August 2026). The
result showed unapproved material (BRAMID) still accounted for roughly **42% of total weight**
within that exact scope in July, and this amount carried over unchanged into August — directly
contradicting the BC's conclusion.

Identified why the BC missed it: the screen only showed the MOST RECENT mixing lots (created in
late July), which genuinely were 100% approved material — but OLDER mixing lots (from late June)
still held unapproved material, further down/requiring more scrolling in the detail grid, which
the BC may not have reviewed fully. Conclusion: disagreed with "system change not required",
responded to the BC with specific figures, and kept the recommendation to proceed with the fix.

**11. Found: a shared processing version already exists for other factories — the fix turned out
to already be in the live system**

The user noted that at some other factories sharing the same platform, the nightly lot-tracking
job simply "calls into" a separate main processing program (which can be passed the month and the
processing type to run), rather than containing all the logic itself like DONGIL's current job —
and asked for DONGIL to be reorganized the same way.

Before making that change, checked and found DONGIL already had such a main processing program in
place (not previously known), structured almost identically in parallel to DONGIL's current job.
While comparing the two programs' content, discovered that **the fix drafted earlier (the 5-point
version) had already been put into the live system** — asked the user and confirmed: they had
checked and applied the fix to the production system themselves (having backed up the old version
first).

Warned the user of a risk: if the original request were followed literally (turning the nightly job
into something that just calls the separate main processing program), and that program did NOT yet
have the fix, the change would UNDO the fix that had just been applied.

**12. Per request, moved the entire fix into the main processing program**

The user clarified the direction: the shared main processing program (able to run for both MONTHLY
closing and DAILY runs) would hold all the logic and the fix; DONGIL's nightly job would become
nothing more than a call into that program, matching how other factories already operate.

For the first time, carefully reviewed the section handling MONTHLY closing (previously never
analyzed, believed to be unused) — found it to be a simpler mechanism than the daily-processing
section, pulling data directly from a snapshot table already computed at closing time. Applied the
same fix principle (replace unapproved material codes with the approved one, keep quantity
unchanged) to all 3 material-recording points within this section.

Prepared a complete fix for the main processing program — **8 fix points in total** (covering both
the daily-processing and monthly-processing sections), carefully checked with a dedicated tool to
confirm no structural errors before sending to the user.

Also prepared a new version of the current nightly job — now just 4 lines, simply calling into the
main processing program, matching the pattern used at other factories. Noted the mandatory order
when applying: the main processing program must be updated FIRST, then the nightly job SECOND — if
done in reverse order, there would be a brief window where the nightly job calls into the
not-yet-fixed version. Added benefit: it's now possible to deliberately re-run for one specific
month (e.g. August) by passing that month directly as a parameter, instead of the old approach of
temporarily editing the code.

**13. Confirmed both changes were successfully applied to the live system**

Re-checked the system: both the main processing program and the new nightly job were successfully
updated, in the correct order (main program first, nightly job second, 21 seconds apart), with no
errors. Confirmed the nightly job is now genuinely just a call-through (4 lines), and the main
processing program carries clear markers for all 8 fix points.

**14. Final review (3 independent reviewers, working in parallel)**

Before treating this as system-complete, ran one last comprehensive review directly against the
version now live in the system, covering 3 independent angles:

- **Reviewed the MONTHLY-closing section**: confirmed all 3 fix points in this section are
  technically correct. However, found that the MONTHLY-closing section currently has **no real
  effect for DONGIL** — a system-wide check confirmed nothing (not even the nightly job) actually
  calls into this section; only the DAILY-processing section actually runs. This may be groundwork
  reserved for other factories sharing the platform, not yet used at DONGIL.
- **Completeness sweep**: scanned the entire program, confirmed all **10 of 10 points** that record
  a material code into the system are correctly normalized (7 fixed directly, the remaining 3
  automatically correct because they inherit from the points already fixed upstream). One
  improvement suggestion (not urgent): add an extra safety check at the temporary-staging step, so
  the system stays protected even if a new data-recording path is added later and someone forgets
  to apply the normalization there.
- **Checked real data**: confirmed AUGUST data has **not been corrected at all yet** — still over
  1,000 rows / over 500 tons of unapproved material sitting exactly as before, because the fix only
  affects the NEXT run, not existing data already on the books. Also, because the nightly job always
  calculates based on the current system month, running it on any day before October rolls in will
  always process SEPTEMBER, not automatically touch the already-closed August ledger — **a separate
  manual run specifically for August is required** before the September 10 closing. Also found:
  September's current data (the month now in progress) still carries leftover unapproved material
  from last night's run (which happened before the fix went live) — by design, tonight's run should
  clean this up automatically, but this should be re-checked tomorrow morning to be sure.

Conclusion: the fix is correct, complete, and already live in the system — but two things still need
to happen urgently before the September 10 closing: (1) proactively re-run the process specifically
for August, (2) re-check tomorrow that tonight's run actually cleaned up September as expected.

---

**Open items still awaiting confirmation from the user/DONGIL side:**
1. Whether the original lot number is kept unchanged when the material code is swapped.
2. Whether the resulting "orphaned" unapproved-material balance (which will never be reduced again
   on this job's books) has any impact on how DONGIL tracks inventory/costing per material type.
3. **[URGENT, not yet done]** A separate manual run specifically for August is needed before the
   September 10 closing — the fix does not retroactively correct data already on the books.
4. Confirm the permanent, unlimited-time scope (applying to the entire raw-cotton material group) is
   really the intended long-term behavior.
5. Whether the secondary issue found in step 5 (balance not reduced by consumption) should be
   reported as a separate request/ticket.
6. **[NEW]** The MONTHLY-closing section currently has no effect for DONGIL (nothing calls it) — is
   it worth keeping the fix there, or is it purely a safeguard for other factories sharing the same
   program?
7. **[NEW]** Whether to add the extra safety check at the temporary-staging step (reviewer's
   suggestion, not mandatory).
8. **[NEW, needs follow-up]** Re-confirm after tonight's run: does September's data actually get
   fully cleaned up to 100% approved material as expected.
