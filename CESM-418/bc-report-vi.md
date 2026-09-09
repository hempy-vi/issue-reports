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

**10. BC bổ sung kết luận "không cần sửa hệ thống" — kiểm chứng lại, phát hiện chưa chính xác**

User gửi kèm ảnh chụp màn hình PM0407 (tháng 07/2026, Factory 1) và 1 kết luận BC đã bổ sung
vào ticket, đại ý: đã kiểm tra dữ liệu thực tế tháng 7/2026, tồn kho nguyên liệu WIP hoàn toàn
chỉ có US Cotton (USOT3137), không phát hiện nguyên liệu trộn khác (vd BRAMID) bị mang qua
tháng — do đó không cần sửa hệ thống nữa.

Thay vì chấp nhận luôn, đã kiểm tra lại bằng cách truy vấn trực tiếp đúng phạm vi mà BC đã xem
(cùng nhà máy, cùng loại thành phẩm, cùng 2 tháng 7 và 8/2026). Kết quả cho thấy: nguyên liệu
chưa được duyệt (BRAMID) vẫn đang chiếm khoảng **42% tổng khối lượng** trong đúng phạm vi đó ở
tháng 7, và số lượng này được mang nguyên vẹn sang tháng 8 — hoàn toàn mâu thuẫn với kết luận
BC đưa ra.

Xác định nguyên nhân BC không phát hiện ra: màn hình chỉ hiển thị các lô trộn nguyên liệu MỚI
NHẤT (tạo cuối tháng 7) đã đúng 100% nguyên liệu đã duyệt thật — nhưng các lô trộn CŨ HƠN (tạo
từ cuối tháng 6) vẫn còn nguyên liệu chưa duyệt, nằm ở phần cần kéo/cuộn thêm trong bảng chi
tiết mà có thể BC chưa xem hết. Kết luận: không đồng ý với nhận định "không cần sửa hệ thống",
đã phản hồi lại BC kèm số liệu cụ thể, giữ nguyên khuyến nghị cần sửa.

**11. Phát hiện: có sẵn 1 phiên bản xử lý dùng chung với các nhà máy khác — bản sửa hoá ra đã
được đưa vào hệ thống thật**

User cho biết: ở 1 số nhà máy khác đang dùng chung nền tảng, job tính lot tracking hàng đêm chỉ
đóng vai trò "gọi vào" 1 chương trình xử lý chính riêng biệt (có thể truyền vào tháng và loại
xử lý cần chạy), thay vì tự chứa toàn bộ logic như job của DONGIL hiện tại — và yêu cầu tổ chức
lại cho DONGIL theo đúng mô hình đó.

Trước khi thực hiện, kiểm tra thì phát hiện DONGIL cũng đã có sẵn 1 chương trình xử lý chính
như vậy từ trước (không nhận biết trước đó), có cấu trúc gần như song song hoàn toàn với job
đang dùng của DONGIL. Đồng thời, khi so sánh nội dung 2 chương trình, phát hiện: **bản sửa đã
soạn trước đó (5 điểm sửa) hoá ra đã được đưa vào hệ thống thật rồi** — hỏi lại và được xác
nhận: user đã tự kiểm tra và áp dụng bản sửa lên hệ thống production (có sao lưu bản cũ trước
khi thay thế).

Đã cảnh báo cho user 1 rủi ro: nếu làm đúng y yêu cầu ban đầu (đổi job hàng đêm thành chỉ gọi
vào chương trình xử lý chính riêng), mà chương trình đó CHƯA có bản sửa, thì việc thay đổi này
sẽ VÔ HIỆU HOÁ bản sửa vừa được áp dụng.

**12. Theo yêu cầu, chuyển toàn bộ bản sửa sang chương trình xử lý chính**

User làm rõ hướng đi: chương trình xử lý chính (dùng chung, có thể chạy cho cả kỳ đóng sổ
THÁNG lẫn hàng NGÀY) sẽ là nơi chứa toàn bộ logic và bản sửa; job hàng đêm của DONGIL chỉ còn
đóng vai trò gọi vào chương trình đó, đúng như mô hình các nhà máy khác đang dùng.

Lần đầu tiên đọc kỹ phần xử lý dành cho kỳ đóng sổ THÁNG (trước giờ chưa từng phân tích vì
tưởng không được dùng tới) — phát hiện đây là 1 cơ chế đơn giản hơn phần xử lý hàng ngày,
lấy dữ liệu trực tiếp từ 1 bảng snapshot đã được tính sẵn khi đóng sổ. Đã áp dụng cùng nguyên
tắc sửa (thay mã nguyên liệu chưa duyệt bằng mã đã duyệt, giữ nguyên khối lượng) cho cả 3 chỗ
ghi nhận nguyên liệu trong phần xử lý này.

Soạn bản sửa hoàn chỉnh cho chương trình xử lý chính — tổng cộng **8 điểm sửa** (bao gồm cả
phần xử lý hàng ngày lẫn phần xử lý tháng), đã kiểm tra kỹ bằng công cụ riêng để đảm bảo không
có lỗi cấu trúc trước khi gửi cho user.

Soạn thêm 1 phiên bản mới cho job hàng đêm hiện tại — giờ chỉ còn 4 dòng, đơn giản chỉ gọi vào
chương trình xử lý chính, đúng mô hình các nhà máy khác. Có ghi chú rõ thứ tự bắt buộc khi áp
dụng: phải cập nhật chương trình xử lý chính TRƯỚC, rồi mới cập nhật job hàng đêm SAU — nếu làm
ngược lại, sẽ có 1 khoảng thời gian ngắn job hàng đêm gọi vào bản CHƯA sửa. Lợi ích thêm: từ giờ
có thể chủ động chạy lại cho đúng 1 tháng cụ thể (ví dụ tháng 8) bằng cách truyền thẳng tham số,
không cần chỉnh sửa tạm thời trong code như cách cũ nữa.

**13. Xác nhận cả 2 thay đổi đã được áp dụng thành công lên hệ thống thật**

