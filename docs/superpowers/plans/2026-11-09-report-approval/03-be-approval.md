# B03, B04: lệnh duyệt, truy vấn và controller

> Đọc `00-index.md` trước. B03 cần B02 đã merge, B04 cần B03 đã merge. Mỗi lần chỉ làm một task.

---

### Task B03: `ReportApprovalCommandService`

**Files:**
- Create:
  - `N service/ReportApprovalCommandService.java`
  - `N model/dto/SubmitApprovalRequest.java`
  - `N model/dto/RejectApprovalRequest.java`
- Test: `NT service/ReportApprovalCommandServiceTest.java`

**Interfaces:**
- Consumes:
  - B01: entity và repository duyệt; `LinkedBranchRepository.findFirstByBranchCodeAndDeleteFlag`; `LinkedBranch.requiredApprovals()`; `ReportApprovalNotifier`; `ReportWordDiscarded`.
  - B02: `ReportApprovalAccess.require/canApprove/canWithdraw/isEditor`, `ReportApprovalScope`, `ReportApprovalContext`.
  - Code sẵn có: `CurrentEmployeeService.require`, `EmployeeRepository`, `Clock` (bean ở `config/ClockConfig`).
- Produces: mỗi method `@Transactional`, nhận `Long callerId`, trả `long reportId` để controller gọi `ReportApprovalQueryService.detail` (B04).
  - `long submit(Long callerId, long reportId, List<Long> assigneeIds)`
  - `long approve(Long callerId, long reportId)`
  - `long reject(Long callerId, long reportId, String reason)`
  - `long withdraw(Long callerId, long reportId)`
  - `long retryWord(Long callerId, long reportId)`
  - `record SubmitApprovalRequest(@Size(max = 10, message = "Chỉ giao tối đa 10 người duyệt") List<@NotNull Long> assigneeIds)`
  - `record RejectApprovalRequest(@NotBlank(message = "Vui lòng nhập lý do từ chối") @Size(max = 1000, message = "Lý do tối đa 1000 ký tự") String reason)`

**Hành vi.** Mọi method đều khóa bản ghi bằng `approvalRepository.findForUpdate(reportId)` trước khi đọc trạng thái.

**`submit`**
1. `caller = currentEmployeeService.require(callerId)` và `ctx = access.require(reportId)`.
2. Nếu `!access.isEditor(caller)` và `caller` không phải admin thì 403 "Bạn không có quyền gửi phê duyệt báo cáo".
3. Chọn bản ghi:
   - Không có bản ghi thì `AppraisalReportApproval.create(reportId)`.
   - Có bản ghi đang `isLocked()` thì 409 "Báo cáo đã được gửi phê duyệt".
4. Xử lý danh sách giao (`assigneeIds`):
   - null coi là danh sách rỗng; bỏ id trùng, giữ thứ tự.
   - Nạp các nhân viên còn làm việc bằng `employeeRepository.findAllById`, lọc `deleteFlag = "0"` và chưa nghỉ việc theo đúng điều kiện của `findActiveMentionsByIds`.
   - Có id không tồn tại, hoặc người mà `!ReportApprovalScope.of(e).covers(ctx.linkedBranch())`, thì 400 "Người được giao không có quyền duyệt báo cáo này".
5. Tính số người duyệt cần: `ctx.linkedBranch()` null hoặc rỗng thì là 1. Ngược lại tìm chi nhánh bằng `findFirstByBranchCodeAndDeleteFlag(code, LinkedBranch.DELETE_FLAG_ACTIVE)`, lấy `requiredApprovals()`; không tìm thấy chi nhánh thì cũng là 1.
6. Ghi dữ liệu: gọi `approval.submit(required, callerId, now)` rồi `save`. Lưu action `SUBMIT`, sau đó mỗi người được giao một action `ASSIGN` (cùng `round`).
7. Gọi `notifier.approvalRequested(caller, reportId, ctx.contractId(), assignees)`.
8. Nếu `create(...)` rồi `save` gặp `DataIntegrityViolationException` (hai request gửi cùng lúc, đụng UNIQUE) thì đổi thành 409 "Báo cáo đã được gửi phê duyệt".

