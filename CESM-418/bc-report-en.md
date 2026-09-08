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

**15. Safety check before manually re-running the fix for August**

Before manually re-running the process for August, checked whether August had already gone through monthly closing and whether the "monthly closing" mode of the process was safe to use. Found that August already has a closing snapshot on record, but a comparison revealed something unexpected: that closing snapshot showed a completely different — and much smaller — set of production lots than what the live tracking data held, and every one of those already-approved-material lots in the snapshot was correctly labeled, while the unapproved-material lots simply did not appear in it at all under any label.

Ran a dedicated 4-person-equivalent investigation before allowing that "monthly closing" mode to run. Findings: that snapshot is generated by an entirely separate, unrelated closing process (not the process this fix touches), which only captured about a third of August's production lots — the two-thirds it left out includes every one of the lots that still carry unapproved material. Test-simulating that monthly-closing mode confirmed it would not error out, but it would silently make roughly 500 tons of real inventory data disappear entirely rather than relabeling it, because it would wipe and rebuild August's records from that incomplete snapshot. Also confirmed that August's closing approval status would not be disturbed either way, and re-running would not create any duplicate closing-history records.

**Conclusion: the "monthly closing" mode must not be used for August.** Prepared a lower-risk fallback (a direct correction of the existing unapproved-material rows, changing only the label, not the quantity) as an alternative. The user then confirmed important business context: the lot-tracking data is a self-contained module, safe to fully clear and rebuild, and that any month already through monthly closing is required to have working monthly lot-tracking data. Based on that, the final recommendation was to use the regular (daily-mechanism) processing mode for both August and September instead of the monthly-snapshot mode — the regular mode is the one that had already been fully verified and computes from complete, real source data across all of August's production lots, not from the incomplete snapshot.

**16. User re-ran the process for August then September — discovered a new bug: real data loss**

After the re-run, unapproved material was correctly gone from the books for both August and September — the ticket's core goal was achieved. However, comparing before/after quantities surfaced a new issue: the "material transferred into the mixing warehouse" figures for August dropped by about 97.5% — from roughly 1,071 tons down to just 27 tons — while the opening-balance figures stayed exactly correct. Confirmed the real warehouse-transfer transactions themselves were still fully intact in the source records (about 1,085 tons) — nothing was actually lost at the source, so the defect had to be in the recording logic.

Root cause confirmed with a specific example: a duplicate-prevention check in the process (there to stop the same transfer record being recorded twice) does not account for which month it's checking — it only looks at whether a matching record already exists anywhere, regardless of month. Because September had already been running nightly and had already carried forward a reference to the same underlying transfer document into its own opening balance, re-running August's process saw that reference and wrongly concluded "already recorded," so it skipped recording August's real transfer-in figure. This is a **pre-existing defect, not caused by this fix** — it was harmless for years because the process had always run in strict forward-only order, and this was the first time a past month was ever re-run while a later month already had data. Nothing was truly lost — the original correct figures are still sitting in the database (marked inactive, not deleted), and are recoverable.

**17. Swept the entire process for every instance of this same defect class**

