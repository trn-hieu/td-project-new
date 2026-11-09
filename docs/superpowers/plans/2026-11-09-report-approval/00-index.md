# Phê duyệt báo cáo thẩm định — Implementation Plan (index)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **Model cho subagent: CHỈ dùng `sonnet`.** Mọi lần gọi tool `Agent` (implementer, spec reviewer, code reviewer, final reviewer) đều truyền `model: "sonnet"`. **Mỗi task phải qua review** (spec review rồi code review) mới được đánh dấu xong.

**Goal:** Thêm luồng phê duyệt cho báo cáo thẩm định (`appraisal_reports`) gồm:
- gửi phê duyệt, có thể giao người duyệt;
- duyệt theo chức vụ và chi nhánh, mỗi chi nhánh cần 1 hoặc 2 người duyệt;
- từ chối (bắt buộc nhập lý do) hoặc rút lại;
- khóa sửa khi báo cáo đang chờ duyệt hoặc đã duyệt;
- file Word được tạo sẵn lúc gửi để người duyệt xem;
- khung trao đổi trên báo cáo, có @nhắc người và thông báo.

**Architecture:**
- **Backend**
  - Hai bảng mới: `appraisal_report_approvals` (1-1 với báo cáo, giữ trạng thái của lượt gửi hiện tại và file Word) và `appraisal_report_approval_actions` (lịch sử và phiếu duyệt).
  - Bảng `appraisal_report_comments` cho trao đổi.
  - Thêm cột cấu hình số người duyệt trên `linked_branch` và cột `ref_id` trên `notifications`.
  - File Word do worker nền `@Scheduled` tạo. Worker dùng chính bản ghi duyệt làm hàng đợi (mẫu `ImportJobWorker`) và gọi lại `AppraisalReportWordService.export`.
  - Khóa sửa chặn tập trung bằng một `HandlerInterceptor`.
- **Frontend**
  - Thanh trạng thái duyệt và khóa form trên màn sửa báo cáo.
  - Màn danh sách duyệt và màn chi tiết duyệt (xem Word bằng Office Online).
  - Khung trao đổi thu gọn được, dùng chung `MentionTextarea` tách ra từ chat MSB.

**Tech Stack:** Spring Boot 4 · Java 21 · Spring Data JPA (MySQL dùng chung với legacy, `ddl-auto=none`) · Flyway · JUnit 5/Mockito/AssertJ · `@MysqlDataJpaTest` · `@WebMvcTest` · AWS S3 · STOMP · Vue 3 + TS · shadcn-vue · vee-validate + zod · Vitest.

**Spec:** Mục "Quyết định đã chốt" bên dưới là spec. Các quyết định này đã được người dùng xác nhận ngày 2026-11-09.

## Viết tắt (dùng trong mọi file)

- **N** = `backend/src/main/java/com/thanhdo/tdprojectbe/`
- **NT** = `backend/src/test/java/com/thanhdo/tdprojectbe/`
- **NR** = `backend/src/main/resources/`
- **F** = `frontend/src/`

Trước khi viết code, implementer phải đọc `backend/CLAUDE.md`, `.claude/rules/jpa-spring.md`, `.claude/rules/design-principles.md`, `frontend/CLAUDE.md` và `frontend/DESIGN_SYSTEM.md` (các mục được dẫn trong task).

## Quyết định đã chốt (spec)

1. **Trạng thái của bản ghi duyệt** (cột `status`): `PENDING` (chờ duyệt), `APPROVED` (đã duyệt), `REJECTED` (bị từ chối), `WITHDRAWN` (đã rút lại).
   - Báo cáo chưa có bản ghi duyệt thì API trả `DRAFT`. Đây là giá trị ảo, không lưu DB.
   - Sửa được khi `DRAFT`, `REJECTED`, `WITHDRAWN`. Khóa khi `PENDING`, `APPROVED`.
   - `APPROVED` khóa vĩnh viễn. Không có chức năng mở khóa.
