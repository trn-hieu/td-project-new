# 20 — Tab "Mô tả đất" — LAND (T12–T15)

> Đọc `00-index.md` trước. Subagent: `model: "sonnet"`, có review.

## Hành vi hệ thống cũ

Nguồn: `L controller/AppraisalReportTabController.java:131-188`, `L controller/AppraisalReportController.java:447-454`, `L service/AppraisalLandDescriptionService.java` (đọc toàn bộ, đặc biệt l.125–404), `L entity/AppraisalLandDescription.java`, `L entity/dto/{LandDetail,LandDirection,LandPrice,…}.java`, `LR templates/demo/land-description.html` (l.2–1008 form, 1043–1300 hướng/giá, 1812–2435 số/validate/submit, 2438–2580 sao chép), `LR static/js/land-detail-table-rows.js`, `LR static/js/number-formatter.js`, `LR static/js/common/common.js:255-288,536-625` (tọa độ).

**Nạp**: mô tả đất 1-1 với nhóm (`appraisal_land_description.appraisal_asset_group_id`); `landDetailsJson` rỗng → `[LandDetail.createDefault()]` (1 hạng mục "Đất" → 1 thửa → 1 tờ → 1 dòng); `landDirectionsJson` rỗng → `[LandDirection(FRONT)]`; `landPriceJson` rỗng → `[LandPrice()]`; `landPosition` null → mặc định `group.extractAllAssetStrings()` (regex "Thửa đất số … tờ bản đồ số …" trên địa chỉ từng tài sản, nối xuống dòng — port hàm này); `otherDirections` null → "Hướng còn lại giáp với thửa đất khác"; QĐ giá đất = `report_juridical` theo **tỉnh của mô tả đất**, `category = 1` (LAND_UNIT_PRICE_DECISION), chỉ đọc. Mô tả không tồn tại → cũ lỗi tab; mới 404 "Chưa có mô tả đất".

**Trường (tên = cột entity)**: 
- Vị trí: `categoryType` (1 Nhà ở, 2 Thương mại, 3 Công nghiệp, 4 Căn hộ chung cư), `provinceId`, `districtId`, `wardId` (đổi tỉnh → nạp quận và **toàn bộ phường của tỉnh**; đổi quận → phường của quận; nhãn kèm "(nay là …)" từ `new_*_name`), `frontageCount`, tọa độ (một ô nhập → `latitude, longitude, latitudeDms, longitudeDms`), `landPosition`, `positionDescription`.
- Diện tích: `landArea`, `privateAreaTxt`, `sharedAreaTxt`, `notPlanningLand`, `planningLand`, `encroachedAreaNote`; bảng thửa `landDetails[] {category, plots[] {plotNumber, mapSheets[] {sheetNumber, landUseRows[] {area, purpose, usagePeriod, landOrigin}}}}` (giữ `landPlanning` khi đọc/ghi dù form không có).
- Ảnh 1, 3; `note` + ảnh ghi chú (4); ẩn nhưng vẫn gửi: `minRoadWidth`, `frontRoadWidth`, `sidewalkWidth`.
- `landShape` (1 L, 2 T, 3 Đa giác, 4 Đa giác nhiều góc cạnh, 5 Thóp hậu, 6 Tam giác, 8 Vuông vức, 9 Tương đối vuông vức, 7 Khác) + `landShapeOther` (≤500, chỉ khi 7; đổi khác 7 thì xóa).
- `landDirections[] {type: FRONT|BACK|SIDE|OPEN_SPACE, direction: EAST..SOUTHWEST, sizeTxt, road_frontage, roadFrontageSizeTxt}` (FRONT/OPEN_SPACE 4 trường, BACK/SIDE 2 trường; tối thiểu 1); `useActualLandDirections` + `actualLandDirections[]` (bỏ chọn → lưu null); `otherDirections`; `statePosition`; `supplementaryResolutionId` (chọn từ nghị quyết loại 2 của người dùng + nghị quyết đang chọn, nội dung cắt 80 ký tự; nút "+" tạo nghị quyết mới); `landPrices[] {uuid, landTypeId, landName, unitPrice}` (tối thiểu 1: "Phải giữ ít nhất một loại đơn giá đất."); `surroundingArea`, `liquidity`, `currentStatus`; ảnh 2 (chỉ xem), 5, 9, 6, 7, 15, 8, 10.
- Validate: tọa độ tùy chọn, đúng 2 phần ngăn bằng dấu phẩy, DMS (N/S, E/W) hoặc thập phân trong ±90/±180; đơn giá 0..999.999.999.999.999; kích thước đường 0..99.999.999,99. Server chỉ lưu lat/lng trong khoảng.
- Bảng thửa (`land-detail-table-rows.js`): tổng = Σ diện tích dòng (chỉ hiển thị); thêm hạng mục/thửa/tờ → diện tích mặc định = `landArea − Σ diện tích khác` nếu ≥ 0; hạng mục mới mặc định "Đất"; không xóa được dòng cuối ("Không thể xóa dòng cuối cùng!"); xóa dòng gốc có dòng con thì dòng con đầu lên thay.
- Sao chép/dán: copy `{type:'LAND_DESCRIPTION', fields}` mọi trường trừ hướng, đơn giá, bảng thửa, checkbox hướng thực tế; dán vào form đất khác áp vào các trường.

