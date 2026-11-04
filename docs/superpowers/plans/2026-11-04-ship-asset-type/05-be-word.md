# 05 — Word xuất báo cáo cho SHIP (B08–B09)

> Đọc `00-index.md` trước, sau đó `surveys/legacy-ui-word.md` (mục 8 Word) và `surveys/new-code.md` (mục Word). Subagent: `model: "sonnet"`, mỗi task qua spec review rồi code review.
>
> Quy ước viết tắt (thêm so với `00-index.md`):
> - **WN** = `N/common/util/export/word/appraisal` (gói exporter mới). **WNT** = `NT/common/util/export/word/appraisal`.
> - **LW** = file `AppraisalReportWordExporter.java` của L. Đọc từng đoạn bằng `git -C td-project-legacy show 82193fe5:src/main/java/com/jvb/thanhdo/controller/component/wordExport/AppraisalReportWordExporter.java | sed -n 'a,bp'`.
> - Không sửa exporter cũ `N/common/util/export/word/AppraisalReportWordExporter.java` (bản golden). Không thêm `if (assetType == …)` vào section, chỉ thêm loại qua `AssetTypeWriter` + `AssetTypeWriters` (xem `backend/CLAUDE.md`, dòng "Thêm loại tài sản / mục vào báo cáo Word thẩm định").

## Nền tảng chung cho cả hai task

### Điều kiện đầu vào (các task khác phải có sẵn)

B08 chạy ở đợt B2, chỉ cần B01. B09 chạy ở đợt B3, cần B01, B06 và B08. Các tên dưới đây lấy theo quy ước của `00-index.md`. **Trước khi code, implementer grep lại tên thật** và sửa plan nếu khác (ghi vào báo cáo task):

| Cần | Tên giả định | Do task |
|---|---|---|
| Entity | `AppraisalShipDescription` (`@Getter`, getter đặt đúng theo tên field của L, ví dụ `getShipName`, `getBuildYear`, `getDimLength`, `getSignalLightSideRed`…) | B01 |
| JSON con | record `ShipConditionDetail(String itemName, String conditionDescription)` và `ShipQualityEvaluationItem(List<String> itemNames, BigDecimal importanceWeight, BigDecimal standardValue, String rating, BigDecimal relativeValue)`, accessor kiểu record (`itemName()`, `itemNames()`…) | B01 |
| Quan hệ | `AppraisalAssetGroup.getAppraisalShipDescription()`, `getCurrencyCode()`, `getCurrencyNote()` | B01 |
| So sánh | `AppraisalGroupCompare.getShipBuildYear()`, `getShipBuildPlace()`, `getShipUsagePurpose()`, `getShipClass()`, `getShipRemainingQualityPct()`, `getShipDeadweight()`, `getShipDimension()`, `getShipMainEnginePowerHp()` | B01 |
| Hằng số | `CommonConstants.AppraisalAssetType.SHIP = 4`; `ReportConstants.AppraisalImageType.SHIP_FRONT=21, SHIP_BOW=22, SHIP_STERN=23, SHIP_HULL_SIDE=24, SHIP_DECK=25, SHIP_IMO_NUMBER=26, SHIP_ANCHOR_SYSTEM=27, SHIP_WHEELHOUSE_CABIN=28, SHIP_CARGO_HOLD=29, SHIP_ENGINE_SYSTEM=30` | B01 |
| Tiền tệ | `common/util/CurrencyCodes`: `resolveCurrencyCode(Integer assetType, String code)`, `currencyUnitLabel(String)`, `currencyMoneyLabel(String)`, `regulatoryPriceUnit(Integer assetType, String code)`, hằng `VND` | B01 |
| Lớp thuần | `service/ship/ShipQuality.remainingQualityPct(List<ShipQualityEvaluationItem>)` (chỉ để so khớp trong test) | B01 |
| Kết quả | `EvaluationResultDetailDto.getShip()` kiểu `ShipEvaluationDetail { Long amount, exchangeRate, vndAmount, roundedAmount }` (`@Data`), do `AppraisalResultService` (Word) dựng cho nhóm SHIP | B06 |

### Bẫy kỹ thuật của khung Word hiện có (đọc trước khi viết test)

- `createParagraphAttr` luôn `replaceAll("m2", "m²")` trên mọi đoạn và ô. Vì vậy chuỗi nguồn `"tổng diện tích (m2)"` ra file là `"tổng diện tích (m²)"`; `"m3"` giữ nguyên `"m3"`; riêng bơm chữa cháy nguồn đã là `"m³"`. **Test phải kỳ vọng chuỗi đã qua phép thay này.**
- `writeCell(cell, text, attr)` tách `text` theo `\n`, mỗi dòng một đoạn. `text == null` thì không làm gì. Chuỗi `""` tạo một đoạn rỗng.
- `mergeCellHorizontally/Vertically` **xóa nội dung** các ô nối tiếp (chỉ ô đầu giữ chữ) và đặt `hMerge`/`vMerge` RESTART (ô đầu) và CONTINUE (các ô sau). Gộp ngang bằng `gridSpan` chỉ có trong bảng lưới đèn hiệu (do code tự đặt).
- Số mục như `6.2.` nằm trong định nghĩa đánh số của đoạn (`bulletChar` của `createSecondLvSection`), không nằm trong `getText()`. Đọc bằng `paragraph.getNumLevelText()`; nếu POI không trả đúng thì spy `WordStyles.createSecondLvSection` và bắt tham số.
- Mockito trả `false`/`0` cho getter kiểu wrapper (`Boolean`, `Integer`) chưa stub. Entity tàu có rất nhiều `Boolean` mà quy tắc "null thì bỏ dòng" cần phân biệt với `false`. **Không dùng `mock(AppraisalShipDescription.class)`.** Dùng thực thể thật tạo bằng `BeanUtils.instantiateClass(AppraisalShipDescription.class)` rồi gán field bằng `ReflectionTestUtils.setField` (xem helper bên dưới).
- `AppraisalWordExporter` dựng được trong test bằng `new AppraisalWordExporter(report, null, null, List.of(), null, null, Map.of(), false)`; `exporter.getDocument()` là `XWPFDocument` trong bộ nhớ. `ReportWordContext` dựng bằng `new ReportWordContext(exporter, report, null, null, List.of(), s3, resultService, Map.of(), false)`. `report` là `mock(AppraisalReport.class)` với `getAppraisalAssetGroups()` trả `new ArrayList<>(...)`.

### Helper test dùng chung (tạo ở B08, B09 dùng lại)

Tạo trong `WNT`, không phải mã sản phẩm:

- `WordTestKit` (public final):
  - `static ReportWordContext newContext(AppraisalAssetGroup... groups)`: dựng report mock, exporter, context như trên, `s3Service`/`appraisalResultService` là `null`.
  - `static ReportWordContext newContext(S3Service s3, AppraisalResultService svc, AppraisalAssetGroup... groups)`.
  - `static XWPFDocument docOf(ReportWordContext ctx)`.
  - `static List<String> paragraphTexts(XWPFDocument doc)`: text các đoạn thuộc thân tài liệu (không gồm đoạn trong bảng), theo thứ tự.
  - `static List<IBodyElement> body(XWPFDocument doc)`: `doc.getBodyElements()`, dùng để kiểm tra thứ tự đoạn xen bảng.
  - `static List<List<String>> cells(XWPFTable t)`: text từng ô theo hàng, mỗi ô nối các đoạn bằng `\n`.
  - `static String numberOf(XWPFParagraph p)`: `p.getNumLevelText()`.
  - `static byte[] tinyPng()`: PNG 1x1 (lấy cách tạo từ `AppraisalWordExporterParityTest.tinyPng`).
- `ShipFixtures` (public final, `WNT/asset/ship/`):
  - `static AppraisalShipDescription blank()`: `BeanUtils.instantiateClass`, rồi đặt `conditionDetails` và `qualityEvaluationDetails` là `new ArrayList<>()`, các field khác để null.
  - `static AppraisalShipDescription with(AppraisalShipDescription s, String field, Object value)`: `ReflectionTestUtils.setField`, trả `s` để nối chuỗi.
  - `static ShipQualityEvaluationItem item(List<String> names, String w, String s, String rating, String r)`: chuỗi số `null` hoặc `"0.3"` đổi thành `BigDecimal`. Mọi test đi qua factory này nên đổi thứ tự tham số của record chỉ phải sửa một nơi.
  - `static ShipConditionDetail cond(String name, String desc)`.
  - `static AppraisalAssetGroup shipGroup(AppraisalShipDescription s)`: `mock(AppraisalAssetGroup.class)` với `getAssetType()=4`, `getAppraisalShipDescription()=s`, `getAppraisalReportAssets()` rỗng, `getCurrencyCode()="VND"`.

Test đầu tiên của B08 kiểm chứng giả định của fixture (`fixture_blank_mọi_boolean_là_null`), vì mọi test sau dựa vào nó.

---

# B08 — Word phần 1: mô tả tàu, tình trạng, đánh giá chất lượng

## Hành vi hệ thống cũ

Nguồn: LW `createShipDescription` (l.6804–7064), `writeShipConditionTable` (l.7067–7152), `writeShipQualityTable` (l.7155–7262), `shipQualityRow` (l.7285–7305), `shipFixed2` (l.7308), `writeShipSection` (l.7318), `addShipLine` (l.7329), `addShipYesNoCell` (l.7338), `writeShipYesNoGrid` (l.7350–7428), các helper `shipText/shipNum/shipUnit/shipYesNo/shipDate/shipJoin/shipSegments` (l.7437–7497).

### Helper (chép nguyên hành vi)

| Helper | Hành vi |
|---|---|
| `shipText(s)` | null hoặc rỗng sau trim thì `null`; ngược lại chuỗi đã trim |
| `shipNum(BigDecimal)` | null thì null; ngược lại `formatFloatToVietnameseStyle(double)` (mẫu `#,##0.##`, ví dụ 1234.5 thành `1.234,5`, 10.00 thành `10`) |
| `shipNum(Integer)` | null thì null; ngược lại `String.valueOf` (không có phân cách nghìn: 1599 thành `1599`) |
| `shipUnit(v, unit)` | `v == null` thì null; ngược lại `v + " " + unit` |
| `shipYesNo(Boolean)` | null thì null; true thì `có`; false thì `-/-` |
| `shipDate(LocalDate)` | null thì null; ngược lại `dd/MM/yyyy` |
| `shipJoin(sep, parts…)` | nối các phần bằng `sep`; phần null ghi `-/-`; **tất cả null thì trả null** |
| `shipSegments(label1, value1, label2, value2, …)` | mỗi cặp ra `label: value` (label null thì chỉ in value); value null ghi `-/-`; nối bằng `; `; tất cả value null thì trả null |
| `addShipLine(lines, prefix, value)` | value null thì bỏ; ngược lại thêm `prefix + value`, và thêm `.` nếu chuỗi chưa kết thúc bằng `.` hoặc `:` |
| `writeShipSection(title, lines)` | danh sách rỗng thì **bỏ cả tiêu đề**; ngược lại tiêu đề = `createCommonDiamondAttr()` in đậm, mỗi dòng một đoạn `createCommonBulletAttr()` |

### Ghi mô tả tàu — 18 khối theo thứ tự

Kiểu `Text` = `shipText`, `Num` = `shipNum`, `Seg` = `shipSegments`, `Join` = `shipJoin`, `Unit` = `shipUnit`. Dòng nào nhận giá trị null thì bỏ.

Trước 18 khối, `createShipDescription` (l.6804–6817): nếu nhóm null hoặc `assetType != SHIP` thì trả `sectionIndex` ngay. **Một nhóm (`!hasManyGroups`)** in đoạn `Mô tả tài sản thẩm định` với `createSecondLvSection(landSubSection + ".")`. Nhiều nhóm thì tiêu đề `6.x.1.` đã in ở `AssetGroupsSection` nên không in lại. Nếu nhóm không có `AppraisalShipDescription` thì trả `sectionIndex` (không in gì thêm).

1. **Thông số chính**
   - `Tên phương tiện: ` Text(shipName)
   - `Số đăng ký: ` Text(registrationNumber)
   - `Công dụng: ` Text(shipTypePurpose)
   - `Số đăng kiểm: ` Text(registryNumber)
   - `Chủ phương tiện: ` Text(ownerName)
   - `Năm đóng: ` `String.valueOf(buildYear)` (Integer, null thì bỏ)
   - `Nơi đóng: ` Text(buildPlace)
   - `Kích thước cơ bản: ` Join(` x `, Num(dimLength), Num(dimBreadth), Num(dimDepth), Num(dimDraft)) rồi nối hậu tố ` (m) (L x B x D x d)`; Join null thì bỏ dòng
   - `Lmax x Bmax: ` Join(` x `, Num(dimLengthMax), Num(dimBreadthMax)) rồi nối ` m`; Join null thì bỏ dòng
   - `Mạn khô F: ` Unit(Num(freeBoard), `mm`)
   - `Vật liệu: ` Text(material)
   - `Tổng dung tích: ` Unit(Num(grossTonnage), `GT`)
   - `Cấp phương tiện: ` Text(shipClass)
   - `Số lượng máy chính/phụ: ` Join(`/`, Num(mainEngineQuantity), Num(auxEngineQuantity))
   - `Tổng công suất: ` Unit(Join(`/`, Num(mainEnginePowerHp), Num(auxEnginePowerHp)), `sức ngựa`)
   - `Vùng hoạt động: ` Text(operatingArea)
   - `Khả năng khai thác: ` Seg(`Lượng hàng`, Unit(Num(cargoCapacity), `(tấn)`), `TTTP`, Unit(Num(deadweight), `(tấn)`))
   - Sau cùng: mỗi dòng của `operationNote` (tách bằng `splitMultilineTextToLines`, trim, bỏ dòng rỗng) là **một bullet nguyên văn, không qua `addShipLine` nên không tự thêm dấu chấm**.
2. **Thiết kế phương tiện**
   - `Ký hiệu thiết kế: ` Text(designSymbol)
   - `Năm thiết kế: ` Num(designYear) (Integer)
   - `Số GCN: ` Text(designCertificateNumber)
   - `Số thẩm định: ` Text(appraisalNumber)
   - `Năm và nơi hoán cải: ` Text(conversionYearPlace)
