# Migrate "Báo cáo theo dõi hợp đồng" (/report/contract-tracking) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **Model cho subagent: CHỈ dùng `sonnet`.** Mọi lần gọi tool `Agent` (implementer, spec reviewer, code reviewer, final reviewer) phải truyền `model: "sonnet"`.

**Goal:** Port màn "Báo cáo theo dõi hợp đồng" (`/report/contract-tracking` + `/report/export-contract-tracking`) từ `td-project-legacy` sang `backend` (REST `/api/reports/contract-tracking`) và `frontend` (route `/report/contract-tracking`): danh sách phân trang, dòng tổng, xuất Excel.

**Architecture:** Theo đúng khuôn "Báo cáo công nợ" (`DebtReport*`): Query (tham số client) → Filter (chuẩn hóa + phạm vi người xem) → repository native SQL dùng chung `WHERE` cho trang / count / tổng / stream → service map Row → Item (tính tỷ lệ, công nợ) → controller trả `Page`, `/totals`, `/export` (SXSSF streaming). Phí theo tiểu mục (phí khai thác, chi khác, công tác phí) gộp bằng một bảng dẫn xuất `SUM(CASE …)` theo `contract_id`, thay cho 2 truy vấn mỗi dòng của hệ thống cũ. Frontend sao chép `DebtReportView.vue`, bộ lọc tháng như `JuridicalReportView.vue`.

**Tech Stack:** Spring Boot 4 · Java 21 · Spring Data JPA native query + interface projection · Apache POI SXSSF · JUnit 5/Mockito/AssertJ · `@MysqlDataJpaTest` · Vue 3 + TS · shadcn-vue · Vitest.

**Spec:** Không có spec riêng; spec là hành vi hệ thống cũ:
- `td-project-legacy/src/main/java/com/jvb/thanhdo/controller/ReportController.java` — `contractTrackingReport` (395-429), `exportContractTrackingToExcel` (430-443)
- `td-project-legacy/src/main/java/com/jvb/thanhdo/service/impl/ReportServiceImpl.java` — hằng tiểu mục (103-109), `getContractTrackingReport` (866-935), `getContractTrackingReportTotalMoney` (939-973), `getContractTrackingReportExport` (976-1038), `getReportYearRange` (1267-1278), `canViewAllContractTracking` (1284-1288), `getPaidAmount` (1987-1994)
- `td-project-legacy/src/main/java/com/jvb/thanhdo/repository/AppraisalRequestRepository.java` — `getPagedAllContractTracking*` (356-403), `getContractTrackingTotalMoney` (405-431)
- `td-project-legacy/src/main/java/com/jvb/thanhdo/repository/AccountingRepository.java` — `getContractTrackingAccounting` (511-530)
- `td-project-legacy/src/main/resources/templates/report/contract-tracking.html`, `controller/component/ContractTrackingExcelExporter.java`
- `td-project-legacy/src/main/resources/db/migration/V106__update_subsections.sql:54-55` (tên tiểu mục "Công tác phí")

Mẫu đã port (đọc trước task tương ứng): `backend/.../{controller/DebtReportController, service/DebtReportService, repository/DebtReportRepository, repository/DebtReportSql, model/dto/DebtReport*, common/util/export/excel/DebtReportExcelExporter}.java` + test cùng tên; `frontend/src/{api/debt-reports.ts, lib/debt-report.ts, views/report/DebtReportView.vue}` + spec; bộ lọc tháng: `frontend/src/views/report/JuridicalReportView.vue` (`MonthPicker`, `monthStartDate`, `monthEndDate`).

**Phụ thuộc plan khác:** `docs/superpowers/plans/2026-10-25-migrate-business-report.md` Task 1 tạo `ReportYearLimit` + `ReportPeriod`. Plan này dùng hai helper đó; nếu lúc thực hiện chúng chưa có trong `backend/`, làm Task 1 của plan kia trước (đúng như ở đó viết), rồi mới vào Task 1 dưới đây.

---

## Hành vi hệ thống cũ

| Mục | Hệ thống cũ |
|---|---|
| Quyền | Trang: role `admin` hoặc chức vụ `BGD`, khác → forbidden. Menu: `BGD`. Export **không kiểm tra quyền** (chỉ chặn chung `/report/**`). |
| Phạm vi | `canViewAllContractTracking`: role admin, hoặc `BGD` có chi nhánh chứa "HN" → mọi hồ sơ; còn lại (BGD chi nhánh) → `ap.linked_branch IN (chi nhánh của user)`. |
| Giới hạn năm | Không có role `ADMIN_REPORT` → chỉ hồ sơ `create_date` trong năm hiện tại (có thông báo trên trang). |
| Bộ lọc | `keyword` (LIKE `contract_id`/`cust_name`/`company_name`, không escape); vùng 1..5 (chi nhánh của vùng); chi nhánh; nguồn (`ap.source` = `bank_code`, nhãn "tên - mã"); đầu mối khách hàng (`ap.business_source_id`); ngày tạo theo **tháng** từ–đến (`fromDate` = ngày 1, `toDate` = ngày cuối tháng; so `00:01`–`23:59`). |
| Sắp xếp | `create_date DESC`, 20 dòng/trang. |
| Cột | STT · Số hợp đồng · Tên khách hàng (`cust_name` khác rỗng → `cust_name`, còn lại `company_name`) · Phát hành (có `appraisal_certificate` cùng `contract_id`, `deleted = 0`) · Giá trị hợp đồng (`parseLongSafe(valuation_fee)`) · Phí khai thác chi nhánh · **[HN]** Phí khai thác Hội sở · Chi khác · Công tác phí tạm tính (`ap.business_fee`) · Tỷ lệ % · Nguồn · Đầu mối kinh doanh (`ap.business_source`) · Công nợ (giá trị HĐ − đã thanh toán) · Công tác phí nhân viên · Ghi chú. |
| Phí theo tiểu mục | Tổng `accounting.amount_of_money` (chưa xóa, mọi `type`, không đổi dấu PT/PC) theo `accounting.subsection`: "Phí khai thác chi nhánh", "Phí khai thác Hội sở" (chỉ khi xem tất cả), "Chi khác", "Công tác phí". |
| Đã thanh toán | Như báo cáo công nợ: `type = 1`, tiểu mục ∈ `PAYMENT_SUBSECTIONS`, PT cộng / PC trừ. |
| Tỷ lệ % | `phí KT chi nhánh / (giá trị HĐ − công tác phí tạm tính − chi khác) × 100`; mẫu số 0 → 0; 2 chữ số. |
| Dòng tổng | Chỉ trang cuối: giá trị HĐ, phí KT chi nhánh, [HN] phí KT hội sở, chi khác, công tác phí tạm tính, công nợ, công tác phí nhân viên. |
| Excel | "BÁO CÁO THEO DÕI HỢP ĐỒNG" dòng 1 cột B, header dòng 3, dữ liệu từ dòng 4, không dòng tổng; cột Hội sở chỉ khi BGD có HN; file `Báo-cáo-theo-dõi-hợp-đồng_<timestamp>.xlsx`. |

