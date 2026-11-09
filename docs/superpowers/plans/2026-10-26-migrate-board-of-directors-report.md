# Migrate "Báo cáo Ban lãnh đạo" (/report/board-of-directors)

> **Subagent CHỈ dùng `sonnet`.** Mọi lần gọi tool `Agent` truyền `model: "sonnet"`.
> Làm thẳng trên `develop` của hai submodule (`git -C backend …`, `git -C frontend …`). Không push, không commit repo gốc. Trước khi bắt đầu, kiểm tra cây làm việc sạch; nếu không sạch thì dừng và báo.
> **Bước đầu khi thực thi:** chép file này vào `docs/superpowers/plans/2026-10-26-migrate-board-of-directors-report.md` (không commit).

## Context

Trong menu FE mới, mục "Báo cáo Ban lãnh đạo" (`frontend/src/components/layout/nav-items.ts:115`, `authorities: ['admin']`) vẫn là placeholder, chưa có `to`. Plan này port màn tổng hợp tài chính theo tháng × chi nhánh của hệ thống cũ:
- **Backend:** thêm `/api/reports/board-of-directors`, `/filter-options`, `/export`.
- **Frontend:** thêm route `/report/board-of-directors`.

Màn này khác các báo cáo trước: **không phân trang, không có danh sách bản ghi.** Kết quả là một ma trận cố định 15 dòng chỉ tiêu × N chi nhánh + cột "Tổng cộng", tính ở Java từ 5 nguồn dữ liệu.

### Spec là hệ thống cũ (`td-project-legacy/src/main/java/com/jvb/thanhdo/…`)

| Nội dung | Vị trí |
|---|---|
| Controller (kiểm `role == admin`; mặc định mọi chi nhánh) | `controller/BoardOfDirectorsReportController.java` |
| Engine 15 dòng | `service/impl/BoardOfDirectorsReportServiceImpl.java` |
| Truy vấn | `AppraisalRequestRepository.findForBoardReport` (dòng 738), `findForBoardReportByYear` (750), `findAllByContractIdInAndDeleteFlag`; `AccountingRepository.findForBoardReport` (577), `sumPaidAmountByContractIds` (580); `AppraisalProfileRepository.findTravelDistancesForBoardReport` (21), `findCompletedAppraisalRequestIds` (159) |
| Hằng số | `CommonConstants.BoardReport.EXCLUDED_BRANCH_CODES = {CM, TH, NB}`, `CommonConstants.Subsection.*` (dòng 251-266) |
| Công tác phí | `CommonUtils.calculateAdvancePayment` (đã port: `backend/…/common/util/AdvancePayment.calculate`) |
| Excel | `controller/component/BoardOfDirectorsReportExcelExporter.java` |
| Giao diện | `resources/templates/report/board-of-directors-report.html` |
| Quyền | `config/SecurityConfig.java:156-158` → `hasAuthority('admin')` cho trang và export |

## Hành vi hệ thống cũ (spec chi tiết)

**Tham số.** `monthYear` (`yyyy-MM`, trống → tháng hiện tại; sai dạng → 500), `branches` (nhiều mã; trống → mọi chi nhánh).

**Chi nhánh.**
- Danh sách hiển thị: `linked_branch` chưa xóa, bỏ `CM`, `TH`, `NB`, sắp HN/HCM đầu rồi theo mã (Collator vi-VN).
- `actualBranchCodes`: mã được chọn trừ mã bị loại; không chọn → toàn bộ danh sách hiển thị.
- Mọi giá trị chỉ cộng khi chi nhánh xác định được nằm trong `actualBranchCodes`.

**Tháng.** `M` = tháng chọn, `M-1` = tháng trước. Lọc yêu cầu theo cột sinh `contract_year = yyyy` và `contract_month IN ('MM', 'M')` (hai dạng có/không số 0).

