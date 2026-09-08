# BC Report — CESM-418

**Task:** [DONGIL][Requirement] Enforce 100% USOT3137 Allocation Rule for All Finished Goods Lots in Monthly Lot Tracking (August 2026)

---

## 2026-09-08

**1. Xác định đúng job và dữ liệu liên quan**

Xác định được đây là job tính lot tracking chạy tự động mỗi đêm, tự tính lại toàn bộ số liệu
của tháng hiện tại (không nhận tham số, luôn lấy theo tháng hệ thống hiện tại). Xác định được
màn hình liên quan (PM0407 - Material Detail tab) và 8 mã nguyên liệu bông thô đang được dùng
trong hệ thống, trong đó có mã đã duyệt (USOT3137) và các mã chưa duyệt (vd BRAMID).

**2. Xác nhận vấn đề "mang số dư qua tháng" bằng dữ liệu thật**

Kiểm tra dữ liệu thực tế cho thấy: khi job chạy mỗi đêm, nó xoá sạch và tính lại toàn bộ dữ
liệu tháng hiện tại, sau đó mang số dư nguyên liệu chưa dùng hết của tháng trước sang tháng
này — nhưng giữ NGUYÊN mã nguyên liệu cũ (kể cả khi đó là mã chưa được duyệt). Kiểm chứng cụ
thể: 1 lô trộn nguyên liệu vẫn còn tồn hơn 200 tấn nguyên liệu chưa duyệt, số liệu này lặp lại
y hệt qua 3 tháng liên tiếp mà không hề giảm đi hay được thay thế.

**3. Xác nhận cơ chế phân bổ nguyên liệu cho lot thành phẩm**

Xác nhận job phân bổ nguyên liệu cho từng lot thành phẩm theo đúng tỷ lệ pha trộn đã định sẵn
từ khi lập kế hoạch sản xuất (Work Instruction) — ví dụ 1 kế hoạch có thể định sẵn 60% nguyên
liệu A / 40% nguyên liệu B, và tỷ lệ này không đổi theo thời gian.

**4. Soạn phương án sửa ban đầu (2 điểm)**

Soạn phương án: tại 2 điểm phát hiện được (mang số dư qua tháng, và phân bổ theo tỷ lệ), tự
động thay thế mọi mã nguyên liệu chưa duyệt bằng mã đã duyệt (USOT3137), giữ nguyên tổng khối
lượng — không thay đổi số lượng, chỉ đổi nhãn mã nguyên liệu.

**5. Kiểm tra chéo độc lập — phát hiện thêm 1 điểm quan trọng bị bỏ sót**

Thực hiện 1 vòng kiểm tra chéo độc lập (1 người tự rà lại toàn bộ job từ đầu mà không xem
phương án đã soạn, cộng thêm 2 người phản biện riêng về mặt kỹ thuật và về mặt rủi ro nghiệp
vụ/kế toán). Kết quả:
- Phát hiện: ngoài 2 điểm đã sửa, còn 1 điểm THỨ 3 hoàn toàn bị bỏ sót — nơi ghi nhận nguyên
  liệu chuyển kho thực tế vào kho pha trộn. Kiểm tra dữ liệu thật cho thấy đây mới là nguồn
  phát sinh nguyên liệu chưa duyệt LỚN NHẤT (hơn 4,300 tấn đã đi qua đường này), lớn hơn cả số
  dư mang qua tháng. Đã bổ sung sửa điểm này.
- Xác nhận 2 điểm sửa ban đầu là đúng, phạm vi áp dụng đúng (chỉ ảnh hưởng nhóm nguyên liệu
  bông thô, không đụng tới nguyên liệu khác).