**`approve`**
- Bản ghi phải ở `PENDING`, nếu không thì 409 "Báo cáo không ở trạng thái chờ duyệt".
- `!access.canApprove(caller, ctx)` thì 403 "Bạn không có quyền duyệt báo cáo này".
- `wordStatus != READY` thì 409 "File Word báo cáo chưa sẵn sàng".
- Đã có action `APPROVE` của caller trong lượt này thì 409 "Bạn đã duyệt báo cáo này".
- Lưu action `APPROVE`, rồi đếm số `APPROVE` của lượt (gồm cả action vừa lưu). Đủ `requiredApprovals` thì gọi `markApproved(callerId, now)`.
- Gọi `notifier.approved(caller, reportId, contractId, submittedBy, count, required)`.

**`reject`**
- Bản ghi phải ở `PENDING`.
- `!canApprove` thì 403.
- `reason` được trim. Rỗng hoặc dài hơn 1000 ký tự thì 400. Kiểm lại ở service, không chỉ dựa vào Bean Validation.
- Gọi `approval.reject(...)` và lưu action `REJECT` có `reason`.
- `key = approval.clearWord()`. Key khác null thì `publisher.publishEvent(new ReportWordDiscarded(key))`.
- Gọi `notifier.rejected(...)`.

**`withdraw`**
- Bản ghi phải ở `PENDING`.
- `!access.canWithdraw(caller, approval)` thì 403 "Chỉ người gửi hoặc quản trị được rút lại".
- Số `APPROVE` của lượt lớn hơn 0 thì 409 "Báo cáo đã có người duyệt, không thể rút lại".
- Gọi `approval.withdraw(...)`, lưu action `WITHDRAW`, xóa file như `reject`. Không gửi thông báo.

**`retryWord`**
- Bản ghi phải ở `PENDING` và `wordStatus == FAILED`, nếu không thì 409 "File Word không ở trạng thái lỗi".
- Chỉ người gửi, admin, hoặc người có `canApprove` mới được gọi, nếu không thì 403.
- Gọi `approval.requeueWord()`.

Không có bản ghi duyệt mà gọi `approve`, `reject`, `withdraw` hay `retryWord` thì 409 "Báo cáo không ở trạng thái chờ duyệt".

- [ ] **Step 1: Viết `ReportApprovalCommandServiceTest` (fail)**

  Dùng Mockito và `Clock.fixed`. Mock repository, `ReportApprovalAccess` (stub `require`, `canApprove`), `ReportApprovalNotifier`, `ApplicationEventPublisher`, `CurrentEmployeeService`, `EmployeeRepository`, `LinkedBranchRepository`.

  Gửi duyệt:
  - `submit_lan_dau_tao_ban_ghi_round_1`: lưu bản ghi `round == 1`, `PENDING`, `QUEUED`; lưu action `SUBMIT`.
  - `submit_khi_pending_409`
  - `submit_khi_approved_409`
  - `submit_lai_sau_tu_choi_tang_round`: từ `round` 1 `REJECTED` lên `round == 2`, `rejectReason == null`.
  - `submit_chup_required_tu_chi_nhanh`: chi nhánh có `report_approvals_required = 2` thì `requiredApprovals == 2`.
  - `submit_linked_branch_null_required_1`
  - `submit_giao_nguoi_ngoai_pham_vi_400`: TPTD chi nhánh `DN` với báo cáo chi nhánh `HCM`.
  - `submit_giao_trung_id_chi_luu_mot_assign`
  - `submit_bao_noti_nguoi_duoc_giao`: verify `approvalRequested(caller, 5L, "HĐ1", [10, 11])`.
  - `submit_dong_thoi_dung_unique_409`: `save` ném `DataIntegrityViolationException`.

  Duyệt:
  - `approve_word_chua_san_sang_409`
  - `approve_lan_hai_cung_nguoi_409`
  - `approve_ngoai_pham_vi_403`
  - `approve_1_nguoi_du_thanh_approved`
  - `approve_2_nguoi_lan_dau_van_pending`: notifier nhận `count = 1, required = 2`.
  - `approve_du_2_nguoi_thanh_approved`
  - `approve_nguoi_gui_tu_duyet_duoc`

  Từ chối:
  - `reject_ly_do_rong_400`
  - `reject_ly_do_1001_400`
  - `reject_xoa_file_va_bao_nguoi_gui`: publish `ReportWordDiscarded("k")`, `wordS3Key == null`, action `REJECT` có `reason` đã trim.
  - `reject_khi_da_co_1_duyet_van_duoc`
  - `reject_khi_word_dang_tao_khong_publish`: `wordS3Key` null nên không publish.

  Rút lại:
  - `withdraw_da_co_nguoi_duyet_409`
  - `withdraw_nguoi_khac_khong_phai_admin_403`
  - `withdraw_admin_duoc`
  - `withdraw_xoa_file`

  Tạo lại file:
  - `retry_chi_khi_failed`: bản ghi `READY` thì 409; bản ghi `FAILED` thì về `QUEUED`, `wordAttempts == 0`.
  - `khong_co_ban_ghi_approve_409`

  Run: `cd backend && ./mvnw -q test -Dtest=ReportApprovalCommandServiceTest`. Expected: FAIL.