**Nguồn dữ liệu.**
1. `currentRequests`: `p_appraisal_request` tháng `M`, `delete_flag='0'`, `linked_branch` NULL hoặc không thuộc CM/TH/NB.
2. `oldRequests`: như trên cho tháng `M-1`.
3. `accountings`: `accounting` có `accounting_date ∈ [ngày 1 của M, cuối M 23:59:59]`, chưa xóa (`0` hoặc NULL); tiểu mục lấy qua join `subsection.subsection_name = accounting.subsection` (không tìm thấy → bỏ bút toán).
4. `travels`: `appraisal_profile` chưa xóa có `travel_distance >= 10000`, nối yêu cầu tháng `M` (`delete_flag='0'`), lấy `linked_branch` của yêu cầu.
5. `yearRequests`: mọi yêu cầu `contract_year = yyyy`, `delete_flag='0'` (không lọc CM/TH/NB ở SQL).
6. `extraRequests`: yêu cầu chưa xóa có `contract_id` thuộc các bút toán ở (3).
- `contractMap`: `contract_id → request`, ưu tiên current, rồi old, rồi extra (`putIfAbsent`).

**15 dòng (STT: nội dung → công thức).**

| STT | Nội dung | Công thức |
|---|---|---|
| 1 | Doanh số tháng hiện tại | Σ `valuation_fee` của `currentRequests` theo `linked_branch`; parse `replace(",","").replace(".","")` → Long, lỗi → 0 |
| 2 | Doanh số các tháng cũ | như dòng 1 với `oldRequests` |
| 3 | Tổng thu | 4 + 5 |
| 4 | Doanh thu HĐ tháng hiện tại | bút toán tiểu mục "Hợp đồng" mà yêu cầu (qua `contractMap`) có hợp đồng tháng `M`; PT dương, PC âm |
| 5 | Doanh thu HĐ tháng cũ | bút toán "Hợp đồng" còn lại (kể cả không tìm thấy yêu cầu); PT dương, PC âm |
| 6 | Tổng thu HO thu từ CN | tiểu mục "Nhận chuyển quỹ" (hoặc id 15); PT dương, PC âm |
| 7 | Tổng chi | 8 + 9 + 10 + 11 + 12 |
| 8 | Phí khai thác đã chuyển kỳ này | bút toán `type = 3`; chỉ PC |
| 9 | Khách gửi phí/Chi phí thuê VP/Chi khác | tiểu mục "Khách gửi phí", "Chi phí thuê VP" (hoặc id 3), "Chi khác"; chỉ PC |
| 10 | Lương/BHXH | "Lương" (hoặc id 14), "BHXH" (hoặc id 1); chỉ PC |
| 11 | Công tác phí tạm tính | Σ `AdvancePayment.calculate(branch, travel_distance)` cho từng hồ sơ ở (4) |
| 12 | Thuế VAT tạm tính | `Math.round(r3 − r3 / 1.08)` |
| 13 | Chênh lệch | 3 − 7 |
| 14 | Công nợ | với `currentRequests` có ít nhất một `appraisal_profile` `status = 6` và `deleted = 0`: Σ max(0, phí − đã trả) |
| 15 | Công nợ cả năm | như dòng 14 cho mọi `yearRequests` |

- Chỉ xét bút toán `accounting_mode` PT hoặc PC (không phân biệt hoa thường), `amount_of_money` khác NULL và khác 0, có tiểu mục.
- Thứ tự phân loại dòng (ưu tiên từ trên xuống): "Hợp đồng" → "Nhận chuyển quỹ" → `type = 3` → dòng 9 → dòng 10 → bỏ qua.
- **Chi nhánh của bút toán:** tiểu mục Lương, BHXH, Chi phí thuê VP, Nhận chuyển quỹ (theo tên hoặc id 14/1/3/15) → `accounting.linked_branch`; còn lại → `linked_branch` của yêu cầu trong `contractMap`, không có thì `accounting.linked_branch`.
- **Phí trong dòng 14-15** dùng `parseLongSafe`: trim, bỏ `,`; có `.` thì `BigDecimal.longValue()`, lỗi → 0 (khác cách parse của dòng 1-2).
- **Đã trả** = `sumPaidAmountByContractIds`: bút toán chưa xóa `type = 1`, tiểu mục thuộc `PAYMENT_SUBSECTIONS`, PT cộng PC trừ, theo `contract_id`.
- Cột "Tổng cộng" = Σ các chi nhánh trong `actualBranchCodes`.

**Hiển thị.**
- Web: bảng STT | Nội dung | từng chi nhánh (tên) | Tổng cộng.
  - Dòng 3, 7, 13 nền màu chính, chữ trắng đậm.
  - Số kiểu `1.234.567`; giá trị 0 hoặc thiếu hiển thị trống.
  - Cột tổng in đậm.
