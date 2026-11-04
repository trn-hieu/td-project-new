# 11 — FE Tab "Mô tả tàu thủy" (F03–F06)

> Đọc `00-index.md` trước (quy trình, Global Constraints, viết tắt **L/LR/F**, hợp đồng dùng chung). Subagent: `model: "sonnet"`, mỗi task có spec review + code review.
>
> Phụ thuộc: **F01** (`10-fe-foundation.md`) đã tạo: kiểu `ShipDescriptionData`, `ShipDescriptionInput`, `ShipQualityItem`, `ShipConditionItem` và hàm `getShipDescription`/`saveShipDescription` trong `F api/report-edit/ship-description.ts`; `IMAGE_TYPE.SHIP_*` trong `F lib/report-edit/image-types.ts`; `ASSET_TYPE.SHIP`; vỏ `F components/appraisal-report/edit/description/ShipDescriptionTab.vue` (đã được `GroupWorkspace` gắn vào tab "Mô tả tàu thủy"). **Đọc file thật của F01 trước khi viết code** và dùng đúng tên F01 đặt (xem mục "Vấn đề cần controller quyết định").

Ký hiệu riêng của file này:
- **JS** = `td-project-legacy` @ `82193fe5`: `src/main/resources/static/js/ship-description.js`.
- **HTML** = cùng repo: `src/main/resources/templates/demo/ship-description.html`.
- **FC** = `F components/appraisal-report/edit/description/` (đã có `CarDescriptionTab.vue`, `CarDescriptionForm.vue`, `LandDetailTable.vue` làm mẫu).
- **FL** = `F lib/report-edit/`.

Đọc code legacy bằng `git -C td-project-legacy show 82193fe5:<đường dẫn>`. Không checkout. Đọc code mới trên `origin/develop` của `frontend` (`git -C frontend show origin/develop:src/...`), không checkout.

## Hành vi hệ thống cũ

Nguồn: JS (toàn bộ), HTML (toàn bộ), `surveys/legacy-ui-word.md` mục 3, `surveys/legacy-be.md` mục 2 và 6.

**Bố cục (HTML l.12–1250)**: tiêu đề "I. Mô tả tàu thủy" (l.12), dòng **Nơi neo đậu** (`mooringLocation`, l.19), 18 khối (liệt kê ở F03), "II. Tình trạng của tài sản thẩm định" (l.973), "III. Đánh giá chất lượng còn lại của tàu" (l.1003), "IV. Hình ảnh" (l.1097, chỉ khi `ship.id != null`, 11 mục: Mặt trước tàu, Đầu tàu, Đuôi tàu, Mạn tàu, Sàn tàu, Số IMO/Số kiểm soát, Hệ thống neo, Phòng lái, phòng nghỉ, Khoang hàng, Hệ thống máy của động cơ, Khác). Footer: nút Tải báo cáo và Lưu.

**Kiểu input và giới hạn (JS l.16–92, l.129–200)**:
- Text: textarea tự giãn **chặn Enter**; độ dài lấy theo `TEXT_MAX` (JS l.16–44, **JS là chuẩn khi `maxlength` HTML khác**). `operationNote` là textarea 3 dòng, 5000 ký tự, cho xuống dòng.
- Số nguyên (`INTEGER_FIELDS`, 18 field, JS l.71): chỉ chữ số, tối đa 9 chữ số; `buildYear`, `designYear`, `mainEngineManufactureYear` tối đa 4 chữ số.
- Thập phân (`DECIMAL_FIELDS`, 23 field, JS l.45): kiểu châu Âu `1.234,50`, **không âm**, tối đa 2 chữ số thập phân; phần nguyên tối đa **8** chữ số (DECIMAL(10,2)), riêng 13 field DECIMAL(12,2) (`DECIMAL_INT_DIGITS`, JS l.53–61) tối đa **10**. Message: "Số không hợp lệ (tối đa N chữ số nguyên, tối đa 2 chữ số thập phân, không âm)". `anchorSternTotalWeightKg` nay là **text** 1000 ký tự.
- Boolean (`BOOLEAN_FIELDS`, 39 field, JS l.77): checkbox (16 đèn hiệu, 10 vật hiệu, 11 trang bị chạy tàu, `fireManualTools`, `fireLayoutPlan`). `navigationRadio/Ais/Sart` là text.
- Ngày (`DATE_FIELDS`, 3 field, JS l.89): ISO `yyyy-MM-dd`, rỗng thì null.
- Tổng: 70 text + 23 thập phân + 18 nguyên + 39 boolean + 3 ngày = **153 field phẳng** (kể cả `mooringLocation`) cộng 2 danh sách JSON = 155 key của `ShipDescriptionInput`.
- Payload (`buildPayload`, JS l.194–217): text trim, rỗng thành null; thập phân/nguyên thành số hoặc null; boolean theo `checked`; ngày rỗng thành null.

**Bảng II Tình trạng (JS l.219–270, HTML l.973–1001)**: nút "+ Thêm hạng mục", nút xóa từng dòng (xóa ngay), STT tự đánh lại. Cột STT, Hạng mục (≤500 ký tự), Tình trạng (không giới hạn), Xóa. Khi lưu: bỏ dòng cả hai ô rỗng (`buildConditionDetails`); dòng có Tình trạng mà thiếu Hạng mục thì ô Hạng mục `is-invalid` và toast "Vui lòng nhập hạng mục cho các dòng tình trạng đã nhập" (`validateConditionRows` l.233, `submitShipForm` l.436).

**Bảng III Chất lượng còn lại (JS l.274–386, HTML l.1003–1095)**:
- Mỗi dòng là một item. Ô "Hạng mục thẩm định" có **nhiều ô tên** (≤500 ký tự), mỗi ô có nút ×; dưới có nút "+ Hạng mục". Bấm × ở ô cuối cùng chỉ xóa nội dung, không xóa ô (JS l.591). Nút "+ Thêm dòng" thêm item; nút xóa dòng xóa ngay, không hỏi.
- 9 cột: STT, Hạng mục thẩm định, (1) Hệ số quan trọng, (2) Giá trị chuẩn so sánh, Đánh giá (select `-- Chọn --`, Tốt, Khá, Trung bình, Kém; ghép với ô (3) dưới tiêu đề "Đánh giá (3)"), (4)=(3)/(2), (5)=(1)*(4), thao tác. Select **không tham gia tính**.
- Ô số: tối đa 8 chữ số nguyên + 2 thập phân, chỉ chữ số và một dấu phẩy (`isPartialQualityNumber`, JS l.536), không có phân cách nghìn.
- Công thức (`refreshQuality`, JS l.314): `raw4 = (s != null && s > 0 && r != null) ? r/s : null`; (4) hiển thị `Math.round((raw4 + EPSILON) * 10) / 10` định dạng `de-DE` đúng 1 chữ số thập phân; `c5 = (w != null && raw4 != null) ? w * raw4 * 100 : null` **dùng raw4 chưa làm tròn**, hiển thị `de-DE` 2 chữ số thập phân kèm `%`; tổng = Σ c5 (2 chữ số kèm `%`), chỉ hiện khi có ít nhất 1 dòng (không có dòng thì ô tổng rỗng; có dòng nhưng không dòng nào tính được thì hiện `0,00%`). Không có dòng: hiện dòng "Chưa có dữ liệu".
- Validate (`validateQualityRows`, JS l.372): dòng có số liệu/đánh giá mà không có tên nào thì ô tên đầu tiên `is-invalid`; `s != null && s <= 0` thì ô (2) `is-invalid`. Toast: "Đánh giá chất lượng: vui lòng nhập tên hạng mục và giá trị chuẩn so sánh lớn hơn 0".
- Payload (`buildQualityEvaluationDetails`, JS l.359): tên trim, bỏ tên rỗng; `rating` rỗng thành null; bỏ item rỗng hoàn toàn (không tên, 3 số null, rating rỗng).

**Lưu (JS l.436–470)**: validate hai bảng rồi `PUT`; thành công toast rồi tải lại tab; lỗi toast message server hoặc "Lưu mô tả tàu thủy không thành công, vui lòng thử lại".

**BE tương ứng (`surveys/legacy-be.md` mục 5)**: `calculateRemainingQualityPct` — với mỗi item đủ w, r, s và `s > 0`: `w × (r ÷ s, scale 10, HALF_UP) × 100`; tổng ngoài 0–100 hoặc không có item hợp lệ thì trả null, ngược lại `setScale(2, HALF_UP)`. FE chỉ hiển thị trực tiếp; server mới là nơi tính `remainingQualityPct` ghi sang tab So sánh.

### Khác hệ thống cũ (cố ý) — ghi vào Javadoc/comment của file liên quan