Lỗi của hệ thống cũ phát hiện khi đọc code (được sửa, xem mục dưới): tổng "Công tác phí nhân viên" so tiểu mục `'Công tác phí nhân viên'` không còn tồn tại sau V106 (luôn 0) trong khi dòng dùng `"Công tác phí"`; trang bỏ qua bút toán có tiểu mục không có trong bảng `subsection`, Excel thì không; cột Hội sở trên trang hiện theo "chi nhánh chứa HN" (admin không có HN bị ẩn dù có số), trong Excel theo "BGD có HN" (admin không bao giờ thấy); Excel ghi tỷ lệ âm thành "0" và ghi `business_fee` không phải số thành chuỗi.

## Quyết định đã chọn (mặc định đề xuất, người dùng xem lại khi duyệt plan)

1. **Quyền** như cũ: `/api/reports/contract-tracking/**` → `BGD`, `admin`; export bị chặn như trang. Route FE: `['BGD']` (như menu).
2. **Cột/tổng "Phí khai thác Hội sở"** chỉ có khi người xem xem được tất cả (role admin hoặc BGD có HN), thống nhất trang, tổng, Excel; API trả `null` cho người khác (không chỉ ẩn trên giao diện).
3. **"Công tác phí nhân viên"** = tiểu mục `"Công tác phí"` cho cả dòng, tổng, Excel.
4. **Giữ giới hạn năm** `ADMIN_REPORT` và bộ lọc theo tháng.

## Global Constraints

- Subagent chỉ `model: "sonnet"`.
- `backend/`, `frontend/` là git submodule: `git -C backend …` / `git -C frontend …`; không commit ở repo gốc, không push.
- Backend theo `backend/CLAUDE.md` và `.claude/rules/jpa-spring.md`; frontend theo `frontend/CLAUDE.md` và `frontend/DESIGN_SYSTEM.md`.
- Comment, Javadoc, message lỗi, nhãn UI: tiếng Việt có dấu.
- API base `/api/reports/contract-tracking`; route FE path `report/contract-tracking`, name `report-contract-tracking`, breadcrumbs `[{ label: 'Báo cáo' }, { label: 'Báo cáo theo dõi hợp đồng' }]`.
- Trang 20 dòng; `size` tối đa 100; từ khóa tối đa 500 ký tự.
- Native SQL được phép (ghi lý do trên repository như `DebtReportRepository`): bảng dẫn xuất `SUM(CASE …)` và `REGEXP` đọc phí lưu VARCHAR.
- Lệnh: backend `cd backend && ./mvnw -q test -Dtest=<Tên>` (toàn bộ `./mvnw test`, test repository cần MySQL theo `backend/.env.example`); frontend `cd frontend && npm run lint && npm run type-check && npm run test:unit`.

### Khác hệ thống cũ (cố ý) — ghi vào Javadoc controller

1. JSON phân trang (`page` từ 0); tổng ở `/totals` (giao diện chỉ hiện ở trang cuối).
2. Khoảng tháng tính trọn ngày `[ngày 1 00:00, ngày đầu tháng sau 00:00)` (cũ bỏ phút đầu và phút cuối).
3. Từ khóa coi `%`, `_` là ký tự thường.
4. Phí theo tiểu mục, đã thanh toán, phát hành lấy trong cùng truy vấn (cũ 2–3 truy vấn mỗi dòng); trang và Excel cùng một cách cộng (cộng theo tên tiểu mục của bút toán).
5. "Công tác phí nhân viên" ở dòng tổng dùng đúng tiểu mục "Công tác phí" (cũ luôn 0).
6. Cột "Phí khai thác Hội sở" theo một quy tắc (Quyết định 2) cho trang, tổng, Excel; không trả số cho người không được xem.
7. Tỷ lệ % cùng một giá trị ở trang và Excel (kể cả âm), Excel ghi số `0.00`; công tác phí tạm tính không đọc được thành số → 0; Excel ghi công nợ là số (âm trong ngoặc).
8. Hồ sơ không có số hợp đồng: các cột tiền là 0 thay vì trống.
9. Export bị chặn theo cùng quyền với trang; vùng/ngày sai → 400.

## Review Focus