- Bộ lọc: `<input type="month">`, select2 nhiều chi nhánh (placeholder theo template), nút Tìm kiếm / Xuất Excel.
- Excel:
  - Tiêu đề "BÁO CÁO BAN LÃNH ĐẠO" + " THÁNG {monthYear}" nếu có (giá trị thô `yyyy-MM`).
  - Header ở hàng 3, cột B: STT (7), Nội dung (40), từng chi nhánh (20), Tổng cộng (20).
  - Dòng 3/7/13 đậm, nền PALE_BLUE.
  - Số 0 hoặc thiếu ghi chuỗi rỗng.
  - Tên file `Bao-cao-Ban-lanh-dao_{monthYear_}dd-MM-yyyy_HH:mm:ss.xlsx`.

**Quyền.** Chỉ `admin` cho menu, trang và export. Không lọc phạm vi theo người xem.

## Quyết định

1. **Giữ quyền:** `hasAuthority("admin")` cho `/api/reports/board-of-directors/**`; FE route `authorities: ['admin']` (như TSSS).
2. **Giữ y nguyên công thức và cách phân loại 15 dòng**, kể cả các điểm lạ. Ghi chúng vào Javadoc engine:
   - Hai cách parse `valuation_fee` khác nhau (dòng 1-2 và dòng 14-15).
   - Nhận diện tiểu mục theo tên hoặc id (14/1/3/15).
   - Dòng 5 gồm cả bút toán "Hợp đồng" không tìm thấy yêu cầu.
   - Yêu cầu trùng `contract_id` lấy bản đầu tiên.
3. **Khác biệt nhỏ** (ghi vào Javadoc controller):
   - `month` sai định dạng trả 400 thay vì 500. Trống thì lấy tháng hiện tại theo `Asia/Ho_Chi_Minh`.
   - Mã chi nhánh được chọn chỉ giữ những mã nằm trong danh sách hiển thị; mã lạ hoặc đã xóa bị bỏ. Ở bản cũ, mã lạ vẫn được cộng vào cột tổng nhưng không có cột riêng.
   - Khoảng ngày hạch toán là `[đầu tháng, đầu tháng sau)` thay vì `≤ 23:59:59`, nên bắt được cả phần giây lẻ.
   - Tiêu đề Excel hiện tháng dạng `MM/yyyy` (bản cũ in `yyyy-MM` thô), merge ngang qua bảng.
   - Tên file `Bao-cao-Ban-lanh-dao_{yyyy-MM}_dd-MM-yyyy_HH-mm-ss.xlsx`, encode UTF-8. Giờ dùng dấu `-` vì `:` không hợp lệ trong tên file trên Windows.
   - Truy vấn trả projection thay vì nạp entity đầy đủ.
   - Các truy vấn phụ ("hoàn thành", "đã trả") dùng `IN` theo lô, mỗi lô tối đa 1000 id.

## Hợp đồng API (BE và FE làm song song)

Query `BoardOfDirectorsReportQuery`: `month` (`yyyy-MM`, tùy chọn), `branches` (danh sách mã, lặp tham số `branches=HN&branches=HP`, tùy chọn).

**`GET /api/reports/board-of-directors` → `BoardOfDirectorsReport`**
```
BoardOfDirectorsReport {
  month: "2026-10",                        // tháng đã áp dụng (sau khi lấy mặc định)
  branches: [ { code, name } ],            // cột hiển thị, đúng thứ tự
  rows: [ { index: 1..15, content, highlight: boolean, values: { [code]: number }, total: number } ]
}
```
- Luôn 15 dòng; `values` có đủ key cho mọi chi nhánh trong `branches`, kể cả giá trị 0.
- `highlight = true` ở dòng 3, 7, 13. Client tự ẩn số 0.

**`GET /api/reports/board-of-directors/filter-options` → `{ branches: [{ code, name }] }`**: danh sách chi nhánh hiển thị, không tính CM/TH/NB.

**`GET /api/reports/board-of-directors/export`**: cùng tham số, trả xlsx với `Content-Disposition` dạng `filename*=UTF-8''…`.

---

## Cấu trúc file

**Backend** (`backend/src/main/java/com/thanhdo/tdprojectbe/`)