Kiểm tra lại hệ thống: cả chương trình xử lý chính và job hàng đêm mới đều đã được cập nhật
thành công, đúng thứ tự (chương trình chính trước, job hàng đêm sau, cách nhau 21 giây), không
có lỗi. Xác nhận job hàng đêm giờ đúng là bản gọi vào (4 dòng), và chương trình xử lý chính có
đầy đủ dấu vết của cả 8 điểm đã sửa.

**14. Tổng duyệt lần cuối (3 người kiểm tra độc lập, làm song song)**

Trước khi coi đây là hoàn tất về mặt hệ thống, thực hiện 1 vòng rà soát toàn diện lần cuối trên
chính phiên bản đã đưa vào hệ thống thật, với 3 hướng kiểm tra độc lập:

- **Kiểm tra phần xử lý THÁNG**: xác nhận cả 3 chỗ sửa trong phần này đều đúng kỹ thuật. Tuy
  nhiên phát hiện: phần xử lý THÁNG hiện **không có tác dụng thực tế** cho DONGIL — kiểm tra
  toàn hệ thống xác nhận không có nơi nào (kể cả job hàng đêm) thực sự gọi tới phần xử lý này,
  chỉ phần xử lý HÀNG NGÀY mới thực sự chạy. Có thể do đây là phần dự phòng cho các nhà máy
  khác dùng chung, chứ chưa dùng tới ở DONGIL.
- **Rà soát tính đầy đủ**: quét toàn bộ chương trình, xác nhận đúng **10/10 chỗ** ghi nhận mã
  nguyên liệu vào hệ thống đều đã được chuẩn hoá đúng (7 chỗ sửa trực tiếp, 3 chỗ còn lại tự
  động đúng theo nhờ kế thừa từ các chỗ đã sửa phía trước). Có 1 khuyến nghị cải thiện thêm
  (không bắt buộc ngay): nên thêm 1 lớp kiểm tra dự phòng ở bước tổng hợp tạm thời, để chắc
  chắn hơn nếu sau này có phát sinh đường ghi dữ liệu mới mà quên áp dụng chuẩn hoá.
- **Kiểm tra dữ liệu thực tế**: xác nhận dữ liệu THÁNG 8 **hoàn toàn chưa được sửa lại** — vẫn
  còn hơn 1,000 dòng nguyên liệu chưa duyệt / hơn 500 tấn y nguyên như trước, vì bản sửa chỉ ảnh
  hưởng LẦN CHẠY TIẾP THEO, không tự động sửa lại dữ liệu đã có sẵn. Đồng thời do job hàng đêm
  luôn tự tính theo tháng hệ thống hiện tại, chạy vào bất kỳ ngày nào trước khi sang tháng 10 sẽ
  luôn tính cho THÁNG 9 chứ không tự động đụng tới tháng 8 đã đóng sổ — **bắt buộc phải có 1 lần
  chạy tay riêng cho tháng 8** trước ngày đóng sổ 10/09. Cũng phát hiện: dữ liệu tháng 9 hiện tại
  (tháng đang chạy) vẫn còn nguyên liệu chưa duyệt tồn đọng từ lần chạy đêm qua (trước khi bản
  sửa được đưa vào hệ thống) — theo thiết kế, lần chạy đêm NAY sẽ tự động làm sạch lại, nhưng
  cần kiểm tra lại vào sáng mai để chắc chắn.

Kết luận: bản sửa đã đúng, đầy đủ, và đã có mặt trên hệ thống thật — nhưng còn 2 việc cần làm
gấp trước ngày đóng sổ 10/09: (1) chủ động chạy lại riêng cho tháng 8, (2) kiểm tra lại kết quả
chạy đêm nay có tự làm sạch tháng 9 như mong đợi hay không.

---

**15. Kiểm tra an toàn trước khi chủ động chạy lại tay cho tháng 8**

Trước khi chạy lại tay cho tháng 8, kiểm tra xem tháng 8 đã trải qua đóng sổ tháng chưa và liệu chế độ "đóng sổ THÁNG" của job có an toàn để dùng không. Phát hiện tháng 8 đã có sẵn 1 bản snapshot đóng sổ, nhưng đối chiếu lại thì thấy điều bất thường: bản snapshot đó chỉ ghi nhận 1 tập lô sản xuất nhỏ hơn nhiều — và khác hẳn — so với dữ liệu tracking đang sống; những lô đã đúng nguyên liệu duyệt trong snapshot thì đúng nhãn, còn những lô đang mang nguyên liệu chưa duyệt thì hoàn toàn không xuất hiện trong snapshot dưới bất kỳ nhãn nào.

Thực hiện 1 vòng điều tra tương đương 4 người trước khi cho phép dùng chế độ "đóng sổ THÁNG". Kết quả: bản snapshot đó do 1 quy trình đóng sổ hoàn toàn khác, không liên quan gì tới job đang sửa, sinh ra — và nó chỉ ghi nhận khoảng 1/3 số lô sản xuất của tháng 8; 2/3 còn lại bị bỏ sót bao gồm TOÀN BỘ các lô đang mang nguyên liệu chưa duyệt. Mô phỏng thử chế độ "đóng sổ THÁNG" xác nhận nó sẽ không báo lỗi, nhưng sẽ âm thầm làm biến mất hoàn toàn khoảng 500 tấn dữ liệu tồn kho thật (chứ không phải chỉ đổi nhãn), vì nó sẽ xoá và dựng lại dữ liệu tháng 8 từ đúng bản snapshot thiếu đó. Cũng xác nhận trạng thái duyệt đóng sổ của tháng 8 sẽ không bị ảnh hưởng dù chạy hay không, và chạy lại không tạo ra bản ghi lịch sử đóng sổ trùng lặp.