3. **Thiết bị neo**
   - `Neo mũi, ` Seg(`kiểu`, Text(anchorBowType), `số lượng (chiếc)`, Num(anchorBowQuantity), `tổng khối lượng neo`, Unit(Num(anchorBowTotalWeightKg), `kg`))
   - `Neo đuôi, ` Seg(`kiểu`, Text(anchorSternType), `số lượng (chiếc)`, Num(anchorSternQuantity), `tổng khối lượng neo`, **Text(anchorSternTotalWeightKg)**) — cột TEXT nên in nguyên chuỗi, không định dạng số, không thêm `kg`
   - `Chiều dài/đường kính xích, cáp: ` Text(anchorChainCable)
   - `Kiểu tời neo: ` Text(anchorWindlassType)
4. **Thiết bị lái**
   - `Kiểu bánh lái: ` Seg(null, Text(steeringRudderType), `số lượng (chiếc)`, Num(steeringRudderQuantity), `tổng diện tích (m2)`, Num(steeringRudderTotalAreaM2)) — ra file là `(m²)`
   - `Máy lái chính: ` Seg(`kiểu`, Text(steeringMainGearType), `mô men lái (tm)`, Num(steeringMainGearTorqueTm))
   - `Máy lái dự phòng: ` Seg(`kiểu`, Text(steeringBackupGearType), `mô men lái (tm)`, Num(steeringBackupGearTorqueTm))
5. **Thiết bị chằng buộc**
   - `Dây buộc: ` Seg(`số lượng (chiếc)`, Num(mooringRopeQuantity), `tổng chiều dài`, Unit(Num(mooringRopeTotalLengthM), `m`))
   - `Cột buộc dây kiểu: ` Seg(null, Text(mooringBollardType), `số lượng (chiếc)`, **Text(mooringBollardQuantity)**)
6. **Thiết bị kéo** (đặt trước Máy chính, sau Chằng buộc)
   - `Kiểu cột kéo: ` Text(towingBollardType)
   - `Kiểu mọc kéo: ` Text(towingHookType)
   - `Kiểu tời thu dây: ` Text(towingWinchType)
   - `Loại dây kéo: ` Text(towingRopeType)
7. **Máy chính**
   - `Số lượng: ` Num(mainEngineQuantity)
   - `Mác máy: ` Text(mainEngineBrand)
   - `Số chế tạo: ` Text(mainEngineSerialNumber)
   - `Năm chế tạo: ` Num(mainEngineManufactureYear) (Integer)
   - `Công suất (HP/KW): ` Num(mainEnginePowerHp)
8. **Máy phụ**
   - `Số lượng, mác, công suất(sức ngựa): ` Join(`; `, Num(auxEngineQuantity), Text(auxEngineBrand), Num(auxEnginePowerHp))
9. **Thiết bị điện**
   - `Máy phát: ` Text(electricGenerator)
   - `Công suất (KVA): ` Text(electricGeneratorPowerKva)
   - `Điện áp (V): ` Text(electricGeneratorVoltage)
   - `Ắc quy, ` Seg(`số bình`, Num(electricBatteryQuantity), `điện áp`, Unit(Num(electricBatteryVoltageV), `V`), `tổng dung lượng`, Unit(Text(electricBatteryTotalCapacity), `Ah`))
10. **Hệ thống khí nén**
    - `Bình khí nén: ` Text(compressedAirTankQuantity)
    - `Máy nén khí: ` Text(compressedAirCompressorQuantity)
11. **Thiết bị nâng**
    - `Số chế tạo hoặc số quản lý: ` Text(liftingSerialNumber)
    - `Tầm với lớn nhất (m)/sức nâng (tấn): ` Text(liftingMaxReachCapacity)
    - `Tầm với nhỏ nhất (m)/sức nâng (tấn): ` Text(liftingMinReachCapacity)
12. **Trang bị chữa cháy**
    - `Bơm chữa cháy, ` Seg(`số lượng`, Unit(Num(firePumpQuantity), `chiếc`), `tổng lưu lượng`, Unit(Num(firePumpTotalFlowM3), `m³`)) — **ký tự `m³`**
    - `Loại khí: ` Seg(null, Text(fireGasType), `tổng khối lượng (kg)`, Num(fireGasTotalWeightKg))
    - `Loại bọt: ` Seg(null, Text(fireFoamType), `tổng thể tích (lít)`, Text(fireFoamTotalVolumeL))
    - `Bình bọt đẩy: ` Seg(`số lượng`, Text(fireFoamPusherQuantity), `dung tích (lít)`, Text(fireFoamPusherVolumeL))
    - `Thiết bị tạo bọt xách tay: ` Text(firePortableFoam)
    - `Bình CO2: ` Seg(`số bình`, Num(fireCo2BottleQuantity), `khối lượng (kg)`, Num(fireCo2BottleWeightKg))
    - `Bình bột: ` Seg(`số bình`, Num(firePowderBottleQuantity), `khối lượng (kg)`, Num(firePowderBottleWeightKg))
    - `Bình bọt: ` Seg(`số bình`, Text(fireFoamBottleQuantity), `dung tích (lít)`, Text(fireFoamBottleVolumeL))
    - `Bộ dụng cụ chữa cháy thủ công: ` shipYesNo(fireManualTools)
    - `Sơ đồ bố trí thiết bị chữa cháy: ` shipYesNo(fireLayoutPlan)
13. **Trang bị cứu sinh**
    - `Phao tròn (chiếc): ` Text(lifesavingRingBuoy)
    - `Phao áo (chiếc): ` Text(lifesavingLifeJacket)
    - `DC nổi/phao bè/Xuồng (chiếc): ` Seg(null, Num(lifesavingRaftBoatQuantity), `tổng sức chở (người)`, Num(lifesavingRaftBoatTotalCapacity))
14. **Trang bị cứu đắm**
    - `Bơm hút khô, ` Seg(`số lượng (chiếc)`, Num(damageControlBilgePumpQuantity), `tổng lưu lượng (m3)`, Num(damageControlBilgePumpTotalFlowM3)) — giữ `m3`
    - `Bộ dụng cụ cứu đắm: ` Text(damageControlTools)
    - `Thiết bị tự động báo mức nước đáy khoang: ` Text(damageControlBilgeAlarm)
15. **Trang bị tín hiệu** (đặc biệt, xem bên dưới)
16. **Trang bị chạy tàu**
    - `Ống nhòm: ` shipYesNo(navigationBinoculars)
    - `Đồng hồ: ` shipYesNo(navigationClock)
    - `Thiết bị đo sâu đơn giản: ` shipYesNo(navigationDepthSounder)
    - `Chuông lệnh: ` shipYesNo(navigationOrderBell)
    - `Thiết bị đo nghiêng: ` shipYesNo(navigationInclinometer)
    - `Thước đo mức nước: ` shipYesNo(navigationDraftGauge)
    - `GPS hoặc thiết bị tương tự: ` shipYesNo(navigationGps)
    - `La bàn từ hoặc la bàn chuẩn: ` shipYesNo(navigationCompass)
    - `Thiết bị truyền thanh chỉ huy: ` shipYesNo(navigationCommandPa)
    - `Ra đa: ` shipYesNo(navigationRadar)
    - `Thiết bị VTĐ: ` Text(navigationRadio)
    - `S.EPIRB: ` shipYesNo(navigationEpirb)
    - `AIS: ` Text(navigationAis)
    - `SART/AIS-SART: ` Text(navigationSart)
17. **Trang bị ngừa ô nhiễm**
    - `Thiết bị phân ly dầu nước, ` Seg(`số lượng (chiếc)`, Text(pollutionOilyWaterSeparatorQuantity), `tổng sản lượng`, Text(pollutionOilyWaterSeparatorTotalOutput))
    - `Két thu hồi hỗn hợp dầu nước, ` Seg(`số lượng (chiếc)`, Num(pollutionOilyWaterTankQuantity), `tổng thể tích (m3)`, Num(pollutionOilyWaterTankTotalVolumeM3))
    - `Két nước thải, ` Seg(`số lượng (chiếc)`, Text(pollutionSewageTankQuantity), `tổng thể tích (m3)`, Text(pollutionSewageTankTotalVolumeM3))
    - `Két lắng, ` Seg(`số lượng (chiếc)`, Text(pollutionSettlingTankQuantity), `tổng thể tích (m3)`, Text(pollutionSettlingTankTotalVolumeM3))
    - `Két dầu bẩn, ` Seg(`số lượng (chiếc)`, Text(pollutionSludgeTankQuantity), `tổng thể tích (m3)`, Text(pollutionSludgeTankTotalVolumeM3))
    - `Bích nối tiêu chuẩn, ` Seg(`số lượng (chiếc)`, Text(pollutionStandardFlangeQuantity), `vị trí`, Text(pollutionStandardFlangePosition))
    - `Thiết bị chứa rác, ` Seg(`số lượng (chiếc)`, Text(pollutionGarbageContainerQuantity))
    - `Thiết bị xử lý nước thải, ` Seg(`số lượng (chiếc)`, Text(pollutionSewageTreatmentQuantity), `sản lượng (m3)`, Text(pollutionSewageTreatmentOutputM3))
18. **Thời hạn kiểm tra**
    - `Định kỳ: ` shipDate(inspectionPeriodicDate)
    - `Trên đà: ` shipDate(inspectionDockDate)
    - `Hàng năm: ` shipDate(inspectionAnnualDate)

Sau khối 18: gọi `writeShipConditionTable` rồi `writeShipQualityTable`, trả `sectionIndex` nguyên (không tăng).

### Khối 15 — Trang bị tín hiệu (l.6975–7045)

Nhãn **16 đèn** (đúng thứ tự, field `signalLight…`): `Đèn mạn đỏ` (SideRed), `Đèn mạn xanh` (SideGreen), `Đèn trắng lái` (SternWhite), `Đèn cột trắng` (MastWhite), `Đèn cột xanh` (MastGreen), `Đèn cột đỏ` (MastRed), `Đèn neo` (Anchor), `Đèn lai đỏ` (TowingRed), `Đèn lai xanh` (TowingGreen), `Đèn vàng` (Yellow), `Đèn trắng nháy` (FlashWhite), `Đèn vàng nháy` (FlashYellow), `Đèn đỏ nháy` (FlashRed), `Đèn cho các tình huống khác` (Other), `Đèn nửa xanh nửa đỏ` (HalfGreenRed), `Đèn dự trữ` (Spare).

Nhãn **10 vật hiệu** (field `signalShape…`): `Tam giác đen` (BlackTriangle), `Tam giác trắng` (WhiteTriangle), `Vuông đen` (BlackSquare), `Chữ nhật đen` (BlackRectangle), `Tròn đen` (BlackCircle), `Tròn trắng` (WhiteCircle), `Hình thoi góc vuông đen` (BlackDiamond), `Bộ cờ hiệu` (FlagSet), `Pháo hiệu dù` (ParachuteFlare), `Súng phóng dây` (LineThrower).

Mỗi nhãn đi với `Boolean`; `null` thì bỏ cặp; ngược lại cặp `(nhãn, ☑)` nếu true, `(nhãn, ☐)` nếu false.

Thứ tự ghi, chỉ khi có ít nhất một trong ba nhóm (đèn, `signalSound`, vật hiệu):
1. Đoạn tiêu đề `Trang bị tín hiệu` (diamond, in đậm).
2. Nếu có đèn: đoạn bullet `Đèn hiệu:` rồi **bảng lưới** của đèn.
3. Nếu `Text(signalSound) != null`: đoạn bullet `Âm hiệu: ` + giá trị (qua `addShipLine`, tự thêm dấu chấm).
4. Nếu có vật hiệu: đoạn bullet `Vật hiệu:` rồi bảng lưới của vật hiệu.

### Bảng lưới ☑/☐ (`writeShipYesNoGrid`, l.7350–7428)

- 4 cặp mỗi hàng, tức 8 cột (`slots = 4`, `longLabel = 20`).
- Nhãn dài hơn 20 ký tự (`length() > 20`) chiếm 2 slot: nhãn gộp 3 cột (`gridSpan = 3`) + 1 cột giá trị. Không đủ slot trong hàng thì xuống hàng mới.
- Cột nhãn 1702 twips, cột giá trị 662 twips, khai báo `tblGrid` đủ 8 `gridCol` (nếu không Word chia đều cột khi layout cố định), `setTableLayoutFixed`, viền `nil`.
- Chiều rộng `tcW` DXA cho từng ô; ô gộp có `tcW` bằng tổng độ rộng các cột gộp; các ô thừa của nhãn gộp bị `row.removeCell` (duyệt từ phải sang trái).
- Chữ nhãn cỡ 11 căn trái; ô ký tự checkbox font **`Segoe UI Symbol`** cỡ 12, căn giữa.

Kết quả bố cục mẫu để dùng làm kỳ vọng test: 16 đèn đầy đủ cho **5 hàng** (hàng 0–2 mỗi hàng 4 cặp; hàng 3 gồm `Đèn đỏ nháy`, `Đèn cho các tình huống khác` (gộp), `Đèn nửa xanh nửa đỏ`; hàng 4 gồm `Đèn dự trữ`). 10 vật hiệu đầy đủ cho **3 hàng** (hàng 0 bốn cặp đầu; hàng 1 gồm `Tròn đen`, `Tròn trắng`, `Hình thoi góc vuông đen` (gộp); hàng 2 gồm `Bộ cờ hiệu`, `Pháo hiệu dù`, `Súng phóng dây`).

### Tình trạng (`writeShipConditionTable`, l.7067–7152)

