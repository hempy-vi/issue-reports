# Dev Report — CESM-726

**Task:** [Samil] [Data handling] Impact Analysis – Yarn Code Changes in SB1.2

---

## 2026-09-10

**1. Xác định form nguồn cần trace**

Ticket liệt kê 5 form: SB1.1 (Item Code), SB1.2 (Yarn Code — nguồn phát sinh xoá/sửa), SB1.13 (R&D Item Register), SB1.16 (Item Code Inquiry), SB1.27 (Yarn Code Inquiry). Không có mapping tĩnh mã "SB1.x" sang tên file `.aspx` trong source (`<title>` chỉ ghi tên framework mặc định "genuwin"), nên xin user cung cấp URL thật qua screenshot menu, khớp lại bằng bảng menu hệ thống `TES_OBJ`:

| Ticket form | Path | Select proc |
|---|---|---|
| SB1.1 | `/form/sa/10/sa100010.aspx` | `SP_SEL_SA100010` |
| SB1.2 | `/form/sa/10/sa100020.aspx` | `SP_SEL_SA100020` |
| SB1.13 | `/form/sa/10/sa100130.aspx` | `SP_SEL_SA100130` |
| SB1.16 | `/form/sa/10/sa100010_V02.aspx` | `SP_SEL_SA100010` (dùng chung SB1.1) |
| SB1.27 | `/form/sa/10/sa100020_V02.aspx` | `SP_SEL_SA100020` (dùng chung SB1.2) |

**2. Trace cơ chế lưu trữ Yarn Code — phát hiện cốt lõi**

`SA_YARN_CODE` không phải bảng riêng mà là VIEW:
```sql
SELECT PK, ITEM_CODE YARN_CODE, ..., DEL_IF, ...
  FROM TLG_IT_ITEM
 WHERE TLG_IT_ITEMGRP_PK = 1243
```
Yarn Code chỉ là các dòng của bảng ITEM MASTER dùng chung (`TLG_IT_ITEM`) thuộc nhóm item 1243. Save/Delete của SB1.2 (`SP_UPD_SA100020`) gọi qua procedure item master dùng chung `LG_UPD_AGCI00070_2`.

- **Xoá = soft-delete**: `TLG_IT_ITEM.DEL_IF := PK` (không phải `=1`) — dòng vẫn tồn tại vật lý, chỉ bị lọc khỏi mọi query có `WHERE DEL_IF = 0`.
- **Sửa = update tại chỗ, PK giữ nguyên** — mọi form phụ thuộc join theo PK (không theo text `YARN_CODE`), nên đổi text mã yarn tự động propagate, không cần cập nhật thủ công.

**3. Phát hiện root cause kỹ thuật — guard xoá thiếu kiểm tra**

`LG_UPD_AGCI00070_2` trước khi cho xoá chỉ check:
```sql
SELECT COUNT(*) FROM TLG_IN_STOCKTR WHERE DEL_IF=0 AND TLG_IT_ITEM_PK = P_TCO_ITEM_PK
```
Nếu > 0 mới chặn xoá. Guard này KHÔNG kiểm tra `SA_ITEM_CODE.YARN01_PK..YARN09_PK` (6.980 dòng đang gán) hay `SA_RND_ITEM_D.SA_YARN_CODE_PK` (50.960 dòng đang tham chiếu) — nghĩa là 1 yarn code đang dùng trong công thức R&D/Item nhưng chưa phát sinh tồn kho vẫn xoá được, không cảnh báo.

**4. Impact chi tiết theo từng form khi yarn bị XOÁ**

| Form | Cách join | Hành vi khi xoá |
|---|---|---|
| SB1.2 / SB1.27 | dùng chung `SP_SEL_SA100020` | Dòng biến mất khỏi lưới (có nút UnDelete) |
| SB1.1 / SB1.16 | `YARN0X_PK` LEFT OUTER JOIN | Item không mất, chỉ cột "Yarn N" đúng slot trống (orphan FK) |
| SB1.13 | `SA_YARN_CODE_PK` INNER JOIN (CTE) | R&D item không mất, nhưng yarn đó âm thầm biến mất khỏi breakdown yarn1-4 — không cảnh báo |

**5. Mở rộng phạm vi theo yêu cầu user — map toàn hệ thống qua `TES_OBJ`**

User yêu cầu xác định TẤT CẢ form bị ảnh hưởng (không chỉ 5 form ticket), map ra menu code thật qua bảng `TES_OBJ`. Quét `USER_SOURCE` toàn bộ tham chiếu literal `SA_YARN_CODE`: ra 74 database object trải trên ≥9 module. Map qua `TES_OBJ` (đi ngược `P_PK` tới root để dựng full breadcrumb menu) ra thêm:
- SB.1.14 "Item Code Inquiry" (trùng tên với SB.1.16 nhưng khác form, dễ bị bỏ sót).
- 17 form khác: Knitting (SK.2.2), Yarn Supplier (SD.13.1/2/3/7), Sales (SS.1.6/6.1), Processing (SS.2.4/2.5/2.5.1/2.5.2), Production (SM.1.6≡SP.11.3/1.6.1/1.7.1), Yarn/Warehouse (SS.6.1/6.2/6.3).

