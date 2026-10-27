# Migrate "Báo cáo VCB" (/vietcombank-report) và "Báo cáo VCBNEO" (/vietcombank-neo-report)

> **Subagent CHỈ dùng `sonnet`.** Mọi lần gọi tool `Agent` truyền `model: "sonnet"`.
> Làm thẳng trên `develop` của hai submodule (`git -C backend …`, `git -C frontend …`). Không push, không commit repo gốc. Trước khi bắt đầu kiểm tra cây làm việc sạch, không sạch thì dừng và báo.

## Context

Trong menu FE mới, "Báo cáo VCB" và "Báo cáo VCBNEO" (`frontend/src/components/layout/nav-items.ts:112-113`, `authorities: ['BGD_HN']`) vẫn là placeholder, chưa có `to`. Plan này port hai màn sang:
- **backend:** `/api/reports/vietcombank` và `/api/reports/vietcombank-neo`, mỗi cái có `/export`
- **frontend:** `/report/vietcombank` và `/report/vietcombank-neo` (hệ thống cũ đã có sẵn alias tên này)

Hai báo cáo đọc cùng một nguồn dữ liệu. Mỗi **báo cáo thẩm định** (hoặc mỗi yêu cầu chưa có báo cáo) là một bản ghi, bung ra nhiều dòng theo **nhóm tài sản**, rồi theo **dòng chi tiết** (đất ở / đất khác / công trình). Đây là gộp ô ba cấp.

**Điểm nặng nhất:** số tiền của nhóm tài sản lấy từ `AppraisalResultServiceImpl.getDetailByReportAndGroup`, một engine tính kết quả thẩm định mà backend mới **chưa có** (chưa có cả 5 entity liên quan). Plan port phần engine này thành một khối độc lập, các màn sau (BIDV, xuất Word, tab Kết quả) có thể dùng lại.

### Spec là hệ thống cũ (`td-project-legacy/src/main/java/com/jvb/thanhdo/…`)

| Nội dung | Vị trí |
|---|---|
| Controller | `controller/VietcombankReportController.java`, `VietcombankNeoReportController.java` |
| Service (gom nhóm, bung dòng, tính span) | `service/impl/VietcombankAppraisalExportServiceImpl.java`, `VietcombankNeoAppraisalExportServiceImpl.java` |
| SQL (truy vấn khóa trang, đếm, dòng thô, stream) | `repository/impl/VietcombankReportRepositoryImpl.java` |
| Form / DTO | `entity/dto/VcbReportSearchForm.java`, `VcbRawReportRowDto.java`, `VcbAppraisalExportRowDto.java`, `VcbNeoAppraisalExportRowDto.java` |
| Excel | `controller/component/VietcombankAppraisalExcelExporter.java` (35 cột, header 4 tầng), `VietcombankNeoAppraisalExcelExporter.java` (33 cột, header 2 tầng) |
| Giao diện | `resources/templates/report/vcb-report.html`, `vcb-neo-report.html` |
| Engine kết quả | `service/impl/AppraisalResultServiceImpl.java:83-133` (nạp), `135-487` (LAND), `489-592` (APARTMENT/CAR), `594-694` (làm tròn, parse) |
| Util engine | `common/LandPriceUtils.java`, `common/NumberFormatUtils.java:35-67`, `common/CommonUtils.java:801-823`, `enums/LandValuationMethod.java`, `CommonConstants.java:372-400, 524-529` |
| Converter JSON | `converter/AreaInsideDetailsConverter`, `AreaOutsideDetailsConverter`, `LandPriceConverter`, `LandDetailConverter` |

## Hành vi hệ thống cũ (tóm tắt)

**Lọc.** Luôn có `r.delete_flag = '0'`. Các điều kiện tùy chọn:

