# Migrate "Báo cáo thu chi" (/report/accounting) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **Model cho subagent: CHỈ dùng `sonnet`.** Mọi lần gọi tool `Agent` (implementer, spec reviewer, code reviewer, final reviewer) phải truyền `model: "sonnet"`.

## Context

Màn "Báo cáo thu chi" (`/report/accounting`, `/report/export-accounting`) còn là placeholder trong menu FE mới (`frontend/src/components/layout/nav-items.ts:105`, chưa có `to`). Plan này port nó sang `backend` (REST `/api/reports/accounting`) và `frontend` (route `/report/accounting`). Màn hiển thị danh sách bút toán, có bộ lọc từ khóa, nhiều tài khoản nguồn, hóa đơn và ngày lập. Kèm theo là dòng tổng thu/chi và xuất Excel.

**Người dùng đã chốt:** giữ **y nguyên** phân quyền hệ thống cũ, **kể cả lỗ hổng**. Export mở cho nhiều chức vụ hơn trang, và những chức vụ này xuất được toàn bộ dữ liệu vì hệ thống cũ chỉ giới hạn chi nhánh cho BGD/NVKT.

**Bước đầu khi thực thi:** chép plan này vào `docs/superpowers/plans/2026-10-26-migrate-accounting-report.md` (repo gốc, không commit trừ khi người dùng yêu cầu).

**Architecture:** Đi theo khuôn `SurplusReport*` / `ContractTrackingReport*` (đã có trên `develop` của `backend`/`frontend`).
- Backend: Query → Filter (chuẩn hóa, phạm vi, giới hạn năm) → repository native SQL. Trang, count, tổng và stream dùng chung một `WHERE`. Service map Row → Item. Controller có `Page`, `/totals`, `/export` (SXSSF) và `/filter-options` (tài khoản nguồn + `limitedYear`).
- Frontend: sao chép `SurplusReportView.vue`. Ô tài khoản dùng `MultiSelect`, ô ngày dùng `DateRangePicker` như `AccountingListView.vue:369`. Có thông báo giới hạn năm như `ContractTrackingReportView.vue` (~315).

**Tech Stack:** Spring Boot 4 · Java 21 · Spring Data JPA native query + interface projection · Apache POI SXSSF · JUnit 5/Mockito/AssertJ · `@MysqlDataJpaTest` · Vue 3 + TS · shadcn-vue · Vitest.

**Spec = hành vi hệ thống cũ** (`td-project-legacy/src/main/java/com/jvb/thanhdo/…`):
- `controller/ReportController.java` — `accountingReport` (353-380), `exportAccountingToExcel` (381-393)
- `service/impl/ReportServiceImpl.java`:
  - `getAccountingReport` (618), `getAccountingReportTotalMoney` (659), `getAccountingReportExport` (823)
  - `fillBillId` (595), bộ lọc tài khoản (576-591)
  - `initBaseSearchReportRequest` (~1228), `getReportYearRange` (~1267), `checkRole` (1552)
- `repository/AccountingRepository.java` — 6 truy vấn (222, 239, 258, 300, 355, 373); `JuridicalRepository.findBillIdByContractIds`
- `service/impl/CreditAccountServiceImpl.java:184` `getListCreditAccountByUserBranch`; `entity/CreditAccount.java:93` `getDisplayName`
- `controller/component/AccountingExcelExporter.java`, `entity/dto/request/BaseSearchReportRequest.java`
- `resources/templates/report/accounting.html`, `fragment/report-year-notice.html`, `fragment/mainnav-container.html:130`, `config/SecurityConfig.java:160`

---

## Hành vi hệ thống cũ