2. **Người được duyệt (`ReportApprovalScope`)**
   - `employee.role = 'admin'`: phạm vi ALL.
   - Chức vụ `BGD`, `TPTD`, `KSV` có mã `HN` trong `MsbRequestScope.branchCodes(employee)`: phạm vi ALL. Phải khớp **nguyên mã sau trim**, không dùng `String.contains`.
   - Chức vụ `BGD`, `TPTD`, `KSV` không có mã `HN`: phạm vi BRANCHES, chỉ các mã chi nhánh của mình.
   - Mọi trường hợp khác: phạm vi NONE.
   - Phạm vi được so với `p_appraisal_request.linked_branch` của yêu cầu chứa báo cáo. Nếu `linked_branch` null hoặc rỗng thì chỉ người có phạm vi ALL được duyệt.
   - Đây là khác biệt **cố ý** so với `AssignAuth.isGlobalAdmin`: hàm đó tính cả NVKT và so chuỗi con. Ghi lý do vào javadoc.
3. **Số người duyệt**
   - Cột mới `linked_branch.report_approvals_required` TINYINT NOT NULL DEFAULT 1. Giá trị khác 2 coi là 1.
   - Lúc gửi duyệt, giá trị này được chụp vào `required_approvals` của bản ghi. Đổi cấu hình chi nhánh sau đó không ảnh hưởng lượt đang chờ.
   - Không làm màn cấu hình.
4. **Duyệt**
   - Người gửi **được** tự duyệt nếu có quyền.
   - Một người chỉ duyệt một lần trong một lượt.
   - Lượt có đủ `required_approvals` người khác nhau duyệt thì chuyển `APPROVED`.
   - Chỉ duyệt được khi file Word ở trạng thái `READY`.
5. **Từ chối**
   - Bắt buộc lý do: trim, dài 1 đến 1000 ký tự.
   - Được từ chối cả khi lượt đã có 1/2 người duyệt. Không cần chờ file Word.
   - Kết quả: báo cáo về `REJECTED`, xóa file Word (async), báo cáo sửa được và gửi lại được (tạo lượt mới).
6. **Rút lại**
   - Chỉ người gửi hoặc admin được rút.
   - Chỉ rút được khi báo cáo đang `PENDING` **và chưa ai duyệt** trong lượt. Nếu đã có người duyệt thì trả 409 "Báo cáo đã có người duyệt, không thể rút lại".
   - Kết quả: báo cáo về `WITHDRAWN`, xóa file Word (async).
7. **Gửi phê duyệt**
   - Ai vào được API sửa báo cáo thì gửi được. Quyền này theo rule `/api/appraisal-reports/**` sẵn có trong `SecurityConfig`.
   - Chỉ gửi được khi báo cáo đang `DRAFT`, `REJECTED` hoặc `WITHDRAWN`. Mỗi lần gửi tăng `round` thêm 1.
   - Có thể giao 0 đến 10 người duyệt. Người được giao phải có phạm vi phủ chi nhánh của báo cáo, nếu không trả 400.
   - Người được giao nhận thông báo, trừ chính người gửi.
   - Việc giao **không** chặn người khác có quyền duyệt.
8. **File Word**
   - Mỗi báo cáo chỉ có một file, gắn với lượt hiện tại.
   - File do **worker nền** tạo bằng cách gọi lại `AppraisalReportWordService.export(reportId)`. Không viết lại logic xuất Word.
   - Key S3: `report/approval/{reportId}/{round}/{uuid}/{fileName}`.
   - Tạo file thử tối đa 3 lần. Hết 3 lần mà vẫn lỗi thì chuyển `FAILED`, gửi thông báo cho người gửi, và trên giao diện có nút "Tạo lại file".
   - Khi từ chối hoặc rút lại, object S3 bị xóa bằng `@Async @TransactionalEventListener(AFTER_COMMIT)`.
   - Worker tạo xong mà lượt đã đổi thì xóa file vừa tạo, không gắn vào bản ghi.
   - Người duyệt xem file qua `OfficeFiles.viewerUrl(S3LinkService.presign(key))` và tải về bằng link presign.