**Lưu** (`update`, l.262–404): ghi đè **mọi** cột (Khác hệ thống cũ 7: bản mới với LAND chỉ ghi cột đất — danh sách cột căn hộ ở `21-apartment-description.md`); `trimToNull` các text; hướng: lọc `type != null`; hướng thực tế null khi bỏ chọn; giá: lọc dòng có `landTypeId` hoặc `unitPrice`, sinh uuid thiếu; sau đó **đồng bộ sang dòng so sánh type 0** (`findFirstByAppraisalReportIdAndAppraisalAssetGroupIdAndTypeAndDeleteFlag(r, g, 0, 0)`, không có → bỏ qua):
- luôn ghi `latitude, longitude, latitudeDms, longitudeDms`;
- `area_outside_details` từ `landPriceJson` (l.126–188): **Trường hợp 1** — mọi chi tiết hiện có đều uuid null: khớp theo `landTypeId`, gán `landTypeId, landType = landName, price = unitPrice.longValue()` (cắt lẻ), `uuid = lp.uuid`; thêm chi tiết mới cho giá chưa có `landTypeId` (area null). **Trường hợp 2** — đã có uuid: dựng lại danh sách theo thứ tự `landPriceJson`, khớp theo uuid (giữ `area`, `areaConverted`, `areaNotConverted`), không khớp → mới (area null); chi tiết không còn giá → **bỏ**; giá có uuid null bị bỏ qua.
- `area_inside_details`: cùng thuật toán (l.196–260), chi tiết mới `area = "0,00"`.
- (Căn hộ — xem file 21.)

### Task T12: Backend danh mục

**Files:** Create entities `N model/entity/LandType.java` (`land_types`), `District.java`, `Ward.java`, `Resolution.java` (`resolutions`); repositories; `N controller/LandTypeController.java`, `AdministrativeUnitController.java`, `ResolutionController.java`; services tương ứng; DTO `Option`/`TextOption` có sẵn hoặc record mới (`AdministrativeUnitOption(long id, String name, String newName)`, `ResolutionOption(long id, String content)`, `ResolutionCreateRequest(@NotBlank @Size(max=…) String content)`); Test repository + controller cho từng cái.

**Interfaces:**
- `GET /api/land-types` → `List<Option>` (chưa xóa, `created` giảm dần như cũ `findAllActive`).
- `GET /api/districts?provinceId=` ; `GET /api/wards?districtId=` hoặc `?provinceId=` → `List<AdministrativeUnitOption>` (port truy vấn của `L controller/api/*Province*/*District*/*Ward*` — tìm bằng grep `wards-by-province`).
- Tỉnh: dùng lại danh sách tỉnh đã có (kiểm tra `ReportFormData` / `ReportJuridicalController`); chưa có endpoint list đơn giản thì thêm `GET /api/provinces` trong `AdministrativeUnitController`.
- `GET /api/resolutions?type=2&selectedId=` → nghị quyết loại 2 do người gọi tạo + nghị quyết `selectedId`; `POST /api/resolutions` `{content}` → tạo loại "2", `created_by` = người gọi (port `L controller/api/ResolutionApiController` hoặc tên tương đương).
- [ ] **Step 1–4:** test FAIL → cài đặt → PASS. Thêm dòng `SecurityConfig` nếu quy tắc mặc định không phải `authenticated()`.
- [ ] **Step 5: Commit** `feat: API danh mục loại đất, đơn vị hành chính, nghị quyết`.

