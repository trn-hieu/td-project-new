# Khảo sát giao diện và Word legacy SHIP — trạng thái tại `82193fe5`

> Tài liệu gộp hai lần khảo sát (bản `a9d647d4` và phần thay đổi tới `82193fe5`), viết theo trạng thái cuối cùng. Số dòng chỉ gần đúng; luôn đọc lại code bằng `git -C td-project-legacy show 82193fe5:<path>`.
>
> Ký hiệu đường dẫn:
> - `T/` = `src/main/resources/templates/`
> - `J/` = `src/main/resources/static/js/`
> - `LW` = `src/main/java/com/jvb/thanhdo/controller/component/wordExport/AppraisalReportWordExporter.java`

## 1. Màn tạo và sửa thông tin cơ bản (`T/demo/basic-information.html`, `basic-information-edit.html`)

- Ô chọn loại tài sản của từng pháp lý có thêm `<option value="4">Tàu thủy</option>`. Bản edit đánh dấu `selected` khi nhóm có assetType = 4.
- Phần cơ sở thẩm định có thêm tab thứ 5 là "Tàu thủy" (`data-basis-tab="4"`), panel `basisItemsShip`, lấy danh sách từ model `valuationBasisListShip`. Payload là `valuationBasisJson[idx+shipOffset]` gồm `{id, assetType=4, checked, content}`.
- Màn này không chọn tiền tệ. Trường "cho ai" nằm ở tab Pháp lý.

## 2. Màn edit-tab báo cáo (`T/demo/demo-report-step-2.html`)

Nhóm SHIP có 5 tab, theo thứ tự:
1. Pháp lý (`legal-document-ship`)
2. **Mô tả tàu thủy** (mặc định đang mở, data-tab-type `land`)
3. Thông tin so sánh (`comparative-information-ship`)
4. Bảng điều chỉnh (dùng chung `appraisal-panel`)
5. Kết quả thẩm định (dùng chung `evaluation-results`)

Tab "Mô tả CTXD" chỉ có ở LAND.

Modal thêm ảnh dùng chung, có thêm hidden `shipDescriptionId` và các option ảnh 21–30. Các mã ảnh và nhãn:

| Mã | Nhãn |
|---|---|
| 21 | Mặt trước tàu |
| 22 | Đầu tàu |
| 23 | Đuôi tàu |
| 24 | Mạn tàu |
| 25 | Sàn tàu |
| 26 | Số IMO/Số kiểm soát |
| 27 | Hệ thống neo |
| 28 | Phòng lái/phòng nghỉ |
| 29 | Khoang hàng |
| 30 | Hệ thống máy của động cơ |
| 10 | Khác |

Trang nạp thêm `ship-description.js`, modal tìm TSSS tàu (`comparative-information-ship-search-modals.html`) và `comparative-information-ship-search.js`. Mỗi lần tab So sánh được tải, `demo-report-step-2-tabs.js` gọi `initShipComparePane`.

## 3. Tab Mô tả tàu (`T/demo/ship-description.html`, `J/ship-description.js`)

### Bố cục

- Hàng đầu tiên: **Nơi neo đậu** (`mooringLocation`, textarea, tối đa 255 ký tự).
- **I. Mô tả tàu thủy**, gồm các khối theo thứ tự:
  1. Thông số chính
  2. Thiết kế phương tiện
  3. Thiết bị neo
  4. Thiết bị lái
  5. Thiết bị chằng buộc
  6. **Thiết bị kéo** (Kiểu cột kéo, Kiểu mọc kéo, Kiểu tời thu dây, Loại dây kéo; textarea tối đa 1000 ký tự)
  7. Máy chính (nhãn công suất là "Công suất (HP/KW)")
  8. Máy phụ
  9. Thiết bị điện
  10. Hệ thống khí nén
  11. Thiết bị nâng
  12. Trang bị chữa cháy (có tiêu đề phụ "Dụng cụ và sơ đồ chữa cháy")
  13. Trang bị cứu sinh
  14. Trang bị cứu đắm
  15. Trang bị tín hiệu (có tiêu đề phụ Đèn hiệu, Âm hiệu, Vật hiệu)
  16. Trang bị chạy tàu
  17. Trang bị ngừa ô nhiễm
  18. Thời hạn kiểm tra
