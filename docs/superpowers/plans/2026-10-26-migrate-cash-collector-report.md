# Migrate "Báo cáo thu tạm ứng" (/report/cash-collector-report) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: superpowers:subagent-driven-development (hoặc executing-plans). Steps dùng checkbox (`- [ ]`).
>
> **Model cho subagent: CHỈ dùng `sonnet`.** Mọi lần gọi tool `Agent` phải truyền `model: "sonnet"`.

## Context

Mục menu "Báo cáo thu tạm ứng" trong FE mới còn là placeholder (`frontend/src/components/layout/nav-items.ts:111`, chưa có `to`). Plan này port màn `/report/cash-collector-report` cùng `/report/export-cash-collector-report` sang:
- `backend`: REST `/api/reports/cash-collector`
- `frontend`: route `/report/cash-collector-report`

Màn hiển thị các hồ sơ thẩm định có tạm ứng (`advance_fee > 0`), kèm số tiền cán bộ thu tiền đã nộp. Số đã nộp lấy từ phiếu chi PC có tài khoản nợ là tài khoản của cán bộ đó. Màn có bộ lọc và xuất Excel. Không có dòng tổng và không giới hạn năm.

Làm thẳng trên branch hiện tại (`develop` của hai submodule), giống lần làm báo cáo thu chi. Không push, không commit ở repo gốc.

**Khuôn mẫu:** đã có trên `develop`, gồm `AccountingReport*` / `SurplusReport*` (backend) và `AccountingReportView.vue` (frontend). Luồng backend: Query → Filter (chuẩn hóa + phạm vi) → repository native `@Query` với SpEL `:#{#filter.x}` → service map Row → Item → controller (`Page`, `/export` SXSSF, `/filter-options`).

**Spec = hệ thống cũ** (`td-project-legacy/src/main/java/com/jvb/thanhdo/…`):
- `controller/ReportController.java:706-730`
- `repository/impl/AppraisalRequestCustomRepositoryImpl.java:345-456` (`createCashCollectorReportQuery`)
- `entity/dto/CashCollectorReportDTO.java`, `entity/dto/request/SearchCashCollectorReportRequest.java`
- `controller/component/CashCollectorReportExporter.java`
- `resources/templates/report/cash-collector-report.html`
- `config/SecurityConfig.java:160` (rule `/report/**`), `fragment/mainnav-container.html:137` (menu không có `sec:authorize`)

## Hành vi hệ thống cũ

