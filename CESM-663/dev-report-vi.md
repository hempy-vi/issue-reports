# Dev Report — CESM-663

**Task:** [SHINWOO][BUG] WePOP - IBL040 Prod Label Issue (SO-ETD Divide)

---

## 2026-09-11

**1. Ghi nhận bug từ report**
BA report cho SO `SOOT202608170010`: khi bấm **Print** (in đơn lẻ), label
hiển thị đúng layout tuỳ chỉnh với 2 dòng `Order Qty` / `Shipment Qty`
tách riêng (802816 pcs / 81 Roll và 826920.48 pcs / 83 Roll). Khi bấm
**Print All** (in hàng loạt) cho cùng SO, label lại load một template
khác/cũ hơn — chỉ có 1 dòng `Qty` gộp (802816 pcs / 802816 ROLL, sai vì
số Roll bị gán trùng số pcs thay vì tính riêng).

**2. Fix**
Đồng bộ để cả 2 action **Print** và **Print All** cùng gọi chung 1 report
template file và cùng 1 dataset handler (thay vì Print All rẽ nhánh sang
template/handler cũ). Store/handler liên quan đã được sửa trực tiếp
(không qua session log riêng của workspace này).