**Kết luận: không được dùng chế độ "đóng sổ THÁNG" cho tháng 8.** Đã chuẩn bị sẵn 1 phương án dự phòng rủi ro thấp hơn (sửa trực tiếp các dòng nguyên liệu chưa duyệt hiện có, chỉ đổi nhãn, không đổi số lượng) làm phương án thay thế. Sau đó user xác nhận thêm bối cảnh nghiệp vụ quan trọng: dữ liệu lot tracking là 1 module độc lập, an toàn để xoá sạch và dựng lại toàn bộ, và bất kỳ tháng nào đã qua đóng sổ đều bắt buộc phải có dữ liệu lot tracking tháng chạy được. Dựa trên đó, khuyến nghị cuối cùng là dùng chế độ xử lý thông thường (cơ chế hàng ngày) cho cả tháng 8 lẫn tháng 9 thay vì chế độ snapshot tháng — chế độ thông thường là chế độ đã được kiểm chứng đầy đủ và tính từ dữ liệu nguồn thật, đầy đủ cho toàn bộ các lô sản xuất của tháng 8, không phụ thuộc snapshot thiếu.

**16. User chạy lại cho tháng 8 rồi tháng 9 — phát hiện lỗi mới: mất dữ liệu thật**

Sau khi chạy lại, nguyên liệu chưa duyệt đã đúng là biến mất khỏi sổ sách cho cả tháng 8 lẫn tháng 9 — đạt đúng mục tiêu chính của ticket. Tuy nhiên, so sánh khối lượng trước/sau phát hiện 1 vấn đề mới: số liệu "nguyên liệu chuyển vào kho pha trộn" của tháng 8 giảm khoảng 97.5% — từ khoảng 1,071 tấn xuống chỉ còn 27 tấn — trong khi số liệu tồn đầu kỳ vẫn đúng nguyên vẹn. Xác nhận các giao dịch chuyển kho thật trong hồ sơ nguồn vẫn còn đầy đủ (khoảng 1,085 tấn) — không hề mất ở nguồn, nên lỗi phải nằm ở logic ghi nhận.

Xác nhận nguyên nhân gốc bằng 1 ví dụ cụ thể: 1 bước kiểm tra chống trùng lặp trong job (dùng để tránh ghi trùng cùng 1 giao dịch chuyển kho 2 lần) không tính đến việc đang kiểm tra cho tháng nào — nó chỉ xem có bản ghi khớp tồn tại ở BẤT KỲ đâu, không phân biệt tháng. Vì tháng 9 đã chạy đêm liên tục từ trước và đã mang theo tham chiếu tới cùng 1 chứng từ chuyển kho gốc vào số dư đầu kỳ của chính nó, nên khi chạy lại cho tháng 8, job thấy tham chiếu đó và tưởng nhầm "đã ghi rồi" nên bỏ qua việc ghi số liệu chuyển kho thật của tháng 8. Đây là **lỗi có sẵn từ trước, không phải do bản sửa này gây ra** — vô hại suốt nhiều năm vì job luôn chạy tuần tự đúng thứ tự thời gian, và đây là lần đầu tiên 1 tháng đã qua được chạy lại trong khi tháng sau đã có dữ liệu. Không có gì thực sự mất — số liệu gốc đúng vẫn còn nằm trong hệ thống (chỉ bị đánh dấu ngừng hoạt động, không bị xoá thật) và có thể khôi phục được.

**17. Rà soát toàn bộ job tìm mọi chỗ dính cùng dạng lỗi**

Kiểm tra toàn bộ job từ đầu đến cuối để tìm mọi chỗ có kiểu kiểm tra chống trùng lặp tương tự. Tìm được đúng 1 chỗ khác dính cùng lỗi (liên quan tới 1 bản ghi sản xuất thành phẩm khác, chưa từng thực sự gây mất dữ liệu trên thực tế nhưng mang cùng rủi ro tiềm ẩn) — đã bổ sung vào cùng bản sửa. Xác nhận mọi chỗ kiểm tra tương tự khác trong job đều an toàn (đều so khớp theo đúng ngày cụ thể, không chỉ theo tháng, nên không dính lỗi này), và xác nhận chế độ "đóng sổ THÁNG" hoàn toàn không có loại kiểm tra này nên không bị ảnh hưởng bởi vấn đề này.

**18. Thiết kế và kiểm chứng bản sửa cho cả 2 chỗ kiểm tra bị lỗi**

Thiết kế bản sửa tối thiểu: bổ sung thêm điều kiện lọc theo đúng tháng (và với 1 trong 2 chỗ, thêm cả loại bản ghi cụ thể) vào mỗi bước kiểm tra chống trùng, để chỉ xét những bản ghi của đúng tháng đang xử lý thay vì bất kỳ tháng nào. Kiểm chứng an toàn cho cả 2 chiều: chạy lại nhiều lần cho cùng 1 tháng vẫn hoàn toàn an toàn (bước xoá-và-dựng-lại đã xoá sạch dữ liệu của tháng đó trước rồi), và chạy lại cho 1 tháng đã qua trong khi tháng sau đã có dữ liệu giờ được cho phép đúng thay vì bị chặn nhầm. Đã đối chiếu lại độc lập nội dung chính xác của cả 2 chỗ kiểm tra bị lỗi trực tiếp trên hệ thống thật (không dựa vào ghi chú trước đó) trước khi áp dụng bản sửa, để loại trừ khả năng có sai lệch. Kiểm tra cấu trúc xác nhận bản đã sửa giống hệt bản gốc ở mọi nơi khác, chỉ khác đúng 2 chỗ đã chủ đích thêm vào.

**19. Điều tra xem dữ liệu theo dõi truy xuất nguồn gốc cho hàng giao khách hàng có cần nằm trong phạm vi khôi phục không**