| Mục | Hệ thống cũ |
|---|---|
| Quyền (trang + export) | Chỉ rule chung `/report/**`: `admin, BGD, NVKT, NVKD, NVTD, TPTD, NVPI`. KSV vào được vì có authority `TPTD`. Menu không giới hạn, tức hiện cho mọi người thấy nhóm Báo cáo. Bộ này trùng với `[...ALL_STAFF, 'admin']` của FE mới. |
| Phạm vi | `isGlobalAdmin` (BGD/NVKT/TPTD/KSV có "HN") → tất cả. Chức vụ `BGD` → `ar.linked_branch IN (chi nhánh)`. Còn lại, **kể cả role admin và TPTD ngoài HN** → `ar.assignment = me OR ar.business_source_id = me OR ar.employee_id = me`. Chức vụ null → NPE. Khác `MsbRequestScope.of`: TPTD và người có quyền phân công ngoài HN **không** được xem theo chi nhánh. |
| Điều kiện chung | `ar.delete_flag != 1`, `ar.advance_fee IS NOT NULL AND > 0` |
| Từ khóa | `LIKE '%kw%'` trên `ar.contract_id`, `ar.cust_name`, `ar.company_name`. Không escape. |
| Nguồn | `ar.source = :source`. Ô chọn liệt kê mọi `source_bank` chưa xóa, nhãn là mã. |
| Người phụ trách | `ar.cash_collector_employee_id = :id`. Ô chọn liệt kê **mọi** nhân viên chưa xóa (`findByDeleteFlag`), nhãn `first + ' ' + last`. |
| Ngày nhận | `DATE(ar.create_date)` nằm trong `[from, to]`, mỗi đầu có thể bỏ trống. |
| Sắp xếp / trang | `ORDER BY ar.create_date DESC`, 20 dòng/trang |
| Chi tiết hạch toán | `LEFT JOIN credit_account ca ON ca.employee_id = ar.cash_collector_employee_id AND ca.deleted = 0`. Sau đó LEFT JOIN một subquery gom `accounting` có `accounting_mode = 'PC'`, `deleted != 1`, `contract_id` không null, GROUP BY `(contract_id, debit_account)`, JSON_ARRAYAGG `{entry_id, amount_of_money, DATE(accounting_date)}`, khớp `contract_id` và `debit_account = ca.id`. |
| Cột (11) | STT · Số hợp đồng · Tên khách hàng (`request_type = 1` → `cust_name`, ngược lại `company_name`) · Nguồn · Cán bộ đầu mối (`ar.business_source`) · Số tiền tạm ứng · Số tiền đã nộp (tổng amount chi tiết, chỉ hiện khi > 0) · Tên người phụ trách (nhận tiền) (`CONCAT_WS(' ', first, last)`) · Ngày nhận tiền (`ar.create_date`, `dd/MM/yyyy`) · Ngày hoàn tiền (danh sách ngày của từng chi tiết) · Số phiếu hạch toán (danh sách `entry_id`) |
| Excel | `ExportCommon` (XSSF): sheet "Report", tiêu đề "BÁO CÁO THU TẠM ỨNG" ở B1. Header dòng 3 từ cột B. Độ rộng 7/30/30/15/25/20/20/25/15/15/35. Tiền `#,##0`. Hai cột cuối là các giá trị nối nhau bằng `\n`, ngày `dd/MM/yyyy`. Tên file `Báo-cáo-thu-tạm-ứng-tiền-mặt_<dd-MM-yyyy_HH:mm:ss>.xlsx`. Kiểm tra `ExportCommon` cũ để biết có gộp tiêu đề và có autofilter không, rồi làm y như vậy. |

Lỗi hệ thống cũ sẽ sửa (ghi vào Javadoc controller, mục "Khác hệ thống cũ"):
1. Cán bộ thu tiền có từ 2 `credit_account` chưa xóa thì hồ sơ bị **nhân đôi dòng**, mỗi dòng chỉ có phần chi tiết của một tài khoản. Bản mới gộp mọi tài khoản của cán bộ đó vào **một dòng**.
2. Subquery GROUP BY toàn bảng `accounting` được thay bằng subquery tương quan theo `ar.contract_id`. Kết quả giữ nguyên, truy vấn nhanh hơn.
3. Thứ tự chi tiết của JSON_ARRAYAGG không xác định. Bản mới sắp theo `accounting_date ASC, entry_id ASC` ở Java.
4. Trang hiện "Ngày hoàn tiền" dạng ISO `yyyy-MM-dd` còn Excel là `dd/MM/yyyy`. Bản mới dùng `dd/MM/yyyy` ở cả hai.
5. Từ khóa escape đủ `\ % _` (`LikePattern.contains`). Ngày sai định dạng hoặc `from > to` trả 400 thay vì 500. `page < 0` trả 400.
6. Chức vụ null không gây NPE mà rơi vào phạm vi "của mình".
7. Tên file encode UTF-8, giờ dạng `HH-mm-ss`. Excel ghi theo streaming (SXSSF) và dùng bộ lọc đã áp dụng trên FE.
8. Thêm `ar.id DESC` vào sau `create_date DESC` để thứ tự giữa các trang ổn định.

## Quyết định

1. **Quyền**: giữ nguyên hệ thống cũ, và ở màn này cũng không có lỗ hổng nào giữa trang và export.
   - Backend: `/api/reports/cash-collector`, `/api/reports/cash-collector/**` cho `hasAnyAuthority("admin", BGD, NVKT, NVKD, NVTD, TPTD, KSV, NVPI)`. Dùng hằng `Positions`. Comment: "Như hệ thống cũ: chỉ rule chung /report/**".
   - FE: route cho `['admin', ...ALL_STAFF]`. Menu giữ không có `authorities`, thừa kế nhóm Báo cáo, giống cũ. Chỉ thêm `to`.