9. **Khóa sửa**
   - Server chặn bằng `HandlerInterceptor` cho mọi request **không phải GET/HEAD/OPTIONS** có biến đường dẫn `{reportId}`. Phạm vi gồm:
     - `/api/appraisal-records/{profileId}/reports/{reportId}/**`
     - `DELETE /api/appraisal-records/{profileId}/reports/{reportId}`
     - `PUT /api/appraisal-reports/{reportId}/basic`
   - Không chặn `/api/appraisal-reports/*/approval/**`.
   - Lỗi trả 409 "Báo cáo đang chờ phê duyệt hoặc đã được duyệt, không thể chỉnh sửa".
   - FE khóa form bằng `<fieldset :disabled>` kèm một `Alert`.
10. **Trao đổi**
    - Bảng mới `appraisal_report_comments`.
    - Người dùng gõ văn bản thuần. Người được nhắc viết dạng token `@[id]`. HTML được dựng bằng `ExchangeMessageHtml.render`.
    - Người được nhắc nhận thông báo, trừ người viết.
    - Không đính kèm file, không sửa hay xóa tin.
    - Không realtime: tải lại khi mở, sau khi gửi và khi bấm "Làm mới". Mỗi trang 30 tin, có nút "Xem tin cũ hơn".
    - Bình luận được cả khi báo cáo đang khóa.
    - Ai xem được chi tiết duyệt thì xem và gửi được tin.
11. **Thông báo**
    - Cột mới `notifications.ref_id` BIGINT NULL, chứa mã báo cáo.
    - Thêm 5 loại thông báo. Cả 5 đều dẫn tới `/report-approvals/{refId}`.
    - Endpoint mới `POST /api/notifications/{id}/read`. FE gọi endpoint này khi người dùng bấm một thông báo bất kỳ trong chuông.

## Global Constraints

- Migration nằm ở `NR/db/migration/`, tên dạng `V<yyyyMMddHHmmss>__<snake_case>.sql`, cú pháp MySQL, có comment tiếng Việt ở đầu file. Bảng mới có `delete_flag CHAR(1) NOT NULL DEFAULT '0'`. Không sửa migration đã chạy.
- Test chạy với `spring.flyway.enabled=false`. Migration phải được chạy trên DB dev trước khi chạy test repository.
- Truy cập dữ liệu ưu tiên JPA (derived query, rồi JPQL, rồi Specification, rồi projection). Không dùng native SQL trong plan này.
- Mọi message trả về client viết bằng tiếng Việt và báo lỗi qua `BusinessException(ErrorCode, message)`. Lỗi sai trạng thái dùng `ErrorCode.CONFLICT` (409), lỗi quyền dùng `FORBIDDEN` (403), không thấy hoặc không được xem dùng `NOT_FOUND` (404).
- Giới hạn:
  - `MAX_ASSIGNEES = 10`
  - lý do từ chối tối đa `1000` ký tự
  - `MAX_WORD_ATTEMPTS = 3`
  - `COMMENT_PAGE_SIZE = 30`
  - nội dung bình luận tối đa `10_000` ký tự, nhắc tối đa `50` người (dùng chung hằng với chat MSB, không copy giá trị)
- Không đụng `td-project-legacy`. Không sửa `SecurityConfig` ngoài việc thay danh sách chức vụ bằng hằng số (B02).
- FE: tuân thủ mọi quy tắc trong `frontend/CLAUDE.md`, gồm token màu, `IconButton`, `AppModal`, `useConfirm`, `FormActions`, `Badge`, `Alert`, `DataPagination` và component form ở `@/components/form/`.

## Review Focus

Mỗi dòng dưới đây đã có test ở task sở hữu.

