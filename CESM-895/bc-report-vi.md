# BC Report — CESM-895

**Task:** Clarify How Form SS2.5.1 Data Is Populated (User-Loaded vs Auto-Updated) with Examples

---

## 2026-09-14

**1. Xác định form SS.2.5.1 (Margin Table After Cost) là do nhân viên tự nhập hay hệ thống tự động**

Kiểm tra dữ liệu order mẫu `202608-0025HNW036`: mỗi dòng dữ liệu Margin Table đều ghi lại rõ ai là người tạo/sửa và lúc nào. Order này do nhân viên **Thanh** (Nguyễn Trần Thanh Thanh) lưu ngày 3/8/2026.

**2. Cho thêm ví dụ vài order khác kèm tên user đã load**

Kiểm tra thêm nhiều order gần đây, tất cả đều gắn với tên nhân viên cụ thể (Hiền, Thuận, Fotl0320...) — không có dòng nào gắn với 1 tài khoản hệ thống/tự động. Riêng 1 khối dữ liệu cũ (~13.900 dòng) là do chuyển đổi 1 lần từ hệ thống cũ khi công ty đổi sang hệ thống hiện tại, không phải chạy định kỳ.

**3. Xác nhận không có cơ chế tự động cập nhật ngầm**

Kiểm tra kỹ hệ thống: không có cơ chế nào (không có tác vụ nền, không có lịch chạy tự động) tự động ghi dữ liệu vào Margin Table. Dữ liệu chỉ được ghi khi có người thực sự bấm nút Save trên form.

**4. Giải đáp thắc mắc: nhân viên nói chỉ dùng SS.2.5, sao SS.2.5.1 vẫn có data?**

Xác nhận: SS.2.5 (Margin Table) và SS.2.5.1 (Margin Table After Cost) thực chất **cùng đọc và ghi trên 1 dữ liệu duy nhất**. Nhân viên nhập và lưu qua SS.2.5 — SS.2.5.1 chỉ là màn hình hiển thị/tính toán thêm trên đúng dữ liệu đó, không cần ai mở riêng SS.2.5.1 để có dữ liệu ở đây. Nên nhân viên phản ánh 'chỉ dùng SS.2.5' là hoàn toàn chính xác.

**5. Giải đáp thắc mắc: vì sao số 'Final quantity' khác nhau giữa 2 form cho cùng order (10,759 vs 8,069.3)**

Đây là 2 con số mang 2 ý nghĩa khác nhau, không phải lỗi:
- **10,759 kg** (SS.2.5) = số lượng **kế hoạch/đặt hàng** ban đầu.
- **8,069.3 kg** (SS.2.5.1) = số lượng **thực tế đã sản xuất và đạt kiểm tra chất lượng (QC)** — tự động lấy từ kết quả cân/kiểm từng cuộn vải ngoài xưởng (431 cuộn đạt trong tổng 468 cuộn được QC kiểm ngày 3/9/2026), không phải ai gõ tay trên form.

Chênh lệch giữa 2 số là do hao hụt/lỗi phát sinh trong quá trình sản xuất thực tế — hiện tượng bình thường trong ngành dệt may, không phải sai sót số liệu.

**6. Giải đáp thêm: vì sao không thấy số 8,069.3 trên 1 report QC khác mà user đối chiếu**

Trong quy trình sản xuất, hàng được kiểm chất lượng (QC) **2 lần ở 2 thời điểm khác nhau**: lần đầu ngay sau sản xuất (cho ra số 8,069.3 trên SS.2.5.1) và lần hai ngay trước khi đóng gói/xuất hàng (Outgoing QC — cho ra số khác, 8,542.4, trên report user đối chiếu). Đây là 2 mốc kiểm tra riêng biệt của cùng 1 lô hàng, nên ra 2 số khác nhau là bình thường, không phải sai lệch dữ liệu.