2. **Phạm vi**: giữ đúng quy tắc riêng của màn này. Thêm factory `MsbRequestScope.ofCashCollectorReport(Employee)` vào `backend/.../model/dto/MsbRequestScope.java`, tái dùng `Mode` và `branchCodes`:
   - `isGlobalAdmin` → `ALL`
   - `BGD` → `BRANCHES`
   - còn lại → `OWN`

   Javadoc ghi rõ khác `of` ở chỗ TPTD/canAssign ngoài HN bị đẩy xuống OWN, và role admin không được miễn. Repository viết điều kiện trong SQL tĩnh, theo cùng nghĩa với `RequestScopeSql.condition`: `filter.scopeAll`, `filter.scopeBranches` + `sqlBranches`, `filter.scopeOwn` + `employeeId`. `isEmpty()` → `noAccess` → trả rỗng mà không chạy truy vấn.
3. **Không giới hạn năm**, như cũ. Khoảng ngày lấy từ `ReportPeriod.ofLimitedYear(null).narrow(fromDate, toDate)` (`model/dto/ReportPeriod.java`) trên `ar.create_date`, dạng `[from, toExclusive)`. Kết quả tương đương `DATE()` của cũ.
4. **Chi tiết hạch toán**: dùng subquery tương quan trả JSON. Native là bắt buộc vì JSON_ARRAYAGG/JSON_OBJECT là hàm riêng của MySQL; lý do này ghi trên repository.
   ```
   (SELECT JSON_ARRAYAGG(JSON_OBJECT('entryId', ac.entry_id, 'amount', ac.amount_of_money,
                                     'date', DATE(ac.accounting_date)))
      FROM accounting ac
     WHERE ac.accounting_mode = 'PC' AND ac.deleted <> 1
       AND ac.contract_id = ar.contract_id COLLATE …
       AND ac.debit_account IN (SELECT ca.id FROM credit_account ca
                                WHERE ca.employee_id = ar.cash_collector_employee_id AND ca.deleted = 0)) accountingDetail
   ```
   Item parse JSON bằng `ObjectMapper` (bean Spring, inject vào service) thành `List<Detail(entryId, amount, LocalDate date)>`, sắp ổn định, rồi tính `paidAmount` (null khi tổng ≤ 0 hoặc không có chi tiết). Hằng `'PC'` lấy từ `Accounting.MODE_PAYMENT` qua SpEL `T(...)` như `AccountingReportSql`. Kiểm tra collation của `accounting.contract_id` và `p_appraisal_request.contract_id` theo cách `AccountingReportSql` đã viết.
5. **Ô lọc** (`/filter-options`):
   - `sources`: tái dùng `ValuationReportFilterOptions.Source` + `SourceBankRepository.findByDeleteFlagOrderByBankCode`, nhãn `bankName - bankCode` như `ContractTrackingReportService.filterOptions:50`. Cũ chỉ hiện mã; đổi để thống nhất với các báo cáo mới và ghi vào mục khác biệt.
   - `employees`: `EmployeeRepository.findByDeleteFlag(ACTIVE)` (mọi nhân viên chưa xóa, như cũ), dùng `Option(id, "first last" đã trim)`, sắp theo tên bằng `Collator` vi-VN. Tái dùng kiểu `Option` của `RequestFilterOptions`.

## Global Constraints

