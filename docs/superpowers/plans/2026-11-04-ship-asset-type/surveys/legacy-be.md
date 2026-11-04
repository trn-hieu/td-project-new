# Khảo sát backend legacy SHIP — trạng thái tại `82193fe5`

> Tài liệu này tổng hợp hai lần khảo sát: bản tại `a9d647d4` và phần thay đổi từ `a9d647d4` đến `82193fe5`. Nội dung đã được hợp nhất theo trạng thái **cuối cùng**.
> Số dòng là tương đối, dấu `~` nghĩa là gần đúng. Trước khi port luôn đọc lại code bằng `git -C td-project-legacy show 82193fe5:<path>`.
> L = `src/main/java/com/jvb/thanhdo/`.

## 1. Hằng số

### `L/common/CommonConstants.java`

- `AppraisalAssetType.SHIP = 4`. Các loại còn lại: LAND=1, APARTMENT=2, CAR=3.
- `ValuationBasisAssetType.SHIP = 4`. Các loại còn lại: COMMON=0, LAND=1, APARTMENT=2, CAR=3.
- `DEFAULT_CONTROL_COMPARE_FACTOR_LABELS_SHIP` có 8 nhãn, giữ đúng thứ tự:
  1. Năm đóng
  2. Nơi đóng
  3. Ngoại quan tổng thể
  4. Khả năng vận hành
  5. Trọng tải toàn phần
  6. Chất lượng còn lại
  7. Công suất (CV)
  8. Cấp phương tiện
- `ControlCompareFactorLabels` có các hằng `NAM_DONG`, `NOI_DONG`, `NGOAI_QUAN_TONG_THE`, `TRONG_TAI_TOAN_PHAN`, `CONG_SUAT_CV`, `CAP_PHUONG_TIEN`. Hằng `NAM_DONG_HOAN_CAI` cũ đã bị xóa. Hai nhãn "Khả năng vận hành" và "Chất lượng còn lại" dùng chung với ô tô.

**Tiền tệ** (dòng ~434-497):
- `CurrencyCode.VND="VND"`, `USD="USD"`.
- `resolveCurrencyCode(Integer assetType, String code)`: chỉ trả USD khi `assetType==SHIP` và `code` bằng "USD" (không phân biệt hoa thường). Mọi trường hợp khác trả VND.
- `currencyUnitLabel(code)`: USD hoặc VND.
- `currencyMoneyLabel(code)`: USD hoặc "đồng".
- `RegulatoryConstraintsPriceUnit.SHIP_VND="đồng"`, `SHIP_USD="USD"`.
- Overload `resolveRegulatoryConstraintsPriceUnit(assetType, currencyCode)`:
  - Tàu trả SHIP_USD hoặc SHIP_VND.
  - Loại khác gọi bản cũ: đất và căn hộ trả "đồng/m²", ô tô trả "đồng/xe".
  - Hai đơn vị của tàu **không** được đưa vào `REGULATORY_CONSTRAINTS_PRICE_UNITS` (danh sách regex bôi đậm).

### `L/common/ReadMoneyUtils.java`

`doc(long, String currencyCode)` đọc như bản VND. Nếu là USD thì đổi hậu tố " đồng" thành " đô la Mỹ".

### `ReportConstants.AppraisalImageType` (ảnh tàu)

| Mã | Hằng |
|---|---|
| 21 | SHIP_FRONT |
| 22 | SHIP_BOW |
| 23 | SHIP_STERN |
| 24 | SHIP_HULL_SIDE |
| 25 | SHIP_DECK |
| 26 | SHIP_IMO_NUMBER |
| 27 | SHIP_ANCHOR_SYSTEM |
| 28 | SHIP_WHEELHOUSE_CABIN |
| 29 | SHIP_CARGO_HOLD |
| 30 | SHIP_ENGINE_SYSTEM |

### Test spec: `src/test/.../common/CurrencyCodeTest.java` (5 case)

