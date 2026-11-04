# Migrate "Báo cáo tháng" (/report/monthly-report) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: superpowers:subagent-driven-development (hoặc executing-plans). Steps dùng checkbox (`- [ ]`).
>
> **Model cho subagent: CHỈ dùng `sonnet`.** Mọi lần gọi tool `Agent` phải truyền `model: "sonnet"`.

## Context

Mục "Báo cáo tháng" trong menu FE mới còn là placeholder (`frontend/src/components/layout/nav-items.ts:109`, có `authorities: ['BGD', 'TPTD', 'NVTD']` nhưng chưa có `to`). Plan này port `/report/monthly-report` và `/report/export-monthly-report` sang:
- backend: REST `/api/reports/monthly`
- frontend: route `/report/monthly-report`

Báo cáo có một dòng cho mỗi **nhân viên** trong phạm vi người xem, với 4 số liệu tính theo **ngày phân công** (`ar.assign_date`):
- số hồ sơ được phân công
- tổng quãng đường
- số ngày thực hiện trung bình
- số ngày thực hiện cao nhất

Không có dòng tổng, không giới hạn năm, mỗi trang 20 dòng.

Làm thẳng trên `develop` của hai submodule như các báo cáo trước. Không push, không commit ở repo gốc.

**Spec là hệ thống cũ** (`td-project-legacy/src/main/java/com/jvb/thanhdo/…`):

| Nội dung | Vị trí |
|---|---|
| Endpoint trang và export | `controller/ReportController.java:567-598` |
| Dựng danh sách cho trang và cho export | `service/impl/ReportServiceImpl.java:1703-1838` (`getMonthlyReport`, `getMonthlyReportExport`) |
| Kiểm tra chi nhánh HN | `ReportServiceImpl.java:1552` (`checkRole`) |
| Mặc định ngày | `ReportServiceImpl.java:1233` (`initBaseSearchReportRequest`) |
| Truy vấn danh sách nhân viên | `repository/EmployeeRepository.java:121-132` |
| Đếm hồ sơ được phân công | `repository/AppraisalRequestRepository.java:541` (`getTotalRequestByEmployee`) |
| Ngày thực hiện và quãng đường | `repository/AppraisalProfileRepository.java:451` (`getWorkDayAndTravelDistanceByEmployee`) |
| DTO | `entity/MonthlyReport.java` |
| Excel | `controller/component/MonthlyReportExcelExporter.java` |
| Giao diện | `templates/report/monthly-report.html` |
| Menu | `fragment/mainnav-container.html:134` |
| Ô chọn cán bộ | `service/impl/EmployeeServiceImpl.java:297` (`getListEmployeesByPosition("NVTD")`) |
| Tên chi nhánh | `EmployeeServiceImpl.java:337` (`setBranchName`) |

**Khuôn mẫu đã có trên `develop`:**
- Backend: `CashCollectorReport*`, `AccountingReport*`. Luồng: Query → Filter (chuẩn hóa + phạm vi) → repository → service map Row → Item → controller (`Page`, `/export` SXSSF, `/filter-options`).
- Frontend: `CashCollectorReportView.vue`.

## Hành vi hệ thống cũ