1. Ô thập phân dùng `DecimalInput` (nhóm hàng nghìn bằng dấu chấm, chặn ký tự lạ, tối đa 2 lẻ) và kiểm tra giới hạn phần nguyên bằng zod khi blur/submit; legacy chặn từng phím. Message dùng `fields.number` ("Tối đa 99.999.999,99").
2. Ô số bảng III cũng dùng `DecimalInput` (có chấm nghìn khi hiển thị), giới hạn 8+2 kiểm tra khi lưu; legacy cấm dấu chấm.
3. Ngày chọn bằng `DatePicker` (hiển thị dd/MM/yyyy, giá trị ISO) thay cho `<input type=date>`.
4. Lưu xong tab tải lại dữ liệu tại chỗ (không dựng lại toàn tab) và **không** vá DOM tab So sánh: tab So sánh tự tải lại khi mở (`00-index.md`, "Khác hệ thống cũ" mục 2).
5. Lỗi hiện ngay dưới từng ô và có `FormErrorSummary` đầu form; không dùng `is-invalid` của Bootstrap.

---

## F03 — Hàm thuần: `ship-quality.ts`, `ship-description-fields.ts`

**Đợt F2. Phụ thuộc F01.** Chỉ tạo file trong `FL` và `FL __tests__/` (không đụng component).

### Task F03: Hàm thuần chất lượng còn lại và metadata field

**Files:**
- Create `F lib/report-edit/ship-quality.ts`
- Create `F lib/report-edit/ship-description-fields.ts`
- Create `F lib/report-edit/__tests__/ship-quality.spec.ts`
- Create `F lib/report-edit/__tests__/ship-description-fields.spec.ts`

**Interfaces** (F04, F05, F06 dùng nguyên tên này):

```ts
// ship-quality.ts  (không import Vue; chỉ import type từ '@/api/report-edit/ship-description')
export const SHIP_RATINGS = ['Tốt', 'Khá', 'Trung bình', 'Kém'] as const
export const QUALITY_NAME_MAX = 500
export const QUALITY_NUMBER_MAX = 99_999_999.99          // 8 chữ số nguyên + 2 thập phân
export const QUALITY_ERROR_MESSAGE =
  'Đánh giá chất lượng: vui lòng nhập tên hạng mục và giá trị chuẩn so sánh lớn hơn 0'

/** (3)/(2) chưa làm tròn; null khi thiếu (2), (2) <= 0 hoặc thiếu (3). */
export function rawRelative(standard: number | null, relative: number | null): number | null
/** (4) hiển thị: làm tròn 1 lẻ kiểu Math.round((raw4 + Number.EPSILON) * 10) / 10; null nếu raw4 null. */
export function relativeDisplay(raw4: number | null): number | null
/** (5) = (1) × raw4 × 100, dùng raw4 CHƯA làm tròn; null nếu thiếu (1) hoặc raw4. */
export function composite(weight: number | null, raw4: number | null): number | null
export interface QualityRowMetrics { raw4: number | null; c4: number | null; c5: number | null }
export function qualityRowMetrics(item: ShipQualityItem): QualityRowMetrics
/** Σ c5 các dòng; null khi không có dòng nào (items.length === 0); có dòng mà không dòng nào tính được thì 0. */
export function qualityTotal(items: readonly ShipQualityItem[]): number | null
export const formatQualityC4: (c4: number | null) => string       // '0,8' | ''  (de-DE, đúng 1 lẻ)
export const formatQualityPercent: (v: number | null) => string   // '24,00%' | '' (de-DE, đúng 2 lẻ + '%')

export interface QualityItemErrors { missingName: boolean; invalidStandard: boolean; numberTooLarge: boolean }
export function trimmedNames(item: ShipQualityItem): string[]               // trim, bỏ tên rỗng
export function isQualityItemEmpty(item: ShipQualityItem): boolean          // không tên + 3 số null + rating rỗng/null
export function qualityItemErrors(item: ShipQualityItem): QualityItemErrors
export function hasQualityErrors(items: readonly ShipQualityItem[]): boolean
/** Bỏ item rỗng, trim tên, bỏ tên rỗng, rating '' → null. Giữ nguyên thứ tự. */
export function normalizeQualityItems(items: readonly ShipQualityItem[]): ShipQualityItem[]
export function emptyQualityItem(): ShipQualityItem   // { itemNames: [''], importanceWeight: null, standardValue: null, rating: null, relativeValue: null }
```

```ts
// ship-description-fields.ts
import type { z } from 'zod'
export type ShipFieldKind = 'text' | 'textarea' | 'integer' | 'decimal' | 'boolean' | 'date'
export interface ShipField {
  key: string            // trùng tên field của ShipDescriptionInput
  label: string          // nhãn tiếng Việt đúng HTML legacy
  kind: ShipFieldKind
  maxLength?: number     // text/textarea: theo TEXT_MAX của JS
  intDigits?: 8 | 10     // decimal: số chữ số phần nguyên tối đa
  max?: number           // integer: giá trị tối đa (9999 cho 3 field năm, 999_999_999 còn lại)
  grouping?: false       // integer là năm: NumberInput không nhóm nghìn
}
export interface ShipFieldGroup { title?: string; fields: readonly ShipField[] }
export interface ShipSection { id: string; title: string; groups: readonly ShipFieldGroup[] }

export const SHIP_MOORING_FIELD: ShipField                  // mooringLocation, hàng đầu
export const SHIP_SECTIONS: readonly ShipSection[]          // đúng 18 khối, đúng thứ tự
export const SHIP_FIELDS: readonly ShipField[]              // phẳng: mooring + 152 field của 18 khối = 153
export type ShipFieldKey = /* union literal key của SHIP_FIELDS; khai báo bằng `as const satisfies` */

export const SHIP_CONDITION_ITEM_MAX = 500
export const SHIP_CONDITION_ERROR = 'Vui lòng nhập hạng mục cho các dòng tình trạng đã nhập'

export type ShipFormValues = Record<ShipFieldKey, string | number | boolean | undefined>
export function buildShipSchema(): z.ZodObject<...>        // zod v4, dùng fields.number của '@/lib/form'
export function shipFormDefaults(data?: ShipDescriptionData | null): ShipFormValues
export function conditionRowInvalid(row: ShipConditionItem): boolean   // có Tình trạng nhưng thiếu Hạng mục (sau trim)
export function normalizeConditionRows(rows: readonly ShipConditionItem[]): ShipConditionItem[]
export function toShipInput(
  values: ShipFormValues,
  conditions: readonly ShipConditionItem[],
  quality: readonly ShipQualityItem[],
): ShipDescriptionInput
```

**Quy tắc `buildShipSchema()`** (mỗi field một entry theo `kind`, nguồn là `SHIP_FIELDS`):
- `text`/`textarea`: `z.string().max(maxLength, 'Tối đa N ký tự')` (không trim trong schema; trim ở `toShipInput`).
- `integer`: `fields.number({ min: 0, max, integer: true }).optional()`.
- `decimal`: `fields.number({ min: 0, max: 10 ** intDigits - 0.01 }).optional()` (8 thành 99.999.999,99; 10 thành 9.999.999.999,99).
- `boolean`: `z.boolean()`.
- `date`: `z.string().regex(/^\d{4}-\d{2}-\d{2}$/, 'Ngày không hợp lệ').optional()`.

**Quy tắc `shipFormDefaults(data)`**: text/textarea thành `data?.[key] ?? ''`; integer/decimal thành `data?.[key] ?? undefined`; boolean thành `!!data?.[key]`; date thành `data?.[key] ?? undefined`. Không có `data` thì toàn bộ rỗng/false.

**Quy tắc `toShipInput`**: text/textarea trim, rỗng thành null; integer/decimal `undefined` thành null; boolean `=== true`; date rỗng/undefined thành null; `conditionDetails = normalizeConditionRows(conditions)`; `qualityEvaluationDetails = normalizeQualityItems(quality)`. `normalizeConditionRows`: `itemName` trim, `conditionDescription` trim và rỗng thành null; bỏ dòng cả hai rỗng; **không** bỏ dòng thiếu hạng mục (việc chặn là của `conditionRowInvalid` ở F06).

**Metadata 18 khối** (nhãn, kind, giới hạn trích từ HTML/JS legacy; `maxLength` theo `TEXT_MAX`, `intDigits 10` cho 13 field DECIMAL(12,2), `max 9999` cho field năm, `max 999999999` cho số nguyên còn lại). `SHIP_SECTIONS[i].id` là slug ổn định (gợi ý: `main-specs`, `design`, `anchor`, `steering`, `mooring`, `towing`, `main-engine`, `aux-engine`, `electric`, `compressed-air`, `lifting`, `fire`, `lifesaving`, `damage-control`, `signal`, `navigation`, `pollution`, `inspection`), `title` đúng tên khối bên dưới. Các khối có tiêu đề phụ (`groups[].title`):
- khối 12: nhóm không tên (15 field đầu) rồi nhóm "Dụng cụ và sơ đồ chữa cháy" (`fireManualTools`, `fireLayoutPlan`);
- khối 15: "Đèn hiệu" (16 boolean), "Âm hiệu" (`signalSound`), "Vật hiệu" (10 boolean);
- khối 16: nhóm không tên gồm 11 boolean rồi 3 text (`navigationRadio`, `navigationAis`, `navigationSart`);
- các khối còn lại: một nhóm không tên.

**Nơi neo đậu (hàng đầu, ngoài 18 khối):** `mooringLocation` | Nơi neo đậu | text | maxLength 255