| Tham số | Điều kiện |
|---|---|
| `fromDate` / `toDate` ("Ngày phát hành") | `hs.max_sign_date` = `MAX(sign_date)` của lịch sử chứng thư chưa xóa |
| `reportFromDate` / `reportToDate` ("Ngày tạo báo cáo", chỉ màn Neo có ô) | `rep.created` |
| `keyword` | `LIKE` trên `r.contract_id / r.cust_name / r.company_name / ac.cust_name` |
| `source` | Có giá trị thì `= source`, rỗng thì `IN ('VCB','VCBNEO')`. Màn VCB có select 3 lựa chọn; màn Neo ép `VCBNEO` khi rỗng. |

**Trang.**
- Mỗi trang 20 **bản ghi** (khóa = `rep.id`, hoặc `REQ_<r.id>` khi chưa có báo cáo), không phải 20 dòng.
- Sắp `hs.max_sign_date DESC, r.create_date DESC, rep.id DESC`.
- STT tăng theo bản ghi.

**Dữ liệu và cách bung dòng.**
- **Truy vấn 1 (khóa trang):** lấy khóa có phân trang (`LIMIT/OFFSET`), cùng truy vấn `COUNT(DISTINCT …)`.
- **Truy vấn 2 (dòng thô):** dùng `BASE_SELECT/FROM/GROUP BY` (repository dòng 30-120), cho ra một dòng mỗi nhóm tài sản.
- **Bản ghi có báo cáo:** với mỗi nhóm tài sản, gọi `getDetail(reportId, groupId)`, rồi bung thành `max(1, đất ở, đất khác, công trình)` dòng. Công trình chỉ lấy loại có `valuationType` null hoặc `INCLUDE_IN_TOTAL`. Danh sách đất phụ thuộc `isMixed`, lấy từ `ag.land_valuation_method` của dòng thô.
- **Bản ghi chưa có báo cáo:** bung một dòng cho mỗi phần tử JSON `appraisal_certificate.legal_documents`. Không có phần tử nào thì ra 1 dòng, với tổng = `ac.rounding_money`.
- **Loại đất:** lấy từ `purpose` trong `land_details_json` (lọc trùng, nối `", "`). Nếu không có thì ghép loại đất ở và loại đất khác của dòng.
- **Neo khác VCB ở các điểm sau:**
  - Có cờ "Có CTXD".
  - Tổng diện tích đất = tổng diện tích các dòng đất; nếu bằng 0 thì lấy `private_area_by_gcn`.
  - Giá trị đất quy hoạch = tổng `amount` của `landInsideDetails`.
  - Giá trị CTXD = `rounded ?? constructionAmount`.
  - Tổng tài sản = `roundingMoney ?? totalMoney`.
  - Đơn giá đổi sang triệu đồng (chia 1e6, 4 chữ số, bỏ số 0 thừa).
  - Mặt tiền = `agc.main_frontage_width`; độ rộng đường = `agc.road_width`.
  - Nhiều cột luôn để trống: Số phiếu yêu cầu, Mặt đường, Mặt ngõ, Khoảng cách, Thanh khoản, Quy hoạch, Cảnh báo, Mức độ phù hợp, 3 cột phí, 2 cột Từ chối.

**Gộp ô** (bảng đầy đủ ở phần khảo sát; exporter cũ là nguồn chuẩn):

| Báo cáo | Cấp gộp | Cột |
|---|---|---|
| VCB | Bản ghi | STT, Số HĐ, Số chứng thư, Ngày ban hành, KH, Chi nhánh, **Tỉnh/TP**, Phí TĐG, TĐV (họ tên, số thẻ), NĐDPL (họ tên, số thẻ) |
| VCB | Nhóm | Địa chỉ, Quận, Phường, Đường (trống), Vị trí, Phương pháp, Mô tả, Loại đất, Giá trị QSD đất, Tổng giá trị, Ghi chú |
| VCB | Chi tiết | 6 ô đất ở/đất khác, 6 ô CTXD |
| Neo | Bản ghi | STT, Số chứng thư, Số phiếu, KH, Chi nhánh, 3 cột phí, 2 cột Từ chối |
| Neo | Nhóm | Địa chỉ, Quận, Tỉnh, 4 cột Vị trí, Có CTXD, Tổng DT, Mặt tiền, Giá trị đất QH, Giá trị CTXD, Tổng giá trị, 4 cột trống |
| Neo | Chi tiết | Loại đất ở/khác, DT và đơn giá đất ở/khác |