| Mục | Hệ thống cũ |
|---|---|
| Quyền trang | Trước hết phải qua rule chung `/report/**`: `admin, BGD, NVKT, NVKD, NVTD, TPTD, NVPI`; KSV qua được vì có thêm authority `TPTD`. Sau đó controller chỉ cho chức vụ `BGD/TPTD/NVTD/KSV` vào, người khác nhận `page-forbidden`. Chức vụ null thì NPE. |
| Quyền export | Chỉ rule chung `/report/**`, controller không kiểm tra gì thêm. Vì vậy admin/NVKT/NVKD/NVPI vẫn xuất được, nhưng chỉ ra dòng của chính mình. |
| Menu | `hasAnyAuthority('BGD','TPTD','NVTD')`. KSV thấy menu vì có authority TPTD. FE mới đã có sẵn. |
| Phạm vi trang | Có "HN" = trong `branch` của người xem có mã đúng bằng `HN` (`checkRole`). Có 3 mức: chức vụ `BGD` có HN → mọi nhân viên chưa xóa. `BGD`/`TPTD`/`KSV` còn lại → nhân viên có `e.branch IN (mã chi nhánh người xem)`, so khớp nguyên chuỗi nên nhân viên nhiều chi nhánh "HN,DN" không khớp. Còn lại (NVTD) → chỉ chính mình. Danh sách gồm nhân viên mọi chức vụ, kể cả người đã nghỉ, miễn chưa xóa. |
| Lọc cán bộ | `assignment` → `employee_id = :assignment`, áp trong phạm vi. Ô chọn là `getListEmployeesByPosition("NVTD")`: quản trị toàn cục thấy mọi NVTD, người khác thấy NVTD cùng chi nhánh (`FIND_IN_SET`); người nghỉ việc xếp cuối. Nhãn `first + ' ' + last`. |
| Sắp xếp / trang | Không có ORDER BY, thứ tự không xác định. 20 dòng/trang. |
| Ngày | `assign_date BETWEEN from 00:01 AND to 23:59`. Bỏ trống thì từ 1900-01-01 00:01 đến 9999-12-30 23:59. |
| Số hồ sơ được phân công | `COUNT` các `p_appraisal_request` có `assignment IN (ids)`, `assign_date` trong khoảng và `delete_flag = '0'`, gom theo `assignment`. Không có thì hiện 0. |
| Ngày thực hiện / quãng đường | Lấy từ `appraisal_profile ap`, `LEFT JOIN p_appraisal_request ar` theo request, `LEFT JOIN appraisal_profile_history hs` (status 3) và `hs2` (status 4). Lọc `ar.assignment IN ids`, `assign_date` trong khoảng, `ap.deleted = 0`. Không lọc `ar.delete_flag`, không lọc `hs.deleted`. `workDay` = `ap.status = 3` → `DATEDIFF(NOW(), hs.status_at)`; `ap.status > 3` → `DATEDIFF(hs2.status_at, hs.status_at)` (cần cả hai mốc); còn lại NULL. Gom theo `assignment`: `AVG(workDay)`, `MAX(workDay)`, `SUM(ap.travel_distance)`. Hồ sơ có nhiều dòng lịch sử thì bị nhân dòng. |
| Cột (7) | STT · Tên nhân viên (`first + ' ' + last`) · Chi nhánh (mã → tên các chi nhánh chưa xóa, nối `", "`, bỏ mã không tìm thấy) · Số lượng hồ sơ được phân công · Quãng đường (mét → `"%.2f km"`, null → `0 km`) · Số ngày thực hiện trung bình (2 chữ số thập phân, null/0 → `0`) · Số ngày thực hiện cao nhất (null → `0`) |
| Excel | `ExportCommon`: tiêu đề "BÁO CÁO THÁNG" ở B1, header ở dòng 3 bắt đầu từ cột B, độ rộng 7/30/40/20/15/20/20. Quãng đường ghi dạng chuỗi "x.xx km", trung bình là chuỗi `%.2f`. Tên file `Báo-cáo-tháng_<dd-MM-yyyy_HH:mm:ss>.xlsx`. |

## Quyết định (đã chốt với người dùng)