- `shipCanUseUsd`: SHIP với "USD" và "usd" đều ra USD.
- `nonShipOrUnknownCodeFallsBackToVnd`: LAND/USD, CAR/USD, null/USD, SHIP/null, SHIP/"EUR" đều ra VND.
- `labels`: kiểm tra nhãn đơn vị (USD/VND) và nhãn tiền (USD/"đồng").
- `regulatoryUnitForShipHasNoSquareMeter`: SHIP/USD ra "USD", SHIP/VND ra "đồng", LAND/USD ra "đồng/m²", CAR/null ra "đồng/xe".
- `readMoneyUsd`: `doc(x,"VND") == doc(x)`. Bản USD đổi hậu tố. `doc(0,"USD") == "Không đô la Mỹ"`.

## 2. Entity và DTO (trạng thái cuối)

### `AppraisalAssetGroup`

- `currencyCode` String: cột `currency_code`, length 10, NOT NULL, mặc định "VND".
- `exchangeRate` Long: cột `exchange_rate`, NOT NULL, mặc định 1.
- `currencyNote` TEXT.
- Quan hệ `@OneToOne(mappedBy) appraisalShipDescription`.
- Chỉ nhóm SHIP mới được dùng USD.

### `AppraisalShipDescription` (bảng `appraisal_ship_description`)

**Khóa và quan hệ**
- `appraisal_report_id`: FK, NOT NULL.
- `appraisal_asset_group_id`: FK, UNIQUE (quan hệ 1-1).
- `delete_flag`: INT, mặc định 0.
- `created`: gán trong `@PrePersist`.
- `@OneToMany appraisalImages`, kèm `@Where("deleted = 0 OR deleted IS NULL")`.

**Thông số chính**
- `ship_name` (500), `registration_number` (255), `ship_type_purpose` (500), `registry_number` (255), `owner_name` (500).
- `build_year` INT, `build_place` TEXT. Hai cột này thay cho `build_year_place` đã bị DROP.
- Kích thước: `dim_length`, `dim_breadth`, `dim_depth`, `dim_draft`, `dim_length_max`, `dim_breadth_max`, đều DECIMAL(10,2).
- `free_board` DECIMAL(10,2), `material` (255), `gross_tonnage` INT, `ship_class` (255), `operating_area` (255).
- `cargo_capacity` DECIMAL(12,2), `deadweight` DECIMAL(12,2), `operation_note` TEXT.
- `mooring_location` VARCHAR(255). Đây là nơi neo đậu, dùng cho câu mở đầu mục Tình trạng trong Word.

**Thiết kế**
- `design_symbol` (255), `design_year` INT, `design_certificate_number` (255), `appraisal_number` (255), `conversion_year_place` (500).

**Neo**
- `anchor_bow_type` (100), `anchor_bow_quantity` INT, `anchor_bow_total_weight_kg` DECIMAL(12,2).
- `anchor_stern_type` (100), `anchor_stern_quantity` INT, `anchor_stern_total_weight_kg` **TEXT**.
- `anchor_chain_cable` (255), `anchor_windlass_type` (100).

**Lái**
- `steering_rudder_type`, `steering_rudder_quantity`, `steering_rudder_total_area_m2` (12,2).
- `steering_main_gear_type`, `steering_main_gear_torque_tm` (12,2).
- `steering_backup_gear_type`, `steering_backup_gear_torque_tm`.

**Chằng buộc**
- `mooring_rope_quantity` INT, `mooring_rope_total_length_m` (12,2), `mooring_bollard_type` (100), `mooring_bollard_quantity` **TEXT**.

**Thiết bị kéo** (mới, đều TEXT)
- `towing_bollard_type`, `towing_hook_type`, `towing_winch_type`, `towing_rope_type`.

**Máy chính**
- `main_engine_quantity` INT, `main_engine_brand` (255), `main_engine_serial_number` (255), `main_engine_manufacture_year` INT, `main_engine_power_hp` DECIMAL(10,2).

**Máy phụ**
- `aux_engine_quantity` INT, `aux_engine_brand` (255), `aux_engine_power_hp` (10,2).

**Điện**
- `electric_generator` (100), `electric_generator_power_kva` (100), `electric_generator_voltage` (255).
- `electric_battery_quantity` INT, `electric_battery_voltage_v` (10,2), `electric_battery_total_capacity` (255).

**Khí nén**
- `compressed_air_tank_quantity`, `compressed_air_compressor_quantity`: **TEXT**.

