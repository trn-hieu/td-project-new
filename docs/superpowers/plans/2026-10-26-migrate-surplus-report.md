# Migrate "Báo cáo số dư" (/report/surplus) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **Model cho subagent: CHỈ dùng `sonnet`.** Mọi lần gọi tool `Agent` (implementer, spec reviewer, code reviewer, final reviewer) phải truyền `model: "sonnet"`.

## Context

Màn "Báo cáo số dư" (`/report/surplus`, `/report/export-surplus`, popup `/api/report/surplus/contract-details`) còn là placeholder trong menu FE mới (`frontend/src/components/layout/nav-items.ts:106`, chưa có `to`). Plan này port nó sang `backend` (REST `/api/reports/surplus`) và `frontend` (route `/report/surplus`): bảng số dư theo tài khoản (tài khoản thu chi / tài khoản thu phí tiền mặt của nhân viên), dòng tổng, xuất Excel, popup chi tiết theo hợp đồng.

**Người dùng đã chốt:** (1) giữ **y nguyên** phân quyền của hệ thống cũ, **kể cả các lỗ hổng** (export mở cho nhiều chức vụ hơn trang; API chi tiết hợp đồng mở cho mọi người đã đăng nhập, không lọc chi nhánh); (2) popup chi tiết hợp đồng làm **cùng đợt**.

**Bước đầu khi thực thi:** chép plan này vào `docs/superpowers/plans/2026-10-26-migrate-surplus-report.md` (repo gốc, không commit trừ khi người dùng yêu cầu).

**Architecture:** Theo khuôn "Báo cáo theo dõi hợp đồng" (`ContractTrackingReport*`, đã có trong `backend`/`frontend` nhánh `develop`): Query → Filter (chuẩn hóa + phạm vi) → repository native SQL dùng chung `WHERE` cho trang / count / tổng / stream → service map Row → Item → controller trả `Page`, `/totals`, `/export` (SXSSF), thêm `/contract-details`. Frontend sao chép `ContractTrackingReportView.vue`, popup dùng `AppModal`.

**Tech Stack:** Spring Boot 4 · Java 21 · Spring Data JPA native query + interface projection · Apache POI SXSSF · JUnit 5/Mockito/AssertJ · `@MysqlDataJpaTest` · Vue 3 + TS · shadcn-vue · Vitest.

**Spec = hành vi hệ thống cũ** (`td-project-legacy/src/main/java/com/jvb/thanhdo/…`):
- `controller/ReportController.java` — `surplusReport` (170-195), `exportSurplusToExcel` (196-210)
- `controller/api/ReportApiController.java` — `getSurplusContractDetails` (19-34)
- `service/impl/ReportServiceImpl.java` — `getPagedSurplysReport`/`getListSurplysReportExport`/`getSurplusContractDetails` (1040-1054), `getSurplusReportTotalMoney` (1093-1130). `getSurplusReport`, `getListSurplus` là code chết, không port.
- `repository/impl/CreditAccountCustomRepositoryImpl.java` — `createSurplysReportQuery` (62-235: ngày mặc định, phạm vi, SQL `base`, HAVING, ORDER), `getSurplusContractDetails` (258-341)
- `entity/dto/request/SearchSurplusReportRequest.java`, `entity/enums/CreditAccountType.java`
- `src/main/resources/templates/report/surplus.html` (bảng 113-183, popup 246-509), `controller/component/SurplusExcelExporter.java`
- `config/SecurityConfig.java:160-167, 182`, `templates/fragment/mainnav-container.html:131`

Mẫu đã port (đọc trước task tương ứng): `backend/src/main/java/com/thanhdo/tdprojectbe/{controller/ContractTrackingReportController, service/ContractTrackingReportService, repository/ContractTrackingReportRepository, repository/ContractTrackingReportSql, model/dto/ContractTrackingReport*, common/util/export/excel/ContractTrackingReportExcelExporter}.java` + test cùng tên; `frontend/src/{api/contract-tracking-reports.ts, lib/contract-tracking-report.ts, views/report/ContractTrackingReportView.vue}` + spec; ô chọn ngày: `views/report/ValuationReportView.vue` (`DatePicker`/`DateRangePicker`); popup: `components/overlay/AppModal.vue` (dùng như `components/accounting/AccountingBatchUploadDialog.vue`).

---

## Hành vi hệ thống cũ

