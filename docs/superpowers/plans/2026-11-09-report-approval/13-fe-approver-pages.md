# F04: Màn danh sách duyệt và màn chi tiết duyệt báo cáo

> Đọc trước:
> - `00-index.md`: các mục 2, 4, 5, 6, 8 của "Quyết định đã chốt" và phần Frontend của Hợp đồng dùng chung.
> - `frontend/CLAUDE.md`.
> - `frontend/DESIGN_SYSTEM.md`: §3, §6 bố cục full width; §10.2 nút; §11 bảng; §12–13 form; §15 modal.
>
> Chỉ làm Task F04. Task này cần F01 và F02 đã merge. Chạy song song được với F03.

### Task F04: `ReportApprovalListView`, `ReportApprovalDetailView`, `RejectApprovalModal`, `ApprovalHistory`, `ApprovalWordViewer`, route + menu

**Files:**
- Create:
  - `F views/report-approval/ReportApprovalListView.vue`
  - `F views/report-approval/ReportApprovalDetailView.vue`
  - `F components/report-approval/RejectApprovalModal.vue`
  - `F components/report-approval/ApprovalHistory.vue`
  - `F components/report-approval/ApprovalWordViewer.vue`
- Modify:
  - `F router/index.ts`: thêm 2 route, đặt cạnh nhóm `update-requests` (khoảng L175)
  - `F components/layout/nav-items.ts`: thêm mục "Duyệt báo cáo"
- Test:
  - `F views/__tests__/ReportApprovalListView.spec.ts`
  - `F views/__tests__/ReportApprovalDetailView.spec.ts`
  - `F components/report-approval/__tests__/RejectApprovalModal.spec.ts`
  - `F components/report-approval/__tests__/ApprovalWordViewer.spec.ts`

**Interfaces:**
- Consumes:
  - Từ F01: `listReportApprovals`, `getReportApproval`, `approveReport`, `rejectReport`, `withdrawReportApproval`, `retryReportWord`, `getReportWordLink`, `REPORT_APPROVAL_STATUS_LABELS`, `reportApprovalStatusVariant`, `wordStatusLabel`, `REPORT_APPROVAL_STATUS_OPTIONS`, `REPORT_APPROVER_AUTHORITIES`, `REPORT_APPROVAL_DETAIL_AUTHORITIES`.
  - Từ F02: `ReportCommentPanel`.
  - Có sẵn trong dự án:
    - `DataPagination`, `SelectBox`, `SearchSelect`, `Checkbox`, `Input`, `Table*`, `Badge`, `Alert`, `AppModal`, `FormActions`, `FormField`, `confirmDialog`, `toast`, `api.saveBlob` (không dùng, link presign mở trực tiếp).
    - Danh sách chi nhánh cho bộ lọc: grep API danh sách chi nhánh đang dùng ở bộ lọc báo cáo (vd. `api/linked-branches.ts` hoặc các option `branches` của màn báo cáo) và dùng lại. Không tìm thấy thì dùng ô nhập mã chi nhánh và ghi lại trong báo cáo.