Trước khi chốt phương án khôi phục, kiểm tra xem dữ liệu theo dõi riêng đứng sau các màn hình giao hàng/truy xuất nguồn gốc cho khách hàng có cần được xoá và dựng lại cùng lúc không. Phát hiện dữ liệu này do 2 quy trình hoàn toàn khác (không phải job đang sửa) sinh ra, các quy trình này lấy số liệu thành phần nguyên liệu trực tiếp từ đúng những bản ghi đang bị lỗi nguyên liệu chưa duyệt — nên có liên quan tới vấn đề này, nhưng việc dựng lại dữ liệu đó chỉ thực sự xảy ra khi có người mở đúng màn hình theo dõi cho đúng khoảng ngày cần xem; job hàng ngày chỉ dọn dẹp 1 phần dữ liệu đó (bỏ qua mọi bản ghi đã từng gửi cho khách qua email/thông báo, vì không lọc theo tháng). Cũng phát hiện thêm 1 lỗi riêng, không liên quan: 1 trường ngày giao hàng trên dữ liệu đó luôn bị để trống do lỗi code trong quy trình dựng dữ liệu đó.

Kiểm tra số liệu thật cho đúng khoảng ngày liên quan và phát hiện: do hệ quả phụ của 2 lần chạy lại ở bước 16, toàn bộ bản ghi theo dõi giao hàng liên quan trong khoảng đó đã tự động bị dọn sạch sẵn rồi — nên lần này không cần thao tác xoá tay riêng. Ghi chú đây chỉ là may mắn cho đúng tình huống này (chưa có bản ghi nào đã gửi khách trong khoảng đó) chứ không nên coi là cơ chế đáng tin cậy cho những lần sau.

**20. User quyết định hướng khôi phục: xoá sạch dữ liệu tháng 8+9 rồi chạy lại**

User quyết định 1 phương án khôi phục triệt để hơn: xoá sạch toàn bộ dữ liệu tháng 8 và tháng 9 trên toàn module (thay vì chỉ dựa vào cờ "ngừng hoạt động" còn sót lại từ lần chạy sự cố trước đó), rồi chạy lại job cho cả 2 tháng. Rà soát mọi bảng có tên gợi ý thuộc module tracking này và xác định chính xác những bảng nào thực sự được job này ghi và có phân theo tháng (đúng 3 bảng, tổng cộng khoảng 105,000 dòng cho cả 2 tháng, tính cả các bản sao lịch sử đã ngừng hoạt động) — dữ liệu theo dõi giao hàng khách hàng (đã xử lý ở bước 19) và 1 bảng tạm nội bộ thuần tuý (tự dựng lại hoàn toàn mỗi lần chạy, không lưu gì theo tháng) không cần nằm trong phạm vi, cùng với vài bảng tên tương tự nhưng không liên quan chức năng. Chuẩn bị sẵn 1 script chạy tay gồm đúng trình tự xoá-và-dựng-lại cần thiết, kèm bước kiểm tra đối chiếu trước/sau, có khuyến nghị mạnh nên compile bản sửa lỗi kiểm tra chống trùng ở bước 18 TRƯỚC, để lỗi gây mất dữ liệu hôm nay không thể tái diễn.

**21. User áp dụng bản sửa và chạy khôi phục — kiểm chứng kết quả**

Kiểm chứng: số liệu "nguyên liệu chuyển vào kho pha trộn" của tháng 8 giờ đã trở lại đúng khoảng 1,085 tấn — khớp chính xác với hồ sơ nguồn thật — xác nhận bản sửa hoạt động đúng, nguyên nhân gốc đã được giải quyết. Số liệu tồn đầu kỳ vẫn đúng, không đổi. Toàn bộ bản ghi nguyên liệu bông thô của cả 2 tháng giờ chỉ còn nguyên liệu đã duyệt — đạt đúng yêu cầu chính của ticket.

Phát hiện thêm 1 điểm cần làm rõ: số liệu "nguyên liệu đã phân bổ cho lô thành phẩm" của tháng 8 thấp hơn khoảng 313 tấn so với con số tồn tại trước khi sự cố hôm nay bắt đầu. Điểm mấu chốt: đúng con số thấp hơn này đã xuất hiện y hệt cả ở lần chạy hỏng trước đó lẫn lần chạy đã sửa hôm nay — chứng minh nó không liên quan tới lỗi vừa sửa (nếu liên quan, con số phải phục hồi tăng lên cùng lúc số liệu chuyển kho được khôi phục, nhưng nó không đổi chút nào). Điều tra thêm phát hiện con số này trùng khớp về thời điểm với lúc tháng này được duyệt đóng sổ chính thức, và truy vết được về mặt cấu trúc rằng việc phân bổ nguyên liệu chỉ có thể lấy từ đúng lượng tồn gắn với lô sản xuất cụ thể đó, không được lấy tự do từ tổng tồn cả tháng — nghĩa là vẫn có thể xảy ra thiếu hụt cục bộ dù tổng thể dư dả. Kiểu này khớp với 1 vấn đề có sẵn từ trước đã được báo cáo riêng trong chính quá trình điều tra này (số dư mang qua tháng không bao giờ được trừ theo tiêu thụ thực) chứ không phải điều gì mới phát sinh hôm nay. Không ảnh hưởng tới yêu cầu chính của ticket và không nên chặn việc đóng sổ 10/09, nhưng đáng để báo riêng cho phía nghiệp vụ.

**22. Phát hiện 1 hệ quả thật trên màn hình theo dõi xuất xứ nguyên liệu cho khách hàng — do chính bản sửa này gây ra**

Phía nghiệp vụ báo qua chat nội bộ: 1 màn hình dùng để xem và chứng nhận xuất xứ (nước sản xuất) của bông nguyên liệu trong các lô hàng giao khách hàng đang hiện nhiều dòng không có xuất xứ và không có file chứng từ đính kèm, cho các lô giao trong tháng 8.

Điều tra và xác nhận: màn hình này tra cứu xuất xứ và file chứng từ bằng cách, trong số các điều kiện khác, so khớp mã nguyên liệu ghi nhận lúc mua hàng với mã nguyên liệu hiện đang có trên bản ghi sản xuất. Vì bản sửa này đổi nhãn mã nguyên liệu trên bản ghi sản xuất (từ mã chưa duyệt sang mã đã duyệt), việc so khớp đó bị đứt đúng cho những lô mà bản sửa đã chạm tới — bản ghi mua hàng vẫn (đúng) giữ mã gốc, còn bản ghi sản xuất giờ đã mang mã mới, nên tra cứu không tìm thấy gì.