**Quyền.**
- Menu: `hasAnyAuthority('BGD_HN')`.
- Trang và export không có rule riêng, ai đã đăng nhập gõ URL cũng vào được. Không lọc phạm vi theo người xem.

**Excel.**
- Tên file: `Bao-cao-Vietcombank.xlsx` / `Bao-cao-VietcombankNeo.xlsx`.
- Tiêu đề merge ở hàng 0, header từ hàng 2.
- Dữ liệu bắt đầu ở hàng 6 (VCB) hoặc hàng 4 (Neo), cột bắt đầu từ A.
- Cửa sổ SXSSF 500.

## Quyết định (đã chốt với người dùng)

1. **Quyền: chỉ `BGD_HN`** cho cả trang, API và export, khớp menu.
   - Thêm hằng public `BranchAuthorities.BGD_HN`. `service/Authorities.java:24` dùng lại hằng này thay literal.
   - FE: route `authorities: ['BGD_HN']`.
   - Đây là điểm khác hệ thống cũ (vá lỗ hổng gõ URL). Ghi vào Javadoc controller.
2. **Sửa hai lỗi cũ:**
   - Excel áp dụng `keyword` giống trang.
   - Bản ghi chưa có báo cáo: mỗi phần tử pháp lý là **một nhóm riêng có 1 dòng**. Như vậy ô địa chỉ (cấp nhóm) không bị gộp, mỗi dòng hiện địa chỉ của mình.
3. **Engine kết quả giữ y nguyên số học cũ:**
   - `parseEuropeanArea` trả `double`.
   - `roundLandAmountVnd` = `BigDecimal.valueOf(double)` nhân giá, rồi `HALF_UP`.
   - Làm tròn tổng: `Math.round((double)x/base)*base`.
   - Căn hộ dùng `(long)` cắt phần thập phân.
   - Không lọc cờ xóa của `control_compare_factor_synthetic` và `appraisal_result`.
   - Định dạng số dùng `Locale.ROOT` cố định (cũ dùng `Locale.getDefault()`).
4. **Các khác biệt nhỏ khác** (ghi vào Javadoc):
   - Thêm `r.id DESC` vào cuối sắp xếp để thứ tự ổn định.
   - Ngày lấy trọn ngày `[from, toExclusive)`.
   - Ngày sai định dạng, `from > to`, `source` lạ, `page < 0` → 400.
   - Tên file encode UTF-8, giữ tên cũ.
   - Màn Neo hiển thị số theo vi-VN (`1.234,50`). Cũ dùng `formatDecimal(…,'POINT',2,'POINT')`, ra kết quả nhập nhằng.
   - Chỉ ô "Giá trị CTXD khác" trên web hiện `—`, giống bản cũ.

## Hợp đồng API (BE và FE làm song song)

Query chung `VcbReportQuery`: `keyword` (≤ 500 ký tự), `source` (`VCB` | `VCBNEO` | rỗng), `fromDate`, `toDate`, `reportFromDate`, `reportToDate` (ISO `yyyy-MM-dd`), `page` (từ 0), `size` (mặc định 20, tối đa 100). Endpoint Neo coi `source` rỗng là `VCBNEO`.

**`GET /api/reports/vietcombank` → `Page<VcbReportItem>`**

Không có cờ rowspan: client tự tính `rowspan` từ số dòng, như `BankReportItem`.

