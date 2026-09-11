# BC Report — CESM-726

**Task:** [Samil] [Data handling] Impact Analysis – Yarn Code Changes in SB1.2

---

## 2026-09-10

**1. Identify the source forms to investigate**

The ticket lists 5 forms related to Yarn Code: Item Code (SB1.1), Yarn Code (SB1.2 — where deletion/editing originates), R&D Item Register (SB1.13), Item Code Inquiry (SB1.16), Yarn Code Inquiry (SB1.27). These 5 forms were confirmed with the user via a real menu screenshot from the live application.

**2. Uncovered how Yarn Code is actually stored in the system**

Yarn Code in the system is actually just one classification sub-group within the shared Item Master catalog used across the entire ERP (every type of goods — yarn, fabric, chemicals, finished goods — is stored in the same place, only distinguished by a group code).

- When a yarn code is **deleted**: the system does not physically delete it, it only "hides" it (soft-delete) — the data still exists, it just no longer shows.
- When a yarn code is **edited** (renamed): other forms automatically pick up the new name immediately, with no further action needed.

**3. Found the root cause of the risk described in the ticket**

The system currently has one safety check that blocks deleting a yarn code IF that yarn has ever had a warehouse in/out transaction. However, this safety check does NOT cover the case where the yarn is being used in a fabric composition (Item) or an R&D formula — meaning a yarn code actively used in a formula, but with no warehouse transaction yet, can still be deleted with absolutely no warning to the user. This is the technical root cause of the risk described in the ticket.

**4. Identified the concrete consequence on each form when a yarn code is deleted**

- Yarn Code (SB1.2) and Yarn Code Inquiry (SB1.27): the deleted yarn's row disappears from the displayed list.
- Item Code (SB1.1) and Item Code Inquiry (SB1.16): the product/fabric still displays normally, only the corresponding yarn-name cell goes blank.
- R&D Item Register (SB1.13): the R&D formula still displays, but that yarn silently disappears from the composition table — with no warning to the viewer, easily misread as "this formula doesn't use that yarn."

**5. Expanded the investigation scope as requested — reviewed the whole system**

Per an additional request, the whole system (not just the original 5 forms) was reviewed to find every screen related to Yarn Code, cross-checked against the application's real menu catalog. Several more affected screens were found: the Knitting area, the Yarn Supplier area, the Order area, and some production output reports.

**6. The user pointed out 2 more overlooked areas — confirmed correct**

The business owner pointed out that the Stock In area and the Bill of Materials (BOM) area had not been included. On review, both areas were confirmed to genuinely use yarn data, with a very large volume of real transactions (tens of thousands).

**7. The expanded review surfaced too many results — needed further filtering for accuracy**

Because the system stores every type of goods in one shared catalog, the initial expanded review produced a very long list (over 200 screens), but it mixed in many screens that are not actually related to yarn — they just happen to share the same goods catalog for a DIFFERENT type of goods (finished products, chemicals, etc.).

**8. The user's pushback was accurate — several screens had been included incorrectly**

The business owner correctly pointed out that screens related to the Dyeing and Printing stages cannot be affected by a yarn code change, because those stages work off the FINISHED PRODUCT's own code, not the yarn code directly. On careful review, this was confirmed to be completely accurate — the Dyeing/Printing screens indeed display no yarn information at all, only their own finished-product code.

**9. Re-verified the entire list against real data instead of assumptions**

To ensure full accuracy, every screen in the expanded list was re-checked directly against real data in the system (rather than relying on screen names or business-area assumptions), to determine precisely which screens genuinely have yarn data flowing through them and which ones merely happen to share the same catalog table without any real yarn connection.

**10. Final result**

After review and removing the cases that were not actually related, the final list contains 93 screens genuinely affected by a yarn code change (a significant reduction from the initial rough list of over 200 screens).

Notable finding: one screen named "Yarn Spec Property" (yarn technical specifications) — despite its name literally containing "Yarn" just like the ticket — was found, upon careful review, to actually manage a completely different "yarn" classification belonging to a separate business area (testing/inspection), unrelated to the Yarn Code in this ticket — so it was excluded from the final list.

The final result was compiled into a single document listing all 93 affected screens along with their real application menu paths, so the Samil team can reference it when planning checks/data synchronization around future yarn code changes.