**1. Thông số chính** (21 field)

| key | Nhãn | kind | Giới hạn |
|---|---|---|---|
| `shipName` | Tên phương tiện | text | maxLength 500 |
| `registrationNumber` | Số đăng ký | text | maxLength 255 |
| `shipTypePurpose` | Công dụng | text | maxLength 500 |
| `registryNumber` | Số đăng kiểm | text | maxLength 255 |
| `ownerName` | Chủ phương tiện | text | maxLength 500 |
| `buildYear` | Năm đóng | integer | max 9999 |
| `buildPlace` | Nơi đóng | text | maxLength 1000 |
| `dimLength` | L - Chiều dài thiết kế (m) | decimal | intDigits 8 |
| `dimBreadth` | B - Chiều rộng thiết kế (m) | decimal | intDigits 8 |
| `dimDepth` | D - Chiều cao mạn (m) | decimal | intDigits 8 |
| `dimDraft` | d - Chiều chìm thiết kế (m) | decimal | intDigits 8 |
| `dimLengthMax` | Lmax - Chiều dài lớn nhất (m) | decimal | intDigits 8 |
| `dimBreadthMax` | Bmax - Chiều rộng lớn nhất (m) | decimal | intDigits 8 |
| `freeBoard` | Mạn khô F (mm) | decimal | intDigits 8 |
| `material` | Vật liệu | text | maxLength 255 |
| `grossTonnage` | Tổng dung tích (GT) | integer | max 999999999 |
| `shipClass` | Cấp phương tiện | text | maxLength 255 |
| `operatingArea` | Vùng hoạt động | text | maxLength 255 |
| `cargoCapacity` | Khả năng khai thác - Lượng hàng (tấn) | decimal | intDigits 10 |
| `deadweight` | Khả năng khai thác - TTTP (tấn) | decimal | intDigits 10 |
| `operationNote` | Ghi chú hoạt động | textarea | maxLength 5000 |

**2. Thiết kế phương tiện** (5 field)

| key | Nhãn | kind | Giới hạn |
|---|---|---|---|
| `designSymbol` | Ký hiệu thiết kế | text | maxLength 255 |
| `designYear` | Năm thiết kế | integer | max 9999 |
| `designCertificateNumber` | Số GCN | text | maxLength 255 |
| `appraisalNumber` | Số thẩm định | text | maxLength 255 |
| `conversionYearPlace` | Năm và nơi hoán cải | text | maxLength 500 |

**3. Thiết bị neo** (8 field)

| key | Nhãn | kind | Giới hạn |
|---|---|---|---|
| `anchorBowType` | Neo mũi - Kiểu | text | maxLength 100 |
| `anchorBowQuantity` | Neo mũi - Số lượng (chiếc) | integer | max 999999999 |
| `anchorBowTotalWeightKg` | Neo mũi - Tổng khối lượng (kg) | decimal | intDigits 10 |
| `anchorSternType` | Neo đuôi - Kiểu | text | maxLength 100 |
| `anchorSternQuantity` | Neo đuôi - Số lượng (chiếc) | integer | max 999999999 |
| `anchorSternTotalWeightKg` | Neo đuôi - Tổng khối lượng (kg) | text | maxLength 1000 |
| `anchorChainCable` | Chiều dài/đường kính xích, cáp | text | maxLength 255 |
| `anchorWindlassType` | Kiểu tời neo | text | maxLength 100 |

**4. Thiết bị lái** (7 field)

| key | Nhãn | kind | Giới hạn |
|---|---|---|---|
| `steeringRudderType` | Bánh lái - Kiểu | text | maxLength 100 |
| `steeringRudderQuantity` | Bánh lái - Số lượng (chiếc) | integer | max 999999999 |
| `steeringRudderTotalAreaM2` | Bánh lái - Tổng diện tích (m2) | decimal | intDigits 10 |
| `steeringMainGearType` | Máy lái chính - Kiểu | text | maxLength 100 |
| `steeringMainGearTorqueTm` | Máy lái chính - Mô men lái (tm) | decimal | intDigits 10 |
| `steeringBackupGearType` | Máy lái dự phòng - Kiểu | text | maxLength 100 |
| `steeringBackupGearTorqueTm` | Máy lái dự phòng - Mô men lái (tm) | decimal | intDigits 10 |

**5. Thiết bị chằng buộc** (4 field)

| key | Nhãn | kind | Giới hạn |
|---|---|---|---|
| `mooringRopeQuantity` | Dây buộc - Số lượng (chiếc) | integer | max 999999999 |
| `mooringRopeTotalLengthM` | Dây buộc - Tổng chiều dài (m) | decimal | intDigits 10 |
| `mooringBollardType` | Cột buộc dây - Kiểu | text | maxLength 100 |
| `mooringBollardQuantity` | Cột buộc dây - Số lượng (chiếc) | text | maxLength 1000 |

**6. Thiết bị kéo** (4 field)

| key | Nhãn | kind | Giới hạn |
|---|---|---|---|
| `towingBollardType` | Kiểu cột kéo | text | maxLength 1000 |
| `towingHookType` | Kiểu mọc kéo | text | maxLength 1000 |
| `towingWinchType` | Kiểu tời thu dây | text | maxLength 1000 |
| `towingRopeType` | Loại dây kéo | text | maxLength 1000 |

**7. Máy chính** (5 field)

| key | Nhãn | kind | Giới hạn |
|---|---|---|---|
| `mainEngineQuantity` | Số lượng (chiếc) | integer | max 999999999 |
| `mainEngineBrand` | Mác máy | text | maxLength 255 |
| `mainEngineSerialNumber` | Số chế tạo | text | maxLength 255 |
| `mainEngineManufactureYear` | Năm chế tạo | integer | max 9999 |
| `mainEnginePowerHp` | Công suất (HP/KW) | decimal | intDigits 8 |

**8. Máy phụ** (3 field)

| key | Nhãn | kind | Giới hạn |
|---|---|---|---|
| `auxEngineQuantity` | Số lượng (chiếc) | integer | max 999999999 |
| `auxEngineBrand` | Mác máy | text | maxLength 255 |
| `auxEnginePowerHp` | Tổng công suất (sức ngựa) | decimal | intDigits 8 |

**9. Thiết bị điện** (6 field)

| key | Nhãn | kind | Giới hạn |
|---|---|---|---|
| `electricGenerator` | Máy phát | text | maxLength 100 |
| `electricGeneratorPowerKva` | Công suất (KVA) | text | maxLength 100 |
| `electricGeneratorVoltage` | Điện áp (V) | text | maxLength 255 |
| `electricBatteryQuantity` | Ắc quy - Số bình | integer | max 999999999 |
| `electricBatteryVoltageV` | Ắc quy - Điện áp (V) | decimal | intDigits 8 |
| `electricBatteryTotalCapacity` | Ắc quy - Tổng dung lượng (Ah) | text | maxLength 255 |

**10. Hệ thống khí nén** (2 field)

| key | Nhãn | kind | Giới hạn |
|---|---|---|---|
| `compressedAirTankQuantity` | Bình khí nén (chiếc) | text | maxLength 1000 |
| `compressedAirCompressorQuantity` | Máy nén khí (chiếc) | text | maxLength 1000 |

**11. Thiết bị nâng** (3 field)

| key | Nhãn | kind | Giới hạn |
|---|---|---|---|
| `liftingSerialNumber` | Số chế tạo hoặc số quản lý | text | maxLength 255 |
| `liftingMaxReachCapacity` | Tầm với lớn nhất (m)/sức nâng (tấn) | text | maxLength 255 |
| `liftingMinReachCapacity` | Tầm với nhỏ nhất (m)/sức nâng (tấn) | text | maxLength 255 |

**12. Trang bị chữa cháy** (17 field)

| key | Nhãn | kind | Giới hạn |
|---|---|---|---|
| `firePumpQuantity` | Bơm chữa cháy - Số lượng (chiếc) | integer | max 999999999 |
| `firePumpTotalFlowM3` | Bơm chữa cháy - Tổng lưu lượng (m3) | decimal | intDigits 10 |
| `fireGasType` | Loại khí | text | maxLength 100 |
| `fireGasTotalWeightKg` | Khí - Tổng khối lượng (kg) | decimal | intDigits 10 |
| `fireFoamType` | Loại bọt | text | maxLength 100 |
| `fireFoamTotalVolumeL` | Bọt - Tổng thể tích (lít) | text | maxLength 1000 |
| `fireFoamPusherQuantity` | Bình bọt đẩy - Số lượng | text | maxLength 1000 |
| `fireFoamPusherVolumeL` | Bình bọt đẩy - Dung tích (lít) | text | maxLength 1000 |
| `firePortableFoam` | Thiết bị tạo bọt xách tay - Số lượng | text | maxLength 100 |
| `fireCo2BottleQuantity` | Bình CO2 - Số bình | integer | max 999999999 |
| `fireCo2BottleWeightKg` | Bình CO2 - Khối lượng (kg) | decimal | intDigits 10 |
| `firePowderBottleQuantity` | Bình bột - Số bình | integer | max 999999999 |
| `firePowderBottleWeightKg` | Bình bột - Khối lượng (kg) | decimal | intDigits 10 |
| `fireFoamBottleQuantity` | Bình bọt - Số bình | text | maxLength 1000 |
| `fireFoamBottleVolumeL` | Bình bọt - Dung tích (lít) | text | maxLength 1000 |
| `fireManualTools` | Bộ dụng cụ chữa cháy thủ công | boolean | — |
| `fireLayoutPlan` | Sơ đồ bố trí thiết bị chữa cháy | boolean | — |