- Subagent chỉ dùng `model: "sonnet"`.
- Dùng `git -C backend …` / `git -C frontend …`, commit trên `develop` hiện tại. Trước khi bắt đầu, kiểm tra cây làm việc sạch; không sạch thì dừng và báo.
- Theo `backend/CLAUDE.md`, `.claude/rules/jpa-spring.md`, `frontend/CLAUDE.md`, `frontend/DESIGN_SYSTEM.md`. Comment, Javadoc, message, nhãn: tiếng Việt có dấu.
- API base `/api/reports/cash-collector`. Route FE: path `report/cash-collector-report`, name `report-cash-collector`, breadcrumbs `[{ label: 'Báo cáo' }, { label: 'Báo cáo thu tạm ứng' }]`.
- Lệnh:
  - Backend: `cd backend && ./mvnw -q test -Dtest=<Tên>`, toàn bộ bằng `./mvnw test`. Test repository cần MySQL.
  - Frontend: `npm run lint && npm run type-check && npm run test:unit`. Đã biết `AppraisalRequestListView.spec.ts` fail sẵn.

## Hợp đồng API (FE và BE làm song song)

- `GET /api/reports/cash-collector?keyword&source&cashCollectorEmployeeId&fromDate&toDate&page&size` (page từ 0, size mặc định 20, tối đa 100) → `Page<CashCollectorReportItem>`
- `CashCollectorReportItem`: `contractId, customerName, source, businessSource, advanceFee (Long|null), paidAmount (Long|null), cashCollectorName, receivedDate (LocalDate, từ create_date), details: [{ entryId, amount, accountingDate (LocalDate) }]`
- `GET /filter-options` → `{ sources: [{ value, label }], employees: [{ value, label }] }`
- `GET /export` (cùng tham số lọc, không có page) → file xlsx, `filename*=UTF-8''B%C3%A1o-c%C3%A1o-thu-t%E1%BA%A1m-%E1%BB%A9ng_…`

---

## File Structure

**Backend** (`backend/src/main/java/com/thanhdo/tdprojectbe/`):

| File | Trách nhiệm |
|---|---|
| Modify `model/dto/MsbRequestScope.java` | Factory `ofCashCollectorReport` |
| Create `model/dto/CashCollectorReport{Query,Filter,Row,Item,FilterOptions}.java` | DTO; `Item` có record lồng `Detail` |
| Create `repository/CashCollectorReportSql.java`, `repository/CashCollectorReportRepository.java` | SQL; `search` / `stream` |
| Create `common/util/export/excel/CashCollectorReportExcelExporter.java` | Excel (lớp con `StreamingExportCommon`) |
| Create `service/CashCollectorReportService.java` | Phạm vi, parse chi tiết, export, filter-options |
| Create `controller/CashCollectorReportController.java` | API |
| Modify `config/SecurityConfig.java` | Quyền, đặt sau khối accounting |
| Modify `backend/CLAUDE.md` | Thêm exporter vào dòng streaming; thêm `ofCashCollectorReport` vào dòng phạm vi |

**Frontend** (`frontend/src/`):
- Create: `lib/cash-collector-report.ts`, `api/cash-collector-reports.ts`, `views/report/CashCollectorReportView.vue`, kèm spec.
- Modify: `router/index.ts`, `components/layout/nav-items.ts:111` (thêm `to`), `router/__tests__/guard.spec.ts`, `components/layout/__tests__/layout.spec.ts`.

---

### Task 1: Phạm vi và DTO (backend)

- **`CashCollectorReportQuery(String keyword, String source, Long cashCollectorEmployeeId, LocalDate fromDate, LocalDate toDate)`**
  - `@DateTimeFormat(iso = DATE)` cho ngày.
  - `keyword` có `@Size(max = 500)`, null → `""`. `source` null → `""`.
  - `@AssertTrue isDateRangeValid()`, message "Từ ngày không được sau đến ngày".
- **`CashCollectorReportFilter(String keyword, String source, long cashCollectorEmployeeId, MsbRequestScope scope, LocalDateTime from, LocalDateTime toExclusive)`**
  - Tham số null đổi sang `""`/`0`.
  - Có các accessor cho SpEL: `scopeAll()`, `scopeBranches()`, `sqlBranches()` (`["(none)"]` khi rỗng), `employeeId()`, `noAccess()`.
  - `static of(query, caller)`.
