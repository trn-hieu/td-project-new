# Migrate "Xuất Excel" (tab Thông tin so sánh) sang code base mới

## Context
Ở hệ thống cũ, tab "Thông tin so sánh" của **nhóm đất** có nút **Xuất Excel** (`comparative-information.html:1581-1587`). Tab căn hộ và ô tô không có nút này. Nút gọi `GET /appraisal-report/compare-information-export-excel/{reportId}` (`AppraisalReportController.java:379-406`). `groupId` được gửi lên nhưng server không đọc, nên file **luôn gồm toàn bộ báo cáo**.

File được dựng bằng `CompareInformationReportExporter` (`td-project-legacy/.../controller/component/CompareInformationReportExporter.java`, 1008 dòng, kế thừa `ExportCommon`, 238 dòng). Workbook là `XSSFWorkbook` tạo từ đầu, không có template và không có công thức.

Cấu trúc workbook:
1. **Mỗi nhóm tài sản một sheet.** Tên sheet là `normalizeExcelSheetName(số thửa của tài sản đầu)`, rỗng thì là `"Tài sản i"`.
   - A0:F15: bảng so sánh (TSTĐ và 3 TSSS).
   - Từ dòng 18: "Bảng điều chỉnh" (A:G). Gồm các khối A, B, C, mỗi yếu tố một khối 5 dòng, sau đó khối tổng hợp D, D1, D2, E, E1 đến E4.
   - Từ cột H: bảng định giá đất, sau đó bảng tổng hợp kết quả.
2. **Sheet `CTXD`:** bên trái là bảng đánh giá chất lượng từng công trình, bên phải (từ cột H) là bảng tổng hợp giá công trình.
3. **Sheet `Tổng hợp`:** STT, hạng mục, thành tiền, tổng cộng và dòng "Bằng chữ".

Mục tiêu là đưa chức năng sang `/appraisal-reports/{id}` và **giữ nguyên cấu trúc file** (sheet, dòng, cột, chuỗi, style, độ rộng cột).

Hai quyết định của bạn:
- **Phạm vi giữ như cũ.** Nút chỉ có ở `LandCompareTab` và xuất toàn bộ báo cáo.
- **Sửa lỗi diện tích ≥ 1.000.** Bản cũ parse chuỗi châu Âu "1.234,50" thất bại, nên ô bị trống. Bản mới ghi thẳng số từ `BigDecimal`, vẫn định dạng `0.00`, vẫn căn phải.

## Nhánh và cách thực hiện
- Tạo nhánh `feat/compare-excel` từ `develop` trong `backend` và `frontend`. Không push, không merge.
- Thực hiện theo subagent-driven-development. **Mọi subagent dùng model sonnet.**
- Lưu plan vào `docs/superpowers/plans/2026-10-31-compare-excel.md` (repo gốc) và đánh dấu ✅ theo từng task.

## Task 1: Backend, exporter Excel ✅
Package: `backend/src/main/java/com/thanhdo/tdprojectbe/common/util/export/excel/compare/`.

**Port nguyên văn** mọi chuỗi, chỉ số dòng và cột, style (font, format, border, fill), độ rộng cột, ô gộp và thứ tự ghi. Giữ cả các "quirk" vì chúng thuộc về cấu trúc:
- `styleRowPrice` bị đổi sang căn giữa giữa chừng.
- `styleComparisonPriceRight` thực ra căn giữa.
- Cột A có độ rộng 15 hoặc 10.
- D2 ghi dạng chuỗi.
- Bảng định giá trả về `startRow - 1` khi detail null.
- Cột C (TSTĐ) trong khối giá để trống.

Nhóm căn hộ và ô tô dùng chung bố cục đất, như bản cũ.

Các file (mỗi file khoảng ≤ 250 dòng):
- **`CompareExcelBase`**: port `ExportCommon` sang `XSSFWorkbook`, ghi ngẫu nhiên được (cần `getOrCreateRow` vì CTXD và các bảng cột H ghi lại vào dòng cũ).
  - Gồm style nền `styleRow`, `styleRowCenter`, `styleRowPrice`, các hằng `COLUMN_SIZE_*`, và các hàm `createCell`, `createCellWithSize`, `setCellStyle`, `createMergeCellHorizontal/Vertical`, `export(OutputStream)`.
  - Không dùng `StreamingExportCommon`, vì SXSSF không ghi lại được dòng đã flush.
