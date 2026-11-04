# 10 — FE nền tảng loại SHIP (F01, F02)

> Đọc `00-index.md` (Hợp đồng API, quy ước tên, Global Constraints) trước. Subagent chỉ dùng `model: "sonnet"`, mỗi task có spec review rồi code review.
>
> Làm việc trong submodule `frontend/`, nhánh `feat/ship-asset-type`. Ký hiệu: **F** = `frontend/src/`, **L** = `td-project-legacy` @ `82193fe5` (đọc bằng `git -C td-project-legacy show 82193fe5:<path>`, không checkout), **LT** = `src/main/resources/templates/` của L.
>
> Số dòng của code mới đo trên `origin/develop` = `6ff0110`. **Luôn grep lại** (`git -C frontend grep -n <mẫu> -- src`) trước khi sửa.
>
> Lệnh kiểm tra cuối task: `cd frontend && npm run lint && npm run type-check && npm run test:unit`. Lưu ý `vitest` mặc định chạy chế độ watch, nên khi chạy tay một file dùng `npx vitest run <đường dẫn>` hoặc `npm run test:unit -- --run`. `npm run lint` tự sửa file, chạy xong phải xem `git -C frontend status` rồi mới commit.

## Phân chia file giữa F01 và F02 (cùng đợt F1, phải rời nhau)

| | F01 | F02 |
|---|---|---|
| `lib/report-edit/*`, `api/report-edit/*` | tất cả, **trừ** `api/report-edit/basic.ts` | chỉ `api/report-edit/basic.ts` |
| `lib/appraisal-report-form.ts` (+spec) | không đụng | sửa |
| `components/appraisal-report/edit/**` | `GroupWorkspace.vue`, 2 vỏ tab, spec và fixture test | không đụng |
| `components/appraisal-report/*.vue` (ngoài `edit/`) | không đụng | `AppraisalReportBasicForm.vue`, spec của `ReportConfigModal`, `ReportLegalDocumentList` |
| `views/__tests__/*` | không đụng | `AppraisalReportCreateView.spec.ts` |

Hai task không chung file nào. `api/report-edit/basic.ts` chỉ F02 sửa; F01 **không** import `ValuationBasisAssetType` mà F02 sẽ thêm vào đó.

---

## Task F01 — Nền FE: hằng số, kiểu, API client, tabs, vỏ tab tàu, tiền tệ

### Hành vi hệ thống cũ

Spec là L tại `82193fe5`.

- **Tab của nhóm tàu**: 5 tab theo thứ tự Pháp lý, Mô tả tàu thủy, Thông tin so sánh, Bảng điều chỉnh, Kết quả thẩm định; "Mô tả CTXD" chỉ có ở đất. Nhãn tab `land` là "Mô tả phương tiện" cho CAR, **"Mô tả tàu thủy"** cho SHIP, còn lại "Mô tả đất" (`LT demo/demo-report-step-2.html:167-176`). Tab mặc định mở là tab `land` (mô tả).
- **Ảnh tàu** mã 21–30 và mục "Khác" mã 10 (`L common/ReportConstants.java:52` trở đi: `SHIP_FRONT=21` … `SHIP_ENGINE_SYSTEM=30`; nhãn ở `LT demo/demo-report-step-2.html` modal thêm ảnh, quanh dòng 349). Nhãn: 21 Mặt trước tàu, 22 Đầu tàu, 23 Đuôi tàu, 24 Mạn tàu, 25 Sàn tàu, 26 Số IMO/Số kiểm soát, 27 Hệ thống neo, 28 Phòng lái, phòng nghỉ, 29 Khoang hàng, 30 Hệ thống máy của động cơ. (Màn legacy ghi "Phòng lái/phòng nghỉ"; FE mới theo `00-index.md`: "Phòng lái, phòng nghỉ".)
- **Tiền tệ** (`L common/CommonConstants.java:446-462`):
  - `resolveCurrencyCode(assetType, code)` chỉ trả `USD` khi `assetType == SHIP` và `code` bằng "USD" không phân biệt hoa thường; mọi trường hợp khác (loại khác, null, mã lạ) trả `VND`.
  - `currencyUnitLabel(code)`: `"USD"` nếu code đúng `"USD"` (so sánh chính xác), ngược lại `"VND"`.
  - `currencyMoneyLabel(code)`: `"USD"` nếu code đúng `"USD"`, ngược lại `"đồng"`.
  - Test BE tương ứng `CurrencyCodeTest` (5 case, `00-index.md` Review Focus). Case `readMoneyUsd` chỉ có ở BE (FE nhận số tiền bằng chữ từ server). FE port 4 case còn lại, gồm `regulatoryUnitForShipHasNoSquareMeter` (hàm `regulatoryPriceUnit`, F09 cần), và thêm 2 case riêng của FE (xem Step 3).
- **DTO mô tả tàu**: `L entity/dto/request/UpdateAppraisalShipDescriptionRequest.java` (153 field phẳng, hai danh sách `conditionDetails`, `qualityEvaluationDetails`), `L entity/dto/ShipConditionDetail.java`, `L entity/dto/ShipQualityEvaluationItem.java`. Quy ước kiểu: DB TEXT là `string`, `Integer`/`BigDecimal` là `number`, `Boolean` là `boolean`, `LocalDate` là chuỗi ISO `YYYY-MM-DD`. Chú thích độ dài/`Digits` trong Interfaces bên dưới sao từ `@Size`/`@Digits` của file này.
- **Đồng bộ giữa các tab (hệ thống cũ vá DOM, FE mới tải lại)**: sau khi lưu mô tả tàu, L patch cột "Tài sản thẩm định" của tab So sánh (`updateShipCompareSyncedFields`). FE mới: tab bị hủy khi ẩn nên không cần (Khác hệ thống cũ số 2, 5 trong `00-index.md`).
- **Nơi else-ngầm-là-CAR ở FE mới** (Review Focus 1): `GroupWorkspace.byAssetType`. Các chỗ còn lại (`AssetResultTable`, `PanelSummaryRows`, `compare-common.ts`, `LegalTab`, `NearbyCompareMapModal`, `adjustment-panel-calc.ts:95`) do F07–F11 xử lý; F01 chỉ đổi kiểu của chúng chứ không đổi hành vi, và **không** đụng file của task khác.

### Files

Đường dẫn trong bảng là dưới `frontend/`.

| Việc | File |
|---|---|
| Sửa | `src/lib/report-edit/tabs.ts` |
| Sửa test | `src/lib/report-edit/__tests__/tabs.spec.ts` |
| Tạo | `src/lib/report-edit/currency.ts` |
| Tạo test | `src/lib/report-edit/__tests__/currency.spec.ts` |
| Sửa | `src/lib/report-edit/image-types.ts` |
| Tạo test | `src/lib/report-edit/__tests__/image-types.spec.ts` |
| Sửa (chỉ đổi kiểu `assetType`) | `src/lib/report-edit/adjustment-panel-calc.ts` |
| Tạo | `src/api/report-edit/ship-description.ts` |
| Tạo test | `src/api/__tests__/ship-description.spec.ts` |
| Sửa | `src/api/report-edit/workspace.ts`, `compare.ts`, `legal.ts`, `panel.ts`, `land-description.ts`, `evaluation-result.ts`, `compare-informations.ts`, `images.ts` (chỉ comment) |
| Tạo test | `src/api/__tests__/compare-informations.spec.ts` |
| Sửa | `src/components/appraisal-report/edit/GroupWorkspace.vue` |
| Sửa test | `src/components/appraisal-report/edit/__tests__/GroupWorkspace.spec.ts` |
| Tạo | `src/components/appraisal-report/edit/description/ShipDescriptionTab.vue` (vỏ) |
| Tạo | `src/components/appraisal-report/edit/compare/ShipCompareTab.vue` (vỏ) |
| Tạo test | `src/components/appraisal-report/edit/description/__tests__/ShipDescriptionTab.spec.ts`, `src/components/appraisal-report/edit/compare/__tests__/ShipCompareTab.spec.ts` (F06, F08 sẽ ghi đè) |
| Sửa fixture/test cho khớp kiểu mới | `src/lib/report-edit/__tests__/compare-fixtures.ts`; `src/components/appraisal-report/edit/compare/__tests__/{Land,Apartment,Car}CompareTab.spec.ts` và `src/composables/__tests__/useCompareForm.spec.ts` (hàm `data()`); `src/components/appraisal-report/edit/panel/__tests__/panel-fixtures.ts`; `src/components/appraisal-report/edit/legal/__tests__/{LegalCarAssetForm,LegalTab}.spec.ts`; `src/components/appraisal-report/edit/results/__tests__/fixtures.ts`; và mọi file khác mà `type-check` báo lỗi vì kiểu mới |

