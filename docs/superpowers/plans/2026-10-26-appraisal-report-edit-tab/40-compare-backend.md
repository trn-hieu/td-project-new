# 40 — Tab "Thông tin so sánh" — Backend (T22–T26)

> Đọc `00-index.md` trước. Subagent: `model: "sonnet"`, có review. **Đây là phần đồng bộ phức tạp nhất — đọc kỹ code cũ ở dòng được dẫn.**

## Hành vi hệ thống cũ

Nguồn: `L controller/AppraisalReportTabController.java:226-255`, `L controller/AppraisalReportController.java:467-551`, `L service/AppraisalGroupCompareService.java` (đọc **toàn bộ**: update l.91–385, merge l.423–541, đồng bộ factor l.576–676 & 816–966, đẩy kho l.1034–1141, area-outside l.1147–1222), `L service/ComparisonAssetImageService.java:41-80`, `L service/ComparisonAssetImageSyncService.java`, `L service/impl/CompareInformationServiceImpl.java:107-161`, `L repository/CompareInformationRepository.java`, `L service/GoogleRouteService.java`, `L entity/AppraisalGroupCompare.java`, `L entity/CompareInformation.java`.

**Cấu trúc**: mỗi nhóm luôn có đúng 4 dòng `appraisal_group_compares` (type 0 = tài sản thẩm định, 1–3 = TSSS 1–3), tạo sẵn khi tạo báo cáo (`N service/ReportGroupDetailsWriter.java` đã làm). Lưu không tạo/xóa dòng; khớp dòng theo `id` (id không thuộc nhóm → bỏ qua).

**Lưu (`update`)** với mỗi dòng:
1. Ghi đè chuỗi (rỗng → null): address, source, information, contact, transactionStatus, transactionListingTime, legal, structure, landUseTerm, landDimension, roadType, roadWidth, landUsePurpose, controlAmplitude.
2. Số: totalArea, areaOutside, areaInside, useArea như gửi; forSale, trade, estimate, priceInPlanning, convertTax, unitPriceConversionTax, constructionPrice, plantPrice, convertToStandardCompareUnitPrice, instructionPrice, instructionPriceAvgValue, totalControlPrice, totalControlValue null → 0; differenceInstructionPrice → 0f; totalControlTime → 0. `unitPricePlanning` **không** cập nhật.
3. `marketPrice` = trade nếu ≠ 0, ngược lại estimate nếu ≠ 0, ngược lại giá trị không null, ngược lại **0** (sửa từ `0111L`).
4. `landUseRightPrice`: LAND lấy từ request; APARTMENT = `apartmentOwnershipRightPrice` từ request (ghi cả hai cột); CAR: `apartmentOwnershipRightPrice = 0`, `landUseRightPrice = (trade ?? 0) + (estimate ?? 0)`.
5. Căn hộ: `apartmentUnitKind` chỉ 1/2; `floorPosition` 1..4, `floorPositionOther` chỉ khi 4; ba diện tích null → 0.0; basis ∉ {1,2} → 1. Ô tô: chuỗi trim, năm/km/CLCL như gửi.
6. `mergeAreaInsideDetails`: type 0 chỉ cập nhật `area` cho uuid đã có; TSSS: uuid null → mới (uuid mới), uuid khớp → copy trường, **uuid không khớp → thêm như dòng mới giữ uuid** (sửa — cũ: bỏ). `mergeAreaOutsideDetails`: type 0 khớp uuid cập nhật `area`, có `areaConverted` thì `areaNotConverted = max(0, area − converted)`; TSSS khớp uuid, rồi `landTypeId`, không có thì thêm.
7. `structureDetails` như gửi; `plantDetails` sinh uuid thiếu.
8. Tọa độ: DMS trim; lat/lng chỉ ghi khi null hoặc trong ±90/±180, sai giữ giá trị cũ.
9. Liên kết kho (TSSS 1–3) — xem T24.
Sau vòng lặp: đồng bộ sang `control_compare_factor` (T23, **chỉ nhóm này**), `appraisal_asset_groups.comparative_analysis = trim/null` (LAND, APARTMENT), `saveAll`, tuyến đi bộ async (T26).