**Nâng**
- `lifting_serial_number`, `lifting_max_reach_capacity`, `lifting_min_reach_capacity`, đều 255.

**Chữa cháy**
- `fire_pump_quantity` INT, `fire_pump_total_flow_m3` (12,2).
- `fire_gas_type` (100), `fire_gas_total_weight_kg` (12,2).
- `fire_foam_type` (100), `fire_foam_total_volume_l` TEXT, `fire_foam_pusher_quantity` TEXT, `fire_foam_pusher_volume_l` TEXT.
- `fire_portable_foam` (100).
- `fire_co2_bottle_quantity` INT, `fire_co2_bottle_weight_kg` (12,2).
- `fire_powder_bottle_quantity` INT, `fire_powder_bottle_weight_kg` (12,2).
- `fire_foam_bottle_quantity` TEXT, `fire_foam_bottle_volume_l` TEXT.
- `fire_manual_tools`, `fire_layout_plan`: **TINYINT(1)**, kiểu Java Boolean.

**Cứu sinh**
- `lifesaving_ring_buoy` (100), `lifesaving_life_jacket` (100), `lifesaving_raft_boat_quantity` INT, `lifesaving_raft_boat_total_capacity` INT.

**Cứu đắm**
- `damage_control_bilge_pump_quantity` INT, `damage_control_bilge_pump_total_flow_m3` (12,2), `damage_control_tools` (100), `damage_control_bilge_alarm` (100).

**Tín hiệu**
- 16 đèn, đều Boolean: `signal_light_side_red`, `_side_green`, `_stern_white`, `_mast_white`, `_mast_green`, `_mast_red`, `_anchor`, `_towing_red`, `_towing_green`, `_yellow`, `_flash_white`, `_flash_yellow`, `_flash_red`, `_other`, `_half_green_red`, `_spare`.
- `signal_sound` (100).
- 10 vật hiệu, đều Boolean: `signal_shape_black_triangle`, `_white_triangle`, `_black_square`, `_black_rectangle`, `_black_circle`, `_white_circle`, `_black_diamond`, `_flag_set`, `_parachute_flare`, `_line_thrower`.

**Chạy tàu**
- Boolean: `navigation_binoculars`, `_clock`, `_depth_sounder`, `_order_bell`, `_inclinometer`, `_draft_gauge`, `_gps`, `_compass`, `_command_pa`, `_radar`, `_epirb`.
- Chuỗi (100): `navigation_radio`, `navigation_ais`, `navigation_sart`.

**Ngừa ô nhiễm**
- Đều **TEXT**: `pollution_oily_water_separator_quantity`, `_total_output`, `pollution_sewage_tank_quantity`, `_total_volume_m3`, `pollution_settling_tank_quantity`, `_total_volume_m3`, `pollution_sludge_tank_quantity`, `_total_volume_m3`, `pollution_standard_flange_quantity`, `pollution_garbage_container_quantity`, `pollution_sewage_treatment_quantity`, `_output_m3`.
- Ngoại lệ, không phải TEXT:
  - `pollution_oily_water_tank_quantity` INT, `pollution_oily_water_tank_total_volume_m3` (12,2).
  - `pollution_standard_flange_position` (255).

**Thời hạn kiểm tra**
- `inspection_periodic_date`, `inspection_dock_date`, `inspection_annual_date`: DATE.

**Cột JSON**
- `condition_details` JSON, chứa `List<ShipConditionDetail>`.
- `quality_evaluation_details` JSON, chứa `List<ShipQualityEvaluationItem>`.

> Độ dài chính xác của từng cột lấy theo `@Size` của `UpdateAppraisalShipDescriptionRequest` tại `82193fe5` (khớp DDL). Có chỗ khác với `docs/SHIP_MIGRATION_PLAN.md`, khi đó code là chuẩn.

### Converter `ShipConditionDetailsConverter` và `ShipQualityEvaluationConverter`

- Dùng `ObjectMapper` mặc định.
- Ghi: null hoặc rỗng thì ghi null. Lỗi thì ghi null.
- Đọc: null hoặc rỗng thì trả `new ArrayList`. Lỗi thì trả list rỗng.

### `ShipConditionDetail`