1. **Từ chối hoặc rút lại trong lúc worker đang tạo Word.** File tạo xong phải bị xóa, không được gắn vào lượt mới hay vào bản ghi đã REJECTED. Test ở B05: `complete_xoa_file_khi_luot_da_doi`, `complete_xoa_file_khi_da_tu_choi`.
2. **Duyệt trùng.** Hai người duyệt gần như cùng lúc ở chi nhánh cần 2 người, hoặc một người bấm Duyệt hai lần: không được đếm sai, lần thứ hai trả 409. Dùng khóa `PESSIMISTIC_WRITE` kết hợp UNIQUE. Test ở B03 (`approve_lan_hai_cung_nguoi_409`) và B01 (`unique_chan_duyet_trung`).
3. **Định dạng `employee.branch`.** Các giá trị `"HCM, HN "`, `"HNX"` hoặc null: chỉ khớp nguyên mã sau trim. Test ở B02 (`ReportApprovalScopeTest`).
4. **Ghi vào báo cáo đang khóa.** Báo cáo bị khóa trong lúc FE vẫn mở tab cũ rồi bấm Lưu, hoặc xóa báo cáo đang chờ duyệt: server trả 409 và FE hiện toast bằng message của server. Test ở B02 (`ReportLockInterceptorTest`, `AppraisalReportBasicControllerLockTest`) và F03.
5. **Form khóa vẫn phải xem được.** Khi form bị khóa, người dùng vẫn chuyển được tab cấp 1 và cấp 2. `fieldset` chỉ bọc nội dung panel, không bọc `TabsList`. Test ở F03 (`khoa_van_chuyen_tab_duoc`).

## Hợp đồng dùng chung

### DB

| File | Nội dung |
|---|---|
| `V20261110090000__add_report_approvals_required_to_linked_branch.sql` | `ALTER TABLE linked_branch ADD COLUMN report_approvals_required TINYINT NOT NULL DEFAULT 1` |
| `V20261110090100__create_appraisal_report_approvals.sql` | Hai bảng bên dưới |
| `V20261110090200__create_appraisal_report_comments.sql` | Bảng bình luận |
| `V20261110090300__add_ref_id_to_notifications.sql` | `ALTER TABLE notifications ADD COLUMN ref_id BIGINT NULL` (comment: hệ thống cũ không đọc cột này) |

`appraisal_report_approvals`:

| Cột | Kiểu |
|---|---|
| `id` | BIGINT AUTO_INCREMENT PK |
| `appraisal_report_id` | BIGINT NOT NULL, **UNIQUE** `uq_report_approvals_report` |
| `status` | VARCHAR(20) NOT NULL |
| `round` | INT NOT NULL |
| `required_approvals` | TINYINT NOT NULL |
| `submitted_by` | BIGINT NOT NULL |
| `submitted_at` | DATETIME NOT NULL |
| `decided_by` | BIGINT NULL |
| `decided_at` | DATETIME NULL |
| `reject_reason` | TEXT NULL |
| `word_status` | VARCHAR(20) NULL |
| `word_attempts` | INT NOT NULL DEFAULT 0 |
| `word_s3_key` | VARCHAR(500) NULL |
| `word_file_name` | VARCHAR(255) NULL |
| `word_generated_at` | DATETIME NULL |
| `word_error` | TEXT NULL |
| `created` | DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP |
| `updated` | DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP |
| `delete_flag` | CHAR(1) NOT NULL DEFAULT '0' |

Index thêm: `KEY idx_report_approvals_word (word_status, id)` và `KEY idx_report_approvals_status (status, submitted_at)`.

`appraisal_report_approval_actions`:
- Cột: `id`, `appraisal_report_id` BIGINT NOT NULL, `round` INT NOT NULL, `action` VARCHAR(20) NOT NULL, `employee_id` BIGINT NOT NULL, `reason` TEXT NULL, `created` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP, `delete_flag`.
- Giá trị `action`: `SUBMIT`, `ASSIGN`, `APPROVE`, `REJECT`, `WITHDRAW`.
- `UNIQUE KEY uq_report_approval_actions (appraisal_report_id, round, action, employee_id)`.
- `KEY idx_report_approval_actions_employee (employee_id, action)`.