- [ ] **Step 2: Viết hai DTO và `ReportApprovalCommandService`**

  Viết theo phần "Hành vi". Mỗi bước kiểm tra là một method private ngắn. Javadoc của lớp tóm tắt luồng trạng thái và quy tắc tự duyệt, rút lại.

- [ ] **Step 3: Chạy test**

  Run: `./mvnw -q test -Dtest=ReportApprovalCommandServiceTest`. Expected: PASS.

- [ ] **Step 4: Commit**

  ```bash
  git -C backend add -A
  git -C backend commit -m "feat(report-approval): gửi, duyệt, từ chối, rút lại và tạo lại file Word"
  ```

---

### Task B04: `ReportApprovalQueryService` và controller

**Files:**
- Create:
  - `N service/ReportApprovalQueryService.java`
  - `N model/dto/` các record `ApproverCandidate`, `EmployeeRef`, `ReportApprovalWord`, `ReportApprovalVote`, `ReportApprovalHistoryItem`, `ReportApprovalDetail`, `ReportApprovalListItem`, `ReportApprovalFilter`, `ReportApprovalWordLink`. Định nghĩa đúng như "Hợp đồng dùng chung" trong `00-index.md`.
  - `N controller/ReportApprovalSubmitController.java`
  - `N controller/ReportApprovalController.java`
- Modify:
  - `N repository/AppraisalReportApprovalRepository.java`: thêm `search`.
  - `N repository/EmployeeRepository.java`: thêm `findActiveApproverCandidates`.
- Test:
  - `NT service/ReportApprovalQueryServiceTest.java`
  - `NT repository/AppraisalReportApprovalRepositoryTest.java` (thêm case)
  - `NT controller/ReportApprovalControllerTest.java`
  - `NT controller/ReportApprovalSubmitControllerTest.java`