| Mục | Hệ thống cũ |
|---|---|
| Quyền trang | Chỉ **chức vụ** `BGD` (admin không phải BGD → forbidden). Menu: `BGD`. |
| Quyền export | Không kiểm tra trong controller; chỉ rule `/report/**`: `admin, BGD, NVKT, NVKD, NVTD, TPTD, NVPI` (KSV lọt qua vì được thêm authority `TPTD`). |
| Quyền chi tiết HĐ | `/api/report/**` không có rule → mọi người đã đăng nhập; **không lọc chi nhánh**, xem được bất kỳ `creditAccountId`. |
| Phạm vi | `isUserBranchHn` = người gọi có authority `BRANCH_HN` (chi nhánh chứa `HN`, mọi chức vụ) → mọi tài khoản của loại đã chọn. Còn lại: không có chi nhánh → rỗng; **Tài khoản thu chi**: `ca.employee_id IS NULL AND ca.branch IN (:branches)`; **Tài khoản nhân viên**: `ca.employee_id IS NOT NULL AND (ca.branch IN (:branches) OR em.branch REGEXP '(^|,)(B1|B2)(,|$)') AND (em.branch NOT REGEXP '(^|,) *HN *(,|$)' OR em.branch IS NULL)` và nếu người gọi không có `HCM` thêm điều kiện tương tự loại `HCM`. |
| Bộ lọc | Loại tài khoản (`ACCOUNTING_ACCOUNT` "Tài khoản thu chi" mặc định / `EMPLOYEE_ACCOUNT` "Tài khoản thu phí tiền mặt"); Ngày hạch toán từ–đến (ngày, không phải tháng); "Tồn quỹ lớn hơn 0". Không giới hạn năm. |
| Ngày | Mặc định kỳ = từ hôm qua 23:59:59 đến hôm nay 23:59:59 (thực tế = hôm nay vì `accounting_date` lưu đầu ngày). Có ngày: `from 00:00`, `to 23:59:00`. |
| Cột | STT · Nguồn (tài khoản thu chi: `credit_account`; nhân viên: `"Họ Tên (email)"`) · Số tồn đầu kỳ · Tồn quỹ · Tổng thu (+) · Nhận chuyển quỹ (+/-) · Chi phí khai thác (-) · Chi khác (-). |
| Công thức (bút toán `deleted = 0`) | `balance_start` = trước `from`: PT có `credit_account = ca.id` cộng, PC có `debit_account = ca.id` trừ. Trong kỳ: `total_receive` = PT, tiểu mục ≠ "Nhận chuyển quỹ"; `transfer` = PT "Nhận chuyển quỹ" − PC "Nhận chuyển quỹ"; `exploitation_fee` = PC `type = 3`; `other` = PC không phải "Nhận chuyển quỹ" và `type` ≠ 3 (hoặc null). `remain = start + receive + transfer − fee − other`. |
| Lọc dòng | "Tồn quỹ > 0" → `remain > 0`; nếu không chọn và loại nhân viên → ẩn dòng `balance_start = 0 AND remain = 0`. |
| Sắp xếp / trang | Thu chi: tài khoản công ty (`type = 2`) trước, rồi `id`; **không phân trang**. Nhân viên: `id`, 20 dòng/trang. |
| Dòng tổng | Chỉ trang cuối, cộng **toàn bộ** kết quả lọc (6 cột tiền). |
| Popup chi tiết | Chỉ tài khoản nhân viên có `remain > 0`, bấm vào ô Tồn quỹ. API bỏ qua `fromDate`, cộng dồn từ đầu đến `toDate` (mặc định hôm nay): theo `TRIM(contract_id)`, tạm ứng = PT vào tài khoản, đã nộp = PC từ tài khoản, tồn = tạm ứng − đã nộp; tên KH = `MAX(cust_name)` hồ sơ chưa xóa; sắp tồn > 0 trước, tồn giảm dần, số HĐ. Giao diện: ô tìm (số HĐ, tên KH), công tắc "Chỉ hiển thị HĐ đang dư tiền (Tồn quỹ > 0)" bật sẵn, badge "Tổng tồn", dòng tổng theo dòng đang lọc (lọc phía client). |
| Excel | Sheet "Report"; "BÁO CÁO SỐ DƯ" dòng 1 cột B gộp B1:I1; header dòng 3 từ cột B, autofilter; 8 cột như trang (độ rộng 7/25/30/30/20/20/20/20); tiền `#,##0`; không dòng tổng; file `Báo-cáo-số-dư_<dd-MM-yyyy_HH:mm:ss>.xlsx`. |

Lỗi của hệ thống cũ (được sửa, ghi vào Javadoc controller): `fromDate=` rỗng gửi từ form có thể gây 500 (`!= ""` so tham chiếu); `to` = 23:59:00 bỏ 59 giây cuối; Excel bỏ qua "Tồn quỹ > 0" và lấy ngày từ URL thay vì ô nhập; popup ghi "Kỳ hạch toán từ … đến …" nhưng số là lũy kế; popup chèn HTML không escape; nhân viên không có email thì tên "Nguồn" thành NULL; chức vụ null gây NPE.

## Quyết định đã chọn

1. **Quyền (người dùng chốt: giữ y nguyên kể cả lỗ hổng)**:
   - `/api/reports/surplus/export` → `admin, BGD, NVKT, NVKD, NVTD, TPTD, KSV, NVPI`.
   - `/api/reports/surplus/contract-details` → `authenticated()`, không lọc chi nhánh.
   - `/api/reports/surplus/**` còn lại (danh sách, `/totals`) → chỉ `BGD` (không có `admin`).
   - Matcher cụ thể đặt **trước** matcher `/**` (Spring Security lấy rule khớp đầu tiên). Comment tiếng Việt ghi rõ "giữ nguyên lỗ hổng theo yêu cầu".
   - Route FE + menu: `['BGD']`.
2. **Phạm vi** đúng như cũ: `allBranches` = chi nhánh người gọi chứa `HN` (bất kể chức vụ, khớp `BRANCH_HN` trong `service/Authorities.java`); không dùng `AssignAuth.isGlobalAdmin`/`isAdmin` (quy tắc khác).
3. **Popup** làm cùng đợt, giữ cách tính lũy kế đến `toDate`; nhãn FE sửa thành "Lũy kế đến ngày …".
4. Excel dùng **bộ lọc đã áp dụng** (kể cả "Tồn quỹ > 0"), tức sửa lỗi export của cũ.