Thay literal `1 | 2 | 3` bằng `AssetType` (grep trên `origin/develop`, đã đối chiếu):

| File | Dòng | Ghi chú |
|---|---|---|
| `src/api/report-edit/workspace.ts` | 14 | `ReportWorkspaceGroup.assetType` |
| `src/api/report-edit/compare.ts` | 128 | `GroupCompareData.assetType` |
| `src/api/report-edit/panel.ts` | 84 | `AdjustmentPanelData.assetType` |
| `src/api/report-edit/land-description.ts` | 139 | |
| `src/api/report-edit/legal.ts` | 41 | `LegalTabData.assetType` |
| `src/api/report-edit/evaluation-result.ts` | 72 | `AssetType | null` |
| `src/lib/report-edit/adjustment-panel-calc.ts` | 19 | chỉ kiểu; hằng `CAR = 3` (dòng 56) và `unit` dòng 95 **để nguyên** cho F09 |
| `src/components/appraisal-report/edit/__tests__/GroupWorkspace.spec.ts` | 49, 60 | `groupOf`, `mountGroup` |

**Không đổi** các literal `1 | 2 | 3` không phải loại tài sản: `composables/useCompareForm.ts:89,95,114` và `components/.../compare/KhoSyncCell.vue:11` (đó là số thứ tự cột so sánh 1..3), `compare/*CompareTab.vue` (`comparedIndex`), `src/lib/report-juridical.ts:11` (nhóm pháp lý).

### Interfaces

Các task FE sau (F03–F11) dựa vào chính xác các khai báo dưới đây.

**`src/lib/report-edit/tabs.ts`**

```ts
export const ASSET_TYPE = { LAND: 1, APARTMENT: 2, CAR: 3, SHIP: 4 } as const
export type AssetType = (typeof ASSET_TYPE)[keyof typeof ASSET_TYPE] // 1 | 2 | 3 | 4

export type GroupTab = 'legal' | 'land' | 'ctxd' | 'compare' | 'panel' | 'results' // không đổi
export const DEFAULT_GROUP_TAB: GroupTab = 'land' // không đổi
export function tabsOf(assetType: number): GroupTab[] // SHIP: ['legal','land','compare','panel','results']
export function tabLabel(tab: GroupTab, assetType: number): string // 'land': CAR "Mô tả phương tiện", SHIP "Mô tả tàu thủy", còn lại "Mô tả đất"
```

**`src/lib/report-edit/currency.ts`** (logic giống hệt BE `CurrencyCodes`)

```ts
export const CURRENCY_CODE = { VND: 'VND', USD: 'USD' } as const
export type CurrencyCode = (typeof CURRENCY_CODE)[keyof typeof CURRENCY_CODE] // 'VND' | 'USD'

/** Chỉ trả 'USD' khi assetType === ASSET_TYPE.SHIP và code là "usd" (không phân biệt hoa thường). Còn lại 'VND'. */
export function resolveCurrencyCode(assetType: number | null | undefined, code: string | null | undefined): CurrencyCode
/** Nhãn cột "Đơn vị tính": 'USD' nếu code === 'USD', còn lại 'VND'. Nhận mã đã resolve. */
export function currencyUnitLabel(code: string | null | undefined): CurrencyCode
/** Nhãn trong ngoặc của cột giá: 'USD' nếu code === 'USD', còn lại 'đồng'. */
export function currencyMoneyLabel(code: string | null | undefined): 'USD' | 'đồng'
/**
 * Đơn vị của câu "Nguyên tắc khống chế" (giống hệt BE `CurrencyCodes.regulatoryPriceUnit`):
 * SHIP -> 'USD' nếu resolveCurrencyCode(assetType, code) === 'USD', ngược lại 'đồng' (không '/tàu', không '/m²');
 * CAR -> 'đồng/xe'; loại khác (kể cả null/undefined) -> 'đồng/m²'. F09 dùng hàm này, không tự đặt đơn vị.
 */
export function regulatoryPriceUnit(assetType: number | null | undefined, code: string | null | undefined): string
```

**`src/lib/report-edit/image-types.ts`** (thêm)

```ts
// IMAGE_TYPE thêm:
SHIP_FRONT: 21, SHIP_BOW: 22, SHIP_STERN: 23, SHIP_HULL_SIDE: 24, SHIP_DECK: 25,
SHIP_IMO_NUMBER: 26, SHIP_ANCHOR_SYSTEM: 27, SHIP_WHEELHOUSE_CABIN: 28, SHIP_CARGO_HOLD: 29, SHIP_ENGINE_SYSTEM: 30,

// IMAGE_TYPE_LABELS thêm:
// 21 'Mặt trước tàu', 22 'Đầu tàu', 23 'Đuôi tàu', 24 'Mạn tàu', 25 'Sàn tàu', 26 'Số IMO/Số kiểm soát',
// 27 'Hệ thống neo', 28 'Phòng lái, phòng nghỉ', 29 'Khoang hàng', 30 'Hệ thống máy của động cơ'

/** 11 mục ảnh của tab Mô tả tàu, theo thứ tự hiển thị: 21..30 rồi 10 (Khác). */
export const SHIP_IMAGE_TYPES: readonly number[] // [21,22,23,24,25,26,27,28,29,30,10]
```

**`src/api/report-edit/ship-description.ts`** (tên field copy nguyên từ `UpdateAppraisalShipDescriptionRequest`; mọi field `| null`)

