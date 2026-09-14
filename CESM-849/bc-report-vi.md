# BC Report — CESM-849

**Task:** Fix WePOP Batch Print Wrong Customer Data and Unresponsive Print All Button (IBL020/IBL040)

---

## 2026-09-14

**1. Nhận báo cáo lỗi và phân tích ban đầu**
Khách hàng SHINWOO báo 3 vấn đề trên phần mềm WePOP (chức năng in tem/label barcode):
- Khi in hàng loạt ("Print All"), tem in ra cho khách hàng ESTEC đôi khi hiển thị sai tên khách hàng (ra tên 1 công ty khác hoàn toàn), trong khi in từng tem lẻ luôn đúng.
- Layout tem của ESTEC có 1 dòng thông tin thừa so với mẫu tem chuẩn khách hàng cung cấp.
- Ở 1 màn hình quản lý tem (IBL020), nút "Print All" bấm không có phản hồi gì, không mở được bản in.

**2. Điều tra cơ chế in hàng loạt**
Rà soát toàn bộ quy trình xử lý khi hệ thống chọn mẫu tem để in — xác định có 4 phần xử lý dữ liệu (dùng chung giữa 2 màn hình quản lý tem khác nhau) chịu trách nhiệm quyết định tem sẽ hiển thị nội dung gì.

**3. Xác định nguyên nhân gốc**
Phát hiện: 2 màn hình quản lý tem (1 dành cho đơn hàng thông thường, 1 dành cho khách hàng đặc biệt "khách khác") dùng CHUNG 4 phần xử lý dữ liệu này, nhưng mỗi màn hình lại truyền vào 1 loại "chìa khóa" định danh dữ liệu khác nhau. Một lần chỉnh sửa trước đó (nhằm phục vụ đúng cho màn hình "khách khác") đã vô tình làm cho màn hình đơn hàng thông thường KHÔNG còn nhận diện đúng chìa khóa của mình:
- Trường hợp phổ biến: không tìm thấy dữ liệu khớp → hệ thống không in gì cả (đúng như lỗi "Print All không phản hồi").
- Trường hợp hiếm: do cách đánh số nội bộ của hệ thống, chìa khóa của 1 đơn hàng bị trùng số với 1 đơn hàng khác thuộc khách hàng KHÁC → hệ thống lấy nhầm dữ liệu của khách hàng đó để in (đúng như lỗi "hiện sai tên khách hàng").

Cả 2 lỗi thực chất là CÙNG 1 nguyên nhân.

**4. Khắc phục**
- Khôi phục lại cách nhận diện chìa khóa gốc cho màn hình đơn hàng thông thường, đồng thời vẫn giữ nguyên cách nhận diện hiện tại cho màn hình "khách khác" — cả 2 màn hình đều hoạt động đúng, không ảnh hưởng lẫn nhau.
- Bổ sung thông báo rõ ràng cho người dùng khi bấm in mà không có dữ liệu hoặc dữ liệu bị lỗi, thay vì hệ thống im lặng không phản hồi như trước — giúp người dùng biết ngay có vấn đề cần báo lại, không còn nhầm là hệ thống bị treo.

**5. Sự cố phát sinh ngoài dự kiến: toàn bộ chức năng in tem bị gián đoạn**
Trong quá trình thử tự khôi phục lại 1 số phần xử lý dữ liệu (bằng cách sao chép nội dung từ báo cáo lỗi gốc), 1 ký tự đặc biệt vô hình (không nhìn thấy được bằng mắt thường) đã bị lẫn vào nội dung khi sao chép từ 1 nguồn văn bản có định dạng — khiến hệ thống không hiểu được câu lệnh, dẫn tới TOÀN BỘ chức năng in tem (cả 2 màn hình, cho MỌI khách hàng, không riêng ESTEC) bị gián đoạn tạm thời.

**6. Chẩn đoán và khôi phục khẩn cấp**
Xác định chính xác đây là do ký tự vô hình lẫn vào từ việc sao chép, không phải lỗi logic. Soạn lại toàn bộ nội dung bằng cách nhập tay (không sao chép), kiểm tra kỹ không còn ký tự lỗi, rồi khôi phục lại toàn bộ 11 phần xử lý dữ liệu bị ảnh hưởng. Xác nhận chức năng in tem hoạt động lại bình thường ngay sau đó.

**7. Phát hiện thêm 2 phần bị ảnh hưởng tương tự**
Màn hình quản lý tem cho "khách khác" báo lỗi khi tìm kiếm đơn hàng — kiểm tra phát hiện 2 phần xử lý tìm kiếm cũng bị dính cùng lỗi ký tự vô hình từ lần sự cố trên (nằm ngoài phạm vi chức năng in nên chưa được phát hiện ở bước trước). Đã khôi phục xong, rà soát lại toàn hệ thống xác nhận không còn phần nào khác bị ảnh hưởng.

**8. Đề xuất cải tiến lâu dài (chưa thực hiện)**
Để tránh lặp lại sự cố tương tự trong tương lai (1 lần sửa cho màn hình này vô tình ảnh hưởng màn hình khác), đã chuẩn bị sẵn phương án tách hẳn 4 phần xử lý dữ liệu đang dùng chung ra thành 2 bộ riêng biệt cho 2 màn hình — chưa thực hiện, chờ xác nhận khi thuận tiện.

**Việc còn tồn (chưa xử lý trong phiên này):**
- Vấn đề layout tem thừa dòng thông tin cho ESTEC — cần rà soát kỹ hơn vì mẫu tem này đang dùng chung cho nhiều khách hàng khác, phải đảm bảo không ảnh hưởng tới các khách hàng đó khi sửa.
- Vấn đề tem cũ của khách hàng khác bị lẫn khi in hàng loạt ở màn hình "khách khác" — cần quyết định hướng xử lý dữ liệu lịch sử.
- Phương án tách riêng 2 màn hình (bước 8) — đã chuẩn bị, chưa thực hiện.