1. Hợp đồng có nhiều bút toán ở nhiều tiểu mục **và** nhiều chứng thư → đúng một dòng, số tiền không bị nhân lên, tổng khớp tổng các dòng (test `ContractTrackingReportRepositoryTest.nhiều_bút_toán_và_chứng_thư_vẫn_một_dòng_không_nhân_tiền`, `tổng_khớp_tổng_các_dòng`).
2. BGD chi nhánh (không HN) gọi thẳng API/Excel → `headOfficeFee` null ở dòng và tổng, Excel không có cột Hội sở (test `ContractTrackingReportServiceTest.bgd_chi_nhánh_không_nhận_phí_hội_sở`, `ContractTrackingReportExcelExporterTest.không_có_cột_hội_sở_khi_không_được_xem`).
3. `valuation_fee`/`business_fee` rỗng, chữ, có phần thập phân; mẫu số tỷ lệ bằng 0 → không lỗi, tỷ lệ 0 (test `ContractTrackingReportServiceTest.mẫu_số_bằng_không_thì_tỷ_lệ_không`, `ContractTrackingReportRepositoryTest.phí_không_đọc_được_thành_số_là_không`).
4. Người không có `ADMIN_REPORT` chọn tháng của năm trước → rỗng, tổng 0, không 400 (test `ContractTrackingReportFilterTest.tháng_ngoài_năm_được_xem_cho_khoảng_rỗng`).
5. BGD không có chi nhánh nào → trang rỗng, tổng 0, Excel chỉ header, không lỗi SQL `IN ()` (test `ContractTrackingReportFilterTest.bgd_không_chi_nhánh_thì_noAccess`, `ContractTrackingReportServiceTest.noAccess_không_gọi_truy_vấn`).

---

## File Structure

**Backend** (`backend/src/main/java/com/thanhdo/tdprojectbe/`):

| File | Trách nhiệm |
|---|---|
| Modify `repository/LinkedBranchRepository.java` | `default List<String> activeBranchCodesOfRegion(Integer region)` — thay 2 bản private `regionBranches` (Debt, Juridical; cả Business nếu đã có). |
| Modify `model/entity/Subsection.java` | Hằng tên 4 tiểu mục phí. |
| Modify `repository/DebtReportSql.java` | Tách `wholeNumber(column)` và `PAY_JOIN` để dùng lại; hành vi không đổi. |
| Create `model/dto/ContractTrackingReport{Query,Filter,Row,Item,TotalsRow,Totals,FilterOptions}.java` | DTO. |
| Create `repository/ContractTrackingReportSql.java`, `repository/ContractTrackingReportRepository.java` | SQL + truy vấn trang/stream/tổng. |
| Modify `common/util/export/excel/StreamingExportCommon.java`, `DebtReportExcelExporter.java` | Đưa `DEBT_FORMAT` lên lớp cha thành `protected static final String SIGNED_MONEY_FORMAT`. |
| Create `common/util/export/excel/ContractTrackingReportExcelExporter.java` | File Excel. |
| Create `service/ContractTrackingReportService.java` | Ghép phạm vi, map dòng, export. |
| Create `controller/ContractTrackingReportController.java`; Modify `config/SecurityConfig.java`, `backend/CLAUDE.md` | API, quyền, bảng helper. |

**Frontend** (`frontend/src/`): Create `lib/contract-tracking-report.ts`, `api/contract-tracking-reports.ts`, `views/report/ContractTrackingReportView.vue` + spec; Modify `router/index.ts`, `components/layout/nav-items.ts:107`, `router/__tests__/guard.spec.ts`, `components/layout/__tests__/layout.spec.ts`.

---

### Task 1: Helper dùng chung (vùng, tên tiểu mục, mảnh SQL)

**Files:**
- Modify: `backend/src/main/java/com/thanhdo/tdprojectbe/repository/LinkedBranchRepository.java`, `service/DebtReportService.java:89-95`, `service/JuridicalReportService.java:121-127` (và `BusinessReportService` nếu có bản copy)
- Modify: `model/entity/Subsection.java`, `repository/DebtReportSql.java`
- Modify: `backend/CLAUDE.md`
- Test: `backend/src/test/java/com/thanhdo/tdprojectbe/repository/LinkedBranchRepositoryTest.java` (thêm test; tạo file nếu chưa có, `@MysqlDataJpaTest`)

**Interfaces:**
- Produces:
  - `LinkedBranchRepository.activeBranchCodesOfRegion(Integer region): List<String>` — `null`/`0` → `List.of()`; còn lại `findBranchCodesByRegionAndDeleteFlag(region, LinkedBranch.DELETE_FLAG_ACTIVE)`.
  - `Subsection.BRANCH_EXPLOITATION_FEE = "Phí khai thác chi nhánh"`, `HEAD_OFFICE_EXPLOITATION_FEE = "Phí khai thác Hội sở"`, `OTHER_EXPENSE = "Chi khác"`, `TRAVEL_FEE = "Công tác phí"` (Javadoc: bút toán lưu tên tiểu mục, xem V106).
  - `DebtReportSql.wholeNumber(String column): String` — biểu thức `CASE … REGEXP …` hiện tại với cột truyền vào; `CONTRACT_VALUE = wholeNumber("ap.valuation_fee")`.
  - `DebtReportSql.PAY_JOIN` — đoạn `LEFT JOIN (SELECT contract_id, SUM(…) paid …) pay ON pay.contract_id = ap.contract_id `; `FROM_FILTERED` ghép lại từ `PAY_JOIN`, chuỗi SQL cuối cùng không đổi.

- [ ] **Step 0: Tạo nhánh** `git -C backend switch -c feat/contract-tracking-report` và `git -C frontend switch -c feat/contract-tracking-report`. Nếu submodule có thay đổi chưa commit thì dừng và báo người dùng.
- [ ] **Step 1: Test thất bại** trong `LinkedBranchRepositoryTest`:
  - `vùng_null_hoặc_không_thì_rỗng`: `activeBranchCodesOfRegion(null)` và `(0)` đều `isEmpty()`.
  - `vùng_trả_mã_chi_nhánh_chưa_xóa`: chèn hai chi nhánh vùng 2 (một `delete_flag = '1'`) → chỉ chứa mã chưa xóa.
