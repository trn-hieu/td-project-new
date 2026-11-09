# F03: Màn sửa báo cáo: trạng thái duyệt, gửi/rút, khóa form, khung trao đổi

> Đọc trước:
> - `00-index.md`: các mục 1, 6, 7, 9 trong "Quyết định đã chốt" và Review Focus 4–5.
> - `frontend/CLAUDE.md`.
> - `frontend/DESIGN_SYSTEM.md`: §10.2 nút, §10.12–10.13 Badge/Alert, §12.3 FormActions, §15 AppModal.
>
> Chỉ làm Task F03. Cần F01 và F02 đã merge. Task này chạy song song được với F04: hai task có tập file rời nhau.

### Task F03: `useReportLock`, `ApprovalStatusBar`, `SubmitApprovalModal`, khóa `GroupWorkspace` / edit-basic

**Files:**
- Create:
  - `F composables/useReportLock.ts`
  - `F components/report-approval/ApprovalStatusBar.vue`
  - `F components/report-approval/SubmitApprovalModal.vue`
- Modify:
  - `F views/appraisal-record/AppraisalReportEditView.vue`: header ở khoảng L84–98.
  - `F components/appraisal-report/edit/GroupWorkspace.vue`
  - `F components/appraisal-report/edit/images/ImageSection.vue`: thêm hành vi `readonly || locked`.
  - `F views/appraisal-record/AppraisalReportEditBasicView.vue`
- Test:
  - `F composables/__tests__/useReportLock.spec.ts`
  - `F components/report-approval/__tests__/ApprovalStatusBar.spec.ts`
  - `F components/report-approval/__tests__/SubmitApprovalModal.spec.ts`
  - `F views/__tests__/AppraisalReportEditView.spec.ts` (thêm case)
  - `F views/__tests__/AppraisalReportEditBasicView.spec.ts` (thêm case, chưa có file thì tạo)

**Interfaces:**
- Consumes:
  - F01: `getReportApproval`, `submitReportApproval`, `withdrawReportApproval`, `getApproverCandidates`, `ReportApprovalDetail`, `REPORT_APPROVAL_STATUS_LABELS`, `reportApprovalStatusVariant`, `isReportLocked`.
  - F02: `ReportCommentPanel`.
  - Có sẵn: `AppModal`, `FormActions`, `MultiSelect`, `confirmDialog`, `toast`, `Badge`, `Alert`.
- Produces:
  - `useReportLock.ts`: `provideReportLock(locked: Ref<boolean> | ComputedRef<boolean>): void` và `useReportLock(): Readonly<Ref<boolean>>`. Không có provider thì trả `ref(false)`. Dùng `InjectionKey` theo mẫu `useDirtyGuard.ts`.
  - `ApprovalStatusBar.vue`
    - Props `{ detail: ReportApprovalDetail }`. Emits `changed: [detail: ReportApprovalDetail]`.
    - Hiển thị:
      - Một `Badge` trạng thái.
      - Khi khác `DRAFT`: dòng "Lượt {round} · Đã duyệt {approvedCount}/{requiredApprovals}" và "Gửi bởi {submittedBy.fullName} lúc {submittedAt}".
      - Khi `REJECTED`: `Alert variant="destructive"` tiêu đề "Báo cáo bị từ chối", nội dung là `rejectReason` và người/ngày quyết định.
    - Nút:
      - `canSubmit`: `Button` variant `default`, icon `Send`, nhãn "Gửi phê duyệt". Bấm thì mở `SubmitApprovalModal`.
      - `canWithdraw`: `Button` variant `outline`, icon `Undo2`, nhãn "Rút lại". Bấm thì gọi `confirmDialog({ title: 'Rút lại yêu cầu phê duyệt?', tone: 'warning', confirmLabel: 'Rút lại yêu cầu', action: () => withdrawReportApproval(id) })`, thành công thì `toast.success('Đã rút lại yêu cầu phê duyệt')` và emit `changed`.
      - Khác `DRAFT`: `RouterLink` dạng `Button` variant `outline`, nhãn "Xem trang duyệt", tới `/report-approvals/{reportId}`.
  - `SubmitApprovalModal.vue`
    - Props `{ reportId: number; open: boolean }`. Emits `update:open`, `submitted: [detail]`.
    - Dùng `AppModal` với `size="md"`, `persistent`, `:busy`.
    - Nội dung là `MultiSelect` "Giao người duyệt (không bắt buộc)". Option lấy từ `getApproverCandidates` khi mở modal: value là `String(employeeId)`, label là `"{fullName} · {position} · {branchNames}"`. Tối đa 10 người, vượt thì báo lỗi trường "Chỉ giao tối đa 10 người duyệt" (vee-validate + zod, `FormField`). Kèm ghi chú "Người khác có quyền vẫn duyệt được báo cáo này."
    - Footer dùng `FormActions`: "Hủy" (outline) rồi "Gửi phê duyệt" (submit).
    - Gọi `submitReportApproval(reportId, ids.map(Number))`. Thành công thì `toast.success('Đã gửi phê duyệt báo cáo')`, emit `submitted`, đóng modal. Lỗi thì `toast.error` kèm `errorMessage`.
    - Có trạng thái loading/error khi tải danh sách ứng viên.
  - `AppraisalReportEditView.vue`
    - Tải `getReportApproval(reportId)` song song với workspace. Lỗi tải trạng thái duyệt **không** chặn trang: hiện `Alert` cảnh báo nhỏ và coi như `DRAFT`, nhưng không cho gửi.
    - `provideReportLock(computed(() => isReportLocked(approval.value?.status)))`.
    - `ApprovalStatusBar` đặt ngay dưới header.
    - Nút "Sửa thông tin cơ bản" bị ẩn khi bị khóa.
    - Bố cục: `xl:grid xl:grid-cols-[minmax(0,1fr)_24rem] gap-4`. Workspace ở cột trái, `ReportCommentPanel` ở cột phải (`xl:sticky xl:top-4 self-start`). Dưới `xl` thì panel nằm dưới workspace.
    - Sau khi `ApprovalStatusBar` emit `changed` hoặc modal emit `submitted`, cập nhật `approval` từ detail trả về.
  - `GroupWorkspace.vue`
    - Bọc **nội dung bên trong từng `TabsContent`**, không bọc `TabsList`/`TabsTrigger`, trong `<fieldset :disabled="locked" class="m-0 flex min-w-0 flex-col gap-6 border-0 p-0">`.
    - Khi khóa, phía trên nội dung hiện `<Alert variant="warning" role="status">` với nội dung "Báo cáo đang chờ phê duyệt hoặc đã được duyệt nên không thể chỉnh sửa."
    - Tab "Xem trước báo cáo" (`ReportPreviewTab`) không bọc fieldset, để vẫn dùng được nút "Làm mới".
  - `ImageSection.vue`: `const effectiveReadonly = computed(() => props.readonly || locked.value)`. Khi khóa thì ẩn vùng upload và nút xóa.
  - `AppraisalReportEditBasicView.vue`: tải `getReportApproval`. Khi khóa thì bọc form trong `fieldset :disabled`, hiện `Alert` như trên và ẩn nút submit.