## Global Constraints

- Subagent chỉ `model: "sonnet"`.
- `backend/`, `frontend/` là git submodule (đang ở `develop`, sạch): `git -C backend …` / `git -C frontend …`; không commit ở repo gốc, không push.
- Backend theo `backend/CLAUDE.md` và `.claude/rules/jpa-spring.md`; frontend theo `frontend/CLAUDE.md`, `frontend/DESIGN_SYSTEM.md`.
- Comment, Javadoc, message lỗi, nhãn UI: tiếng Việt có dấu.
- API base `/api/reports/surplus`; route FE path `report/surplus`, name `report-surplus`, breadcrumbs `[{ label: 'Báo cáo' }, { label: 'Báo cáo số dư' }]`.
- Native SQL được phép (ghi lý do trên repository): gộp có điều kiện `SUM(CASE …)` theo tài khoản trong bảng dẫn xuất, lọc trên cột tính toán, `REGEXP` trên chuỗi chi nhánh phân tách dấu phẩy. JOIN chuỗi giữa hai bảng (`accounting.contract_id` ↔ `p_appraisal_request.contract_id`) cần `COLLATE`.
- Lệnh: backend `cd backend && ./mvnw -q test -Dtest=<Tên>` (toàn bộ `./mvnw test`, test repository cần MySQL theo `backend/.env.example`); frontend `cd frontend && npm run lint && npm run type-check && npm run test:unit`.

### Khác hệ thống cũ (cố ý) — ghi vào Javadoc controller

1. JSON: danh sách trả `Page` (`page` từ 0); loại "Tài khoản thu chi" trả mọi dòng trong một trang như cũ (`Pageable.unpaged()`); loại nhân viên `size` mặc định 20, tối đa 100. Tổng ở `/totals` (giao diện chỉ hiện ở trang cuối).
2. Ngày tính trọn ngày: đầu kỳ `< from 00:00`, trong kỳ `[from 00:00, to + 1 ngày 00:00)`; ngày trống → hôm nay (theo `Clock` của `ClockConfig`); `fromDate > toDate` hoặc sai định dạng → 400 (cũ có thể 500).
3. Excel theo bộ lọc đã áp dụng, gồm "Tồn quỹ > 0".
4. Tên nhân viên không có email vẫn hiện `"Họ Tên ()"` thay vì trống.
5. Popup: trả mảng JSON đã escape khi hiển thị; nhãn kỳ đúng "lũy kế đến ngày".

## Review Focus

1. Bút toán chuyển quỹ giữa hai tài khoản trong phạm vi → mỗi tài khoản đúng dấu, không đếm hai lần trong một dòng; tổng = tổng các dòng (test `SurplusReportRepositoryTest.chuyển_quỹ_ghi_đúng_dấu_ở_hai_tài_khoản`, `tổng_khớp_tổng_các_dòng`).
2. Phạm vi nhân viên ngoài HN: thấy nhân viên cùng chi nhánh (qua `ca.branch` hoặc `em.branch`), không thấy nhân viên có HN, không thấy HCM khi người xem không có HCM (test `phạm_vi_tài_khoản_nhân_viên_ngoài_hn`).
3. Quyền đúng 3 tầng: NVKT gọi `/api/reports/surplus` → 403 nhưng `/export` → 200; người đăng nhập bất kỳ gọi `/contract-details` → 200; chưa đăng nhập → 401 (test `SurplusReportControllerTest`).
4. Người xem không có chi nhánh nào → rỗng, tổng 0, Excel chỉ header, không lỗi `IN ()` (test `SurplusReportFilterTest.không_chi_nhánh_thì_noAccess`).
5. Loại "Tài khoản thu chi" không bị cắt 20/100 dòng (test `SurplusReportServiceTest.thu_chi_không_phân_trang`).

---

## File Structure

**Backend** (`backend/src/main/java/com/thanhdo/tdprojectbe/`):

| File | Trách nhiệm |
|---|---|
| Modify `model/entity/Subsection.java` | Hằng `FUND_TRANSFER = "Nhận chuyển quỹ"`. |
| Modify `model/entity/Accounting.java` | Hằng `TYPE_EXPLOITATION_FEE = 3` nếu chưa có (kiểm tra trước). |
| Create `model/dto/SurplusReport{Query,Filter,Row,Item,TotalsRow,Totals,ContractDetail}.java`, `model/dto/CreditAccountKind.java` (enum) | DTO. |
| Create `repository/SurplusReportSql.java`, `repository/SurplusReportRepository.java` | SQL + truy vấn trang/stream/tổng/chi tiết. |
| Create `common/util/export/excel/SurplusReportExcelExporter.java` | File Excel. |
| Create `service/SurplusReportService.java` | Phạm vi, ngày mặc định, map, export. |
| Create `controller/SurplusReportController.java`; Modify `config/SecurityConfig.java`, `backend/CLAUDE.md` (nhắc exporter mới ở dòng streaming nếu hợp) | API, quyền. |

**Frontend** (`frontend/src/`): Create `lib/surplus-report.ts`, `api/surplus-reports.ts`, `views/report/SurplusReportView.vue`, `components/report/SurplusContractDetailModal.vue` + spec; Modify `router/index.ts`, `components/layout/nav-items.ts:106`, `router/__tests__/guard.spec.ts`, `components/layout/__tests__/layout.spec.ts`.

