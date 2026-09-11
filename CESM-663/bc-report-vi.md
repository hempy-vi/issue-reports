# BC Report — CESM-663

**Task:** [SHINWOO][BUG] WePOP - IBL040 Prod Label Issue (SO-ETD Divide)

---

## 2026-09-11

**1. Ghi nhận bug từ report**
BA report: cùng 1 đơn hàng (SO), nếu in bằng nút **Print** (in đơn lẻ) thì
label hiển thị đúng, tách riêng số lượng Order Qty và Shipment Qty. Nhưng
nếu in bằng nút **Print All** (in hàng loạt) thì label lại hiển thị sai —
dùng mẫu label cũ, chỉ gộp 1 dòng số lượng và số Roll bị hiển thị sai.

**2. Fix**
Đã chỉnh để nút **Print** và **Print All** luôn dùng chung đúng 1 mẫu
label chuẩn, tránh tình trạng in ra 2 kết quả khác nhau cho cùng 1 đơn
hàng.
