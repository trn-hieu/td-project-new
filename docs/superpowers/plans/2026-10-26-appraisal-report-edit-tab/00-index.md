# Migrate màn "Báo cáo thẩm định" (edit-tab) — Implementation Plan (index)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **Model cho subagent: CHỈ dùng `sonnet`.** Mọi lần gọi tool `Agent` (implementer, spec reviewer, code reviewer, final reviewer) phải truyền `model: "sonnet"`. Không dùng opus/haiku/fable cho subagent. **Mỗi task bắt buộc qua review** (spec review rồi code review) trước khi sang task kế tiếp.

**Goal:** Port màn `/appraisal-report/edit-tab/{id}` của `td-project-legacy` sang `backend` (REST dưới `/api/appraisal-records/{profileId}/reports/{reportId}`) và `frontend` (view `AppraisalReportEditView`, route `appraisal-report-edit` đã có), gồm 6 tab con × 3 loại tài sản, logic đồng bộ giữa các entity và API tính kết quả thẩm định port y nguyên.

**Architecture:** Thay fragment HTML + redirect bằng JSON API theo từng tab, mỗi tab một cặp `GET` (dữ liệu màn) / `PUT` (lưu). Logic đồng bộ chéo nằm ở backend, trong các lớp nhỏ thuần (không truy cập DB) được service gọi trong cùng transaction: `LandCompareSync`, `VehicleCompareSync`, `CompareFactorSync`, `CompareInformationSync`. Công thức tính trên form (bảng điều chỉnh, thông tin so sánh, CTXD) port từ JS sang hàm TS thuần trong `frontend/src/lib/report-edit/` có unit test; server lưu nguyên giá trị client gửi như hệ thống cũ. Kết quả thẩm định tính ở server bằng bản port đầy đủ `AppraisalResultCalculator`.

**Tech Stack:** Spring Boot 4 · Java 21 · Spring Data JPA (MySQL dùng chung, `ddl-auto=none`) · S3/CloudFront (`S3StorageService`, `S3LinkService`) · JUnit 5/Mockito/AssertJ · `@MysqlDataJpaTest` · Vue 3 + TS · shadcn-vue · vee-validate + zod · Google Maps JS (`lib/google-maps.ts`) · Vitest.

**Spec:** Không có spec riêng; spec là hành vi hệ thống cũ, đã khảo sát và tóm tắt trong từng file tab (mục "Hành vi hệ thống cũ"). Điểm vào hệ thống cũ:
- `td-project-legacy/src/main/java/com/jvb/thanhdo/controller/AppraisalReportController.java` (edit-tab, các POST `/save/*`)
- `td-project-legacy/src/main/java/com/jvb/thanhdo/controller/AppraisalReportTabController.java` (nạp từng tab)
- `td-project-legacy/src/main/resources/templates/demo/demo-report-step-2.html` + `static/js/demo-report-step-2-tabs.js`

Viết tắt dùng trong mọi file: **L** = `td-project-legacy/src/main/java/com/jvb/thanhdo/`, **LR** = `td-project-legacy/src/main/resources/`, **N** = `backend/src/main/java/com/thanhdo/tdprojectbe/`, **NT** = `backend/src/test/java/com/thanhdo/tdprojectbe/`, **F** = `frontend/src/`.

## Danh sách file kế hoạch và thứ tự task

Thực hiện **tuần tự** đúng thứ tự (các task cùng sửa entity, `SecurityConfig`, router nên không chạy song song).

