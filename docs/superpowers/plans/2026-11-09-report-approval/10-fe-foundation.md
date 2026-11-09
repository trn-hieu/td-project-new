# F01: Nền frontend: API client, hằng số, thông báo

> Đọc `00-index.md` trước (Hợp đồng dùng chung: API, Frontend, Loại thông báo) và `frontend/CLAUDE.md`. Chỉ làm Task F01. FE không phụ thuộc BE: mọi test đều mock `@/lib/api`.

### Task F01: `api/report-approvals.ts`, `api/report-comments.ts`, `lib/report-approval.ts`, thông báo

**Files:**
- Create:
  - `F api/report-approvals.ts`
  - `F api/report-comments.ts`
  - `F lib/report-approval.ts`
- Modify:
  - `F api/notifications.ts`: thêm 5 loại, `refId`, `markNotificationRead`
  - `F lib/notification.ts`: `notificationHref`
  - `F components/layout/NotificationPopover.vue`: icon và đánh dấu đã đọc khi bấm
  - `F stores/notifications.ts`: thêm `markRead(id)` nếu chưa có
- Test:
  - `F api/__tests__/report-approvals.spec.ts`
  - `F api/__tests__/report-comments.spec.ts`
  - `F lib/__tests__/report-approval.spec.ts`
  - `F lib/__tests__/notification.spec.ts` (thêm case)
  - `F components/layout/__tests__/NotificationPopover.spec.ts` (thêm vào file sẵn có, chưa có thì tạo)

**Interfaces:**
- Consumes: `api` của `@/lib/api` (`get`, `post`, `download`), `PageData<T>` của `@/lib/http`, `MentionEmployee` của `@/api/msb-profiles`.
- Produces:
  - `api/report-approvals.ts` gồm type và hàm. Type map đúng record backend trong `00-index.md`, ngày giờ là `string`.

    ```ts
    export type ReportApprovalStatus = 'DRAFT' | 'PENDING' | 'APPROVED' | 'REJECTED' | 'WITHDRAWN'
    export type ReportWordStatus = 'QUEUED' | 'RUNNING' | 'READY' | 'FAILED'
    export interface EmployeeRef { employeeId: number; fullName: string }
    export interface ReportApprovalDetail { /* đủ trường như ReportApprovalDetail ở 00-index.md */ }
    export interface ReportApprovalListItem { /* như ReportApprovalListItem */ }
    export interface ApproverCandidate { employeeId: number; fullName: string; position: string; branchNames: string }
    export interface ReportApprovalWordLink { fileName: string; downloadUrl: string; viewerUrl: string }
    export interface ReportApprovalListQuery { status?: ReportApprovalStatus; branch?: string; keyword?: string; assignedToMe?: boolean; page: number; size: number }

    listReportApprovals(query: ReportApprovalListQuery, signal?: AbortSignal): Promise<PageData<ReportApprovalListItem>>   // GET /report-approvals
    getReportApproval(reportId: number, signal?: AbortSignal): Promise<ReportApprovalDetail>                              // GET /report-approvals/{id}
    approveReport(reportId: number): Promise<ReportApprovalDetail>                                                        // POST /report-approvals/{id}/approve
    rejectReport(reportId: number, reason: string): Promise<ReportApprovalDetail>                                         // POST …/reject {reason}
    withdrawReportApproval(reportId: number): Promise<ReportApprovalDetail>                                               // POST …/withdraw
    retryReportWord(reportId: number): Promise<ReportApprovalDetail>                                                      // POST …/word/retry
    getReportWordLink(reportId: number, signal?: AbortSignal): Promise<ReportApprovalWordLink>                            // GET …/word
    submitReportApproval(reportId: number, assigneeIds: number[]): Promise<ReportApprovalDetail>                          // POST /appraisal-reports/{id}/approval/submit
    getApproverCandidates(reportId: number, signal?: AbortSignal): Promise<ApproverCandidate[]>                           // GET /appraisal-reports/{id}/approval/approver-candidates
    ```

    Trong `ReportApprovalListQuery`, page bắt đầu từ 0, theo `PageData`.

  - `api/report-comments.ts`:

    ```ts
    export interface ReportComment { id: number; html: string; createdBy: number; createdByName: string; createdDate: string }
    listReportComments(reportId: number, page: number, signal?: AbortSignal): Promise<PageData<ReportComment>>   // GET /report-comments/{id}?page
    postReportComment(reportId: number, content: string): Promise<ReportComment>                               // POST /report-comments/{id} {content}
    searchReportMentions(search: string, signal?: AbortSignal): Promise<MentionEmployee[]>                     // GET /report-comments/mention-employees?search
    ```

  - `lib/report-approval.ts`:
    - `REPORT_APPROVAL_STATUS_LABELS: Record<ReportApprovalStatus, string>` và `reportApprovalStatusVariant(status): BadgeVariant`, lấy đúng theo bảng trong `00-index.md`. `BadgeVariant` lấy từ `@/components/ui/badge`, theo mẫu `lib/msb-profile.ts`.
    - `wordStatusLabel(status: ReportWordStatus | null): string`. Null trả `''`.
    - `isReportLocked(status: ReportApprovalStatus | undefined | null): boolean`: chỉ `PENDING` và `APPROVED` trả `true`.
    - `REPORT_APPROVER_AUTHORITIES: Authority[] = ['admin', 'BGD', 'TPTD', 'KSV']`.
    - `REPORT_APPROVAL_DETAIL_AUTHORITIES`: `APPRAISAL_RECORD_AUTHORITIES` cộng `'admin'` và `'KSV'`, đã bỏ trùng.
    - `REPORT_APPROVAL_STATUS_OPTIONS`: danh sách `{ value, label }` cho bộ lọc, gồm PENDING, APPROVED, REJECTED, WITHDRAWN (không có DRAFT).
  - `api/notifications.ts`:
    - `NotificationType` thêm `'REPORT_APPROVAL_REQUEST' | 'REPORT_APPROVED' | 'REPORT_REJECTED' | 'REPORT_WORD_FAILED' | 'REPORT_COMMENT_MENTION'`.
    - `NotificationItem` thêm `refId: number | null`.
    - `markNotificationRead(id: number): Promise<void>` gọi `POST /notifications/{id}/read`.
  - `lib/notification.ts`: `notificationHref(item)` trả `/report-approvals/${item.refId}` cho cả 5 loại mới. Nếu `refId` null thì trả `null` hoặc giá trị mặc định hiện tại, theo cách hàm đang xử lý loại không xác định (grep để chọn đúng).
  - `NotificationPopover.vue`:
    - Icon theo loại: `REPORT_APPROVAL_REQUEST` dùng `FileCheck`, `REPORT_APPROVED` dùng `CircleCheckBig`, `REPORT_REJECTED` dùng `CircleX`, `REPORT_WORD_FAILED` dùng `TriangleAlert`, `REPORT_COMMENT_MENTION` dùng `AtSign`.
    - Bấm một thông báo **chưa đọc** thì gọi `store.markRead(id)`. Store gọi `markNotificationRead`, đặt `read = true` và giảm `unreadCount`. Không `await` trước khi điều hướng; lỗi chỉ ghi `console.warn`, không hiện toast.

