# BC Report — CESM-785

**Task:** Fix Roll Location/Date Mismatch Between Warehouses HOQC301 and HDF102

---

## 2026-09-10

**1. Tiếp nhận yêu cầu**
Một cuộn vải (barcode D25091920979, đơn hàng 202508-0343HSAEROW, lot 005, roll 030R) đã được chuyển kho từ HOQC301 sang HDF102, nhưng hệ thống bị lệch dữ liệu: vào kho HOQC301 vẫn thấy cuộn vải nhưng không xuất được (báo hết tồn kho); vào kho HDF102 (kho đích thực tế) thì không tìm thấy cuộn vải để chọn. Bộ phận nghiệp vụ yêu cầu cập lại ngày nhập kho = 20/09/2025 và kho thực tế = HDF102.

**2. Tìm nguyên nhân báo hết tồn kho ở HOQC301**
Kiểm tra lịch sử nhập/xuất/chuyển kho của cuộn vải, phát hiện: 2 phiếu nhập kho gốc lập ngày 20/09/2025 đã bị hủy trước đó (có thể do nhập trùng), phiếu chuyển kho HOQC301→HDF102 đã chạy đúng lúc 20/09/2025, sau đó một phiếu nhập kho thay thế được lập lại nhưng vào NGÀY 21/09/2025 — tức là sau khi hàng đã chuyển kho. Kết quả: theo đúng dữ liệu hệ thống, tồn kho tại HOQC301 hiện = 0 (đã chuyển hết sang HDF102), nên báo lỗi hết tồn kho là đúng theo số liệu — vấn đề nằm ở chỗ khác.

**3. Tìm nguyên nhân không thấy cuộn vải ở HDF102**
Xác định được: màn hình tra cứu vị trí cuộn vải (Item Barcode Checking) và màn hình chọn cuộn vải để lập phiếu đang lấy thông tin "kho hiện tại" từ một bảng dữ liệu riêng (không phải từ sổ tồn kho), và bảng này CHƯA được cập nhật khi phiếu chuyển kho chạy — vẫn còn ghi kho hiện tại là HOQC301 dù cuộn vải đã thực sự nằm ở HDF102. Đây chính là lý do cuộn vải "vẫn thấy" ở HOQC301 (dù không xuất được) và "không có để chọn" ở HDF102 (dù hàng đã đủ ở đó).

**4. Kiểm tra an toàn trước khi sửa**
Đã kiểm tra các bút toán liên quan chưa thuộc kỳ đóng sổ nào (không ảnh hưởng báo cáo tài chính/kế toán đã chốt) → an toàn để điều chỉnh.

**5. Chuẩn bị bản sửa dữ liệu**
Đã soạn sẵn script cập nhật:
- Ngày nhập kho của phiếu đang active: 21/09/2025 → 20/09/2025 (đúng thời điểm hàng thực nhập, khớp với ngày chuyển kho).
- Kho hiện tại của cuộn vải trên bảng vị trí: HOQC301 → HDF102 (có sao lưu giá trị cũ để có thể khôi phục nếu cần).

**6. Ghi chú vận hành**
Phiếu xuất kho nháp cũ (đang lưu tạm, chưa xác nhận) được lập chọn kho HOQC301 do lúc đó bị hiển thị nhầm vị trí — sau khi dữ liệu được sửa, cuộn vải sẽ không còn xuất hiện ở HOQC301 nữa. Cần hủy phiếu nháp cũ và lập lại phiếu xuất kho mới, chọn đúng kho HDF102.