Checked the entire process end-to-end for every place with this kind of duplicate-prevention logic. Found exactly one other spot with the identical defect (affecting a related finished-goods production record, not yet observed to have actually caused any real-world data loss, but carrying the same latent risk) — added it to the same fix. Confirmed every other similar check in the process is safe (they all match on a precise calendar date, not just the month, so they don't have this problem), and confirmed the monthly-snapshot mode used for the process's month-end closing has no checks of this kind at all, so it is not affected by this issue either way.

**18. Designed and verified the fix for both defective checks**

Designed the minimal fix: add the missing month (and, for one of the two, the specific record-type) condition to each duplicate-prevention check, so it only considers records from the exact month currently being processed rather than any month. Verified this is safe in both directions: re-running the same month repeatedly stays fully safe (the existing wipe-and-rebuild step already clears that month's own records first), and re-running a past month while a later month already has data is now correctly allowed rather than blocked. Independently re-confirmed the exact wording of both defective checks directly against the live system (not relying on earlier notes) before applying the fix, to rule out any drift. Structurally verified the corrected version is identical to the original everywhere except the two intended additions.

**19. Investigated whether the customer-delivery traceability tracking data needed to be part of the recovery**

Before finalizing the recovery approach, checked whether the separate tracking data behind the customer-delivery/goods-traceability screens also needed to be cleared and rebuilt. Found that data is generated by two entirely separate processes (not the one being fixed here), which read their material-composition figures directly from the same records affected by the unapproved-material bug — so they are related to this issue, but the actual rebuilding of that data only happens on demand, when someone opens the relevant tracking screen for the date range in question; the daily process only does partial cleanup of it (skipping any record already sent to a customer by email/notification, since it doesn't check by month). Also found and noted a separate, unrelated defect: a delivery-date field on that data is always left blank due to a coding mistake in the process that builds it.

Checked the real numbers for the affected date range and found that, as a side effect of the two re-runs done in step 16, all of the relevant delivery-tracking records for that range had already been cleared out on their own — so no separate manual clearing was needed this time. This was noted as coincidental to this specific situation (no customer-notified records happened to exist in that range yet) rather than something to rely on generally going forward.

**20. User's recovery decision: fully clear August and September data, then re-run**

The user decided on a more thorough recovery approach: fully clear all August and September records across the module (rather than relying only on the "inactive marker" left by the earlier accidental run), then re-run the process for both months. Reviewed every table with a name suggesting it belongs to this tracking module and confirmed exactly which ones are actually written by this process and scoped by month (three of them, holding roughly 105,000 combined rows across both months, counting all historical inactive copies) — the customer-delivery tracking data (already addressed in step 19) and one purely internal working table (which is rebuilt fresh on every run and holds no month-persistent data) did not need to be included, along with a couple of similarly-named but functionally unrelated tables. Prepared a ready-to-run script covering exactly the necessary clear-and-rebuild sequence, plus before/after verification checks, with a strong recommendation to compile the guard-check fix from step 18 first so the defect that caused today's data loss cannot resurface.

**21. User applied the fix and ran the recovery — verified results**

Verified: August's "material transferred into the mixing warehouse" figure is now back to roughly 1,085 tons — an exact match with the true source records — confirming the fix worked and the root cause is resolved. The opening-balance figure remains correct and unchanged. Every raw-cotton record for both months now shows only approved material, meeting this ticket's core requirement.

One further item was flagged for clarification: the "material allocated out to finished-goods lots" figure for August came out roughly 313 tons lower than the figure that existed before today's incident began. Critically, this exact lower figure was present both in the earlier broken run and in today's corrected run — proving it is unrelated to the defect just fixed (if it were related, the figure would have recovered once the transfer-in data was restored, but it stayed exactly the same). Investigated further and found this figure lines up closely with the timing of this month's official closing approval, and traced it structurally to the fact that raw-material allocation can only draw from stock tied to the specific production batch it belongs to, not from the overall monthly total — meaning a shortfall can occur locally even when the overall total is more than sufficient. This pattern is consistent with an already-known, separate pre-existing system issue (reported earlier in this same investigation — material balances carried forward are never actually reduced by real consumption) rather than anything newly introduced today. It does not affect this ticket's core requirement and should not block the September 10 closing, but is worth flagging to business separately.

**22. Found a real side-effect on the customer-facing raw-material-origin tracking screen — caused by this fix**

Business reported, through internal chat, that a screen used to review and certify the country-of-origin of raw cotton in customer deliveries was showing many rows with no origin and no supporting document files attached, for August deliveries.

Investigated and confirmed: this screen looks up origin and certificate files by matching, among other things, the material code recorded at the time of purchase against the material code currently on the production record. Because this fix relabels the material code on production records (from the unapproved code to the approved one), that match breaks for exactly the lots this fix touched — the purchase record still (correctly) shows the original code, but the production record now shows the relabeled one, so the lookup finds nothing.

Verified with real data: every one of the affected rows shown in the reported screenshot was in fact purchased under the original unapproved-material code — confirming with certainty this is a direct side effect of this fix, not a separate issue. This particular screen exists specifically to support country-of-origin certification for customs/export purposes, where the origin shown must reflect the true physical origin of the material, not the internal accounting label used for allocation bookkeeping. Presented two options to the user: (A) adjust the origin/document lookup to match on the production lot's own reference number alone, so it continues to find the correct original purchase record and true origin regardless of how the material code was relabeled for accounting purposes; (B) leave the screen as-is. **The user chose option (A).**

Before making the change, verified it would be completely safe: confirmed that no lot reference number in the purchase records is ever associated with more than one material code, and no lot reference number is ever associated with more than one purchase document or origin — meaning the adjusted lookup cannot introduce duplicate or incorrect rows. Prepared the corrected version of the screen's underlying logic.

**23. Swept for related screens and found 5 more places with the identical issue**

At the user's request, also checked the file-viewing popup opened from this screen, and found it had the exact same defect. Given that, swept the entire system for every place using the same combination of data (the original purchase record and the production tracking record) and found a total of 6 places sharing this defect: the main screen already fixed in step 22, the file-viewing popup, one related tracking screen, an older version of the same tracking screen, a second variant of the file popup, and — notably — the feature that sends a raw-material traceability report directly to customers by email. Applied the identical fix (match on the lot reference number only) to all 5 remaining places, following the same safety verification already confirmed in step 22. The customer-facing email feature is worth flagging specifically: if left unfixed, traceability reports sent to customers would show missing or incorrect origin information for exactly the lots this fix relabeled.

---

**Open items still awaiting confirmation from the user/DONGIL side:**
1. Whether the original lot number is kept unchanged when the material code is swapped.
2. Whether the resulting "orphaned" unapproved-material balance (which will never be reduced again
   on this job's books) has any impact on how DONGIL tracks inventory/costing per material type.
3. Confirm the permanent, unlimited-time scope (applying to the entire raw-cotton material group) is
   really the intended long-term behavior.
4. Whether the secondary issue found in step 5 (balance not reduced by consumption) should be
   reported as a separate request/ticket.
5. The MONTHLY-closing mode currently has no effect for DONGIL (nothing calls it, and its own data
   source is also incomplete — see step 15) — is it worth keeping the fix there, or is it purely a
   safeguard for other factories sharing the same program?
6. Whether to add the extra safety check at the temporary-staging step (reviewer's suggestion from
   step 14, not mandatory).
7. **[NEW]** The roughly 313-ton gap in material allocated to finished-goods lots for August (step
   21, suspected related to the already-known pre-existing carry-over issue) — does this need
   dedicated further investigation to quantify the impact precisely, or is it acceptable to treat it
   as a consequence of that already-known issue and address it together when that issue is reported?
8. **[NEW]** After the 6 fixes in step 23 — should other, currently-unchecked screens/features also
   be swept for possible impact from the material-code relabeling, or are these 6 the complete scope?