**13. Trang bị cứu sinh** (4 field)

| key | Nhãn | kind | Giới hạn |
|---|---|---|---|
| `lifesavingRingBuoy` | Phao tròn (chiếc): không dây; có dây | text | maxLength 100 |
| `lifesavingLifeJacket` | Phao áo (chiếc); phao áo trẻ em | text | maxLength 100 |
| `lifesavingRaftBoatQuantity` | DC nổi/phao bè/Xuồng - Số lượng (chiếc) | integer | max 999999999 |
| `lifesavingRaftBoatTotalCapacity` | DC nổi/phao bè/Xuồng - Tổng sức chở (người) | integer | max 999999999 |

**14. Trang bị cứu đắm** (4 field)

| key | Nhãn | kind | Giới hạn |
|---|---|---|---|
| `damageControlBilgePumpQuantity` | Bơm hút khô - Số lượng (chiếc) | integer | max 999999999 |
| `damageControlBilgePumpTotalFlowM3` | Bơm hút khô - Tổng lưu lượng (m3) | decimal | intDigits 10 |
| `damageControlTools` | Bộ dụng cụ cứu đắm | text | maxLength 100 |
| `damageControlBilgeAlarm` | Thiết bị tự động báo mức nước đáy khoang | text | maxLength 100 |

**15. Trang bị tín hiệu** (27 field)

| key | Nhãn | kind | Giới hạn |
|---|---|---|---|
| `signalLightSideRed` | Đèn mạn đỏ | boolean | — |
| `signalLightSideGreen` | Đèn mạn xanh | boolean | — |
| `signalLightSternWhite` | Đèn trắng lái | boolean | — |
| `signalLightMastWhite` | Đèn cột trắng | boolean | — |
| `signalLightMastGreen` | Đèn cột xanh | boolean | — |
| `signalLightMastRed` | Đèn cột đỏ | boolean | — |
| `signalLightAnchor` | Đèn neo | boolean | — |
| `signalLightTowingRed` | Đèn lai đỏ | boolean | — |
| `signalLightTowingGreen` | Đèn lai xanh | boolean | — |
| `signalLightYellow` | Đèn vàng | boolean | — |
| `signalLightFlashWhite` | Đèn trắng nháy | boolean | — |
| `signalLightFlashYellow` | Đèn vàng nháy | boolean | — |
| `signalLightFlashRed` | Đèn đỏ nháy | boolean | — |
| `signalLightOther` | Đèn cho các tình huống khác | boolean | — |
| `signalLightHalfGreenRed` | Đèn nửa xanh nửa đỏ | boolean | — |
| `signalLightSpare` | Đèn dự trữ | boolean | — |
| `signalSound` | Âm hiệu | text | maxLength 100 |
| `signalShapeBlackTriangle` | Tam giác đen | boolean | — |
| `signalShapeWhiteTriangle` | Tam giác trắng | boolean | — |
| `signalShapeBlackSquare` | Vuông đen | boolean | — |
| `signalShapeBlackRectangle` | Chữ nhật đen | boolean | — |
| `signalShapeBlackCircle` | Tròn đen | boolean | — |
| `signalShapeWhiteCircle` | Tròn trắng | boolean | — |
| `signalShapeBlackDiamond` | Hình thoi góc vuông đen | boolean | — |
| `signalShapeFlagSet` | Bộ cờ hiệu | boolean | — |
| `signalShapeParachuteFlare` | Pháo hiệu dù | boolean | — |
| `signalShapeLineThrower` | Súng phóng dây | boolean | — |

**16. Trang bị chạy tàu** (14 field)

| key | Nhãn | kind | Giới hạn |
|---|---|---|---|
| `navigationBinoculars` | Ống nhòm | boolean | — |
| `navigationClock` | Đồng hồ | boolean | — |
| `navigationDepthSounder` | Thiết bị đo sâu đơn giản | boolean | — |
| `navigationOrderBell` | Chuông lệnh | boolean | — |
| `navigationInclinometer` | Thiết bị đo nghiêng | boolean | — |
| `navigationDraftGauge` | Thước đo mức nước | boolean | — |
| `navigationGps` | GPS hoặc thiết bị tương tự | boolean | — |
| `navigationCompass` | La bàn từ hoặc la bàn chuẩn | boolean | — |
| `navigationCommandPa` | Thiết bị truyền thanh chỉ huy | boolean | — |
| `navigationRadar` | Ra đa | boolean | — |
| `navigationEpirb` | S.EPIRB | boolean | — |
| `navigationRadio` | Thiết bị VTĐ (MF/HF; VHF) | text | maxLength 100 |
| `navigationAis` | AIS (cấp A; cấp B) | text | maxLength 100 |
| `navigationSart` | SART/AIS-SART | text | maxLength 100 |

**17. Trang bị ngừa ô nhiễm** (15 field)

| key | Nhãn | kind | Giới hạn |
|---|---|---|---|
| `pollutionOilyWaterSeparatorQuantity` | Thiết bị phân ly dầu nước - Số lượng (chiếc) | text | maxLength 1000 |
| `pollutionOilyWaterSeparatorTotalOutput` | Thiết bị phân ly dầu nước - Tổng sản lượng | text | maxLength 1000 |
| `pollutionOilyWaterTankQuantity` | Két thu hồi hỗn hợp dầu nước - Số lượng (chiếc) | integer | max 999999999 |
| `pollutionOilyWaterTankTotalVolumeM3` | Két thu hồi hỗn hợp dầu nước - Tổng thể tích (m3) | decimal | intDigits 10 |
| `pollutionSewageTankQuantity` | Két nước thải - Số lượng (chiếc) | text | maxLength 1000 |
| `pollutionSewageTankTotalVolumeM3` | Két nước thải - Tổng thể tích (m3) | text | maxLength 1000 |
| `pollutionSettlingTankQuantity` | Két lắng - Số lượng (chiếc) | text | maxLength 1000 |
| `pollutionSettlingTankTotalVolumeM3` | Két lắng - Tổng thể tích (m3) | text | maxLength 1000 |
| `pollutionSludgeTankQuantity` | Két dầu bẩn - Số lượng (chiếc) | text | maxLength 1000 |
| `pollutionSludgeTankTotalVolumeM3` | Két dầu bẩn - Tổng thể tích (m3) | text | maxLength 1000 |
| `pollutionStandardFlangeQuantity` | Bích nối tiêu chuẩn - Số lượng (chiếc) | text | maxLength 1000 |
| `pollutionStandardFlangePosition` | Bích nối tiêu chuẩn - Vị trí | text | maxLength 255 |
| `pollutionGarbageContainerQuantity` | Thiết bị chứa rác - Số lượng (chiếc) | text | maxLength 1000 |
| `pollutionSewageTreatmentQuantity` | Thiết bị xử lý nước thải - Số lượng (chiếc) | text | maxLength 1000 |
| `pollutionSewageTreatmentOutputM3` | Thiết bị xử lý nước thải - Sản lượng (m3) | text | maxLength 1000 |

**18. Thời hạn kiểm tra** (3 field)

| key | Nhãn | kind | Giới hạn |
|---|---|---|---|
| `inspectionPeriodicDate` | Định kỳ | date | — |
| `inspectionDockDate` | Trên đà | date | — |
| `inspectionAnnualDate` | Hàng năm | date | — |



**Bộ số liệu dùng chung BE/FE cho chất lượng còn lại** (FE test ở `ship-quality.spec.ts`; B01 phải dùng đúng bộ này cho `ShipQualityTest` để hai bên khẳng định cùng kết quả). Mỗi dòng là `(w=(1), s=(2), r=(3))`. Cột FE: giá trị hiển thị; cột BE: `ShipQuality.remainingQualityPct` (BigDecimal, `null` nếu không hợp lệ).