- [ ] **Step 1: Tạo nhánh**

  Làm theo mục "Nhánh làm việc" trong `00-index.md` (lệnh cho frontend).

- [ ] **Step 2: Viết test (fail)**

  - `api/__tests__/report-approvals.spec.ts`: theo mẫu `api/__tests__/appraisal-records.spec.ts`. Mỗi hàm một test, kiểm path, method và body. Hai case quan trọng:
    - `submit_gui_assignee_ids`: `api.post('/appraisal-reports/5/approval/submit', { assigneeIds: [1, 2] })`.
    - `list_truyen_query`: đủ `status`, `assignedToMe`, `page`, `size`.
  - `api/__tests__/report-comments.spec.ts`: `list_truyen_page`, `post_gui_content`, `search_truyen_search_va_signal`.
  - `lib/__tests__/report-approval.spec.ts`:
    - `nhan_va_variant_dung_bang`: cả 5 trạng thái.
    - `is_report_locked`: PENDING và APPROVED trả true; DRAFT, REJECTED, WITHDRAWN, undefined, null trả false.
    - `word_status_label`: QUEUED và RUNNING ra `'Đang tạo file Word…'`, FAILED ra `'Tạo file Word lỗi'`, READY ra `'Sẵn sàng'`, null ra `''`.
    - `detail_authorities_khong_trung`
  - `lib/__tests__/notification.spec.ts`: thêm `report_types_dan_toi_trang_duyet`, tức 5 loại với `refId: 7` đều cho `'/report-approvals/7'`.
  - `NotificationPopover.spec.ts`:
    - `bam_thong_bao_chua_doc_goi_mark_read`
    - `bam_thong_bao_da_doc_khong_goi`
    - `icon_theo_loai_report`: chỉ cần kiểm có render, không lỗi.

  Run: `cd frontend && npx vitest run src/api/__tests__/report-approvals.spec.ts src/api/__tests__/report-comments.spec.ts src/lib/__tests__/report-approval.spec.ts src/lib/__tests__/notification.spec.ts src/components/layout/__tests__/NotificationPopover.spec.ts`

  Expected: FAIL.

- [ ] **Step 3: Viết code theo phần Interfaces**

  Làm theo quy ước module API hiện có: hàm `export` thuần, type đặt cạnh hàm, mọi getter nhận `signal`.

- [ ] **Step 4: Chạy test**

  Run: lệnh ở Step 2. Expected: PASS.

- [ ] **Step 5: Lint, type-check, toàn bộ test**

  Run: `npm run lint && npm run type-check && npm run test:unit`. Expected: PASS. Chạy type-check **sau** lint, vì lint tự sửa file.

- [ ] **Step 6: Commit**

  ```bash
  git -C frontend add -A
  git -C frontend commit -m "feat(report-approval): API client, hằng số trạng thái duyệt và thông báo báo cáo"
  ```