- Lọc `conditionDetails`: giữ phần tử khác null mà `Text(itemName)` hoặc `Text(conditionDescription)` khác null. Không còn dòng nào thì **bỏ cả mục**.
- Tiêu đề mục: số `hasManyGroups ? landSubSection + ".2." : sectionIndex + ".2."`, chữ `Tình trạng của tài sản thẩm định`, `createSecondLvSection(number).toBuilder().spacingBefore(12 * 20)`.
- **Đoạn mở đầu** (dùng `normalAttr` truyền vào):
  - `subject`: nếu `Text(shipName)` null thì `tài sản thẩm định`; nếu tên (không phân biệt hoa thường) bắt đầu bằng `tàu ` thì giữ nguyên tên; ngược lại `Tàu ` + tên.
  - Chuỗi = `Tại thời điểm thẩm định giá ` + subject
    - nếu có `Text(registrationNumber)`: nối `, số đăng ký: ` + giá trị
    - nếu có `buildYear`: nối `, năm đóng: ` + năm
    - nếu có `Text(buildPlace)`: nối `, nơi đóng: ` + giá trị
    - nếu có `Text(mooringLocation)`: nối ` đang neo đậu tại ` + giá trị (không có dấu phẩy)
    - nếu ký tự cuối chưa là `.` thì nối `.`
    - nối nguyên văn: ` Tiếp cận kiểm tra các bộ phận nhận thấy các chi tiết có thể tiếp cận được tổ thẩm định nhận thấy tình trạng các bộ phận như sau:`
- Bảng 3 cột `8% / 32% / 60%`, layout fixed, header 1 hàng `STT`, `Hạng mục`, `Tình trạng` (in đậm, cỡ 11, căn giữa, spacing 60/60, căn giữa dọc).
- Mỗi dòng dữ liệu: STT đánh lại từ 1 theo thứ tự giữ lại (căn giữa), `Hạng mục` = `textOrEmpty(Text(itemName))` (căn trái), `Tình trạng`: nếu `Text(conditionDescription)` null thì ô rỗng; ngược lại tách theo `\n`, mỗi dòng đã trim là một đoạn (căn trái).

### Đánh giá chất lượng (`writeShipQualityTable`, l.7155–7262)

- Duyệt `qualityEvaluationDetails`; bỏ phần tử null và phần tử rỗng (không có tên nào sau trim, và weight, standard, rating (sau `Text`), relative đều null). Không còn phần tử nào thì **bỏ cả mục**. STT đánh lại từ 1.
- Tiêu đề mục: số `hasManyGroups ? landSubSection + ".3." : sectionIndex + ".3."`, chữ `Đánh giá chất lượng còn lại của tài sản thẩm định:`, kiểu như mục Tình trạng.
- **Đoạn dẫn** (`normalAttr`), nguyên văn, **`QĐ–TĐC` dùng gạch en-dash U+2013**:
  `Căn cứ quyết định số 57/1999/QĐ–TĐC ngày 11/03/1999 của Tổng cục đo lường chất lượng và thông tư số 45/2013/TT-BTC ngày 25/04/2013 của Bộ Tài Chính để đánh giá chất lượng còn lại (CLCL) của từng bộ phận và chất lượng của tàu. Kết quả được thể hiện trong bảng tính sau:`
- Bảng 8 cột `5% / 25% / 9% / 10% / 11% / 10% / 15% / 15%`, số hàng = `2 + Σ max(1, số tên) + 1`.
  - Hàng 0: `STT`, `Hạng mục thẩm định`, `Hệ số quan trọng`, `Giá trị chuẩn so sánh`, `Đánh giá`, ``(ô 5 rỗng), `Giá trị chỉ tiêu CL tương đối`, `Giá trị chỉ tiêu CL tổng hợp`; gộp ngang ô 4–5.
  - Hàng 1: ``, ``, `(1)`, `(2)`, ``, `(3)`, `(4)=(3)/(2)`, `(5)=(1)*(4)`.
- Mỗi item chiếm `span = max(1, số tên)` hàng. Cột 1 (hạng mục) ghi **mỗi tên một hàng** (căn trái; hàng thừa ghi rỗng). Các cột 0, 2–7 chỉ ghi ở hàng đầu của item (căn giữa) rồi **gộp dọc** `startRow..startRow+span-1` khi `span > 1`.
- Công thức một item (số liệu đủ chữ số, không làm tròn trung gian):
  - `raw4 = (standard != null && standard.signum() > 0 && relative != null) ? relative.divide(standard, 10, HALF_UP) : null`
  - `c5 = (weight != null && raw4 != null) ? weight * raw4 * 100 : null`; `total += c5` (dùng `raw4` **chưa làm tròn**)
  - Cột (1) và (2): `formatFloatToVietnameseStyle(double)` hoặc rỗng; cột Đánh giá: `textOrEmpty(Text(rating))`; cột (3): `shipFixed2` (`#,##0.00`, HALF_UP, dấu `.` nghìn và `,` thập phân) hoặc rỗng; cột (4): `raw4.setScale(1, HALF_UP)` qua `formatFloatToVietnameseStyleWith1Decimal` hoặc rỗng (**làm tròn chỉ để hiển thị**); cột (5): `shipFixed2(c5) + "%"` hoặc rỗng.
- Hàng cuối: gộp ngang ô 0–6 với chữ `Chất lượng còn lại của tàu` (in đậm, căn giữa); ô 7 = `shipFixed2(total) + "%"` (in đậm). **Luôn in tổng, kể cả ngoài khoảng 0–100** (khác `ShipQuality.remainingQualityPct`, hàm đó trả null ngoài khoảng và đó là giá trị ghi sang so sánh).
- **Câu kết luận** (đoạn `normalAttr` sau bảng): `Vậy chất lượng còn lại của ` + subject + ` là: ` + `shipFixed2(total)` + `%.`; `subject` = các phần có dữ liệu nối bằng `, `: `Text(shipName)` (không thêm "Tàu"), `số đăng ký: …`, `năm đóng: …`, `nơi đóng: …`; không có phần nào thì `tài sản thẩm định`.

### Khác hệ thống cũ

Phần Word này **chép nguyên hành vi** của L; không có điểm khác. Điểm cần ghi vào Javadoc của `ShipQualityTable`: tổng ở Word không bị chặn 0–100 như giá trị ghi sang so sánh.

---

## Task B08: Word phần 1 (`ShipDescriptionWriter`, `ShipConditionTable`, `ShipQualityTable`)

**Files:**
- Create (sản phẩm), toàn bộ trong `WN/asset/ship/`:
  - `ShipText.java` (helper thuần, `public final`)
  - `ShipYesNoGrid.java`
  - `ShipDescriptionWriter.java`
  - `ShipConditionTable.java`
  - `ShipQualityTable.java`
- Create (test), trong `WNT/`: `WordTestKit.java`; trong `WNT/asset/ship/`: `ShipFixtures.java`, `ShipTextTest.java`, `ShipYesNoGridTest.java`, `ShipDescriptionWriterTest.java`, `ShipConditionTableTest.java`, `ShipQualityTableTest.java`.
- Không sửa file nào có sẵn. Không đăng ký vào `AssetTypeWriters` (việc của B09).

**Interfaces:**

```java
// WN/asset/ship — tất cả public, extends WordBlock (trừ ShipText)
public final class ShipText {
    public static String text(String v);                       // shipText
    public static String num(BigDecimal v);                    // shipNum
    public static String num(Integer v);
    public static String unit(String v, String unit);
    public static String yesNo(Boolean v);
    public static String date(LocalDate v);
    public static String join(String sep, String... parts);
    public static String segments(String... labelValuePairs);
    public static void addLine(List<String> lines, String prefix, String value);
    public static String fixed2(BigDecimal v);                 // #,##0.00 HALF_UP kiểu Việt
}
public class ShipYesNoGrid extends WordBlock {
    public ShipYesNoGrid(ReportWordContext ctx);
    public void write(List<String[]> items);                   // items: {nhãn, "☑"|"☐"}
    public static List<List<int[]>> layout(List<String[]> items); // {chỉ số mục, cột nhãn đầu, cột nhãn cuối} theo hàng
}
public class ShipDescriptionWriter extends WordBlock {
    public ShipDescriptionWriter(ReportWordContext ctx);
    public int createShipDescription(AppraisalAssetGroup group, boolean hasManyGroups, String landSubSection,
            int sectionIndex, WriteTextAttribute normalAttr, WriteTextAttribute normalAttrWithBullet); // trả sectionIndex nguyên
}
public class ShipConditionTable extends WordBlock {
    public ShipConditionTable(ReportWordContext ctx);
    public void write(AppraisalShipDescription s, boolean hasManyGroups, String landSubSection,
            int sectionIndex, WriteTextAttribute normalAttr);
    public static String introSentence(AppraisalShipDescription s);   // đoạn mở đầu đầy đủ
}
public class ShipQualityTable extends WordBlock {
    public ShipQualityTable(ReportWordContext ctx);
    public void write(AppraisalShipDescription s, boolean hasManyGroups, String landSubSection,
            int sectionIndex, WriteTextAttribute normalAttr);
    public static String[] row(String stt, ShipQualityEvaluationItem it, BigDecimal[] total); // cộng (5) vào total[0]
    public static String conclusionSubject(AppraisalShipDescription s);
}
```

B09 gọi `new ShipDescriptionWriter(ctx).createShipDescription(…)`; hai bảng được gọi nội bộ từ `ShipDescriptionWriter` (đúng thứ tự: Tình trạng rồi Chất lượng) nên B09 không gọi trực tiếp.

- [ ] **Step 1: Đọc code.** Đọc các đoạn LW đã nêu ở "Hành vi hệ thống cũ" (6804–7064, 7067–7262, 7265–7497); đọc `WN/asset/VehicleDescriptionWriter.java`, `WN/asset/DetailTableBlock.java`, `WN/WordBlock.java`, `WordExportCommon.createTable/mergeCell*/writeCell`; grep tên thật của entity/record B01 (bảng "Điều kiện đầu vào"). Ghi vào báo cáo tên nào lệch.

- [ ] **Step 2: Dựng helper test.** Tạo `WordTestKit` và `ShipFixtures` (xem "Helper test dùng chung"). Viết `ShipFixturesTest` hoặc một test đầu trong `ShipTextTest`: `fixture_blank_mọi_boolean_là_null` (duyệt mọi field `Boolean` của `AppraisalShipDescription` bằng reflection, tất cả null) và `fixture_blank_hai_danh_sách_json_rỗng`. Chạy: `cd backend && ./mvnw -q test -Dtest=ShipTextTest` — FAIL vì `ShipText` chưa có (lỗi biên dịch). Cài `WordTestKit`/`ShipFixtures` cho biên dịch được phần helper test.

- [ ] **Step 3: `ShipText` (helper thuần).** Viết test FAIL trước (`ShipTextTest`), từng ca:
  - `text_cắt_khoảng_trắng_rỗng_và_null_thành_null`
  - `num_bigdecimal_kiểu_việt`: `1234.5`→`1.234,5`, `10.00`→`10`, `0.456`→`0,46`, null→null
  - `num_integer_không_phân_cách_nghìn`: `1599`→`1599`, null→null
  - `unit_null_giữ_null_ngược_lại_nối_khoảng_trắng`: `unit("12","mm")`→`12 mm`
  - `yesNo_true_có_false_gạch_null_bỏ`: `có`, `-/-`, null
  - `date_dd_MM_yyyy`: `2026-03-05`→`05/03/2026`
  - `join_phần_thiếu_ghi_gạch_tất_cả_thiếu_trả_null`: `join("/", "2", null)`→`2/-/-`; `join(" x ", null, null)`→null
  - `segments_nhãn_null_chỉ_in_giá_trị`, `segments_giá_trị_thiếu_ghi_gạch`: `segments("kiểu", null, "số lượng (chiếc)", "2")`→`kiểu: -/-; số lượng (chiếc): 2`; tất cả null→null
  - `addLine_giá_trị_null_bỏ_dòng`, `addLine_tự_thêm_dấu_chấm`, `addLine_không_thêm_khi_đã_kết_thúc_bằng_chấm_hoặc_hai_chấm`
  - `fixed2_half_up_hai_chữ_số`: `1234.565`→`1.234,57`, `0`→`0,00`
  Chạy FAIL → cài đặt `ShipText` (chép đúng hành vi bảng helper) → PASS: `./mvnw -q test -Dtest=ShipTextTest`.

- [ ] **Step 4: Bảng lưới `ShipYesNoGrid`.** Test FAIL trước (`ShipYesNoGridTest`), dùng `WordTestKit.newContext()` rồi `new ShipYesNoGrid(ctx).write(items)` và đọc bảng đầu tiên của `doc`:
  - `bố_cục_16_đèn_có_5_hàng`: 16 đèn đầy đủ, `layout()` trả 5 hàng, kích thước hàng `[4,4,4,3,1]`
  - `nhãn_trên_20_ký_tự_chiếm_hai_slot`: `Đèn cho các tình huống khác` (27 ký tự) có cột nhãn đầu 2 và cột nhãn cuối 4; `Đèn nửa xanh nửa đỏ` (19 ký tự) chỉ chiếm 1 slot
  - `nhãn_đúng_20_ký_tự_chỉ_chiếm_một_slot` (ngưỡng `> 20`, dùng nhãn thử 20 ký tự)
  - `không_đủ_chỗ_thì_xuống_hàng`: 3 mục ngắn rồi một mục dài thì mục dài xuống hàng 2
  - `bố_cục_10_vật_hiệu_có_3_hàng`: `[4,3,3]`, `Hình thoi góc vuông đen` gộp ở hàng 1
  - `tblGrid_có_8_cột_xen_kẽ_1702_và_662`: `table.getCTTbl().getTblGrid().getGridColArray()` có 8 phần tử `w` lần lượt `1702,662,1702,662,…`
  - `ô_nhãn_gộp_có_gridSpan_3_và_tcW_bằng_tổng`: ô gộp `gridSpan.val == 3`, `tcW = 1702*2 + 662`... tính đúng: ba cột gộp là nhãn+giá trị+nhãn = `1702 + 662 + 1702`; xác nhận bằng tổng độ rộng các cột `cell[1]..cell[2]` theo mảng `colTwips`
  - `ô_thừa_của_nhãn_gộp_bị_xóa`: hàng 3 của lưới đèn có 6 ô (không phải 8)
  - `layout_cố_định_và_không_viền`: `tblLayout` fixed; `tblBorders` là `nil`
  - `ký_tự_checkbox_font_Segoe_UI_Symbol_cỡ_12_căn_giữa`: run của ô giá trị có `getFontFamily()=="Segoe UI Symbol"`, `getFontSize()==12`, đoạn căn giữa; ô nhãn cỡ 11 căn trái
  - `true_là_☑_false_là_☐`
  Chạy FAIL → cài đặt (chép `writeShipYesNoGrid`, `setShipCellWidth`; tách phần tính layout thành `layout()` static để test thuần) → PASS: `./mvnw -q test -Dtest=ShipYesNoGridTest`.