- **II. Tình trạng của tài sản thẩm định**
- **III. Đánh giá chất lượng còn lại của tàu**
- **IV. Hình ảnh**: 11 mục, chỉ hiện khi đã có `ship.id`.
- Footer: nút Tải báo cáo và nút Lưu.

Trong Thông số chính, ô "Năm đóng và nơi đóng" cũ đã tách thành **Năm đóng** (`buildYear`, số nguyên tối đa 4 chữ số) và **Nơi đóng** (`buildPlace`, textarea tối đa 1000 ký tự).

### Kiểu input

| Kiểu | Cách nhập | Ghi chú |
|---|---|---|
| Text | textarea tự giãn, chặn phím Enter | `operationNote` là textarea 3 dòng, tối đa 5000 ký tự |
| Số nguyên | `.ship-integer`, chỉ cho nhập chữ số | `maxlength` 9; với năm (`buildYear`, `designYear`, `mainEngineManufactureYear`) là 4 |
| Thập phân | `.ship-decimal`, hiển thị kiểu châu Âu `1.234,50` | xem rule `shipDecimal` bên dưới |
| Boolean | checkbox | 16 đèn hiệu, 10 vật hiệu, 11 trang bị chạy tàu, `fireManualTools`, `fireLayoutPlan`. `navigationRadio`, `navigationAis`, `navigationSart` là text |
| Ngày | `type=date` (ISO) | `inspectionPeriodicDate`, `inspectionDockDate`, `inspectionAnnualDate` |

Rule `shipDecimal` của ô thập phân:
- Không âm.
- Regex `^(\d{1,3}(\.\d{3})+|\d+)(,\d{1,2})?$`.
- Phần nguyên tối đa 8 chữ số. Riêng 13 field DECIMAL(12,2) được 10 chữ số: `cargoCapacity`, `deadweight`, `anchorBowTotalWeightKg`, `steeringRudderTotalAreaM2`, `steering*TorqueTm`, `mooringRopeTotalLengthM`, `firePumpTotalFlowM3`, `fireGasTotalWeightKg`, `fireCo2BottleWeightKg`, `firePowderBottleWeightKg`, `damageControlBilgePumpTotalFlowM3`, `pollutionOilyWaterTankTotalVolumeM3`.
- `anchorSternTotalWeightKg` nay là text (1000 ký tự), không còn là ô thập phân.

Độ dài text lấy theo `TEXT_MAX` trong JS, trùng với độ dài cột DB. Khi `maxlength` của HTML khác JS thì **JS là chuẩn**. Các trường dạng "số lượng/thể tích" mà DB lưu TEXT đều là text tối đa 1000 ký tự.

### Bảng II: Tình trạng

- Có nút "+ Thêm hạng mục" và nút xóa từng dòng; STT tự đánh lại.
- Cột: STT, Hạng mục (tối đa 500 ký tự), Tình trạng (không giới hạn), Xóa.
- Khi lưu:
  - Bỏ dòng mà cả hai ô đều rỗng.
  - Dòng có Tình trạng nhưng thiếu Hạng mục thì đánh `is-invalid` và hiện toast "Vui lòng nhập hạng mục cho các dòng tình trạng đã nhập".

### Bảng III: Đánh giá chất lượng (cấu trúc mới)

- **Mỗi dòng là một item.** Ô "Hạng mục thẩm định" chứa **nhiều ô nhập tên** (textarea tối đa 500 ký tự), mỗi ô có nút × để xóa. Bên dưới có nút "+ Hạng mục". Bấm × ở ô cuối cùng thì chỉ xóa nội dung, không xóa ô.
- Nút "+ Thêm dòng" để thêm item; nút xóa dòng xóa ngay, không hỏi xác nhận.
- 9 cột, `colgroup` theo tỉ lệ 5/22/9/10/12/10/11/11/10%:

| # | Cột | Ghi chú |
|---|---|---|
| 1 | STT | |
| 2 | Hạng mục thẩm định | nhiều ô tên |
| 3 | (1) Hệ số quan trọng | |
| 4 | (2) Giá trị chuẩn so sánh | |
| 5 | Đánh giá | select: `-- Chọn --`, Tốt, Khá, Trung bình, Kém |
| 6 | (3) | giá trị số |
| 7 | (4)=(3)/(2) | chỉ đọc |
| 8 | (5)=(1)*(4) | chỉ đọc |
| 9 | Thao tác | |

