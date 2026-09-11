# BC Report — CESM-801

**Task:** [SAMIL] [BUG] Cannot Save Order After Editing Color #2 — GW_NOTE Value Exceeds Column Length (ORA-12899)

---

## 2026-09-11

**1. Issue reported**
Ms. Oanh reported that an Order could not be saved after editing Color #2, with the system showing an error related to a note field exceeding its allowed length.

**2. Root cause**
The color note field had a length limit, while the content entered by the user was longer than that limit, so the save was rejected.

**3. Fix**
Extended the length limit of the note field so it can hold the actual content users need to enter.

**4. Result**
Users can now save an Order after editing Color #2 normally, with no more error.
