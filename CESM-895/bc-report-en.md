# BC Report — CESM-895

**Task:** Clarify How Form SS2.5.1 Data Is Populated (User-Loaded vs Auto-Updated) with Examples

---

## 2026-09-14

**1. Determined whether form SS.2.5.1 (Margin Table After Cost) is user-entered or system-automated**

Checked the sample order `202608-0025HNW036`: every Margin Table record clearly logs who created/modified it and when. This order was saved by staff member **Thanh** (Nguyen Tran Thanh Thanh) on 2026-08-03.

**2. Provided additional example orders with the corresponding username**

Checked several more recent orders - all tied to specific staff members (Hien, Thuan, Fotl0320...), none linked to a system/automatic account. One older block of data (about 13,900 rows) came from a one-time migration from the legacy system during the platform switch, not a recurring automated process.

**3. Confirmed there is no automatic background update mechanism**

Thoroughly checked the system: there is no background task and no scheduled job that automatically writes data into the Margin Table. Data is only written when a user actually clicks Save on the form.

**4. Addressed the question: staff say they only use SS.2.5, so why does SS.2.5.1 still show data?**

Confirmed that SS.2.5 (Margin Table) and SS.2.5.1 (Margin Table After Cost) actually read and write **one single shared record**. Staff enter and save data through SS.2.5 - SS.2.5.1 is only a display/computed screen over that same record, so no one needs to open SS.2.5.1 separately for data to appear there. Staff reporting that they 'only use SS.2.5' is therefore fully accurate.

**5. Addressed the question: why does the 'Final quantity' figure differ between the two forms for the same order (10,759 vs 8,069.3)**

These are two figures with two different meanings, not an error:
- **10,759 kg** (SS.2.5) = the original **planned/ordered** quantity.
- **8,069.3 kg** (SS.2.5.1) = the **actual quantity produced and passed quality inspection (QC)** - automatically pulled from the real roll-by-roll weighing/inspection results on the factory floor (431 rolls passed out of 468 rolls inspected on 2026-09-03), not something typed manually on the form.

The gap between the two figures reflects normal production loss/defects that occur during actual manufacturing - a routine occurrence in the textile industry, not a data error.

**6. Addressed a further question: why the 8,069.3 figure could not be found on another QC report the user cross-checked**

In the production process, goods go through quality inspection (QC) **twice, at two different points in time**: once right after production (which produces the 8,069.3 figure seen on SS.2.5.1) and once again right before packing/shipment (Outgoing QC - which produces a different figure, 8,542.4, on the report the user cross-checked). These are two separate inspection checkpoints for the same production lot, so it is normal for them to show different figures - not a data discrepancy.

## 2026-09-15

**1. Explained the Profit 2 formula and why it showed a heavily negative value for order 202607-0457WHTX**

User asked a follow-up: form SS.2.5.2 'Margin Table Packages' showed Profit 2 = -9,239.61 (about -73 percent) for this order, while the regular Profit figure on the same screen was a normal positive value. Explained that Profit 2 is a more complete profitability calculation: it uses the **negotiated selling price** (the actual price agreed with the customer) minus **all cost items** (including commission, shipping fees, export charges, and so on) and minus any confirmed claim amount - unlike the regular Profit figure, which only subtracts the basic production cost.

**2. Identified the root cause: the negotiated price data was misattributed to a different order**

This order actually DID have a negotiated price entered (**$6,429.78**), but due to a data-linking error, the system failed to recognize it as belonging to this order - it was mistakenly attributed to a completely different, unrelated order instead. As a result, when calculating Profit 2, the system treated this order's negotiated revenue as zero, driving Profit 2 down to nearly the full amount of the production cost.

**3. Corrected the link and confirmed the result**

After thoroughly checking that no other order would be affected, corrected the link for this order. After the fix, Profit 2 recalculated to **-2,158.7** - still negative, but now a REAL figure (reflecting that the price agreed with the customer is lower than the production cost), no longer a false negative caused by missing data.

**4. Found that this linking issue is fairly widespread across the system**

A broader check found that about **10 percent of all negotiated-price data** across the entire system (nearly 24,000 records) has the same kind of linking error, occurring continuously over more than a year and a half across multiple staff members - suggesting this may be a routine step in the data-entry workflow that has been quietly causing errors, rather than a one-time incident. This was reported back to the user for a decision on whether a broader data cleanup is needed - a much larger scope than fixing a single order, and not addressed in this pass.
