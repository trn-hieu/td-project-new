# Migrate "Báo cáo kế toán" (/report/declaration) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **Subagent CHỈ dùng `sonnet`.** Mọi lần gọi tool `Agent` truyền `model: "sonnet"`.
> Làm thẳng trên `develop` của hai submodule (`git -C backend …`, `git -C frontend …`). Không push, không commit repo gốc. Trước khi bắt đầu kiểm tra cây làm việc sạch; không sạch thì dừng và báo.
> **Bước đầu khi thực thi:** chép file này vào `docs/superpowers/plans/2026-10-27-migrate-declaration-report.md` (không commit).

## Context

Mục "Báo cáo kế toán" trong menu FE mới (`frontend/src/components/layout/nav-items.ts:109`, `authorities: ['admin']`) còn là placeholder, chưa có `to`. Màn cũ là `/report/declaration` (trong code cũ gọi "báo cáo kê khai"), **khác** "Báo cáo thu chi" (`/report/accounting`, đã có plan riêng). Màn liệt kê hồ sơ thẩm định đã xuất hóa đơn hoặc đã thu hợp đồng vào TK công ty, kèm giá trị hợp đồng, số đã CK TK công ty, đã thu khác, công nợ, số/ngày hóa đơn. Có tổng ở trang cuối và xuất Excel.

Plan port màn này sang:
- **Backend:** `GET /api/reports/declaration`, `/totals`, `/export`.
- **Frontend:** route `/report/declaration`.

**Người dùng đã chốt:**
1. Trang **và** export đều chỉ `admin` (đóng lỗ hổng export của cũ).
2. Giữ y nguyên cách JOIN `juridical` của cũ (không lọc `deleted`, một hợp đồng có nhiều `juridical` → nhiều dòng, tổng cộng lặp) để số liệu đối chiếu được.
3. Làm thẳng trên `develop`.

**Architecture:** khuôn `DebtReport*` (cùng bảng gốc `p_appraisal_request ap`, đã có giá trị hợp đồng, số đã thanh toán, seri chứng thư) + `SurplusReport*` (luồng Query → Filter → Sql/Repository → Service → Controller, SXSSF). FE sao chép `CertificateReportView.vue` (view duy nhất có 2 `DateRangePicker` + từ khóa + tổng + export).

**Tech Stack:** Spring Boot 4 · Java 21 · native query + interface projection · Apache POI SXSSF · JUnit 5/Mockito/AssertJ · `@MysqlDataJpaTest` · Vue 3 + TS · shadcn-vue · Vitest.

**Spec = hệ thống cũ** (`td-project-legacy/src/main/java/com/jvb/thanhdo/…`):
- `controller/ReportController.java` — `getDeclarationReport` (612-632, kiểm `role == admin`), `exportDeclarationReportExcel` (634-646, không kiểm role)
- `service/impl/ReportServiceImpl.java` — `getDeclarationReport` (1883), `getDeclarationReportTotalMoney` (1909), `getDeclarationReportExport` (1924), `declarationKeyword` (1948), `initSearchRequestWithNullDate` (1951), `parseDateTimeOrNull` (1981), `getPaidAmount` (2023)
- `repository/AppraisalRequestRepository.java:761-827` — `SIGNED_AMOUNT`, `DECLARATION_ACCOUNTING_JOIN`, `DECLARATION_PAID_JOIN`, `DECLARATION_WHERE/SELECT/FROM/ORDER`
- `repository/AppraisalCertificateRepository.java:28` — `getCertificateSeriByContractIdAndDeleted` (`LIMIT 1`)
- `common/CommonConstants.java:251-272` — `Subsection.CONTRACT`, `PAYMENT_SUBSECTIONS`, `OTHER_TRANSFER_SUBSECTIONS_SQL`
- `controller/component/DeclarationReportExporter.java`, `entity/dto/DeclarationReportDTO.java`
- `resources/templates/report/declaration.html`; JS `static/js/appraisal-request/appraisal-request.js` (`renderLegalDocument`), `static/js/report/certificate.js` (`renderJsonArrayString`)
- `config/SecurityConfig.java:160` (rule chung `/report/**`), `fragment/mainnav-container.html:133`

---

## Hành vi hệ thống cũ