```
VcbReportItem { stt, contractId, certificateNo, issueDate (chuỗi "dd/MM/yyyy, …"), custName, linkedOffice,
  provinceName, valuationFee (String), tdvName, tdvCardNo, nddName, nddCardNo,
  groups: [ { appraisalAddress, districtName, wardName, landPosition, valuationMethod, positionDescription,
              landType, totalLandAmount (Long), totalMoney (Long), note,
              lines: [ { residentialLandArea (String), residentialLandUnitPrice, residentialLandAmount,
                         otherLandArea (String), otherLandUnitPrice, otherLandAmount (Long),
                         constructionType, constructionArea (BigDecimal), constructionUnitPrice (BigDecimal),
                         qualityRemain (BigDecimal), constructionAmount (Long) } ] } ] }
```

**`GET /api/reports/vietcombank-neo` → `Page<VcbNeoReportItem>`**

```
VcbNeoReportItem { stt, certificateNo, custName, linkedOffice,
  groups: [ { appraisalAddress, districtName, provinceName, roadWidth (String), hasConstruction ("Có"/"Không"),
              totalLandArea (String), frontageWidth (String), planningLandValue, constructionValue, totalAssetValue (Long),
              lines: [ { residentialLandType, otherLandType, residentialLandArea (String), residentialLandUnitPrice (BigDecimal trđ),
                         otherLandArea (String), otherLandUnitPrice (BigDecimal trđ) } ] } ] }
```

Các cột luôn trống không gửi qua API; FE và Excel tự vẽ ô trống.

**Export:**
- `GET /api/reports/vietcombank/export` và `GET /api/reports/vietcombank-neo/export`: cùng tham số, không có `page`/`size`.
- Trả xlsx, header `filename*=UTF-8''Bao-cao-Vietcombank.xlsx` / `…VietcombankNeo.xlsx`.

Mọi đường dẫn đều nhỏ hơn 1 nhóm và ít nhất 1 dòng: `groups` có ít nhất 1 phần tử, mỗi `lines` có ít nhất 1 phần tử. Phía server bảo đảm điều này.

---

## Cấu trúc file

**Backend** (`backend/src/main/java/com/thanhdo/tdprojectbe/`)

