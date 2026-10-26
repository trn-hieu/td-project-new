# 02 — API tính kết quả thẩm định (T03–T05)

> Đọc `00-index.md` trước. Subagent: `model: "sonnet"`, có review. **Yêu cầu người dùng: port y nguyên** — công thức, thứ tự làm tròn, giá trị null, tên trường JSON phải giống hệ thống cũ. Không "cải tiến".

## Hành vi hệ thống cũ

Nguồn: `L controller/api/EvaluationResultApiController.java`, `L service/impl/AppraisalResultServiceImpl.java` (đọc **toàn bộ**), `L entity/dto/EvaluationResultDetailDto.java`, `L common/LandPriceUtils.java`, `L common/ReadMoneyUtils.java`, `L common/NumberFormatUtils.java:35`, `L common/CommonUtils.java:801`.

- `GET /api/evaluation-results/detail/{reportId}/{groupId}?roundingMoneyType=` → `getDetailByReportAndGroup(r, g, override)` (l.89–107): nhóm không có / khác báo cáo → DTO rỗng (mọi trường null). `compare0` = dòng `type == 0` đầu tiên chưa xóa; synthetic theo nhóm; công trình chưa xóa → `buildDetailFromEntities` (l.110–133) theo `assetType` 1/2/3, khác → rỗng.
- **Làm tròn** (l.594–625): override → `appraisalResult.roundingMoneyType` của nhóm (chỉ khi cùng báo cáo) → 1; khác 1/2 → 1. Base: 2 → 1.000, còn lại 1.000.000. `round(x) = Math.round((double) x / base) * base`.
- **Đất** (l.135–487): dòng ngoài QH mặc định từ `compare0.areaOutsideDetails` (`amount = roundLandAmountVnd(parseEuropeanArea(area), price)`); theo phương pháp (`compare0.appraisalAssetGroup.landValuationMethod`, fallback nhóm):
  - AGRICULTURAL_STATE_PRICE (l.184–237): danh sách rỗng → một dòng `(formatFixed2(areaOutside), avg)`; `landOutside.area = normalize(Σ area)`, `unitPrice = null`; dòng giá lớn nhất (hòa giữ dòng đầu, `Stream.max`) thay giá bằng `avgPriceGuide`, đánh `maxPriceRecord = TRUE`; `maxPriceOutside`, `totalAreaBelowMaxPrice = normalize(Σ area có price < max)`; `resolutionContent` luôn null.
  - LAND_USE_CONVERSION (l.238–327): `LandPriceUtils.calculate(landPriceJson, avg)`; dựng lại dòng: đất ở (landTypeId 1) giá `avg`; mỗi loại phi nông nghiệp: phần chuyển đổi `landName + " chuyển đổi thành đất ở"` giá `donGiaThiTruong`, phần không chuyển đổi giá đơn giá nhà nước; chỉ hiện phần có diện tích > 0.
  - STATE_PRICE (l.328–363): giá mỗi dòng = `landPriceJson[landTypeId].unitPrice.longValue()`.
  - Khác/MIXED/null (l.364–374): `landOutside = {formatFixed2(areaOutside), avg, roundLand(areaOutside, avg)}`, giữ dòng mặc định.
  - Trong QH (l.377–422): chỉ khi có chi tiết (không có → `landInsideDetails = null`); giá = đơn giá nhà nước nếu STATE_PRICE, ngược lại `detail.price`.
  - Công trình (l.424–467; danh sách rỗng → `constructions = null`): `area = getValuationUsageArea()`, `originalAmount = round0(area×up)`, `amount = (area×up×(q ?? 100)) / 100` HALF_UP, `roundedAmount = round(amount)`, `depreciationRate = 100 − q`, `depreciationAmount = originalAmount×rate/100`; tổng tính vào giá trị khi `valuationType` null/INCLUDE_IN_TOTAL, tham khảo khi REFERENCE.
  - Tổng (l.470–485): `landAmount`, `roundedLandAmount`, `constructionAmount`, `roundedConstructionAmount`, `referenceConstructionAmount`, `roundedReferenceConstructionAmount`, `totalMoney = roundingMoney = roundedLand + roundedConstruction`, `totalPriceWordAppraisal = ReadMoneyUtils.doc(totalMoney)`.
