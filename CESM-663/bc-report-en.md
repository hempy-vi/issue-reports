# BC Report — CESM-663

**Task:** [SHINWOO][BUG] WePOP - IBL040 Prod Label Issue (SO-ETD Divide)

---

## 2026-09-11

**1. Bug reported from screenshots**
BA reported that for the same order (SO), printing with the **Print**
button shows the correct label, with Order Qty and Shipment Qty
displayed separately. But printing with the **Print All** button shows
an incorrect label — it uses an old template, merges the quantity into a
single line, and shows the wrong Roll count.

**2. Fix**
Adjusted so that both **Print** and **Print All** always use the exact
same standard label template, preventing the same order from producing
two different printed results.
