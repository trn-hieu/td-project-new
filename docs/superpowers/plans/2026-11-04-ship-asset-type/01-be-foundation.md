# 01 — Nền backend SHIP (B01, đợt B1)

> Đọc `00-index.md` trước (hợp đồng, quy ước tên, Global Constraints, "Khác hệ thống cũ", Review Focus). Subagent: `model: "sonnet"`, có spec review rồi code review.
> Đọc thêm `surveys/legacy-be.md` (mục 1, 2, 5) và `surveys/new-code.md` (mục Backend).
> Đọc `backend/CLAUDE.md` và `backend/.claude/rules/*.md` trước khi viết code.
> B01 là task duy nhất của đợt B1 nên mọi file dưới đây do B01 sở hữu. B02–B10 (đợt B2/B3) chỉ **dùng** các chữ ký ở khối Interfaces.
> **Không tạo Flyway migration.** Schema do legacy sở hữu, `ddl-auto=none`. Không thêm file vào `src/main/resources/db/migration`.

Ký hiệu: L = legacy `82193fe5` (`src/main/java/com/jvb/thanhdo/`), N = `backend/src/main/java/com/thanhdo/tdprojectbe/`, NT = `backend/src/test/java/com/thanhdo/tdprojectbe/`.

## Hành vi hệ thống cũ

Đọc bằng `git -C td-project-legacy show 82193fe5:<path>`.

- **Hằng số tiền tệ**: `L common/CommonConstants.java` l.428–429 (`SHIP_VND="đồng"`, `SHIP_USD="USD"`), l.435–497 (`CurrencyCode`, `resolveCurrencyCode` l.446, `currencyUnitLabel` l.455, `currencyMoneyLabel` l.460, `resolveRegulatoryConstraintsPriceUnit(assetType, currencyCode)` l.~474–490), l.469 (chú thích: hai đơn vị tàu **cố ý không** nằm trong `REGULATORY_CONSTRAINTS_PRICE_UNITS`).
- **Nhãn yếu tố tàu**: `L common/CommonConstants.java` l.321–325 (`DEFAULT_CONTROL_COMPARE_FACTOR_LABELS_SHIP`), l.381–386 (`NAM_DONG`, `CAP_PHUONG_TIEN`, ...).
- **Đọc tiền USD**: `L common/ReadMoneyUtils.java` l.34–44 `doc(long, String)`: USD và kết quả kết thúc " đồng" thì đổi thành " đô la Mỹ"; mã khác giữ nguyên.
- **Loại ảnh 21–30**: `L common/ReportConstants.java` l.52–61.
- **Test spec 5 case**: `L src/test/.../common/CurrencyCodeTest.java` (xem `surveys/legacy-be.md` mục 1).
- **Entity mô tả tàu**: `L entity/AppraisalShipDescription.java` (157 `@Column`; 155 cột giá trị + `delete_flag` + `created`; hai cột JSON; `@Where` ảnh; `@PrePersist` gán `created`).
- **Tạo/reset mô tả**: `L service/AppraisalShipDescriptionService.java` l.37–52 (`createShipDescriptions`: `conditionDetails` = 18 mục mặc định, `qualityEvaluationDetails` null), l.62–76 (`createOrResetShipDescription`: giữ `id` và `created`, ghi đè **mọi** cột về null, `conditionDetails` mặc định, `deleteFlag = 0`).
- **Đồng bộ sang so sánh type 0**: cùng file l.284–310 (`syncToMainAssetCompare`, `applyIfPresent` chỉ ghi khi nguồn khác null).
- **CLCL**: l.313–345 (`calculateRemainingQualityPct`, `qualityContribution` l.332): bỏ item thiếu w/s/r hoặc `s <= 0`; `contribution = w x (r / s, scale 10, HALF_UP) x 100`; tổng ngoài [0,100] hoặc không có item nào thì null; còn lại `setScale(2, HALF_UP)`. Ở đây `r` là `relativeValue` (cột 3), `s` là `standardValue` (cột 2), `w` là `importanceWeight` (cột 1); `rating` không tham gia.
- **`formatDimension`**: l.347–358: `"{L}m x {B}m"`, mỗi số `stripTrailingZeros().toPlainString()` rồi đổi "." thành ","; thiếu một trong hai thì null.
- **18 hạng mục tình trạng mặc định**: l.360–387 (`DEFAULT_CONDITION_ITEMS`, `defaultConditionDetails`).
- **Cột tàu của dòng so sánh**: `L entity/AppraisalGroupCompare.java` l.~180–210; ghi ở `L service/AppraisalGroupCompareService.java` l.~250–275: chuỗi trim, rỗng thành null; `shipBuildYear` là `Integer`; `shipForSalePriceUsd` gán thẳng từ request. Giá của SHIP (`marketPrice = 0`, `apartmentOwnershipRightPrice = 0`, `landUseRightPrice = tp != 0 ? tp : ep`) cùng khối đó.
- **Reset cột tàu khi đổi loại**: `L service/impl/AppraisalReportServiceImpl.java` l.639–690 (`resetGroupCompares`, 9 setter `setShip*(null)` ở l.675–683); `resetGroupFields` đặt `currencyCode=VND`, `exchangeRate=1`, `currencyNote=null`.
- **Pháp lý tàu**: `L service/impl/AppraisalReportAssetServiceImpl.java` l.96 (SHIP và CAR không ghi đè `asset`), l.109–113 (5 setter `shipInspection*`); form `legal-document-ship` chỉ có `certificateNumber`, `cnIssuedPlace`, `gcnNoIssuedDate`, `forWhom`, 5 cột `shipInspection*`, `legalDocuments`.
- **Migration (tham chiếu schema)**: `LR db/migration/V20261015100000__create_appraisal_ship_description.sql` (NOT NULL chỉ ở `appraisal_report_id`, `appraisal_asset_group_id`, `delete_flag`; UNIQUE theo nhóm), `V20261103110000__split_ship_build_year_place.sql`, `V20261103100000__add_ship_fields_to_compare_information.sql`, `V20261031100000/110000` (tiền tệ nhóm).

### Điểm "Khác hệ thống cũ" liên quan B01

- **#1** (xóa nhóm thì xóa mềm mô tả tàu): B01 chỉ cung cấp `AppraisalShipDescription.softDelete()` / `revive()`; B07 gọi.
- **#3** (lỗi 400 qua `BusinessException`): B01 là tầng entity/helper thuần nên không ném `BusinessException`. Kiểm tra CLCL ngoài 0–100 là việc của B05.
- Bản mới dùng id kiểu `Long` thay cho quan hệ `@ManyToOne` (giống `AppraisalVehicleDescription` đang có) và `@SQLRestriction` thay cho `@Where`. Entity không dùng `@Data`.

## Quyết định thiết kế của B01 (đã chốt, các task sau tuân theo)

1. **Không tạo Flyway migration.** Test repository chứng minh map đúng schema thật.
2. **`CurrencyCodes` chỉ là helper mới.** `RegulatoryConstraints.priceUnit(int)` và `CommonConstants.resolveRegulatoryConstraintsPriceUnit(Integer)` **không đổi ở B01**. B05 sửa `RegulatoryConstraints` để gọi `CurrencyCodes.regulatoryPriceUnit(assetType, currencyCode)`. `CurrencyCodes.regulatoryPriceUnit` với loại không phải SHIP gọi lại bản một tham số của `CommonConstants` nên B05 **không được** để bản một tham số gọi ngược `CurrencyCodes` (tránh vòng). `REGULATORY_CONSTRAINTS_PRICE_UNITS` giữ nguyên, không thêm "đồng"/"USD".
3. **`AppraisalShipDescription.apply(ShipDescriptionInput)` thuộc B03** (Input DTO do B03 tạo). B01 chỉ tạo `empty`, `resetToDefault`, `softDelete`, `revive`.
4. **Hai record tham số trung gian trong `model/dto`** để B01 không phụ thuộc DTO của task khác: `ShipCompareFields` (B05 dựng từ `GroupCompareRowInput`) và `ShipLegalFields` (B04 dựng từ `LegalAssetInput`). Không sửa `GroupCompareRowInput` / `LegalAssetInput` ở B01.
5. **Phương thức reset gắn vào hàm reset sẵn có**: `AppraisalGroupCompare.reset(String)` gọi thêm `clearShipFields()`; `AppraisalAssetGroup.resetForAssetType(int, LocalDateTime)` gọi thêm `resetCurrency()`. Nhờ vậy B07 (`GroupAssetTypeReset`) tự phủ cả cột tàu và tiền tệ, gọi thêm cũng vô hại (idempotent).
6. **Đọc tiền bằng chữ USD**: thêm `MoneyWords.read(long, String currencyCode)`. `ReadMoneyUtils` (bản chép cho Word golden) **không sửa**; B09 dùng `MoneyWords`.
7. **Lớp thuần `ShipQuality` ở `service/ship`** được entity `AppraisalGroupCompare.applyShipSync` và `AppraisalShipDescription.empty/resetToDefault` gọi (hàm tĩnh, không phụ thuộc Spring).
8. **Mở cận trên loại cơ sở thẩm định lên 4** ở `ReportContentMerge` và `AppraisalReportFormService` (Step 1) vì thiếu thì cơ sở thẩm định tàu không bao giờ được nạp. Hai file này không thuộc task nào khác ở đợt B1.

## Interfaces (B02–B10 dùng)