| Ca | Các dòng (w; s; r) | (4) FE | (5) FE | Tổng FE | BE trả |
|---|---|---|---|---|---|
| Q1 hai dòng thường | (0,3; 5; 4), (0,7; 3; 2) | `0,8`; `0,7` | `24,00%`; `46,67%` | `70,67%` | `70.67` |
| Q2 (5) dùng raw4 chưa làm tròn | (1; 3; 1) | `0,3` | `33,33%` (nếu dùng (4) đã làm tròn sẽ sai thành `30,00%`) | `33,33%` | `33.33` |
| Q3 làm tròn nửa lên ở (4) | (0,4; 4; 1) | `0,3` (raw4 = 0,25) | `10,00%` | `10,00%` | `10.00` |
| Q4 giá trị chuẩn bằng 0 | (0,5; 0; 1) | rỗng | rỗng | `0,00%` (có dòng nên vẫn hiện) | `null` (không có item hợp lệ) |
| Q5 thiếu (3) | (0,5; 2; null) | rỗng | rỗng | `0,00%` | `null` |
| Q6 tổng vượt 100 | (2; 1; 1) | `1,0` | `200,00%` | `200,00%` | `null` (> 100) |
| Q7 hệ số 0 vẫn hợp lệ | (0; 1; 1) | `1,0` | `0,00%` | `0,00%` | `0.00` (không phải null) |
| Q8 trộn hợp lệ và không hợp lệ | Q1 + (0,5; 0; 1) | như Q1, dòng thêm rỗng | như Q1, dòng thêm rỗng | `70,67%` | `70.67` |
| Q9 không có dòng | (không có) | — | — | rỗng (`qualityTotal` = null) | `null` |

Khác nhau có chủ ý giữa FE và BE: Q4/Q5 (FE hiện `0,00%`, BE `null`) và Q6 (FE hiện tổng thật, BE `null`). FE chỉ hiển thị; giá trị ghi sang tab So sánh luôn do BE tính. Số thực JS: so sánh số bằng `toBeCloseTo(…, 6)`, so sánh chuỗi hiển thị bằng `toBe`.

**Legacy cần đọc trước khi code:** JS l.16–92 (`TEXT_MAX`, `DECIMAL_FIELDS`, `DECIMAL_INT_DIGITS`, `INTEGER_FIELDS`, `BOOLEAN_FIELDS`, `DATE_FIELDS`), JS l.274–386 (công thức và validate), HTML cho từng nhãn.

- [ ] **Step 1: Đọc legacy và F01.** Mở JS/HTML như trên; mở `F api/report-edit/ship-description.ts` (kiểu thật của F01); mở `FL vehicle-description-fields.ts` và `FL __tests__/tabs.spec.ts` làm mẫu phong cách.
- [ ] **Step 2: Viết test `ship-quality.spec.ts` (FAIL).** Tên test:
  - `raw4_thiếu_hoặc_chuẩn_không_dương_là_null` (s = null, 0, -1; r = null).
  - `c4_làm_tròn_một_lẻ_nửa_lên` (raw4 0,25 thành 0,3; 0,8 thành 0,8; 2/3 thành 0,7) và `định_dạng_c4_đúng_một_lẻ_kiểu_de` (`1,0`, `0,3`).
  - `c5_dùng_raw4_chưa_làm_tròn` (Q2 thành `33,33%`, không phải `30,00%`).
  - `bộ_số_liệu_Q1_đến_Q9_khớp_BE` (bảng ở trên, mỗi ca một dòng `it.each`; kiểm `qualityRowMetrics`, `qualityTotal`, `formatQualityPercent`).
  - `tổng_null_khi_không_có_dòng_và_0_khi_không_dòng_nào_tính_được`.
  - `đánh_giá_không_tham_gia_tính` (đổi `rating` không đổi kết quả).
  - `lỗi_có_số_mà_không_tên` (`missingName`), `lỗi_chuẩn_nhỏ_hơn_hoặc_bằng_0` (`invalidStandard` với 0 và -1; null thì không lỗi), `lỗi_số_vượt_8_chữ_số_nguyên` (`numberTooLarge` với 100000000), `dòng_rỗng_hoàn_toàn_không_lỗi`.
  - `chuẩn_hóa_bỏ_dòng_rỗng_trim_tên_rating_rỗng_thành_null` và `chuẩn_hóa_giữ_nguyên_thứ_tự`.
  Chạy: `cd frontend && npx vitest run src/lib/report-edit/__tests__/ship-quality.spec.ts` → FAIL (thiếu module).
- [ ] **Step 3: Cài đặt `ship-quality.ts`** theo Interfaces và công thức JS l.314–335 (dùng `Intl.NumberFormat('de-DE', { minimumFractionDigits, maximumFractionDigits })`). Chạy lại → PASS.
- [ ] **Step 4: Viết test `ship-description-fields.spec.ts` (FAIL).** Tên test:
  - `có_đúng_18_khối_theo_thứ_tự` (so tên khối với danh sách ở trên).
  - `đủ_153_field_không_trùng_key`.
  - `key_trùng_danh_sách_ShipDescriptionInput_trừ_hai_danh_sách`: test khai báo mảng `EXPECTED_KEYS` 153 phần tử (chép từ các bảng trên), so sánh tập hợp với `SHIP_FIELDS.map(f => f.key)`; thêm kiểm tra kiểu `expectTypeOf<ShipFieldKey>().toEqualTypeOf<Exclude<keyof ShipDescriptionInput, 'conditionDetails' | 'qualityEvaluationDetails'>>()`.
  - `đếm_kind`: 70 text/textarea, 23 decimal, 18 integer, 39 boolean, 3 date.
  - `giới_hạn_tiêu_biểu`: `mooringLocation` 255; `shipName` 500; `buildPlace` 1000; `towingBollardType` 1000; `anchorSternTotalWeightKg` text 1000; `anchorBowType` 100; `operationNote` textarea 5000; `deadweight` decimal intDigits 10; `dimLength` decimal intDigits 8; `buildYear` integer max 9999; `grossTonnage` integer max 999999999.
  - `nhãn_tiêu_biểu` (`dimLength` = "L - Chiều dài thiết kế (m)", `mainEnginePowerHp` = "Công suất (HP/KW)", `navigationEpirb` = "S.EPIRB", `fireLayoutPlan` = "Sơ đồ bố trí thiết bị chữa cháy").
  - `mooring_không_nằm_trong_18_khối` và `khối_tín_hiệu_có_ba_nhóm_đèn_âm_vật`.
  - Schema: `schema_text_vượt_giới_hạn_báo_Tối_đa_N_ký_tự`, `schema_decimal_9_chữ_số_nguyên_bị_từ_chối_8_chữ_số_được`, `schema_decimal_10_chữ_số_cho_field_12_2`, `schema_integer_năm_5_chữ_số_bị_từ_chối`, `schema_cho_phép_để_trống_số_và_ngày`, `schema_ngày_sai_định_dạng_bị_từ_chối`.
  - `shipFormDefaults_null_cho_form_rỗng` và `shipFormDefaults_đổi_null_thành_chuỗi_rỗng_và_undefined`.
  - `toShipInput_trim_rỗng_thành_null_boolean_đủ_key` (đúng 155 key, kiểm `Object.keys`), `toShipInput_bỏ_dòng_tình_trạng_rỗng_giữ_dòng_thiếu_hạng_mục`, `toShipInput_chuẩn_hóa_chất_lượng`.
  - `conditionRowInvalid_chỉ_khi_có_tình_trạng_mà_thiếu_hạng_mục`.
- [ ] **Step 5: Cài đặt `ship-description-fields.ts`.** Khai báo `as const satisfies readonly ShipSection[]` để có `ShipFieldKey` literal. Chạy test → PASS.
- [ ] **Step 6: Kiểm tra:** `cd frontend && npm run lint && npm run type-check && npm run test:unit`
- [ ] **Step 7: Commit** (trong submodule): `git -C frontend add src/lib/report-edit && git -C frontend commit -m "feat: hàm thuần chất lượng còn lại và metadata field mô tả tàu"`

---

## F04 — `ShipSpecSections.vue`

**Đợt F3. Phụ thuộc F03.** Tập file rời với F05.

### Task F04: Khối "I. Mô tả tàu thủy" (nơi neo đậu + 18 khối)

**Files:**
- Create `F components/appraisal-report/edit/description/ship/ShipSpecSections.vue`
- Create `F components/appraisal-report/edit/description/ship/__tests__/ShipSpecSections.spec.ts`

**Interfaces:**
- Props: `{ disabled?: boolean }`. Không có v-model: component chạy **bên trong** `useForm` của cha (F06), mỗi ô là một `FormField name=<key>`; không tự tạo `useForm`.
- Render: tiêu đề "I. Mô tả tàu thủy" (`<h2 class="text-base font-semibold">`), ô `SHIP_MOORING_FIELD` ngay dưới tiêu đề, rồi 18 `<section :aria-label="section.title" :data-section="section.id">` theo `SHIP_SECTIONS`; mỗi khối có `<h3 class="text-sm font-semibold">` tên khối và `<h4 class="text-sm font-medium">` cho `group.title`.
- Ánh xạ kind: `text` thành `AutoTextarea single-line` (`:maxlength`, `:counter` khi `maxLength >= 500`); `textarea` thành `AutoTextarea` nhiều dòng; `integer` thành `NumberInput` (`:grouping="field.grouping !== false"`, `:max`); `decimal` thành `DecimalInput` (`:decimals="2"`); `boolean` thành `Checkbox` trong `FormField orientation="horizontal"` (gọi `field['onUpdate:modelValue'](checked === true)`); `date` thành `DatePicker clearable`.
- Lưới: nhóm text/số/ngày `grid gap-5 sm:grid-cols-2 xl:grid-cols-3`; nhóm boolean `grid gap-3 sm:grid-cols-2 xl:grid-cols-4`; `operationNote` chiếm `sm:col-span-2 xl:col-span-3`. Nhóm hỗn hợp (khối 16): boolean trước, text sau, hai lưới riêng.
- Hook cho test: mỗi `FormField` đã có `data-field-name`.