Kiểm chứng bằng dữ liệu thật: mọi dòng bị ảnh hưởng trong ảnh chụp màn hình mà phía nghiệp vụ gửi đều thực sự được mua dưới mã nguyên liệu chưa duyệt ban đầu — xác nhận chắc chắn đây là hệ quả trực tiếp của bản sửa này, không phải 1 lỗi khác. Màn hình này tồn tại cụ thể để phục vụ chứng nhận xuất xứ cho mục đích hải quan/xuất khẩu, nơi xuất xứ hiển thị phải phản ánh đúng xuất xứ vật lý thật của nguyên liệu, không phải nhãn kế toán nội bộ dùng cho việc phân bổ sổ sách. Đã đưa ra 2 phương án cho user: (A) chỉnh lại cách tra cứu xuất xứ/chứng từ để chỉ so khớp theo số hiệu lô sản xuất, nhờ đó vẫn tìm đúng bản ghi mua hàng gốc và xuất xứ thật bất kể mã nguyên liệu đã bị đổi nhãn cho mục đích kế toán; (B) giữ nguyên hiện trạng màn hình. **User chọn phương án (A).**

Trước khi thực hiện thay đổi, đã kiểm chứng đây là hoàn toàn an toàn: xác nhận không có số hiệu lô nào trong hồ sơ mua hàng từng gắn với hơn 1 mã nguyên liệu, và không có số hiệu lô nào gắn với hơn 1 chứng từ mua hàng/xuất xứ — nghĩa là cách tra cứu mới không thể tạo ra dòng trùng lặp hay sai lệch. Đã soạn sẵn phiên bản đã sửa cho logic đứng sau màn hình này.

**23. Rà soát các màn hình liên quan — phát hiện thêm 5 chỗ khác dính đúng vấn đề này**

Theo yêu cầu của user, kiểm tra thêm popup xem file chứng từ mở ra từ chính màn hình này, và phát hiện popup đó cũng dính đúng lỗi tương tự. Từ đó, rà soát toàn hệ thống mọi nơi dùng chung tổ hợp dữ liệu (bản ghi mua hàng gốc và bản ghi theo dõi sản xuất) và tìm được tổng cộng 6 nơi dính lỗi này: màn hình chính đã sửa ở bước 22, popup xem file, 1 màn hình theo dõi liên quan khác, 1 phiên bản cũ hơn của cùng màn hình theo dõi, 1 biến thể thứ 2 của popup file, và — đáng chú ý nhất — tính năng gửi báo cáo truy xuất nguồn gốc nguyên liệu trực tiếp cho khách hàng qua email. Đã áp dụng đúng 1 cách sửa giống hệt (chỉ so khớp theo số hiệu lô) cho cả 5 chỗ còn lại, theo đúng bước kiểm chứng an toàn đã xác nhận ở bước 22. Tính năng gửi email cho khách hàng đáng được lưu ý riêng: nếu không sửa, báo cáo truy xuất nguồn gốc gửi cho khách sẽ thiếu hoặc sai thông tin xuất xứ cho đúng những lô mà bản sửa này đã đổi nhãn.

---

**Các điểm còn cần user/phía DONGIL xác nhận (tính tới hết 2026-09-08):**
1. Số hiệu lô hàng (lot number) gốc có được giữ nguyên khi đổi mã nguyên liệu hay không.
2. Số lượng nguyên liệu chưa duyệt còn tồn (sẽ không bao giờ bị trừ nữa trên sổ sách của job
   này) có ảnh hưởng gì tới cách DONGIL theo dõi tồn kho/giá vốn theo từng loại nguyên liệu.
3. Xác nhận phạm vi áp dụng vĩnh viễn cho toàn bộ nhóm nguyên liệu bông thô, không giới hạn
   thời gian, có đúng ý muốn lâu dài hay không.
4. Có muốn báo cáo riêng lỗi phụ phát hiện được ở bước 5 (số dư không được trừ theo tiêu thụ)
   thành 1 yêu cầu/ticket khác hay không.
5. Phần xử lý dành cho kỳ đóng sổ THÁNG hiện không có tác dụng cho DONGIL (không nơi nào gọi tới,
   và bản thân nguồn dữ liệu snapshot của nó cũng thiếu — xem bước 15) — có cần thiết phải duy
   trì bản sửa ở phần này không, hay chỉ để dự phòng cho nhà máy khác dùng chung?
6. Có muốn bổ sung thêm 1 lớp kiểm tra dự phòng ở bước tổng hợp tạm thời (khuyến nghị từ bước 14,
   không bắt buộc) hay không?
7. **[MỚI]** Khoảng chênh lệch khoảng 313 tấn ở số liệu phân bổ cho lô thành phẩm tháng 8 (bước
   21, nghi liên quan tới vấn đề số dư mang qua tháng đã biết từ trước) — có cần điều tra sâu
   thêm để định lượng chính xác tác động, hay chấp nhận đây là hệ quả của vấn đề đã biết và xử lý
   chung khi báo cáo vấn đề đó?
8. **[MỚI]** Sau 6 bản sửa ở bước 23 — có cần rà soát thêm các màn hình/tính năng khác (hiện chưa
   kiểm tra) có thể cũng bị ảnh hưởng bởi việc đổi nhãn mã nguyên liệu không, hay 6 chỗ này đã là
   toàn bộ phạm vi?

---

## 2026-09-09

**1. Kiểm chứng độc lập lời khẳng định của phía kinh doanh: "từ tháng 8 dùng 100% nguyên liệu US"**

Phía kinh doanh phản hồi qua chat nội bộ, khẳng định từ tháng 8 công ty chỉ dùng 100% nguyên liệu US (dẫn chứng 1 lô cụ thể). Được yêu cầu kiểm chứng độc lập, không dựa vào đúng những bảng dữ liệu đang nghi có lỗi mà đi thẳng vào chứng từ chuyển kho thật (giao dịch xuất/nhập kho pha trộn, thuộc quy trình sản xuất, hoàn toàn tách biệt với dữ liệu lot tracking).

