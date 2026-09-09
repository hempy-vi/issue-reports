# BC Report — CESM-698

**Task:** [SAMIL] [BUG] POP Shows Inconsistent Information Between Inner and Outer Screens (Order 202606-0420SAWMT)

---

## 2026-09-09

**1. Identifying the two screens reported**

From the 3 screenshots attached to the ticket, identified the exact two screens on the factory-floor POP (Windows) application showing mismatched information for the same order: the processing-card data-entry screen showed a G/M2 value of **265**, while the per-roll fabric inspection screen showed **250**. Cross-checking against the original order screen on the web system confirmed the order's correct required value is **250** — meaning the first screen was displaying the wrong number.

**2. System connectivity issue**

During the investigation, the connection to SAMIL's data system was interrupted due to a VPN issue. The user was asked to check their VPN connection; once reconnected, the investigation continued normally.

**3. Root cause identified**

Confirmed the root cause: order `202606-0420SAWMT` had its required weight figure corrected on 09/07 due to an initial data-entry error. The per-roll inspection screen always reads the order's latest required figure, so it displays correctly. The processing-card data-entry screen, on the other hand, reads a value that was "snapshotted" only once, at the moment the processing card was created — if a card was created before the order's correction, that value stays frozen forever and never picks up the later correction. Real production data was checked and confirmed: 17 of the 20 processing cards for this order were stuck with the old value exactly as described, matching precisely the incorrect 265 seen on screen.

**4. System-wide review**

To make sure no other screen in the POP system carries the same bug, a full review was carried out across every function that shares this same calculation. One additional function currently in active use (part of a different version of the processing-card management module) was found to have the identical bug. Every other function reviewed was confirmed unaffected — either because it sources its figure through a safer mechanism, or because it turned out to be an internal developer test copy never used by any real user.

**5. Resolution**

Corrective fixes were prepared for both affected functions (the one named in the ticket plus the one found during the review), adjusted so they always read the order's latest required figure, ensuring every screen displays consistent information. No historical data needed to be corrected — once the fix is applied, every processing card (including the older ones that were stuck on outdated figures) will automatically display correctly against the order's latest figure.

After the fix was applied to the system, the user confirmed understanding and the ticket was closed out.