| Mục | Hệ thống cũ |
|---|---|
| Quyền trang | `role = admin` **hoặc** chức vụ `BGD`/`NVKT` (chức vụ null → NPE). Menu: `BGD`, `NVKT` (admin vào được bằng URL). |
| Quyền export | Không kiểm tra trong controller; chỉ rule `/report/**`: `admin, BGD, NVKT, NVKD, NVTD, TPTD, NVPI` (KSV lọt qua vì có authority `TPTD`). |
| Phạm vi | Chỉ khi **không phải admin**, chức vụ là **BGD hoặc NVKT** và **không có chi nhánh HN** thì mới bị giới hạn. Mọi trường hợp khác (admin, có HN, và các chức vụ khác khi export) xem **toàn bộ** dữ liệu. Khi bị giới hạn, có thêm hai điều kiện: `ar.linked_branch IN (chi nhánh người xem)`, qua JOIN `p_appraisal_request` theo `contract_id`, nên bút toán không gắn hồ sơ bị ẩn; và `ap.subsection` khác `'hoa hồng HO'` và `'hoa hồng hội sở'`, nên tiểu mục NULL cũng bị ẩn. |
| Điều kiện chung | `ap.deleted = 0`; `ap.type IN (1, 3)`. Giới hạn năm: người không có role `ADMIN_REPORT` chỉ xem bút toán có `accounting_date` trong năm hiện tại. |
| Từ khóa | `LIKE '%kw%'` trên `entry_id`, `cust_name`, `contract_id` (chỉ escape `%`). |
| Tài khoản nguồn (lọc) | Nhiều id. Khớp `credit_account IN ids OR debit_account IN ids`, không xét PT/PC. Không chọn thì không lọc. Id không bị kiểm tra phạm vi. |
| Hóa đơn | `1` = hợp đồng có hồ sơ pháp lý (`juridical`) chưa xóa với `bill_id` khác NULL/rỗng; `0` = không có; trống = tất cả. |
| Ngày lập | Lọc theo `accounting_date`. Có ngày thì từ `from 00:01` đến `to 23:59`. Không có ngày thì không giới hạn (ngoài giới hạn năm). |
| Ô tài khoản nguồn | Tài khoản `employee_id IS NULL`, chưa xóa. HN thấy tất cả, còn lại chỉ thấy tài khoản có `branch IN (chi nhánh)`. Sắp tài khoản công ty (`type = 2`) trước, rồi theo mã. Hiển thị mã tài khoản. |
| Sắp xếp / trang | `accounting_date DESC, created DESC`; 20 dòng/trang. |
| Cột (12) | STT · Ngày lập (`dd/MM/yyyy`) · Tên khách hàng · Số hợp đồng · Số hóa đơn · Số tiền thu (PT) · Số tiền chi (PC) · Tiểu mục · Nội dung · Tài khoản nguồn · Ghi chú · Số bút toán. |
| Số hóa đơn | `bill_id` của `juridical` chưa xóa theo `contract_id`. Nhiều dòng thì lấy ngẫu nhiên. |
| Tài khoản nguồn (hiển thị) | PT: tài khoản `credit_account`. PC: tài khoản `debit_account`. Hiển thị họ tên nhân viên nếu có, ngược lại là mã tài khoản. |
| Dòng tổng | Chỉ ở trang cuối, tính trên **toàn bộ** kết quả: "Tổng cộng:" · tổng PT · tổng PC. |
| Năm | Dòng đỏ "Báo cáo sẽ chỉ hiển thị dữ liệu được tạo trên hệ thống trong năm <năm>." cho người không có `ADMIN_REPORT`. |
| Excel | Sheet "Report". Tiêu đề "BÁO CÁO THU CHI" ở B1, gộp B1:M1. Header dòng 3 từ cột B, có autofilter. 12 cột, độ rộng 7/15/25/30/20/15/15/25/25/15/30/40. Tiền định dạng `#,##0`. Ngày là chuỗi `MM/dd/yyyy`. Không có dòng tổng. Lọc như trang, xuất mọi dòng. Tên file `Báo-cáo-thu-chi_<dd-MM-yyyy_HH:mm:ss>.xlsx`. |

Lỗi hệ thống cũ sẽ được sửa (ghi vào Javadoc controller):
- `!= ""` so sánh tham chiếu; ngày sai định dạng gây 500.
- Khoảng giờ 00:01–23:59 làm mất phút đầu ngày và 59 giây cuối ngày.
- Chỉ escape `%` mà không escape `_`.
- `pageNo < 1` gây 500.
- Excel ghi ngày dạng `MM/dd/yyyy` (khác trang) và tên file không được encode.
- Số hóa đơn có thể khác nhau giữa các lần tải.
- LEFT JOIN `p_appraisal_request` có thể nhân đôi dòng khi trùng `contract_id`.
- Chức vụ null gây NPE.

## Quyết định đã chọn

1. **Quyền** (người dùng chốt: giữ y nguyên kể cả lỗ hổng):
   - `/api/reports/accounting/export` → `admin, BGD, NVKT, NVKD, NVTD, TPTD, KSV, NVPI`.
   - `/api/reports/accounting` và `/api/reports/accounting/**` còn lại (danh sách, `/totals`, `/filter-options`) → `admin, BGD, NVKT`.
   - Matcher của export đặt **trước** matcher `/**`. Comment tiếng Việt ghi "giữ nguyên lỗ hổng theo yêu cầu".
   - Route FE: `['admin', 'BGD', 'NVKT']`. Menu giữ `['BGD', 'NVKT']` như cũ. Kiểm tra cách route guard nhận `admin` (role hay authority) theo các route có admin sẵn, ví dụ contract-tracking.
2. **Phạm vi** đúng như cũ:
   - `restricted = !AssignAuth.isAdmin(caller) && chức vụ ∈ {BGD, NVKT} && chi nhánh không có "HN"`.
   - Chi nhánh lấy bằng `MsbRequestScope.branchCodes`, so `"HN"` không phân biệt hoa thường.
   - Không dùng `isGlobalAdmin`, vì quy tắc khác.
   - Bị giới hạn mà không có chi nhánh nào → `noAccess` (trả rỗng; cũ gây NPE/rỗng).
   - Điều kiện chi nhánh viết bằng `EXISTS` thay cho JOIN, để không nhân đôi dòng.
