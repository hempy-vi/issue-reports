# Dev Report — CESM-698

**Task:** [SAMIL] [BUG] POP Shows Inconsistent Information Between Inner and Outer Screens (Order 202606-0420SAWMT)

---

## 2026-09-09

**1. Identifying the two screens in the ticket**

From the 3 screenshots attached to the ticket, identified these as two Windows desktop (C#/WinForms) forms belonging to the **QC_DYE_V2_MAIN** module (`D:\WINDOWS APP PROJECT\windows-app\SAMIL\QC_DYE_V2_MAIN\ScanSystem\ScanSystem`):
- `frmQC0012.cs` — processing-card data-entry screen (receiving greige fabric from the knitting factory), showing `265 SQM`.
- `frmGC0011.cs` — per-roll fabric inspection screen, showing `250 SQM`.

Ruled out the `QC_PRINT_V2` module (uses a different SP prefix, `sp_sel_pop_p_qc0012_*`) and `Processing_cardV2`/`ITEM_LABEL` (older non-`_v02` copies; `ITEM_LABEL` already confirmed as unused legacy).

Read the source of both forms:
- `frmQC0012.cs:73` — `lbSQM.Text` reads the `sqm` column from procedure **`SP_SEL_POP_QC0012_PC_V02`** (called with the processing-card PK `prod_card_pk`).
- `frmQC0012.cs:206-298` (Save button, `sp_upd_pop_qc0012_v02`) — confirmed `sqm`/`weight` are NOT part of the save payload — this is a display-only (read-only) field.
- `frmGC0011.cs:189` — `lbSqm.Text` reads the `sqm` column from **`SP_SEL_POP_QC0011_V02`** (called with the PK of the QC record tied to the specific roll).
- `frmGC0011.cs:1552-1581` — confirmed `lbSqm` (250) is used as the ±15% validation threshold for 3 manually-entered actual G/M readings — by design `lbSqm` MUST be the order's required G/M2 spec, not the roll's own actual measurement.

Cross-checked against the original GASP order screen (`sa1000031.aspx`) — the order's required weight = `250` G/m², matching `frmGC0011` (`250`), mismatched with `frmQC0012` (`265`).

**2. DB connectivity issue — VPN down, switched to REST API after VPN reconnected**

MCP `oracle-samil` timed out on session start. Checked:
```
docker logs oracle-api-samil --tail 40
→ ORA-12170: Cannot connect. TCP connect timeout of 20000ms for host 192.168.240.204 port 1521
```
Direct connectivity tests from the host (`Test-NetConnection 192.168.240.204 -Port 1521`, `/dev/tcp`) both failed — confirmed this was a VPN/network issue to SAMIL's internal network, not a docker container problem. Asked the user to check their VPN.

After the user reconnected the VPN: `docker restart oracle-api-samil` → healthy, `Test-NetConnection` OK. MCP `oracle-samil` still did not auto-reconnect within the session — switched to calling the `oracle-api-samil` REST API directly (`http://127.0.0.1:8084`, see `platform/oracle-api/README.md`) via `curl`.

Side finding: the `/api/db/objects` and `/api/db/sources/{name}` endpoints of `oracle-api` have a pre-existing bug — omitting the `owner`/`type` param crashes with `ORA-17004: Invalid column type` (JDBC cannot infer the type for `setNull` when an optional param is null) → had to always pass `?owner=SAMILV2&type=PROCEDURE`. Used `POST /api/db/query/read-only` (direct SELECT against `all_objects`/`all_source`) instead of the broken `/api/db/objects` endpoint. (This bug belongs to `platform/oracle-api`, out of scope for this ticket — noted only, not fixed.)

**3. Root cause — confirmed with real PL/SQL + real data**

Fetched the full PL/SQL source of both procedures via `GET /api/db/sources/{name}?owner=SAMILV2&type=PROCEDURE`. Both use the identical GSM formula:
```sql
round(DECODE(unit,'MTS',0.9144,1)
  * NVL((<unit_gravity> / SUBSTR(REPLACE(<a_demission>,'"'),-2) / 0.02323), 0), 2)
```
but source `<unit_gravity>` from two different tables:
- `SP_SEL_POP_QC0011_V02` (correct): `C.A_UNIT_GRAVITY` — table `sa_order_production` (alias c) — the order's **current/live** spec value.
- `SP_SEL_POP_QC0012_PC_V02` (buggy): `a.unit_gravity` — table `sa_processing_card` (alias a) — a VARCHAR2 column holding a one-time copy taken when the processing card was created, **never refreshed afterward**.

Order `202606-0420SAWMT` carries a revision note: "09/07 REV 기존 70/72" 250GSM 418GYD >>>66/68" 250GSM 395GYD" — the spec was corrected on 09/07 from weight 418 G/YD to 395 G/YD (same 250GSM target). Processing cards created before that correction still carry the stale `unit_gravity='418'`.

Verified against real data (joining `sa_processing_card`/`sa_order_production` for PO `202606-0420SAWMT`):
```sql
SELECT a.pk card_pk, a.prod_card_no, a.lot, a.unit_gravity card_unit_gravity,
       b.pk order_pk, b.po_no, b.a_unit_gravity order_unit_gravity, b.a_demission, b.unit,
       round(DECODE(b.unit,'MTS',0.9144,1) * NVL((a.unit_gravity / SUBSTR(REPLACE(b.a_demission,'"'),-2) / 0.02323),0),2) sqm_using_card_gravity,
       round(DECODE(b.unit,'MTS',0.9144,1) * NVL((b.a_unit_gravity / SUBSTR(REPLACE(b.a_demission,'"'),-2) / 0.02323),0),2) sqm_using_order_gravity
FROM sa_processing_card a, sa_order_production b
WHERE a.sa_order_production_pk = b.pk AND a.del_if=0 AND b.del_if=0 AND b.po_no='202606-0420SAWMT'
```
Result (20 processing cards): LOT 001 (`260905-020`, the card shown in the screenshot) has `card_unit_gravity='418'` (stale) vs `order_unit_gravity=395` (live) → `sqm_using_card_gravity = 264.62` (displays as **265**, exact match to screenshot 2) vs `sqm_using_order_gravity = 250.06` (displays as **250**, exact match to screenshot 3). 17 of the 20 processing cards for this order were similarly stale; only lots 002-004 already had the corrected value.

**4. Extended audit — checking other procedures for the same bug**

Found every procedure in schema SAMILV2 that shares the `0.02323` constant (the fingerprint of this GSM formula):
```sql
SELECT owner, name, type, COUNT(*) hit_lines FROM all_source
WHERE UPPER(text) LIKE '%0.02323%' GROUP BY owner, name, type ORDER BY owner, name
```
→ 35 procedures. Narrowed to 10 procedures in the actual POP-QC family (excluding `SP_RPT_*`/`SP_SEL_SA*` — reports/sales, a different domain) and audited each: read the full PL/SQL + grep client code for the calling screen + judge whether using a snapshot is a bug or intentional (e.g. a tag-reprint screen should legitimately preserve a historical value — that would not be a bug).

Results:
- **Bug found, same pattern** — `SP_SEL_POP_QC0012_PC` (older, non-`_V02` version): used by `frmQC0012.cs` in the **`Processing_cardV2`** module (build artifact dated 2026-05-29 → actively deployed). Uses `a.unit_gravity` in the identical two spots (lines ~66, ~130).
- **Not a bug** — `SP_SEL_POP_QC0011` (already correct — only uses `sa_processing_card` as a join hop, never reads a value from it); the entire `SP_SEL_POP_P_QC00xx` family in the `QC_PRINT_V2` module (uses a different table set, `sa_order_production_p`/`sa_prod_card_printting`, whose card table has no unit_gravity column of its own at all, so this bug pattern cannot architecturally occur there); `SP_SEL_POP_QC0018N_D`/`SP_SEL_POP_QC0018_D` (their displayed sqm is computed from actual hand-measured roll readings, unrelated to unit_gravity).
- **Same logic bug, but dead code** — `SP_SEL_POP_QC0012_PC_V02_TEST`: its WHERE clause hard-codes two specific `prod_card_no` values and ignores its input parameter; an exhaustive grep across both client apps (GASP + Windows POP, ~11K files) found zero callers → an orphaned/scratch procedure with no real-world impact.
- Unrelated side finding: `SP_SEL_POP_P_QC0012_PO` is missing a join predicate between `sa_order_production_p` and `sa_order_prod_color_p` (a latent cross-join) — but this procedure is also dead code (0 callers), dormant, noted separately and out of this ticket's scope.

**5. Fix**

Prepared two `CREATE OR REPLACE PROCEDURE` scripts (auto-generated by replacing exactly the two occurrences of `a.unit_gravity` with `b.a_unit_gravity` in the original source fetched from `ALL_SOURCE`, verified to be exactly 2 matches per file with nothing else touched):
- `SP_SEL_POP_QC0012_PC_V02` (QC_DYE_V2_MAIN module — the exact screen in the ticket)
- `SP_SEL_POP_QC0012_PC` (Processing_cardV2 module — additional finding from the audit)

Along with two matching rollback scripts (= the original unmodified source). No historical data fix needed — the fix only changes the read formula, so after deployment all existing processing cards automatically display correctly against the current `sa_order_production.a_unit_gravity`.

After the two scripts were applied to the database, the user confirmed understanding and the ticket was closed out.