**Sửa (Khác hệ thống cũ 7)**: chỉ ghi các cột mà form của loại tài sản đó có. Bước đầu của T23: implementer **lập bảng "cột được ghi theo loại tài sản"** bằng cách liệt kê các ô `name="groupCompares[i].<field>"` trên 3 template cũ (`comparative-information.html` → LAND, `comparative-information-apartment.html` → APARTMENT, `comparative-information-car.html` → CAR), cộng các cột tính ở bước 2–5 và liên kết kho; mọi cột không có trong bảng của loại đó giữ nguyên giá trị DB. Ví dụ: form ô tô không có Địa chỉ, Tọa độ, Tình trạng giao dịch → lưu ô tô không ghi các cột đó. Bảng này đặt trong Javadoc `GroupCompareWriter` và là dữ liệu của test `lưu_ô_tô_không_đụng_cột_đất`.

**Đồng bộ sang bảng điều chỉnh** (`computeSyncedValuesForLabelName`, l.816–966) — khớp theo **text** `c_label` của factor chưa xóa, ghi `c_value`, `c_value_cp1..3` từ dòng type 0..3 (null → ""):

| Nhãn | Loại | Nguồn |
|---|---|---|
| Diện tích | LAND | `areaOutside` 2 lẻ kiểu châu Âu |
| Diện tích | APARTMENT | basis 2 → `netFloorArea`, khác → `wallCenterArea` |
| Diện tích cơi nới | APARTMENT | `extensionArea`, 0 → null |
| Loại căn hộ | APARTMENT | "Căn góc" / "Căn thường" |
| Nội thất | APARTMENT | `interiorDescription` |
| Vị trí tầng | APARTMENT | Tầng cao / Tầng trung / Tầng thấp / text "Khác" |
| Chiều rộng đường, Loại đường, Kích thước, Số mặt tiền, Mục đích sử dụng đất | mọi loại | roadWidth, roadType, landDimension, numberOfFrontages, landUsePurpose |
| Xuất xứ, Năm sản xuất, Chất lượng còn lại (2 lẻ), Số km đã đi | CAR | carOrigin, year, CLCL, odometer |

Factor có `control_compare_factor_label_id = 8`: `residential_land_ratio_pct*` = `areaOutside ≤ 0 ? 0 : area(chi tiết ngoài QH đầu tiên có landTypeId = 1) × 100 / areaOutside` HALF_UP 2 lẻ; dòng null/areaOutside null → null. **Sửa (Khác 6)**: chỉ nhóm đang lưu; nhãn không áp dụng cho loại tài sản (vd. "Diện tích" của CAR) → không ghi (cũ: ghi "").

**Kho TSSS** (`compare_information`) — dòng so sánh có `compare_information_id` (chọn từ kho, FK) và `sync_to_compare_information_id` (đã đẩy lên kho, Long), loại trừ nhau:
- Request có `compareInformationId` → gán FK (không kiểm tra deleted, không thấy → null), `syncTo = null`; nếu liên kết đổi: xóa ảnh dòng đã clone từ kho cũ (`synced_from_compare_information_id = oldId`), clone ảnh kho mới vào dòng (bỏ qua nếu multiset `file_path` đã khớp). **Không ghi dòng kho.**
- Ngược lại `syncToCompareInfo == 1` → (có liên kết kho cũ thì xóa ảnh clone) → đẩy lên kho: dedupe **chỉ theo** `syncToCompareInformationId` (có và tìm thấy → cập nhật, không → tạo mới); copy toàn bộ trường (type = slot 1..3, assetType = loại nhóm mặc định LAND, các chuỗi/số/diện tích/chi tiết JSON/căn hộ trừ `apartmentOwnershipRightPrice`/ô tô, lat/lng BigDecimal → Double khi khác null, `deleted = 0`, `updated = now`, tạo mới thêm `created`, `created_by`); clone ảnh dòng → kho (khác multiset thì xóa ảnh kho an toàn S3 rồi clone cùng `file_path`); gán `syncTo = ci.id`, FK = null.
- Ngược lại → xóa ảnh clone từ kho (nếu có), cả hai liên kết null; dòng kho đã đẩy **giữ nguyên**.
- Ai được tick: KSV/TPTD (UI). Người khác: nếu dòng đã đồng bộ thì UI gửi `syncToCompareInfo = 1` để tiếp tục đồng bộ (giữ hành vi này).
- Ảnh TSSS: `GET /api/comparison-asset-images?appraisalGroupCompareId=`, `POST …/upload` (multipart, key `{assetFolder}compare/{reportId}/{agcId}/{uuid}/{fileName}`), `DELETE /{id}` (chỉ xóa S3 khi không dòng nào khác dùng cùng `file_path`). Ảnh gắn ngay, không chờ lưu form.

