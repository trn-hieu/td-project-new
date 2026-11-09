# B05: Worker tạo file Word khi gửi phê duyệt

> Đọc `00-index.md` trước (mục 8 của "Quyết định đã chốt"). Chỉ làm Task B05. Task này cần B01 đã được merge và chạy song song được với B02.

### Task B05: `ReportApprovalWordProcessor` + `ReportApprovalWordWorker`

**Files:**
- Create:
  - `N service/ReportApprovalWordProcessor.java`
  - `N model/dto/WordClaim.java`
  - `N schedule/ReportApprovalWordWorker.java`
- Test:
  - `NT service/ReportApprovalWordProcessorTest.java`
  - `NT schedule/ReportApprovalWordWorkerTest.java`

**Interfaces:**
- Consumes:
  - B01:
    - `AppraisalReportApprovalRepository`: `findFirstByWordStatusAndDeleteFlagOrderByIdAsc`, `findForUpdate`, `requeueRunningWords`
    - các method `claimWord`, `completeWord`, `failWord` của entity
    - `S3StorageService.upload(String, byte[], String)`
    - `ReportWordDiscarded`
    - `ReportApprovalNotifier.wordFailed`
  - Code sẵn có:
    - `AppraisalReportWordService.export(long reportId)` trả `ExportedReport(fileName, content)`. Method này `@Transactional(readOnly = true)`, được gọi qua proxy nên tự mở transaction đọc riêng.
    - `AppraisalReportRepository`, `AppraisalRequestRepository` (lấy `contractId` cho thông báo).
- Produces:
  - `public record WordClaim(long reportId, int round)`
  - `ReportApprovalWordProcessor` (`@Service`, không `@Transactional` ở cấp lớp):
    - `@Transactional int requeueRunning()`: gọi `requeueRunningWords()`.
    - `@Transactional(propagation = REQUIRES_NEW) Optional<WordClaim> claimNext()`: đọc dòng `QUEUED` cũ nhất, khóa lại bằng `findForUpdate`, kiểm lại vẫn `QUEUED`, rồi `claimWord()`.
    - `void generate(WordClaim claim)`: **không** mở transaction. Gọi `export`, rồi `upload(key, content, DOCX)`, rồi `self.complete(...)`.
      - `key = "report/approval/" + reportId + "/" + round + "/" + UUID + "/" + fileName`
      - `DOCX = "application/vnd.openxmlformats-officedocument.wordprocessingml.document"`
      - Gọi `complete` qua chính bean (inject `@Lazy` self, hoặc tách thành lớp con `ReportApprovalWordStore`) để `REQUIRES_NEW` có hiệu lực. Chọn cách tách lớp nếu lớp quá 150 dòng.
    - `@Transactional(propagation = REQUIRES_NEW) void complete(WordClaim claim, String key, String fileName)`:
      - Khóa dòng. Nếu vẫn `PENDING`, `round == claim.round()` và `wordStatus == RUNNING` thì `completeWord(key, fileName, now)`.
      - Nếu không thì `publishEvent(new ReportWordDiscarded(key))`. Listener của B01 có `fallbackExecution`, nên vẫn chạy sau commit.
    - `@Transactional(propagation = REQUIRES_NEW) void fail(WordClaim claim, Exception error)`:
      - Khóa dòng. Nếu lượt đã đổi (không `PENDING`, khác `round`, hoặc không `RUNNING`) thì bỏ qua.
      - Nếu không thì gọi `failWord(shortMessage(error))`. Trả `true` (đã `FAILED`) thì `notifier.wordFailed(reportId, contractId, submittedBy)`.
  - `ReportApprovalWordWorker` (`@Component`, package-private): chép cấu trúc `N schedule/ImportJobWorker.java`. Cụ thể:
    - `@Scheduled(fixedDelayString = "${report.approval.word.poll-delay-ms:5000}") void poll()`
    - Lần poll đầu gọi `requeueRunning`.
    - Lặp `claimNext`, rồi `generate`. Nếu `generate` ném `RuntimeException` thì gọi `fail`. Mọi lỗi chỉ ghi log, không làm dừng vòng lặp.
    - Javadoc ghi rõ: đọc bản ghi duyệt như một hàng đợi; chạy một luồng; báo cáo nhiều ảnh có thể mất vài chục giây.