| File | Task | Nội dung | Phụ thuộc |
|---|---|---|---|
| `01-shell.md` | T01 | BE: `ReportEditScope` + API workspace (danh sách nhóm) | — |
| | T02 | FE: khung màn 2 cấp tab, chặn rời tab khi chưa lưu | T01 |
| `02-result-api.md` | T03 | BE: `MoneyWords` (đọc số tiền bằng chữ) | — |
| | T04 | BE: `AppraisalResultCalculator` port đầy đủ + bản rút gọn ủy quyền | T03 |
| | T05 | BE: API `GET/PUT evaluation-result` + entity `AppraisalResult` ghi được | T01, T04 |
| `03-results-tab.md` | T06 | FE: tab Kết quả thẩm định (Đất / Căn hộ / Ô tô) + bảng kết quả dùng chung | T02, T05 |
| `10-legal.md` | T07 | BE: tab Pháp lý (mọi loại) + file đính kèm | T01 |
| | T08 | FE: Pháp lý Đất / Căn hộ | T07 |
| | T09 | FE: Pháp lý Ô tô | T07 |
| `11-images.md` | T10 | BE: ảnh mô tả (`appraisal_image`) theo nhóm | T01 |
| | T11 | FE: khối ảnh + modal thêm ảnh dùng chung | T10 |
| `20-land-description.md` | T12 | BE: danh mục (loại đất, quận/huyện, phường/xã, nghị quyết) | — |
| | T13 | BE: Mô tả đất (LAND) + `LandCompareSync` | T12 |
| | T14 | FE: Mô tả đất phần 1 (vị trí, diện tích, bảng thửa đất) | T13, T11 |
| | T15 | FE: Mô tả đất phần 2 (hình dáng, kích thước/hướng, đơn giá, QĐ, ảnh, sao chép/dán) | T14 |
| `21-apartment-description.md` | T16 | BE: Mô tả căn hộ + tìm/đồng bộ tòa chung cư | T13 |
| | T17 | FE: Mô tả căn hộ + modal tìm tòa nhà | T16, T14 |
| `22-car-description.md` | T18 | BE: Mô tả phương tiện + `VehicleCompareSync` | T01 |
| | T19 | FE: Mô tả phương tiện | T18, T11 |
| `30-ctxd.md` | T20 | BE: CTXD (công trình, kết cấu, đánh giá, ảnh, ghi chú) | T01 |
| | T21 | FE: tab Mô tả CTXD | T20, T06 |
| `40-compare-backend.md` | T22 | BE: map đủ `AppraisalGroupCompare`/`CompareInformation` + API GET so sánh | T01 |
| | T23 | BE: lưu so sánh (ghi theo loại tài sản, gộp chi tiết DT, `CompareFactorSync`) | T22 |
| | T24 | BE: đồng bộ kho TSSS + ảnh TSSS | T23 |
| | T25 | BE: tìm TSSS gần (3 km) và theo tên xe | T22 |
| | T26 | BE: tuyến đường đi bộ (Google Routes, async) | T23 |
| `41-compare-common-fe.md` | T27 | FE: thành phần dùng chung tab so sánh (ảnh TSSS, ô đồng bộ kho, bản đồ chọn TSSS, điền từ kho) | T24, T25 |
| `42-compare-land.md` | T28 | FE: Thông tin so sánh — Đất | T27 |
| `43-compare-apartment.md` | T29 | FE: Thông tin so sánh — Căn hộ | T27 |
| `44-compare-car.md` | T30 | FE: Thông tin so sánh — Ô tô (+ tìm theo tên xe) | T27 |
| `50-panel-backend.md` | T31 | BE: Bảng điều chỉnh (GET/PUT, sync-preview, chi tiết đất ngoài QH) | T23 |
| `51-panel-fe.md` | T32 | FE: hàm tính bảng điều chỉnh (thuần TS) | — |
| | T33 | FE: Bảng điều chỉnh — Đất | T31, T32, T06 |
| | T34 | FE: Bảng điều chỉnh — Căn hộ, Ô tô | T33 |
| `90-final-review.md` | T35 | Kiểm tra cuối + review toàn nhánh | tất cả |

## Quy trình cho controller (phiên chính) — bắt buộc