- [ ] **Step 5: `ShipConditionTable`.** Test FAIL trước (`ShipConditionTableTest`):
  - Đoạn mở đầu (gọi `introSentence`):
    - `intro_không_có_tên_dùng_tài_sản_thẩm_định`: chuỗi bằng đúng `Tại thời điểm thẩm định giá tài sản thẩm định. Tiếp cận kiểm tra các bộ phận nhận thấy các chi tiết có thể tiếp cận được tổ thẩm định nhận thấy tình trạng các bộ phận như sau:`
    - `intro_tên_chưa_có_tàu_thì_thêm_Tàu`: `Hoàng Phát 09` → `Tại thời điểm thẩm định giá Tàu Hoàng Phát 09. …`
    - `intro_tên_bắt_đầu_bằng_tàu_không_phân_biệt_hoa_thường_giữ_nguyên`: `tàu Sao Mai` và `TÀU X` giữ nguyên
    - `intro_đủ_các_vế`: tên `Hoàng Phát 09`, số đăng ký `HP-1234`, năm `2015`, nơi đóng `Hải Phòng`, neo đậu `Cảng Cát Lái` → `Tại thời điểm thẩm định giá Tàu Hoàng Phát 09, số đăng ký: HP-1234, năm đóng: 2015, nơi đóng: Hải Phòng đang neo đậu tại Cảng Cát Lái. Tiếp cận …`
    - `intro_chỉ_có_nơi_neo_đậu`: `… giá tài sản thẩm định đang neo đậu tại Cảng A. Tiếp cận …`
    - `intro_nơi_neo_đậu_đã_kết_thúc_bằng_chấm_không_thêm_chấm_thứ_hai`
  - Bảng:
    - `không_có_dòng_hợp_lệ_thì_không_ghi_gì`: danh sách rỗng, và danh sách chỉ có phần tử rỗng/khoảng trắng, đều không thêm đoạn hay bảng nào vào `doc`
    - `bỏ_dòng_cả_hai_ô_rỗng_và_đánh_lại_STT`: 3 dòng, giữa rỗng cả hai thì STT là `1`, `2`
    - `dòng_chỉ_có_tình_trạng_thì_hạng_mục_rỗng`
    - `tình_trạng_nhiều_dòng_tách_thành_nhiều_đoạn_trong_ô`: `a\nb` thì ô có 2 đoạn `a`, `b`
    - `bảng_ba_cột_8_32_60_và_header`: header `STT`, `Hạng mục`, `Tình trạng` in đậm căn giữa
    - `số_mục_một_nhóm_là_sectionIndex_chấm_2`: `sectionIndex=6` thì `numberOf(tiêu đề)=="6.2."`
    - `số_mục_nhiều_nhóm_là_landSubSection_chấm_2`: `landSubSection="6.1"` thì `"6.1.2."`
    - `thứ_tự_tiêu_đề_đoạn_mở_đầu_rồi_bảng`: body gồm 3 phần tử liên tiếp (tiêu đề, đoạn, bảng)
  Chạy FAIL → cài đặt (chép `writeShipConditionTable`, tách `introSentence`) → PASS: `./mvnw -q test -Dtest=ShipConditionTableTest`.

- [ ] **Step 6: `ShipQualityTable`.** Test FAIL trước (`ShipQualityTableTest`). Bộ dữ liệu dùng chung: item A `(["Thân vỏ"], w=0.3, s=3, r=2, rating "Tốt")`, item B `(["Máy chính","Hệ trục"], w=0.5, s=3, r=1, "Khá")`.
  - Hàng dữ liệu (gọi `row`):
    - `row_A`: `["1","","0,3","3","Tốt","2,00","0,7","20,00%"]`
    - `row_B`: `["2","","0,5","3","Khá","1,00","0,3","16,67%"]` (chứng minh (5) dùng `raw4` chưa làm tròn: dùng 0,3 sẽ ra 15,00%)
    - `row_standard_bằng_0_thì_(4)_và_(5)_rỗng`: standard 0, relative 2, weight 0.3 thì cột 6, 7 rỗng và cột (3) vẫn là `2,00`
    - `row_thiếu_weight_thì_(5)_rỗng_nhưng_(4)_vẫn_có`
    - `row_tổng_cộng_dồn_vào_total0`: sau A và B, `total[0]` làm tròn `fixed2` là `36,67`
  - Bảng:
    - `không_có_item_hợp_lệ_thì_không_ghi_gì`: null, rỗng, chỉ item rỗng (không tên, các số đều null, rating rỗng)
    - `item_chỉ_có_tên_vẫn_là_một_hàng_với_ô_số_rỗng`
    - `số_hàng_bằng_2_cộng_tổng_max_1_số_tên_cộng_1`: A (1 tên) + B (2 tên) thì `2 + 1 + 2 + 1 = 6` hàng
    - `header_hai_hàng_nguyên_văn`: hàng 0 và hàng 1 như mô tả; ô (0,4) có `hMerge=RESTART`, ô (0,5) `CONTINUE`
    - `mỗi_tên_một_hàng_ở_cột_hạng_mục`: item B cho hạng mục `Máy chính`, `Hệ trục` ở hai hàng liên tiếp
    - `item_nhiều_tên_gộp_dọc_các_cột_0_2_đến_7`: các cột `0,2,3,4,5,6,7` của hàng đầu item B có `vMerge=RESTART`, hàng sau `CONTINUE`; cột 1 không gộp
    - `item_một_tên_không_gộp_dọc`
    - `hàng_cuối_gộp_ngang_0_đến_6_và_in_tổng`: ô 0 là `Chất lượng còn lại của tàu`, ô 7 là `36,67%`
    - `tổng_không_bị_chặn_ngoài_0_100`: item w=2, s=1, r=1 thì tổng và câu kết luận in `200,00%`
    - `bảng_tám_cột_tỉ_lệ_5_25_9_10_11_10_15_15`
    - `đoạn_dẫn_nguyên_văn_với_en_dash`: đoạn ngay sau tiêu đề bằng đúng chuỗi trong "Hành vi hệ thống cũ" (có `QĐ–TĐC`, U+2013)
    - `câu_kết_luận_đủ_vế`: `Vậy chất lượng còn lại của Hoàng Phát 09, số đăng ký: HP-1234, năm đóng: 2015, nơi đóng: Hải Phòng là: 36,67%.` (tên không có tiền tố "Tàu")
    - `câu_kết_luận_không_có_dữ_liệu_dùng_tài_sản_thẩm_định`: `Vậy chất lượng còn lại của tài sản thẩm định là: 36,67%.`
    - `số_mục_một_nhóm_và_nhiều_nhóm`: `6.3.` và `6.1.3.`
    - `tổng_khớp_ShipQuality_khi_trong_khoảng`: với bộ A+B, `ShipQuality.remainingQualityPct(items)` (B01) bằng `36.67` và khớp chữ số trong ô tổng; với tổng ngoài khoảng thì `ShipQuality` trả null trong khi Word vẫn in số (ghi chú ca này trong Javadoc)
  Chạy FAIL → cài đặt (chép `writeShipQualityTable`, `shipQualityRow`, `shipQualityNames`, `shipQualityItemIsEmpty`) → PASS: `./mvnw -q test -Dtest=ShipQualityTableTest`.

- [ ] **Step 7: `ShipDescriptionWriter`.** Test FAIL trước (`ShipDescriptionWriterTest`). Mỗi test dựng `ctx = WordTestKit.newContext(group)`, gọi `createShipDescription(group, hasMany, "6.1", 6, attr, attrBullet)` rồi đọc `paragraphTexts(doc)`/`body(doc)`. Dữ liệu mẫu cho thông số chính: `shipName=Hoàng Phát 09, registrationNumber=HP-1234, shipTypePurpose=Chở hàng khô, registryNumber=VR-77, ownerName=Cty ABC, buildYear=2015, buildPlace=Hải Phòng, dimLength=70.5, dimBreadth=11.2, dimDepth=4.5, dimDraft=null, dimLengthMax=72, dimBreadthMax=null, freeBoard=1250, material=Thép, grossTonnage=1599, shipClass=VR-SB, mainEngineQuantity=1, auxEngineQuantity=null, mainEnginePowerHp=1200, auxEnginePowerHp=150.5, operatingArea=SI, cargoCapacity=2500.75, deadweight=null`.
  - Khung:
    - `một_nhóm_in_tiêu_đề_mô_tả_tài_sản_thẩm_định_số_6_1`: đoạn đầu là `Mô tả tài sản thẩm định`, `numberOf == "6.1."`
    - `nhiều_nhóm_không_in_lại_tiêu_đề`
    - `nhóm_không_có_mô_tả_chỉ_in_tiêu_đề`: `getAppraisalShipDescription()==null` thì chỉ có tiêu đề (một nhóm) hoặc không có gì (nhiều nhóm)
    - `nhóm_không_phải_tàu_không_ghi_gì`: `assetType=3`
    - `trả_nguyên_sectionIndex`: truyền `6` và `7`, nhận lại đúng giá trị truyền vào
  - Thứ tự và khối:
    - `mười_tám_khối_đúng_thứ_tự`: điền ít nhất một dòng ở mỗi khối thì các tiêu đề (đoạn diamond in đậm) theo thứ tự `Thông số chính`, `Thiết kế phương tiện`, `Thiết bị neo`, `Thiết bị lái`, `Thiết bị chằng buộc`, `Thiết bị kéo`, `Máy chính`, `Máy phụ`, `Thiết bị điện`, `Hệ thống khí nén`, `Thiết bị nâng`, `Trang bị chữa cháy`, `Trang bị cứu sinh`, `Trang bị cứu đắm`, `Trang bị tín hiệu`, `Trang bị chạy tàu`, `Trang bị ngừa ô nhiễm`, `Thời hạn kiểm tra`
    - `thiết_bị_kéo_nằm_giữa_chằng_buộc_và_máy_chính` (cả ba có dữ liệu)
    - `khối_không_có_dòng_thì_bỏ_cả_tiêu_đề`: chỉ có thiết kế thì chỉ thấy `Thiết kế phương tiện`
    - `mô_tả_rỗng_hoàn_toàn_không_in_khối_nào`
    - `tiêu_đề_in_đậm_kiểu_diamond_dòng_là_bullet`: run tiêu đề `isBold`, đoạn dòng có `numId` khác tiêu đề
  - Thông số chính:
    - `thông_số_chính_đủ_dòng`: đúng 17 dòng theo mẫu: `Tên phương tiện: Hoàng Phát 09.`, `Số đăng ký: HP-1234.`, `Công dụng: Chở hàng khô.`, `Số đăng kiểm: VR-77.`, `Chủ phương tiện: Cty ABC.`, `Năm đóng: 2015.`, `Nơi đóng: Hải Phòng.`, `Kích thước cơ bản: 70,5 x 11,2 x 4,5 x -/- (m) (L x B x D x d).`, `Lmax x Bmax: 72 x -/- m.`, `Mạn khô F: 1.250 mm.`, `Vật liệu: Thép.`, `Tổng dung tích: 1599 GT.`, `Cấp phương tiện: VR-SB.`, `Số lượng máy chính/phụ: 1/-/-.`, `Tổng công suất: 1.200/150,5 sức ngựa.`, `Vùng hoạt động: SI.`, `Khả năng khai thác: Lượng hàng: 2.500,75 (tấn); TTTP: -/-.`
    - `kích_thước_cơ_bản_tất_cả_thiếu_thì_bỏ_dòng` và `lmax_bmax_tất_cả_thiếu_thì_bỏ_dòng`
    - `năm_đóng_và_nơi_đóng_là_hai_dòng_riêng`
    - `tổng_công_suất_chỉ_có_máy_phụ`: `-/-/150,5 sức ngựa.`
    - `operationNote_mỗi_dòng_một_bullet_không_thêm_dấu_chấm`: `Chạy sông\n\n  Vùng SI  ` ra hai bullet `Chạy sông`, `Vùng SI` ngay sau dòng `Khả năng khai thác`
    - `năm_đóng_buildYear_null_bỏ_dòng`
  - Các khối khác:
    - `neo_đuôi_in_nguyên_chuỗi_tổng_khối_lượng`: `anchorSternTotalWeightKg="1.200 kg (2 neo)"`, kiểu `A`, số lượng `2` thì `Neo đuôi, kiểu: A; số lượng (chiếc): 2; tổng khối lượng neo: 1.200 kg (2 neo).`
    - `neo_mũi_có_đơn_vị_kg_và_thiếu_ghi_gạch`
    - `bánh_lái_diện_tích_ra_m_vuông`: `Kiểu bánh lái: Treo; số lượng (chiếc): 1; tổng diện tích (m²): 3,5.` (m2 thành m²)
    - `cột_buộc_dây_số_lượng_là_chuỗi`
    - `máy_chính_nhãn_công_suất_HP_KW`: `Công suất (HP/KW): 1.200.`
    - `máy_phụ_nối_bằng_chấm_phẩy_thiếu_ghi_gạch`: `Số lượng, mác, công suất(sức ngựa): 2; -/-; 150,5.`
    - `ắc_quy_đơn_vị_V_và_Ah`
    - `bơm_chữa_cháy_dùng_m_khối_ký_tự_m3_mũ_ba`: `Bơm chữa cháy, số lượng: 2 chiếc; tổng lưu lượng: 30 m³.`
    - `bơm_hút_khô_giữ_m3`: `Bơm hút khô, số lượng (chiếc): 1; tổng lưu lượng (m3): 5.`
    - `boolean_có_và_gạch_và_null_bỏ_dòng`: `fireManualTools=true` thì `Bộ dụng cụ chữa cháy thủ công: có.`; `false` thì `…: -/-.`; null thì không có dòng
    - `chạy_tàu_14_dòng_đúng_thứ_tự_nhãn` (true, false, text ở VTĐ/AIS/SART)
    - `ngừa_ô_nhiễm_8_dòng`: kể cả `Thiết bị chứa rác, số lượng (chiếc): 3.`
    - `thời_hạn_kiểm_tra_định_dạng_dd_MM_yyyy`: `Định kỳ: 05/03/2026.`
  - Trang bị tín hiệu:
    - `chỉ_có_âm_hiệu_thì_không_có_bảng_và_không_có_dòng_Đèn_hiệu`
    - `có_đèn_và_vật_hiệu_thứ_tự_tiêu_đề_dòng_bảng`: body theo thứ tự `Trang bị tín hiệu`, `Đèn hiệu:`, bảng, `Âm hiệu: Còi hơi.`, `Vật hiệu:`, bảng, rồi `Trang bị chạy tàu`
    - `không_có_đèn_thì_không_in_dòng_Đèn_hiệu_nhưng_vẫn_in_vật_hiệu`
    - `không_có_tín_hiệu_nào_thì_không_in_tiêu_đề`
    - `đèn_null_bị_bỏ_khỏi_lưới`: 3 đèn trong 16 non-null thì lưới 3 mục
  - Gọi hai bảng:
    - `gọi_tình_trạng_rồi_chất_lượng_sau_khối_cuối`: có cả hai danh sách JSON thì sau `Thời hạn kiểm tra` lần lượt là tiêu đề `6.2.` rồi `6.3.` (một nhóm: `sectionIndex=6`)
    - `nhiều_nhóm_số_mục_dùng_landSubSection`
  Chạy FAIL → cài đặt (chép `createShipDescription` dùng `ShipText` và `ShipYesNoGrid`) → PASS: `./mvnw -q test -Dtest=ShipDescriptionWriterTest`.