- [ ] **Step 2: Chạy** `./mvnw -q test -Dtest=LinkedBranchRepositoryTest` → FAIL (biên dịch).
- [ ] **Step 3: Cài đặt** các Interfaces trên; xóa hai hàm private `regionBranches` và gọi helper mới.
- [ ] **Step 4: Chạy** `./mvnw -q test -Dtest='LinkedBranchRepositoryTest,DebtReport*Test,JuridicalReport*Test'` → PASS (test Debt chứng minh SQL không đổi).
- [ ] **Step 5: Bảng helper `backend/CLAUDE.md`** thêm: `| Mã chi nhánh của vùng cho bộ lọc báo cáo (vùng trống/0 là không lọc) | LinkedBranchRepository.activeBranchCodesOfRegion(region) | Viết lại hàm private regionBranches trong service báo cáo |`.
- [ ] **Step 6: Commit** `git -C backend commit -am "refactor(report): helper vùng, tên tiểu mục phí, tách mảnh SQL công nợ"` (thêm file test mới bằng `git -C backend add` trước).

### Task 2: DTO và quy tắc phạm vi

**Files:**
- Create: `model/dto/ContractTrackingReportQuery.java`, `ContractTrackingReportFilter.java`, `ContractTrackingReportRow.java`, `ContractTrackingReportItem.java`, `ContractTrackingReportTotalsRow.java`, `ContractTrackingReportTotals.java`, `ContractTrackingReportFilterOptions.java`
- Test: `model/dto/ContractTrackingReportFilterTest.java`, `model/dto/ContractTrackingReportTotalsTest.java`

**Interfaces:**
- Consumes: `ReportPeriod.ofLimitedYear(Integer).narrow(LocalDate, LocalDate)`, `LikePattern.contains`, `AssignAuth.isAdmin/isGlobalAdmin`, `MsbRequestScope.branchCodes`.
- Produces:
  - `record ContractTrackingReportQuery(@Size(max=500) String keyword, @Min(0) @Max(5) Integer region, @Size(max=255) String branch, @Size(max=255) String source, Long businessSourceId, @DateTimeFormat(iso=DATE) LocalDate fromDate, LocalDate toDate)` — constructor chuẩn hóa `keyword`/`branch`/`source` null → `""` (branch, source `trim()`); `@AssertTrue isDateRangeValid()` message "Từ ngày không được sau đến ngày"; message `@Min/@Max` vùng "Vùng không hợp lệ".
  - `record ContractTrackingReportFilter(String keyword, boolean allBranches, List<String> branches, List<String> regionBranches, String branch, String source, long businessSourceId, LocalDateTime from, LocalDateTime toExclusive)` với `noAccess()` (`!allBranches && branches.isEmpty()`), `filterRegion()` (`!regionBranches.isEmpty()`), `showHeadOfficeFee()` (= `allBranches`), và `static of(ContractTrackingReportQuery q, Employee caller, Integer limitedYear, List<String> regionBranches)`:
    - `allBranches = AssignAuth.isAdmin(caller) || (Positions.BGD.equals(caller.getPosition()) && AssignAuth.isGlobalAdmin(caller))`
    - `branches = allBranches ? List.of() : MsbRequestScope.branchCodes(caller)`
    - `businessSourceId = q.businessSourceId() == null ? 0 : …`; khoảng ngày `ReportPeriod.ofLimitedYear(limitedYear).narrow(q.fromDate(), q.toDate())`.
  - `interface ContractTrackingReportRow` (projection, getter `Long` cho số như `DebtReportRow`): `getContractId, getCustName, getCompanyName, getReleased (Long 0/1), getContractValue, getBranchFee, getHeadOfficeFee, getOtherExpense, getBusinessFee, getPaidAmount, getTravelFee, getSource, getBusinessSource, getNote`.
  - `record ContractTrackingReportItem(String contractId, String customerName, boolean released, long contractValue, long branchFee, Long headOfficeFee, long otherExpense, long businessFee, BigDecimal rate, String source, String businessSource, long debt, long travelFee, String note)`.
  - `interface ContractTrackingReportTotalsRow` (`BigDecimal` getters: `contractValue, branchFee, headOfficeFee, otherExpense, businessFee, paidAmount, travelFee`).
  - `record ContractTrackingReportTotals(BigDecimal contractValue, BigDecimal branchFee, BigDecimal headOfficeFee, BigDecimal otherExpense, BigDecimal businessFee, BigDecimal debt, BigDecimal travelFee)` với `static ZERO(boolean showHeadOfficeFee)` và `static from(ContractTrackingReportTotalsRow row, boolean showHeadOfficeFee)` — null → 0; `debt = contractValue − paidAmount`; `headOfficeFee = show ? giá trị : null`.
  - `record ContractTrackingReportFilterOptions(List<BranchItem> branches, List<ValuationReportFilterOptions.Source> sources, List<Option> businessSources, Integer limitedYear, boolean showHeadOfficeFee)`.

- [ ] **Step 1: Test thất bại `ContractTrackingReportFilterTest`** (helper `employee(position, branch, role)` như `AppraisalProfileReportFilterTest`):
  - `admin_và_bgd_hn_xem_tất_cả_và_thấy_phí_hội_sở`: `("NVKT","DN","admin")`, `("BGD","HN,HCM",null)` → `allBranches()`, `showHeadOfficeFee()` true, `branches()` rỗng, `noAccess()` false.
  - `bgd_chi_nhánh_xem_chi_nhánh_mình`: `("BGD"," HCM , DN ",null)` → `branches()` = `["HCM","DN"]`, `showHeadOfficeFee()` false.
  - `bgd_không_chi_nhánh_thì_noAccess`: `("BGD",null,null)` → `noAccess()` true.
  - `tháng_ngoài_năm_được_xem_cho_khoảng_rỗng`: limitedYear 2026, from/to = 2025-03-01/2025-03-31 → `!from().isBefore(toExclusive())`.
  - `khoảng_tháng_tính_trọn_ngày_và_từ_khóa_escape`: keyword `" a_b "`, from 2026-03-01, to 2026-04-30, limitedYear null → keyword `"%a\\_b%"`, `from` = 2026-03-01T00:00, `toExclusive` = 2026-05-01T00:00; businessSourceId null → 0.