3. **Giới hạn năm**: dùng `ReportYearLimit.limitedYear` + `ReportPeriod.ofLimitedYear(y).narrow(fromDate, toDate)` trên `accounting_date`. Mốc là trọn ngày `[from, toExclusive)`.
4. **Số hóa đơn**: subquery `MAX(bill_id)` để kết quả cố định. Lọc hóa đơn dùng `EXISTS` trên `juridical`.
5. **Ô tài khoản nguồn**: endpoint `/filter-options` tái dùng `AccountingService.sharedAccounts(branches, headOffice)` (`backend/.../service/AccountingService.java:196`). Sắp công ty trước (sort ổn định), nhãn là mã tài khoản, trả cả `limitedYear`.
6. **Excel** sửa định dạng ngày thành `dd/MM/yyyy` (chuỗi, giống trang). Lọc theo bộ lọc đã áp dụng trên FE, không phải giá trị đang gõ trong form.

## Global Constraints

- Subagent chỉ dùng `model: "sonnet"`.
- `backend/`, `frontend/` là git submodule. Cả hai đang ở `develop` (đã có commit merge báo cáo số dư, chưa push).
  - Dùng `git -C backend …` / `git -C frontend …`.
  - Không commit ở repo gốc, không push.
- Backend theo `backend/CLAUDE.md` và `.claude/rules/jpa-spring.md`. Frontend theo `frontend/CLAUDE.md` và `frontend/DESIGN_SYSTEM.md`.
- Comment, Javadoc, message lỗi, nhãn UI: tiếng Việt có dấu.
- API base `/api/reports/accounting`. Route FE: path `report/accounting`, name `report-accounting`, breadcrumbs `[{ label: 'Báo cáo' }, { label: 'Báo cáo thu chi' }]`.
- Native SQL được phép, và phải ghi lý do trên repository. Các lý do:
  - JOIN/EXISTS chuỗi giữa `accounting`, `juridical`, `p_appraisal_request` và `credit_account`/`employees` cần `COLLATE` (schema lệch collation), mà JPQL không biểu diễn được.
  - `bill_id` chưa được map trên `Accounting`.
  - Giữ chung một `WHERE` cho trang, count, tổng và stream, như các báo cáo đã port.
- Lệnh:
  - Backend: `cd backend && ./mvnw -q test -Dtest=<Tên>`. Toàn bộ: `./mvnw test`. Test repository cần MySQL theo `backend/.env.example`.
  - Frontend: `cd frontend && npm run lint && npm run type-check && npm run test:unit`.

### Khác hệ thống cũ (cố ý) — ghi vào Javadoc controller

1. JSON trả `Page` (`page` tính từ 0, `size` mặc định 20, tối đa 100). Tổng lấy ở `/totals`, giao diện chỉ hiện ở trang cuối.
2. Ngày tính trọn ngày. `fromDate > toDate` hoặc sai định dạng → 400.
3. Từ khóa escape đủ `\ % _` (`LikePattern.contains`). Từ khóa trống thì không lọc (cũ vô tình loại dòng có cả 3 cột NULL).
4. Không nhân đôi dòng khi một `contract_id` có nhiều hồ sơ. Số hóa đơn lấy `MAX` cố định.
5. Excel ghi ngày `dd/MM/yyyy`, tên file encode UTF-8 (`dd-MM-yyyy_HH-mm-ss`), dùng bộ lọc đã áp dụng.
6. Tiểu mục hiện thẳng `accounting.subsection`. Cũ JOIN bảng `subsection` theo tên, nên sẽ trống nếu tên không còn trong bảng.

## Review Focus

1. **Phạm vi 4 nhánh**:
   - admin: xem tất cả.
   - BGD/NVKT có HN: tất cả.
   - BGD/NVKT không có HN: theo `linked_branch` của hồ sơ, loại "hoa hồng HO"/"hoa hồng hội sở"/NULL, loại bút toán không gắn hồ sơ.
   - NVKD gọi export: tất cả (lỗ hổng được giữ).
   - Test: `AccountingReportFilterTest`, `AccountingReportRepositoryTest.phạm_vi_*`.
2. **Lọc tài khoản** khớp cả `credit_account` và `debit_account`, và "Tài khoản nguồn" hiển thị đúng theo PT/PC (`lọc_tài_khoản_khớp_hai_phía`).
3. **Tổng = tổng các dòng** với cùng bộ lọc (`tổng_khớp_tổng_các_dòng`).
4. **Một hợp đồng có 2 hồ sơ / 2 juridical** → không nhân đôi dòng (`không_nhân_đôi_dòng`).
5. **Quyền 2 tầng**: NVKD gọi `/api/reports/accounting` → 403 nhưng `/export` → 200; admin → 200 cả hai; chưa đăng nhập → 401.

---

## File Structure

**Backend** (`backend/src/main/java/com/thanhdo/tdprojectbe/`):