- **`CompareExcelStyles`**: các style riêng của exporter (constructor và dòng 106-231 bản cũ).
- **`GroupSheetWriter`**: `writeComparisonAssetTable` (bỏ lỗi parse diện tích) và `writeAdjustmentTableSection`/`writeOneGroup`.
- **`ValuationTablesWriter`**: `writeLandValuationSummaryTable` (kèm `appendLandOutsideRowsExcel`, `countLandOutsideRows`) và `writeAppraisalResultSummaryTable`.
- **`ConstructionSheetWriter`**: sheet `CTXD` (`writeCTXDSheet`, `writeConstructionEvaluationTable`, `toRoman`).
- **`CompareInformationExcelExporter`**: điều phối `export()` (dòng 233-260). Tạo sheet theo nhóm, rồi `CTXD`, rồi `Tổng hợp` (`writeTotalSummarySheet` nằm ở đây). Không có nhóm thì ra sheet "Report" rỗng như cũ.

**Đầu vào:** exporter đọc entity trực tiếp, bỏ lớp DTO `ControlCompareFactorDTOList`. Constructor nhận `List<CompareExcelGroup>` và `List<AppraisalConstructionDescription> constructions`.
- `CompareExcelGroup` là record trong `model/dto/` gồm `(AppraisalAssetGroup group, EvaluationResultDetailDto detail, String assetName, String sheetName)`.
- Exporter tự lấy TSSS theo `type` 0..3, `ControlCompareFactor` và `ControlCompareFactorSynthetic` từ group.
- Cách lọc và định dạng phải giống `AppraisalReportServiceImpl.mapGroupToControlCompareFactorDTOList` (dòng 1256-1388 bản cũ). Implementer đọc lại hàm này để chép đúng:
  - `NumberFormatUtils.formatBigDecimalToEuropeanStyleFixed2` cho chuỗi diện tích (riêng ô số diện tích thì ghi `BigDecimal.doubleValue()`).
  - `amplitudeControlN` lấy từ getter `Real`.
  - Điều kiện lọc delete-flag của TSSS và yếu tố.

**Tái dùng các hàm có sẵn trong code mới:**
- `common/util/export/word/AppraisalResultService.buildDetailFromEntities(...)`
- `ReadMoneyUtils.doc`
- `CommonUtils.formatFloatToVietnameseStyle` và `...WithDecimal`
- `LandPriceUtils.RESIDENTIAL_LAND_TYPE_ID`
- `CommonConstants.ConstructionValuationType`

`normalizeExcelSheetName` và `extractParcelNumberOnly` chưa có, nên port vào chỗ phù hợp (`CommonUtils` hoặc `ReportUltis`). Nếu hai nhóm trùng tên sheet thì bản cũ ném lỗi POI; bản mới thêm hậu tố ` (2)` để không vỡ export.

**Test `CompareInformationExcelExporterTest`** (mở workbook bằng `XSSFWorkbook(ByteArrayInputStream)`):
- Thứ tự và tên sheet: tên theo nhóm, sau đó `CTXD`, `Tổng hợp`.
- Header và nhãn của bảng so sánh (A0:F15) đúng vị trí, ô A1:B1 gộp, A5:A9 gộp dọc.
- Diện tích 1234.5 ghi thành ô số 1234.5, format `0.00`.
- Bảng điều chỉnh bắt đầu ở dòng 18/19. Mỗi yếu tố có khối 5 dòng với A gộp dọc "C1". Khối tổng hợp D..E4 có E1:G1 của D1 gộp.
- Có các bảng định giá ở cột H, với dòng "Bằng chữ" đúng `ReadMoneyUtils.doc`.
- `CTXD` bỏ qua cấu trúc "Khác" và công trình `NOT_INCLUDED`.
- `Tổng hợp` có tổng cộng đúng.
- Nhóm không có synthetic thì không có khối D/E.
- Hai nhóm trùng tên sheet vẫn xuất được.

