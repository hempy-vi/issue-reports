# BC Report — CESM-726

**Task:** [Samil] [Data handling] Impact Analysis – Yarn Code Changes in SB1.2

---

## 2026-09-10

**1. Xác định form nguồn cần điều tra**

Ticket liệt kê 5 form liên quan tới Yarn Code: Item Code (SB1.1), Yarn Code (SB1.2 — nơi phát sinh xoá/sửa), R&D Item Register (SB1.13), Item Code Inquiry (SB1.16), Yarn Code Inquiry (SB1.27). Đã xác nhận lại đúng 5 form này với user qua screenshot menu thật trên ứng dụng.

**2. Tìm ra cơ chế lưu trữ Yarn Code trong hệ thống**

Yarn Code trong hệ thống thực chất chỉ là 1 nhóm phân loại con của bảng danh mục Item Master dùng chung cho toàn bộ ERP (mọi loại hàng hoá: sợi, vải, hoá chất, thành phẩm... đều lưu chung 1 chỗ, chỉ khác mã phân loại nhóm).

- Khi **xoá** yarn code: hệ thống không xoá vật lý mà chỉ "ẩn" đi (soft-delete) — dữ liệu vẫn còn, chỉ không hiển thị nữa.
- Khi **sửa** (đổi tên/mã yarn): các form khác tự động cập nhật tên mới ngay lập tức, không cần thao tác gì thêm.

**3. Phát hiện lỗ hổng gốc rễ (root cause) của rủi ro trong ticket**

Hệ thống hiện có 1 lớp bảo vệ chặn xoá yarn code NẾU yarn đó đã từng phát sinh giao dịch nhập/xuất kho. Tuy nhiên lớp bảo vệ này KHÔNG kiểm tra trường hợp yarn đang được dùng trong công thức vải (Item) hoặc công thức R&D — nghĩa là 1 mã yarn đang được dùng trong công thức nhưng CHƯA từng có giao dịch kho vẫn có thể bị xoá mà không có bất kỳ cảnh báo nào cho người dùng. Đây chính là nguyên nhân kỹ thuật gốc rễ của rủi ro mà ticket mô tả.

**4. Xác định cụ thể hậu quả trên từng form khi xoá yarn code**

- Yarn Code (SB1.2) và Yarn Code Inquiry (SB1.27): dòng yarn bị xoá biến mất khỏi danh sách hiển thị.
- Item Code (SB1.1) và Item Code Inquiry (SB1.16): sản phẩm/vải vẫn hiển thị bình thường, chỉ riêng ô tên loại sợi tương ứng bị trống.
- R&D Item Register (SB1.13): công thức R&D vẫn hiển thị, nhưng loại sợi đó âm thầm biến mất khỏi bảng thành phần — không có cảnh báo gì cho người xem, dễ hiểu nhầm là "công thức này không dùng sợi đó".

**5. Mở rộng phạm vi điều tra theo yêu cầu — rà soát toàn bộ hệ thống**

Theo yêu cầu bổ sung, đã rà soát toàn bộ hệ thống (không chỉ 5 form ban đầu) để tìm mọi màn hình có liên quan tới Yarn Code, đối chiếu với danh mục menu thật của ứng dụng. Phát hiện thêm nhiều màn hình khác cũng bị ảnh hưởng: khu vực Kéo sợi/Dệt (Knitting), khu vực Nhà cung cấp sợi (Yarn Supplier), khu vực Đơn hàng và một số báo cáo sản lượng.

**6. Người dùng chỉ ra thêm 2 khu vực bị bỏ sót — xác nhận đúng**

Người phụ trách nghiệp vụ chỉ ra thêm khu vực Nhập kho (Stock In) và khu vực Định mức nguyên vật liệu (BOM) chưa được đưa vào danh sách. Đã kiểm tra lại và xác nhận đúng — cả 2 khu vực này thực sự có sử dụng dữ liệu sợi với số lượng giao dịch rất lớn trong thực tế (hàng chục nghìn giao dịch).

**7. Rà soát mở rộng phát hiện quá nhiều kết quả — cần lọc lại cho chính xác**

Vì hệ thống lưu chung mọi loại hàng hoá vào 1 danh mục, việc rà soát mở rộng ban đầu cho ra một danh sách rất dài (hơn 200 màn hình) nhưng có lẫn nhiều màn hình không thực sự liên quan tới sợi — chỉ là các màn hình khác cũng dùng chung danh mục hàng hoá cho loại hàng KHÁC (thành phẩm, hoá chất...).

**8. Người dùng phản biện chính xác — nhiều màn hình bị đưa vào nhầm**

Người phụ trách nghiệp vụ nhận định đúng: các màn hình liên quan tới công đoạn Nhuộm và In không thể bị ảnh hưởng bởi thay đổi mã sợi, vì các công đoạn này làm việc trên mã của THÀNH PHẨM (vải đã nhuộm/in), không dùng trực tiếp mã sợi. Đã kiểm tra kỹ và xác nhận nhận định này hoàn toàn chính xác — các màn hình Nhuộm/In quả thực không hiển thị bất kỳ thông tin sợi nào, chỉ hiển thị mã thành phẩm của riêng chúng.

**9. Kiểm tra lại toàn bộ danh sách bằng dữ liệu thực tế thay vì suy đoán**

Để đảm bảo độ chính xác tuyệt đối, đã kiểm tra lại từng màn hình trong danh sách mở rộng bằng cách đối chiếu trực tiếp với dữ liệu thật trong hệ thống (không chỉ dựa vào tên gọi màn hình hay suy đoán theo khu vực nghiệp vụ), để xác định rõ màn hình nào THỰC SỰ có dữ liệu sợi đi qua và màn hình nào chỉ tình cờ dùng chung 1 bảng danh mục nhưng không liên quan tới sợi.

**10. Kết quả cuối cùng**

Sau khi kiểm tra và loại bỏ các trường hợp không thực sự liên quan, danh sách cuối cùng còn lại 93 màn hình thực sự bị ảnh hưởng khi thay đổi mã sợi (giảm đáng kể so với danh sách sơ bộ ban đầu hơn 200 màn hình).

Phát hiện đáng lưu ý: có 1 màn hình tên là "Yarn Spec Property" (thông số kỹ thuật sợi) — dù tên gọi có chữ "Yarn" giống hệt ticket, nhưng sau khi kiểm tra kỹ, màn hình này thực chất quản lý 1 loại phân loại "sợi" khác hoàn toàn, thuộc 1 khu vực nghiệp vụ riêng biệt (kiểm định/thử nghiệm), không liên quan gì tới mã Yarn Code trong ticket này — nên đã loại khỏi danh sách.

Kết quả cuối cùng được tổng hợp thành 1 tài liệu duy nhất, liệt kê đầy đủ 93 màn hình bị ảnh hưởng kèm đường dẫn menu thật trong ứng dụng, để đội ngũ Samil có thể tra cứu và lên kế hoạch kiểm tra/đồng bộ dữ liệu khi có thay đổi mã sợi trong tương lai.