| Mục | Hệ thống cũ |
|---|---|
| Quyền | Trang: `role == admin`, ngược lại trang forbidden. Menu: `admin`. Export: chỉ rule `/report/**` (admin, BGD, NVKT, NVKD, NVTD, TPTD, NVPI). |
| Bảng gốc | `p_appraisal_request ar` `LEFT JOIN juridical ju ON ju.contract_id = ar.contract_id` (**không** lọc `ju.deleted`) `LEFT JOIN acc` (tổng hợp bút toán theo hợp đồng). |
| `acc` (theo `contract_id`) | Bút toán `accounting_mode IN (PT, PC)`, `contract_id IS NOT NULL`, `deleted = 0`; `LEFT JOIN credit_account ca` theo `credit_account` (PT) / `debit_account` (PC), `ca.deleted = 0 OR NULL`. Số có dấu: PT +, PC −.<br>• `totalCompanyTransfer` = Σ có dấu khi `ca.type = 2` (mọi `ac.type`).<br>• `otherTransfer` = Σ có dấu khi `(ca.type != 2 OR NULL) AND ac.type = 1 AND subsection IN ('Hoàn tạm ứng','Hợp đồng')`.<br>• `hasContractCk` = có bút toán tiểu mục 'Hợp đồng' vào `ca.type = 2`.<br>• `created` = `MAX(ac.created)`. |
| Điều kiện hiển thị | `ar.delete_flag = 0` **và** (`hasContractCk = 1` **hoặc** `ju.bill_id` khác NULL/rỗng). |
| Ngày xuất hóa đơn | `ju.bill_date` từ `from 00:00` đến `to 23:59`; trống thì không lọc (dòng `bill_date` NULL vẫn hiện). |
| Ngày tạo hồ sơ | `ar.create_date` từ `contractFrom 00:00` đến `contractTo 23:59`; trống thì không lọc. |
| Từ khóa | trim, `LIKE %kw%` (không escape) trên `ar.contract_id`, `cust_name`, `company_name`, `ju.bill_id`. Placeholder "Số hợp đồng, Khách hàng, Số hóa đơn". |
| Giới hạn năm | Không có. |
| Sắp xếp / trang | `created DESC, billDate DESC, ar.id DESC`; 20 dòng/trang. |
| Cột (14) | STT · Số hợp đồng · Khách hàng (`request_type = '1'` → `cust_name`, ngược lại `company_name`) · Tài sản thẩm định (`legal_documents` render bằng `renderLegalDocument`) · Số seri chứng thư (seri của **một** chứng thư chưa xóa, `LIMIT 1`, render mảng JSON) · Người lấy số (`business_source`) · Nguồn (`source`) · Giá trị hợp đồng (`valuation_fee`, rỗng/0 → 0) · Đã CK TK Cty · Đã thu khác · Công nợ (= giá trị HĐ − đã thanh toán) · Số hóa đơn · Ngày xuất hóa đơn (`dd/MM/yyyy`) · Ghi chú (`ju.note`). |
| Đã thanh toán | `sumPaidAmountByContractId`: bút toán chưa xóa, `type = 1`, tiểu mục ∈ `PAYMENT_SUBSECTIONS`, PT +, PC −. |
| Dòng tổng | Chỉ ở trang cuối, trên toàn bộ kết quả: "Tổng cộng:" (span 7) · Σ giá trị HĐ · Σ CK TK Cty · Σ đã thu khác · Σ công nợ (= Σ giá trị HĐ − Σ đã thanh toán). |
| Excel | Tiêu đề "BÁO CÁO KẾ TOÁN" ở B1; header dòng 3 từ cột B; 14 cột, độ rộng 7/30/30/30/25/25/15/20/20/20/20/15/15/30; tiền `#,##0`; ngày chuỗi `dd/MM/yyyy`; không dòng tổng. Header cột 9 ghi "Đã CK T Cty" (lỗi chính tả). Tài sản/seri render bằng `CommonUtils.renderLegalDocumentJson` ("1) …" mỗi dòng). Tên file `Báo-cáo-kê-khai_<dd-MM-yyyy_HH:mm:ss>.xlsx`. |

## Quyết định đã chọn