- Chân bảng: "Chất lượng còn lại của tàu" (colspan 7) và ô tổng.
- Khi chưa có dòng nào, hiện dòng "Chưa có dữ liệu".
- Ô số: tối đa 11 ký tự, chỉ cho chữ số và **một** dấu phẩy, tối đa 8 chữ số nguyên và 2 chữ số thập phân. Không cho dấu chấm (không có phân cách nghìn).
- **Công thức**, với `w`, `s`, `r` lần lượt là (1), (2), (3):
  - `raw4 = (s != null && s > 0 && r != null) ? r / s : null`
  - (4) hiển thị `Math.round((raw4 + EPSILON) * 10) / 10`, định dạng `de-DE`, đúng 1 chữ số thập phân.
  - `c5 = (w != null && raw4 != null) ? w × raw4 × 100 : null`. Dùng **raw4 chưa làm tròn**. Hiển thị `de-DE` 2 chữ số thập phân kèm "%".
  - Tổng = Σ c5, hiển thị 2 chữ số kèm "%". Chỉ hiện khi có ít nhất 1 dòng.
  - Đánh giá (Tốt/Khá/…) không tham gia tính toán.
- **Validate**:
  - Dòng có số liệu mà không có tên nào thì `is-invalid` ở ô tên đầu tiên.
  - `s <= 0` thì `is-invalid`.
  - Toast: "Đánh giá chất lượng: vui lòng nhập tên hạng mục và giá trị chuẩn so sánh lớn hơn 0".
- **Payload**: `qualityEvaluationDetails: [{itemNames: string[], importanceWeight, standardValue, relativeValue, rating|null}]`. Dòng rỗng hoàn toàn bị bỏ.

### Ảnh

- 11 mục, mỗi mục có nút "+ Thêm ảnh", mở modal chung đã khóa sẵn loại ảnh.
- Ảnh gom theo `groupUuid`, hiển thị lưới 2 cột kèm caption `note`. Mỗi ảnh có nút xóa.

### Lưu

- Gọi `PUT /api/appraisal-ship-descriptions/{shipId}` với JSON:
  - text: trim, rỗng thành null
  - thập phân: parse kiểu châu Âu
  - số nguyên: `parseInt`
  - boolean: theo `checked`
  - ngày: `yyyy-MM-dd`
- Sau khi lưu: hiện toast, patch các ô cột Tài sản thẩm định ở tab So sánh (`updateShipCompareSyncedFields`), rồi tải lại tab.

## 4. Tab Pháp lý tàu (`T/demo/legal-document-ship.html`)

Mỗi tài sản trong nhóm có một pill riêng. Form gửi multipart tới `/appraisal-report/save/legal-tab/{reportId}`. Các mục:

1. **Giấy chứng nhận đăng ký phương tiện thủy nội địa**
   - 1.1 Số GCN: `certificateNumber`
   - 1.2 Nơi cấp: `cnIssuedPlace`
   - 1.3 Ngày cấp: `gcnNoIssuedDate`
   - 1.4 Cho ai: `forWhom`
2. **Giấy chứng nhận an toàn kỹ thuật và bảo vệ môi trường phương tiện thủy nội địa**
   - 2.1 Số GCN: `shipInspectionCertificateNumber`
   - 2.2 Nơi cấp: `shipInspectionIssuedPlace`
   - 2.3 Ngày cấp: `shipInspectionIssuedDate`
   - 2.4 Ngày hết hiệu lực: `shipInspectionExpiryDate`
   - 2.5 Cho ai: `shipInspectionForWhom`
3. **Giấy tờ pháp lý của tài sản**: textarea `legalDocuments`, gồm 2 dòng tự sinh và phần nhập tay. Word xuất trường này.
4. **File đính kèm**.

Tự sinh dòng (`updateLegalDocumentsShip`):
- Hai dòng bắt đầu bằng hai tiền tố đúng bằng tên giấy chứng nhận ở mục 1 và 2.
- Mỗi dòng ghép các phần sau, bỏ phần rỗng, và kết thúc bằng dấu "." Ngày đổi sang dạng `dd tháng MM năm yyyy`.
  - Dòng 1: ` số: …` + ` do …` + ` cấp ngày …` + ` cho …`
  - Dòng 2: ` số: …` + ` do …` + ` ngày …` + ` cho …`
