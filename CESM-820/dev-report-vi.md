# Dev Report — CESM-820

**Task:** Allow an Optional Trailing Letter After Lot No on Print Label (e.g. LOT 036 -> LOT 036A)

---

## 2026-09-11

**1. Xac dinh diem vao trong source**
Theo yeu cau tu Sales (Ms.Oanh), file duoc chi dinh la
`frmGC0011.cs`, control `lbLotNo` (thuc chat la 1 `Label`, khong
phai `TextBox` — nguoi dung click vao no de mo ban phim ao dung
chung `frmQC0013`, gom numpad + cac nut chu 'A/B/C/D/F' + to hop
'QA'/'AC' + cac ky tu '- / \"'). Ban phim nay dung chung cho nhieu
field khac trong form (weight, length, loss, grade...), moi field
tu xu ly chuoi tra ve theo nhu cau rieng.

**2. Phan tich handler `lbLotNo_Click`**
Handler nay (truoc khi sua) strip toan bo ky tu chu 'A/B/C/D/F/Q'
va cac ky tu '- / \"' khoi input, roi `PadLeft(3, '0')`. Day chinh
la ly do anh chup man hinh cho thay nguoi dung nhap `39A` nhung Lot
No hien thi `039` (chu A bi strip mat).

**3. Truy vet duong luu de dam bao khong vo neu them ky tu chu**
Duong luu (`btnSave_Click` -> stored procedure
`sp_upd_pop_p_v1_qc0011`): tham so `p_lot_no` khai bao kieu
`VARCHAR2`, insert thang vao cot `SA_QC_D_P.LOT_NO`
(`VARCHAR2(100)`) — khong co `TO_NUMBER`/ep kieu so o duong nay.
```sql
SELECT line, text
FROM user_source
WHERE name = 'SP_UPD_POP_P_V1_QC0011' AND type = 'PROCEDURE' AND UPPER(text) LIKE '%LOT_NO%'
ORDER BY line;
```

**4. Truy vet duong in tem thuc su dang dung**
`OnPrint2()` (duoc goi tu `btnSave_Click`, khac voi method cu
`OnPrint()` khong ai goi) -> `Report.getLabel()` (class dung chung
`PACK_LABEL_SAMIL/Report.cs`) -> stored procedure `sp_rpt_pop_d_p_v2`.
SP nay lay `a.lot_no lot` truc tiep (passthrough, khong
`TO_NUMBER`). Co 1 cot phu `NEW_LOT_NO` dung
`LPAD(a.lot_no, 4, '0')` cho vai bien the label (EXPORT/TORAY) —
`LPAD` hoat dong binh thuong tren chuoi co chu, khong loi.
```sql
SELECT line, text
FROM user_source
WHERE name = 'SP_RPT_POP_D_P_V2' AND type = 'PROCEDURE' AND UPPER(text) LIKE '%LOT%'
ORDER BY line;
```

**5. Phat hien method cu co rui ro nhung xac nhan la dead code**
Method `OnPrint()` trong `frmGC0011.cs` dung stored procedure
`sp_rpt_pop_p_qc0011` voi `TO_NUMBER(a.lot_no)` — se loi neu lot_no
co them ky tu chu. Grep toan bo `frmGC0011.cs` xac nhan khong co
noi nao goi `OnPrint()` (chi `OnPrint2()` duoc goi khi Save), nen
day la dead code, khong anh huong toi feature dang xet.

**6. Ap dung fix trong `lbLotNo_Click`**
Truoc khi strip ky tu chu nhu cu: neu ky tu cuoi cung cua input la
chu cai, tach rieng ra lam `suffix` (giu nguyen, khong strip), phan
con lai xu ly y het logic cu (strip chu, pad 3 chu so). Ket qua
cuoi: `lbLotNo.Text = a.PadLeft(3, '0') + suffix`. Neu khong nhap
chu, `suffix = ""`, hanh vi y het truoc day.
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

**7. Xac nhan lai tu Sales — dung, revert code**
Sau khi fix da san sang, Sales bao dung lai vi ben mua hang (Han
Quoc) da trao doi qua lai va quyet dinh khong ap dung thay doi nay
nua. Da revert file `frmGC0011.cs` ve dung nguyen trang ban dau
(`git checkout` tai repo thuc te), xac nhan `git status` sach,
khong con thay doi nao ton dong tren he thong. Task duoc dong o
trang thai hold, khong deploy.