---

### Task 1: DTO và quy tắc phạm vi

**Files:**
- Modify: `model/entity/Subsection.java` (+ `Accounting.java` nếu thiếu hằng type 3)
- Create: DTO ở trên
- Test: `model/dto/SurplusReportFilterTest.java`, `model/dto/SurplusReportTotalsTest.java`

**Interfaces:**
- Consumes: `MsbRequestScope.branchCodes(Employee)`.
- Produces:
  - `enum CreditAccountKind { ACCOUNTING_ACCOUNT("Tài khoản thu chi"), EMPLOYEE_ACCOUNT("Tài khoản thu phí tiền mặt") }` (tên hằng giữ như cũ để URL tương thích).
  - `record SurplusReportQuery(CreditAccountKind creditAccountType, Boolean hasRemainGreaterThanZero, @DateTimeFormat(iso=DATE) LocalDate fromDate, LocalDate toDate)` — constructor: type null → `ACCOUNTING_ACCOUNT`; `@AssertTrue isDateRangeValid()` (chỉ khi cả hai có) message "Từ ngày không được sau đến ngày".
  - `record SurplusReportFilter(boolean employeeAccounts, boolean remainPositive, boolean allBranches, List<String> branches, String branchRegex, boolean hideHcm, LocalDateTime from, LocalDateTime toExclusive)` với `noAccess()` (`!allBranches && branches.isEmpty()`) và `static of(SurplusReportQuery q, Employee caller, LocalDate today)`:
    - `branches = MsbRequestScope.branchCodes(caller)` (bỏ phần tử rỗng); `allBranches = branches` chứa `"HN"` (không phân biệt hoa thường); khi `allBranches` thì `branches = List.of()`.
    - `branchRegex = "(^|,)(" + String.join("|", branches) + ")(,|$)"` — chỉ nhận mã khớp `[A-Za-z0-9_]+` (mã khác bị bỏ khỏi regex) để không chèn ký tự regex; rỗng khi `allBranches`/không chi nhánh (giá trị giả `"^$"` để SQL hợp lệ).
    - `hideHcm = !allBranches && branches không chứa "HCM"` (không phân biệt hoa thường).
    - `from = (q.fromDate ?? today).atStartOfDay()`, `toExclusive = (q.toDate ?? today).plusDays(1).atStartOfDay()`.
  - `interface SurplusReportRow` (projection): `Long getId(), String getCreditAccount(), Long getBalanceStart(), getTotalReceive(), getTotalTransfer(), getTotalExploitationFee(), getTotalOther(), getRemain()`.
  - `record SurplusReportItem(long id, String creditAccount, long balanceStart, long remain, long totalReceive, long totalTransfer, long totalExploitationFee, long totalOther)` + `static from(SurplusReportRow)` (null → 0).
  - `interface SurplusReportTotalsRow` (`BigDecimal` cho 6 cột) và `record SurplusReportTotals(BigDecimal balanceStart, remain, totalReceive, totalTransfer, totalExploitationFee, totalOther)` với `ZERO` và `from(row)` (null → 0).
  - `interface SurplusReportContractDetail` (projection): `getContractId, getCustName, getAdvanceAmount, getPaidAmount, getRemain` (hoặc record + map trong service).

- [ ] **Step 0: Tạo nhánh** `git -C backend switch -c feat/surplus-report` và `git -C frontend switch -c feat/surplus-report` từ `develop`. Nếu submodule có thay đổi chưa commit thì dừng và báo người dùng.
- [ ] **Step 1: Test thất bại `SurplusReportFilterTest`** (helper `employee(position, branch)`):
  - `chi_nhánh_có_hn_xem_tất_cả_bất_kể_chức_vụ`: `("NVKT","DN, hn")` → `allBranches` true, `branches` rỗng, `noAccess` false.
  - `bgd_chi_nhánh_có_regex_và_ẩn_hcm`: `("BGD"," DN , HP ")` → `branches` `["DN","HP"]`, `branchRegex` `"(^|,)(DN|HP)(,|$)"`, `hideHcm` true; `("BGD","HCM")` → `hideHcm` false.
  - `không_chi_nhánh_thì_noAccess`: branch null / `" , "` → `noAccess` true.
  - `ngày_trống_là_hôm_nay_và_tính_trọn_ngày`: today 2026-10-26, from/to null → `from` 2026-10-26T00:00, `toExclusive` 2026-10-27T00:00; from 2026-10-01, to 2026-10-15 → `toExclusive` 2026-10-16T00:00.
  - `loại_mặc_định_thu_chi`: type null → `employeeAccounts` false; `hasRemainGreaterThanZero` null → `remainPositive` false.
  - `mã_chi_nhánh_lạ_không_vào_regex`: `("BGD","DN,A|B")` → regex chỉ có `DN`.
- [ ] **Step 2: Test thất bại `SurplusReportTotalsTest`**: mọi getter null → mọi số 0.
- [ ] **Step 3: Chạy** `./mvnw -q test -Dtest='SurplusReport*Test'` → FAIL.
- [ ] **Step 4: Tạo** hằng + DTO (Javadoc tham số như `ContractTrackingReportFilter`, ghi nguồn quy tắc cũ).
- [ ] **Step 5: Chạy lại** → PASS. **Step 6: Commit** `feat(report): DTO và phạm vi báo cáo số dư`.

