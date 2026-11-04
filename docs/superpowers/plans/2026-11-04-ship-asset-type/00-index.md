# Migrate loại tài sản SHIP (tàu thủy) — Implementation Plan (index)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **Model cho subagent: CHỈ dùng `sonnet`.** Mọi lần gọi tool `Agent` (implementer, spec reviewer, code reviewer, final reviewer) phải truyền `model: "sonnet"`. Không dùng opus/haiku/fable cho subagent. **Mỗi task bắt buộc qua review** (spec review rồi code review) trước khi đánh dấu xong.

**Goal:** Port loại tài sản thứ 4 là **SHIP (tàu thủy)** từ nhánh legacy `report/ship` (chốt tại commit **`82193fe5`**) sang `backend` và `frontend`. Phạm vi port:
- màn sửa thông tin cơ bản;
- 5 tab của nhóm tàu: Pháp lý, Mô tả tàu thủy, Thông tin so sánh, Bảng điều chỉnh, Kết quả thẩm định;
- tiền tệ theo nhóm (VND/USD, tỷ giá, ghi chú);
- kho TSSS cho tàu;
- vòng đời nhóm khi đổi loại, xóa hoặc khôi phục;
- xuất Word.

**Architecture:**
- Làm theo đúng kiến trúc đã port cho LAND/APARTMENT/CAR (bộ plan `docs/superpowers/plans/2026-10-26-appraisal-report-edit-tab/`). SHIP lấy CAR làm mẫu gần nhất.
- Backend thêm:
  - entity `AppraisalShipDescription` (1-1 với nhóm);
  - cột tàu trên các entity sẵn có;
  - API `ship-description`;
  - nhánh SHIP trong các service hiện có;
  - `ShipAssetWriter` cho Word.
- Frontend thêm:
  - `ASSET_TYPE.SHIP`;
  - các component tab tàu;
  - hàm tính thuần (chất lượng còn lại, tiền tệ) kèm unit test.
- Mọi chỗ đang dùng `else` để ngầm hiểu là CAR phải tách ra thành nhánh SHIP riêng.

**Tech Stack:** Spring Boot 4 · Java 21 · Spring Data JPA (MySQL dùng chung với legacy, `ddl-auto=none`) · JUnit 5/Mockito/AssetJ · `@MysqlDataJpaTest` · Apache POI (Word) · Vue 3 + TS · shadcn-vue · vee-validate + zod · Vitest.

**Spec:** Spec là hành vi của legacy tại `82193fe5`. Kết quả khảo sát nằm trong thư mục `surveys/` cạnh file này, **implementer phải đọc phần liên quan**:
- `surveys/legacy-be.md`: backend legacy (hằng số, entity, migration, API, service, công thức, test spec).
- `surveys/legacy-ui-word.md`: giao diện legacy (từng tab) và Word.
- `surveys/new-code.md`: điểm cần chạm ở code mới.

Đọc code legacy bằng `git -C td-project-legacy show 82193fe5:<path>`. **Không checkout.** Tài liệu legacy: `git -C td-project-legacy show 82193fe5:docs/SHIP.md` và `docs/SHIP_MIGRATION_PLAN.md` (có phần đã cũ, code tại `82193fe5` là chuẩn).

Viết tắt dùng trong mọi file:
- **L** = `td-project-legacy` @ `82193fe5`, gốc `src/main/java/com/jvb/thanhdo/`.
- **LR** = `src/main/resources/` của L.
- **LW** = `src/main/java/com/jvb/thanhdo/controller/component/wordExport/AppraisalReportWordExporter.java` của L.
- **N** = `backend/src/main/java/com/thanhdo/tdprojectbe/`.
- **NT** = `backend/src/test/java/com/thanhdo/tdprojectbe/`.
- **F** = `frontend/src/`.

## Danh sách file kế hoạch và thứ tự task

Các task **cùng đợt có tập file rời nhau**, nên chạy song song được (xem "Quy trình"). Các đợt chạy nối tiếp nhau. BE và FE là hai submodule độc lập nên hai luồng chạy song song. FE chỉ phụ thuộc vào **Hợp đồng API** bên dưới và mock API trong test.

