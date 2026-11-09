# B01 — Nền backend: schema, entity, repository, thông báo, S3

> Đọc `00-index.md` trước (Quyết định đã chốt, Hợp đồng dùng chung, Global Constraints). Chỉ làm Task B01.

### Task B01: Nền BE

**Files:**
- Create:
  - `NR/db/migration/V20261110090000__add_report_approvals_required_to_linked_branch.sql`
  - `NR/db/migration/V20261110090100__create_appraisal_report_approvals.sql`
  - `NR/db/migration/V20261110090200__create_appraisal_report_comments.sql`
  - `NR/db/migration/V20261110090300__add_ref_id_to_notifications.sql`
  - `N model/entity/AppraisalReportApproval.java`
  - `N model/entity/AppraisalReportApprovalAction.java`
  - `N model/entity/AppraisalReportComment.java`
  - `N repository/AppraisalReportApprovalRepository.java`
  - `N repository/AppraisalReportApprovalActionRepository.java`
  - `N repository/AppraisalReportCommentRepository.java`
  - `N common/constant/BranchCodes.java`
  - `N service/ReportApprovalNotifier.java`
  - `N service/ReportWordDiscarded.java`
  - `N service/ReportWordCleanupListener.java`
- Modify:
  - `N model/entity/LinkedBranch.java`: thêm cột `report_approvals_required`
  - `N model/entity/Notification.java`: thêm `refId` và factory `about`
  - `N model/dto/NotificationItem.java`: thêm `refId` và 5 hằng loại
  - `N controller/NotificationController.java` và `N service/NotificationService.java` (hoặc service đang phục vụ controller này, grep để tìm): thêm `POST /{id}/read`
  - `N service/S3StorageService.java`: thêm overload `upload(String, byte[], String)`
  - `NR/application.properties`: thêm `spring.task.scheduling.pool.size=2` và `report.approval.word.poll-delay-ms=5000`
  - `N schedule/ImportJobWorker.java`: chỉ sửa javadoc
  - `backend/CLAUDE.md`: thêm dòng `BranchCodes` vào bảng helper
- Test:
  - `NT model/entity/AppraisalReportApprovalTest.java`
  - `NT repository/AppraisalReportApprovalRepositoryTest.java`
  - `NT repository/AppraisalReportCommentRepositoryTest.java`
  - `NT repository/LinkedBranchRepositoryTest.java` (thêm case vào file sẵn có, nếu chưa có thì tạo)
  - `NT repository/NotificationRepositoryTest.java` (thêm case)
  - `NT service/ReportApprovalNotifierTest.java`
  - `NT service/ReportWordCleanupListenerTest.java`
  - `NT service/S3StorageServiceTest.java` (thêm case, nếu chưa có thì tạo)
  - `NT controller/NotificationControllerTest.java` (thêm case)