**Interfaces:**
- Consumes: B01, B02, B03 (`ReportApprovalCommandService`), cùng các thành phần sẵn có `BranchNames.load(linkedBranchRepository)`, `LikePattern.contains`, `S3LinkService.presign`, `OfficeFiles.viewerUrl`.
- Produces:
  - `Page<ReportApprovalListItem> list(Long callerId, ReportApprovalFilter filter, Pageable pageable)`: phạm vi `isNone()` thì 403 "Bạn không có quyền duyệt báo cáo". Status null hoặc rỗng thì mặc định `PENDING`. Status không hợp lệ thì 400.
  - `ReportApprovalDetail detail(Long callerId, long reportId)`: gọi `access.requireView`.
  - `List<ApproverCandidate> candidates(Long callerId, long reportId)`
  - `ReportApprovalWordLink wordLink(Long callerId, long reportId)`: chỉ trả khi `READY`, nếu không thì 404 "File Word chưa sẵn sàng".
    - `downloadUrl` = `s3LinkService.presign(key)`.
    - `viewerUrl` = `OfficeFiles.viewerUrl(downloadUrl)`.
  - `AppraisalReportApprovalRepository.search(...)`: JPQL, trả `Page<ReportApprovalRow>`. `ReportApprovalRow` là record projection gồm `reportId, contractId, linkedBranch, submittedBy, submittedAt, round, requiredApprovals, status, wordStatus`.
    - Join các entity qua id: `FROM AppraisalReportApproval a JOIN AppraisalReport r ON r.id = a.appraisalReportId JOIN AppraisalRequest q ON q.id = r.appraisalRequestId`. Điều kiện cố định: `a.deleteFlag = '0' AND r.deleteFlag = 0`.
    - Tham số:
      - `:status`
      - `:all` (boolean) và `:branches`, ghép thành `(:all = true OR q.linkedBranch IN :branches)`
      - `:branch` theo dạng `(:branch IS NULL OR q.linkedBranch = :branch)`
      - `:keyword`, dùng `LikePattern.contains`, rỗng thì null
      - `:assignedTo`, dạng `(:assignedTo IS NULL OR EXISTS (SELECT 1 FROM AppraisalReportApprovalAction x WHERE x.appraisalReportId = a.appraisalReportId AND x.round = a.round AND x.action = ASSIGN AND x.employeeId = :assignedTo AND x.deleteFlag = '0'))`
    - Có `countQuery` riêng.
    - Danh sách branches rỗng mà phạm vi là BRANCHES thì service trả `Page.empty` luôn, không gọi query.
    - Phạm vi ALL thì truyền `branches = List.of("")` để tránh `IN ()`.
  - `EmployeeRepository.findActiveApproverCandidates()`: JPQL lấy nhân viên còn làm việc (cùng điều kiện với `searchActiveMentions`) có `role = 'admin'` hoặc `position IN (BGD, TPTD, KSV)`, sắp theo họ tên.
  - Controller (`ReportApprovalController`, prefix `/api/report-approvals`):
    - `GET ""` (`@RequestParam` gồm status, branch, keyword, `assignedToMe` mặc định false, `Pageable` size tối đa 100)
    - `GET /{reportId}`
    - `POST /{reportId}/approve`
    - `POST /{reportId}/reject` (`@Valid RejectApprovalRequest`)
    - `POST /{reportId}/withdraw`
    - `POST /{reportId}/word/retry`
    - `GET /{reportId}/word`
  - Controller (`ReportApprovalSubmitController`, prefix `/api/appraisal-reports/{reportId}/approval`):
    - `POST /submit` (`@Valid SubmitApprovalRequest`)
    - `GET /approver-candidates`
  - Mọi POST trả `ApiResponse.ok(queryService.detail(callerId, id), "<message>")`, message lấy đúng theo bảng API.

**Cách dựng `detail`:**
- Không có bản ghi duyệt thì trả `status = "DRAFT"`, `round = null`, `requiredApprovals` = số người duyệt của chi nhánh, `history = []`, `word = null`.
  - `canSubmit = isEditor || admin`.
  - Các cờ còn lại là false, trừ `canOpenReport`.
- Các danh sách lấy từ action:
  - `approvals`: action `APPROVE` của lượt hiện tại.
  - `assignees`: action `ASSIGN` của lượt hiện tại.
  - `history`: mọi action, trừ `ASSIGN`.
  - Tên nhân viên nạp một lần bằng `employeeRepository.findAllById`.
- Các cờ:
  - `canSubmit`: `(isEditor || admin) && !approval.isLocked()`.
  - `canApprove`: `PENDING && READY && canApprove && chưa duyệt lượt này`.
  - `canReject`: `PENDING && canApprove`.
  - `canWithdraw`: `PENDING && canWithdraw && approvedCount == 0`.
  - `canRetryWord`: `PENDING && FAILED && (người gửi || admin || canApprove)`.
  - `canOpenReport`: `isEditor`.
- `branchName` lấy bằng `BranchNames.load(linkedBranchRepository).join(linkedBranch)`.

**Ứng viên duyệt:** `findActiveApproverCandidates()`, lọc theo `ReportApprovalScope.of(e).covers(ctx.linkedBranch())`. `branchNames` lấy qua `BranchNames`.