| File | Task | Đợt | Nội dung | Phụ thuộc |
|---|---|---|---|---|
| `01-be-foundation.md` | B01 | B1 | Nền BE: kiểm tra schema, hằng số, helper tiền tệ, entity/converter/repository, phương thức entity, lớp thuần `ShipQuality` | — |
| `02-be-description-images.md` | B02 | B2 | Ảnh tàu trong `AppraisalImageService` | B01 |
| | B03 | B2 | API Mô tả tàu + đồng bộ sang so sánh type 0; `LandDescriptionService` chặn SHIP | B01 |
| `03-be-legal-compare-panel-kho.md` | B04 | B2 | Pháp lý tàu | B01 |
| | B05 | B2 | So sánh + bảng điều chỉnh (giá, tiền tệ, chặn TSSS lệch tiền tệ, đồng bộ nhãn tàu) | B01 |
| | B10 | B2 | Kho TSSS tàu (tìm theo chủng loại, đồng bộ lên kho, quyền BGD) | B01 |
| `04-be-result-lifecycle.md` | B06 | B2 | Kết quả thẩm định + lưu tiền tệ | B01 |
| | B07 | B3 | Vòng đời nhóm (tạo, đổi loại, xóa, khôi phục) | B01, B02 |
| `05-be-word.md` | B08 | B2 | Word phần 1: writer mô tả tàu, tình trạng, chất lượng | B01 |
| | B09 | B3 | Word phần 2: `ShipAssetWriter` + so sánh, kết quả, ảnh, pháp lý liên quan, bảng điều chỉnh | B06, B08 |
| `10-fe-foundation.md` | F01 | F1 | Nền FE: hằng số, kiểu, API client, tabs, `GroupWorkspace`, `currency.ts` | — |
| | F02 | F1 | Màn cơ bản: loại "Tàu thủy", cơ sở thẩm định tàu | — |
| `11-fe-ship-description.md` | F03 | F2 | Hàm thuần `ship-quality.ts`, `ship-description-fields.ts` | F01 |
| | F04 | F3 | `ShipSpecSections.vue` | F03 |
| | F05 | F3 | `ShipConditionTable.vue`, `ShipQualityTable.vue` | F03 |
| | F06 | F4 | `ShipDescriptionTab.vue` (lắp ráp, ảnh, lưu) | F04, F05 |
| `12-fe-compare-panel-results-legal.md` | F07 | F2 | Pháp lý tàu | F01 |
| | F09 | F2 | Bảng điều chỉnh tàu | F01 |
| | F10 | F2 | Modal tìm TSSS tàu từ kho | F01 |
| | F11 | F2 | Kết quả tàu + chọn tiền tệ | F01 |
| | F08 | F3 | Tab So sánh tàu | F01, F10 |
| `90-final-review.md` | Z01 | cuối | Review toàn nhánh, chạy toàn bộ test, E2E thủ công | tất cả |

## Nhánh làm việc

- `backend` và `frontend` là **git submodule**. Trong mỗi submodule, tạo nhánh `feat/ship-asset-type` từ **`origin/develop` mới nhất**. Việc này làm ở B01 và F01:
  ```bash
  git -C backend -c http.sslBackend=schannel fetch origin develop && git -C backend checkout -b feat/ship-asset-type origin/develop
  git -C frontend -c http.sslBackend=schannel fetch origin develop && git -C frontend checkout -b feat/ship-asset-type origin/develop
  ```
  Lúc lập plan, `origin/develop` của backend là `b3d6d7e`, frontend là `6ff0110`. Hai commit này đã chứa edit-tab, edit-basic và word-exporter.
- Số dòng của code mới trong `surveys/new-code.md` đo trên `refactor/appraisal-word-exporter` / `feat/appraisal-report-edit-basic`, có thể lệch so với develop. **Luôn grep lại trước khi sửa.**
- Mọi commit chạy bằng `git -C backend …` hoặc `git -C frontend …`. **Không push**, không cập nhật con trỏ submodule ở repo gốc khi chưa được người dùng đồng ý.
- Nếu `origin/develop` có commit mới trong lúc làm, chỉ merge vào **giữa các đợt**, không merge giữa chừng một task.
- Nếu nhánh legacy `report/ship` có commit mới sau `82193fe5`, controller phải khảo sát phần chênh lệch và cập nhật plan trước khi làm tiếp.

## Quy trình cho controller (phiên chính) — bắt buộc

Dùng superpowers:subagent-driven-development. Với **mỗi task**:

1. **Implementer** — `Agent({ subagent_type: "general-purpose", model: "sonnet", prompt })`.
   - Prompt gồm: (a) đường dẫn `00-index.md` và file chứa task, yêu cầu đọc cả hai; (b) "Chỉ làm Task Xnn; đọc các file survey và code legacy được nêu trong task trước khi viết code; TDD theo các Step; commit trong submodule theo Step cuối"; (c) khối Interfaces của các task trước mà task này dùng.
   - Implementer không được tự mở rộng phạm vi.
   - Gặp điều plan không quyết định được thì **dừng và báo lại**, không đoán. Ví dụ: cột không tồn tại trong DB, hành vi legacy khác mô tả, file của task khác cùng đợt cần sửa.