| File | Trách nhiệm |
|---|---|
| Modify `model/entity/Subsection.java` | Hằng `HEAD_OFFICE_COMMISSION = "hoa hồng HO"`, `HEAD_OFFICE_COMMISSION_ALT = "hoa hồng hội sở"`. Kiểm tra trước, tránh trùng hằng đã có. |
| Create `model/dto/AccountingReport{Query,Filter,Row,Item,TotalsRow,Totals,FilterOptions}.java` | DTO. |
| Create `repository/AccountingReportSql.java`, `repository/AccountingReportRepository.java` | SQL; trang / stream / tổng. |
| Create `common/util/export/excel/AccountingReportExcelExporter.java` | File Excel. |
| Create `service/AccountingReportService.java` | Phạm vi, năm, map, export, filter-options. |
| Create `controller/AccountingReportController.java` | API. |
| Modify `config/SecurityConfig.java` | Quyền. |
| Modify `backend/CLAUDE.md` | Thêm exporter vào dòng streaming. |

**Frontend** (`frontend/src/`):
- Create: `lib/accounting-report.ts`, `api/accounting-reports.ts`, `views/report/AccountingReportView.vue`, kèm spec.
- Modify: `router/index.ts`, `components/layout/nav-items.ts:105`, `router/__tests__/guard.spec.ts`, `components/layout/__tests__/layout.spec.ts` (gồm cả selector `:not([href=…])` ~396).

---

### Task 1: DTO và quy tắc phạm vi

**Files:**
- Modify: `Subsection.java`
- Create: DTO
- Test: `model/dto/AccountingReportFilterTest.java`, `model/dto/AccountingReportTotalsTest.java`

**Interfaces:**
- **`record AccountingReportQuery(String keyword, List<Long> creditAccount, Integer hasBill, LocalDate fromDate, LocalDate toDate)`**
  - Ngày dùng `@DateTimeFormat(iso = DATE)`.
  - `keyword` null → `""`, có `@Size(max = 500)`.
  - `creditAccount` null → `List.of()`.
  - `hasBill`: `@Min(0) @Max(1)`, message "Giá trị hóa đơn không hợp lệ".
  - `@AssertTrue isDateRangeValid()`, message "Từ ngày không được sau đến ngày".
  - Tên tham số `creditAccount` giữ như URL cũ (lặp `creditAccount=1&creditAccount=2`).
- **`record AccountingReportFilter(String keyword, List<Long> creditAccounts, int hasBill, boolean restricted, List<String> branches, LocalDateTime from, LocalDateTime toExclusive)`**
  - `hasBill`: `-1` = không lọc, `0`/`1`.
  - `noAccess()` = `restricted && branches.isEmpty()`.
  - `filterAccounts()` = `!creditAccounts.isEmpty()`.
  - `sqlAccounts()` trả `[-1L]` khi rỗng, `sqlBranches()` trả `["(none)"]` khi rỗng, để tránh `IN ()`.
  - **`static of(query, Employee caller, Integer limitedYear)`**:
    - `restricted = !AssignAuth.isAdmin(caller) && (BGD hoặc NVKT) && branchCodes không chứa "HN"`, so không phân biệt hoa thường.
    - `branches = restricted ? branchCodes : List.of()`.
    - `keyword = LikePattern.contains(...)`.
    - Ngày lấy từ `ReportPeriod.ofLimitedYear(limitedYear).narrow(...)`.
    - Id null bị bỏ khỏi `creditAccounts`.
- **`interface AccountingReportRow`** (projection): `getId`, `getAccountingDate` (`LocalDateTime`), `getCustName`, `getContractId`, `getBillId`, `getAccountingMode`, `getAmountOfMoney` (`Long`), `getSubsection`, `getAccountingContent`, `getSourceAccount`, `getAccountingNote`, `getEntryId`.
- **`record AccountingReportItem(...)`** có `receiptAmount` / `paymentAmount` (`Long`, null khi không đúng mode) thay cho mode + amount. Tạo bằng `static from(row)`, dùng hằng `Accounting.MODE_RECEIPT`/`MODE_PAYMENT`.
- **`interface AccountingReportTotalsRow`** (`BigDecimal getTotalReceipt()`, `getTotalPayment()`) và **`record AccountingReportTotals(BigDecimal totalReceipt, BigDecimal totalPayment)`**, có `ZERO` và `from(row)` (null → 0).
- **`record AccountingReportFilterOptions(List<Option> creditAccounts, Integer limitedYear)`**. Tái dùng `Option(Long value, String label)` đang dùng trong `AccountGroup`; xem kiểu ở `AccountingService`.