1. **Quyền giữ y nguyên.**
   - Backend `SecurityConfig`:
     - Trang, gồm `/api/reports/monthly` và `/api/reports/monthly/filter-options`: `hasAnyAuthority(BGD, TPTD, NVTD, KSV)`. Rule theo authority tương đương bước kiểm tra chức vụ trong controller cũ, vì KSV có authority `KSV` trong token mới.
     - `/api/reports/monthly/export`: `admin, BGD, NVKT, NVKD, NVTD, TPTD, KSV, NVPI` (rule chung). Đặt matcher export **trước** matcher trang.
     - Dùng hằng `Positions`. Comment ghi rõ: trang có kiểm tra chức vụ trong controller cũ, còn export chỉ qua rule chung.
     - Kiểm tra trước với agent: token mới có cấp authority `KSV` cho chức vụ KSV không (xem `guard.spec`/`Authorities`). Nếu không, dùng đúng authority token cấp.
   - FE: route `['BGD', 'TPTD', 'NVTD', 'KSV']`. Menu giữ nguyên `authorities` hiện có, chỉ thêm `to`. Kiểm tra KSV có thấy menu không, theo cách FE ánh xạ KSV↔TPTD ở các mục khác. Nếu chưa thấy thì ghi chú, không tự mở rộng.
2. **Một quy tắc phạm vi cho cả trang và Excel**, lấy theo quy tắc của trang:
   - `ALL`: chức vụ BGD có mã `HN` (`MsbRequestScope.branchCodes(caller)` chứa `HN`, không phân biệt hoa thường, như `AccountingReportFilter`).
   - `BRANCHES`: chức vụ BGD, TPTD hoặc KSV, theo các mã chi nhánh của mình. Giữ so khớp nguyên chuỗi `e.branch IN (:codes)` như cũ.
   - `OWN`: mọi người còn lại, kể cả admin/NVKT/NVKD/NVPI gọi export và người có chức vụ null.

   Bộ lọc cán bộ áp dụng trong mọi phạm vi, kể cả OWN, nên chọn người khác thì ra rỗng. Đây là điểm khác cũ, ghi vào Javadoc.
3. **Công thức số liệu giữ y nguyên**, kể cả việc nhân dòng theo lịch sử, không lọc `ar.delete_flag` và `hs.deleted`, dùng `DATEDIFF(NOW(), …)`. Javadoc trên query ghi rõ là cố ý giữ, kèm ví dụ lỗi.
4. **Truy vấn.**
   - Bước 1, nhân viên trong phạm vi: JPA `Specification<Employee>` gồm `deleteFlag = '0'`, tùy chọn `employeeId`, `branch IN codes` hoặc `employeeId = me`. Trang dùng `findAll(spec, pageable)` sắp `employeeId ASC` (cũ không sắp nên ta thêm để thứ tự ổn định). Export dùng `findAll(spec, Sort)`. Nếu `EmployeeRepository` chưa `extends JpaSpecificationExecutor<Employee>` thì thêm vào.
   - Bước 2, số hồ sơ: JPQL `SELECT ar.assignment, COUNT(ar) … GROUP BY ar.assignment` vào projection record. Kiểm tra tên field `assignment`/`assignDate`/`deleteFlag` trong entity `AppraisalRequest`.
   - Bước 3, ngày thực hiện và quãng đường: native `@Query` port nguyên câu cũ. Lý do native (ghi trên query): `DATEDIFF`/`NOW()` của MySQL và bảng dẫn xuất gom theo hồ sơ. Kiểu cột `ap.status`, `ap.deleted`, `travel_distance` lấy theo entity `AppraisalProfile`; số trạng thái lấy từ hằng `ProfileStatus` qua SpEL nếu có.
   - Ngày dùng `ReportPeriod.ofLimitedYear(null).narrow(fromDate, toDate)`, khoảng `[from, toExclusive)`. Bước 2 và 3 chỉ chạy khi danh sách id không rỗng.
