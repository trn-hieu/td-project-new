# B02 — Phạm vi duyệt, quyền và khóa sửa báo cáo

> Đọc `00-index.md` trước. Chỉ làm Task B02. Cần B01 đã merge.

### Task B02: `ReportApprovalScope`, `ReportApprovalAccess`, `ReportApprovalLocks`, `ReportLockInterceptor`

**Files:**
- Create:
  - `N model/dto/ReportApprovalScope.java`
  - `N model/dto/ReportApprovalContext.java`
  - `N service/ReportApprovalAccess.java`
  - `N service/ReportApprovalLocks.java`
  - `N config/ReportLockInterceptor.java`
  - `N config/WebMvcConfig.java`
- Modify:
  - `N common/constant/Positions.java`: thêm `REPORT_EDITOR_POSITIONS`.
  - `N config/SecurityConfig.java`: rule `"/api/appraisal-records/**", "/api/appraisal-reports/**"` (khoảng dòng 76–78) chuyển sang dùng `Positions.REPORT_EDITOR_POSITIONS`, không đổi hành vi.
  - `backend/CLAUDE.md`: thêm 2 dòng vào bảng helper (xem Step 9).
- Test:
  - `NT model/dto/ReportApprovalScopeTest.java`
  - `NT service/ReportApprovalAccessTest.java`
  - `NT service/ReportApprovalLocksTest.java`
  - `NT config/ReportLockInterceptorTest.java`
  - `NT controller/AppraisalReportBasicControllerLockTest.java`

**Interfaces:**
- Consumes (B01):
  - `AppraisalReportApprovalRepository.existsByAppraisalReportIdAndStatusInAndDeleteFlag`
  - `AppraisalReportApproval.Status`
  - `BranchCodes.HEAD_OFFICE`
- Produces:
  - `Positions.REPORT_EDITOR_POSITIONS`: `List<String>` gồm `BGD, TPTD, NVTD, NVPI, NVKT, NVKD`, đúng thứ tự trong `SecurityConfig` hiện tại.
  - `public record ReportApprovalScope(Mode mode, List<String> branches)`, với `enum Mode { ALL, BRANCHES, NONE }`:
    - `static ReportApprovalScope of(Employee employee)`: áp quy tắc ở mục 2 của "Quyết định đã chốt". Chuẩn hóa mã chi nhánh bằng `MsbRequestScope.branchCodes`. Kiểm tra admin bằng `"admin".equals(employee.getRole())`, dùng hằng có sẵn `Authorities.ROLE_ADMIN` nếu nó public; nếu không public thì dùng `AssignAuth.isAdmin`.
    - `boolean covers(String linkedBranch)`: ALL luôn trả true. BRANCHES trả true khi `linkedBranch` khác null, không rỗng và trùng một mã đã trim. NONE trả false.
    - `boolean isNone()`
  - `public record ReportApprovalContext(AppraisalReport report, AppraisalRequest request, String linkedBranch)` với `long reportId()`, `String contractId()`.
  - `ReportApprovalAccess` (`@Component`):
    - `ReportApprovalContext require(long reportId)`: báo cáo phải tồn tại và chưa xóa, dùng `ReportEditScope.requireReport(reportId)`. Yêu cầu đọc qua `AppraisalRequestRepository` theo `report.getAppraisalRequestId()`. Không có thì 404 "Không tìm thấy báo cáo".
    - `boolean isEditor(Employee e)`: `REPORT_EDITOR_POSITIONS` chứa `e.getPosition()`, hoặc `Positions.KSV` bằng `e.getPosition()`.
    - `boolean canApprove(Employee e, ReportApprovalContext ctx)`: `ReportApprovalScope.of(e).covers(ctx.linkedBranch())`.
    - `boolean canView(Employee e, ReportApprovalContext ctx)`: `canApprove(e, ctx) || isEditor(e)`.
    - `boolean canWithdraw(Employee e, AppraisalReportApproval a)`: `a.getSubmittedBy()` bằng mã của `e`, hoặc `e` là admin.
    - `ReportApprovalContext requireView(Employee e, long reportId)`: gọi `require`. Nếu `!canView` thì ném 404 với cùng message, để không lộ báo cáo có tồn tại hay không.
  - `ReportApprovalLocks` (`@Service`): `boolean isLocked(long reportId)` trả true khi trạng thái thuộc `{PENDING, APPROVED}` (dùng `exists…StatusIn`).
  - `ReportLockInterceptor implements HandlerInterceptor`. **Không** gắn `@Component`. Constructor nhận `ObjectProvider<ReportApprovalLocks>`.
    - `preHandle`: method là GET, HEAD hoặc OPTIONS thì cho qua.
    - Đọc `Map<String,String>` từ `request.getAttribute(HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE)`. Không có khóa `reportId`, hoặc giá trị không phải số, thì cho qua.
    - `provider.getIfAvailable()` trả null thì cho qua.
    - `isLocked` trả true thì ném `BusinessException(ErrorCode.CONFLICT, "Báo cáo đang chờ phê duyệt hoặc đã được duyệt, không thể chỉnh sửa")`.
  - `WebMvcConfig implements WebMvcConfigurer` (`@Configuration`):
    - đăng ký `new ReportLockInterceptor(provider)` cho `/api/appraisal-records/*/reports/**` và `/api/appraisal-reports/*/basic`;
    - `excludePathPatterns("/api/appraisal-reports/*/approval/**")`.
    - Javadoc giải thích lý do dùng `ObjectProvider`: `@WebMvcTest` nạp mọi `WebMvcConfigurer` nhưng không nạp `@Service`. Nhờ provider, các test controller hiện có không phải mock thêm bean.

