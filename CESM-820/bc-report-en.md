# BC Report — CESM-820

**Task:** Allow an Optional Trailing Letter After Lot No on Print Label (e.g. LOT 036 -> LOT 036A)

---

## 2026-09-11

**1. Received the request from Sales**
Sales (Ms.Oanh) requested allowing one optional trailing letter to
be appended to the Lot No on the printed label (e.g. Lot `036` ->
`036A`). Reason: the customer needs to distinguish shipments that
share the same Lot number by appending one extra letter. The
existing rule for the numeric part of the Lot No stays unchanged;
the trailing letter is optional.

**2. Checked current system behavior**
Confirmed that the current Lot No entry screen only accepts digits
— if the user typed a letter as well (e.g. `39A`), the system
automatically dropped the letter, keeping only the numeric part
(`039`), exactly matching what Sales reported along with the
screenshot.

**3. Checked impact on related functions**
Reviewed the entire save and print-label workflow to make sure
adding one trailing letter to the Lot No would not break saving or
printing — result: no risk found, the Lot No value is saved and
printed as plain text and is not restricted to digits only.

**4. Prepared the change**
Prepared the change to allow entering one optional trailing letter
after Lot No, keeping the existing numeric display rule unchanged
and not affecting Lot Nos entered previously (without a letter).

**5. Customer withdrew the request — system kept unchanged**
Before handover, Sales informed us that the buyer side (Korean
partner) had further discussions and decided this change was no
longer needed. Per Sales' request, work was stopped and the system
was restored exactly to its original state, with no change applied.
The task is closed in hold status.
