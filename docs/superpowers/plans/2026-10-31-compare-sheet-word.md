# Migrate "Xuất phiếu TSSS" sang code base mới

## Context
Ở code cũ, tab "Thông tin so sánh" của `/appraisal-report/edit-tab/{id}` (đất, căn hộ, ô tô) có nút **Xuất phiếu TSSS**. Nút gọi `GET /appraisal-report/compare-information-export-word/{reportId}?groupId=`, và endpoint này dựng file bằng `ComparativeInformationWordExporter` (`td-project-legacy/.../controller/component/wordExport/ComparativeInformationWordExporter.java`, 659 dòng).

Mục tiêu là đưa chức năng sang màn `/appraisal-reports/{id}` (backend + frontend mới) và **giữ nguyên cấu trúc file Word**. Phần dựng tài liệu được port gần như nguyên văn. Chỉ thay nguồn dữ liệu và các dịch vụ phụ trợ.

Thứ tự trong file Word cũ:
1. Banner.
2. Tiêu đề "PHIẾU THÔNG TIN TÀI SẢN SO SÁNH", kèm dòng "(Áp dụng cho đất, nhà phố, biệt thự, căn hộ)" nếu là đất hoặc căn hộ.
3. "1. THÔNG TIN CHUNG": bảng 3×2 gồm Hợp đồng số, Địa chỉ TSTĐ, Khách hàng.
4. "2. THÔNG TIN VỀ TÀI SẢN SO SÁNH": với mỗi TSSS (type > 0, chưa xóa, có ảnh) có một dòng nhãn (nguồn, liên hệ, tọa độ) và bảng ảnh 2 cột.
5. Ảnh "SƠ ĐỒ VỊ TRÍ…" từ Google Static Maps, chỉ với đất hoặc căn hộ.
6. Dòng cam kết, dòng ngày tháng theo chi nhánh, bảng ký tên (thêm hàng họ tên người được giao nếu có).

## Nhánh và cách thực hiện
- Tạo nhánh `feat/compare-sheet-word` từ `develop` trong cả `backend` và `frontend`. Không push.
- Thực hiện theo subagent-driven-development, **mọi subagent dùng model sonnet**: implementer và reviewer cho từng task, và review cuối.
- Lưu bản plan này vào `docs/superpowers/plans/2026-10-31-compare-sheet-word.md` (repo gốc), đánh dấu ✅ theo từng task như các plan trước.

## Task 1: Backend, exporter Word ✅ (backend 2d4f164)
Đặt ở `backend/src/main/java/com/thanhdo/tdprojectbe/common/util/export/word/compare/`.

- **`CompareSheetWordExporter extends WordExportCommon`** (`common/util/export/word/WordExportCommon.java` đã có đủ `createTable`, `writeCell`, `createParagraphAttr`, `setTableLayoutFixed`, `setRowCellWidths`, `mergeCellHorizontally`, `removeEmptyParagraphCell`, `addRunPicture*DualInputStreams`, `export(OutputStream)`).
  - **Port nguyên văn** `writeDataLines`, `setupPageMargin`, `createGeneralInfoTable`, `createComparativeImage`, `addImageToCell`, `addBannerImage`, `generateBranchCurrentDate`, `createSignatureTable` và mọi `WriteTextAttribute`: chuỗi, cỡ chữ, spacing, độ rộng cột giữ nguyên.
  - **Constructor nhận dữ liệu đã chuẩn bị sẵn** để exporter không gọi HTTP hay DB: `(AppraisalRequest request, LinkedBranch branch /*nullable*/, AppraisalAssetGroup group, String assignedEmployeeName /*nullable*/, byte[] staticMap /*nullable*/, S3Service s3, LocalDate today)`.
  - **Bỏ khỏi exporter:** `CloudFrontService` (code cũ không dùng), `GoogleRouteService` và lệnh gọi Static Maps.
  - **Ngày tháng:** `today` thay cho `LocalDate.now()`; service lấy giá trị này từ `Clock` (`config/ClockConfig.java`).
  - **Sửa lỗi kiểu dữ liệu:** khách hàng doanh nghiệp so bằng `AppraisalRequest.REQUEST_TYPE_BUSINESS.equals(requestType)`. `requestType` giờ là String nên `CommonConstants.HO_SO_DOANH_NGHIEP` (Integer) luôn ra false.
  - **Địa chỉ:** đất và căn hộ dùng `ReportUltis.formatAssetNames(group)` (đã có trong `common/util/ReportUltis.java`); ô tô dùng `vehicleLocation`.