**Truy vấn:** `repository/BoardOfDirectorsReportRepository`. Native query vì cần cột sinh `contract_month`/`contract_year` chưa được map, và cần join tiểu mục theo tên. Gồm 6 method trả interface projection:
- `requestsOfMonth(month, monthNoZero, year, excluded)`: trả `id`, `contractId`, `linkedBranch`, `valuationFee`.
- `requestsOfYear(year)`
- `requestsByContractIds(ids)`
- `accountingsBetween(from, toExclusive)`: trả `contractId`, `linkedBranch`, `amount`, `mode`, `type`, `subsectionName`, `subsectionId`, `contractMonth`, `contractYear`. Các cột hợp đồng lấy qua `LEFT JOIN` sang yêu cầu chỉ khi cần; nếu không thì để Java tra `contractMap`.
- `travelsOfMonth(...)`
- `completedRequestIds(ids)`
- `paidByContractIds(ids, Subsection.PAYMENT_SUBSECTIONS)`

Theo mẫu `VcbReportRepository` / `DebtReportSql`.

**Hằng số và tham số:**
- Tiểu mục: thêm vào `model/entity/Subsection` các hằng còn thiếu (`RECEIVE_FUND_TRANSFER`, `CUSTOMER_FEE_DEPOSIT`, `SALARY`, `SOCIAL_INSURANCE`, `OFFICE_RENTAL_FEE`, `OTHER_EXPENSES`). Kiểm trước, không tạo trùng.
- `model/dto/BoardOfDirectorsReportQuery`: `month` có `@Pattern` `yyyy-MM`; `branches` là `List<String>`.

**Engine:** `service/BoardOfDirectorsReportCalculator` là lớp thuần, không truy cập DB.
- Nhận một record `BoardReportInputs` gồm các danh sách projection, map đã trả, tập id hoàn thành, `actualBranchCodes` và `YearMonth`.
- Trả `List<Row>`.
- Port 1:1 các vòng lặp của `getReportData`.
- Tách hàm nhỏ: `classify(acc)`, `branchOf(acc, req)`, `parseFeeRevenue` / `parseFeeDebt`.

Để dễ test, toàn bộ logic nằm ở lớp này còn service chỉ gom dữ liệu.

**DTO:** `model/dto/BoardOfDirectorsReport` (+ `Row`, `BranchColumn`), `BoardOfDirectorsReportFilterOptions`.

**Excel:** `common/util/export/excel/BoardOfDirectorsReportExcelExporter`.
- Lớp con của `StreamingExportCommon`, dù dữ liệu chỉ 15 dòng, để giữ một kiểu exporter chung.
- Header động theo chi nhánh; 3 dòng nổi bật; số dạng `#,##0`, giá trị 0 để trống.
- Tham khảo `MonthlyReportExcelExporter` / `TsssReportExcelExporter`.

**Service và controller:**
- `service/BoardOfDirectorsReportService`, chạy trong `@Transactional(readOnly = true)`:
  - Chi nhánh lấy từ `BranchService.activeBranches()` (đã sắp HN/HCM đầu), lọc bỏ CM/TH/NB.
  - Tiếp theo: các truy vấn → calculator → DTO.
  - `filterOptions()`.
  - `export(query, out)`.
- `controller/BoardOfDirectorsReportController`: theo mẫu `CashCollectorReportController`, bỏ phân trang.

**Cấu hình:**
- `config/SecurityConfig`: matcher `/api/reports/board-of-directors`, `/api/reports/board-of-directors/**` → `hasAuthority("admin")`, đặt cạnh rule TSSS.
- `backend/CLAUDE.md`: thêm exporter vào danh sách mẫu nếu hợp lý.

**Frontend** (`frontend/src/`)
- `api/board-of-directors-reports.ts`: `getBoardOfDirectorsReport`, `exportBoardOfDirectorsReport`, `getBoardOfDirectorsFilterOptions` cùng các type. `branches` gửi dạng lặp tham số; bỏ tham số rỗng. Theo mẫu `api/tsss-reports.ts`.
- `views/report/BoardOfDirectorsReportView.vue`, theo khuôn `TsssReportView.vue` / `ContractTrackingReportView.vue`:
  - Bộ lọc:
    - `MonthPicker` (`components/form/MonthPicker.vue`, v-model `YYYY-MM`) cho "Tháng".
    - `MultiSelect` (`components/form/MultiSelect.vue`) cho "Chi nhánh", placeholder "Tất cả chi nhánh".
    - Nút "Tìm kiếm" và "Xuất excel".
  - Bảng: `ScrollableTable`, cột "Nội dung" dính trái. Dòng `highlight` dùng token màu chính và chữ đậm theo `DESIGN_SYSTEM.md`.
  - Số căn phải, `tabular-nums`, `formatMoney`; giá trị 0 để trống. Cột Tổng cộng in đậm.
  - URL lưu bộ lọc đã áp dụng (`month`, `branches`). Có `AbortController`, trạng thái tải, lỗi và rỗng.
