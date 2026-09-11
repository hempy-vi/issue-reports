# Dev Report — CESM-726

**Task:** [Samil] [Data handling] Impact Analysis – Yarn Code Changes in SB1.2

---

## 2026-09-10

**1. Identify the source forms to trace**

The ticket lists 5 forms: SB1.1 (Item Code), SB1.2 (Yarn Code — the source of delete/edit), SB1.13 (R&D Item Register), SB1.16 (Item Code Inquiry), SB1.27 (Yarn Code Inquiry). There is no static mapping of 'SB1.x' codes to `.aspx' file names in source (the `<title>` tag only holds the default framework name 'genuwin'), so the user supplied the real URLs via a menu screenshot, cross-checked against the system menu table `TES_OBJ`:

| Ticket form | Path | Select proc |
|---|---|---|
| SB1.1 | `/form/sa/10/sa100010.aspx` | `SP_SEL_SA100010` |
| SB1.2 | `/form/sa/10/sa100020.aspx` | `SP_SEL_SA100020` |
| SB1.13 | `/form/sa/10/sa100130.aspx` | `SP_SEL_SA100130` |
| SB1.16 | `/form/sa/10/sa100010_V02.aspx` | `SP_SEL_SA100010` (shared with SB1.1) |
| SB1.27 | `/form/sa/10/sa100020_V02.aspx` | `SP_SEL_SA100020` (shared with SB1.2) |

**2. Trace the Yarn Code storage mechanism — core finding**

`SA_YARN_CODE` is not its own table — it is a VIEW:
```sql
SELECT PK, ITEM_CODE YARN_CODE, ..., DEL_IF, ...
  FROM TLG_IT_ITEM
 WHERE TLG_IT_ITEMGRP_PK = 1243