- Produces:
  - **Route** (con của layout chính, cùng nhóm với `update-requests`):
    - `{ path: 'report-approvals', name: 'report-approvals', component: ReportApprovalListView, meta: { authorities: REPORT_APPROVER_AUTHORITIES, breadcrumbs: [{ label: 'Duyệt báo cáo' }] } }`
    - `{ path: 'report-approvals/:reportId(\\d+)', name: 'report-approval-detail', component: ReportApprovalDetailView, meta: { authorities: REPORT_APPROVAL_DETAIL_AUTHORITIES, breadcrumbs: [{ label: 'Duyệt báo cáo', to: '/report-approvals' }, { label: 'Chi tiết' }] } }`
    - Breadcrumb theo đúng dạng `meta.breadcrumbs` đang dùng; grep để khớp kiểu dữ liệu.
  - **Menu**: mục `{ label: 'Duyệt báo cáo', to: '/report-approvals', icon: CircleCheckBig, authorities: REPORT_APPROVER_AUTHORITIES }`, đặt cạnh "Phê duyệt yêu cầu" theo cấu trúc `NavSubItem`/`NavItem` hiện có.
  - **`ReportApprovalListView.vue`**:
    - Bố cục full width: `flex w-full flex-col gap-4 px-4 py-4 sm:py-6`.
    - Bộ lọc: `SelectBox` trạng thái (mặc định `PENDING`), chi nhánh, `Input` số hợp đồng (debounce 300 ms), `Checkbox` "Được giao cho tôi".
    - Đồng bộ bộ lọc với query URL (`status`, `branch`, `keyword`, `mine`, `page`), theo cách các màn danh sách hiện có đang làm; grep để chọn đúng composable. Mỗi khi bộ lọc đổi thì `page` về 1.
    - Bảng có các cột: STT (`text-center tabular-nums`), Số HĐ (link tới chi tiết), Chi nhánh, Người gửi, Ngày gửi (`text-center tabular-nums`), Đã duyệt `{approvedCount}/{requiredApprovals}` (`text-center tabular-nums`), Trạng thái (`Badge`), File Word (`wordStatusLabel`), Được giao (`Badge variant="info"` "Giao cho tôi" khi `assignedToMe`).
    - Có trạng thái loading, empty ("Không có báo cáo nào"), error (`Alert` + "Thử lại"). Phân trang bằng `DataPagination`, size 20. Mỗi lần tải dùng `AbortController` riêng.
  - **`ReportApprovalDetailView.vue`**:
    - Header: nút "Quay lại", tiêu đề `Duyệt báo cáo thẩm định {contractId}`, `Badge` trạng thái. Nhóm nút bên phải (`ml-auto`), theo đúng thứ tự:
      1. "Mở báo cáo": `outline`, icon `ExternalLink`, tới `/appraisal-reports/{reportId}`; hiện khi `canOpenReport`.
      2. "Rút lại": `outline`, icon `Undo2`, dùng `confirmDialog` tone `warning`; hiện khi `canWithdraw`.
      3. "Từ chối": `warning`, icon `X`, mở `RejectApprovalModal`; hiện khi `canReject`.
      4. "Duyệt": `success`, icon `Check`, dùng `confirmDialog({ title: 'Duyệt báo cáo?', tone: 'success', confirmLabel: 'Duyệt báo cáo', action: () => approveReport(id) })`; hiện khi `canApprove`.
    - Sau mỗi hành động thành công: `toast.success(message)`, cập nhật `detail` từ giá trị trả về.
    - Bố cục `xl:grid xl:grid-cols-[minmax(0,1fr)_24rem] gap-4`.
      - Cột trái gồm:
        - Thẻ thông tin: chi nhánh, người gửi và thời điểm gửi, lượt, `Đã duyệt n/m` kèm danh sách người đã duyệt (tên · giờ), người được giao, lý do từ chối nếu có.
        - `ApprovalWordViewer`.
        - `ApprovalHistory`.
      - Cột phải là `ReportCommentPanel`.
    - Có trạng thái loading, 404 ("Không tìm thấy báo cáo hoặc bạn không có quyền xem", kèm nút "Quay lại") và lỗi khác (`Alert` + "Thử lại").
  - **`ApprovalWordViewer.vue`**:
    - Props `{ reportId: number; word: ReportApprovalDetail['word']; status: ReportApprovalStatus; canRetry: boolean }`.
    - Emits `changed: [detail]`, `refresh: []`.
    - Hiển thị theo trạng thái file:
      - `word` null: `Alert variant="info"` với nội dung "Chưa có file Word (báo cáo chưa gửi duyệt hoặc đã bị từ chối/rút lại)".
      - `QUEUED`/`RUNNING`: spinner và chữ "Đang tạo file Word…". Cứ 5 giây emit `refresh` (`useIntervalFn`). Dừng khi trạng thái đổi hoặc khi unmount.
      - `READY`: gọi `getReportWordLink`, rồi hiện `<iframe :src="viewerUrl" title="Xem file Word báo cáo" class="h-[…]">`. Chiều cao lấy theo class sẵn có của `ReportPreviewTab.vue`, không dùng giá trị tùy ý mới. Kèm `Button` `outline` "Tải file Word" (icon `Download`), là thẻ `<a :href="downloadUrl" target="_blank" rel="noopener">`.
      - `FAILED`: `Alert variant="destructive"` với nội dung "Tạo file Word lỗi". Khi `canRetry` thì thêm nút `outline` "Tạo lại file", gọi `retryReportWord` rồi emit `changed`.
  - **`ApprovalHistory.vue`**:
    - Props `{ items: ReportApprovalHistoryItem[] }`.
    - Danh sách theo thời gian, mỗi dòng: "Lượt {round}", nhãn hành động, tên người, giờ, lý do (nếu có).
    - Nhãn hành động: `SUBMIT` → "Gửi phê duyệt", `APPROVE` → "Duyệt", `REJECT` → "Từ chối", `WITHDRAW` → "Rút lại".
    - Danh sách rỗng thì hiện "Chưa có lịch sử".
  - **`RejectApprovalModal.vue`**:
    - Props `{ reportId: number; open: boolean }`. Emits `update:open`, `rejected: [detail]`.
    - `AppModal` `size="md"`, `persistent`, `:busy`, `before-close`.
    - Form vee-validate + zod: `reason` là `Textarea` bắt buộc.
      - Message khi trống (sau trim): "Vui lòng nhập lý do từ chối".
      - Message khi quá 1000 ký tự: "Lý do tối đa 1000 ký tự".
    - Footer `FormActions`: nút "Hủy", rồi nút submit "Từ chối báo cáo" (submit dùng variant `default` theo §12.3). Nếu DESIGN_SYSTEM §10.2 quy định nút xác nhận hành động tiêu cực phải dùng variant khác thì theo DESIGN_SYSTEM và ghi lại.
    - Gọi `rejectReport(reportId, reason.trim())`. Thành công thì `toast.success('Đã từ chối báo cáo')`, emit `rejected` và đóng modal.