- Sửa:
  - `router/index.ts`: route `report/board-of-directors` (`report-board-of-directors`), breadcrumb `Báo cáo › Báo cáo Ban lãnh đạo`, `authorities: ['admin']`.
  - `components/layout/nav-items.ts:115`: thêm `to`.
  - `router/__tests__/guard.spec.ts`, `components/layout/__tests__/layout.spec.ts`.

---

### Task 1: Truy vấn (backend)
- [ ] Kiểm kiểu thật của các cột bằng `INFORMATION_SCHEMA`: `contract_month`/`contract_year` (sinh tự động, kiểu chuỗi?), `valuation_fee`, `travel_distance`, `accounting.type`, `subsection.id`.
- [ ] Viết test trước: `BoardOfDirectorsReportRepositoryTest` (`@MysqlDataJpaTest`, dữ liệu có MARK riêng), các ca:
  - tháng có và không có số 0
  - loại yêu cầu đã xóa
  - loại CM/TH/NB nhưng giữ `linked_branch` NULL
  - hạch toán: biên đầu/cuối tháng, bỏ dòng đã xóa
  - join tiểu mục theo tên; bỏ bút toán có tiểu mục không tồn tại
  - travel ≥ 10000 m và hồ sơ chưa xóa
  - `completedRequestIds` chỉ lấy `status = 6` chưa xóa
  - `paidByContractIds`: PT/PC, `type = 1`, đúng tiểu mục
- [ ] Chạy để thấy FAIL lúc chạy (không chỉ lỗi compile; tạo trước chữ ký method rỗng nếu cần) → cài đặt → PASS → commit `feat(report): truy vấn báo cáo Ban lãnh đạo`.

### Task 2: Engine và Excel (backend)
- [ ] Viết test trước `BoardOfDirectorsReportCalculatorTest`, mỗi dòng ít nhất một ca:
  - dòng 1-2 parse `"1.500.000"` và `"1,500,000"`; chuỗi lỗi → 0
  - dòng 4 và 5 tách theo tháng hợp đồng; dòng 5 khi không có yêu cầu
  - dòng 4-6 PT dương, PC âm
  - dòng 8-10 chỉ PC
  - chi nhánh của Lương/BHXH/thuê VP/chuyển quỹ lấy từ bút toán; tiểu mục khác lấy từ yêu cầu
  - nhận diện tiểu mục theo id khi tên khác
  - `type = 3` xếp sau "Hợp đồng"
  - dòng 11 dùng `AdvancePayment`
  - dòng 12 làm tròn VAT (ví dụ `r3 = 1_080_000` → 80_000)
  - dòng 3, 7, 13 tính đúng công thức
  - dòng 14 chỉ yêu cầu hoàn thành; công nợ âm → 0
  - dòng 15 gồm cả yêu cầu tháng hiện tại
  - chi nhánh ngoài `actualBranchCodes` bị bỏ
  - cột tổng
  - `contractMap` ưu tiên current > old > extra
- [ ] Viết test trước `BoardOfDirectorsReportExcelExporterTest`:
  - tiêu đề có tháng
  - header động (2 chi nhánh → 5 cột)
  - dòng 3/7/13 đậm
  - giá trị 0 là chuỗi rỗng
  - định dạng `#,##0`
- [ ] FAIL → cài đặt → PASS → commit `feat(report): tính toán và file Excel báo cáo Ban lãnh đạo`.

### Task 3: Service, controller, quyền (backend)
- [ ] Viết test trước service (Mockito):
  - tháng trống → tháng hiện tại
  - lọc chi nhánh: bỏ CM/TH/NB và mã lạ
  - không chọn → tất cả
  - gọi truy vấn với tháng trước đúng khi `M` là tháng 1 (lùi năm)
  - `filterOptions`