- Nếu có ngày hết hiệu lực, dòng 2 nối thêm câu "Giấy chứng nhận này có hiệu lực đến ngày … .".
- Hai dòng tự sinh luôn nằm **đầu**. Các dòng nhập tay (không bắt đầu bằng hai tiền tố) giữ nguyên bên dưới.

Validate: 6 trường text (`certificateNumber`, `cnIssuedPlace`, `forWhom`, `shipInspectionCertificateNumber`, `shipInspectionIssuedPlace`, `shipInspectionForWhom`) tối đa 500 ký tự.

## 5. Tab So sánh tàu (`T/demo/comparative-information-ship.html`, `J/comparative-information-ship-search.js`)

Form có `data-currency-code` của nhóm, hidden `compareInformationId` cho các cột 1–3 và hidden `syncToCompareInfo`. Bảng 5 cột: Chỉ tiêu, Tài sản thẩm định, TSSS 1, TSSS 2, TSSS 3. **Cả 4 cột đều nhập tay được.**

| # | Dòng | Field | Kiểu |
|---|---|---|---|
| 1 | Chủng loại | `address` | textarea 1000. Cột 1–3 có nút kính lúp "Tìm TSSS theo chủng loại" |
| 2 | Nguồn tham khảo | `source` | textarea 500 |
| 3 | Thông tin liên hệ | `contact` | textarea 255 |
| 4 | **Đồng bộ lên cơ sở dữ liệu** | `syncToCompareInfo` | checkbox "Đồng bộ TS so sánh N" ở cột 1–3; cột TĐG hiện "—" |
| 5 | Hình ảnh tài sản | — | cột 1–3 có nút "Xem hình ảnh" và "Tải hình ảnh" |
| 6 | Năm đóng | `shipBuildYear` | number 1900–2100 |
| 7 | Nơi đóng | `shipBuildPlace` | text 1000 |
| 8 | Mục đích sử dụng | `shipUsagePurpose` | textarea 500 |
| 9 | Cấp phương tiện | `shipClass` | textarea 500 |
| 10 | Chất lượng còn lại (%) | `shipRemainingQualityPct` | số kiểu châu Âu, 0–100 ("CLCL phải từ 0 đến 100") |
| 11 | Trọng tải toàn phần (tấn) | `shipDeadweight` | số kiểu châu Âu |
| 12 | Kích thước (Lmax x Bmax) | `shipDimension` | text 255 |
| 13 | Công suất (HP) | `shipMainEnginePowerHp` | số kiểu châu Âu |
| 14 | Giá rao bán / Giá giao dịch / Giá ước tính (đơn vị) | `forSalePrice`, `tradePrice`, `estimatePrice` | `format-number`. Đơn vị là `currencyMoneyLabel`: "(đồng)" hoặc "(USD)" |
| 15 | Tỷ lệ ước tính | — | chỉ hiển thị, = ước tính / rao bán × 100, dạng `xx,xx %`. Rỗng nếu thiếu số hoặc giá rao bán bằng 0 |

Không có ô USD riêng, không có tỷ giá, không có nút đồng bộ từ mô tả. Việc đồng bộ xảy ra ở server khi lưu mô tả tàu.

### Checkbox đồng bộ kho

- Quyền: KSV, TPTD, **BGD**. Trước đây chỉ KSV và TPTD; đổi này áp dụng cho **mọi loại tài sản**, cả đất, ô tô và căn hộ.
- Checkbox bị disabled khi cột đã chọn TSSS từ kho, hoặc khi người dùng không có quyền.
- Người không có quyền mà cột đang đồng bộ thì vẫn giữ giá trị 1 qua hidden.
- Ghi chú nhãn: "Tích chọn để đưa tài sản so sánh vào cơ sở dữ liệu; tài sản đã đồng bộ sẽ hiển thị khi tìm kiếm theo chủng loại."

### Modal tìm TSSS tàu