`appraisal_report_comments`:
- Cột: `id`, `appraisal_report_id` BIGINT NOT NULL, `content` MEDIUMTEXT NOT NULL (HTML đã render), `created_by` BIGINT NOT NULL, `created_date` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP, `updated_date` DATETIME NULL, `delete_flag`.
- `KEY idx_report_comments_report (appraisal_report_id, id)`.

### Tên dùng chung (backend)

| Thành phần | Tên |
|---|---|
| Hằng số mã chi nhánh | `N common/constant/BranchCodes.HEAD_OFFICE = "HN"` |
| Chức vụ được sửa báo cáo | `N common/constant/Positions.REPORT_EDITOR_POSITIONS` (`List<String>`: BGD, TPTD, NVTD, NVPI, NVKT, NVKD) |
| Entity | `N model/entity/AppraisalReportApproval` (enum lồng `Status`, `WordStatus { QUEUED, RUNNING, READY, FAILED }`), `AppraisalReportApprovalAction` (enum lồng `Action`), `AppraisalReportComment` |
| Repository | `AppraisalReportApprovalRepository`, `AppraisalReportApprovalActionRepository`, `AppraisalReportCommentRepository` |
| Phạm vi | `N model/dto/ReportApprovalScope` |
| Quyền | `N service/ReportApprovalAccess`, `ReportApprovalContext` (record trong `model/dto`) |
| Khóa | `N service/ReportApprovalLocks`, `N config/ReportLockInterceptor`, `N config/WebMvcConfig` |
| Thông báo | `N service/ReportApprovalNotifier` |
| Xóa file | `N service/ReportWordDiscarded` (record), `N service/ReportWordCleanupListener` |
| Lệnh | `N service/ReportApprovalCommandService` |
| Truy vấn | `N service/ReportApprovalQueryService` |
| Worker | `N service/ReportApprovalWordProcessor`, `N schedule/ReportApprovalWordWorker` |
| Trao đổi | `N service/ReportCommentService`, `N controller/ReportCommentController` |
| Controller | `N controller/ReportApprovalSubmitController` (`/api/appraisal-reports/{reportId}/approval`), `N controller/ReportApprovalController` (`/api/report-approvals`) |

### Loại thông báo

Các loại thông báo là hằng chuỗi trong `NotificationItem`. Chuỗi giữ nguyên như bảng dưới.

Ký hiệu trong cột nội dung:
- `{hđ}` là `AppraisalRequest.contractId`. Nếu null thì dùng `#` + reportId.
- `{Tên}` là `Employee.getFullName()` của người thực hiện.

| Loại | Người nhận | Nội dung |
|---|---|---|
| `REPORT_APPROVAL_REQUEST` | người được giao | `{Tên} gửi phê duyệt báo cáo thẩm định số hợp đồng {hđ}` |
| `REPORT_APPROVED` (chưa đủ người) | người gửi | `{Tên} đã duyệt báo cáo thẩm định số hợp đồng {hđ} ({n}/{m})` |
| `REPORT_APPROVED` (đủ người) | người gửi | `Báo cáo thẩm định số hợp đồng {hđ} đã được phê duyệt` |
| `REPORT_REJECTED` | người gửi | `{Tên} từ chối báo cáo thẩm định số hợp đồng {hđ}: {lý do, cắt còn 200 ký tự}` |
| `REPORT_WORD_FAILED` | người gửi | `Không tạo được file Word báo cáo thẩm định số hợp đồng {hđ}, vui lòng tạo lại` |
| `REPORT_COMMENT_MENTION` | người được nhắc | `{Tên} đã nhắc bạn trong báo cáo thẩm định số hợp đồng {hđ}` |

Khi người nhận là chính người thực hiện thì không gửi. Ví dụ: người gửi tự duyệt thì không nhận `REPORT_APPROVED`.

Thông báo được tạo bằng factory mới `Notification.about(String type, Long fromId, Long toId, Long refId, String content)`. Field `appraisal_request_id` để null.

### API

Quyền được kiểm trong service. `/api/report-approvals/**` và `/api/report-comments/**` thuộc nhánh `anyRequest().authenticated()` sẵn có, nên không phải sửa `SecurityConfig`. Nhờ vậy admin không có chức vụ duyệt vẫn gọi được.