- [ ] **Step 8: Kiểm tra tổng và quy tắc.** `cd backend && ./mvnw -q test -Dtest='Ship*Test'` PASS. Grep xác nhận không có `if (assetType` trong gói mới; Javadoc mỗi lớp ghi "Chép từ LW `<method>` (l.xxxx), không đổi hành vi" và, với `ShipQualityTable`, ghi chú tổng không bị chặn 0–100.

- [ ] **Step 9: Commit** (trong submodule): `git -C backend add -A && git -C backend commit -m "feat: Word mô tả tàu thủy, tình trạng và đánh giá chất lượng còn lại"`.

**Review Focus B08:** (a) chuỗi cố định khớp từng ký tự với L (đoạn mở đầu Tình trạng, đoạn dẫn chất lượng có en-dash, câu kết luận, 18 tiêu đề, 26 nhãn đèn/vật hiệu); (b) quy tắc `-/-` và null ở từng helper; (c) `m²`/`m³`/`m3` đúng như phần "Bẫy"; (d) tblGrid 1702/662, gridSpan 3, nhãn `> 20`; (e) gộp dọc đúng cột và đúng số hàng; (f) (4) làm tròn hiển thị còn (5) dùng `raw4` chưa làm tròn.

---

# B09 — Word phần 2: `ShipAssetWriter` và bảng so sánh, kết quả, ảnh, điều chỉnh

## Hành vi hệ thống cũ

Nguồn LW:

| Nội dung | Vị trí |
|---|---|
| Cờ `hasShipGroup` | l.119, 271–272 |
| Pháp lý liên quan (`hasRelatedJuridical`, nhánh `hasCarGroup` / tàu) | l.480–505 |
| `createShipRelatedJuridical` | l.6342–6353 |
| Tiêu đề mục 6 khi một nhóm tàu | l.551–556 |
| Tiêu đề phụ nhiều nhóm `Mô tả tài sản thẩm định` | l.858–861 |
| Mục `.4` của nhiều nhóm tàu (`landUseSubOrdinal`) | l.955–957 |
| Nhãn `Thẩm định giá trị tài sản` (`createFixedLandUseRight`) | l.1222–1225 |
| Cụm chủ thể `giá trị tàu chở hàng khô` | l.1279–1280 |
| Dòng tổng hợp nhiều nhóm | l.1489–1503 |
| `createShipComparativeInfoAndImage` | l.1115–1208 |
| `createShipAssetImages` | l.2091–2127 |
| Công tắc bảng so sánh theo loại | l.2842–2846 |
| `createShipGroupCompareTable` | l.3583–3708 |
| Bảng điều chỉnh (đơn vị) | l.4068–4075, 4091, 4098, 4147–4173, 4185, 4215, 4251 |
| `createShipValuationSummaryTable` | l.5186–5300 |
| `buildRegulatoryConstraintsAvgPriceClause` | l.5716 |
| Địa chỉ tóm tắt mục 1 | l.7545–7551 |

Tóm tắt các hook của loại SHIP:

| Hook | Giá trị cho SHIP | Nguồn |
|---|---|---|
| `addressSummaryLine` | như đất: `ReportUltis.formatAssetNames(group)`, rỗng thì null | l.7548 |
| `singleGroupTitle` | `THẨM ĐỊNH GIÁ TRỊ TÀI SẢN` | l.553–554 |
| `manyGroupsDescriptionLabel` | `Mô tả tài sản thẩm định` | l.861 |
| `writeDescription` | `ShipDescriptionWriter` (B08) | l.938 |
| `valuationTitle` | `Thẩm định giá trị tài sản` | l.1222–1224 |
| `valuationSubjectPhrase` | `giá trị tàu chở hàng khô` | l.1279 |
| `isRealEstate` | `false` (nhãn tổng `Tổng giá trị tài sản`) | l.1506 |
| `hasGroupResultSummary` | `false` (chỉ đất) | l.973 |
| `relatedJuridicalItems` | 4 văn bản bên dưới | l.6342 |

### Bốn văn bản pháp lý liên quan của tàu (nguyên văn, l.6342–6353)

1. `Thông tư số: 45/2013/TT-BTC ngày 25/04/2013 của Bộ Tài chính về việc hướng dẫn chế độ quản lý, sử dụng và trích khấu hao tài sản cố định;`
2. `Thông tư số: 147/2016/TT-BTC ngày 13/10/2016 của Bộ Tài chính sửa đổi, bổ sung một số điều của thông tư 45/2013/TT-BTC ngày 25/04/2013 của Bộ Tài chính hướng dẫn về chế độ quản lý, sử dụng và trích khấu hao tài sản cố định;`
3. `Thông tư số: 28/2017/TT-BTC ngày 12/04/2017 của Bộ Tài chính về việc sửa đổi, bổ sung một số điều của thông tư 45/2013/TT-BTC ngày 25/04/2013 và thông tư 147/2016/TT-BTC ngày 13/10/2016 của Bộ Tài chính hướng dẫn về chế độ quản lý, sử dụng và trích khấu hao tài sản cố định.`
4. `Quyết định số 57/1999/QĐ-TĐC ngày 11/03/1999 của Tổng cục đo lường chất lượng về việc ban hành quy trình tạm thời thẩm định kỹ thuật thiết bị đã qua sử dụng.`

Chú ý bản của tàu khác bản của ô tô ở hai chỗ: văn bản 3 viết `thông tư 147/2016/TT-BTC` (không có `số`), và văn bản 4 kết thúc bằng dấu chấm. **Không dùng lại chuỗi của `CarAssetWriter`.**

Khi báo cáo có cả xe lẫn tàu: L chỉ in bộ văn bản của xe (`if (hasCarGroup) createCarRelatedJuridical(); else createShipRelatedJuridical();`, l.499–504) vì bộ của xe đã gồm văn bản tàu.

### Bảng so sánh tàu (l.3583–3708) — 5 cột `18.5% / 20.375% / 20.375% / 20.375% / 20.375%`, layout fixed

Header: `Nội dung`, `Tài sản thẩm định`, `Tài sản so sánh 1`, `Tài sản so sánh 2`, `Tài sản so sánh 3` (in đậm cỡ 11, spacing 60/60, căn giữa cả hai chiều). Ô dữ liệu cỡ 11, căn giữa, thụt lề 0,1cm hai bên; nhãn hàng in đậm. **Không có hàng ảnh, không có hàng `Tỷ lệ ước tính`.** Hàng nào cả 4 ô rỗng thì bỏ. 14 hàng theo thứ tự (kiểu `M` = nhiều dòng, tách theo `\n`, mỗi dòng một đoạn; kiểu `S` = một giá trị):

| # | Nhãn | Nguồn | Kiểu |
|---|---|---|---|
| 1 | `Chủng loại` | `address` | M |
| 2 | `Nguồn tham khảo` | `source` | M |
| 3 | `Thông tin liên hệ` | `contact` | M |
| 4 | `Năm đóng` | `shipBuildYear` (`String.valueOf`) | S |
| 5 | `Nơi đóng` | `shipBuildPlace` | S |
| 6 | `Mục đích sử dụng` | `shipUsagePurpose` | S |
| 7 | `Cấp phương tiện` | `shipClass` | S |
| 8 | `Chất lượng còn lại (%)` | `shipRemainingQualityPct`: `formatFloatToVietnameseStyle(double) + "%"` | S |
| 9 | `Trọng tải toàn phần (tấn)` | `shipDeadweight`: `formatFloatToVietnameseStyle(double)` (số 0 vẫn in `0`) | S |
| 10 | `Kích thước (Lmax x Bmax)` | `shipDimension` | S |
| 11 | `Công suất` | `shipMainEnginePowerHp`: `formatFloatToVietnameseStyle(double)` | S |
| 12 | `Giá rao bán (` + moneyLabel + `)` | `formatComparePriceForExport(forSalePrice)` (null hoặc 0 thì rỗng) | S |
| 13 | `Giá giao dịch (` + moneyLabel + `)` | `tradePrice` | S |
| 14 | `Giá ước tính (` + moneyLabel + `)` | `estimatePrice` | S |

`moneyLabel = currencyMoneyLabel(resolveCurrencyCode(assetType, currencyCode))`: `đồng` hoặc `USD`. Mỗi giá trị `S` đi qua `textOrEmpty`.

### Thông tin và ảnh tài sản so sánh (l.1115–1208)

Chạy ngay sau bảng so sánh. Lấy các `AppraisalGroupCompare` của nhóm theo thứ tự trong danh sách, bỏ bản ghi đã xóa (`deleteFlag != 0`) và `type == null || type == 0`. Mỗi bản ghi:

1. **Đoạn đậm** (căn đều hai bên, giãn dòng 1,5, spacing trước 120): dựng như sau.
   - `parts`: nếu `address` (trim) khác rỗng thì `là ` + address.
   - `specs`: `trọng tải toàn phần ` + `formatFloatToVietnameseStyle(shipDeadweight)` + ` tấn` (khi khác null và khác 0); `năm đóng ` + năm (khi khác null); `nơi đóng ` + nơi đóng (trim khác rỗng); `máy chính công suất ` + `formatFloatToVietnameseStyle(shipMainEnginePowerHp)` + ` HP` (khi khác null và khác 0). Có `specs` thì thêm vào `parts` chuỗi `specs` nối bằng `, `.
   - Chuỗi = `-` + TAB + `Tài sản so sánh ` + type; có `parts` thì nối `: ` + `parts` nối bằng `; `; nối `.`.
   - Nếu `tradePrice` khác null và khác 0: nối ` Giá giao dịch là ` + `formatFloatToVietnameseStyle(tradePrice)` + ` ` + moneyLabel.
   - Nếu `shipRemainingQualityPct` khác null: nối (`, chất lượng` nếu vừa có giá giao dịch, ngược lại ` Chất lượng`) + ` còn lại ước tính ` + `formatFloatToVietnameseStyleWithDecimal(pct)` + `%`.
   - Nếu ký tự cuối khác `.` thì nối `.`.
2. **Link tham khảo**: `source` (trim) tách theo `;`, trim từng phần, bỏ phần rỗng. Có ít nhất một link thì in đoạn `Link tham khảo:` (căn trái, không đậm) rồi mỗi link một đoạn bullet `createCommonBulletAttr()`.
3. **Ảnh**: ảnh hợp lệ = `ComparisonAssetImage` có `filePath` khác rỗng. Có ảnh thì in đoạn đậm `Hình ảnh tài sản so sánh ` + type, rồi bảng 2 cột `50%/50%` không viền, layout fixed, 2 ảnh mỗi hàng, hàng lẻ gộp ngang 2 ô.

### Bảng kết quả thẩm định giá tàu (l.5186–5300)

- Dữ liệu `dto = getOrBuildEvaluationDetail(group)`; `dto == null` hoặc `dto.getShip() == null` thì không ghi gì. `currencyCode = resolveCurrencyCode(assetType, currencyCode)`; `isForeign = !VND`.
- Cột: VND `8% / 50% / 42%` (3 cột); ngoại tệ `8% / 40% / 26% / 26%` (4 cột); layout fixed.
- Header: `STT`, `Hạng mục`, `Thành tiền (` + moneyLabel + `)`; ngoại tệ thêm `Thành tiền (đồng)`.
- `assetName = dto.getAssetName()` trim; rỗng thì nối các `asset` (trim, khác rỗng) của `getAppraisalReportAssets()` bằng `\n`.
- `amount = ship.amount ?? 0`, `vndAmount = ship.vndAmount ?? amount`, `roundedAmount = ship.roundedAmount ?? 0`.
- Hàng 1: `1` (đậm, giữa) | `Giá trị ` + `firstLetterLower(assetName)` (đậm) | `amount` (căn phải) [| `vndAmount` nếu ngoại tệ].
- Hàng `Tổng giá trị` (gộp ngang ô 0–1, cao 1cm, đậm): `amount` [| `vndAmount`].
- Hàng `Làm tròn` (gộp ngang ô 0–1): VND thì ô 2 = `roundedAmount`; **ngoại tệ thì ô 2 = `amount` (không làm tròn) và ô 3 = `roundedAmount`**.
- Số tiền định dạng `formatFloatToVietnameseStyle(long)`. Mọi ô có `isKeepNext(true)` trừ hàng `Làm tròn`.
- **Ghi chú chuyển đổi**: chỉ khi ngoại tệ và `group.getCurrencyNote()` (trim) khác rỗng, in ngay dưới bảng: tách theo `\n`, mỗi dòng một đoạn **không đậm, in nghiêng**, giãn dòng 1,5, căn đều hai bên, spacing trước `12*20` ở đoạn đầu và `0` ở các đoạn sau, spacing sau `3*20`.
- Sau đó `appraisalResult.note` (nếu có): đoạn đầu `Ghi chú: ` đậm nghiêng + dòng đầu, các dòng sau một đoạn riêng (giống `CarValuationSummaryTable`).

