# BC Report — CESM-849

**Task:** Fix WePOP Batch Print Wrong Customer Data and Unresponsive Print All Button (IBL020/IBL040)

---

## 2026-09-14

**1. Received the bug report and initial analysis**
Customer SHINWOO reported 3 issues in the WePOP software (barcode label printing feature):
- When batch printing ("Print All"), labels printed for customer ESTEC sometimes showed the wrong customer name (an entirely unrelated company), while printing single labels one at a time was always correct.
- The ESTEC label layout had one extra line of information compared to the standard label sample the customer provided.
- On one of the label management screens (IBL020), the "Print All" button produced no response at all when clicked, and no print output opened.

**2. Investigated the batch-printing mechanism**
Reviewed the full process the system uses to select and render label content — identified 4 data-processing components (shared between two different label management screens) responsible for deciding what content appears on each label.

**3. Identified the root cause**
Found that the two label management screens (one for regular sales orders, one for a special "other customer" workflow) share these same 4 data-processing components, but each screen passes a different type of identifying key. A prior change (made specifically to support the "other customer" screen) had unintentionally broken the regular sales-order screen's ability to recognize its own key:
- Common case: no matching data is found, so the system prints nothing at all (matching the "Print All does not respond" symptom).
- Rare case: due to how the system's internal numbering works, one order's key number happens to coincidentally match a different order's key number belonging to a DIFFERENT customer — so the system pulls and prints that other customer's data by mistake (matching the "wrong customer name" symptom).

Both issues turned out to be the SAME underlying cause.

**4. Fix applied**
- Restored the original key-recognition logic for the regular sales-order screen, while keeping the existing logic for the "other customer" screen fully intact — both screens now work correctly without interfering with each other.
- Added a clear on-screen message for the user whenever a print request has no matching data or encounters corrupted data, instead of the system silently doing nothing as before — so users immediately know something needs to be reported, rather than mistaking it for the system being frozen.

**5. An unexpected incident: the entire label-printing feature was interrupted**
While attempting to self-restore some of the data-processing components (by copying content from the original bug report), an invisible special character (not visible to the naked eye) was accidentally introduced into the content during the copy from a formatted text source — this caused the system to be unable to understand the instructions, resulting in the ENTIRE label-printing feature (both screens, for EVERY customer, not just ESTEC) being temporarily interrupted.

**6. Emergency diagnosis and recovery**
Confirmed this was caused by an invisible character introduced during copying, not a logic error. Re-typed all the affected content by hand (rather than copying), carefully verified no stray characters remained, and restored all 11 affected data-processing components. Confirmed the label-printing feature was working normally again immediately afterward.

**7. Found 2 more affected components**
The "other customer" label management screen reported an error when searching for orders — investigation found 2 search-related components were affected by the same invisible-character issue from the incident above (these fell outside the printing feature itself, so were missed in the earlier check). Restored both, and re-scanned the entire system to confirm nothing else was affected.

**8. Proposed a longer-term improvement (not yet implemented)**
To prevent a similar incident from recurring in the future (a fix intended for one screen unintentionally affecting the other), a plan was prepared to fully separate the 4 shared data-processing components into two independent sets, one per screen. Not yet implemented — pending confirmation at a convenient time.

**Remaining open items (not addressed in this session):**
- The extra layout line on ESTEC's labels still needs careful review, since this label template is shared with many other customers and any change must not affect them.
- Stale labels from other customers appearing when batch-printing on the "other customer" screen — requires a decision on how to handle the historical data.
- The plan to separate the two screens (step 8) — prepared, not yet implemented.