- **`CompareSheetImages`** (helper cùng package). Tách phần ảnh TSSS và sơ đồ để mỗi file dưới khoảng 250 dòng. Logic copy nguyên từ code cũ, kể cả bố cục: ảnh rộng 3.0", ảnh lẻ thì merge ô, bảng không viền, ghi tên file khi tải ảnh lỗi.
- **Banner:** chép `td-project-legacy/src/main/resources/static/img/Thanhdo_branchs_banner_lite.png` vào `backend/src/main/resources/word/Thanhdo_branchs_banner_lite.png`, đọc bằng `getResourceAsStream("/word/…")`.
- **Thứ tự TSSS và ảnh:** giữ thứ tự collection của entity như code cũ. Cả hai bên đều không có `@OrderBy`.
- **Test** `CompareSheetWordExporterTest` (JUnit + Mockito, mock `S3Service` trả PNG nhỏ; mở docx theo kiểu `PrintingCertificateExportServiceTest`):
  - Đất có 2 TSSS (3 ảnh và 1 ảnh) cùng sơ đồ:
    - thứ tự đoạn văn: tiêu đề, dòng "(Áp dụng…)", "1. THÔNG TIN CHUNG", "2. …", nhãn TSSS, "SƠ ĐỒ VỊ TRÍ…", cam kết, ngày;
    - bảng chung 3×2 có đúng giá trị;
    - bảng ảnh có 2 hàng (hàng 2 merge) và 1 hàng;
    - số ảnh = banner + 4 + sơ đồ;
    - bảng ký có 4 hàng khi có tên người được giao.
  - Ô tô: không có dòng "(Áp dụng…)" và không có sơ đồ, dù truyền `staticMap`.
  - TSSS bị xóa, type 0 hoặc không có ảnh thì bị bỏ qua.
  - Nhãn có tọa độ: ưu tiên DMS, nếu không có thì dùng `latitude.stripTrailingZeros()`.
  - Doanh nghiệp (`requestType="2"`) lấy `companyName`.
  - S3 ném lỗi thì ô ghi tên file.

## Task 2: Backend, service, Static Maps và endpoint
- **`service/StaticMapClient`** (`@Component`): `byte[] fetch(List<AppraisalGroupCompare> activeCompares, String apiKey, String polyline)`.
  - URL dựng y hệt `fetchGoogleStaticMapBytes` cũ: `format=png32&scale=2&size=500x300&maptype=satellite`, marker có `label:<type>` khi type > 0, `path=color:0x0000ffff|weight:3|enc:`.
  - Dùng `java.net.http.HttpClient` như `WalkingRouteService`, timeout 5s/10s. Lỗi thì trả null.
  - Tách hàm static thuần `buildUrl(...)` để test.
- **`service/CompareSheetWordService`**: `ExportedReport export(long profileId, long reportId, long groupId, Long callerId)`.
  1. `scope.requireGroup(profileId, reportId, groupId)`.
  2. Lấy TSSS còn hiệu lực bằng `AppraisalGroupCompareRepository.findByAppraisalAssetGroupIdAndDeleteFlag`. Chỉ với đất hoặc căn hộ, khi `walkingRoutePolyline` trống và `WalkingRoutePoints.isRoutable(...)`: gọi `walkingRouteService.fetchPolyline(callerId, WalkingRoutePoints.of(...))`. Có kết quả thì `walkingRouteService.store(groupId, polyline)` (tương đương `recalculateAndGet` cũ).
  3. Key bản đồ: `MapApiKeyProperties.hn()`, nếu trống thì `MapKeyService.keyForBranch(request.getLinkedBranch())`. Code cũ dùng `map.api.static`, giá trị mặc định chính là `MAP_API_KEY_HN`. Lấy ảnh qua `StaticMapClient` khi key không trống.
  4. Trong `TransactionTemplate` read-only: nạp group, request, `LinkedBranch` (`findFirstByBranchCodeAndDeleteFlag(…, DELETE_FLAG_ACTIVE)`), tên người được giao, rồi ghi docx ra `ByteArrayOutputStream`.
     - Tên người được giao: `employeeRepository.findById(assignment)`, rồi `getFullName()` (rỗng thì `""`, để giữ hàng ký tên như `CONCAT_WS` cũ).
  - Các bước gọi HTTP chạy ngoài transaction, theo Javadoc của `fetchPolyline`.
  - Tên file: `"Phieu_TSSS_" + CommonUtils.generateAppraisalReportFileName(request) + ".docx"`.
