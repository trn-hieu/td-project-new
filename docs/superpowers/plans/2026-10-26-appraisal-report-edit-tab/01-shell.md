# 01 — Khung màn hình (T01–T02)

> Đọc `00-index.md` trước. Subagent: `model: "sonnet"`, có review.

## Hành vi hệ thống cũ

- `L controller/AppraisalReportController.java:252-294` (`/edit-tab/{id}`): nạp báo cáo, yêu cầu, nhóm tài sản sắp theo `groupIndex` (null cuối; nhóm đã xóa bị lọc bởi `@Where delete_flag=0`). Không có nhóm → "Chưa có nhóm tài sản.".
- `LR templates/demo/demo-report-step-2.html:131-251`: tab cấp 1 mỗi nhóm, nhãn `"Tài sản " + n` khi `type == SINGLE`, ngược lại `"Nhóm TS " + n` (n = vị trí 1-based). Tab cuối "Xem trước báo cáo" — **bỏ**.
- Tab cấp 2 theo thứ tự: Pháp lý (`legal`), "Mô tả đất" / "Mô tả phương tiện" khi CAR (`land`), Mô tả CTXD (`ctxd`, **chỉ LAND**), Thông tin so sánh (`compare`), Bảng điều chỉnh (`panel`), Kết quả thẩm định (`results`). Tab mặc định `land`.
- `LR static/js/demo-report-step-2-tabs.js`: tab nạp lười, nhớ tab qua hash `#asset-inner-{groupId}-{tab}`; lưu xong reload cả trang.

## Thiết kế

- URL: `/appraisal-records/:id/reports/:reportId?group=<groupId>&tab=<tab>`. Cấp 1 `useRouteTab('group', groupIds, { resets: ['tab'] })` với `<TabsList variant="line">`; cấp 2 `useRouteTab('tab', tabsOf(group))` với `TabsList` mặc định, mặc định `land`. `ctxd` trên nhóm không phải LAND → về `land` (useRouteTab tự fallback vì không nằm trong `values`).
- `Tabs` cấp 2 để mặc định `unmount-on-hide` (tab bị hủy khi ẩn → mở lại là tải mới, thay cờ `data-loaded`). Tabs cấp 1 cũng vậy.
- Chặn rời tab chưa lưu: `provideDirtyGuard()` ở view; tab con gọi `useDirtyTab(isDirty)`. Đổi tab cấp 1/cấp 2 và rời route (`onBeforeRouteLeave`) đều `await guard.confirmLeave()` → `confirmDiscard(...)` của `useConfirm` khi có tab bẩn.

---

### Task T01: `ReportEditScope` + API workspace

**Files:**
- Create: `N service/ReportEditScope.java`, `N service/ReportWorkspaceService.java`, `N controller/ReportWorkspaceController.java`, `N model/dto/ReportWorkspace.java`, `N model/dto/ReportWorkspaceGroup.java`
- Modify: `N repository/AppraisalAssetGroupRepository.java` (thêm `findByAppraisalReportIdAndDeleteFlag(Long reportId, …)` — kiểu cờ theo entity), `N repository/AppraisalReportRepository.java` nếu cần finder
- Test: `NT service/ReportEditScopeTest.java`, `NT service/ReportWorkspaceServiceTest.java`, `NT controller/ReportWorkspaceControllerTest.java`, `NT repository/AppraisalAssetGroupRepositoryTest.java` (thêm case)

**Interfaces:**
- Consumes: `AppraisalProfileRequests.requestOf(long profileId)` (`N service/AppraisalProfileRequests.java:25`), `AssetGroupType`, `AppraisalAssetType`.
- Produces:
  - `@Service ReportEditScope`: `AppraisalReport requireReport(long profileId, long reportId)` — báo cáo chưa xóa và `appraisalRequestId == requestOf(profileId).getId()`, ngược lại `BusinessException(ErrorCode.NOT_FOUND, "Không tìm thấy báo cáo thẩm định")`; `AppraisalAssetGroup requireGroup(long profileId, long reportId, long groupId)` — gọi `requireReport`, nhóm chưa xóa và thuộc báo cáo, ngược lại NOT_FOUND "Không tìm thấy nhóm tài sản"; `List<AppraisalAssetGroup> sortedGroups(long reportId)`.
  - `record ReportWorkspaceGroup(long id, Integer groupIndex, String label, String type, int assetType, String landValuationMethod)`.
  - `record ReportWorkspace(long reportId, long profileId, long requestId, List<ReportWorkspaceGroup> groups)`.
  - `GET /api/appraisal-records/{profileId}/reports/{reportId}/workspace` → `ReportWorkspace`.

