# 10 — Tab "Pháp lý" (T07–T09)

> Đọc `00-index.md` trước. Subagent: `model: "sonnet"`, có review.

## Hành vi hệ thống cũ

Nguồn: `L controller/AppraisalReportTabController.java:81-121`, `L controller/AppraisalReportController.java:437-444`, `L service/impl/AppraisalReportAssetServiceImpl.java:72-156`, `L entity/dto/request/UpdateAppraisalReportAssetRequest*.java`, `LR templates/demo/legal-document.html` (Đất/Căn hộ), `LR templates/demo/legal-document-car.html` (Ô tô), `L service/impl/AppraisalReportFileServiceImpl.java`.

- Mỗi `appraisal_report_assets` của nhóm là một pill "Tài sản N" với **form riêng**, lưu từng tài sản (multipart, có file).
- Nạp: `otherInfo` rỗng → một dòng trống; file đính kèm hiển thị link S3 presigned.
- **Đất/Căn hộ** — trường: 1 Chủ tài sản `owner` (mặc định khi null: `requestType == 1 ? custName : companyName` của yêu cầu), 2 Địa chỉ `asset`, 3.1 Loại GCN `certificateNumberType` (gợi ý 5 giá trị cố định ở `legal-document.html:94-100`, hiện hết khi focus), 3.2 Số GCN `certificateNumber`, 3.3 Nơi cấp `cnIssuedPlace`, 3.4 Số vào sổ `gcnNo`, 3.5 Ngày cấp `gcnNoIssuedDate`, 3.6 Cho ai `forWhom`, 3.7 Khác `other`, 3.8 Giấy tờ pháp lý của tài sản `legalDocuments` (tự sinh), 3.9 Giấy tờ đính kèm (pdf, doc(x), jpg, png, xls(x); kéo thả; xóa từng file), 3.10 Khác `otherInfo[] {uuid, content}` (thêm/xóa dòng). Không validate.
- **Công thức 3.8** (`legal-document.html:406-449`): ngày `dd/mm/yyyy` → `"dd tháng mm năm yyyy"`; nếu có ít nhất một trong loại/số/sổ/nơi cấp/ngày/cho ai: `gen = "${type || 'Giấy chứng nhận quyền sử dụng đất, quyền sở hữu nhà ở và tài sản khác gắn liền với đất'} số: ${number}, số vào sổ cấp GCN: ${gcnNo} do ${place} cấp ngày ${date}${forWhom ? ' cho ' + forWhom : ''}."` rồi chuẩn hóa khoảng trắng/dấu câu; nếu textarea đã có dòng khớp `/^[^\n]*(?:Giấy chứng nhận quyền sử dụng đất|[^\n]*số:.*cấp ngày)[^\n]*\./` thì thay dòng đó (gen rỗng → xóa), ngược lại chèn `gen` lên đầu; có `other` thì nối thêm " " + other. Chạy khi sửa 3.1–3.7. **Lỗi cũ**: `other` bị nối lặp mỗi phím — bản mới chỉ sinh lại phần tự sinh và giữ một lần `other` (ghi vào "Khác hệ thống cũ" của component).
- **Ô tô** — địa chỉ chỉ đọc; 1 GCN đăng ký xe: `certificateNumber`, `cnIssuedPlace`, `gcnNoIssuedDate`, `gcnNoExpiryDate`; 2 GCN kiểm định: `carInspectionCertificateNumber`, `carInspectionIssuedPlace`, `carInspectionIssuedDate`, `carInspectionExpiryDate`; 3 `legalDocuments` tự sinh; 4 file đính kèm. Không có chủ tài sản, không có "Khác".
- **Công thức 3 (ô tô)** (`legal-document-car.html:360-415`): dòng R = `"Giấy chứng nhận đăng ký xe ô tô" + [" số "+no] + [" do "+place] + [" cấp ngày "+fmt(issue)] + [", có hiệu lực đến hết ngày "+fmt(expiry)] + "."` (chỉ khi có ít nhất một phần); dòng I tương tự với tiền tố "Giấy chứng nhận kiểm định an toàn kỹ thuật và bảo vệ môi trường xe cơ giới"; giữ các dòng người dùng tự viết (không bắt đầu bằng hai tiền tố và không khớp `/^(?:Giấy đăng ký|Giấy đăng kiểm)…số…(cấp ngày|hiệu lực)…\./`); kết quả `[R, I].filter(Boolean).concat(manual).join('\n')`.
- **Lưu** (`updateAsset`): ghi đè mọi cột (CAR không ghi `asset`); file mới upload S3 key `{assetFolder}legalDoc/{reportId}/{assetId}/{uuid}/{fileName}`, thêm vào `appraisal_report_files` (file cũ giữ); `otherInfo`: bỏ dòng nội dung rỗng, sinh uuid thiếu. Xóa file: `DELETE` xóa cứng dòng + object S3. Không đồng bộ sang bảng khác.
- **Sửa (Khác hệ thống cũ 7–8)**: Đất/Căn hộ chỉ ghi nhóm cột đất (`owner, asset, certificateNumberType, certificateNumber, cnIssuedPlace, gcnNo, gcnNoIssuedDate, forWhom, other, legalDocuments, otherInfo`); Ô tô chỉ ghi `certificateNumber, cnIssuedPlace, gcnNoIssuedDate, gcnNoExpiryDate, carInspection*, legalDocuments`. Danh sách `otherInfo` rỗng → lưu rỗng.