- Tiêu đề "Tìm kiếm tài sản so sánh (tàu thủy)". Ô "Chủng loại / Từ khóa" kèm nút Tìm kiếm.
- Hai cột: Danh sách kết quả và Chi tiết tài sản. Danh sách cuộn tới gần cuối (cách đáy 80px) thì tải thêm.
- API: `GET …/search-compare-informations?keyword&assetType=4&pageNo&pageSize=20`.
- Khi mở modal, lấy sẵn nội dung ô Chủng loại của cột làm từ khóa và tự tìm.
- Mỗi item hiển thị: tiêu đề `address`; Năm đóng, Trọng tải, Công suất; Kích thước, CLCL; giá rao bán và giá giao dịch kèm đơn vị (`đ` nếu VND, ngược lại là mã tiền); Nguồn; ngày cập nhật. Có hai nút **Chọn TSSS** và **Chi tiết**.
- Panel chi tiết gồm: Chủng loại, Nguồn, Liên hệ, Năm đóng, Nơi đóng, Mục đích, Cấp, CLCL, Trọng tải, Kích thước, Công suất, Loại tiền, 3 loại giá, Ngày cập nhật.
- **Khác tiền tệ với nhóm**: hiện badge vàng ghi mã tiền ("Khác đơn vị tiền tệ của nhóm (X)"), nút Chọn bị disabled với tooltip "Không thể chọn: TSSS dùng X, nhóm dùng Y".
- **Khi Chọn**, điền vào cột idx:
  - `address`, `source`, `contact`, `shipBuildYear`, `shipBuildPlace`, `shipUsagePurpose`, `shipClass`, `shipDimension`
  - CLCL, trọng tải, công suất (định dạng châu Âu)
  - 3 loại giá (làm tròn số nguyên)

  Sau đó gán `compareInformationId`, khóa checkbox đồng bộ, hiện nút "Bỏ chọn", rồi toast "Tải dữ liệu tài sản so sánh thành công".
- **Nút "Bỏ chọn tài sản so sánh"** (chỉ hiện khi cột đang gắn với kho): xóa `compareInformationId` và 14 ô ở trên, mở lại checkbox (nếu có quyền), toast "Đã bỏ chọn tài sản so sánh {idx}.".

## 6. Bảng điều chỉnh (`T/demo/appraisal-panel.html`)

- `currencyUnit = currencyUnitLabel(resolveCurrencyCode(group))`, giá trị là VND hoặc USD.
- SHIP **ẩn dòng "A. Giá thị trường (giá trước điều chỉnh)"**. Dòng giá đầu tiên của tàu: STT là **A**, nhãn **"Giá bán"**, đơn vị là `currencyUnit`. Để so sánh: đất dùng B "Giá quy đổi…" đơn vị VND/m²; ô tô dùng B "Đơn giá".
- Các ô đơn vị sau dùng `currencyUnit` cho CAR và SHIP (đất vẫn là "VND/m²"): Mức điều chỉnh, Mức giá sau điều chỉnh, Mức giá chỉ dẫn, D1, E1, E4.
- Câu "Nguyên tắc khống chế" dùng `regPriceUnit = resolveRegulatoryConstraintsPriceUnit(assetType, currencyCode)`. Với tàu ra "đồng" hoặc "USD".
- Công thức điều chỉnh không đổi (dùng chung `appraisal-panel-tabbed.js`). "Giá bán" lấy từ `landUseRightPrice` mà server đã tính.

## 7. Kết quả thẩm định (`T/appraisal-records/evaluation-results.html`)

### Khối tiền tệ (chỉ SHIP)

- Select `currencyCode` có VNĐ và USD, `data-initial` là giá trị ban đầu.
- Khi chọn USD thì hiện thêm:
  - "Tỷ lệ chuyển đổi (1 USD = ? VNĐ)": chỉ nhận chữ số, tối đa 9 chữ số; rời ô thì định dạng `toLocaleString('vi-VN')`, ví dụ `25.450`.
  - "Ghi chú chuyển đổi": textarea tối đa 1000 ký tự.
- Đổi sang VND thì tỷ lệ về 1.
- Khi lưu:
  - USD có nhập tỷ lệ: kiểm tra `^[1-9][0-9]{0,8}$`. Sai thì báo "Tỷ lệ chuyển đổi phải là số nguyên từ 1 đến 999.999.999". Để trống thì gửi null.
  - Payload thêm `currencyCode`, `exchangeRate` (VND gửi 1), `currencyNote`.
  - Lưu xong mà tiền tệ khác ban đầu thì đánh dấu tab So sánh và tab Bảng điều chỉnh "chưa tải", rồi vẽ lại kết quả.

### Bảng kết quả tàu

Dữ liệu lấy từ `data.ship.{amount, vndAmount, roundedAmount}`, `data.currencyCode`, `data.totalPriceWordAppraisal`, `data.assetName`.