### Task 2: SQL và repository

**Files:**
- Create: `repository/SurplusReportSql.java`, `repository/SurplusReportRepository.java`
- Test: `repository/SurplusReportRepositoryTest.java` (`@MysqlDataJpaTest`, dữ liệu qua `RepositoryTestData`; tài khoản/nhân viên/chi nhánh có tiền tố MARK để cô lập — vì báo cáo không có từ khóa, test lọc bằng chi nhánh MARK riêng, ví dụ `"ZT1"`, và người xem có chi nhánh đó)

**Interfaces:**
- Produces: `SurplusReportRepository extends Repository<CreditAccount, Long>`:
  - `default Page<SurplusReportRow> search(filter, pageable)`, `default Stream<SurplusReportRow> stream(filter)`, `default SurplusReportTotals totals(filter)` (`noAccess` → `Page.empty` / `Stream.empty` / `ZERO`), cùng `findPage` (có `countQuery`), `streamAll` (hint fetchSize `-2147483648`), `sums` — khuôn `ContractTrackingReportRepository`.
  - `List<SurplusReportContractDetail> contractDetails(@Param("creditAccountId") Long id, @Param("toExclusive") LocalDateTime toExclusive)`.

SQL (`final class SurplusReportSql`, chỉ ghép hằng; tham số SpEL `:#{#filter.x}`; tên tiểu mục `:#{T(com.thanhdo.tdprojectbe.model.entity.Subsection).FUND_TRANSFER}`; dùng bảng dẫn xuất thay cho CTE để Spring Data ghép `LIMIT` và `countQuery` an toàn):

```
BASE = SELECT ca.id, ca.employee_id, ca.credit_account, ca.type,
   COALESCE(SUM(CASE WHEN acc.accounting_date < :from AND acc.accounting_mode='PT' AND acc.credit_account=ca.id THEN acc.amount_of_money
                     WHEN acc.accounting_date < :from AND acc.accounting_mode='PC' AND acc.debit_account=ca.id THEN -acc.amount_of_money
                     ELSE 0 END),0) balance_start,
   … total_receive / transfer_in / transfer_out / total_exploitation_fee / total_other đúng như cũ,
     với "trong kỳ" = acc.accounting_date >= :from AND acc.accounting_date < :toExclusive
 FROM credit_account ca
 LEFT JOIN employees em_base ON em_base.employee_id = ca.employee_id
 LEFT JOIN accounting acc ON acc.deleted = 0 AND (acc.credit_account = ca.id OR acc.debit_account = ca.id)
                         AND acc.accounting_date < :toExclusive
 WHERE ca.deleted = 0
   AND ((:employeeAccounts = FALSE AND ca.employee_id IS NULL) OR (:employeeAccounts = TRUE AND ca.employee_id IS NOT NULL))
   AND (:allBranches = TRUE
        OR (:employeeAccounts = FALSE AND ca.branch IN (:branches))
        OR (:employeeAccounts = TRUE
            AND (ca.branch IN (:branches) OR em_base.branch REGEXP :branchRegex)
            AND (em_base.branch IS NULL OR em_base.branch NOT REGEXP '(^|,) *HN *(,|$)')
            AND (:hideHcm = FALSE OR em_base.branch IS NULL OR em_base.branch NOT REGEXP '(^|,) *HCM *(,|$)')))
 GROUP BY ca.id, ca.employee_id, ca.credit_account, ca.type
REPORT = SELECT b.id, b.type,
   CASE WHEN b.employee_id IS NULL THEN b.credit_account
        ELSE CONCAT(CONCAT_WS(' ', em.first_name, em.last_name), ' (', COALESCE(em.email,''), ')') END creditAccount,
   b.balance_start balanceStart, b.total_receive totalReceive, (b.transfer_in - b.transfer_out) totalTransfer,
   b.total_exploitation_fee totalExploitationFee, b.total_other totalOther,
   (b.balance_start + b.total_receive + b.transfer_in - b.transfer_out - b.total_exploitation_fee - b.total_other) remain
 FROM (BASE) b LEFT JOIN employees em ON em.employee_id = b.employee_id
FILTERED = FROM (REPORT) r WHERE (:remainPositive = TRUE AND r.remain > 0)
                              OR (:remainPositive = FALSE AND (:employeeAccounts = FALSE OR NOT (r.balanceStart = 0 AND r.remain = 0)))
ROWS   = SELECT r.id, r.creditAccount, … FILTERED ORDER BY CASE WHEN :employeeAccounts = FALSE AND r.type = 2 THEN 0 ELSE 1 END, r.id
COUNT  = SELECT COUNT(*) FILTERED
TOTALS = SELECT SUM(r.balanceStart) balanceStart, SUM(r.remain) remain, … FILTERED
CONTRACT_DETAILS = như cũ, thay `accounting_date <= :toDate` bằng `< :toExclusive`,
   JOIN `ar.contract_id = cb.contract_id` kèm COLLATE theo quy ước backend/CLAUDE.md.
```

Lưu ý dấu: cũ lưu `transfer_out` âm rồi cộng; ở đây lưu dương rồi trừ — kết quả như nhau. Kiểm tra `BranchAuthorities`/collation của `credit_account.branch` và `employees.branch` khi viết test.

