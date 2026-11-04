# 12 — FE: Pháp lý, Bảng điều chỉnh, Kết quả, Modal tìm TSSS và Tab So sánh của tàu (F07, F09, F10, F11, F08)

> Đọc `00-index.md` (quy ước, hợp đồng dùng chung, Global Constraints, "Khác hệ thống cũ", Review Focus) và `surveys/legacy-ui-word.md` (mục 4 Pháp lý, 5 So sánh, 6 Bảng điều chỉnh, 7 Kết quả) trước. Làm xong `10-fe-foundation.md` (F01) rồi mới bắt đầu. Subagent: `model: "sonnet"`; mỗi task qua spec review rồi code review.
>
> Ký hiệu: **F** = `frontend/src/`, **LT** = `td-project-legacy/src/main/resources/templates/`, **LJ** = `td-project-legacy/src/main/resources/static/js/`. Đọc code legacy bằng `git -C td-project-legacy show 82193fe5:<đường dẫn>`, **không checkout**. Số dòng legacy là số dòng của file tại `82193fe5`.
>
> Số dòng của code mới đo trên `origin/develop` (`6ff0110`) và có thể lệch; luôn grep lại trước khi sửa.

## Đợt chạy và chủ sở hữu file

F07, F09, F10, F11 chạy **song song** ở đợt F2 vì tập file rời nhau. F08 chạy ở đợt F3, sau F10, vì dùng modal và hàm thuần của F10.

| Task | Đợt | File **chủ sở hữu** (tạo hoặc sửa) |
|---|---|---|
| F07 | F2 | `legal/LegalShipAssetForm.vue`, `legal/LegalTab.vue`, `lib/report-edit/legal-documents-text.ts`, `legal/__tests__/LegalShipAssetForm.spec.ts`, `legal/__tests__/LegalTab.spec.ts`, `lib/report-edit/__tests__/legal-documents-text.spec.ts` |
| F09 | F2 | `panel/PanelSummaryRows.vue`, `panel/FactorRows.vue`, `panel/AdjustmentFactorTable.vue`, `panel/ValuationMethodCard.vue`, `panel/AdjustmentPanelTab.vue`, `lib/report-edit/adjustment-panel-calc.ts`, `lib/report-edit/__tests__/adjustment-panel-calc.spec.ts`, `panel/__tests__/AdjustmentPanelTab.spec.ts`, `panel/__tests__/panel-fixtures.ts` |
| F10 | F2 | `compare/ShipCompareSearchModal.vue`, `lib/report-edit/ship-compare.ts` (**file mới**, không đụng `compare-common.ts`), `compare/__tests__/ShipCompareSearchModal.spec.ts`, `lib/report-edit/__tests__/ship-compare.spec.ts`, `lib/report-edit/__tests__/ship-compare-fixtures.ts` |
| F11 | F2 | `results/AssetResultTable.vue`, `results/ShipResultTable.vue` (mới), `results/ShipCurrencyBlock.vue` (mới), `results/EvaluationResultTab.vue`, `lib/report-edit/exchange-rate.ts` (mới), `lib/report-edit/__tests__/exchange-rate.spec.ts`, `results/__tests__/EvaluationResultTables.spec.ts`, `results/__tests__/EvaluationResultTab.spec.ts`, `results/__tests__/fixtures.ts` |
| F08 | F3 | `compare/ShipCompareTab.vue` (thay vỏ F01), `lib/report-edit/compare-common.ts`, `composables/useCompareForm.ts`, `lib/report-edit/ship-compare.ts` (chỉ **thêm** hàm kiểm tra, sau F10), `compare/__tests__/ShipCompareTab.spec.ts`, `lib/report-edit/__tests__/compare-common.spec.ts`, `composables/__tests__/useCompareForm.spec.ts` |

Quyết định phân chia:
- Hàm điền cột từ kho và xóa cột của tàu nằm trong **`lib/report-edit/ship-compare.ts` do F10 sở hữu**. F10 **không** sửa `compare-common.ts`. F08 là task duy nhất sửa `compare-common.ts` và `useCompareForm.ts`: thêm nhánh `SHIP` tường minh gọi sang `ship-compare.ts` (tránh `else` ngầm hiểu là đất).
- `GroupWorkspace.vue` do F01 sở hữu (đã map các tab SHIP, vỏ `ShipCompareTab.vue`). Task trong file này **không** sửa `GroupWorkspace.vue`.
- Không task nào sửa `legal.ts`, `compare.ts` (trừ một dòng comment ở F08), `compare-informations.ts`, `panel.ts`, `evaluation-result.ts`, `currency.ts`, `tabs.ts`: đó là file của F01. Thiếu kiểu thì dừng và báo.

## Giả định về kết quả của F01 (phải khớp trước khi bắt đầu)

Đây là các tên dựa vào quy ước của `00-index.md`. Nếu code F01 thực tế khác, **dừng và báo controller**, không tự đổi tên.