2. **Spec reviewer** — `Agent({ model: "sonnet" })`.
   - Đối chiếu diff với task: Files, Interfaces, Steps.
   - Đối chiếu với mục "Hành vi hệ thống cũ" (đọc lại code L ở dòng được dẫn), danh sách "Khác hệ thống cũ" và Review Focus.
   - Kết luận APPROVED, hoặc liệt kê lỗi kèm file:dòng.
3. **Code reviewer** — `Agent({ model: "sonnet" })`.
   - Đối chiếu với `backend/CLAUDE.md`, `backend/.claude/rules/*.md`, `frontend/CLAUDE.md`, `frontend/DESIGN_SYSTEM.md`.
   - Chạy lại lệnh kiểm tra của task.
4. Lỗi được xác nhận thì gửi lại implementer (SendMessage cùng agent) để sửa, rồi review lại bước đã fail. Chỉ đánh dấu `[x]` khi cả hai reviewer APPROVED.

**Chạy song song trong một đợt.** Mỗi implementer làm trong worktree riêng của submodule, tạo từ đầu nhánh `feat/ship-asset-type` lúc bắt đầu đợt:
```bash
git -C backend worktree add ../backend-wt-B03 -b wt/B03 feat/ship-asset-type
```
Hết đợt, merge từng nhánh `wt/<ID>` vào `feat/ship-asset-type` theo thứ tự ID rồi xóa worktree. Merge có conflict nghĩa là phân chia file sai: dừng và báo lại. Nếu không muốn chạy song song thì chạy tuần tự theo ID trong chính `backend/` hoặc `frontend/`.

**Hết mỗi đợt:**
- BE: chạy `cd backend && ./mvnw test`.
- FE: chạy `cd frontend && npm run lint && npm run type-check && npm run test:unit`.

## Điều kiện tiên quyết: schema do legacy sở hữu

Backend mới **không tạo Flyway migration** cho phần tàu. Các migration dưới đây của L (`LR/db/migration/`) phải **đã chạy trên DB dev/test** trước khi chạy test repository. B01 Step 1 kiểm tra bằng `DESCRIBE`. Thiếu bảng hoặc cột thì **dừng, báo người dùng**.

```
V20261015100000__create_appraisal_ship_description.sql
V20261015100100__add_appraisal_ship_description_to_appraisal_image.sql
V20261015100200__add_ship_fields_to_appraisal_group_compares.sql
V20261015100300__seed_ship_control_compare_factor_labels.sql
V20261017100000__change_ship_dim_depth_draft_to_decimal.sql
V20261017100100__change_ship_deadweight_to_decimal.sql
V20261017100200__change_ship_build_year_place_to_255.sql
V20261017100300__add_ship_inspection_certificate_fields.sql
V20261017100400__seed_valuation_basis_ship.sql
V20261018100000__add_quality_evaluation_to_ship_description.sql
V20261023100000__add_ship_inspection_for_whom.sql
V20261024100000__change_ship_quantity_volume_fields_to_text.sql
V20261024110000__change_ship_fire_manual_tools_layout_plan_to_boolean.sql
V20261027100001__change_anchor_stern_total_weight_to_text.sql
V20261028100000__add_ship_towing_equipment_fields.sql
V20261028110000__add_ship_mooring_location.sql
V20261028120000__seed_valuation_method_ship.sql
V20261028130000__seed_report_juridical_common.sql
V20261031100000__add_currency_code_to_appraisal_asset_groups.sql
V20261031110000__add_exchange_rate_and_currency_note_to_appraisal_asset_groups.sql
V20261103100000__add_ship_fields_to_compare_information.sql
V20261103110000__split_ship_build_year_place.sql
V20261103120000__rename_ship_control_compare_factor_label_nam_dong.sql
V20261103130000__reset_ship_quality_evaluation_details.sql
```

Entity map theo **DDL cuối cùng** sau khi chạy hết các file trên. Một số kiểu dễ nhầm:
- `build_year_place` đã bị DROP, thay bằng `build_year` INT và `build_place` TEXT.
- `dim_depth` và `dim_draft` là DECIMAL(10,2). `deadweight` là DECIMAL(12,2).
- 20 cột "số lượng/thể tích", `anchor_stern_total_weight_kg`, `mooring_bollard_quantity` là TEXT.
- `fire_manual_tools` và `fire_layout_plan` là TINYINT(1).
- `appraisal_group_compares.ship_build_year` là INT. Có thêm `ship_build_place` TEXT.
- `appraisal_asset_groups` có `currency_code` VARCHAR(10) NOT NULL DEFAULT 'VND', `exchange_rate` BIGINT NOT NULL DEFAULT 1 và `currency_note` TEXT.