- [ ] **Step 1: Test thất bại** (mỗi test tự chèn dữ liệu):
  - `công_thức_các_cột`: tài khoản thu chi chi nhánh ZT1; trước kỳ PT 1000, PC 200; trong kỳ PT 500 "Hợp đồng", PT 300 "Nhận chuyển quỹ", PC 100 "Nhận chuyển quỹ", PC 50 type 3, PC 70 type 1 "Chi khác", 999 đã xóa, 1 bút toán sau kỳ → `balanceStart` 800, `totalReceive` 500, `totalTransfer` 200, `totalExploitationFee` 50, `totalOther` 70, `remain` 1380.
  - `chuyển_quỹ_ghi_đúng_dấu_ở_hai_tài_khoản`: PC từ A sang B cùng bút toán (`debit_account` A, `credit_account` B) — theo cách hệ thống đang ghi (đọc `AccountingPosting` để dựng đúng) → A trừ, B cộng.
  - `mốc_ngày_trọn_ngày`: bút toán đúng `from` vào trong kỳ; đúng `toExclusive` bị loại; trước `from` vào đầu kỳ.
  - `thu_chi_công_ty_đứng_đầu_rồi_theo_id`; `nhân_viên_ẩn_dòng_không_đầu_kỳ_không_tồn`; `tồn_lớn_hơn_không`.
  - `phạm_vi_thu_chi_theo_chi_nhánh`; `phạm_vi_tài_khoản_nhân_viên_ngoài_hn` (nhân viên ZT1 thấy; nhân viên `"ZT1,HN"` không thấy; nhân viên `"ZT1,HCM"` không thấy khi người xem không có HCM; tài khoản `ca.branch = ZT1` của nhân viên chi nhánh khác vẫn thấy).
  - `tên_nguồn_nhân_viên_không_email`; `tài_khoản_đã_xóa_bị_loại`.
  - `tổng_khớp_tổng_các_dòng`; `noAccess_không_chạy_truy_vấn`; `stream_trả_cùng_các_dòng_như_trang` (trong transaction như test ContractTracking).
  - `chi_tiết_hợp_đồng_lũy_kế_đến_ngày`: PT/PC nhiều HĐ, contract_id có khoảng trắng → gộp theo TRIM, tên KH từ hồ sơ chưa xóa, thứ tự tồn > 0 trước, bỏ bút toán sau `toExclusive` và đã xóa.
- [ ] **Step 2: Chạy** `./mvnw -q test -Dtest=SurplusReportRepositoryTest` → FAIL.
- [ ] **Step 3: Tạo** `SurplusReportSql`, `SurplusReportRepository` (Javadoc: lý do native, quy tắc phạm vi giữ nguyên cũ).
- [ ] **Step 4: Chạy lại** → PASS (cần MySQL; nếu không có DB thì ghi rõ chưa chạy, không coi là pass).
- [ ] **Step 5: Commit** `feat(report): truy vấn báo cáo số dư`.

### Task 3: File Excel

**Files:** Create `common/util/export/excel/SurplusReportExcelExporter.java`; Test `…/SurplusReportExcelExporterTest.java`

**Interfaces:** `public static void export(Consumer<Consumer<SurplusReportItem>> dataSource, OutputStream out)`; `static final String TITLE = "BÁO CÁO SỐ DƯ"`; sheet "Report".

Cột (độ rộng như cũ): STT 7 · Nguồn 25 · Số tồn đầu kỳ 30 · Tồn quỹ 30 · Tổng thu (+) 20 · Nhận chuyển quỹ (+/-) 20 · Chi phí khai thác (-) 20 · Chi khác (-) 20. Tiền `styleRowPrice` (`#,##0`, như cũ). Không dòng tổng.

- [ ] **Step 1: Test thất bại** (mẫu `ContractTrackingReportExcelExporterTest`): `bố_cục_như_cũ` (B1 = TITLE, gộp `B1:I1`, autofilter `B3:I3`, header đúng thứ tự); `ghi_số_tiền_kể_cả_âm_là_số`; `nhiều_dòng_hơn_cửa_sổ_streaming_đều_được_ghi` (300 dòng).
- [ ] **Step 2: Chạy** → FAIL. **Step 3: Cài đặt.** **Step 4: Chạy lại** → PASS. **Step 5: Commit** `feat(report): file Excel báo cáo số dư`.

### Task 4: Service

**Files:** Create `service/SurplusReportService.java`; Test `service/SurplusReportServiceTest.java`

**Interfaces:**
- Consumes: repository (Task 2), exporter (Task 3), `CurrentEmployeeService.require`, `Clock` (bean `ClockConfig`).
- Produces:
  - `Page<SurplusReportItem> search(Long callerId, SurplusReportQuery q, Pageable p)` — loại thu chi dùng `Pageable.unpaged()` (như cũ: không phân trang).
  - `SurplusReportTotals totals(Long callerId, SurplusReportQuery q)`.
  - `@Transactional(readOnly = true) void export(Long callerId, SurplusReportQuery q, OutputStream out)` — `try (Stream<…> rows = repository.stream(filter))`.
  - `List<SurplusReportContractDetail> contractDetails(Long creditAccountId, LocalDate toDate)` — `toDate` null → hôm nay; **không** kiểm tra phạm vi (giữ nguyên cũ, Javadoc ghi rõ đây là quyết định của người dùng).