- **`CashCollectorReportRow`** (projection): `getId, getContractId, getCustomerName, getSource, getBusinessSource, getAdvanceFee (Long), getCashCollectorName, getCreateDate (LocalDateTime), getAccountingDetail (String)`.
- **`CashCollectorReportItem`** + `Detail`. `static from(row, List<Detail> details)` tính `paidAmount`.
- **`CashCollectorReportFilterOptions(List<Source> sources, List<Option> employees)`**.

- [ ] **Test thất bại** `MsbRequestScopeTest` (thêm vào nếu file đã có, nếu chưa thì tạo):
  - BGD có HN → ALL; KSV có HN → ALL
  - BGD "DN, HP" → BRANCHES `[DN, HP]`
  - TPTD "DN" → OWN; NVKT ngoài HN → OWN
  - role admin + NVKD → OWN; chức vụ null → OWN
  - BGD không có chi nhánh → `isEmpty`
- [ ] **Test** `CashCollectorReportFilterTest`:
  - escape từ khóa
  - null → `""`/`0`
  - ngày: null/null → min/max của `ReportPeriod`; `2026-03-15` → toExclusive `2026-03-16`
- [ ] **Test** `CashCollectorReportItemTest`:
  - `paidAmount` = tổng
  - tổng bằng 0 hoặc danh sách rỗng → null
- [ ] FAIL → cài đặt → PASS → commit `feat(report): DTO và phạm vi báo cáo thu tạm ứng`.

### Task 2: SQL và repository

`CashCollectorReportRepository extends Repository<AppraisalRequest, Long>`, khuôn theo `AccountingReportRepository`:
- default `search(filter, pageable)` và `stream(filter)`; `noAccess` → `Page.empty` / `Stream.empty`
- native `findPage` (có `countQuery`) và `streamAll` (hint fetchSize `Integer.MIN_VALUE`)

```
FROM_WHERE = FROM p_appraisal_request ar
  LEFT JOIN employees em ON em.employee_id = ar.cash_collector_employee_id
 WHERE ar.delete_flag <> '1' AND ar.advance_fee > 0
   AND ar.create_date >= :from AND ar.create_date < :toExclusive
   AND (:keyword = '' OR ar.contract_id LIKE :keyword OR ar.cust_name LIKE :keyword OR ar.company_name LIKE :keyword)
   AND (:source = '' OR ar.source = :source)
   AND (:employeeFilter = 0 OR ar.cash_collector_employee_id = :employeeFilter)
   AND (:scopeAll = TRUE
        OR (:scopeBranches = TRUE AND ar.linked_branch IN (:sqlBranches))
        OR (:scopeOwn = TRUE AND (ar.assignment = :me OR ar.business_source_id = :me OR ar.employee_id = :me)))
COLUMNS = ar.id, ar.contract_id, CASE WHEN ar.request_type = '1' THEN ar.cust_name ELSE ar.company_name END customerName,
   ar.source, ar.business_source, ar.advance_fee, ar.create_date,
   NULLIF(TRIM(CONCAT_WS(' ', em.first_name, em.last_name)), '') cashCollectorName, <subquery JSON ở Quyết định 4>
ORDER BY ar.create_date DESC, ar.id DESC
```

Kiểm tra trước khi viết:
- Kiểu cột `delete_flag` của `p_appraisal_request` và `request_type`: so sánh dạng chuỗi hay số (xem entity và các query `AppraisalRequest` đã có).
- Cột `accounting.deleted`.
- `advance_fee > 0` đã tự loại NULL.
- `employees` JOIN theo id số nên không cần COLLATE.