- [ ] **Step 2: Test thất bại `ContractTrackingReportTotalsTest`**: `from(row(100,10,5,…, paid 30), false)` → `debt` 70, `headOfficeFee` null; cùng row với `true` → `headOfficeFee` 5; mọi getter null → mọi số 0.
- [ ] **Step 3: Chạy** `./mvnw -q test -Dtest='ContractTrackingReport*Test'` → FAIL.
- [ ] **Step 4: Tạo các DTO** theo Interfaces (Javadoc tham số như `DebtReportFilter`).
- [ ] **Step 5: Chạy lại** → PASS.
- [ ] **Step 6: Commit** `feat(report): DTO và phạm vi báo cáo theo dõi hợp đồng`.

### Task 3: SQL và repository

**Files:**
- Create: `repository/ContractTrackingReportSql.java`, `repository/ContractTrackingReportRepository.java`
- Test: `repository/ContractTrackingReportRepositoryTest.java` (`@MysqlDataJpaTest`, dữ liệu qua `RepositoryTestData`, contract id có tiền tố MARK như `AppraisalProfileReportRepositoryTest`)

**Interfaces:**
- Consumes: Task 1 (`DebtReportSql.wholeNumber`, `CONTRACT_VALUE`, `PAID`, `PAY_JOIN`; hằng `Subsection`), Task 2 (Filter, Row, TotalsRow, Totals).
- Produces: `ContractTrackingReportRepository extends Repository<AppraisalRequest, Long>` với `default Page<ContractTrackingReportRow> search(filter, pageable)`, `default Stream<ContractTrackingReportRow> stream(filter)`, `default ContractTrackingReportTotals totals(filter)` (`noAccess` → `Page.empty` / `Stream.empty` / `ContractTrackingReportTotals.ZERO(filter.showHeadOfficeFee())`), cùng `findPage` (có `countQuery`), `streamAll` (hint fetchSize `-2147483648`), `sums` — khuôn y hệt `DebtReportRepository`.

SQL (`final class ContractTrackingReportSql`, chỉ ghép hằng; tham số qua SpEL `:#{#filter.x}`; tên tiểu mục qua `:#{T(com.thanhdo.tdprojectbe.model.entity.Subsection).X}`):

```
FEE_JOIN = LEFT JOIN (SELECT contract_id,
     SUM(CASE WHEN subsection = <BRANCH_EXPLOITATION_FEE> THEN amount_of_money ELSE 0 END) branchFee,
     … <HEAD_OFFICE_EXPLOITATION_FEE> headOfficeFee, <OTHER_EXPENSE> otherExpense, <TRAVEL_FEE> travelFee
   FROM accounting WHERE (deleted = 0 OR deleted IS NULL)
     AND subsection IN (<4 tên>) GROUP BY contract_id) fee ON fee.contract_id = ap.contract_id
FROM_FILTERED = "FROM p_appraisal_request ap " + DebtReportSql.PAY_JOIN + FEE_JOIN
COLUMNS = ap.contract_id contractId, ap.cust_name custName, ap.company_name companyName,
   EXISTS(SELECT 1 FROM appraisal_certificate c WHERE c.contract_id = ap.contract_id AND c.deleted = 0) released,
   CONTRACT_VALUE contractValue, COALESCE(fee.branchFee,0) branchFee, COALESCE(fee.headOfficeFee,0) headOfficeFee,
   COALESCE(fee.otherExpense,0) otherExpense, wholeNumber("ap.business_fee") businessFee, PAID paidAmount,
   COALESCE(fee.travelFee,0) travelFee, ap.source source, ap.business_source businessSource, ap.note note
WHERE = ap.delete_flag = '0'
  AND keyword trên contract_id / cust_name / company_name (như DebtReportSql)
  AND (:allBranches = TRUE OR ap.linked_branch IN (:branches))
  AND (:filterRegion() = FALSE OR ap.linked_branch IN (:regionBranches))
  AND (:branch = '' OR ap.linked_branch = :branch)
  AND (:source = '' OR ap.source = :source)
  AND (:businessSourceId = 0 OR ap.business_source_id = :businessSourceId)
  AND ap.create_date >= :from AND ap.create_date < :toExclusive
ORDER_BY = ap.create_date DESC, ap.id DESC
ROWS = SELECT COLUMNS FROM_FILTERED WHERE ORDER_BY;  COUNT = SELECT COUNT(*) FROM_FILTERED WHERE
TOTALS = SELECT SUM(CONTRACT_VALUE) contractValue, SUM(COALESCE(fee.branchFee,0)) branchFee, … ,
         SUM(wholeNumber("ap.business_fee")) businessFee, SUM(PAID) paidAmount, SUM(COALESCE(fee.travelFee,0)) travelFee
         FROM_FILTERED WHERE
```

Javadoc repository: lý do native (bảng dẫn xuất `SUM(CASE …)`, `REGEXP` cho phí VARCHAR), cộng phí theo tên tiểu mục không đổi dấu PT/PC như hệ thống cũ.