5. **Tên chi nhánh.** `EmployeeService.branchNames()` (private) đã có logic "mã → tên". Chuyển nó thành helper dùng chung `BranchNames` (`common/util` hoặc `service/`): tải bản đồ mã → tên một lần, có hàm `join(String branchCodes)` nối bằng `", "`. Sau đó cho `EmployeeService` dùng lại helper này, vì đây là lần dùng thứ hai và cùng một quy tắc. Thêm một dòng vào bảng trong `backend/CLAUDE.md`.
6. **Ô lọc cán bộ.** Gọi thẳng `RequestFilterOptions.assignees(caller, List.of(Positions.NVTD))`, vì nó đã port đúng `getListEmployeesByPosition`.
7. **Hiển thị.**
   - API trả số thô: `totalDistance` (mét, Long), `avgWorkDay` (BigDecimal, 2 chữ số, HALF_UP), `maxWorkDay`, `assignedCount`. Giá trị null đổi thành 0.
   - FE hiện quãng đường là `km` với 2 chữ số thập phân theo định dạng vi-VN, ví dụ `1.234,56 km`.
   - Excel ghi ô số thay cho chuỗi: km định dạng `#,##0.00" km"`, trung bình `0.00`. Ghi vào mục khác biệt.

Khác hệ thống cũ (Javadoc controller):
1. Excel dùng cùng phạm vi và bộ lọc với trang.
2. Sắp theo `employee_id ASC`.
3. Ngày lấy trọn ngày `[from, toExclusive)` thay cho 00:01–23:59.
4. Ngày sai định dạng, `from > to` hoặc `page < 0` trả 400.
5. Chức vụ null không gây NPE.
6. Excel ghi theo streaming, ô là số, tên file encode UTF-8 với giờ `HH-mm-ss`.
7. Trang và Excel định dạng số giống nhau.

## Hợp đồng API (FE và BE làm song song)

- `GET /api/reports/monthly?employeeId&fromDate&toDate&page&size` (page từ 0, size mặc định 20, tối đa 100) → `Page<MonthlyReportItem>`
- `MonthlyReportItem`: `employeeId (Long), fullName (String), branchNames (String), assignedCount (long), totalDistance (long, mét), avgWorkDay (BigDecimal 2 chữ số), maxWorkDay (long)`
- `GET /api/reports/monthly/filter-options` → `{ employees: [{ value, label }] }`
- `GET /api/reports/monthly/export` (cùng tham số lọc, không có page) → xlsx, `filename*=UTF-8''B%C3%A1o-c%C3%A1o-th%C3%A1ng_…`

---

## File Structure

**Backend** (`backend/src/main/java/com/thanhdo/tdprojectbe/`):

| File | Trách nhiệm |
|---|---|
| Create `model/dto/MonthlyReport{Query,Filter,Item,FilterOptions}.java`, `MonthlyReportCount`/`MonthlyReportWorkDay` (projection) | DTO. `Filter.of(query, caller)` chứa quy tắc phạm vi ở Quyết định 2. |
| Create `repository/MonthlyReportSpecs.java` (hoặc static trong Filter) | `Specification<Employee>` |
| Modify `repository/EmployeeRepository.java` | Thêm `JpaSpecificationExecutor<Employee>` nếu chưa có |
| Modify `repository/AppraisalRequestRepository.java` hoặc tạo `MonthlyReportRepository` | JPQL đếm hồ sơ và native ngày thực hiện. Tạo repository riêng `MonthlyReportRepository extends Repository<AppraisalProfile, Long>` để không phình repository chung. |
| Create `BranchNames` + modify `service/EmployeeService.java` | Helper tên chi nhánh (Quyết định 5) |
| Create `common/util/export/excel/MonthlyReportExcelExporter.java` | Excel, lớp con `StreamingExportCommon` |
| Create `service/MonthlyReportService.java` | `search`, `export`, `filterOptions(callerId)` |
| Create `controller/MonthlyReportController.java` | API |
| Modify `config/SecurityConfig.java`, `backend/CLAUDE.md` | Quyền, bảng helper |

**Frontend** (`frontend/src/`):
- Create: `lib/monthly-report.ts`, `api/monthly-reports.ts`, `views/report/MonthlyReportView.vue`, kèm spec.
- Modify: `router/index.ts`, `components/layout/nav-items.ts:109` (thêm `to`), `router/__tests__/guard.spec.ts`, `components/layout/__tests__/layout.spec.ts`.

---