**6. User chỉ thêm 2 gap thật (Stock In, BOM) — mở rộng scan, phát hiện methodology gap**

User (kinh nghiệm nghiệp vụ thật) chỉ ra IV0102 (Stock In) và BOM bị bỏ sót. Trace `LG_SEL_BINI00030_2` (Stock In Entry): join trực tiếp `TLG_IT_ITEM` qua `income_item_pk`/`req_item_pk`, KHÔNG qua view `SA_YARN_CODE` — root cause: scan cũ (grep text "SA_YARN_CODE") chỉ bắt được form "biết mình đang xử lý yarn", bỏ sót form dùng "item" chung chung (generic item_pk, không phân biệt loại) dù thực tế vẫn ảnh hưởng.

Xác nhận qua data thật: BOM (`TLG_DO_BO_BOM_V3.CHILD_PK`) có 13.296 dòng thật group=1243; Stock In (`TLG_ST_INCOME_D.INCOME_ITEM_PK`) có 58.093 dòng thật group=1243 — cả 2 xác nhận đúng.

**7. Quét toàn bộ ERP tìm form dùng chung Item Master — ra 237 candidate (quá nhiều false positive)**

Do scan text "TLG_IT_ITEM" quá chung chung (bảng chứa MỌI loại item toàn ERP, ~60 group khác nhau), quét full ERP ra 237 form candidate (~25% toàn hệ thống). Query trực tiếp DB bị timeout (USER_SOURCE có 2,58 triệu dòng) — giải pháp: tách 2 query nhẹ (list proc + list form active, phân trang qua `maxRows`), tải về xử lý local bằng script Node.js, match string offline.

**8. User phản biện đúng — nhiều form Nhóm 5 là false positive**

User chỉ ra: "thay đổi item code sợi không thể ảnh hưởng tới xuất thành phẩm/nhuộm/in — in và nhuộm dùng item code THÀNH PHẨM chứ không dùng trực tiếp code sợi". Trace cụ thể G/D Entry (`SP_SEL_SA2200010`): join `SA_ORDER_PRODUCTION.SA_ITEM_CODE_PK` → hiển thị item code của THÀNH PHẨM (group 1263 "IT01-Item Code"), KHÔNG hiển thị cột Yarn nào — xác nhận đúng: xoá yarn không gây thay đổi gì quan sát được trên form này.

Lấy bảng `TLG_IT_ITEMGRP` (61 dòng) xác nhận: group 1243 ("YN01-Yarn Code") là group DUY NHẤT SB1.2 quản lý, nhưng có ít nhất 3 group khác cũng tên "yarn"-ish nhưng hoàn toàn khác (1206, 1393, 1394) — nếu form nào dùng nhóm khác thì không liên quan gì tới SB1.2.

**9. Launch Workflow verify TỪNG form trong 199 candidate bằng data thật (không chỉ text-match)**

Mỗi agent: đọc source proc đã match, tìm cột join thật tới `TLG_IT_ITEM`, chạy query đếm dòng thật có `TLG_IT_ITEMGRP_PK=1243` qua đúng cột đó → verdict CONFIRMED/REJECTED/INCONCLUSIVE kèm bằng chứng số liệu thật.

Chạy 2 lần bị chặn bởi rate limit (session limit rồi weekly limit), lần 3 (fresh full run, 15/15 agent thành công) ra kết quả cuối:
```
80 CONFIRMED, 115 REJECTED, 4 INCONCLUSIVE (trên 199 form)
```

**10. Kết quả cuối cùng**

Danh sách impacted form cuối: 24 (Nhóm 1-3, xác nhận literal `SA_YARN_CODE`) + 69 (Nhóm 5 xác nhận data thật, dedupe) = **93 form** — giảm từ 213 (bản trước, lẫn false positive) xuống 93 (bản chuẩn, từng dòng có bằng chứng data thật).

Phát hiện đáng chú ý: `SM0201` ("Sales Order"/"BOM Creation v4") tự thêm trước đó chỉ dựa đếm text-match, verify kỹ ra 0 dòng thật group 1243 (kể cả 3 cột tên như yarn: `WARP_MATERIAL_PK`/`TABBY_ITEM_PK`/`FILLCORD_ITEM_PK`) → REJECTED, tự sửa loại khỏi list. `TM1020 "Yarn Spec Property"` — REJECTED dù tên có chữ "Yarn" y hệt ticket, vì bảng riêng `TLG_KL_YAN_SPEC` không có dòng nào thuộc group 1243 (thuộc hệ thống Test Management riêng, khác group).

Deliverable cuối: 1 file PDF duy nhất (`impacted_forms_list.pdf`) — full menu path, 93 form, chia theo module, card màu theo nhóm độ tin cậy.
