# 03 — Pháp lý, So sánh, Bảng điều chỉnh, Kho TSSS cho tàu — Backend (B04, B05, B10)

> Đọc `00-index.md` (hợp đồng, quy ước tên, Global Constraints, "Khác hệ thống cũ", Review Focus) và các file trong `surveys/` trước. Subagent: `model: "sonnet"`, mỗi task qua spec review rồi code review.
>
> Viết tắt: **L** = `td-project-legacy` @ `82193fe5` (gốc `src/main/java/com/jvb/thanhdo/`), **N** = `backend/src/main/java/com/thanhdo/tdprojectbe/`, **NT** = `backend/src/test/java/com/thanhdo/tdprojectbe/`, **T/** = `src/main/resources/templates/` của L. Mọi dòng của L đọc bằng `git -C td-project-legacy show 82193fe5:<path>`.
>
> Đối chiếu code mới đo trên `origin/develop` của backend. Số dòng có thể lệch: **grep lại trước khi sửa**.

## Phân chia file giữa ba task (đợt B2, tập file RỜI nhau)

| Task | File main sở hữu | File test sở hữu |
|---|---|---|
| **B04** Pháp lý | `N service/ReportLegalService`, `N model/dto/LegalAssetInput`, `N model/dto/LegalAssetView`, `N model/entity/AppraisalReportAsset` (chỉ `mergeLegalFieldsFrom`; thân `applyShipLegal` do B01 viết, B04 không sửa) | `NT service/ReportLegalServiceTest`, `NT controller/ReportLegalControllerTest`, `NT repository/AppraisalReportAssetRepositoryTest`, mới `NT model/entity/AppraisalReportAssetShipLegalTest` (chỉ gộp pháp lý; test `applyShipLegal` thuộc B01) |
| **B05** So sánh + bảng điều chỉnh | `N service/GroupCompareService`, `N service/CompareFactorSync`, `N service/AdjustmentPanelService`, `N service/RegulatoryConstraints`, `N model/dto/GroupCompareRowInput`, `GroupCompareRow`, `GroupCompareData`, `AdjustmentPanelData`, `N model/entity/AppraisalGroupCompare` (chỉ nhánh `SHIP` trong `applyForm`, kèm ghi `address`; thân `applyShipForm` do B01 viết) | `NT service/GroupCompareServiceTest`, `CompareFactorSyncTest`, `AdjustmentPanelServiceTest`, `NT controller/GroupCompareControllerTest`, `AdjustmentPanelControllerTest`, `NT repository/GroupCompareSaveRepositoryTest`, mới `NT model/entity/AppraisalGroupCompareShipFormTest` (chỉ kiểm nối `applyForm` đến `applyShipForm`; test thân hàm thuộc B01) |
| **B10** Kho TSSS | `N service/CompareInformationSearchService`, `N controller/CompareInformationController`, `N repository/CompareInformationRepository`, `N service/CompareInformationSync`, `N model/entity/CompareInformation` (`copyFrom`, `useCurrency`), `N controller/GroupCompareController` (cờ `canSyncToKho`), mới `N model/dto/ShipCompareInformationItem` | `NT repository/CompareInformationSearchRepositoryTest`, `NT service/CompareInformationSearchServiceTest`, `NT service/CompareInformationSyncTest`, `NT controller/CompareInformationControllerTest`, mới `NT model/entity/CompareInformationShipTest`, mới `NT controller/GroupCompareControllerKhoPermissionTest` |

Các quyết định chọn chủ sở hữu duy nhất (đã xác minh trên code thật):

1. **Chặn gắn TSSS khác tiền tệ nằm trong `GroupCompareService.save` (B05), không nằm trong `CompareInformationSync` (B10).** Lý do: `save` chạy vòng lặp `writeRow` rồi `khoSync.apply` cho từng cột; `CompareInformationSync.select` gọi `images.cloneKhoImagesToRow` (ghi S3) ngay trong vòng lặp, nên kiểm tra phải đứng **trước vòng lặp** và cần `group.getCurrencyCode()`, mà `CompareInformationSync.apply(row, assetType, input, callerId)` không có. B05 thêm `CompareInformationRepository` vào constructor của `GroupCompareService`. B10 không đụng việc này (chỉ đọc mục này để biết nó có tồn tại).
2. **Tiền tệ khi đẩy dòng lên kho do `CompareInformationSync.push` (B10) tự lấy từ `row.getAppraisalAssetGroup()`** (quan hệ LAZY chỉ đọc đã có trên `AppraisalGroupCompare`, giống `agc.getAppraisalAssetGroup()` của L). Nhờ vậy **không đổi chữ ký** `CompareInformationSync.apply`, `GroupCompareService` (B05) không phải sửa chỗ gọi, hai task không chạm nhau.
3. **Quyền tick đồng bộ kho nằm ở BE**, trong `GroupCompareController.get` (`canSyncToKho` chỉ xét authority `TPTD`; KSV có authority TPTD). Chủ sở hữu là **B10**. B05 không sửa controller này (B05 chỉ sửa `GroupCompareData` và `GroupCompareService.get`, chữ ký `service.get(profileId, reportId, groupId, canSyncToKho)` giữ nguyên). FE (F08) không phải đổi vì chỉ đọc cờ `canSyncToKho`.
4. **`CommonConstants.resolveRegulatoryConstraintsPriceUnit` KHÔNG sửa trong ba task này.** Bảng điều chỉnh trên web dùng `CurrencyCodes.regulatoryPriceUnit` (B01). `CommonConstants.resolveRegulatoryConstraintsPriceUnit` chỉ phục vụ Word (`LandValuationNotes`, `WordExportCommon`), thuộc nhóm task Word (B08/B09).
5. **Quyết định #2 của controller: `applyShipLegal(ShipLegalFields)` và `applyShipForm(ShipCompareFields, long, long)` do B01 viết thân hàm** (record trung gian trong `model/dto`). B04 và B05 chỉ **dựng record từ DTO của mình** (một dòng `new ShipLegalFields(...)` / `new ShipCompareFields(...)`) và gọi; **không** tạo hay sửa thân hai hàm này, không đổi chữ ký.

---

# B04 — Pháp lý tàu

## Hành vi hệ thống cũ

Nguồn đọc: `L service/impl/AppraisalReportAssetServiceImpl.java:72-150` (`updateAsset`), `L entity/dto/request/UpdateAppraisalReportAssetRequest.java:25-52`, `L entity/AppraisalReportAsset.java:60-90`, `L service/impl/AppraisalReportServiceImpl.java:920-945` (merge pháp lý khi sửa thông tin cơ bản), `T/demo/legal-document-ship.html` (tab Pháp lý tàu, xem `surveys/legacy-ui-word.md` mục 4).

- Tab Pháp lý tàu có 4 mục: (1) Giấy chứng nhận đăng ký phương tiện thủy nội địa gồm `certificateNumber`, `cnIssuedPlace`, `gcnNoIssuedDate`, `forWhom`; (2) Giấy chứng nhận an toàn kỹ thuật và bảo vệ môi trường gồm `shipInspectionCertificateNumber`, `shipInspectionIssuedPlace`, `shipInspectionIssuedDate`, `shipInspectionExpiryDate`, `shipInspectionForWhom`; (3) textarea `legalDocuments`; (4) file đính kèm. Đây là toàn bộ trường mà form tàu gửi (`UpdateAppraisalReportAssetRequest.java:42-48` thêm đúng 5 trường `shipInspection*`).
- `updateAsset` (`AppraisalReportAssetServiceImpl.java:88-116`) ghi đè **mọi** cột pháp lý của tài sản từ request, kể cả cột của loại khác (owner, gcnNo, gcnNoExpiryDate, car_*, other, otherInfo) về null nếu form không gửi.
- Dòng 92-98: khi nhóm là **CAR hoặc SHIP** thì **không** ghi đè `asset` (tên/địa chỉ tài sản do màn thông tin cơ bản quản lý); các loại khác ghi `asset`.
- Cột DB (`L entity/AppraisalReportAsset.java:72-85`): `ship_inspection_certificate_number` 500, `ship_inspection_issued_place` 500, `ship_inspection_issued_date` DATE, `ship_inspection_expiry_date` DATE, `ship_inspection_for_whom` **1000**. `for_whom` cũng 1000.
- Validate: server legacy **không** kiểm độ dài (chỉ DB cắt/ lỗi). UI kiểm 6 trường text (`certificateNumber`, `cnIssuedPlace`, `forWhom`, `shipInspectionCertificateNumber`, `shipInspectionIssuedPlace`, `shipInspectionForWhom`) **tối đa 500 ký tự** ở JS (`surveys/legacy-ui-word.md` mục 4).
- Merge khi sửa thông tin cơ bản (`AppraisalReportServiceImpl.java:928-941`): với 5 trường tàu, "target rỗng thì lấy source" (chuỗi dùng `isEmpty`, ngày dùng `== null`).

## Khác hệ thống cũ (liên quan task này, ghi vào Javadoc)

- Mục "Khác hệ thống cũ" số 3 của `00-index.md`: lỗi validate trả 400 qua `BusinessException`/bean validation.
- Đã áp dụng sẵn cho Đất/Căn hộ/Ô tô ở `ReportLegalService`: mỗi loại **chỉ ghi nhóm cột của mình**. Tàu tuân theo: chỉ ghi `certificateNumber`, `cnIssuedPlace`, `gcnNoIssuedDate`, `forWhom`, 5 cột `ship_inspection_*`, `legalDocuments`. **Không** ghi `owner`, `asset`, `gcnNo`, `gcnNoExpiryDate`, `certificateNumberType`, `other`, `otherInfo`, `car_*` (cũ ghi đè các cột này thành null).
- **Quyết định độ dài (BE chọn theo cột DB):** `shipInspectionCertificateNumber` ≤ 500, `shipInspectionIssuedPlace` ≤ 500, `shipInspectionForWhom` ≤ **1000** (khớp cột `ship_inspection_for_whom` VARCHAR(1000) và `forWhom` hiện có ≤ 1000). Giới hạn 500 của legacy chỉ là kiểm tra phía trình duyệt; F07 vẫn được giữ 500 ở form để khớp trải nghiệm cũ, BE không siết hơn cột vì legacy server chưa từng từ chối chuỗi dài 501..1000 cho hai trường "cho ai".
- Không ghi đè `asset` cho CAR/SHIP: code mới **đã** đúng (`applyCarLegal` không ghi `asset`); `applyShipLegal` cũng không ghi, và có test khóa hành vi này (khác với `applyLandLegal` ghi `asset`).

### Task B04: Pháp lý tàu (DTO, entity, ReportLegalService)

**Files:**
- Modify `N model/dto/LegalAssetInput.java`: thêm 5 component **ngay sau `carInspectionExpiryDate`, trước `forWhom`** (cùng thứ tự với L):
  `@Size(max = 500, message = "Số giấy chứng nhận an toàn kỹ thuật tối đa 500 ký tự") String shipInspectionCertificateNumber`, `@Size(max = 500, message = "Nơi cấp giấy chứng nhận an toàn kỹ thuật tối đa 500 ký tự") String shipInspectionIssuedPlace`, `LocalDate shipInspectionIssuedDate`, `LocalDate shipInspectionExpiryDate`, `@Size(max = 1000, message = "Cấp cho (giấy an toàn kỹ thuật) tối đa 1000 ký tự") String shipInspectionForWhom`. Cập nhật Javadoc ("Tàu dùng nhóm giấy đăng ký + giấy an toàn kỹ thuật").
- Modify `N model/dto/LegalAssetView.java`: thêm 5 component cùng tên, cùng vị trí.
- Modify `N model/entity/AppraisalReportAsset.java`: **chỉ** thêm 5 trường tàu vào `mergeLegalFieldsFrom` (Quyết định #10: B04 làm, B01 không làm). Cột `ship_inspection_*`, getter và `applyShipLegal(ShipLegalFields)` do B01 cung cấp (Step 1 kiểm tra); B04 không sửa chúng.
- Modify `N service/ReportLegalService.java`: `save` rẽ theo loại tài sản bằng `switch` thay cho `if CAR / else`; `view(...)` map thêm 5 trường tàu; cập nhật Javadoc lớp.
- Modify test: `NT service/ReportLegalServiceTest.java`, `NT controller/ReportLegalControllerTest.java`, `NT repository/AppraisalReportAssetRepositoryTest.java`.
- Create test: `NT model/entity/AppraisalReportAssetShipLegalTest.java`.

**Interfaces:**
- Yêu cầu từ B01: `AppraisalReportAsset` đã có 5 cột `shipInspectionCertificateNumber`, `shipInspectionIssuedPlace`, `shipInspectionIssuedDate`, `shipInspectionExpiryDate`, `shipInspectionForWhom` kèm getter (`@Getter` cấp lớp); `AppraisalAssetType.SHIP == 4`.
- B01 cung cấp `public void applyShipLegal(ShipLegalFields f)` — ghi đúng 10 cột: `certificateNumber`, `cnIssuedPlace`, `gcnNoIssuedDate`, `forWhom`, 5 cột `shipInspection*`, `legalDocuments`; giá trị null ghi thành null (giống `applyCarLegal`, ghi đè nguyên nhóm cột). B04 gọi: `asset.applyShipLegal(new ShipLegalFields(input.certificateNumber(), input.cnIssuedPlace(), input.gcnNoIssuedDate(), input.forWhom(), input.shipInspectionCertificateNumber(), input.shipInspectionIssuedPlace(), input.shipInspectionIssuedDate(), input.shipInspectionExpiryDate(), input.shipInspectionForWhom(), input.legalDocuments()))`.
- `LegalAssetInput` / `LegalAssetView` có thêm 5 component (vị trí nêu trên). JSON API: `shipInspectionCertificateNumber`, `shipInspectionIssuedPlace`, `shipInspectionIssuedDate`, `shipInspectionExpiryDate` (ISO `YYYY-MM-DD`), `shipInspectionForWhom`.
- `ReportLegalService.save`: `switch (group.getAssetType())`: `CAR` → `applyCarLegal`; `SHIP` → `applyShipLegal`; còn lại → `applyLandLegal(input, cleanOther(...))` (Đất/Căn hộ như cũ). Không còn `else` ngầm hiểu cho CAR.
- (B04 làm) `mergeLegalFieldsFrom` điền thêm 5 trường tàu vào ô trống của đích (chuỗi trống/khoảng trắng, ngày null), không ghi đè ô đã có. **B07 (vòng đời) phụ thuộc phần này; không làm lại ở B07.**

- [ ] **Step 1: Kiểm tra điều kiện từ B01**
  ```bash
  git -C backend grep -n "ship_inspection\|shipInspection" -- src/main/java/com/thanhdo/tdprojectbe/model/entity/AppraisalReportAsset.java
  git -C backend grep -n "SHIP" -- src/main/java/com/thanhdo/tdprojectbe/common/constant/AppraisalAssetType.java
  git -C backend grep -n "applyShipLegal\|record ShipLegalFields" -- src
  ```
  Kỳ vọng: 5 cột `shipInspection*` có; `SHIP = 4` có; `applyShipLegal(ShipLegalFields)` và record `ShipLegalFields` (B01) có. Thiếu thì **dừng, báo controller** (B01 chưa xong); không tự viết thay. Kiểm tra DB (bảng dùng chung với legacy): `DESCRIBE appraisal_report_assets;` phải có 5 cột `ship_inspection_*`; thiếu thì dừng, báo người dùng.

- [ ] **Step 2: Viết test thất bại — entity (unit, không DB)** trong `NT model/entity/AppraisalReportAssetShipLegalTest.java`. **Phạm vi: chỉ `mergeLegalFieldsFrom` (3 test `gộp_*`).** Các test `applyShipLegal_*` và `applyLandLegal/applyCarLegal_không_đụng_cột_tàu` bên dưới đã thuộc B01 (`AppraisalReportAssetShipTest`): không viết lại ở file này, chỉ giữ ở tầng service (dùng `AppraisalReportAsset.create(...)` và `ReflectionTestUtils` để đặt giá trị có sẵn như `ReportLegalServiceTest`):
  - `applyShipLegal_ghi_mười_cột_của_tàu`: input đủ 4 trường GCN đăng ký + 5 trường giấy an toàn + `legalDocuments`; assert 10 getter đúng giá trị.
  - `applyShipLegal_null_ghi_đè_thành_null`: tài sản đã có giá trị, input toàn null (trừ không cột nào ngoài nhóm) → 10 cột về null.
  - `applyShipLegal_không_đụng_owner_asset_gcnNo_ngày_hết_hạn_và_cột_xe`: đặt sẵn `owner`, `asset`, `gcnNo`, `gcnNoExpiryDate`, `certificateNumberType`, `other`, `otherInfo`, `carInspectionCertificateNumber`, `carInspectionIssuedPlace`, `carInspectionIssuedDate`, `carInspectionExpiryDate`; sau `applyShipLegal` các cột này **không đổi**.
  - `applyLandLegal_không_đụng_cột_tàu` và `applyCarLegal_không_đụng_cột_tàu`: đặt sẵn 5 cột tàu, gọi hai phương thức cũ với input có giá trị khác, 5 cột tàu giữ nguyên.
  - `gộp_điền_năm_trường_tàu_vào_ô_trống`: đích trống (chuỗi `"  "`, ngày null) và nguồn có giá trị → đích nhận giá trị.
  - `gộp_không_ghi_đè_trường_tàu_đã_có`: đích đã có `shipInspectionCertificateNumber` và `shipInspectionIssuedDate` → giữ nguyên dù nguồn khác.
  - `gộp_nguồn_null_không_đổi_gì`.

- [ ] **Step 3: Viết test thất bại — service** trong `NT service/ReportLegalServiceTest.java`. Sửa helper `input(...)` (dòng ~90) cho khớp chữ ký mới của `LegalAssetInput` (thêm 5 đối số sau `carInspectionExpiryDate`), thêm helper `shipInput(...)`. Thêm:
  - `lưu_tàu_chỉ_ghi_nhóm_cột_tàu`: `givenGroup(AppraisalAssetType.SHIP)`; assert 10 cột đúng giá trị, `assets.save(asset)` được gọi.
  - `lưu_tàu_không_đụng_chủ_và_địa_chỉ`: tài sản có `asset = "Địa chỉ ban đầu"` và `owner = "Chủ cũ"`; input gửi `owner="X"`, `asset="Y"` → sau lưu vẫn `"Địa chỉ ban đầu"` và `"Chủ cũ"` (khóa hành vi "không ghi đè asset" của `AppraisalReportAssetServiceImpl.java:92-98`).
  - `lưu_tàu_không_đụng_cột_xe_và_các_cột_đất`.
  - `tàu_không_rơi_vào_nhánh_đất`: input có `otherInfo` không rỗng và `certificateNumberType="X"` → `asset.getOtherInfo()` và `getCertificateNumberType()` **không** bị ghi (chứng minh SHIP không đi nhánh `applyLandLegal`).
  - `lưu_đất_không_đụng_cột_tàu` và `lưu_ô_tô_không_đụng_cột_tàu` (chỉnh test hiện có `lưu_đất_không_đụng_cột_ô_tô`, `lưu_ô_tô_không_đụng_chủ_và_địa_chỉ` nếu cần cho khớp constructor, thêm assert 5 cột tàu giữ nguyên).
  - `get_tàu_trả_năm_trường_giấy_an_toàn_và_loại_tài_sản_4`: `LegalTabData.assetType()==4`, view có 5 trường tàu đúng giá trị.
  - `lưu_tàu_trả_view_có_năm_trường_tàu` (giá trị trả về của `save`).

- [ ] **Step 4: Viết test thất bại — controller** trong `NT controller/ReportLegalControllerTest.java`. Sửa `view()` (dòng ~53) cho khớp constructor `LegalAssetView` mới. Thêm:
  - `get_trả_năm_trường_giấy_an_toàn_với_ngày_iso`: `$.data.assets[0].shipInspectionIssuedDate` = `"2025-03-01"`, `shipInspectionExpiryDate` = `"2027-03-01"`, `shipInspectionForWhom`.
  - `put_nhận_năm_trường_tàu_trong_phần_data`: body JSON có 5 trường tàu → `ArgumentCaptor<LegalAssetInput>` có đúng giá trị và `LocalDate`.
  - `put_giấy_an_toàn_quá_dài_trả_400`: `shipInspectionCertificateNumber` 501 ký tự → 400; đúng 500 → 200; `shipInspectionIssuedPlace` 501 → 400; `shipInspectionForWhom` 1001 → 400, 1000 → 200.
  - `put_ngày_sai_định_dạng_trả_400` (vd `"01/03/2025"`).

- [ ] **Step 5: Viết test thất bại — repository** trong `NT repository/AppraisalReportAssetRepositoryTest.java` (`@MysqlDataJpaTest`). Sửa hai chỗ `new LegalAssetInput(...)` (dòng ~101 và ~133) thêm 5 đối số `null`/giá trị. Thêm `ghi_pháp_lý_tàu_rồi_đọc_lại_đủ_chín_cột`: `applyShipLegal` trên tài sản nhóm tàu, `em.flush(); em.clear();` rồi truy vấn native các cột `certificate_number, cn_issued_place, gcn_no_issued_date, for_whom, ship_inspection_certificate_number, ship_inspection_issued_place, ship_inspection_issued_date, ship_inspection_expiry_date, ship_inspection_for_whom, legal_documents`; kỳ vọng đúng giá trị, `ship_inspection_for_whom` chứa được chuỗi 1000 ký tự, ngày lưu đúng `yyyy-MM-dd`; các cột `owner`, `asset` giữ giá trị chèn ban đầu.

- [ ] **Step 6: Chạy xác nhận FAIL**
  ```bash
  cd backend && ./mvnw -q test -Dtest='AppraisalReportAssetShipLegalTest,ReportLegalServiceTest,ReportLegalControllerTest'
  ```
  Kỳ vọng: lỗi biên dịch/FAIL vì chưa có component và `applyShipLegal`.

- [ ] **Step 7: Cài đặt**
  1. `LegalAssetInput`, `LegalAssetView`: thêm 5 component (Step đầu mục Files). Javadoc tiếng Việt, nhắc tàu dùng nhóm trường nào.
  2. `AppraisalReportAsset.mergeLegalFieldsFrom`: bổ sung bằng `fillText`/`fillValue` cho 5 trường (Javadoc của `applyShipLegal` do B01 viết) (như `AppraisalReportServiceImpl.java:928-941`).
  3. `ReportLegalService.save`:
     ```java
     switch (group.getAssetType()) {
         case AppraisalAssetType.CAR -> asset.applyCarLegal(input);
         case AppraisalAssetType.SHIP -> asset.applyShipLegal(new ShipLegalFields(/* dựng từ input, xem Interfaces */));
         default -> asset.applyLandLegal(input, cleanOther(input.otherInfo()));
     }
     ```
     `view(...)`: thêm 5 getter vào `new LegalAssetView(...)` đúng thứ tự mới. Sửa Javadoc lớp: "Đất/Căn hộ chỉ ghi nhóm cột đất, Ô tô chỉ ghi nhóm cột xe, Tàu chỉ ghi nhóm cột tàu (giấy đăng ký, giấy an toàn kỹ thuật, giấy tờ pháp lý)".
  4. Không đổi `ReportLegalController` (nhận `LegalAssetInput` qua part `data`, bean validation lo độ dài).

- [ ] **Step 8: Chạy xác nhận PASS**
  ```bash
  cd backend && ./mvnw -q test -Dtest='AppraisalReportAssetShipLegalTest,ReportLegalServiceTest,ReportLegalControllerTest,AppraisalReportAssetRepositoryTest'
  ```
  Test repository cần MySQL (xem `backend/.env.example`) và schema đã có 5 cột ship. Chạy thêm bộ có thể bị ảnh hưởng bởi constructor record: `./mvnw -q test -Dtest='*Legal*,AppraisalReportBasicEdit*'`. Kỳ vọng: tất cả PASS.

- [ ] **Step 9: Commit**
  ```bash
  git -C backend add src/main/java/com/thanhdo/tdprojectbe/model/dto/LegalAssetInput.java src/main/java/com/thanhdo/tdprojectbe/model/dto/LegalAssetView.java src/main/java/com/thanhdo/tdprojectbe/model/entity/AppraisalReportAsset.java src/main/java/com/thanhdo/tdprojectbe/service/ReportLegalService.java src/test/java/com/thanhdo/tdprojectbe/model/entity/AppraisalReportAssetShipLegalTest.java src/test/java/com/thanhdo/tdprojectbe/service/ReportLegalServiceTest.java src/test/java/com/thanhdo/tdprojectbe/controller/ReportLegalControllerTest.java src/test/java/com/thanhdo/tdprojectbe/repository/AppraisalReportAssetRepositoryTest.java
  git -C backend commit -m "feat: pháp lý tàu thủy (giấy đăng ký, giấy an toàn kỹ thuật) cho tab Pháp lý"
  ```

---

# B05 — So sánh tàu, đồng bộ sang bảng điều chỉnh, bảng điều chỉnh tàu

## Hành vi hệ thống cũ

Nguồn đọc (tất cả `L service/AppraisalGroupCompareService.java` trừ khi ghi khác): `createGroupCompares` 81-110; `update` 112-390 (kiểm tra liên kết kho 130-132; giá 150-186; trường tàu 253-275; liên kết/đồng bộ kho 326-366; `comparativeAnalysis` 374-383; tuyến đi bộ 389-428); đồng bộ yếu tố `formatDecimalForControlCompareSync` 862-867, `formatShip*ForControlCompareSync` 869-895, `computeSyncedValuesForLabelName` 904-1110 (CLCL 1026-1040, Năm đóng 1043-1050, Nơi đóng 1052-1059, Trọng tải 1061-1068, Công suất 1070-1077, Cấp phương tiện 1079-1086); `validateCompareInformationLinks` 1173-1194; `validateCompareInformationCurrency` 1199-1209. Panel: `T/demo/appraisal-panel.html:4` (`currencyUnit`), `:78`, `:270-327`, `:352,389,437` (ô đơn vị), `:461` (`regPriceUnit`); `L common/CommonConstants.java:446-497` (`resolveCurrencyCode`, `currencyUnitLabel`, `resolveRegulatoryConstraintsPriceUnit`). Tab so sánh: `T/demo/comparative-information-ship.html` và `surveys/legacy-ui-word.md` mục 5, 6.

Lưu dòng so sánh (`update`), phần dành cho **SHIP**:
- Chuỗi `address` (rỗng thành null), `source`, `contact` ghi như loại khác; mọi cột (type 0..3) nhập tay được; cột tài sản thẩm định (type 0) cũng ghi `address` (Chủng loại).
- `forSalePrice`, `tradePrice`, `estimatePrice` null thành 0 (dòng 151-153).
- `marketPrice`: **tàu luôn 0** (dòng 159-160); loại khác `tp ≠ 0 ? tp : (ep ?? 0)`.
- `apartmentOwnershipRightPrice = 0`; `landUseRightPrice = (tp != null && tp != 0) ? tp : (ep ?? 0)` (dòng 183-186), tức "Giá bán" của bảng điều chỉnh (ưu tiên giá giao dịch). CAR là `tp + ep`.
- 9 trường tàu (dòng 253-275): `shipBuildYear` (Integer), `shipBuildPlace`/`shipUsagePurpose`/`shipClass`/`shipDimension` trim rỗng thành null, `shipRemainingQualityPct` ngoài [0,100] thì ném `IllegalArgumentException("Chất lượng còn lại tàu phải từ 0 đến 100")` (dòng 263-268), `shipDeadweight`, `shipMainEnginePowerHp`, `shipForSalePriceUsd` ghi như request.
- Không có `comparativeAnalysis` cho tàu ở form (UI không có), không có tuyến đi bộ (dòng 389-428 chỉ LAND/APARTMENT), không có diện tích/chi tiết đất.
- Kiểm tra tiền tệ TSSS (dòng 1173-1209): chạy **trước vòng cập nhật** (dòng 130-132), chỉ xét cột type 1..3 có `compareInformationId` **khác** liên kết hiện tại; nếu `resolveCurrencyCode(group.assetType, group.currencyCode)` khác `resolveCurrencyCode(ci.assetType, ci.currencyCode)` thì ném IAE "Tài sản so sánh trong kho dùng đơn vị {X}, khác đơn vị tiền tệ của nhóm ({Y})"; `ci == null` thì bỏ qua.
- Đồng bộ sang yếu tố điều chỉnh (dòng 904-1110), ghi `cValue`, `cValueCp1..3` từ cột type 0..3, null thành `""`: Năm đóng (`String.valueOf(shipBuildYear)`), Nơi đóng (trim), Trọng tải toàn phần / Công suất (CV) / Chất lượng còn lại (`stripTrailingZeros().toPlainString()` rồi `formatNumberStringToEuropeanStyleFixed2`: luôn 2 chữ số lẻ, 0 thành `0,00`), Cấp phương tiện (trim). "Chất lượng còn lại" rẽ **CAR → `carRemainingQualityPct`, SHIP → `shipRemainingQualityPct`, loại khác không đồng bộ** (dòng 1028-1040). "Ngoại quan tổng thể" và "Khả năng vận hành" không đồng bộ (rơi vào `default`).
- Bảng điều chỉnh (`appraisal-panel.html`): `currencyUnit = currencyUnitLabel(resolveCurrencyCode(group.assetType, group.currencyCode))`; tàu ẩn dòng "A. Giá thị trường", dòng đầu là A "Giá bán" lấy `landUseRightPrice`; đơn vị của các dòng dùng `currencyUnit`; câu "Nguyên tắc khống chế" dùng `regPriceUnit = resolveRegulatoryConstraintsPriceUnit(assetType, currencyCode)` (tàu ra "đồng" hoặc "USD"). Công thức điều chỉnh không đổi (do FE tính).

## Khác hệ thống cũ (liên quan task này, ghi vào Javadoc)

- Số 3: lỗi trả 400 qua `BusinessException` (`ErrorCode.VALIDATION_ERROR`): CLCL tàu ngoài 0..100, TSSS khác tiền tệ. Cũ: `IllegalArgumentException`.
- Đã có sẵn ở code mới (giữ nguyên, chỉ mở rộng): `applyForm` chỉ ghi các cột mà form của loại tài sản đó có (cũ ghi mọi cột); `market_price` rỗng ghi 0 thay vì `0111L`; yếu tố không áp dụng cho loại tài sản thì không đồng bộ (cũ ghi "").
- **`shipForSalePriceUsd`**: ghi khi request gửi giá trị khác null, **giữ nguyên giá trị DB khi request không gửi** (hợp đồng `00-index.md`: có trong DTO, không hiện trên UI, được giữ nguyên khi lưu; L ghi đè bằng null nếu form không gửi).
- Dòng GET trả `currencyCode` **đã resolve** của nhóm (cũ: fragment nạp `groupCurrencyCode`).
- `AdjustmentPanelData.currencyUnit` mới; `regulatoryPriceUnit` của tàu không có "/m²" hay "/tàu".

### Task B05a: DTO, entity và GroupCompareService cho tàu

**Files:**
- Modify `N model/dto/GroupCompareRowInput.java`: thêm 9 component sau `carNote`, trước `areaInsideDetails`:
  `@Min(value = 1900, message = "Năm đóng từ 1900 đến 2100") @Max(value = 2100, message = "Năm đóng từ 1900 đến 2100") Integer shipBuildYear`, `@Size(max = 1000, message = "Nơi đóng tối đa 1.000 ký tự") String shipBuildPlace`, `@Size(max = 500, message = "Mục đích sử dụng tàu tối đa 500 ký tự") String shipUsagePurpose`, `@Size(max = 500, message = "Cấp phương tiện tối đa 500 ký tự") String shipClass`, `BigDecimal shipRemainingQualityPct` (**không** gắn `@DecimalMin/@DecimalMax`: chỉ kiểm khi nhóm là tàu, xem ghi chú bên dưới), `@PositiveOrZero(message = "Trọng tải toàn phần không được âm") @Digits(integer = 10, fraction = 2, message = "Trọng tải toàn phần tối đa 10 chữ số nguyên và 2 chữ số thập phân") BigDecimal shipDeadweight`, `@Size(max = 255, message = "Kích thước tối đa 255 ký tự") String shipDimension`, `@PositiveOrZero(message = "Công suất không được âm") @Digits(integer = 8, fraction = 2, message = "Công suất tối đa 8 chữ số nguyên và 2 chữ số thập phân") BigDecimal shipMainEnginePowerHp`, `@PositiveOrZero(message = "Giá rao bán (USD) không được âm") @Digits(integer = 13, fraction = 2, message = "Giá rao bán (USD) tối đa 13 chữ số nguyên và 2 chữ số thập phân") BigDecimal shipForSalePriceUsd`. Cập nhật Javadoc lớp.
  - Ghi chú: CLCL không gắn annotation vì một nhóm đất/ô tô có thể gửi kèm `shipRemainingQualityPct` thừa (bị bỏ qua); bean validation sẽ từ chối nhầm. Kiểm tra 0..100 làm trong `GroupCompareService` và chỉ cho nhóm tàu.
- Modify `N model/dto/GroupCompareRow.java`: thêm 9 component cùng tên/kiểu `Integer shipBuildYear, String shipBuildPlace, String shipUsagePurpose, String shipClass, BigDecimal shipRemainingQualityPct, BigDecimal shipDeadweight, String shipDimension, BigDecimal shipMainEnginePowerHp, BigDecimal shipForSalePriceUsd` (sau `carNote`) và map trong `from(...)`.
- Modify `N model/dto/GroupCompareData.java`: thêm component cuối `String currencyCode` (đã resolve: `"VND"` hoặc `"USD"`), Javadoc `@param`.
- Modify `N model/entity/AppraisalGroupCompare.java`: **chỉ** `applyForm` rẽ `SHIP`: ghi `address` (`trimToNull(in.address())`, cả 4 cột kể cả type 0; B01 `applyShipForm` không ghi `address`) rồi gọi `applyShipForm(<ShipCompareFields dựng từ in>, trade, estimate)`; `applyShipForm` do B01 viết, B05 không sửa thân; cập nhật bảng "Cột được ghi theo loại tài sản" trong Javadoc (thêm hàng tàu). `marketPrice` của tàu = 0.
- Modify `N service/GroupCompareService.java` (xem Interfaces).
- Modify test: `NT service/GroupCompareServiceTest.java`, `NT controller/GroupCompareControllerTest.java`, `NT repository/GroupCompareSaveRepositoryTest.java`.
- Create test: `NT model/entity/AppraisalGroupCompareShipFormTest.java`.

**Interfaces:**
- Yêu cầu từ B01: 9 cột tàu + getter trên `AppraisalGroupCompare` (`getShipBuildYear()`…`getShipForSalePriceUsd()`); `AppraisalAssetGroup.getCurrencyCode()`; `CurrencyCodes.resolveCurrencyCode(int assetType, String code)`; `CompareInformation.getCurrencyCode()`; `AppraisalAssetType.SHIP`.
- `AppraisalGroupCompare.applyForm(in, assetType, insideDetails, outsideDetails, plants)`: `switch (assetType)`: `APARTMENT` → `applyApartmentForm`; `CAR` → `applyCarForm`; `SHIP` → `address = trimToNull(in.address())` rồi `applyShipForm(new ShipCompareFields(in.shipBuildYear(), in.shipBuildPlace(), in.shipUsagePurpose(), in.shipClass(), in.shipRemainingQualityPct(), in.shipDeadweight(), in.shipDimension(), in.shipMainEnginePowerHp(), <forSalePriceUsd, xem Vấn đề #9>), trade, estimate)`; còn lại → `applyLandForm`. Dòng chung đầu hàm (`source`, `contact`, 3 giá, `marketPrice = trade != 0 ? trade : estimate`) giữ nguyên; `applyShipForm` ghi đè `marketPrice = 0`.
- (B01 viết thân) `applyShipForm(ShipCompareFields f, long trade, long estimate)` ghi **đúng** các cột sau (mọi cột khác giữ nguyên; `address` do nhánh `SHIP` của `applyForm` ghi, không phải `applyShipForm`):
  `shipBuildYear`, `shipBuildPlace`/`shipUsagePurpose`/`shipClass`/`shipDimension` (`trimToNull`), `shipRemainingQualityPct`/`shipDeadweight`/`shipMainEnginePowerHp` (`scale2`: HALF_UP 2 chữ số lẻ), `shipForSalePriceUsd` (B01 ghi `f.forSalePriceUsd()`; muốn giữ giá trị cũ khi request không gửi xem Vấn đề #9), `marketPrice = 0L`, `apartmentOwnershipRightPrice = 0L`, `landUseRightPrice = trade != 0 ? trade : estimate`.
- `GroupCompareService.get(...)` trả `GroupCompareData(..., canSyncToKho, CurrencyCodes.resolveCurrencyCode(assetType, group.getCurrencyCode()))`.
- `GroupCompareService` thêm field `CompareInformationRepository kho` (constructor: `scope, compares, factors, groups, khoSync, events, kho, clock`; sửa `setUp` của test).
- `GroupCompareService.save(...)`:
  1. Sau khi nạp `saved` và `byId`, **trước vòng lặp** gọi `requireValid(group, assetType, byId, request)`:
     - nếu `assetType == SHIP`: mỗi `input` có `shipRemainingQualityPct != null` và `< 0` hoặc `> 100` → `BusinessException(ErrorCode.VALIDATION_ERROR, "Chất lượng còn lại tàu phải từ 0 đến 100")`;
     - với mỗi `input` có `compareInformationId != null`, dòng `byId.get(input.id())` tồn tại, `type` trong 1..3 và `compareInformationId` **khác** `row.getCompareInformationId()`: gom id, `kho.findAllById(ids)`; với mỗi dòng kho tìm thấy, nếu `CurrencyCodes.resolveCurrencyCode(assetType, group.getCurrencyCode())` khác `CurrencyCodes.resolveCurrencyCode(ci.getAssetType(), ci.getCurrencyCode())` → `BusinessException(ErrorCode.VALIDATION_ERROR, "Tài sản so sánh trong kho dùng đơn vị " + ciCurrency + ", khác đơn vị tiền tệ của nhóm (" + groupCurrency + ")")`. Dòng kho không tìm thấy thì bỏ qua (khớp L, `CompareInformationSync.select` sẽ bỏ liên kết). Dòng kho đã xóa mềm vẫn được xét (L không kiểm `deleted`).
     - Kiểm tra chạy hết **trước** `writeRow` và `khoSync.apply`; lỗi thì không ghi cột nào, không gọi `compares.saveAll`, `khoSync`, `events`.
  2. `writeRow`: gộp chi tiết diện tích (`AreaDetailsMerge`) và cây trồng **chỉ khi `assetType == LAND`** (thay cho `!= APARTMENT && != CAR`); SHIP, APARTMENT, CAR không gộp.
  3. `if (assetType == LAND || assetType == APARTMENT)`: ghi `comparativeAnalysis` và `requestWalkingRoute` (thay cho `!= CAR`). Tàu **không** ghi `comparativeAnalysis` và **không** phát sự kiện đường đi bộ.
  4. `syncFactors` giữ nguyên chỗ gọi (mục B05b lo nội dung).

- [ ] **Step 1: Kiểm tra điều kiện từ B01**
  ```bash
  git -C backend grep -n "shipBuildYear\|shipForSalePriceUsd" -- src/main/java/com/thanhdo/tdprojectbe/model/entity/AppraisalGroupCompare.java
  git -C backend grep -n "getCurrencyCode\|applyCurrency" -- src/main/java/com/thanhdo/tdprojectbe/model/entity/AppraisalAssetGroup.java
  git -C backend grep -n "resolveCurrencyCode" -- src/main/java/com/thanhdo/tdprojectbe/common/util/CurrencyCodes.java
  git -C backend grep -n "currencyCode" -- src/main/java/com/thanhdo/tdprojectbe/model/entity/CompareInformation.java
  git -C backend grep -n "applyShipForm\|record ShipCompareFields" -- src
  ```
  Thiếu cột/hàm/record của B01 (kể cả `applyShipForm(ShipCompareFields, long, long)`) thì **dừng, báo controller**; không tự viết thân `applyShipForm`. Kiểm tra DB: `DESCRIBE appraisal_group_compares;` có 9 cột `ship_*`; `DESCRIBE compare_information;` có `currency_code`.

- [ ] **Step 2: Viết test thất bại — entity** `NT model/entity/AppraisalGroupCompareShipFormTest.java`. **Phạm vi: nối dây `applyForm(..., SHIP, ...)` đến `applyShipForm` (`marketPrice` bị ghi đè 0 sau dòng chung, `address` 4 cột, cột đất/xe không đổi, giữ `shipForSalePriceUsd`). Test thân `applyShipForm` đã có ở B01 (`AppraisalGroupCompareShipTest`), không nhân đôi các ca trim/scale2/giá bán** (dùng `GroupCompareFixtures.row(id).with("shipClass", ...).build()`; `with(...)` tìm component theo tên nên 9 tên mới phải có trong record trước khi test chạy). Dựng dòng bằng `GroupCompareFixtures.saved(id, type)`:
  - `tàu_giá_bán_ưu_tiên_giá_giao_dịch`: trade 200, estimate 300 → `landUseRightPrice == 200`.
  - `tàu_giá_bán_lấy_giá_ước_tính_khi_giao_dịch_bằng_0_hoặc_null`: trade null/0, estimate 300 → 300; cả hai null → 0.
  - `tàu_market_price_luôn_0`: trade 200 → `marketPrice == 0` (khác đất: 200).
  - `tàu_apartmentOwnershipRightPrice_là_0` (đặt sẵn 999 trước đó).
  - `tàu_ghi_địa_chỉ_cho_cả_bốn_cột`: type 0..3 đều nhận `address` đã trim, rỗng thành null.
  - `chuỗi_tàu_trim_và_rỗng_thành_null`: `shipBuildPlace "  Hải Phòng "` → `"Hải Phòng"`; `shipUsagePurpose "   "` → null; tương tự `shipClass`, `shipDimension`.
  - `số_tàu_làm_tròn_hai_chữ_số_lẻ`: `shipDeadweight 1234.5678` → `1234.57`; `shipMainEnginePowerHp 450.005` → `450.01`; `shipRemainingQualityPct 85.555` → `85.56`.
  - `giữ_shipForSalePriceUsd_khi_không_gửi`: dòng có sẵn 1500.00, input không gửi (null) → vẫn 1500.00.
  - `ghi_shipForSalePriceUsd_khi_có_gửi`: input 2000 → 2000.00.
  - `tàu_không_đụng_cột_đất_xe_căn_hộ_và_cột_bảng_điều_chỉnh`: đặt sẵn `carName`, `landUseTerm`, `areaOutside`, `latitude`, `information`, `legal`, `convertToStandardCompareUnitPrice`, `unitPricePlanning`, `structure`, `wallCenterAreaHundredths`; sau `applyForm(..., SHIP, ...)` các cột đó **không đổi** (input cố tình gửi giá trị khác).
  - `ô_tô_vẫn_dùng_tổng_giá_và_không_đụng_cột_tàu`: `applyForm(CAR)` → `landUseRightPrice = trade + estimate`; 9 cột tàu giữ nguyên.
  - `đất_và_căn_hộ_không_đụng_cột_tàu`: đặt sẵn 9 cột tàu, `applyForm(LAND/APARTMENT)` với input có giá trị tàu → giữ nguyên.

- [ ] **Step 3: Viết test thất bại — service** `NT service/GroupCompareServiceTest.java`. Sửa `setUp` (thêm mock `CompareInformationRepository kho`), giữ `givenGroup`/`givenRealGroup` hiện có; nhóm tàu dùng `AppraisalAssetGroup` thật + `group.applyCurrency("USD", 25450L, null)` (B01). Thêm:
  - `get_tàu_trả_currencyCode_USD_đã_resolve`; `get_tàu_currencyCode_null_là_VND`; `get_nhóm_đất_lưu_USD_vẫn_trả_VND` (khóa `resolveCurrencyCode` ép loại khác về VND); `get_nhóm_ô_tô_trả_VND`.
  - `get_tàu_trả_chín_trường_tàu_trong_dòng`.
  - `save_tàu_ghi_cột_tàu_và_giá_bán_từ_giá_giao_dịch` (dựng request 4 dòng, kiểm `landUseRightPrice` theo trade/estimate, `marketPrice 0`).
  - `tàu_clcl_101_trả_400`: `BusinessException`, `errorCode == VALIDATION_ERROR`, message "Chất lượng còn lại tàu phải từ 0 đến 100"; `verify(compares, never()).saveAll(any())`, `verifyNoInteractions(khoSync, events)`.
  - `tàu_clcl_âm_trả_400` (-0,01); `tàu_clcl_0_và_100_hợp_lệ`; `tàu_clcl_null_hợp_lệ`.
  - `clcl_ngoài_khoảng_bị_bỏ_qua_với_nhóm_đất` (nhóm LAND gửi `shipRemainingQualityPct=500` → không lỗi, cột tàu không đổi).
  - `tàu_không_ghi_phân_tích_so_sánh` (group.getComparativeAnalysis giữ nguyên, `groups.save` không gọi) — tương ứng `ô_tô_không_ghi_phân_tích_so_sánh` hiện có.
  - `tàu_không_phát_sự_kiện_đường_đi_bộ` (nhân bản `ô_tô_không_phát`: `verifyNoInteractions(events)` dù tọa độ có đổi).
  - `tàu_không_gộp_chi_tiết_diện_tích` (dòng đã có `areaOutsideDetails` → giữ nguyên dù request gửi chi tiết).
  - Tiền tệ TSSS: `tsss_khác_tiền_tệ_trả_400_trước_mọi_ghi` (nhóm tàu USD, kho VND/null → ném; `verify(khoSync, never()).apply(...)`; `compares.saveAll` không gọi; dòng không bị `applyForm`), message "Tài sản so sánh trong kho dùng đơn vị VND, khác đơn vị tiền tệ của nhóm (USD)"; `tsss_usd_gắn_nhóm_tàu_usd_hợp_lệ`; `tsss_kho_currency_null_coi_là_vnd_gắn_nhóm_vnd_hợp_lệ`; `nhóm_đất_gắn_tsss_usd_trả_400` (kho tàu USD, nhóm đất VND); `liên_kết_không_đổi_không_kiểm_tra_và_không_truy_vấn_kho` (`verify(kho, never()).findAllById(any())`); `tsss_không_tồn_tại_bỏ_qua_kiểm_tra`; `cột_type_0_không_kiểm_tra_tiền_tệ`; `bỏ_chọn_kho_không_kiểm_tra` (input `compareInformationId=null`); `lỗi_ở_cột_thứ_ba_vẫn_chặn_cả_lần_lưu` (cột 1 hợp lệ, cột 3 lệch → cột 1 cũng không được ghi).
  - Sửa `setUp` mọi chỗ khác dựng service. Các test cũ phải vẫn PASS.

- [ ] **Step 4: Viết test thất bại — controller + repository**
  - `NT controller/GroupCompareControllerTest.java`: sửa helper `data(...)` (dòng ~60) cho khớp constructor `GroupCompareData` mới (thêm `"VND"`). Thêm `get_trả_currencyCode_và_trường_tàu` (`$.data.currencyCode` = `"USD"`, `$.data.rows[0].shipBuildYear`, `shipForSalePriceUsd`), `put_nhận_chín_trường_tàu` (captor kiểm `shipBuildYear`, `shipDeadweight`…), `put_tàu_clcl_101_trả_400_có_thông_báo` (stub `doThrow(new BusinessException(ErrorCode.VALIDATION_ERROR, "Chất lượng còn lại tàu phải từ 0 đến 100")).when(service).save(...)` → `status().isBadRequest()` và `$.message`), `put_năm_đóng_ngoài_1900_2100_trả_400`, `put_trọng_tải_âm_trả_400`, `put_dimension_quá_dài_trả_400` (256 ký tự).
  - `NT repository/GroupCompareSaveRepositoryTest.java` (`@MysqlDataJpaTest`): `lưu_tàu_ghi_đúng_cột_xuống_db_và_giữ_shipForSalePriceUsd` — dòng nhóm tàu, `applyForm(..., SHIP, ...)` rồi flush/clear, truy vấn native các cột `ship_*`, `land_use_right_price`, `market_price`, `apartment_ownership_right_price`, `address`; giá trị đúng; sau lần lưu thứ hai không gửi USD thì `ship_for_sale_price_usd` giữ nguyên.

- [ ] **Step 5: Chạy xác nhận FAIL**
  ```bash
  cd backend && ./mvnw -q test -Dtest='AppraisalGroupCompareShipFormTest,GroupCompareServiceTest,GroupCompareControllerTest'
  ```
  Kỳ vọng: lỗi biên dịch (record chưa có component) hoặc FAIL.

- [ ] **Step 6: Cài đặt** theo Files/Interfaces. Lưu ý:
  - `GroupCompareRow.from(...)` thêm 9 getter đúng thứ tự record.
  - `AppraisalGroupCompare.applyForm` bổ sung nhánh SHIP: ghi `address`, rồi gọi `applyShipForm(<ShipCompareFields dựng từ in>, trade, estimate)`, đặt SAU dòng `this.marketPrice = ...` chung (Quyết định #2: `applyShipForm` ghi `marketPrice = 0`). Ghi Javadoc: "tàu: address, source, contact, 3 giá, 9 cột tàu (`ship_for_sale_price_usd` chỉ ghi khi gửi), `land_use_right_price` (giá giao dịch nếu khác 0, không thì giá ước tính; là 'Giá bán' của bảng điều chỉnh), `market_price` = 0, `apartment_ownership_right_price` = 0".
  - `GroupCompareService` Javadoc lớp/`save` ghi các điểm "Khác hệ thống cũ" nêu trên.
  - Dùng `Texts.trimToNull`, `BigDecimal`, `CurrencyCodes` từ B01; không viết lại helper.

- [ ] **Step 7: Chạy xác nhận PASS**
  ```bash
  cd backend && ./mvnw -q test -Dtest='AppraisalGroupCompareShipFormTest,GroupCompareServiceTest,GroupCompareControllerTest,AppraisalGroupCompareFormTest,GroupCompareSaveRepositoryTest,GroupCompareEntitiesRepositoryTest'
  ```
  Kỳ vọng: PASS, các test cũ của đất/căn hộ/ô tô không đổi kết quả (`ô_tô_vẫn_...`).

- [ ] **Step 8: Commit**
  ```bash
  git -C backend add src/main/java/com/thanhdo/tdprojectbe/model/dto/GroupCompareRowInput.java src/main/java/com/thanhdo/tdprojectbe/model/dto/GroupCompareRow.java src/main/java/com/thanhdo/tdprojectbe/model/dto/GroupCompareData.java src/main/java/com/thanhdo/tdprojectbe/model/entity/AppraisalGroupCompare.java src/main/java/com/thanhdo/tdprojectbe/service/GroupCompareService.java src/test/java/com/thanhdo/tdprojectbe
  git -C backend commit -m "feat: lưu và đọc tab so sánh tàu thủy, chặn gắn tài sản so sánh khác tiền tệ"
  ```
  (Chỉ `git add` các file thuộc B05 trong danh sách; thay `src/test/java/com/thanhdo/tdprojectbe` bằng từng file test của B05 nếu worktree có file test của task khác.)

### Task B05b: Đồng bộ nhãn tàu sang yếu tố điều chỉnh (`CompareFactorSync`)

**Files:**
- **Không** sửa `ControlCompareFactorLabels` (Quyết định #3: B01 tạo `BUILD_YEAR`, `BUILD_PLACE`, `OVERALL_APPEARANCE`, `OPERABILITY`, `DEADWEIGHT`, `POWER_HP`, `VESSEL_CLASS`; "Chất lượng còn lại" dùng `REMAINING_QUALITY` sẵn có). B05b chỉ dùng.
- Modify `N service/CompareFactorSync.java`: `sourceOf` thêm 5 nhãn tàu, rẽ `REMAINING_QUALITY` theo CAR/SHIP; thêm helper định dạng; Javadoc.
- Modify test `NT service/CompareFactorSyncTest.java`, `NT service/GroupCompareServiceTest.java` (một vài test đồng bộ qua `save`).

**Interfaces:**
- `CompareFactorSync.syncedValues(label, assetType, byType)` (chữ ký không đổi) với `assetType == SHIP`:

  | Nhãn | Nguồn (cột `AppraisalGroupCompare`) | Định dạng |
  |---|---|---|
  | Năm đóng | `shipBuildYear` | `String.valueOf`, null → null |
  | Nơi đóng | `shipBuildPlace` | `trimToNull` |
  | Trọng tải toàn phần | `shipDeadweight` | `EuropeanNumbers.formatAlwaysFixed2(value.doubleValue())` (vd `12.345,50`; `0` → `0,00`) |
  | Công suất (CV) | `shipMainEnginePowerHp` | như trên |
  | Chất lượng còn lại | `shipRemainingQualityPct` | như trên (CAR vẫn `carRemainingQualityPct`; loại khác không đồng bộ) |
  | Cấp phương tiện | `shipClass` | `trimToNull` |

  Nhãn tàu với loại tài sản khác SHIP → `Optional.empty()`. "Ngoại quan tổng thể", "Khả năng vận hành" → `Optional.empty()` (rơi `default`). Nhãn đất/căn hộ/ô tô ("Diện tích", "Xuất xứ", "Năm sản xuất", "Số km đã đi", …) với SHIP → `Optional.empty()`. Các nhãn dùng chung mọi loại (Chiều rộng đường, Loại đường, Kích thước, Số mặt tiền, Mục đích sử dụng đất) giữ hành vi hiện tại, không đổi.
- Giá trị null ghi thành `""` ở `GroupCompareService.syncFactors` (đã có), không đổi.
- Lưu ý tên nhãn khớp theo **text trim** của `c_label` (seed legacy `V20261103120000` đã đổi "Năm đóng/hoán cải" thành "Năm đóng").

- [ ] **Step 1: Kiểm tra hằng nhãn của B01 và trạng thái**
  ```bash
  git -C backend grep -n "Năm đóng\|Nơi đóng\|Trọng tải toàn phần\|Công suất (CV)\|Cấp phương tiện" -- src/main/java
  ```
  B01 đã tạo các hằng; thiếu thì **dừng, báo controller**, không tự thêm. (`case SHIP` của `ControlFactorDefaults` thuộc B07, không sửa ở đây.)

- [ ] **Step 2: Viết test thất bại** trong `NT service/CompareFactorSyncTest.java` (dùng helper `compare(type, field, value)` hiện có):
  - `đồng_bộ_năm_đóng_sang_yếu_tố`: `shipBuildYear` 2015 ở type 0, 2012 ở type 2 → `["2015", null, "2012", null]`.
  - `đồng_bộ_nơi_đóng_trim_và_rỗng_thành_null`: `"  Hải Phòng "` → `"Hải Phòng"`, `"   "` → null.
  - `trọng_tải_định_dạng_châu_âu_2_lẻ`: `12345.5` → `"12.345,50"`; `5000` → `"5.000,00"`; `0` → `"0,00"`.
  - `công_suất_định_dạng_châu_âu_2_lẻ`: `1500` → `"1.500,00"`; `450.25` → `"450,25"`.
  - `clcl_tàu_lấy_cột_tàu`: SHIP, `shipRemainingQualityPct` 85.5 → `"85,50"`, dù `carRemainingQualityPct` có giá trị khác.
  - `clcl_ô_tô_vẫn_lấy_cột_xe`: CAR → `carRemainingQualityPct`, không đọc cột tàu.
  - `clcl_đất_và_căn_hộ_không_đồng_bộ`: `Optional.empty()`.
  - `cấp_phương_tiện_trim`.
  - `ngoại_quan_và_khả_năng_vận_hành_không_đồng_bộ`: SHIP → `Optional.empty()`.
  - `nhãn_tàu_không_đồng_bộ_cho_loại_khác`: "Năm đóng"/"Nơi đóng"/"Trọng tải toàn phần"/"Công suất (CV)"/"Cấp phương tiện" với LAND, APARTMENT, CAR → empty.
  - `nhãn_đất_và_ô_tô_không_đồng_bộ_cho_tàu`: "Diện tích", "Xuất xứ", "Năm sản xuất", "Số km đã đi", "Loại căn hộ" với SHIP → empty.
  - `nhãn_dùng_chung_vẫn_đồng_bộ_cho_tàu`: "Chiều rộng đường" với SHIP → vẫn có (khóa không phá hành vi hiện có).
  - Trong `NT service/GroupCompareServiceTest.java`: `tàu_chỉ_đồng_bộ_yếu_tố_đồng_bộ_được` (nhóm tàu có 8 yếu tố mặc định "Năm đóng", "Nơi đóng", "Ngoại quan tổng thể", "Khả năng vận hành", "Trọng tải toàn phần", "Chất lượng còn lại", "Công suất (CV)", "Cấp phương tiện": 6 yếu tố đầu được `applySyncedValues`, hai yếu tố "Ngoại quan tổng thể"/"Khả năng vận hành" không bị ghi), và `tàu_không_ghi_tỷ_lệ_đất_ở` (yếu tố có `labelId == 8` không dính tới nhóm tàu).

- [ ] **Step 3: Chạy xác nhận FAIL**
  ```bash
  cd backend && ./mvnw -q test -Dtest='CompareFactorSyncTest'
  ```

- [ ] **Step 4: Cài đặt** `sourceOf`:
  ```java
  boolean ship = assetType == AppraisalAssetType.SHIP;
  ...
  case ControlCompareFactorLabels.REMAINING_QUALITY -> car ? CompareFactorSync::carQuality
          : ship ? CompareFactorSync::shipQuality : null;
  case ControlCompareFactorLabels.BUILD_YEAR -> ship ? CompareFactorSync::shipBuildYear : null;
  case ControlCompareFactorLabels.BUILD_PLACE -> ship ? c -> trimToNull(c.getShipBuildPlace()) : null;
  case ControlCompareFactorLabels.DEADWEIGHT -> ship ? c -> fixed2(c.getShipDeadweight()) : null;
  case ControlCompareFactorLabels.POWER_HP -> ship ? c -> fixed2(c.getShipMainEnginePowerHp()) : null;
  case ControlCompareFactorLabels.VESSEL_CLASS -> ship ? c -> trimToNull(c.getShipClass()) : null;
  ```
  Helper `fixed2(BigDecimal)`: null → null, ngược lại `EuropeanNumbers.formatAlwaysFixed2(value.doubleValue())`. Cập nhật Javadoc lớp: bảng nhãn tàu và "Khác hệ thống cũ" (nhãn không áp dụng cho loại tài sản thì không đồng bộ).

- [ ] **Step 5: Chạy xác nhận PASS**
  ```bash
  cd backend && ./mvnw -q test -Dtest='CompareFactorSyncTest,GroupCompareServiceTest'
  ```

- [ ] **Step 6: Commit**
  ```bash
  git -C backend add src/main/java/com/thanhdo/tdprojectbe/service/CompareFactorSync.java src/test/java/com/thanhdo/tdprojectbe/service/CompareFactorSyncTest.java src/test/java/com/thanhdo/tdprojectbe/service/GroupCompareServiceTest.java
  git -C backend commit -m "feat: đồng bộ sáu nhãn tàu thủy từ tab so sánh sang bảng điều chỉnh"
  ```

### Task B05c: Bảng điều chỉnh tàu (tiền tệ và đơn vị)

**Files:**
- Modify `N model/dto/AdjustmentPanelData.java`: thêm component cuối `String currencyUnit` (Javadoc `@param`: "đơn vị tiền tệ của nhóm: `VND` hoặc `USD`; dùng cho nhãn đơn vị của các dòng tàu"). Sửa `@param regulatoryPriceUnit`: "`đồng/m²`, `đồng/xe` (ô tô), `đồng` hoặc `USD` (tàu)".
- Modify `N service/RegulatoryConstraints.java`: `priceUnit(int assetType)` đổi thành `priceUnit(int assetType, String currencyCode)` gọi `CurrencyCodes.regulatoryPriceUnit(assetType, currencyCode)` (B01); xóa hai hằng `PER_SQM`, `PER_VEHICLE` đã chuyển sang `CurrencyCodes`. Cập nhật Javadoc. (Hàm này chỉ có `AdjustmentPanelService` gọi.)
- Modify `N service/AdjustmentPanelService.java`: `get` tính `currencyUnit` và `regulatoryPriceUnit`.
- Modify test `NT service/AdjustmentPanelServiceTest.java`, `NT controller/AdjustmentPanelControllerTest.java`.
- **Không** sửa `N common/constant/CommonConstants.java` (Word, B08/B09).

**Interfaces:**
- `AdjustmentPanelService.get(...)`:
  ```java
  String currencyCode = CurrencyCodes.resolveCurrencyCode(assetType, group.getCurrencyCode());
  ... new AdjustmentPanelData(..., RegulatoryConstraints.TEMPLATE,
          RegulatoryConstraints.priceUnit(assetType, group.getCurrencyCode()),
          CurrencyCodes.currencyUnitLabel(currencyCode));
  ```
  - Tàu VND: `regulatoryPriceUnit == "đồng"`, `currencyUnit == "VND"`; tàu USD: `"USD"` và `"USD"`.
  - CAR: `"đồng/xe"`, `currencyUnit "VND"`; LAND/APARTMENT: `"đồng/m²"`, `currencyUnit "VND"` (kể cả khi cột `currency_code` của nhóm lưu `USD` sai chỗ: `resolveCurrencyCode` ép loại khác về VND).
- Thứ tự và lọc nhãn yếu tố không đổi về cách làm: `labels.findByDeleteFlagOrderByIdAsc(0)` rồi `appliesToAssetType(4)`; seed legacy (`V20261015100300`, `V20261103120000`) đặt "Khả năng vận hành" và "Chất lượng còn lại" với `asset_types` [3,4], các nhãn riêng của tàu với [4]. Không cần thêm code; test khóa hành vi trên SHIP.
- `PanelColumn` không đổi: với tàu `landUseRightPrice` là "Giá bán" (dòng A mới của FE), `marketPrice` luôn 0.
- Phần lưu (`save`) không đổi (công thức do FE tính, server chỉ ghi giá trị).

- [ ] **Step 1: Viết test thất bại** trong `NT service/AdjustmentPanelServiceTest.java` (dùng `givenGroup(assetType)`, `group.applyCurrency(...)` của B01; dùng `label(id, name, assetTypes...)` có sẵn):
  - `đơn_vị_tàu_usd`: nhóm SHIP + `applyCurrency("USD", 25450L, null)` → `regulatoryPriceUnit == "USD"`, `currencyUnit == "USD"`.
  - `đơn_vị_tàu_vnd`: SHIP + mặc định → `"đồng"` và `"VND"`; không chứa `"/m²"` hay `"/tàu"`.
  - `ô_tô_vẫn_đồng_xe`: thêm assert `currencyUnit == "VND"` (nới test `ô_tô_cột_A_là_tên_xe_và_đơn_vị_giá_là_đồng_trên_xe`).
  - `đất_vẫn_đồng_m2_và_căn_hộ_vẫn_đồng_m2`.
  - `loại_khác_tàu_lưu_usd_vẫn_trả_vnd`: LAND/CAR có `currency_code = USD` (đặt bằng `ReflectionTestUtils`) → `currencyUnit "VND"` và đơn vị đồng/m² hoặc đồng/xe.
  - `nhãn_tàu_lọc_theo_loại_4_và_giữ_thứ_tự_mã`: labels `[2 "Diện tích" (1,2)] [14 "Năm đóng" (4)] [15 "Nơi đóng" (4)] [16 "Ngoại quan tổng thể" (4)] [17 "Khả năng vận hành" (3,4)] [18 "Trọng tải toàn phần" (4)] [19 "Chất lượng còn lại" (3,4)] [20 "Công suất (CV)" (4)] [21 "Cấp phương tiện" (4)] [9 "Khác" (1,2,3,4)]` trả về theo thứ tự mã tăng dần của repository → với SHIP: `[9, 14, 15, 16, 17, 18, 19, 20, 21]`; với CAR: `[9, 17, 19]`; không có "Diện tích".
  - `cột_tàu_giá_bán_là_landUseRightPrice_và_market_price_0`: dòng type 0 có `landUseRightPrice=4000`, `marketPrice=0` → `PanelColumn(0, 0L, 4000L, null)`.
  - Sửa mọi test khác dựng `AdjustmentPanelData` để khớp constructor mới.
  - `NT controller/AdjustmentPanelControllerTest.java`: sửa fixture (dòng ~67) thêm `"VND"`; thêm `get_trả_currencyUnit_và_đơn_vị_tàu` (`$.data.currencyUnit` = `"USD"`, `$.data.regulatoryPriceUnit` = `"USD"`).

- [ ] **Step 2: Chạy xác nhận FAIL**
  ```bash
  cd backend && ./mvnw -q test -Dtest='AdjustmentPanelServiceTest,AdjustmentPanelControllerTest'
  ```

- [ ] **Step 3: Cài đặt** theo Files/Interfaces. Cập nhật Javadoc `AdjustmentPanelService`: thêm "Khác hệ thống cũ: dữ liệu có `currencyUnit`; đơn vị 'Nguyên tắc khống chế' của tàu theo tiền tệ của nhóm".

- [ ] **Step 4: Chạy xác nhận PASS**
  ```bash
  cd backend && ./mvnw -q test -Dtest='AdjustmentPanelServiceTest,AdjustmentPanelControllerTest,AdjustmentPanelRepositoryTest'
  ```

- [ ] **Step 5: Commit**
  ```bash
  git -C backend add src/main/java/com/thanhdo/tdprojectbe/model/dto/AdjustmentPanelData.java src/main/java/com/thanhdo/tdprojectbe/service/RegulatoryConstraints.java src/main/java/com/thanhdo/tdprojectbe/service/AdjustmentPanelService.java src/test/java/com/thanhdo/tdprojectbe/service/AdjustmentPanelServiceTest.java src/test/java/com/thanhdo/tdprojectbe/controller/AdjustmentPanelControllerTest.java
  git -C backend commit -m "feat: bảng điều chỉnh tàu thủy trả đơn vị tiền tệ và đơn vị nguyên tắc khống chế theo nhóm"
  ```

- [ ] **Step cuối B05: chạy bộ liên quan**
  ```bash
  cd backend && ./mvnw -q test -Dtest='GroupCompare*,CompareFactorSync*,AdjustmentPanel*,AppraisalGroupCompare*'
  ```

---

# B10 — Kho TSSS tàu

## Hành vi hệ thống cũ

Nguồn đọc: `L controller/AppraisalReportController.java:562-569` (API tìm); `L service/impl/CompareInformationServiceImpl.java:136-170` (`searchCompareInformationByKeyword`); `L repository/CompareInformationRepository.java:33-40` (`searchByShipTypeKeyword`); `L entity/dto/CompareInformationShipSearchDTO.java:15-60`; `L service/AppraisalGroupCompareService.java:1215-1318` (`syncGroupCompareToCompareInformation`, ship 1307-1317); `T/demo/comparative-information-ship.html:186-218` (quyền checkbox), `J/comparative-information-ship-search.js`, `surveys/legacy-ui-word.md` mục 5 ("Checkbox đồng bộ kho", "Modal tìm TSSS tàu").

- **Tìm kiếm**: `keyword` rỗng → 400 "Vui lòng nhập từ khóa để tìm kiếm"; `assetType` chỉ nhận CAR và SHIP (khác → 400 "Loại tài sản không hợp lệ"). SHIP: `deleted = 0 AND asset_type = 4 AND address IS NOT NULL AND address LIKE %keyword%` (native, keyword đã trim, không escape ký tự `%`/`_`), sắp `updated DESC`, trang 20 (tối đa 50). Trả `ShipSearchDTO` gồm `id, address, source, contact, shipBuildYear, shipBuildPlace, shipUsagePurpose, shipClass, shipRemainingQualityPct, shipDeadweight, shipDimension, shipMainEnginePowerHp, currencyCode` (đã `resolveCurrencyCode(SHIP, ...)`, null → VND), `forSalePrice, tradePrice, estimatePrice, updated` (chuỗi `dd/MM/yyyy`).
- **Đồng bộ dòng 1..3 lên kho** (`syncGroupCompareToCompareInformation`): dedupe theo `syncToCompareInformationId`; `ci.assetType` = loại nhóm; copy toàn bộ trường; với tàu copy 8 trường tàu (`shipBuildYear, shipBuildPlace, shipUsagePurpose, shipClass, shipRemainingQualityPct, shipDeadweight, shipDimension, shipMainEnginePowerHp`, dòng 1307-1314); `ci.currencyCode = resolveCurrencyCode(group.assetType, group.currencyCode)` (dòng 1315-1317), `VND` nếu không có nhóm. **Không** copy `shipForSalePriceUsd` (kho không có cột này).
- **Quyền checkbox đồng bộ kho** (`comparative-information-ship.html:186-218`, `hasAnyAuthority('KSV','TPTD','BGD')`): trước đây chỉ KSV, TPTD; legacy mở thêm **BGD cho mọi loại tài sản** (đất, căn hộ, ô tô, tàu). Người không có quyền mà cột đang đồng bộ thì giữ giá trị 1 qua hidden (việc của FE).
- Chặn gắn TSSS khác tiền tệ: xem B05 (mục "Phân chia file", quyết định 1).

## Khác hệ thống cũ (liên quan task này, ghi vào Javadoc)

- Số 3: lỗi 400 qua `BusinessException` (keyword rỗng, loại tài sản sai).
- Kết quả tìm là JSON phân trang chuẩn của code mới (`Page`, trang đánh số từ 0), `keyword` được **escape** (`LikePattern.contains`), `updated` là `LocalDate` ISO `YYYY-MM-DD` (Quyết định #7; cũ: chuỗi `dd/MM/yyyy`). Nhất quán với `CompareInformationSearchService.searchCars`.
- Thông báo thiếu từ khóa của tàu là "Vui lòng nhập chủng loại để tìm kiếm" (tìm theo chủng loại, không phải tên xe).
- `shipForSalePriceUsd` không đồng bộ lên kho.
- Quyền tick đồng bộ kho ở code mới nằm ở BE (`GroupCompareController.get`), mở thêm BGD cho mọi loại tài sản như L.

### Task B10: Tìm kho tàu, đồng bộ lên kho, quyền tick đồng bộ

**Files:**
- Create `N model/dto/ShipCompareInformationItem.java`.
- Modify `N repository/CompareInformationRepository.java`: thêm `searchByShipAddress`.
- Modify `N service/CompareInformationSearchService.java`: thêm `searchShips`.
- Modify `N controller/CompareInformationController.java`: `search` rẽ theo `assetType`.
- Modify `N model/entity/CompareInformation.java`: `copyFrom` copy 8 trường tàu; thêm `useCurrency(String)`.
- Modify `N service/CompareInformationSync.java`: `push` ghi tiền tệ của nhóm.
- Modify `N controller/GroupCompareController.java`: `canSyncToKho` cho TPTD **hoặc BGD**.
- Modify test: `NT repository/CompareInformationSearchRepositoryTest.java`, `NT service/CompareInformationSearchServiceTest.java`, `NT service/CompareInformationSyncTest.java`, `NT controller/CompareInformationControllerTest.java`.
- Create test: `NT model/entity/CompareInformationShipTest.java`, `NT controller/GroupCompareControllerKhoPermissionTest.java`.
- **Không** sửa `GroupCompareService`, `GroupCompareControllerTest` (B05).

**Interfaces:**
- Yêu cầu từ B01: `CompareInformation` có 8 cột tàu và `currencyCode` kèm getter (`getShipBuildYear()`…, `getCurrencyCode()`); `AppraisalGroupCompare` có 8 getter tàu; `CurrencyCodes.resolveCurrencyCode(int, String)`; `AppraisalAssetGroup.getCurrencyCode()`; `AppraisalAssetType.SHIP`.
- `record ShipCompareInformationItem(long id, String address, String source, String contact, Integer shipBuildYear, String shipBuildPlace, String shipUsagePurpose, String shipClass, BigDecimal shipRemainingQualityPct, BigDecimal shipDeadweight, String shipDimension, BigDecimal shipMainEnginePowerHp, String currencyCode, Long forSalePrice, Long tradePrice, Long estimatePrice, LocalDate updated)` với `static ShipCompareInformationItem from(CompareInformation c)` (`updated = c.getUpdated().toLocalDate()`, JSON ISO `YYYY-MM-DD` theo Quyết định #7; FE tự định dạng `dd/MM/yyyy`); `currencyCode = CurrencyCodes.resolveCurrencyCode(AppraisalAssetType.SHIP, c.getCurrencyCode())`.
- Repository:
  ```java
  @Query("SELECT c FROM CompareInformation c WHERE c.deleted = 0 AND c.assetType = :assetType"
          + " AND c.address IS NOT NULL AND c.address LIKE :pattern ESCAPE '\\'")
  Page<CompareInformation> searchByShipAddress(@Param("pattern") String pattern, @Param("assetType") int assetType, Pageable pageable);
  ```
  JPQL (không cần native). Dùng index `(asset_type, deleted, updated)` sẵn có.
- `CompareInformationSearchService.searchShips(String keyword, Pageable pageable)` → `Page<ShipCompareInformationItem>`: keyword null/trống → `BusinessException(VALIDATION_ERROR, "Vui lòng nhập chủng loại để tìm kiếm")`; sắp `updated DESC, id DESC` (giống `searchCars`); pattern `LikePattern.contains(keyword)`; `assetType` cố định `AppraisalAssetType.SHIP`. `searchCars` và `nearby` giữ nguyên (SHIP không có tọa độ nên không thêm vào `nearby`).
- `CompareInformationController.search(...)`: kiểu trả về đổi thành `Page<?>`; nếu `assetType != null && assetType == AppraisalAssetType.SHIP` gọi `service.searchShips(keyword, PageRequest.of(page, size))`, ngược lại `service.searchCars(...)` (giữ kiểm `assetType == CAR`, loại khác 400). URL: `GET /api/compare-informations/search?keyword&assetType=4&page&size` (page từ 0, size 1..50, mặc định 20). Quyền vào do `SecurityConfig` hiện có (nhóm `/api/compare-informations/**`), không đổi.
- `CompareInformation.copyFrom(row, assetType, now)`: thêm copy 8 trường tàu từ `row` (kể cả null) trong `copyApartmentAndCar`-tương đương mới `copyShip(row)`; `copyFrom` **không** đổi chữ ký. Thêm `public void useCurrency(String resolvedCode)` ghi `currencyCode`.
- `CompareInformationSync.push`: sau `target.copyFrom(row, assetType, now)` gọi `target.useCurrency(CurrencyCodes.resolveCurrencyCode(assetType, group == null ? null : group.getCurrencyCode()))` với `group = row.getAppraisalAssetGroup()`. Áp dụng cho **mọi loại** (L ghi `resolveCurrencyCode(group)` cho mọi loại; loại khác ra `VND`). `shipForSalePriceUsd` không được copy. Chữ ký `apply(row, assetType, input, callerId)` và constructor `(kho, images, clock)` **không đổi**.
- `GroupCompareController.get`: `canSyncToKho = authentication.getAuthorities().stream().anyMatch(a -> Positions.TPTD.equals(a.getAuthority()) || Positions.BGD.equals(a.getAuthority()))`; áp dụng cho mọi loại tài sản. Cập nhật Javadoc cờ: "TPTD (KSV có authority TPTD) hoặc BGD, như `hasAnyAuthority('KSV','TPTD','BGD')` của hệ thống cũ". Server `save` hiện **không** chặn người không có quyền (chỉ cờ cho UI), giữ nguyên.

- [ ] **Step 1: Kiểm tra điều kiện từ B01 và quy tắc quyền hiện có**
  ```bash
  git -C backend grep -n "shipBuildYear\|currencyCode" -- src/main/java/com/thanhdo/tdprojectbe/model/entity/CompareInformation.java
  git -C backend grep -n "getAppraisalAssetGroup" -- src/main/java/com/thanhdo/tdprojectbe/model/entity/AppraisalGroupCompare.java
  git -C backend grep -n "canSyncToKho\|Positions.TPTD" -- src/main/java/com/thanhdo/tdprojectbe/controller/GroupCompareController.java
  git -C frontend grep -n "canSyncToKho" -- src | head
  ```
  Kỳ vọng: cột tàu + `currencyCode` có ở `CompareInformation`; rule quyền nằm ở **BE** (`GroupCompareController`, chỉ xét `TPTD`), FE chỉ đọc cờ `canSyncToKho` (`useCompareForm.ts`). Nếu rule đã chuyển sang FE ở develop mới hơn thì **bỏ phần `GroupCompareController` và test quyền khỏi task** (B10 không đổi, ghi chú cho F08). Kiểm tra DB: `DESCRIBE compare_information;` có `ship_*` và `currency_code`.

- [ ] **Step 2: Viết test thất bại — repository** trong `NT repository/CompareInformationSearchRepositoryTest.java` (helper `insert(...)` hiện có; thêm tham số/ helper để đặt `address`, `updated`, `deleted`, `assetType`):
  - `tìm_tàu_theo_chủng_loại_không_phân_biệt_hoa_thường_mới_cập_nhật_trước`: 3 dòng asset_type 4 với address "Tàu chở hàng khô", "tàu chở hàng khô 2", "Tàu container" và `updated` khác nhau; keyword "hàng khô" → 2 dòng, thứ tự `updated DESC`.
  - `tìm_tàu_bỏ_đã_xóa_loại_khác_và_địa_chỉ_null`: dòng `deleted=1`, dòng asset_type 3 có address khớp, dòng asset_type 4 address null → bị loại.
  - `tìm_tàu_ký_tự_đặc_biệt_được_escape`: address "Tàu 100% thép" và "Tàu 1000 thép"; keyword "100%" chỉ khớp dòng đầu (qua `LikePattern.contains`); `_` tương tự.
  - `tìm_tàu_chia_trang`: 25 dòng, trang 0 cỡ 20 → 20, trang 1 → 5, `totalElements 25`.
  - `ghi_và_đọc_lại_cột_tàu_và_currency_code_của_kho`: lưu `CompareInformation` tàu có 8 cột ship + `currency_code "USD"` → đọc native đúng; dòng cũ `currency_code` NULL đọc ra `null`.

- [ ] **Step 3: Viết test thất bại — service tìm** trong `NT service/CompareInformationSearchServiceTest.java`:
  - `searchShips_từ_khóa_trống_trả_400_với_thông_báo_chủng_loại`; `searchShips_null_trả_400`.
  - `searchShips_gọi_repository_với_assetType_4_pattern_escape_và_sắp_updated_giảm_dần` (captor `Pageable`: sort `updated DESC` rồi `id DESC`, cùng page/size).
  - `searchShips_map_sang_ShipCompareInformationItem_đủ_trường`: kho tàu có `currency_code` null → `item.currencyCode() == "VND"`; `"usd"` → `"USD"`; các giá `forSalePrice/tradePrice/estimatePrice`, `updated` là `LocalDate` đúng ngày.
  - `searchCars_vẫn_chỉ_nhận_ô_tô` (assetType 4 vào `searchCars` → 400 "Loại tài sản không hợp lệ").

- [ ] **Step 4: Viết test thất bại — controller tìm** trong `NT controller/CompareInformationControllerTest.java`:
  - `search_tàu_gọi_searchShips_và_trả_dto_tàu`: `assetType=4&keyword=hàng khô&page=0&size=20` → `$.data.content[0].shipBuildYear`, `$.data.content[0].currencyCode`, `$.data.content[0].updated` (`"YYYY-MM-DD"`); `verify(service).searchShips(eq("hàng khô"), any())`, `verify(service, never()).searchCars(...)`.
  - `search_ô_tô_vẫn_gọi_searchCars`.
  - `search_tàu_thiếu_keyword_trả_400`, `search_tàu_size_51_trả_400`.
  (Kiểm tra JSON path đúng với cấu trúc hiện có của test `search_trả_trang_theo_cấu_trúc_chung`.)

- [ ] **Step 5: Viết test thất bại — đồng bộ lên kho** :
  - `NT model/entity/CompareInformationShipTest.java` (unit): `copyFrom_tàu_copy_tám_trường_tàu` (dòng so sánh type 1 có đủ 8 trường), `copyFrom_tàu_không_có_shipForSalePriceUsd` (kiểm bằng reflection: lớp `CompareInformation` không có field `shipForSalePriceUsd`), `copyFrom_ghi_đè_trường_tàu_cũ_bằng_null`, `copyFrom_đất_để_trường_tàu_null`, `useCurrency_ghi_mã`, `copyFrom_vẫn_đặt_assetType_và_deleted_0`.
  - `NT service/CompareInformationSyncTest.java` (giữ `setUp` hiện có; nhóm gắn vào dòng bằng `ReflectionTestUtils.setField(row, "appraisalAssetGroup", group)`, `group` = `AppraisalAssetGroup.create(...)` thật + `applyCurrency`):
    - `đẩy_dòng_tàu_lên_kho_ghi_currency_usd`: nhóm SHIP USD, `syncToCompareInfo=1` → dòng kho lưu có `currencyCode "USD"` và 8 trường tàu.
    - `đẩy_dòng_tàu_vnd_ghi_currency_vnd`.
    - `đẩy_dòng_đất_ghi_currency_vnd` (loại khác ra VND).
    - `đẩy_dòng_không_có_nhóm_ghi_vnd` (`appraisalAssetGroup == null`).
    - `đẩy_tàu_không_ghi_shipForSalePriceUsd` (dòng so sánh có `shipForSalePriceUsd=1500`, kho không có cột; kiểm qua `assertThat(CompareInformation.class.getDeclaredFields())` không chứa tên đó).
    - `đẩy_lần_hai_cập_nhật_cùng_dòng_kho_và_cập_nhật_tiền_tệ` (đổi nhóm sang VND giữa hai lần → `currencyCode` cập nhật).
    - `chọn_từ_kho_không_ghi_kho_và_không_đổi_tiền_tệ_kho` (nhánh `select` không gọi `useCurrency`; `verify(kho, never()).save(any())`).
    - Các test cũ giữ nguyên và PASS (constructor không đổi).
  - `NT controller/GroupCompareControllerKhoPermissionTest.java` (`@WebMvcTest(GroupCompareController.class)`, `@Import({SecurityConfig.class, JwtConfig.class, TokenService.class})`, `@TestPropertySource(properties = "app.security.jwt.secret=test-secret-test-secret-test-secret-123")`, `@MockitoBean GroupCompareService service` và `CompareSheetWordService wordService`, helper `bearer(authority)` copy từ `GroupCompareControllerTest`; **không** dựng `GroupCompareData`, chỉ `verify` đối số boolean):
    - `bgd_được_đẩy_kho`: `bearer("BGD")` → `verify(service).get(5L, 11L, 21L, true)`.
    - `tptd_được_đẩy_kho` (`"TPTD"` → true).
    - `nvtd_không_được_đẩy_kho` (`"NVTD"` → false).
    - `ksv_có_authority_tptd_được_đẩy_kho` (token của KSV theo cách `TokenService` cấp authority; nếu `bearer` chỉ nhận một authority thì dùng `"TPTD"` đại diện và ghi chú).
    - `quyền_không_phụ_thuộc_loại_tài_sản` (lặp `get` với groupId khác nhau, kết quả không đổi).

- [ ] **Step 6: Chạy xác nhận FAIL**
  ```bash
  cd backend && ./mvnw -q test -Dtest='CompareInformationShipTest,CompareInformationSyncTest,CompareInformationSearchServiceTest,CompareInformationControllerTest,GroupCompareControllerKhoPermissionTest'
  ```

- [ ] **Step 7: Cài đặt** theo Files/Interfaces:
  1. `ShipCompareInformationItem` (Javadoc: "Một tài sản tàu trong kho so sánh để chọn làm cột TSSS; port `CompareInformationShipSearchDTO`. Khác hệ thống cũ: `updated` là `LocalDate` ISO").
  2. Repository `searchByShipAddress` (Javadoc lý do JPQL, không native).
  3. `CompareInformationSearchService.searchShips` + hằng `SHIP_KEYWORD_REQUIRED_MESSAGE`; cập nhật Javadoc lớp.
  4. Controller `search` trả `Page<?>`, rẽ theo `assetType`; Javadoc nêu API tàu.
  5. `CompareInformation.copyShip` + `useCurrency`; cập nhật Javadoc `copyFrom` ("tàu: 8 trường tàu, không có giá rao bán USD; tiền tệ ghi riêng qua `useCurrency`").
  6. `CompareInformationSync.push` gọi `useCurrency`.
  7. `GroupCompareController`: thêm `Positions.BGD` vào điều kiện, Javadoc.

- [ ] **Step 8: Chạy xác nhận PASS**
  ```bash
  cd backend && ./mvnw -q test -Dtest='CompareInformationShipTest,CompareInformationSyncTest,CompareInformationSearchServiceTest,CompareInformationControllerTest,CompareInformationSearchRepositoryTest,GroupCompareControllerKhoPermissionTest,GroupCompareControllerTest'
  ```
  Test repository cần MySQL theo `backend/.env.example` và schema đã có cột ship + `currency_code` của `compare_information`.

- [ ] **Step 9: Commit**
  ```bash
  git -C backend add src/main/java/com/thanhdo/tdprojectbe/model/dto/ShipCompareInformationItem.java src/main/java/com/thanhdo/tdprojectbe/repository/CompareInformationRepository.java src/main/java/com/thanhdo/tdprojectbe/service/CompareInformationSearchService.java src/main/java/com/thanhdo/tdprojectbe/controller/CompareInformationController.java src/main/java/com/thanhdo/tdprojectbe/model/entity/CompareInformation.java src/main/java/com/thanhdo/tdprojectbe/service/CompareInformationSync.java src/main/java/com/thanhdo/tdprojectbe/controller/GroupCompareController.java src/test/java/com/thanhdo/tdprojectbe/repository/CompareInformationSearchRepositoryTest.java src/test/java/com/thanhdo/tdprojectbe/service/CompareInformationSearchServiceTest.java src/test/java/com/thanhdo/tdprojectbe/service/CompareInformationSyncTest.java src/test/java/com/thanhdo/tdprojectbe/controller/CompareInformationControllerTest.java src/test/java/com/thanhdo/tdprojectbe/model/entity/CompareInformationShipTest.java src/test/java/com/thanhdo/tdprojectbe/controller/GroupCompareControllerKhoPermissionTest.java
  git -C backend commit -m "feat: kho tài sản so sánh cho tàu thủy (tìm theo chủng loại, đồng bộ lên kho, quyền BGD)"
  ```

---

## Hết đợt B2 (các task trong file này): kiểm tra tích hợp sau khi merge

```bash
cd backend && ./mvnw -q test -Dtest='*Legal*,GroupCompare*,CompareFactorSync*,AdjustmentPanel*,CompareInformation*,AppraisalGroupCompare*,AppraisalReportAsset*'
```

Merge `wt/B04`, `wt/B05`, `wt/B10` theo thứ tự ID: không được có conflict (tập file rời nhau theo bảng đầu file). Conflict nghĩa là phân chia file sai, dừng và báo lại. Sau merge phải chạy riêng một test kết hợp để chắc ba task khớp nhau: `GroupCompareServiceTest` (B05) dùng `CompareInformationSync` thật bị mock nên không bắt được lỗi kết hợp; kiểm tra thủ công bằng tay hoặc ở Z01: lưu so sánh tàu USD chọn TSSS VND phải 400 và không có ảnh nào được clone.

## Vấn đề cần controller quyết định

1. **→ Đã chốt: Quyết định #2 (B01 viết thân hai hàm với `ShipLegalFields`/`ShipCompareFields`; B04/B05 chỉ dựng record và gọi).** (Nội dung cũ:) **`applyShipLegal` (B04) và `applyShipForm` (B05) phụ thuộc DTO của chính task, nhưng `00-index.md` giao B01 tạo hai phương thức này.** `LegalAssetInput` và `GroupCompareRowInput` do B04/B05 thêm component tàu, nên B01 (chạy trước) không thể biên dịch được chữ ký nhận DTO nếu DTO chưa có trường. Đề xuất: B01 chỉ thêm **cột + getter** trên `AppraisalReportAsset`/`AppraisalGroupCompare` (và `applyShipSync`, `clearShipFields`), còn **thân hàm** `applyShipLegal`/`applyShipForm` do B04/B05 viết. Plan này đã viết theo hướng đó (Step 1 của mỗi task kiểm tra B01 đã tạo nháp chưa; có thì sửa tại chỗ). Cần báo agent soạn `01-be-foundation.md` để không tạo hai phiên bản. Nếu B01 muốn giữ `applyShipForm(…)` nhận tham số rời (không dùng DTO) thì B05 chỉ đổi chỗ gọi trong `applyForm`.
2. **→ Đã chốt: Quyết định #10.** **`mergeLegalFieldsFrom` thêm 5 trường tàu giao cho B04** (cùng file `AppraisalReportAsset.java` với `applyShipLegal`, tránh hai task cùng sửa một file). B07 (vòng đời nhóm, đợt B3) chỉ dùng, không sửa; cần ghi chú sang `04-be-result-lifecycle.md`.
3. **→ Đã chốt: Quyết định #5.** **Chặn lệch tiền tệ TSSS đặt ở B05 (`GroupCompareService`), không ở B10.** Đã nêu lý do ở mục "Phân chia file". Hệ quả: `GroupCompareService` thêm dependency `CompareInformationRepository` (đổi constructor, test dựng service phải sửa `setUp`). B10 không làm gì cho việc này; nếu controller muốn B10 sở hữu thì phải đổi chữ ký `CompareInformationSync.apply` kèm chỗ gọi trong `GroupCompareService` và hai task sẽ không còn rời nhau.
4. **→ Đã chốt: Quyết định #6.** **Quy tắc quyền tick đồng bộ kho nằm ở BE, không phải FE** (`GroupCompareController.get`, chỉ xét `TPTD`). B10 sở hữu thay đổi (thêm BGD, áp dụng mọi loại). F08/FE không đổi. `GroupCompareControllerTest` (B05) không được B10 sửa, test quyền để ở file mới. Nếu `origin/develop` đã đổi chỗ đặt rule thì Step 1 của B10 yêu cầu bỏ phần này.
5. **→ Đã chốt: Quyết định #7.** **Hợp đồng nói kiểu trả về của API tìm kho không đổi URL**, nhưng `CompareInformationController.search` hiện khai báo `Page<CompareInformationDetail>`; plan đổi thành `Page<?>` và rẽ theo `assetType`. Cách khác (endpoint riêng `/search-ships`) rõ kiểu hơn nhưng sai hợp đồng "mở rộng API đang có".
6. **Giới hạn độ dài pháp lý (B04)**: BE chọn theo cột DB (`shipInspectionForWhom` ≤ 1000, hai trường text còn lại ≤ 500), không siết 500 cho `forWhom`/`shipInspectionForWhom` như JS legacy. Nếu muốn BE giữ đúng 500 thì đổi `@Size(max = 1000)` thành 500 ở `LegalAssetInput.shipInspectionForWhom` (nhưng `forWhom` hiện có đã 1000 cho cả đất/ô tô nên sẽ lệch nhau).
7. **CLCL tàu 0..100 kiểm trong `GroupCompareService` (BusinessException), không bằng bean validation**, vì trường này chỉ có nghĩa với nhóm tàu (khác `carRemainingQualityPct` đã có `@DecimalMin/@DecimalMax` trong DTO). Nếu muốn nhất quán với ô tô thì thêm annotation, nhưng khi đó nhóm khác gửi thừa trường sẽ bị 400.
8. **→ Đã chốt: Quyết định #15 (ngoài phạm vi).** **"Xuất phiếu TSSS" (Word, `CompareSheetWordService`) cho tàu chưa thuộc task nào** trong 00-index (không thấy ở B08/B09). Cần controller xác nhận có nằm trong phạm vi tàu hay không.
9. **`shipForSalePriceUsd` khi request không gửi (null):** hợp đồng index nói "được giữ nguyên khi lưu", nhưng B01 chốt `applyShipForm` ghi `f.forSalePriceUsd()` như L (null thì ghi null). Để giữ giá trị DB, B05 dựng `ShipCompareFields` với `forSalePriceUsd = in.shipForSalePriceUsd() != null ? in.shipForSalePriceUsd() : getShipForSalePriceUsd()` (giá trị hiện có của dòng) ở bước dựng record, không sửa `applyShipForm`. Hai ca test `giữ_shipForSalePriceUsd_khi_không_gửi` / `ghi_shipForSalePriceUsd_khi_có_gửi` của B05 kiểm qua `applyForm`.