1. **Quyền:** `/api/reports/declaration` và `/api/reports/declaration/**` → `hasAuthority("admin")`, đặt cạnh khối TSSS / Ban lãnh đạo (`backend/.../config/SecurityConfig.java` ~L163-172). Route FE `authorities: ['admin']` inline như TSSS (`frontend/src/router/index.ts` ~L581). Menu chỉ thêm `to`.
2. **JOIN `juridical` giữ nguyên cũ** (không lọc `deleted`, có thể nhân dòng). Ghi rõ trong Javadoc SQL và controller là cố ý.
3. **Ngày:** `[from, toExclusive)` trọn ngày, nhưng **có cờ** để trống ngày thì không lọc (`(:filterBillDate = FALSE OR (ju.bill_date >= :billFrom AND ju.bill_date < :billToExclusive))`). Không dùng min/max của `ReportPeriod`, vì `ju.bill_date >= MIN` sẽ loại dòng `bill_date` NULL (cũ vẫn hiện).
4. **Giá trị hợp đồng:** tái dùng `DebtReportSql.CONTRACT_VALUE` / `wholeNumber(...)` (VARCHAR → phần nguyên, không phải số → 0). **Đã thanh toán:** tái dùng `DebtReportSql.PAY_JOIN` + `PAID`. Hai lớp cùng package `repository`, hằng package-private nên dùng thẳng được.
5. **Seri chứng thư:** tái dùng JOIN `cert` (`JSON_ARRAYAGG`) như `DebtReportSql.FROM_ALL` + làm phẳng như `DebtReportService.seris`. Để không chép logic, chuyển `seris(String)` thành helper package-private dùng chung (ví dụ `CertificateNumbers.flatten(String)` trong `service/CertificateNumbers.java`) và cho `DebtReportService` gọi lại. Khác cũ: lấy seri của mọi chứng thư chưa xóa thay vì một chứng thư ngẫu nhiên.
6. **Tài sản thẩm định:** `LegalDocumentAddresses.describe(new LegalDocumentItemListConverter().convertToEntityAttribute(raw))` như `BusinessReportService` (L55). API trả `List<String>`; Excel nối bằng `ValuationReportItem.numbered`.
7. **Hằng mới:** `Subsection.OTHER_TRANSFER_SUBSECTIONS = List.of(ADVANCE_REFUND, CONTRACT)` (Javadoc: cột "Đã thu khác" của báo cáo kế toán). Loại tài khoản công ty lấy qua `T(...CreditAccount).TYPE_COMPANY_ACCOUNT`; tiểu mục qua `T(...Subsection).CONTRACT`.
8. **COLLATE:** mọi so sánh `contract_id` giữa `p_appraisal_request`, `juridical`, `accounting` có `COLLATE utf8mb3_unicode_ci` như `AccountingReportSql` (L15, `backend/CLAUDE.md:50`).

## Global Constraints

- Subagent chỉ dùng `model: "sonnet"`.
- Backend theo `backend/CLAUDE.md`, `.claude/rules/jpa-spring.md`; Frontend theo `frontend/CLAUDE.md`, `frontend/DESIGN_SYSTEM.md`.
- Comment, Javadoc, message lỗi, nhãn UI: tiếng Việt có dấu.
- API base `/api/reports/declaration`. Route FE: path `report/declaration`, name `report-declaration`, breadcrumbs `[{ label: 'Báo cáo' }, { label: 'Báo cáo kế toán' }]`.
- Native SQL được phép; ghi lý do trên repository: bảng dẫn xuất tổng hợp theo hợp đồng + `COLLATE` + chung một `WHERE` cho trang, count, tổng, stream.
- Lệnh: Backend `cd backend && ./mvnw -q test -Dtest=<Tên>` (repository test cần MySQL theo `backend/.env.example`; không có DB thì ghi rõ chưa chạy). Frontend `cd frontend && npm run lint && npm run type-check && npm run test:unit`.

### Khác hệ thống cũ (cố ý) — ghi vào Javadoc controller