```ts
import { api } from '@/lib/api'
import { reportEditPath } from '@/api/report-edit/workspace'

/** Một hạng mục của "Tình trạng của tài sản thẩm định" (`ShipConditionDetail`). itemName ≤500 ký tự. */
export interface ShipConditionItem {
  itemName: string | null
  conditionDescription: string | null
}

/**
 * Một dòng của "Đánh giá chất lượng còn lại của tàu" (`ShipQualityEvaluationItem`). (4) = (3)/(2), (5) = (1)*(4)
 * và tổng không lưu, luôn tính lại. Mỗi tên ≤500 ký tự; các số ≥0, 8 nguyên + 2 lẻ; rating ≤50 ký tự
 * ('Tốt' | 'Khá' | 'Trung bình' | 'Kém' hoặc null).
 */
export interface ShipQualityItem {
  itemNames: string[]
  /** (1) Hệ số quan trọng */
  importanceWeight: number | null
  /** (2) Giá trị chuẩn so sánh */
  standardValue: number | null
  rating: string | null
  /** (3) Giá trị chỉ tiêu chất lượng tương đối */
  relativeValue: number | null
}

/** 153 field phẳng của mô tả tàu (cột của `appraisal_ship_description`). */
export interface ShipDescriptionFields {
  // I.1 Thông số chính
  shipName: string | null // ≤500 ký tự
  registrationNumber: string | null // ≤255 ký tự
  shipTypePurpose: string | null // ≤500 ký tự
  registryNumber: string | null // ≤255 ký tự
  ownerName: string | null // ≤500 ký tự
  buildYear: number | null // nguyên ≥0
  buildPlace: string | null // ≤1000 ký tự
  mooringLocation: string | null // ≤255 ký tự
  dimLength: number | null // thập phân ≥0, 8 nguyên + 2 lẻ
  dimBreadth: number | null // thập phân ≥0, 8 nguyên + 2 lẻ
  dimDepth: number | null // thập phân ≥0, 8 nguyên + 2 lẻ
  dimDraft: number | null // thập phân ≥0, 8 nguyên + 2 lẻ
  dimLengthMax: number | null // thập phân ≥0, 8 nguyên + 2 lẻ
  dimBreadthMax: number | null // thập phân ≥0, 8 nguyên + 2 lẻ
  freeBoard: number | null // thập phân ≥0, 8 nguyên + 2 lẻ
  material: string | null // ≤255 ký tự
  grossTonnage: number | null // nguyên ≥0
  shipClass: string | null // ≤255 ký tự
  operatingArea: string | null // ≤255 ký tự
  cargoCapacity: number | null // thập phân ≥0, 10 nguyên + 2 lẻ
  deadweight: number | null // thập phân ≥0, 10 nguyên + 2 lẻ
  operationNote: string | null // ≤5000 ký tự

  // I.2 Thiết kế phương tiện
  designSymbol: string | null // ≤255 ký tự
  designYear: number | null // nguyên ≥0
  designCertificateNumber: string | null // ≤255 ký tự
  appraisalNumber: string | null // ≤255 ký tự
  conversionYearPlace: string | null // ≤500 ký tự

  // I.3 Thiết bị neo
  anchorBowType: string | null // ≤100 ký tự
  anchorBowQuantity: number | null // nguyên ≥0
  anchorBowTotalWeightKg: number | null // thập phân ≥0, 10 nguyên + 2 lẻ
  anchorSternType: string | null // ≤100 ký tự
  anchorSternQuantity: number | null // nguyên ≥0
  anchorSternTotalWeightKg: string | null // ≤1000 ký tự (TEXT, không phải số)
  anchorChainCable: string | null // ≤255 ký tự
  anchorWindlassType: string | null // ≤100 ký tự

  // I.4 Thiết bị lái
  steeringRudderType: string | null // ≤100 ký tự
  steeringRudderQuantity: number | null // nguyên ≥0
  steeringRudderTotalAreaM2: number | null // thập phân ≥0, 10 nguyên + 2 lẻ
  steeringMainGearType: string | null // ≤100 ký tự
  steeringMainGearTorqueTm: number | null // thập phân ≥0, 10 nguyên + 2 lẻ
  steeringBackupGearType: string | null // ≤100 ký tự
  steeringBackupGearTorqueTm: number | null // thập phân ≥0, 10 nguyên + 2 lẻ

  // I.5 Thiết bị chằng buộc
  mooringRopeQuantity: number | null // nguyên ≥0
  mooringRopeTotalLengthM: number | null // thập phân ≥0, 10 nguyên + 2 lẻ
  mooringBollardType: string | null // ≤100 ký tự
  mooringBollardQuantity: string | null // ≤1000 ký tự (TEXT)

  // I.6 Thiết bị kéo
  towingBollardType: string | null // ≤1000 ký tự
  towingHookType: string | null // ≤1000 ký tự
  towingWinchType: string | null // ≤1000 ký tự
  towingRopeType: string | null // ≤1000 ký tự

  // I.7 Máy chính
  mainEngineQuantity: number | null // nguyên ≥0
  mainEngineBrand: string | null // ≤255 ký tự
  mainEngineSerialNumber: string | null // ≤255 ký tự
  mainEngineManufactureYear: number | null // nguyên ≥0
  mainEnginePowerHp: number | null // thập phân ≥0, 8 nguyên + 2 lẻ

  // I.8 Máy phụ
  auxEngineQuantity: number | null // nguyên ≥0
  auxEngineBrand: string | null // ≤255 ký tự
  auxEnginePowerHp: number | null // thập phân ≥0, 8 nguyên + 2 lẻ

  // I.9 Thiết bị điện
  electricGenerator: string | null // ≤100 ký tự
  electricGeneratorPowerKva: string | null // ≤100 ký tự
  electricGeneratorVoltage: string | null // ≤255 ký tự
  electricBatteryQuantity: number | null // nguyên ≥0
  electricBatteryVoltageV: number | null // thập phân ≥0, 8 nguyên + 2 lẻ
  electricBatteryTotalCapacity: string | null // ≤255 ký tự

  // I.10 Hệ thống khí nén
  compressedAirTankQuantity: string | null // ≤1000 ký tự
  compressedAirCompressorQuantity: string | null // ≤1000 ký tự

  // I.11 Thiết bị nâng
  liftingSerialNumber: string | null // ≤255 ký tự
  liftingMaxReachCapacity: string | null // ≤255 ký tự
  liftingMinReachCapacity: string | null // ≤255 ký tự

  // I.12 Trang bị chữa cháy
  firePumpQuantity: number | null // nguyên ≥0
  firePumpTotalFlowM3: number | null // thập phân ≥0, 10 nguyên + 2 lẻ
  fireGasType: string | null // ≤100 ký tự
  fireGasTotalWeightKg: number | null // thập phân ≥0, 10 nguyên + 2 lẻ
  fireFoamType: string | null // ≤100 ký tự
  fireFoamTotalVolumeL: string | null // ≤1000 ký tự
  fireFoamPusherQuantity: string | null // ≤1000 ký tự
  fireFoamPusherVolumeL: string | null // ≤1000 ký tự
  firePortableFoam: string | null // ≤100 ký tự
  fireCo2BottleQuantity: number | null // nguyên ≥0
  fireCo2BottleWeightKg: number | null // thập phân ≥0, 10 nguyên + 2 lẻ
  firePowderBottleQuantity: number | null // nguyên ≥0
  firePowderBottleWeightKg: number | null // thập phân ≥0, 10 nguyên + 2 lẻ
  fireFoamBottleQuantity: string | null // ≤1000 ký tự
  fireFoamBottleVolumeL: string | null // ≤1000 ký tự
  fireManualTools: boolean | null
  fireLayoutPlan: boolean | null

  // I.13 Trang bị cứu sinh
  lifesavingRingBuoy: string | null // ≤100 ký tự
  lifesavingLifeJacket: string | null // ≤100 ký tự
  lifesavingRaftBoatQuantity: number | null // nguyên ≥0
  lifesavingRaftBoatTotalCapacity: number | null // nguyên ≥0

  // I.14 Trang bị cứu đắm
  damageControlBilgePumpQuantity: number | null // nguyên ≥0
  damageControlBilgePumpTotalFlowM3: number | null // thập phân ≥0, 10 nguyên + 2 lẻ
  damageControlTools: string | null // ≤100 ký tự
  damageControlBilgeAlarm: string | null // ≤100 ký tự

  // I.15 Trang bị tín hiệu: đèn hiệu (16), âm hiệu, vật hiệu (10)
  signalLightSideRed: boolean | null
  signalLightSideGreen: boolean | null
  signalLightSternWhite: boolean | null
  signalLightMastWhite: boolean | null
  signalLightMastGreen: boolean | null
  signalLightMastRed: boolean | null
  signalLightAnchor: boolean | null
  signalLightTowingRed: boolean | null
  signalLightTowingGreen: boolean | null
  signalLightYellow: boolean | null
  signalLightFlashWhite: boolean | null
  signalLightFlashYellow: boolean | null
  signalLightFlashRed: boolean | null
  signalLightOther: boolean | null
  signalLightHalfGreenRed: boolean | null
  signalLightSpare: boolean | null
  signalSound: string | null // ≤100 ký tự
  signalShapeBlackTriangle: boolean | null
  signalShapeWhiteTriangle: boolean | null
  signalShapeBlackSquare: boolean | null
  signalShapeBlackRectangle: boolean | null
  signalShapeBlackCircle: boolean | null
  signalShapeWhiteCircle: boolean | null
  signalShapeBlackDiamond: boolean | null
  signalShapeFlagSet: boolean | null
  signalShapeParachuteFlare: boolean | null
  signalShapeLineThrower: boolean | null

  // I.16 Trang bị chạy tàu (11 checkbox + 3 text)
  navigationBinoculars: boolean | null
  navigationClock: boolean | null
  navigationDepthSounder: boolean | null
  navigationOrderBell: boolean | null
  navigationInclinometer: boolean | null
  navigationDraftGauge: boolean | null
  navigationGps: boolean | null
  navigationCompass: boolean | null
  navigationCommandPa: boolean | null
  navigationRadar: boolean | null
  navigationRadio: string | null // ≤100 ký tự (text, không phải checkbox)
  navigationEpirb: boolean | null
  navigationAis: string | null // ≤100 ký tự (text)
  navigationSart: string | null // ≤100 ký tự (text)

  // I.17 Trang bị ngừa ô nhiễm
  pollutionOilyWaterSeparatorQuantity: string | null // ≤1000 ký tự
  pollutionOilyWaterSeparatorTotalOutput: string | null // ≤1000 ký tự
  pollutionOilyWaterTankQuantity: number | null // nguyên ≥0
  pollutionOilyWaterTankTotalVolumeM3: number | null // thập phân ≥0, 10 nguyên + 2 lẻ
  pollutionSewageTankQuantity: string | null // ≤1000 ký tự
  pollutionSewageTankTotalVolumeM3: string | null // ≤1000 ký tự
  pollutionSettlingTankQuantity: string | null // ≤1000 ký tự
  pollutionSettlingTankTotalVolumeM3: string | null // ≤1000 ký tự
  pollutionSludgeTankQuantity: string | null // ≤1000 ký tự
  pollutionSludgeTankTotalVolumeM3: string | null // ≤1000 ký tự
  pollutionStandardFlangeQuantity: string | null // ≤1000 ký tự
  pollutionStandardFlangePosition: string | null // ≤255 ký tự
  pollutionGarbageContainerQuantity: string | null // ≤1000 ký tự
  pollutionSewageTreatmentQuantity: string | null // ≤1000 ký tự
  pollutionSewageTreatmentOutputM3: string | null // ≤1000 ký tự

  // I.18 Thời hạn kiểm tra
  inspectionPeriodicDate: string | null // ISO YYYY-MM-DD
  inspectionDockDate: string | null // ISO YYYY-MM-DD
  inspectionAnnualDate: string | null // ISO YYYY-MM-DD
}

/**
 * Body PUT (`ShipDescriptionInput`). Ghi đè mọi field kể cả null. Hai danh sách chỉ được ghi khi khác null: gửi null
 * nghĩa là giữ nguyên danh sách đang lưu.
 */
export interface ShipDescriptionInput extends ShipDescriptionFields {
  conditionDetails: ShipConditionItem[] | null
  qualityEvaluationDetails: ShipQualityItem[] | null
}

/** Dữ liệu màn Mô tả tàu (`ShipDescriptionData`): Input cộng mã mô tả và chất lượng còn lại do server tính (có thể null). */
export interface ShipDescriptionData extends ShipDescriptionInput {
  id: number
  /** Tổng (5) của bảng chất lượng, server tính; null khi chưa có dòng nào hoặc tổng ngoài 0..100. */
  remainingQualityPct: number | null
}

export function getShipDescription(profileId: number, reportId: number, groupId: number, signal?: AbortSignal): Promise<ShipDescriptionData>
// GET  {reportEditPath}/groups/{groupId}/ship-description
export function saveShipDescription(profileId: number, reportId: number, groupId: number, input: ShipDescriptionInput): Promise<ShipDescriptionData>
// PUT  cùng đường dẫn; server trả ShipDescriptionData ("Lưu mô tả tàu thủy thành công")
```

