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

**Các điểm còn cần user/phía DONGIL xác nhận:**
1. Số hiệu lô hàng (lot number) gốc có được giữ nguyên khi đổi mã nguyên liệu hay không.
2. Số lượng nguyên liệu chưa duyệt còn tồn (sẽ không bao giờ bị trừ nữa trên sổ sách của job
   này) có ảnh hưởng gì tới cách DONGIL theo dõi tồn kho/giá vốn theo từng loại nguyên liệu.
3. **[GẤP, chưa thực hiện]** Cần chủ động chạy lại riêng cho tháng 8 trước ngày đóng sổ 10/09 —
   bản sửa không tự động sửa lại dữ liệu cũ đã có sẵn.
4. Xác nhận phạm vi áp dụng vĩnh viễn cho toàn bộ nhóm nguyên liệu bông thô, không giới hạn
   thời gian, có đúng ý muốn lâu dài hay không.
5. Có muốn báo cáo riêng lỗi phụ phát hiện được ở bước 5 (số dư không được trừ theo tiêu thụ)
   thành 1 yêu cầu/ticket khác hay không.
6. **[MỚI]** Phần xử lý dành cho kỳ đóng sổ THÁNG hiện không có tác dụng cho DONGIL — có cần
   thiết phải duy trì bản sửa ở phần này không, hay chỉ để dự phòng cho nhà máy khác dùng chung?
7. **[MỚI]** Có muốn bổ sung thêm 1 lớp kiểm tra dự phòng ở bước tổng hợp tạm thời (khuyến nghị
   không bắt buộc) hay không?
8. **[MỚI, cần theo dõi]** Xác nhận lại sau lần chạy đêm nay: dữ liệu tháng 9 có tự làm sạch hết
   nguyên liệu chưa duyệt như mong đợi hay không.