- **Căn hộ** (l.489–592): chỉ DIRECT_COMPARISON có `directComparison` (`basis` mặc định 1; diện tích theo basis; `amount = (long)(areaM2 × avg)` — **cắt phần lẻ**; `roundedAmount = round(amount)`; nhãn "Tim tường"/"Thông thủy"); phương pháp khác → `apartment = {valuationMethod}`, `totalMoney = 0`.
- **Ô tô** (l.529–557): `car.amount = avg`, `car.roundedAmount = round(avg)`, `totalMoney = roundedAmount ?? 0`.
- Trường chung (l.627–648): `note` từ kết quả đã lưu, `landValuationMethod` (tên enum), `appraisalAssetGroupId`, `assetName` = tài sản đầu tiên của nhóm, `assetType`.
- **Lưu** (l.697–744): upsert `appraisal_result` theo (report, group), **ghi đè mọi cột** từ request (UI chỉ gửi `appraisalRequestId, appraisalReportId, appraisalAssetGroupId, totalMoneyAppraisal, roundingMoney, roundingMoneyType, totalPriceWordAppraisal, note`; 16 cột diện tích/đơn giá/chất lượng/tiền và `appraisalProfileId` thành null); insert: `created = now`, `createdBy = employeeId`; luôn `updated = now`, `deleted = 0`. Cột yêu cầu tên thật là `appraisa_request_id` (sai chính tả có thật).
- Danh sách trường đầy đủ của DTO: xem bảng ở cuối file.

Bản port rút gọn hiện có (`N service/result/`): `EvaluationResultCalculator` (thuần), `EvaluationResultDetail` (record rút gọn), `EvaluationResultLoader`, `LandOutsideValuation`, `LandPrices`, `ResultMath`; test `NT service/result/EvaluationResultCalculatorTest` (25 test), `EvaluationResultLoaderTest`, `LandPricesTest`, `EvaluationFixtures`; người dùng duy nhất `N service/VcbReportAssembler.java` (+ `VcbReportAssemblerTest`, `VcbReportServiceTest`, `VcbReportServiceMysqlTest`). Thiếu so với cũ: `maxPriceRecord`, `landAmount`, `referenceConstruction*`, `roundingMoneyType` + override, `totalPriceWordAppraisal`, `appraisalAssetGroupId`, `assetName`, `apartment`, `car`, `landValuationMethod`, `totalAreaBelowMaxPrice`, `maxPriceOutside`, `resolutionContent`, các cột khấu hao của công trình; danh sách rỗng là `List.of()` thay vì null.

---

### Task T03: `MoneyWords`

**Files:** Create `N common/util/MoneyWords.java`; Test `NT common/util/MoneyWordsTest.java`; Modify `backend/CLAUDE.md` (bảng helper).

**Interfaces:** Produces `public final class MoneyWords { public static String read(long amount) }` — port `L common/ReadMoneyUtils.java` `doc(long)`: 0 → "Không đồng"; > 8.999.999.999.999.999 → ""; thêm "," sau "tỷ" khi còn nhóm thấp hơn; viết hoa chữ đầu; hậu tố " đồng"; số âm tiền tố "Âm ".

- [ ] **Step 1: Test thất bại**: lấy **ít nhất 8 cặp** đầu vào/đầu ra bằng cách chạy chính `ReadMoneyUtils.doc` cũ (viết test tạm trong legacy hoặc dịch tay từng nhánh) — bắt buộc gồm 0, 5, 15, 105, 1.000.000, 1.250.000.000, 2.000.000.005, số âm, 9.000.000.000.000.000. Đối chiếu thêm với `F lib/read-money.ts` (phải cùng kết quả).
- [ ] **Step 2: Chạy** `./mvnw -q test -Dtest=MoneyWordsTest` → FAIL.
- [ ] **Step 3: Port** giữ nguyên thuật toán cũ.
- [ ] **Step 4: Chạy** → PASS. Thêm dòng helper `| Số tiền bằng chữ tiếng Việt (giống ReadMoneyUtils cũ) | MoneyWords.read(amount) (common/util) | Tự viết bảng chữ số trong service mới |`.
- [ ] **Step 5: Commit** `feat: MoneyWords đọc số tiền bằng chữ`.

### Task T04: `AppraisalResultCalculator` port đầy đủ, bản rút gọn ủy quyền

**Files:**
- Create: `N model/dto/EvaluationResultResponse.java` (record + record lồng), `N service/result/AppraisalResultCalculator.java`
- Modify: `N service/result/LandOutsideValuation.java` (kết quả mang thêm `maxPriceRecord` theo dòng, `maxPrice`, `totalAreaBelowMax`), `N service/result/EvaluationResultCalculator.java` (giữ chữ ký `calculate(...)`, thân = gọi `AppraisalResultCalculator.calculate(..., assetName=null, override=null)` rồi map sang `EvaluationResultDetail`: null → `List.of()`, rỗng → `EMPTY`)
- Test: `NT service/result/AppraisalResultCalculatorTest.java` (mới), giữ nguyên `EvaluationResultCalculatorTest`, `VcbReportAssemblerTest`