## Hợp đồng dùng chung

### Hằng số

**Loại tài sản, cơ sở thẩm định, loại ảnh**
- `SHIP = 4`. Backend phải thêm ở **cả hai** nơi: `N/common/constant/AppraisalAssetType` và `CommonConstants.AppraisalAssetType`.
- `ValuationBasis.ASSET_TYPE_SHIP = 4` (cơ sở thẩm định).
- FE: `ASSET_TYPE.SHIP = 4` (`F/lib/report-edit/tabs.ts`).
- Loại ảnh tàu (BE + FE):

| Mã | Tên |
|---|---|
| 21 | Mặt trước tàu |
| 22 | Đầu tàu |
| 23 | Đuôi tàu |
| 24 | Mạn tàu |
| 25 | Sàn tàu |
| 26 | Số IMO/Số kiểm soát |
| 27 | Hệ thống neo |
| 28 | Phòng lái, phòng nghỉ |
| 29 | Khoang hàng |
| 30 | Hệ thống máy của động cơ |

  Mục "Khác" dùng mã 10 chung.

**Tiền tệ** (BE `N/common/util/CurrencyCodes`; FE `F/lib/report-edit/currency.ts`; logic hai bên giống hệt nhau)
- Mã: `VND`, `USD`.
- `resolveCurrencyCode(assetType, code)` chỉ trả `"USD"` khi `assetType == SHIP` và `code` bằng "usd" (không phân biệt hoa thường). Mọi trường hợp khác (loại khác, null, mã lạ) trả `"VND"`.
- Nhãn và đơn vị theo mã:

| Hàm | VND | USD |
|---|---|---|
| `currencyUnitLabel` | `"VND"` | `"USD"` |
| `currencyMoneyLabel` | `"đồng"` | `"USD"` |

- `regulatoryPriceUnit(assetType, code)`: SHIP trả `"đồng"` hoặc `"USD"` (**không** có "/tàu", không có "/m²"); CAR trả `"đồng/xe"`; loại khác trả `"đồng/m²"`. Hai đơn vị của tàu **không** đưa vào danh sách regex bôi đậm đơn vị của Word.
- Đọc tiền bằng chữ theo USD: lấy bản VND rồi thay hậu tố `" đồng"` bằng `" đô la Mỹ"`. Ví dụ `0` đọc là `"Không đô la Mỹ"`.

**Nhãn yếu tố điều chỉnh mặc định của tàu** (8 nhãn, đúng thứ tự): `Năm đóng`, `Nơi đóng`, `Ngoại quan tổng thể`, `Khả năng vận hành`, `Trọng tải toàn phần`, `Chất lượng còn lại`, `Công suất (CV)`, `Cấp phương tiện`.

Đồng bộ từ dòng so sánh (type 0..3) sang `c_value*`:

| Nhãn | Nguồn | Định dạng |
|---|---|---|
| Năm đóng | `shipBuildYear` | `String.valueOf` |
| Nơi đóng | `shipBuildPlace` | trim, rỗng thành null |
| Trọng tải toàn phần | `shipDeadweight` | số kiểu châu Âu, 2 chữ số thập phân |
| Công suất (CV) | `shipMainEnginePowerHp` | như Trọng tải |
| Chất lượng còn lại | `shipRemainingQualityPct` | như Trọng tải; CAR vẫn dùng `carRemainingQualityPct` |
| Cấp phương tiện | `shipClass` | trim |

`Ngoại quan tổng thể` và `Khả năng vận hành` không đồng bộ.

### API

Mọi API của màn nằm dưới `/api/appraisal-records/{profileId}/reports/{reportId}` và kiểm tra bằng `ReportEditScope` như bộ edit-tab.