**Đọc trước khi code:** `F components/form/FormField.vue`, `AutoTextarea.vue`, `DecimalInput.vue`, `NumberInput.vue`, `DatePicker.vue`; `FC CarDescriptionForm.vue` (mẫu lưới và Checkbox); `frontend/DESIGN_SYSTEM.md` §12, §13.

**Test harness trong spec:** component `Host` (defineComponent) gọi `useForm({ validationSchema: toTypedSchema(buildShipSchema()), initialValues: shipFormDefaults(data) })` rồi render `<ShipSpecSections />`, phơi `values`, `errors`, `validate`. Polyfill `ResizeObserver` như `CarDescriptionTab.spec.ts`. Helper `control(wrapper, label)` giống spec của Car.

- [ ] **Step 1: Viết test (FAIL).**
  - `hiện_tiêu_đề_và_18_khối_theo_thứ_tự` (đếm `section[data-section]` = 18, tên theo `SHIP_SECTIONS`).
  - `nơi_neo_đậu_nằm_trước_khối_thông_số_chính` (so thứ tự DOM).
  - `hiện_đủ_153_nhãn` (mỗi `SHIP_FIELDS[i].label` có đúng một `<label>`).
  - `khối_tín_hiệu_có_ba_tiêu_đề_phụ` ("Đèn hiệu", "Âm hiệu", "Vật hiệu") và `khối_chữa_cháy_có_tiêu_đề_phụ_dụng_cụ_và_sơ_đồ`.
  - `ô_text_có_maxlength_theo_metadata` (`shipName` 500, `towingBollardType` 1000, `mooringLocation` 255).
  - `text_chặn_xuống_dòng_còn_ghi_chú_hoạt_động_cho_phép` (Enter không thêm dòng ở `shipName`; `operationNote` thì cho).
  - `nhập_thập_phân_kiểu_VN_ra_số` (gõ `1234,5` vào "L - Chiều dài thiết kế (m)" thì `values.dimLength === 1234.5`, hiển thị `1.234,5`).
  - `thập_phân_vượt_phần_nguyên_báo_lỗi` (`dimLength` 123456789 báo "Tối đa 99.999.999,99"; `deadweight` 123456789 hợp lệ).
  - `năm_không_nhóm_nghìn` (`buildYear` 2015 hiển thị `2015`, không `2.015`).
  - `checkbox_bật_tắt_ghi_vào_form` (`signalLightSideRed`) và `ngày_chọn_ghi_ISO` (đặt qua `setFieldValue`, DatePicker hiển thị dd/MM/yyyy).
  - `disabled_khóa_mọi_ô` (prop `disabled`).
  Chạy: `cd frontend && npx vitest run src/components/appraisal-report/edit/description/ship/__tests__/ShipSpecSections.spec.ts` → FAIL.
- [ ] **Step 2: Cài đặt `ShipSpecSections.vue`.** Duyệt metadata bằng `v-for`; không hardcode field trong template. Javadoc đầu file: "Khối I của tab Mô tả tàu thủy (`ship-description.html` cũ)", kèm "Khác hệ thống cũ" mục 1, 3, 5 ở trên. Chạy lại → PASS.
- [ ] **Step 3: Kiểm tra:** `cd frontend && npm run lint && npm run type-check && npm run test:unit`
- [ ] **Step 4: Commit:** `git -C frontend add src/components/appraisal-report/edit/description/ship && git -C frontend commit -m "feat: khối mô tả tàu thủy theo metadata"`

---

## F05 — `ShipConditionTable.vue`, `ShipQualityTable.vue`

**Đợt F3. Phụ thuộc F03.** Tập file rời với F04.

### Task F05: Hai bảng tình trạng và chất lượng còn lại

**Files:**
- Create `F components/appraisal-report/edit/description/ship/ShipConditionTable.vue`
- Create `F components/appraisal-report/edit/description/ship/ShipQualityTable.vue`
- Create `F components/appraisal-report/edit/description/ship/__tests__/ShipConditionTable.spec.ts`
- Create `F components/appraisal-report/edit/description/ship/__tests__/ShipQualityTable.spec.ts`

**Interfaces:**

```ts
// ShipConditionTable.vue
defineProps<{ disabled?: boolean; showErrors?: boolean }>()
const rows = defineModel<ShipConditionItem[]>({ required: true })   // v-model
// ShipQualityTable.vue
defineProps<{ disabled?: boolean; showErrors?: boolean }>()
const items = defineModel<ShipQualityItem[]>({ required: true })    // v-model
```

- Cả hai **không** nằm trong vee-validate: danh sách do cha giữ bằng `v-model`; lỗi hiện khi `showErrors` bật (cha bật sau lần lưu thất bại). Sửa một phần tử trên **bản sao** rồi phát `update:modelValue` (mẫu `edit()` của `LandDetailTable.vue`).
- `ShipConditionTable`: tiêu đề `<h3>` "II. Tình trạng của tài sản thẩm định" và `Button` "Thêm hạng mục" (`variant="outline"`, icon `Plus`) ở thanh trên. Bảng `@/components/ui/table`: STT (`text-center tabular-nums`, tự đánh theo vị trí), Hạng mục (`AutoTextarea` single-line `maxlength=500`), Tình trạng (`AutoTextarea` nhiều dòng, không giới hạn), Thao tác (`IconButton label="Xóa dòng" tone="danger"`, icon `Trash2`, xóa ngay không hỏi). Dòng lỗi (`conditionRowInvalid`) khi `showErrors`: ô Hạng mục `aria-invalid="true"` và dòng chữ lỗi `text-destructive` "Vui lòng nhập hạng mục". Dòng thêm mới: `{ itemName: '', conditionDescription: '' }`. Rỗng thì hiện `TableEmpty` "Chưa có hạng mục".
- `ShipQualityTable`: tiêu đề "III. Đánh giá chất lượng còn lại của tàu" và nút "Thêm dòng". Cột (đúng 9 ô trong hàng thân): STT; Hạng mục thẩm định; (1) Hệ số quan trọng; (2) Giá trị chuẩn so sánh; Đánh giá (select); (3) giá trị; (4)=(3)/(2); (5)=(1)*(4); thao tác. Tiêu đề cột "Đánh giá (3)" có `colspan=2`. Ô "Hạng mục thẩm định": danh sách `AutoTextarea` single-line (`maxlength=500`, `data-testid="quality-name"`), mỗi ô một `IconButton label="Xóa hạng mục"` (icon `X`), dưới cùng `Button size="sm" variant="outline"` tên "Hạng mục" kèm icon `Plus` (hiển thị "+ Hạng mục"). Xóa ô cuối cùng chỉ xóa nội dung. Ô số (1), (2), (3): `DecimalInput` `decimals=2`, căn phải; map `null` và `undefined` hai chiều. Select đánh giá: `SelectBox` với `SHIP_RATINGS` và một option đầu "-- Chọn --" có value hằng `RATING_NONE = '__none__'` ánh xạ về `rating: null` (reka Select không cho value rỗng). (4): `formatQualityC4(qualityRowMetrics(item).c4)`; (5): `formatQualityPercent(c5)`; tính trực tiếp trong computed qua `ship-quality.ts`, không viết lại công thức. Rỗng: một hàng `TableEmpty :colspan="9"` "Chưa có dữ liệu". `TableFooter`: hàng "Chất lượng còn lại của tàu" `colspan=7` căn giữa đậm, ô tổng `data-testid="quality-total"` = `formatQualityPercent(qualityTotal(items))` (rỗng khi không có dòng), ô cuối trống. Nút xóa dòng: `IconButton label="Xóa dòng" tone="danger"`, **xóa ngay không hỏi xác nhận**.
- Lỗi khi `showErrors`, theo `qualityItemErrors(item)`: `missingName` thì ô tên đầu tiên `aria-invalid`; `invalidStandard` hoặc `numberTooLarge` thì ô (2) `aria-invalid`; `numberTooLarge` ở (1)/(3) thì ô đó `aria-invalid`; kèm dòng chữ lỗi nhỏ trong ô ("Cần nhập hạng mục" / "Phải lớn hơn 0" / "Tối đa 8 chữ số nguyên").
- Bảng rộng: bọc `overflow-auto`; không `rounded-*` quanh bảng (CLAUDE.md quy tắc 10); không dùng kích thước tùy ý kiểu `min-w-[900px]`, dùng lớp thang `min-w-*`.

**Đọc trước khi code:** `FC LandDetailTable.vue` + `__tests__/LandDetailTable.spec.ts` (mẫu v-model + test), `F components/actions/IconButton.vue`, `F components/form/SelectBox.vue`, `F components/ui/table/*`, `DESIGN_SYSTEM.md` §10.7, §11. JS l.219–386 và HTML l.973–1095.