```java
// N common/constant/AppraisalAssetType + CommonConstants.AppraisalAssetType
public static final int SHIP = 4;
// N model/entity/ValuationBasis
public static final int ASSET_TYPE_SHIP = 4;
// N common/util/ReportConstants.AppraisalImageType  (cạnh CAR_* = 16/18/19/20)
SHIP_FRONT=21, SHIP_BOW=22, SHIP_STERN=23, SHIP_HULL_SIDE=24, SHIP_DECK=25, SHIP_IMO_NUMBER=26,
SHIP_ANCHOR_SYSTEM=27, SHIP_WHEELHOUSE_CABIN=28, SHIP_CARGO_HOLD=29, SHIP_ENGINE_SYSTEM=30
// N common/constant/ControlCompareFactorLabels (thêm hằng, REMAINING_QUALITY đã có)
BUILD_YEAR="Năm đóng", BUILD_PLACE="Nơi đóng", OVERALL_APPEARANCE="Ngoại quan tổng thể",
OPERABILITY="Khả năng vận hành", DEADWEIGHT="Trọng tải toàn phần", POWER_HP="Công suất (CV)",
VESSEL_CLASS="Cấp phương tiện"

// N common/util/CurrencyCodes (final, constructor private)
public static final String VND = "VND", USD = "USD";
public static String resolveCurrencyCode(Integer assetType, String code);      // chỉ SHIP + "usd" (không phân biệt hoa thường) -> "USD", còn lại "VND"
public static String currencyUnitLabel(String code);                           // "USD" / "VND"
public static String currencyMoneyLabel(String code);                          // "USD" / "đồng"
public static String regulatoryPriceUnit(Integer assetType, String code);      // SHIP: "USD"/"đồng"; CAR: "đồng/xe"; khác: "đồng/m²"
// N common/util/MoneyWords
public static String read(long amount, String currencyCode);                   // "USD" thay hậu tố " đồng" bằng " đô la Mỹ"

// N model/dto
@JsonIgnoreProperties(ignoreUnknown = true)
public record ShipConditionDetail(String itemName, String conditionDescription) {}
@JsonIgnoreProperties(ignoreUnknown = true)
public record ShipQualityEvaluationItem(List<String> itemNames, BigDecimal importanceWeight,
        BigDecimal standardValue, String rating, BigDecimal relativeValue) {}
public record ShipCompareFields(Integer buildYear, String buildPlace, String usagePurpose, String shipClass,
        BigDecimal remainingQualityPct, BigDecimal deadweight, String dimension,
        BigDecimal mainEnginePowerHp, BigDecimal forSalePriceUsd) {}
public record ShipLegalFields(String certificateNumber, String cnIssuedPlace, LocalDate gcnNoIssuedDate,
        String forWhom, String shipInspectionCertificateNumber, String shipInspectionIssuedPlace,
        LocalDate shipInspectionIssuedDate, LocalDate shipInspectionExpiryDate,
        String shipInspectionForWhom, String legalDocuments) {}

// N model/converter (public @Converter, kế thừa JsonListConverter; emptyAsNull() mặc định true)
public class ShipConditionDetailsConverter extends JsonListConverter<ShipConditionDetail> {}
public class ShipQualityEvaluationConverter extends JsonListConverter<ShipQualityEvaluationItem> {}

// N service/ship/ShipQuality (final, hàm tĩnh thuần)
public static BigDecimal remainingQualityPct(List<ShipQualityEvaluationItem> items);   // 2 lẻ HALF_UP; null nếu không tính được / ngoài [0,100]
public static BigDecimal contribution(ShipQualityEvaluationItem item);                 // w*(r/s scale10)*100 chưa làm tròn; null nếu thiếu số hoặc s<=0
public static String formatDimension(BigDecimal lengthMax, BigDecimal breadthMax);     // "118,35m x 19,4m"; thiếu một số -> null
public static List<ShipConditionDetail> defaultConditionDetails();                     // 18 mục, conditionDescription null, danh sách mới sửa được

// N model/entity/AppraisalShipDescription  (@Getter, @NoArgsConstructor(PROTECTED), không @Data)
public static final int NOT_DELETED = 0;
public static AppraisalShipDescription empty(Long reportId, Long groupId, LocalDateTime now);
public void resetToDefault();     // giữ id/created/reportId/groupId; mọi cột giá trị null; conditionDetails mặc định; quality null; deleteFlag 0
public void softDelete();         // deleteFlag = 1
public void revive();             // deleteFlag = 0, không xóa dữ liệu đã nhập
public List<AppraisalImage> getAppraisalImages();   // chỉ đọc, chỉ ảnh deleted = 0 hoặc NULL
// N repository/AppraisalShipDescriptionRepository
Optional<AppraisalShipDescription> findByAppraisalAssetGroupId(Long appraisalAssetGroupId);   // kể cả bản đã xóa mềm

// N model/entity/AppraisalGroupCompare  (+9 getter ship*)
public void applyShipForm(ShipCompareFields f, long tradePrice, long estimatePrice);
public void applyShipSync(AppraisalShipDescription ship, String address, BigDecimal remainingQualityPct);
public void clearShipFields();
// N model/entity/AppraisalReportAsset  (+5 getter shipInspection*)
public void applyShipLegal(ShipLegalFields f);
// N model/entity/AppraisalImage
public Long getAppraisalShipDescriptionId();
public static AppraisalImage ofShip(Long shipDescriptionId, int type, String note, String name, String path,
        String groupUuid, LocalDateTime now);
// N repository/AppraisalImageRepository  (B01 thêm, cạnh finder vehicle)
List<AppraisalImage> findByAppraisalShipDescriptionIdAndDeletedOrderByCreatedAscIdAsc(Long shipDescriptionId, Integer deleted);
// N model/entity/CompareInformation  (+9 getter: shipBuildYear, shipBuildPlace, shipUsagePurpose, shipClass,
//   shipRemainingQualityPct, shipDeadweight, shipDimension, shipMainEnginePowerHp, currencyCode) — chưa có hàm ghi, B10 thêm vào copyFrom
// N model/entity/AppraisalAssetGroup  (+ getCurrencyCode/getExchangeRate/getCurrencyNote)
public void applyCurrency(String code, Long rate, String note);   // code đã resolve ("VND"/"USD")
public void resetCurrency();                                       // "VND", 1, null
// NT repository/RepositoryTestData (helper test dùng chung, B02–B10 tái sử dụng)
static long insertReport(EntityManager em);
static long insertGroup(EntityManager em, long reportId, int assetType);
```

Ngữ nghĩa chốt cho từng phương thức entity:

