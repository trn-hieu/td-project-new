# Migrate "Báo cáo TSSS" (/report/tsss)

> **Subagent CHỈ dùng `sonnet`.** Mọi lần gọi tool `Agent` truyền `model: "sonnet"`.
> Làm thẳng trên `develop` của hai submodule (`git -C backend …`, `git -C frontend …`). Không push, không commit repo gốc. Trước khi bắt đầu, kiểm tra cây làm việc sạch; nếu không sạch thì dừng và báo.

## Context

Trong menu FE mới, mục "Báo cáo TSSS" (`frontend/src/components/layout/nav-items.ts:114`, `authorities: ['admin']`) vẫn là placeholder, chưa có `to`. Plan này port màn "Báo cáo Giao dịch Tài sản So sánh" sang hệ thống mới:
- **Backend:** thêm `/api/reports/tsss` và các endpoint con `/filter-options`, `/export`.
- **Frontend:** thêm route `/report/tsss`.

Màn này đơn giản hơn VCB nhiều: không cần engine tính kết quả. Mỗi **tài sản so sánh đã giao dịch** (một dòng `appraisal_group_compares`) là một bản ghi. Bản ghi bung ra nhiều dòng theo danh sách công trình (JSON `structure_details`), nên chỉ có **gộp ô một cấp**.

### Spec là hệ thống cũ (`td-project-legacy/src/main/java/com/jvb/thanhdo/…`)

| Nội dung | Vị trí |
|---|---|
| Controller (`@PreAuthorize("hasAuthority('admin')")` cho trang và export) | `controller/TsssReportController.java` |
| Service (bung dòng theo công trình, tách liên hệ, tọa độ, loại đất) | `service/impl/TsssReportServiceImpl.java` |
| SQL (trang, đếm, stream, `formatCertificateNo`) | `repository/impl/TsssReportRepositoryImpl.java` |
| Form / DTO | `entity/dto/TsssReportSearchForm.java`, `TsssRawReportRowDto.java`, `TsssAppraisalExportRowDto.java`, `StructureDetail.java` |
| Excel (37 cột, sheet "TSSS") | `controller/component/TsssReportExcelExporter.java` |
| Giao diện | `resources/templates/report/tsss-report.html` |

## Hành vi hệ thống cũ (tóm tắt)

**Nguồn dữ liệu.**
- `appraisal_group_compares agc` nối INNER với `appraisal_reports rep` và `p_appraisal_request r`, cả hai bảng đều chỉ lấy dòng chưa xóa (cờ `0` hoặc NULL).
- Nối LEFT với `appraisal_certificate ac` (chỉ lấy dòng chưa xóa).
- Nối LEFT với bảng con `hs` gom lịch sử chứng thư chưa xóa:
  - `merged_certificate_no` = `GROUP_CONCAT(DISTINCT $[0])`
  - `merged_sign_date` = `GROUP_CONCAT(DISTINCT dd/MM/yyyy ORDER BY sign_date)`
- Luôn lọc: `agc` chưa xóa, `agc.type IN (1,2,3)`, `agc.transaction_status = 'Đã giao dịch'`.
- Số chứng thư lấy `COALESCE(hs.merged_certificate_no, ac.certificate_no dạng mảng JSON)`, sau đó qua `formatCertificateNo`: JSON mảng → bỏ rỗng, trim, bỏ trùng, nối `", "`.

**Bộ lọc** (tất cả đều tùy chọn):

| Tham số | Điều kiện |
|---|---|
| `source` | `r.source = ?` |
| `keyword` | `LIKE %kw%` trên `hs.merged_certificate_no`, `ac.certificate_no`, `r.linked_office`, `agc.contact`, `agc.address` |
| `fromDate` / `toDate` ("Ngày ban hành") | `EXISTS` một lần ký trong lịch sử chứng thư chưa xóa có `sign_date ∈ [from, to+1)` |

**Trang.**
- Mỗi trang 20 bản ghi `agc`, sắp `agc.id DESC`. STT tăng theo bản ghi.
- Đếm bằng `COUNT(*)` với cùng các JOIN.
- Ô chọn nguồn lấy từ danh sách source bank, nhãn `bankName - bankCode`, placeholder "Nguồn".