1. Export chỉ admin (cũ mở cho mọi chức vụ trong `/report/**`).
2. JSON trả `Page` (`page` từ 0, `size` mặc định 20, tối đa 100); tổng ở `/totals`, FE chỉ hiện ở trang cuối.
3. Ngày trọn ngày (cũ mất 59 giây cuối ngày); sai định dạng hoặc from > to → 400 (cũ 500).
4. Từ khóa escape `\ % _` (`LikePattern.contains`).
5. Giá trị hợp đồng chỉ lấy phần nguyên của chuỗi số (cũ để MySQL ép kiểu ngầm / ModelMapper).
6. Seri chứng thư lấy mọi chứng thư chưa xóa, cố định giữa các lần tải.
7. Công nợ luôn tính (cũ để trống khi `contract_id` null — không xảy ra vì điều kiện hiển thị).
8. Excel: header "Đã CK TK Cty" (sửa chính tả); tài sản dùng cùng cách render với trang; tên file encode UTF-8 `Báo-cáo-kê-khai_dd-MM-yyyy_HH-mm-ss.xlsx`; dùng bộ lọc đã áp dụng.

## Review Focus

1. Ba cột tiền và điều kiện hiển thị khớp SQL cũ (`DECLARATION_ACCOUNTING_JOIN`): `ca.type` NULL rơi vào "đã thu khác", tài khoản đã xóa không tính, bút toán `type = 3` vào TK công ty vẫn tính CK.
2. Trống ngày → dòng `bill_date` NULL vẫn hiện; có ngày → bị loại.
3. Tổng = tổng các dòng cùng bộ lọc, kể cả khi nhân dòng do nhiều `juridical`.
4. Quyền: BGD/NVKT/NVKD gọi cả `/` và `/export` → 403; admin → 200; chưa đăng nhập → 401.

---

## File Structure

**Backend** (`backend/src/main/java/com/thanhdo/tdprojectbe/`):

| File | Trách nhiệm |
|---|---|
| Modify `model/entity/Subsection.java` | Hằng `OTHER_TRANSFER_SUBSECTIONS`. |
| Create `model/dto/DeclarationReport{Query,Filter,Row,Item,TotalsRow,Totals}.java` | DTO. |
| Create `repository/DeclarationReportSql.java`, `repository/DeclarationReportRepository.java` | SQL; trang / stream / tổng. |
| Create `common/util/export/excel/DeclarationReportExcelExporter.java` | Excel. |
| Modify `service/CertificateNumbers.java`, `service/DebtReportService.java` | Rút `seris` thành helper dùng chung. |
| Create `service/DeclarationReportService.java` | Map Row → Item, export. |
| Create `controller/DeclarationReportController.java` | API. |
| Modify `config/SecurityConfig.java` | Quyền admin. |
| Modify `backend/CLAUDE.md` | Thêm exporter vào dòng streaming (nếu có danh sách). |

**Frontend** (`frontend/src/`):
- Create: `lib/declaration-report.ts`, `api/declaration-reports.ts`, `views/report/DeclarationReportView.vue`, kèm spec.
- Modify: `router/index.ts`, `components/layout/nav-items.ts:109`, `router/__tests__/guard.spec.ts`, `components/layout/__tests__/layout.spec.ts` (gồm selector placeholder `:not([href=…])`).

---

### Task 1: DTO và bộ lọc

**Files:** Modify `Subsection.java`; Create DTO; Test `model/dto/DeclarationReportFilterTest.java`, `DeclarationReportTotalsTest.java`.

**Interfaces:**
- `record DeclarationReportQuery(String keyword, LocalDate fromDate, LocalDate toDate, LocalDate contractFromDate, LocalDate contractToDate)` — ngày `@DateTimeFormat(iso = DATE)`; `keyword` `@Size(max = 500)`; hai `@AssertTrue` "Từ ngày không được sau đến ngày" cho từng cặp. Tên tham số giữ như URL cũ.
- `record DeclarationReportFilter(String keyword, boolean filterBillDate, LocalDateTime billFrom, LocalDateTime billToExclusive, boolean filterCreateDate, LocalDateTime createFrom, LocalDateTime createToExclusive)`; `static of(query)`:
  - `keyword = LikePattern.contains(...)`.
  - Mỗi cặp ngày: có ít nhất một đầu → cờ `true`; đầu trống thay bằng mốc min/max (xem `ReportPeriod`), `to` → `to.plusDays(1).atStartOfDay()`.
