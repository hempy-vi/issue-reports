# Dev Report — CESM-820

**Task:** Allow an Optional Trailing Letter After Lot No on Print Label (e.g. LOT 036 -> LOT 036A)

---

## 2026-09-11

**1. Locate the entry point in source**
Per the request from Sales (Ms.Oanh), the target file is
`frmGC0011.cs`, control `lbLotNo` (actually a `Label`, not a
`TextBox` — the user clicks it to open the shared virtual keypad
`frmQC0013`, which has a numpad + letter buttons 'A/B/C/D/F' +
combo buttons 'QA'/'AC' + symbols '- / \"'). This keypad is shared
across many other fields in the form (weight, length, loss, grade,
etc.), and each field handler decides what to keep or strip from
the returned string.

**2. Analyze the `lbLotNo_Click` handler**
This handler (before the fix) stripped every letter character
'A/B/C/D/F/Q' and the symbols '- / \"' from the input, then applied
`PadLeft(3, '0')`. This is why the screenshot showed the user
typing `39A` but Lot No displaying `039` (the letter A was
stripped).

**3. Trace the save path to make sure adding a letter would not break anything**
Save path (`btnSave_Click` -> stored procedure
`sp_upd_pop_p_v1_qc0011`): parameter `p_lot_no` is declared as
`VARCHAR2` and is inserted straight into column
`SA_QC_D_P.LOT_NO` (`VARCHAR2(100)`) — no `TO_NUMBER`/numeric
casting on this path.
```sql
SELECT line, text
FROM user_source
WHERE name = 'SP_UPD_POP_P_V1_QC0011' AND type = 'PROCEDURE' AND UPPER(text) LIKE '%LOT_NO%'
ORDER BY line;
```

**4. Trace the actual print path in use**
`OnPrint2()` (called from `btnSave_Click`, distinct from the older
unused `OnPrint()` method) -> `Report.getLabel()` (shared class
`PACK_LABEL_SAMIL/Report.cs`) -> stored procedure
`sp_rpt_pop_d_p_v2`. This SP reads `a.lot_no lot` directly
(passthrough, no `TO_NUMBER`). There is one extra column
`NEW_LOT_NO` using `LPAD(a.lot_no, 4, '0')` for a few label
variants (EXPORT/TORAY) — `LPAD` works fine on a string containing
a letter, no error.
```sql
SELECT line, text
FROM user_source
WHERE name = 'SP_RPT_POP_D_P_V2' AND type = 'PROCEDURE' AND UPPER(text) LIKE '%LOT%'
ORDER BY line;
```

**5. Found a risky legacy method, confirmed it is dead code**
Method `OnPrint()` in `frmGC0011.cs` uses stored procedure
`sp_rpt_pop_p_qc0011` with `TO_NUMBER(a.lot_no)`, which would fail
if lot_no had a trailing letter. Grepping the entire
`frmGC0011.cs` confirmed no caller invokes `OnPrint()` (only
`OnPrint2()` is called on Save), so this is dead code and does not
affect the feature under investigation.

**6. Applied the fix in `lbLotNo_Click`**
Before stripping letters as before: if the last character of the
input is a letter, extract it as a `suffix` (kept as-is, not
stripped); the remaining part is processed exactly as before (strip
letters, pad to 3 digits). Final result:
`lbLotNo.Text = a.PadLeft(3, '0') + suffix`. If no letter is typed,
`suffix = ""`, behavior is identical to before.
```csharp
string a = f._Value1;

string suffix = "";
if (a.Length > 0 && char.IsLetter(a[a.Length - 1]))
{
    suffix = a.Substring(a.Length - 1, 1);
    a = a.Substring(0, a.Length - 1);
}

a = a.Replace("A", "");
a = a.Replace("B", "");
a = a.Replace("C", "");
a = a.Replace("D", "");
a = a.Replace("F", "");
a = a.Replace("Q", "");
a = a.Replace("-", "");
a = a.Replace("/", "");
a = a.Replace("\"", "");
if (a != "")
{
    lbLotNo.Text = a.PadLeft(3, '0') + suffix;
}
```

**7. Re-confirmation from Sales — hold, revert code**
After the fix was ready, Sales reported the requirement was
withdrawn because the buyer side (Korea) had further discussions
and decided not to apply this change. The file `frmGC0011.cs` was
reverted to its original state (`git checkout` on the actual
repository), confirmed by a clean `git status` with no remaining
changes on the system. The task is closed in hold status, not
deployed.