- [ ] **Step 1: Viết `ReportApprovalScopeTest` (fail)**

  Dựng `Employee` bằng setter sẵn có (`setRole`, `setPosition`, `setBranch`). Các case:

  | Test | Dữ liệu | Kỳ vọng |
  |---|---|---|
  | `admin_la_all` | role `admin`, position `NVKD`, branch `HCM` | ALL; `covers("DN")` và `covers(null)` true |
  | `bgd_co_hn_co_khoang_trang_la_all` | BGD, `"HCM, HN "` | ALL |
  | `ksv_hnx_khong_phai_hn` | KSV, `"HNX"` | BRANCHES `["HNX"]`; `covers("HN")` false; `covers("HNX")` true |
  | `tptd_theo_chi_nhanh` | TPTD, `"HCM,DN"` | `covers("DN")` true; `covers("HN")` false; `covers(" ")` false; `covers(null)` false |
  | `nvkt_hn_la_none` | NVKT, `"HN"` | NONE; `covers("HN")` false (khác `AssignAuth.isGlobalAdmin`) |
  | `branch_null_khong_loi` | BGD, branch null | BRANCHES rỗng; `covers("HN")` false |
  | `nvtd_la_none` | NVTD, `"HN"` | NONE |

  Run: `cd backend && ./mvnw -q test -Dtest=ReportApprovalScopeTest`. Expected: FAIL.

- [ ] **Step 2: Viết `ReportApprovalScope`**

  Javadoc ghi rõ đây là khác biệt cố ý so với `AssignAuth.isGlobalAdmin` (hàm đó tính cả NVKT và dùng `contains`).

- [ ] **Step 3: Chạy test**

  Run: `./mvnw -q test -Dtest=ReportApprovalScopeTest`. Expected: PASS.

- [ ] **Step 4: Viết `ReportApprovalAccessTest` và `ReportApprovalLocksTest` (fail)**

  `ReportApprovalAccessTest` (Mockito, mock `ReportEditScope`, `AppraisalRequestRepository`):
  - `require_bao_cao_khong_ton_tai_404`
  - `can_view_editor_ngoai_pham_vi_van_xem`: NVTD xem báo cáo chi nhánh khác được.
  - `can_view_nvkt_ngoai_editor`: NVKT nằm trong editor list nên xem được. Test này khóa lại hành vi hiện tại.
  - `require_view_khong_quyen_404`: employee có position `X` (không hợp lệ) và không phải admin.
  - `can_approve_linked_branch_null_chi_all`: BGD chi nhánh HCM nhận false, admin nhận true.
  - `can_withdraw_nguoi_gui`, `can_withdraw_admin`, `can_withdraw_nguoi_khac_false`.

  `ReportApprovalLocksTest`: `pending_khoa`, `approved_khoa`, `rejected_khong_khoa`. Verify repository được gọi với tập `{PENDING, APPROVED}` và `"0"`.

  Run: `./mvnw -q test -Dtest='ReportApprovalAccessTest,ReportApprovalLocksTest'`. Expected: FAIL.