- `interface DeclarationReportRow`: `getContractId`, `getRequestType`, `getCustName`, `getCompanyName`, `getLegalDocuments`, `getCertificateSeris`, `getBusinessSource`, `getSource`, `getContractValue` (Long), `getCompanyTransfer` (Long), `getOtherTransfer` (Long), `getPaidAmount` (Long), `getBillId`, `getBillDate` (LocalDateTime), `getNote`.
- `record DeclarationReportItem(String contractId, String customerName, List<String> assets, List<String> certificateSeris, String businessSource, String source, long contractValue, long companyTransfer, long otherTransfer, long debt, String billId, LocalDate billDate, String note)`.
- `interface DeclarationReportTotalsRow` (`BigDecimal getContractValue/getCompanyTransfer/getOtherTransfer/getPaidAmount`) và `record DeclarationReportTotals(BigDecimal contractValue, companyTransfer, otherTransfer, debt)` với `ZERO`, `from(row)` (null → 0, `debt = contractValue − paidAmount`).

- [ ] **Step 0:** kiểm tra `git -C backend status`, `git -C frontend status` sạch và đang ở `develop`.
- [ ] **Step 1: Test thất bại `DeclarationReportFilterTest`:** `ngày_trống_không_lọc` (cờ false); `chỉ_có_from` (cờ true, to = max); `trọn_ngày` (to 2026-03-15 → 2026-03-16T00:00); `từ_khóa_escape` ("a_b%" → `"%a\_b\%%"`); `từ_khóa_trống` → `""`.
- [ ] **Step 2:** `DeclarationReportTotalsTest`: null → 0; `debt = contractValue − paidAmount`.
- [ ] **Step 3–5:** FAIL → tạo hằng + DTO (Javadoc ghi nguồn cũ) → PASS.
- [ ] **Step 6: Commit** `feat(report): DTO báo cáo kế toán`.

### Task 2: SQL và repository

**Files:** Create `repository/DeclarationReportSql.java`, `repository/DeclarationReportRepository.java`; Test `repository/DeclarationReportRepositoryTest.java` (`@MysqlDataJpaTest`, dữ liệu qua `RepositoryTestData`, cô lập bằng tiền tố MARK trong `contract_id` + từ khóa MARK). Khuôn `DebtReportRepository` / `SurplusReportRepository`: `extends Repository<AppraisalRequest, Long>`, default `search`/`stream`/`totals`, `findPage` + `countQuery`, `streamAll` fetchSize `-2147483648`, `sums`.

SQL (`final class DeclarationReportSql`, chỉ ghép hằng, tham số SpEL):