**Interfaces:**
- Consumes: không có (task đầu tiên).
- Produces:
  - `BranchCodes.HEAD_OFFICE` (`"HN"`).
  - `AppraisalReportApproval`, có `@Getter`, `@NoArgsConstructor(access = PROTECTED)` và các thành phần sau:
    - enum `Status { PENDING, APPROVED, REJECTED, WITHDRAWN }` lưu `@Enumerated(STRING)`; enum `WordStatus { QUEUED, RUNNING, READY, FAILED }`.
    - hằng `DELETE_FLAG_ACTIVE = "0"`, `MAX_WORD_ATTEMPTS = 3`.
    - `static AppraisalReportApproval create(long reportId)`: chưa submit. Gọi `submit` ngay sau đó.
    - `void submit(int requiredApprovals, long submittedBy, LocalDateTime now)`:
      - `round` tăng 1 (lần đầu thành 1), `status = PENDING`;
      - xóa `decidedBy`, `decidedAt`, `rejectReason`;
      - `wordStatus = QUEUED`, `wordAttempts = 0`, `wordError = null`;
      - nếu trạng thái hiện tại là PENDING hoặc APPROVED thì ném `IllegalStateException`.
    - `void markApproved(long by, LocalDateTime now)`: yêu cầu đang PENDING.
    - `void reject(long by, String reason, LocalDateTime now)`: yêu cầu đang PENDING.
    - `void withdraw(long by, LocalDateTime now)`: yêu cầu đang PENDING.
    - `boolean isLocked()`: trả true khi PENDING hoặc APPROVED.
    - `void claimWord()`: chuyển QUEUED sang RUNNING.
    - `void completeWord(String key, String fileName, LocalDateTime now)`: chuyển sang READY.
    - `boolean failWord(String error)`: `wordAttempts++`. Nếu chưa tới `MAX_WORD_ATTEMPTS` thì về QUEUED và trả `false`. Tới ngưỡng thì chuyển FAILED và trả `true` (đã hết lượt thử). `error` cắt còn 1000 ký tự.
    - `void requeueWord()`: chuyển FAILED (hoặc RUNNING khi requeue lúc khởi động) sang QUEUED, `wordAttempts = 0`.
    - `String clearWord()`: trả key cũ (có thể null), đặt `wordStatus`, `wordS3Key`, `wordFileName`, `wordGeneratedAt` về null.
  - `AppraisalReportApprovalAction`:
    - enum `Action { SUBMIT, ASSIGN, APPROVE, REJECT, WITHDRAW }`;
    - `static AppraisalReportApprovalAction of(long reportId, int round, Action action, long employeeId, String reason)`.
  - `AppraisalReportComment`: `static AppraisalReportComment create(long reportId, String html, long createdBy)`.
  - `AppraisalReportApprovalRepository extends JpaRepository<AppraisalReportApproval, Long>`:
    - `Optional<AppraisalReportApproval> findByAppraisalReportIdAndDeleteFlag(Long reportId, String deleteFlag)`
    - `@Lock(PESSIMISTIC_WRITE) @Query("SELECT a FROM AppraisalReportApproval a WHERE a.appraisalReportId = :reportId AND a.deleteFlag = '0'") Optional<AppraisalReportApproval> findForUpdate(@Param("reportId") long reportId)`
    - `boolean existsByAppraisalReportIdAndStatusInAndDeleteFlag(Long reportId, Collection<Status> statuses, String deleteFlag)`
    - `Optional<AppraisalReportApproval> findFirstByWordStatusAndDeleteFlagOrderByIdAsc(WordStatus status, String deleteFlag)`
    - `@Modifying @Query int requeueRunningWords()`: chuyển RUNNING về QUEUED, trả số dòng.
  - `AppraisalReportApprovalActionRepository`:
    - `List<AppraisalReportApprovalAction> findByAppraisalReportIdAndDeleteFlagOrderByIdAsc(Long reportId, String deleteFlag)`
    - `long countByAppraisalReportIdAndRoundAndActionAndDeleteFlag(Long reportId, Integer round, Action action, String deleteFlag)`
    - `boolean existsByAppraisalReportIdAndRoundAndActionAndEmployeeIdAndDeleteFlag(Long reportId, Integer round, Action action, Long employeeId, String deleteFlag)`
  - `AppraisalReportCommentRepository`: `Page<AppraisalReportComment> findByAppraisalReportIdAndDeleteFlag(Long reportId, String deleteFlag, Pageable pageable)`.
  - `LinkedBranch.getReportApprovalsRequired()` (Integer) và `int requiredApprovals()`: trả 2 khi giá trị là 2, mọi giá trị khác (kể cả null) trả 1.
  - `Notification.about(String type, Long fromId, Long toId, Long refId, String content)`: `appraisalRequestId = null`, `read = false`. `Notification.getRefId()`.
  - `NotificationItem`:
    - thêm trường `Long refId` (thêm vào cuối record, sửa mọi chỗ khởi tạo);
    - hằng `REPORT_APPROVAL_REQUEST`, `REPORT_APPROVED`, `REPORT_REJECTED`, `REPORT_WORD_FAILED`, `REPORT_COMMENT_MENTION`.
  - `ReportApprovalNotifier` (`@Component`):
    - `void approvalRequested(Employee sender, long reportId, String contractId, Collection<Long> assigneeIds)`
    - `void approved(Employee approver, long reportId, String contractId, long submitterId, long approvedCount, int required)`: chọn câu "chưa đủ" hoặc "đủ" theo `approvedCount >= required`.
    - `void rejected(Employee rejecter, long reportId, String contractId, long submitterId, String reason)`
    - `void wordFailed(long reportId, String contractId, long submitterId)`: `fromId = submitterId`.
    - `void mentioned(Employee author, long reportId, String contractId, Collection<Long> mentionedIds)`
    - Mọi method bỏ người nhận trùng `fromId`. Không còn ai nhận thì không ghi DB. Có người nhận thì `notificationRepository.saveAll(...)` rồi `publisher.publishEvent(new NotificationsCreated(saved))`.
    - Nội dung đúng bảng "Loại thông báo" trong `00-index.md`. `contractId` null thì dùng `"#" + reportId`.
  - `record ReportWordDiscarded(String s3Key)` (public).
  - `ReportWordCleanupListener`: `@Async @TransactionalEventListener(phase = AFTER_COMMIT, fallbackExecution = true) void on(ReportWordDiscarded e)`. Gọi `s3StorageService.delete(e.s3Key())`. `RuntimeException` chỉ được log `warn`, không ném ra ngoài.
  - `S3StorageService.upload(String key, byte[] content, String contentType)`.
  - `POST /api/notifications/{id}/read` trả `ApiResponse.ok(null)`. Dùng `NotificationRepository.markRead(id, callerId)` sẵn có (grep để biết đúng signature). Thông báo của người khác thì không đổi gì và vẫn trả 200, giống `markRead` hiện có.