**Tìm TSSS**: gần — bán kính **3000 m**, `ST_Distance_Sphere(POINT(lng,lat), POINT(:lng,:lat), 6378000)`, `deleted = 0`, có tọa độ, lọc `asset_type`; theo tên xe — `assetType = 3`, `car_name LIKE %kw%` (từ khóa bắt buộc), `updated DESC`, trang 20.

**Tuyến đi bộ** (LAND/APARTMENT, async sau commit): tọa độ không hợp lệ → `clearAsync(groupId)`; tọa độ thay đổi hoặc `walking_route_polyline` null → `recalculateAsync` ghi `appraisal_asset_groups.walking_route_polyline`.

### Task T22: Map đủ entity + API GET so sánh

**Files:** Modify `N model/entity/AppraisalAssetGroup.java` (cột `comparative_analysis`), `N model/entity/AppraisalGroupCompare.java` (mọi cột còn thiếu — so với `L entity/AppraisalGroupCompare.java`; quan hệ `compareInformation` `@ManyToOne(LAZY)` hoặc cột Long — chọn cột Long nếu không điều hướng), `N model/entity/CompareInformation.java` (mọi cột), `N model/dto/AreaInsideDetail.java`, `AreaOutsideDetail.java`, `StructureDetail.java` (`@JsonIgnoreProperties(ignoreUnknown = true)`); Create `N model/dto/PlantDetail.java` + converter, `N model/dto/GroupCompareData.java`, `GroupCompareRow.java`, `N service/GroupCompareService.java` (phần đọc), `N controller/GroupCompareController.java`. Test: repository (ghi/đọc đủ cột hai entity, JSON có khóa lạ vẫn đọc được), service, controller; chạy lại `service/result`, `TsssReport*`, `CoordinateSearch*` test (entity dùng chung).

**Interfaces:** `GET …/groups/{g}/compare` → `GroupCompareData(int assetType, List<GroupCompareRow> rows /* 4 dòng theo type */, String comparativeAnalysis, boolean canSyncToKho /* người gọi có authority TPTD (KSV có TPTD) */)`; `GroupCompareRow` gồm mọi trường của dòng + `Long compareInformationId`, `Long syncToCompareInformationId`.

- [ ] **Step 1–4:** FAIL → cài đặt → PASS. **Step 5: Commit** `feat: map đủ dòng so sánh và kho TSSS, API đọc tab so sánh`.

### Task T23: Lưu so sánh + `CompareFactorSync`

**Files:** Create `N model/dto/GroupCompareSaveRequest.java` (`List<@Valid GroupCompareRowInput> rows`, `String comparativeAnalysis`), `GroupCompareRowInput.java` (trường như `GroupCompareRow` + `Integer syncToCompareInfo`), `N service/GroupCompareWriter.java` (thuần: áp input lên entity theo loại tài sản, bước 1–8), `N service/AreaDetailsMerge.java` (thuần), `N service/CompareFactorSync.java` (thuần: `Map<String, String[]> syncedValues(String label, int assetType, AppraisalGroupCompare[] byType)` + `BigDecimal residentialLandRatio(AppraisalGroupCompare)`), Modify `GroupCompareService` (`save`), `N model/entity/ControlCompareFactor.java` (cột `c_value*`, `residential_land_ratio_pct*`; bỏ `updatable=false`), `N model/entity/AppraisalAssetGroup.java` (phương thức ghi `comparative_analysis`), `GroupCompareController` (`PUT …/groups/{g}/compare` → `ApiResponse.ok(null, "Lưu thông tin so sánh thành công")`). Test: `GroupCompareWriterTest`, `AreaDetailsMergeTest`, `CompareFactorSyncTest`, `GroupCompareServiceTest`, controller.