```
ACC = LEFT JOIN (SELECT ac.contract_id,
   SUM(CASE WHEN ca.type = <TYPE_COMPANY_ACCOUNT> THEN SIGNED ELSE 0 END) companyTransfer,
   SUM(CASE WHEN (ca.type <> <TYPE_COMPANY_ACCOUNT> OR ca.type IS NULL) AND ac.type = <TYPE_CREATED_BY_USER>
            AND ac.subsection IN (<OTHER_TRANSFER_SUBSECTIONS>) THEN SIGNED ELSE 0 END) otherTransfer,
   MAX(CASE WHEN ac.subsection = <CONTRACT> AND ca.type = <TYPE_COMPANY_ACCOUNT> THEN 1 ELSE 0 END) hasContractCk,
   MAX(ac.created) created
  FROM accounting ac
  LEFT JOIN credit_account ca ON ca.id = CASE ac.accounting_mode WHEN 'PT' THEN ac.credit_account WHEN 'PC' THEN ac.debit_account END
        AND (ca.deleted = 0 OR ca.deleted IS NULL)
  WHERE ac.accounting_mode IN ('PT','PC') AND ac.contract_id IS NOT NULL AND ac.deleted = 0
  GROUP BY ac.contract_id) acc ON acc.contract_id COLLATE … = ap.contract_id COLLATE …
FROM_FILTERED = FROM p_appraisal_request ap
  LEFT JOIN juridical ju ON ju.contract_id COLLATE … = ap.contract_id COLLATE …   -- giữ như cũ: không lọc deleted
  + ACC + DebtReportSql.PAY_JOIN
FROM_ALL = FROM_FILTERED + cert JOIN (như DebtReportSql.FROM_ALL)
WHERE ap.delete_flag = '0'
  AND (acc.hasContractCk = 1 OR (ju.bill_id IS NOT NULL AND ju.bill_id <> ''))
  AND (:filterBillDate = FALSE OR (ju.bill_date >= :billFrom AND ju.bill_date < :billToExclusive))
  AND (:filterCreateDate = FALSE OR (ap.create_date >= :createFrom AND ap.create_date < :createToExclusive))
  AND (:keyword = '' OR ap.contract_id LIKE :keyword OR ap.cust_name LIKE :keyword
       OR ap.company_name LIKE :keyword OR ju.bill_id LIKE :keyword)
COLUMNS = ap.contract_id, ap.request_type, ap.cust_name, ap.company_name, ap.legal_documents,
  cert.seris certificateSeris, ap.business_source, ap.source, CONTRACT_VALUE contractValue,
  COALESCE(acc.companyTransfer,0), COALESCE(acc.otherTransfer,0), PAID paidAmount,
  ju.bill_id, ju.bill_date, ju.note
ORDER BY acc.created DESC, ju.bill_date DESC, ap.id DESC
COUNT  = SELECT COUNT(*) + FROM_FILTERED + WHERE
TOTALS = SELECT SUM(CONTRACT_VALUE), SUM(COALESCE(acc.companyTransfer,0)), SUM(COALESCE(acc.otherTransfer,0)), SUM(PAID) + FROM_FILTERED + WHERE
```
`SIGNED` = `CASE ac.accounting_mode WHEN 'PT' THEN ac.amount_of_money WHEN 'PC' THEN -ac.amount_of_money ELSE 0 END`. Kiểm tra cột `accounting.created` tồn tại; nếu cần `COLLATE` cho JOIN `cert` / `PAY_JOIN` thì làm theo kết quả test MySQL.

- [ ] **Step 1: Test thất bại** (mỗi test tự chèn dữ liệu):
  - `ba_cột_tiền`: PT 'Hợp đồng' vào TK công ty 1.000 + PC 'Hợp đồng' TK công ty 200 → CK 800; PT 'Hoàn tạm ứng' TK thường 300 → đã thu khác 300; PT tài khoản `type` NULL 'Hợp đồng' → vào đã thu khác; bút toán vào TK công ty đã xóa không tính.
  - `công_nợ_bằng_giá_trị_trừ_đã_thanh_toán` (valuation_fee "1500000", "1.500.000" → 0 như `CONTRACT_VALUE`).
  - `điều_kiện_hiển_thị`: có `hasContractCk` hoặc có `bill_id` thì hiện; chỉ bút toán 'Hợp đồng' vào TK thường và không hóa đơn → ẩn; hồ sơ `delete_flag = '1'` → ẩn.
  - `lọc_ngày_hóa_đơn_trọn_ngày` và `trống_ngày_vẫn_hiện_bill_date_null`.
  - `lọc_ngày_tạo_hồ_sơ`.
  - `từ_khóa_bốn_cột` (contract_id, cust_name, company_name, bill_id).
  - `nhiều_juridical_nhân_dòng_như_cũ` (2 juridical → 2 dòng; juridical đã xóa vẫn tính).
  - `seri_chứng_thư_gộp` (bỏ chứng thư đã xóa).
  - `sắp_xếp_theo_bút_toán_mới_nhất`.
  - `tổng_khớp_tổng_các_dòng`.
  - `stream_trả_cùng_các_dòng_như_trang` (trong transaction).
- [ ] **Step 2–4:** FAIL → cài đặt (Javadoc lý do native + "giữ JOIN juridical như cũ") → PASS. Không có MySQL thì ghi rõ chưa chạy, không coi là pass.
- [ ] **Step 5: Commit** `feat(report): truy vấn báo cáo kế toán`.

### Task 3: File Excel

**Files:** Create `common/util/export/excel/DeclarationReportExcelExporter.java`; Test `…/DeclarationReportExcelExporterTest.java` (mẫu `DebtReportExcelExporterTest`).