- [ ] **Step 0: Tạo nhánh** `git -C backend switch -c feat/accounting-report` và `git -C frontend switch -c feat/accounting-report` từ `develop`. Nếu có thay đổi chưa commit thì dừng và báo người dùng.
- [ ] **Step 1: Test thất bại `AccountingReportFilterTest`**:
  - `admin_xem_tất_cả`: role admin, chức vụ NVKT, chi nhánh "DN" → `restricted` false.
  - `bgd_nvkt_có_hn_xem_tất_cả`: `("BGD","DN, hn")` → false.
  - `bgd_nvkt_không_hn_bị_giới_hạn`: `("NVKT"," DN , HP ")` → true, `branches` là `[DN, HP]`.
  - `chức_vụ_khác_không_bị_giới_hạn`: NVKD, KSV, `null` → false. Đây là lỗ hổng được giữ, export mở toàn bộ.
  - `không_chi_nhánh_thì_noAccess`: BGD, chi nhánh null → true.
  - `ngày_và_năm`: limitedYear 2026 với from/to null → `[2026-01-01, 2027-01-01)`. from 2026-03-01, to 2026-03-15 → `toExclusive` 2026-03-16. limitedYear null với ngày null → `ReportPeriod` min/max.
  - `từ_khóa_escape`: "a_b%" → `"%a\_b\%%"`.
  - `tài_khoản_và_hóa_đơn`: null → rỗng/-1, `sqlAccounts` `[-1]`. `[3, null, 5]` → `[3, 5]`.
- [ ] **Step 2: Test thất bại `AccountingReportTotalsTest`**: null → 0. Thêm `AccountingReportItemTest`: PT → `receiptAmount`, PC → `paymentAmount`.
- [ ] **Step 3: Chạy** `./mvnw -q test -Dtest='AccountingReport*Test'` → FAIL.
- [ ] **Step 4: Tạo** hằng và DTO. Javadoc ghi nguồn quy tắc cũ (`ReportServiceImpl`, `checkRole`).
- [ ] **Step 5: Chạy lại** → PASS.
- [ ] **Step 6: Commit** `feat(report): DTO và phạm vi báo cáo thu chi`.

### Task 2: SQL và repository

**Files:**
- Create: `repository/AccountingReportSql.java`, `repository/AccountingReportRepository.java`
- Test: `repository/AccountingReportRepositoryTest.java` (`@MysqlDataJpaTest`, dữ liệu qua `RepositoryTestData`, tiền tố MARK)
  - Cô lập dữ liệu bằng từ khóa MARK trong `entry_id`.
  - Với test phạm vi, dùng chi nhánh MARK riêng, ví dụ `"ZT1"`.

**Interfaces:** `AccountingReportRepository extends Repository<Accounting, Long>`.
- Default methods `search`, `stream`, `totals` (`noAccess` → `Page.empty` / `Stream.empty` / `ZERO`).
- `findPage` có `countQuery`; `streamAll` với hint fetchSize `-2147483648`; `sums`.
- Khuôn theo `SurplusReportRepository`.

SQL (`final class AccountingReportSql`, chỉ ghép hằng; tham số SpEL `:#{#filter.x}`; tên tiểu mục lấy qua `T(...Subsection).X`; type lấy qua `T(...Accounting).TYPE_CREATED_BY_USER`/`TYPE_EXPLOITATION_FEE`; COLLATE theo quy ước `backend/CLAUDE.md`, xem cách `SurplusReportSql.CONTRACT_DETAILS` viết):

```
FROM_WHERE = FROM accounting ap
 WHERE ap.deleted = 0 AND ap.type IN (1, 3)
   AND ap.accounting_date >= :from AND ap.accounting_date < :toExclusive
   AND (:keyword = '' OR ap.entry_id LIKE :keyword OR ap.cust_name LIKE :keyword OR ap.contract_id LIKE :keyword)
   AND (:filterAccounts = FALSE OR ap.credit_account IN (:sqlAccounts) OR ap.debit_account IN (:sqlAccounts))
   AND (:hasBill = -1
        OR (:hasBill = 1 AND EXISTS (BILL))
        OR (:hasBill = 0 AND NOT EXISTS (BILL)))
   AND (:restricted = FALSE
        OR (ap.subsection <> 'hoa hồng HO' AND ap.subsection <> 'hoa hồng hội sở'
            AND EXISTS (SELECT 1 FROM p_appraisal_request ar
                        WHERE ar.contract_id = ap.contract_id COLLATE … AND ar.linked_branch IN (:sqlBranches))))
BILL = SELECT 1 FROM juridical ju WHERE ju.deleted = 0 AND ju.bill_id IS NOT NULL AND ju.bill_id <> ''
        AND ju.contract_id = ap.contract_id COLLATE …
COLUMNS = SELECT ap.id, ap.accounting_date accountingDate, ap.cust_name custName, ap.contract_id contractId,
   (SELECT MAX(ju.bill_id) FROM juridical ju WHERE … cùng điều kiện BILL) billId,
   ap.accounting_mode accountingMode, ap.amount_of_money amountOfMoney, ap.subsection,
   ap.accounting_content accountingContent, ap.accounting_note accountingNote, ap.entry_id entryId,
   (SELECT CASE WHEN TRIM(CONCAT_WS(' ', TRIM(em.first_name), TRIM(em.last_name))) <> ''
                THEN TRIM(CONCAT_WS(' ', TRIM(em.first_name), TRIM(em.last_name))) ELSE ca.credit_account END
      FROM credit_account ca LEFT JOIN employees em ON em.employee_id = ca.employee_id
     WHERE ca.id = CASE ap.accounting_mode WHEN 'PT' THEN ap.credit_account WHEN 'PC' THEN ap.debit_account END) sourceAccount
ROWS   = COLUMNS + FROM_WHERE + ORDER BY ap.accounting_date DESC, ap.created DESC, ap.id DESC
COUNT  = SELECT COUNT(*) + FROM_WHERE
TOTALS = SELECT SUM(CASE WHEN ap.accounting_mode='PT' THEN ap.amount_of_money ELSE 0 END) totalReceipt,
                SUM(CASE WHEN ap.accounting_mode='PC' THEN ap.amount_of_money ELSE 0 END) totalPayment + FROM_WHERE
```

