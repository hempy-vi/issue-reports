# BC Report — CESM-801

**Task:** [SAMIL] [BUG] Cannot Save Order After Editing Color #2 — GW_NOTE Value Exceeds Column Length (ORA-12899)

---

## 2026-09-11

**1. Ghi nhận lỗi**
Ms. Oanh báo không lưu được đơn hàng (Order) sau khi chỉnh sửa thông tin màu #2, hệ thống hiện thông báo lỗi liên quan đến độ dài dữ liệu ghi chú (note) vượt quá giới hạn cho phép.

**2. Nguyên nhân**
Ô ghi chú màu (note) trong hệ thống có giới hạn độ dài, trong khi nội dung người dùng nhập vào dài hơn giới hạn này nên bị từ chối lưu.

**3. Khắc phục**
Mở rộng giới hạn độ dài cho ô ghi chú để đủ chứa nội dung người dùng thực tế cần nhập.

**4. Kết quả**
Người dùng lưu đơn hàng sau khi sửa màu #2 bình thường, không còn báo lỗi.
