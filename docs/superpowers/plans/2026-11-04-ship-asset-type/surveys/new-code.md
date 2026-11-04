# Khảo sát code mới: điểm cần chạm khi thêm SHIP

> Khảo sát được làm trên backend nhánh `refactor/appraisal-word-exporter` và frontend nhánh `feat/appraisal-report-edit-basic`. Cả hai nhánh đã được merge vào `origin/develop`. Số dòng ở đây có thể lệch so với develop, **phải grep lại** trước khi sửa.
>
> Ký hiệu: N = `backend/src/main/java/com/thanhdo/tdprojectbe/`, F = `frontend/src/`.

## Backend

### Hằng số loại tài sản

Loại tài sản là `int`, lưu ở cột `appraisal_asset_groups.asset_type`. Giá trị: LAND=1, APARTMENT=2, CAR=3. Không dùng enum.

Có **hai bản khai báo** loại tài sản:
- `N/common/constant/AppraisalAssetType.java`: entity và service dùng bản này.
- `N/common/constant/CommonConstants.java`, lớp lồng `AppraisalAssetType`: Word exporter, `AppraisalResultService` và các test dùng bản này.

Khi thêm SHIP phải thêm vào **cả hai**.

Các trường `asset_type` khác có ngữ nghĩa riêng:

| Trường | Vị trí | Ghi chú |
|---|---|---|
| `ValuationBasis.ASSET_TYPE_COMMON` / `ASSET_TYPE_CAR` | `model/entity/ValuationBasis.java` | `ASSET_TYPE_CAR=3` đang là cận trên của vòng lặp ở `service/ReportContentMerge.java:~69` và `service/AppraisalReportFormService.java:~66`. Cần nới lên 4. |
| `ValuationMethod.assetType` | | Dữ liệu master, Word mục 2.4 đọc. Đã có seed `asset_type=4` từ legacy. |
| `ControlCompareFactorLabel.assetTypes` | | JSON. `AdjustmentPanelService:~62` lọc theo `appliesToAssetType`. |
| `CompareInformation.assetType` | | Kho TSSS. |
| `LegalDocumentItem.assetType` | | Hệ mã 1/2/3/5/6 của hồ sơ. Không liên quan. |

### Tạo báo cáo / edit-basic / vòng đời nhóm

- `service/ReportGroupDetailsWriter.writeFor`: LAND/APARTMENT tạo `AppraisalLandDescription.empty`, CAR tạo `AppraisalVehicleDescription.emptyCar`. Sau đó luôn tạo 4 dòng `AppraisalGroupCompare.forGroup` và các yếu tố mặc định. **Cần thêm nhánh SHIP.**
- `service/ControlFactorDefaults.java`: có danh sách LAND/APARTMENT/CAR, nhánh `default -> LAND`. `allLabels()` ghép cả ba danh sách. Cần thêm SHIP.
- `model/entity/AppraisalAssetGroup.java`: `defaultLandValuationMethod`, `resetForAssetType(newType, now)`.
- `service/GroupAssetTypeReset.java`, theo thứ tự xử lý:
  1. `resetLandDescription`
  2. `resetVehicleDescription` (chỉ biết CAR)
  3. `resetGroupCompares`
  4. `resetSynthetic`
  5. `replaceDefaultFactors`
  6. `resetResult`
  7. `softDeleteConstructions`
  8. `group.resetForAssetType`

  Cần thêm `resetShipDescription`, reset field tàu ở compare và reset tiền tệ.
- `service/ReportGroupResolver.java:~145,175,187`: khôi phục CTXD, land, vehicle. Cần thêm phần khôi phục ship.
- Xóa nhóm: tìm chỗ soft-delete land/vehicle khi xóa nhóm (grep `softDelete` trong `ReportGroupResolver` / `AppraisalReportBasicEditService`) và thêm ship.
- `service/ReportLegalDocumentValidator.java:~72` (`isKnownType`): hiện chỉ cho {1,2,3}. Cần thêm 4.
- `service/AppraisalReportBasicEditService.java:~269`: gọi `changeAssetType` và `typeReset`.

### Tab

- `ReportWorkspaceService` / `ReportWorkspaceGroup`: chỉ trả `assetType`. Danh sách tab nằm ở FE.
- **Pháp lý**: `service/ReportLegalService.java:~81`. CAR gọi `asset.applyCarLegal(input)` (`model/entity/AppraisalReportAsset.java`, cột `car_inspection_*`), còn lại gọi `applyLandLegal`. DTO: `LegalAssetInput`, `LegalAssetView`, `LegalTabData`.
- **Ảnh**: `service/AppraisalImageService.java`. `isCar(group)` quyết định chọn mô tả vehicle hay land, thư mục S3 `vehicle`/`land`, factory `AppraisalImage.ofVehicle`/`ofLand`. Có `deleteAllOfVehicleDescription` và `deleteAllOfLandDescription`.
- **Mô tả xe** (mẫu để làm SHIP):
  - `controller/VehicleDescriptionController.java`: `.../groups/{groupId}/vehicle-description`, GET và PUT.
  - `service/VehicleDescriptionService.java`: `requireCarGroup`, có `NOT_CAR_MESSAGE`.
  - `model/entity/AppraisalVehicleDescription.java`: `emptyCar`, `reviveAsCar`, `apply`, `softDelete`.
  - DTO `VehicleDescriptionData` / `VehicleDescriptionInput`.
  - Đồng bộ sang so sánh nằm ở `AppraisalGroupCompare.applyVehicleSync`. Không có lớp `VehicleCompareSync`.