Kiểm tra chứng từ chuyển kho thật cho 3 lô mix mẫu đã tạo cuối tháng 6 và tháng 7 — xác nhận cả nguyên liệu chưa duyệt lẫn nguyên liệu đã duyệt đều thực sự được chuyển vào kho pha trộn đúng ngày tạo lô đó, có chứng từ đầy đủ. Mở rộng kiểm tra cho toàn bộ tháng 6-7/2026: xác nhận **165 lô mix dùng nguyên liệu đã duyệt (khoảng 1,100 tấn) và 138 lô mix dùng nguyên liệu chưa duyệt (khoảng 1,091 tấn)** — cả 2 loại đều có chứng từ chuyển kho thật, không phải số liệu ảo hay lỗi hệ thống. => Lời khẳng định "100% US từ tháng 8" của phía kinh doanh **không đúng xét theo dữ liệu lịch sử thực tế** đã ghi nhận.

**2. Chuẩn bị (chưa thực thi) phương án xoá lại dữ liệu theo dõi giao hàng cho tháng 8+9**

Theo yêu cầu, soạn sẵn 1 phương án xoá sạch và dựng lại dữ liệu theo dõi giao hàng/truy xuất nguồn gốc (đã đề cập ở bước 19-20 ngày 08/09) đúng cho phạm vi tháng 8 và tháng 9/2026 — đã xác nhận lại phạm vi này (không phải toàn bộ lịch sử) trước khi soạn.

**3. Xác nhận lại: dữ liệu phân bổ nguyên liệu đã đúng 100% nguyên liệu đã duyệt sau lần sửa lỗi ngày 08/09**

Kiểm tra lại toàn bộ dữ liệu tháng 8 và tháng 9 sau khi đã sửa lỗi mất dữ liệu (bước 18-21 ngày 08/09): xác nhận **không còn bất kỳ dòng dữ liệu nào** ghi nhận nguyên liệu chưa duyệt cho 2 tháng này — hoàn toàn 100% nguyên liệu đã duyệt, đúng như mục tiêu ban đầu của yêu cầu này.

**4. Chạy phương án xoá ở bước 2 — phát hiện chỉ khôi phục được 1 phần dữ liệu giao hàng**

Sau khi xoá, số liệu giao hàng chỉ khôi phục lại được khoảng 1/4 so với trước (khoảng 400/1,400 và 600/16,000 dòng tương ứng ở 2 tầng dữ liệu) — không đầy đủ như dự kiến ban đầu. Kiểm tra lại kỹ hơn: hoá ra cơ chế "tự dựng lại khi mở màn hình" chỉ dựng đúng phần dữ liệu tương ứng với khoảng ngày mà người dùng đã search qua giao diện, KHÔNG tự động phủ hết toàn bộ 2 tháng như đã hiểu nhầm ban đầu. Đã chuẩn bị 1 lệnh gọi trực tiếp để ép dựng lại toàn bộ 2 tháng trong 1 lần, dựa trên đúng dữ liệu đã sửa lỗi (an toàn, không tạo trùng lặp nếu chạy nhiều lần).

**5. Phía kinh doanh gửi ảnh chụp màn hình theo dõi giao hàng — thấy nhiều dòng thiếu xuất xứ và thiếu file chứng từ**

Điều tra sâu nguyên nhân: cách màn hình này tra cứu xuất xứ/chứng từ có 1 điều kiện phụ giới hạn theo đúng danh sách các lô đang hiển thị trên màn hình tại thời điểm search — điều kiện phụ này đang loại nhầm 1 số dòng ra, dù dữ liệu chứng từ gốc phía sau (hồ sơ mua hàng, xuất xứ, file đính kèm) **hoàn toàn còn nguyên vẹn và đầy đủ**. Đã kiểm chứng trực tiếp: lấy đúng 1 lô hàng đang bị hiện "không có xuất xứ/không có file" trên màn hình, tra ngược lại hồ sơ mua hàng gốc — vẫn tìm thấy đầy đủ hồ sơ, đúng xuất xứ Brazil, và đủ 7 file chứng từ đính kèm. => **Xác nhận đây là 1 lỗi hiển thị riêng của màn hình (do điều kiện lọc phụ), KHÔNG PHẢI mất dữ liệu** do bước xoá/dựng lại ở bước 2-4 gây ra.

**6. Phía kinh doanh phản hồi bằng bằng chứng tồn kho vật lý thật — khẳng định nguyên liệu chưa duyệt đã hết sạch từ 22/7**

Phía kinh doanh dẫn chứng bằng 1 màn hình kiểm tra tồn kho vật lý (không liên quan gì tới dữ liệu lot tracking đang điều tra), khẳng định nguyên liệu chưa duyệt đã hết sạch trong kho từ ngày 22/7/2026. Kiểm chứng độc lập bằng đúng nguồn dữ liệu kho vật lý thật (sổ cân đối tồn kho + giao dịch nhập/xuất kho thật) — số liệu tính lại khớp CHÍNH XÁC với ảnh chụp màn hình đã gửi. Xác nhận: tồn nguyên liệu chưa duyệt tại kho nguyên liệu chính, tính đến 22/7/2026, đúng là **bằng 0**, và không có bất kỳ giao dịch nhập/xuất nào sau đó. Kiểm tra thêm kho nguyên liệu cotton còn lại duy nhất khác — cũng cho kết quả tương tự (gần như bằng 0, không đủ dùng). => **Phía kinh doanh hoàn toàn đúng về hiện trạng tồn kho vật lý.**

**7. Xác định chính xác ngày cuối cùng thực sự còn dùng nguyên liệu chưa duyệt để pha trộn**

