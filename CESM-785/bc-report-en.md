# BC Report — CESM-785

**Task:** Fix Roll Location/Date Mismatch Between Warehouses HOQC301 and HDF102

---

## 2026-09-10

**1. Request received**
A fabric roll (barcode D25091920979, order 202508-0343HSAEROW, lot 005, roll 030R) had been moved from warehouse HOQC301 to HDF102, but the system data became inconsistent: at HOQC301 the roll still appeared but could not be issued out (out-of-stock error); at HDF102 (the actual destination) the roll could not be found to select. The business team requested correcting the income date to 20/09/2025 and the actual warehouse to HDF102.

**2. Investigating the out-of-stock error at HOQC301**
Reviewed the roll's inbound/outbound/transfer history and found: two original inbound documents dated 20/09/2025 had previously been cancelled (likely a duplicate entry), the transfer document from HOQC301 to HDF102 had run correctly on 20/09/2025, and a replacement inbound document was later created but dated 21/09/2025 — i.e. after the goods had already been transferred out. As a result, according to the system's own records, stock at HOQC301 is genuinely 0 (already moved to HDF102), so the out-of-stock error was technically correct given the data — the real problem lay elsewhere.

**3. Investigating why the roll could not be found at HDF102**
Determined that the roll-location lookup screen (Item Barcode Checking) and the roll-selection screen used when creating vouchers both read the roll's 'current warehouse' from a separate master record (not from the stock ledger), and this master record had NOT been updated when the transfer document ran — it still showed the current warehouse as HOQC301, even though the roll had actually already moved to HDF102. This is exactly why the roll still 'appeared' at HOQC301 (though it could not be issued) and 'could not be found' at HDF102 (even though stock was already sufficient there).

**4. Safety check before applying the fix**
Verified that the related stock entries are not linked to any closed accounting/closing period (no impact on already-finalized financial reports) — safe to adjust.

**5. Data fix prepared**
Prepared an update script covering:
- Inbound date of the currently active document: 21/09/2025 → 20/09/2025 (matching the actual goods-received date and the transfer date).
- The roll's current warehouse on the location master record: HOQC301 → HDF102 (with the previous value backed up so it can be restored if needed).

**6. Operational note**
The old draft outbound voucher (currently saved but not yet confirmed) had been created selecting warehouse HOQC301 because the location was displayed incorrectly at the time. After the data fix, the roll will no longer appear at HOQC301, so the old draft should be discarded and a new outbound voucher created, correctly selecting warehouse HDF102.