### Task 1: DTO, phạm vi và tên chi nhánh (backend)

- `MonthlyReportQuery(Long employeeId, LocalDate fromDate, LocalDate toDate)`: `@DateTimeFormat(iso = DATE)`, `@AssertTrue isDateRangeValid()` với message "Từ ngày không được sau đến ngày".
- `MonthlyReportFilter(long employeeFilter, Mode mode, List<String> branches, long me, LocalDateTime from, LocalDateTime toExclusive)`, có `noAccess()` (BRANCHES mà rỗng) và `static of(query, caller)`.
- `MonthlyReportItem.from(Employee, branchNames, count, workDay)`: null → 0, avg làm tròn 2 chữ số HALF_UP. Họ tên `first + ' ' + last`, trim.
- `BranchNames`: refactor từ `EmployeeService`. Test cũ của `EmployeeService` phải vẫn pass.

- [ ] Test thất bại `MonthlyReportFilterTest`:
  - BGD "HN" → ALL; BGD "hn,DN" → ALL
  - BGD "DN" → BRANCHES; TPTD "HN" → BRANCHES `[HN]`; KSV "DN, HP" → BRANCHES
  - NVTD, NVKT, admin+NVKD, chức vụ null → OWN
  - BGD không có chi nhánh → `noAccess`
  - ngày null → mốc min/max; `toDate` 15/3 → `toExclusive` 16/3
- [ ] Test `MonthlyReportItemTest` (null → 0, làm tròn) và `BranchNamesTest` (nối `", "`, bỏ mã lạ, mã có khoảng trắng).
- [ ] FAIL → cài đặt → PASS → commit `feat(report): DTO và phạm vi báo cáo tháng`.

### Task 2: Truy vấn (repository)

- Specification nhân viên, như Quyết định 4.
- `MonthlyReportRepository`:
  - `List<MonthlyReportCount> countAssigned(Collection<Long> ids, LocalDateTime from, LocalDateTime toExclusive)` bằng JPQL.
  - `List<MonthlyReportWorkDay> workDays(Collection<Long> ids, from, toExclusive)` bằng native, port nguyên câu cũ:
    ```
    SELECT assignment employeeId, AVG(workDay) avgWorkDay, MAX(workDay) maxWorkDay, SUM(travel_distance) totalDistance FROM (
      SELECT ar.assignment, ap.travel_distance,
             CASE WHEN ap.status = 3 THEN IF(hs.status IS NOT NULL, DATEDIFF(NOW(), hs.status_at), NULL)
                  WHEN ap.status > 3 THEN IF(hs.status IS NOT NULL AND hs2.status IS NOT NULL, DATEDIFF(hs2.status_at, hs.status_at), NULL)
                  ELSE NULL END workDay
        FROM appraisal_profile ap
        LEFT JOIN p_appraisal_request ar ON ap.appraisal_request_id = ar.id
        LEFT JOIN appraisal_profile_history hs ON ap.id = hs.appraisal_profile_id AND hs.status = 3
        LEFT JOIN appraisal_profile_history hs2 ON ap.id = hs2.appraisal_profile_id AND hs2.status = 4
       WHERE ar.assignment IN (:ids) AND ar.assign_date >= :from AND ar.assign_date < :toExclusive AND ap.deleted = 0) t
    GROUP BY assignment
    ```
    Kiểm tra kiểu `ap.deleted` (cũ bind chuỗi `'0'`) và để ý `AVG` trả DECIMAL.

