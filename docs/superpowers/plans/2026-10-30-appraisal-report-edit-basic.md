# Migrate "Sửa thông tin cơ bản" báo cáo thẩm định (`/appraisal-report/edit-basic/{id}`)

> **Subagent: CHỈ `model: "sonnet"`** cho mọi lần gọi `Agent` (implementer, spec reviewer, code reviewer). Quy trình như `docs/superpowers/plans/2026-10-26-appraisal-report-edit-tab/00-index.md` (superpowers:subagent-driven-development, mỗi task: implementer → spec review → code review, tuần tự, không chạy song song). Bước 0: copy plan này vào `docs/superpowers/plans/2026-10-30-appraisal-report-edit-basic.md`.

## Context

Hệ thống mới đã có màn **tạo** báo cáo (`AppraisalReportCreateService`, `AppraisalReportBasicForm.vue`) nhưng chưa có màn **sửa thông tin cơ bản**. Màn cũ không chỉ ghi đè trường của báo cáo mà còn đồng bộ lại nhóm tài sản: bỏ tích/tích lại tài sản, chuyển nhóm, đánh lại số nhóm, đổi loại tài sản (reset dây chuyền dữ liệu của nhóm), dọn nhóm rỗng và khôi phục nhóm đã xóa mềm. DB dùng chung với hệ thống cũ (vẫn đọc/ghi) nên dữ liệu ghi phải đúng như hệ thống cũ ghi.

Viết tắt: **L** = `td-project-legacy/src/main/java/com/jvb/thanhdo/`, **LS** = `L service/impl/AppraisalReportServiceImpl.java`, **N** = `backend/src/main/java/com/thanhdo/tdprojectbe/`, **NT** = `backend/src/test/java/com/thanhdo/tdprojectbe/`, **F** = `frontend/src/`.

Nhánh: tạo `feat/appraisal-report-edit-basic` trong `backend` và `frontend` từ `feat/appraisal-report-edit-tab` hiện tại; commit bằng `git -C backend …`/`git -C frontend …`. Không push, không cập nhật con trỏ submodule ở repo gốc.

## Hành vi hệ thống cũ (spec — implementer phải đọc lại dòng code được dẫn)

**Controller** `L controller/AppraisalReportController.java:198-249`. GET nạp form; POST gọi `updateBasicInfo` rồi redirect `/appraisal-report/edit-tab/{id}`; lỗi `IllegalArgumentException` → flash + quay lại form.

**GET (dữ liệu form)**
- Dòng tài sản: tài sản active của báo cáo (`checked=true`, `group`=groupIndex, `assetType`=loại của nhóm, `id`, `uuid`, `asset`) + dòng "phantom" cho mỗi pháp lý của yêu cầu có uuid chưa có trên báo cáo (`id=null`, chưa tích, tên = `legalDocuments` nếu loại 5/6, ngược lại `landParcelNo`) — `L service/impl/AppraisalReportAssetServiceImpl.java:41-67`. N đã có quy tắc tên ở `N service/AppraisalReportFormService.legalRow` (97-101) → tái dùng.
- Pháp lý chung / theo tỉnh / cơ sở thẩm định: trộn JSON đã lưu với danh mục (`mergeJuridicalRowsWithSavedJson`, `mergeValuationBasisWithSavedJson`, controller ~558-660): mục danh mục có trong JSON → dùng mục đã lưu (ghi đè `category`/`assetType` từ danh mục); không có → mục mới chưa tích; cuối cùng nối mục đã lưu có id null hoặc id không còn trong danh mục. Danh mục sắp theo id.
- Phạm vi công việc, căn cứ dữ liệu, thị trường, giả thiết: JSON đã lưu nếu khác rỗng, ngược lại danh mục tích sẵn (`templates/demo/basic-information-edit.html:303-420`).
- Không kiểm quyền/trạng thái (chỉ cần đăng nhập).