**Bung dòng.** Số dòng của một bản ghi = số phần tử `structure_details` (tối thiểu 1). Dòng i lấy công trình thứ i: `buildingType`, `area`, `clclPercent` (chuỗi), `unitPrice` (Long).

**Các field cấp bản ghi.**
- **Liên hệ:** `contact.split(":")`. Có ít nhất 2 phần thì tên = phần 0, SĐT = phần 1 (đều trim); ngược lại tên = cả chuỗi, SĐT rỗng.
- **Tọa độ:** `latitude_dms` nếu có, không thì `latitude`. Tương tự cho kinh độ. Nối bằng `"; "`.
- **Loại tài sản:** luôn là `"QSDĐ + CTXD"`.
- **Loại đất:** `landType` của `area_outside_details` rồi `area_inside_details`, trim, bỏ trùng giữ thứ tự, nối `", "`. Rỗng thì `"N/A"`.
- Các cột lấy thẳng từ DB:
  - `total_area`, `main_frontage_width`: `CAST AS CHAR`, hiển thị chuỗi thô
  - `road_width`, `transaction_listing_time`, `legal`, `address`
  - `estimate_price`, `land_use_right_price` (Long, có thể null)

**Bảng / Excel: 37 cột.**
- Cột theo dòng (không gộp): 26 "Tài sản gắn liền với đất", 27 "Số tầng" (trống), 28 "Diện tích sàn", 29 "Tỷ lệ CLCL", 32 "Đơn giá xây dựng".
- Mọi cột còn lại gộp theo bản ghi.
- Các cột luôn trống: 10–15 (Tỉnh/TP, Quận/Huyện, Phường/Xã, Đường/phố, Khu đô thị, Vị trí thửa đất), 20 (Hình dạng), 23–25 (Khoảng cách, Lợi thế, Hạn chế), 27, 31 (Chi phí khác), 34–36.
- Excel:
  - Hàng 0 là tiêu đề "BÁO CÁO GIAO DỊCH TÀI SẢN SO SÁNH", không merge.
  - Hàng 1 là header (nhãn Excel dài hơn nhãn web ở vài cột).
  - Dữ liệu bắt đầu từ hàng 2.
  - Độ rộng cột: 7 (STT), 40 (Địa chỉ), 35 (Khu đô thị), còn lại 20.
  - Tiền định dạng `#,##0`.
  - Cửa sổ SXSSF 500.
  - Excel bỏ giá trị `"N/A"` ở 3 cột công trình; web thì không bỏ.
  - Tên file `Bao-cao-TSSS.xlsx`.
- Web: số tiền hiển thị kiểu `1,234,567`, trạng thái rỗng là "Không có dữ liệu.".

**Quyền.** Chỉ `admin` cho menu, trang và export. Không lọc phạm vi theo người xem.

## Quyết định

1. **Giữ nguyên quyền:** chỉ `hasAuthority("admin")` cho `/api/reports/tsss/**`; FE route `authorities: ['admin']`.
2. **Giữ y nguyên SQL, bộ lọc và cách bung dòng.** Không sửa lỗi nhân dòng khi một yêu cầu có nhiều chứng thư chưa xóa. Ghi điều này vào Javadoc.
3. **Khác biệt nhỏ** (ghi vào Javadoc controller):
   - `keyword` escape `%` `_` `\` bằng `LikePattern.contains`, tối đa 500 ký tự.
   - Ngày sai định dạng, `from > to`, `page < 0`, `size > 100` trả 400.
   - Web bỏ `"N/A"` ở 3 cột công trình giống Excel. Quy tắc đặt một chỗ ở backend nên web và Excel thống nhất.
   - Web hiển thị tiền theo vi-VN (`1.234.567`) như các báo cáo mới khác.
   - Tiêu đề Excel merge ngang qua 37 cột (bản cũ ghi chú định merge nhưng chưa làm).
   - Tên file encode UTF-8 (`filename*`), vẫn giữ tên `Bao-cao-TSSS.xlsx`.

## Hợp đồng API (BE và FE làm song song)

Query `TsssReportQuery`: `keyword` (≤ 500), `source`, `fromDate`, `toDate` (ISO `yyyy-MM-dd`), `page` (từ 0), `size` (mặc định 20, tối đa 100).

**`GET /api/reports/tsss` → `Page<TsssReportItem>`**
```
TsssReportItem { stt, contractId, certificateNo, issueDate, linkedOffice, transactionListingTime, legal,
  contactName, contactPhone, address, coordinates, assetType, landType, totalArea, mainFrontageWidth, roadWidth,
  estimatePrice (Long|null), landUseRightPrice (Long|null),
  structures: [ { buildingType, area, clclPercent (String, "" thay null/"N/A"), unitPrice (Long|null) } ] }