- [ ] **Step 5: Viết `ReportApprovalContext`, `ReportApprovalAccess`, `ReportApprovalLocks` và `Positions.REPORT_EDITOR_POSITIONS`; sửa `SecurityConfig` để dùng hằng mới**

  Rule `SecurityConfig` vẫn giữ nguyên tập authority đang có. Chỉ thay chuỗi thuần bằng hằng `Positions.REPORT_EDITOR_POSITIONS.toArray(String[]::new)`.

- [ ] **Step 6: Chạy test**

  Chạy lại lệnh ở Step 4. Expected: PASS.

- [ ] **Step 7: Viết test interceptor (fail)**

  `ReportLockInterceptorTest`, dùng `MockHttpServletRequest` và set `HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE`:
  - `put_bao_cao_khoa_409`: ném `BusinessException`, `getErrorCode() == CONFLICT`, message đúng chuỗi trong Interfaces.
  - `post_multipart_bao_cao_khoa_409`
  - `delete_bao_cao_khoa_409`: vars `{profileId=1, reportId=5}`.
  - `get_luon_cho_qua`: không gọi `isLocked`.
  - `khong_co_report_id_cho_qua`: vars `{profileId=1}`, tương ứng `/reports/form/...`.
  - `report_id_khong_phai_so_cho_qua`
  - `provider_rong_cho_qua`
  - `bao_cao_khong_khoa_cho_qua`

  `AppraisalReportBasicControllerLockTest` dùng `@WebMvcTest(AppraisalReportBasicController.class)`, chép phần `@Import` và `@TestPropertySource` của `AppraisalReportBasicControllerTest` hoặc `AppraisalReportWorkspaceControllerTest`, cộng `@Import(WebMvcConfig.class)`, `@MockitoBean ReportApprovalLocks`, và các `@MockitoBean` mà controller cần:
  - `put_khi_khoa_409`: JSON có `message` bằng chuỗi trên, service không được gọi.
  - `put_khi_khong_khoa_goi_service`

  Run: `./mvnw -q test -Dtest='ReportLockInterceptorTest,AppraisalReportBasicControllerLockTest'`. Expected: FAIL.

- [ ] **Step 8: Viết `ReportLockInterceptor` và `WebMvcConfig`, chạy test**

  Run: chạy lại lệnh ở Step 7 rồi `./mvnw test`. Toàn bộ `@WebMvcTest` cũ vẫn phải PASS mà không cần sửa. Expected: PASS.

- [ ] **Step 9: Cập nhật `backend/CLAUDE.md`**

  Thêm 2 dòng vào bảng "Rules":
  - "Phạm vi người duyệt báo cáo thẩm định (admin / BGD-TPTD-KSV có HN xem tất cả, còn lại theo chi nhánh)" → `ReportApprovalScope.of(employee).covers(linkedBranch)` (`model/dto`) | Dùng `AssignAuth.isGlobalAdmin` hay so `branch.contains("HN")`.
  - "Chặn sửa báo cáo đang chờ duyệt hoặc đã duyệt" → `ReportLockInterceptor` (`config/WebMvcConfig`, tự áp cho mọi API ghi có `{reportId}`) | Kiểm tra trạng thái duyệt trong từng service tab.

- [ ] **Step 10: Commit**

  ```bash
  git -C backend add -A
  git -C backend commit -m "feat(report-approval): phạm vi duyệt, quyền xem và khóa sửa báo cáo"
  ```