- [ ] **Step 1: Viết test (fail)**

  - `ReportApprovalListView.spec.ts` (mock `@/api/report-approvals`, router memory):
    - `mac_dinh_loc_pending_page_0`
    - `doi_trang_thai_ve_trang_1`
    - `tick_duoc_giao_cho_toi_gui_assigned_to_me`
    - `hien_badge_giao_cho_toi`
    - `rong_hien_khong_co_bao_cao`
    - `loi_hien_alert_thu_lai`
  - `ReportApprovalDetailView.spec.ts`:
    - `an_hien_nut_theo_can`: `canApprove` false thì không có nút "Duyệt"; `canReject` true thì có nút "Từ chối".
    - `duyet_xac_nhan_goi_api_va_cap_nhat`
    - `tu_choi_mo_modal`
    - `rut_lai_goi_api`
    - `khong_tim_thay_404_hien_thong_bao`
    - `hien_khung_trao_doi`
  - `RejectApprovalModal.spec.ts`:
    - `ly_do_rong_bao_loi_khong_goi_api`
    - `ly_do_chi_khoang_trang_bao_loi`
    - `ly_do_1001_bao_loi`
    - `gui_trim_ly_do`
    - `loi_server_toast_giu_modal`
  - `ApprovalWordViewer.spec.ts`:
    - `queued_poll_5_giay_va_dung_khi_unmount` (fake timers)
    - `ready_hien_iframe_va_link_tai`
    - `failed_co_nut_tao_lai_khi_can_retry`
    - `word_null_hien_thong_tin`

  Run: `cd frontend && npx vitest run src/views/__tests__/ReportApprovalListView.spec.ts src/views/__tests__/ReportApprovalDetailView.spec.ts src/components/report-approval/__tests__/RejectApprovalModal.spec.ts src/components/report-approval/__tests__/ApprovalWordViewer.spec.ts`

  Expected: FAIL.

- [ ] **Step 2: Viết `RejectApprovalModal`, `ApprovalHistory`, `ApprovalWordViewer`**

- [ ] **Step 3: Viết `ReportApprovalListView`, `ReportApprovalDetailView`, thêm route và menu**

- [ ] **Step 4: Chạy test**

  Run: lệnh ở Step 1. Expected: PASS.

- [ ] **Step 5: Lint, type-check, toàn bộ test, build**

  Run: `npm run lint && npm run type-check && npm run test:unit && npm run build`. Expected: PASS.

- [ ] **Step 6: Commit**

  ```bash
  git -C frontend add -A
  git -C frontend commit -m "feat(report-approval): màn danh sách và chi tiết duyệt báo cáo"
  ```