"Được xem" (`canView`) nghĩa là thỏa một trong hai điều kiện:
- có phạm vi duyệt phủ chi nhánh của báo cáo;
- hoặc `position` thuộc `REPORT_EDITOR_POSITIONS` hay là KSV.

| API | Quyền | Kết quả |
|---|---|---|
| `GET /api/appraisal-reports/{reportId}/approval/approver-candidates` | rule sửa báo cáo | `List<ApproverCandidate>` |
| `POST /api/appraisal-reports/{reportId}/approval/submit` body `SubmitApprovalRequest{assigneeIds}` | rule sửa báo cáo | `ApiResponse.ok(ReportApprovalDetail, "Đã gửi phê duyệt báo cáo")` |
| `GET /api/report-approvals?status&branch&keyword&assignedToMe&page&size` | phạm vi khác NONE, nếu không thì 403 | `Page<ReportApprovalListItem>`. Bỏ trống `status` thì mặc định là PENDING. Sắp `submittedAt DESC`, `size` tối đa 100. |
| `GET /api/report-approvals/{reportId}` | `canView`, nếu không thì 404 | `ReportApprovalDetail` |
| `POST /api/report-approvals/{reportId}/approve` | `canApprove` | `ApiResponse.ok(detail, "Đã duyệt báo cáo")` |
| `POST /api/report-approvals/{reportId}/reject` body `RejectApprovalRequest{reason}` | `canApprove` | `ApiResponse.ok(detail, "Đã từ chối báo cáo")` |
| `POST /api/report-approvals/{reportId}/withdraw` | người gửi hoặc admin | `ApiResponse.ok(detail, "Đã rút lại yêu cầu phê duyệt")` |
| `POST /api/report-approvals/{reportId}/word/retry` | người gửi, admin hoặc `canApprove` | `ApiResponse.ok(detail, "Đang tạo lại file Word")` |
| `GET /api/report-approvals/{reportId}/word` | `canView`; file phải `READY`, nếu không thì 404 | `ReportApprovalWordLink{fileName, downloadUrl, viewerUrl}` |
| `GET /api/report-comments/{reportId}?page` | `canView` | `Page<ReportCommentItem>`, sắp `id DESC`, `size` cố định 30 |
| `POST /api/report-comments/{reportId}` body `ReportCommentRequest{content}` | `canView` | `ReportCommentItem` |
| `GET /api/report-comments/mention-employees?search` | đăng nhập | `List<MentionEmployee>` |
| `POST /api/notifications/{id}/read` | chủ thông báo | `ApiResponse.ok(null)` |

DTO (`N model/dto/`, record):

```text
ApproverCandidate(Long employeeId, String fullName, String position, String branchNames)
EmployeeRef(Long employeeId, String fullName)
ReportApprovalWord(String status, String fileName, LocalDateTime generatedAt)
ReportApprovalVote(Long employeeId, String fullName, LocalDateTime at)
ReportApprovalHistoryItem(int round, String action, Long employeeId, String fullName, String reason, LocalDateTime created)
ReportApprovalDetail(Long reportId, Long requestId, String contractId, String linkedBranch, String branchName,
    String status, Integer round, Integer requiredApprovals, int approvedCount,
    EmployeeRef submittedBy, LocalDateTime submittedAt, String rejectReason, EmployeeRef decidedBy, LocalDateTime decidedAt,
    ReportApprovalWord word, List<EmployeeRef> assignees, List<ReportApprovalVote> approvals,
    List<ReportApprovalHistoryItem> history,
    boolean canSubmit, boolean canWithdraw, boolean canApprove, boolean canReject, boolean canRetryWord, boolean canOpenReport)
ReportApprovalListItem(Long reportId, String contractId, String linkedBranch, String branchName, String submittedByName,
    LocalDateTime submittedAt, int round, long approvedCount, int requiredApprovals, String status, String wordStatus,
    boolean assignedToMe)
ReportApprovalFilter(String status, String branch, String keyword, boolean assignedToMe)
ReportApprovalWordLink(String fileName, String downloadUrl, String viewerUrl)
ReportCommentItem(Long id, String html, Long createdBy, String createdByName, LocalDateTime createdDate)
SubmitApprovalRequest(@Size(max = 10) List<@NotNull Long> assigneeIds)
RejectApprovalRequest(@NotBlank @Size(max = 1000) String reason)
ReportCommentRequest(@NotBlank @Size(max = 10000) String content)
```