- `applyShipForm`: trim + rỗng thành null cho 4 chuỗi; 4 `BigDecimal` `setScale(2, HALF_UP)` (null giữ null); `shipBuildYear` gán thẳng; `marketPrice = 0L`; `apartmentOwnershipRightPrice = 0L`; `landUseRightPrice = tradePrice != 0 ? tradePrice : estimatePrice`. Hàm này **ghi đè `marketPrice` mà `applyForm` đã đặt trước đó**. B05 phải gọi nó trong nhánh `AppraisalAssetType.SHIP` của `applyForm` sau dòng `this.marketPrice = ...`. `shipForSalePriceUsd` ghi từ `f.forSalePriceUsd()` (có thể null) như L.
- `applyShipSync` (`applyIfPresent`, nguồn null thì giữ giá trị cũ): `buildYear→shipBuildYear`, `buildPlace→shipBuildPlace`, `shipTypePurpose→shipUsagePurpose`, `shipClass→shipClass`, `deadweight→shipDeadweight`, `ShipQuality.formatDimension(dimLengthMax, dimBreadthMax)→shipDimension`, `mainEnginePowerHp→shipMainEnginePowerHp`, tham số `address→address`, tham số `remainingQualityPct→shipRemainingQualityPct`. Không trim, không đụng `shipForSalePriceUsd`, giá, nguồn, liên hệ.
- `clearShipFields`: 9 cột `ship_*` về null (kể cả `shipForSalePriceUsd`).
- `applyShipLegal`: chỉ ghi `certificateNumber`, `cnIssuedPlace`, `gcnNoIssuedDate`, `forWhom`, 5 cột `shipInspection*`, `legalDocuments`; **không** đụng `owner`, `asset`, `gcnNo`, `gcnNoExpiryDate`, `other`, cột đất/ô tô (giống `applyCarLegal`: tên tài sản `asset` không bị ghi đè với SHIP). `mergeLegalFieldsFrom` **không sửa ở B01** (Quyết định #10: B04 thêm 5 cột ship vào gộp pháp lý).
- `AppraisalAssetGroup.applyCurrency(code, rate, note)`: `code` null hoặc khác "VND"/"USD" thì `IllegalArgumentException` (lỗi lập trình, B06 đã resolve và chỉ gọi khi request có `currencyCode`). VND thì `exchangeRate = 1`. USD: `rate != null` thì gán, `rate == null` thì giữ tỷ giá cũ. `currencyNote = Texts.trimToNull(note)`. **Không ghi `updated`**; B06 gọi `touch(now)` nếu cần.
- `AppraisalAssetGroup`: `currencyCode` khởi tạo `"VND"`, `exchangeRate` khởi tạo `1L` ở khai báo field (Hibernate INSERT ghi mọi cột, null sẽ vi phạm NOT NULL).

---

### Task B01: Nền backend SHIP

**Files:**

- Create `N common/util/CurrencyCodes.java`
- Create `N model/dto/ShipConditionDetail.java`, `N model/dto/ShipQualityEvaluationItem.java`, `N model/dto/ShipCompareFields.java`, `N model/dto/ShipLegalFields.java`
- Create `N model/converter/ShipConditionDetailsConverter.java`, `N model/converter/ShipQualityEvaluationConverter.java`
- Create `N service/ship/ShipQuality.java`
- Create `N model/entity/AppraisalShipDescription.java`, `N repository/AppraisalShipDescriptionRepository.java`
- Modify `N common/constant/AppraisalAssetType.java`, `N common/constant/CommonConstants.java` (chỉ lớp lồng `AppraisalAssetType` và `ValuationBasisAssetType`), `N common/constant/ControlCompareFactorLabels.java`, `N common/util/ReportConstants.java`, `N common/util/MoneyWords.java`
- Modify `N model/entity/ValuationBasis.java`, `N service/ReportContentMerge.java`, `N service/AppraisalReportFormService.java`
- Modify `N model/entity/AppraisalGroupCompare.java`, `AppraisalReportAsset.java`, `AppraisalImage.java`, `CompareInformation.java`, `AppraisalAssetGroup.java`
- Modify `N repository/AppraisalImageRepository.java` (thêm finder ảnh của mô tả tàu, B02 dùng)
- Modify `backend/CLAUDE.md` (bảng helper)
- Create tests: `NT common/constant/ShipConstantsTest.java`, `NT common/util/CurrencyCodesTest.java`, `NT model/converter/ShipConditionDetailsConverterTest.java`, `NT model/converter/ShipQualityEvaluationConverterTest.java`, `NT service/ship/ShipQualityTest.java`, `NT model/entity/AppraisalShipDescriptionTest.java`, `NT model/entity/AppraisalGroupCompareShipTest.java`, `NT model/entity/AppraisalReportAssetShipTest.java`, `NT model/entity/AppraisalAssetGroupCurrencyTest.java`, `NT repository/AppraisalShipDescriptionRepositoryTest.java`, `NT repository/ShipColumnsRepositoryTest.java`
- Modify tests: `NT common/util/MoneyWordsTest.java`, `NT service/ReportContentMergeTest.java`, `NT service/AppraisalReportFormServiceTest.java`, `NT repository/RepositoryTestData.java`

Không tạo file nào trong `src/main/resources/db/migration`.

#### Step 0 — Nhánh làm việc và kiểm tra schema (dừng nếu thiếu)

- [ ] **0.1 Tạo nhánh** (không `git checkout` ở repo gốc, chỉ trong submodule `backend`):

```bash
git -C backend -c http.sslBackend=schannel fetch origin develop && git -C backend checkout -b feat/ship-asset-type origin/develop
git -C backend log -1 --format=%h   # kỳ vọng b3d6d7e hoặc commit mới hơn của origin/develop
```

(Quyết định #17: máy này lỗi chứng chỉ SSL nên bắt buộc dùng `-c http.sslBackend=schannel` như trên; không dùng `sslVerify=false`.)

- [ ] **0.2 Kết nối MySQL** theo cấu hình backend: biến `SPRING_DATASOURCE_URL` / `SPRING_DATASOURCE_USERNAME` / `SPRING_DATASOURCE_PASSWORD` trong `backend/.env` (mẫu `backend/.env.example`; mặc định `jdbc:mysql://localhost:3306/thanhdo`, user `root`). Client: `mysql` (Windows: `C:\Program Files\MySQL\MySQL Server 8.0\bin\mysql`). Đặt mật khẩu qua biến môi trường `MYSQL_PWD`, **không** in mật khẩu ra log, không ghi vào file, không commit `.env`:

```bash
M() { mysql -h localhost -P 3306 -u "$DB_USER" thanhdo -e "$1"; }   # DB_USER, MYSQL_PWD lấy từ backend/.env
```

- [ ] **0.3 Kiểm tra các migration của legacy đã chạy** (24 phiên bản ở mục "Điều kiện tiên quyết" của `00-index.md`):

```sql
SELECT COUNT(*) AS da_chay FROM flyway_schema_history
WHERE success = 1 AND version IN ('20261015100000','20261015100100','20261015100200','20261015100300',
 '20261017100000','20261017100100','20261017100200','20261017100300','20261017100400','20261018100000',
 '20261023100000','20261024100000','20261024110000','20261027100001','20261028100000','20261028110000',
 '20261028120000','20261028130000','20261031100000','20261031110000','20261103100000','20261103110000',
 '20261103120000','20261103130000');
```

Kỳ vọng `24`. (Nếu DB dev không dùng Flyway của legacy thì bước 0.4 là phép thử quyết định.)

- [ ] **0.4 `DESCRIBE` các bảng và đối chiếu**:

```sql
DESCRIBE appraisal_ship_description;
DESCRIBE appraisal_group_compares;
DESCRIBE appraisal_report_assets;
DESCRIBE appraisal_image;
DESCRIBE compare_information;
DESCRIBE appraisal_asset_groups;
SELECT name, asset_types FROM control_compare_factor_labels WHERE JSON_CONTAINS(asset_types,'4');
SELECT id, asset_type, delete_flag FROM valuation_basis WHERE asset_type = 4;
```

Tiêu chí đạt (đối chiếu từng dòng):

| Bảng | Phải có |
|---|---|
| `appraisal_ship_description` | 155 cột giá trị theo danh sách ở "Nhóm cột" bên dưới + `id`, `appraisal_report_id` (NOT NULL), `appraisal_asset_group_id` (NOT NULL, UNIQUE `uk_asd_group`), `delete_flag` INT NOT NULL DEFAULT 0, `created`. **Không** còn cột `build_year_place`. `build_year` INT, `build_place` TEXT, `dim_depth`/`dim_draft` DECIMAL(10,2), `deadweight` DECIMAL(12,2), `fire_manual_tools`/`fire_layout_plan` TINYINT(1), `condition_details`/`quality_evaluation_details` JSON |
| `appraisal_group_compares` | `ship_build_year` INT, `ship_build_place` TEXT, `ship_usage_purpose` TEXT, `ship_class` TEXT, `ship_remaining_quality_pct` DECIMAL(5,2), `ship_deadweight` DECIMAL(12,2), `ship_dimension` TEXT, `ship_main_engine_power_hp` DECIMAL(10,2), `ship_for_sale_price_usd` DECIMAL(15,2) |
| `appraisal_report_assets` | `ship_inspection_certificate_number` VARCHAR(500), `ship_inspection_issued_place` VARCHAR(500), `ship_inspection_issued_date` DATE, `ship_inspection_expiry_date` DATE, `ship_inspection_for_whom` VARCHAR(1000) |
| `appraisal_image` | `appraisal_ship_description_id` BIGINT NULL (FK `fk_ai_ship_description`) |
| `compare_information` | `ship_build_year` INT, `ship_build_place` TEXT, `ship_usage_purpose` TEXT, `ship_class` TEXT, `ship_remaining_quality_pct` DECIMAL(5,2), `ship_deadweight` DECIMAL(12,2), `ship_dimension` TEXT, `ship_main_engine_power_hp` DECIMAL(10,2), `currency_code` VARCHAR(10) NULL |
| `appraisal_asset_groups` | `currency_code` VARCHAR(10) NOT NULL DEFAULT 'VND', `exchange_rate` BIGINT NOT NULL DEFAULT 1, `currency_note` TEXT NULL |
| `control_compare_factor_labels` | có đủ 8 nhãn tàu: "Khả năng vận hành" và "Chất lượng còn lại" có `asset_types` chứa 3 và 4; "Năm đóng", "Nơi đóng", "Ngoại quan tổng thể", "Trọng tải toàn phần", "Công suất (CV)", "Cấp phương tiện" có `asset_types` chứa 4. Không còn "Năm đóng/hoán cải" |
| `valuation_basis` | ít nhất 1 dòng `asset_type = 4`, `delete_flag = 0` |

**Tiêu chí dừng**: thiếu bảng, thiếu cột, sai kiểu nêu trên, số phiên bản ở 0.3 khác 24 mà 0.4 lại thiếu cột, hoặc thiếu nhãn tàu thì **dừng toàn bộ task, báo người dùng** liệt kê chính xác thiếu gì. **Không** tự `ALTER TABLE`, không tự chạy migration, không tạo Flyway migration trong backend.

- [ ] **0.5 Lưu kết quả**: dán đầu ra rút gọn (chỉ tên cột thiếu/thừa, không có mật khẩu) vào mô tả commit cuối của B01.

#### Nhóm cột `appraisal_ship_description` (nguồn cho Step 4)

Kiểu Java: `S` = `String`, `I` = `Integer`, `D` = `BigDecimal`, `B` = `Boolean`, `Date` = `LocalDate`. `S/100` = `@Column(length = 100)`; `S/T` = `@Column(columnDefinition = "TEXT")`; `D/10,2` = `@Column(precision = 10, scale = 2)`. Nguồn chuẩn: `L entity/AppraisalShipDescription.java` (đọc nguyên file để copy `length`/`precision`, không đoán). Tên field Java là camelCase của tên cột (`ship_name` → `shipName`, `anchor_bow_total_weight_kg` → `anchorBowTotalWeightKg`, `steering_rudder_total_area_m2` → `steeringRudderTotalAreaM2`).

| Nhóm | Cột |
|---|---|
| Khóa (không nằm trong 155) | `id` (IDENTITY), `appraisal_report_id` `Long` NOT NULL, `appraisal_asset_group_id` `Long` NOT NULL |
| Thông số chính (22) | `ship_name` S/500, `registration_number` S/255, `ship_type_purpose` S/500, `registry_number` S/255, `owner_name` S/500, `build_year` I, `build_place` S/T, `mooring_location` S/255, `dim_length`, `dim_breadth`, `dim_depth`, `dim_draft`, `dim_length_max`, `dim_breadth_max`, `free_board` đều D/10,2, `material` S/255, `gross_tonnage` I, `ship_class` S/255, `operating_area` S/255, `cargo_capacity` D/12,2, `deadweight` D/12,2, `operation_note` S/T |
| Thiết kế (5) | `design_symbol` S/255, `design_year` I, `design_certificate_number` S/255, `appraisal_number` S/255, `conversion_year_place` S/500 |
| Neo (8) | `anchor_bow_type` S/100, `anchor_bow_quantity` I, `anchor_bow_total_weight_kg` D/12,2, `anchor_stern_type` S/100, `anchor_stern_quantity` I, `anchor_stern_total_weight_kg` **S/T**, `anchor_chain_cable` S/255, `anchor_windlass_type` S/100 |
| Lái (7) | `steering_rudder_type` S/100, `steering_rudder_quantity` I, `steering_rudder_total_area_m2` D/12,2, `steering_main_gear_type` S/100, `steering_main_gear_torque_tm` D/12,2, `steering_backup_gear_type` S/100, `steering_backup_gear_torque_tm` D/12,2 |
| Chằng buộc (4) | `mooring_rope_quantity` I, `mooring_rope_total_length_m` D/12,2, `mooring_bollard_type` S/100, `mooring_bollard_quantity` **S/T** |
| Kéo (4, đều S/T) | `towing_bollard_type`, `towing_hook_type`, `towing_winch_type`, `towing_rope_type` |
| Máy chính (5) | `main_engine_quantity` I, `main_engine_brand` S/255, `main_engine_serial_number` S/255, `main_engine_manufacture_year` I, `main_engine_power_hp` D/10,2 |
| Máy phụ (3) | `aux_engine_quantity` I, `aux_engine_brand` S/255, `aux_engine_power_hp` D/10,2 |
| Điện (6) | `electric_generator` S/100, `electric_generator_power_kva` S/100, `electric_generator_voltage` S/255, `electric_battery_quantity` I, `electric_battery_voltage_v` D/10,2, `electric_battery_total_capacity` S/255 |
| Khí nén (2, S/T) | `compressed_air_tank_quantity`, `compressed_air_compressor_quantity` |
| Nâng (3, S/255) | `lifting_serial_number`, `lifting_max_reach_capacity`, `lifting_min_reach_capacity` |
| Chữa cháy (17) | `fire_pump_quantity` I, `fire_pump_total_flow_m3` D/12,2, `fire_gas_type` S/100, `fire_gas_total_weight_kg` D/12,2, `fire_foam_type` S/100, `fire_foam_total_volume_l` S/T, `fire_foam_pusher_quantity` S/T, `fire_foam_pusher_volume_l` S/T, `fire_portable_foam` S/100, `fire_co2_bottle_quantity` I, `fire_co2_bottle_weight_kg` D/12,2, `fire_powder_bottle_quantity` I, `fire_powder_bottle_weight_kg` D/12,2, `fire_foam_bottle_quantity` S/T, `fire_foam_bottle_volume_l` S/T, `fire_manual_tools` B, `fire_layout_plan` B |
| Cứu sinh (4) | `lifesaving_ring_buoy` S/100, `lifesaving_life_jacket` S/100, `lifesaving_raft_boat_quantity` I, `lifesaving_raft_boat_total_capacity` I |
| Cứu đắm (4) | `damage_control_bilge_pump_quantity` I, `damage_control_bilge_pump_total_flow_m3` D/12,2, `damage_control_tools` S/100, `damage_control_bilge_alarm` S/100 |
| Đèn hiệu (16, đều B) | `signal_light_` + `side_red`, `side_green`, `stern_white`, `mast_white`, `mast_green`, `mast_red`, `anchor`, `towing_red`, `towing_green`, `yellow`, `flash_white`, `flash_yellow`, `flash_red`, `other`, `half_green_red`, `spare` |
| Âm hiệu (1) | `signal_sound` S/100 |
| Vật hiệu (10, đều B) | `signal_shape_` + `black_triangle`, `white_triangle`, `black_square`, `black_rectangle`, `black_circle`, `white_circle`, `black_diamond`, `flag_set`, `parachute_flare`, `line_thrower` |
| Chạy tàu (14) | B: `navigation_binoculars`, `_clock`, `_depth_sounder`, `_order_bell`, `_inclinometer`, `_draft_gauge`, `_gps`, `_compass`, `_command_pa`, `_radar`, `_epirb`; S/100: `navigation_radio`, `navigation_ais`, `navigation_sart` |
| Ngừa ô nhiễm (15) | S/T: `pollution_oily_water_separator_quantity`, `..._separator_total_output`, `pollution_sewage_tank_quantity`, `..._sewage_tank_total_volume_m3`, `pollution_settling_tank_quantity`, `..._settling_tank_total_volume_m3`, `pollution_sludge_tank_quantity`, `..._sludge_tank_total_volume_m3`, `pollution_standard_flange_quantity`, `pollution_garbage_container_quantity`, `pollution_sewage_treatment_quantity`, `pollution_sewage_treatment_output_m3`; **không phải TEXT**: `pollution_oily_water_tank_quantity` I, `pollution_oily_water_tank_total_volume_m3` D/12,2, `pollution_standard_flange_position` S/255 |
| Thời hạn kiểm tra (3) | `inspection_periodic_date`, `inspection_dock_date`, `inspection_annual_date` đều `Date` |
| JSON (2) | `condition_details` `List<ShipConditionDetail>` (`columnDefinition = "JSON"`, `@Convert(ShipConditionDetailsConverter)`), `quality_evaluation_details` `List<ShipQualityEvaluationItem>` (`@Convert(ShipQualityEvaluationConverter)`) |
| Cuối | `delete_flag` `Integer` NOT NULL, `created` `LocalDateTime` (`updatable = false`) |

Các nhóm giá trị cộng lại là 155 cột (22+5+8+7+4+4+5+3+6+2+3+17+4+4+16+1+10+14+15+3+2). Phần mô tả bằng chữ trong `surveys/legacy-be.md` mục 2 có chỗ gom nhóm khác; khi lệch thì **code L là chuẩn**.

Quan hệ chỉ đọc (giống `AppraisalVehicleDescription`):

```java
@OneToMany(fetch = FetchType.LAZY)
@JoinColumn(name = "appraisal_ship_description_id", referencedColumnName = "id", insertable = false, updatable = false)
@SQLRestriction("deleted = 0 OR deleted IS NULL")
private List<AppraisalImage> appraisalImages;

@OneToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "appraisal_asset_group_id", insertable = false, updatable = false)
private AppraisalAssetGroup appraisalAssetGroup;
```

Trong `AppraisalAssetGroup` thêm đầu còn lại: `@OneToOne(mappedBy = "appraisalAssetGroup", fetch = FetchType.LAZY) private AppraisalShipDescription appraisalShipDescription;` (cho Word, giống `appraisalVehicleDescription`).

---

#### Step 1 — Hằng số (loại tài sản, cơ sở thẩm định, loại ảnh, nhãn yếu tố)

- [ ] **1.1 Viết test FAIL** `NT common/constant/ShipConstantsTest.java`:
  - `ship_bằng_4_ở_cả_hai_nơi_khai_báo`: `AppraisalAssetType.SHIP == 4` và `CommonConstants.AppraisalAssetType.SHIP == AppraisalAssetType.SHIP`.
  - `cơ_sở_thẩm_định_tàu_bằng_4`: `ValuationBasis.ASSET_TYPE_SHIP == 4` và `CommonConstants.ValuationBasisAssetType.SHIP == 4`.
  - `loại_ảnh_tàu_từ_21_đến_30_liên_tục_không_trùng_ô_tô`: 10 hằng `ReportConstants.AppraisalImageType.SHIP_*` đúng bảng 21..30 trong `00-index.md`; không trùng 16/18/19/20 và `OTHER = 10`.
  - `tám_nhãn_yếu_tố_tàu_đúng_chữ`: 7 hằng mới + `REMAINING_QUALITY` cho ra đúng chuỗi thứ tự `Năm đóng, Nơi đóng, Ngoại quan tổng thể, Khả năng vận hành, Trọng tải toàn phần, Chất lượng còn lại, Công suất (CV), Cấp phương tiện` (test tự ghép danh sách theo thứ tự này và so sánh với chuỗi nguyên văn).
- [ ] **1.2 Thêm test vào test hiện có** (FAIL):
  - `NT service/ReportContentMergeTest`: `cơ_sở_thẩm_định_loại_4_được_gộp_sau_loại_3` (master `assetType = 4` xuất hiện trong kết quả `valuationBases`, nằm sau mục loại 3).
  - `NT service/AppraisalReportFormServiceTest`: `giữ_cơ_sở_thẩm_định_loại_tàu` (dựng như test `dropsValuationBasisWithAnUnknownAssetType`: mục loại 4 được giữ, loại 9 vẫn bị bỏ).
- [ ] **1.3 Chạy FAIL**: `cd backend && ./mvnw -q test -Dtest=ShipConstantsTest,ReportContentMergeTest,AppraisalReportFormServiceTest` (lỗi biên dịch/assert là FAIL mong đợi).
- [ ] **1.4 Cài đặt**:
  - `N common/constant/AppraisalAssetType.java`: `public static final int SHIP = 4;` (Javadoc: tàu thủy).
  - `N common/constant/CommonConstants.java`: `AppraisalAssetType.SHIP = 4; // Tàu thủy` và `ValuationBasisAssetType.SHIP = 4`. **Không** sửa `RegulatoryConstraintsPriceUnit`, `REGULATORY_CONSTRAINTS_PRICE_UNITS`, `resolveRegulatoryConstraintsPriceUnit`.
  - `N model/entity/ValuationBasis.java`: `ASSET_TYPE_SHIP = 4`; sửa Javadoc dòng "Giá trị asset_type: 0 chung, 1 đất, 2 chung cư, 3 ô tô, 4 tàu thủy".
  - `N service/ReportContentMerge.java` (vòng lặp ~l.69) và `N service/AppraisalReportFormService.java` (~l.66): thay cận trên `ASSET_TYPE_CAR` bằng `ASSET_TYPE_SHIP`; sửa Javadoc "ngoài 0..3" thành "ngoài 0..4".
  - `N common/util/ReportConstants.java`: thêm 10 hằng `SHIP_*` ngay sau `CAR_CHASSIS_AND_ODOMETER`, mỗi hằng một comment tiếng Việt như bảng ở `00-index.md`.
  - `N common/constant/ControlCompareFactorLabels.java`: thêm `BUILD_YEAR`, `BUILD_PLACE`, `OVERALL_APPEARANCE`, `OPERABILITY`, `DEADWEIGHT`, `POWER_HP`, `VESSEL_CLASS` (chỉ hằng tên; `case SHIP` trong `ControlFactorDefaults` thuộc B07 (Quyết định #11), **không sửa** ở B01; B07 dùng các hằng này, thay literal `"Khả năng vận hành"` bằng `OPERABILITY`).
- [ ] **1.5 Chạy PASS** cùng lệnh 1.3, rồi `./mvnw -q test -Dtest=ControlFactorDefaultsTest,MoneyWordsTest` để chắc không vỡ test cũ.
- [ ] **1.6 Commit**: `git -C backend add -A && git -C backend commit -m "feat: hằng số loại tài sản tàu thủy, loại ảnh 21-30 và nhãn yếu tố tàu"`.

#### Step 2 — `CurrencyCodes` và đọc tiền USD

- [ ] **2.1 Viết test FAIL** `NT common/util/CurrencyCodesTest.java` (port 4 case đầu của `CurrencyCodeTest`, dùng `AppraisalAssetType.*` mới):
  - `ship_dùng_được_usd`: `(SHIP,"USD")` và `(SHIP,"usd")` ra `"USD"`.
  - `không_phải_tàu_hoặc_mã_lạ_về_vnd`: `(LAND,"USD")`, `(CAR,"USD")`, `(null,"USD")`, `(SHIP,null)`, `(SHIP,"EUR")` đều ra `"VND"`.
  - `nhãn_đơn_vị_và_nhãn_tiền`: `currencyUnitLabel("USD")="USD"`, `("VND")="VND"`; `currencyMoneyLabel("USD")="USD"`, `("VND")="đồng"`.
  - `đơn_vị_ràng_buộc_của_tàu_không_có_m2`: `(SHIP,"USD")="USD"`, `(SHIP,"VND")="đồng"`, `(LAND,"USD")="đồng/m²"`, `(CAR,null)="đồng/xe"`.
- [ ] **2.2 Thêm test** vào `NT common/util/MoneyWordsTest` (case thứ 5): `đọc_tiền_usd_đổi_hậu_tố`: `read(1234567L,"VND") == read(1234567L)`; `read(1234567L,"USD") == read(1234567L).replaceAll(" đồng$"," đô la Mỹ")`; `read(0L,"USD") == "Không đô la Mỹ"`; `read(5L, null)` giữ " đồng"; số vượt giới hạn (trả `""`) vẫn là `""`.
- [ ] **2.3 Chạy FAIL**: `./mvnw -q test -Dtest=CurrencyCodesTest,MoneyWordsTest`.
- [ ] **2.4 Cài đặt**:
  - `N common/util/CurrencyCodes.java` (`final`, constructor private, Javadoc ghi: port `resolveCurrencyCode` / `currencyUnitLabel` / `currencyMoneyLabel` / `resolveRegulatoryConstraintsPriceUnit(assetType, currencyCode)` của `CommonConstants` L l.435–497). `resolveCurrencyCode` so sánh `equalsIgnoreCase("USD")` và `assetType` bằng `Integer.valueOf(AppraisalAssetType.SHIP).equals(assetType)`. `regulatoryPriceUnit`: SHIP thì theo mã đã resolve; còn lại gọi `CommonConstants.resolveRegulatoryConstraintsPriceUnit(assetType)`. Nhắc trong Javadoc: B05 mới chuyển `RegulatoryConstraints` sang dùng hàm này.
  - `N common/util/MoneyWords.java`: thêm `read(long amount, String currencyCode)`: gọi `read(amount)`; nếu `"USD".equals(currencyCode)` và kết quả kết thúc " đồng" thì thay bằng " đô la Mỹ". Cập nhật Javadoc lớp.
- [ ] **2.5 Chạy PASS** cùng lệnh 2.3.
- [ ] **2.6 Cập nhật `backend/CLAUDE.md`** (cùng commit): thêm hai dòng vào bảng helper:
  - `| Mã/nhãn/đơn vị tiền tệ của nhóm (chỉ tàu mới được USD) | CurrencyCodes.resolveCurrencyCode/currencyUnitLabel/currencyMoneyLabel/regulatoryPriceUnit (common/util/CurrencyCodes) | Tự so chuỗi "USD", hay tự chọn "đồng/m²" cho tàu |`
  - `| Chất lượng còn lại của tàu, kích thước "Lm x Bm", 18 hạng mục tình trạng mặc định | ShipQuality (service/ship) | Tự cộng w x (r/s) x 100 hay ghép chuỗi kích thước trong service/Word |`
  - và sửa dòng "Số tiền bằng chữ" thêm: `MoneyWords.read(amount, currencyCode)` cho USD.
- [ ] **2.7 Commit**: `feat: helper tiền tệ CurrencyCodes và đọc tiền bằng chữ theo USD`.

#### Step 3 — Record JSON, converter và lớp thuần `ShipQuality`

- [ ] **3.1 Viết test FAIL**:
  - `NT model/converter/ShipConditionDetailsConverterTest` (theo mẫu `StructureDetailsConverterTest`): `đọc_hai_trường_mỗi_dòng`; `bỏ_qua_khóa_lạ` (`[{"itemName":"Thân vỏ","conditionDescription":null,"extra":1}]`); `null_rỗng_hoặc_hỏng_thành_danh_sách_rỗng` (`null`, `"  "`, `"{khong phai json"`, `"{\"a\":1}"`); `ghi_null_cho_danh_sách_rỗng` (`null` và `List.of()` ghi `null`); `ghi_rồi_đọc_lại_giữ_nguyên`.
  - `NT model/converter/ShipQualityEvaluationConverterTest`: `đọc_cấu_trúc_mới` (`[{"itemNames":["Vỏ tàu","Boong"],"importanceWeight":0.3,"standardValue":10,"rating":"Khá","relativeValue":7}]` ra `itemNames=[Vỏ tàu, Boong]`, `importanceWeight` so sánh `0.3`, `standardValue 10`, `rating Khá`, `relativeValue 7`); `đọc_cấu_trúc_cũ_không_lỗi` (có `itemName` đơn và `subItems`: được đọc với `itemNames == null`, không ném lỗi); `bỏ_qua_khóa_lạ`; `null_rỗng_hoặc_hỏng_thành_danh_sách_rỗng`; `ghi_rồi_đọc_lại` (khóa JSON ghi ra đúng tên field: chứa `"itemNames"`, `"importanceWeight"`, `"standardValue"`, `"rating"`, `"relativeValue"`, vì legacy đọc đúng các khóa này).
  - `NT service/ship/ShipQualityTest`, bộ số liệu tính tay (mỗi ca ghi phép tính trong comment):
    - `một_dòng`: w=0,3; s=10; r=7 → 0,3×0,7×100 = 21 → `21.00`.
    - `hai_dòng_cộng_lại`: (0,3; 10; 7) + (0,7; 8; 6): 21 + 0,7×0,75×100 = 52,5 → `73.50`.
    - `tỷ_lệ_làm_tròn_scale_10`: (0,5; 3; 1) + (0,5; 3; 2): ratio 0,3333333333 và 0,6666666667 → 16,666666665 + 33,333333335 = 50 → `50.00`.
    - `làm_tròn_HALF_UP_hai_lẻ`: w=0,25; s=8; r=1 → 3,125 → `3.13`.
    - `đúng_100`: w=1; s=10; r=10 → `100.00`; `đúng_0`: w=0; s=5; r=3 → `0.00` (có item hợp lệ nên không null).
    - `s_bằng_0_bị_bỏ_qua`: (1; 0; 5) + (0,4; 5; 5) → chỉ dòng sau tính: `40.00`; chỉ có dòng s=0 → `null`.
    - `s_âm_bị_bỏ_qua`: s=-5 → bỏ (null nếu là dòng duy nhất).
    - `thiếu_số_bỏ_qua`: dòng thiếu w, hoặc thiếu s, hoặc thiếu r bị bỏ; dòng khác vẫn tính; `rating` không ảnh hưởng (đổi `rating` không đổi kết quả).
    - `tổng_trên_100_trả_null`: (0,8; 5; 6) → 96 và (0,1; 1; 1) → 10: tổng 106 → `null`.
    - `tổng_âm_trả_null`: (1; 5; -1) → -20 → `null`.
    - `danh_sách_null_hoặc_rỗng_trả_null`; `phần_tử_null_bị_bỏ_qua`.
    - **`bộ_số_liệu_chung_Q1_đến_Q9`** (Quyết định #12; `@ParameterizedTest` hoặc 9 ca, bộ này dùng chung với `ship-quality.spec.ts` của F03 trong `11-fe-ship-description.md`; mỗi dòng là `(w; s; r)`, so `compareTo`): Q1 `(0,3; 5; 4), (0,7; 3; 2)` thành `70.67`; Q2 `(1; 3; 1)` thành `33.33`; Q3 `(0,4; 4; 1)` thành `10.00`; Q4 `(0,5; 0; 1)` thành `null`; Q5 `(0,5; 2; null)` thành `null`; Q6 `(2; 1; 1)` thành `null` (200 > 100); Q7 `(0; 1; 1)` thành `0.00` (không phải null); Q8 Q1 cộng `(0,5; 0; 1)` thành `70.67`; Q9 danh sách rỗng thành `null`. Các ca tính tay phía trên giữ nguyên, bộ Q là bổ sung.
    - `contribution_không_làm_tròn`: (0,5; 3; 1) cho `16.666666665` (so `compareTo`); thiếu số hoặc s<=0 cho `null`.
    - `formatDimension`: `(118.35, 19.40) → "118,35m x 19,4m"`; `(100.00, 20) → "100m x 20m"` (không ra `1E+2`); `(0.50, 3) → "0,5m x 3m"`; `(null, 19.4) → null`; `(118.35, null) → null`.
    - `defaultConditionDetails`: đúng 18 phần tử; phần tử đầu "Thân vỏ", cuối "Hệ thống chiếu sáng"; mục 6 là "Mặt boong ngoài trời"; mọi `conditionDescription` null; mỗi lần gọi trả danh sách **mới** sửa được (`add` không ném lỗi, không ảnh hưởng lần gọi sau).
- [ ] **3.2 Chạy FAIL**: `./mvnw -q test -Dtest=ShipConditionDetailsConverterTest,ShipQualityEvaluationConverterTest,ShipQualityTest`.
- [ ] **3.3 Cài đặt**:
  - Record `ShipConditionDetail`, `ShipQualityEvaluationItem` (cả hai `@JsonIgnoreProperties(ignoreUnknown = true)` import `com.fasterxml.jackson.annotation`, như `PlantDetail`). **Mang sẵn ràng buộc Bean Validation** vì `ShipDescriptionInput` (B03) cascade `@Valid` vào hai record này và B03 không được sửa file của B01: `ShipConditionDetail.itemName` `@Size(max = 500, message = "Hạng mục không được vượt quá 500 ký tự")`, `conditionDescription` **không giới hạn độ dài** (L `entity/dto/ShipConditionDetail.java` không có `@Size`); `ShipQualityEvaluationItem.itemNames` phần tử `List<@Size(max = 500, message = "Hạng mục thẩm định không được vượt quá 500 ký tự") String>`, `importanceWeight`/`standardValue`/`relativeValue` `@PositiveOrZero(message = "… không được âm")` + `@Digits(integer = 8, fraction = 2, message = "… tối đa 8 chữ số nguyên và 2 chữ số thập phân")`, `rating` `@Size(max = 50, message = "Đánh giá không được vượt quá 50 ký tự")` (nhãn và message đúng như mục "Quy tắc ràng buộc" của `02-be-description-images.md`). Thêm test thuần trong `ShipQualityTest` hoặc test converter: 501 ký tự/số âm bị `Validator` báo vi phạm; `ShipCompareFields`, `ShipLegalFields` (không cần annotation, Javadoc nêu B05/B04 dựng từ DTO của mình). Javadoc ghi: tên khóa JSON phải giữ nguyên như L vì legacy cũng đọc/ghi cột này; **không** có `itemName` đơn và `subItems`.
  - Hai converter `public`, `@Converter`, đặt trong `model/converter` (cùng package với `JsonListConverter` package-private), `elementType()` trả `new TypeReference<>() { }`; giữ `emptyAsNull()` mặc định (rỗng ghi NULL, đúng L). Javadoc ghi cột `appraisal_ship_description.condition_details` / `quality_evaluation_details`.
  - `ShipQuality`: hằng `DEFAULT_CONDITION_ITEMS` 18 mục đúng chữ L l.360–378. `contribution` dùng `divide(s, 10, RoundingMode.HALF_UP)` rồi `multiply(w).multiply(100)`. `remainingQualityPct` đúng thuật toán L l.313–330. Javadoc ghi: FE `ship-quality.ts` (F03) và Word (B08) phải cho cùng kết quả; "(4) làm tròn 1 chữ số chỉ để hiển thị" là việc của FE/Word, không nằm ở đây.
- [ ] **3.4 Chạy PASS** cùng lệnh 3.2.
- [ ] **3.5 Commit**: `feat: record JSON, converter và công thức chất lượng còn lại của tàu`.

#### Step 4 — Entity `AppraisalShipDescription` và repository

- [ ] **4.1 Test helper** `NT repository/RepositoryTestData.java`: thêm `static long insertReport(EntityManager em)` (`insertRequest(...)` rồi `INSERT INTO appraisal_reports (appraisal_request_id, delete_flag) VALUES (...)`, trả `lastInsertId`) và `static long insertGroup(EntityManager em, long reportId, int assetType)` (`INSERT INTO appraisal_asset_groups (appraisal_report_id, type, asset_type, land_valuation_method, delete_flag) VALUES (:r, 'SINGLE', :t, 'MIXED', 0)`). Ghi chú Javadoc: dùng chung cho test các task ship.
- [ ] **4.2 Viết test FAIL (thuần, không DB)** `NT model/entity/AppraisalShipDescriptionTest`:
  - `empty_đặt_mặc_định`: `empty(10L, 20L, now)` có report/group đúng, `deleteFlag == 0`, `created == now`, `conditionDetails` có 18 mục, `qualityEvaluationDetails` null (hoặc rỗng nếu getter trả list; so sánh theo `isNullOrEmpty`), mọi cột mô tả null.
  - `softDelete_và_revive`: `softDelete` → 1; `revive` → 0; `revive` không xóa `shipName` đã đặt (đặt bằng `ReflectionTestUtils.setField`).
  - `resetToDefault_xóa_mọi_cột_mô_tả_giữ_khóa`: điền **mọi** field có `@Column` trừ `id`, `appraisalReportId`, `appraisalAssetGroupId`, `deleteFlag`, `created` bằng giá trị mẫu theo kiểu (reflection: String "x", Integer 7, BigDecimal 12.34, Boolean true, LocalDate 2026-11-04; hai list JSON bằng một phần tử), đặt `deleteFlag = 1`, gọi `resetToDefault()`, rồi khẳng định: mọi field kiểu String/Integer/BigDecimal/Boolean/LocalDate bằng null; `conditionDetails` có 18 mục mặc định; `qualityEvaluationDetails` null; `deleteFlag == 0`; `id`, `created`, `appraisalReportId`, `appraisalAssetGroupId` không đổi. Test này bắt buộc để chứng minh không sót cột nào trong ~155 cột.
- [ ] **4.3 Viết test FAIL (DB thật)** `NT repository/AppraisalShipDescriptionRepositoryTest` (`@MysqlDataJpaTest`, mẫu `AppraisalVehicleDescriptionRepositoryTest`; mỗi test tự dựng báo cáo + nhóm `assetType = SHIP` bằng `RepositoryTestData`, rollback cuối test):
  - `mọi_cột_của_bảng_đều_được_map`: lấy danh sách cột thật `SELECT COLUMN_NAME FROM information_schema.COLUMNS WHERE TABLE_SCHEMA = DATABASE() AND TABLE_NAME = 'appraisal_ship_description'` và đối chiếu với tập tên cột suy ra từ `@Column`/`@JoinColumn` (không `insertable=false`) của entity bằng reflection; hai tập phải **bằng nhau** (bắt cả cột thiếu lẫn cột map thừa).
  - `đọc_cột_do_hệ_thống_cũ_ghi`: `INSERT` thẳng (SQL native) một dòng có `ship_name`, `build_year = 2005`, `build_place` dài > 255 ký tự, `dim_length_max = 118.35`, `dim_depth = 3.25`, `deadweight = 5300.50`, `anchor_stern_total_weight_kg = '2 x 150 kg'`, `fire_manual_tools = 1`, `signal_light_side_red = 1`, `inspection_dock_date = '2026-03-15'`, `condition_details = '[{"itemName":"Thân vỏ","conditionDescription":"Tốt"}]'`, `quality_evaluation_details` (cấu trúc mới); đọc bằng `findByAppraisalAssetGroupId` và kiểm từng giá trị/kiểu (BigDecimal bằng `isEqualByComparingTo`).
  - `ghi_đủ_mọi_cột_rồi_đọc_lại_bằng_nhau`: `empty(...)`, điền mọi field mô tả bằng giá trị mẫu theo kiểu (như 4.2, String "x" vừa mọi `length`; BigDecimal 12.34 vừa mọi precision), `saveAndFlush`, `em.clear()`, nạp lại, so từng field bằng reflection (BigDecimal bằng `compareTo`); kiểm `created` được ghi, `deleteFlag = 0`, `conditionDetails` 18 mục sau vòng JSON.
  - `json_khóa_lạ_không_lỗi`: `INSERT` có `condition_details = '[{"itemName":"A","conditionDescription":"B","khoaLa":1}]'` và `quality_evaluation_details = '[{"itemNames":["A"],"importanceWeight":0.3,"standardValue":10,"rating":"Khá","relativeValue":7,"khoaLa":{"x":1}}]'`; nạp entity không lỗi, giá trị đọc đúng.
  - `json_cấu_trúc_cũ_không_lỗi`: `quality_evaluation_details = '[{"itemName":"Vỏ","importanceWeight":0.3,"subItems":[{"itemName":"x"}]}]'`: nạp được, phần tử có `itemNames == null`.
  - `bản_ghi_xóa_mềm_vẫn_tìm_được_và_unique_theo_nhóm`: dòng `delete_flag = 1` vẫn trả từ `findByAppraisalAssetGroupId`; chèn dòng thứ hai cùng nhóm bằng SQL ném `DataIntegrityViolationException` (chứng minh B07 phải dùng lại id qua `resetToDefault`).
  - `reset_giữ_id_và_ghi_đè_cột`: nạp dòng đã có dữ liệu, `resetToDefault()`, `saveAndFlush`, `clear`, nạp lại: cùng `id`, `ship_name` null, `condition_details` 18 mục, `quality_evaluation_details` NULL ở DB (`SELECT quality_evaluation_details IS NULL`).
  - `ảnh_chỉ_gồm_ảnh_chưa_xóa`: chèn 2 dòng `appraisal_image` (cột `appraisal_ship_description_id`) với `deleted = 0` và `deleted = 1`; `getAppraisalImages()` trả 1 phần tử (gọi trong cùng transaction test, sau `em.clear()`).
- [ ] **4.4 Chạy FAIL**: `./mvnw -q test -Dtest=AppraisalShipDescriptionTest,AppraisalShipDescriptionRepositoryTest`.
- [ ] **4.5 Cài đặt**:
  - `N model/entity/AppraisalShipDescription.java`: `@Entity @Getter @NoArgsConstructor(access = AccessLevel.PROTECTED) @Table(name = "appraisal_ship_description")`, Javadoc nêu: bảng dùng chung với hệ thống cũ, map theo DDL cuối, `apply(...)` do B03 thêm. Khai báo field theo bảng "Nhóm cột" (mỗi nhóm một dòng comment `// ===== Neo =====` như L để dễ rà). Mọi kiểu `Boolean` nullable (không `nullable = false`). Hai cột JSON `columnDefinition = "JSON"`. Phương thức `empty`, `resetToDefault` (private `clearDescriptionColumns()` gán null từng cột; **không** dùng reflection ở mã chạy), `softDelete`, `revive`.
  - `N repository/AppraisalShipDescriptionRepository.java` (`JpaRepository<AppraisalShipDescription, Long>`, Javadoc như bản vehicle).
  - `N model/entity/AppraisalAssetGroup.java`: thêm quan hệ `appraisalShipDescription` (xem trên). Chưa thêm cột tiền tệ ở step này.
- [ ] **4.6 Chạy PASS** cùng lệnh 4.4. Nếu `mọi_cột_của_bảng_đều_được_map` lệch: sửa entity theo DB (không sửa DB); nếu DB thiếu cột so với DDL cuối thì **dừng và báo** (xem tiêu chí dừng Step 0).
- [ ] **4.7 Commit**: `feat: entity và repository mô tả tàu thủy (appraisal_ship_description)`.

#### Step 5 — Cột và phương thức tàu trên entity sẵn có

- [ ] **5.1 Viết test FAIL (thuần)**:
  - `NT model/entity/AppraisalGroupCompareShipTest` (mẫu `AppraisalGroupCompareFormTest`, dùng `GroupCompareFixtures.saved(id, type)`):
    - `applyShipForm_trim_và_làm_tròn`: chuỗi `"  Bắc Ninh "` → `"Bắc Ninh"`, `"   "` → null; `88.888` → `88.89`; `buildYear` giữ nguyên; `forSalePriceUsd` ghi.
    - `applyShipForm_giá_của_tàu`: `(trade=5000, estimate=3000)` → `marketPrice 0`, `apartmentOwnershipRightPrice 0`, `landUseRightPrice 5000`; `(0, 3000)` → `landUseRightPrice 3000`; `(0, 0)` → 0.
    - `applyShipForm_không_đụng_cột_ô_tô_và_đất`: đặt trước `carBrand = "Kia"`, `address = "A"`; sau khi gọi vẫn nguyên.
    - `applyShipSync_chỉ_ghi_khi_nguồn_có_dữ_liệu`: mô tả có `buildYear 2005`, `buildPlace "Hải Phòng"`, `shipTypePurpose "Chở hàng"`, `shipClass null`, `deadweight 5300.5`, `dimLengthMax 118.35`, `dimBreadthMax 19.4`, `mainEnginePowerHp 800`; so sánh đã có `shipClass = "VR-SI"`, `address = "Cũ"` → sau sync: các trường có nguồn được ghi đè, `shipClass` giữ `"VR-SI"`, `shipDimension == "118,35m x 19,4m"`, `address` đổi theo tham số khi khác null, giữ `"Cũ"` khi tham số null, `shipRemainingQualityPct` theo tham số (null giữ cũ).
    - `applyShipSync_thiếu_một_cạnh_thì_giữ_kích_thước_cũ`: `dimBreadthMax null` → `shipDimension` không đổi.
    - `applyShipSync_không_đụng_giá_và_ship_for_sale_price_usd`.
    - `clearShipFields_xóa_cả_9_cột`; `reset_xóa_cả_cột_tàu` (`reset("addr")` gọi `clearShipFields`).
  - `NT model/entity/AppraisalReportAssetShipTest`: `applyShipLegal_chỉ_ghi_nhóm_cột_tàu` (owner, asset, gcnNo, gcnNoExpiryDate, other, 4 cột `carInspection*` giữ nguyên; 10 giá trị của `ShipLegalFields` được ghi).
  - `NT model/entity/AppraisalAssetGroupCurrencyTest`: `nhóm_mới_mặc_định_vnd_tỷ_giá_1` (`AppraisalAssetGroup.create(...)` có `currencyCode "VND"`, `exchangeRate 1`, `currencyNote` null); `applyCurrency_usd_có_tỷ_giá` (`("USD", 25450L, "  Tỷ giá VCB  ")` → note `"Tỷ giá VCB"`); `applyCurrency_usd_không_tỷ_giá_giữ_tỷ_giá_cũ` (cũ 24000); `applyCurrency_vnd_ép_tỷ_giá_1_và_note_rỗng_thành_null` (`("VND", 25450L, "  ")` → 1, null); `applyCurrency_mã_lạ_ném_IllegalArgumentException` (`"EUR"`, `null`); `resetCurrency_về_mặc_định`; `resetForAssetType_cũng_reset_tiền_tệ` (nhóm USD/25000/note → `resetForAssetType(LAND, now)` → VND/1/null).
  - `AppraisalImage.ofShip`: thêm vào `NT repository/AppraisalImageRepositoryTest`? Không sửa file đó; kiểm trong `ShipColumnsRepositoryTest` bên dưới.
- [ ] **5.2 Viết test FAIL (DB thật)** `NT repository/ShipColumnsRepositoryTest` (`@MysqlDataJpaTest`, chứng minh map đúng schema thật cho cột mới của 5 bảng hiện có):
  - `dòng_so_sánh_đọc_9_cột_ship_do_hệ_thống_cũ_ghi`: `INSERT INTO appraisal_group_compares` (cột bắt buộc theo `forGroup` + `ship_*`) rồi đọc bằng `AppraisalGroupCompareRepository`; kiểm 9 getter và kiểu (`ship_build_year` Integer, `ship_remaining_quality_pct` 77.50, `ship_for_sale_price_usd` 123456.78).
  - `dòng_so_sánh_ghi_applyShipForm_và_applyShipSync_rồi_đọc_lại`: `forGroup(...)`, `applyShipForm(...)`, `saveAndFlush`, `clear`, nạp lại; sau đó `applyShipSync` với mô tả đã lưu, lưu lại, nạp lại; sau đó `clearShipFields`, lưu, nạp lại thấy 9 cột null (`SELECT` native xác nhận NULL ở DB).
  - `tài_sản_pháp_lý_ghi_đọc_5_cột_ship_inspection`: `AppraisalReportAsset.create(...)`, `applyShipLegal(...)`, lưu qua `AppraisalReportAssetRepository`, nạp lại: 5 cột + `certificateNumber`/`forWhom`/`gcnNoIssuedDate`/`legalDocuments` đúng; ngày là `LocalDate`.
  - `ảnh_tàu_ghi_đọc_khóa_ngoại`: tạo mô tả tàu (`empty` + lưu), `AppraisalImage.ofShip(id, ReportConstants.AppraisalImageType.SHIP_FRONT, "ghi chú", "a.jpg", "ship/1/2/x/a.jpg", uuid, now)` lưu qua `AppraisalImageRepository`; nạp lại: `getAppraisalShipDescriptionId()` đúng, `getAppraisalLandDescriptionId()` và `getAppraisalVehicleDescriptionId()` null, `type == 21`, `deleted == 0`; `mô tả.getAppraisalImages()` chứa ảnh này.
  - `kho_tsss_đọc_8_cột_ship_và_currency_code`: `INSERT INTO compare_information (created, updated, created_by, deleted, asset_type, ship_build_year, ship_build_place, ship_usage_purpose, ship_class, ship_remaining_quality_pct, ship_deadweight, ship_dimension, ship_main_engine_power_hp, currency_code)` (theo `CompareInformationSearchRepositoryTest`: `created_by = 1`, `deleted = 0`, `asset_type = 4`; nếu DB báo cột NOT NULL khác thì thêm giá trị 0) rồi đọc qua `CompareInformationRepository.findById`; kiểm 9 getter; thêm một dòng `currency_code` NULL đọc ra `null` (B10 hiểu NULL là VND).
  - `nhóm_ghi_đọc_tiền_tệ`: nhóm chèn bằng `insertGroup` (cột tiền tệ lấy default của DB) đọc ra `VND`/`1`/null; `applyCurrency("USD", 25450L, "Tỷ giá VCB")` lưu rồi nạp lại đúng; một nhóm tạo bằng `AppraisalAssetGroup.create(...)` rồi `save` **không** vi phạm NOT NULL của `currency_code`/`exchange_rate`.
  - `nhóm_có_đầu_quan_hệ_mô_tả_tàu`: nhóm có mô tả tàu thì `group.getAppraisalShipDescription()` khác null (sau `clear`).
- [ ] **5.3 Chạy FAIL**: `./mvnw -q test -Dtest=AppraisalGroupCompareShipTest,AppraisalReportAssetShipTest,AppraisalAssetGroupCurrencyTest,ShipColumnsRepositoryTest`.
- [ ] **5.4 Cài đặt** (tất cả theo Interfaces và "Ngữ nghĩa chốt"; mọi cột dùng `@Column(name = ...)` khớp schema, không `@Setter`):
  - `AppraisalGroupCompare`: 9 field `shipBuildYear` (`Integer`), `shipBuildPlace` (TEXT), `shipUsagePurpose` (TEXT), `shipClass` (TEXT), `shipRemainingQualityPct` (`precision 5, scale 2`), `shipDeadweight` (12,2), `shipDimension` (TEXT), `shipMainEnginePowerHp` (10,2), `shipForSalePriceUsd` (15,2); `applyShipForm`, `applyShipSync` (private `applyIfPresent`), `clearShipFields`; gọi `clearShipFields()` trong `reset(String)`; cập nhật Javadoc lớp và bảng `applyForm` (thêm dòng "tàu: do `applyShipForm`, B05 nối vào `applyForm`"). **Không** sửa `applyForm`, `copyFrom` (B05/B10).
  - `AppraisalReportAsset`: 5 field `shipInspectionCertificateNumber` (length 500), `shipInspectionIssuedPlace` (500), `shipInspectionIssuedDate`, `shipInspectionExpiryDate` (`LocalDate`), `shipInspectionForWhom` (1000); `applyShipLegal`; cập nhật Javadoc lớp (không đụng `mergeLegalFieldsFrom`, của B04).
  - `AppraisalImageRepository`: thêm `findByAppraisalShipDescriptionIdAndDeletedOrderByCreatedAscIdAsc(Long, Integer)` (Javadoc "Ảnh chưa xóa của mô tả tàu, theo thứ tự tải lên"); test ở `ảnh_tàu_ghi_đọc_khóa_ngoại` (gọi finder này, kỳ vọng đúng 1 ảnh chưa xóa).
  - `AppraisalImage`: field `Long appraisalShipDescriptionId` (`@Column(name = "appraisal_ship_description_id")`), factory `ofShip(...)` (cùng `create(...)`), cập nhật Javadoc lớp ("đất, phương tiện hoặc tàu thủy").
  - `CompareInformation`: 8 cột ship (`shipBuildYear` `Integer`, `shipBuildPlace` TEXT, `shipUsagePurpose` TEXT, `shipClass` TEXT, `shipRemainingQualityPct` 5,2, `shipDeadweight` 12,2, `shipDimension` TEXT, `shipMainEnginePowerHp` 10,2) và `currencyCode` (`length = 10`, nullable); cập nhật Javadoc lớp ("`ship_*` chỉ dùng khi `asset_type = 4`; `currency_code` NULL là VND"). Chưa sửa `copyFrom`.
  - `AppraisalAssetGroup`: `currencyCode` (`nullable = false, length = 10`, khởi tạo `= "VND"`), `exchangeRate` (`nullable = false`, khởi tạo `= 1L`), `currencyNote` (TEXT); `applyCurrency`, `resetCurrency`; `resetForAssetType` gọi `resetCurrency()`; `create(...)` không cần đổi nhờ khởi tạo field (kiểm bằng test 5.2).
- [ ] **5.5 Chạy PASS** cùng lệnh 5.3.
- [ ] **5.6 Chạy các test cũ có thể bị ảnh hưởng**: `./mvnw -q test -Dtest=AppraisalGroupCompareFormTest,GroupCompareSaveRepositoryTest,GroupCompareEntitiesRepositoryTest,AppraisalAssetGroupRepositoryTest,AppraisalReportAssetRepositoryTest,AppraisalImageRepositoryTest,CompareInformationSearchRepositoryTest,CompareInformationSyncTest,GroupAssetTypeResetTest`.
- [ ] **5.7 Commit** (có thể tách hai commit: cột/phương thức của `AppraisalGroupCompare`+`AppraisalReportAsset`; cột của `AppraisalImage`+`CompareInformation`+`AppraisalAssetGroup`): `feat: cột tàu thủy, tiền tệ nhóm và phương thức đồng bộ trên entity sẵn có`.

#### Step 6 — Kiểm tra cuối

- [ ] **6.1** Chạy toàn bộ: `cd backend && ./mvnw test` (cần MySQL như Step 0). Mọi test xanh; không có test cũ bị sửa ngoài các file ở danh sách "Modify tests".
- [ ] **6.2** Kiểm tra không có migration mới: `git -C backend diff --stat origin/develop -- src/main/resources/db/migration` rỗng.
- [ ] **6.3** Rà `git -C backend diff origin/develop --stat`: chỉ chạm các file ở mục Files; `CLAUDE.md` đã có 2 dòng helper mới; Javadoc các entity/helper mới viết tiếng Việt có dấu và nêu lớp L được port.
- [ ] **6.4** Commit nếu còn thay đổi chưa commit: `chore: rà soát nền backend tàu thủy`. **Không push**, không cập nhật con trỏ submodule ở repo gốc.

#### Lệnh kiểm tra tổng

```bash
cd backend
./mvnw -q test -Dtest=ShipConstantsTest,CurrencyCodesTest,MoneyWordsTest,ShipConditionDetailsConverterTest,ShipQualityEvaluationConverterTest,ShipQualityTest,AppraisalShipDescriptionTest,AppraisalShipDescriptionRepositoryTest,AppraisalGroupCompareShipTest,AppraisalReportAssetShipTest,AppraisalAssetGroupCurrencyTest,ShipColumnsRepositoryTest,ReportContentMergeTest,AppraisalReportFormServiceTest
./mvnw test
```

#### Review Focus riêng cho B01

1. Tập cột `@Column` của `AppraisalShipDescription` bằng đúng tập cột trong DB (test `mọi_cột_của_bảng_đều_được_map`); kiểu TEXT/INT/DECIMAL/TINYINT đúng DDL cuối (không dùng `build_year_place`).
2. Không có `@Data`/`@Setter` trên entity; không tạo Flyway migration.
3. `ShipQuality` khớp L: `r` là `relativeValue`, `s` là `standardValue`, bỏ `s <= 0`, scale 10 rồi mới `x 100`, tổng ngoài [0,100] thì null; số liệu tính tay trong test đúng.
4. `AppraisalAssetGroup` khai báo mặc định `VND`/`1` để INSERT không vi phạm NOT NULL; `resetForAssetType` reset tiền tệ.
5. `applyShipForm` ghi đè `marketPrice = 0` (B05 phải gọi sau dòng đặt `marketPrice` trong `applyForm`).
6. `CurrencyCodes`: chỉ SHIP mới USD; `regulatoryPriceUnit` của SHIP không bao giờ có "/m²"; không đụng `REGULATORY_CONSTRAINTS_PRICE_UNITS`.

---

## Vấn đề cần controller quyết định

1. **→ Đã chốt: Quyết định #4 (B01 làm).** **Mở rộng phạm vi nhỏ (đã đưa vào Step 1)**: `ReportContentMerge` (~l.69) và `AppraisalReportFormService` (~l.66) dùng `ASSET_TYPE_CAR` làm cận trên nên loại cơ sở thẩm định 4 bị lọc mất. Không task nào trong bảng của `00-index.md` nhận việc này, nên B01 nhận (đợt B1 chạy một mình, không đụng file task khác). F02 (cơ sở thẩm định tàu ở FE) phụ thuộc việc này. Nếu controller muốn giao cho task khác thì bỏ mục 1.2 và dòng sửa hai file ở 1.4.
2. **Hợp đồng `AppraisalShipDescription.resetToDefault()` không tham số**: giữ `reportId`/`groupId` cũ (1-1 theo nhóm nên không đổi). Nếu B07 cần đổi báo cáo thì phải thêm tham số; chưa thấy trường hợp nào.
3. **→ Đã chốt: Quyết định #2.** **`applyShipLegal`/`applyShipForm` nhận record trung gian** (`ShipLegalFields`, `ShipCompareFields`) thay vì `LegalAssetInput`/`GroupCompareRowInput`, vì hai DTO này do B04/B05 mở rộng ở đợt sau. B04/B05 tự dựng record từ DTO của mình (một dòng `new ...`). Nếu controller muốn chữ ký nhận thẳng DTO thì phải đổi thứ tự: B04/B05 thêm field vào DTO trước.
4. **→ Đã chốt (B07 không gọi thêm reset tiền tệ/cột tàu).** **Gọi lồng reset**: `AppraisalGroupCompare.reset` gọi `clearShipFields`, `AppraisalAssetGroup.resetForAssetType` gọi `resetCurrency`. Plan của B07 nên ghi rõ "không cần gọi thêm" (gọi thêm vô hại).
5. **→ Đã chốt: Quyết định #17 cho fetch.** **Kết nối DB cho Step 0**: không đọc được `backend/.env` trong lúc lập plan (không kiểm tra thực tế schema); các tiêu chí đạt/dừng ở Step 0 lấy từ DDL trong `surveys/legacy-be.md` và migration của L. `origin/develop` của backend lúc lập plan là `b3d6d7e` (lệnh `git fetch` báo lỗi chứng chỉ SSL trong môi trường lập plan nên chưa cập nhật thêm).
6. **Lệch số dòng**: các số dòng của code mới (`ReportContentMerge ~69`, `AppraisalReportFormService ~66`) khớp `origin/develop` `b3d6d7e`; số dòng L `AppraisalGroupCompareService ~250–275` và `AppraisalReportServiceImpl 639–690` là gần đúng, implementer đọc lại trước khi port.
