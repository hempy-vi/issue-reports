# Dev Report — CESM-801

**Task:** [SAMIL] [BUG] Cannot Save Order After Editing Color #2 — GW_NOTE Value Exceeds Column Length (ORA-12899)

---

## 2026-09-11

**1. Ghi nhận lỗi**
Ms. Oanh báo lỗi không lưu được Order sau khi sửa Color #2 tại màn hình SA1000031 (Order Solid detail), P/O No `202607-0307HSATZ`, Item `VS-5332_CB (PERY)`. Hệ thống trả lỗi:
```
ORA-20999: ORA-12899: value too large for column 'SAMILV2'.'SA_ORDER_PRODCOLOR'.'GW_NOTE' (actual: 535, maximum: 500)
ORA-06512: at 'SAMILV2.SP_UPD_SA1000031_COLOR', line 177
```

**2. Phân tích nguyên nhân**
Cột `SA_ORDER_PRODCOLOR.GW_NOTE` giới hạn 500 ký tự, trong khi giá trị note người dùng nhập vào dài 535 ký tự. Procedure `SP_UPD_SA1000031_COLOR` (line 177) insert/update thẳng giá trị này vào cột nên vượt giới hạn và raise `ORA-12899`.

**3. Khắc phục**
Tăng kích thước cột `SAMILV2.SA_ORDER_PRODCOLOR.GW_NOTE` để chứa được độ dài note thực tế phát sinh (535 ký tự trở lên), thay vì giới hạn cứng 500. `SP_UPD_SA1000031_COLOR` không cần sửa logic, chỉ cần cột đủ chỗ chứa.

**4. Kết quả**
Sau khi tăng size cột, thao tác lưu Order sau khi sửa Color #2 chạy bình thường, không còn lỗi ORA-12899.