**VND**: 2 cột, "Hạng mục" (58%) và "Thành tiền (VNĐ)" (42%).

| Hạng mục | Thành tiền |
|---|---|
| "Giá trị {tên tài sản, chữ đầu viết thường}" | `amount` |
| **Làm tròn** | `roundedAmount` |
| Bằng chữ: … ./. (colspan 2) | |

**Ngoại tệ**: 3 cột, tỷ lệ 40/30/30.

| Hạng mục | Thành tiền ({mã}) | Thành tiền (VNĐ) |
|---|---|---|
| Dòng 1 | `amount` | `vndAmount ?? amount` |
| Làm tròn | `amount` (giữ nguyên, không làm tròn) | `roundedAmount` |
| Bằng chữ (colspan 3) | | |

Số tiền bằng chữ luôn đọc theo VNĐ.

## 8. Word (`LW`)

### Khung chung

- Cờ `hasShipGroup`.
- Một nhóm tàu: tiêu đề mục 6 là "THẨM ĐỊNH GIÁ TRỊ TÀI SẢN". Nhiều nhóm: tiêu đề `6.i.1` là "Mô tả tài sản thẩm định".
- Mục con của nhóm tàu: `.1` Mô tả, `.2` Tình trạng, `.3` Đánh giá chất lượng, `.4` phần chung (biến `landUseSubOrdinal` = 4).
- Tiêu đề mục thẩm định giá: "Thẩm định giá trị tài sản".

### Pháp lý liên quan

Mục 2.1.2 hiện khi có xe hoặc tàu. Có xe thì dùng bộ văn bản của xe (đã gồm văn bản tàu). Chỉ có tàu thì dùng `createShipRelatedJuridical` (`LW` ~6341-6353), gồm 4 bullet:
- Thông tư 45/2013/TT-BTC ngày 25/04/2013
- Thông tư 147/2016/TT-BTC ngày 13/10/2016
- Thông tư 28/2017/TT-BTC ngày 12/04/2017
- Quyết định 57/1999/QĐ-TĐC ngày 11/03/1999

Lấy nguyên văn từ code.

### Cụm chủ thể (`appraisalOwnershipPhrase`)

SHIP dùng "giá trị tàu chở hàng khô". Các loại khác giữ như cũ: căn hộ "giá trị quyền sở hữu căn hộ chung cư", ô tô "giá trị phương tiện vận tải", đất "giá trị quyền sử dụng đất", còn lại "giá trị tài sản".

### Mô tả tàu (`createShipDescription`, ~6800)

- Dạng danh sách, không dùng bảng. `writeShipSection(title, lines)`: tiêu đề in đậm kiểu diamond, mỗi dòng là một bullet. Khối không có dòng nào thì bỏ cả tiêu đề.
- `addShipLine` bỏ qua giá trị null và tự thêm dấu "." nếu cuối dòng chưa có "." hoặc ":".
- Thứ tự khối giống web. Có thêm khối **Thiết bị kéo** đặt trước Máy chính, gồm 4 dòng "Kiểu cột kéo:", "Kiểu mọc kéo:", "Kiểu tời thu dây:", "Loại dây kéo:".
- Các dòng ghép:
  - "Kích thước cơ bản: L x B x D x d (m) (L x B x D x d)"
  - "Lmax x Bmax: … m"
  - "Số lượng máy chính/phụ: a/b"
  - "Tổng công suất: x/y sức ngựa"
  - "Khả năng khai thác: Lượng hàng: … (tấn); TTTP: … (tấn)"
  - "Năm đóng:" và "Nơi đóng:" là hai dòng riêng.
  - Mỗi dòng của `operationNote` thành một bullet.
- Helper:
  - `shipNum`: số kiểu Việt.
  - `shipDate`: dd/MM/yyyy.
  - `shipYesNo`: true thành "có", false thành "-/-", null thì bỏ dòng.
  - `shipJoin`/`shipSegments`: phần thiếu ghi "-/-"; thiếu tất cả thì bỏ dòng.