Dùng superpowers:subagent-driven-development. Với **mỗi task**:
1. **Implementer** — `Agent({ subagent_type: "general-purpose", model: "sonnet", prompt })`. Prompt gồm: (a) đường dẫn `00-index.md` và file tab chứa task, yêu cầu đọc cả hai; (b) "Chỉ làm Task Txx; đọc các file hệ thống cũ được nêu trong task trước khi viết code; TDD theo các Step; commit trong submodule theo Step cuối"; (c) Interfaces của các task trước mà task này dùng (copy khối Interfaces). Implementer không được tự mở rộng phạm vi; gặp điều plan không quyết định được (cột không tồn tại, hành vi cũ khác mô tả) thì **dừng và báo lại**, không đoán.
2. **Spec reviewer** — `Agent({ model: "sonnet" })`: đối chiếu diff với task (Files, Interfaces, Steps), mục "Hành vi hệ thống cũ" của file tab (đọc lại code cũ ở dòng được dẫn), danh sách "Khác hệ thống cũ" bên dưới và Review Focus. Kết luận APPROVED hoặc danh sách lỗi có file:dòng.
3. **Code reviewer** — `Agent({ model: "sonnet" })`: đối chiếu với `backend/CLAUDE.md`, `.claude/rules/jpa-spring.md` (checklist cuối), `.claude/rules/design-principles.md`, `frontend/CLAUDE.md`, `frontend/DESIGN_SYSTEM.md`; chạy lại lệnh kiểm tra của task.
4. Lỗi được xác nhận → gửi lại implementer (SendMessage cùng agent) sửa → review lại bước đã fail. Chỉ đánh dấu `[x]` khi cả hai reviewer APPROVED.
5. Hết mỗi file kế hoạch: chạy toàn bộ `./mvnw test` (nếu có task BE) và `npm run lint && npm run type-check && npm run test:unit` (nếu có task FE).

Không chạy song song hai implementer. Không push, không cập nhật con trỏ submodule ở repo gốc khi chưa được người dùng đồng ý.

## Global Constraints

- Subagent: chỉ `model: "sonnet"`; mỗi task có spec review + code review.
- `backend/` và `frontend/` là **git submodule**: tạo nhánh `feat/appraisal-report-edit-tab` trong cả hai ở T01/T02; mọi commit chạy `git -C backend …` / `git -C frontend …`.
- Tuân thủ `backend/CLAUDE.md` + `.claude/rules/jpa-spring.md` (đọc trước task BE đầu tiên): JPA trước, native SQL chỉ khi cần hàm MySQL (`ST_Distance_Sphere`) và có comment lý do; entity không `@Data`; `@ManyToOne` LAZY; service `@Transactional`; lỗi bằng `BusinessException(ErrorCode, "…")`. Frontend tuân thủ `frontend/CLAUDE.md` + `frontend/DESIGN_SYSTEM.md` (tab tối đa 2 cấp, `useRouteTab`, `AppModal`, `useConfirm`, `components/form/*`, không tự format số).
- **DB dùng chung với hệ thống cũ, hệ thống cũ vẫn đọc/ghi**: không tạo Flyway migration (mọi cột đã có). Dữ liệu ghi phải đúng định dạng hệ thống cũ đang ghi: chuỗi diện tích kiểu châu Âu trong JSON (`"1.234,56"`, dùng `EuropeanNumbers`), tên khóa JSON giống DTO cũ (copy `@JsonProperty` từ lớp cũ), cờ `delete_flag`/`deleted` đúng kiểu của từng bảng. Thấy thiếu cột/bảng → dừng, báo lại.
- Entity hiện có chỉ `@Getter`: thêm cột còn thiếu bằng `@Column(name=…)` khớp schema; thay đổi qua phương thức nghiệp vụ (`apply…(…)`, `overwrite…(…)`) hoặc `@Setter` trên đúng trường cần ghi, không `@Setter` cả lớp mới. Cột `updated` đang `updatable = false` ở `AppraisalAssetGroup`, `AppraisalReportAsset`, `ControlCompareFactor` → bỏ `updatable = false` và gán qua `@PreUpdate` khi task cần cập nhật entity đó; repository test phải chứng minh `updated` đổi sau khi lưu.
- Cột JSON: converter kế thừa `N model/converter/JsonListConverter` (đọc lỗi → rỗng như cũ); record JSON mới có `@JsonIgnoreProperties(ignoreUnknown = true)`. Thêm annotation này cho `AreaInsideDetail`, `AreaOutsideDetail`, `StructureDetail` ở T22 kèm test đọc JSON có khóa lạ.
- API: đường dẫn gốc màn này `/api/appraisal-records/{profileId}/reports/{reportId}` (đã nằm trong quy tắc `SecurityConfig` cho `/api/appraisal-records/**`: BGD, TPTD, NVTD, NVPI, NVKT, NVKD). Danh mục dùng chung đặt ở `/api/<tài-nguyên>` (chỉ cần đăng nhập, kiểm tra quy tắc mặc định của `SecurityConfig`). GET trả DTO trực tiếp; PUT/POST/DELETE trả `ApiResponse.ok(data, "…thành công")` như `AppraisalReportController.create`. Số tiền/diện tích trong JSON là số (`Long`/`BigDecimal`/`Double`), ngày ISO `YYYY-MM-DD`; riêng giá trị hệ thống cũ lưu là chuỗi thì giữ chuỗi.
- Mọi API của màn kiểm tra chuỗi đường dẫn bằng `ReportEditScope` (T01): hồ sơ → báo cáo → nhóm, sai trả 404. Không thêm kiểm tra quyền theo dòng ngoài chuỗi này (theo quyết định người dùng); id dòng không thuộc nhóm thì bỏ qua như cũ.
- UI: nhãn, comment, Javadoc, message tiếng Việt có dấu. Javadoc controller/service ghi lớp/màn cũ được port và mọi điểm "Khác hệ thống cũ" liên quan.
- Kiểm tra: backend `cd backend && ./mvnw -q test -Dtest=<Tên>` (toàn bộ `./mvnw test`, test repository cần MySQL theo `backend/.env.example`); frontend `cd frontend && npm run lint && npm run type-check && npm run test:unit` (type-check **sau** lint).
- Ngoài phạm vi phase này: tab "Xem trước báo cáo", "Tải báo cáo" (docx), xuất Excel/Word phiếu TSSS, nút "Sửa thông tin cơ bản" (màn edit-basic chưa port). Không hiển thị các nút này.