### Task T07: Backend Pháp lý

**Files:**
- Modify: `N model/entity/AppraisalReportAsset.java` (thêm cột ở trên + `other_info` JSON; bỏ `updatable=false` ở `updated` + `@PreUpdate`; phương thức `applyLandLegal(LegalAssetInput)`, `applyCarLegal(LegalAssetInput)`)
- Create: `N model/dto/AppraisalAssetOther.java` (`record (UUID uuid, String content)`), `N model/converter/AppraisalAssetOtherListConverter.java`, `N model/entity/AppraisalReportFile.java` (bảng `appraisal_report_files`), `N repository/AppraisalReportFileRepository.java`, `N model/dto/LegalTabData.java`, `LegalAssetView.java`, `LegalAssetInput.java`, `LegalFileView.java`, `N service/ReportLegalService.java`, `N controller/ReportLegalController.java`
- Test: `NT repository/AppraisalReportAssetRepositoryTest.java`, `NT repository/AppraisalReportFileRepositoryTest.java`, `NT service/ReportLegalServiceTest.java`, `NT controller/ReportLegalControllerTest.java`

**Interfaces:**
- Consumes: `ReportEditScope.requireGroup`, `S3StorageService.upload/delete`, `S3LinkService` (link presigned — đọc lớp để chọn hàm đúng), `UploadedFiles.fileName`.
- Produces:
  - `record LegalTabData(int assetType, String defaultOwner, List<LegalAssetView> assets)`.
  - `record LegalAssetView(long id, String owner, String asset, String certificateNumberType, String certificateNumber, String cnIssuedPlace, String gcnNo, LocalDate gcnNoIssuedDate, LocalDate gcnNoExpiryDate, String carInspectionCertificateNumber, String carInspectionIssuedPlace, LocalDate carInspectionIssuedDate, LocalDate carInspectionExpiryDate, String forWhom, String other, String legalDocuments, List<AppraisalAssetOther> otherInfo, List<LegalFileView> files)`; `record LegalFileView(long id, String fileName, String url)`.
  - `record LegalAssetInput(...)` = các trường của `LegalAssetView` trừ `id`, `files` (`@Size` theo độ dài cột).
  - `GET …/groups/{groupId}/legal` → `LegalTabData` (tài sản chưa xóa theo id tăng dần, `otherInfo` rỗng → một dòng trống).
  - `PUT …/groups/{groupId}/legal/assets/{assetId}` multipart: part `data` (JSON `LegalAssetInput`), part `files` (tùy chọn) → `ApiResponse.ok(LegalAssetView, "Lưu pháp lý thành công")`.
  - `DELETE …/legal-files/{fileId}` → xóa cứng + S3 (file phải thuộc tài sản của báo cáo, sai → 404).