```
Yarn Code rows are simply rows of the shared ITEM MASTER table (`TLG_IT_ITEM`) belonging to item group 1243. Save/Delete on SB1.2 (`SP_UPD_SA100020`) goes through the shared item-master procedure `LG_UPD_AGCI00070_2`.

- **Delete = soft-delete**: `TLG_IT_ITEM.DEL_IF := PK` (not `=1`) — the row still physically exists, it is only filtered out by any query with `WHERE DEL_IF = 0`.
- **Edit = in-place update, PK unchanged** — every dependent form joins by PK (not by the `YARN_CODE` text), so renaming a yarn code text automatically propagates everywhere without manual updates.

**3. Root cause finding — the delete guard is incomplete**

Before allowing a delete, `LG_UPD_AGCI00070_2` only checks:
```sql
SELECT COUNT(*) FROM TLG_IN_STOCKTR WHERE DEL_IF=0 AND TLG_IT_ITEM_PK = P_TCO_ITEM_PK
```
If > 0, the delete is blocked. This guard does NOT check `SA_ITEM_CODE.YARN01_PK..YARN09_PK` (6,980 rows currently assigned) or `SA_RND_ITEM_D.SA_YARN_CODE_PK` (50,960 rows currently referencing) — meaning a yarn code in active use in an R&D/Item formula but with no inventory transaction yet can still be deleted with no warning at all.

**4. Detailed per-form impact when a yarn code is DELETED**

| Form | Join mechanism | Behavior on delete |
|---|---|---|
| SB1.2 / SB1.27 | share `SP_SEL_SA100020` | Row disappears from the grid (an UnDelete button exists) |
| SB1.1 / SB1.16 | `YARN0X_PK` LEFT OUTER JOIN | Item itself doesn't disappear, only that specific 'Yarn N' cell goes blank (orphan FK) |
| SB1.13 | `SA_YARN_CODE_PK` INNER JOIN (CTE) | R&D item doesn't disappear, but that yarn silently vanishes from the yarn1-4 breakdown — no warning |

**5. Expand scope per user request — map the whole system via `TES_OBJ`**

The user asked to identify ALL affected forms (not just the ticket's 5), mapped to real menu codes via `TES_OBJ`. A full scan of `USER_SOURCE` for the literal `SA_YARN_CODE` text turned up 74 database objects spanning ≥9 modules. Mapping through `TES_OBJ` (walking `P_PK` up to root to build the full breadcrumb) revealed additionally:
- SB.1.14 'Item Code Inquiry' (same display name as SB.1.16 but a different form, easy to miss).
- 17 more forms: Knitting (SK.2.2), Yarn Supplier (SD.13.1/2/3/7), Sales (SS.1.6/6.1), Processing (SS.2.4/2.5/2.5.1/2.5.2), Production (SM.1.6≡SP.11.3/1.6.1/1.7.1), Yarn/Warehouse (SS.6.1/6.2/6.3).

**6. User pointed out 2 real gaps (Stock In, BOM) — expanded scan revealed a methodology gap**

The user (from real business experience) flagged IV0102 (Stock In) and BOM as missing. Tracing `LG_SEL_BINI00030_2` (Stock In Entry): it joins `TLG_IT_ITEM` directly via `income_item_pk`/`req_item_pk`, NOT through the `SA_YARN_CODE` view — root cause: the earlier scan (grepping for the literal text 'SA_YARN_CODE') only caught forms that 'know' they're handling yarn, missing forms that use a generic item_pk without distinguishing item type, even though they are genuinely affected.

Confirmed with real data: BOM (`TLG_DO_BO_BOM_V3.CHILD_PK`) has 13,296 real rows in item group 1243; Stock In (`TLG_ST_INCOME_D.INCOME_ITEM_PK`) has 58,093 real rows in group 1243 — both confirmed correct.

**7. Full-ERP scan for forms sharing the item master — 237 candidates (too many false positives)**

Since scanning for the plain text 'TLG_IT_ITEM' is far too generic (that table holds EVERY item type across the whole ERP, ~60 different groups), a full-ERP scan produced 237 candidate forms (~25% of the whole system). A direct DB join query timed out (USER_SOURCE has 2.58 million rows) — solution: split into two lighter queries (proc list + active-form list, paginated via `maxRows`), download locally, and match via a local Node.js script.

**8. User's pushback was correct — many Nhóm-5 forms were false positives**

The user pointed out: 'changing a yarn item code cannot affect finished-goods output/dyeing/printing — dyeing and printing use the FINISHED-GOODS item code, not the yarn code directly.' Tracing G/D Entry (`SP_SEL_SA2200010`) specifically: it joins `SA_ORDER_PRODUCTION.SA_ITEM_CODE_PK` → displays the FINISHED GOODS item code (group 1263 'IT01-Item Code'), and never displays any Yarn column — confirming that deleting yarn causes no observable change on this form.

Pulling the `TLG_IT_ITEMGRP` table (61 rows) confirmed: group 1243 ('YN01-Yarn Code') is the ONLY group SB1.2 manages, but at least 3 other groups are also 'yarn'-ish in name yet entirely unrelated (1206, 1393, 1394) — any form using those other groups has nothing to do with SB1.2.

**9. Launched a workflow to verify every one of the 199 candidates against real data (not just text matching)**

Each agent: reads the source of the matched procedure, identifies the real join column into `TLG_IT_ITEM`, and runs a real count query for rows with `TLG_IT_ITEMGRP_PK=1243` through that exact column → verdict CONFIRMED/REJECTED/INCONCLUSIVE with concrete numeric evidence.

Two runs were blocked by rate limits (session limit, then weekly limit); the third run (a fresh full run, 15/15 agents succeeded) produced the final result:
```
80 CONFIRMED, 115 REJECTED, 4 INCONCLUSIVE (out of 199 forms)
```

**10. Final result**

Final impacted-form list: 24 (Groups 1-3, confirmed via literal `SA_YARN_CODE` reference) + 69 (Group 5, confirmed via real data, deduped) = **93 forms** — down from 213 (the earlier list, which mixed in false positives) to 93 (the accurate list, every entry backed by real data evidence).

Noteworthy finding: `SM0201` ('Sales Order'/'BOM Creation v4'), previously added based only on a rough text-match count, was found on rigorous verification to have 0 real rows in item group 1243 (even checking its 3 yarn-sounding columns: `WARP_MATERIAL_PK`/`TABBY_ITEM_PK`/`FILLCORD_ITEM_PK`) → REJECTED, self-corrected out of the list. `TM1020 'Yarn Spec Property'` — REJECTED despite literally containing the word 'Yarn' just like the ticket, because its dedicated table `TLG_KL_YAN_SPEC` has zero rows in item group 1243 (it belongs to a separate Test Management system with its own, unrelated group).

Final deliverable: a single PDF file (`impacted_forms_list.pdf`) — full menu path, 93 forms, grouped by module, cards color-coded by confidence tier.