| Khối | File |
|---|---|
| Engine: entity chỉ đọc, chỉ map cột cần | `model/entity/AppraisalAssetGroup`, `AppraisalGroupCompare` (~9 cột), `AppraisalConstructionDescription` (kèm `getValuationUsageArea()` y cũ: `valuation_area_basis` null/ACTUAL → `actual_total_usage_area`, ngược lại `total_usage_area`, null → 0), `ControlCompareFactorSynthetic` (`appraisal_asset_group_id`, `avg_price_guide`), `AppraisalResult` (`rounding_money_type`, `note`). Sửa `AppraisalLandDescription` để thêm `land_price_json`. |
| Engine: converter, DTO, hằng | `model/converter/AreaInsideDetailsConverter`, `AreaOutsideDetailsConverter`, `LandPriceConverter` (Jackson 3, lỗi/null → rỗng); DTO record `AreaInsideDetail`, `AreaOutsideDetail`, `LandPrice`; `common/constant/LandValuationMethod` (enum), `AppraisalAssetType`, `RoundingMoneyType`, `ConstructionValuationType` |
| Engine: logic | `common/util/EuropeanNumbers` (`parseArea` → double, `parseDecimal`, `formatFixed2`, `formatVietnamese`), `service/result/LandPrices` (port `LandPriceUtils.calculate`), `service/result/EvaluationResultCalculator` (thuần, nhận entity đã nạp, port `buildDetailFromEntities` 3 nhánh, chỉ các field được dùng), `service/result/EvaluationResultLoader` (nạp entity theo `(reportId, groupId)` như dòng 89-107), record `EvaluationResultDetail` |
| Engine: repository | `AppraisalAssetGroupRepository`, `AppraisalGroupCompareRepository`, `AppraisalConstructionDescriptionRepository`, `ControlCompareFactorSyntheticRepository`, `AppraisalResultRepository` (derived query như cũ) |
| Báo cáo VCB: truy vấn | `repository/VcbReportRepository` (EntityManager native, một hàm `where(...)` dùng chung cho khóa / đếm / stream, như `ValuationReportRepository`); record `VcbRawRow`, `VcbReportKey`; `model/dto/VcbReportQuery`, `VcbReportFilter` (chuẩn hóa: `LikePattern.contains`, `ReportPeriod` hai khoảng, `source`) |
| Báo cáo VCB: dựng bản ghi | `service/VcbReportAssembler` dùng chung cho cả hai báo cáo: gom dòng thô theo khóa → `VcbRecord(raw đầu, List<VcbGroup(raw, detail, landItems, constructions)>)`; port `extractLandDetailItems`, `isAreaPositive`, loại đất theo `purpose` (đọc `land_details_json` bằng Jackson tree, chỉ duyệt `plots[].mapSheets[].landUseRows[].purpose`), bung bản ghi chưa có báo cáo qua `CertificateLegalDocumentListConverter` / record `CertificateLegalDocument` đã có |
| Báo cáo VCB: ánh xạ ra item | `model/dto/VcbReportItem` (+ `Group`, `Line`), `VcbNeoReportItem` (+ `Group`, `Line`); `VcbReportItem.from(VcbRecord, stt)` và `VcbNeoReportItem.from(...)` chứa các quy tắc riêng từng báo cáo |
| Excel | `common/util/export/excel/VcbReportExcelExporter`, `VcbNeoReportExcelExporter` (lớp con `StreamingExportCommon`, cửa sổ 500, header nhiều tầng merge ngang/dọc như cũ, gộp ô dọc theo mẫu `BankReportExcelExporter.writeCertificate`) |
| Service, controller | `service/VcbReportService`, `VcbNeoReportService` (hoặc một service có 2 cặp hàm, tùy khối lượng code: file > 200 dòng thì tách); `controller/VcbReportController`, `VcbNeoReportController` |
| Cấu hình | `config/SecurityConfig` (matcher `/api/reports/vietcombank/**`, `/api/reports/vietcombank-neo/**` → `hasAuthority(BranchAuthorities.BGD_HN)`), `common/constant/BranchAuthorities`, `service/Authorities`, `backend/CLAUDE.md` (thêm dòng `EvaluationResultLoader`/`Calculator` và `EuropeanNumbers` vào bảng helper) |

**Frontend** (`frontend/src/`)
- `lib/vcb-report.ts`:
  - `VCB_REPORT_AUTHORITIES = ['BGD_HN']`, `PAGE_SIZE = 20`, `MAX_KEYWORD_LENGTH = 500`, `SOURCE_OPTIONS` (nhãn như cũ)
  - `flattenRecord(item)`: trả các dòng có `reportRowSpan` / `groupRowSpan` / cờ đầu nhóm, dùng chung cho hai view
  - định dạng số
- `api/vcb-reports.ts`: `listVcbReport`, `exportVcbReport`, `listVcbNeoReport`, `exportVcbNeoReport`, kèm types.
- `views/report/VcbReportView.vue`, `VcbNeoReportView.vue`: khuôn `BankReportView.vue` (`ScrollableTable` + `widths`, URL là bộ lọc đã áp dụng, `DataPagination`, `AbortController`), header nhiều tầng bằng `TableHead` `colspan` / `rowspan`.
- Sửa `router/index.ts` (2 route), `nav-items.ts:112-113` (thêm `to`), `guard.spec.ts`, `layout.spec.ts`.

---

### Task 1: Engine, phần nền (backend)
- Hằng/enum, `EuropeanNumbers`, 3 converter + DTO, 5 entity + repository, thêm cột `land_price_json`.
- [ ] Test trước:
  - `EuropeanNumbersTest`: `"1.234,5"` → 1234.5, `"1,234.5"` → 1234.5, `"abc"` → 0, `formatFixed2(0)` → `""`
  - test 3 converter: JSON hỏng hoặc null → rỗng
  - `EvaluationEntitiesRepositoryTest` (`@MysqlDataJpaTest`): các derived query lọc `delete_flag = 0` đúng như cũ; `getValuationUsageArea` cả 3 trường hợp