- [ ] **Step 1: Test thất bại** (Mockito, mẫu `ContractTrackingReportServiceTest`): `search_gắn_phạm_vi_và_ngày_mặc_định` (Clock cố định 2026-10-26; captor filter); `thu_chi_không_phân_trang` (truyền `PageRequest.of(0,20)` với loại thu chi → repository nhận `Pageable.unpaged()`; loại nhân viên giữ nguyên pageable); `noAccess_trả_rỗng`; `export_ghi_file_từ_stream_và_đóng_stream`; `chi_tiết_hợp_đồng_ngày_trống_là_hôm_nay`.
- [ ] **Step 2: Chạy** → FAIL. **Step 3: Cài đặt.** **Step 4: Chạy lại** → PASS. **Step 5: Commit** `feat(report): service báo cáo số dư`.

### Task 5: Controller, quyền

**Files:** Create `controller/SurplusReportController.java`; Modify `config/SecurityConfig.java` (cạnh khối `/api/reports/contract-tracking/**`); Test `controller/SurplusReportControllerTest.java`

**Interfaces:** `GET /api/reports/surplus` (`page` 0, `size` 20, 1..100), `GET /totals`, `GET /export` → `"Báo-cáo-số-dư_" + dd-MM-yyyy_HH-mm-ss + ".xlsx"`, `GET /contract-details?creditAccountId=&toDate=` (`creditAccountId` bắt buộc; trả `List`). Không cần `/filter-options` (bộ lọc tĩnh — YAGNI).

Matcher (thứ tự bắt buộc):
```java
// Báo cáo số dư — giữ nguyên quyền hệ thống cũ, kể cả lỗ hổng (theo yêu cầu):
// export chỉ chịu rule chung /report/** của cũ; chi tiết hợp đồng chỉ cần đăng nhập; trang chỉ chức vụ BGD.
.requestMatchers("/api/reports/surplus/export")
.hasAnyAuthority("admin", Positions.BGD, Positions.NVKT, Positions.NVKD, Positions.NVTD, Positions.TPTD, Positions.KSV, Positions.NVPI)
.requestMatchers("/api/reports/surplus/contract-details").authenticated()
.requestMatchers("/api/reports/surplus/**").hasAuthority(Positions.BGD)
```
(Kiểm tra `"/api/reports/surplus/**"` có khớp chính `/api/reports/surplus` không; nếu không, thêm cả `"/api/reports/surplus"`.)

- [ ] **Step 1: Test thất bại** (mẫu `ContractTrackingReportControllerTest`):
  - `cần_đăng_nhập` cho cả 4 path.
  - `trang_và_tổng_chỉ_bgd`: `admin`, NVKT… → 403 ở `""`, `/totals`; BGD → 200.
  - `export_mở_cho_các_chức_vụ_như_cũ`: mỗi authority trong danh sách → 200; authority lạ (vd. chỉ `ROLE_X`) → 403.
  - `chi_tiết_hợp_đồng_chỉ_cần_đăng_nhập`: authority lạ → 200; thiếu `creditAccountId` → 400.
  - `danh_sách_bind_bộ_lọc`: `creditAccountType=EMPLOYEE_ACCOUNT&hasRemainGreaterThanZero=true&fromDate=2026-10-01&toDate=2026-10-15&page=1&size=10` → captor đúng query, `PageRequest.of(1,10)`.
  - `tham_số_sai_trả_400`: `fromDate=01/10/2026`, from > to, `creditAccountType=X`, `size=101`, `page=-1`.
  - `export_trả_file_tên_đúng` (Content-Type xlsx, `filename*=UTF-8''B%C3%A1o-c%C3%A1o-s%E1%BB%91-d%C6%B0_`).
- [ ] **Step 2: Chạy** → FAIL. **Step 3: Cài đặt** controller (Javadoc đủ mục "Khác hệ thống cũ" + ghi chú giữ nguyên lỗ hổng quyền) + matcher. **Step 4: Chạy** `./mvnw -q test -Dtest='SurplusReport*Test'` → PASS.
- [ ] **Step 5: Toàn backend** `./mvnw test` → `BUILD SUCCESS`, 0 failure. **Step 6: Commit** `feat(report): API báo cáo số dư`.

### Task 6: Frontend — lib và api

**Files:** Create `frontend/src/lib/surplus-report.ts`, `frontend/src/api/surplus-reports.ts`; Test `lib/__tests__/surplus-report.spec.ts`, `api/__tests__/surplus-reports.spec.ts`

**Interfaces:**
- `SURPLUS_REPORT_AUTHORITIES = ['BGD']`, `SURPLUS_REPORT_PAGE_SIZE = 20`, `CREDIT_ACCOUNT_TYPE_OPTIONS = [{ value: 'ACCOUNTING_ACCOUNT', label: 'Tài khoản thu chi' }, { value: 'EMPLOYEE_ACCOUNT', label: 'Tài khoản thu phí tiền mặt' }]`.
- Types `SurplusReportItem`, `SurplusReportTotals`, `SurplusContractDetail`, `SurplusReportFilter` (`creditAccountType, hasRemainGreaterThanZero, fromDate, toDate`), `SurplusReportQuery extends Filter` (`page`, `size`).
- `listSurplusReport(query, signal?)`, `getSurplusReportTotals(filter, signal?)`, `exportSurplusReport(filter)`, `getSurplusContractDetails(creditAccountId, toDate, signal?)` — khuôn `api/contract-tracking-reports.ts` (không gửi tham số rỗng; `hasRemainGreaterThanZero` chỉ gửi khi `true`).
- Hàm thuần cho popup: `filterContractDetails(rows, keyword, onlyPositive)` (tìm không phân biệt hoa thường trên số HĐ, tên KH) và `sumContractDetails(rows)`.