**`src/api/report-edit/compare.ts`**

```ts
// GroupCompareRow thêm (đặt sau carNote):
shipBuildYear: number | null
shipBuildPlace: string | null
shipUsagePurpose: string | null
shipClass: string | null
/** 0..100 */
shipRemainingQualityPct: number | null
shipDeadweight: number | null
shipDimension: string | null
shipMainEnginePowerHp: number | null
/** Có trong DTO nhưng không hiện trên UI; GIỮ NGUYÊN giá trị khi lưu (nằm trong GroupCompareRowInput, không cho vào COMPARE_ROW_READ_ONLY_KEYS). */
shipForSalePriceUsd: number | null

// GroupCompareData: assetType: AssetType; thêm
/** Tiền tệ hiệu lực của nhóm (đã resolve): giá của cả 4 cột tính theo đơn vị này. */
currencyCode: CurrencyCode
```

`GroupCompareRowInput` tự có 9 field mới vì là `Omit<GroupCompareRow, …>`.

**`src/api/report-edit/legal.ts`**

```ts
// LegalAsset thêm (đặt sau carInspectionExpiryDate), ngày ISO YYYY-MM-DD:
shipInspectionCertificateNumber: string | null
shipInspectionIssuedPlace: string | null
shipInspectionIssuedDate: string | null
shipInspectionExpiryDate: string | null
shipInspectionForWhom: string | null
// Quyết định #13: trong LegalAssetInput 5 field shipInspection* là OPTIONAL (shipInspectionX?: string | null) để mã/test cũ (LegalCarAssetForm, fixtures) không bị chặn type-check:
//   type ShipInspectionKey = 'shipInspectionCertificateNumber' | 'shipInspectionIssuedPlace' | 'shipInspectionIssuedDate' | 'shipInspectionExpiryDate' | 'shipInspectionForWhom'
//   LegalAssetInput = Omit<LegalAsset, 'id' | 'files' | ShipInspectionKey> & Partial<Pick<LegalAsset, ShipInspectionKey>>
// LegalAsset (dữ liệu đọc từ server) vẫn khai báo 5 field là bắt buộc `string | null`. F07 là task duy nhất ghi 5 field này khi lưu.
// Giấy chứng nhận đăng ký của tàu dùng lại certificateNumber, cnIssuedPlace, gcnNoIssuedDate, forWhom (đã có).
// LegalTabData.assetType: AssetType
```

**`src/api/report-edit/panel.ts`**

```ts
// AdjustmentPanelData: assetType: AssetType; thêm
/** Đơn vị tính tiền của nhóm: 'VND' | 'USD' (loại khác tàu luôn 'VND'). */
currencyUnit: CurrencyCode
// regulatoryPriceUnit giữ nguyên: tàu nhận "đồng" hoặc "USD" từ server
```

**`src/api/report-edit/evaluation-result.ts`**

```ts
/** Kết quả của nhóm tàu (khóa `ship`, Quyết định #1): amount theo tiền tệ nhóm, không làm tròn; vndAmount = amount × rate; roundedAmount là VNĐ đã làm tròn. */
export interface ShipResultDetail {
  amount: number | null
  exchangeRate: number | null
  vndAmount: number | null
  roundedAmount: number | null
}

// EvaluationResult: assetType: AssetType | null; thêm
ship: ShipResultDetail | null
/** Tiền tệ của nhóm; có thể null ở bản ghi rỗng, khi đó dùng resolveCurrencyCode(assetType, null) = 'VND'. */
currencyCode: string | null
exchangeRate: number | null
currencyNote: string | null

// SaveEvaluationResultBody thêm (tuỳ chọn: chỉ nhóm SHIP có hiệu lực, loại khác server bỏ qua)
currencyCode?: CurrencyCode       // VND | USD
exchangeRate?: number             // 1..999.999.999 (VND thì gửi 1)
currencyNote?: string | null      // ≤1000 ký tự
```

**`src/api/report-edit/compare-informations.ts`**

```ts
/** Một dòng kết quả tìm kho tàu (`ShipCompareInformationItem` của backend). Giá theo `currencyCode`. */
export interface ShipCompareInformationItem {
  id: number
  address: string | null            // chủng loại
  source: string | null
  contact: string | null
  shipBuildYear: number | null
  shipBuildPlace: string | null
  shipUsagePurpose: string | null
  shipClass: string | null
  shipRemainingQualityPct: number | null
  shipDeadweight: number | null
  shipDimension: string | null
  shipMainEnginePowerHp: number | null
  currencyCode: CurrencyCode        // đã resolve
  forSalePrice: number | null
  tradePrice: number | null
  estimatePrice: number | null
  updated: string | null            // ISO ngày YYYY-MM-DD (Quyết định #7); UI tự định dạng dd/MM/yyyy
}

export const SHIP_KHO_PAGE_SIZE = 20 // như legacy (pageSize=20)

/** Tìm tàu trong kho theo chủng loại (`address LIKE %keyword%`), mới cập nhật trước. `page` đánh số từ 0. */
export function searchShipCompareInformations(
  keyword: string,
  page: number,
  size: number = SHIP_KHO_PAGE_SIZE,
  signal?: AbortSignal,
): Promise<PageData<ShipCompareInformationItem>>
// GET /compare-informations/search?keyword&assetType=4&page&size
// searchCarCompareInformations giữ chữ ký; đổi literal 3 thành ASSET_TYPE.CAR.
```

**`src/components/appraisal-report/edit/GroupWorkspace.vue`**

```ts
function byAssetType(land: Component, apartment: Component, car: Component, ship: Component): Component
// switch exhaustive trên AssetType, không còn nhánh else
// 'land'    -> byAssetType(LandDescriptionTab, ApartmentDescriptionTab, CarDescriptionTab, ShipDescriptionTab)
// 'compare' -> byAssetType(LandCompareTab, ApartmentCompareTab, CarCompareTab, ShipCompareTab)
```

**Vỏ tab** (F06 thay `description/ShipDescriptionTab.vue`, F08 thay `compare/ShipCompareTab.vue`; giữ nguyên tên file và props):

```ts
defineProps<{ profileId: number; reportId: number; group: ReportWorkspaceGroup }>()
```

Vỏ hiện trạng rỗng theo design system: icon `Ship` (`@lucide/vue`) `size-6 text-muted-foreground` cộng một dòng thông báo ngắn; không gọi API, không có state.

### Steps

- [ ] **Step 1: Tạo nhánh làm việc**

```bash
git -C frontend -c http.sslBackend=schannel fetch origin develop && git -C frontend checkout -b feat/ship-asset-type origin/develop
git -C frontend log --oneline -1   # mong đợi 6ff0110 hoặc mới hơn
cd frontend && npm ci && npm run type-check   # baseline phải sạch trước khi sửa
```

Nếu nhánh đã tồn tại (đợt trước tạo rồi) thì chỉ `checkout`. Baseline lỗi thì dừng và báo lại.

- [ ] **Step 2: `tabs.ts` (test trước)**

Sửa `src/lib/report-edit/__tests__/tabs.spec.ts`, thêm:

```ts
it('SHIP có 5 tab và không có ctxd', () => {
  expect(tabsOf(ASSET_TYPE.SHIP)).toEqual(['legal', 'land', 'compare', 'panel', 'results'])
})
it('SHIP nhãn Mô tả tàu thủy, CAR và đất giữ nguyên', () => {
  expect(tabLabel('land', ASSET_TYPE.SHIP)).toBe('Mô tả tàu thủy')
  expect(tabLabel('land', ASSET_TYPE.CAR)).toBe('Mô tả phương tiện')
  expect(tabLabel('land', ASSET_TYPE.LAND)).toBe('Mô tả đất')
})
it('ASSET_TYPE.SHIP là 4', () => {
  expect(ASSET_TYPE.SHIP).toBe(4)
})
```

Chạy `cd frontend && npx vitest run src/lib/report-edit/__tests__/tabs.spec.ts`: mong đợi FAIL (`ASSET_TYPE.SHIP` undefined). Sửa `tabs.ts` theo Interfaces: thêm `SHIP: 4`, đổi `tabLabel` sang `switch (assetType)` trong nhánh `land`, cập nhật Javadoc ("`land` là Mô tả đất; Ô tô: Mô tả phương tiện; Tàu: Mô tả tàu thủy"). `tabsOf` giữ nguyên logic nhưng ghi comment rằng SHIP thuộc nhánh "loại khác": 5 tab. Chạy lại: PASS.

- [ ] **Step 3: `currency.ts` (test trước)**

Tạo `src/lib/report-edit/__tests__/currency.spec.ts`:

```ts
import { describe, expect, it } from 'vitest'
import { ASSET_TYPE } from '../tabs'
import { CURRENCY_CODE, currencyMoneyLabel, currencyUnitLabel, regulatoryPriceUnit, resolveCurrencyCode } from '../currency'

describe('resolveCurrencyCode', () => {
  it('shipCanUseUsd_không_phân_biệt_hoa_thường', () => {
    expect(resolveCurrencyCode(ASSET_TYPE.SHIP, 'USD')).toBe('USD')
    expect(resolveCurrencyCode(ASSET_TYPE.SHIP, 'usd')).toBe('USD')
  })

  it('nonShipOrUnknownCodeFallsBackToVnd', () => {
    expect(resolveCurrencyCode(ASSET_TYPE.LAND, 'USD')).toBe('VND')
    expect(resolveCurrencyCode(ASSET_TYPE.CAR, 'USD')).toBe('VND')
    expect(resolveCurrencyCode(null, 'USD')).toBe('VND')
    expect(resolveCurrencyCode(undefined, 'USD')).toBe('VND')
    expect(resolveCurrencyCode(ASSET_TYPE.SHIP, null)).toBe('VND')
    expect(resolveCurrencyCode(ASSET_TYPE.SHIP, 'EUR')).toBe('VND')
  })
})

describe('nhãn tiền tệ', () => {
  it('labels', () => {
    expect(currencyUnitLabel('USD')).toBe('USD')
    expect(currencyUnitLabel('VND')).toBe('VND')
    expect(currencyMoneyLabel('USD')).toBe('USD')
    expect(currencyMoneyLabel('VND')).toBe('đồng')
  })

  it('mã_lạ_hoặc_trống_về_vnd_như_BE', () => {
    expect(currencyUnitLabel(null)).toBe('VND')
    expect(currencyUnitLabel('usd')).toBe('VND') // nhãn so khớp chính xác, mã phải được resolve trước
    expect(currencyMoneyLabel(undefined)).toBe('đồng')
    expect(currencyMoneyLabel('EUR')).toBe('đồng')
  })

  it('CURRENCY_CODE_chỉ_có_VND_và_USD', () => {
    expect(CURRENCY_CODE).toEqual({ VND: 'VND', USD: 'USD' })
  })
})

describe('regulatoryPriceUnit', () => {
  it('tàu_không_có_m2_hay_tàu', () => {
    expect(regulatoryPriceUnit(ASSET_TYPE.SHIP, 'USD')).toBe('USD')
    expect(regulatoryPriceUnit(ASSET_TYPE.SHIP, 'VND')).toBe('đồng')
    expect(regulatoryPriceUnit(ASSET_TYPE.SHIP, null)).toBe('đồng')
  })
  it('ô_tô_và_đất_giữ_nguyên_dù_mã_là_USD', () => {
    expect(regulatoryPriceUnit(ASSET_TYPE.CAR, 'USD')).toBe('đồng/xe')
    expect(regulatoryPriceUnit(ASSET_TYPE.LAND, 'USD')).toBe('đồng/m²')
    expect(regulatoryPriceUnit(ASSET_TYPE.APARTMENT, null)).toBe('đồng/m²')
    expect(regulatoryPriceUnit(null, undefined)).toBe('đồng/m²')
  })
})
```

Chạy: FAIL (module chưa có). Tạo `currency.ts` theo Interfaces (comment tiếng Việt nêu "giống hệt `CurrencyCodes` của BE và `CommonConstants.resolveCurrencyCode` của L:446"). Chạy lại: PASS.

- [ ] **Step 4: `image-types.ts` (test trước)**

Tạo `src/lib/report-edit/__tests__/image-types.spec.ts`:

```ts
import { describe, expect, it } from 'vitest'
import { IMAGE_TYPE, IMAGE_TYPE_LABELS, imageTypeLabel, SHIP_IMAGE_TYPES } from '../image-types'

describe('ảnh tàu', () => {
  it('mã_21_đến_30_đúng_hằng_của_legacy', () => {
    expect([
      IMAGE_TYPE.SHIP_FRONT, IMAGE_TYPE.SHIP_BOW, IMAGE_TYPE.SHIP_STERN, IMAGE_TYPE.SHIP_HULL_SIDE,
      IMAGE_TYPE.SHIP_DECK, IMAGE_TYPE.SHIP_IMO_NUMBER, IMAGE_TYPE.SHIP_ANCHOR_SYSTEM,
      IMAGE_TYPE.SHIP_WHEELHOUSE_CABIN, IMAGE_TYPE.SHIP_CARGO_HOLD, IMAGE_TYPE.SHIP_ENGINE_SYSTEM,
    ]).toEqual([21, 22, 23, 24, 25, 26, 27, 28, 29, 30])
  })

  it('nhãn_đủ_10_loại', () => {
    expect(imageTypeLabel(21)).toBe('Mặt trước tàu')
    expect(imageTypeLabel(26)).toBe('Số IMO/Số kiểm soát')
    expect(imageTypeLabel(28)).toBe('Phòng lái, phòng nghỉ')
    expect(imageTypeLabel(30)).toBe('Hệ thống máy của động cơ')
    for (let type = 21; type <= 30; type++) expect(IMAGE_TYPE_LABELS[type]).toBeTruthy()
  })

  it('SHIP_IMAGE_TYPES_có_11_mục_Khác_cuối', () => {
    expect(SHIP_IMAGE_TYPES).toEqual([21, 22, 23, 24, 25, 26, 27, 28, 29, 30, IMAGE_TYPE.OTHER])
  })
})
```

FAIL → thêm 10 hằng vào `IMAGE_TYPE`, 10 nhãn vào `IMAGE_TYPE_LABELS`, export `SHIP_IMAGE_TYPES` → PASS.

- [ ] **Step 5: Kiểu API (compare, legal, panel, evaluation-result, workspace, land-description, adjustment-panel-calc)**

Theo Interfaces và bảng "Thay literal". Thứ tự: sửa kiểu (tạo trước các import `AssetType` từ `@/lib/report-edit/tabs`, `CurrencyCode` từ `@/lib/report-edit/currency`), rồi chạy `cd frontend && npm run type-check`. Lỗi mong đợi là các object literal thiếu field mới ở fixture/spec. Sửa **chỉ thêm giá trị mặc định**, không đổi hành vi test:

- `compare-fixtures.ts` → `compareRow()` thêm 9 field `ship*: null`; `khoDetail()` không đổi.
- `data()` trong `Land/Apartment/CarCompareTab.spec.ts` và `useCompareForm.spec.ts` → thêm `currencyCode: 'VND'`.
- `panel-fixtures.ts` → `panelData()` thêm `currencyUnit: 'VND'`.
- `LegalCarAssetForm.spec.ts`, `LegalTab.spec.ts` (đối tượng `LegalAsset`) → thêm 5 field `shipInspection*: null` (đối tượng `LegalAssetInput` không phải sửa vì 5 field là optional; Quyết định #13).
- `results/__tests__/fixtures.ts` → `evaluationResult()` thêm `ship: null, currencyCode: 'VND', exchangeRate: 1, currencyNote: null`.
- File nào khác `type-check` báo thì xử lý tương tự, rồi **liệt kê vào thông điệp commit**.

Thêm test hồi quy kiểu hợp đồng vào `src/lib/report-edit/__tests__/compare-common.spec.ts`? Không: file đó thuộc F08/F10. Thay vào đó thêm vào `src/api/__tests__/compare-informations.spec.ts` ở Step 7.

Chạy `npm run type-check`: sạch. Chạy `npx vitest run src/lib src/composables src/components/appraisal-report/edit` để chắc không hồi quy: PASS.

Commit:

```bash
git -C frontend add -A src
git -C frontend commit -m "feat: thêm ASSET_TYPE.SHIP, tiền tệ, ảnh tàu và mở rộng kiểu API tab báo cáo"
```

- [ ] **Step 6: API client mô tả tàu (test trước)**

Tạo `src/api/__tests__/ship-description.spec.ts`:

```ts
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { api } from '@/lib/api'
import { getShipDescription, saveShipDescription, type ShipDescriptionInput } from '../report-edit/ship-description'

vi.mock('@/lib/api')
beforeEach(() => vi.resetAllMocks())

describe('ship-description api', () => {
  it('get_gọi_đúng_đường_dẫn_và_truyền_signal', async () => {
    const signal = new AbortController().signal
    await getShipDescription(5, 11, 21, signal)
    expect(api.get).toHaveBeenCalledWith('/appraisal-records/5/reports/11/groups/21/ship-description', { signal })
  })

  it('save_put_nguyên_body', async () => {
    const input = { shipName: 'Sà lan 01', conditionDetails: [], qualityEvaluationDetails: null } as unknown as ShipDescriptionInput
    await saveShipDescription(5, 11, 21, input)
    expect(api.put).toHaveBeenCalledWith('/appraisal-records/5/reports/11/groups/21/ship-description', input)
  })
})
```

FAIL (module chưa có) → tạo `ship-description.ts` đầy đủ 153 field + 4 interface + 2 hàm đúng Interfaces (chép khối kiểu từ mục Interfaces; doc-comment tiếng Việt, nêu nguồn `UpdateAppraisalShipDescriptionRequest`) → PASS. Cách chứng minh đủ field: thêm một test đếm không cần thiết thì bỏ qua; thay vào đó `npm run type-check` phải sạch và reviewer đối chiếu danh sách với `UpdateAppraisalShipDescriptionRequest` (đủ 153 field, không thừa, không thiếu).

Commit: `git -C frontend add -A src && git -C frontend commit -m "feat: API client mô tả tàu thủy"`.

- [ ] **Step 7: `compare-informations.ts` (test trước)**

Tạo `src/api/__tests__/compare-informations.spec.ts`:

```ts
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { api } from '@/lib/api'
import { searchCarCompareInformations, searchShipCompareInformations, SHIP_KHO_PAGE_SIZE } from '../report-edit/compare-informations'

vi.mock('@/lib/api')
beforeEach(() => vi.resetAllMocks())

describe('tìm kho so sánh', () => {
  it('tàu_gửi_assetType_4_keyword_page_và_size_mặc_định_20', async () => {
    const signal = new AbortController().signal
    await searchShipCompareInformations('sà lan', 2, undefined, signal)
    expect(SHIP_KHO_PAGE_SIZE).toBe(20)
    expect(api.get).toHaveBeenCalledWith('/compare-informations/search', {
      query: { keyword: 'sà lan', assetType: 4, page: 2, size: 20 },
      signal,
    })
  })

  it('tàu_nhận_size_tùy_chọn', async () => {
    await searchShipCompareInformations('tàu', 0, 50)
    expect(api.get).toHaveBeenCalledWith('/compare-informations/search', {
      query: { keyword: 'tàu', assetType: 4, page: 0, size: 50 },
      signal: undefined,
    })
  })

  it('ô_tô_vẫn_gửi_assetType_3', async () => {
    await searchCarCompareInformations('vios', 0, 10)
    expect(api.get).toHaveBeenCalledWith('/compare-informations/search', {
      query: { keyword: 'vios', assetType: 3, page: 0, size: 10 },
      signal: undefined,
    })
  })
})
```

FAIL (chưa có hàm tàu) → thêm `ShipCompareInformationItem`, `SHIP_KHO_PAGE_SIZE`, `searchShipCompareInformations`; đổi literal `3` ở `searchCarCompareInformations` thành `ASSET_TYPE.CAR`, sửa Javadoc "tìm ô tô/tàu"; import `ASSET_TYPE` từ `@/lib/report-edit/tabs` và `CurrencyCode` từ `@/lib/report-edit/currency` → PASS.

Ghi chú comment: endpoint `/compare-informations/search` trả kiểu khác nhau theo `assetType` (3: `CompareInformationDetail`, 4: `ShipCompareInformationItem`), Quyết định #7.

Cũng ở step này sửa comment của `listImages` trong `images.ts` thành "(đất/căn hộ: mô tả đất; ô tô: mô tả phương tiện; tàu: mô tả tàu)". Commit: `git -C frontend add -A src && git -C frontend commit -m "feat: API tìm tài sản so sánh tàu thủy trong kho"`.

- [ ] **Step 8: `GroupWorkspace` và hai vỏ tab (test trước)**

Sửa `GroupWorkspace.spec.ts`:

1. Đổi `groupOf`/`mountGroup` sang `AssetType` (import `type AssetType` từ `@/lib/report-edit/tabs`).
2. Thêm vào `STUBS`: `ShipDescriptionTab: plain('ShipDescriptionTab')`, `ShipCompareTab: plain('ShipCompareTab')`.
3. Thêm dòng vào `it.each`: `[4, 'Mô tả tàu thủy', 'ShipDescriptionTab', 'ShipCompareTab']`.
4. Thêm các test:

```ts
it('nhóm_tàu_có_5_tab_đúng_thứ_tự_và_mở_tab_mô_tả_tàu', async () => {
  const { wrapper } = await mountGroup(4)
  expect(wrapper.findAll('[role="tab"]').map((tab) => tab.text())).toEqual([
    'Pháp lý', 'Mô tả tàu thủy', 'Thông tin so sánh', 'Bảng điều chỉnh', 'Kết quả thẩm định',
  ])
  expect(tabButton(wrapper, 'Mô tả tàu thủy').attributes('data-state')).toBe('active')
})

it('nhóm_tàu_không_render_component_của_ô_tô_hay_đất', async () => {
  const { wrapper } = await mountGroup(4)
  expect(wrapper.text()).toContain('ShipDescriptionTab')
  for (const other of ['CarDescriptionTab', 'LandDescriptionTab', 'ApartmentDescriptionTab']) {
    expect(wrapper.text()).not.toContain(other)
  }
  await clickTab(wrapper, 'Thông tin so sánh')
  expect(wrapper.text()).toContain('ShipCompareTab')
  for (const other of ['CarCompareTab', 'LandCompareTab', 'ApartmentCompareTab']) {
    expect(wrapper.text()).not.toContain(other)
  }
})

it('nhóm_tàu_không_có_tab_ctxd', async () => {
  const { wrapper } = await mountGroup(4)
  expect(wrapper.findAll('[role="tab"]').map((tab) => tab.text())).not.toContain('Mô tả CTXD')
})
```

Tạo hai spec vỏ (mount thật, không stub):

```ts
// description/__tests__/ShipDescriptionTab.spec.ts
import { describe, expect, it } from 'vitest'
import { mount } from '@vue/test-utils'
import type { ReportWorkspaceGroup } from '@/api/report-edit/workspace'
import ShipDescriptionTab from '../ShipDescriptionTab.vue'

const group: ReportWorkspaceGroup = { id: 21, groupIndex: 1, label: 'Tàu 1', type: 'SINGLE', assetType: 4, landValuationMethod: null }

describe('ShipDescriptionTab (vỏ)', () => {
  it('hiển_thị_thông_báo_đang_xây_dựng', () => {
    const wrapper = mount(ShipDescriptionTab, { props: { profileId: 5, reportId: 11, group } })
    expect(wrapper.text()).toContain('Mô tả tàu thủy')
  })
})
```

và tương tự `ShipCompareTab.spec.ts` (`toContain('Thông tin so sánh tàu thủy')`). Ghi chú đầu file spec: "Vỏ tạm của F01; F06/F08 ghi đè".

Chạy `npx vitest run src/components/appraisal-report/edit/__tests__/GroupWorkspace.spec.ts src/components/appraisal-report/edit/description/__tests__/ShipDescriptionTab.spec.ts src/components/appraisal-report/edit/compare/__tests__/ShipCompareTab.spec.ts`: FAIL. Cài đặt:

- Tạo hai vỏ `.vue` (`<script setup lang="ts">` với `defineProps` như Interfaces, template: `<div class="flex flex-col items-center gap-2 py-12 text-center text-muted-foreground"><Ship class="size-6" /><p class="text-sm">Mô tả tàu thủy đang được xây dựng.</p></div>` và câu "Thông tin so sánh tàu thủy đang được xây dựng." cho vỏ so sánh). Chỉ dùng token semantic, không màu cứng. Javadoc đầu file: "Vỏ tạm của nhóm tàu, F06 (hoặc F08) thay bằng bản đầy đủ."
- Sửa `GroupWorkspace.vue`: import 2 vỏ, đổi `byAssetType` thành `switch` exhaustive 4 nhánh (không `else`), cập nhật Javadoc ("Đất, Căn hộ, Ô tô hoặc Tàu thủy") và hai chỗ gọi.

Chạy lại: PASS.

Commit: `git -C frontend add -A src && git -C frontend commit -m "feat: GroupWorkspace nhánh tàu thủy với vỏ tab mô tả và so sánh"`.

- [ ] **Step 9: Kiểm tra toàn bộ**

```bash
cd frontend && npm run lint && npm run type-check && npm run test:unit -- --run
git -C frontend status --short   # lint có thể sửa file: commit nốt nếu có
```

Mong đợi: lint sạch, type-check sạch, toàn bộ test PASS. Tự kiểm bằng `git -C frontend grep -n "1 | 2 | 3" -- src` rồi đối chiếu: chỉ còn các chỗ không phải loại tài sản đã liệt kê trong bảng "Không đổi". Nếu có sửa do lint thì `git -C frontend commit -am "chore: lint F01"`.

Review focus cho reviewer F01:
- Không còn `else` ngầm hiểu CAR trong `GroupWorkspace`.
- Danh sách field `ShipDescriptionFields` khớp từng tên và kiểu với `UpdateAppraisalShipDescriptionRequest`.
- Không sửa file của F02, không sửa hành vi của `AssetResultTable`, `PanelSummaryRows`, `compare-common.ts`, `LegalTab`, `adjustment-panel-calc.ts:95`.

---

## Task F02 — Màn cơ bản: loại "Tàu thủy" và cơ sở thẩm định tàu

### Hành vi hệ thống cũ

- **Ô loại tài sản của từng pháp lý** có thêm `<option value="4">Tàu thủy</option>` sau Ôtô (`LT demo/basic-information.html:53`). Màn sửa đánh dấu `selected` khi nhóm của tài sản có `assetType == 4` (`LT demo/basic-information-edit.html:56`). FE mới: `LEGAL_ASSET_TYPE_OPTIONS` thêm `{ value: '4', label: 'Tàu thủy' }`; `toEditRows` đã chuyển số thành chuỗi nên loại 4 tự được chọn.
- **Cơ sở thẩm định** có tab thứ 5 "Tàu thủy" (`data-basis-tab="4"`, `LT demo/basic-information.html:206`; bản edit `basic-information-edit.html:209`), panel `basisItemsShip` lấy từ `valuationBasisListShip` (`basic-information.html:271-284`). Mỗi mục ghi `valuationBasisJson[idx+shipOffset]` gồm `{id, assetType=4, checked, content}`; tab cuối thứ tự: Chung(0), Đất(1), Chung cư(2), Ôtô(3), Tàu thủy(4). FE mới: tab `{ value: 4, label: 'Tàu thủy' }` và mục thêm mới ở tab này mang `assetType = 4` (`newConfigItem(4)` đã có sẵn trong `ReportConfigModal.addItem`).
- Màn này **không** chọn tiền tệ (tiền tệ chọn ở tab Kết quả). Trường "cho ai" nằm ở tab Pháp lý (`legacy-ui-word.md` mục 1).
- **Cảnh báo đổi loại tài sản** (`LT demo/demo-report-edit-basic.html:70-85`) giữ nguyên văn như cũ (L không thêm chữ "tàu"); FE `ReportLegalDocumentList.vue` không đổi.
- Hợp đồng BE: dữ liệu `valuationBases` của `GET /appraisal-reports/{id}/basic` và của màn tạo gộp theo loại 0..4 (B-task bên backend lo; mục `ValuationBasis.ASSET_TYPE_SHIP = 4`).

Lệch nhỏ với bản giao việc: tab lọc cơ sở thẩm định **không nằm trong `ReportConfigModal.vue`** mà ở hằng `BASIS_TABS` trong `AppraisalReportBasicForm.vue` (dòng 68-73), `ReportConfigModal` nhận `tabs` qua prop và thêm mục theo `assetType` của tab đang chọn, nên `ReportConfigModal.vue` **không phải sửa**, chỉ thêm test cho tab thứ 5.

### Files

| Việc | File (dưới `frontend/src/`) |
|---|---|
| Sửa | `lib/appraisal-report-form.ts` (thêm option, `VALUATION_BASIS_TABS`) |
| Sửa test | `lib/__tests__/appraisal-report-form.spec.ts` |
| Sửa | `components/appraisal-report/AppraisalReportBasicForm.vue` (dùng `VALUATION_BASIS_TABS`, xóa `BASIS_TABS` cục bộ) |
| Sửa | `api/report-edit/basic.ts` (kiểu loại 0..4, comment) |
| Sửa test | `components/appraisal-report/__tests__/ReportConfigModal.spec.ts` (tab Tàu thủy) |
| Sửa test | `components/appraisal-report/__tests__/ReportLegalDocumentList.spec.ts` (đổi sang loại 4) |
| Sửa test | `views/__tests__/AppraisalReportCreateView.spec.ts` (modal Cơ sở thẩm định có tab Tàu thủy) |

Không sửa `ReportConfigModal.vue`, `ReportLegalDocumentList.vue`, `api/appraisal-records.ts`, `AppraisalReportEditBasicView.vue`.

### Interfaces

**`src/api/report-edit/basic.ts`**

```ts
// KHÔNG import AssetType từ lib/report-edit/tabs: file đó do F01 sửa cùng đợt F1 (hai task phải độc lập).
/** Loại tài sản của cơ sở thẩm định (`valuation_basis.asset_type`): 0 chung, 1 đất, 2 chung cư, 3 ôtô, 4 tàu thủy. */
export type ValuationBasisAssetType = 0 | 1 | 2 | 3 | 4

// ReportBasicAssetRow.assetType giữ `number | null` (1 đất, 2 chung cư, 3 ôtô, 4 tàu thủy; null chưa chọn)
// ReportBasicEditData.valuationBases: comment đổi thành "Gộp theo loại tài sản 0..4; lọc theo `assetType` ở modal."
```

**`src/lib/appraisal-report-form.ts`**

```ts
export const LEGAL_ASSET_TYPE_OPTIONS: Option[] = [
  { value: '1', label: 'Đất' },
  { value: '2', label: 'Chung cư' },
  { value: '3', label: 'Ôtô' },
  { value: '4', label: 'Tàu thủy' },
]

/** Các tab của modal Cơ sở thẩm định theo `valuation_basis.asset_type`, theo thứ tự của hệ thống cũ. */
export const VALUATION_BASIS_TABS: { value: ValuationBasisAssetType; label: string }[] = [
  { value: 0, label: 'Chung' },
  { value: 1, label: 'Đất' },
  { value: 2, label: 'Chung cư' },
  { value: 3, label: 'Ôtô' },
  { value: 4, label: 'Tàu thủy' },
]
```

(`import type { ValuationBasisAssetType } from '@/api/report-edit/basic'` trong `appraisal-report-form.ts`.)

### Steps

- [ ] **Step 1: Test của `appraisal-report-form` (trước)**

Mở `src/lib/__tests__/appraisal-report-form.spec.ts`, thêm `LEGAL_ASSET_TYPE_OPTIONS`, `VALUATION_BASIS_TABS` vào import rồi thêm:

```ts
describe('loại tài sản tàu thủy', () => {
  it('ô_loại_tài_sản_có_Tàu_thủy_mã_4_ở_cuối', () => {
    expect(LEGAL_ASSET_TYPE_OPTIONS.map((option) => [option.value, option.label])).toEqual([
      ['1', 'Đất'], ['2', 'Chung cư'], ['3', 'Ôtô'], ['4', 'Tàu thủy'],
    ])
  })

  it('tab_cơ_sở_thẩm_định_có_5_tab_Tàu_thủy_cuối', () => {
    expect(VALUATION_BASIS_TABS.map((tab) => [tab.value, tab.label])).toEqual([
      [0, 'Chung'], [1, 'Đất'], [2, 'Chung cư'], [3, 'Ôtô'], [4, 'Tàu thủy'],
    ])
  })

  it('toEditRows_đổi_loại_4_thành_chuỗi_để_ô_chọn_Tàu_thủy_được_chọn', () => {
    expect(toEditRows([{ id: 7, uuid: 'a', checked: true, group: 1, assetType: 4, name: 'Sà lan' }])[0]!.assetType).toBe('4')
  })

  it('cùng_nhóm_không_được_trộn_tàu_với_ôtô', () => {
    expect(legalDocumentsProblem([row({ group: '1', assetType: '4' }), row({ group: '1', assetType: '3' })])).toBe(
      LEGAL_GROUP_MIXED_MESSAGE,
    )
    expect(legalDocumentsProblem([row({ group: '1', assetType: '4' }), row({ group: '1', assetType: '4' })])).toBeNull()
  })
})
```

Chạy `cd frontend && npx vitest run src/lib/__tests__/appraisal-report-form.spec.ts`: FAIL (thiếu `VALUATION_BASIS_TABS`, option chưa có).

- [ ] **Step 2: Cài đặt lib + kiểu**

Sửa `api/report-edit/basic.ts` (thêm `ValuationBasisAssetType`, đổi comment `valuationBases`), `lib/appraisal-report-form.ts` (thêm option, export `VALUATION_BASIS_TABS`, JSDoc tiếng Việt). Chạy lại spec: PASS.

- [ ] **Step 3: Test component (trước)**

`ReportConfigModal.spec.ts`, thêm:

```ts
it('tab_Tàu_thủy_chỉ_hiện_mục_loại_4_và_thêm_mục_mang_assetType_4', async () => {
  const wrapper = await open(
    toConfigItems([
      { id: 1, content: 'Giá trị thị trường', category: null, assetType: 0 },
      { id: 2, content: 'Giá trị thị trường tàu', category: null, assetType: 4 },
    ]),
    VALUATION_BASIS_TABS,
  )
  const tabs = [...document.body.querySelectorAll<HTMLElement>('[role="tab"]')]
  expect(tabs.map((tab) => tab.textContent!.trim())).toEqual(['Chung', 'Đất', 'Chung cư', 'Ôtô', 'Tàu thủy'])

  const shipTab = tabs.find((tab) => tab.textContent!.includes('Tàu thủy'))!
  shipTab.dispatchEvent(new MouseEvent('mousedown', { bubbles: true }))
  shipTab.click()
  await flushPromises()
  expect(rows()).toHaveLength(1)
  expect(rows()[0]!.querySelector('textarea')!.value).toBe('Giá trị thị trường tàu')

  button('Thêm mới').click()
  await flushPromises()
  const items = wrapper.props('items') as ConfigItem[]
  expect(items[items.length - 1]).toMatchObject({ id: null, assetType: 4, checked: false })
})
```

(import `VALUATION_BASIS_TABS` từ `@/lib/appraisal-report-form`; mở rộng kiểu tham số `tabs` của helper `open` cho khớp nếu type-check báo.)

`ReportLegalDocumentList.spec.ts`, thêm một ca: dòng đã lưu (`id: 5`) đổi sang loại `'4'` thì vẫn hỏi `confirmDialog` một lần và áp dụng khi xác nhận (cùng mẫu với ca đổi sang `'3'`), và `wrapper.findAllComponents(SelectBox)[1]!.props('options')` chứa `{ value: '4', label: 'Tàu thủy' }`.

`AppraisalReportCreateView.spec.ts`, thêm ca:

```ts
it('modal_cơ_sở_thẩm_định_có_tab_Tàu_thủy_ở_cuối', async () => {
  const wrapper = await mountAt()
  await button(wrapper, 'Cơ sở thẩm định').trigger('click')
  await settle()
  const tabs = [...document.body.querySelectorAll('[role="dialog"] [role="tab"]')].map((tab) => tab.textContent!.trim())
  expect(tabs).toEqual(['Chung', 'Đất', 'Chung cư', 'Ôtô', 'Tàu thủy'])
})
```

Chạy `npx vitest run src/components/appraisal-report/__tests__ src/views/__tests__/AppraisalReportCreateView.spec.ts`: ca `ReportConfigModal` có thể PASS vì modal nhận `tabs` từ prop (đó là kiểm tra hợp đồng); ca `CreateView` FAIL vì `AppraisalReportBasicForm` còn `BASIS_TABS` cũ.

- [ ] **Step 4: Cài đặt `AppraisalReportBasicForm.vue`**

Xóa hằng `BASIS_TABS` (dòng 67-73 cùng comment), import `VALUATION_BASIS_TABS` từ `@/lib/appraisal-report-form`, đổi `:tabs="config.key === 'valuationBases' ? VALUATION_BASIS_TABS : undefined"`. Giữ comment "Chủ đề cấu hình…" chuyển sang JSDoc của hằng trong lib. Chạy lại bước test ở Step 3: PASS.

- [ ] **Step 5: Kiểm tra và commit**

```bash
cd frontend && npm run lint && npm run type-check && npm run test:unit -- --run
git -C frontend add -A src
git -C frontend commit -m "feat: màn thông tin cơ bản thêm loại tài sản Tàu thủy và tab cơ sở thẩm định Tàu thủy"
```

Review focus cho reviewer F02:
- Đối chiếu `LT demo/basic-information.html:53,206,271-284` và `basic-information-edit.html:56,209`.
- Thứ tự tab cuối cùng đúng (Chung, Đất, Chung cư, Ôtô, Tàu thủy) và mục thêm mới ở tab Tàu thủy có `assetType = 4`.
- Không đụng file của F01 (`lib/report-edit/*`, `api/report-edit/*` khác `basic.ts`, `edit/**`).

---

## Vấn đề cần controller quyết định

1. **→ Đã chốt: Quyết định #1.** Khóa JSON kết quả tàu là `ship` (theo `car`); F01 khai báo `ship: ShipResultDetail | null`, F11 dùng `ship`.
2. **→ Đã chốt: Quyết định #7.** Endpoint kho dùng chung rẽ theo `assetType` (3: `CompareInformationDetail`, 4: `ShipCompareInformationItem`); `updated` của tàu là ISO `YYYY-MM-DD`, FE định dạng `dd/MM/yyyy`.
3. **→ Đã chốt: Quyết định #17.** `git fetch` lỗi chứng chỉ SSL trên máy này: dùng `git -c http.sslBackend=schannel fetch …` (đã sửa lệnh ở Step 1). Nếu develop có commit mới đụng `GroupWorkspace.vue`, `tabs.ts` hay `appraisal-report-form.ts` thì grep lại số dòng.
4. **→ Đã gán chủ (không còn mở):** `compare-common.ts` (`fillFromKho`, `emptyColumn`) và `useCompareForm.ts` -> F08 (gọi hàm của F10); `adjustment-panel-calc.ts:56,95`, `PanelSummaryRows.vue`, `ValuationMethodCard.vue` -> F09; `AssetResultTable.vue` -> F11; `LegalTab.vue` -> F07; `NearbyCompareMapModal.vue:201` không dùng cho tàu (`ShipCompareTab` không có bản đồ) nên không đổi, Z01 grep `assetType` sẽ xác nhận.
5. **Boolean của mô tả tàu để `boolean | null`** (hàng cũ trong DB có thể null). F03/F06 coi `null` như `false` khi hiển thị và gửi `false`/`true` khi lưu; nếu BE đảm bảo luôn non-null thì có thể siết kiểu về `boolean`.
6. **Tiền tệ trong `EvaluationResult` để nullable** (`currencyCode: string | null`, `exchangeRate: number | null`) vì BE có bản ghi rỗng (`EvaluationResultResponse.empty()` mọi trường null). Nếu B06 luôn trả đủ cho mọi nhóm thì siết lại thành `CurrencyCode` và `number`.
7. **`CurrencyCodeTest` chỉ port được 3/5 case sang FE.** Hai case `regulatoryUnitForShipHasNoSquareMeter`, `readMoneyUsd` là của BE (FE nhận đơn vị và chữ từ server). Bù bằng 2 case riêng của FE (nhãn với mã lạ/null, hằng `CURRENCY_CODE`). Nếu controller muốn FE có thêm hàm đọc tiền hay `regulatoryPriceUnit` thì phải nêu rõ.
8. **Bản giao việc nói thêm tab ở `ReportConfigModal`**, thực tế tab nằm ở `AppraisalReportBasicForm.vue` (`BASIS_TABS`), nên F02 sửa file này và chỉ thêm test cho `ReportConfigModal`.