- [ ] **Step 0:** `git -C backend switch -c feat/appraisal-report-edit-tab`
- [ ] **Step 1: Test thất bại** `ReportEditScopeTest` (Mockito): `báo_cáo_khác_yêu_cầu_trả_404`, `báo_cáo_đã_xóa_trả_404`, `nhóm_thuộc_báo_cáo_khác_trả_404`, `nhóm_hợp_lệ_trả_về`. `ReportWorkspaceServiceTest`: `nhãn_tài_sản_và_nhóm_ts` (SINGLE ở vị trí 1 → "Tài sản 1", GROUP ở vị trí 2 → "Nhóm TS 2"), `groupIndex_null_xếp_cuối`. Repository test: finder bỏ nhóm `delete_flag` đã xóa.
- [ ] **Step 2: Chạy** `./mvnw -q test -Dtest='ReportEditScopeTest,ReportWorkspaceServiceTest'` → FAIL.
- [ ] **Step 3: Cài đặt** theo Interfaces; nhãn tính theo vị trí sau khi sắp xếp (không theo `groupIndex`).
- [ ] **Step 4: Controller test** `@WebMvcTest`: 200 với JSON `groups[0].label`; 404 khi scope ném NOT_FOUND.
- [ ] **Step 5: Chạy** các test trên + `AppraisalAssetGroupRepositoryTest` → PASS.
- [ ] **Step 6: Thêm dòng bảng helper** vào `backend/CLAUDE.md`: `| Kiểm tra hồ sơ → báo cáo → nhóm tài sản trên API màn báo cáo thẩm định | ReportEditScope.requireReport/requireGroup/sortedGroups | Tự so appraisalReportId/appraisalRequestId trong service tab mới |`
- [ ] **Step 7: Commit** `git -C backend add -A && git -C backend commit -m "feat: ReportEditScope và API workspace màn báo cáo thẩm định"`

### Task T02: Khung màn 2 cấp tab

**Files:**
- Modify: `F views/appraisal-record/AppraisalReportEditView.vue` (thay placeholder), `F composables/useRouteTab.ts` (nhận `values` là `MaybeRefOrGetter<readonly T[]>` vì danh sách nhóm nạp bất đồng bộ; giữ tương thích chỗ gọi cũ), `F router/index.ts` (cập nhật comment route: không còn "tạm để trống")
- Create: `F api/report-edit/workspace.ts`, `F composables/useDirtyGuard.ts`, `F lib/report-edit/tabs.ts`, `F components/appraisal-report/edit/GroupWorkspace.vue`, `F components/appraisal-report/edit/TabPlaceholder.vue` (tab chưa port: "Đang chuyển đổi", dùng tạm đến khi task tab tương ứng thay)
- Test: `F views/__tests__/AppraisalReportEditView.spec.ts` (viết lại), `F composables/__tests__/useRouteTab.spec.ts` (thêm case getter), `F composables/__tests__/useDirtyGuard.spec.ts`, `F lib/report-edit/__tests__/tabs.spec.ts`

**Interfaces:**
- Consumes: `GET …/workspace` (T01).
- Produces:
  - `api/report-edit/workspace.ts`: `reportEditPath(profileId, reportId)`, `interface ReportWorkspaceGroup { id: number; groupIndex: number | null; label: string; type: string; assetType: 1 | 2 | 3; landValuationMethod: string | null }`, `interface ReportWorkspace {…}`, `getReportWorkspace(profileId, reportId, signal?)`.
  - `lib/report-edit/tabs.ts`: `ASSET_TYPE = { LAND: 1, APARTMENT: 2, CAR: 3 } as const`, `type GroupTab = 'legal' | 'land' | 'ctxd' | 'compare' | 'panel' | 'results'`, `tabsOf(assetType): GroupTab[]`, `tabLabel(tab, assetType): string` ("Mô tả phương tiện" khi CAR).
  - `useDirtyGuard.ts`: `provideDirtyGuard(): { confirmLeave(): Promise<boolean> }`, `useDirtyTab(isDirty: Ref<boolean>): void` (tự gỡ đăng ký khi unmount).
  - `GroupWorkspace.vue` props `{ profileId, reportId, group }`: render tab cấp 2; mỗi tab là một `<component :is>` lấy từ map `tab → component` trong file này (task tab sau thay `TabPlaceholder` bằng component thật — chỉ sửa một dòng map).

- [ ] **Step 0:** `git -C frontend switch -c feat/appraisal-report-edit-tab`
- [ ] **Step 1: Test thất bại:** `tabs.spec.ts` (`LAND có ctxd`, `CAR nhãn Mô tả phương tiện`, `APARTMENT không có ctxd`); `useDirtyGuard.spec.ts` (`không bẩn thì cho rời`, `bẩn thì gọi confirmDiscard và trả kết quả`, `unmount thì gỡ đăng ký`); `useRouteTab.spec.ts` (`values là getter cập nhật sau khi nạp`); `AppraisalReportEditView.spec.ts`: `hiển_thị_tab_nhóm_theo_nhãn`, `không_có_nhóm_hiện_thông_báo` ("Chưa có nhóm tài sản."), `group_trong_url_chọn_đúng_nhóm`, `hỏi_xác_nhận_khi_rời_tab_chưa_lưu`, `lỗi_tải_hiện_alert_và_nút_thử_lại`.
- [ ] **Step 2: Chạy** `npm run test:unit -- AppraisalReportEditView useDirtyGuard useRouteTab tabs` → FAIL.
- [ ] **Step 3: Cài đặt.** View theo khuôn tải/hủy/lỗi của `F views/report-juridical/ReportJuridicalEditView.vue`; layout data page full width (frontend/CLAUDE.md quy tắc 5); header "Báo cáo thẩm định" + nút "Quay lại".
- [ ] **Step 4: Chạy** `npm run lint && npm run type-check && npm run test:unit` → PASS.
- [ ] **Step 5: Commit** `git -C frontend add -A && git -C frontend commit -m "feat: khung màn báo cáo thẩm định hai cấp tab"`