- [ ] **Step 1: Test thất bại**: repo test đọc/ghi `other_info` (khóa JSON `uuid`, `content`) và file; service: `chủ_tài_sản_mặc_định_theo_loại_yêu_cầu`, `lưu_đất_không_đụng_cột_ô_tô`, `lưu_ô_tô_không_đụng_chủ_và_địa_chỉ`, `khác_rỗng_bị_bỏ_và_sinh_uuid`, `upload_file_đúng_key_s3`, `tài_sản_nhóm_khác_404`; controller multipart.
- [ ] **Step 2–4:** FAIL → cài đặt → `./mvnw -q test -Dtest='*Legal*,AppraisalReportAssetRepositoryTest,AppraisalReportFileRepositoryTest'` PASS.
- [ ] **Step 5: Commit** `feat: API tab pháp lý báo cáo thẩm định`.

### Task T08: FE Pháp lý — Đất / Căn hộ

**Files:** Create `F api/report-edit/legal.ts`, `F lib/report-edit/legal-documents-text.ts`, `F components/appraisal-report/edit/legal/LegalTab.vue` (pill tài sản + chọn form theo `assetType`), `LegalLandAssetForm.vue`, `LegalAttachedFiles.vue` (danh sách + kéo thả + xóa, dùng chung với T09), `OtherInfoRows.vue`; Modify `GroupWorkspace.vue` (map `legal`). Test: `lib/report-edit/__tests__/legal-documents-text.spec.ts`, `components/appraisal-report/edit/legal/__tests__/LegalTab.spec.ts`.

**Interfaces:** Produces `legal-documents-text.ts`: `landLegalText(current: string, fields: LandLegalFields): string`, `carLegalText(current: string, fields: CarLegalFields): string`; `api/report-edit/legal.ts`: `getLegal(...)`, `saveLegalAsset(profileId, reportId, groupId, assetId, data, files: File[])` (FormData với part `data` là `Blob` JSON), `deleteLegalFile(...)`.

- [ ] **Step 1: Test thất bại**: `legal-documents-text.spec.ts` — `đủ_trường_sinh_đúng_câu` (so chuỗi đầy đủ theo công thức), `thay_dòng_đã_sinh_giữ_dòng_khác`, `không_có_trường_nào_xóa_dòng_sinh`, `khác_nối_một_lần`, `ngày_dạng_dd_tháng_mm_năm_yyyy`; `LegalTab.spec.ts` — `mỗi_tài_sản_một_pill`, `sửa_số_gcn_cập_nhật_3.8`, `lưu_gửi_formdata_có_file`, `xóa_file_hỏi_xác_nhận` (`confirmDelete`).
- [ ] **Step 2–4:** FAIL → cài đặt (vee-validate + zod, `DatePicker` ISO, gợi ý 3.1 bằng `SuggestInput`, `FormActions` nút "Lưu"; `useDirtyTab`) → lint/type-check/test PASS.
- [ ] **Step 5: Commit** `feat: tab pháp lý đất và căn hộ`.

### Task T09: FE Pháp lý — Ô tô

**Files:** Create `F components/appraisal-report/edit/legal/LegalCarAssetForm.vue`; Modify `LegalTab.vue`; Test `legal/__tests__/LegalCarAssetForm.spec.ts` (+ case `carLegalText` trong `legal-documents-text.spec.ts`: `giữ_dòng_tự_viết`, `chỉ_dòng_kiểm_định`).

- [ ] **Step 1–4:** test FAIL → cài đặt (địa chỉ chỉ đọc, 2 nhóm trường + file dùng `LegalAttachedFiles`) → PASS.
- [ ] **Step 5: Commit** `feat: tab pháp lý ô tô`.