- [ ] FAIL → cài đặt → PASS → commit `feat(result): entity và tiện ích tính kết quả thẩm định`.

### Task 2: Engine, phần tính toán (backend)
- `LandPrices` (port `LandPriceUtils` đầy đủ: thuế 0,5 %, chi phí 1 %, `roundVnd`).
- `EvaluationResultCalculator`:
  - LAND, các nhánh: mặc định, AGRICULTURAL_STATE_PRICE, LAND_USE_CONVERSION, STATE_PRICE, MIXED
  - đất trong quy hoạch
  - công trình
  - tổng làm tròn, gồm `resolveRoundingMoneyType`: dùng `AppraisalResult` chỉ khi `reportId` khớp, giá trị lạ thì về MILLION
  - APARTMENT, CAR
  - `note`
- `EvaluationResultLoader`: nhóm không tồn tại, hoặc thuộc báo cáo khác → detail rỗng.
- [ ] Test `EvaluationResultCalculatorTest`, dữ liệu tay, so với con số tính bằng công thức cũ:
  - mỗi nhánh đất 1 ca
  - `INCLUDE_IN_TOTAL` / `REFERENCE` / `NOT_INCLUDED`
  - làm tròn triệu/nghìn và HALF_UP biên `x.5`
  - căn hộ cắt phần thập phân (`(long)`), cả hai `basis`
  - xe không có synthetic → 0
  - diện tích `"1.234,56"`
- [ ] `LandPricesTest`, `EvaluationResultLoaderTest` (Mockito).
- [ ] FAIL → cài đặt → PASS → commit `feat(result): tính kết quả thẩm định theo nhóm tài sản`.

### Task 3: Truy vấn báo cáo VCB (backend)
- `VcbReportQuery`:
  - `@AssertTrue` cho hai khoảng ngày, message "Từ ngày không được sau đến ngày"
  - `@Pattern` `source` ∈ {VCB, VCBNEO} hoặc rỗng
  - `@Size(max=500)` keyword
- `VcbReportFilter.of(query, defaultSource)`.
- `VcbReportRepository`:
  - `keys(filter, pageable)`, `count(filter)`, `rows(reportIds, requestIdsNoReport)`
  - `stream(filter)`: hint fetchSize `Integer.MIN_VALUE`
  - port nguyên SQL cũ (dòng 30-259), giữ hàm `formatCertificateNo` (JSON mảng → nối `", "`, bỏ trùng)
  - kiểm tra collation khi JOIN `employees`
- [ ] Test `VcbReportRepositoryTest` (`@MysqlDataJpaTest`, dữ liệu MARK):
  - lọc từng tham số
  - mặc định `IN ('VCB','VCBNEO')` (nguồn khác bị loại)
  - yêu cầu đã xóa bị loại
  - 2 báo cáo cùng 1 yêu cầu → 2 khóa
  - yêu cầu chưa có báo cáo → khóa `REQ_`
  - đếm khớp số khóa
  - sắp xếp đúng thứ tự, có tie-break `r.id`
  - stream ra các dòng của cùng bản ghi liền nhau
  - chứng thư nhiều số / nhiều ngày ký nối đúng
- [ ] commit `feat(report): truy vấn báo cáo VCB`.

### Task 4: Dựng bản ghi, item (backend)
- `VcbReportAssembler` + `VcbReportItem.from` + `VcbNeoReportItem.from`.
- [ ] Test (Mockito cho loader):
  - nhóm không có detail → 1 dòng
  - bung `max(đất ở, đất khác, CTXD)` dòng, giá trị chỉ ở dòng đầu nhóm
  - MIXED và không MIXED lấy đúng nguồn đất; "Đất ngoài quy hoạch"
  - diện tích ≤ 0 bị bỏ
  - loại đất từ `purpose` lọc trùng, fallback ghép đất ở/khác
  - bản ghi chưa có báo cáo: N phần tử pháp lý → N nhóm 1 dòng, địa chỉ riêng; 0 phần tử → tổng = `rounding_money`
  - Neo: Có CTXD, tổng DT (fallback GCN), giá trị QH, CTXD, tổng tài sản, đơn giá trđ (`1500000` → `1.5`), mặt tiền bỏ số 0 thừa
