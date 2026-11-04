# 04 — BE: Kết quả thẩm định + tiền tệ (B06) và vòng đời nhóm (B07)

> Đọc `00-index.md` (hợp đồng, quy ước tên, Global Constraints, Khác hệ thống cũ, Review Focus) và `surveys/legacy-be.md` trước. Subagent: `model: "sonnet"`, mỗi task qua spec review rồi code review. **Yêu cầu người dùng: port đúng hành vi legacy `82193fe5`** — công thức, thứ tự làm tròn, thông điệp lỗi, giá trị null. Chỉ dừng và báo lại khi gặp điều plan không quyết định được.
>
> Viết tắt: L, LR, N, NT như `00-index.md`. Mọi lệnh Maven chạy trong `backend/` (submodule, nhánh `feat/ship-asset-type`). Số dòng của code mới đo trên `origin/develop` = `b3d6d7e`, **phải grep lại** trước khi sửa.

| Task | Đợt | Phụ thuộc | Tập file (rời với các task cùng đợt) |
|---|---|---|---|
| B06 | B2 | B01 | `N/service/result/*`, `N/service/EvaluationResultService`, `N/controller/EvaluationResultController`, `N/model/dto/EvaluationResult*`, `N/common/util/ShipValuationMath` (mới), `N/common/util/export/word/AppraisalResultService` |
| B07 | B3 | B01, **B02** (`AppraisalImageService.deleteAllOfShipDescription`, chỉ dùng ở `GroupAssetTypeReset` khi đổi loại) | `N/service/{ControlFactorDefaults,ReportGroupDetailsWriter,GroupAssetTypeReset,ReportGroupResolver,ReportLegalDocumentValidator}`, `N/model/entity/AppraisalGroupCompare` (chỉ thêm 1 factory), các test tương ứng. **Không** còn `ReportContentMerge`/`AppraisalReportFormService` (Quyết định #4: B01 làm cận 3→4) |

**Giả định về B01** (đối chiếu khối Interfaces của B01 trước khi code; sai tên thì dùng tên thật của B01 và ghi lại trong báo cáo):
- Hằng: `AppraisalAssetType.SHIP = 4` ở **cả hai** nơi (`N/common/constant/AppraisalAssetType`, `CommonConstants.AppraisalAssetType`), `ValuationBasis.ASSET_TYPE_SHIP = 4`.
- `N/common/util/CurrencyCodes`: hằng `VND`, `USD`; `resolveCurrencyCode(Integer assetType, String code)`.
- `AppraisalAssetGroup`: cột `currencyCode` (khởi tạo `"VND"` ở khai báo field), `exchangeRate` (khởi tạo `1L`), `currencyNote`; getter `getCurrencyCode/getExchangeRate/getCurrencyNote`; `applyCurrency(String code, Long rate, String note)` do B01 chốt **có luật**: code đã resolve (`"VND"`/`"USD"`, mã khác ném `IllegalArgumentException`), VND thì rate = 1, USD với rate null thì giữ rate cũ, note được `Texts.trimToNull`, không ghi `updated`; `resetCurrency()` đặt `VND/1/null`.
- `AppraisalShipDescription`: `NOT_DELETED = 0`, `empty(Long reportId, Long groupId, LocalDateTime now)` (đã gán `conditionDetails = ShipQuality.defaultConditionDetails()`), `resetToDefault()` (mọi cột về null/mặc định, **giữ id, created, reportId, groupId**; `conditionDetails` = 18 mục mặc định; `qualityEvaluationDetails` = null), `softDelete()` (`deleteFlag = 1`), `revive()` (`deleteFlag = 0`), `getId()`, `getDeleteFlag()`, `getConditionDetails()`.
- `AppraisalShipDescriptionRepository.findByAppraisalAssetGroupId(Long): Optional<…>` (không lọc cờ xóa, giống `AppraisalVehicleDescriptionRepository`).
- `AppraisalGroupCompare.clearShipFields()` xóa 9 cột tàu (`shipBuildYear, shipBuildPlace, shipUsagePurpose, shipClass, shipRemainingQualityPct, shipDeadweight, shipDimension, shipMainEnginePowerHp, shipForSalePriceUsd`) và **idempotent**.

---

# Task B06 — Kết quả thẩm định tàu + lưu tiền tệ

## Hành vi hệ thống cũ

Nguồn: `L service/impl/AppraisalResultServiceImpl.java` @ `82193fe5`, `L entity/dto/EvaluationResultDetailDto.java`, `L entity/dto/request/SaveEvaluationResultRequest.java`, `L common/CommonConstants.java` (khối `CurrencyCode`, `resolveCurrencyCode`), `L common/ReadMoneyUtils.java`, test spec `src/test/java/com/jvb/thanhdo/service/AppraisalResultServiceSaveCurrencyTest.java` (226 dòng, 9 test).

- **Điều phối theo loại** (`buildDetailFromEntities`, ~l.89-133): LAND / APARTMENT / CAR / **SHIP** mỗi loại một hàm `build*EvaluationDetailFromEntities`; loại khác trả DTO rỗng. Legacy **không** có `else = car`.
- **`buildShipEvaluationDetailFromEntities`** (l.572-622):
  - `roundingMoneyType = resolveRoundingMoneyType(override, reportId, group)`, `roundingBase = resolveRoundingBase(type)` (THOUSAND → 1.000, còn lại 1.000.000) — dùng chung với 3 loại kia.
  - `currencyCode = resolveCurrencyCode(SHIP, group.currencyCode)` (l.585-588); đặt vào `dto.currencyCode`.
  - `exchangeRate = 1`; chỉ khi `currencyCode != VND` **và** `group.exchangeRate != null` mới lấy `group.exchangeRate` (l.589-593). Nghĩa là VND luôn 1, USD rate null → 1.
  - `avgPriceGuide = synthetic?.avgPriceGuide`. Chỉ khi `avgPriceGuide != null` (l.599-613): `vndAmount = Math.multiplyExact(avg, rate)`; `ArithmeticException` → `IllegalArgumentException("Thành tiền quy đổi sang VNĐ quá lớn, vui lòng kiểm tra lại tỷ lệ chuyển đổi")` (l.604); `ship.amount = avg` (**không làm tròn**); `ship.vndAmount`; `ship.roundedAmount = BigDecimal.valueOf(vnd).divide(BigDecimal.valueOf(base), 0, HALF_UP).multiply(base).longValue()` (l.609-612, dùng BigDecimal vì double chỉ chính xác tới ~9e15).
  - `ship.exchangeRate` luôn được đặt (kể cả khi avg null; lúc đó `amount/vndAmount/roundedAmount` null).
  - `totalMoney = roundingMoney = ship.roundedAmount ?? 0` (VNĐ); `totalPriceWordAppraisal = ReadMoneyUtils.doc(totalMoney)` — **luôn đọc theo VNĐ**, kết thúc " đồng" (l.616-619).
  - `populateEvaluationResultCommonFields` (note từ kết quả đã lưu, `landValuationMethod`, `appraisalAssetGroupId`, `assetName`).
- **`saveEvaluationResult(SaveEvaluationResultRequest)`** (l.761-793): `SaveEvaluationResultRequest extends AppraisalResult` + `currencyCode @Pattern("VND|USD")`, `exchangeRate @Min(1) @Max(999999999)`, `currencyNote @Size(max=1000)`.
  - Chỉ khi nhóm tồn tại **và** `assetType == SHIP` (l.767):
    - `if (request.currencyCode != null)` (l.768-779): `code = resolveCurrencyCode(assetType, request.currencyCode)`; `group.currencyCode = code`; VND → `exchangeRate = 1`; USD và `request.exchangeRate != null` → gán; USD rate null → **giữ rate cũ**; `note = request.currencyNote == null ? null : trim`, rỗng → null, gán `group.currencyNote`.
    - Lưu nhóm (l.780). Lấy `synthetic` theo nhóm (l.783-784), gọi `buildShipEvaluationDetailFromEntities(synthetic, reportId, group, request.roundingMoneyType)` (l.785-786), rồi **ghi đè** `request.roundingMoneyType = detail.roundingMoneyType`, `request.roundingMoney = detail.roundingMoney`, `request.totalMoneyAppraisal = String.valueOf(detail.totalMoney)` (không phân cách nghìn), `request.totalPriceWordAppraisal = detail.totalPriceWordAppraisal` (l.787-790).
  - Nhóm khác: bỏ qua ba trường tiền tệ, lưu đúng tổng client gửi. Cuối cùng gọi `saveEvaluationResult((AppraisalResult) request)` (ghi đè mọi cột như bộ edit-tab đã port ở `AppraisalResult.overwrite`).
- **NPE `getAppraisalLandDescription()`** (legacy l.73-80 `getByReportAndGroup` dùng `Optional.ofNullable`; l.249-251, 339-341, 389-391 kiểm tra null trước khi `.getLandPriceJson()`): hành vi đúng là **tính kết quả cho nhóm không có mô tả đất** (tàu, ô tô) mà không NPE.
- **Test spec 9 case** (bảng trong `surveys/legacy-be.md` mục `AppraisalResultServiceImpl`; đọc nguyên file test để copy số liệu). Cách port: case 1-4 (build chi tiết) → test **thuần** trên `AppraisalResultCalculator`; case 5-9 (lưu) → test service trên `EvaluationResultService`.

### Đối chiếu code mới (origin/develop)

- `N service/result/AppraisalResultCalculator.java`: `isKnown` (l.84-87) chỉ 1/2/3; điều phối l.73-79 là `LAND … APARTMENT … else → car(...)` — **`else` ngầm hiểu CAR** (Review Focus #1). Loại lạ trả `EvaluationResultResponse.empty()` (l.55-57).
- `N service/result/ResultMath.roundTo(long, long)` dùng `Math.round(double)` — **không dùng cho tàu** (legacy dùng BigDecimal HALF_UP).
- `N model/dto/EvaluationResultResponse.java` (record + `@Builder`, 24 trường): chưa có tiền tệ/tàu. Test `NT controller/EvaluationResultControllerTest` đang ghim **đúng 24 khóa** (`LEGACY_KEYS`) — sẽ thành **28** (cố ý, xem Khác hệ thống cũ).
- `N service/EvaluationResultService.save` (l.38-48): chỉ `AppraisalResult.create/overwrite` với `EvaluationResultSaveRequest(5 trường)`; chưa có tiền tệ. Constructor `(scope, loader, results, clock)`.
- `N service/result/EvaluationResultLoader.inputs` lấy mô tả đất qua `landDescriptions.findByAppraisalAssetGroupId(...).map(...).orElse(List.of())` → **không NPE** với nhóm không có mô tả đất (cần test ghim).
- `N common/util/export/word/AppraisalResultService.buildDetailFromEntities` (l.50-73): `LAND` / `APARTMENT` / `CAR` rồi `return new EvaluationResultDetailDto()`. Chỗ `getAppraisalLandDescription()` ở file này (l.181-183, 271-273, 321-323) **đã** kiểm tra null; các chỗ ngoài phạm vi B06 (B08/B09 phải rà): `appraisal/asset/LandParcelBlock:30`, `SitePhotoImages:35`, `ApartmentDetailWriter:99,118`, `section/AdjustmentTableBlock:357`, `section/LandValuationNotes:73,364`, `AppraisalReportWordService:100`.
- Hai nơi dựng kết quả có công thức riêng: `AppraisalResultCalculator` (API + báo cáo VCB qua `EvaluationResultCalculator`) và `common/util/export/word/AppraisalResultService` (Word, DTO `EvaluationResultDetailDto` @Data, bản chép nguyên văn từ legacy). B06 sửa **cả hai** và dùng chung helper tính tiền để không lệch.

### Khác hệ thống cũ (ghi vào Javadoc)

1. **Tràn số → `BusinessException(ErrorCode.BAD_REQUEST, …)` (400)** với đúng message legacy, thay `IllegalArgumentException` (Khác #3).
2. **JSON** của kết quả tàu: khóa `ship` (record `ShipDetail`), theo khóa `car`/`CarDetail` đang có (Quyết định #1 của controller; không dùng `ship`). DTO Word `EvaluationResultDetailDto` cũng dùng `ship` (`getShip()`), nên hai nơi cùng tên.
3. `EvaluationResultResponse` thêm `currencyCode`, `exchangeRate`, `currencyNote` (mức nhóm); 24 → 28 khóa JSON. Với nhóm khác SHIP ba trường này là `null`.
4. `PUT` lưu **tính lại tổng ở server** cho SHIP; `totalMoneyAppraisal` ghi `String.valueOf(totalMoney)` đúng legacy (không dấu chấm nghìn), khác chuỗi `1.000.000` client gửi.
5. Báo cáo VCB (`EvaluationResultCalculator`) gọi chung calculator: nhóm SHIP lúc này **được tính** (trước đây loại 4 → EMPTY). Tràn số cũng ném `BusinessException` ở đó (rủi ro thấp: cần `avg × rate > 9,2e18`).

## Files

**Create**
- `N common/util/ShipValuationMath.java`
- `NT common/util/ShipValuationMathTest.java`
- `NT common/util/export/word/AppraisalResultServiceShipTest.java`

**Modify**
- `N model/dto/EvaluationResultResponse.java` (thêm `ShipDetail` + 4 trường)
- `N service/result/AppraisalResultCalculator.java` (nhánh SHIP riêng, `switch`, bỏ `else = car`)
- `N model/dto/EvaluationResultSaveRequest.java` (3 trường tiền tệ + `withTotals`)
- `N service/EvaluationResultService.java` (nhánh lưu SHIP)
- `N controller/EvaluationResultController.java` (chỉ Javadoc)
- `N model/dto/EvaluationResultDetailDto.java` (`currencyCode`, `ship`, `ShipEvaluationDetail`)
- `N common/util/export/word/AppraisalResultService.java` (nhánh SHIP)
- Test: `NT service/result/AppraisalResultCalculatorTest.java`, `NT service/result/EvaluationFixtures.java` (helper `shipGroup`), `NT service/result/EvaluationResultLoaderTest.java`, `NT service/EvaluationResultServiceTest.java`, `NT controller/EvaluationResultControllerTest.java`

**Không sửa** (để tránh xung đột đợt B2): `backend/CLAUDE.md`, `AppraisalGroupCompare`, `AppraisalAssetGroup`, các file Word ngoài `AppraisalResultService`. Dòng cần thêm vào bảng helper của `backend/CLAUDE.md` ghi ở cuối task để controller gộp.

## Interfaces

- **Consumes (B01):** `AppraisalAssetType.SHIP`, `CurrencyCodes.resolveCurrencyCode/VND/USD`, `AppraisalAssetGroup.getCurrencyCode/getExchangeRate/getCurrencyNote/applyCurrency`, `MoneyWords.read(long)` (đã có).
- **Produces:**
  - `public final class ShipValuationMath` (`N common/util`):
    - `static long effectiveRate(String resolvedCurrencyCode, Long storedRate)`: VND → 1; còn lại `storedRate ?? 1`.
    - `static long toVnd(long amount, long rate)`: `Math.multiplyExact`; tràn → `BusinessException(ErrorCode.BAD_REQUEST, "Thành tiền quy đổi sang VNĐ quá lớn, vui lòng kiểm tra lại tỷ lệ chuyển đổi")`.
    - `static long roundToBase(long vndAmount, long base)`: `BigDecimal` HALF_UP.
    - `public static final String OVERFLOW_MESSAGE`.
  - `EvaluationResultResponse`: thêm cuối record `ShipDetail ship, String currencyCode, Long exchangeRate, String currencyNote`; `record ShipDetail(Long amount, Long exchangeRate, Long vndAmount, Long roundedAmount)`.
  - `EvaluationResultSaveRequest(String totalMoneyAppraisal, Long roundingMoney, Integer roundingMoneyType, String totalPriceWordAppraisal, String note, @Pattern(regexp = "VND|USD", message = "Đơn vị tiền tệ không hợp lệ") String currencyCode, @Min(1) @Max(999_999_999) Long exchangeRate, @Size(max = 1000) String currencyNote)`; giữ **constructor 5 tham số** (ba trường mới = null) để mã cũ biên dịch; `withTotals(Integer type, Long roundingMoney, String total, String word)` trả bản mới.
  - `EvaluationResultService(ReportEditScope scope, EvaluationResultLoader loader, AppraisalResultRepository results, ControlCompareFactorSyntheticRepository synthetics, Clock clock)` — thêm `synthetics`.
  - `EvaluationResultDetailDto`: `String currencyCode`, `ShipEvaluationDetail ship`, `static class ShipEvaluationDetail { Long amount; Long exchangeRate; Long vndAmount; Long roundedAmount; }` (@Data).
  - Hợp đồng JSON: GET/PUT `…/groups/{groupId}/evaluation-result` đúng `00-index.md` mục "Kết quả thẩm định".

## Steps

### Phần A — Máy tính kết quả (pure) và JSON

- [ ] **A1. Đọc** `AppraisalResultServiceSaveCurrencyTest.java` nguyên file, đoạn `buildShipEvaluationDetailFromEntities` và `saveEvaluationResult(SaveEvaluationResultRequest)` (đã dẫn ở trên); đọc `N …/AppraisalResultCalculator.java`, `EvaluationResultResponse.java`, `EvaluationFixtures.java`; mở khối Interfaces của B01 để xác nhận tên getter/`applyCurrency`.
- [ ] **A2. Test thất bại — helper** `NT common/util/ShipValuationMathTest`:
  - `effectiveRate_VND_luôn_1_kể_cả_khi_đã_lưu_tỷ_giá`: `("VND", 25450L) → 1`.
  - `effectiveRate_USD_rate_null_dùng_1`: `("USD", null) → 1`; `("USD", 25450L) → 25450`.
  - `toVnd_nhân_đúng`: `(100123, 25450) → 2_548_130_350`.
  - `toVnd_tràn_số_ném_BusinessException_400_đúng_thông_điệp`: `(9_000_000_000_000L, 999_999_999L)` → `BusinessException`, `getErrorCode() == BAD_REQUEST`, message = `OVERFLOW_MESSAGE` = "Thành tiền quy đổi sang VNĐ quá lớn, vui lòng kiểm tra lại tỷ lệ chuyển đổi".
  - `roundToBase_nửa_làm_tròn_lên`: `(2_500_000, 1_000_000) → 3_000_000`; `(2_499_999, 1_000_000) → 2_000_000`; `(2_548_130_350, 1_000) → 2_548_130_000`; `(1_500, 1_000) → 2_000`.
- [ ] **A3. Test thất bại — calculator** (`AppraisalResultCalculatorTest`, thêm helper `EvaluationFixtures.shipGroup(String currencyCode, Long exchangeRate, String note)` bằng `ReflectionTestUtils.setField` các trường `currencyCode/exchangeRate/currencyNote`; nhóm `assetType = 4`). So **toàn bộ record** khi có thể. Port 4 case "chi tiết" của legacy:
  - `tàu_usd_giữ_nguyên_ngoại_tệ_và_làm_tròn_vnd` (= legacy #1): USD, rate 25450, avg 100123, override `RoundingMoneyType.MILLION` → `ship == new ShipDetail(100_123L, 25_450L, 2_548_130_350L, 2_548_000_000L)`; `roundingMoney == totalMoney == 2_548_000_000L`; `currencyCode == "USD"`; `exchangeRate == 25_450L`; `totalPriceWordAppraisal == MoneyWords.read(2_548_000_000L)` và `endsWith(" đồng")`.
  - `tàu_usd_loại_làm_tròn_chỉ_ảnh_hưởng_cột_vnd` (= #2): override THOUSAND → `ship.amount == 100_123L`, `roundedAmount == 2_548_130_000L`.
  - `tàu_usd_tỷ_giá_null_dùng_1` (= #3): rate null, avg 1_500_000, MILLION → `exchangeRate == 1`, `vndAmount == 1_500_000L`, `roundedAmount == 2_000_000L`.
  - `tàu_vnd_bỏ_qua_tỷ_giá_đã_lưu` (= #4): VND, rate lưu 25450, avg 1_250_000_000 → `exchangeRate == 1`, `vndAmount == roundedAmount == 1_250_000_000L`.
  - Test **chứng minh nhánh SHIP riêng** (Review Focus #1):
    - `tàu_không_ra_carDetail`: nhóm SHIP, avg 100_000 → `car() == null`, `apartment() == null`, `landOutside() == null`, `ship() != null`, `assetType() == 4`.
    - `ô_tô_không_ra_ship_và_không_có_tiền_tệ`: nhóm CAR → `ship() == null`, `currencyCode() == null`, `exchangeRate() == null`, `currencyNote() == null`, `car() != null`.
    - `loại_5_không_rơi_vào_nhánh_ô_tô`: `assetType = 5` → `EvaluationResultResponse.empty()` (xóa `else`, nên loại lạ vẫn rỗng).
  - Biên:
    - `tàu_chưa_có_đơn_giá_tổng_0_nhưng_giữ_tỷ_giá`: synthetic null, USD/25450 → `ship == new ShipDetail(null, 25_450L, null, null)`, `totalMoney == 0`, `totalPriceWordAppraisal == "Không đồng"`.
    - `tàu_ghi_chú_tiền_tệ_trả_về_nguyên_văn`: note "Tỷ giá VCB" → `currencyNote()` bằng đó.
    - `tàu_tràn_số_ném_BusinessException_400`: avg 9_000_000_000_000L, USD, rate 999_999_999 → `assertThatThrownBy` `BusinessException`, `ErrorCode.BAD_REQUEST`, đúng `OVERFLOW_MESSAGE`.
    - `tàu_làm_tròn_nửa_lên_theo_vnd`: VND, avg 2_500_000, MILLION → `roundedAmount == 3_000_000L`.
    - `tàu_lấy_loại_làm_tròn_đã_lưu_khi_không_override`: `stored` cùng báo cáo có `roundingMoneyType = 2` → `roundingMoneyType() == 2`.
- [ ] **A4. Test thất bại — loader/null mô tả đất** (`EvaluationResultLoaderTest`): `loadFull_nhóm_tàu_không_có_mô_tả_đất_không_NPE_và_ra_ship`: `groups.findById` trả nhóm SHIP USD/25450; `landDescriptions.findByAppraisalAssetGroupId` trả `Optional.empty()`; `synthetics` trả avg 100_000; `assets` trả tài sản đầu "Tàu A" → `ship.roundedAmount == 2_545_000_000L`, `assetName == "Tàu A"`, `car() == null`.
- [ ] **A5. Chạy** `./mvnw -q test -Dtest='ShipValuationMathTest,AppraisalResultCalculatorTest,EvaluationResultLoaderTest'` → **FAIL** (không biên dịch/ thiếu nhánh).
- [ ] **A6. Cài đặt:**
  - `ShipValuationMath` (Javadoc dẫn `AppraisalResultServiceImpl.buildShipEvaluationDetailFromEntities` l.589-613).
  - `EvaluationResultResponse`: thêm `ShipDetail` và 4 trường vào record (cuối danh sách), Javadoc nêu "Khác hệ thống cũ #1-#3". Grep `new EvaluationResultResponse(` trong `src` để sửa chỗ dùng constructor trực tiếp (nếu có).
  - `AppraisalResultCalculator`: `isKnown` thêm SHIP; thay `if/else if/else` bằng `switch (assetType)` có đủ 4 `case` và `default -> throw new IllegalStateException(...)` (không thể tới vì đã qua `isKnown`); thêm `ship(builder, group, avgPriceGuide, base)`:
    - `code = CurrencyCodes.resolveCurrencyCode(SHIP, group.getCurrencyCode())`, `rate = ShipValuationMath.effectiveRate(code, group.getExchangeRate())`.
    - `avg != null` → `vnd = toVnd(avg, rate)`, `rounded = roundToBase(vnd, base)`, `ShipDetail(avg, rate, vnd, rounded)`; `avg == null` → `ShipDetail(null, rate, null, null)`.
    - `builder.currencyCode(code).exchangeRate(rate).currencyNote(group.getCurrencyNote())`; trả `rounded ?? 0`.
    - Cập nhật Javadoc đầu lớp: nêu thêm "tàu: port `buildShipEvaluationDetailFromEntities`; Khác: tràn số → BusinessException".
- [ ] **A7. Chạy** lệnh A5 → **PASS**. Chạy thêm `./mvnw -q test -Dtest='EvaluationResultCalculatorTest,VcbReportAssemblerTest,VcbReportServiceTest'` → PASS (test cũ **không sửa**; sửa mới là dừng và báo).
- [ ] **A8. Commit** (trong `backend/`): `feat: kết quả thẩm định tàu thủy theo tiền tệ nhóm, tách nhánh SHIP khỏi ô tô`.

### Phần B — Lưu kết quả kèm tiền tệ

- [ ] **B1. Test thất bại — service** (`EvaluationResultServiceTest`; sửa `setUp` truyền thêm mock `ControlCompareFactorSyntheticRepository synthetics`; nhóm SHIP dựng bằng `AppraisalAssetGroup.create(REPORT_ID, 1, AssetGroupType.SINGLE, 4, NOW)` + `ReflectionTestUtils.setField(group, "id", GROUP_ID)`, USD/rate đặt qua `group.applyCurrency(...)`; `scope.requireGroup` trả nhóm đó; `results.findBy…` trả `Optional.empty()`; request chung: `totalMoneyAppraisal "999"`, `roundingMoney 999L`, `roundingMoneyType 1 (MILLION)`, `totalPriceWordAppraisal "client-word đồng"`). Port 5 case lưu của legacy:
  - `tàu_lưu_tính_lại_tổng_từ_tỷ_giá_mới` (= #5): synthetic avg 100_000; request USD, rate 25450, note `"  Tỷ giá VCB  "` → nhóm: `USD`, `25450`, note `"Tỷ giá VCB"`; `AppraisalResult` lưu: `roundingMoney == 2_545_000_000L`, `totalMoneyAppraisal == "2545000000"`, `totalPriceWordAppraisal` bằng `MoneyWords.read(2_545_000_000L)` và `endsWith(" đồng")`, `roundingMoneyType == 1`; **giá trị client 999 bị ghi đè**.
  - `tàu_lưu_vnd_ép_tỷ_giá_về_1` (= #6): nhóm đang USD/25450/note "cũ"; request `VND` + rate 25450 → `VND`, `1L`, note `null`.
  - `tàu_lưu_usd_không_gửi_tỷ_giá_giữ_tỷ_giá_cũ` (= #7): nhóm rate 24000; request USD, rate null → rate vẫn 24000.
  - `nhóm_không_phải_tàu_bỏ_qua_tiền_tệ_và_giữ_tổng_client` (= #8): nhóm LAND; request USD/25450/"x" → nhóm `VND`, `1L`, note `null`; `AppraisalResult` lưu `999L`, `"999"`, `"client-word đồng"`; `verifyNoInteractions(synthetics)`.
  - `tàu_không_gửi_trường_tiền_tệ_giữ_tiền_tệ_nhóm` (= #9): nhóm USD/25450, request chỉ 5 trường → nhóm không đổi; tổng **vẫn được tính lại** theo tiền tệ nhóm (avg 100_000 → `2_545_000_000L`).
  - Ca bổ sung:
    - `tàu_ghi_chú_trắng_thành_null`: note `"   "` → `null`.
    - `tàu_lưu_loại_làm_tròn_nghìn`: avg 100_123, USD/25450, request `roundingMoneyType = 2` → `roundingMoney == 2_548_130_000L`, `roundingMoneyType == 2`.
    - `tàu_tràn_số_400_và_không_lưu_kết_quả`: avg 9_000_000_000_000L, rate 999_999_999 → `BusinessException` BAD_REQUEST; `verify(results, never()).save(any())`.
    - `tàu_chưa_có_synthetic_tổng_0`: `synthetics` trả `Optional.empty()` → `roundingMoney == 0L`, `totalMoneyAppraisal == "0"`, bằng chữ `"Không đồng"`.
- [ ] **B2. Test thất bại — controller** (`EvaluationResultControllerTest`):
  - Sửa `LEGACY_KEYS` thành **28 khóa** (thêm `ship`, `currencyCode`, `exchangeRate`, `currencyNote`); đổi tên hằng/Javadoc cho đúng ý ("24 trường legacy + 4 trường tàu").
  - `put_nhận_trường_tiền_tệ_và_chuyển_xuống_service`: body có `"currencyCode":"USD","exchangeRate":25450,"currencyNote":"Tỷ giá VCB"` → `ArgumentCaptor` thấy 3 giá trị.
  - `put_không_có_trường_tiền_tệ_vẫn_hợp_lệ`: giữ test `put_lưu_và_trả_thông_báo` hiện có (body cũ) chạy xanh, các trường mới = `null` (chứng minh constructor 5 tham số không làm Jackson hỏng).
  - `put_tiền_tệ_không_hợp_lệ_trả_400` (parameterized hoặc vòng lặp): `currencyCode:"EUR"`, `currencyCode:"usd"`, `exchangeRate:0`, `exchangeRate:1000000000`, `currencyNote` 1001 ký tự → 400, `verifyNoInteractions(service)`. Biên hợp lệ: `exchangeRate:1` và `999999999` → 200.
- [ ] **B3. Chạy** `./mvnw -q test -Dtest='EvaluationResultServiceTest,EvaluationResultControllerTest'` → **FAIL**.
- [ ] **B4. Cài đặt:**
  - `EvaluationResultSaveRequest`: thêm 3 trường + constructor 5 tham số + `withTotals`.
  - `EvaluationResultService`: inject `ControlCompareFactorSyntheticRepository`. Trong `save`, sau `scope.requireReport/requireGroup`, nếu `group.getAssetType() == AppraisalAssetType.SHIP` thì:
    1. nếu `request.currencyCode() != null`: `code = CurrencyCodes.resolveCurrencyCode(SHIP, request.currencyCode())`; `group.applyCurrency(code, request.exchangeRate(), request.currencyNote())` (luật VND→1, USD rate null giữ rate cũ, trim note rỗng→null nằm trong `applyCurrency` của B01; B06 **không** lặp lại). `touch(now)` nếu entity có.
    2. `synthetic = synthetics.findByAppraisalAssetGroupId(group.getId()).orElse(null)`; `computed = AppraisalResultCalculator.calculate(group, null, synthetic, List.of(), List.of(), null, null, request.roundingMoneyType())` (SHIP chỉ dùng `synthetic` và `group`, nên không cần nạp dữ liệu khác).
    3. `request = request.withTotals(computed.roundingMoneyType(), computed.roundingMoney(), String.valueOf(computed.totalMoney()), computed.totalPriceWordAppraisal())`.
    - Nhóm khác: không đụng ba trường tiền tệ. Phần còn lại giữ nguyên. Không cần `groups.save(group)` vì nhóm là entity managed trong `@Transactional`.
  - Javadoc lớp và `EvaluationResultController`: ghi "port `saveEvaluationResult(SaveEvaluationResultRequest)` (l.761-793)" và các điểm Khác (#1, #3, #4).
- [ ] **B5. Chạy** lệnh B3 → **PASS**; chạy lại `./mvnw -q test -Dtest='EvaluationResultServiceTest,EvaluationResultControllerTest,AppraisalResultRepositoryTest'` (repository test cần MySQL theo `.env.example`; không đổi, chỉ để chắc `AppraisalResult` không hỏng).
- [ ] **B6. Commit:** `feat: lưu tiền tệ nhóm tàu và tính lại tổng ở server khi lưu kết quả thẩm định`.

### Phần C — Nhánh SHIP của `AppraisalResultService` (Word)

- [ ] **C1. Test thất bại** `NT common/util/export/word/AppraisalResultServiceShipTest` (tạo `new AppraisalResultService(mock(AppraisalResultRepository.class))`; gọi `buildDetailFromEntities(null, synthetic, List.of(), 1L, group, override)`; nhóm/ synthetic dựng bằng `BeanUtils.instantiateClass` + `ReflectionTestUtils` như `EvaluationFixtures`, `appraisalReportAssets` để null):
  - `tàu_usd_có_ship_và_không_có_car`: USD/25450, avg 100123, MILLION → `getAssetType()==4`, `getCurrencyCode()=="USD"`, `getShip()` = `(100_123, 25_450, 2_548_130_350, 2_548_000_000)`, `getCar()==null`, `getTotalMoney()==2_548_000_000L`, `getRoundingMoney()` cùng giá trị, bằng chữ kết thúc " đồng".
  - `tàu_vnd_thousand`: VND, rate lưu 25450, avg 1_250_000_500, THOUSAND → rate 1, `roundedAmount == 1_250_001_000L` (HALF_UP trên 1_250_000.5).
  - `tàu_usd_rate_null_dùng_1`, `tàu_chưa_có_synthetic_total_0`.
  - `tàu_tràn_số_ném_BusinessException_400`.
  - `loại_5_trả_dto_rỗng_không_phải_ô_tô`: `getCar()==null`, `getAssetType()==null`.
  - `tàu_không_có_mô_tả_đất_không_NPE` (nhóm có `appraisalLandDescription == null` và `appraisalReportAssets == null`).
  - `ô_tô_vẫn_ra_car` (hồi quy, đảm bảo không phá exporter cũ).
- [ ] **C2. Chạy** `./mvnw -q test -Dtest=AppraisalResultServiceShipTest` → **FAIL**.
- [ ] **C3. Cài đặt:** `EvaluationResultDetailDto` thêm `currencyCode`, `ship`, `ShipEvaluationDetail` (kèm Javadoc). `buildDetailFromEntities`: thêm nhánh `SHIP` gọi `buildShipEvaluationDetailFromEntities(synthetic, reportId, group, override)`, port từ legacy l.572-622 nhưng dùng `ShipValuationMath` + `CurrencyCodes` + `MoneyWords`/`ReadMoneyUtils` tương ứng file hiện tại (giữ đúng thứ tự đặt trường của legacy; gọi `populateEvaluationResultCommonFields`). Không đụng `AppraisalReportWordExporter` (golden).
- [ ] **C4. Chạy** `./mvnw -q test -Dtest='AppraisalResultServiceShipTest,AssetTypeWritersTest'` rồi `./mvnw -q test -Dtest='*Word*'` → PASS (parity test mặc định tắt, không ảnh hưởng).
- [ ] **C5. Commit:** `feat: nhánh tàu thủy trong AppraisalResultService phục vụ xuất Word`.

### Phần D — Rà soát cuối task

- [ ] **D1.** `git -C backend grep -n "else" -- src/main/java/com/thanhdo/tdprojectbe/service/result/AppraisalResultCalculator.java` — không còn nhánh `else` ngầm hiểu CAR; `git -C backend grep -n "AppraisalAssetType.CAR" -- src/main/java/com/thanhdo/tdprojectbe/common/util/export/word/AppraisalResultService.java` — mỗi nhánh dùng so sánh tường minh.
- [ ] **D2.** `cd backend && ./mvnw -q test -Dtest='*Result*,*Vcb*,ShipValuationMathTest'` → PASS.
- [ ] **D3.** Báo cáo cuối task: danh sách file, kết quả test, **dòng cần thêm vào bảng helper của `backend/CLAUDE.md`** (controller gộp để tránh xung đột đợt): `| Quy đổi tiền tàu thủy ngoại tệ → VNĐ, làm tròn HALF_UP theo bội số | ShipValuationMath (common/util) | Tự nhân/làm tròn bằng double |`.

---

# Task B07 — Vòng đời nhóm tàu (tạo, đổi loại, xóa, khôi phục)

## Hành vi hệ thống cũ

Nguồn: `L service/impl/AppraisalReportServiceImpl.java` @ `82193fe5`, `L service/impl/AppraisalShipDescriptionServiceImpl.java` (`createShipDescriptions`, `createOrResetShipDescription`), `L service/impl/AppraisalGroupCompareServiceImpl.java` (`createGroupCompares`), test spec `src/test/java/com/jvb/thanhdo/service/impl/AppraisalReportServiceImplUpdateBasicInfoTest.java` (TC-10b l.491-518, TC-10c l.520-564, TC-10d l.566-598).

- **Tạo nhóm** (`createShipDescriptions`): mỗi nhóm SHIP một bản ghi mô tả tàu, `conditionDetails` = 18 hạng mục mặc định (`conditionDescription` null), `qualityEvaluationDetails` null. Được gọi sau mô tả xe, trước compare, ở cả tạo báo cáo lẫn `initDefaultEntitiesForNewGroup`. `createGroupCompares`: nhóm SHIP thì dòng `type = 0` gán `address = joinAssetAddresses(tài sản chưa xóa)` ("\n", trim, bỏ dòng rỗng, rỗng hết → null).
- **Yếu tố điều chỉnh mặc định** (`resolveDefaultControlCompareFactorLabels`, l.1877-1888): APARTMENT, CAR, **SHIP** (`DEFAULT_CONTROL_COMPARE_FACTOR_LABELS_SHIP`: Năm đóng, Nơi đóng, Ngoại quan tổng thể, Khả năng vận hành, Trọng tải toàn phần, Chất lượng còn lại, Công suất (CV), Cấp phương tiện), **còn lại = nhãn đất** (không kiểm tra loại lạ). `getDefaultUnitOfMeasure` (l.1890-1904) không có đơn vị cho nhãn tàu.
- **`handleAssetTypeChange`** (l.371-405), thứ tự: 1 đất, 2 xe, **2b `resetOrCreateShipDescription`**, 3 `resetGroupCompares`, 4 synthetic, 5 factor, 6 result, 7 CTXD (khi loại mới ≠ đất), 8 `resetGroupFields`. Loại không đổi → thoát sớm.
- **`resetOrCreateShipDescription`** (l.594-614): `oldIsShip && !newIsShip` → nếu có bản ghi và `deleteFlag == 0` thì `deleteFlag = 1`, lưu, **xóa ảnh**; `!oldIsShip && newIsShip` → nếu có bản ghi (kể cả đã xóa mềm) thì **xóa ảnh** của nó, rồi `createOrResetShipDescription(report, group)`: **giữ id và `created`**, ghi đè mọi cột về null/mặc định, `deleteFlag = 0` (tránh vi phạm unique `appraisal_asset_group_id`); chưa có bản ghi thì tạo mới như `createShipDescriptions`.
- **`resetGroupCompares`** (l.639-735): reset mọi cột của **bốn** dòng còn hiệu lực, trong đó **9 trường tàu về null** (l.675-683: `shipBuildYear … shipForSalePriceUsd`), dòng type 0 gán lại `address = joinAssetAddresses(tài sản)`, xóa ảnh so sánh bất đồng bộ.
- **`resetGroupFields`** (l.869-886): luôn chạy ở **mọi** lần đổi loại: `comparativeAnalysis/ghi chú → null`, `totalValue/roundedValue → 0`, **`currencyCode = VND`, `exchangeRate = 1`, `currencyNote = null`** (l.876-878; "chỉ Tàu mới được dùng ngoại tệ"), loại mới ≠ đất → xóa hai ghi chú CTXD, `updated = now`.
- **Khôi phục nhóm** (`restoreDeletedGroupAndDependencies`, mục 3.2b ~l.1257-1269): `shipDesc = group.getAppraisalShipDescription()`; `groupIsShip = group.assetType == SHIP`; có bản ghi: chỉ khi `groupIsShip` **và** `deleteFlag == 1` thì `deleteFlag = 0` ("mô tả tàu bị xóa do đổi loại thì giữ nguyên trạng thái xóa"); chưa có bản ghi và nhóm là SHIP → `createShipDescriptions`.
- **Xóa nhóm** (nhóm không còn tài sản, ~l.975-1003): legacy **không** xử lý mô tả tàu → lỗ hổng, bản mới sửa (**Khác hệ thống cũ #1**).
- **Pháp lý**: `LegalDocumentAssetTypeValidator.isAllowedAssetType` chấp nhận SHIP.
- **Test spec** (đọc nguyên file):
  - **TC-10b** Ô tô → Tàu: mô tả xe `deleteFlag = 1`, gọi `createOrResetShipDescription(report, group)`, **không** lưu mô tả đất.
  - **TC-10c** Tàu (USD/25000/"Tỷ giá VCB") → Đất: mô tả tàu `deleteFlag = 1` và được lưu, **không** gọi `createOrReset`; `compare.shipClass`, `shipDeadweight` thành null; nhóm về `VND`/`1`/`null`; lưu mô tả đất.
  - **TC-10d** Đất → Tàu khi đã có mô tả tàu bị xóa mềm: mô tả đất `deleteFlag = 1`; gọi `createOrResetShipDescription` để dùng lại bản ghi.

### Đối chiếu code mới (origin/develop)

- `N service/ReportGroupDetailsWriter.writeFor` (l.57-68): chỉ LAND/APARTMENT/CAR tạo mô tả; `groupCompareRepository.saveAll(... forGroup ...)` không gán địa chỉ; `defaultFactors` theo `ControlFactorDefaults.labelsFor(assetType)`; `labelIds()` tra theo `ControlFactorDefaults.allLabels()`. Constructor 5 tham số.
- `N service/ControlFactorDefaults` (l.34-40): `switch` có `APARTMENT`, `CAR`, `default -> LAND`; `allLabels()` ghép 3 danh sách; `NT service/ControlFactorDefaultsTest.landAndUnknownTypesShareTheLandLabels` ghim `labelsFor(9) == labelsFor(1)`.
- `N service/GroupAssetTypeReset.reset` (l.59-77): thứ tự `resetLandDescription → resetVehicleDescription → resetGroupCompares (compare.reset(address) + xóa ảnh so sánh) → resetSynthetic → replaceDefaultFactors → resetResult → softDeleteConstructions (≠ LAND) → group.resetForAssetType`. **Chưa** có ship, reset 9 trường tàu, reset tiền tệ. Constructor 12 tham số (`NT GroupAssetTypeResetTest.setUp` gọi trực tiếp).
- `compare.reset(address)` (`AppraisalGroupCompare` l.496-570) chỉ biết `car_*`; B01 đã nối `clearShipFields()` vào `reset(String)` và `resetCurrency()` vào `AppraisalAssetGroup.resetForAssetType`, nên B07 **không cần gọi thêm** (gọi thêm vô hại); test ở Phần C chứng minh kết quả.
- `N service/ReportGroupResolver`: `restore(...)` (l.135-150) gọi `restoreLandDescription`, `restoreVehicleDescription`, `restoreCompares`, kết quả, CTXD; `syncTypesAndCleanup` → `softDeleteChildren` (l.105-113) xóa mềm compare, land, vehicle, result của **nhóm rỗng** — đây là **chỗ xóa nhóm** duy nhất trên code mới (grep `markDeleted`/`softDelete` trong `service/` không thấy luồng xóa nhóm nào khác). Constructor 8 tham số (`NT ReportGroupResolverTest.setUp`).
- `N service/ReportLegalDocumentValidator.isKnownType` (l.72-75): chỉ 1/2/3; `NT ReportLegalDocumentValidatorTest.onlyLandApartmentAndCarAreValidAssetTypes` đang dùng **4 làm loại không hợp lệ** (phải đổi sang 5).
- (Đã do B01 xử lý, Quyết định #4; chỉ để tham khảo) `N service/ReportContentMerge.valuationBases` (l.69) và `N service/AppraisalReportFormService.formData` (l.65-67): cận trên `ValuationBasis.ASSET_TYPE_CAR`; Javadoc ReportContentMerge l.55-56 ghi "loại ngoài 0..3 bị bỏ".
- `N service/AppraisalReportCreateService.saveGroups` lưu nhóm + `assetRepository.saveAll` **trước** `detailsWriter.write` ⇒ lúc ghi bản ghi con của nhóm mới, tài sản đã có trong giao dịch (đọc qua `assetRepository.findByAppraisalAssetGroupIdAndDeleteFlagOrderByIdAsc`). Còn ở `ReportGroupResolver.restoreOrCreate` (sửa cơ bản) nhóm mới được tạo **trước khi** tài sản chuyển sang ⇒ địa chỉ lúc đó là null (giống legacy, xem "Vấn đề").

### Khác hệ thống cũ (ghi vào Javadoc)

1. **Xóa nhóm rỗng → chỉ xóa mềm mô tả tàu, KHÔNG xóa ảnh** (Khác #1, Quyết định #8: giống đất/xe, để khôi phục nhóm lấy lại được cả ảnh). Khôi phục nhóm hồi sinh mô tả cùng ảnh. Ảnh chỉ bị xóa khi **đổi loại** (rời SHIP hoặc vào SHIP với bản ghi cũ), như legacy.
2. **Ảnh xóa trong giao dịch** qua service ảnh (S3 xóa sau commit), không `CompletableFuture` như legacy (đã là quy ước của `GroupAssetTypeReset`).
3. **Reset tiền tệ và 9 trường tàu đi qua `resetForAssetType`/`compare.reset` của B01** (B07 không gọi thêm; test Phần C khóa kết quả theo thứ tự legacy).
4. **`ControlFactorDefaults.default` giữ nhãn đất** như legacy (quyết định, xem dưới). SHIP thêm `case` riêng.

## Files

**Create:** không có file production mới.

**Modify**
- `N service/ControlFactorDefaults.java`
- `N service/ReportGroupDetailsWriter.java`
- `N model/entity/AppraisalGroupCompare.java` (chỉ **thêm** overload `forGroup(Long reportId, Long groupId, int type, String address)`)
- `N service/GroupAssetTypeReset.java`
- `N service/ReportGroupResolver.java`
- `N service/ReportLegalDocumentValidator.java`
- **Không** sửa `N common/constant/ControlCompareFactorLabels.java` (Quyết định #3: B01 đã tạo `BUILD_YEAR`, `BUILD_PLACE`, `OVERALL_APPEARANCE`, `OPERABILITY`, `DEADWEIGHT`, `POWER_HP`, `VESSEL_CLASS`; B07 chỉ dùng)
- Test: `NT service/ControlFactorDefaultsTest`, `ReportGroupDetailsWriterTest`, `GroupAssetTypeResetTest`, `ReportGroupResolverTest`, `ReportLegalDocumentValidatorTest`, và (nếu có) `AppraisalGroupCompareFormTest` cho factory mới.

## Interfaces

- **Consumes:** B01 (xem "Giả định về B01"), B02: `AppraisalImageService.deleteAllOfShipDescription(long shipDescriptionId)` (cùng dạng `deleteAllOfVehicleDescription(long)` l.155), `AppraisalShipDescriptionRepository`.
- **Produces (chữ ký chính xác):**
  - `ControlFactorDefaults.labelsFor(int)`: thêm `case AppraisalAssetType.SHIP -> SHIP`; `allLabels()` ghép 4 danh sách (`distinct`).
  - `ReportGroupDetailsWriter(AppraisalLandDescriptionRepository, AppraisalVehicleDescriptionRepository, AppraisalShipDescriptionRepository, AppraisalGroupCompareRepository, AppraisalReportAssetRepository, ControlCompareFactorRepository, ControlCompareFactorLabelRepository)`.
  - `AppraisalGroupCompare.forGroup(Long reportId, Long groupId, int type, String address)`: như `forGroup` 3 tham số, và với `type == TYPE_APPRAISED` gán `address`; `type` khác để null.
  - `GroupAssetTypeReset(land, vehicle, ship, groupCompare, asset, synthetic, factor, result, construction, detailsWriter, imageService, comparisonImageService, constructionService)` — thêm `AppraisalShipDescriptionRepository` **ngay sau** `vehicleDescriptionRepository`.
  - `ReportGroupResolver(groupRepository, assetRepository, landRepository, vehicleRepository, shipRepository, compareRepository, resultRepository, constructionRepository, detailsWriter)` — thêm `AppraisalShipDescriptionRepository` sau `vehicleRepository`; **không** thêm `AppraisalImageService` (xóa nhóm không xóa ảnh).
  - `ReportLegalDocumentValidator.isKnownType(int)`: 1, 2, 3 hoặc 4.

## Steps

### Phần A — `ControlFactorDefaults` (8 nhãn tàu)

- [ ] **A1. Kiểm tra hằng nhãn của B01:** `git -C backend grep -n "BUILD_YEAR\|BUILD_PLACE\|OVERALL_APPEARANCE\|OPERABILITY\|DEADWEIGHT\|POWER_HP\|VESSEL_CLASS\|REMAINING_QUALITY" -- src/main/java/com/thanhdo/tdprojectbe/common/constant/ControlCompareFactorLabels.java` phải đủ 8 hằng. Thiếu thì **dừng, báo controller** (B01 chưa xong), không tự thêm. Danh sách `SHIP` dùng đúng thứ tự: `BUILD_YEAR, BUILD_PLACE, OVERALL_APPEARANCE, OPERABILITY, DEADWEIGHT, REMAINING_QUALITY, POWER_HP, VESSEL_CLASS`. Danh sách CAR hiện có đang dùng literal "Khả năng vận hành": thay bằng `OPERABILITY` (cùng giá trị) nếu hằng đã có.
- [ ] **A2. Test thất bại** (`ControlFactorDefaultsTest`):
  - `shipHasEightLabelsInLegacyOrder`: `labelsFor(4)` `containsExactly("Năm đóng", "Nơi đóng", "Ngoại quan tổng thể", "Khả năng vận hành", "Trọng tải toàn phần", "Chất lượng còn lại", "Công suất (CV)", "Cấp phương tiện")`.
  - `shipLabelsHaveNoUnit`: mỗi nhãn tàu `unitOf(label)` là `null`.
  - `unknownTypesStillFallBackToLandLabelsButShipDoesNot`: `labelsFor(9)` = `labelsFor(1)` (**giữ** test cũ, quyết định "default vẫn là đất" như legacy) và `labelsFor(4)` khác `labelsFor(1)`.
  - `allLabelsIncludesShipLabelsWithoutDuplicates`: `allLabels()` `doesNotHaveDuplicates()` và chứa "Năm đóng", "Cấp phương tiện"; "Khả năng vận hành" và "Chất lượng còn lại" xuất hiện **một lần** dù có ở CAR và SHIP.
- [ ] **A3. Chạy** `./mvnw -q test -Dtest=ControlFactorDefaultsTest` → FAIL. **Cài đặt:** thêm `private static final List<String> SHIP`; `switch` thêm `case SHIP`, giữ `default -> LAND`; `allLabels()` ghép `LAND, APARTMENT, CAR, SHIP`; sửa Javadoc lớp (bốn bộ nhãn; "loại lạ theo nhãn đất như legacy"). Chạy → PASS.
- [ ] **A4. Commit:** `feat: yếu tố điều chỉnh mặc định của tàu thủy`.

### Phần B — `ReportGroupDetailsWriter` (mô tả tàu + địa chỉ dòng so sánh)

- [ ] **B1. Test thất bại** (`ReportGroupDetailsWriterTest`; cập nhật `setUp` theo constructor mới, thêm mock `AppraisalShipDescriptionRepository ships`, `AppraisalReportAssetRepository assets`, mặc định `assets.findByAppraisalAssetGroupIdAndDeleteFlagOrderByIdAsc(GROUP_ID, 0)` trả `List.of()` bằng `lenient()`):
  - `writeFor_tau_ghi_mo_ta_tau_voi_18_tinh_trang_mac_dinh`: nhóm SHIP → `ArgumentCaptor<AppraisalShipDescription>` của `ships.save`: `getAppraisalReportId()==REPORT_ID`, `getAppraisalAssetGroupId()==GROUP_ID`, `getDeleteFlag()==0`, `getConditionDetails()` có **18** phần tử, phần tử đầu `itemName == "Thân vỏ"`, cuối `"Hệ thống chiếu sáng"`, mọi `conditionDescription == null`; `getQualityEvaluationDetails()` rỗng/null; `verify(landDescriptions, never()).save(any())`, `verify(vehicleDescriptions, never()).save(any())`.
  - `writeFor_tau_gan_dia_chi_cac_tai_san_cho_dong_so_sanh_loai_0`: `assets` trả hai tài sản chưa xóa tên `" Tàu A "`, `"Tàu B"` (mock `getAsset/getDeleteFlag`) → 4 dòng, `type 0` có `getAddress()=="Tàu A\nTàu B"`, các dòng type 1..3 `getAddress()==null`.
  - `writeFor_tau_khong_co_tai_san_thi_dia_chi_null`.
  - `writeFor_tau_ghi_8_yeu_to_mac_dinh`: `savedFactors()` có `ControlFactorDefaults.labelsFor(4).size()` = 8 phần tử.
  - `writeFor_dat_va_o_to_khong_ghi_mo_ta_tau_va_khong_gan_dia_chi`: `verify(ships, never()).save(any())`, `type 0` `getAddress()==null`; `verifyNoInteractions(assets)`.
  - `writeDefaultFactors_tau_chi_ghi_8_yeu_to`: `verifyNoInteractions(ships, assets)`.
- [ ] **B2. Chạy** `./mvnw -q test -Dtest=ReportGroupDetailsWriterTest` → FAIL.
- [ ] **B3. Cài đặt:**
  - `AppraisalGroupCompare.forGroup(…, String address)` (overload; Javadoc: dùng cho nhóm tàu; test entity ở `AppraisalGroupCompareFormTest` hoặc file test entity tương đương: `forGroup_voi_dia_chi_chi_gan_cho_dong_tham_dinh`).
  - `ReportGroupDetailsWriter.writeFor`: thêm nhánh `assetType == AppraisalAssetType.SHIP` → `shipDescriptionRepository.save(AppraisalShipDescription.empty(reportId, group.getId(), now))`; địa chỉ chỉ tính khi SHIP: `GroupAssetTypeReset.joinAssetAddresses(assetRepository.findByAppraisalAssetGroupIdAndDeleteFlagOrderByIdAsc(group.getId(), 0))` (đã package-private cùng gói), còn loại khác `address = null`; dùng `forGroup(reportId, groupId, type, address)`.
  - `AppraisalShipDescription.empty` của B01 đã gán 18 tình trạng mặc định. Nếu không có 18 mục thì **dừng, báo controller** (B01 sai hợp đồng); B07 không sửa file entity ship.
  - Javadoc lớp: bổ sung "tàu thủy: `createShipDescriptions` + địa chỉ `createGroupCompares`".
- [ ] **B4. Chạy** → PASS. Chạy kèm `./mvnw -q test -Dtest='AppraisalReportCreateServiceTest,AppraisalGroupCompare*'` (đúng tên lớp test tồn tại; dùng `ls` để xác nhận) → PASS.
- [ ] **B5. Commit:** `feat: tạo mô tả tàu, 18 tình trạng mặc định và địa chỉ so sánh khi tạo nhóm tàu`.

### Phần C — `GroupAssetTypeReset` (đổi loại)

- [ ] **C1. Test thất bại** (`GroupAssetTypeResetTest`; cập nhật `setUp`: thêm `@Mock AppraisalShipDescriptionRepository ships`, constructor mới, `lenient().when(ships.findByAppraisalAssetGroupId(GROUP_ID)).thenReturn(Optional.empty())`; helper `ship(long id, int deleteFlag)` = `AppraisalShipDescription.empty(REPORT_ID, GROUP_ID, NOW)` + `ReflectionTestUtils.setField(ship, "id", id)` (+ `"deleteFlag"`) + gán `shipName = "Tàu cũ"`; helper `compareWithShipFields(long id, int type)` đặt đủ 9 trường tàu bằng `ReflectionTestUtils`). Port TC-10b/c/d và mở rộng:
  - `o_to_sang_tau_xoa_mem_mo_ta_xe_tao_mo_ta_tau_khong_luu_mo_ta_dat` (**TC-10b**): xe `getDeleteFlag()==1`, `verify(images).deleteAllOfVehicleDescription(51L)`, `verify(ships).save(any(AppraisalShipDescription.class))` (chưa có bản ghi), `verify(lands, never()).save(any())`.
  - `tau_sang_dat_xoa_mem_mo_ta_tau_va_anh_xoa_truong_tau_ve_vnd` (**TC-10c**): nhóm SHIP với `group.applyCurrency("USD", 25_000L, "Tỷ giá VCB")`; mô tả tàu active id 80; compare type 1 có `shipClass="VR-SI"`, `shipDeadweight=1200.50` → sau `reset(..., SHIP, LAND, ...)`: `ship.getDeleteFlag()==1`, `verify(images).deleteAllOfShipDescription(80L)`, `verify(ships, never()).save(any())` (entity managed, không gọi save; không tạo mới), `compare.getShipClass()==null`, `compare.getShipDeadweight()==null`, `group.getCurrencyCode()=="VND"`, `group.getExchangeRate()==1L`, `group.getCurrencyNote()==null`, `verify(lands).save(any(AppraisalLandDescription.class))`.
  - `dat_sang_tau_dung_lai_ban_ghi_da_xoa_mem_giu_id_va_xoa_anh` (**TC-10d**): đất active id 31 + tàu cũ `deleteFlag=1` id 80 có `shipName="Tàu cũ"` → `land.getDeleteFlag()==1` và `verify(images).deleteAllOfLandDescription(31L)`; `ship.getId()==80L`, `ship.getDeleteFlag()==0`, `ship.getShipName()==null`, `ship.getConditionDetails()` **18 mục mặc định**, `verify(images).deleteAllOfShipDescription(80L)` (xóa ảnh **trước** `resetToDefault`: dùng `InOrder`), `verify(ships, never()).save(any())` (không tạo bản ghi mới ⇒ không vi phạm unique).
  - `dat_sang_tau_chua_co_mo_ta_tau_thi_tao_moi_khong_xoa_anh`: `verify(ships).save(any())`, `verify(images, never()).deleteAllOfShipDescription(anyLong())`.
  - `roi_tau_khi_mo_ta_tau_da_xoa_mem_thi_khong_xoa_anh` (đối xứng `roi_o_to_khi_mo_ta_xe_da_xoa_mem_thi_khong_lam_gi`).
  - `tau_sang_o_to_xoa_mem_mo_ta_tau_va_tao_mo_ta_xe`: `ship` deleteFlag 1, `verify(vehicles).save(any())`.
  - `o_to_sang_dat_khong_dung_den_mo_ta_tau`: `verifyNoInteractions(ships)` (hoặc `ships` chỉ bị gọi `find…` rồi không `save`) — đảm bảo ca cũ không bị phá.
  - `vao_tau_xoa_trang_9_truong_tau_tren_ca_4_dong_so_sanh`: 4 compare `type 0..3` đều có 9 trường tàu (kể cả `shipForSalePriceUsd`) → sau reset cả 9 trường ở cả 4 dòng đều `null`; dòng type 0 `address` = ghép tài sản.
  - `tien_te_ve_vnd_khi_doi_bat_ky_loai`: nhóm LAND có `applyCurrency("USD", 25_000L, "x")` (dữ liệu bẩn) → đổi sang APARTMENT → `VND/1/null` (đúng `resetGroupFields` luôn chạy); `khong_doi_loai_thi_khong_doi_tien_te` (cùng loại → `reset` thoát sớm, tiền tệ giữ nguyên, `verifyNoInteractions` mở rộng thêm `ships`).
  - `dat_sang_tau_ghi_yeu_to_mac_dinh_moi_va_xoa_mem_cong_trinh`: giữ mẫu `yeu_to_cu_bi_xoa_mem…` cho SHIP (`order.verify(writer).writeDefaultFactors(group, NOW)`; công trình bị `softDelete` vì loại mới ≠ đất; `constructionNote` bị xóa).
  - Cập nhật test hiện có `khong_doi_loai_thi_khong_lam_gi` để `verifyNoInteractions(..., ships)`.
- [ ] **C2. Chạy** `./mvnw -q test -Dtest=GroupAssetTypeResetTest` → FAIL.
- [ ] **C3. Cài đặt** `GroupAssetTypeReset`:
  - Thêm field/constructor `AppraisalShipDescriptionRepository shipDescriptionRepository` (đúng vị trí đã nêu ở Interfaces).
  - Sau `resetVehicleDescription`, thêm `resetShipDescription(reportId, group, oldType, newType, now)`:
    - `wasShip && !isShip`: có bản ghi và `getDeleteFlag() == NOT_DELETED` → `ship.softDelete()` + `imageService.deleteAllOfShipDescription(ship.getId())` (đã xóa mềm thì bỏ qua, như xe).
    - `!wasShip && isShip`: có bản ghi → `imageService.deleteAllOfShipDescription(id)` **rồi** `ship.resetToDefault()` + `ship.revive()`; chưa có → `shipDescriptionRepository.save(AppraisalShipDescription.empty(reportId, group.getId(), now))`.
  - `resetGroupCompares` giữ nguyên lời gọi `compare.reset(address)` (B01 đã gọi `clearShipFields()` bên trong); `reset` giữ nguyên lời gọi `group.resetForAssetType(newType, now)` (B01 đã gọi `resetCurrency()` bên trong). Chỉ bổ sung Javadoc nêu legacy l.675-683 và l.876-878 ("luôn chạy ở mọi lần đổi loại; chỉ tàu được dùng ngoại tệ").
  - Cập nhật Javadoc lớp (thứ tự: mô tả đất, mô tả xe, **mô tả tàu**, bốn dòng so sánh…, trường nhóm **kèm tiền tệ**; loại tài sản "Đất, Căn hộ, Ô tô, Tàu thủy").
- [ ] **C4. Chạy** `./mvnw -q test -Dtest=GroupAssetTypeResetTest` → PASS. Chạy `./mvnw -q test -Dtest='AppraisalReportBasicEditServiceTest,AppraisalReportBasicEditIntegrationTest'` (test integration cần MySQL; nếu không chạy được ghi rõ) → PASS, không sửa test cũ ngoài constructor.
- [ ] **C5. Commit:** `feat: đổi loại sang/từ tàu thủy dọn mô tả tàu, trường tàu ở so sánh và tiền tệ nhóm`.

### Phần D — `ReportGroupResolver` (khôi phục và xóa nhóm)

- [ ] **D1. Test thất bại** (`ReportGroupResolverTest`; cập nhật `setUp`: thêm `@Mock AppraisalShipDescriptionRepository ships`, constructor mới; `lenient().when(ships.findByAppraisalAssetGroupId(GROUP_ID)).thenReturn(Optional.empty())`; helper `group(...)` có sẵn):
  - Khôi phục (theo mẫu `khoi_phuc_loai_o_to_hoi_sinh_mo_ta_xe…`):
    - `khoi_phuc_loai_tau_hoi_sinh_mo_ta_tau_da_xoa_mem`: `ship` mock `getDeleteFlag()==1` → `verify(ship).revive()`; `verifyNoInteractions(constructions)`; `verify(lands, never()).save(any())`; `verify(ships, never()).save(any())`.
    - `khoi_phuc_loai_tau_chua_co_mo_ta_tau_thi_tao_moi_voi_18_tinh_trang`: `ships.save` captor → 18 mục.
    - `khoi_phuc_nhom_dat_khong_hoi_sinh_mo_ta_tau_bi_xoa_do_doi_loai`: nhóm LAND, `ships` có bản ghi `getDeleteFlag()==1` → `verify(ship, never()).revive()`, `verify(ships, never()).save(any())` (đúng legacy: chỉ khôi phục khi nhóm còn là tàu).
    - `khoi_phuc_loai_tau_mo_ta_tau_dang_hoat_dong_thi_giu_nguyen`: `getDeleteFlag()==0` → `verify(ship, never()).revive()`.
  - Xóa nhóm (**Khác hệ thống cũ #1**):
    - `cleanup_nhom_tau_rong_xoa_mem_mo_ta_tau_giu_anh`: nhóm SHIP không còn tài sản; `ships` trả bản ghi mock `getDeleteFlag()==0` → `verify(ship).softDelete()`; **không** có `AppraisalImageService` trong resolver nên ảnh không bị đụng (Quyết định #8); kèm xác nhận compare/land/vehicle/result vẫn bị xóa mềm như ca cũ.
    - `cleanup_nhom_tau_rong_mo_ta_tau_da_xoa_mem_thi_khong_xoa_mem_lai`: `getDeleteFlag()==1` → `verify(ship, never()).softDelete()`.
    - `cleanup_nhom_dat_rong_khong_dung_den_mo_ta_tau`: `ships` trống → `verify(ships, never()).save(any())`.
    - Giữ nguyên `cleanup_dong_bo_kieu_theo_so_tai_san` (không tương tác ship/ảnh khi nhóm còn tài sản).
- [ ] **D2. Chạy** `./mvnw -q test -Dtest=ReportGroupResolverTest` → FAIL.
- [ ] **D3. Cài đặt** `ReportGroupResolver`:
  - Thêm `shipRepository` (constructor theo Interfaces).
  - `restore(...)`: gọi `restoreShipDescription(reportId, group, now)` ngay sau `restoreVehicleDescription`:
    - có bản ghi: `if (group.getAssetType() == SHIP && deleteFlag == DELETED) ship.revive()`;
    - chưa có bản ghi và nhóm là SHIP → `shipRepository.save(AppraisalShipDescription.empty(reportId, group.getId(), now))`.
  - `softDeleteChildren(reportId, group)`: thêm `shipRepository.findByAppraisalAssetGroupId(group.getId()).filter(ship -> Objects.equals(ship.getDeleteFlag(), AppraisalShipDescription.NOT_DELETED)).ifPresent(AppraisalShipDescription::softDelete)` (không gọi `deleteAllOfShipDescription`).
  - Cập nhật Javadoc lớp: đưa "Khác hệ thống cũ #1: xóa nhóm rỗng xóa mềm mô tả tàu (giữ ảnh); khôi phục chỉ hồi sinh mô tả khi nhóm vẫn là tàu (đúng legacy)".
- [ ] **D4. Chạy** → PASS; chạy `./mvnw -q test -Dtest='ReportGroupResolverTest,AppraisalReportBasicEditServiceTest'` → PASS.
- [ ] **D5. Commit:** `feat: khôi phục và xóa nhóm tàu thủy xử lý mô tả tàu, ảnh`.

### Phần E — Pháp lý nhận loại 4 (`ReportLegalDocumentValidator`)

> Nới cận cơ sở thẩm định 3→4 (`ReportContentMerge`, `AppraisalReportFormService`) **do B01 làm** (Quyết định #4). B07 chỉ làm `ReportLegalDocumentValidator.isKnownType`.

- [ ] **E1. Test thất bại:** `ReportLegalDocumentValidatorTest`: đổi `onlyLandApartmentAndCarAreValidAssetTypes` thành `onlyLandApartmentCarAndShipAreValidAssetTypes`: `problem(doc(true, null, 5, "A"))` và `6` là `"Loại tài sản không hợp lệ."`; `4` và `3` là `null`. Thêm `documentsOfTheSameGroupMustShareTheAssetType` có cặp `(4, 3)` cùng nhóm: lỗi loại khác nhau (dùng đúng thông điệp hiện có của test cũ).
- [ ] **E2. Chạy** `./mvnw -q test -Dtest=ReportLegalDocumentValidatorTest` -> FAIL.
- [ ] **E3. Cài đặt:** `ReportLegalDocumentValidator.isKnownType`: thêm `|| assetType == AppraisalAssetType.SHIP`; sửa Javadoc/ghi chú loại 1/2/3/4. `LegalDocumentItem`/`ReportLegalDocumentInput` Javadoc ("1 đất, 2 căn hộ, 3 ô tô") thêm "4 tàu thủy" (chỉ comment) nếu các file này chưa được B04/B01 sửa.
- [ ] **E4. Chạy** lệnh E2 -> PASS.
- [ ] **E5. Commit:** `feat: pháp lý của báo cáo nhận loại tài sản tàu thủy`.

### Phần F — Rà soát cuối task

- [ ] **F1. Không còn `default` ngầm hiểu CAR/LAND ở vòng đời:** `git -C backend grep -n "AppraisalAssetType.CAR" -- src/main/java/com/thanhdo/tdprojectbe/service/{GroupAssetTypeReset,ReportGroupResolver,ReportGroupDetailsWriter,ControlFactorDefaults}.java` — mỗi chỗ được so sánh tường minh với SHIP song song.
- [ ] **F2.** `cd backend && ./mvnw -q test -Dtest='ControlFactorDefaultsTest,ReportGroupDetailsWriterTest,GroupAssetTypeResetTest,ReportGroupResolverTest,ReportLegalDocumentValidatorTest,AppraisalReportBasicEditServiceTest'` → PASS.
- [ ] **F3.** Báo cáo cuối task: file đã sửa, kết quả test, các điểm lệch so với "Giả định về B01".

---

## Vấn đề cần controller quyết định

1. **→ Đã chốt: Quyết định #1.** Khóa JSON kết quả tàu là `ship` (record `ShipDetail`) ở cả API mới lẫn DTO Word; F01/F11 dùng `ship`.
2. **→ Đã chốt (01-be-foundation, "Ngữ nghĩa chốt"): `applyCurrency` mang luật VND→1, USD null giữ rate cũ, trim ghi chú; `create` khởi tạo VND/1 qua khai báo field.** B06 chỉ resolve mã rồi gọi (test B1 vẫn đúng).
3. **→ Đã chốt: Quyết định #8.** Xóa nhóm chỉ xóa mềm mô tả tàu, không xóa ảnh; đổi loại mới xóa ảnh.
4. **Địa chỉ dòng so sánh type 0 của nhóm tàu tạo ở màn sửa cơ bản.** `ReportGroupResolver.restoreOrCreate` tạo nhóm trước khi tài sản chuyển sang nên `address` lúc đó null (legacy `createGroupCompares` cũng đọc collection rỗng ở thời điểm này). Địa chỉ chỉ chắc chắn đúng ở luồng tạo báo cáo; ở luồng sửa cơ bản sẽ được ghi khi lưu mô tả tàu (B03 đồng bộ). Chấp nhận hay yêu cầu B07 gán lại địa chỉ sau khi tài sản đã chuyển?
5. **→ Đã chốt: controller gộp dòng helper của B06 (`ShipValuationMath`) vào `backend/CLAUDE.md` sau đợt B2** (B06 không sửa file này; B01 sửa ở đợt B1).
6. **→ Đã chốt: Quyết định #15 (ngoài phạm vi).** Báo cáo VCB với nhóm tàu: sau B06 `EvaluationResultCalculator` tính được nhóm SHIP; nếu cần chặn thì làm riêng.
7. **`ControlFactorDefaults.default → LAND`.** Quyết định: **giữ** (khớp legacy `resolveDefaultControlCompareFactorLabels` và test `labelsFor(9)==labelsFor(1)`), SHIP có `case` riêng. Loại lạ không thể tới đây vì `ReportLegalDocumentValidator.isKnownType` chặn từ đầu vào; nếu muốn chặt hơn (ném lỗi với loại lạ) là thay đổi hành vi ngoài phạm vi port.
8. **→ Đã đối chiếu với khối Interfaces của B01:** `empty` gán 18 tình trạng, `NOT_DELETED` có, `resetToDefault()` giữ id/created, `clearShipFields()` idempotent và được `reset` gọi, hằng nhãn tàu do B01 tạo (Quyết định #3).
9. **Test hiện có bị đổi có chủ đích** (không phải "sửa để xanh"): `EvaluationResultControllerTest.LEGACY_KEYS` 24→28 khóa; `ReportLegalDocumentValidatorTest` loại "không hợp lệ" 4→5; `GroupAssetTypeResetTest`/`ReportGroupResolverTest`/`ReportGroupDetailsWriterTest` đổi constructor. Nhắc reviewer đừng coi đây là vi phạm "không sửa test cũ".