### Dòng tổng hợp nhiều nhóm (l.1489–1503)

Một dòng cho nhóm tàu trong bảng `TỔNG HỢP KẾT QUẢ THẨM ĐỊNH GIÁ` (cột `Thành tiền (đồng)`): STT tăng 1, nhãn `Giá trị ` + `firstLetterLower(assetName)`, số tiền = `ship.roundedAmount ?? 0` (đã là VNĐ), cộng vào tổng. Nhãn tổng cuối bảng là `Tổng giá trị tài sản` (nhờ `isRealEstate() = false`). Dòng được ghi kể cả khi `ship == null` (số 0).

### Ảnh tàu (l.2091–2127)

Giống `CarAssetImages`: bỏ cả mục nếu không có `AppraisalShipDescription`, không có ảnh, hoặc không ảnh nào thuộc 11 loại; nhiều nhóm thì in thêm đoạn cấp 2 là `asset` đầu tiên của nhóm với số `sectionIndex.(groupIndex+1).`; mỗi loại có ảnh in một đoạn trống rồi `renderGroupedImageTable(typeImages, nhãn)`. Ảnh lọc theo `deleted == null || 0`, `path` khác rỗng, đúng loại; sắp theo `created` rồi `id` (null cuối). Thứ tự và nhãn:

| Mã | Nhãn |
|---|---|
| 21 `SHIP_FRONT` | `Mặt trước tàu` |
| 22 `SHIP_BOW` | `Đầu tàu` |
| 23 `SHIP_STERN` | `Đuôi tàu` |
| 24 `SHIP_HULL_SIDE` | `Mạn tàu` |
| 25 `SHIP_DECK` | `Sàn tàu` |
| 26 `SHIP_IMO_NUMBER` | `Số IMO/Số kiểm soát` |
| 27 `SHIP_ANCHOR_SYSTEM` | `Hệ thống neo` |
| 28 `SHIP_WHEELHOUSE_CABIN` | `Phòng lái, phòng nghỉ` |
| 29 `SHIP_CARGO_HOLD` | `Khoang hàng` |
| 30 `SHIP_ENGINE_SYSTEM` | `Hệ thống máy của động cơ` |
| 10 `OTHER` | `Khác` |

### Bảng điều chỉnh (l.4068–4251) và Nguyên tắc khống chế (l.5716)

Trong L, với tàu: `currencyUnit = currencyUnitLabel(resolve…)` (`VND` hoặc `USD`); `controlComparePriceUnit = currencyUnit`. Ô đơn vị dùng `currencyUnit` ở các dòng A (l.4091), D1, E1, E4 (l.4185, 4215, 4251); dùng `controlComparePriceUnit` ở B, `Mức điều chỉnh`, `Mức giá sau điều chỉnh`, D. Câu "Nguyên tắc khống chế" dùng đơn vị theo `resolveRegulatoryConstraintsPriceUnit(assetType, currencyCode)`: tàu ra `đồng` hoặc `USD`.

### Khác hệ thống cũ (#4 trong `00-index.md`)

Bảng điều chỉnh của **tàu** trong Word mới khớp màn web, khác L:
- Dòng đầu là **A `Giá bán`**, đơn vị `currencyUnit`, giá trị cột TSSS 1–3 lấy từ `landUseRightPrice` (server đã tính), cột `Tài sản thẩm định` để trống.
- **Không có** dòng `Giá thị trường (giá trước điều chỉnh)` và dòng B `Giá quy đổi về đơn vị so sánh chuẩn`.
- Mục `C Điều chỉnh các yếu tố so sánh` và các dòng sau giữ nguyên.
- Đất, căn hộ, ô tô **không đổi**.

Điểm này phải nằm trong Javadoc của `ShipAssetWriter.adjustmentPriceLayout()` và của `AdjustmentTableBlock`.

## Thiết kế hook mới trên `AssetTypeWriter`

Hiện hook chưa đủ để SHIP đi qua mà không thêm `if` vào section. Cần 5 thay đổi, đều theo nguyên tắc "compiler buộc mọi loại trả lời", **bốn writer cũ (`LandAssetWriter`, `ApartmentAssetWriter`, `CarAssetWriter`, `OtherAssetWriter`) trả giá trị cũ nên output loại 1–3 không đổi**:

| # | Thay đổi | Land / Apartment / Other | Car | Ship | Người dùng |
|---|---|---|---|---|---|
| H1 | `adjustmentPriceUnit()` đổi thành `adjustmentPriceUnit(AppraisalAssetGroup group)` | `VND/m²` | `VND` | `currencyUnitLabel(resolveCurrencyCode(...))` | `AdjustmentTableBlock` (dòng B, C-rows, D) |
| H2 | Thêm `String adjustmentCurrencyUnit(AppraisalAssetGroup group)` | `VND` | `VND` | như H1 | `AdjustmentTableBlock` (A, D1, E1, E4 — hiện đang gõ cứng `"VND"`) |
| H3 | Thêm `AdjustmentPriceLayout adjustmentPriceLayout()`, enum lồng `{ MARKET_AND_CONVERTED, SALE_PRICE_ONLY }` | `MARKET_AND_CONVERTED` | `MARKET_AND_CONVERTED` | `SALE_PRICE_ONLY` | `AdjustmentTableBlock` |
| H4 | Thêm `int valuationSubOrdinal(boolean hasConstructionBlock)` | `hasConstructionBlock ? 3 : 2` | `hasConstructionBlock ? 3 : 2` | `4` (`.1` mô tả, `.2` tình trạng, `.3` chất lượng, `.4` thẩm định) | `AssetGroupsSection` thay biểu thức `landUseSubOrdinal` đang gõ cứng |
| H5 | Thêm `Set<Integer> subsumedRelatedJuridicalTypes()` | `Set.of()` | `Set.of(SHIP)` (bộ văn bản của xe đã gồm bộ của tàu) | `Set.of()` | `LegalBasisSection` |

Cùng lúc:
- `GroupCompares` thêm thành phần cuối `AppraisalAssetGroup group` (record). `CompareTableBlock` truyền `appraisalAssetGroup`. Các bảng đất, căn hộ, ô tô không dùng. Lý do: bảng và mục "thông tin TSSS" của tàu cần `currencyCode` và danh sách `getAppraisalGroupCompares()` của nhóm.
- `LandValuationNotes.buildRegulatoryConstraintsAvgPriceClause` đổi dòng lấy đơn vị từ `CommonConstants.resolveRegulatoryConstraintsPriceUnit(assetType)` sang `CurrencyCodes.regulatoryPriceUnit(assetType, currencyCode)` (B01) với `group == null` thì `(null, null)`. Đất, căn hộ, null cho `đồng/m²`, ô tô cho `đồng/xe`, tàu cho `đồng` hoặc `USD`. Không sửa `CommonConstants.REGULATORY_CONSTRAINTS_PRICE_UNITS` (regex bôi đậm): hai đơn vị của tàu không bôi đậm theo `00-index.md`.
- `LegalBasisSection.writersWithRelatedJuridical()` tách phần chọn thành hàm tĩnh package-private `selectRelatedJuridicalWriters(List<AppraisalAssetGroup>)` (để test thuần): gom `Map<Integer, AssetTypeWriter>` theo loại như cũ (nhóm chưa xóa, `assetType != null`, `relatedJuridicalItems()` khác rỗng), rồi bỏ writer có loại nằm trong `subsumedRelatedJuridicalTypes()` của một writer **khác** trong map. Không có `==` theo loại cụ thể.
- Cập nhật Javadoc `AssetTypeWriter` (mục "Hướng dẫn mở rộng": thêm H1–H5 và `group` của `GroupCompares`) và `AssetTypeWritersTest`.

## Task B09: Word phần 2 (`ShipAssetWriter` và các bảng)

**Files:**
- Create (sản phẩm), trong `WN/asset/`:
  - `ShipAssetWriter.java` (package-private `final class`, như `CarAssetWriter`)
  - `ShipCompareTable.java`
  - `ShipCompareInfoBlock.java` (thông tin và ảnh TSSS, `extends WordImages`)
  - `ShipValuationSummaryTable.java`
  - `ShipAssetImages.java` (`extends WordImages`, package-private)
- Modify (sản phẩm):
  - `WN/asset/AssetTypeWriter.java` (H1–H5 + Javadoc)
  - `WN/asset/LandAssetWriter.java`, `ApartmentAssetWriter.java`, `CarAssetWriter.java`, `OtherAssetWriter.java` (cài H1–H5 với giá trị cũ; `CarAssetWriter` trả `Set.of(SHIP)` cho H5)
  - `WN/asset/AssetTypeWriters.java` (thêm `SHIP -> new ShipAssetWriter()`)
  - `WN/asset/GroupCompares.java` (thêm `group`)
  - `WN/asset/ResultSummaryRowsBlock.java` (thêm `ship(...)`)
  - `WN/section/CompareTableBlock.java` (truyền `group`)
  - `WN/section/AdjustmentTableBlock.java` (dùng H1–H3 theo bảng trên)
  - `WN/section/AssetGroupsSection.java` (dùng H4)
  - `WN/section/LegalBasisSection.java` (H5, hàm chọn writer)
  - `WN/section/LandValuationNotes.java` (đơn vị khống chế theo tiền tệ)
- Create (test), trong `WNT/asset/`: `ShipAssetWriterTest.java`, `ShipCompareTableTest.java`, `ShipCompareInfoBlockTest.java`, `ShipValuationSummaryTableTest.java`, `ShipAssetImagesTest.java`, `ResultSummaryRowsBlockShipTest.java`; trong `WNT/section/`: `AdjustmentTableBlockTest.java`, `LandValuationNotesRegulatoryUnitTest.java`, `LegalBasisSectionRelatedJuridicalTest.java`, `AssetGroupsSectionShipTest.java`.
- Modify (test): `WNT/asset/AssetTypeWritersTest.java` (đổi chữ ký `adjustmentPriceUnit`, thêm ca SHIP).
- **Không sửa** `N/common/util/export/word/AppraisalReportWordExporter.java`, `CommonConstants.resolveRegulatoryConstraintsPriceUnit`.

**Interfaces:**

```java
// AssetTypeWriter — thay đổi
String adjustmentPriceUnit(AppraisalAssetGroup group);                 // H1 (thay cho bản không tham số)
String adjustmentCurrencyUnit(AppraisalAssetGroup group);              // H2
AdjustmentPriceLayout adjustmentPriceLayout();                         // H3
enum AdjustmentPriceLayout { MARKET_AND_CONVERTED, SALE_PRICE_ONLY }   // lồng trong AssetTypeWriter
int valuationSubOrdinal(boolean hasConstructionBlock);                 // H4
Set<Integer> subsumedRelatedJuridicalTypes();                          // H5
// GroupCompares
public record GroupCompares(AppraisalGroupCompare compare0, AppraisalGroupCompare compare1,
        AppraisalGroupCompare compare2, AppraisalGroupCompare compare3, AppraisalAssetGroup group) {}
// Ship*  (package asset)
final class ShipAssetWriter implements AssetTypeWriter { … 14 hook cũ + H1–H5 … }
public class ShipCompareTable extends WordBlock { public void write(GroupCompares compares); }
public class ShipCompareInfoBlock extends WordImages { public void write(AppraisalAssetGroup group); }
public class ShipValuationSummaryTable extends WordBlock { public void write(AppraisalAssetGroup group); }
class ShipAssetImages extends WordImages {
    void createShipAssetImages(AppraisalAssetGroup group, int sectionIndex);
    static LinkedHashMap<Integer, String> imageLabels();               // 11 loại theo thứ tự
}
// ResultSummaryRowsBlock
ResultSummaryRows ship(ResultSummaryTable t, AppraisalAssetGroup group, EvaluationResultDetailDto dto,
        String assetName, int firstStt);                                // roundedAmount (VNĐ), STT +1
// LegalBasisSection
static List<AssetTypeWriter> selectRelatedJuridicalWriters(List<AppraisalAssetGroup> groups);
```

`ShipAssetWriter.writeCompareTable` gọi lần lượt `new ShipCompareTable(ctx).write(compares)` rồi `new ShipCompareInfoBlock(ctx).write(compares.group())` (đúng thứ tự L, l.2843–2845).

- [ ] **Step 1: Đọc code và kiểm tra tên.** Đọc các đoạn LW trong bảng "Hành vi hệ thống cũ"; đọc `WN/asset/AssetTypeWriter.java`, `CarAssetWriter.java`, `CarCompareTable.java`, `CarValuationSummaryTable.java`, `CarAssetImages.java`, `ResultSummaryRowsBlock.java`, `WN/section/AssetGroupsSection.java`, `AdjustmentTableBlock.java`, `LegalBasisSection.java`, `LandValuationNotes.java`, `CompareTableBlock.java`, `WNT/asset/AssetTypeWritersTest.java`; xác nhận tên B01/B06 trong bảng "Điều kiện đầu vào"; xác nhận B08 đã merge (`ShipDescriptionWriter` có mặt). Ghi lệch tên vào báo cáo; không đoán.