- [ ] **Test thất bại** `CashCollectorReportRepositoryTest` (`@MysqlDataJpaTest`, dữ liệu MARK qua `RepositoryTestData`):
  - `chỉ_hồ_sơ_có_tạm_ứng`: `advance_fee` NULL, 0, đã xóa → loại
  - `tên_khách_theo_loại`: cá nhân → `cust_name`; doanh nghiệp → `company_name`
  - `từ_khóa_ba_cột` · `lọc_nguồn` · `lọc_người_phụ_trách`
  - `mốc_ngày_trọn_ngày`: 23:59:59 của ngày cuối được tính, 00:00 của ngày sau thì không
  - `sắp_xếp_ngày_tạo_giảm_dần`
  - `chi_tiết_chỉ_PC_từ_tài_khoản_cán_bộ`. Chỉ phiếu PC chưa xóa, cùng hợp đồng, có `debit_account` là tài khoản chưa xóa của cán bộ mới được tính. Loại: phiếu PT; PC từ tài khoản khác; PC đã xóa; tài khoản đã xóa.
  - `hai_tài_khoản_không_nhân_đôi_dòng`: cán bộ có 2 tài khoản, mỗi tài khoản 1 PC → 1 dòng, 2 chi tiết
  - `phạm_vi_all_branches_own`: OWN thấy hồ sơ có `assignment`, `business_source_id` hoặc `employee_id` là mình, và không thấy hồ sơ khác
  - `noAccess_không_chạy_truy_vấn`
  - `stream_trả_cùng_các_dòng_như_trang` (trong transaction)
- [ ] FAIL → cài đặt (Javadoc ghi lý do native và quy tắc cũ) → PASS. Không có MySQL thì ghi rõ là chưa chạy, không tính là pass → commit `feat(report): truy vấn báo cáo thu tạm ứng`.

### Task 3: Excel

`CashCollectorReportExcelExporter.export(Consumer<Consumer<CashCollectorReportItem>> dataSource, OutputStream out)`. Mẫu là `AccountingReportExcelExporter`.
- `TITLE = "BÁO CÁO THU TẠM ỨNG"`, 11 cột, độ rộng như cũ.
- Hai cột cuối nối bằng `\n`, dùng style wrap.
- Tiền `styleRowPrice`. Đã nộp để trống khi null.

- [ ] Test thất bại: `bố_cục_như_cũ`, `chi_tiết_nhiều_dòng_nối_xuống_dòng`, `ô_tiền_trống_khi_null`, `nhiều_dòng_hơn_cửa_sổ_streaming` (300 dòng).
- [ ] FAIL → cài đặt → PASS → commit `feat(report): file Excel báo cáo thu tạm ứng`.

### Task 4: Service

`CashCollectorReportService` dùng repository, exporter, `CurrentEmployeeService.require`, `ObjectMapper`, `SourceBankRepository`, `EmployeeRepository`.
- `Page<Item> search(callerId, query, pageable)`
- `@Transactional(readOnly = true) export(callerId, query, out)` với `try (Stream<…>)`
- `filterOptions()`
- Hàm private `details(String json)`:
  - null hoặc rỗng → `List.of()`
  - parse rồi sắp `accountingDate` (null cuối), sau đó `entryId`
  - JSON hỏng → ném `IllegalStateException`, không nuốt lỗi như cũ

- [ ] Test Mockito:
  - `search_gắn_phạm_vi` (captor filter, BGD → BRANCHES)
  - `noAccess_trả_rỗng`
  - `parse_và_sắp_chi_tiết`
  - `export_ghi_file_từ_stream_và_đóng_stream`
  - `filter_options_nhân_viên_sắp_theo_tên`
- [ ] FAIL → cài đặt → PASS → commit `feat(report): service báo cáo thu tạm ứng`.

### Task 5: Controller và quyền

- `GET /api/reports/cash-collector`: `page` mặc định 0 (`@Min(0)`), `size` mặc định 20 (1..100), `@Valid @ModelAttribute CashCollectorReportQuery`.
- `GET /filter-options`.
- `GET /export` → tên file `"Báo-cáo-thu-tạm-ứng_" + dd-MM-yyyy_HH-mm-ss + ".xlsx"`, encode UTF-8 như `AccountingReportController`.
- Javadoc liệt kê mục "Khác hệ thống cũ" 1–8 và quyết định 5.
- `SecurityConfig`: một matcher cho cả `/api/reports/cash-collector` và `/**`.