- `itemName`: `@Size(max=500)`.
- `conditionDescription`: không giới hạn độ dài.

### `ShipQualityEvaluationItem` (cấu trúc mới tại 82193fe5)

- Có `@JsonIgnoreProperties(ignoreUnknown=true)`.
- `List<@Size(max=500) String> itemNames`.
- `importanceWeight` (1), `standardValue` (2), `relativeValue` (3): BigDecimal, có `@PositiveOrZero` và `@Digits(8,2)`.
- `rating`: `@Size(max=50)`, nhận Tốt, Khá, Trung bình hoặc Kém. Trường này **không** tham gia tính toán.
- Không còn `itemName` đơn và `subItems`.

### `AppraisalGroupCompare` (bảng `appraisal_group_compares`)

| Cột | Kiểu SQL | Kiểu Java |
|---|---|---|
| `ship_build_year` | INT | Integer |
| `ship_build_place` | TEXT | String |
| `ship_usage_purpose` | TEXT | String |
| `ship_class` | TEXT | String |
| `ship_remaining_quality_pct` | DECIMAL(5,2) | BigDecimal |
| `ship_deadweight` | DECIMAL(12,2) | BigDecimal |
| `ship_dimension` | TEXT | String |
| `ship_main_engine_power_hp` | DECIMAL(10,2) | BigDecimal |
| `ship_for_sale_price_usd` | DECIMAL(15,2) | BigDecimal |

`ship_for_sale_price_usd` chỉ được lưu, không có logic nào dùng.

### `AppraisalReportAsset`

- `ship_inspection_certificate_number` (500), `ship_inspection_issued_place` (500).
- `ship_inspection_issued_date`, `ship_inspection_expiry_date`: DATE.
- `ship_inspection_for_whom` (1000).

### `AppraisalImage`

- Thêm `appraisal_ship_description_id` FK (LAZY).
- Repository có `findByAppraisalShipDescriptionIdAndDeletedOrderByCreatedAscIdAsc`.

### `CompareInformation` (kho TSSS, bảng `compare_information`)

- `ship_build_year` INT, `ship_build_place` TEXT, `ship_usage_purpose` TEXT, `ship_class` TEXT.
- `ship_remaining_quality_pct` (5,2), `ship_deadweight` (12,2), `ship_dimension` TEXT, `ship_main_engine_power_hp` (10,2).
- `currency_code` VARCHAR(10) NULL. Giá trị NULL được hiểu là VND.
- Có index `(asset_type, deleted, updated)`.

### `EvaluationResultDetailDto`

- `currencyCode`.
- `ship: ShipEvaluationDetail { Long amount; Long exchangeRate; Long vndAmount; Long roundedAmount }`.

## 3. Migration ship

Danh sách đầy đủ nằm ở `00-index.md`. Các seed cần biết:

- **`V20261015100300` + `V20261103120000`**:
  - `control_compare_factor_labels`: "Khả năng vận hành" và "Chất lượng còn lại" có `asset_types` [3,4].
  - Các nhãn tàu "Năm đóng", "Nơi đóng", "Ngoại quan tổng thể", "Trọng tải toàn phần", "Công suất (CV)", "Cấp phương tiện" có `asset_types` [4].
  - Đã đổi "Năm đóng/hoán cải" thành "Năm đóng", kể cả `control_compare_factor.c_label`.
- **`V20261017100400`**: `valuation_basis` có 1 dòng `asset_type=4`, câu "Cách tiếp cận từ thị trường … tàu chở hàng khô …". File này đã bị sửa sau khi tạo (bỏ dòng 2), nên có rủi ro checksum Flyway ở phía legacy.
- **`V20261028120000`**: `valuation_method(method_type=1, asset_type=4)` với nội dung "Thẩm định giá trị tàu chở hàng khô: cách tiếp cận từ thị trường - phương pháp so sánh (…)".
- **`V20261028130000`**: `report_juridical` thêm 3 dòng chung (VBHN 157/VBHN-VPQH, TB 1294/TB-BTC, "Các văn bản sửa đổi…"). Dòng nào cũng có `province_id` NULL.
- **`V20261103110000`**: tách năm đóng và nơi đóng, xóa dữ liệu cũ.
- **`V20261103130000`**: xóa toàn bộ `quality_evaluation_details` vì cấu trúc JSON đã đổi.