- Neo đuôi: tổng khối lượng in nguyên chuỗi.
- Máy chính: "Công suất (HP/KW): ".
- Bơm chữa cháy dùng "m³". Các chỗ khác vẫn ghi "m3" như cũ.
- **Trang bị tín hiệu**: tiêu đề in đậm, dòng "Đèn hiệu:" kèm **bảng lưới** `writeShipYesNoGrid`, rồi "Âm hiệu: …", rồi "Vật hiệu:" kèm bảng lưới. Quy cách bảng lưới:
  - Mỗi ô là cặp (nhãn, ☑ hoặc ☐); giá trị null thì bỏ qua.
  - 8 cột, tức 4 cặp mỗi hàng. Nhãn dài hơn 20 ký tự chiếm 2 chỗ (nhãn `gridSpan` 3 cột cộng 1 cột giá trị); không đủ chỗ thì xuống hàng.
  - Cột nhãn rộng 1702 twips, cột giá trị 662. Có khai báo `tblGrid`, layout fixed, không viền.
  - Chữ cỡ 11. Ký tự checkbox dùng font "Segoe UI Symbol" cỡ 12, căn giữa.

### Tình trạng (`writeShipConditionTable`, ~7066)

- Số mục `{6}.2.` hoặc `{6.i}.2.`, tiêu đề "Tình trạng của tài sản thẩm định".
- **Đoạn mở đầu** (chữ thường):
  - `subject` = "tài sản thẩm định" nếu không có tên; giữ nguyên tên nếu đã bắt đầu bằng "tàu " (không phân biệt hoa thường); ngược lại là "Tàu {tên}".
  - Câu ghép: "Tại thời điểm thẩm định giá " + subject + [", số đăng ký: …"] + [", năm đóng: …"] + [", nơi đóng: …"] + [" đang neo đậu tại {mooringLocation}"]. Thêm "." nếu chưa có.
  - Nối thêm nguyên văn: " Tiếp cận kiểm tra các bộ phận nhận thấy các chi tiết có thể tiếp cận được tổ thẩm định nhận thấy tình trạng các bộ phận như sau:"
- Bảng 3 cột, tỷ lệ 8/32/60%, header in đậm căn giữa. Ô "Tình trạng" tách mỗi dòng thành một đoạn.
- Chỉ in dòng có dữ liệu; không có dòng nào thì bỏ cả mục.

### Đánh giá chất lượng (`writeShipQualityTable`, ~7156)

- Số mục `.3.`, tiêu đề "Đánh giá chất lượng còn lại của tài sản thẩm định:".
- **Đoạn dẫn** (nguyên văn; "QĐ–TĐC" dùng gạch en-dash): "Căn cứ quyết định số 57/1999/QĐ–TĐC ngày 11/03/1999 của Tổng cục đo lường chất lượng và thông tư số 45/2013/TT-BTC ngày 25/04/2013 của Bộ Tài Chính để đánh giá chất lượng còn lại (CLCL) của từng bộ phận và chất lượng của tàu. Kết quả được thể hiện trong bảng tính sau:"
- Bảng 8 cột, tỷ lệ 5/25/9/10/11/10/15/15%. Header 2 hàng:
  - Hàng 1: STT, Hạng mục thẩm định, Hệ số quan trọng, Giá trị chuẩn so sánh, Đánh giá (gộp 2 ô), Giá trị chỉ tiêu CL tương đối, Giá trị chỉ tiêu CL tổng hợp.
  - Hàng 2: (1), (2), (3), (4)=(3)/(2), (5)=(1)*(4).
- **Mỗi item chiếm `max(1, số tên)` hàng.** Cột Hạng mục ghi mỗi tên một hàng; các cột còn lại chỉ ghi ở hàng đầu và **gộp dọc** (cột 0, 2–7).
- Số liệu:
  - (4) = `raw4` làm tròn 1 chữ số thập phân, số kiểu Việt.
  - (5) và (3) dùng `shipFixed2` (`#,##0.00`, HALF_UP) kèm "%" cho (5).
  - (1) và (2) dùng số kiểu Việt.
  - Dòng cuối "Chất lượng còn lại của tàu" (gộp cột 0–6) hiện tổng kèm "%".
- **Câu kết luận**: "Vậy chất lượng còn lại của {subject} là: {tổng}%." Với `subject` = các phần có dữ liệu nối bằng ", ": tên tàu, "số đăng ký: …", "năm đóng: …", "nơi đóng: …"; không có phần nào thì là "tài sản thẩm định".
- Không có item hợp lệ thì bỏ cả mục.

### Bảng so sánh tàu (`createShipGroupCompareTable`, ~3585)