### Task T13: Backend Mô tả đất + `LandCompareSync`

**Files:**
- Modify: `N model/entity/AppraisalLandDescription.java` (map **toàn bộ** cột còn thiếu so với `L entity/AppraisalLandDescription.java`; `land_details_json`, `land_directions_json`, `actual_land_directions_json` với converter mới), `N model/entity/AppraisalGroupCompare.java` (chỉ thêm `latitude, longitude, latitude_dms, longitude_dms` + phương thức `applyLandSync(...)`; phần còn lại ở T22), `N model/dto/LandPrice.java` (nếu thiếu `uuid`/`landName`)
- Create: `N model/dto/LandDetail.java` (+ record lồng `LandPlot`, `MapSheet`, `PlotLandUseRow`), `LandDirection.java`, converters `LandDetailListConverter`, `LandDirectionListConverter`; `N model/dto/LandDescriptionData.java`, `LandDescriptionInput.java`; `N service/LandCompareSync.java` (final, thuần); `N service/LandDescriptionService.java`; `N service/AssetAddressParcels.java` (port `extractAllAssetStrings`); `N controller/LandDescriptionController.java`
- Test: `NT repository/AppraisalLandDescriptionRepositoryTest.java` (đọc JSON thật của hệ thống cũ: tạo dữ liệu bằng chuỗi JSON copy từ DB/legacy test, khóa `road_frontage`, `road_frontage_size`, …), `NT service/LandCompareSyncTest.java`, `NT service/LandDescriptionServiceTest.java`, `NT service/AssetAddressParcelsTest.java`, `NT controller/LandDescriptionControllerTest.java`

**Interfaces:**
- Produces:
  - `LandCompareSync`: `static List<AreaOutsideDetail> syncOutside(List<AreaOutsideDetail> current, List<LandPrice> prices)`, `static List<AreaInsideDetail> syncInside(List<AreaInsideDetail> current, List<LandPrice> prices)` (đúng hai trường hợp ở trên).
  - `record LandDescriptionData(long id, int assetType, …mọi trường…, List<String> unitPriceJuridicals, String defaultLandPosition)`; `record LandDescriptionInput(…mọi trường cho cả LAND và APARTMENT…, Integer syncToApartmentBuildingEnabled, Long syncFromApartmentBuilding)` — tọa độ gửi 4 trường đã tách ở FE (`latitude, longitude: BigDecimal`, `latitudeDms, longitudeDms: String`).
  - `GET/PUT …/groups/{groupId}/land-description` (nhóm LAND hoặc APARTMENT; CAR → 400). PUT trả `ApiResponse.ok(LandDescriptionData, "Lưu mô tả thành công")`.
  - T13 chỉ xử lý nhóm LAND (YAGNI); GET/PUT với nhóm APARTMENT trả 400 "Chưa hỗ trợ mô tả căn hộ" cho đến khi T16 thêm nhánh căn hộ.

- [ ] **Step 1: Test thất bại** `LandCompareSyncTest`: `uuid_null_khớp_theo_loại_đất_và_gán_uuid`, `uuid_null_thêm_loại_mới_area_null`, `uuid_khớp_giữ_diện_tích` (giữ cả `areaConverted`/`areaNotConverted`), `uuid_không_còn_giá_bị_bỏ`, `thứ_tự_theo_bảng_giá`, `giá_cắt_phần_lẻ`, `trong_qh_mới_area_0,00`; `LandDescriptionServiceTest`: `mặc_định_khi_rỗng`, `lưu_đất_không_ghi_cột_căn_hộ`, `bỏ_chọn_hướng_thực_tế_lưu_null`, `tọa_độ_ngoài_khoảng_không_lưu`, `đồng_bộ_compare_type0`, `không_có_compare_type0_vẫn_lưu`; `AssetAddressParcelsTest` (lấy ví dụ từ regex cũ).
- [ ] **Step 2–4:** FAIL → cài đặt → `./mvnw -q test -Dtest='LandCompareSyncTest,LandDescriptionServiceTest,AssetAddressParcelsTest,AppraisalLandDescriptionRepositoryTest,LandDescriptionControllerTest'` PASS + toàn bộ `service/result` (entity `AppraisalLandDescription` bị sửa).
- [ ] **Step 5: Commit** `feat: API mô tả đất và đồng bộ sang tài sản thẩm định`.