```
- `structures` luôn có ít nhất 1 phần tử; khi không có công trình, server thêm một dòng rỗng.
- Client tự tính `rowspan = structures.length`.
- Các cột luôn trống không gửi qua API.

**`GET /api/reports/tsss/filter-options` → `{ sources: TextOption[] }`**: lấy theo mẫu `CashCollectorReportService.filterOptions` (`findByDeleteFlagOrderByBankCode`, nhãn `bankName - bankCode`).

**`GET /api/reports/tsss/export`**: cùng tham số, không có `page`/`size`. Trả xlsx, `Content-Disposition` có `filename*=UTF-8''Bao-cao-TSSS.xlsx`.

---

## Cấu trúc file

**Backend** (`backend/src/main/java/com/thanhdo/tdprojectbe/`)

| Khối | File |
|---|---|
| JSON công trình | `model/dto/StructureDetail` (record: `buildingType`, `area`, `unitPrice` Long, `clclPercent`; annotation `com.fasterxml.jackson.annotation` như `AreaInsideDetail`), `model/converter/StructureDetailsConverter` theo mẫu `AreaInsideDetailsConverter` (Jackson 3; lỗi hoặc null → rỗng) |
| Truy vấn | `repository/TsssReportRepository`: Spring Data, native `@Query` + `countQuery` + interface projection `TsssRawRow`. Lý do native: `JSON_EXTRACT`, `GROUP_CONCAT`, bảng con. Theo mẫu `VcbReportRepository` (cờ "đã lọc" bằng SpEL, `Stream<TsssRawRow> stream(...)` với hint `org.hibernate.fetchSize = Integer.MIN_VALUE`). Port nguyên SQL cũ dòng 28-78 và 119-166, tham số đặt tên. `model/dto/TsssReportQuery` (`@AssertTrue` from ≤ to, `@Size(max=500)`), `TsssReportFilter.of(query)` (`LikePattern.contains`, `ReportPeriod` hoặc `[from, to+1)`) |
| Ánh xạ | `model/dto/TsssReportItem` (+ `Structure`), `TsssReportItem.from(TsssRawRow, stt, converters)`; port `formatCertificateNo`, tách liên hệ, tọa độ, loại đất (dùng `AreaInsideDetailsConverter` / `AreaOutsideDetailsConverter` đã có), quy tắc `N/A` → `""` |
| Excel | `common/util/export/excel/TsssReportExcelExporter`: lớp con `StreamingExportCommon`, cửa sổ 500, gộp ô dọc theo mẫu `BankReportExcelExporter`, header một tầng với nhãn Excel y cũ |
| Service, controller | `service/TsssReportService` (`search`, `filterOptions`, `export` trong `@Transactional(readOnly = true)` + try-with-resources), `controller/TsssReportController` theo mẫu `CashCollectorReportController` |
| Cấu hình | `config/SecurityConfig`: matcher `/api/reports/tsss/**` → `hasAuthority("admin")`, đặt cạnh các rule `/api/reports/**` hiện có. `backend/CLAUDE.md`: thêm `TsssReportExcelExporter` vào danh sách mẫu streaming nếu hợp lý |

**Frontend** (`frontend/src/`)
- `api/tsss-reports.ts`: `listTsssReport`, `exportTsssReport`, `getTsssFilterOptions` và các type; bỏ tham số rỗng.
- `views/report/TsssReportView.vue`, theo khuôn `CashCollectorReportView.vue` / `BankReportView.vue`:
  - Bộ lọc: ô từ khóa (placeholder "Tìm kiếm: Mã chứng thư, địa chỉ, chi nhánh, liên hệ..."), `SelectBox` nguồn (placeholder "Nguồn"), `DateRangePicker` "Ngày ban hành", nút "Tìm" / "Xuất excel".
  - `ScrollableTable` 37 cột, nhãn web y cũ, ô cấp bản ghi dùng `rowspan` ở dòng đầu.
  - Tiền căn phải `tabular-nums`, dùng `formatMoney` sẵn có (`lib/valuation-report.ts`).
  - URL là bộ lọc đã áp dụng (`q/source/from/to/page`), có `DataPagination` và `AbortController`.
- Sửa:
  - `router/index.ts`: route `report/tsss` (`report-tsss`), breadcrumb `Báo cáo › Báo cáo TSSS`, `authorities: ['admin']`, comment trỏ về `TsssReportController` cũ.
  - `nav-items.ts:114`: thêm `to`.
  - `guard.spec.ts`, `layout.spec.ts`.

---

### Task 1: Truy vấn (backend)
- `StructureDetail`, `StructureDetailsConverter`, `TsssReportQuery`, `TsssReportFilter`, `TsssReportRepository`.
- [ ] Viết test trước: test converter (JSON hỏng hoặc null → rỗng, đọc đúng 4 field).
- [ ] Viết test trước: `TsssReportRepositoryTest` (`@MysqlDataJpaTest`, dữ liệu có MARK riêng), các ca:
  - chỉ lấy `type` 1/2/3 và trạng thái "Đã giao dịch"
  - loại `agc`, `rep`, `r` đã xóa
  - lọc `source`
  - `keyword` khớp từng cột trong 5 cột, ký tự `%` được escape
  - ngày: một lần ký bất kỳ nằm trong khoảng thì khớp; biên `to` lấy trọn ngày
  - số chứng thư lấy từ lịch sử, fallback `ac.certificate_no` khi không có lịch sử
  - `issue_date` nối nhiều ngày theo thứ tự ký
  - sắp `agc.id DESC`
  - đếm khớp số dòng
  - stream trả đủ dòng
- [ ] Chạy để thấy test FAIL → cài đặt → PASS → commit `feat(report): truy vấn báo cáo TSSS`.

### Task 2: Ánh xạ item và Excel (backend)
- [ ] Viết test trước `TsssReportItemTest`:
  - 0 công trình → 1 dòng rỗng; 3 công trình → 3 dòng đúng thứ tự
  - `"N/A"` → `""`
  - liên hệ: `"A: 090"`, `"A"`, `"A:090:x"`, null
  - tọa độ: có DMS, chỉ có số, thiếu một vế, cả hai null
  - loại đất: bỏ trùng giữa ngoài và trong quy hoạch; rỗng → `"N/A"`
  - `formatCertificateNo`: mảng, chuỗi thường, mảng rỗng
- [ ] Viết test trước `TsssReportExcelExporterTest`:
  - tiêu đề và vùng merge của tiêu đề
  - đủ 37 nhãn header
  - bản ghi 3 công trình: merge đúng các cột cấp bản ghi, không merge cột 26-29 và 32
  - ô tiền có định dạng `#,##0`
  - số bản ghi lớn hơn cửa sổ streaming
- [ ] Chạy để thấy test FAIL → cài đặt → PASS → commit `feat(report): dữ liệu và file Excel báo cáo TSSS`.

### Task 3: Service, controller, quyền (backend)
- [ ] Viết test trước service (Mockito):
  - STT bắt đầu từ `page*size+1`
  - `filterOptions` trả nhãn `bankName - bankCode`
  - export áp dụng bộ lọc và đóng stream
- [ ] Viết test trước `@WebMvcTest`:
  - chưa đăng nhập → 401 (3 path)
  - `admin` → 200
  - `BGD`, `BGD_HN`, `NVKT` → 403
  - bind đủ 4 tham số
  - 400 khi: ngày sai, `from > to`, `size=101`, `page=-1`, keyword quá 500 ký tự
  - header tên file
- [ ] Chứng minh test quyền FAIL khi gỡ rule.
- [ ] `./mvnw test` toàn bộ → BUILD SUCCESS. Cập nhật `backend/CLAUDE.md` nếu thêm helper.
- [ ] Commit `feat(report): API báo cáo TSSS`.

### Task 4: Frontend
- [ ] Spec API: bỏ tham số rỗng, map `from` / `to`, tên file export.
- [ ] Spec view:
  - 37 nhãn header đúng thứ tự
  - rowspan đúng với bản ghi có 3 công trình và bản ghi có 0 công trình
  - Tìm kiếm → API và URL nhận đúng tham số
  - ô chọn nguồn lấy từ `filter-options`
  - export dùng bộ lọc đã áp dụng
  - định dạng tiền
  - trạng thái rỗng / lỗi
- [ ] `guard.spec`: `admin` vào được, `BGD` bị chặn.
- [ ] `layout.spec`: link hiện; bỏ href này khỏi selector placeholder.
- [ ] Chạy lần lượt: `npm run lint`, `npm run type-check`, `npm run test:unit -- --maxWorkers=2`. Riêng `AppraisalRequestListView.spec.ts` đã fail từ trước.
- [ ] Commit theo hai bước: `feat(report): api báo cáo TSSS`, rồi `feat(report): màn báo cáo TSSS`.

### Task 5: Kiểm tra cuối
- [ ] Đọc log test BE và FE, ghi số thật.
- [ ] Code review bằng subagent `model: "sonnet"`: so mapper và SQL với `TsssReportServiceImpl` / `TsssReportRepositoryImpl`.
- [ ] Nếu có môi trường: đăng nhập admin, chọn cùng bộ lọc trên màn cũ và màn mới, so ≥ 3 bản ghi (có 0, 1, nhiều công trình), mở file Excel.
- [ ] Báo người dùng: số test, phần chưa kiểm, các khác biệt cố ý.

## Lưu ý kỹ thuật

- **Máy test thiếu RAM:**
  - `MAVEN_OPTS="-Xms32m -Xmx256m -XX:+UseSerialGC -XX:TieredStopAtLevel=1"`
  - `-DargLine="-Xms32m -Xmx768m -XX:+UseSerialGC -XX:TieredStopAtLevel=1 -Djdk.attach.allowAttachSelf=true -XX:+EnableDynamicAgentLoading"`
  - Chạy test từng lệnh một.
- **Trước khi map:** kiểm kiểu thật trong schema của các cột `agc` (`transaction_listing_time`, `road_width`, `latitude`, `estimate_price`, …) bằng `INFORMATION_SCHEMA`.
- **JOIN:** toàn bộ JOIN theo id nên không vướng collation. Riêng chuỗi `'Đã giao dịch'` truyền thành tham số đặt tên để tránh lệch charset.
- **Export:** không gọi truy vấn khác trong lúc stream. Mapper là hàm thuần nên ghi Excel ngay trong lúc đọc stream, không cần nạp hết vào RAM như VCB.
- Theo `backend/CLAUDE.md`, `.claude/rules/jpa-spring.md`, `.claude/rules/design-principles.md`, `frontend/CLAUDE.md`, `frontend/DESIGN_SYSTEM.md`. Comment, Javadoc và nhãn đều bằng tiếng Việt có dấu.

## Thực thi

- **Bước đầu:** chép plan vào `docs/superpowers/plans/2026-10-26-migrate-tsss-report.md` (không commit).
- **Agent backend (Sonnet):** Task 1 → 3 tuần tự.
- **Agent frontend (Sonnet):** Task 4, song song theo "Hợp đồng API".
- **Task 5:** chạy sau cùng.