- `service/LandDescriptionService.java:~93`: `requireLandGroup` chặn CAR. **SHIP sẽ lọt vào nhánh đất**, phải chặn.
- `AreaOutsideConversionService` và `ConstructionDescriptionService` chỉ cho LAND, nên SHIP tự bị chặn.
- **So sánh**:
  - `service/GroupCompareService.java:~89`: CAR không có `comparativeAnalysis` và không gọi tuyến đi bộ.
  - `:~114` (`writeRow`): APARTMENT/CAR không gộp chi tiết diện tích.
  - `model/entity/AppraisalGroupCompare.java:~373`: `applyForm` switch APARTMENT/CAR/mặc định đất. Có `applyCarForm` và đoạn xóa trắng `car_*`.
  - `service/CompareFactorSync.java:~78` (`sourceOf`): nhãn nào đồng bộ từ field nào.
  - `service/CompareInformationSync.java`: đẩy dữ liệu lên kho và chọn từ kho (clone ảnh), dùng `CompareInformation.copyFrom`.
  - `service/CompareInformationSearchService.java`: `nearby` kiểm tra `isKnown`, `searchCars` chỉ cho CAR.
  - `controller/CompareInformationController.java` và `repository/CompareInformationRepository.java`: có `searchByCarName`.
- **Bảng điều chỉnh**:
  - `service/AdjustmentPanelService.java:~62-76`: lọc nhãn, gọi `RegulatoryConstraints.priceUnit(assetType)`.
  - `service/RegulatoryConstraints.java:~21,28`: CAR dùng "đồng/xe", còn lại "đồng/m²".
  - `CommonConstants.resolveRegulatoryConstraintsPriceUnit` và `REGULATORY_CONSTRAINTS_PRICE_UNITS` (regex in đậm trong Word).
  - DTO: `AdjustmentPanelData`, `PanelColumn`.
- **Kết quả**:
  - `service/result/AppraisalResultCalculator.java:~55` (loại lạ trả `empty()`).
  - `:~73-79`: LAND, APARTMENT, **else = car**.
  - `:~85` (`isKnown`), `:~194` (`car()` tạo `CarDetail`).
  - DTO `EvaluationResultResponse` / `EvaluationResultDetailDto` (có `apartmentDetail`, `carDetail`).
  - `EvaluationResultController` / `EvaluationResultService`: GET và PUT `evaluation-result`.
  - `common/util/export/word/AppraisalResultService.java:~58-68,83,436,475`: rẽ nhánh LAND/APARTMENT/CAR khi dựng `EvaluationResultDetailDto` cho Word.

### Word

- Luồng xử lý: `service/AppraisalReportWordService` gọi `common/util/export/word/appraisal/AppraisalWordExporter`, chạy 11 section.
- Cơ chế theo loại là interface `…/appraisal/asset/AssetTypeWriter` với 14 hook:
  - `addressSummaryLine`, `singleGroupTitle`, `manyGroupsDescriptionLabel`
  - `writeDescription`, `valuationTitle`, `valuationSubjectPhrase`
  - `writeCompareTable`, `adjustmentPriceUnit`, `writeValuationSummaryTable`
  - `isRealEstate`, `relatedJuridicalItems`, `hasGroupResultSummary`
  - `writeResultSummaryRows`, `writeAssetImages`
- Registry `…/asset/AssetTypeWriters.java`: `Map.of(LAND, APARTMENT, CAR)`, null trả `OtherAssetWriter.MISSING`, loại lạ trả `UNKNOWN`.
- Hướng dẫn mở rộng nằm trong Javadoc của `AssetTypeWriter.java` (dòng ~15-35) và `AppraisalWordExporter.java` (dòng ~56-65).
- Mẫu cho SHIP: `CarAssetWriter`, `VehicleDescriptionWriter`, `CarCompareTable`, `CarValuationSummaryTable`, `CarAssetImages`, `ResultSummaryRowsBlock.car(...)`.
- Còn `if` theo loại ngoài writer, cần rà lại:
  - `…/section/LegalBasisSection.java:~157-174`
  - `AssetGroupsSection`
  - `AdjustmentTableBlock` (nhãn và đơn vị dòng A/B)
  - `ApartmentDetailWriter:~99`, `VehicleDescriptionWriter:~82`