- [ ] **Step 1: Test thất bại** (mỗi test tự chèn dữ liệu, lọc theo keyword MARK để cô lập):
  - `ánh_xạ_cột_phí_theo_tiểu_mục`: bút toán 100 "Phí khai thác chi nhánh", 20 "Phí khai thác Hội sở", 30 "Chi khác", 40 "Công tác phí", 999 tiểu mục khác, 7 bút toán đã xóa → `branchFee` 100, `headOfficeFee` 20, `otherExpense` 30, `travelFee` 40.
  - `đã_thanh_toán_như_báo_cáo_công_nợ`: PT 500 "Hợp đồng" type 1, PC 100 "Hoàn tạm ứng" type 1, PT 50 "Hợp đồng" type 2 → `paidAmount` 400.
  - `nhiều_bút_toán_và_chứng_thư_vẫn_một_dòng_không_nhân_tiền`: 3 bút toán + 2 chứng thư cùng contract → `totalElements` 1, `released` 1, phí đúng tổng.
  - `phát_hành_bỏ_chứng_thư_đã_xóa`: chỉ có chứng thư `deleted = 1` → `released` 0.
  - `phí_không_đọc_được_thành_số_là_không`: `valuation_fee` "abc", `business_fee` "1500.75" → `contractValue` 0, `businessFee` 1500.
  - `lọc_vùng_chi_nhánh_nguồn_đầu_mối_và_khoảng_ngày`: mỗi bộ lọc loại đúng hồ sơ; `toExclusive` loại hồ sơ đúng mốc đó.
  - `phạm_vi_chi_nhánh_và_tất_cả`; `hồ_sơ_đã_xóa_bị_loại`; `thứ_tự_ngày_tạo_giảm_dần_và_phân_trang`.
  - `tổng_khớp_tổng_các_dòng`: `totals(filter)` = tổng các dòng `search` cùng filter (từng cột; `debt` = contractValue − paid).
  - `noAccess_không_chạy_truy_vấn`: filter BGD không chi nhánh → trang rỗng, `totals` = `ZERO(false)`, stream rỗng.
  - `stream_trả_cùng_các_dòng_như_trang` (trong `TransactionTemplate`/`@Transactional` như test Debt).
- [ ] **Step 2: Chạy** `./mvnw -q test -Dtest=ContractTrackingReportRepositoryTest` → FAIL.
- [ ] **Step 3: Tạo** `ContractTrackingReportSql`, `ContractTrackingReportRepository`.
- [ ] **Step 4: Chạy lại** → PASS (cần MySQL; nếu không có DB, ghi rõ là chưa chạy, không coi là pass).
- [ ] **Step 5: Commit** `feat(report): truy vấn báo cáo theo dõi hợp đồng`.

### Task 4: File Excel

**Files:**
- Modify: `common/util/export/excel/StreamingExportCommon.java`, `DebtReportExcelExporter.java` (`DEBT_FORMAT` → `SIGNED_MONEY_FORMAT` ở lớp cha)
- Create: `common/util/export/excel/ContractTrackingReportExcelExporter.java`
- Test: `common/util/export/excel/ContractTrackingReportExcelExporterTest.java`

**Interfaces:**
- Consumes: `ContractTrackingReportItem`.
- Produces: `public static void export(Consumer<Consumer<ContractTrackingReportItem>> dataSource, boolean showHeadOfficeFee, OutputStream out)`; `static final String TITLE = "BÁO CÁO THEO DÕI HỢP ĐỒNG"`.

Cột (độ rộng như cũ): STT 7 · Số hợp đồng 25 · Tên khách hàng 25 · Phát hành 7 ("x" hoặc trống, căn giữa) · Giá trị hợp đồng 25 · Phí khai thác chi nhánh 20 · [Phí khai thác Hội sở 20] · Chi khác 20 · Công tác phí tạm tính 20 · Tỷ lệ 15 (số, định dạng `0.00`) · Nguồn 15 · Đầu mối kinh doanh 25 · Công nợ 20 (`SIGNED_MONEY_FORMAT`) · Công tác phí nhân viên 20 · Ghi chú 30. Tiền dùng `styleRowPrice`. Không dòng tổng.

- [ ] **Step 1: Test thất bại** (mẫu `AppraisalProfileReportExcelExporterTest`):
  - `bố_cục_như_cũ_có_cột_hội_sở`: `showHeadOfficeFee = true` → B1 = TITLE, vùng gộp `B1:P1`, autofilter `B3:P3`, header 15 cột đúng thứ tự trên.
  - `không_có_cột_hội_sở_khi_không_được_xem`: `false` → 14 cột, `B1:O1`, không có "Phí khai thác Hội sở".
  - `ghi_số_tiền_tỷ_lệ_và_công_nợ_âm_là_số`: item rate `12.35`, debt `-500`, released true → ô tỷ lệ NUMERIC 12.35 có format `0.00`, công nợ NUMERIC −500 format `SIGNED_MONEY_FORMAT`, phát hành "x".
  - `nhiều_dòng_hơn_cửa_sổ_streaming_đều_được_ghi` (300 dòng).
- [ ] **Step 2: Chạy** `./mvnw -q test -Dtest='ContractTrackingReportExcelExporterTest,DebtReportExcelExporterTest'` → FAIL.
- [ ] **Step 3: Cài đặt** exporter + chuyển hằng format.
- [ ] **Step 4: Chạy lại** → PASS.
- [ ] **Step 5: Commit** `feat(report): file Excel báo cáo theo dõi hợp đồng`.

### Task 5: Service

**Files:**
- Create: `service/ContractTrackingReportService.java`
- Test: `service/ContractTrackingReportServiceTest.java`

**Interfaces:**
- Consumes: repository (Task 3), exporter (Task 4), `CurrentEmployeeService`, `ReportYearLimit`, `LinkedBranchRepository.activeBranchCodesOfRegion`, `RequestFilterOptions.businessSources(Long)`, `BranchService.listActive()`, `SourceBankRepository.findByDeleteFlagOrderByBankCode`.
- Produces:
  - `@Transactional(readOnly = true) ContractTrackingReportFilterOptions filterOptions(Long callerId, Long businessSourceId)` — nguồn nhãn `bankName + " - " + bankCode`; `showHeadOfficeFee` từ filter của người xem.
  - `Page<ContractTrackingReportItem> search(Long callerId, ContractTrackingReportQuery q, Pageable p)` (không transaction, như Debt).
  - `ContractTrackingReportTotals totals(Long callerId, ContractTrackingReportQuery q)`.
  - `@Transactional(readOnly = true) void export(Long callerId, ContractTrackingReportQuery q, OutputStream out)` — tính filter trước, rồi `try (Stream<…> rows = repository.stream(filter))`.
  - package-private `static ContractTrackingReportItem toItem(ContractTrackingReportRow row, boolean showHeadOfficeFee)`:
    - `customerName` = `custName` khác rỗng (sau `trim`) → `custName`, còn lại `companyName` (null → `""`).
    - `debt = contractValue − paidAmount`; `headOfficeFee = show ? giá trị : null`; `released = released != 0`.
    - `rate`: mẫu = `contractValue − businessFee − otherExpense`; mẫu 0 → `BigDecimal.ZERO.setScale(2)`; còn lại `BigDecimal.valueOf(branchFee * 100).divide(BigDecimal.valueOf(mẫu), 2, RoundingMode.HALF_UP)`.