- [ ] **Step 2: Test đặc tả hành vi cũ của bảng điều chỉnh (TRƯỚC khi sửa).** Tạo `AdjustmentTableBlockTest` với các ca cho LAND và CAR, chạy **PASS trên code chưa sửa** để làm lưới an toàn cho loại 1–3. Dựng nhóm bằng mock: `getAssetType()`, `getLandValuationMethod()=null`, `getAppraisalGroupCompares()` có 3 bản ghi type 1–3 (`deleteFlag=0`, `getMarketPrice`, `getLandUseRightPrice`), `getControlCompareFactors()` rỗng (và một ca có 1 yếu tố), `getControlCompareFactorSynthetic()` là mock có `getPriceGuide1..3`, `getAvgPriceGuide`… Cho `ctx` `appraisalResultService=null` nên `writeValuationSummaryTable` không ghi. Các ca:
  - `đất_cột_nhãn_và_đơn_vị_giữ_nguyên`: hàng 1 `A` / `Giá thị trường (giá trước điều chỉnh)` / `VND`; hàng 2 `B` / `Giá quy đổi về đơn vị so sánh chuẩn` / `VND/m²`; hàng 3 `C` / `Điều chỉnh các yếu tố so sánh`; sau `Mức điều chỉnh` đơn vị `VND/m²`; D `VND/m²`; D1, E1, E4 `VND`
  - `ô_tô_cột_nhãn_và_đơn_vị_giữ_nguyên`: như trên nhưng B và các dòng dùng `VND`
  - `đất_số_hàng_bảng_không_đổi`: bảng có hàng `A`, `B`, `C`, tức hàng 0 là header và hàng 3 là `C`
  Chạy: `cd backend && ./mvnw -q test -Dtest=AdjustmentTableBlockTest` — PASS (đây là baseline, nếu FAIL thì mock chưa đủ: sửa mock, không sửa code sản phẩm).