**Interface:** `public static void export(Consumer<Consumer<DeclarationReportItem>> dataSource, OutputStream out)`; `TITLE = "BÁO CÁO KẾ TOÁN"`; 14 cột, độ rộng như cũ; tài sản/seri `ValuationReportItem.numbered` (style wrap); tiền `styleRowPrice`; ngày chuỗi `dd/MM/yyyy`.

- [ ] **Step 1: Test thất bại:** `bố_cục_như_cũ` (B1 TITLE, header dòng 3 đúng thứ tự, "Đã CK TK Cty"); `tiền_và_ngày`; `tài_sản_nhiều_dòng_đánh_số`; `nhiều_dòng_hơn_cửa_sổ_streaming` (300 dòng).
- [ ] **Step 2–4:** FAIL → cài đặt → PASS.
- [ ] **Step 5: Commit** `feat(report): file Excel báo cáo kế toán`.

### Task 4: Service

**Files:** Modify `service/CertificateNumbers.java` (thêm `flatten`), `service/DebtReportService.java` (gọi `flatten`); Create `service/DeclarationReportService.java`; Test `service/DeclarationReportServiceTest.java`.

**Interfaces:** `Page<DeclarationReportItem> search(DeclarationReportQuery q, Pageable p)`, `DeclarationReportTotals totals(q)`, `@Transactional(readOnly = true) void export(q, OutputStream out)` (`try (Stream<Row> rows = repository.stream(filter))`). Không cần `callerId` (không có phạm vi theo người xem). `toItem`: `customerName` theo `AppraisalRequest.REQUEST_TYPE_PERSONAL`; `assets` qua `LegalDocumentAddresses.describe`; `debt = contractValue − paidAmount`; `billDate` → `LocalDate`.

- [ ] **Step 1: Test thất bại** (Mockito): `map_khách_hàng_cá_nhân_và_doanh_nghiệp`; `map_tài_sản_và_seri`; `công_nợ`; `export_ghi_file_từ_stream_và_đóng_stream`. Chạy lại `DebtReportServiceTest` để chắc refactor `seris` không đổi hành vi.
- [ ] **Step 2–4:** FAIL → cài đặt → PASS.
- [ ] **Step 5: Commit** `feat(report): service báo cáo kế toán`.

### Task 5: Controller và quyền

**Files:** Create `controller/DeclarationReportController.java`; Modify `config/SecurityConfig.java` (cạnh TSSS/Ban lãnh đạo); Test `controller/DeclarationReportControllerTest.java` (mẫu `TsssReportControllerTest` + `DebtReportControllerTest`).

**Interfaces:** `GET /api/reports/declaration` (`page` 0, `size` 20, 1..100, `@Valid @ModelAttribute DeclarationReportQuery`), `GET /totals`, `GET /export` → `"Báo-cáo-kê-khai_" + dd-MM-yyyy_HH-mm-ss + ".xlsx"`, `ContentDisposition…filename(name, UTF_8)`.
```java
// Báo cáo kế toán (/report/declaration của hệ thống cũ): trang và export chỉ admin.
.requestMatchers("/api/reports/declaration", "/api/reports/declaration/**").hasAuthority("admin")
```

- [ ] **Step 1: Test thất bại:** `cần_đăng_nhập` (3 path); `chỉ_admin` (BGD, NVKT, NVKD → 403 cả 3 path; admin → 200); `bind_bộ_lọc` (`keyword=x&fromDate=2026-03-01&toDate=2026-03-15&contractFromDate=…&contractToDate=…&page=1&size=10`); `tham_số_sai_trả_400` (`fromDate=01/03/2026`, from > to ở mỗi cặp, `size=101`, `page=-1`); `export_trả_file_tên_đúng` (`filename*=UTF-8''B%C3%A1o-c%C3%A1o-k%C3%AA-khai_`).
- [ ] **Step 2–3:** FAIL → controller (Javadoc đủ mục "Khác hệ thống cũ") + matcher.
- [ ] **Step 4:** `./mvnw -q test -Dtest='DeclarationReport*Test,DebtReport*Test'` → PASS; `./mvnw test` → `BUILD SUCCESS`.
- [ ] **Step 5:** cập nhật `backend/CLAUDE.md` nếu có danh sách exporter streaming.
- [ ] **Step 6: Commit** `feat(report): API báo cáo kế toán`.

### Task 6: Frontend — lib và api