- [ ] **Step 1: Viết test (fail)**

  - `useReportLock.spec.ts`:
    - `khong_provider_tra_false`
    - `provider_true_tra_true_va_phan_ung`
  - `ApprovalStatusBar.spec.ts`:
    - `draft_hien_gui_phe_duyet_khong_hien_xem_trang_duyet`
    - `pending_co_1_duyet_khong_hien_rut_lai`: `canWithdraw = false`.
    - `rejected_hien_ly_do`
    - `rut_lai_xac_nhan_roi_emit_changed`: mock `@/composables/useConfirm` để chạy `action`.
    - `badge_dung_variant`
  - `SubmitApprovalModal.spec.ts`:
    - `tai_ung_vien_khi_mo`
    - `gui_khong_chon_ai_goi_api_mang_rong`
    - `gui_chon_hai_nguoi_goi_api_so`: `[1, 2]`.
    - `qua_10_nguoi_bao_loi_khong_goi_api`
    - `loi_server_toast`
  - `AppraisalReportEditView.spec.ts`:
    - `pending_khoa_input`: một `input` trong tab Pháp lý (hoặc tab được render mặc định) có `disabled` hoặc nằm trong `fieldset[disabled]`.
    - `khoa_van_chuyen_tab_duoc`: bấm `TabsTrigger` cấp 1 và cấp 2 vẫn đổi tab. Kiểm `TabsTrigger` không nằm trong `fieldset[disabled]`.
    - `draft_khong_khoa`
    - `an_sua_thong_tin_co_ban_khi_khoa`
    - `loi_tai_trang_thai_khong_chan_trang`
    - `hien_khung_trao_doi`
    - `server_409_khi_luu_hien_toast_message`: tab đang mở với `DRAFT`, API lưu của tab trả `ApiError` 409 "Báo cáo đang chờ phê duyệt…", kiểm `toast.error` có description chứa message đó. Chỉ thêm test này nếu tab có sẵn đường báo lỗi bằng toast. Nếu tab đã có test lỗi lưu thì chỉ cần xác nhận hành vi sẵn có.
  - `AppraisalReportEditBasicView.spec.ts`:
    - `khoa_an_nut_luu_va_hien_alert`

  Run: `cd frontend && npx vitest run src/composables/__tests__/useReportLock.spec.ts src/components/report-approval src/views/__tests__/AppraisalReportEditView.spec.ts src/views/__tests__/AppraisalReportEditBasicView.spec.ts`

  Expected: FAIL.

- [ ] **Step 2: Viết `useReportLock.ts`, `ApprovalStatusBar.vue`, `SubmitApprovalModal.vue`**

- [ ] **Step 3: Sửa `AppraisalReportEditView.vue`, `GroupWorkspace.vue`, `ImageSection.vue`, `AppraisalReportEditBasicView.vue`**

  Grep các component tab có cờ riêng (`canSave`, `readonly`, `disabled`) như `EvaluationResultTab.vue:49`. Nếu nút hay vùng thao tác nào không phải phần tử form native (ví dụ vùng kéo thả, `div` có `@click`) thì kết hợp thêm `useReportLock()`. Liệt kê các chỗ đã sửa trong báo cáo.

- [ ] **Step 4: Chạy test**

  Run: lệnh ở Step 1. Expected: PASS.

- [ ] **Step 5: Lint, type-check, toàn bộ test, build**

  Run: `npm run lint && npm run type-check && npm run test:unit && npm run build`. Expected: PASS.

- [ ] **Step 6: Commit**

  ```bash
  git -C frontend add -A
  git -C frontend commit -m "feat(report-approval): gửi/rút phê duyệt, khóa sửa và khung trao đổi trên màn sửa báo cáo"
  ```