- [ ] commit `feat(report): dựng dữ liệu báo cáo VCB và VCBNEO`.

### Task 5: Excel (backend)
- Hai exporter, bố cục y cũ: tiêu đề, nhãn header, merge header, độ rộng, kiểu ô, cột gộp theo cấp.
- VCB: `constructionAmount` chỉ ghi khi > 0; phí `parseLongQuietly`.
- Neo: đơn giá `#,##0.00`, tiền `#,##0`.
- [ ] Test:
  - header (đọc nhãn và vùng merge)
  - một bản ghi 2 nhóm × 3 dòng: vùng merge đúng cấp
  - ô số và định dạng
  - nhiều bản ghi hơn cửa sổ streaming
- [ ] commit `feat(report): file Excel báo cáo VCB và VCBNEO`.

### Task 6: Service, controller, quyền (backend)
- Service:
  - `search`: khóa trang → dòng thô → assembler → item, STT = `page*size+1…`
  - `export`: stream và gom dòng liền nhau theo khóa như cũ, trong `@Transactional(readOnly = true)`
- Controller theo mẫu `BankReportController`. Javadoc liệt kê "Khác hệ thống cũ" (Quyết định 1, 2, 4).
- `SecurityConfig` + `BranchAuthorities.BGD_HN` + `Authorities` dùng hằng.
- [ ] Test `@WebMvcTest`:
  - chưa đăng nhập → 401 (4 path)
  - `BGD_HN` → 200
  - `BGD` (không HN), `BRANCH_HN`, `admin`, `NVKT` → 403
  - bind đủ 6 tham số
  - Neo `source` rỗng → `VCBNEO`
  - 400 cho ngày sai, `from > to` (cả hai khoảng), `source=ABC`, `size=101`, `page=-1`
  - tên file export
  - chứng minh test quyền FAIL khi gỡ rule
- [ ] Service test: STT theo trang, trang rỗng không gọi `rows`, export áp dụng `keyword`.
- [ ] `AuthServiceTest` vẫn pass.
- [ ] `./mvnw test` toàn bộ → BUILD SUCCESS. Cập nhật `backend/CLAUDE.md`.
- [ ] commit `feat(report): API báo cáo VCB và VCBNEO`.

### Task 7: Frontend lib, api
- [ ] Spec `flattenRecord`:
  - 2 nhóm × (1, 3) dòng → 4 dòng; `reportRowSpan = 4` ở dòng đầu; `groupRowSpan` đúng; cờ đầu nhóm đúng
- [ ] Spec định dạng: tiền `1.234.567`, trđ `1,5`, null → rỗng.
- [ ] Spec API: bỏ tham số rỗng, map `from` / `to` / `reportFrom` / `reportTo`.
- [ ] commit `feat(report): api báo cáo VCB`.

### Task 8: Frontend màn hình, route, menu
- VCB:
  - Lọc: ô từ khóa (placeholder "Số hợp đồng, Tên khách hàng"), select nguồn (3 lựa chọn, nhãn như cũ), `DateRangePicker` "Ngày phát hành"
  - Header 4 tầng, 35 cột
- Neo:
  - Lọc: ô từ khóa, `DateRangePicker` "Ngày phát hành" và "Ngày tạo báo cáo", không có select nguồn
  - Header 2 tầng, 33 cột
- Chung:
  - Ô cấp bản ghi / nhóm render với `rowspan` ở dòng đầu
  - Số căn phải `tabular-nums`
  - Trạng thái rỗng "Không có dữ liệu"
  - Export dùng bộ lọc đã áp dụng
