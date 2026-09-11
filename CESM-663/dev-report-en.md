# Dev Report — CESM-663

**Task:** [SHINWOO][BUG] WePOP - IBL040 Prod Label Issue (SO-ETD Divide)

---

## 2026-09-11

**1. Bug reported from screenshots**
BA reported for SO `SOOT202608170010`: using **Print** (single print)
renders the correct custom label layout with separate `Order Qty` /
`Shipment Qty` lines (802816 pcs / 81 Roll and 826920.48 pcs / 83 Roll).
Using **Print All** (batch print) for the same SO loads a different,
outdated template — only a single merged `Qty` line (802816 pcs / 802816
ROLL, wrong because the Roll count is set equal to the pcs count instead
of being computed separately).

**2. Fix**
Synced both **Print** and **Print All** actions to call the exact same
report template file and dataset handler (instead of **Print All**
branching to an old template/handler). The related store/handler was
edited directly (not tracked through this workspace's session log).