Ghi chú khi viết SQL:
- `getDisplayName` cũ không xét tài khoản đã xóa, nên ở đây cũng không lọc `ca.deleted`. Ghi điều này vào comment.
- Kiểm tra cột `created` có tồn tại trên `accounting` không (theo entity hoặc schema). Nếu không có thì bỏ khỏi `ORDER BY` và ghi lại.

- [ ] **Step 1: Test thất bại** (mỗi test tự chèn dữ liệu):
  - `cột_và_tài_khoản_nguồn`:
    - PT: `credit_account` = tài khoản của nhân viên "Nguyễn An" → nguồn là "Nguyễn An".
    - PC: `debit_account` = tài khoản công ty mã "TCT" → nguồn là "TCT".
    - Số hóa đơn lấy từ `juridical` chưa xóa; `juridical` đã xóa hoặc `bill_id` rỗng bị bỏ qua.
  - `loại_bút_toán_đã_xóa_và_type_2`.
  - `từ_khóa_ba_cột`: tìm theo `entry_id`, `cust_name`, `contract_id`.
  - `lọc_tài_khoản_khớp_hai_phía`.
  - `lọc_hóa_đơn_có_và_chưa_có`: bút toán không có `contract_id` rơi vào "chưa có".
  - `mốc_ngày_trọn_ngày`.
  - `sắp_xếp_ngày_giảm_dần`.
  - `phạm_vi_giới_hạn`: thấy bút toán có hồ sơ chi nhánh ZT1; không thấy hồ sơ chi nhánh khác, bút toán không có hồ sơ, tiểu mục "hoa hồng HO"/"hoa hồng hội sở", hay tiểu mục NULL.
  - `phạm_vi_không_giới_hạn_thấy_tất_cả`.
  - `không_nhân_đôi_dòng`: hai hồ sơ cùng `contract_id` và hai `juridical` có `bill_id` → một dòng, `billId` = MAX.
  - `tổng_khớp_tổng_các_dòng`.
  - `noAccess_không_chạy_truy_vấn`.
  - `stream_trả_cùng_các_dòng_như_trang` (chạy trong transaction).
- [ ] **Step 2: Chạy** `./mvnw -q test -Dtest=AccountingReportRepositoryTest` → FAIL.
- [ ] **Step 3: Tạo** `AccountingReportSql` và `AccountingReportRepository`. Javadoc ghi lý do dùng native và quy tắc phạm vi giữ nguyên hệ thống cũ.
- [ ] **Step 4: Chạy lại** → PASS. Test cần MySQL; không có DB thì ghi rõ là chưa chạy, không coi là pass.
- [ ] **Step 5: Commit** `feat(report): truy vấn báo cáo thu chi`.

### Task 3: File Excel

**Files:**
- Create: `common/util/export/excel/AccountingReportExcelExporter.java`
- Test: `…/AccountingReportExcelExporterTest.java`

**Interfaces:**
- `public static void export(Consumer<Consumer<AccountingReportItem>> dataSource, OutputStream out)`.
- `TITLE = "BÁO CÁO THU CHI"`, sheet "Report".
- 12 cột, độ rộng như cũ.
- Ngày là chuỗi `dd/MM/yyyy`. Tiền dùng `styleRowPrice`; ô trống khi không đúng mode. STT căn giữa.

- [ ] **Step 1: Test thất bại** (mẫu `SurplusReportExcelExporterTest`):
  - `bố_cục_như_cũ`: B1 là TITLE, gộp `B1:M1`, autofilter `B3:M3`, header đúng thứ tự.
  - `pt_ghi_cột_thu_pc_ghi_cột_chi`.
  - `ngày_dd_MM_yyyy`.
  - `nhiều_dòng_hơn_cửa_sổ_streaming_đều_được_ghi` (300 dòng).
- [ ] **Step 2–4:** FAIL → cài đặt → PASS.
- [ ] **Step 5: Commit** `feat(report): file Excel báo cáo thu chi`.