Đối chiếu lại với đúng chứng từ chuyển kho thật (đã dùng ở bước 1): xác nhận lô pha trộn cuối cùng thực sự dùng nguyên liệu chưa duyệt là ngày **21/7/2026** — không có lô nào sau ngày đó còn dùng loại nguyên liệu này, khớp hoàn toàn với việc kho báo hết sạch từ 22/7. => **Chốt được mốc chính xác: từ 22/7/2026 trở đi, mọi lô pha trộn mới đều chắc chắn 100% nguyên liệu đã duyệt thật sự; các lô pha trộn từ 21/7/2026 trở về trước thì có dùng nguyên liệu chưa duyệt thật — đây là sự thật đã xảy ra, không thể thay đổi ngược lại được.**

**8. Tổng hợp giải thích cho phía kinh doanh: phân biệt "lúc pha trộn" và "lúc giao hàng"**

Giải thích rõ cho phía kinh doanh: sở dĩ dữ liệu tháng 8 vẫn còn ghi nhận nguyên liệu chưa duyệt là vì có 2 mốc thời gian khác nhau trong cùng 1 quy trình — lúc PHA TRỘN nguyên liệu (xảy ra 1 lần, đã dừng hẳn từ 21/7) khác với lúc GIAO HÀNG thành phẩm cho khách (kéo dài nhiều tuần sau đó, có thể tới tháng 8-9). Lô hàng giao trong tháng 8, nếu được sản xuất từ mẻ đã pha trộn nguyên liệu chưa duyệt từ trước 21/7, thì đúng là còn dùng nguyên liệu đó thật — không phải pha trộn mới trong tháng 8, cũng không phải lỗi dữ liệu.

Đồng thời làm rõ thêm 1 điểm quan trọng cho quyết định tiếp theo: mã nguyên liệu hiện đang hiển thị là "đã duyệt" cho các lô này (sau khi được chuẩn hoá theo yêu cầu ban đầu) **không phản ánh đúng thực tế nguyên liệu đã dùng** — hồ sơ mua hàng gốc của các lô này (đã kiểm chứng ở bước 5) vẫn đúng là hồ sơ nguyên liệu chưa duyệt thật, có đầy đủ chứng từ. Nếu muốn dữ liệu phản ánh đúng 100% thực tế, các lô pha trộn trước 22/7 cần hiển thị lại đúng là nguyên liệu chưa duyệt (cả mã nguyên liệu lẫn xuất xứ); còn các lô pha trộn từ 22/7 trở đi thì chắc chắn đã là nguyên liệu đã duyệt thật, không cần chỉnh sửa gì thêm.

---

**Các điểm còn cần user/phía DONGIL xác nhận (bổ sung 2026-09-09):**
9. **[MỚI]** Có nên ép hiển thị xuất xứ là "đã duyệt" cho những lô ĐÃ XÁC NHẬN có hồ sơ mua hàng thật ghi nhận là nguyên liệu chưa duyệt (kèm đầy đủ chứng từ) hay không — đây là quyết định liên quan tới chứng nhận xuất xứ/tuân thủ, không đơn thuần là chọn cách hiển thị. Tính năng gửi báo cáo truy xuất nguồn gốc qua email cho khách hàng vẫn đang tạm giữ, chưa áp dụng thay đổi này, chờ đúng quyết định này.
10. **[MỚI]** Có nên sửa lại để những lô pha trộn TRƯỚC ngày 22/7/2026 hiển thị đúng lại là nguyên liệu chưa duyệt (thay vì nhãn đã chuẩn hoá) cho khớp đúng thực tế đã sản xuất — hay giữ nguyên cách chuẩn hoá 100% như yêu cầu ban đầu (chấp nhận đây là nhãn sổ sách nội bộ, không phải xuất xứ vật lý thật của lô hàng)?
11. **[MỚI]** Lỗi hiển thị riêng ở màn hình theo dõi giao hàng (bước 5 — làm ẩn xuất xứ/chứng từ dù dữ liệu gốc vẫn còn nguyên) — có cần điều tra thêm để sửa dứt điểm hay không (độc lập với 2 quyết định nghiệp vụ ở mục 9/10)?
12. **[MỚI]** Phạm vi xoá dữ liệu giao hàng tháng 8+9 (bước 2) và lệnh dựng lại toàn bộ (bước 4) đã chuẩn bị sẵn — đang chờ được thực thi.

**9. Nhiều khách hàng báo màn hình theo dõi giao hàng hiện trống — xác nhận cùng 1 nguyên nhân đã biết, không phải sự cố mới**

User gửi liên tiếp báo cáo "không lên dữ liệu" cho 4 khách hàng khác nhau trên màn hình theo dõi giao hàng. Kiểm tra trực tiếp xác nhận: đây đúng là hệ quả của việc dựng lại dữ liệu giao hàng trước đó (bước 4, phần trước) mới chỉ phủ được 1 phần, chưa đầy đủ toàn bộ — không phải lỗi riêng của từng khách hàng. Đã xác nhận: chạy đủ lệnh dựng lại toàn bộ đã chuẩn bị sẵn sẽ khắc phục cho TẤT CẢ khách hàng cùng lúc.

**10. Gặp lỗi kỹ thuật khi user áp dụng bản sửa — tìm và khắc phục ngay**

Khi user đưa bản sửa (đã chuẩn bị ở bước trước) vào hệ thống, gặp lỗi kỹ thuật khiến không thể lưu được. Xác định nguyên nhân: trong quá trình soạn bản sửa, có 1 đoạn ghi chú kỹ thuật bị lồng vào nhau sai cách khiến hệ thống hiểu nhầm ranh giới của phần code đang tắt/bật. Đã sửa ngay và xác nhận lại toàn bộ cấu trúc file cân bằng đúng trước khi gửi lại cho user áp dụng lần 2.

**11. User báo lại: sau khi áp dụng bản sửa, kết quả "100% đã duyệt" mà dev lead từng tạo ra không còn nữa — xác nhận đây là kết quả ĐÚNG, không phải bị lùi tiến độ**

Sau khi áp dụng xong bản sửa, user kiểm tra lại và thấy kết quả không còn hiện "100% đã duyệt" như trước, mà quay lại hiện tỷ lệ pha trộn thật (khoảng 38% đã duyệt / 62% chưa duyệt cho nhóm dữ liệu đang xem).