- **Endpoint** trong `controller/GroupCompareController.java`: `GET /api/appraisal-records/{profileId}/reports/{reportId}/groups/{groupId}/compare/word`. Trả `ResponseEntity<byte[]>`, content-type và `ContentDisposition.attachment().filename(name, UTF_8)` giống `AppraisalReportWorkspaceController#word`. `callerId = CurrentUser.id(jwt)`.
- **Test:**
  - `CompareSheetWordServiceTest` (Mockito): polyline đã có thì không gọi route; polyline trống và đủ tọa độ thì gọi `fetchPolyline` rồi `store`; ô tô không gọi map; key HN trống thì dùng key chi nhánh; tên file đúng; nhóm sai trả NOT_FOUND.
  - `StaticMapClientTest.buildUrl`: marker, path, key.
  - `GroupCompareController` WebMvc: 200 kèm header `filename*=UTF-8''Phieu_TSSS_…docx`, 404, 403 với GUEST, 401 khi không có token.

## Task 3: Frontend, nút "Xuất phiếu TSSS"
- **API:** `src/api/report-edit/compare.ts` thêm `downloadCompareSheet(profileId, reportId, groupId)`, trả `api.download(\`${groupPath(...)}/compare/word\`)`.
- **Component** `src/components/appraisal-report/edit/compare/ExportCompareSheetButton.vue`, mô phỏng `edit/DownloadReportButton.vue`:
  - props `profileId`, `reportId`, `groupId`;
  - `<Button type="button" variant="outline">`, icon `FileDown`/`Download`, chữ "Xuất phiếu TSSS", trong lúc chạy là "Đang xuất…" và nút bị disabled;
  - `saveBlob(blob, filename ?? 'Phieu_TSSS.docx')`;
  - lỗi thì `toast.error('Xuất phiếu TSSS không thành công', { description: errorMessage(e) })`.
- **Vị trí:** đặt nút đầu tiên trong `<FormActions sticky>` của `LandCompareTab.vue`, `ApartmentCompareTab.vue` và `CarCompareTab.vue`, trước `DownloadReportButton` (code cũ cũng để nút này đầu tiên). Sửa doc comment "chưa có … nút xuất phiếu" ở `LandCompareTab.vue`.
- **Nguồn dữ liệu:** như code cũ, file xuất từ dữ liệu đã lưu, không gồm thay đổi chưa bấm Lưu.
- **Test** `ExportCompareSheetButton.spec.ts` theo mẫu `DownloadReportButton.spec.ts`: dùng tên file từ server; dùng tên dự phòng khi server không gửi tên; disabled khi đang xuất; lỗi thì toast và không lưu file. Mỗi spec của 3 tab thêm một assert là có nút "Xuất phiếu TSSS".

## Kiểm chứng
- **Backend:** `./mvnw test -Dtest='CompareSheet*,StaticMapClient*,GroupCompareController*'`, rồi `./mvnw test` toàn bộ.
- **Frontend:** `npm run lint`, `npm run type-check`, `npm run test:unit`. `AppraisalRequestListView.spec.ts` › "Thao tác khác" vốn đã fail, bỏ qua.
- **So cấu trúc với file cũ (thủ công):** chạy app cũ và app mới trên cùng DB, xuất phiếu của cùng một báo cáo và nhóm đất, giải nén hai file `.docx`, rồi so `word/document.xml`. Chỉ được khác ở rId và tên ảnh, ảnh bản đồ và ngày. Lặp lại với nhóm ô tô.
- **Trên trình duyệt:** `/appraisal-reports/{id}`, tab Thông tin so sánh, bấm "Xuất phiếu TSSS"; file tải về tên `Phieu_TSSS_<HĐ>_<nguồn>_<KH>.docx`.