### Khác hệ thống cũ (cố ý) — phải ghi vào Javadoc nơi liên quan

1. JSON API thay fragment HTML/redirect; sau khi lưu, FE tải lại dữ liệu tab thay vì reload cả trang. Tab con bị hủy khi ẩn (`unmount-on-hide`) nên mỗi lần mở tab đều đọc dữ liệu mới (thay cho cờ `data-loaded`).
2. Lỗi trả 400/404 qua `BusinessException` (cũ: 200 với DTO rỗng hoặc trang lỗi fragment).
3. `market_price` khi cả giá giao dịch và giá ước tính đều null ghi `0` (cũ: hằng `0111L` = 73 bát phân).
4. `control_compare_factor_synthetic.appraisal_profile_id` ghi `null` (cũ: ghi nhầm id báo cáo).
5. Lưu so sánh: chi tiết "DT trong QH" của TSSS 1–3 có uuid không khớp (lấy từ kho) được giữ lại như dòng mới (cũ: bị rơi).
6. Lưu so sánh chỉ đồng bộ `c_value*`/tỷ lệ đất ở sang bảng điều chỉnh của **nhóm đang lưu** (cũ: mọi nhóm của báo cáo) và không ghi nhãn không áp dụng cho loại tài sản (cũ: xóa trắng "Diện tích" của nhóm ô tô).
7. Lưu so sánh / mô tả đất / pháp lý chỉ ghi cột thuộc loại tài sản của nhóm (cũ: null hóa cột của loại khác, vd. lưu căn hộ làm rỗng chi tiết đất của dòng so sánh type 0).
8. "Khác" (3.10) của pháp lý: gửi danh sách rỗng thì xóa hết (cũ: không gửi gì nên giữ nguyên).
9. Bỏ tab Xem trước, nút Tải báo cáo, xuất phiếu TSSS, nút Sửa thông tin cơ bản (phase sau).

## Review Focus