| API | Hợp đồng |
|---|---|
| `GET /groups/{groupId}/ship-description` | Trả `ShipDescriptionData`. Nhóm không phải SHIP thì 400 (`BusinessException`). |
| `PUT /groups/{groupId}/ship-description` | Body `ShipDescriptionInput`. Trả `ApiResponse.ok(ShipDescriptionData, "Lưu mô tả tàu thủy thành công")`. Ghi đè **mọi** field, kể cả null. Hai danh sách JSON chỉ ghi khi request khác null. |
| Ảnh | Dùng lại API ảnh theo nhóm sẵn có (`AppraisalImageController`). Nhóm SHIP thì owner là `AppraisalShipDescription`, thư mục S3 `ship`. |
| Pháp lý | DTO pháp lý thêm `shipInspectionCertificateNumber`, `shipInspectionIssuedPlace`, `shipInspectionIssuedDate`, `shipInspectionExpiryDate` (ISO), `shipInspectionForWhom`. Các trường GCN đăng ký dùng chung field sẵn có: `certificateNumber`, `cnIssuedPlace`, `gcnNoIssuedDate`, `forWhom`. |
| So sánh | Dòng so sánh thêm `shipBuildYear` (int), `shipBuildPlace`, `shipUsagePurpose`, `shipClass`, `shipRemainingQualityPct` (0–100), `shipDeadweight`, `shipDimension`, `shipMainEnginePowerHp`, `shipForSalePriceUsd`. `shipForSalePriceUsd` có trong DTO, không hiện trên UI, được giữ nguyên khi lưu. GET so sánh trả thêm `currencyCode` (đã resolve) của nhóm. |
| Kho TSSS | Mở rộng API tìm kho đang có (đang tìm theo tên xe, `CompareInformationController`) để nhận `assetType=4` và `keyword`. Tìm `address LIKE %keyword%` (chủng loại), `asset_type=4`, chưa xóa, sắp `updated DESC`, phân trang. DTO `ShipCompareInformationItem` gồm `id, address, source, contact, shipBuildYear, shipBuildPlace, shipUsagePurpose, shipClass, shipRemainingQualityPct, shipDeadweight, shipDimension, shipMainEnginePowerHp, currencyCode, forSalePrice, tradePrice, estimatePrice, updated`. |
| Bảng điều chỉnh | Dữ liệu thêm `currencyUnit` (`currencyUnitLabel`). `priceUnit` của tàu theo `regulatoryPriceUnit`. |
| Kết quả thẩm định | `ship { amount, exchangeRate, vndAmount, roundedAmount }` (khóa JSON `ship`, theo khóa `car` đang có; record `ShipDetail`), kèm `currencyCode`, `exchangeRate`, `currencyNote` của nhóm. PUT nhận thêm `currencyCode`, `exchangeRate`, `currencyNote` (mô tả bên dưới). |

`ShipDescriptionInput` và `ShipDescriptionData`:
- Tên field copy nguyên từ L `entity/dto/request/UpdateAppraisalShipDescriptionRequest.java` @ `82193fe5`.
- Decimal và số nguyên là number. Các field mà DB là TEXT là string. Boolean giữ nguyên, ngày theo ISO `YYYY-MM-DD`.
- `conditionDetails: { itemName, conditionDescription }[]`.
- `qualityEvaluationDetails: { itemNames: string[], importanceWeight, standardValue, rating, relativeValue }[]`, không có tiểu mục.
- `ShipDescriptionData` = Input + `id` + `remainingQualityPct` (server tính, có thể null).

PUT kết quả thẩm định, ba trường mới:

| Trường | Ràng buộc |
|---|---|
| `currencyCode` | `VND` hoặc `USD` |
| `exchangeRate` | 1..999.999.999 |
| `currencyNote` | tối đa 1000 ký tự |

- Ba trường chỉ có hiệu lực với nhóm SHIP. Nhóm khác thì bỏ qua và giữ tổng do client gửi.
- Với SHIP:
  - `currencyCode` null thì giữ tiền tệ cũ.
  - VND thì rate = 1.
  - USD mà rate null thì giữ rate cũ.
  - note được trim; rỗng thành null.
  - Server **tính lại** `roundingMoney`/`totalMoneyAppraisal`/bằng chữ (luôn theo VNĐ).

Cách tính `ship` (ShipDetail):
- `amount = avgPriceGuide`, tính theo tiền tệ của nhóm và không làm tròn.
- `vndAmount = Math.multiplyExact(amount, rate)`. Tràn số thì trả 400: "Thành tiền quy đổi sang VNĐ quá lớn, vui lòng kiểm tra lại tỷ lệ chuyển đổi".
- `roundedAmount = vndAmount / base` làm tròn HALF_UP rồi nhân lại `base`, với `base` = 1.000 hoặc 1.000.000 theo kiểu làm tròn.

### Quy ước tên dùng chung

**Backend (`N`)**

| Thành phần | Tên |
|---|---|
| Entity | `model/entity/AppraisalShipDescription` |
| Repository | `repository/AppraisalShipDescriptionRepository` (`Optional<…> findByAppraisalAssetGroupId(Long)`) |
| Converter | `model/converter/ShipConditionDetailsConverter`, `ShipQualityEvaluationConverter` |
| Record | `model/dto/ShipConditionDetail`, `ShipQualityEvaluationItem` |
| Lớp thuần | `service/ship/ShipQuality` (`BigDecimal remainingQualityPct(List<ShipQualityEvaluationItem>)`, `String formatDimension(BigDecimal lmax, BigDecimal bmax)`, `List<ShipConditionDetail> defaultConditionDetails()`) |
| Tiền tệ | `common/util/CurrencyCodes` |