- [ ] Test thất bại `MonthlyReportRepositoryTest` (`@MysqlDataJpaTest`, dữ liệu MARK):
  - `phạm_vi_all_branches_own`: đúng nhân viên. Nhân viên "HN,DN" **không** khớp BRANCHES `[DN]` (giữ như cũ). Nhân viên đã xóa bị loại, người nghỉ việc vẫn có.
  - `lọc_cán_bộ` · `sắp_theo_id`
  - `đếm_theo_ngày_phân_công`: hồ sơ đã xóa bị loại; mốc 23:59:59 ngày cuối được tính, 00:00 ngày sau thì không.
  - `ngày_thực_hiện`: status 3 → tính tới NOW; status 4 có đủ mốc → hiệu số; thiếu mốc → NULL; status < 3 → NULL.
  - `quãng_đường_cộng_theo_hồ_sơ`: hồ sơ đã xóa (`ap.deleted`) bị loại.
  - `giữ_lỗi_nhân_dòng`: 2 dòng lịch sử status 3 → quãng đường cộng 2 lần (khẳng định là hành vi cũ).
  - `danh_sách_id_rỗng`: không chạy truy vấn, test ở service.
- [ ] FAIL → cài đặt → PASS. Không có MySQL thì ghi rõ là chưa chạy → commit `feat(report): truy vấn báo cáo tháng`.

### Task 3: Excel

`MonthlyReportExcelExporter.export(Consumer<Consumer<MonthlyReportItem>>, OutputStream)`, mẫu `CashCollectorReportExcelExporter`:
- 7 cột, độ rộng như cũ, title "BÁO CÁO THÁNG"
- số hồ sơ / max căn giữa
- km = `totalDistance / 1000.0` định dạng `#,##0.00" km"`
- trung bình định dạng `0.00`

- [ ] Test: `bố_cục_như_cũ`, `ô_số_và_định_dạng`, `nhiều_dòng_hơn_cửa_sổ_streaming` → commit `feat(report): file Excel báo cáo tháng`.

### Task 4: Service

`MonthlyReportService` dùng `CurrentEmployeeService`, `EmployeeRepository`, `MonthlyReportRepository`, `BranchNames`, `RequestFilterOptions`, exporter.
- `@Transactional(readOnly = true)` cho `search`: lấy trang nhân viên → nếu rỗng thì không gọi bước 2 và 3 → ghép kết quả theo `Map<employeeId, …>` (không lọc trong vòng lặp như code cũ).
- `export`: cùng luồng nhưng không phân trang.
- `filterOptions(callerId)`.

- [ ] Test Mockito:
  - `search_gắn_phạm_vi`
  - `noAccess_trả_rỗng_không_gọi_repo`
  - `ghép_số_liệu_theo_nhân_viên` (có người không có số liệu → 0)
  - `export_ghi_toàn_bộ`
  - `filter_options_dùng_assignees_NVTD`
- [ ] FAIL → cài đặt → PASS → commit `feat(report): service báo cáo tháng`.

### Task 5: Controller và quyền

- `GET /api/reports/monthly`: `page` mặc định 0 (`@Min(0)`), `size` mặc định 20 (1..100).
- `GET /api/reports/monthly/filter-options`.
- `GET /api/reports/monthly/export`, tên file `"Báo-cáo-tháng_" + dd-MM-yyyy_HH-mm-ss + ".xlsx"`.
- Javadoc liệt kê mục "Khác hệ thống cũ".
- `SecurityConfig` theo Quyết định 1.

- [ ] Test `@WebMvcTest`:
  - chưa đăng nhập → 401 trên cả 3 path
  - trang và filter-options: BGD/TPTD/NVTD/KSV → 200; NVKD/NVKT/NVPI/admin → 403
  - export: cả 8 authority → 200; authority lạ → 403
  - `bind_bộ_lọc`
  - tham số sai → 400: `fromDate=01/03/2026`, `from > to`, `employeeId=abc`, `size=101`, `page=-1`
  - `export_tên_file`
  - Chứng minh test quyền fail khi gỡ rule.
- [ ] `./mvnw test` toàn bộ → BUILD SUCCESS. Cập nhật `backend/CLAUDE.md` → commit `feat(report): API báo cáo tháng`.

### Task 6: Frontend lib và api