1. **API kết quả trả JSON giống hệt cũ** cho từng phương pháp (MIXED, AGRICULTURAL_STATE_PRICE có `maxPriceRecord`, LAND_USE_CONVERSION, STATE_PRICE), căn hộ (cơ sở diện tích 2 = thông thủy), ô tô, và `roundingMoneyType=2` ghi đè; danh sách rỗng là `null` chứ không `[]` (test: T04 `AppraisalResultCalculatorTest.*`, T05 `EvaluationResultControllerTest.json_giống_hệ_thống_cũ`).
2. **Lưu mô tả đất khi dòng so sánh type 0 đã có uuid**: chi tiết DT ngoài/trong QH dựng lại theo thứ tự bảng giá đất, diện tích đã nhập ở tab so sánh được giữ cho uuid khớp; lưu **căn hộ** không làm rỗng chi tiết (test: T13 `LandCompareSyncTest.uuid_khớp_giữ_diện_tích`, T16 `LandDescriptionServiceTest.lưu_căn_hộ_không_đụng_chi_tiết_đất`).
3. **TSSS chọn từ kho rồi lưu**: `compare_information_id` được gán, dòng kho không bị ghi, ảnh kho clone một lần; tick "Đồng bộ lên CSDL" lưu hai lần cập nhật **cùng** dòng kho (test: T24 `CompareInformationSyncTest.chọn_từ_kho_không_ghi_kho`, `đẩy_hai_lần_cùng_một_dòng_kho`).
4. **Bảng điều chỉnh**: TSSS trống vẫn chia 3; làm tròn nghìn lưu `avgPriceGuide` là giá trị đã làm tròn; biên độ dạng `"1,00% - 5,00%"`; nhãn không áp dụng cho ô tô không bị xóa khi lưu so sánh (test: T32 `adjustment-panel-calc.spec.ts`, T23 `CompareFactorSyncTest.ô_tô_không_đồng_bộ_diện_tích`).
5. **Đổi tab khi đang sửa dở** hỏi xác nhận bỏ thay đổi; lưu so sánh xong mở Bảng điều chỉnh thấy giá trị mới (test: T02 `AppraisalReportEditView.spec.ts › hỏi_xác_nhận_khi_rời_tab_chưa_lưu`, T33 `AdjustmentPanelTab.spec.ts › tải_lại_khi_mở_tab`).

## Quy ước tên dùng chung (để các task khớp nhau)

**Backend (`N`)**
- `service/ReportEditScope` (T01): `AppraisalReport requireReport(long profileId, long reportId)`, `AppraisalAssetGroup requireGroup(long profileId, long reportId, long groupId)`, `List<AppraisalAssetGroup> sortedGroups(long reportId)` (chưa xóa, `groupIndex` tăng dần, null cuối).
- Controller mới (một controller mỗi tab, cùng tiền tố `/api/appraisal-records/{profileId}/reports/{reportId}`): `ReportWorkspaceController`, `EvaluationResultController`, `ReportLegalController`, `AppraisalImageController`, `LandDescriptionController`, `VehicleDescriptionController`, `ConstructionDescriptionController`, `GroupCompareController`, `AdjustmentPanelController`. Danh mục: `LandTypeController` (`/api/land-types`), `AdministrativeUnitController` (`/api/districts`, `/api/wards`), `ResolutionController` (`/api/resolutions`), `ApartmentBuildingController` (`/api/apartment-buildings`), `CompareInformationController` (`/api/compare-informations`).

**Frontend (`F`)**
- API: `api/report-edit/<tab>.ts`, hàm dùng `api.get/put/post/delete` của `lib/api.ts`; `api/report-edit/workspace.ts` export `reportEditPath(profileId: number, reportId: number): string` = `` `/appraisal-records/${profileId}/reports/${reportId}` `` cho mọi file khác.
- Component: `components/appraisal-report/edit/<tab>/…`; props chung của mọi tab: `{ profileId: number; reportId: number; group: ReportWorkspaceGroup }`.
- Hàm tính thuần: `lib/report-edit/<tên>.ts`, test ở `lib/report-edit/__tests__/`.
- Chặn rời tab: `useDirtyTab(isDirty: Ref<boolean>)` (T02) — mọi tab có form gọi một lần.