- [ ] **Step 1: Test thất bại** (Mockito, mẫu `AppraisalProfileReportServiceTest`):
  - `search_gắn_phạm_vi_vùng_và_năm`: caller BGD "HCM", `reportYearLimit.limitedYear` = 2026, region 2 → `activeBranchCodesOfRegion(2)` = `["DN"]` → filter bắt được có `branches` `["HCM"]`, `regionBranches` `["DN"]`, `from` 2026-01-01T00:00.
  - `tỷ_lệ_và_công_nợ`: contractValue 1000, businessFee 100, otherExpense 100, branchFee 200, paid 300 → `rate` `25.00`, `debt` 700.
  - `mẫu_số_bằng_không_thì_tỷ_lệ_không`: contractValue 200, businessFee 100, otherExpense 100 → `rate` `0.00`.
  - `tên_khách_hàng_rỗng_thì_lấy_tên_công_ty`: custName `"  "` → companyName.
  - `bgd_chi_nhánh_không_nhận_phí_hội_sở`: `toItem(row, false).headOfficeFee()` null; `totals` của BGD chi nhánh có `headOfficeFee` null (mock repository trả `ContractTrackingReportTotals.from(row, false)` — kiểm tra service truyền đúng filter).
  - `noAccess_không_gọi_truy_vấn`: BGD không chi nhánh → `search` rỗng, verify repository chỉ được gọi qua `default` (dùng mock trả `Page.empty`, hoặc kiểm tra `filter.noAccess()` qua captor).
  - `export_ghi_file_từ_stream_và_đóng_stream` (như test Appraisal).
- [ ] **Step 2: Chạy** `./mvnw -q test -Dtest=ContractTrackingReportServiceTest` → FAIL.
- [ ] **Step 3: Cài đặt** service (Javadoc: màn cũ, một quy tắc phạm vi cho trang/tổng/Excel).
- [ ] **Step 4: Chạy lại** → PASS.
- [ ] **Step 5: Commit** `feat(report): service báo cáo theo dõi hợp đồng`.

### Task 6: Controller, quyền, tài liệu

**Files:**
- Create: `controller/ContractTrackingReportController.java`
- Modify: `config/SecurityConfig.java` (sau khối `/api/reports/appraisal-profile/**`), `backend/CLAUDE.md` (dòng `SIGNED_MONEY_FORMAT` nếu cần, nhắc trong dòng streaming export)
- Test: `controller/ContractTrackingReportControllerTest.java`

**Interfaces:**
- Produces (khuôn `DebtReportController`): `GET /api/reports/contract-tracking/filter-options?businessSourceId=`, `GET /api/reports/contract-tracking` (`page` mặc định 0, `size` 20, 1..100), `GET /totals`, `GET /export` → file `"Báo-cáo-theo-dõi-hợp-đồng_" + dd-MM-yyyy_HH-mm-ss + ".xlsx"`. Matcher: `.requestMatchers("/api/reports/contract-tracking/**").hasAnyAuthority(Positions.BGD, "admin")` kèm comment tiếng Việt (như cũ trang chỉ admin/BGD; khác cũ: export cũng bị chặn).

- [ ] **Step 1: Test thất bại** (mẫu `AppraisalProfileReportControllerTest`, `ALLOWED = ["BGD","admin"]`, `FORBIDDEN = ["NVKT","NVKD","TPTD","NVTD","KSV","NVPI"]`, `PATHS = ["", "/filter-options", "/totals", "/export"]`):
  - `cần_đăng_nhập`, `chức_vụ_khác_bị_chặn_kể_cả_export`.
  - `danh_sách_bind_bộ_lọc`: params keyword, region 2, branch, source, businessSourceId 5, fromDate 2026-03-01, toDate 2026-04-30, page 1, size 10 → captor bằng `new ContractTrackingReportQuery("TDVC", 2, "HN", "MSB", 5L, …)`, `PageRequest.of(1, 10)`; JSON `$.data.items[0].rate`, `headOfficeFee`.
  - `tham_số_sai_trả_400` cho `""`, `/totals`, `/export`: fromDate `01/03/2026`, from > to, region 6, businessSourceId `abc`; và `size=101`, `page=-1`.
  - `export_trả_file_tên_đúng` (Content-Type xlsx, `filename*=UTF-8''B%C3%A1o-c%C3%A1o-theo-d%C3%B5i-h%E1%BB%A3p-%C4%91%E1%BB%93ng_`).
- [ ] **Step 2: Chạy** → FAIL. **Step 3: Cài đặt** controller (Javadoc liệt kê đủ 9 điểm "Khác hệ thống cũ") + matcher. **Step 4: Chạy** `./mvnw -q test -Dtest='ContractTrackingReport*Test'` → PASS.
- [ ] **Step 5: Chạy toàn bộ backend** `./mvnw test` → `BUILD SUCCESS`, 0 failure.
- [ ] **Step 6: Commit** `feat(report): API báo cáo theo dõi hợp đồng`.

### Task 7: Frontend — lib và api