- [ ] **Step 1: Viết `ShipConditionTable.spec.ts` (FAIL).**
  - `đánh_lại_STT_khi_thêm_xóa` (thêm 2 dòng, xóa dòng giữa, cột đầu là 1, 2).
  - `thêm_hạng_mục_phát_dòng_rỗng_ở_cuối`.
  - `xóa_dòng_không_hỏi_xác_nhận` (không mở dialog; phát danh sách thiếu dòng đó).
  - `sửa_hạng_mục_và_tình_trạng_phát_bản_sao` (không đổi mảng gốc).
  - `bắt_buộc_hạng_mục_khi_có_tình_trạng_hiện_lỗi_khi_showErrors` (dòng `{ itemName: '', conditionDescription: 'Han gỉ' }`: `aria-invalid` và chữ "Vui lòng nhập hạng mục" chỉ có khi `showErrors`; dòng cả hai rỗng hoặc đủ cả hai thì không lỗi).
  - `hạng_mục_tối_đa_500_ký_tự` (`maxlength`), `rỗng_hiện_thông_báo`, `disabled_khóa_ô_và_nút`.
- [ ] **Step 2: Viết `ShipQualityTable.spec.ts` (FAIL).**
  - `rỗng_hiện_Chưa_có_dữ_liệu_và_không_hiện_tổng`.
  - `thêm_dòng_có_một_ô_tên_rỗng` (`emptyQualityItem`).
  - `thêm_hạng_mục_thêm_ô_tên_vào_đúng_dòng` và `xóa_ô_tên_giữ_ít_nhất_một_ô` (xóa ô cuối cùng thì còn 1 ô, nội dung rỗng).
  - `xóa_dòng_không_hỏi`.
  - `tính_c4_c5_và_tổng_trực_tiếp_theo_bộ_Q1` (modelValue = Q1: hiển thị `0,8`, `0,7`, `24,00%`, `46,67%`, tổng `70,67%`).
  - `c5_dùng_raw4_chưa_làm_tròn_Q2` (hiển thị `0,3` và `33,33%`).
  - `chuẩn_bằng_0_không_tính_nhưng_vẫn_hiện_tổng_0` (Q4: ô (4), (5) rỗng, tổng `0,00%`).
  - `nhập_ô_số_phát_số_thập_phân` (gõ `0,3` vào (1) thì `importanceWeight === 0.3`; xóa hết thì `null`).
  - `chọn_đánh_giá_ghi_rating_và_không_đổi_kết_quả_tính` (chọn "Khá" thì `rating: 'Khá'`; chọn "-- Chọn --" thì `null`; tổng không đổi).
  - `select_có_đủ_bốn_mức` (Tốt, Khá, Trung bình, Kém).
  - `lỗi_khi_showErrors`: `thiếu_tên_có_số_liệu`, `chuẩn_nhỏ_hơn_hoặc_bằng_0`, `số_vượt_8_chữ_số_nguyên`; không `showErrors` thì không lỗi.
  - `tên_tối_đa_500_ký_tự`, `disabled_khóa_mọi_ô`.
  Polyfill `ResizeObserver`.
  Chạy: `cd frontend && npx vitest run src/components/appraisal-report/edit/description/ship/__tests__/ShipConditionTable.spec.ts src/components/appraisal-report/edit/description/ship/__tests__/ShipQualityTable.spec.ts` → FAIL.
- [ ] **Step 3: Cài đặt `ShipConditionTable.vue`** (dùng `conditionRowInvalid` từ F03). Chạy spec của nó → PASS.
- [ ] **Step 4: Cài đặt `ShipQualityTable.vue`** (dùng các hàm trong `ship-quality.ts`, **không** tự viết công thức trong component). Chạy spec của nó → PASS.
- [ ] **Step 5: Kiểm tra:** `cd frontend && npm run lint && npm run type-check && npm run test:unit`
- [ ] **Step 6: Commit:** `git -C frontend add src/components/appraisal-report/edit/description/ship && git -C frontend commit -m "feat: bảng tình trạng và bảng chất lượng còn lại của tàu"`

---

## F06 — Lắp `ShipDescriptionTab.vue`

**Đợt F4. Phụ thuộc F04, F05.**

### Task F06: Tab "Mô tả tàu thủy" hoàn chỉnh

**Files:**
- Modify `F components/appraisal-report/edit/description/ShipDescriptionTab.vue` (vỏ do F01 tạo; thay bằng bản đầy đủ)
- Create `F components/appraisal-report/edit/description/__tests__/ShipDescriptionTab.spec.ts`
- Không sửa `GroupWorkspace.vue` (F01 đã gắn tab).