- [ ] **Step 1: Test thất bại** gồm: `market_price_ưu_tiên_giao_dịch`, `market_price_cả_hai_null_là_0`, `căn_hộ_ghi_cả_hai_cột_giá`, `ô_tô_giá_qsd_là_tổng`, `unitPricePlanning_không_đổi`, `lưu_ô_tô_không_đụng_cột_đất`, `type0_chỉ_sửa_diện_tích_chi_tiết`, `tsss_trong_qh_uuid_lạ_được_giữ`, `ngoài_qh_khớp_landTypeId`, `areaNotConverted_không_âm`, `tọa_độ_ngoài_khoảng_giữ_cũ`, `id_nhóm_khác_bị_bỏ_qua`, `chỉ_đồng_bộ_factor_nhóm_đang_lưu`, `ô_tô_không_đồng_bộ_diện_tích`, `căn_hộ_diện_tích_theo_basis`, `tỷ_lệ_đất_ở_nhãn_8`, `comparativeAnalysis_trim`.
- [ ] **Step 2–4:** FAIL → cài đặt → PASS. **Step 5: Commit** `feat: lưu tab so sánh và đồng bộ sang bảng điều chỉnh`.

### Task T24: Đồng bộ kho TSSS + ảnh TSSS

**Files:** Create `N model/entity/ComparisonAssetImage.java` (bảng theo entity cũ — đọc `L entity/ComparisonAssetImage*.java`), repository, `N service/ComparisonImageService.java` (list/upload/delete + `cloneCompareImagesToRow`, `cloneRowImagesToKho`, `removeImagesSafely`), `N service/CompareInformationSync.java`, `N controller/ComparisonImageController.java` (`GET/POST …/compare-rows/{rowId}/images`, `DELETE …/compare-images/{imageId}`); Modify `GroupCompareService.save` (gọi `CompareInformationSync` cho TSSS 1–3). Test: repository, `CompareInformationSyncTest`, `ComparisonImageServiceTest`, controller.

- [ ] **Step 1: Test thất bại**: `chọn_từ_kho_không_ghi_kho`, `đổi_kho_xóa_ảnh_clone_cũ_và_clone_mới`, `ảnh_kho_đã_khớp_không_clone_lại`, `đẩy_lần_đầu_tạo_dòng_kho_và_created_by`, `đẩy_hai_lần_cùng_một_dòng_kho`, `đẩy_không_copy_apartmentOwnershipRightPrice`, `bỏ_tick_giữ_dòng_kho_và_null_liên_kết`, `xóa_ảnh_dùng_chung_file_path_không_xóa_s3`.
- [ ] **Step 2–4:** FAIL → cài đặt → PASS. **Step 5: Commit** `feat: đồng bộ kho tài sản so sánh và ảnh`.

### Task T25: Tìm TSSS

**Files:** Modify `N repository/CompareInformationRepository.java` (native `@Query` trả id trong bán kính + loại tài sản, comment lý do native = `ST_Distance_Sphere`; tham khảo khung bounding-box của `N repository/CoordinateSearchRepository.java`; tìm tên xe bằng JPQL + `LikePattern`), Create `N model/dto/CompareInformationDetail.java` (đủ trường để điền một cột so sánh, gồm chi tiết JSON), `N service/CompareInformationSearchService.java`, `N controller/CompareInformationController.java`. Test repository (`@MysqlDataJpaTest`: điểm 2,9 km có, 3,1 km không; khác loại không; đã xóa không), service, controller.

**Interfaces:** `GET /api/compare-informations/nearby?latitude&longitude&assetType` → `List<CompareInformationDetail>`; `GET /api/compare-informations/search?keyword&assetType=3&page&size=20` → `Page<CompareInformationDetail>` (keyword trống → 400).

- [ ] **Step 1–5:** FAIL → cài đặt → PASS → commit `feat: tìm tài sản so sánh trong kho`.

### Task T26: Tuyến đường đi bộ

**Files:** Create `N service/WalkingRouteService.java` (port `L service/GoogleRouteService.java`: gọi Google Routes API, ghi `walking_route_polyline`), `N config/GoogleRoutesProperties.java` (`@ConfigurationProperties`, key lấy từ cấu hình như các key bản đồ hiện có — xem `MapController`/`MapKeyResponse`), Modify `N model/entity/AppraisalAssetGroup.java` (`walking_route_polyline`), `GroupCompareService.save` (phát sự kiện, xử lý `@Async @TransactionalEventListener(AFTER_COMMIT)` như `PasswordResetMailListener`). Test: `WalkingRouteServiceTest` (HTTP giả lập; key trống → bỏ qua, không lỗi), `GroupCompareServiceTest` (`tọa_độ_đổi_phát_sự_kiện_tính_lại`, `tọa_độ_sai_phát_sự_kiện_xóa`, `ô_tô_không_phát`).

- [ ] **Step 1–5:** FAIL → cài đặt → PASS → commit `feat: tính lại tuyến đường đi bộ sau khi lưu so sánh`.