## 4. API legacy

- **`PUT /api/appraisal-ship-descriptions/{id}`** (`AppraisalShipDescriptionApiController`):
  - Request `@Valid UpdateAppraisalShipDescriptionRequest`.
  - Thành công trả 200 với `AppraisalAssetGroupShipSyncResponse {id, appraisalGroupCompare: ShipSyncDto}`.
  - Không tìm thấy trả 404.
  - Lỗi validate trả 400, message là message của field lỗi đầu tiên.
- **`GET /api/appraisal-images/ship-description/{id}`**: lấy ảnh. Upload dùng `appraisalShipDescriptionId`, và phải chỉ định đúng một trong ba owner (đất, xe, tàu).
- **`POST /appraisal-report/save/comparative-information-ship-tab/{reportId}`**: cùng `appraisalGroupCompareService.update`. Gặp `IllegalArgumentException` thì trả 400.
- **`GET /appraisal-report/search-compare-informations?keyword&assetType&pageNo&pageSize`**: chấp nhận assetType CAR và SHIP.
- **`POST /api/evaluation-results/save`**:
  - Request `@Valid SaveEvaluationResultRequest`, extends `AppraisalResult`, thêm 3 trường:
    - `currencyCode` `@Pattern("VND|USD")`.
    - `exchangeRate` `@Min(1) @Max(999999999)`.
    - `currencyNote` `@Size(max=1000)`.
  - `IllegalArgumentException` trả 400.
- **Tab fragment**: SHIP dùng `legal-document-ship`, `ship-description`, `comparative-information-ship` (nạp `groupCurrencyCode`).
- **Cơ sở thẩm định**: màn cơ bản có model `valuationBasisListShip`.

## 5. Service

### `AppraisalShipDescriptionService`

**Tạo và reset**
- `createShipDescriptions(report, groups)`: mỗi nhóm SHIP được một bản ghi, `conditionDetails` gồm 18 mục mặc định, `qualityEvaluationDetails` để null.
- `createOrResetShipDescription(report, group)`: nếu đã có bản ghi (kể cả đã xóa mềm) thì **giữ id và created**, ghi đè mọi cột về null hoặc mặc định, đặt `deleteFlag=0`. Cách này tránh vi phạm unique.

**18 hạng mục tình trạng mặc định** (`conditionDescription` để null):
1. Thân vỏ
2. Phía ngoài vỏ từ đường nước không tải trở xuống
3. Bên ngoài vỏ từ mớn nước đầy tải trở lên và sơn phủ
4. Các đường ngấm nước và thước đọc mớn nước bên ngoài mạn
5. Mặt boong chính và sơn phủ bề mặt
6. Mặt boong ngoài trời
7. Thành lan can bảo vệ
8. Hệ thống tời neo + con lăn
9. Các cọc bích
10. Các cầu thang lên xuống
11. Khoang lái
12. Khoang hầm máy động cơ
13. Các phòng
14. Hệ thống cứu hỏa
15. Hệ thống bơm nước
16. Máy phát điện
17. Thiết bị tín hiệu
18. Hệ thống chiếu sáng

**`update(id, req)`**
- Ghi đè **mọi** field, kể cả null. Chuỗi đi qua `trimToNull`.
- `conditionDetails` và `qualityEvaluationDetails` chỉ cập nhật khi request khác null.
- Sau khi lưu thì gọi `syncToMainAssetCompare`.

**Chuẩn hóa**
- Condition: trim, bỏ dòng có cả hai field rỗng.
- Quality: trim từng tên, bỏ tên rỗng, trim `rating`. Bỏ item rỗng hoàn toàn (không tên và 4 field đều null).

**CLCL** (`calculateRemainingQualityPct`):
- Với mỗi item có đủ w, r, s và `s > 0`: `contribution = w × (r ÷ s, scale 10, HALF_UP) × 100`.
- `total = Σ contribution`.
- Trả null khi không có item hợp lệ nào, hoặc `total < 0`, hoặc `total > 100`.
- Ngược lại trả `total.setScale(2, HALF_UP)`.