**Interfaces:**
- Produces:
  - `record EvaluationResultResponse(LandOutsideRow landOutside, List<AreaOutsideRow> landOutsideDetails, List<AreaInsideRow> landInsideDetails, List<ConstructionRow> constructions, Long landAmount, Long roundedLandAmount, Long constructionAmount, Long roundedConstructionAmount, Long referenceConstructionAmount, Long roundedReferenceConstructionAmount, Long totalMoney, Long roundingMoney, Integer roundingMoneyType, String totalPriceWordAppraisal, String note, Long appraisalAssetGroupId, String assetName, Integer assetType, ApartmentDetail apartment, CarDetail car, String landValuationMethod, String totalAreaBelowMaxPrice, String resolutionContent, Long maxPriceOutside)` + `static EvaluationResultResponse empty()` (mọi trường null). Record lồng (tên trường giống DTO cũ): `LandOutsideRow(String area, Long unitPrice, Long amount)`, `AreaOutsideRow(Long landTypeId, String landType, String area, Long price, Long amount, Boolean maxPriceRecord)`, `AreaInsideRow(Long landTypeId, String landType, String area, Long price, Long amount)`, `ConstructionRow(String constructionName, String constructionType, BigDecimal area, BigDecimal unitPrice, BigDecimal qualityRemain, BigDecimal depreciationRate, Long amount, Long originalAmount, Long roundedAmount, Long depreciationAmount, String valuationType)`, `ApartmentDetail(String valuationMethod, DirectComparison directComparison)`, `DirectComparison(Integer ownershipAreaBasis, String ownershipAreaBasisLabel, String area, Long unitPrice, Long amount, Long roundedAmount)`, `CarDetail(Long amount, Long roundedAmount)`.
  - `public final class AppraisalResultCalculator { static EvaluationResultResponse calculate(AppraisalAssetGroup group, AppraisalGroupCompare compare0, ControlCompareFactorSynthetic synthetic, List<AppraisalConstructionDescription> constructions, List<LandPrice> landPrices, AppraisalResult stored, String assetName, Integer roundingMoneyTypeOverride) }` — hàm thuần; `group.getAssetType()` null hoặc khác 1/2/3 → `empty()`.

- [ ] **Step 1: Test thất bại** `AppraisalResultCalculatorTest` (dùng/ mở rộng `EvaluationFixtures`), mỗi test so **toàn bộ record** với giá trị tính tay theo code cũ:
  - `mixed_một_dòng_ngoài_qh_làm_tròn_triệu`
  - `nông_nghiệp_dòng_giá_lớn_nhất_thay_bằng_avg_và_đánh_dấu` (2 dòng cùng giá lớn nhất → chỉ dòng đầu `maxPriceRecord=true`; `totalAreaBelowMaxPrice`, `maxPriceOutside`)
  - `nông_nghiệp_không_có_chi_tiết_dùng_areaOutside`
  - `chuyển_đổi_tách_phần_chuyển_đổi_và_không_chuyển_đổi` (dùng số liệu `LandPricesTest`)
  - `giá_nhà_nước_lấy_đơn_giá_bảng_giá_đất` (ngoài và trong QH)
  - `không_có_chi_tiết_trong_qh_và_công_trình_thì_null`
  - `công_trình_khấu_hao_và_tham_khảo` (INCLUDE, REFERENCE, NOT_INCLUDED; `depreciationAmount`, `roundedAmount` từng dòng)
  - `override_làm_tròn_nghìn_thắng_giá_trị_đã_lưu`; `kết_quả_đã_lưu_thuộc_báo_cáo_khác_bị_bỏ_qua`
  - `căn_hộ_thông_thủy_cắt_phần_lẻ`; `căn_hộ_phương_pháp_khác_tổng_0`; `căn_hộ_không_có_compare0_nhãn_tim_tường`
  - `ô_tô_làm_tròn_avg`; `assetType_null_trả_rỗng`
  - `bằng_chữ_dùng_MoneyWords`