**Interfaces:**
- Props: `{ profileId: number; reportId: number; group: ReportWorkspaceGroup }` (đúng quy ước chung).
- Một file (tải + form), theo mẫu `CarDescriptionTab.vue` + `CarDescriptionForm.vue`:
  1. `load()` song song `getShipDescription(profileId, reportId, group.id, signal)` và `listImages(...)`; có `AbortController`, `Skeleton` khi tải, `Alert` + nút "Thử lại" khi lỗi (message `errorMessage(failure, 'Không tải được mô tả tàu thủy')`), hủy khi unmount.
  2. Form: `useForm({ validationSchema: toTypedSchema(buildShipSchema()), initialValues: shipFormDefaults(data) })`; `conditions = ref<ShipConditionItem[]>`, `quality = ref<ShipQualityItem[]>` (sao chép sâu từ `data.conditionDetails ?? []`, `data.qualityEvaluationDetails ?? []`).
  3. Dirty: `useDirtyTab(computed(() => meta.value.dirty || tablesDirty.value))`, với `tablesDirty` so `JSON.stringify` của `normalizeConditionRows(conditions)` cộng `normalizeQualityItems(quality)` với bản chụp `baseline` lấy lúc tải/sau khi lưu.
  4. Bố cục: `<form class="flex flex-col gap-6" novalidate @submit.prevent="onSubmit">`: `FormErrorSummary` (`provideFieldLabels()`, mẫu `AppraisalReportBasicForm.vue`), `<ShipSpecSections :disabled="isSubmitting" />`, `<ShipConditionTable v-model="conditions" :show-errors="showTableErrors" :disabled="isSubmitting" />`, `<ShipQualityTable v-model="quality" ... />`, khối "IV. Hình ảnh" (xem dưới), `<FormActions sticky>` với `DownloadReportButton` và `Button type="submit"` "Lưu" (đang lưu thì "Đang lưu…", disabled).
  5. Khối ảnh: render khi đã tải được mô tả (GET 404 thì chỉ hiện Alert, Quyết định #9). 11 `ImageSection` theo `SHIP_IMAGE_TYPES` và `imageTypeLabel` của F01 (không tự khai báo mảng khác), thứ tự và tiêu đề: 21 "Mặt trước tàu", 22 "Đầu tàu", 23 "Đuôi tàu", 24 "Mạn tàu", 25 "Sàn tàu", 26 "Số IMO/Số kiểm soát", 27 "Hệ thống neo", 28 "Phòng lái, phòng nghỉ", 29 "Khoang hàng", 30 "Hệ thống máy của động cơ", rồi `IMAGE_TYPE.OTHER` "Khác"; hằng kiểu lấy từ `IMAGE_TYPE.SHIP_*` của F01 (Quyết định #14). Mỗi `ImageSection` nhận `v-bind="imageProps"` và `@changed="reloadImages"` (mẫu Car); tiêu đề nhóm `<h2>` "IV. Hình ảnh".
  6. Lưu (`onSubmit`): (a) tính `conditionBad = conditions.some(conditionRowInvalid)` và `qualityBad = hasQualityErrors(quality)`; nếu có lỗi thì đặt `showTableErrors = true`, `toast.error(SHIP_CONDITION_ERROR)` (ưu tiên lỗi tình trạng) hoặc `toast.error(QUALITY_ERROR_MESSAGE)` và **không gọi API**; (b) còn lại `handleSubmit`: `saveShipDescription(profileId, reportId, group.id, toShipInput(form, conditions, quality))`; thành công thì `toast.success('Lưu mô tả tàu thủy thành công')`, tải lại `getShipDescription` + `listImages` **tại chỗ** (không hiện Skeleton, giữ vị trí cuộn), `resetForm({ values: shipFormDefaults(fresh) })`, đặt lại `conditions`, `quality`, `baseline`, `showTableErrors = false`; thất bại thì `toast.error(errorMessage(failure, 'Lưu mô tả tàu thủy không thành công, vui lòng thử lại'))` và giữ nguyên dữ liệu đang nhập. Submit không hợp lệ (zod): `FormErrorSummary` hiện và focus (mẫu `AppraisalReportBasicForm.vue`).
- Javadoc đầu file: nêu màn cũ port (`ship-description.html`, `ship-description.js`) và các điểm "Khác hệ thống cũ" mục 1–5 ở đầu file này.

**Đọc trước khi code:** `FC CarDescriptionTab.vue`, `CarDescriptionForm.vue`, `__tests__/CarDescriptionTab.spec.ts`; `F composables/useDirtyGuard.ts` + `__tests__/useDirtyGuard.spec.ts`; `F components/appraisal-report/AppraisalReportBasicForm.vue` (cách dùng `FormErrorSummary`); file F01 (kiểu, tên `IMAGE_TYPE.SHIP_*`, vỏ tab).

**Test harness:** `vi.mock('@/api/report-edit/images')`, `vi.mock('@/api/report-edit/ship-description')`, `vi.mock('@/composables/useToast')`; polyfill `ResizeObserver`; hàm `settle()` như spec của Car; fixture `ship(overrides)` đủ mọi key của `ShipDescriptionData` (lấy từ `shipFormDefaults(null)` cộng `id`, `conditionDetails`, `qualityEvaluationDetails`, `remainingQualityPct`); bọc tab trong component cha có `provideDirtyGuard()` để test hỏi xác nhận (dùng `ConfirmHost` như `useDirtyGuard.spec.ts` hoặc mock `confirmDiscard`).

- [ ] **Step 1: Viết `ShipDescriptionTab.spec.ts` (FAIL).** Tên test:
  - `nạp_mô_tả_và_điền_form` (GET gọi với `(5, 11, 21, AbortSignal)`; "Tên phương tiện", "Năm đóng", "Nơi neo đậu"… hiển thị đúng; hai bảng có dữ liệu).
  - `lỗi_tải_hiện_thông_báo_và_thử_lại`.
  - `lưu_gửi_json_đúng_trường`: sửa "Tên phương tiện" (có khoảng trắng đầu/cuối), "L - Chiều dài thiết kế (m)" = `12,5`, bật "Đèn mạn đỏ", "Năm đóng" = 2015, thêm một dòng tình trạng đầy đủ và một dòng rỗng, một item chất lượng đủ số, bấm Lưu: `saveShipDescription` được gọi với `(5, 11, 21, input)`; `input` có đúng 155 key, `shipName` đã trim, `dimLength === 12.5`, `signalLightSideRed === true`, `buildYear === 2015`, text để trống là `null`, số để trống là `null`, boolean không tích là `false`; `conditionDetails` đã bỏ dòng rỗng; `qualityEvaluationDetails` đã trim tên, bỏ tên rỗng, `rating` rỗng thành `null`.
  - `lưu_xong_toast_và_tải_lại` (`toast.success` "Lưu mô tả tàu thủy thành công"; GET gọi lần hai; form hiển thị dữ liệu mới; không còn dirty).
  - `lưu_lỗi_toast_và_giữ_dữ_liệu` (reject `ApiError` thì `toast.error` đúng message; giá trị đang nhập còn nguyên).
  - `404_chỉ_hiện_thông_báo_không_form_không_ảnh` (Quyết định #9: GET reject `ApiError` 404 "Chưa có mô tả tàu thủy" thì chỉ có `Alert` + nút "Thử lại", không có form, không có `ImageSection`, không có tiêu đề "IV. Hình ảnh").
  - `hiện_đủ_11_khối_ảnh_khi_đã_có_mô_tả` (đúng thứ tự tiêu đề ở trên; `type` truyền xuống là 21–30 rồi 10).
  - `chặn_lưu_khi_tình_trạng_thiếu_hạng_mục` (dòng `{ itemName: '', conditionDescription: 'Han gỉ' }` thì không gọi API; `toast.error('Vui lòng nhập hạng mục cho các dòng tình trạng đã nhập')`; ô hạng mục `aria-invalid`).
  - `chặn_lưu_khi_giá_trị_chuẩn_bằng_0` (item `standardValue: 0` thì không gọi API; toast "Đánh giá chất lượng: vui lòng nhập tên hạng mục và giá trị chuẩn so sánh lớn hơn 0"; ô (2) `aria-invalid`).
  - `chặn_lưu_khi_có_số_liệu_mà_không_có_tên_hạng_mục`.
  - `chặn_lưu_khi_text_vượt_giới_hạn` (gán 256 ký tự vào "Nơi neo đậu" bằng `setValue` thì không gọi API, `FormErrorSummary` hiện).
  - `hỏi_xác_nhận_khi_rời_tab_chưa_lưu`: bọc trong cha có `provideDirtyGuard()`; sửa một ô rồi `await guard.confirmLeave()` thì hộp xác nhận "Bỏ các thay đổi chưa lưu?" xuất hiện; không sửa gì thì `confirmLeave()` trả `true` ngay; chỉ thêm một dòng tình trạng (không đổi field) cũng làm dirty; lưu xong thì hết dirty.
  - `tab_bị_hủy_thì_hủy_yêu_cầu_đang_tải` (unmount thì `AbortSignal.aborted`).
  Chạy: `cd frontend && npx vitest run src/components/appraisal-report/edit/description/__tests__/ShipDescriptionTab.spec.ts` → FAIL.
- [ ] **Step 2: Cài đặt `ShipDescriptionTab.vue`** theo Interfaces (giữ vỏ F01 làm điểm bắt đầu; không đổi props). Chạy spec → PASS.
- [ ] **Step 3 (tùy chọn): Rà soát giao diện thủ công:** `cd frontend && npm run dev`, mở báo cáo có nhóm tàu, tab "Mô tả tàu thủy": đủ 18 khối, bảng II/III, ảnh, lưu, đổi ô rồi chuyển tab thấy hộp xác nhận. Ghi kết quả vào báo cáo task.
- [ ] **Step 4: Kiểm tra:** `cd frontend && npm run lint && npm run type-check && npm run test:unit`
- [ ] **Step 5: Commit:** `git -C frontend add src/components/appraisal-report/edit/description && git -C frontend commit -m "feat: tab mô tả tàu thủy hoàn chỉnh (form, bảng, ảnh, lưu)"`

---

## Review Focus (spec reviewer / code reviewer của cả F03–F06)

1. 153 field khớp tuyệt đối với legacy: key, nhãn, kind, `maxLength` (JS là chuẩn), `intDigits` 8/10; `anchorSternTotalWeightKg` là text; 3 field năm tối đa 4 chữ số.
2. Công thức chất lượng chỉ tồn tại ở `ship-quality.ts` (không trùng lặp trong component); (4) làm tròn chỉ để hiển thị, (5) dùng raw4.
3. Payload `ShipDescriptionInput` đủ 155 key, chuẩn hóa đúng (trim, rỗng thành null, bỏ dòng rỗng) và không làm mất dòng thiếu hạng mục trước khi chặn.
4. Dirty guard hoạt động cho cả field lẫn hai bảng; lưu xong hết dirty.
5. Tuân thủ `frontend/CLAUDE.md`: token màu semantic, không control native, `IconButton` cho nút icon, bảng `ui/table` không bo góc, `FormField`/`FormActions`/`FormErrorSummary`, không `alert()`/`confirm()`.

## Vấn đề cần controller quyết định

1. **→ Đã chốt: Quyết định #14.** Hằng ảnh FE là `IMAGE_TYPE.SHIP_FRONT, SHIP_BOW, SHIP_STERN, SHIP_HULL_SIDE, SHIP_DECK, SHIP_IMO_NUMBER, SHIP_ANCHOR_SYSTEM, SHIP_WHEELHOUSE_CABIN, SHIP_CARGO_HOLD, SHIP_ENGINE_SYSTEM` (21–30), trùng tên BE và đã do F01 tạo (kèm `SHIP_IMAGE_TYPES` và `IMAGE_TYPE_LABELS`). F06 dùng đúng các tên này, không dùng `SHIP_SIDE`/`SHIP_CABIN`. Nhãn 28 là "Phòng lái, phòng nghỉ".
2. **→ Đã chốt: Quyết định #9.** GET mô tả tàu khi chưa có bản ghi trả 404 (`BusinessException`), như mô tả xe; FE hiển thị trạng thái lỗi (Alert + Thử lại), không tự tạo. `ShipDescriptionData.id` luôn khác null khi tải được; test `ảnh_ẩn_khi_chưa_có_mô_tả` đổi thành `404_chỉ_hiện_thông_báo_không_form_không_ảnh`.
3. **→ Đã đối chiếu với F01 (`10-fe-foundation.md`):** `ShipConditionItem { itemName: string | null; conditionDescription: string | null }`, `ShipQualityItem { itemNames: string[]; importanceWeight/standardValue/relativeValue: number | null; rating: string | null }`. F03/F05 phải xử lý `itemName === null` như chuỗi rỗng (trim, kiểm tra rỗng, `AutoTextarea` nhận `?? ''`).
4. **(Chưa chốt, giữ nguyên.)** **Lỗi `numberTooLarge` ở bảng III**: plan thêm lỗi này (> 99.999.999,99) vì BE có `@Digits(8,2)` (legacy chặn từng phím). Đây là bổ sung nhỏ so với legacy; nếu muốn bám sát hoàn toàn thì bỏ nhánh này.
5. **→ Đã chốt: Quyết định #12.** Bộ Q1–Q9 ở F03 là bộ dùng chung; `01-be-foundation.md` Step 3.1 đã được bổ sung ca `bộ_số_liệu_chung_Q1_đến_Q9`.