- Route `report/vietcombank` (`report-vietcombank`), `report/vietcombank-neo` (`report-vietcombank-neo`), breadcrumbs `Báo cáo › Báo cáo VCB` / `Báo cáo VCBNEO`, `authorities: ['BGD_HN']`, comment trỏ về controller cũ. Thêm `to` vào menu.
- [ ] Spec mỗi view:
  - số cột header lá và thứ tự nhãn
  - rowspan đúng với dữ liệu 2 nhóm
  - Tìm kiếm → API và URL nhận đúng tham số
  - export dùng bộ lọc đã áp dụng
  - định dạng số
  - trạng thái rỗng / lỗi
- [ ] `guard.spec`: `BGD_HN` vào được; `BGD` không HN và `admin` bị chặn.
- [ ] `layout.spec`: link hiện; bỏ 2 href khỏi selector placeholder.
- [ ] Chạy lần lượt (máy yếu): `npm run lint`, `npm run type-check`, `npm run test:unit -- --maxWorkers=2`. Test `AppraisalRequestListView.spec.ts` đã fail sẵn.
- [ ] commit `feat(report): màn báo cáo VCB và VCBNEO`.

### Task 9: Kiểm tra cuối
- [ ] Chạy lại toàn bộ test BE và FE, ghi số thật.
- [ ] Nếu có môi trường: đăng nhập BGD HN, chọn cùng bộ lọc trên màn cũ và màn mới.
  - So từng cột của ≥ 3 bản ghi: một nhóm đất MIXED, một nhóm không MIXED có CTXD, một yêu cầu chưa có báo cáo.
  - Đặc biệt so "Giá trị QSD đất", "Tổng giá trị" vì đây là số do engine tính.
  - Mở cả hai file Excel.
- [ ] Code review bằng subagent `model: "sonnet"`, tập trung vào engine (so với `AppraisalResultServiceImpl`).
- [ ] Báo người dùng: số test, phần chưa kiểm, khác biệt cố ý.

## Lưu ý kỹ thuật

- **Máy test thiếu RAM:**
  - `MAVEN_OPTS="-Xms32m -Xmx256m -XX:+UseSerialGC -XX:TieredStopAtLevel=1"`
  - `-DargLine="-Xms32m -Xmx768m -XX:+UseSerialGC -XX:TieredStopAtLevel=1 -Djdk.attach.allowAttachSelf=true -XX:+EnableDynamicAgentLoading"`
- **N+1:** engine nạp 4 truy vấn cho mỗi nhóm tài sản, như cũ. Trang có 20 bản ghi nên chấp nhận được. Export toàn bộ có thể chậm, nhưng giữ như cũ (YAGNI): ghi trong Javadoc, không tối ưu trước.
- **Jackson:** DTO mới dùng `com.fasterxml.jackson.annotation` như `CertificateLegalDocument`; converter dùng `tools.jackson`.
- **Gộp ô Excel:** một bản ghi phải ít hơn cửa sổ 500 dòng, như cũ.
- **Sidebar FE:** `AppSidebar.vue` chưa lọc theo `authorities`, nên mục menu vẫn hiện với mọi người như các mục khác. Không sửa trong plan này, route guard đã chặn.
- Theo `backend/CLAUDE.md`, `.claude/rules/jpa-spring.md`, `.claude/rules/design-principles.md`, `frontend/CLAUDE.md`, `frontend/DESIGN_SYSTEM.md`. Comment, Javadoc, nhãn đều bằng tiếng Việt có dấu.

## Thực thi

- **Bước đầu:** chép plan vào `docs/superpowers/plans/2026-10-26-migrate-vcb-report.md` (không commit).
- **Agent backend (Sonnet):** làm Task 1 → 6 tuần tự.
- **Agent frontend (Sonnet):** làm Task 7 → 8 song song với backend, theo mục "Hợp đồng API".
- **Task 9:** chạy sau cùng.