**Files:**
- Create: `frontend/src/lib/contract-tracking-report.ts`, `frontend/src/api/contract-tracking-reports.ts`
- Test: `frontend/src/lib/__tests__/contract-tracking-report.spec.ts`, `frontend/src/api/__tests__/contract-tracking-reports.spec.ts`

**Interfaces:**
- Produces:
  - `CONTRACT_TRACKING_REPORT_AUTHORITIES = ['BGD']`, `CONTRACT_TRACKING_REPORT_PAGE_SIZE = 20`, `CONTRACT_TRACKING_REPORT_MAX_KEYWORD_LENGTH = 500`, `formatRate(value: number | null | undefined): string` (`12.5` → `"12,50"`, null → `"0"`; dùng định dạng số của dự án như `formatDebt`).
  - Types `ContractTrackingReportItem`, `ContractTrackingReportTotals` (`headOfficeFee: number | null`), `ContractTrackingReportFilterOptions` (`branches`, `sources: {code,name}[]`, `businessSources`, `limitedYear: number | null`, `showHeadOfficeFee: boolean`), `ContractTrackingReportFilter` (`keyword, region, branch, source, businessSourceId, fromDate, toDate`), `ContractTrackingReportQuery extends Filter` (`page`, `size`).
  - `listContractTrackingReport(query, signal?)`, `getContractTrackingReportTotals(filter, signal?)`, `getContractTrackingFilterOptions(businessSourceId, signal?)`, `exportContractTrackingReport(filter)` — đúng khuôn `api/debt-reports.ts` (không gửi tham số rỗng).

- [ ] **Step 1: Spec thất bại** theo khuôn `debt-reports.spec.ts` / `debt-report.spec.ts`: URL `/api/reports/contract-tracking`, `/totals`, `/filter-options`, `/export`; bỏ tham số rỗng; `formatRate`.
- [ ] **Step 2: Chạy** `npx vitest run src/lib/__tests__/contract-tracking-report.spec.ts src/api/__tests__/contract-tracking-reports.spec.ts` → FAIL. **Step 3: Cài đặt.** **Step 4: Chạy lại** → PASS.
- [ ] **Step 5: Commit** `git -C frontend commit` — `feat(report): api báo cáo theo dõi hợp đồng`.

### Task 8: Frontend — màn hình, route, menu

**Files:**
- Create: `frontend/src/views/report/ContractTrackingReportView.vue`, `frontend/src/views/__tests__/ContractTrackingReportView.spec.ts`
- Modify: `frontend/src/router/index.ts` (route sau `report-appraisal-profile`, comment "Màn /report/contract-tracking … của hệ thống cũ: BGD"), `frontend/src/components/layout/nav-items.ts:107` → `to: '/report/contract-tracking'`, `router/__tests__/guard.spec.ts`, `components/layout/__tests__/layout.spec.ts`

Sao chép `DebtReportView.vue` (bộ lọc nháp → áp dụng, đồng bộ URL, tổng tải riêng và chỉ hiện ở trang cuối, thông báo `limitedYear`, export dùng bộ lọc đã áp dụng). Bộ lọc: từ khóa (placeholder "Số hợp đồng, Tên khách hàng"), Vùng (`REGION_OPTIONS`), Chi nhánh, Nguồn (nhãn `name`), Đầu mối khách hàng, Ngày tạo từ–đến bằng `MonthPicker` → gửi `monthStartDate(from)` / `monthEndDate(to)` (như `JuridicalReportView`). Bảng đúng thứ tự cột của cũ; "Phát hành" là checkbox disabled; tiền định dạng `1.234.567`; cột và ô tổng "Phí khai thác Hội sở" chỉ khi `filterOptions.showHeadOfficeFee`; cột "Số hợp đồng" sticky như màn cũ nếu `DebtReportView` có sẵn kiểu đó.

- [ ] **Step 1: Spec thất bại** (khuôn `DebtReportView.spec.ts`):
  - render cột theo thứ tự; `showHeadOfficeFee: false` → không có header/ô "Phí khai thác Hội sở"; `true` → có.
  - chọn tháng 03/2026–04/2026 rồi Tìm kiếm → gọi API với `fromDate: '2026-03-01'`, `toDate: '2026-04-30'` và URL đồng bộ.
  - dòng tổng chỉ ở trang cuối; tỷ lệ hiện `formatRate`.
  - Xuất Excel gửi bộ lọc đã áp dụng (không phải bản nháp).
  - `limitedYear` 2026 → hiện thông báo năm.
- [ ] **Step 2: Chạy** `npx vitest run src/views/__tests__/ContractTrackingReportView.spec.ts` → FAIL. **Step 3: Cài đặt** view, route, menu; cập nhật `guard.spec.ts` (BGD vào được, NVKT bị chặn) và `layout.spec.ts` (menu có link). **Step 4: Chạy lại** → PASS.
- [ ] **Step 5: Kiểm tra toàn frontend** `npm run lint && npm run type-check && npm run test:unit` → sạch; nếu còn fail `AppraisalRequestListView.spec.ts` (lỗi có từ trước, không liên quan) thì ghi rõ trong báo cáo.
- [ ] **Step 6: Commit** `feat(report): màn báo cáo theo dõi hợp đồng`.

### Task 9: Kiểm tra cuối

- [ ] `cd backend && ./mvnw test` → `BUILD SUCCESS`; `cd frontend && npm run lint && npm run type-check && npm run test:unit`.
- [ ] Chạy app (nếu có môi trường): BGD HN, BGD chi nhánh, admin vào `/report/contract-tracking`; so số dòng và các cột tiền với legacy cùng bộ lọc (nhớ các điểm "Khác hệ thống cũ" 2, 5, 6, 7, 8); mở file Excel; NVKT gọi `/api/reports/contract-tracking/export` → 403.
- [ ] Báo cáo người dùng: kết quả test thật (số test, lỗi), phần nào chưa kiểm (chạy app, so số liệu, export lớn).