**Files:** Create `lib/declaration-report.ts`, `api/declaration-reports.ts`; spec `lib/__tests__/declaration-report.spec.ts`, `api/__tests__/declaration-reports.spec.ts`. Khuôn `api/certificate-reports.ts` / `api/debt-reports.ts`.

**Interfaces:** `DECLARATION_REPORT_PAGE_SIZE = 20`; types `DeclarationReportItem`, `DeclarationReportTotals`, `DeclarationReportFilter` (`keyword`, `fromDate`, `toDate`, `contractFromDate`, `contractToDate`); hàm `listDeclarationReport`, `getDeclarationReportTotals`, `exportDeclarationReport` (bỏ tham số rỗng, `saveBlob`); helper đọc/ghi filter ↔ query URL như lib của certificate report.

- [ ] **Step 1–4:** spec thất bại (URL, bỏ tham số rỗng, parse query sai ngày) → cài đặt → PASS.
- [ ] **Step 5: Commit** `feat(report): api báo cáo kế toán`.

### Task 7: Frontend — màn hình, route, menu

**Files:** Create `views/report/DeclarationReportView.vue`, `views/__tests__/DeclarationReportView.spec.ts`; Modify `router/index.ts` (route + comment "Màn /report/declaration (Báo cáo → Báo cáo kế toán) của hệ thống cũ: chỉ admin", `meta.authorities: ['admin']` inline như TSSS), `nav-items.ts:109` (thêm `to: '/report/declaration'`), `guard.spec.ts`, `layout.spec.ts`.

Sao chép `CertificateReportView.vue`: bộ lọc nháp → áp dụng, đồng bộ URL, tổng tải riêng (`AbortController`) và chỉ hiện ở trang cuối, export dùng bộ lọc đã áp dụng. Bộ lọc: từ khóa (placeholder "Số hợp đồng, Khách hàng, Số hóa đơn"), `DateRangePicker` "Ngày xuất hóa đơn", `DateRangePicker` "Ngày tạo hồ sơ". Bảng 14 cột đúng thứ tự cũ, cột Số hợp đồng sticky như cũ; tài sản `v-for` từng dòng như `BusinessReportView.vue:449`; seri mỗi dòng một mục; tiền căn phải `formatMoney` (`lib/valuation-report.ts`); ngày `dd/MM/yyyy`. Dòng tổng: "Tổng cộng:" span 7 · 4 cột tổng · span 3 trống.

- [ ] **Step 1: Spec thất bại:** 14 cột đúng thứ tự; khách hàng/tài sản/seri hiển thị đúng; Tìm kiếm → API nhận đúng tham số và URL đồng bộ; dòng tổng chỉ ở trang cuối; Xuất Excel gửi bộ lọc đã áp dụng.
- [ ] **Step 2:** `npx vitest run src/views/__tests__/DeclarationReportView.spec.ts` → FAIL.
- [ ] **Step 3:** cài đặt view, route, menu; `guard.spec.ts` (admin vào được; BGD, NVKT bị chặn); `layout.spec.ts` (link hiện cho admin; bỏ href mới khỏi selector placeholder).
- [ ] **Step 4:** PASS.
- [ ] **Step 5:** `npm run lint && npm run type-check && npm run test:unit`. Lỗi có sẵn trên `develop` không liên quan thì ghi rõ.
- [ ] **Step 6: Commit** `feat(report): màn báo cáo kế toán`.

### Task 8: Kiểm tra cuối

- [ ] `cd backend && ./mvnw test`; `cd frontend && npm run lint && npm run type-check && npm run test:unit`.
- [ ] Chạy app nếu có môi trường (skill `run`): đăng nhập admin, vào `/report/declaration`; so số dòng và 4 tổng với hệ thống cũ cùng bộ lọc (nhớ mục "Khác hệ thống cũ" 3–6); mở file Excel. Gọi API bằng tài khoản BGD → 403.
- [ ] Code review (`superpowers:requesting-code-review`, subagent `model: "sonnet"`).
- [ ] Báo cáo người dùng: kết quả test thật (số test, lỗi), phần chưa kiểm (chạy app, đối chiếu số liệu), nhắc lại việc giữ nhân dòng `juridical` và đóng quyền export.
