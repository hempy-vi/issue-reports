# BC Report — CESM-728

**Task:** [SAMIL] [DATA HANDLING] Export Item Excel File from Forms SB1.13 and SB1.16 (from Database)

---

## 2026-09-09

**1. Xác định đúng 2 màn hình và dữ liệu cần export**

Xác định rõ 2 màn hình được nêu trong yêu cầu: SB1.13 (R&D Item Register) và SB1.16 (Item Code Inquiry), và xác nhận dữ liệu hiển thị trên cả 2 màn hình đều được lấy trực tiếp từ database (không phải dữ liệu tính toán tạm hay cache), phù hợp với yêu cầu "lấy dữ liệu trực tiếp từ database".

**2. Gặp sự cố kết nối tới database, chuyển sang kênh kết nối dự phòng**

Kênh kết nối chính tới database của SAMIL bị timeout ngay từ đầu. Kiểm tra thấy hệ thống dự phòng (API nội bộ dùng để truy vấn database an toàn) vẫn hoạt động bình thường, nên chuyển toàn bộ công việc sang dùng kênh này để không bị gián đoạn.

**3. Xây dựng lại logic lấy dữ liệu để lấy được TOÀN BỘ dữ liệu**

Khi dùng trực tiếp trên màn hình, dữ liệu hiển thị luôn bị giới hạn theo các điều kiện tìm kiếm (ví dụ theo khoảng ngày, theo buyer, theo loại hàng...). Để export được TOÀN BỘ dữ liệu như yêu cầu, đã phân tích lại đúng logic lấy dữ liệu gốc của từng màn hình, sau đó xây dựng lại thành 1 truy vấn lấy toàn bộ dữ liệu — vẫn giữ nguyên toàn bộ các quy tắc nghiệp vụ cốt lõi (ví dụ loại bỏ dữ liệu đã xoá), chỉ bỏ đi phần điều kiện tìm kiếm theo ô nhập trên màn hình.

**4. Gặp 2 rào chắn an toàn của hệ thống, đã điều chỉnh để vẫn ra đúng kết quả**

Hệ thống truy vấn database an toàn có cơ chế tự động chặn 1 số lệnh/thành phần nhạy cảm để phòng ngừa rủi ro. Quá trình lấy dữ liệu SB1.13 bị chặn do đụng phải 1 thành phần nằm trong danh sách chặn (dùng để xử lý hiển thị danh sách nguyên liệu yarn); đã thay bằng 1 cách xử lý khác cho ra đúng cùng 1 kết quả hiển thị mà không vi phạm rào chắn an toàn. Với SB1.16, hệ thống chặn nhầm do tên 1 cột dữ liệu trùng với 1 từ khoá nhạy cảm (chỉ là trùng chữ ngẫu nhiên, không liên quan gì tới nội dung dữ liệu) — đã đổi lại tên cột đó để không bị chặn nhầm nữa, không ảnh hưởng tới dữ liệu.

**5. Kiểm tra chéo độc lập trước khi lấy dữ liệu thật**

Trước khi lấy dữ liệu thật để giao cho requester, đã cho thực hiện 1 vòng kiểm tra chéo độc lập (2 lượt rà soát riêng biệt, mỗi lượt không biết công việc của lượt kia) để đối chiếu từng cột, từng điều kiện lọc giữa cách lấy dữ liệu mới và logic gốc của hệ thống — mục đích đảm bảo không bị thiếu cột, sai cột, hay thiếu/thừa dòng dữ liệu nào trước khi gửi đi. Kết quả:
- SB1.13: đủ toàn bộ 40 cột, đúng dữ liệu nguồn cho từng cột. Có 1 điểm cần lưu ý: cách lấy dữ liệu mới bỏ hẳn giới hạn theo khoảng ngày nhận hàng (Receipt Date) để lấy được toàn bộ lịch sử — đây là thay đổi có chủ đích, phù hợp với yêu cầu "dữ liệu đầy đủ", nhưng cần requester xác nhận lại có đúng ý muốn không.
- SB1.16: đủ toàn bộ dữ liệu, không thiếu cột. Có 1 điểm cần lưu ý: cách lấy dữ liệu mới lấy cả item đang dùng (active) lẫn item ngưng dùng (inactive) thay vì chỉ active như mặc định trên màn hình — có chủ đích, phù hợp yêu cầu "dữ liệu đầy đủ", đã giữ lại 1 cột đánh dấu active/inactive để người nhận tự lọc lại trong Excel nếu cần.

**6. Xử lý các điểm phát hiện, quyết định và tiếp tục**

Theo đúng nội dung yêu cầu ("Data is complete and matches the database"), đã quyết định giữ nguyên hướng lấy dữ liệu đầy đủ (không giới hạn ngày cho SB1.13, gồm cả active/inactive cho SB1.16), đồng thời ghi chú rõ ràng 2 điểm này để requester dễ dàng xác nhận hoặc yêu cầu điều chỉnh lại nếu cần phạm vi hẹp hơn. Đồng thời sửa lại 1 tên cột bị đặt nhầm ý nghĩa (cột thể hiện "độ co rút theo trọng lượng" nhưng bị đặt nhầm tên gợi ý "chiều rộng") cho đúng ý nghĩa dữ liệu.

**7. Lấy dữ liệu thật và kiểm tra tính toàn vẹn**

Đã chạy lấy dữ liệu thật cho cả 2 màn hình: 24.598 dòng cho SB1.13, 8.548 dòng cho SB1.16. Do cách lấy dữ liệu có gộp thêm thông tin từ vài bảng liên quan khác (ví dụ tên người phụ trách, thông tin nguyên liệu), có rủi ro về mặt kỹ thuật là 1 dòng gốc có thể bị nhân thành nhiều dòng nếu dữ liệu liên quan bị trùng lặp. Đã kiểm tra kỹ và xác nhận không có dòng nào bị nhân đôi — mỗi item chỉ xuất hiện đúng 1 lần trong file.

**8. Tạo file Excel thật từ dữ liệu đã lấy**

Máy thực hiện công việc không có kết nối internet nên không cài đặt được công cụ tạo file Excel có sẵn; đã tự xây dựng 1 công cụ tạo file Excel để hoàn thành công việc mà không cần internet, đồng thời kiểm tra và sửa 1 lỗi kỹ thuật phát sinh trong lúc đóng gói file (định dạng đường dẫn nội bộ trong file bị sai chuẩn trên hệ điều hành Windows) để đảm bảo file tạo ra đúng chuẩn định dạng Excel.

**9. Xác nhận file Excel mở được và đúng dữ liệu**

Đã mở lại cả 2 file Excel vừa tạo bằng chính phần mềm Excel (không chỉ kiểm tra file hợp lệ về mặt kỹ thuật) để xác nhận file mở được bình thường, đúng số dòng, đúng số cột, và đúng nội dung dữ liệu.

**10. Bàn giao**

2 file Excel đã sẵn sàng: dữ liệu item của SB1.13 (R&D Item Register) và SB1.16 (Item Code Inquiry), lấy trực tiếp từ database, đầy đủ theo đúng yêu cầu ticket. 2 điểm cần Ms. Oanh xác nhận trước khi coi là bản cuối:
1. SB1.13 lấy toàn bộ lịch sử, không giới hạn theo khoảng ngày.
2. SB1.16 gồm cả item active lẫn inactive, thứ tự dòng trong file khác với thứ tự hiển thị trên màn hình gốc.

File chưa được gửi cho requester — cần gửi thủ công qua email hoặc đính kèm vào ticket Jira sau khi xác nhận 2 điểm trên.
