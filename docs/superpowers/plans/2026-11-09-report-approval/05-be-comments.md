# B06: Trao đổi trên báo cáo (bình luận, @nhắc, thông báo)

> Đọc `00-index.md` trước (mục 10 của "Quyết định đã chốt"). Chỉ làm Task B06. Task này cần B02 đã merge và chạy song song được với B03.

### Task B06: `ReportCommentService` + `ReportCommentController`

**Files:**
- Create:
  - `N service/ReportCommentService.java`
  - `N controller/ReportCommentController.java`
  - `N model/dto/ReportCommentItem.java`
  - `N model/dto/ReportCommentRequest.java`
- Modify:
  - `N service/ExchangeMessageHtml.java`: chỉ khi cần, để dùng chung giới hạn (xem Step 2).
  - `N service/MsbExchangeService.java`: chỉ khi chuyển hằng số.
- Test:
  - `NT service/ReportCommentServiceTest.java`
  - `NT controller/ReportCommentControllerTest.java`

**Interfaces:**
- Consumes:
  - B01: `AppraisalReportComment.create`, `AppraisalReportCommentRepository.findByAppraisalReportIdAndDeleteFlag`, `ReportApprovalNotifier.mentioned`.
  - B02: `ReportApprovalAccess.requireView`.
  - Code sẵn có: `ExchangeMessageHtml.mentionedIds(String)` và `render(String, Map<Long, MentionEmployee>)` (package-private, cùng package `service`); `EmployeeRepository.findActiveMentionsByIds`, `searchActiveMentions(String, Limit)`; `CurrentEmployeeService.require`.
- Produces:
  - `record ReportCommentRequest(@NotBlank(message = "Vui lòng nhập nội dung") @Size(max = 10000, message = "Nội dung tối đa 10000 ký tự") String content)`
  - `record ReportCommentItem(Long id, String html, Long createdBy, String createdByName, LocalDateTime createdDate)`
  - `ReportCommentService`:
    - `@Transactional(readOnly = true) Page<ReportCommentItem> list(Long callerId, long reportId, int page)`
      - `PageRequest.of(page, 30, Sort.by(DESC, "id"))`; `page < 0` thì coi là 0.
      - Tên người viết nạp một lần cho cả trang.
    - `@Transactional ReportCommentItem post(Long callerId, long reportId, String content)`
      - Content trim xong rỗng thì 400 "Vui lòng nhập nội dung". Dài hơn 10000 ký tự thì 400.
      - `ids = mentionedIds(content)`. Nhiều hơn 50 người thì 400 "Chỉ được nhắc tối đa 50 người".
      - `mentions = findActiveMentionsByIds(ids)` rồi chuyển thành map. Id không còn làm việc thì bị bỏ, không báo lỗi; `render` để nguyên token.
      - `html = render(content, mentions)`, lưu `AppraisalReportComment.create(reportId, html, callerId)`.
      - Gọi `notifier.mentioned(caller, reportId, ctx.contractId(), mentions.keySet())`. Notifier tự bỏ người viết.
    - `List<MentionEmployee> searchMentions(Long callerId, String keyword)`: `currentEmployeeService.require`, rồi `searchActiveMentions(keyword.trim(), Limit.of(20))`.
    - Không kiểm trạng thái khóa: báo cáo đang khóa vẫn bình luận được.
  - `ReportCommentController` (`/api/report-comments`):
    - `GET /{reportId}?page=0`
    - `POST /{reportId}` (`@Valid @RequestBody ReportCommentRequest`), trả `ApiResponse.ok(item, "Đã gửi tin nhắn")`
    - `GET /mention-employees?search=` (`@Size(min = 1, max = 100)`, cùng message với `MsbExchangeController.mentionEmployees`)
    - Javadoc ghi rõ: chức năng mới, không realtime, lưu HTML cùng định dạng tin MSB để FE dùng lại `MessageContent.vue`.

- [ ] **Step 1: Viết `ReportCommentServiceTest` (fail)**

  Dùng Mockito, mock `ReportApprovalAccess.requireView` để trả một `ReportApprovalContext` có `contractId` là `"HĐ1"`.

  - `post_render_html_va_bao_nguoi_duoc_nhac`: content `"Anh @[10] xem giúp"`, `findActiveMentionsByIds([10])` trả nhân viên 10.
    - HTML đã lưu chứa `data-employee-id="10"` và bắt đầu bằng `<p>`.
    - Verify `mentioned(caller, 5L, "HĐ1", {10})`.
  - `post_khong_bao_ban_than`: caller 9 nhắc `@[9]`. Notifier nhận `{9}`; test riêng của notifier ở B01 đã chứng minh nó bỏ người viết, ở đây chỉ verify tham số.
  - `post_rong_400`: content `"   "`.
  - `post_qua_10000_ky_tu_400`
  - `post_qua_50_nguoi_nhac_400`
  - `post_nhac_nguoi_da_nghi_bo_qua`
  - `nguoi_khong_co_quyen_404`: `requireView` ném `NOT_FOUND`, repository không được gọi.
  - `binh_luan_duoc_khi_bao_cao_khoa`: không gọi `ReportApprovalLocks`.
  - `list_sap_moi_nhat_va_nap_ten_mot_lan`: verify `employeeRepository.findAllById` được gọi đúng 1 lần.
  - `list_page_am_ve_0`

  Run: `cd backend && ./mvnw -q test -Dtest=ReportCommentServiceTest`. Expected: FAIL.

- [ ] **Step 2: Dùng chung giới hạn nội dung và số người nhắc**

  Grep `MAX_CONTENT_LENGTH` và `MAX_MENTIONS` trong `MsbExchangeService`.
  - Nếu chúng `private`, chuyển sang `ExchangeMessageHtml` dưới dạng `static final int` package-private. Sửa `MsbExchangeService` dùng hằng mới.
  - Chạy `./mvnw -q test -Dtest=MsbExchangeServiceTest`. Expected: PASS.

- [ ] **Step 3: Viết hai DTO và `ReportCommentService`, chạy test**

  Run: `./mvnw -q test -Dtest=ReportCommentServiceTest`. Expected: PASS.

- [ ] **Step 4: Viết `ReportCommentControllerTest` (fail)**

  Dùng `@WebMvcTest`, theo mẫu `AppraisalReportWorkspaceControllerTest`.
  - `list_200_page`
  - `post_rong_400`
  - `post_200_message`
  - `mention_search_rong_400`
  - `khong_token_401`

  Run: `./mvnw -q test -Dtest=ReportCommentControllerTest`. Expected: FAIL.

- [ ] **Step 5: Viết `ReportCommentController`, chạy test**

  Chạy lại lệnh ở Step 4, sau đó chạy `./mvnw test`. Expected: PASS.

- [ ] **Step 6: Commit**

  ```bash
  git -C backend add -A
  git -C backend commit -m "feat(report-approval): trao đổi trên báo cáo có @nhắc và thông báo"
  ```