**`syncToMainAssetCompare`**
- Đích là dòng compare `type=0` chưa xóa của nhóm. Không có dòng đó thì bỏ qua.
- `applyIfPresent`: chỉ ghi khi nguồn khác null.

| Nguồn | Đích |
|---|---|
| `buildYear` | `shipBuildYear` |
| `buildPlace` | `shipBuildPlace` |
| `shipTypePurpose` | `shipUsagePurpose` |
| `shipClass` | `shipClass` |
| `deadweight` | `shipDeadweight` |
| `formatDimension(dimLengthMax, dimBreadthMax)` | `shipDimension` |
| `mainEnginePowerHp` | `shipMainEnginePowerHp` |
| `joinAssetAddresses(tài sản chưa xóa của nhóm)` | `address` |
| CLCL | `shipRemainingQualityPct` |

- `formatDimension` cho ra chuỗi `"{L}m x {B}m"`. Mỗi số đi qua `stripTrailingZeros().toPlainString()` rồi đổi "." thành ",". Thiếu một trong hai giá trị thì trả null.
- `joinAssetAddresses`: ghép `asset` của các tài sản có `deleteFlag==0` bằng "\n", trim từng dòng, bỏ dòng rỗng. Rỗng hết thì trả null.

### `AppraisalGroupCompareService`

**`createGroupCompares`**: nhóm SHIP thì dòng type 0 được gán `address = joinAssetAddresses(...)`.

**`update`**
- Giá:
  - `marketPrice`: SHIP luôn 0. Loại khác: `tp≠0 ? tp : (ep ?? 0)`, đã bỏ hằng `0111L`.
  - SHIP: `apartmentOwnershipRightPrice = 0`, `landUseRightPrice = (tp != null && tp != 0) ? tp : (ep ?? 0)`.
  - CAR: `landUseRightPrice = tp + ep`.
- Field tàu: trim chuỗi, rỗng thành null; `shipBuildYear` là Integer.
- `shipRemainingQualityPct` ngoài [0,100] thì ném IAE "Chất lượng còn lại tàu phải từ 0 đến 100".

**Tiền tệ TSSS**
- `validateCompareInformationLinks` chạy **trước** vòng cập nhật, tức trước mọi side-effect ảnh/S3.
- Chỉ xét type 1-3 có `compareInformationId` mà liên kết thay đổi.
- Nếu `resolveCurrencyCode(group)` khác `resolveCurrencyCode(ci.assetType, ci.currencyCode)` thì ném IAE "Tài sản so sánh trong kho dùng đơn vị X, khác đơn vị tiền tệ của nhóm (Y)".
- Cờ `linkUnchanged` được tính **trước** `setCompareInformation`.

**Đồng bộ lên kho** (`syncGroupCompareToCompareInformation`, type 1-3, checkbox `syncToCompareInfo`, không có `compareInformationId`):
- Map 8 field tàu.
- `ci.currencyCode = resolveCurrencyCode(group)`.
- Không đồng bộ `shipForSalePriceUsd`.

**Đồng bộ sang bảng điều chỉnh** (`computeSyncedValuesForLabelName`, chỉ cho SHIP):

| Nhãn | Nguồn | Cách định dạng |
|---|---|---|
| Năm đóng | `shipBuildYear` | `String.valueOf` |
| Nơi đóng | `shipBuildPlace` | trim |
| Trọng tải toàn phần | `shipDeadweight` | `stripTrailingZeros().toPlainString()`, rồi `formatNumberStringToEuropeanStyleFixed2` |
| Công suất (CV) | `shipMainEnginePowerHp` | như trên |
| Chất lượng còn lại | `shipRemainingQualityPct` (CAR dùng `carRemainingQualityPct`) | như trên |
| Cấp phương tiện | `shipClass` | trim |

- Ghi vào `cValue`, `cValueCp1`, `cValueCp2`, `cValueCp3`; giá trị null ghi thành "".

### `CompareInformationServiceImpl.searchCompareInformationByKeyword`

- Keyword rỗng: "Vui lòng nhập từ khóa để tìm kiếm".
- Chấp nhận CAR và SHIP.
- SHIP dùng native query: `deleted=:flag AND asset_type=4 AND address IS NOT NULL AND address LIKE %kw%`, `Sort DESC updated`.
- Trả `Paged<CompareInformationShipSearchDTO>`. Trong DTO, `currencyCode` đã resolve và `updated` định dạng dd/MM/yyyy.