- [ ] **Step 2: Chạy** `./mvnw -q test -Dtest=AppraisalResultCalculatorTest` → FAIL.
- [ ] **Step 3: Cài đặt** port từng hàm `build*` của `AppraisalResultServiceImpl` (Javadoc dẫn số dòng cũ), dùng lại `ResultMath`, `LandPrices`, `EuropeanNumbers`, `LandOutsideValuation` (mở rộng, không copy). Khác biệt duy nhất được phép: `EuropeanNumbers` dùng `Locale.ROOT` (đã ghi chú sẵn).
- [ ] **Step 4: Chuyển `EvaluationResultCalculator.calculate` sang ủy quyền**, xóa phần trùng lặp công thức.
- [ ] **Step 5: Chạy** `./mvnw -q test -Dtest='AppraisalResultCalculatorTest,EvaluationResultCalculatorTest,EvaluationResultLoaderTest,LandPricesTest,VcbReportAssemblerTest,VcbReportServiceTest'` → PASS (test cũ **không sửa** để làm xanh; nếu phải sửa → dừng, báo lại).
- [ ] **Step 6: Commit** `feat: port đầy đủ AppraisalResultCalculator, bản rút gọn ủy quyền`.

### Task T05: API kết quả thẩm định + lưu

**Files:**
- Modify: `N model/entity/AppraisalResult.java` (`@GeneratedValue(IDENTITY)`, thêm toàn bộ cột ở mục Hành vi; factory `static AppraisalResult create(long reportId, long groupId, long createdBy)` và `void overwrite(EvaluationResultSaveRequest req, Long requestId)`), `N repository/AppraisalReportAssetRepository.java` (`findFirstByAppraisalAssetGroupIdAndDeleteFlagOrderByIdAsc`), `N service/result/EvaluationResultLoader.java` (thêm `loadFull`)
- Create: `N model/dto/EvaluationResultSaveRequest.java`, `N service/EvaluationResultService.java`, `N controller/EvaluationResultController.java`
- Test: `NT repository/AppraisalResultRepositoryTest.java` (`@MysqlDataJpaTest`: ghi đủ cột rồi đọc lại; xác minh tên cột `created_by`/`createdBy` thật), `NT service/result/EvaluationResultLoaderTest.java` (thêm), `NT service/EvaluationResultServiceTest.java`, `NT controller/EvaluationResultControllerTest.java`

**Interfaces:**
- Consumes: `ReportEditScope.requireGroup` (T01), `AppraisalResultCalculator.calculate` (T04).
- Produces:
  - `EvaluationResultLoader.loadFull(long reportId, long groupId, Integer roundingMoneyTypeOverride): EvaluationResultResponse`.
  - `record EvaluationResultSaveRequest(String totalMoneyAppraisal, Long roundingMoney, @NotNull @Min(1) @Max(2) Integer roundingMoneyType, @Size(max = 500) String totalPriceWordAppraisal, String note)`.
  - `GET  …/reports/{reportId}/groups/{groupId}/evaluation-result?roundingMoneyType=` → `EvaluationResultResponse` (JSON có cả trường null).
  - `PUT  …/reports/{reportId}/groups/{groupId}/evaluation-result` body `EvaluationResultSaveRequest` → `ApiResponse.ok(null, "Lưu kết quả thẩm định thành công")`. Giống cũ: lưu giá trị client gửi; `appraisa_request_id` = yêu cầu của báo cáo; `appraisal_profile_id` và 16 cột cũ = null; audit như cũ.

- [ ] **Step 1: Test thất bại**: repository test ghi/đọc; `EvaluationResultServiceTest`: `lưu_lần_đầu_tạo_dòng_và_created_by`, `lưu_lần_hai_ghi_đè_và_null_cột_cũ`, `nhóm_sai_báo_cáo_404`; controller: `json_giống_hệ_thống_cũ` (so tập khóa JSON với danh sách 24 trường ở cuối file, gồm khóa có giá trị null), `roundingMoneyType_3_trả_400`.
- [ ] **Step 2–4:** chạy FAIL → cài đặt → chạy PASS (kèm toàn bộ test `service/result`).
- [ ] **Step 5: Commit** `feat: API kết quả thẩm định theo nhóm tài sản`.

### Bảng trường `EvaluationResultResponse` (khớp `EvaluationResultDetailDto` cũ)

| Trường | Đất | Căn hộ | Ô tô |
|---|---|---|---|
| landOutside, landOutsideDetails | có khi có compare0 | null | null |
| landInsideDetails, constructions | có hoặc **null** | null | null |
| landAmount, roundedLandAmount, constructionAmount, roundedConstructionAmount, referenceConstructionAmount, roundedReferenceConstructionAmount | có | null | null |
| totalMoney, roundingMoney, roundingMoneyType, totalPriceWordAppraisal, appraisalAssetGroupId, assetName, assetType, landValuationMethod | có | có | có |
| note | khi có kết quả đã lưu | như Đất | như Đất |
| apartment | null | có | null |
| car | null | null | có |
| totalAreaBelowMaxPrice, maxPriceOutside | chỉ AGRICULTURAL | null | null |
| resolutionContent | luôn null | luôn null | luôn null |
