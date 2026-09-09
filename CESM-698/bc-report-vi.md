# BC Report — CESM-698

**Task:** [SAMIL] [BUG] POP Shows Inconsistent Information Between Inner and Outer Screens (Order 202606-0420SAWMT)

---

## 2026-09-09

**1. Xác định 2 màn hình bị báo lỗi**

Từ 3 hình ảnh đính kèm ticket, xác định đúng 2 màn hình trên ứng dụng POP (Windows) tại xưởng đang hiển thị lệch nhau cho cùng 1 đơn hàng: màn hình nhập liệu thẻ công đoạn hiển thị chỉ số G/M2 là **265**, còn màn hình kiểm vải theo cuộn hiển thị **250**. Đối chiếu với màn hình đơn hàng gốc trên hệ thống web thì chỉ số yêu cầu đúng của đơn hàng là **250** — tức màn hình đầu tiên đang hiển thị sai.

**2. Sự cố kết nối hệ thống**

Trong lúc điều tra, kết nối tới hệ thống dữ liệu của SAMIL bị gián đoạn do VPN. Đã báo user kiểm tra lại kết nối; sau khi VPN được nối lại, tiếp tục điều tra bình thường.

**3. Xác định nguyên nhân gốc**

Xác nhận được nguyên nhân: đơn hàng `202606-0420SAWMT` từng bị chỉnh sửa lại số liệu yêu cầu (trọng lượng) vào ngày 07/09 do nhập sai lúc đầu. Màn hình kiểm vải theo cuộn luôn lấy số liệu yêu cầu mới nhất của đơn hàng nên hiển thị đúng. Ngược lại, màn hình nhập liệu thẻ công đoạn lại đang lấy số liệu được "chụp lại" 1 lần duy nhất từ lúc thẻ công đoạn được tạo ra — nếu thẻ được tạo trước thời điểm đơn hàng bị chỉnh sửa, số liệu đó bị đứng yên mãi mãi, không tự cập nhật theo bản chỉnh sửa mới. Đã kiểm tra dữ liệu thật và xác nhận: 17 trong 20 thẻ công đoạn của đơn hàng này đang bị "kẹt" số liệu cũ theo đúng cơ chế trên, khớp chính xác với chỉ số sai 265 thấy trên màn hình.

**4. Rà soát mở rộng toàn hệ thống**

Để đảm bảo không còn màn hình nào khác trong hệ thống POP mắc lỗi tương tự, đã rà soát toàn bộ các chức năng liên quan dùng chung công thức tính chỉ số này. Phát hiện thêm **1 chức năng khác đang hoạt động thật** (thuộc module quản lý thẻ công đoạn phiên bản khác) cũng mắc đúng lỗi này. Các chức năng còn lại được rà soát đều xác nhận không bị ảnh hưởng — hoặc do dùng cơ chế lấy số liệu khác an toàn hơn, hoặc do là phiên bản thử nghiệm nội bộ của lập trình viên không được người dùng thật nào sử dụng.

**5. Khắc phục**

Đã chuẩn bị bản sửa cho cả 2 chức năng bị ảnh hưởng (chức năng gốc trong ticket + chức năng phát hiện thêm), chỉnh lại đúng chỗ để luôn lấy số liệu yêu cầu mới nhất của đơn hàng, đảm bảo mọi màn hình hiển thị nhất quán. Không cần chỉnh sửa lại dữ liệu cũ trong hệ thống — sau khi áp dụng bản sửa, mọi thẻ công đoạn (kể cả các thẻ cũ đã bị kẹt số liệu) sẽ tự động hiển thị đúng theo số liệu mới nhất của đơn hàng.

Sau khi bản sửa được áp dụng vào hệ thống, user xác nhận đã hiểu và đóng ticket.