- [ ] **Step 3: Refactor hook (H1–H5, `GroupCompares.group`) — chưa có SHIP.**
  1. Viết trước các test FAIL trong `AssetTypeWritersTest`: đổi hai test `adjustmentPriceUnit_*` sang gọi `adjustmentPriceUnit(group)`; thêm `adjustmentCurrencyUnit_mọi_loại_cũ_là_VND`, `adjustmentPriceLayout_mọi_loại_cũ_là_MARKET_AND_CONVERTED`, `valuationSubOrdinal_loại_cũ_3_khi_có_ctxd_2_khi_không` (với `true` ra 3, `false` ra 2), `subsumedRelatedJuridicalTypes_chỉ_ô_tô_gồm_tàu` (CAR ra `Set.of(4)`, còn lại rỗng). Chạy: FAIL (lỗi biên dịch).
  2. Cài H1–H5 trong `AssetTypeWriter` (Javadoc tiếng Việt) và 4 writer cũ. Sửa `AdjustmentTableBlock` (dùng H1, H2, H3 nhưng với `MARKET_AND_CONVERTED` đi đúng đường cũ), `AssetGroupsSection` (`int landUseSubOrdinal = writer.valuationSubOrdinal(hasConstructionBlock)`), `GroupCompares` + `CompareTableBlock`, `LegalBasisSection` (hàm `selectRelatedJuridicalWriters`, `LandValuationNotes` (đơn vị theo tiền tệ).
  3. Test mới cho phần section: `LegalBasisSectionRelatedJuridicalTest` (xem Step 11) và `LandValuationNotesRegulatoryUnitTest` ca đất, căn hộ, null, ô tô (chưa có ca tàu).
  4. Chạy: `./mvnw -q test -Dtest='AssetTypeWritersTest,AdjustmentTableBlockTest,LandValuationNotesRegulatoryUnitTest,LegalBasisSectionRelatedJuridicalTest'` — PASS (đặc biệt `AdjustmentTableBlockTest` của Step 2 vẫn PASS, chứng minh loại 1–3 không đổi).
  5. Parity: nếu máy có MySQL theo `backend/.env.example` thì chạy `./mvnw -q test -Dtest=AppraisalWordExporterParityTest -Dparity=true` (có thể kèm `-Dparity.reportIds=…`) và đính kết quả vào báo cáo; không có DB thì ghi rõ "chưa chạy parity" và dựa vào `AdjustmentTableBlockTest`.
  6. Commit riêng: `git -C backend commit -am "refactor: thêm hook đơn vị tiền tệ, kiểu dòng giá, số mục thẩm định và văn bản pháp lý bao hàm cho AssetTypeWriter"`.

- [ ] **Step 4: `ShipCompareTable`.** Test FAIL trước (`ShipCompareTableTest`, dùng `GroupCompares` với 4 mock `AppraisalGroupCompare` và một nhóm mock):
  - `header_năm_cột_và_tỉ_lệ_cột`: 5 header đúng, tỉ lệ `18.5% / 20.375% ×4`
  - `đủ_14_hàng_đúng_thứ_tự_nhãn_khi_có_dữ_liệu`: nhãn cột 0 lần lượt `Chủng loại`, `Nguồn tham khảo`, `Thông tin liên hệ`, `Năm đóng`, `Nơi đóng`, `Mục đích sử dụng`, `Cấp phương tiện`, `Chất lượng còn lại (%)`, `Trọng tải toàn phần (tấn)`, `Kích thước (Lmax x Bmax)`, `Công suất`, `Giá rao bán (đồng)`, `Giá giao dịch (đồng)`, `Giá ước tính (đồng)`; số hàng bảng là 15
  - `hàng_cả_bốn_ô_rỗng_bị_bỏ`
  - `không_có_hàng_hình_ảnh_và_không_có_hàng_tỷ_lệ_ước_tính`
  - `nhãn_giá_theo_tiền_tệ_USD`: nhóm `currencyCode="USD"` thì `Giá rao bán (USD)`; nhóm có `currencyCode="USD"` nhưng `assetType` khác SHIP vẫn ra `đồng` (nhờ `resolveCurrencyCode`)
  - `năm_đóng_và_nơi_đóng_là_hai_hàng_tách`
  - `chất_lượng_còn_lại_có_dấu_phần_trăm`: `85.5`→`85,5%`
  - `trọng_tải_và_công_suất_định_dạng_việt_số_0_vẫn_in`: `0`→`0`
  - `giá_null_hoặc_0_để_trống_giá_dương_định_dạng_nghìn`: `1500000000`→`1.500.000.000`
  - `nhiều_dòng_ở_chủng_loại_nguồn_liên_hệ_tách_đoạn`: `a\nb` thành 2 đoạn trong ô
  - `compare_null_không_ném_lỗi`: `compare0 == null`
  Chạy FAIL → cài đặt (chép `createShipGroupCompareTable` + hai formatter; `moneyLabel` lấy từ `compares.group()`) → PASS.

- [ ] **Step 5: `ShipCompareInfoBlock` (thông tin và ảnh TSSS).** Test FAIL trước (`ShipCompareInfoBlockTest`):
  - `đoạn_đầy_đủ_có_giá_giao_dịch_và_CLCL`: địa chỉ `Tàu Sông Hồng 01`, trọng tải `1500`, năm `2012`, nơi đóng `Hải Phòng`, công suất `300`, giá giao dịch `2500000000`, CLCL `62.5`, type 1 → `-\tTài sản so sánh 1: là Tàu Sông Hồng 01; trọng tải toàn phần 1.500 tấn, năm đóng 2012, nơi đóng Hải Phòng, máy chính công suất 300 HP. Giá giao dịch là 2.500.000.000 đồng, chất lượng còn lại ước tính 62,5%.`
  - `không_có_giá_giao_dịch_thì_viết_hoa_Chất_lượng`: `… HP. Chất lượng còn lại ước tính 62,5%.`
  - `tiền_tệ_USD_in_USD`: `Giá giao dịch là 1.200 USD`
  - `phần_rỗng_hoặc_0_bị_bỏ`: trọng tải 0 và công suất 0 không có trong dòng; địa chỉ rỗng thì bỏ vế `là …`
  - `chỉ_có_type_không_có_dữ_liệu_thì_dòng_là_Tài_sản_so_sánh_k_chấm`: `-\tTài sản so sánh 2.`
  - `đoạn_in_đậm`
  - `link_tham_khảo_tách_theo_chấm_phẩy_mỗi_link_một_bullet`: `a;  ;b ` thì tiêu đề `Link tham khảo:` rồi 2 bullet `a`, `b`
  - `không_có_link_thì_không_có_tiêu_đề_Link_tham_khảo`
  - `bỏ_compare_type_0_và_đã_xóa`
  - `có_ảnh_thì_in_tiêu_đề_hình_ảnh_và_bảng_hai_cột`: mock `S3Service` trả PNG; 3 ảnh thì bảng 2 hàng, hàng cuối gộp ngang; tiêu đề `Hình ảnh tài sản so sánh 1` in đậm
  - `không_có_ảnh_thì_không_in_tiêu_đề_hình_ảnh`
  - `nhóm_không_có_compare_không_ghi_gì`
  Chạy FAIL → cài đặt (chép l.1115–1208 và `getValidComparisonImages`/`createComparativeImageTable`; hai hàm sau là private ở `ComparativeImagesSection`, nên chép vào lớp này, ghi Javadoc "cùng quy tắc với mục ảnh so sánh") → PASS.

- [ ] **Step 6: `ShipValuationSummaryTable`.** Test FAIL trước (`ShipValuationSummaryTableTest`). Dựng `ctx = WordTestKit.newContext(s3=null, svc=mock(AppraisalResultService.class), group)`; `when(svc.buildDetailFromEntities(any(), any(), any(), any(), eq(group), any())).thenReturn(dto)`; `dto` là `new EvaluationResultDetailDto()` có `setAssetName`, `setShip(...)`.
  - `dto_null_hoặc_ship_null_không_ghi_gì`
  - `vnd_ba_cột_tỉ_lệ_8_50_42_và_header`: header `STT`, `Hạng mục`, `Thành tiền (đồng)`
  - `vnd_các_hàng`: hàng 1 `1`, `Giá trị tàu hoàng phát` (chữ đầu thường), `1.234.567.000`; `Tổng giá trị` cùng số; `Làm tròn` = `roundedAmount`
  - `ngoại_tệ_bốn_cột_tỉ_lệ_8_40_26_26_và_header`: `Thành tiền (USD)`, `Thành tiền (đồng)`
  - `ngoại_tệ_hàng_1_và_tổng_có_cột_vnd`: `amount` ở cột 2 và `vndAmount` ở cột 3
  - `ngoại_tệ_vndAmount_null_dùng_amount`
  - `ngoại_tệ_làm_tròn_cột_ngoại_tệ_giữ_amount_cột_vnd_là_roundedAmount`
  - `tên_tài_sản_lấy_từ_danh_sách_asset_khi_dto_không_có`
  - `ghi_chú_chuyển_đổi_in_nghiêng_ngay_dưới_bảng_ngoại_tệ`: 2 dòng thì 2 đoạn nghiêng, đoạn đầu `spacingBefore=240`, đoạn sau `0`, `spacingAfter=60`, căn đều hai bên
  - `ghi_chú_chuyển_đổi_không_in_khi_vnd_dù_có_nội_dung`
  - `ghi_chú_chuyển_đổi_chỉ_khoảng_trắng_không_in`
  - `ghi_chú_kết_quả_in_sau_ghi_chú_chuyển_đổi`: thứ tự `Ghi chú: …` sau các đoạn nghiêng
  - `amount_null_thì_in_0`
  Chạy FAIL → cài đặt (chép l.5186–5300) → PASS.

- [ ] **Step 7: Dòng tổng hợp nhiều nhóm — `ResultSummaryRowsBlock.ship`.** Test FAIL trước (`ResultSummaryRowsBlockShipTest`, cùng package để gọi hàm package-private; dựng `ResultSummaryTable` bằng bảng 1 hàng header và 4 style):
  - `ghi_một_dòng_và_trả_roundedAmount_và_stt_kế_tiếp`: `amount()==roundedAmount`, `nextStt()==stt+1`; hàng có STT, `Giá trị tàu abc`, số tiền
  - `roundedAmount_là_vnd_dù_nhóm_là_usd`: `ship.amount=1000, rate=25000, rounded=25000000` thì số tiền cộng vào tổng là `25000000`
  - `ship_null_vẫn_ghi_dòng_với_số_0`
  - `hai_nhóm_tàu_cộng_dồn_và_tổng_nhãn_là_tài_sản`: gọi liên tiếp, tổng cộng đúng; nhãn tổng trong `ResultSummarySection` do `isRealEstate()` (kiểm bằng `AssetTypeWriters.forType(4).isRealEstate()==false` ở Step 9)
  Chạy FAIL → cài đặt (chép nhánh SHIP l.1489–1503 vào `ship(...)`) → PASS.

- [ ] **Step 8: `ShipAssetImages`.** Test FAIL trước (`ShipAssetImagesTest`):
  - `imageLabels_11_loại_đúng_thứ_tự_và_nhãn`: keys `[21,22,23,24,25,26,27,28,29,30,10]`, nhãn từng mã đúng bảng
  - `không_có_mô_tả_hoặc_không_có_ảnh_thì_không_ghi_gì`
  - `có_ảnh_nhưng_không_thuộc_loại_nào_trong_11_loại_thì_không_ghi_gì` (ví dụ loại 20 của xe)
  - `ảnh_đã_xóa_hoặc_path_rỗng_bị_loại`
  - `mỗi_loại_có_ảnh_in_một_bảng_theo_đúng_thứ_tự_nhãn`: ảnh loại 30, 21, 10 (nhập lộn xộn) thì bảng theo thứ tự 21, 30, 10; chú thích mặc định của nhóm là nhãn tương ứng (đọc từ ô ghi chú của bảng)
  - `nhiều_nhóm_in_đoạn_cấp_2_là_asset_đầu_tiên`: report có 2 nhóm; nhóm thứ 2 có `numberOf == "10.2."`
  - `một_nhóm_không_in_đoạn_cấp_2`
  Cần mock `S3Service` trả PNG (`getFileStreamFromS3` trả stream mới mỗi lần gọi). Chạy FAIL → cài đặt (chép `CarAssetImages`, đổi sang `getAppraisalShipDescription()` và `imageLabels()`) → PASS.

- [ ] **Step 9: `ShipAssetWriter`, đăng ký, và test hook.** Test FAIL trước, thêm vào `AssetTypeWritersTest` và `ShipAssetWriterTest`:
  - `forType_ánh_xạ_tàu`: `AssetTypeWriters.forType(4) instanceof ShipAssetWriter`
  - `singleGroupTitle_tàu`: `THẨM ĐỊNH GIÁ TRỊ TÀI SẢN`
  - `manyGroupsDescriptionLabel_tàu`: `Mô tả tài sản thẩm định`
  - `valuationTitle_và_chủ_thể_của_tàu`: `Thẩm định giá trị tài sản`, `giá trị tàu chở hàng khô`
  - `isRealEstate_tàu_false` và `hasGroupResultSummary_tàu_false`
  - `relatedJuridicalItems_tàu_đúng_bốn_văn_bản_nguyên_văn`: size 4, so từng chuỗi với bốn chuỗi trong "Bốn văn bản pháp lý liên quan của tàu"; khẳng định chuỗi 3 không chứa `thông tư số 147` (khác bản ô tô) và chuỗi 4 kết thúc bằng `.`
  - `addressSummaryLine_tàu_dùng_tên_tài_sản`: như đất (`thửa…` thành `Thửa…`), rỗng thì null
  - `adjustmentPriceUnit_tàu_theo_tiền_tệ`: nhóm `currencyCode="USD"` thì `USD`; `VND` thì `VND`; `null` thì `VND`
  - `adjustmentCurrencyUnit_tàu_giống_adjustmentPriceUnit`
  - `adjustmentPriceLayout_tàu_là_SALE_PRICE_ONLY`; mọi loại cũ là `MARKET_AND_CONVERTED`
  - `valuationSubOrdinal_tàu_luôn_là_4`: với cả `true` và `false`
  - `subsumedRelatedJuridicalTypes_tàu_rỗng`
  - `writeDescription_gọi_ShipDescriptionWriter`: nhóm tàu có mô tả tối thiểu thì thân tài liệu có `Mô tả tài sản thẩm định` (một nhóm) và trả đúng `sectionIndex`
  - `writeValuationSummaryTable_ủy_quyền_cho_bảng_tàu`, `writeCompareTable_ghi_bảng_rồi_thông_tin_TSSS` (body: bảng so sánh đứng trước đoạn `-\tTài sản so sánh 1`)
  - `writeResultSummaryRows_tàu_dùng_roundedAmount`
  - `writeAssetImages_ủy_quyền_cho_ShipAssetImages`
  Chạy FAIL → cài đặt `ShipAssetWriter` (14 hook cũ + H1–H5), thêm dòng `SHIP -> new ShipAssetWriter()` vào `AssetTypeWriters` → PASS.

- [ ] **Step 10: Bảng điều chỉnh của tàu (Khác hệ thống cũ #4) và Nguyên tắc khống chế.**
  1. Thêm vào `AdjustmentTableBlockTest` (FAIL trước):
     - `tàu_vnd_dòng_A_là_Giá_bán_đơn_vị_VND_không_có_Giá_thị_trường_và_B`: hàng 1 = `A`, `Giá bán`, `VND`; giá trị cột 4–6 = `landUseRightPrice` của TSSS 1–3 (`formatFloatToVietnameseStyle`), cột 3 (tài sản thẩm định) rỗng; hàng 2 = `C`, `Điều chỉnh các yếu tố so sánh`; **không** có ô nào chứa `Giá thị trường (giá trước điều chỉnh)` hay `Giá quy đổi về đơn vị so sánh chuẩn`
     - `tàu_usd_mọi_ô_đơn_vị_là_USD`: A, `Mức điều chỉnh`, `Mức giá sau điều chỉnh`, D, D1, E1, E4 đều `USD`; `Tỷ lệ điều chỉnh` vẫn `%`; `Tổng số lần điều chỉnh` vẫn `Lần`
     - `tàu_bảng_có_3_hàng_trước_yếu_tố_điều_chỉnh`: khi `getControlCompareFactors()` rỗng và không có synthetic thì bảng có đúng 3 hàng (header, A, C)
     - `tàu_vẫn_có_các_hàng_C_D_E_khi_có_yếu_tố_và_synthetic`
     - Các ca đất/ô tô của Step 2 vẫn PASS
  2. Thêm vào `LandValuationNotesRegulatoryUnitTest`: `tàu_vnd_đơn_vị_đồng` (kết quả `… đồng.`), `tàu_usd_đơn_vị_USD` (`… USD.`), `tàu_usd_làm_tròn_hàng_nghìn_có_hai_đơn_vị_USD` nếu `avgPriceGuideRoundingType=THOUSANDS` (`… USD, làm tròn … USD.`), `đất_căn_hộ_null_vẫn_đồng_trên_m2`, `ô_tô_đồng_trên_xe`, `nhóm_null_dùng_đồng_trên_m2`.
  3. Cài đặt trong `AdjustmentTableBlock`: khi `writer.adjustmentPriceLayout() == SALE_PRICE_ONLY` thì `createTable(3, 7, widths)`; hàng 1 ghi `A` / `Giá bán` / `currencyUnit` / giá trị `landUseRightPrice` (cột 4–6) và chỉ vậy; hàng 2 ghi `C` / `Điều chỉnh các yếu tố so sánh`. Ngược lại giữ đường cũ (4 hàng). Các ô đơn vị A, D1, E1, E4 dùng `writer.adjustmentCurrencyUnit(group)`, các ô còn lại dùng `writer.adjustmentPriceUnit(group)`. Không có `if` theo loại cụ thể trong section. Thêm Javadoc "Khác hệ thống cũ #4" vào `AdjustmentTableBlock`.
  4. Chạy: `./mvnw -q test -Dtest='AdjustmentTableBlockTest,LandValuationNotesRegulatoryUnitTest'` — PASS.

- [ ] **Step 11: Pháp lý liên quan và số mục nhiều nhóm.**
  1. `LegalBasisSectionRelatedJuridicalTest` (FAIL trước, gọi `LegalBasisSection.selectRelatedJuridicalWriters(groups)` từ cùng package): dùng group mock với `getDeleteFlag()=0`, `getAssetType()`:
     - `chỉ_tàu_ra_một_writer_bốn_văn_bản`
     - `chỉ_ô_tô_ra_một_writer_tám_văn_bản`
     - `có_cả_ô_tô_và_tàu_chỉ_giữ_ô_tô`: kết quả là writer của CAR; tổng số bullet là 8, không có chuỗi nào trong số 4 chuỗi của tàu
     - `hai_nhóm_tàu_chỉ_ghi_một_lần`
     - `đất_và_tàu_ra_writer_tàu`
     - `nhóm_đã_xóa_bị_bỏ_và_assetType_null_bị_bỏ`
     - `không_có_loại_nào_có_văn_bản_thì_rỗng`
  2. `AssetGroupsSectionShipTest` (FAIL trước): báo cáo mock có 2 nhóm SHIP (mock đủ để các block không ném lỗi: `getAppraisalGroupCompares()` rỗng, `getControlCompareFactors()` rỗng, `getAppraisalConstructionDescriptions()` rỗng, `getAppraisalReportAssets()` có một `asset`), gọi `new AssetGroupsSection(ctx).write()`:
     - `nhiều_nhóm_tàu_mục_thẩm_định_có_số_6_x_4`: có đoạn `Thẩm định giá trị tài sản` với `numberOf == "6.1.4."` và `"6.2.4."`
     - `nhiều_nhóm_tàu_tiêu_đề_mô_tả_là_6_x_1`: `Mô tả tài sản thẩm định` với `6.1.1.`
     - `một_nhóm_tàu_tiêu_đề_mục_6_và_mục_7`: đoạn mục 6 `THẨM ĐỊNH GIÁ TRỊ TÀI SẢN` và mục 7 `THẨM ĐỊNH GIÁ TRỊ TÀI SẢN` (viết hoa `valuationTitle`); các mục con `6.1.` (mô tả), `6.2.`, `6.3.` (nếu có tình trạng, chất lượng)
     - `nhóm_đất_vẫn_dùng_số_6_x_2_khi_không_có_ctxd` (hồi quy H4)
     Nếu mock quá nặng thì dùng entity thật qua `BeanUtils.instantiateClass`; ghi lại trong báo cáo.
  3. Cài đặt đã làm ở Step 3, ở đây chỉ bổ sung ca SHIP; chạy `./mvnw -q test -Dtest='LegalBasisSectionRelatedJuridicalTest,AssetGroupsSectionShipTest'` — PASS.

- [ ] **Step 12: Kiểm tra tổng, parity, quy tắc.**
  - `cd backend && ./mvnw -q test -Dtest='Ship*Test,AssetTypeWritersTest,AdjustmentTableBlockTest,LandValuationNotesRegulatoryUnitTest,LegalBasisSectionRelatedJuridicalTest,AssetGroupsSectionShipTest,ResultSummaryRowsBlockShipTest'` PASS; sau đó `./mvnw test` toàn bộ (test repository cần MySQL).
  - Parity loại 1–3: chạy lại `AppraisalWordExporterParityTest -Dparity=true` nếu có DB (như Step 3.5).
  - `git -C backend diff origin/develop -- src/main/java/com/thanhdo/tdprojectbe/common/util/export/word/AppraisalReportWordExporter.java` phải rỗng. `git grep -n "AppraisalAssetType.SHIP"` trong `…/appraisal/section/` phải không có kết quả (chỉ `ShipAssetWriter`, `AssetTypeWriters`, `CarAssetWriter` được nhắc SHIP).
  - Cập nhật Javadoc `AssetTypeWriter` (hướng dẫn mở rộng), và **không sửa** `backend/CLAUDE.md` (hàng "Thêm loại tài sản / mục vào báo cáo Word thẩm định" vẫn đúng; nếu cần thêm dòng thì ghi vào báo cáo cuối task để controller gộp sau đợt).

- [ ] **Step 13: Commit** (trong submodule, có thể tách thêm commit cho phần SHIP): `git -C backend add -A && git -C backend commit -m "feat: Word tàu thủy (ShipAssetWriter, bảng so sánh, kết quả, ảnh, bảng điều chỉnh theo tiền tệ)"`.

**Review Focus B09:**
1. Không còn `if` theo loại tài sản trong section; mọi khác biệt qua hook. Loại 1–3 không đổi output (baseline `AdjustmentTableBlockTest` + parity).
2. Bốn văn bản pháp lý của tàu nguyên văn và không bị trùng với bộ của xe; báo cáo có cả xe và tàu chỉ in bộ của xe.
3. Số mục nhiều nhóm tàu là `.1` mô tả, `.2` tình trạng, `.3` chất lượng, `.4` thẩm định.
4. Tiền tệ: nhãn giá `(đồng)`/`(USD)`, bảng kết quả 3 hoặc 4 cột, ghi chú chuyển đổi chỉ khi ngoại tệ, đơn vị bảng điều chỉnh, đơn vị "Nguyên tắc khống chế"; dòng tổng hợp nhiều nhóm luôn là VNĐ.
5. Bảng điều chỉnh của tàu: A `Giá bán`, không có dòng Giá thị trường và B (Khác hệ thống cũ #4), được ghi ở Javadoc.
6. Không sửa exporter Word cũ.

---

## Vấn đề cần controller quyết định

1. **→ Đã chốt: Quyết định #1 (khóa JSON kết quả tàu là `ship`, DTO Word cũng `ship`, nên `getShip()` khớp cả hai).** **Tên giả định từ B01/B06.** Plan dựa vào quy ước tên trong `00-index.md` nhưng B01/B06 chưa có code để đối chiếu: getter entity (`getShipName()`…), record `ShipConditionDetail`/`ShipQualityEvaluationItem` (thứ tự tham số, accessor kiểu record), `EvaluationResultDetailDto.getShip()` kiểu `ShipEvaluationDetail` (index ghi JSON `ship`), `ReportConstants.AppraisalImageType.SHIP_*`, `CurrencyCodes.*`. Fixture test (`ShipFixtures`) cô lập thứ tự record tại một nơi. Implementer phải grep lại ở Step 1 và báo nếu lệch.
2. **→ Đã chốt: Quyết định #18 (chấp nhận).** **B09 thêm 5 hook mới và một thành phần cho `GroupCompares`.** Bắt buộc để SHIP không thêm `if` vào section (nhãn dòng A/B và đơn vị theo tiền tệ, số mục `.4` của nhiều nhóm, bộ văn bản xe bao hàm bộ tàu, nhãn giá bảng so sánh cần `currencyCode`). B09 vì vậy sửa 4 writer cũ, `AssetGroupsSection`, `AdjustmentTableBlock`, `CompareTableBlock`, `LegalBasisSection`, `LandValuationNotes`, `AssetTypeWritersTest`. Output loại 1–3 được kiểm bằng test baseline (Step 2) và golden parity (cần MySQL). Cần controller chấp nhận phạm vi sửa này và xếp B09 sau B06 và B08, không chạy song song với task khác sửa các file section nêu trên.
3. **→ Đã chốt: Quyết định #18 (parity `-Dparity=true` chạy một lần ở Z01).** **`AppraisalWordExporterParityTest` cần MySQL và tắt mặc định.** Nếu máy chạy plan không có DB dev thì parity không chạy được; thay bằng `AdjustmentTableBlockTest`, `AssetTypeWritersTest` và test hook. Cần controller chạy parity một lần ở `Z01`.
4. **Tổng CLCL trong Word không bị chặn 0–100** (L in tổng thô), trong khi `ShipQuality.remainingQualityPct` trả null ngoài khoảng (giá trị ghi sang so sánh). Plan giữ hành vi L cho Word và có test ghi nhận khác biệt; nếu muốn Word cũng ẩn hoặc cảnh báo khi ngoài khoảng thì đó là thay đổi sản phẩm, cần quyết định riêng.
5. **Ảnh TSSS có thể in hai lần** (mục 6 qua `ShipCompareInfoBlock` và mục 11 `ComparativeImagesSection`) giống L. Plan giữ nguyên để khớp L; nếu muốn bỏ ảnh ở một trong hai chỗ thì cần quyết định.
6. **Số tiền USD kiểu `Long`.** `ShipEvaluationDetail.amount` là `Long` như L nên bảng kết quả ngoại tệ không hiển thị phần lẻ của USD. Giữ nguyên theo hợp đồng B06 trong `00-index.md`; nếu B06 đổi sang `BigDecimal` thì `ShipValuationSummaryTable` và `ResultSummaryRowsBlock.ship` phải sửa theo.