- [ ] Test `@WebMvcTest`:
  - chưa đăng nhập → 401 trên cả 3 path
  - mỗi authority trong danh sách → 200; authority lạ → 403
  - `bind_bộ_lọc`: captor nhận đủ tham số
  - tham số sai → 400: `fromDate=01/03/2026`, `from > to`, `cashCollectorEmployeeId=abc`, `size=101`, `page=-1`
  - `export_tên_file`
- [ ] `./mvnw test` toàn bộ → BUILD SUCCESS. Cập nhật `backend/CLAUDE.md` → commit `feat(report): API báo cáo thu tạm ứng`.

### Task 6: Frontend lib và api

- `CASH_COLLECTOR_REPORT_AUTHORITIES = ['admin', 'BGD', 'NVKT', 'NVKD', 'NVTD', 'TPTD', 'KSV', 'NVPI']` và `CASH_COLLECTOR_REPORT_PAGE_SIZE = 20`.
- Types theo hợp đồng API.
- `listCashCollectorReport`, `getCashCollectorReportFilterOptions`, `exportCashCollectorReport`, khuôn theo `api/accounting-reports.ts`. Bỏ tham số rỗng.
- [ ] Spec (URL, bỏ tham số rỗng) → FAIL → cài đặt → PASS → commit `feat(report): api báo cáo thu tạm ứng`.

### Task 7: Frontend màn hình, route, menu

`CashCollectorReportView.vue` sao chép `AccountingReportView.vue`, bỏ phần dòng tổng và phần thông báo năm. Bộ lọc nháp → áp dụng, đồng bộ URL, export dùng bộ lọc đã áp dụng. Bộ lọc gồm:
- từ khóa, placeholder "Mã hồ sơ, tên khách hàng"
- `SearchSelect` "Người phụ trách" (danh sách dài, cần tìm kiếm)
- `SelectBox` "Nguồn"
- `DateRangePicker` "Ngày nhận"

Bảng 11 cột theo thứ tự cũ:
- Tiền `text-right tabular-nums` bằng `formatMoney`.
- Ngày `text-center`, định dạng `dd/MM/yyyy`.
- "Ngày hoàn tiền" và "Số phiếu hạch toán" hiện mỗi chi tiết một dòng, hai cột thẳng hàng theo thứ tự.

- [ ] Spec:
  - 11 cột đúng thứ tự
  - tải filter-options vào hai ô chọn
  - Tìm kiếm → API và URL nhận đúng tham số
  - chi tiết nhiều dòng
  - "Đã nộp" trống khi null
  - export dùng bộ lọc đã áp dụng
  - trạng thái rỗng / lỗi
- [ ] Route (comment "Màn /report/cash-collector-report của hệ thống cũ: chỉ rule chung /report/**") và `to` trong menu.
  - `guard.spec`: KSV, NVPI, admin vào được; tài khoản không có authority nào trong danh sách bị chặn.
  - `layout.spec`: link hiện cho NVKD; bỏ href mới khỏi selector placeholder.
- [ ] `npm run lint && npm run type-check && npm run test:unit` → commit `feat(report): màn báo cáo thu tạm ứng`.

### Task 8: Kiểm tra cuối

- [ ] Chạy lại toàn bộ test BE và FE, ghi số thật.
- [ ] Nếu có môi trường: đăng nhập BGD ngoài HN, TPTD ngoài HN (phải bị OWN) và BGD HN. So số dòng và cột "Đã nộp" với màn cũ cùng bộ lọc. Lưu ý mục khác biệt 1: cán bộ có 2 tài khoản thì bên cũ ra nhiều dòng hơn. Mở file Excel.
- [ ] Code review bằng subagent `model: "sonnet"`.
- [ ] Báo người dùng: số test thật, phần chưa kiểm, các khác biệt cố ý.

## Thực thi

Backend (Task 1–5) và frontend (Task 6–7) chạy song song bằng 2 agent Sonnet, dùng chung mục "Hợp đồng API". Task 8 chạy sau cùng. Bước đầu: chép plan này vào `docs/superpowers/plans/2026-10-26-migrate-cash-collector-report.md` (không commit).