- **Không sửa** file Word cũ `common/util/export/word/AppraisalReportWordExporter.java` (khoảng 6.300 dòng, chỉ dùng cho golden test).

### Entity / DB / test

- Chưa có entity ship nào.
- Converter: `model/converter/JsonListConverter.java` (abstract: `elementType()`, `emptyAsNull()`), có các lớp con như `PlantDetailsConverter`…
- Flyway: thư mục `db/migration` chỉ có `V20261012104500__create_refresh_tokens.sql`. Cấu hình `out-of-order`, `ignore-migration-patterns=*:missing`, dùng chung `flyway_schema_history` với legacy.
- Test:
  - Repository: `@MysqlDataJpaTest` (MySQL thật, `flyway.enabled=false`, rollback sau mỗi test). Ví dụ: `AppraisalVehicleDescriptionRepositoryTest`, `GroupCompareSaveRepositoryTest`.
  - Controller: `@WebMvcTest`. Ví dụ: `VehicleDescriptionControllerTest`.
  - Service: Mockito. Ví dụ: `VehicleDescriptionServiceTest`, `GroupAssetTypeResetTest`, `CompareFactorSyncTest`, `ControlFactorDefaultsTest`, `CompareInformationSearchServiceTest`, `CompareInformationSyncTest`, `AppraisalImageServiceTest`, `EvaluationResultServiceTest`.
  - Entity: `AppraisalGroupCompareFormTest`.
  - Fixture: `GroupCompareFixtures`, `PanelFixtures`.
  - Word: `…/appraisal/asset/AssetTypeWritersTest` (`UNKNOWN_TYPE = 99`) và `AppraisalWordExporterParityTest` (tắt mặc định, chỉ chạy loại 1-3).

## Frontend

- `F/lib/report-edit/tabs.ts`: `ASSET_TYPE = { LAND: 1, APARTMENT: 2, CAR: 3 }`, `AssetType`, `tabsOf` (LAND có 6 tab, loại khác 5 tab), `tabLabel` (`land` → "Mô tả phương tiện" nếu CAR).
- Literal `1 | 2 | 3` phải đổi sang `AssetType` tại:
  - `api/report-edit/workspace.ts`
  - `compare.ts`
  - `panel.ts`
  - `land-description.ts`
  - `legal.ts`
  - `evaluation-result.ts`
  - `lib/report-edit/adjustment-panel-calc.ts` (có hằng `CAR = 3`, đơn vị "đồng/xe")
- `api/report-edit/compare-informations.ts:~100` hardcode `assetType: 3` cho tìm theo tên xe.
- Màn cơ bản:
  - `lib/appraisal-report-form.ts:~31` (`LEGAL_ASSET_TYPE_OPTIONS`), dùng ở `components/appraisal-report/ReportLegalDocumentList.vue`.
  - `ReportConfigModal.vue`: tab lọc cơ sở thẩm định theo `assetType`.
  - `api/report-edit/basic.ts`.
- `components/appraisal-report/edit/GroupWorkspace.vue`: `byAssetType(land, apartment, car)` có **else = car**. `componentFor(tab)` ánh xạ tab sang component.
- Component theo loại, dưới `components/appraisal-report/edit/`:
  - `description/`: `CarDescriptionTab`, `CarDescriptionForm`, `LandDescriptionTab`, `ApartmentDescriptionTab`
  - `compare/`: `CarCompareTab`, `CarCompareSearchModal`, `LandCompareTab`, `ApartmentCompareTab`, `NearbyCompareMapModal`
  - `legal/LegalTab.vue`: `isCar` chọn `LegalCarAssetForm` hoặc `LegalLandAssetForm`
  - `panel/`: `AdjustmentPanelTab`, `PanelSummaryRows` (`isCar`, đơn vị VND), `ValuationMethodCard` (danh sách theo loại)
  - `results/`: `AssetResultTable` (else là xe), `EvaluationResultTables`, `LandResultTable`, `EvaluationResultTab`
- Hàm thuần `F/lib/report-edit/`:
  - `compare-common.ts`: `fillFromKho`, `emptyColumn`, `carPatch`
  - `vehicle-description-fields.ts`
  - `image-types.ts`: `IMAGE_TYPE.CAR_*`
  - `legal-documents-text.ts`: `isCarGeneratedLine`
  - `adjustment-panel-calc.ts`, `adjustment-panel-form.ts`, `evaluation-result-format.ts`
- Composable: `useCompareForm`, `useDirtyGuard`/`useDirtyTab`, `useRouteTab`.
- Test mẫu: `CarDescriptionTab.spec.ts`, `CarCompareTab.spec.ts`, `CarCompareSearchModal.spec.ts`, `LegalCarAssetForm.spec.ts`.
- Lệnh kiểm tra: `npm run lint` (tự sửa file), `npm run type-check` (chạy sau lint), `npm run test:unit`.