- 5 cột: 18,5% và 4 cột 20,375% mỗi cột. Header: Nội dung / Tài sản thẩm định / TSSS 1 / TSSS 2 / TSSS 3. **Không có hàng ảnh.**
- Hàng nào cả 4 ô đều rỗng thì bỏ.
- Thứ tự hàng: Chủng loại, Nguồn tham khảo, Thông tin liên hệ, Năm đóng, Nơi đóng, Mục đích sử dụng, Cấp phương tiện, Chất lượng còn lại (%) (dạng "…%"), Trọng tải toàn phần (tấn), Kích thước (Lmax x Bmax), Công suất, Giá rao bán ({đồng|USD}), Giá giao dịch (…), Giá ước tính (…).
- Không có hàng "Tỷ lệ ước tính".

### Thông tin và ảnh TSSS (`createShipComparativeInfoAndImage`, ~1112-1215)

Với mỗi compare type 1–3 chưa xóa:
- Dòng in đậm: "-\tTài sản so sánh k: là {address}; trọng tải toàn phần X tấn, năm đóng Y, nơi đóng Z, máy chính công suất W HP. Giá giao dịch là N {đồng|USD}, chất lượng còn lại ước tính Q%." Phần nào rỗng hoặc bằng 0 thì bỏ.
- "Link tham khảo:" rồi mỗi link trong `source` (tách bằng `;`, bỏ phần rỗng) là một bullet.
- Dòng in đậm "Hình ảnh tài sản so sánh k" kèm bảng ảnh, nếu có ảnh.

### Bảng điều chỉnh

- `currencyUnit` dùng cho SHIP; `controlComparePriceUnit = currencyUnit` cho CAR và SHIP. Ô đơn vị của các dòng A, D1, E1, E4 dùng `currencyUnit`.
- **Còn sót ở L**: dòng A vẫn là "Giá thị trường (giá trước điều chỉnh)" và dòng B "Giá quy đổi về đơn vị so sánh chuẩn". Hệ thống mới sửa thành A "Giá bán" và bỏ dòng giá thị trường (xem Khác hệ thống cũ #4).
- "Nguyên tắc khống chế" (`buildRegulatoryConstraintsAvgPriceClause`): đơn vị theo tiền tệ.

### Bảng kết quả tàu (`createShipValuationSummaryTable`, ~5195-5300)

- Độ rộng cột: VND {8, 50, 42}%, ngoại tệ {8, 40, 26, 26}%.
- Header: STT / Hạng mục / Thành tiền ({đồng|USD}), ngoại tệ có thêm cột "Thành tiền (đồng)".
- Các dòng:
  - "Giá trị {tên}": `amount` (và `vndAmount`).
  - "Tổng giá trị": `amount` (và `vndAmount`).
  - "Làm tròn": VND là `roundedAmount`; ngoại tệ thì cột ngoại tệ là `amount`, cột VNĐ là `roundedAmount`.
- **Ghi chú chuyển đổi**: khi là ngoại tệ và `currencyNote` không rỗng, in dưới bảng. Mỗi dòng một đoạn, in nghiêng, giãn dòng 1,5, đoạn đầu cách trên 12pt, các đoạn sau cách trên 0, cách dưới 3pt, căn đều hai bên.
- Phần ghi chú `appraisalResult.note` theo sau.

### Bảng tổng hợp nhiều nhóm

- Nhánh SHIP lấy `ship.roundedAmount` (VNĐ).
- Tiêu đề cột "Thành tiền (đồng)", nhãn tổng "Tổng giá trị tài sản".

### Ảnh tàu (`createShipAssetImages`, ~2091-2143)

- Ảnh lấy từ `appraisalShipDescription.appraisalImages`; không có ảnh thì bỏ cả mục.
- Nhiều nhóm thì tiêu đề cấp 2 là địa chỉ hoặc tên tài sản đầu tiên.
- Thứ tự nhóm ảnh: Mặt trước tàu, Đầu tàu, Đuôi tàu, Mạn tàu, Sàn tàu, Số IMO/Số kiểm soát, Hệ thống neo, Phòng lái, phòng nghỉ, Khoang hàng, Hệ thống máy của động cơ, Khác.
- Dùng chung `renderGroupedImageTable` với ô tô: 2 ảnh mỗi hàng, hàng lẻ thì gộp 2 ô.

### Pháp lý tài sản

Dùng chung luồng: xuất `asset.legalDocuments` (văn bản tự sinh), mỗi dòng một bullet.