Giải thích lại rõ cho user: con số "100% đã duyệt" trước đó **là giả** — do chính bản sửa lỗi của dev lead (đã phát hiện và revert ở bước 4) làm mất khoảng 90% dữ liệu tồn kho mang qua tháng thật, chỉ còn sót lại đúng phần vốn dĩ đã "sạch" từ trước, khiến nhìn có vẻ đã sửa xong nhưng thực chất là do thiếu dữ liệu. Sau khi khôi phục đúng cơ chế mang số dư qua tháng, dữ liệu thật — bao gồm cả phần hàng tồn có dùng nguyên liệu chưa duyệt thật từ trước — quay lại đúng như thực tế. Đây là kết quả ĐÚNG.

**12. Dev lead nghi ngờ có 1 lô nguyên liệu "không nên còn xuất hiện" trong dữ liệu tháng 8 — điều tra, xác nhận có 1 vấn đề kỹ thuật cũ THẬT nhưng bản chất khác hẳn**

User chuyển lời dev lead: nghi ngờ 1 lô pha trộn cụ thể (từ cuối tháng 6) không nên còn xuất hiện trong dữ liệu phân bổ của tháng 8, và cho rằng "dữ liệu đầu kỳ đang sai". Kiểm tra kỹ toàn bộ lịch sử nhập/xuất của đúng lô đó qua các tháng: phát hiện số lượng tồn của lô này thực sự **không hề giảm** suốt 3 tháng liên tiếp (tháng 7, 8, 9) — luôn báo "đã dùng hết rồi mang sang y nguyên", dù thực tế phải giảm dần khi được sử dụng.

→ Xác nhận: đây **đúng là 1 vấn đề kỹ thuật có sẵn từ trước** (đã được ghi nhận ngay từ những bước đầu tiên của ticket này) — cách hệ thống tính số dư mang qua tháng đang không trừ đúng theo phần đã thực sự sử dụng, khiến con số bị lặp lại thay vì giảm dần. Vấn đề này hoàn toàn KHÔNG liên quan tới lỗi mới của dev lead vừa được khắc phục ở bước trước.

**Nhưng đã làm rõ với user**: vấn đề này chỉ làm sai **SỐ LƯỢNG** tồn kho hiển thị (có thể đang bị đếm lặp), **KHÔNG làm sai LOẠI nguyên liệu** — bản chất lô hàng đó (trộn cuối tháng 6, có dùng nguyên liệu chưa duyệt thật) không hề thay đổi dù số lượng tồn có tính sai. Hướng sửa đúng (nếu cần) là sửa lại cách tính số lượng tiêu thụ, **hoàn toàn không phải** đổi nhãn xuất xứ nguyên liệu thành "đã duyệt" — đã từ chối yêu cầu này vì sẽ là ghi đè sai lên sự thật đã được xác minh độc lập nhiều lần trong ngày qua nhiều nguồn khác nhau.

**13. User tiếp tục cho rằng "hệ thống lấy nhầm lô nguyên liệu" — kiểm tra toàn bộ nguồn gốc của lô thành phẩm liên quan, bác bỏ dứt điểm**

User đưa ra giả thuyết: có thể tồn tại 1 lô pha trộn ĐÚNG từ tháng 8 (100% đã duyệt) mà hệ thống lẽ ra phải dùng, nhưng lại lấy nhầm sang lô từ tháng 6. Kiểm tra TOÀN BỘ (không chỉ 1 trường hợp lẻ) nguồn nguyên liệu đã cấu thành nên đúng lô thành phẩm đang tranh luận: xác nhận lô này được sản xuất từ tổng cộng 74 lô pha trộn khác nhau — 9 lô từ tháng 6 và 65 lô từ tháng 7, tổng hơn 520 tấn — **không có bất kỳ lô pha trộn nào từ tháng 8 cả**.

→ Đây là 1 lô thành phẩm rất lớn, được sản xuất bằng cách rút dần từ rất nhiều lô nguyên liệu đã có sẵn trong kho từ tháng 6-7 — không hề tồn tại lô tháng 8 nào để mà "lấy nhầm". Đã bác bỏ dứt điểm giả thuyết này bằng chính dữ liệu đầy đủ, không phải suy đoán hay tranh cãi cảm tính.

---

**Cập nhật các điểm còn cần user/phía DONGIL xác nhận (tính tới hết ngày làm việc 2026-09-09):**

Mục 9 và mục 10 ở phần đầu ngày (liên quan tới việc có ép hiển thị "đã duyệt" hay không) **vẫn còn mở, chưa có quyết định cuối** — riêng mục này giờ có thêm 1 yếu tố quan trọng: vấn đề kỹ thuật cũ vừa phát hiện lại (mục 14 dưới đây) cho thấy SỐ LƯỢNG tồn kho mang qua tháng hiện tại có thể chưa hoàn toàn đáng tin cậy, dù LOẠI nguyên liệu của từng lô vẫn được xác định đúng.

14. **[MỚI]** Vấn đề kỹ thuật cũ về cách tính số dư mang qua tháng (không trừ đúng theo tiêu thụ thực) vừa được xác nhận LẠI bằng dữ liệu thật hôm nay — nên được ưu tiên báo cáo/xử lý thành 1 yêu cầu riêng, vì hiện đang bị dùng làm căn cứ (không chính xác) để nghi ngờ ngược lại tính đúng đắn của bản sửa chính đang làm.
15. **[MỚI]** Cần thống nhất lại với dev lead/phía nghiệp vụ: đề xuất "ép thành phẩm tháng 8 hiện 100% đã duyệt" không có dữ liệu nào ủng hộ (đã bác bỏ ở bước 12-13) — cần đồng thuận hướng xử lý đúng (sửa vấn đề tính SỐ LƯỢNG, không đụng vào NHÃN nguyên liệu) trước khi có bất kỳ thay đổi nào tiếp theo lên hệ thống.
