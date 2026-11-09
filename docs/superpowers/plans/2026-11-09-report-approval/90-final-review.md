# Z01 — Review toàn nhánh, test toàn bộ, E2E thủ công

> Chỉ làm sau khi mọi task B01–B06 và F01–F04 đã được đánh dấu `[x]`.

### Task Z01: Review cuối

**Files:** không tạo code mới. Chỉ sửa lỗi mà review xác nhận, và mỗi lần sửa là một commit riêng.

- [ ] **Step 1: Chạy toàn bộ test**

  ```bash
  cd backend && ./mvnw test
  cd frontend && npm run lint && npm run type-check && npm run test:unit && npm run build
  ```

  Expected: mọi lệnh PASS. Ghi lại số test và thời gian chạy vào báo cáo.

- [ ] **Step 2: Review toàn nhánh**

  Gọi `Agent({ model: "sonnet" })` mới. Đưa cho agent `00-index.md` và diff `git -C backend diff origin/develop...feat/report-approval` cùng diff tương ứng của frontend. Agent đối chiếu ba nhóm:

  - **Quyết định đã chốt 1–11:** mỗi mục chỉ ra được code và test tương ứng.
  - **Review Focus 1–5:** mỗi mục có test, và test đó đang PASS.
  - **Quy tắc dự án:**
    - Backend:
      - Không có native SQL mới.
      - Không có N+1 ở danh sách duyệt hay danh sách bình luận.
      - Mọi query đều lọc `delete_flag`.
      - Không serialize entity ra response.
      - Message lỗi bằng tiếng Việt.
      - `backend/CLAUDE.md` đã thêm các dòng `BranchCodes`, `ReportApprovalScope`, `ReportLockInterceptor`.
    - Frontend:
      - Không có màu thuần hay hex.
      - Nút icon-only dùng `IconButton`.
      - Modal dùng `AppModal`.
      - Xác nhận dùng `useConfirm`.
      - Bảng có đủ trạng thái loading, empty, error.
      - Không dùng `leading-tight`.

  Lỗi đã xác nhận thì sửa, chạy lại Step 1, rồi commit bằng `fix(report-approval): …`.

- [ ] **Step 3: E2E thủ công trên DB dev**

  Điều kiện: đã chạy 4 migration; backend và frontend chạy local, có S3 dev.

  1. NVTD mở báo cáo, bấm "Gửi phê duyệt" rồi giao cho một TPTD khác chi nhánh. Kỳ vọng: 400 "Người được giao không có quyền duyệt báo cáo này". (Ô chọn chỉ liệt kê ứng viên hợp lệ, nên kiểm bằng cách gọi API trực tiếp.)
  2. Gửi lại, lần này giao TPTD cùng chi nhánh. Kỳ vọng:
     - Trên màn sửa, form bị khóa nhưng vẫn chuyển tab được.
     - TPTD nhận chuông `REPORT_APPROVAL_REQUEST`. Bấm vào thì mở `/report-approvals/{id}` và thông báo chuyển thành đã đọc.
  3. Trong lúc file Word đang tạo: nút "Duyệt" bị ẩn, còn nút "Từ chối" vẫn có. Khi file READY: iframe Office Online hiển thị được, "Tải file Word" tải được file.
  4. `UPDATE linked_branch SET report_approvals_required = 2 WHERE branch_code = '<mã>'` rồi gửi một báo cáo khác của chi nhánh đó. Kỳ vọng:
     - Người duyệt thứ nhất duyệt xong thì hiện 1/2, người gửi nhận chuông "(1/2)".
     - Người gửi bấm "Rút lại" thì nút bị ẩn. Gọi API trực tiếp thì 409.
  5. Người duyệt thứ hai từ chối, để trống lý do. Kỳ vọng: báo lỗi trường. Nhập lý do rồi từ chối lại. Kỳ vọng:
     - Người gửi nhận chuông có lý do.
     - Object S3 bị xóa (kiểm tra trên bucket).
     - Màn sửa mở khóa và hiện lý do từ chối.
  6. Gửi lại, thành lượt 2. Hai người duyệt xong thì `APPROVED`. Kỳ vọng:
     - Mọi nút Lưu bị khóa.
     - Gọi `PUT /api/appraisal-records/{p}/reports/{r}/groups/{g}/land-description` trả 409 "Báo cáo đang chờ phê duyệt hoặc đã được duyệt, không thể chỉnh sửa".
     - `DELETE` báo cáo cũng trả 409.
  7. Gửi một báo cáo, rồi từ chối ngay khi worker đang tạo file (đặt `report.approval.word.poll-delay-ms` lớn, hoặc từ chối lúc `word_status = RUNNING`). Kỳ vọng: sau khi worker chạy xong, bucket không còn file của lượt đó và bản ghi có `word_s3_key` null.
  8. Ở màn sửa báo cáo, gõ `@` rồi nhắc một người. Kỳ vọng: người đó nhận chuông `REPORT_COMMENT_MENTION`. Bình luận vẫn gửi được khi báo cáo đang khóa.
  9. Kiểm phạm vi duyệt:
     - BGD có chi nhánh `HN` thấy báo cáo của mọi chi nhánh trong `/report-approvals`.
     - KSV chỉ có chi nhánh `HCM` chỉ thấy báo cáo `HCM`.
     - NVKT không thấy menu "Duyệt báo cáo", gọi API danh sách thì 403.
  10. Admin (role `admin`, chức vụ không thuộc BGD/TPTD/KSV) mở được trang duyệt, duyệt được và rút lại được báo cáo của người khác (khi chưa có ai duyệt).

  Bước nào không làm được (thiếu dữ liệu, thiếu S3) thì ghi rõ trong báo cáo, không đánh dấu là đã kiểm.

- [ ] **Step 4: Báo cáo cho người dùng**

  Báo cáo gồm các phần:
  - Kết quả Step 1: lệnh đã chạy và kết quả.
  - Kết quả Step 3: từng bước, đạt hay chưa kiểm.
  - Ngoài phạm vi, lấy từ mục cuối `00-index.md`:
    - Hệ thống cũ vẫn sửa được báo cáo đang khóa.
    - Không có màn cấu hình số người duyệt.
    - Không mở khóa báo cáo đã duyệt.
    - Trao đổi không realtime.
  - Nhắc người dùng: chưa push, chưa cập nhật con trỏ submodule. Hỏi người dùng có muốn merge hoặc push không.