- Controller và service mới: `ShipDescriptionController`, `ShipDescriptionService`.
- Word: `common/util/export/word/appraisal/asset/ShipAssetWriter` và gói `…/asset/ship/*`.
- Phương thức entity do B01 tạo, các task sau dùng (chữ ký chính xác ở khối Interfaces của `01-be-foundation.md`):
  - `AppraisalGroupCompare.applyShipForm(ShipCompareFields f, long trade, long estimate)`, `applyShipSync(AppraisalShipDescription, String address, BigDecimal remainingQualityPct)`, `clearShipFields()`
  - `AppraisalReportAsset.applyShipLegal(ShipLegalFields)`
  - `AppraisalAssetGroup.applyCurrency(String code, Long rate, String note)`, `resetCurrency()`
  - `AppraisalShipDescription.empty(reportId, groupId, now)`, `resetToDefault()`, `softDelete()`, `revive()`
  - `AppraisalImage.ofShip(…)`
- `AppraisalShipDescription.apply(ShipDescriptionInput)` do **B03** tạo, vì `ShipDescriptionInput` là DTO của B03.

## Quyết định của controller (chốt mâu thuẫn giữa các file task — ưu tiên hơn nội dung file task)

1. **Khóa JSON kết quả tàu là `ship`** (record `ShipDetail`), theo khóa `car`/`CarDetail` đang có. Không dùng `shipDetail`. Áp dụng cho B06, F01, F11. DTO Word `EvaluationResultDetailDto` cũng dùng `ship`.
2. **Phương thức entity nhận record trung gian** `ShipCompareFields` / `ShipLegalFields` do B01 tạo. B04 và B05 tự dựng record từ DTO của mình, **không** viết lại thân `applyShipLegal`/`applyShipForm`. `applyShipForm` ghi `marketPrice = 0`, nên B05 gọi nó sau dòng đặt `marketPrice` trong `applyForm`.
3. **Hằng nhãn yếu tố tàu** do B01 tạo trong `ControlCompareFactorLabels` (tên tiếng Anh: `BUILD_YEAR`, `BUILD_PLACE`, `OVERALL_APPEARANCE`, `OPERABILITY`, `DEADWEIGHT`, `POWER_HP`, `VESSEL_CLASS`; "Chất lượng còn lại" dùng hằng sẵn có). B05 và B07 chỉ dùng, không thêm.
4. **Nới cận cơ sở thẩm định 3→4** (`ReportContentMerge`, `AppraisalReportFormService`) do **B01** làm. B07 bỏ phần này, chỉ giữ `ReportLegalDocumentValidator.isKnownType`.
5. **Chặn TSSS khác tiền tệ** do **B05** làm trong `GroupCompareService.save`, chạy trước `writeRow` và `khoSync` (vì `CompareInformationSync.select` clone ảnh ngay trong vòng lặp). B10 không làm việc này.
6. **Quyền tick đồng bộ kho (thêm BGD)** nằm ở BE (`GroupCompareController`, cờ `canSyncToKho`), do **B10** làm. FE (F08) chỉ đổi tooltip.
7. **Tìm kho tàu** dùng chung endpoint tìm kiếm hiện có của `CompareInformationController`, rẽ theo `assetType` (kiểu trả `Page<?>`). FE gọi qua `searchShipCompareInformations(keyword, page, size = 20, signal?)`. Trường `updated` trả theo ISO `YYYY-MM-DD`, FE tự định dạng `dd/MM/yyyy`.
8. **Xóa nhóm**: chỉ xóa mềm mô tả tàu, **không xóa ảnh**, giống đất/xe, để khôi phục nhóm lấy lại được cả ảnh. Đổi loại (rời SHIP) mới xóa ảnh như legacy.
9. **GET mô tả tàu khi chưa có bản ghi** trả 404 (`BusinessException`), như mô tả xe. FE hiển thị trạng thái lỗi, không tự tạo.
10. **`mergeLegalFieldsFrom`** thêm 5 trường `shipInspection*` do **B04** làm. B07 chỉ dùng.
11. **`ControlFactorDefaults`**: thêm `case SHIP`. Nhánh `default` giữ nguyên như hiện tại.
12. **Hiển thị CLCL**: FE và Word hiển thị tổng thật (kể cả ngoài 0–100, không có dòng tính được thì FE hiện `0,00%`). BE `ShipQuality.remainingQualityPct` trả null ngoài 0–100, nên không đồng bộ sang so sánh. Hành vi này giống legacy. Bộ số liệu Q1–Q9 trong `11-fe-ship-description.md` là bộ dùng chung cho test `ShipQualityTest` (B01) và `ship-quality.spec.ts` (F03).
13. **F01** phải khai báo 5 field `shipInspection*` là optional trong kiểu legal input, và cập nhật mọi fixture test FE có kiểu bị mở rộng (`compare-fixtures`, `panel-fixtures`, `results/__tests__/fixtures`...), để các task F2 không bị chặn type-check.
14. **Hằng ảnh FE**: `IMAGE_TYPE.SHIP_FRONT … SHIP_ENGINE_SYSTEM` (21–30), trùng tên BE.
15. **Ngoài phạm vi**: xuất phiếu TSSS (Word/Excel) cho tàu; báo cáo VCB với nhóm tàu (sau B06 có thể tính được tàu; nếu cần chặn thì làm riêng); sai khác cũ "VND" và "VND/m²" ở bảng điều chỉnh đất/căn hộ.
16. **Lệnh test FE khi chạy tay**: `npm run test:unit -- --run` (mặc định chạy chế độ watch).
17. **`git fetch` lỗi chứng chỉ SSL** trên máy này: dùng `git -c http.sslBackend=schannel fetch …`.
18. **B09 sửa thêm writer/section cũ** (thêm hook vào `AssetTypeWriter`): chấp nhận. Writer cũ trả giá trị cũ, nên output loại 1–3 không đổi. Ở Z01 phải chạy parity test (`-Dparity=true`) một lần.
19. **`shipForSalePriceUsd` khi request không gửi**: B05 dựng `ShipCompareFields` bằng giá trị hiện có của dòng, nên giá trị được giữ nguyên. `applyShipForm` (B01) vẫn ghi thẳng giá trị nhận vào. `ShipConditionDetail.conditionDescription` không giới hạn độ dài, giống L.
20. **Các mục "Vấn đề cần controller quyết định" còn mở trong file task** (chưa đánh dấu "Đã chốt") thì **chấp nhận phương án đề xuất của chính file task đó**. Implementer làm theo phương án ấy. Nếu code thật cho thấy phương án không khả thi thì dừng và báo lại.

