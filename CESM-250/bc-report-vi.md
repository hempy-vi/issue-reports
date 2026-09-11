# BC Report — CESM-250

**Task:** WePOP Issue Barcode Label (Stock In Request): Label Modify

---

## 2026-09-11

**1. Tiếp nhận yêu cầu chỉnh nhãn in**
Trên màn hình 'Issue Barcode Labels (Stock In Request)' của WePOP, yêu cầu chỉnh mẫu nhãn in mã vạch: (1) tăng cỡ chữ số PO No cho dễ đọc hơn, (2) đổi tên trường 'Roll' thành 'LOCATION' cho đúng thuật ngữ nghiệp vụ của site.

**2. Xác định đúng vị trí cần sửa trên mẫu nhãn**
Xác định đúng 2 vùng trên mẫu nhãn cần chỉnh: vùng hiển thị giá trị PO No (ô số ngay dưới tiêu đề PO No) và vùng tiêu đề của ô 'Roll' (không đụng tới giá trị mã vị trí bên dưới nó).

**3. Kiểm tra cơ chế tự co chữ của mẫu nhãn**
Mẫu nhãn có sẵn cơ chế tự động thu nhỏ chữ nếu nội dung dài không vừa khung in, để tránh chữ tràn ra ngoài khung. Cơ chế này chỉ co nhỏ, không tự phóng to — nên việc tăng cỡ chữ gốc là an toàn, không làm vỡ layout khi PO No dài.

**4. Áp dụng thay đổi**
Tăng cỡ chữ hiển thị PO No và đổi tên tiêu đề 'Roll' thành 'LOCATION' trên mẫu nhãn IBL530.

**5. Kiểm tra biên dịch chương trình**
Build thử chương trình để đảm bảo thay đổi không gây lỗi. Phần liên quan tới 2 thay đổi biên dịch thành công; có 1 lỗi phát sinh nhưng thuộc về một mẫu nhãn khác (không liên quan tới thay đổi lần này) và do công cụ build môi trường thử nghiệm, không ảnh hưởng tới bản chính thức.

**6. Phát hiện lỗi hiển thị dư ký tự khi kiểm tra bản in thử**
Trong lúc kiểm tra bản in thử với dữ liệu thật, phát hiện thêm: giá trị PO No trên 1 số nhãn (nhóm phiếu nhập từ nhà cung cấp AD) bị dính thêm 1 ký tự khoảng trắng đặc biệt (tab) ở cuối, khiến giá trị hiển thị/khớp dữ liệu không sạch.

**7. Truy nguyên nguồn gốc lỗi**
Xác định ký tự thừa này đã có sẵn trong dữ liệu PO No gốc được nhập vào hệ thống từ trước, không phải do lần chỉnh sửa nhãn lần này gây ra. Hàm xử lý dữ liệu hiện tại chỉ loại bỏ khoảng trắng thông thường ở đầu/cuối, không loại được loại ký tự tab này, nên ký tự thừa vẫn lọt ra ngoài.

**8. Kiểm chứng và chuẩn bị bản sửa**
Đã kiểm chứng cách xử lý đúng để loại bỏ ký tự thừa này, xác nhận cho ra kết quả sạch trên dữ liệu thật. Đã chuẩn bị sẵn bản sửa để áp dụng khi có xác nhận từ phía quản trị hệ thống.