- `MONTHLY_REPORT_AUTHORITIES = ['BGD', 'TPTD', 'NVTD', 'KSV']`, `MONTHLY_REPORT_PAGE_SIZE = 20`.
- Types theo hợp đồng API.
- `listMonthlyReport`, `getMonthlyReportFilterOptions`, `exportMonthlyReport`, khuôn `api/cash-collector-reports.ts`. Bỏ tham số rỗng.
- Hàm định dạng km (mét → `1.234,56 km`) và trung bình (`1,50`): dùng helper sẵn có ở `lib/` nếu có, không thì đặt trong `lib/monthly-report.ts`.
- [ ] Spec → FAIL → cài đặt → PASS → commit `feat(report): api báo cáo tháng`.

### Task 7: Frontend màn hình, route, menu

`MonthlyReportView.vue` khuôn theo `CashCollectorReportView.vue`:
- Bộ lọc: `SearchSelect` "Cán bộ khảo sát" và `DateRangePicker` "Ngày phân công". Có nháp → áp dụng, đồng bộ URL (`employee, from, to, page`). Export dùng bộ lọc đã áp dụng.
- Bảng 7 cột theo thứ tự cũ. Cột số `text-right tabular-nums`. Trạng thái rỗng là "Không có dữ liệu".

- [ ] Spec:
  - 7 cột đúng thứ tự
  - ô chọn cán bộ nạp từ filter-options
  - Tìm kiếm → API và URL nhận đúng tham số
  - định dạng km và trung bình, giá trị 0 hiện `0`
  - export dùng bộ lọc đã áp dụng
  - trạng thái rỗng / lỗi
- [ ] Route `report/monthly-report` (name `report-monthly`, breadcrumbs `Báo cáo › Báo cáo tháng`, authorities như Task 6, comment trỏ về controller cũ) và `to` trong menu.
  - `guard.spec`: BGD/TPTD/NVTD/KSV vào được; NVKD và admin bị chặn.
  - `layout.spec`: link hiện cho NVTD; bỏ href mới khỏi selector placeholder.
- [ ] `npm run lint && npm run type-check && npm run test:unit` (`AppraisalRequestListView.spec.ts` đã fail sẵn) → commit `feat(report): màn báo cáo tháng`.

### Task 8: Kiểm tra cuối

- [ ] Chạy lại toàn bộ test BE và FE, ghi số thật.
- [ ] Nếu có môi trường: đăng nhập BGD HN, BGD ngoài HN, TPTD HN, KSV ngoài HN, NVTD. So từng cột với màn cũ cùng khoảng ngày. "Ngày thực hiện" của hồ sơ status 3 phụ thuộc NOW, nên phải so cùng ngày. Lưu ý mốc 00:00–00:01 và 23:59–24:00 có thể lệch. Mở file Excel.
- [ ] Code review bằng subagent `model: "sonnet"`.
- [ ] Báo người dùng: số test thật, phần chưa kiểm, các khác biệt cố ý.

## Global Constraints

- Subagent chỉ dùng `model: "sonnet"`.
- Dùng `git -C backend …` / `git -C frontend …`, commit trên `develop` hiện tại. Trước khi bắt đầu, kiểm tra cây làm việc sạch; không sạch thì dừng và báo.
- Theo `backend/CLAUDE.md`, `.claude/rules/jpa-spring.md`, `frontend/CLAUDE.md`, `frontend/DESIGN_SYSTEM.md`. Comment, Javadoc, message, nhãn: tiếng Việt có dấu.
- Lệnh:
  - Backend: `cd backend && ./mvnw -q test -Dtest=<Tên>`, toàn bộ bằng `./mvnw test`.
  - Frontend: `npm run lint && npm run type-check && npm run test:unit`.

## Thực thi

Backend (Task 1–5) và frontend (Task 6–7) chạy song song bằng 2 agent Sonnet, dùng chung mục "Hợp đồng API". Task 8 chạy sau cùng. Bước đầu: chép plan này vào `docs/superpowers/plans/2026-10-26-migrate-monthly-report.md` (không commit).