## Task 2: Backend, service và endpoint ✅
- **`service/CompareExcelService.export(long profileId, long reportId)`** trả `ExportedReport`:
  - `scope.requireReport(profileId, reportId)`, rồi `profileRequests.requestOf(profileId)`.
  - Làm việc trong `TransactionTemplate` read-only (cùng pattern `CompareSheetWordService`), vì exporter đi qua các quan hệ lazy.
  - Với mỗi nhóm của báo cáo (theo thứ tự collection, lọc delete-flag như cũ): dựng `detail` tương đương `AppraisalResultServiceImpl.getDetailByReportAndGroup` của bản cũ.
    - TSSS `type 0` lấy qua `AppraisalGroupCompareRepository` với deleteFlag active.
    - Synthetic lấy qua `ControlCompareFactorSyntheticRepository`.
    - Công trình còn hiệu lực lấy theo repository tương ứng.
    - Sau đó gọi `appraisalResultService.buildDetailFromEntities(...)`. Tham số `roundingMoneyTypeOverride` chép đúng như cách hàm cũ gọi.
  - `assetName` là các `AppraisalReportAsset.asset` nối bằng `\n`.
  - `constructions` là các công trình có `valuationType != NOT_INCLUDED`.
  - Tên file là `CommonUtils.generateAppraisalReportFileName(request) + ".xlsx"`, giữ đúng như cũ, không thêm tiền tố.
- **Endpoint:** controller mới `controller/CompareExcelController`, `GET /api/appraisal-records/{profileId}/reports/{reportId}/compare/excel`. Trả `ResponseEntity<byte[]>`, content-type XLSX, `ContentDisposition.attachment().filename(name, UTF_8)` (như `EmployeeController`).
  - Đảm bảo `SecurityConfig` cho `/api/appraisal-records/**` đã phủ đường dẫn này.
- **Test:**
  - `CompareExcelServiceTest` (Mockito): tên file; báo cáo sai trả NOT_FOUND; mỗi nhóm gọi `buildDetailFromEntities` một lần; công trình `NOT_INCLUDED` bị loại.
  - `CompareExcelControllerTest` (WebMvc, theo mẫu `GroupCompareControllerTest`): 200 kèm `filename*=UTF-8''….xlsx`, 404, 403 với GUEST, 401 khi không có token.

## Task 3: Frontend, nút "Xuất Excel" ✅
- `src/api/report-edit/compare.ts`: thêm `downloadCompareExcel(profileId, reportId)`, trả `api.download(\`/appraisal-records/${profileId}/reports/${reportId}/compare/excel\`)` (theo prefix mà `groupPath` đang dùng).
- `src/components/appraisal-report/edit/compare/ExportCompareExcelButton.vue`, mô phỏng `ExportCompareSheetButton.vue`:
  - props `profileId`, `reportId`; icon `FileSpreadsheet`.
  - Chữ "Xuất Excel", trong lúc chạy là "Đang xuất…".
  - Tên dự phòng `Report.xlsx`.
  - Lỗi thì toast `Xuất Excel không thành công`.
- Chỉ đặt nút trong `LandCompareTab.vue`, ngay sau `ExportCompareSheetButton` (thứ tự như cũ: phiếu TSSS, Excel, Tải báo cáo, Lưu).
- Test `ExportCompareExcelButton.spec.ts` theo mẫu `ExportCompareSheetButton.spec.ts`. `LandCompareTab.spec` assert có nút. Spec của tab căn hộ và ô tô assert **không** có nút.

## Kiểm chứng
- **Backend:** `./mvnw test -Dtest='CompareInformationExcel*,CompareExcel*'`, rồi `./mvnw test` toàn bộ.
- **Frontend:** `npm run lint`, `npm run type-check`, `npm run test:unit`. Lỗi có sẵn ở `AppraisalRequestListView.spec.ts` › "Thao tác khác" thì bỏ qua.
- **So với bản cũ (thủ công):** dùng cùng DB và cùng một báo cáo đất có công trình. Xuất ở app cũ và app mới, mở hai file và so tên sheet, vị trí ô, ô gộp, giá trị. Chỉ được khác ở ô diện tích ≥ 1.000 (bản mới có số).
- **Trên trình duyệt:** vào `/appraisal-reports/{id}`, tab Thông tin so sánh của nhóm đất, bấm "Xuất Excel". Tab căn hộ và ô tô không có nút.