### `AppraisalResultServiceImpl`

**`buildShipEvaluationDetailFromEntities`** (dòng ~567-623):
- `currencyCode = resolve(SHIP, group.currencyCode)`.
- `rate`: VND luôn là 1. USD lấy `group.exchangeRate`, null thì dùng 1.
- `amount = synthetic.avgPriceGuide`.
- `vndAmount = Math.multiplyExact(avg, rate)`. Tràn số thì ném IAE "Thành tiền quy đổi sang VNĐ quá lớn, vui lòng kiểm tra lại tỷ lệ chuyển đổi".
- `roundedAmount = BigDecimal(vnd) / base` (HALF_UP, scale 0) rồi nhân lại `base`. `base` là 1.000 nếu kiểu làm tròn là THOUSAND, ngược lại 1.000.000.
- `totalMoney = roundingMoney = roundedAmount ?? 0`.
- `totalPriceWordAppraisal = ReadMoneyUtils.doc(totalMoney)`, đọc theo VNĐ.

**`saveEvaluationResult(SaveEvaluationResultRequest)`** (dòng ~761-793):
- Nhóm SHIP:
  - `currencyCode != null`: resolve; VND thì `rate=1`; USD có rate thì gán, USD không có rate thì giữ rate cũ. `note` trim, rỗng thì null.
  - Lưu nhóm.
  - Tính lại tổng từ synthetic theo `request.roundingMoneyType`, rồi ghi đè `roundingMoneyType`, `roundingMoney`, `totalMoneyAppraisal` và chữ trên request.
- Nhóm khác: bỏ qua ba trường tiền tệ, giữ tổng client gửi.
- Cuối cùng gọi `saveEvaluationResult((AppraisalResult) req)`.

**Ngoài ra**: tránh NPE khi `getAppraisalLandDescription()` null (áp dụng cho tàu và xe).

**Test spec `AppraisalResultServiceSaveCurrencyTest`** (9 case):

| # | Tên | Đầu vào | Kỳ vọng |
|---|---|---|---|
| 1 | `usdDetailKeepsForeignAmountAndRoundsVnd` | USD, rate 25450, avg 100123, làm tròn MILLION | amount 100123, vnd 2.548.130.350, rounded 2.548.000.000, chữ kết thúc " đồng" |
| 2 | `usdDetailRoundingTypeOnlyAffectsVndColumn` | làm tròn THOUSAND | 2.548.130.000 |
| 3 | `usdDetailWithNullRateUsesOne` | rate null, avg 1.500.000, MILLION | rounded 2.000.000 |
| 4 | `vndDetailIgnoresStoredRate` | VND, rate lưu sẵn 25450 | rate dùng là 1 |
| 5 | `shipSaveRecomputesTotalsFromNewRate` | USD, rate 25450, note "  Tỷ giá VCB  ", avg 100000 | nhóm có rate 25450 và note đã trim; `roundingMoney` 2.545.000.000; ghi đè tổng 999 do client gửi |
| 6 | `shipVndForcesRateToOne` | nhóm USD/25450, gửi VND | VND, rate 1, note null |
| 7 | `shipUsdWithNullRateKeepsOldRate` | rate cũ 24000, gửi USD không có rate | giữ 24000 |
| 8 | `nonShipIgnoresCurrencyFieldsAndKeepsClientTotals` | nhóm LAND | không đổi tiền tệ, giữ 999 / "client-word đồng" |
| 9 | `requestWithoutCurrencyFieldsOnShipKeepsGroupCurrency` | không gửi trường tiền tệ | giữ nguyên tiền tệ của nhóm |

### `AppraisalReportServiceImpl` (vòng đời nhóm)

**Tạo báo cáo**: gọi `createShipDescriptions` sau mô tả xe, trước compare. `initDefaultEntitiesForNewGroup` cũng gọi hàm này.