| Nơi | Giả định |
|---|---|
| `lib/report-edit/tabs.ts` | `ASSET_TYPE.SHIP = 4`; mọi literal `1 \| 2 \| 3` đã đổi thành `AssetType` |
| `lib/report-edit/currency.ts` | `resolveCurrencyCode(assetType, code)`, `currencyUnitLabel(code)` (`VND`/`USD`), `currencyMoneyLabel(code)` (`đồng`/`USD`), `regulatoryPriceUnit(assetType: number, code: string \| null \| undefined): string` (SHIP: `đồng`/`USD`; CAR: `đồng/xe`; khác: `đồng/m²`) |
| `api/report-edit/legal.ts` | (Quyết định #13) `LegalAsset` có thêm `shipInspectionCertificateNumber`, `shipInspectionIssuedPlace`, `shipInspectionIssuedDate`, `shipInspectionExpiryDate`, `shipInspectionForWhom` (`string \| null`). `LegalAssetInput` để 5 field này **tùy chọn** (`?`) để form Đất và Ô tô hiện có không phải sửa |
| `api/report-edit/compare.ts` | `GroupCompareRow` có `shipBuildYear`, `shipBuildPlace`, `shipUsagePurpose`, `shipClass`, `shipRemainingQualityPct`, `shipDeadweight`, `shipDimension`, `shipMainEnginePowerHp`, `shipForSalePriceUsd` (đều `number\|string \| null` đúng kiểu cột) và các field này **không** nằm trong `COMPARE_ROW_READ_ONLY_KEYS`. `GroupCompareData.currencyCode: 'VND' \| 'USD'` |
| `api/report-edit/compare-informations.ts` | `ShipCompareInformationItem` (đúng DTO ở `00-index.md`: `id, address, source, contact, shipBuildYear, shipBuildPlace, shipUsagePurpose, shipClass, shipRemainingQualityPct, shipDeadweight, shipDimension, shipMainEnginePowerHp, currencyCode, forSalePrice, tradePrice, estimatePrice, updated`) và `searchShipCompareInformations(keyword: string, page: number, size: number, signal?: AbortSignal): Promise<PageData<ShipCompareInformationItem>>` (cùng dạng `searchCarCompareInformations`, gọi `/compare-informations/search` với `assetType: 4`) |
| `api/report-edit/panel.ts` | `AdjustmentPanelData.currencyUnit: string` (`VND`/`USD`) |
| `api/report-edit/evaluation-result.ts` | `EvaluationResult` có `ship: { amount, exchangeRate, vndAmount, roundedAmount } \| null`, `currencyCode`, `exchangeRate`, `currencyNote`; `SaveEvaluationResultBody` có `currencyCode?: CurrencyCode`, `exchangeRate?: number`, `currencyNote?: string \| null` (khóa kết quả tàu là `ship`, Quyết định #1) |
| Fixture test | `compare-fixtures.ts` (`compareRow`, `fourRows`, `khoDetail`), `results/__tests__/fixtures.ts` (`evaluationResult`), `panel/__tests__/panel-fixtures.ts` (`panelData`) đã được F01 bổ sung field ship với giá trị mặc định `null` để `type-check` qua |
| `GroupWorkspace.vue` | `compare` của SHIP trỏ vào `compare/ShipCompareTab.vue` (vỏ rỗng nhận `{ profileId, reportId, group }`); `legal`, `panel`, `results` dùng chung component sẵn có |

## Lệnh kiểm tra

- Chạy nhanh từng file test: `cd frontend && npx vitest run <đường dẫn file test>`.
- Cuối mỗi task: `cd frontend && npm run lint && npm run type-check && npm run test:unit` (`lint` tự sửa file nên chạy `type-check` **sau** nó). Lỗi của file thuộc task khác thì báo, không tự sửa.
- Commit trong submodule: `git -C frontend add <file> && git -C frontend commit -m "<thông điệp>"`. Không push.

---

# F07 — Pháp lý tàu

## Hành vi hệ thống cũ

Nguồn: `LT demo/legal-document-ship.html` (form l.4-215; script sinh văn bản l.215-267; kiểm tra độ dài l.277-334), `LT demo/legal-document-car.html` (hàm dùng chung `formatCarLegalDate` l.316-330, `buildCarLegalLine` l.335-347), `L service/impl/AppraisalReportAssetServiceImpl.java:96-113` (ghi 5 cột `shipInspection*` khi nhóm SHIP).

- Mỗi tài sản của nhóm là một pill "Tài sản N", **form riêng**, lưu từng tài sản (multipart, có file). Khối "Danh sách tài sản" ở đầu chỉ **đọc** `asset.asset` (rỗng thì "Chưa có mô tả tài sản.").
- Mục **1. Thông tin giấy chứng nhận đăng ký phương tiện thủy nội địa** (l.60-95):
  - 1.1 Số GCN `certificateNumber` (textarea một dòng, chặn Enter)
  - 1.2 Nơi cấp `cnIssuedPlace`
  - 1.3 Ngày cấp `gcnNoIssuedDate` (date picker)
  - 1.4 Cho ai `forWhom` (placeholder "VD: cho ông A, bà B...")
- Mục **2. Thông tin giấy chứng nhận an toàn kỹ thuật và bảo vệ môi trường phương tiện thủy nội địa** (l.97-142):
  - 2.1 Số GCN `shipInspectionCertificateNumber`
  - 2.2 Nơi cấp `shipInspectionIssuedPlace`
  - 2.3 Ngày cấp `shipInspectionIssuedDate`
  - 2.4 Ngày hết hiệu lực `shipInspectionExpiryDate`
  - 2.5 Cho ai `shipInspectionForWhom` (cùng placeholder)
- Mục **3. Giấy tờ pháp lý của tài sản**: textarea `legalDocuments` (4 dòng), tự sinh từ mục 1 và 2, người dùng sửa tay được.
- Mục **4**: file đính kèm (pdf, doc, docx, jpg, png, xls, xlsx; kéo thả; xóa từng file). Dùng chung với ô tô.
- Không có chủ tài sản, số vào sổ, loại GCN, ngày hết hiệu lực của GCN đăng ký, mục "Khác".
- **Công thức mục 3** (`updateLegalDocumentsShip`, l.226-267), mọi giá trị đã `trim`, ngày đổi bằng `formatCarLegalDate` (`dd tháng MM năm yyyy`, không đọc được thì giữ nguyên):
  - Hằng số: `SHIP_REGISTRATION_PREFIX = "Giấy chứng nhận đăng ký phương tiện thủy nội địa"`, `SHIP_INSPECTION_PREFIX = "Giấy chứng nhận an toàn kỹ thuật và bảo vệ môi trường phương tiện thủy nội địa"`.
  - `buildCarLegalLine(prefix, parts)`: không có phần nào có giá trị thì trả rỗng; ngược lại là `prefix` + với mỗi phần có giá trị `text + value`, rồi `trim()` và thêm `"."`.
  - **Dòng 1** (đăng ký): các phần `[" số: ", certificateNumber]`, `[" do ", cnIssuedPlace]`, `[" cấp ngày ", ngày cấp]`, `[" cho ", forWhom]`.
  - **Dòng 2** (an toàn kỹ thuật): các phần `[" số: ", shipInspectionCertificateNumber]`, `[" do ", shipInspectionIssuedPlace]`, `[" ngày ", ngày cấp]` (**không** có chữ "cấp"), `[" cho ", shipInspectionForWhom]`.
  - Có ngày hết hiệu lực thì tạo câu `"Giấy chứng nhận này có hiệu lực đến ngày " + ngày + "."`. Nếu dòng 2 đã có nội dung thì `dòng2 + " " + câu`; nếu dòng 2 rỗng thì `SHIP_INSPECTION_PREFIX + ". " + câu`.
  - Dòng nhập tay = các dòng của textarea sau `trim`, bỏ dòng rỗng và bỏ dòng **bắt đầu bằng** một trong hai tiền tố (`isShipAutoGeneratedLine` chỉ kiểm tra tiền tố, không có regex như ô tô).
  - Kết quả = `[dòng1, dòng2].filter(không rỗng).concat(dòng nhập tay).join("\n")`. Hai dòng sinh luôn nằm đầu.
  - Chạy mỗi khi sửa một ô trong mục 1 hoặc 2 (class `auto-update-legal-doc-ship`).
- **Kiểm tra** (l.277-334): 6 ô text (`certificateNumber`, `cnIssuedPlace`, `forWhom`, `shipInspectionCertificateNumber`, `shipInspectionIssuedPlace`, `shipInspectionForWhom`) tối đa 500 ký tự; vượt thì báo "Vui lòng nhập tối đa 500 ký tự", cuộn và focus ô lỗi đầu tiên, không gửi form.
- **Lưu**: ghi đè mọi cột pháp lý của tài sản kể cả null (`AppraisalReportAssetServiceImpl:96-113` ghi thêm 5 cột `shipInspection*` cho nhóm SHIP); file mới upload thêm vào, file cũ giữ; xóa file là xóa cứng. Nút "Tải báo cáo" và "Lưu" ở cuối form.

## Khác hệ thống cũ (ghi vào Javadoc của `LegalShipAssetForm.vue`)

1. Tiêu đề mục 4 là "Giấy tờ đính kèm" (cũ ghi "4. Giấy tờ pháp lý", trùng tên mục 3) để thống nhất với form Ô tô.
2. Ngày nhập và lưu theo ISO `YYYY-MM-DD` (cũ nhập `dd/mm/yyyy`); chỉ đổi sang `dd tháng MM năm yyyy` khi sinh văn bản (hàm `formatLegalDate` sẵn có).
3. Báo lỗi độ dài 500 ký tự bằng vee-validate + zod (thông điệp `Tối đa 500 ký tự` của helper sẵn có), không bằng nhãn `ship-legal-error`.

## Task F07: Form Pháp lý tàu

**Files:**
- Create: `F components/appraisal-report/edit/legal/LegalShipAssetForm.vue`
- Modify: `F components/appraisal-report/edit/legal/LegalTab.vue` (rẽ nhánh SHIP), `F lib/report-edit/legal-documents-text.ts` (thêm `shipLegalText`)
- Test: `F lib/report-edit/__tests__/legal-documents-text.spec.ts` (thêm), `F components/appraisal-report/edit/legal/__tests__/LegalShipAssetForm.spec.ts` (mới), `F components/appraisal-report/edit/legal/__tests__/LegalTab.spec.ts` (thêm 2 ca)

**Interfaces:**
- Consumes (F01): `LegalAsset` (có 5 field `shipInspection*`), `LegalAssetInput`, `ASSET_TYPE.SHIP`; sẵn có: `saveLegalAsset`, `deleteLegalFile`, `formatLegalDate`, `LegalAttachedFiles`, `AutoTextarea`, `DatePicker`, `FormField`, `FormActions`, `DownloadReportButton`, `useDirtyTab`, `confirmDelete`, `toast`, `toTypedSchema`, `errorMessage`.
- Produces:
  - `legal-documents-text.ts`:
    - `export interface ShipLegalFields { certificateNumber: string; cnIssuedPlace: string; gcnNoIssuedDate: string; forWhom: string; shipInspectionCertificateNumber: string; shipInspectionIssuedPlace: string; shipInspectionIssuedDate: string; shipInspectionExpiryDate: string; shipInspectionForWhom: string }` (ngày là ISO hoặc rỗng).
    - `export function shipLegalText(current: string, fields: ShipLegalFields): string`.
    - Hằng số nội bộ `SHIP_REGISTRATION_PREFIX`, `SHIP_INSPECTION_PREFIX` đúng nguyên văn ở trên; `isShipGeneratedLine(line)` chỉ kiểm tra tiền tố.
  - `LegalShipAssetForm.vue`: props `{ profileId: number; reportId: number; groupId: number; asset: LegalAsset }`, emit `saved: [asset: LegalAsset]` (cùng hợp đồng với `LegalCarAssetForm`).
  - `LegalTab.vue`: `isShip = data.assetType === ASSET_TYPE.SHIP` chọn `LegalShipAssetForm`; chuỗi `v-if isCar / v-else-if isShip / v-else` (Đất và Căn hộ). Không còn nhánh `else` nào ngầm là một loại khác ngoài Đất và Căn hộ.

- [ ] **Step 1: Đọc** `LT demo/legal-document-ship.html` l.4-335, `LT demo/legal-document-car.html` l.316-347, và `F components/appraisal-report/edit/legal/LegalCarAssetForm.vue` + `LegalTab.vue` + `legal-documents-text.ts` (mẫu cần bám).

- [ ] **Step 2: Test thất bại cho hàm thuần** — thêm `describe('shipLegalText')` vào `legal-documents-text.spec.ts`. Bộ dữ liệu chung `FULL`: `certificateNumber "TV-001"`, `cnIssuedPlace "Sở GTVT Hải Phòng"`, `gcnNoIssuedDate "2020-05-01"`, `forWhom "Công ty A"`, `shipInspectionCertificateNumber "SĐ-77"`, `shipInspectionIssuedPlace "Cục Đăng kiểm"`, `shipInspectionIssuedDate "2024-01-09"`, `shipInspectionExpiryDate "2026-01-08"`, `shipInspectionForWhom "Công ty A"`. Các ca:
  - `đủ_trường_sinh_hai_dòng_đúng_câu`: với `current = ""` kết quả đúng bằng
    ```
    Giấy chứng nhận đăng ký phương tiện thủy nội địa số: TV-001 do Sở GTVT Hải Phòng cấp ngày 01 tháng 05 năm 2020 cho Công ty A.
    Giấy chứng nhận an toàn kỹ thuật và bảo vệ môi trường phương tiện thủy nội địa số: SĐ-77 do Cục Đăng kiểm ngày 09 tháng 01 năm 2024 cho Công ty A. Giấy chứng nhận này có hiệu lực đến ngày 08 tháng 01 năm 2026.
    ```
  - `dòng_hai_dùng_chữ_ngày_không_có_chữ_cấp`: dòng 2 chứa ` ngày 09 tháng 01 năm 2024` và không chứa `cấp ngày`.
  - `chỉ_số_gcn_đăng_ký`: chỉ `certificateNumber "TV-001"` thì kết quả là `Giấy chứng nhận đăng ký phương tiện thủy nội địa số: TV-001.` (một dòng, không có dòng 2).
  - `chỉ_ngày_hết_hiệu_lực`: chỉ `shipInspectionExpiryDate "2026-01-08"` thì kết quả là `Giấy chứng nhận an toàn kỹ thuật và bảo vệ môi trường phương tiện thủy nội địa. Giấy chứng nhận này có hiệu lực đến ngày 08 tháng 01 năm 2026.`
  - `ngày_hết_hiệu_lực_nối_sau_dòng_hai_một_khoảng_trắng`: có số GCN kiểm định và hạn thì dòng 2 là `…số: SĐ-77. Giấy chứng nhận này có hiệu lực đến ngày …` (đúng một dấu cách sau dấu chấm).
  - `giữ_dòng_nhập_tay_bên_dưới`: `current = "Tàu còn thế chấp\nGiấy chứng nhận đăng ký phương tiện thủy nội địa số: CŨ."` thì kết quả là `[dòng1, dòng2, "Tàu còn thế chấp"].join("\n")` (dòng sinh cũ bị thay).
  - `dòng_sinh_luôn_nằm_đầu`: `current = "Ghi chú trước\nGiấy chứng nhận an toàn kỹ thuật và bảo vệ môi trường phương tiện thủy nội địa số: CŨ."` thì hai dòng sinh đứng trước `Ghi chú trước`.
  - `không_trường_nào_xóa_dòng_sinh_giữ_dòng_tay`: mọi trường rỗng, `current` có một dòng sinh và `"Dòng tay"` thì kết quả là `Dòng tay`.
  - `cắt_khoảng_trắng_và_bỏ_phần_rỗng`: `certificateNumber "  TV-001  "`, `forWhom "   "` thì kết quả `…số: TV-001.` (không có ` cho`).
  - `dòng_tay_trùng_tiền_tố_bị_coi_là_dòng_sinh`: dòng tay bắt đầu bằng tiền tố thì bị bỏ (khớp hành vi cũ).
  - `ngày_không_đọc_được_giữ_nguyên`: `gcnNoIssuedDate "abc"` thì câu chứa ` cấp ngày abc`.
- [ ] **Step 3: Chạy, thấy FAIL**: `cd frontend && npx vitest run src/lib/report-edit/__tests__/legal-documents-text.spec.ts`.
- [ ] **Step 4: Cài đặt `shipLegalText`** trong `legal-documents-text.ts` (tái dùng `carLine` và `formatLegalDate`; thêm câu hiệu lực theo đúng hai nhánh của hành vi cũ); thêm Javadoc nêu nguồn `legal-document-ship.html` và điểm khác dòng 2 (chữ "ngày", không "cấp"). Chạy lại: PASS.
- [ ] **Step 5: Test thất bại cho form** — `LegalShipAssetForm.spec.ts` (mount `LegalTab` với `getLegal` trả `{ assetType: 4, defaultOwner: null, assets: [ship] }`, `ship.id 51`, `asset "Tàu hàng khô Hải Long 3000T"`, `legalDocuments "Tàu còn thế chấp"`; mock `legal`, `useConfirm`, `useToast` như `LegalCarAssetForm.spec.ts`):
  - `địa_chỉ_chỉ_đọc_không_có_chủ_tài_sản_hay_mục_khác`: `[data-testid="ship-address"]` có chữ `Tàu hàng khô Hải Long 3000T`; không có `Chủ tài sản`, `3.10 Khác`, `Số vào sổ`.
  - `hiển_thị_đủ_chín_ô_của_mục_1_và_2`: các nhãn `1.1. Số GCN`, `1.2. Nơi cấp`, `1.3. Ngày cấp`, `1.4. Cho ai`, `2.1. Số GCN`, `2.2. Nơi cấp`, `2.3. Ngày cấp`, `2.4. Ngày hết hiệu lực`, `2.5. Cho ai` đều có; hai tiêu đề mục 1, 2 đúng nguyên văn; có `3. Giấy tờ pháp lý của tài sản` và `4. Giấy tờ đính kèm`.
  - `sửa_số_gcn_đăng_ký_sinh_dòng_và_giữ_dòng_tay`: gõ `1.1` là `TV-001` thì ô mục 3 bằng `Giấy chứng nhận đăng ký phương tiện thủy nội địa số: TV-001.\nTàu còn thế chấp`.
  - `sửa_hạn_hiệu_lực_nối_câu_vào_dòng_hai`: chọn `2.4` là `2026-01-08` (qua `DatePicker`, hoặc `setFieldValue` nếu khó mô phỏng) thì mục 3 chứa `Giấy chứng nhận này có hiệu lực đến ngày 08 tháng 01 năm 2026.`.
  - `trường_text_quá_500_ký_tự_báo_lỗi_và_không_lưu`: đặt `2.2` dài 501 ký tự, submit thì hiện `Tối đa 500 ký tự` và `saveLegalAsset` không được gọi.
  - `lưu_gửi_đủ_trường_tàu_và_null_cho_cột_xe`: điền vài ô rồi submit, `saveLegalAsset` nhận `(5, 11, 22, 51, body, [])` với `body` khớp `{ owner: <giữ>, asset: <giữ>, certificateNumberType: null, certificateNumber, cnIssuedPlace, gcnNo: null, gcnNoIssuedDate, gcnNoExpiryDate: null, carInspectionCertificateNumber: null, carInspectionIssuedPlace: null, carInspectionIssuedDate: null, carInspectionExpiryDate: null, forWhom, other: null, shipInspectionCertificateNumber, shipInspectionIssuedPlace, shipInspectionIssuedDate, shipInspectionExpiryDate, shipInspectionForWhom, legalDocuments, otherInfo: [] }`; ngày rỗng gửi `null`; toast `Lưu pháp lý thành công`.
  - `lưu_xong_nạp_lại_giá_trị_và_phát_saved`: sau khi lưu form không còn bẩn, phát `saved` với bản server trả.
  - `lưu_lỗi_hiện_toast_lỗi`: `saveLegalAsset` reject thì `toast.error` được gọi với `Không lưu được pháp lý` khi lỗi không có thông điệp.
  - `file_mới_được_gửi_cùng_form` và `xóa_file_hỏi_xác_nhận` (`confirmDelete` được gọi với `object: 'file đính kèm'`) — copy cách dựng của `LegalCarAssetForm.spec.ts`.
  - `form_bẩn_chặn_đổi_pill` — bỏ qua nếu `LegalTab.spec.ts` đã phủ.
  Thêm vào `LegalTab.spec.ts`: `nhóm_tàu_dùng_form_tàu` (có `[data-testid="ship-address"]`, không có `car-address`) và `nhóm_ô_tô_vẫn_dùng_form_ô_tô` (hồi quy).
- [ ] **Step 6: Chạy, thấy FAIL**: `npx vitest run src/components/appraisal-report/edit/legal`.
- [ ] **Step 7: Cài đặt `LegalShipAssetForm.vue`** theo mẫu `LegalCarAssetForm.vue`:
  - Schema zod: 6 ô text `text(500)`, 4 ô ngày `z.string()`, `legalDocuments: z.string()`.
  - `watch` chín giá trị của mục 1 và 2 (không `immediate`) rồi `setFieldValue('legalDocuments', shipLegalText(values.legalDocuments ?? '', {...}))`.
  - Bố cục: `FieldGroup` lưới `sm:grid-cols-2 xl:grid-cols-3`; ô text là `AutoTextarea single-line` (`maxlength 500`); ngày là `DatePicker clearable`; ô "Cho ai" có `placeholder="VD: cho ông A, bà B..."`; ô mục 3 là `AutoTextarea` có mô tả "Tự sinh từ mục 1–2; các dòng tự viết được giữ nguyên."; mục 4 là `LegalAttachedFiles`; `FormActions sticky` có `DownloadReportButton` và nút `Lưu` (icon `Save`).
  - `useDirtyTab(meta.dirty || pendingFiles.length > 0)`.
  - Javadoc nêu nguồn legacy và 3 điểm "Khác hệ thống cũ" ở trên.
- [ ] **Step 8: Sửa `LegalTab.vue`**: import `LegalShipAssetForm`, thêm `isShip`, đổi khối `<template v-if="selected">` thành `LegalCarAssetForm v-if="isCar"`, `LegalShipAssetForm v-else-if="isShip"`, `LegalLandAssetForm v-else`; cập nhật comment đầu file ("Đất/Căn hộ dùng LegalLandAssetForm, Ô tô LegalCarAssetForm, Tàu LegalShipAssetForm").
- [ ] **Step 9: Chạy, thấy PASS**: `npx vitest run src/components/appraisal-report/edit/legal src/lib/report-edit/__tests__/legal-documents-text.spec.ts`; sau đó `npm run lint && npm run type-check && npm run test:unit`.
- [ ] **Step 10: Commit** `git -C frontend commit -m "feat: tab pháp lý tàu thủy"` (kèm các dòng đồng tác giả theo quy ước repo).

---

# F09 — Bảng điều chỉnh tàu

## Hành vi hệ thống cũ

Nguồn: `LT demo/appraisal-panel.html` (`currencyUnit` l.4; dòng A/B l.31-79; đơn vị "Mức điều chỉnh" và "Mức giá sau điều chỉnh" l.270-283; D l.327-328; D1 l.352; E1 l.389; E4 l.437; "Nguyên tắc khống chế" `regPriceUnit` l.461-493), `LJ appraisal-panel-tabbed.js` (công thức dùng chung, không đổi).

- `currencyUnit = currencyUnitLabel(resolveCurrencyCode(assetType, currencyCode))`: giá trị `VND` hoặc `USD`. FE mới nhận sẵn `AdjustmentPanelData.currencyUnit`.
- **SHIP ẩn dòng "A. Giá thị trường (giá trước điều chỉnh)"** (l.31-35 chỉ vẽ dòng này khi **không** phải CAR và **không** phải SHIP).
- Dòng giá đầu tiên của tàu (l.64-79): STT là **A** (đất, căn hộ, ô tô là B), nhãn **"Giá bán"**, đơn vị `currencyUnit`, cột TS TĐG trống, ba cột so sánh là `landUseRightPrice` của từng cột (server đã tính: `landUseRightPrice = giá giao dịch, thiếu thì giá ước tính`; `marketPrice` luôn 0 cho tàu).
- Các ô đơn vị sau dùng `currencyUnit` cho CAR và SHIP (đất, căn hộ vẫn "VND/m²"): "Mức điều chỉnh" (l.270), "Mức giá sau điều chỉnh" (l.282), D "Mức giá chỉ dẫn" (l.327). D1 (l.352), E1 (l.389), E4 (l.437) luôn dùng `currencyUnit`.
- "Nguyên tắc khống chế": đơn vị `regPriceUnit = resolveRegulatoryConstraintsPriceUnit(assetType, currencyCode)`: tàu ra `đồng` hoặc `USD` (không "/tàu", không "/m²"). Câu làm tròn nghìn: `, làm tròn {D1 hiển thị} {đơn vị}.`.
- Phương pháp định giá (l.504-546): chỉ hiện nhóm radio khi `assetType` null/1 (đất) hoặc 2 (căn hộ). Ô tô và tàu **không có** khối phương pháp, ghi chú giá nhà nước hay nút "Chi tiết loại đất ngoài QH".
- Công thức điều chỉnh (tỷ lệ theo B, cộng dồn, D1 chia 3, D2, E1–E4) **không đổi** so với CAR.

## Khác hệ thống cũ (ghi vào Javadoc)

1. Giá trị hiển thị trong bảng USD vẫn là số nguyên (cùng cách làm tròn của VNĐ; `landUseRightPrice` là số nguyên ở server).
2. Các ô "Mức điều chỉnh" và "Mức giá sau điều chỉnh" của Đất/Căn hộ ở FE hiện tại ghi "VND" (cũ ghi "VND/m²"). Task này **không** đổi hành vi đó (chỉ đổi đơn vị của Ô tô/Tàu theo `currencyUnit`); controller quyết định có đồng bộ lại không (xem mục cuối).

## Task F09: Bảng điều chỉnh tàu

**Files:**
- Modify: `F components/appraisal-report/edit/panel/PanelSummaryRows.vue`, `FactorRows.vue`, `AdjustmentFactorTable.vue`, `ValuationMethodCard.vue`, `AdjustmentPanelTab.vue`, `F lib/report-edit/adjustment-panel-calc.ts`
- Test: `F lib/report-edit/__tests__/adjustment-panel-calc.spec.ts` (thêm), `F components/appraisal-report/edit/panel/__tests__/AdjustmentPanelTab.spec.ts` (thêm), `panel/__tests__/panel-fixtures.ts` (thêm `shipGroup`, `shipPanelData`)

**Interfaces:**
- Consumes (F01): `AdjustmentPanelData.currencyUnit`, `AssetType`, `ASSET_TYPE.SHIP`, `regulatoryPriceUnit(assetType, code)` từ `lib/report-edit/currency.ts`.
- Produces:
  - `PanelInput` thêm `currencyCode?: string | null` (mã tiền tệ đã resolve, `VND`/`USD`); `assetType: AssetType`. `calculatePanel` lấy đơn vị của "Nguyên tắc khống chế" từ `regulatoryPriceUnit(input.assetType, input.currencyCode)` thay cho hằng `CAR = 3` và chuỗi `đồng/xe`, `đồng/m²` đang nằm trong `regulatoryText`. Kết quả cho Đất, Căn hộ, Ô tô **không đổi**.
  - `PanelSummaryRows.vue`: thêm prop `currencyUnit?: string` (mặc định `'VND'`). Rẽ nhánh tường minh `isShip` / `isCar` / (đất, căn hộ). Bảng tổng:

    | Dòng | Đất, Căn hộ | Ô tô | Tàu |
    |---|---|---|---|
    | A | "Giá thị trường (giá trước điều chỉnh)", VND | "Loại xe" (tên xe) | **không có** |
    | B hoặc A | B "Giá quy đổi về đơn vị so sánh chuẩn", VND/m² | B "Đơn giá", đơn vị trống | **A "Giá bán", đơn vị `currencyUnit`** |
    | Đơn vị D | VND/m² | `currencyUnit` | `currencyUnit` |
    | Đơn vị D1, E1, E4 | `currencyUnit` | `currencyUnit` | `currencyUnit` |
  - `FactorRows.vue`: prop `unit?: string` (mặc định `'VND'`) cho hai ô đơn vị "Mức điều chỉnh" và "Mức giá sau điều chỉnh".
  - `AdjustmentFactorTable.vue`: truyền `:currency-unit="data.currencyUnit ?? 'VND'"` cho `PanelSummaryRows` (cả hai `section`) và `:unit` cho `FactorRows`.
  - `ValuationMethodCard.vue`: `methods` đổi sang `switch` tường minh trên `assetType` (LAND, APARTMENT, và `CAR`/`SHIP` trả `[]` có comment lý do); không còn nhánh "mặc định là ô tô".
  - `AdjustmentPanelTab.vue`: `calculatePanel({ …, assetType: props.group.assetType, currencyCode: panel?.currencyUnit })`. Tàu không có `DEFAULT_METHOD` nên `landValuationMethod` gửi `null`. `resultSections` của tàu là `['asset']` (đã đúng nhánh `!== LAND`; thêm comment).

- [ ] **Step 1: Đọc** `LT demo/appraisal-panel.html` l.1-100, 262-290, 320-330, 345-355, 385-392, 434-440, 455-495; đọc `F components/appraisal-report/edit/panel/*` và `adjustment-panel-calc.ts` đã dẫn ở trên.
- [ ] **Step 2: Test thất bại — hàm tính** (`adjustment-panel-calc.spec.ts`, dùng helper `input()` sẵn có với `assetType: 4`, `template: 'Giá trị chỉ dẫn là: '`):
  - `tàu_vnd_nguyên_tắc_khống_chế_đơn_vị_đồng`: `b [3_000_000, 3_000_000, 3_000_000]`, `rates [['0','0','0']]`, `currencyCode 'VND'` thì `regulatoryConstraints === 'Giá trị chỉ dẫn là: 3.000.000 đồng.'`.
  - `tàu_usd_nguyên_tắc_khống_chế_đơn_vị_USD`: `b [1000,1000,1000]`, `rates [['10','0','0']]`, `currencyCode 'USD'` thì giá chỉ dẫn `[1100, 1000, 1000]` và `regulatoryConstraints === 'Giá trị chỉ dẫn là: 1.033 USD.'` (trung bình thực 1033,33, lấy phần nguyên).
  - `tàu_usd_làm_tròn_nghìn_có_hậu_tố_đơn_vị`: `roundingType 1`, `b` ba giá `1_000_400`, `currencyCode 'USD'` thì `'Giá trị chỉ dẫn là: 1.000.400 USD, làm tròn 1.000.000 USD.'`.
  - `tàu_không_truyền_currencyCode_coi_là_đồng`: bỏ `currencyCode` thì đơn vị `đồng`.
  - `đất_và_ô_tô_giữ_nguyên_đơn_vị`: `assetType 1` ra `đồng/m²`, `assetType 3` ra `đồng/xe` (hồi quy; có thể đã có sẵn thì chỉ cần giữ xanh).
  - Công thức không đổi cho tàu: tái dùng một bộ số của `điều_chỉnh_luôn_theo_B_không_cộng_dồn_cơ_sở` với `assetType 4` cho cùng kết quả.
- [ ] **Step 3: Test thất bại — tab** (`AdjustmentPanelTab.spec.ts`; thêm vào `panel-fixtures.ts`: `shipGroup = { ...landGroup, id: 24, label: 'Tàu 1', assetType: 4, landValuationMethod: null }` và `shipPanelData(overrides)` = `panelData({ assetType: 4, columns: [{type:0, marketPrice:null, landUseRightPrice:null, carName:null}, {type:1, marketPrice:0, landUseRightPrice:1_200_000, carName:null}, {type:2 …1_200_000}, {type:3 …1_200_000}], landValuationMethod: null, currencyUnit: 'VND', regulatoryPriceUnit: 'đồng', ...overrides })`; dùng khuôn `ô_tô_nhãn_loại_xe_và_đơn_giá`):
  - `tàu_ẩn_giá_thị_trường_dòng_a_là_giá_bán`: hàng đầu của bảng bắt đầu bằng `A`, chứa `Giá bán`, `VND`, `1.200.000`; trang không chứa `Giá thị trường`, `Giá quy đổi về đơn vị so sánh chuẩn`, `Loại xe`, `VND/m²`; không có hàng nào bắt đầu bằng `B`.
  - `tàu_vnd_mọi_ô_đơn_vị_VND_nguyên_tắc_khống_chế_đồng`: `#panel-regulatory` chứa `1.200.000 đồng.` và không chứa `đồng/`.
  - `tàu_usd_mọi_ô_đơn_vị_USD`: `shipPanelData({ currencyUnit: 'USD', regulatoryPriceUnit: 'USD' })` thì các hàng "Mức điều chỉnh", "Mức giá sau điều chỉnh", D, D1, E1, E4 và "Giá bán" đều chứa `USD` và không chứa `VND`; `#panel-regulatory` chứa `1.200.000 USD.`.
  - `tàu_không_có_phương_pháp_định_giá`: không có `Phương pháp định giá`, `Chi tiết loại đất ngoài QH`, `Ghi chú`; có `Nguyên tắc khống chế`.
  - `tàu_lưu_gửi_regulatoryConstraints_theo_đơn_vị_và_không_gửi_phương_pháp`: submit thì `savePanel` nhận `body.landValuationMethod === null` và `body.synthetic.regulatoryConstraints === 'Giá trị chỉ dẫn là: 1.200.000 USD.'` (với bản USD).
  - `tàu_kết_quả_nhúng_dùng_section_asset`: stub `EvaluationResultTables` có `data-sections="asset"` và `data-agricultural="false"`.
  - `ô_tô_không_đổi`: các ca ô tô sẵn có vẫn xanh (đơn vị `VND`, `đồng/xe`).
- [ ] **Step 4: Chạy, thấy FAIL**: `npx vitest run src/lib/report-edit/__tests__/adjustment-panel-calc.spec.ts src/components/appraisal-report/edit/panel`.
- [ ] **Step 5: Cài đặt** `adjustment-panel-calc.ts` (import `regulatoryPriceUnit`, bỏ hằng `CAR` và hai chuỗi đơn vị, đổi `assetType: AssetType`), rồi `PanelSummaryRows.vue`, `FactorRows.vue`, `AdjustmentFactorTable.vue`, `ValuationMethodCard.vue`, `AdjustmentPanelTab.vue` theo Interfaces. Cập nhật Javadoc đầu `PanelSummaryRows.vue` (`top`: A/B theo loại; tàu chỉ có dòng A "Giá bán") và Javadoc của tab (dòng "Khác hệ thống cũ" mục 1, 2 ở trên).
- [ ] **Step 6: Chạy, thấy PASS** các file ở Step 4, rồi `npm run lint && npm run type-check && npm run test:unit`.
- [ ] **Step 7: Commit** `feat: bảng điều chỉnh tàu thủy theo đơn vị tiền tệ`.

---

# F10 — Modal tìm TSSS tàu từ kho và hàm điền cột

## Hành vi hệ thống cũ

Nguồn: `LJ comparative-information-ship-search.js` (hằng l.9-50; hàm tiện ích l.56-120; trạng thái đồng bộ l.134-155; panel chi tiết l.156-188; item l.211-292; tìm và tải thêm l.314-392; điền cột l.393-443; bỏ chọn l.445-471; mở/đóng modal l.473-492; cuộn l.520-540), `LT demo/comparative-information-ship-search-modals.html`.

- Tiêu đề "Tìm kiếm tài sản so sánh (tàu thủy)". Ô "Chủng loại / Từ khóa" (placeholder "Nhập chủng loại tàu để tìm kiếm...") kèm nút "Tìm kiếm"; Enter trong ô cũng tìm. Hai cột: "Danh sách kết quả" và "Chi tiết tài sản".
- **Mở modal** (`openModal`): từ khóa lấy sẵn bằng nội dung ô **Chủng loại** (`address`) của cột đang bấm (đã `trim`); đặt lại danh sách, trang, panel chi tiết; **nếu có từ khóa thì tự tìm ngay**, rỗng thì không tìm.
- **Tìm** (`fetchSearchResults`): từ khóa rỗng khi bấm Tìm thì cảnh báo "Vui lòng nhập chủng loại tàu để tìm kiếm." và không gọi server. API `GET search-compare-informations?keyword&assetType=4&pageNo&pageSize=20`. Tìm mới luôn được gửi và hủy/bỏ kết quả của request đang chạy (số thứ tự `searchRequestSeq`); tải thêm dùng đúng từ khóa của lần tìm hiện tại (không đọc lại ô đã sửa dở) và chỉ chạy khi còn trang và không đang tải.
- **Cuộn tải thêm**: danh sách cuộn tới cách đáy ≤ **80px** (`SCROLL_LOAD_THRESHOLD`) thì tải trang tiếp. Có dòng "Đang tải thêm..." khi tải trang ≥ 2 và dòng "Không có kết quả phù hợp." khi trang đầu rỗng.
- **Mỗi kết quả** (l.211-292):
  - Tiêu đề = `address` (thiếu thì `TSSS #n`).
  - Badge vàng ghi **mã tiền** khi khác tiền tệ nhóm, tooltip "Khác đơn vị tiền tệ của nhóm (X)".
  - Dòng thông số 1: "Năm đóng · Trọng tải · Công suất". Dòng 2: "Kích thước · CLCL". Phần nào không có (`—`) thì bỏ khỏi dòng; cả dòng rỗng thì bỏ dòng.
  - Dòng giá: "Giá rao bán: **N** {đơn vị} — Giá giao dịch: **N** {đơn vị}" với đơn vị là `đ` nếu VND, ngược lại là mã tiền.
  - Dòng meta: nguồn · "Cập nhật {ngày}".
  - Hai nút **Chọn TSSS** (khác tiền tệ thì `disabled`, tooltip "Không thể chọn: TSSS dùng {X}, nhóm dùng {Y}") và **Chi tiết**.
  - Tiền tệ của item thiếu thì coi là `VND`.
- **Panel chi tiết** (`DETAIL_FIELDS` l.33-50), theo thứ tự: Chủng loại, Nguồn tham khảo, Thông tin liên hệ, Năm đóng, Nơi đóng, Mục đích sử dụng, Cấp phương tiện, Chất lượng còn lại (%), Trọng tải toàn phần (tấn), Kích thước (Lmax x Bmax), Công suất (HP), Loại tiền, Giá rao bán, Giá giao dịch, Giá ước tính (kèm đơn vị), Ngày cập nhật. Giá trị thiếu hiển thị `—`. Chưa chọn thì hiện "Chọn một kết quả và bấm "Chi tiết" để xem trước khi áp dụng."
- **Chọn TSSS** (`fillFormWithShipCompareInfo` l.393-443), điền vào cột:
  - Văn bản: `address`, `source`, `contact`, `shipBuildYear`, `shipBuildPlace`, `shipUsagePurpose`, `shipClass`, `shipDimension`.
  - Số thập phân: `shipRemainingQualityPct`, `shipDeadweight`, `shipMainEnginePowerHp`.
  - Ba giá `forSalePrice`, `tradePrice`, `estimatePrice` làm tròn số nguyên (`toIntegerStringForInput`).
  - Trường kho là `null` thì ô cột **được xóa trắng** (khác ô tô, không giữ giá trị cũ).
  - Gán `compareInformationId`, khóa và bỏ tick ô đồng bộ, hiện nút "Bỏ chọn", đóng modal, toast "Tải dữ liệu tài sản so sánh thành công".
- **Bỏ chọn** (`clearShipCompareColumn` l.445-471): xóa `compareInformationId` và đúng **14 ô** (`SHIP_COMPARE_COLUMN_FIELD_SUFFIXES` l.27-31: `address, source, contact, shipBuildYear, shipBuildPlace, shipUsagePurpose, shipClass, shipRemainingQualityPct, shipDeadweight, shipDimension, shipMainEnginePowerHp, forSalePrice, tradePrice, estimatePrice`), mở lại ô đồng bộ (nếu có quyền), toast "Đã bỏ chọn tài sản so sánh {idx}.". Không đụng `shipForSalePriceUsd`.

## Khác hệ thống cũ

1. Phần "điền cột, xóa cột" là hàm thuần có test (cũ gắn chặt vào DOM).
2. Hiển thị ngày cập nhật theo `dd/MM/yyyy` (cũ in nguyên chuỗi server).
3. Từ khóa rỗng hiện lỗi ngay dưới ô nhập (`role="alert"`) thay cho toast cảnh báo.

## Task F10: `ShipCompareSearchModal` và `ship-compare.ts`

**Files:**
- Create: `F components/appraisal-report/edit/compare/ShipCompareSearchModal.vue`, `F lib/report-edit/ship-compare.ts`
- Test: `F components/appraisal-report/edit/compare/__tests__/ShipCompareSearchModal.spec.ts`, `F lib/report-edit/__tests__/ship-compare.spec.ts`, `F lib/report-edit/__tests__/ship-compare-fixtures.ts` (mới: `shipKhoItem(overrides)` dựng `ShipCompareInformationItem` đầy đủ field null và `shipPage(items, page, totalPages)`)

**Interfaces:**
- Consumes (F01): `ShipCompareInformationItem`, `searchShipCompareInformations(keyword, page, size, signal?)`, `CompareFormRow` (type-only import từ `compare-common.ts` để không tạo vòng phụ thuộc giá trị). Sẵn có: `AppModal`, `formatNumber`, `formatIsoDate`, `errorMessage`. `ShipCompareInformationItem.updated` là chuỗi ngày ISO `YYYY-MM-DD` (Quyết định #7), `formatIsoDate` đổi sang `dd/MM/yyyy`.
- Produces `lib/report-edit/ship-compare.ts` (chỉ hàm thuần, không import Vue):
  - `export const SHIP_COMPARE_KEYS = ['address','source','contact','shipBuildYear','shipBuildPlace','shipUsagePurpose','shipClass','shipRemainingQualityPct','shipDeadweight','shipDimension','shipMainEnginePowerHp','forSalePrice','tradePrice','estimatePrice'] as const` (đúng 14 ô, đúng thứ tự cũ).
  - `export function shipPatchFromKho(item: ShipCompareInformationItem): Partial<CompareFormRow>`: sao 8 ô văn bản và số, 3 ô thập phân (null giữ null), ba giá `Math.round` (null giữ null).
  - `export function fillShipFromKho(row: CompareFormRow, item: ShipCompareInformationItem): CompareFormRow` = `{ ...row, ...shipPatchFromKho(item), compareInformationId: item.id, syncToCompareInfo: 0 }`.
  - `export function emptyShipColumn(row: CompareFormRow): CompareFormRow`: 14 ô thành `null`, `compareInformationId: null`, `syncToCompareInfo: 0`; các field khác (kể cả `shipForSalePriceUsd`, `id`, `type`) giữ nguyên.
  - `export function shipItemCurrency(item: { currencyCode: string | null }): string` (thiếu thì `'VND'`).
  - `export function isShipCurrencyMismatch(item, groupCurrencyCode: string): boolean`.
  - `export function shipMoneyUnit(currencyCode: string): string` (`VND` thì `'đ'`, ngược lại là mã).
  - `export const SHIP_LOCKED_BY_SEARCH = 'Đã chọn tài sản từ tìm kiếm; không dùng đồng bộ.'`.
- Produces `ShipCompareSearchModal.vue`: props `{ title: string; initialKeyword: string; groupCurrencyCode: string }`; `v-model:open`; emit `select: [item: ShipCompareInformationItem]`. `data-testid`: `ship-results` (danh sách cuộn), `ship-result` (mỗi item), `ship-preview` (panel chi tiết).

- [ ] **Step 1: Đọc** toàn bộ `LJ comparative-information-ship-search.js`, `LT demo/comparative-information-ship-search-modals.html`, và `F compare/CarCompareSearchModal.vue` + `__tests__/CarCompareSearchModal.spec.ts` + `detail-test-helpers.ts` (mẫu và helper `modal`, `type`, `buttonByText`).
- [ ] **Step 2: Test thất bại — hàm thuần** (`ship-compare.spec.ts`; `row` dựng bằng `toFormRow(compareRow({...}))` hoặc object tối thiểu ép kiểu):
  - `patch_từ_kho_điền_14_ô_và_làm_tròn_giá`: item `{ address 'Tàu hàng khô 3000T', source 'https://x.vn/1', contact '0900', shipBuildYear 2012, shipBuildPlace 'Hải Phòng', shipUsagePurpose 'Chở hàng', shipClass 'VR-SI', shipRemainingQualityPct 62.5, shipDeadweight 3000.5, shipDimension '85 x 14', shipMainEnginePowerHp 1200, forSalePrice 10_000_000.6, tradePrice 9_500_000.4, estimatePrice null }` thì ba giá là `10_000_001`, `9_500_000`, `null`, các ô còn lại bằng giá trị kho.
  - `ô_null_trong_kho_xóa_trắng_ô_cột`: cột đang có `shipBuildYear 1990`, `shipDimension 'cũ'`, kho `null` thì hai ô thành `null`.
  - `fillShipFromKho_gán_id_bỏ_tick_và_giữ_trường_khác`: `compareInformationId === item.id`, `syncToCompareInfo === 0`, `id`/`type`/`shipForSalePriceUsd` giữ nguyên, `landDimension` không bị đụng.
  - `emptyShipColumn_xóa_đúng_14_ô_và_id`: 14 khóa của `SHIP_COMPARE_KEYS` đều `null`, `compareInformationId null`, `syncToCompareInfo 0`, `shipForSalePriceUsd` và `carName` giữ nguyên.
  - `SHIP_COMPARE_KEYS_có_14_khóa_không_trùng`.
  - `lệch_tiền_tệ`: item `USD` với nhóm `VND` là `true`; item `null` với nhóm `VND` là `false`; item `null` với nhóm `USD` là `true`; item `VND` với nhóm `VND` là `false`.
  - `đơn_vị_tiền`: `shipMoneyUnit('VND') === 'đ'`, `shipMoneyUnit('USD') === 'USD'`.
- [ ] **Step 3: Test thất bại — modal** (`ShipCompareSearchModal.spec.ts`; mock `searchShipCompareInformations`; `mountModal(props)` với `open: true`, `title: 'Tài sản so sánh 1'`, `initialKeyword`, `groupCurrencyCode: 'VND'`):
  - `mở_với_từ_khóa_có_sẵn_tự_tìm_ngay`: `initialKeyword '  tàu hàng khô '` thì ô nhập (`aria-label="Chủng loại / Từ khóa"`) có giá trị `tàu hàng khô`, `search` được gọi một lần với `('tàu hàng khô', 0, 20, expect.anything())` mà không cần bấm.
  - `mở_không_từ_khóa_không_gọi_máy_chủ`: `initialKeyword ''` thì `search` không được gọi, có chữ `Nhập chủng loại tàu để tìm tài sản so sánh.`.
  - `tìm_rỗng_báo_lỗi_và_không_gọi_máy_chủ`: sửa ô thành khoảng trắng, submit thì hiện `Vui lòng nhập chủng loại tàu để tìm kiếm.`, không gọi thêm.
  - `enter_trong_ô_tìm_mới_từ_trang_đầu`: submit form với từ khóa mới gọi `('từ khóa mới', 0, 20, …)` và thay toàn bộ danh sách.
  - `hiển_thị_thông_số_giá_đơn_vị_nguồn_ngày`: item VND `{address 'Tàu A', shipBuildYear 2012, shipDeadweight 3000.5, shipMainEnginePowerHp 1200, shipDimension '85 x 14', shipRemainingQualityPct 62.5, forSalePrice 10_000_000, tradePrice 9_500_000, source 'Nguồn X', updated '2026-03-05'} (Quyết định #7: ISO ngày)` thì item chứa `Tàu A`, `Năm đóng: 2012`, `Trọng tải: 3.000,5`, `Công suất: 1.200`, `Kích thước: 85 x 14`, `CLCL: 62,5`, `Giá rao bán: 10.000.000 đ`, `Giá giao dịch: 9.500.000 đ`, `Nguồn X`, `Cập nhật 05/03/2026`.
  - `item_usd_dùng_mã_tiền_làm_đơn_vị`: nhóm `USD`, item `USD` thì chứa `Giá rao bán: … USD`, không có badge.
  - `thiếu_thông_số_bỏ_phần_và_dòng_rỗng`: item chỉ có `address` thì không có `Năm đóng`, `Kích thước`, `CLCL`.
  - `tiêu_đề_dự_phòng_khi_không_có_chủng_loại`: `address null` thì `TSSS #1`.
  - `cuộn_gần_cuối_tải_trang_tiếp_và_nối_danh_sách`: trang 0 có 2 item, `totalPages 2`; cuộn tới cách đáy 80px (đặt `scrollHeight 1000`, `clientHeight 400`, `scrollTop 520`) thì gọi `('…', 1, 20, …)` và danh sách thành 4 item. Cuộn khi còn cách đáy 200px thì **không** gọi.
  - `hết_trang_không_tải_thêm` và `đang_tải_không_gọi_trùng`.
  - `tải_thêm_dùng_từ_khóa_của_lần_tìm_hiện_tại`: sửa ô nhập sang chữ khác mà chưa bấm Tìm, rồi cuộn, thì trang 1 vẫn gọi với từ khóa cũ.
  - `kết_quả_của_lần_tìm_cũ_bị_bỏ_khi_tìm_mới`: hai lời gọi, lời gọi đầu giải quyết sau lời gọi hai, danh sách chỉ có kết quả của lời gọi hai.
  - `khác_tiền_tệ_hiện_badge_và_khóa_nút_chọn`: nhóm `VND`, item `USD` thì có badge `USD` (tooltip `Khác đơn vị tiền tệ của nhóm (VND)`), nút `Chọn TSSS` `disabled` với `title` `Không thể chọn: TSSS dùng USD, nhóm dùng VND`; bấm không phát `select`.
  - `chọn_phát_select_với_item_và_đóng_modal`: item cùng tiền tệ, bấm `Chọn TSSS` thì phát `select` với đúng item và `update:open` `false`.
  - `chi_tiết_hiện_đủ_16_dòng_theo_thứ_tự`: bấm `Chi tiết` thì panel `ship-preview` có đúng 16 nhãn theo thứ tự ở mục Hành vi, giá có đơn vị, thiếu thì `—`, `Loại tiền` thiếu thì `VND`; item đang xem có viền nổi bật.
  - `chưa_chọn_chi_tiết_hiện_gợi_ý`.
  - `lỗi_máy_chủ_hiện_thông_báo_và_không_tải_thêm`: `search` reject `ApiError` thì hiện thông điệp trong `Alert`.
  - `không_có_kết_quả`: trang đầu rỗng thì hiện `Không có kết quả phù hợp.`.
  - `đóng_modal_hủy_request_đang_chạy` (signal bị abort).
- [ ] **Step 4: Chạy, thấy FAIL**: `npx vitest run src/lib/report-edit/__tests__/ship-compare.spec.ts src/components/appraisal-report/edit/compare/__tests__/ShipCompareSearchModal.spec.ts`.
- [ ] **Step 5: Cài đặt** `ship-compare.ts` rồi `ShipCompareSearchModal.vue` theo khuôn `CarCompareSearchModal.vue`:
  - `watch(open, …, { immediate: true })`: khi mở đặt `keyword = initialKeyword.trim()`, xóa mọi trạng thái, nếu `keyword` không rỗng thì `loadPage(0)`.
  - Ngưỡng cuộn `80`px, kích thước trang `20`, `AbortController` cho mỗi lần tải; `hasMore = result.page + 1 < result.totalPages`.
  - `AppModal size="xl"`, tiêu đề `Tìm kiếm tài sản so sánh (tàu thủy) — {title}`; lưới `lg:grid-cols-2`; nút `Chọn TSSS` và `Chi tiết` là `Button variant="outline" size="sm"`; badge dùng `Badge variant="warning"`; thông báo lỗi dùng `Alert`; footer có nút `Đóng`.
  - Javadoc nêu nguồn legacy và các điểm "Khác hệ thống cũ".
- [ ] **Step 6: Chạy, thấy PASS** như Step 4, rồi `npm run lint && npm run type-check && npm run test:unit`.
- [ ] **Step 7: Commit** `feat: modal tìm tài sản so sánh tàu thủy từ kho`.

---

# F11 — Kết quả thẩm định tàu và chọn tiền tệ

## Hành vi hệ thống cũ

Nguồn: `LT appraisal-records/evaluation-results.html` (hằng `SHIP_ASSET_TYPE = 4` l.700; bảng tàu `buildShipEvaluationResultTable` l.1043-1073; chọn bảng theo loại l.1100-1135; `afterEvaluationSaved` l.1355-1375; kiểm tra và payload khi lưu l.1400-1500; khối tiền tệ l.1569-1597; bật/tắt tỷ lệ l.1775-1796).

- **Khối tiền tệ (chỉ nhóm SHIP)**, nằm trong cùng form với nút Lưu:
  - Select `currencyCode` có hai lựa chọn "VNĐ" và "USD", giá trị ban đầu từ nhóm (`USD` thì chọn USD, còn lại VND).
  - Khi chọn USD hiện thêm hai ô:
    - "Tỷ lệ chuyển đổi (1 USD = ? VNĐ)": chỉ nhận chữ số, tối đa 9 chữ số; rời ô thì định dạng `vi-VN` (ví dụ `25.450`); giá trị ban đầu là `exchangeRate` của nhóm, mặc định 1.
    - "Ghi chú chuyển đổi": textarea tối đa 1000 ký tự.
  - Đổi sang VND thì ẩn hai ô và đặt tỷ lệ về `1`.
  - **Đổi select không gọi API**, chỉ bật/tắt hiển thị; mọi thứ gửi cùng nút Lưu.
- **Lưu** (l.1400-1500):
  - Nhóm USD có nhập tỷ lệ: bỏ dấu "." phân cách rồi kiểm tra `^[1-9][0-9]{0,8}$`; sai thì toast lỗi "Tỷ lệ chuyển đổi phải là số nguyên từ 1 đến 999.999.999" và không gửi. Để trống thì gửi `null`.
  - Payload thêm `currencyCode`, `exchangeRate` (USD: số đã nhập hoặc null; VND: `1`), `currencyNote`.
  - Server tính lại tổng và bằng chữ theo tiền tệ mới.
- **Sau khi lưu** (`afterEvaluationSaved`): nếu tiền tệ khác ban đầu thì đánh dấu tab So sánh và tab Bảng điều chỉnh "chưa tải"; sau đó **tải lại bảng kết quả** (tiêu đề cột và "Bằng chữ" theo tiền tệ vừa lưu).
- **Bảng kết quả tàu** (`buildShipEvaluationResultTable`): dữ liệu `data.ship.{amount, vndAmount, roundedAmount}`, `data.currencyCode`, `data.totalPriceWordAppraisal`, `data.assetName`. Nhãn dòng 1 là `"Giá trị " + chữ-đầu-thường(assetName)`. Câu bằng chữ là `"Bằng chữ: " + totalPriceWordAppraisal + " ./."` (rỗng thì để trống). Không có cột STT.
  - **VND (2 cột)**: "Hạng mục" (58%), "Thành tiền (VNĐ)" (42%). Dòng 1 = `amount`; dòng đậm giữa "Làm tròn" = `roundedAmount` (thiếu thì `roundingMoney`, thiếu nữa thì 0); dòng "Bằng chữ" gộp 2 cột.
  - **Ngoại tệ** (`currencyCode` khác VND): 3 cột "Hạng mục" (40%), "Thành tiền ({mã})" (30%), "Thành tiền (VNĐ)" (30%). Dòng 1 = `amount` và `vndAmount` (thiếu thì `amount`); dòng "Làm tròn" = `amount` (**giữ nguyên, không làm tròn**) và `roundedAmount`; dòng "Bằng chữ" gộp 3 cột.
  - Bằng chữ luôn đọc theo VNĐ.
- Nút Lưu hiện cho BGD, TPTD, NVTD, NVKD (đã có ở FE hiện tại).

## Khác hệ thống cũ

1. Khối tiền tệ bị khóa (`disabled`) với người không có quyền lưu (cũ hiện cho sửa nhưng không có nút Lưu).
2. Không cần đánh dấu "chưa tải" cho tab So sánh và tab Bảng điều chỉnh: FE mới hủy tab khi ẩn nên tự tải lại khi mở (`00-index.md`, "Khác hệ thống cũ" mục 5).
3. Ô tỷ lệ dùng `DecimalInput` 0 chữ số lẻ (tự nhóm nghìn theo `vi-VN`) thay cho ô text tự định dạng khi rời ô; vượt 9 chữ số vẫn bị chặn bởi bước kiểm tra khi lưu.

## Task F11: Bảng kết quả tàu và khối tiền tệ

**Files:**
- Create: `F components/appraisal-report/edit/results/ShipResultTable.vue`, `F components/appraisal-report/edit/results/ShipCurrencyBlock.vue`, `F lib/report-edit/exchange-rate.ts`
- Modify: `F components/appraisal-report/edit/results/AssetResultTable.vue`, `F components/appraisal-report/edit/results/EvaluationResultTab.vue`
- Test: `F lib/report-edit/__tests__/exchange-rate.spec.ts` (mới), `F components/appraisal-report/edit/results/__tests__/EvaluationResultTables.spec.ts` (thêm), `EvaluationResultTab.spec.ts` (thêm), `results/__tests__/fixtures.ts` (thêm `shipResult(overrides)`)

**Interfaces:**
- Consumes (F01): `EvaluationResult.ship/currencyCode/exchangeRate/currencyNote`, `SaveEvaluationResultBody` có 3 field tiền tệ, `ASSET_TYPE.SHIP`. Sẵn có: `formatResultMoney`, `lowerFirst`, `readMoney`, `DecimalInput`, `SelectBox`, `AutoTextarea`.
- Produces:
  - `exchange-rate.ts`:
    - `export const EXCHANGE_RATE_MAX = 999_999_999` và `export const EXCHANGE_RATE_ERROR = 'Tỷ lệ chuyển đổi phải là số nguyên từ 1 đến 999.999.999'`.
    - `export function validateExchangeRate(value: number | null | undefined): string | null` (trống hợp lệ; còn lại phải là số nguyên 1..`EXCHANGE_RATE_MAX`, sai thì trả `EXCHANGE_RATE_ERROR`).
    - `export function formatExchangeRate(value: number | null | undefined): string` (`25450` thành `'25.450'`, trống thành `''`).
    - `export function currencyPayload(code: 'VND' | 'USD', rate: number | undefined, note: string): { currencyCode: 'VND' | 'USD'; exchangeRate: number | null; currencyNote: string | null }` (VND: rate `1`; USD: `rate ?? null`; `currencyNote` là `note` hoặc `null` khi rỗng).
  - `ShipResultTable.vue`: props `{ result: EvaluationResult }`; chỉ vẽ `<Table>` của tàu (VND 2 cột, ngoại tệ 3 cột). Không có cột STT.
  - `AssetResultTable.vue`: `section` + `h3 "Kết quả thẩm định giá tài sản"`; **rẽ tường minh** `LAND` / `APARTMENT` / `CAR` (bảng 3 cột STT hiện có) / `SHIP` (`ShipResultTable`); loại tài sản khác không vẽ bảng. Không còn `v-else` ngầm hiểu là ô tô.
  - `ShipCurrencyBlock.vue`: `v-model:currency-code` (`'VND' | 'USD'`), `v-model:exchange-rate` (`number | undefined`), `v-model:note` (`string`), props `{ disabled?: boolean; rateError?: string }`. Chọn USD hiện ô tỷ lệ và ghi chú; chọn VND ẩn hai ô và đặt tỷ lệ `1`.
  - `EvaluationResultTab.vue`: với nhóm SHIP thêm trạng thái tiền tệ (`currencyCode`, `exchangeRate`, `currencyNote` và bản đã lưu), dirty tính cả ba, lưu gửi 3 field, lưu xong **tải lại kết quả** (`load(false)`). Nhóm khác: hành vi cũ, **không** gửi 3 field.

- [ ] **Step 1: Đọc** `LT appraisal-records/evaluation-results.html` l.1043-1135, 1355-1500, 1569-1597, 1775-1796; `F results/AssetResultTable.vue`, `EvaluationResultTab.vue`, `EvaluationResultTables.vue` và các spec hiện có.
- [ ] **Step 2: Test thất bại — hàm thuần** (`exchange-rate.spec.ts`):
  - `tỷ_lệ_hợp_lệ`: `1`, `25450`, `999_999_999` đều `null`; trống (`undefined`, `null`) cũng `null`.
  - `tỷ_lệ_sai_báo_lỗi`: `0`, `-5`, `1.5`, `1_000_000_000`, `NaN` đều trả `EXCHANGE_RATE_ERROR`.
  - `định_dạng_vi_VN`: `25450` thành `'25.450'`, `1` thành `'1'`, `undefined` thành `''`.
  - `payload_vnd_luôn_rate_1`: `currencyPayload('VND', 25450, 'x')` ra `{ currencyCode: 'VND', exchangeRate: 1, currencyNote: 'x' }`.
  - `payload_usd_giữ_tỷ_lệ_và_ghi_chú`, `payload_usd_tỷ_lệ_trống_ra_null`, `payload_ghi_chú_rỗng_ra_null`.
- [ ] **Step 3: Test thất bại — bảng** (`EvaluationResultTables.spec.ts`; `shipResult()` trong `fixtures.ts` = `evaluationResult({ assetType: 4, assetName: 'Tàu Hải Long', car: null, ship: {...}, currencyCode: 'VND', exchangeRate: 1, currencyNote: null, totalPriceWordAppraisal: '...' })`; render `EvaluationResultTables` với `sections: ['asset']`):
  - `tàu_vnd_hai_cột_không_có_stt`: `ship {amount 1_234_567, exchangeRate 1, vndAmount 1_234_567, roundedAmount 1_235_000}`, `totalPriceWordAppraisal 'Một triệu hai trăm ba mươi lăm nghìn đồng'` thì tiêu đề cột đúng `['Hạng mục','Thành tiền (VNĐ)']`; hàng 1 `Giá trị tàu Hải Long` | `1.234.567`; hàng `Làm tròn` | `1.235.000`; hàng cuối `Bằng chữ: Một triệu hai trăm ba mươi lăm nghìn đồng ./.` có `colspan="2"`; không có chữ `STT`.
  - `tàu_usd_ba_cột`: `currencyCode 'USD'`, `ship {amount 49_990, exchangeRate 25_450, vndAmount 1_272_245_500, roundedAmount 1_272_000_000}` thì tiêu đề `['Hạng mục','Thành tiền (USD)','Thành tiền (VNĐ)']`; hàng 1: `49.990` | `1.272.245.500`; hàng `Làm tròn`: `49.990` (cột ngoại tệ giữ `amount`) | `1.272.000.000`; hàng `Bằng chữ` có `colspan="3"`.
  - `tàu_ngoại_tệ_thiếu_vndAmount_dùng_amount`: `vndAmount null` thì ô VNĐ của hàng 1 bằng `amount`.
  - `tàu_vnd_thiếu_roundedAmount_dùng_roundingMoney`: `ship.roundedAmount null`, `roundingMoney 2_000_000` thì `Làm tròn` là `2.000.000`.
  - `tàu_thiếu_ship_hiển_thị_ô_trống_không_lỗi`: `ship null` thì không ném lỗi, các ô tiền trống (quy ước 0 và null hiển thị trống).
  - `tàu_không_rơi_vào_nhánh_ô_tô`: `car` có dữ liệu khác `ship` thì bảng tàu chỉ dùng `ship` (không dùng `car.amount`).
  - `ô_tô_giữ_nguyên_bảng_ba_cột_stt` (hồi quy) và `loại_lạ_không_vẽ_hàng_dữ_liệu` (`assetType: 99 as never` không ra dòng "Giá trị").
- [ ] **Step 4: Test thất bại — tab** (`EvaluationResultTab.spec.ts`; mock giống ca sẵn có, `useAuthStore` có quyền `BGD`):
  - `nhóm_tàu_hiện_khối_tiền_tệ_vnd_ẩn_tỷ_lệ_và_ghi_chú`: nhãn `Đơn vị tiền tệ` hiện, giá trị `VNĐ`; không có `Tỷ lệ chuyển đổi`, `Ghi chú chuyển đổi`.
  - `nhóm_khác_không_hiện_khối_tiền_tệ`: Đất, Căn hộ, Ô tô không có `Đơn vị tiền tệ`.
  - `nhóm_usd_nạp_tỷ_lệ_và_ghi_chú_đã_lưu`: result `currencyCode 'USD'`, `exchangeRate 25_450`, `currencyNote 'Tỷ giá VCB'` thì ô tỷ lệ hiển thị `25.450`, ghi chú `Tỷ giá VCB`.
  - `chọn_usd_hiện_tỷ_lệ_và_ghi_chú_không_gọi_api`: đổi select sang USD thì hiện hai ô, `getEvaluationResult` không bị gọi thêm.
  - `đổi_về_vnd_ẩn_và_tỷ_lệ_về_1`: từ USD (25.450) sang VND rồi lưu thì `exchangeRate === 1`.
  - `lưu_usd_gửi_ba_trường_tiền_tệ`: USD, rate `25450`, note `Tỷ giá Vietcombank` thì `saveEvaluationResult` nhận body khớp `{ currencyCode: 'USD', exchangeRate: 25450, currencyNote: 'Tỷ giá Vietcombank' }`.
  - `lưu_usd_tỷ_lệ_trống_gửi_null`: `exchangeRate: null`.
  - `lưu_tỷ_lệ_sai_báo_lỗi_không_gọi_api`: tỷ lệ `1_000_000_000` thì `toast.error('Tỷ lệ chuyển đổi phải là số nguyên từ 1 đến 999.999.999')` và `saveEvaluationResult` không được gọi.
  - `lưu_tàu_dùng_roundedAmount_của_ship_cho_roundingMoney_và_bằng_chữ_vnd`: `ship.roundedAmount 1_272_000_000` thì body có `roundingMoney: 1_272_000_000` và `totalPriceWordAppraisal: readMoney(1_272_000_000)`.
  - `lưu_tàu_xong_tải_lại_kết_quả`: sau khi lưu `getEvaluationResult` được gọi lại (một lần nữa) với `roundingMoneyType` đang chọn, ghi chú đang nhập không bị ghi đè.
  - `nhóm_không_phải_tàu_lưu_không_gửi_trường_tiền_tệ`: body của Ô tô không có khóa `currencyCode`, `exchangeRate`, `currencyNote` (`not.toHaveProperty`) và **không** tải lại sau khi lưu.
  - `đổi_tiền_tệ_hoặc_ghi_chú_đánh_dấu_bẩn_và_lưu_xong_hết_bẩn`.
  - `không_có_quyền_lưu_khóa_khối_tiền_tệ`: người chỉ có `NVPI` thì select và ô tỷ lệ `disabled`, không có nút `Lưu`.
- [ ] **Step 5: Chạy, thấy FAIL**: `npx vitest run src/lib/report-edit/__tests__/exchange-rate.spec.ts src/components/appraisal-report/edit/results`.
- [ ] **Step 6: Cài đặt** `exchange-rate.ts`, `ShipResultTable.vue`, `ShipCurrencyBlock.vue`, sửa `AssetResultTable.vue` và `EvaluationResultTab.vue`:
  - Bảng dùng `@/components/ui/table`, tiền `text-right tabular-nums`, dòng "Làm tròn" và "Bằng chữ" `font-semibold`, độ rộng cột 58/42 và 40/30/30 bằng `class` hoặc `colgroup` token Tailwind; tiêu đề cột tiền `text-right`. Ngoại tệ khi `currencyCode` có và khác `VND`.
  - Khối tiền tệ: `SelectBox` (`VNĐ`, `USD`), `DecimalInput :decimals="0" unit="VNĐ"` cho tỷ lệ, `AutoTextarea maxlength="1000"` cho ghi chú; nhãn qua `Label`; lỗi tỷ lệ `role="alert"`.
  - `EvaluationResultTab`: tính `isShip = props.group.assetType === ASSET_TYPE.SHIP`; `roundingMoney = isShip ? (result.ship?.roundedAmount ?? result.roundingMoney ?? 0) : (result.roundingMoney ?? 0)`; kiểm tra `validateExchangeRate` khi `currencyCode === 'USD'` trước khi gọi API; spread `currencyPayload(...)` vào body chỉ khi `isShip`; sau `save` của tàu gọi `await load(false)`.
  - Javadoc nêu nguồn legacy và điểm "Khác hệ thống cũ".
- [ ] **Step 7: Chạy, thấy PASS** như Step 5, rồi `npm run lint && npm run type-check && npm run test:unit`.
- [ ] **Step 8: Commit** `feat: kết quả thẩm định tàu thủy và chọn tiền tệ`.

---

# F08 — Tab Thông tin so sánh tàu

> Chạy **sau F10** (cần `ShipCompareSearchModal.vue` và `ship-compare.ts`).

## Hành vi hệ thống cũ

Nguồn: `LT demo/comparative-information-ship.html` (toàn bộ 660 dòng: bảng l.18-300, các dòng l.56-620, tỷ lệ l.623-640, chân form l.642-660), `LJ comparative-information-ship-search.js` (đồng bộ l.134-155, điền/bỏ chọn l.393-471), `LT demo/comparative-information.html` (kiểm tra CLCL `carRemainingQualityPctRange` l.2046-2056, ép kiểu `shipBuildYear` l.2153), `L controller` lưu `comparative-information-ship-tab`.

- Form có `data-currency-code` của nhóm. Bảng 5 cột: "CHỈ TIÊU SO SÁNH", "TÀI SẢN THẨM ĐỊNH GIÁ", "TÀI SẢN SO SÁNH 1/2/3". **Cả bốn cột đều nhập tay được** (không còn trường chỉ đọc do đồng bộ từ mô tả như ô tô; việc đồng bộ từ Mô tả tàu sang cột 0 xảy ra ở server khi lưu mô tả).
- Thứ tự dòng (số thứ tự theo khảo sát; mục 14 gồm ba dòng giá nên tổng cộng 17 hàng):

  | # | Nhãn | Field | Ô nhập |
  |---|---|---|---|
  | 1 | Chủng loại | `address` | textarea ≤1000. Cột 1–3 có nút kính lúp "Tìm TSSS theo chủng loại" |
  | 2 | Nguồn tham khảo | `source` | textarea ≤500 |
  | 3 | Thông tin liên hệ | `contact` | textarea ≤255 |
  | 4 | Đồng bộ lên cơ sở dữ liệu | `syncToCompareInfo` | cột 1–3: checkbox "Đồng bộ TS so sánh N"; cột TĐG: "—" |
  | 5 | Hình ảnh tài sản | | cột 1–3: nút xem/thêm ảnh |
  | 6 | Năm đóng | `shipBuildYear` | số nguyên 1900–2100 |
  | 7 | Nơi đóng | `shipBuildPlace` | text ≤1000 |
  | 8 | Mục đích sử dụng | `shipUsagePurpose` | textarea ≤500 |
  | 9 | Cấp phương tiện | `shipClass` | textarea ≤500 |
  | 10 | Chất lượng còn lại (%) | `shipRemainingQualityPct` | thập phân 0–100, thông báo "CLCL phải từ 0 đến 100" |
  | 11 | Trọng tải toàn phần (tấn) | `shipDeadweight` | thập phân |
  | 12 | Kích thước (Lmax x Bmax) | `shipDimension` | text ≤255 |
  | 13 | Công suất (HP) | `shipMainEnginePowerHp` | thập phân |
  | 14 | `Giá rao bán ({đồng\|USD})`, `Giá giao dịch (…)`, `Giá ước tính (…)` | `forSalePrice`, `tradePrice`, `estimatePrice` | số nguyên có nhóm nghìn. Đơn vị là `currencyMoneyLabel(currencyCode)` |
  | 15 | Tỷ lệ ước tính | | chỉ hiển thị = ước tính / rao bán × 100 dạng `xx,xx %`; rỗng nếu thiếu số hoặc rao bán bằng 0 |

  Không có tọa độ, bản đồ, phân tích so sánh, ô USD riêng (`shipForSalePriceUsd` không hiện trên UI, được giữ nguyên khi lưu), tỷ giá, nút đồng bộ từ mô tả.
- **Ô đồng bộ lên kho** (l.175-255), cột 1–3:
  - Checkbox tick khi `syncToCompareInformationId != null`.
  - **Quyền: KSV, TPTD, BGD** (`hasAnyAuthority('KSV','TPTD','BGD')`). Trước đây chỉ KSV, TPTD; thay đổi này áp dụng cho **mọi loại tài sản** (Đất, Căn hộ, Ô tô) tại commit đích.
  - Checkbox `disabled` khi cột đã chọn TSSS từ kho (`compareInformation != null`) hoặc người dùng không có quyền. Tooltip: "Đã chọn tài sản từ tìm kiếm; không dùng đồng bộ." hoặc "Chỉ vai trò KSV, TPTD hoặc BGD mới được thay đổi đồng bộ.".
  - Người không có quyền mà cột đang đồng bộ thì vẫn gửi `syncToCompareInfo = 1` (hidden).
  - Ghi chú dưới nhãn: "Tích chọn để đưa tài sản so sánh vào cơ sở dữ liệu; tài sản đã đồng bộ sẽ hiển thị khi tìm kiếm theo chủng loại."
  - Cột đã gắn kho có nút "Bỏ chọn tài sản so sánh".
- **Nút tìm** cạnh ô Chủng loại cột 1–3 mở modal (F10) với từ khóa là nội dung ô đó. Chọn TSSS điền 14 ô, gán `compareInformationId`, bỏ tick và khóa ô đồng bộ, toast "Tải dữ liệu tài sản so sánh thành công". Bỏ chọn xóa 14 ô và `compareInformationId`, mở lại ô đồng bộ, toast "Đã bỏ chọn tài sản so sánh {n}." (chi tiết ở F10).
- **Ảnh TSSS** (cột 1–3): dùng chung với đất, căn hộ và ô tô (xem, thêm, xóa ngay qua API ảnh TSSS).
- **Lưu**: gửi 4 cột; lưu xong tải lại tab. Chân form có "Tải báo cáo" và "Lưu" (không có "Xuất phiếu TSSS").

## Khác hệ thống cũ (ghi vào Javadoc của `ShipCompareTab.vue`)

1. Kiểm tra trước khi lưu bằng thông báo dưới ô: "Năm đóng phải từ 1900 đến 2100" (cũ chỉ ràng buộc bằng `min`/`max` của HTML) và "CLCL phải từ 0 đến 100" (đúng cũ). Server cũng kiểm tra CLCL (400).
2. Lý do khóa ô đồng bộ của cột đã chọn từ kho ghi "từ tìm kiếm" cho tàu (Đất, Căn hộ giữ "từ bản đồ").
3. Không có nút "Xuất phiếu TSSS" ở tàu (trang tàu cũ không có; xem mục cuối).
4. Tab bị hủy khi ẩn nên không cần `data-loaded=false` khi đổi tiền tệ ở tab Kết quả.

## Task F08: Tab So sánh tàu

**Files:**
- Modify (đợt F3, sau F10): `F components/appraisal-report/edit/compare/ShipCompareTab.vue` (thay vỏ của F01), `F lib/report-edit/compare-common.ts`, `F composables/useCompareForm.ts`, `F lib/report-edit/ship-compare.ts` (thêm `validateShipCompareRow`), `F api/report-edit/compare.ts` (chỉ đổi dòng comment `canSyncToKho` thành "(KSV, TPTD, BGD)")
- Test: `F components/appraisal-report/edit/compare/__tests__/ShipCompareTab.spec.ts` (mới), `F lib/report-edit/__tests__/compare-common.spec.ts` (thêm), `F composables/__tests__/useCompareForm.spec.ts` (thêm và sửa chuỗi), `F lib/report-edit/__tests__/ship-compare.spec.ts` (thêm ca kiểm tra)

**Interfaces:**
- Consumes: F10 (`ShipCompareSearchModal`, `fillShipFromKho`, `emptyShipColumn`, `SHIP_LOCKED_BY_SEARCH`), F01 (`GroupCompareData.currencyCode`, `ShipCompareInformationItem`, field ship của `GroupCompareRow`, `currencyMoneyLabel`, `ASSET_TYPE.SHIP`); sẵn có: `useCompareForm`, `CompareGrid`, `CompareRow`, `CompareField`, `KhoSyncCell`, `CompareImagesModal`, `estimateRatio`, `isComparedColumn`.
- Produces:
  - `ship-compare.ts` (thêm): `export const SHIP_YEAR_RANGE = { min: 1900, max: 2100 }`; `export const SHIP_YEAR_ERROR = 'Năm đóng phải từ 1900 đến 2100'`; `export const SHIP_CLCL_ERROR = 'CLCL phải từ 0 đến 100'`; `export function validateShipCompareRow(row: Pick<CompareFormRow,'shipBuildYear'|'shipRemainingQualityPct'>): Partial<Record<'shipBuildYear'|'shipRemainingQualityPct', string>>`.
  - `compare-common.ts`:
    - `fillFromKho(row, detail: CompareInformationDetail | ShipCompareInformationItem, assetType)` thêm nhánh `ASSET_TYPE.SHIP` gọi `fillShipFromKho`; `emptyColumn` thêm nhánh `ASSET_TYPE.SHIP` gọi `emptyShipColumn`. Các nhánh còn lại rẽ **tường minh** theo `LAND`, `APARTMENT`, `CAR`, `SHIP` (bỏ nhánh `else` ngầm hiểu là đất/ô tô; loại lạ thì trả `row` không đổi, `emptyColumn` loại lạ chỉ bỏ liên kết kho và tick).
    - `toFormRow`/`toInputRow` giữ nguyên các field ship (không thuộc `COMPARE_ROW_READ_ONLY_KEYS`).
  - `useCompareForm.ts`: `pickFromKho(column, detail: CompareInformationDetail | ShipCompareInformationItem)`; `SYNC_NEEDS_PERMISSION = 'Chỉ vai trò KSV, TPTD hoặc BGD mới được thay đổi đồng bộ.'`; `syncLockReason(column)` trả `SHIP_LOCKED_BY_SEARCH` khi cột gắn kho và nhóm là SHIP (giữ `SYNC_LOCKED_BY_KHO` cho loại khác); cập nhật comment ("KSV/TPTD/BGD").
  - `ShipCompareTab.vue`: props `{ profileId: number; reportId: number; group: ReportWorkspaceGroup }`. Dùng `useCompareForm(props)` và các thành phần chung. `data-testid`: `estimate-ratio` (mỗi cột), không thêm id khác.

- [ ] **Step 1: Đọc** `LT demo/comparative-information-ship.html` (toàn bộ), `LJ comparative-information-ship-search.js` l.134-155 và 393-471, `F compare/CarCompareTab.vue` + `CarCompareTab.spec.ts` + `detail-test-helpers.ts`, `compare-common.ts`, `useCompareForm.ts` + spec, và vỏ `ShipCompareTab.vue` do F01 tạo. Xác nhận F10 đã merge (có `ShipCompareSearchModal.vue`, `ship-compare.ts`).
- [ ] **Step 2: Test thất bại — hàm thuần**:
  - `ship-compare.spec.ts`: `validateShipCompareRow` — `shipBuildYear 1899` và `2101` ra `SHIP_YEAR_ERROR`; `1900` và `2100` và `null` ra không lỗi; `shipRemainingQualityPct -0.01` và `100.01` ra `SHIP_CLCL_ERROR`; `0`, `100`, `null` ra không lỗi; lỗi cả hai thì trả cả hai khóa.
  - `compare-common.spec.ts`: `fillFromKho_tàu_dùng_patch_tàu` (gọi với `ASSET_TYPE.SHIP` và item kho thì 14 ô được điền, `compareInformationId` gán, `syncToCompareInfo 0`); `emptyColumn_tàu_xóa_14_ô_không_đụng_trường_đất` (`landDimension: 'giữ'` còn nguyên, `areaInsideDetails` không bị đặt `[]`); `emptyColumn_tàu_không_để_mảng_chi_tiết_đất_rỗng` (cột tàu không có khóa `areaInsideDetails: []` mới); `toFormRow_giữ_trường_tàu_và_shipForSalePriceUsd`; `toInputRow_gửi_lại_shipForSalePriceUsd_nguyên_giá_trị`; hồi quy `fillFromKho_ô_tô_và_đất_không_đổi`.
  - `useCompareForm.spec.ts`: sửa kỳ vọng ở ca khóa vì không quyền thành `'Chỉ vai trò KSV, TPTD hoặc BGD mới được thay đổi đồng bộ.'`; thêm `tàu_pickFromKho_điền_cột_và_toast`, `tàu_clearColumn_xóa_14_ô_và_toast_bỏ_chọn`, `tàu_cột_gắn_kho_lý_do_khóa_từ_tìm_kiếm`, `ô_tô_cột_gắn_kho_lý_do_khóa_vẫn_từ_bản_đồ`.
- [ ] **Step 3: Test thất bại — tab** (`ShipCompareTab.spec.ts`; khuôn `CarCompareTab.spec.ts`: mock `compare`, `compare-informations` (giữ `importOriginal`, ghi đè `searchShipCompareInformations`), `useToast`, `useConfirm`; `group.assetType: 4`; `data()` trả `{ assetType: 4, rows: fourRows(), comparativeAnalysis: null, canSyncToKho: true, currencyCode: 'VND' }`; helper `cell(label, column)` và `text(label, column)`):
  - `hiển_thị_đủ_17_hàng_theo_thứ_tự`: nhãn cột đầu của `tbody tr` đúng `['Chủng loại','Nguồn tham khảo','Thông tin liên hệ','Đồng bộ lên cơ sở dữ liệu','Hình ảnh tài sản','Năm đóng','Nơi đóng','Mục đích sử dụng','Cấp phương tiện','Chất lượng còn lại (%)','Trọng tải toàn phần (tấn)','Kích thước (Lmax x Bmax)','Công suất (HP)','Giá rao bán (đồng)','Giá giao dịch (đồng)','Giá ước tính (đồng)','Tỷ lệ ước tính']`; tiêu đề cột `Tài sản thẩm định giá`, `Tài sản so sánh 1..3`.
  - `nhãn_giá_theo_tiền_tệ_usd`: `currencyCode 'USD'` thì `Giá rao bán (USD)`, `Giá giao dịch (USD)`, `Giá ước tính (USD)`; thiếu `currencyCode` thì `(đồng)`.
  - `cả_bốn_cột_nhập_tay`: mọi hàng nhập liệu (trừ đồng bộ, ảnh, tỷ lệ) ở cột 0 đều có `input`/`textarea`, không có phần tử chỉ đọc.
  - `nút_tìm_chỉ_ở_chủng_loại_cột_1_đến_3`: có đúng 3 nút `Tìm TSSS theo chủng loại`, không có ở cột 0 và không có ở hàng khác.
  - `mở_modal_với_từ_khóa_là_chủng_loại_của_cột`: cột 2 có `address 'Tàu hàng khô 3000T'`, bấm nút ở cột 2 thì modal mở, tiêu đề chứa `Tài sản so sánh 2`, ô từ khóa là `Tàu hàng khô 3000T` và `searchShipCompareInformations` được gọi.
  - `chọn_từ_modal_điền_cột_gán_id_khóa_đồng_bộ_và_hiện_bỏ_chọn`: kết quả có 1 item `USD`? Không: item `VND`; bấm `Chọn TSSS` thì 14 ô của cột 2 có giá trị của item (giá làm tròn), ô đồng bộ cột 2 `disabled` có tooltip `Đã chọn tài sản từ tìm kiếm; không dùng đồng bộ.`, hiện nút `Bỏ chọn tài sản so sánh`, `toast.success('Tải dữ liệu tài sản so sánh thành công')`; lưu thì `rows[2].compareInformationId === item.id` và `syncToCompareInfo === 0`.
  - `modal_chặn_chọn_tsss_khác_tiền_tệ`: nhóm `USD` hoặc `VND` thì truyền đúng `groupCurrencyCode` (kiểm bằng badge/nút `disabled` trong modal với item lệch).
  - `bỏ_chọn_xóa_14_ô_id_mở_khóa_và_toast`: cột đã gắn kho (`compareInformationId 7`), bấm `Bỏ chọn tài sản so sánh` thì 14 ô rỗng, ô đồng bộ không còn `disabled` (khi `canSyncToKho`), `toast.info('Đã bỏ chọn tài sản so sánh 1.')`, lưu thì `compareInformationId === null`.
  - `đồng_bộ_khóa_và_giải_thích_khi_không_có_quyền`: `canSyncToKho false` thì ba checkbox `disabled`, `title`/mô tả chứa `Chỉ vai trò KSV, TPTD hoặc BGD mới được thay đổi đồng bộ.`.
  - `không_quyền_nhưng_đã_đồng_bộ_vẫn_gửi_1`: cột 1 có `syncToCompareInformationId 9` và `canSyncToKho false` thì lưu vẫn gửi `syncToCompareInfo === 1`.
  - `có_quyền_tick_gửi_1_và_cột_tđg_gửi_null`: `rows[0].syncToCompareInfo === null`.
  - `ô_đồng_bộ_cột_tđg_hiện_dấu_gạch`: ô cột 0 của hàng đồng bộ và hàng ảnh chỉ có `—`.
  - `năm_đóng_ngoài_1900_2100_báo_lỗi_và_không_lưu`: nhập `1800` ở cột 1 thì submit hiện `Năm đóng phải từ 1900 đến 2100` dưới ô, `saveGroupCompare` không được gọi; sửa về `2012` thì lưu được.
  - `clcl_ngoài_0_100_báo_lỗi_và_không_lưu`: nhập `100,5` thì hiện `CLCL phải từ 0 đến 100`; `0` và `100` hợp lệ.
  - `lưu_gửi_đủ_trường_tàu_cho_bốn_cột_và_giữ_shipForSalePriceUsd`: dữ liệu nạp có `shipForSalePriceUsd 12345` ở cột 1, sửa vài ô rồi lưu thì `request.rows` có 4 phần tử, các field ship đúng giá trị, `shipForSalePriceUsd` vẫn `12345`.
  - `tỷ_lệ_ước_tính_hiển_thị`: `forSalePrice 200`, `estimatePrice 150` thì `[data-testid="estimate-ratio"]` ở cột đó là `75,00 %`; thiếu số hoặc rao bán `0` thì rỗng.
  - `ảnh_tsss_mở_modal_ảnh_của_đúng_cột`: bấm nút ảnh cột 3 thì `CompareImagesModal` nhận `row-id` của dòng `type 3`.
  - `lưu_xong_tải_lại_và_toast` (từ `useCompareForm`, kiểm `getGroupCompare` gọi 2 lần).
  - `không_có_tọa_độ_bản_đồ_phân_tích_so_sánh_và_nút_xuất_phiếu`: không có `Tọa độ`, `#comparative-analysis`, `Xuất phiếu TSSS`; có `Tải báo cáo` và `Lưu`.
  - `lỗi_tải_hiện_thông_báo_và_thử_lại`.
- [ ] **Step 4: Chạy, thấy FAIL**: `npx vitest run src/lib/report-edit/__tests__ src/composables/__tests__/useCompareForm.spec.ts src/components/appraisal-report/edit/compare/__tests__/ShipCompareTab.spec.ts`.
- [ ] **Step 5: Cài đặt** theo thứ tự: `ship-compare.ts` (thêm `validateShipCompareRow` và hằng), `compare-common.ts`, `useCompareForm.ts`, comment trong `compare.ts`, rồi `ShipCompareTab.vue` theo khuôn `CarCompareTab.vue`:
  - Bảng định nghĩa hàng (`TOP`, `SPECS`, `PRICES`) như `CarCompareTab`; `kind`: Năm đóng `integer`; CLCL `decimal` + `unit '%'`; Trọng tải và Công suất `decimal`; ba giá `decimal` với `decimals 0` và `unit` là `₫` (VND) hoặc `USD`, để cùng một thành phần cho hai tiền tệ (không dùng `kind 'money'` vì `CurrencyInput` cố định ký hiệu `₫`). Kiểm tra `DecimalInput` có nhận `unit` và `decimals` như `CompareField` đang truyền.
  - Nhãn giá: ``Giá rao bán (${currencyMoneyLabel(currencyCode)})`` với `currencyCode = data.currencyCode ?? 'VND'`.
  - `validate()` gọi `validateShipCompareRow` cho từng cột, gom lỗi theo `${column}:${key}`; `submit` chỉ gọi `form.save()` khi hợp lệ.
  - Nút tìm ở hàng Chủng loại cột 1–3 (`IconButton label="Tìm TSSS theo chủng loại"`), mở `ShipCompareSearchModal` với `:initial-keyword="rowAt(column).address ?? ''"`, `:group-currency-code`, `@select="pickShip"`; `pickShip` gọi `form.pickFromKho(column, item)` rồi xóa lỗi kiểm tra.
  - Hàng đồng bộ dùng `KhoSyncCell` với `form.isSyncLocked`, `form.syncLockReason`, `form.isLinkedToKho`, `form.setSync`, `form.clearColumn`.
  - Hàng ảnh dùng `CompareImagesModal`. Hàng "Tỷ lệ ước tính" dùng `estimateRatio`.
  - `FormActions sticky` chỉ có `DownloadReportButton` và nút `Lưu`.
  - Javadoc nêu nguồn legacy và 4 điểm "Khác hệ thống cũ".
- [ ] **Step 6: Chạy, thấy PASS** như Step 4, rồi `npm run lint && npm run type-check && npm run test:unit` (toàn bộ test Đất, Căn hộ, Ô tô phải vẫn xanh).
- [ ] **Step 7: Commit** `feat: tab thông tin so sánh tàu thủy`.

---

## Vấn đề cần controller quyết định

1. **→ Đã chốt: Quyết định #6.** Quyền tick đồng bộ kho (thêm BGD, mọi loại tài sản) nằm ở BE (`GroupCompareController`, cờ `canSyncToKho`), do B10 làm. F08 chỉ đổi tooltip và comment.
2. **→ Đã chốt: Quyết định #13.** F01 khai báo 5 field `shipInspection*` là optional trong `LegalAssetInput` và cập nhật mọi fixture test FE bị ảnh hưởng; F07 không phải sửa `LegalLandAssetForm`/`LegalCarAssetForm`.
3. **→ Đã chốt: Quyết định #7 và `10-fe-foundation.md`.** F01 cung cấp `searchShipCompareInformations(keyword, page, size = 20, signal?)` và `ShipCompareInformationItem` đúng DTO của index.
4. **→ Đã chốt: Quyết định #15 (ngoài phạm vi).** Sai khác cũ "VND" và "VND/m²" ở bảng điều chỉnh đất/căn hộ không sửa trong F09.
5. **→ Đã chốt: Quyết định #15 (ngoài phạm vi).** Không thêm "Xuất phiếu TSSS" cho tàu.
6. **Giá USD là số nguyên.** F08 dùng `DecimalInput` 0 chữ số lẻ cho ba giá để khớp legacy (`format-number`, làm tròn số nguyên khi chọn từ kho). Nếu nghiệp vụ cần giá USD có phần lẻ thì phải đổi cả cột DB và DTO (ngoài phạm vi).
7. **Số hàng của tab So sánh.** Khảo sát ghi "15 dòng" theo số thứ tự; thực tế mục 14 gồm ba dòng giá nên bảng có **17 hàng**. Plan và test F08 theo 17 hàng.
8. **→ Đã chốt: Quyết định #13.** F01 (đợt F1) sửa `panel-fixtures.ts`, `results/__tests__/fixtures.ts`, `compare-fixtures.ts` trước; F09/F11 chỉ thêm helper mới.