- [ ] Viết test trước `@WebMvcTest`:
  - chưa đăng nhập → 401 (3 path)
  - `admin` → 200
  - `BGD` / `BGD_HN` / `NVKT` → 403
  - bind `branches` lặp tham số
  - `month=2026-13` hoặc `abc` → 400
  - header tên file
- [ ] Chứng minh test quyền FAIL khi gỡ rule.
- [ ] `./mvnw test` toàn bộ → BUILD SUCCESS, ghi số test. Commit `feat(report): API báo cáo Ban lãnh đạo`.

### Task 4: Frontend
- [ ] Spec API: `branches` lặp tham số, bỏ tham số rỗng, tên file export.
- [ ] Spec view:
  - header động theo `branches`
  - 15 dòng, 3 dòng nổi bật
  - số 0 để trống, định dạng tiền
  - Tìm kiếm → URL và API nhận `month`/`branches`
  - export dùng bộ lọc đã áp dụng
  - trạng thái lỗi
- [ ] `guard.spec`: `admin` vào được, `BGD` bị chặn. `layout.spec`: link hiện; bỏ href này khỏi selector placeholder.
- [ ] Chạy lần lượt: `npm run lint`, `npm run type-check`, `npm run test:unit -- --maxWorkers=2`. Riêng `AppraisalRequestListView.spec.ts` đã fail từ trước.
- [ ] Commit theo hai bước: `feat(report): api báo cáo Ban lãnh đạo`, rồi `feat(report): màn báo cáo Ban lãnh đạo`.

### Task 5: Kiểm tra cuối
- [ ] Đọc log test BE và FE, ghi số thật.
- [ ] Code review bằng subagent `model: "sonnet"`: so calculator với `BoardOfDirectorsReportServiceImpl` từng dòng, và so SQL với 7 query cũ.
- [ ] Nếu có môi trường: đăng nhập admin, chọn ≥ 2 tháng (gồm tháng 1) và tất cả / một phần chi nhánh, so từng ô giữa màn cũ và mới, mở file Excel.
- [ ] Báo người dùng: số test, phần chưa kiểm, các khác biệt cố ý.

## Lưu ý kỹ thuật

- **Máy test thiếu RAM:**
  - `MAVEN_OPTS="-Xms32m -Xmx256m -XX:+UseSerialGC -XX:TieredStopAtLevel=1"`
  - `-DargLine="-Xms32m -Xmx768m -XX:+UseSerialGC -XX:TieredStopAtLevel=1 -Djdk.attach.allowAttachSelf=true -XX:+EnableDynamicAgentLoading"`
  - Chạy test từng lệnh một.
- **Tiểu mục:**
  - `accounting.subsection` lưu tên, không có khóa ngoại (V106).
  - Join `subsection` theo tên có thể nhân dòng nếu trùng tên. Kiểm trên DB thật; nếu có trùng thì lấy `MIN(id)` qua subquery.
  - Chuỗi tiểu mục truyền làm tham số, không viết cứng trong SQL.
- **Ngày:** dùng `ReportPeriod` hoặc `YearMonth` cho `[start, nextStart)`; tháng hiện tại lấy theo `Clock` inject được để test.
- **Code có sẵn để dùng lại:**
  - `AdvancePayment.calculate`
  - `BranchService.activeBranches()`
  - `Subsection.PAYMENT_SUBSECTIONS`
  - `AppraisalRequest.DELETE_FLAG_ACTIVE`, `Accounting.MODE_RECEIPT/MODE_PAYMENT/TYPE_EXPLOITATION_FEE/TYPE_CREATED_BY_USER`, `AppraisalProfile.NOT_DELETED`
  - `MonthPicker`, `MultiSelect`, `formatMoney`, `monthStartDate` (`lib/juridical-report.ts`)
- **Tài liệu phải theo:** `backend/CLAUDE.md`, `.claude/rules/jpa-spring.md`, `.claude/rules/design-principles.md` (calculator dưới ~200 dòng, hàm dưới ~30 dòng), `frontend/CLAUDE.md`, `frontend/DESIGN_SYSTEM.md`. Comment, Javadoc và nhãn đều bằng tiếng Việt có dấu.

## Thực thi

- **Agent backend (Sonnet):** Task 1 → 3 tuần tự.
- **Agent frontend (Sonnet):** Task 4, song song theo "Hợp đồng API".
- **Task 5:** chạy sau cùng.