Ghi chú:
- `approvals` và `assignees` chỉ tính lượt hiện tại.
- `history` là mọi action, sắp từ cũ đến mới.
- `word` là null khi `DRAFT` hoặc khi file đã bị xóa.
- `ReportApprovalDetail` ở trạng thái `DRAFT` có `round = null` và `requiredApprovals` là số chi nhánh đang cấu hình.

### Frontend

| Trạng thái | Nhãn | Badge variant |
|---|---|---|
| `DRAFT` | Chưa gửi duyệt | `neutral` |
| `PENDING` | Chờ duyệt | `warning` |
| `APPROVED` | Đã duyệt | `success` |
| `REJECTED` | Bị từ chối | `destructive` |
| `WITHDRAWN` | Đã rút lại | `neutral` |

- Nhãn trạng thái file Word: `QUEUED` và `RUNNING` hiện "Đang tạo file Word…", `READY` hiện "Sẵn sàng", `FAILED` hiện "Tạo file Word lỗi".
- Route:
  - `report-approvals`, path `report-approvals`, `meta.authorities = REPORT_APPROVER_AUTHORITIES = ['admin','BGD','TPTD','KSV']`.
  - `report-approval-detail`, path `report-approvals/:reportId(\\d+)`, `meta.authorities = REPORT_APPROVAL_DETAIL_AUTHORITIES` = `APPRAISAL_RECORD_AUTHORITIES` cộng `'admin'` và `'KSV'`, bỏ trùng.
- Menu: "Duyệt báo cáo", icon `CircleCheckBig`, đặt trong `F components/layout/nav-items.ts`.

## Danh sách file kế hoạch và thứ tự task

Các task **cùng đợt có tập file rời nhau**, nên chạy song song được. Các đợt chạy nối tiếp nhau. Luồng BE và luồng FE chạy song song; FE chỉ dựa vào **Hợp đồng dùng chung** và mock API trong test.

| File | Task | Đợt | Nội dung | Phụ thuộc |
|---|---|---|---|---|
| `01-be-foundation.md` | B01 | B1 | Migration, entity, repository, `Notification.refId`/`about`, `ReportApprovalNotifier`, sự kiện và listener xóa file Word, API đánh dấu đã đọc, `S3StorageService.upload(byte[])`, cấu hình scheduler | — |
| `02-be-scope-lock.md` | B02 | B2 | `ReportApprovalScope`, `ReportApprovalAccess`, khóa sửa | B01 |
| `04-be-word-worker.md` | B05 | B2 | Worker tạo Word | B01 |
| `03-be-approval.md` | B03 | B3 | Lệnh: gửi, rút, duyệt, từ chối, tạo lại file | B02 |
| `05-be-comments.md` | B06 | B3 | Trao đổi, nhắc người, thông báo | B02 |
| `03-be-approval.md` | B04 | B4 | Truy vấn (danh sách, chi tiết, ứng viên, link Word) và hai controller | B03 |
| `10-fe-foundation.md` | F01 | F1 | API client, `lib/report-approval.ts`, thông báo ở FE | — |
| `11-fe-comments.md` | F02 | F2 | `MentionTextarea` (tách từ `ChatComposer`), `ReportCommentPanel` | F01 |
| `12-fe-report-edit.md` | F03 | F3 | Màn sửa báo cáo: thanh trạng thái, gửi và rút, khóa form, khung trao đổi | F01, F02 |
| `13-fe-approver-pages.md` | F04 | F3 | Màn danh sách duyệt và màn chi tiết duyệt | F01, F02 |
| `90-final-review.md` | Z01 | cuối | Review toàn nhánh, chạy toàn bộ test, E2E thủ công | tất cả |