**Frontend (`F`)**

| Thành phần | Tên |
|---|---|
| API client | `api/report-edit/ship-description.ts` (`getShipDescription`, `saveShipDescription`, types `ShipDescriptionData`, `ShipDescriptionInput`, `ShipQualityItem`, `ShipConditionItem`) |
| Hàm thuần | `lib/report-edit/currency.ts`, `ship-quality.ts`, `ship-description-fields.ts` (test ở `lib/report-edit/__tests__/`) |
| Component | `components/appraisal-report/edit/description/ShipDescriptionTab.vue` + `description/ship/*`, `compare/ShipCompareTab.vue`, `compare/ShipCompareSearchModal.vue`, `legal/LegalShipAssetForm.vue` |

- Props chung của tab: `{ profileId: number; reportId: number; group: ReportWorkspaceGroup }`.

## Global Constraints

- Subagent chỉ dùng `model: "sonnet"`. Mỗi task phải qua spec review và code review.
- Tuân thủ `backend/CLAUDE.md` và `backend/.claude/rules/*.md` (đọc trước task BE đầu tiên):
  - Dùng JPA trước; native SQL phải có comment lý do.
  - Entity không dùng `@Data`. `@ManyToOne`/`@OneToOne` để LAZY.
  - Service đặt `@Transactional`.
  - Lỗi trả qua `BusinessException(ErrorCode, "…")`.
  - Controller không inject repository.
  - Thêm endpoint thì phải có đủ test controller, service và repository.
  - Cập nhật bảng helper trong `backend/CLAUDE.md` khi thêm helper ở `common/`.
- Tuân thủ `frontend/CLAUDE.md` và `frontend/DESIGN_SYSTEM.md`:
  - Tab tối đa 2 cấp.
  - Dùng `useRouteTab`, `AppModal`, `useConfirm`, `useToast`, `components/form/*`, vee-validate + zod.
  - Chỉ dùng token màu semantic. Không dùng control HTML native.
- **DB dùng chung với legacy, legacy vẫn đọc và ghi**:
  - Không tạo Flyway migration.
  - Dữ liệu ghi phải đúng định dạng legacy đang ghi: tên khóa JSON giống DTO của L (copy `@JsonProperty` hoặc tên field), `delete_flag`/`deleted` đúng kiểu từng bảng.
  - Converter JSON đọc lỗi thì trả rỗng. Record JSON mới có `@JsonIgnoreProperties(ignoreUnknown = true)`.
  - Thấy thiếu cột hoặc bảng thì dừng và báo.