**POST `updateBasicInfo` (LS:156-360)** — `@Transactional`:
0. Ghi đè trường báo cáo + 7 JSON, `surveyTime/valuationTime` rỗng → null, `updated_by`, `updated`.
1. Validate dòng đã tích (`L validation/LegalDocumentAssetTypeValidator`) ≡ `N service/ReportLegalDocumentValidator`. Xóa mềm tài sản active có id không nằm trong các dòng đã tích.
2. Với mỗi dòng đã tích (theo thứ tự gửi lên), `groupCache` theo groupIndex, `processedGroupIds`:
   - `id == null`: `resolveOrCreateGroup(...)`; có tài sản cùng (nhóm, uuid) (mọi cờ) → hồi sinh + đổi tên; không → tạo mới.
   - `id != null` (chỉ khi thuộc báo cáo và active): đổi tên/uuid.
     - **A (đổi groupIndex)**: **A1** nếu nhóm đích chưa có (DB active/cache) và nhóm cũ bị bỏ (index null hoặc không còn dòng nào nhắm tới) → đổi index ngay trên nhóm cũ, đổi loại; loại đổi và chưa xử lý → `handleAssetTypeChange` + `applyDefaultLandValuationMethod`. **A2** ngược lại → `resolveOrCreateGroup`, nếu nhóm đích có tài sản trùng uuid khác id → `mergeLegalFields(existing, asset)` + xóa mềm bản trùng; chèn bản ghi xóa mềm (lưu vết) vào nhóm cũ; chuyển tài sản sang nhóm đích. *(Lỗi cũ: bỏ qua loại gửi lên — xem "Khác hệ thống cũ" #1.)*
     - **B (giữ index)**: đổi loại nhóm; loại đổi và chưa xử lý → reset như A1.
3. `syncGroupTypesAndCleanupEmptyGroups` (LS:922-1002): nhóm 0 tài sản → xóa mềm nhóm + 4 so sánh + mô tả đất + mô tả xe + kết quả (`deleted=1`); **không** đụng `control_compare_factor(_synthetic)` và CTXD. 1 tài sản → SINGLE, ≥2 → GROUP.

**`resolveOrCreateGroup` (LS:1009-1074)**: cache → nhóm active cùng index (SINGLE→GROUP) → `findMatchingDeletedGroup` (native, `L repository/AppraisalAssetGroupRepository.java:22-33`: cùng báo cáo, cùng loại, đã xóa, tập uuid khớp chính xác, mới nhất) → `restoreDeletedGroupAndDependencies` → tạo nhóm SINGLE mới + bản ghi con mặc định. Index null → mỗi tài sản một nhóm riêng (vẫn thử khôi phục trước).

**`restoreDeletedGroupAndDependencies` (LS:1097-1240)**: nhóm `delete_flag=0` + index mới; tài sản của nhóm: mỗi uuid giữ bản mới nhất (updated, created, id) active, còn lại xóa mềm; mô tả đất/xe hồi sinh (hoặc tạo nếu thiếu theo loại); so sánh đã xóa hồi sinh, type 0 đặt `address = joinAssetAddresses` (thiếu hết → tạo 4 dòng); kết quả hồi sinh; loại LAND → hồi sinh CTXD `delete_flag=1` của nhóm. Không khôi phục control factor/synthetic.

**`handleAssetTypeChange` (LS:365-819)** theo đúng thứ tự: (1) mô tả đất: loại mới Đất/Căn hộ → hồi sinh + `clearLandDescriptionFields` (LS:433-500, **không** xóa `private_area`) + xóa ảnh, hoặc tạo; loại mới Ô tô → xóa mềm + xóa ảnh. (2) mô tả xe: rời Ô tô → xóa mềm + xóa ảnh; vào Ô tô → hồi sinh (không xóa trường) hoặc tạo. (3) reset 4 so sánh active (LS:585-672, danh sách cột trong báo cáo khảo sát; type 0 `address` = tên tài sản active nối `\n`) + xóa ảnh TSSS. (4) reset `control_compare_factor_synthetic` (LS:696-728). (5) xóa mềm `control_compare_factor` cũ + tạo mặc định theo loại mới. (6) reset `appraisal_result` (LS:747-769, `rounding_money_type=1`). (7) loại mới ≠ Đất → xóa mềm CTXD + kết cấu + đánh giá + ảnh CTXD. (8) reset trường nhóm (LS:806-819). Sau đó `applyDefaultLandValuationMethod`.

## Khác hệ thống cũ (đã chốt — ghi vào javadoc)

1. **Sửa lỗi A2 / dòng mới vào nhóm có sẵn**: sau bước 2, với mỗi nhóm đích nhận dòng đã tích, nếu `assetType` của nhóm ≠ loại các dòng gửi lên và nhóm chưa reset trong request → chạy `handleAssetTypeChange` + `applyDefaultLandValuationMethod`.
2. Xóa ảnh S3 **sau commit** qua `N common/util/AfterCommit` và các service ảnh hiện có (cũ: `CompletableFuture.runAsync` ngoài giao dịch, có thể xóa ảnh dù rollback).
3. Validate ở server trả 400 kèm lỗi trường như màn tạo (field `legalDocuments`); dòng có `id` không thuộc báo cáo/không active → 400 (cũ: bỏ qua im lặng); dòng `id=null` phải có uuid thuộc yêu cầu (`belongToRequest`).
4. `appraisal_standard` không ghi (form cũ đã comment ô này nên cũ ghi null; mới giữ nguyên).
5. Ô Nhóm: `max(số dòng + 1, groupIndex lớn nhất hiện có)` nhóm, như màn tạo mới (cũ: số nhóm + 1).
6. Hộp xác nhận đổi loại tài sản dùng `confirmDialog`, bỏ tùy chọn "không hiện lại" (localStorage).
7. Hộp xác nhận chỉ hiện với dòng đã có trên báo cáo (có `id`); dòng phantom đổi loại không hỏi (chưa có dữ liệu để xóa).

## API

- `GET /api/appraisal-reports/{reportId}/basic` → `ReportBasicEditData { reportId, profileId, appraisalProperty, valuationBasis, valuationMethod, provinceId, surveyTime, valuationTime, provinces, assets: [ReportBasicAssetRow{id, uuid, checked, group, assetType, name}], juridicals, provinceJuridicals, valuationBases, valuationScopes, dataBases, marketOverviews, assumptions: List<ReportContentItem> }`.
- `PUT /api/appraisal-reports/{reportId}/basic` body `AppraisalReportBasicUpdateRequest` (= trường của `AppraisalReportCreateRequest`, `legalDocuments` dùng `ReportLegalDocumentInput` thêm `Long id` nullable — màn tạo bỏ qua) → `ApiResponse.ok(null, "Cập nhật thông tin cơ bản thành công")`.
- Cùng controller-prefix với `N controller/AppraisalReportWorkspaceController` (`/api/appraisal-reports/{reportId}`), dùng `ReportEditScope.requireReport(reportId)`; `SecurityConfig:72` đã phủ đường dẫn. Đổi tỉnh ở form dùng lại `GET /api/appraisal-records/{profileId}/reports/form/province-juridicals`.

## Tasks

### Task 1 — BE: entity, mapping, repository (nền)
Files: `N model/entity/{AppraisalReport, AppraisalReportAsset, AppraisalAssetGroup, AppraisalLandDescription, AppraisalVehicleDescription, AppraisalGroupCompare, ControlCompareFactorSynthetic, AppraisalResult}.java`, các repository tương ứng, `NT repository/AppraisalReportBasicEditRepositoryTest.java` (`@MysqlDataJpaTest`).
- `AppraisalReport.applyBasicInfo(fields, updatedBy, now)`; bỏ `updatable=false` ở `updated`, `updated_by` (đọc comment N:95-106 trước; test chứng minh hai cột đổi).
- `AppraisalReportAsset`: map 5 cột `land_area, private_area, shared_area, not_planning_land, planning_land` (kiểu khớp schema, xem entity cũ); `rename`, `softDelete`, `revive`, `moveTo(groupId)`, `mergeLegalFieldsFrom(source)` (port `mergeLegalFields` LS:~824-888: chỉ điền khi đích rỗng).
- `AppraisalAssetGroup`: `moveToIndex`, `changeAssetType` (+ default land valuation method tách từ `create`), `resetForAssetType(newType, now)` (bước 8), `syncType(count)`, `markDeleted`, `restore(index)`, `promoteToGroup`.
- `AppraisalLandDescription.clear()` (đúng danh sách LS:433-500, `use_actual_land_directions=false`), `softDelete/revive`; `AppraisalVehicleDescription.softDelete/revive`; `AppraisalGroupCompare.reset(address)` (LS:585-672), `softDelete/restore`; `ControlCompareFactorSynthetic.reset(now)`; `AppraisalResult.reset()`, `softDelete/restore`.
- Finder: asset `findByAppraisalReportIdAndDeleteFlag`, `findByAppraisalAssetGroupId`, `findFirstByAppraisalAssetGroupIdAndUuid`; group `findByAppraisalReportIdAndGroupIndexAndDeleteFlag`, `findMatchingDeletedGroup` (native — lý do: so khớp tập uuid chính xác bằng 2 subquery đếm, `LIMIT 1`; comment trên query); compare by group mọi cờ; land desc by group + flag.
- Repository test chạy mọi query mới + đọc 5 cột mới + `updated` đổi sau lưu.

### Task 2 — BE: tách seam từ service hiện có
- `N service/ReportGroupDetailsWriter`: public, tách `writeDefaultFactors(group, now)` và `writeFor(reportId, group, now)` (1 nhóm) — create dùng lại, không đổi hành vi.
- `N service/ConstructionDescriptionService`: tách `softDelete(constructionId)` không kiểm scope (delete hiện tại gọi lại nó).
- `N service/ComparisonImageService`: public `removeAllOfGroupCompare(compareId)` dùng `removeSafely`.
- `N service/AppraisalImageService` (hoặc helper nhỏ): `deleteAllOfLandDescription(id)`, `deleteAllOfVehicleDescription(id)` = `markDeleted` + `UnreferencedImageKeys.delete`.
- Chạy lại test hiện có của các service này.

### Task 3 — BE: `GroupAssetTypeReset` (port `handleAssetTypeChange`)
File mới `N service/GroupAssetTypeReset.java` (+ `NT service/GroupAssetTypeResetTest.java`, Mockito). Một method `reset(reportId, group, oldType, newType, now)` thực hiện 8 bước đúng thứ tự (mỗi bước một method private ngắn). Test: Đất→Căn hộ (xóa trường mô tả đất, xóa mềm CTXD như cũ — bước 7 áp dụng mọi loại ≠ Đất), Đất→Ô tô (xóa mềm mô tả đất, tạo mô tả xe, xóa mềm CTXD, xóa ghi chú CTXD), Ô tô→Đất, factor mặc định đúng nhãn theo loại, ảnh xóa qua seam T2.

### Task 4 — BE: `ReportGroupResolver` (resolve/restore/cleanup)
File mới `N service/ReportGroupResolver.java` (+ test Mockito): `resolveOrCreate(...)` (LS:1009-1074), `restore(...)` (LS:1097-1240), `syncTypesAndCleanup(reportId, now)` (LS:922-1002). Test từng nhánh: cache, nhóm active (SINGLE→GROUP), khôi phục khớp uuid, tạo mới + bản ghi con, cleanup không đụng factor/CTXD, restore chọn bản tài sản mới nhất, CTXD chỉ hồi sinh khi LAND.

### Task 5 — BE: cập nhật (PUT)
Files: `N service/AppraisalReportBasicEditService.java` (`update`), `N model/dto/AppraisalReportBasicUpdateRequest.java`, `ReportLegalDocumentInput` (+`id`), `N controller/AppraisalReportBasicController.java` (`@RequestMapping("/api/appraisal-reports/{reportId}/basic")`), tests: `NT service/AppraisalReportBasicEditServiceTest`, `NT controller/AppraisalReportBasicControllerTest` (`@WebMvcTest`).
- Luồng: `scope.requireReport` → validate (tỉnh, `ReportLegalDocumentValidator`, belongToRequest, id thuộc báo cáo) → bước 0-2 (A1/A2/B như spec) → **bước đối soát loại (Khác #1)** → `syncTypesAndCleanup`.
- Service test bắt buộc các kịch bản: đổi tên; bỏ tích (xóa mềm); tích lại phantom → hồi sinh/tạo; đánh lại số nhóm (A1 giữ id nhóm); hoán đổi 2 tài sản giữa 2 nhóm (A2 + bản ghi lưu vết); đổi loại ở B chỉ reset 1 lần cho nhóm nhiều dòng; A1 kèm đổi loại; ví dụ lỗi A2 (X→Nhóm 2 loại Đất, Z→Nhóm 3) → Nhóm 2 được reset sang Đất; gom lại đúng tập cũ → khôi phục; nhóm rỗng bị dọn; validation lỗi → không ghi.

### Task 6 — BE: dữ liệu form sửa (GET)
`AppraisalReportBasicEditService.formData(reportId)` (`readOnly`), DTO `ReportBasicEditData`, `ReportBasicAssetRow`; hàm trộn thuần `ReportContentMerge` (juridical + valuation basis theo spec, tạo bản sao record); tái dùng `legalRow` của `AppraisalReportFormService` (đưa thành static dùng chung) và finder danh mục hiện có. Test: unit test hàm trộn, service test phantom, controller test GET, repository test nếu có query mới.

### Task 7 — FE: form dùng chung tạo/sửa
Files: `F lib/appraisal-report-form.ts` (+ spec), `F components/appraisal-report/AppraisalReportBasicForm.vue`, `F components/appraisal-report/ReportLegalDocumentList.vue`, `F api/appraisal-records.ts` (hoặc `F api/report-edit/basic.ts`).
- `LegalDocumentFormRow` thêm `id: number | null`; `buildAppraisalReportInput` gửi `id`; `toEditRows(assets)`; `groupOptions` nhận số tối thiểu (Khác #5); `toConfigItemsFromSaved(items)` giữ `checked`.
- Form nhận `initialValues`, dòng pháp lý, cấu hình ban đầu, `profileId`, và hàm `save(input) → reportId` + thông điệp thành công từ view; màn tạo giữ nguyên hành vi (test cũ vẫn qua).
- `ReportLegalDocumentList`: prop `confirmTypeChange` — dòng có `id` đổi loại từ giá trị khác rỗng → `confirmDialog` với nội dung cảnh báo của modal cũ (`templates/demo/demo-report-edit-basic.html:70-102`); hủy → trả giá trị cũ.

### Task 8 — FE: route, view, lối vào
- Route `appraisal-reports/:reportId(\d+)/basic`, name `appraisal-report-edit-basic`, `APPRAISAL_RECORD_AUTHORITIES`, breadcrumb (`F router/index.ts` gần dòng 277).
- `F views/appraisal-record/AppraisalReportEditBasicView.vue`: tải `GET …/basic` (AbortController, skeleton, lỗi + thử lại như `AppraisalReportCreateView`), lưu `PUT` → toast → `router.replace` về `appraisal-report-edit`; nút Quay lại về `appraisal-report-edit`.
- Nút "Sửa thông tin cơ bản" ở header `F views/appraisal-record/AppraisalReportEditView.vue` (cập nhật doc comment dòng 18-20) và icon bút ở `F components/appraisal-record/AppraisalReportListModal.vue` (như `list-appraisal-records.html:584`).

### Task 9 — Kiểm tra cuối + review toàn nhánh (sonnet).

## Tái dùng (không viết lại)
`ReportEditScope`, `CurrentUser.id(jwt)`, `ReportLegalDocumentValidator`, `ReportAssetGroupPlanner` (không dùng cho update, chỉ tham khảo), `ReportGroupDetailsWriter`, `ControlFactorDefaults`, `AfterCommit`, `UnreferencedImageKeys`, `AppraisalReportFormService.legalRow`, FE: `AppraisalReportBasicForm`, `ReportLegalDocumentList`, `ReportConfigModal`, `legalDocumentsProblem`, `listProvinceJuridicals`, `confirmDialog`, `useToast`.

## Verification
- Backend (`backend/`): `./mvnw -q test -Dtest='AppraisalReportBasic*,GroupAssetTypeReset*,ReportGroupResolver*,ReportContentMerge*,AppraisalReportCreate*'` sau mỗi task; cuối cùng `./mvnw test` (repository test cần MySQL).
- Frontend (`frontend/`): `npm run lint && npm run type-check && npm run test:unit`.
- E2E thủ công (skill `run`): tạo báo cáo 2 nhóm (Đất, Ô tô) → nhập mô tả/so sánh/kết quả → vào "Sửa thông tin cơ bản": (a) đổi tên + cấu hình → kiểm tra lưu; (b) đổi loại Nhóm 1 Đất→Căn hộ → mô tả bị xóa, CTXD bị xóa mềm; (c) bỏ tích tài sản Nhóm 2 → nhóm biến mất ở edit-tab; tích lại cùng loại → dữ liệu cũ quay về; (d) kịch bản lỗi A2 → nhóm đích đổi loại và reset. Mở song song màn cũ `/appraisal-report/edit-tab/{id}` để chắc dữ liệu hệ thống cũ đọc đúng.