## Nhánh làm việc

`backend` và `frontend` là git submodule. B01 và F01 tạo nhánh `feat/report-approval` từ `origin/develop` mới nhất:

```bash
git -C backend -c http.sslBackend=schannel fetch origin develop && git -C backend checkout -b feat/report-approval origin/develop
git -C frontend -c http.sslBackend=schannel fetch origin develop && git -C frontend checkout -b feat/report-approval origin/develop
```

- Lúc lập plan, `origin/develop` của backend ở `08f2957`, của frontend ở `11821e6`.
- Số dòng được dẫn trong plan đo trên hai commit này, nên **luôn grep lại trước khi sửa**.
- Mọi commit chạy bằng `git -C backend …` hoặc `git -C frontend …`.
- **Không push**, không cập nhật con trỏ submodule ở repo gốc khi chưa được người dùng đồng ý.

## Quy trình cho controller (phiên chính), bắt buộc

Dùng superpowers:subagent-driven-development. Với **mỗi task**:

1. **Implementer**: gọi `Agent({ subagent_type: "general-purpose", model: "sonnet", prompt })`.
   - Prompt gồm ba phần:
     - (a) đường dẫn `00-index.md` và file chứa task, yêu cầu đọc cả hai;
     - (b) chỉ thị: "Chỉ làm Task Xnn; TDD theo các Step; commit trong submodule theo Step cuối";
     - (c) khối Interfaces của các task trước mà task này dùng.
   - Implementer không được tự mở rộng phạm vi.
   - Gặp điều plan không quyết định được thì **dừng và báo lại**. Ví dụ: cột không tồn tại, migration chưa chạy được, cần sửa file của task khác cùng đợt.
2. **Spec reviewer**: gọi `Agent({ model: "sonnet" })`.
   - Đối chiếu diff với phần Files, Interfaces, Steps của task, với "Quyết định đã chốt" và với Review Focus.
   - Kết luận APPROVED, hoặc liệt kê lỗi kèm file:dòng.
3. **Code reviewer**: gọi `Agent({ model: "sonnet" })`.
   - Đối chiếu với `backend/CLAUDE.md`, `.claude/rules/*.md`, `frontend/CLAUDE.md`, `frontend/DESIGN_SYSTEM.md`.
   - Chạy lại lệnh kiểm tra của task.
4. Lỗi được xác nhận thì gửi lại cho implementer (SendMessage tới cùng agent) để sửa, rồi review lại bước đã fail. Chỉ đánh dấu `[x]` khi cả hai reviewer APPROVED.

**Chạy song song trong một đợt.** Mỗi implementer làm trong worktree riêng của submodule, tạo từ đầu nhánh lúc bắt đầu đợt:

```bash
git -C backend worktree add ../backend-wt-B05 -b wt/B05 feat/report-approval
```

Hết đợt, merge từng nhánh `wt/<ID>` vào `feat/report-approval` theo thứ tự ID rồi xóa worktree. Merge có conflict nghĩa là chia file sai: dừng và báo lại. Không muốn chạy song song thì chạy tuần tự theo ID ngay trong `backend/` hoặc `frontend/`.

**Hết mỗi đợt:**
- BE: `cd backend && ./mvnw test`
- FE: `cd frontend && npm run lint && npm run type-check && npm run test:unit`

## Ngoài phạm vi

- Hệ thống cũ vẫn sửa được báo cáo đang bị khóa, vì hai hệ thống dùng chung DB.
- Không có màn cấu hình số người duyệt: sửa thẳng cột `linked_branch.report_approvals_required`.
- Không mở khóa báo cáo đã duyệt.
- Trao đổi không realtime, không đính kèm file, không sửa hay xóa tin.
- Duyệt báo cáo không đổi `appraisal_profile.status` hay `appraisal_profile.approve_status`.