- Entity hiện có chỉ có `@Getter`. Thêm cột còn thiếu bằng `@Column(name=…)` khớp schema. Thay đổi giá trị qua phương thức nghiệp vụ, không đặt `@Setter` cho cả lớp.
- API: GET trả DTO trực tiếp. PUT/POST/DELETE trả `ApiResponse.ok(data, "…thành công")`. Số trong JSON là số; ngày theo ISO.
- Nhãn, comment, Javadoc và message viết tiếng Việt có dấu. Javadoc của controller/service ghi rõ lớp hoặc màn cũ được port và các điểm "Khác hệ thống cũ" liên quan.
- Kiểm tra:
  - BE: `cd backend && ./mvnw -q test -Dtest=<Tên>`. Toàn bộ là `./mvnw test`; test repository cần MySQL theo `backend/.env.example`.
  - FE: `cd frontend && npm run lint && npm run type-check && npm run test:unit`. Chạy type-check **sau** lint.
- **Không sửa** exporter Word cũ `N/common/util/export/word/AppraisalReportWordExporter.java` (bản golden). Thêm SHIP vào Word chỉ qua `AssetTypeWriter` + `AssetTypeWriters`, không thêm `if (assetType == …)` vào section (theo `backend/CLAUDE.md`).
- Ngoài phạm vi:
  - nhánh `report/ship_market_overview`;
  - role "Báo cáo Ban lãnh đạo", báo cáo công nợ, báo cáo kê khai (có trong diff nhưng không thuộc tàu);
  - tab Xem trước.

### Khác hệ thống cũ (cố ý) — phải ghi vào Javadoc nơi liên quan

1. **Xóa nhóm thì xóa mềm mô tả tàu** (giữ ảnh, như đất/xe). L chỉ xử lý mô tả tàu khi đổi loại và khi khôi phục, nên bỏ sót trường hợp xóa nhóm.
2. **PUT mô tả tàu trả `ShipDescriptionData`.** L trả DTO để patch tại chỗ tab So sánh; ở FE mới, tab So sánh bị hủy khi ẩn nên tự tải lại khi mở.
3. **Lỗi trả 400 qua `BusinessException`.** Áp dụng cho CLCL ngoài 0–100, TSSS khác tiền tệ, tràn số khi quy đổi, và validate. L dùng `IllegalArgumentException`, có chỗ trả 500 với thông báo chung.
4. **Bảng điều chỉnh trong Word cho tàu** có dòng A là "Giá bán", không có dòng "Giá thị trường (giá trước điều chỉnh)" và "Giá quy đổi về đơn vị so sánh chuẩn", để khớp màn web. Word của L vẫn dùng nhãn của đất.
5. **JSON API thay cho fragment HTML.** Đổi tiền tệ ở tab Kết quả thì tab So sánh và Bảng điều chỉnh tải lại khi mở (thay cho `data-loaded='false'`).

## Review Focus

1. **Không còn `else` ngầm hiểu là CAR**:
   - BE: `AppraisalResultCalculator`, `ControlFactorDefaults` (`default` đang là LAND), `LandDescriptionService.requireLandGroup`, `ReportLegalService`, `AppraisalImageService.isCar`.
   - FE: `GroupWorkspace.byAssetType`, `AssetResultTable`, `PanelSummaryRows`, `compare-common.ts`.
   - Phải có test chứng minh nhóm SHIP đi đúng nhánh.
2. **Tiền tệ**:
   - `resolveCurrencyCode` ép loại khác về VND.
   - Lưu kết quả với USD thì tổng tính lại theo rate mới.
   - VND thì rate = 1.
   - Nhóm không phải tàu bỏ qua ba trường tiền tệ.
   - Ca kiểm thử: port đủ 9 case của `AppraisalResultServiceSaveCurrencyTest` (B06) và 5 case của `CurrencyCodeTest` (B01).
3. **CLCL**:
   - Công thức BE (`ShipQuality`), FE (`ship-quality.ts`) và Word phải cho cùng kết quả trên cùng bộ dữ liệu, kể cả (4) làm tròn 1 chữ số chỉ để hiển thị, còn (5) dùng giá trị chưa làm tròn.
   - Tổng ngoài 0–100 thì không ghi sang so sánh.
4. **Vòng đời**:
   - Ô tô sang Tàu, Tàu sang Đất, Đất sang Tàu khi đã có bản ghi cũ bị xóa mềm (dùng lại id, không vi phạm unique).
   - Xóa nhóm và khôi phục nhóm.
   - Ca kiểm thử: TC-10b/c/d của L và ca xóa nhóm (B07).
5. **Kho TSSS**:
   - Chọn TSSS khác tiền tệ thì 400 **trước** mọi side-effect ảnh/S3.
   - Đồng bộ lên kho ghi `currencyCode` đã resolve và không ghi `shipForSalePriceUsd`.