**`handleAssetTypeChange`** (dòng ~372-405), theo thứ tự:
1. Đất.
2. Xe.
3. `resetOrCreateShipDescription` (dòng ~590-615):
   - Rời SHIP: xóa mềm nếu `deleteFlag==0`, rồi `asyncDeleteShipDescriptionImages`.
   - Vào SHIP: nếu có bản ghi cũ thì xóa ảnh trước, sau đó `createOrResetShipDescription`.
4. `resetGroupCompares`: thêm việc reset 9 field tàu về null.
5. Synthetic.
6. Factor: xóa mềm, tạo lại theo nhãn mặc định của loại mới.
7. Result.
8. CTXD.
9. `resetGroupFields`: `currencyCode=VND`, `exchangeRate=1`, `currencyNote=null`.

**`restoreDeletedGroup`** (dòng ~1257-1269): nhóm SHIP có mô tả `deleteFlag==1` thì khôi phục về 0; chưa có mô tả thì tạo mới.

**Khi xóa nhóm** (dòng ~975-1003): **chưa** xóa mềm mô tả tàu. Đây là lỗ hổng của L, hệ thống mới sẽ sửa (Khác hệ thống cũ #1).

**Merge thông tin pháp lý** ("target rỗng thì lấy source"): thêm 5 field `shipInspection*`.

**Nhãn mặc định**: `resolveDefaultControlCompareFactorLabels(SHIP)` trả 8 nhãn.

**Test spec `AppraisalReportServiceImplUpdateBasicInfoTest`**:
- **TC-10b**, Ô tô sang Tàu: gọi `createOrResetShipDescription(report, group)`; mô tả xe bị xóa mềm; không lưu mô tả đất.
- **TC-10c**, Tàu (USD/25000/note) sang Đất:
  - Mô tả tàu `deleteFlag=1`, được save, không gọi `createOrReset`.
  - `compare.shipClass` và `shipDeadweight` thành null.
  - Nhóm về VND/1/null. Có lưu mô tả đất.
- **TC-10d**, Đất sang Tàu khi đã có mô tả tàu bị xóa mềm: gọi `createOrReset` để dùng lại bản ghi; mô tả đất bị xóa mềm.

### Các service khác

- **`AppraisalReportAssetServiceImpl`**: khi nhóm là CAR hoặc SHIP thì không ghi đè `asset`. Thêm việc copy 5 field `shipInspection*`.
- **`LegalDocumentAssetTypeValidator.isAllowedAssetType`**: chấp nhận SHIP.
- **`AppraisalImageService`**:
  - Thư mục S3: `{land|vehicle|ship}/{reportId}/{descId}/{uuid}/{filename}`.
  - Thiếu owner thì ném lỗi "Phải chỉ định đúng một mô tả đất, phương tiện hoặc tàu thủy".

## 6. Request validate mô tả tàu (`UpdateAppraisalShipDescriptionRequest`)

- Hằng số:
  - `TEXT_MAX_LENGTH=500`.
  - `DECIMAL_MESSAGE`: "tối đa 8 chữ số nguyên và 2 chữ số thập phân".
  - `DECIMAL_LARGE_MESSAGE`: 10 chữ số nguyên.
- Số không âm: `@PositiveOrZero`, message "<tên> không được âm".
- `@Digits(8,2)`: kích thước, công suất, điện áp ắc quy.
- `@Digits(10,2)`: các cột DECIMAL(12,2).
- `@Size` theo cột:

| Giới hạn | Áp dụng cho |
|---|---|
| 100 | các `*Type` ngắn |
| 255 | `registrationNumber`, `registryNumber`, `material`, `shipClass`, `operatingArea`, `design*`, `appraisalNumber`, `anchorChainCable`, `mainEngineBrand`, `mainEngineSerialNumber`, `auxEngineBrand`, `electricGeneratorVoltage`, `electricBatteryTotalCapacity`, `lifting*`, `pollutionStandardFlangePosition`, `mooringLocation` |
| 500 | `shipName`, `shipTypePurpose`, `ownerName`, `conversionYearPlace` |
| 1000 | các field TEXT, `buildPlace`, `towing*`, `anchorSternTotalWeightKg` |
| 5000 | `operationNote` |

- `buildYear`: `@PositiveOrZero`.
- Hai danh sách có `@Valid`.
- Đọc nguyên file để copy chính xác từng message.
