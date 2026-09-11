# BC Report — CESM-250

**Task:** WePOP Issue Barcode Label (Stock In Request): Label Modify

---

## 2026-09-11

**1. Received the label modification request**
On the WePOP 'Issue Barcode Labels (Stock In Request)' screen, requested changes to the printed barcode label layout: (1) enlarge the PO No text for better readability, (2) rename the 'Roll' field to 'LOCATION' to match the site's business terminology.

**2. Identified the correct areas to modify on the label**
Identified the two areas on the label that needed changing: the area showing the PO No value (the number line right under the PO No heading) and the heading of the 'Roll' box (without touching the location code value printed below it).

**3. Reviewed the label's auto-shrink mechanism**
The label already has a built-in mechanism that automatically shrinks text when content is too long to fit the print area, preventing text overflow. This mechanism only shrinks, never enlarges — so raising the base text size is safe and will not break the layout even for long PO No values.

**4. Applied the change**
Enlarged the PO No display text and renamed the 'Roll' heading to 'LOCATION' on the IBL530 label.

**5. Verified the build**
Ran a test build of the program to confirm the change introduced no errors. The parts related to the two changes compiled successfully; one unrelated build error surfaced, but it belonged to a different label template (unrelated to this change) and was caused by the test build tooling, with no impact on the production build.

**6. Found an extra-character display issue while checking a real test print**
While checking a test print against real data, found an additional issue: the PO No value on some labels (from a batch of stock-in slips for supplier AD) had an extra special whitespace character (a tab) attached at the end, making the displayed/matched value not clean.

**7. Traced the root cause**
Determined that this extra character was already present in the original PO No data entered into the system beforehand, not caused by this label edit. The current data-cleaning logic only strips ordinary leading/trailing spaces, not this type of tab character, so the extra character was passing through unfiltered.

**8. Verified and prepared the fix**
Verified the correct handling to remove this extra character, confirming it produces a clean result against real data. The fix has been prepared and is ready to apply once confirmed by system administration.