### Task 4: Service

**Files:**
- Create: `service/AccountingReportService.java`
- Test: `service/AccountingReportServiceTest.java`

**Interfaces:** service dùng repository, exporter, `CurrentEmployeeService.require`, `ReportYearLimit.limitedYear`, `AccountingService.sharedAccounts`.
- `Page<AccountingReportItem> search(Long callerId, AccountingReportQuery q, Pageable p)`.
- `AccountingReportTotals totals(callerId, q)`.
- `@Transactional(readOnly = true) void export(callerId, q, OutputStream out)`: `try (Stream<…> rows = repository.stream(filter))`.
- `AccountingReportFilterOptions filterOptions(callerId)`:
  - Gọi `sharedAccounts(branchCodes, chứa HN)`.
  - Sắp ổn định `type == TYPE_COMPANY_ACCOUNT` lên đầu.
  - `Option(id, creditAccount)`.
  - Kèm `limitedYear`.

- [ ] **Step 1: Test thất bại** (Mockito):
  - `search_gắn_phạm_vi_và_năm` (captor kiểm filter).
  - `noAccess_trả_rỗng`.
  - `export_ghi_file_từ_stream_và_đóng_stream`.
  - `filter_options_tài_khoản_công_ty_trước`.
  - `filter_options_hn_thấy_tất_cả`.
- [ ] **Step 2–4:** FAIL → cài đặt → PASS.
- [ ] **Step 5: Commit** `feat(report): service báo cáo thu chi`.

### Task 5: Controller và quyền

**Files:**
- Create: `controller/AccountingReportController.java`
- Modify: `config/SecurityConfig.java`, đặt cạnh khối `/api/reports/surplus/**`
- Test: `controller/AccountingReportControllerTest.java`

**Interfaces:**
- `GET /api/reports/accounting`: `page` mặc định 0, `size` mặc định 20 (1..100), dùng `@Valid @ModelAttribute AccountingReportQuery`.
- `GET /totals`, `GET /filter-options`.
- `GET /export` → tên file `"Báo-cáo-thu-chi_" + dd-MM-yyyy_HH-mm-ss + ".xlsx"`.

Matcher (thứ tự bắt buộc; kiểm tra chuỗi authority admin đang dùng, ví dụ `"admin"` ở khối contract-tracking):
```java
// Báo cáo thu chi — giữ nguyên quyền hệ thống cũ, kể cả lỗ hổng (theo yêu cầu):
// export chỉ chịu rule chung /report/** của cũ; trang chỉ role admin hoặc chức vụ BGD/NVKT.
.requestMatchers("/api/reports/accounting/export")
.hasAnyAuthority("admin", Positions.BGD, Positions.NVKT, Positions.NVKD, Positions.NVTD, Positions.TPTD, Positions.KSV, Positions.NVPI)
.requestMatchers("/api/reports/accounting", "/api/reports/accounting/**")
.hasAnyAuthority("admin", Positions.BGD, Positions.NVKT)
```

- [ ] **Step 1: Test thất bại** (mẫu `SurplusReportControllerTest`):
  - `cần_đăng_nhập` cho cả 4 path.
  - `trang_tổng_options_chỉ_admin_bgd_nvkt`: NVKD, KSV → 403; admin, BGD, NVKT → 200.
  - `export_mở_cho_các_chức_vụ_như_cũ`: mỗi authority trong danh sách → 200; authority lạ → 403.
  - `danh_sách_bind_bộ_lọc`: `keyword=x&creditAccount=1&creditAccount=2&hasBill=0&fromDate=2026-03-01&toDate=2026-03-15&page=1&size=10` → captor đúng.
  - `tham_số_sai_trả_400`: `fromDate=01/03/2026`, from > to, `hasBill=2`, `creditAccount=abc`, `size=101`, `page=-1`.
  - `export_trả_file_tên_đúng`: `filename*=UTF-8''B%C3%A1o-c%C3%A1o-thu-chi_`.
- [ ] **Step 2–3:** FAIL → cài đặt controller (Javadoc đủ mục "Khác hệ thống cũ" và ghi chú giữ lỗ hổng) + matcher.
- [ ] **Step 4: Chạy** `./mvnw -q test -Dtest='AccountingReport*Test'` → PASS.
- [ ] **Step 5: Toàn backend** `./mvnw test` → `BUILD SUCCESS`.
- [ ] **Step 6:** cập nhật `backend/CLAUDE.md` (exporter mới).
- [ ] **Step 7: Commit** `feat(report): API báo cáo thu chi`.

### Task 6: Frontend — lib và api

**Files:**
- Create: `lib/accounting-report.ts`, `api/accounting-reports.ts`
- Spec: `lib/__tests__/accounting-report.spec.ts`, `api/__tests__/accounting-reports.spec.ts`