- [ ] **Step 1: Viết `ReportApprovalWordProcessorTest` (fail)**

  Mockito, `Clock.fixed`. Dựng bản ghi bằng các method của entity B01 (`create`, `submit`, `claimWord`).

  - `claim_next_chuyen_queued_sang_running`: kết quả `WordClaim(5, 1)`, `wordStatus == RUNNING`.
  - `claim_next_khong_co_tra_empty`
  - `generate_upload_dung_key_va_content_type`: bắt key và kiểm `key.startsWith("report/approval/5/1/") && key.endsWith("/BC.docx")`, content type là DOCX.
  - `complete_gan_file_khi_luot_con_hieu_luc`: `READY`, `wordS3Key == key`, `wordFileName == "BC.docx"`, không publish.
  - `complete_xoa_file_khi_luot_da_doi`: bản ghi bị reject rồi submit lại (round 2), claim ở round 1. Kết quả: publish `ReportWordDiscarded(key)`, `wordStatus` của round 2 vẫn `QUEUED`.
  - `complete_xoa_file_khi_da_tu_choi`: bản ghi `REJECTED`, publish discard.
  - `fail_lan_1_quay_lai_queued`: không gọi notifier.
  - `fail_lan_3_thanh_failed_va_bao_nguoi_gui`: verify `wordFailed(5L, "HĐ1", 9L)`.
  - `fail_khi_luot_da_doi_bo_qua`
  - `upload_loi_khong_de_lai_running`: `upload` ném lỗi, `generate` ném lại để worker gọi `fail`. Kiểm `complete` không được gọi.

  Run: `cd backend && ./mvnw -q test -Dtest=ReportApprovalWordProcessorTest`. Expected: FAIL.

- [ ] **Step 2: Viết `WordClaim` và `ReportApprovalWordProcessor`, chạy test**

  Chạy lại lệnh ở Step 1. Expected: PASS.

- [ ] **Step 3: Viết `ReportApprovalWordWorkerTest` (fail)**

  Mẫu: test của `ImportJobWorker` nếu có, không có thì viết bằng Mockito.
  - `poll_dau_tien_requeue_mot_lan`: gọi `poll()` hai lần, `requeueRunning` chỉ chạy 1 lần.
  - `poll_xu_ly_het_hang_doi`: `claimNext` trả hai claim rồi empty, nên `generate` được gọi 2 lần.
  - `generate_loi_goi_fail_va_tiep_tuc`
  - `requeue_loi_thu_lai_lan_sau`

  Run: `./mvnw -q test -Dtest=ReportApprovalWordWorkerTest`. Expected: FAIL.

- [ ] **Step 4: Viết `ReportApprovalWordWorker`, chạy test**

  Chạy lại lệnh ở Step 3, rồi chạy `./mvnw test`. Expected: PASS.

- [ ] **Step 5: Kiểm tra thủ công trên DB dev (không bắt buộc nếu không có S3 dev)**

  1. Chạy app.
  2. Chèn tay một dòng `appraisal_report_approvals` (`status='PENDING'`, `word_status='QUEUED'`, `round=1`) cho một báo cáo có thật.
  3. Trong khoảng 5 đến 60 giây, `word_status` phải thành `READY`, có `word_s3_key`, và object tồn tại trên bucket.

  Không có S3 dev thì ghi rõ "chưa kiểm thủ công" trong báo cáo.

- [ ] **Step 6: Commit**

  ```bash
  git -C backend add -A
  git -C backend commit -m "feat(report-approval): worker nền tạo file Word báo cáo khi gửi phê duyệt"
  ```