- [ ] **Step 1: Tạo nhánh và migration, chạy migration trên DB dev**

  Tạo nhánh theo `00-index.md` mục "Nhánh làm việc". Viết 4 file migration đúng DDL trong `00-index.md`, mỗi file có một dòng comment tiếng Việt ở đầu. Chạy app một lần (`./mvnw spring-boot:run`, Flyway bật) hoặc chạy tay 4 file trên DB dev, rồi kiểm tra:

  ```sql
  DESCRIBE appraisal_report_approvals; DESCRIBE appraisal_report_approval_actions;
  DESCRIBE appraisal_report_comments; SHOW COLUMNS FROM linked_branch LIKE 'report_approvals_required';
  SHOW COLUMNS FROM notifications LIKE 'ref_id';
  ```

  Expected: có đủ bảng và cột. **Không chạy được (thiếu quyền, không có DB) thì dừng và báo lại.**

- [ ] **Step 2: Viết test entity thuần (fail)**

  `AppraisalReportApprovalTest`:
  - `submit_lan_dau_round_1_pending_queued`: `create(5).submit(1, 9L, now)` cho `round == 1`, `status == PENDING`, `wordStatus == QUEUED`, `submittedBy == 9`.
  - `submit_khi_pending_nem_loi`: `assertThrows(IllegalStateException.class, ...)`.
  - `submit_lai_sau_reject_tang_round_va_xoa_ly_do`: sau `reject(1L, "sai", now)` rồi `submit(2, 9L, now)` thì `round == 2`, `rejectReason == null`, `requiredApprovals == 2`.
  - `fail_word_lan_3_thanh_failed`: gọi `failWord("x")` hai lần đầu trả `false` và `wordStatus == QUEUED`, lần ba trả `true` và `wordStatus == FAILED`.
  - `clear_word_tra_key_cu`: sau `completeWord("k", "a.docx", now)` thì `clearWord()` trả `"k"` và `wordS3Key == null`.
  - `is_locked`: PENDING và APPROVED cho true; REJECTED và WITHDRAWN cho false.
  - Trong cùng file, thêm case cho `LinkedBranch.requiredApprovals()`: null, 0, 1, 3 cho 1; 2 cho 2. Có thể dựng bằng reflection hoặc constructor test sẵn có của entity.

  Run: `cd backend && ./mvnw -q test -Dtest=AppraisalReportApprovalTest`. Expected: FAIL (lớp chưa tồn tại).

- [ ] **Step 3: Viết entity, enum, hằng số, sửa `LinkedBranch` và `Notification`**

  Ánh xạ mọi cột bằng `@Column(name = ...)`. Ngày giờ dùng `LocalDateTime`. `created` và `updated` để DB tự đặt (`insertable = false, updatable = false`), giống cách `Notification.createDate` đang làm. Không dùng `@Data`, `@Setter`. Javadoc của `AppraisalReportApproval` tóm tắt luồng trạng thái và ghi rõ "bảng của hệ thống mới, hệ thống cũ không dùng".

- [ ] **Step 4: Chạy test entity**

  Run: `./mvnw -q test -Dtest=AppraisalReportApprovalTest`. Expected: PASS.