- [ ] **Step 1: Spec thất bại** (URL, bỏ tham số rỗng, lọc/tổng popup). **Step 2: Chạy** `npx vitest run src/lib/__tests__/surplus-report.spec.ts src/api/__tests__/surplus-reports.spec.ts` → FAIL. **Step 3: Cài đặt.** **Step 4: Chạy lại** → PASS. **Step 5: Commit** `feat(report): api báo cáo số dư`.

### Task 7: Frontend — màn hình, popup, route, menu

**Files:**
- Create: `views/report/SurplusReportView.vue`, `components/report/SurplusContractDetailModal.vue` (dùng `AppModal`), `views/__tests__/SurplusReportView.spec.ts`
- Modify: `router/index.ts` (route sau `report-contract-tracking`, comment "Màn /report/surplus (Báo cáo → Báo cáo số dư) của hệ thống cũ: chỉ chức vụ BGD"), `components/layout/nav-items.ts:106` → thêm `to: '/report/surplus'`, `router/__tests__/guard.spec.ts`, `components/layout/__tests__/layout.spec.ts`

Sao chép bố cục `ContractTrackingReportView.vue` (bộ lọc nháp → áp dụng, đồng bộ URL, tổng tải riêng và chỉ hiện ở trang cuối, export dùng bộ lọc đã áp dụng). Bộ lọc: Loại tài khoản (select), Ngày hạch toán từ–đến (ô chọn ngày như `ValuationReportView`, để trống = hôm nay, có ghi chú nhỏ "Để trống: hôm nay"), checkbox "Tồn quỹ lớn hơn 0". Bảng 8 cột đúng thứ tự cũ, tiền định dạng `1.234.567`, căn phải. Loại thu chi: ẩn phân trang. Loại nhân viên và `remain > 0`: ô Tồn quỹ là nút mở popup.

Popup: tiêu đề "Chi tiết số dư theo Hợp đồng - <tên>", dòng "Lũy kế đến ngày <toDate đã áp dụng hoặc hôm nay>", ô tìm "Tìm theo số hợp đồng, tên khách hàng...", công tắc "Chỉ hiển thị HĐ đang dư tiền (Tồn quỹ > 0)" bật sẵn, badge "Tổng tồn", bảng STT · Số hợp đồng · Tên khách hàng · Số tiền tạm ứng · Số tiền đã nộp · Tồn quỹ (xanh > 0, xám = 0, đỏ < 0 theo token của `DESIGN_SYSTEM.md`), dòng "Tổng cộng" theo dòng đang lọc. Hiển thị bằng interpolation của Vue (đã escape).

- [ ] **Step 1: Spec thất bại** (khuôn `ContractTrackingReportView.spec.ts`): render 8 cột đúng thứ tự; loại thu chi không có phân trang, loại nhân viên có; tick "Tồn quỹ > 0" + chọn ngày rồi Tìm kiếm → API nhận đúng tham số, URL đồng bộ; dòng tổng chỉ ở trang cuối; Xuất Excel gửi bộ lọc đã áp dụng (gồm `hasRemainGreaterThanZero`); bấm Tồn quỹ của tài khoản nhân viên → gọi `getSurplusContractDetails(id, toDate)` và hiện popup, công tắc/ô tìm lọc dòng và cập nhật tổng.
- [ ] **Step 2: Chạy** `npx vitest run src/views/__tests__/SurplusReportView.spec.ts` → FAIL. **Step 3: Cài đặt** view, popup, route, menu; cập nhật `guard.spec.ts` (BGD vào được, admin không phải BGD và NVKT bị chặn) và `layout.spec.ts` (menu có link cho BGD). **Step 4: Chạy lại** → PASS.
- [ ] **Step 5: Toàn frontend** `npm run lint && npm run type-check && npm run test:unit` → sạch (lỗi có từ trước không liên quan thì ghi rõ trong báo cáo).
- [ ] **Step 6: Commit** `feat(report): màn báo cáo số dư`.

### Task 8: Kiểm tra cuối

- [ ] `cd backend && ./mvnw test` → `BUILD SUCCESS`; `cd frontend && npm run lint && npm run type-check && npm run test:unit`.
- [ ] Chạy app (nếu có môi trường): BGD HN, BGD chi nhánh vào `/report/surplus`, cả hai loại tài khoản; so từng cột với legacy cùng khoảng ngày (nhớ "Khác hệ thống cũ" 2, 4); mở popup; mở file Excel. Gọi thẳng API: NVKT `/api/reports/surplus` → 403, `/export` → 200; người đăng nhập bất kỳ `/contract-details` → 200.
- [ ] Code review (`superpowers:requesting-code-review`, subagent `model: "sonnet"`).
- [ ] Báo cáo người dùng: kết quả test thật (số test, lỗi), phần chưa kiểm (chạy app, so số liệu), nhắc lại các lỗ hổng quyền được giữ theo yêu cầu.