- Phát hiện thêm 1 lỗi có sẵn trong hệ thống (không liên quan tới yêu cầu này): số dư nguyên
  liệu mang qua tháng không bao giờ được trừ đi phần đã thực sự tiêu thụ — số dư 1 lô cụ thể
  giữ nguyên y hệt suốt 3 tháng liên tiếp dù vẫn có tiêu thụ thực tế mỗi tháng. Đây là vấn đề
  ảnh hưởng tới MỌI loại nguyên liệu (không riêng vụ USOT3137 này), nên khuyến nghị báo cáo
  thành 1 yêu cầu riêng, không gộp xử lý cùng lúc.
- Xác nhận thêm 1 vướng mắc vận hành: vì job luôn tự tính theo tháng hệ thống hiện tại (không
  chọn tháng được), nếu chạy job vào ngày 10/09/2026 như lịch dự kiến, hệ thống lúc đó đã sang
  tháng 9 nên job sẽ tính lại cho tháng 9, KHÔNG tính lại cho tháng 8 đã đóng sổ — cần có 1
  bước thao tác riêng để chạy lại đúng cho tháng 8.

**6. Bổ sung điểm sửa còn thiếu**

Bổ sung phương án sửa cho điểm thứ 3 (nguyên liệu chuyển kho vào kho pha trộn) phát hiện ở
bước 5, theo đúng nguyên tắc: chỉ đổi nhãn mã nguyên liệu, không đổi số lượng.

**7. Theo yêu cầu, soạn sẵn 1 bản chỉnh sửa hoàn chỉnh, sẵn sàng để DBA áp dụng**

Ghép toàn bộ nội dung gốc của job với đầy đủ 3 điểm sửa thành 1 file hoàn chỉnh, có đánh dấu rõ
từng chỗ đã sửa để người kiểm tra dễ đối chiếu.

**8. Xử lý lỗi khi thử chạy thử (compile) file đã soạn**

User thử chạy thử file bằng công cụ quản trị Oracle (Toad) và báo lỗi cú pháp. Đã kiểm tra kỹ
lưỡng bằng công cụ so sánh riêng, xác nhận 3 điểm sửa hoàn toàn không phải nguyên nhân. Để xác
minh chắc chắn, đã tạo thêm 1 bản "đối chứng" — chính là bản gốc chưa sửa gì cả — để user thử
chạy song song: bản đối chứng CŨNG bị lỗi y hệt, chứng minh vấn đề nằm ở khâu sao chép/tái tạo
nội dung gốc, không phải ở nội dung đã sửa.

Xác định được nguyên nhân: khi sao chép nội dung gốc của job, đã dừng lại thiếu mất đúng 1 dòng
kết thúc ở cuối cùng (một block lồng bên trong đã comment hết code cũ ở giữa khiến dễ nhầm
tưởng đã hết bài, nhưng thực tế có thêm ĐÚNG 1 dòng kết thúc thật ở tít phía sau).

**9. Đã sửa xong**

Bổ sung lại đúng 1 dòng kết thúc còn thiếu vào cả bản đã sửa lẫn bản đối chứng, kiểm tra lại
bằng công cụ riêng xác nhận không còn lỗi cấu trúc nào. Đã gửi lại cho user thử chạy lại.

---

**Các điểm còn cần user/phía DONGIL xác nhận trước khi đưa vào chạy chính thức:**
1. Số hiệu lô hàng (lot number) gốc có được giữ nguyên khi đổi mã nguyên liệu hay không.
2. Số lượng nguyên liệu chưa duyệt còn tồn (sẽ không bao giờ bị trừ nữa trên sổ sách của job
   này) có ảnh hưởng gì tới cách DONGIL theo dõi tồn kho/giá vốn theo từng loại nguyên liệu.
3. Xác nhận quy trình chạy lại cho đúng tháng 8 có khớp với dự tính của DONGIL/consultant.
4. Xác nhận phạm vi áp dụng vĩnh viễn cho toàn bộ nhóm nguyên liệu bông thô, không giới hạn
   thời gian, có đúng ý muốn lâu dài hay không.
5. Có muốn báo cáo riêng lỗi phụ phát hiện được ở bước 5 (số dư không được trừ theo tiêu thụ)
   thành 1 yêu cầu/ticket khác hay không.