**Danh sách:**
- `approvedCount` của các dòng trong trang lấy bằng **một** query đếm theo lô: JPQL `GROUP BY appraisalReportId, round` trên action `APPROVE`, với `reportId IN :ids`.
- `assignedToMe` cũng lấy bằng **một** query cho cả trang.
- Không truy vấn theo từng dòng (N+1).

- [ ] **Step 1: Viết test repository `search` (fail)**

  Thêm vào `AppraisalReportApprovalRepositoryTest`. Dữ liệu là 2 báo cáo thật (lấy qua `EntityManager`) có `linked_branch` khác nhau.
  - `search_loc_trang_thai`
  - `search_pham_vi_branches`
  - `search_all_khong_loc_chi_nhanh`
  - `search_loc_mot_chi_nhanh`
  - `search_assigned_to_me_chi_luot_hien_tai`: action `ASSIGN` của round cũ không được tính.
  - `search_keyword_co_phan_tram_khong_loi`
  - `search_bo_qua_bao_cao_da_xoa`

  Cũng thêm `EmployeeRepositoryTest.find_active_approver_candidates` (file đã có thì thêm case, chưa có thì tạo).

  Run: `cd backend && ./mvnw -q test -Dtest='AppraisalReportApprovalRepositoryTest,EmployeeRepositoryTest'`. Expected: FAIL.

- [ ] **Step 2: Viết hai query, chạy lại test repository**

  Expected: PASS.

- [ ] **Step 3: Viết `ReportApprovalQueryServiceTest` (fail)**

  Mockito:
  - `detail_chua_gui_la_draft_can_submit`
  - `detail_pending_nguoi_duyet_trong_pham_vi_can_approve`
  - `detail_word_queued_khong_can_approve_nhung_can_reject`
  - `detail_da_duyet_luot_nay_khong_can_approve`
  - `detail_co_1_duyet_khong_can_withdraw`
  - `detail_failed_can_retry_cho_nguoi_gui`
  - `detail_lich_su_cu_den_moi_khong_co_assign`
  - `detail_khong_quyen_404`
  - `list_pham_vi_none_403`
  - `list_status_mac_dinh_pending`
  - `list_status_sai_400`
  - `list_branches_rong_tra_trang_rong`
  - `list_dem_duyet_theo_lo_mot_query`: verify query đếm được gọi đúng 1 lần.
  - `candidates_loc_theo_pham_vi`
  - `word_link_chua_ready_404`
  - `word_link_ready_tra_viewer_url`

  Run: `./mvnw -q test -Dtest=ReportApprovalQueryServiceTest`. Expected: FAIL.

- [ ] **Step 4: Viết các DTO và `ReportApprovalQueryService`, chạy test**

  Expected: PASS.

- [ ] **Step 5: Viết test controller (fail)**

  `@WebMvcTest`, mẫu theo `AppraisalReportWorkspaceControllerTest`, mock `ReportApprovalCommandService` và `ReportApprovalQueryService`.

  `ReportApprovalControllerTest`:
  - `list_tra_page`: kiểm tra tham số `assignedToMe=true` được truyền vào filter.
  - `detail_200`
  - `approve_goi_command_roi_detail`
  - `reject_ly_do_rong_400`
  - `reject_ly_do_1001_400`
  - `withdraw_200_message`
  - `retry_200`
  - `word_200`

  `ReportApprovalSubmitControllerTest`:
  - `submit_11_nguoi_400`
  - `submit_null_assignee_ids_duoc`
  - `submit_khong_token_401`
  - `submit_nvtd_200`: dùng `bearer("NVTD")`.
  - `candidates_200`

  Run: `./mvnw -q test -Dtest='ReportApprovalControllerTest,ReportApprovalSubmitControllerTest'`. Expected: FAIL.

- [ ] **Step 6: Viết hai controller, chạy test**

  Controller dùng `CurrentUser.id(jwt)`, trả DTO hoặc `Page`. Javadoc ghi rõ đây là chức năng mới, hệ thống cũ không có. Chạy lại lệnh ở Step 5, sau đó chạy `./mvnw test`. Expected: PASS.

- [ ] **Step 7: Commit**

  ```bash
  git -C backend add -A
  git -C backend commit -m "feat(report-approval): API danh sách, chi tiết, ứng viên duyệt và link file Word"
  ```
