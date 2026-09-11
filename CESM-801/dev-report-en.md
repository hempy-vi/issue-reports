# Dev Report — CESM-801

**Task:** [SAMIL] [BUG] Cannot Save Order After Editing Color #2 — GW_NOTE Value Exceeds Column Length (ORA-12899)

---

## 2026-09-11

**1. Issue reported**
Ms. Oanh reported that saving an Order fails after editing Color #2 on screen SA1000031 (Order Solid detail), P/O No `202607-0307HSATZ`, Item `VS-5332_CB (PERY)`. The system returned:
```
ORA-20999: ORA-12899: value too large for column 'SAMILV2'.'SA_ORDER_PRODCOLOR'.'GW_NOTE' (actual: 535, maximum: 500)
ORA-06512: at 'SAMILV2.SP_UPD_SA1000031_COLOR', line 177
```

**2. Root cause analysis**
Column `SA_ORDER_PRODCOLOR.GW_NOTE` was limited to 500 characters, while the note entered by the user was 535 characters long. Procedure `SP_UPD_SA1000031_COLOR` (line 177) writes this value directly into the column, so the value exceeded the limit and raised `ORA-12899`.

**3. Fix**
Increased the size of column `SAMILV2.SA_ORDER_PRODCOLOR.GW_NOTE` to accommodate the actual note length that occurs (535+ characters), instead of the hard 500-character cap. `SP_UPD_SA1000031_COLOR` did not need a logic change — the column just needed enough room.

**4. Result**
After enlarging the column, saving an Order after editing Color #2 works normally, with no more ORA-12899 error.