- [ ] **Step 5: Viết test repository (fail)**

  `AppraisalReportApprovalRepositoryTest` (`@MysqlDataJpaTest`, lấy một `appraisal_reports.id` có thật qua `EntityManager` như `NotificationRepositoryTest`):
  - `luu_va_doc_moi_cot`: mọi field ghi ra đọc lại đúng, `created` khác null sau `flush` + `clear`.
  - `find_for_update_tra_ban_ghi`
  - `exists_status_in_pending_approved`
  - `find_first_queued_theo_id`
  - `requeue_running_words_tra_so_dong`
  - `unique_chan_duyet_trung`: lưu 2 action `APPROVE` cùng `(report, round, employee)` rồi `flush`, expected `DataIntegrityViolationException`.
  - `count_approve_theo_luot`

  `AppraisalReportCommentRepositoryTest`: `phan_trang_moi_nhat_truoc` (sắp `id DESC`) và `bo_qua_ban_ghi_da_xoa`.

  Thêm case `doc_report_approvals_required` vào `LinkedBranchRepositoryTest` và `luu_doc_ref_id` vào `NotificationRepositoryTest`.

  Run: `./mvnw -q test -Dtest='AppraisalReportApprovalRepositoryTest,AppraisalReportCommentRepositoryTest,LinkedBranchRepositoryTest,NotificationRepositoryTest'`. Expected: FAIL.

- [ ] **Step 6: Viết 3 repository**

  Dùng derived query và JPQL như phần Interfaces. Không dùng native SQL.

- [ ] **Step 7: Chạy test repository**

  Chạy lại lệnh ở Step 5. Expected: PASS.

- [ ] **Step 8: Viết test cho notifier, listener, S3 và controller (fail)**

  - `ReportApprovalNotifierTest` (Mockito):
    - `approval_requested_bo_nguoi_gui`: người được giao là `[9, 10]`, người gửi là 9, nên chỉ lưu 1 thông báo, `toEmployeeId = 10`, `type = REPORT_APPROVAL_REQUEST`, `refId = reportId`.
    - `approved_chua_du`: content đúng bằng `"Nguyễn A đã duyệt báo cáo thẩm định số hợp đồng HĐ1 (1/2)"`.
    - `approved_du`: content `"Báo cáo thẩm định số hợp đồng HĐ1 đã được phê duyệt"`.
    - `rejected_cat_ly_do_200`: lý do dài 300 ký tự thì phần lý do trong content dài 200 ký tự.
    - `contract_null_dung_ma_bao_cao`: content chứa `"số hợp đồng #5"`.
    - `khong_ai_nhan_khong_luu`: không gọi `saveAll`, không publish.
    - `publish_notifications_created`
  - `ReportWordCleanupListenerTest`:
    - `xoa_key`
    - `s3_loi_khong_nem`: `delete` ném `RuntimeException`, gọi `on(...)` không ném gì.
  - `S3StorageServiceTest.upload_bytes_dat_content_type_va_cache_control`: mock `S3Client`, bắt `PutObjectRequest`.
  - `NotificationControllerTest.read_goi_mark_read`: `POST /api/notifications/7/read` trả 200, verify `markRead(7, callerId)`.

  Run: `./mvnw -q test -Dtest='ReportApprovalNotifierTest,ReportWordCleanupListenerTest,S3StorageServiceTest,NotificationControllerTest'`. Expected: FAIL.

- [ ] **Step 9: Viết `ReportApprovalNotifier`, `ReportWordDiscarded`, `ReportWordCleanupListener`, overload S3 và endpoint read**

  Listener viết theo mẫu `N service/ExchangeFileCleanupListener.java`. Overload S3 viết theo `upload(String, MultipartFile)`, cùng header `Cache-Control`.

- [ ] **Step 10: Chạy test**

  Chạy lại lệnh ở Step 8. Expected: PASS.

- [ ] **Step 11: Cấu hình scheduler, sửa javadoc, cập nhật CLAUDE.md**

  - `application.properties`: thêm `spring.task.scheduling.pool.size=2` và `report.approval.word.poll-delay-ms=5000`, kèm comment tiếng Việt: hai worker không chặn nhau; một `@Scheduled` không bao giờ chạy chồng lên chính nó.
  - `ImportJobWorker`: sửa câu "Scheduler mặc định chỉ có một luồng nên các lô không bao giờ chạy song song" thành lý do đúng: một method `@Scheduled(fixedDelay)` không chạy chồng lên chính nó.
  - `backend/CLAUDE.md`: thêm dòng `Mã chi nhánh hội sở "HN"` → `BranchCodes.HEAD_OFFICE` (`common/constant`) | Khai báo `HEAD_OFFICE = "HN"` private trong service mới.

- [ ] **Step 12: Chạy toàn bộ test backend**

  Run: `./mvnw test`. Expected: PASS. Mọi chỗ khởi tạo `NotificationItem` đã được sửa theo record mới.

- [ ] **Step 13: Commit**

  ```bash
  git -C backend add -A
  git -C backend commit -m "feat(report-approval): schema, entity, repository, notifier và dọn file Word"
  ```