**Interfaces:**
- `ACCOUNTING_REPORT_AUTHORITIES = ['admin', 'BGD', 'NVKT']`. Kiểm tra cách guard nhận admin như ghi chú ở Quyết định 1.
- `ACCOUNTING_REPORT_PAGE_SIZE = 20`.
- `HAS_BILL_OPTIONS = [{ value: '1', label: 'Có hóa đơn' }, { value: '0', label: 'Chưa có hóa đơn' }]`.
- Types `AccountingReportItem`, `AccountingReportTotals`, `AccountingReportFilterOptions`, `AccountingReportFilter` (`keyword`, `creditAccount: number[]`, `hasBill`, `fromDate`, `toDate`), `AccountingReportQuery`.
- Hàm: `listAccountingReport`, `getAccountingReportTotals`, `getAccountingReportFilterOptions`, `exportAccountingReport`. Khuôn theo `api/surplus-reports.ts`. `creditAccount` gửi dạng lặp; mảng rỗng và tham số rỗng thì không gửi. Xem cách `lib/api` serialize mảng; nếu cần, dùng `URLSearchParams.append`.

- [ ] **Step 1–4:** spec thất bại (URL, mảng lặp, bỏ tham số rỗng) → cài đặt → PASS.
- [ ] **Step 5: Commit** `feat(report): api báo cáo thu chi`.

### Task 7: Frontend — màn hình, route, menu

**Files:**
- Create: `views/report/AccountingReportView.vue`, `views/__tests__/AccountingReportView.spec.ts`
- Modify:
  - `router/index.ts`: thêm route, comment "Màn /report/accounting (Báo cáo → Báo cáo thu chi) của hệ thống cũ: role admin hoặc chức vụ BGD/NVKT".
  - `nav-items.ts:105`: thêm `to: '/report/accounting'`.
  - `guard.spec.ts`, `layout.spec.ts`.

Bố cục sao chép `SurplusReportView.vue`: bộ lọc nháp → áp dụng, đồng bộ URL, tổng tải riêng và chỉ hiện ở trang cuối, export dùng bộ lọc đã áp dụng. Thêm phần tải `filter-options` và `Alert` năm theo `ContractTrackingReportView.vue`. Bộ lọc gồm:
- Từ khóa, placeholder "Số bút toán, Tên khách hàng, Số hợp đồng".
- `MultiSelect` tài khoản nguồn, placeholder "Tài khoản nguồn". Id là số, khi gắn vào `MultiSelect` thì đổi qua lại `string`.
- Select hóa đơn, mục trống "Tất cả hóa đơn".
- `DateRangePicker` "Ngày lập".

Bảng 12 cột theo đúng thứ tự cũ. Ngày `dd/MM/yyyy`, tiền căn phải định dạng `1.234.567` (`formatMoney`). Dòng tổng: "Tổng cộng:" span 5 cột · tổng thu · tổng chi · span 5 cột.

- [ ] **Step 1: Spec thất bại**:
  - Render 12 cột đúng thứ tự.
  - Tải `filter-options` và hiện tài khoản trong `MultiSelect`.
  - Chọn bộ lọc rồi Tìm kiếm → API nhận đúng tham số, URL đồng bộ (gồm `creditAccount` lặp).
  - Dòng tổng chỉ ở trang cuối.
  - Xuất Excel gửi bộ lọc đã áp dụng.
  - Có `Alert` năm khi `limitedYear` khác null.
- [ ] **Step 2:** `npx vitest run src/views/__tests__/AccountingReportView.spec.ts` → FAIL.
- [ ] **Step 3: Cài đặt** view, route, menu. Cập nhật `guard.spec.ts` (admin, BGD, NVKT vào được; NVKD, KSV bị chặn) và `layout.spec.ts` (menu có link cho BGD và NVKT; bỏ href mới khỏi selector placeholder).
- [ ] **Step 4:** PASS.
- [ ] **Step 5: Toàn frontend** `npm run lint && npm run type-check && npm run test:unit`. Lỗi có từ trước và không liên quan thì ghi rõ (đã biết 1 test fail sẵn trên `develop`).
- [ ] **Step 6: Commit** `feat(report): màn báo cáo thu chi`.

### Task 8: Kiểm tra cuối

- [ ] `cd backend && ./mvnw test`; `cd frontend && npm run lint && npm run type-check && npm run test:unit`.
- [ ] Chạy app nếu có môi trường:
  - Đăng nhập BGD có HN và NVKT ngoài HN, vào `/report/accounting`. So số dòng và tổng thu/chi với hệ thống cũ cùng bộ lọc (nhớ mục "Khác hệ thống cũ" 2–4). Mở file Excel.
  - Gọi thẳng API: NVKD gọi `/api/reports/accounting` → 403, `/export` → 200.
- [ ] Code review (`superpowers:requesting-code-review`, subagent `model: "sonnet"`).
- [ ] Báo cáo người dùng: kết quả test thật (số test, lỗi); phần chưa kiểm (chạy app, so số liệu); nhắc lại lỗ hổng quyền và phạm vi export được giữ theo yêu cầu.