### Task T14: FE Mô tả đất — phần 1

**Files:** Create `F api/report-edit/land-description.ts`, `F api/report-edit/master-data.ts` (land types, provinces/districts/wards, resolutions), `F lib/report-edit/land-detail-table.ts`, `F lib/report-edit/coordinate-input.ts` (tách ô tọa độ → 4 trường, dùng lại `isLatitude`, `isLongitude`, `toDecimal` của `F lib/coordinates.ts`), `F components/appraisal-report/edit/description/LandDescriptionTab.vue`, `LocationFields.vue` (tỉnh/quận/phường + tọa độ, dùng lại ở T17), `LandDetailTable.vue`; Modify `GroupWorkspace.vue` (map `land` cho LAND). Test tương ứng trong `__tests__`.

**Interfaces:** `land-detail-table.ts`: các hàm thuần trên cây `LandDetail[]` — `totalArea(details): number`, `addCategory/addPlot/addSheet/addLandUseRow(details, path, landArea): LandDetail[]` (áp diện tích còn lại), `removeRow(details, path): { details; error?: string }`; `coordinate-input.ts`: `splitCoordinate(text): { latitude: number|null; longitude: number|null; latitudeDms: string|null; longitudeDms: string|null } | { error: string }`, `displayCoordinate(lat, lng, latDms, lngDms): string`.

- [ ] **Step 1–4:** test FAIL (`land-detail-table.spec.ts`: `tổng_diện_tích`, `thêm_thửa_điền_phần_còn_lại`, `không_xóa_dòng_cuối`, `xóa_gốc_đẩy_con_lên`; `coordinate-input.spec.ts`: `dms`, `thập_phân`, `ngoài_khoảng_lỗi`, `rỗng_null`; tab: `đổi_tỉnh_nạp_quận_và_phường_của_tỉnh`, `lưu_gửi_tọa_độ_đã_tách`) → cài đặt (`DecimalInput` cho diện tích, `SelectBox`/`SearchSelect`) → PASS.
- [ ] **Step 5: Commit** `feat: mô tả đất phần vị trí, diện tích, bảng thửa`.

### Task T15: FE Mô tả đất — phần 2

**Files:** Create `F components/appraisal-report/edit/description/LandShapeField.vue`, `LandDirectionList.vue`, `LandPriceList.vue`, `ResolutionSelect.vue` (+ modal thêm), `DescriptionCopyPaste.vue` + `F lib/report-edit/description-clipboard.ts` (dùng chung T17/T19); Modify `LandDescriptionTab.vue` (gắn ảnh `ImageSection` các loại 1, 2 chỉ xem, 3, 4, 5, 9, 6, 7, 15, 8, 10). Test tương ứng.

**Interfaces:** `description-clipboard.ts`: `toClipboardPayload(kind: 'LAND_DESCRIPTION' | 'LAND_APARTMENT_DESCRIPTION' | 'CAR_DESCRIPTION', values): string`, `fromClipboardPayload(kind, text): Partial<values> | null` (sai loại → null + toast "Dữ liệu sao chép không phù hợp"). Khác cũ: dán bằng nút "Dán dữ liệu" (đọc clipboard) thay vì dán vào một ô bất kỳ.

- [ ] **Step 1–4:** test FAIL (`hướng_tối_thiểu_một`, `mặt_hậu_chỉ_hai_trường`, `bỏ_hướng_thực_tế_ẩn_khối`, `đơn_giá_tối_thiểu_một_báo_lỗi`, `hình_dáng_khác_7_xóa_mô_tả`, `clipboard_bỏ_hướng_giá_bảng_thửa`, `dán_sai_loại_trả_null`) → cài đặt (`CurrencyInput` đơn giá) → PASS.
- [ ] **Step 5: Commit** `feat: mô tả đất phần hướng, đơn giá, ảnh, sao chép`.
