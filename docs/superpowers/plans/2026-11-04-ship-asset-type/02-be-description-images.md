# 02 — Ảnh tàu và API Mô tả tàu thủy (B02–B03)

> Đọc `00-index.md` trước (hợp đồng, quy ước tên, Global Constraints, "Khác hệ thống cũ", Review Focus), rồi `surveys/legacy-be.md` mục 2, 4, 5, 6 và `surveys/new-code.md` mục "Tab". Subagent: `model: "sonnet"`, bắt buộc spec review rồi code review.
>
> **Thứ tự:** cả hai task thuộc **đợt B2**, chạy sau B01 (`01-be-foundation.md`), song song với B04, B05, B06, B08, B10. **Tập file rời nhau:**
> - B02 chỉ sửa `AppraisalImageService` và `AppraisalImageServiceTest`.
> - B03 tạo `ShipDescriptionController`, `ShipDescriptionService`, `ShipDescriptionInput`, `ShipDescriptionData`, sửa `AppraisalShipDescription` (thêm `apply`) và `LandDescriptionService` (kèm test).
> - Cả hai **không chạm** `GroupCompareService`, `ReportLegalService` và các lớp vòng đời (`GroupAssetTypeReset`, `ReportGroupResolver`, `ReportGroupDetailsWriter`...).
>
> Ký hiệu: **L** = legacy `td-project-legacy` @ `82193fe5` (gốc `src/main/java/com/jvb/thanhdo/`), **N** = `backend/src/main/java/com/thanhdo/tdprojectbe/`, **NT** = `backend/src/test/java/com/thanhdo/tdprojectbe/`. Mọi lệnh git chạy bằng `git -C backend …`; **không push**.

## Điều kiện tiên quyết từ B01 (kiểm tra ở Step 0 của từng task)

B01 phải đã merge vào `feat/ship-asset-type`. Hai task này dùng các thứ sau; **thiếu thứ nào thì dừng và báo lại, không tự thêm** (file đó thuộc B01):

| Thành phần | Chữ ký / ghi chú giả định (đối chiếu với khối Interfaces của B01) |
|---|---|
| `common/constant/AppraisalAssetType.SHIP`, `CommonConstants.AppraisalAssetType.SHIP` | `SHIP = 4`, kiểu `int` như `CAR` |
| `model/entity/AppraisalShipDescription` | `Long getId()`, `Long getAppraisalReportId()`, `Integer getDeleteFlag()`, hằng `NOT_DELETED = 0`, `static empty(Long reportId, Long groupId, LocalDateTime now)`; tên field Java **trùng tên field của `UpdateAppraisalShipDescriptionRequest`** (L), `@Getter` cho mọi field; `getConditionDetails()` / `getQualityEvaluationDetails()` trả `List<…>` (có thể null) |
| `model/dto/ShipConditionDetail`, `ShipQualityEvaluationItem` | record, `@JsonIgnoreProperties(ignoreUnknown = true)`; `ShipConditionDetail(String itemName, String conditionDescription)`; `ShipQualityEvaluationItem(List<String> itemNames, BigDecimal importanceWeight, BigDecimal standardValue, String rating, BigDecimal relativeValue)` (thứ tự như `AllArgsConstructor` của L) |
| `repository/AppraisalShipDescriptionRepository` | `Optional<AppraisalShipDescription> findByAppraisalAssetGroupId(Long)` |
| `service/ship/ShipQuality` | `static BigDecimal remainingQualityPct(List<ShipQualityEvaluationItem>)` (null-safe, trả null khi không tính được hoặc ngoài 0–100), `formatDimension(BigDecimal, BigDecimal)` |
| `AppraisalGroupCompare.applyShipSync` | `void applyShipSync(AppraisalShipDescription ship, String address, BigDecimal remainingQualityPct)`: tự ghi 7 trường từ `ship` (năm đóng, nơi đóng, công dụng, cấp, trọng tải, kích thước ghép từ `dimLengthMax`/`dimBreadthMax`, công suất) cộng `address` và CLCL; **chỉ ghi khi nguồn khác null** |
| `AppraisalImage` (phần B02 dùng) | cột `appraisal_ship_description_id` + `Long getAppraisalShipDescriptionId()`, `static ofShip(Long shipDescriptionId, int type, String note, String name, String path, String groupUuid, LocalDateTime now)` (giống `ofVehicle`) |
| `AppraisalImageRepository` | `List<AppraisalImage> findByAppraisalShipDescriptionIdAndDeletedOrderByCreatedAscIdAsc(Long shipDescriptionId, Integer deleted)` |

> **Lưu ý `apply`:** `00-index.md` liệt kê `AppraisalShipDescription.apply(ShipDescriptionInput)` trong nhóm phương thức entity do B01 tạo, nhưng `ShipDescriptionInput` do **B03** tạo nên B01 không thể biên dịch được. Kế hoạch này giao `apply` cho **B03** (xem mục "Vấn đề cần controller quyết định" cuối file). B01 chỉ được khai báo `empty`, `resetToDefault`, `softDelete`, `revive`.

---

# Task B02 — Ảnh tàu trong `AppraisalImageService`

## Hành vi hệ thống cũ

Nguồn: `L service/AppraisalImageService.java`, `L controller/api/AppraisalImageApiController.java`, `L common/ReportConstants.java` (`AppraisalImageType`).

- Ảnh tàu là ảnh của **mô tả tàu** (`appraisal_image.appraisal_ship_description_id`). Lấy ảnh: `findByAppraisalShipDescriptionId` (L `AppraisalImageService.java:53-55`, repository `findByAppraisalShipDescriptionIdAndDeletedOrderByCreatedAscIdAsc(id, 0)`), API `GET /api/appraisal-images/ship-description/{id}` (L `AppraisalImageApiController.java:41-44`).
- Upload (L `AppraisalImageService.java:97-132`): request mang một trong ba owner (đất, xe, tàu); `ownerCount != 1` thì ném `IllegalArgumentException("Phải chỉ định đúng một mô tả đất, phương tiện hoặc tàu thủy")`. Thư mục S3: `{S3_ASSET_FOLDER}{land|vehicle|ship}/{reportId}/{descId}/{uuid}/{fileName}` (dòng 128-133). Mỗi lần upload chung một `group_uuid`.
- Loại ảnh tàu 21–30 (L `ReportConstants.AppraisalImageType`, dòng 52-61): 21 Mặt trước tàu, 22 Đầu tàu, 23 Đuôi tàu, 24 Mạn tàu, 25 Sàn tàu, 26 Số IMO/Số kiểm soát, 27 Hệ thống neo, 28 Phòng lái/phòng nghỉ, 29 Khoang hàng, 30 Hệ thống máy của động cơ. Mục "Khác" dùng mã 10 chung. L không kiểm tra khoảng giá trị của `type`.

## Khác hệ thống cũ (liên quan)

- **#3 của index:** lỗi trả 400 qua `BusinessException`. Cũ ném `IllegalArgumentException`; mới owner **suy ra từ nhóm** (không nhận từ client) nên không còn lỗi "sai số owner".
- **#1 của index:** `deleteAllOfShipDescription` là hàm B07 gọi khi xóa nhóm hoặc đổi loại rời SHIP.
- **Bổ sung (không có trong danh sách #1–#5, controller xác nhận khi review):** loại ảnh 21–30 **chỉ nhận cho nhóm SHIP**; đất, căn hộ, ô tô vẫn chỉ nhận 1–20 trừ 17 như hiện nay (không nới cho loại khác).
- Loại tài sản lạ (không phải 1–4) trả 400 thay vì ngầm hiểu là đất (xóa `else` ngầm hiểu, Review Focus #1).

### Task B02: Ảnh tàu theo nhóm

**Files:**
- Modify: `N/service/AppraisalImageService.java` (trên `origin/develop`: `isCar` ở dòng ~210, `list` ~69, `add` ~88, `belongsToReport` ~169, `validate` ~181; grep lại trước khi sửa).
- Modify: `NT/service/AppraisalImageServiceTest.java`.
- **Không sửa:** `N/controller/AppraisalImageController.java` (đường dẫn `…/groups/{groupId}/images` và `…/images/{imageId}` không đổi; owner suy từ nhóm; quyền nằm ở `SecurityConfig` nhóm `/api/appraisal-records/**`), `NT/controller/AppraisalImageControllerTest.java`, `ComparisonImageService`/`ConstructionImageService` (chỉ dùng `ASSET_FOLDER`, `NO_IMAGE_MESSAGE`, `checkedName`: **giữ nguyên chữ ký và độ hiển thị package**).

**Interfaces:**
- Produces:
  - Constructor mới (thứ tự field, vì `@RequiredArgsConstructor`):
    `AppraisalImageService(ReportEditScope scope, AppraisalImageRepository images, AppraisalLandDescriptionRepository landDescriptions, AppraisalVehicleDescriptionRepository vehicleDescriptions, AppraisalShipDescriptionRepository shipDescriptions, UnreferencedImageKeys unreferencedKeys, S3StorageService storage, S3LinkService links, Clock clock)`
    (field `shipDescriptions` đặt ngay sau `vehicleDescriptions`; `AppraisalImageServiceTest.setUp` là nơi duy nhất gọi `new`).
  - `@Transactional public void deleteAllOfShipDescription(long shipDescriptionId)`: xóa mềm mọi ảnh chưa xóa của mô tả tàu rồi xóa S3 (sau commit) các khóa không còn ảnh nào dùng; dùng chung `deleteAll(List)` với hai hàm cùng loại. B07 gọi.
  - Hằng package-private mới: `static final int MAX_SHIP_TYPE = 30;`, `static final String ASSET_TYPE_UNSUPPORTED_MESSAGE = "Loại tài sản này không có ảnh mô tả";`. `MAX_TYPE = 20`, `UNUSED_TYPE = 17` và mọi `*_MESSAGE` khác **giữ nguyên**.
- Behavior:
  - `list`/`add`: rẽ **ba nhánh tường minh** theo loại nhóm: LAND và APARTMENT dùng mô tả đất (thư mục `land`), CAR dùng mô tả phương tiện (`vehicle`), SHIP dùng mô tả tàu (`ship`); loại khác ném `BusinessException(VALIDATION_ERROR, ASSET_TYPE_UNSUPPORTED_MESSAGE)`. Không còn `isCar`/`else`.
  - SHIP: mô tả tàu phải tồn tại **và `deleteFlag == NOT_DELETED`**, ngược lại `add` trả 400 `DESCRIPTION_MISSING_MESSAGE` ("Vui lòng lưu mô tả trước khi thêm ảnh"), `list` trả rỗng. (Đất/ô tô giữ nguyên: không lọc `deleteFlag`.)
  - `validate(type, note, kind)`: khoảng loại hợp lệ là `MIN_TYPE..MAX_TYPE` trừ `UNUSED_TYPE`; riêng nhóm SHIP là `MIN_TYPE..MAX_SHIP_TYPE` trừ `UNUSED_TYPE`.
  - `belongsToReport`: thêm nhánh `getAppraisalShipDescriptionId() != null` tra `shipDescriptions.findById(...)` rồi so `getAppraisalReportId() == reportId`.
  - Khóa S3: `ASSET_FOLDER + "ship/" + reportId + "/" + descId + "/" + uuid + "/" + fileName`; tạo ảnh bằng `AppraisalImage.ofShip(...)`.

**Gợi ý cấu trúc (không bắt buộc, miễn đúng hành vi):** enum private `Kind { LAND("land"), VEHICLE("vehicle"), SHIP("ship") }` có field `folder`; `private static Kind kindOf(AppraisalAssetGroup)` dùng `switch` trên `int type = group.getAssetType()` (`LAND, APARTMENT -> LAND`, `CAR -> VEHICLE`, `SHIP -> SHIP`, `default -> throw`); `descriptionIdOf(Kind, long groupId)` trả `Optional<Long>` (SHIP lọc `deleteFlag`); `imagesOf(Kind, long descId)`; `newImage(Kind, …)` dùng `switch` đầy đủ, không `default` ngầm.

- [ ] **Step 0: Kiểm tra điều kiện tiên quyết B01.**
  ```bash
  git -C backend grep -n "ofShip\|appraisalShipDescriptionId" HEAD -- src/main/java/com/thanhdo/tdprojectbe/model/entity/AppraisalImage.java
  git -C backend grep -n "findByAppraisalShipDescriptionIdAndDeletedOrderByCreatedAscIdAsc" HEAD -- src/main/java/com/thanhdo/tdprojectbe/repository/AppraisalImageRepository.java
  git -C backend grep -n "SHIP" HEAD -- src/main/java/com/thanhdo/tdprojectbe/common/constant/AppraisalAssetType.java
  git -C backend ls-files src/main/java/com/thanhdo/tdprojectbe/repository/AppraisalShipDescriptionRepository.java src/main/java/com/thanhdo/tdprojectbe/model/entity/AppraisalShipDescription.java
  ```
  Thiếu kết quả nào thì **dừng, báo lại**. Đọc nguyên `AppraisalImageService.java` và `AppraisalImageServiceTest.java` hiện tại, và `L service/AppraisalImageService.java:90-140` để so với mô tả ở trên.

- [ ] **Step 1: Viết test FAIL** trong `NT/service/AppraisalImageServiceTest.java`:
  - Sửa `setUp`: thêm `@Mock AppraisalShipDescriptionRepository shipDescriptions;` và truyền vào constructor theo thứ tự mới (lúc này **không biên dịch được** vì service chưa có tham số; đó là trạng thái FAIL mong đợi).
  - Thêm helper `givenShipDescription(int deleteFlag)`: tạo `AppraisalShipDescription.empty(REPORT_ID, GROUP_ID, now)`, `ReflectionTestUtils.setField(description, "id", 88L)`, đặt `deleteFlag` nếu khác 0, stub `shipDescriptions.findByAppraisalAssetGroupId(GROUP_ID)`.
  - Test mới (mỗi tên là một `@Test`):
    1. `tàu_gắn_ship_description_và_thư_mục_ship`: nhóm SHIP, `add(…, type 21, "Mặt trước", [a.jpg, b.png])` → mọi ảnh lưu có `getAppraisalShipDescriptionId() == 88`, `getAppraisalLandDescriptionId()` và `getAppraisalVehicleDescriptionId()` đều null, `getPath()` bắt đầu `report/ship/11/88/` và kết thúc `/a.jpg`, cùng `groupUuid` và ghi chú; `views.get(0).url()` bắt đầu `https://cdn/report/ship/11/88/`; `verify(landDescriptions, never()).findByAppraisalAssetGroupId(any())` và tương tự cho `vehicleDescriptions`.
    2. `tàu_nhận_loại_10_và_21_đến_30`: vòng lặp qua `{10, 21, …, 30}`, mỗi loại `add` thành công, `getType()` đúng.
    3. `tàu_từ_chối_loại_17_0_và_31`: `TYPE_INVALID_MESSAGE`, `verify(storage, never()).upload(...)`.
    4. `đất_căn_hộ_ô_tô_vẫn_từ_chối_loại_21_và_30`: lần lượt LAND, APARTMENT, CAR với type 21 và 30 → `TYPE_INVALID_MESSAGE`, `verifyNoInteractions(shipDescriptions)`, không upload. Test cũ `loại_17_và_ngoài_khoảng_bị_từ_chối` (`{17, 0, 21}` trên LAND) **giữ nguyên và vẫn phải PASS**.
    5. `tàu_chưa_có_mô_tả_trả_400`: `shipDescriptions` trả `Optional.empty()` → `VALIDATION_ERROR`, message "Vui lòng lưu mô tả trước khi thêm ảnh", không upload.
    6. `tàu_mô_tả_đã_xóa_mềm_trả_400`: mô tả có `deleteFlag = 1`, kết quả như trên.
    7. `danh_sách_tàu_đọc_theo_mô_tả_tàu`: stub `images.findByAppraisalShipDescriptionIdAndDeletedOrderByCreatedAscIdAsc(88L, NOT_DELETED)` trả một `AppraisalImage.ofShip(88L, 26, "n", "imo.jpg", "report/ship/k.jpg", "g", …)` → view có `type 26`, `url "https://cdn/report/ship/k.jpg"`.
    8. `danh_sách_tàu_rỗng_khi_chưa_có_mô_tả_hoặc_đã_xóa`: hai tình huống như 5, 6 → `list` rỗng và không gọi finder ảnh ship.
    9. `loại_tài_sản_lạ_trả_400`: `givenGroup(99)` → cả `list` lẫn `add` ném `ASSET_TYPE_UNSUPPORTED_MESSAGE`, `VALIDATION_ERROR`.
    10. `xóa_ảnh_tàu_thuộc_báo_cáo`: ảnh `ofShip(88L, …)`; `shipDescriptions.findById(88L)` trả mô tả có `reportId 11` → `markDeleted`, `saveAndFlush`, `unreferencedKeys.delete(Set.of(path))`.
    11. `ảnh_tàu_báo_cáo_khác_404`: mô tả có `reportId 999` → `NOT_FOUND`, ảnh vẫn `NOT_DELETED`, `storage.delete` không được gọi.
    12. `deleteAllOfShipDescription_xóa_mềm_mọi_ảnh_và_dọn_khóa_s3`: hai ảnh `ofShip` → cả hai `DELETED`, `verify(images).saveAllAndFlush(List.of(a, b))`, `verify(unreferencedKeys).delete(List.of("k/a.jpg", "k/b.jpg"))`.
    13. `deleteAllOfShipDescription_không_có_ảnh_thì_không_ghi`: finder trả rỗng → `verify(images, never()).saveAllAndFlush(any())`, `verify(unreferencedKeys, never()).delete(any())`.
    14. `tàu_s3_lỗi_giữa_chừng_xóa_lại_ảnh_đã_lên`: `doNothing().doThrow(IllegalStateException).when(storage).upload(...)` → `INTERNAL_ERROR`, `storage.delete` được gọi một lần với khóa kết thúc `/a.jpg`.
    15. `đất_và_ô_tô_không_đụng_kho_mô_tả_tàu`: chạy lại hai luồng cũ (một `add` LAND, một `add` CAR) rồi `verifyNoInteractions(shipDescriptions)`.
  - Lệnh (kỳ vọng FAIL, lỗi biên dịch constructor):
    ```bash
    cd backend && ./mvnw -q test -Dtest=AppraisalImageServiceTest
    ```

- [ ] **Step 2: Cài đặt** trong `N/service/AppraisalImageService.java` đúng Interfaces/Behavior ở trên. Cập nhật Javadoc lớp: thêm "nhóm tàu thủy gắn vào mô tả tàu (thư mục S3 `ship`)" và các điểm **Khác hệ thống cũ** ở trên (owner suy từ nhóm, loại 21–30 chỉ cho SHIP, loại tài sản lạ trả 400). Không đổi API controller. Giữ nguyên thông điệp lỗi cũ.

- [ ] **Step 3: Chạy test, PASS.**
  ```bash
  cd backend && ./mvnw -q test -Dtest=AppraisalImageServiceTest
  ```
  Kỳ vọng: toàn bộ test cũ (đất, ô tô) và 15 test mới PASS.

- [ ] **Step 4: Hồi quy các lớp dùng chung.**
  ```bash
  cd backend && ./mvnw -q test -Dtest=AppraisalImageControllerTest,GroupAssetTypeResetTest,ComparisonImageServiceTest,ConstructionImageServiceTest
  ```
  (Bỏ tên test nào không tồn tại trên nhánh. `AppraisalReportBasicEditIntegrationTest` cần MySQL; chạy nếu có DB.) Grep `new AppraisalImageService(` trong `src` phải chỉ còn `AppraisalImageServiceTest`.

- [ ] **Step 5: Commit.**
  ```bash
  git -C backend add src/main/java/com/thanhdo/tdprojectbe/service/AppraisalImageService.java src/test/java/com/thanhdo/tdprojectbe/service/AppraisalImageServiceTest.java
  git -C backend commit -m "feat: ảnh mô tả cho nhóm tàu thủy"
  ```

---

# Task B03 — API Mô tả tàu thủy + đồng bộ so sánh type 0 + chặn SHIP ở mô tả đất

## Hành vi hệ thống cũ

Nguồn: `L controller/api/AppraisalShipDescriptionApiController.java`, `L service/AppraisalShipDescriptionService.java`, `L entity/dto/request/UpdateAppraisalShipDescriptionRequest.java`, `L service/AppraisalGroupCompareService.java:68` (`joinAssetAddresses`).

- `PUT /api/appraisal-ship-descriptions/{id}` (L controller): body `@Valid UpdateAppraisalShipDescriptionRequest`; thành công 200 "Lưu mô tả tàu thủy thành công"; không tìm thấy 404 "Không tìm thấy mô tả tàu thủy"; lỗi validate 400 với message của field lỗi đầu tiên.
- `AppraisalShipDescriptionService.update` (L `service/AppraisalShipDescriptionService.java:86-255`):
  - **ghi đè mọi field**, kể cả null; chuỗi qua `trimToNull` (dòng 90-240); số, boolean và ngày gán thẳng (không làm tròn);
  - `conditionDetails` và `qualityEvaluationDetails` **chỉ ghi khi request khác null** ("Form chưa gửi tình trạng thì giữ nguyên", dòng 243-249) và đi qua chuẩn hóa;
  - lưu xong gọi `syncToMainAssetCompare` (dòng 252).
- Chuẩn hóa tình trạng (dòng 430-446): trim `itemName` và `conditionDescription`, **bỏ dòng có cả hai rỗng**, phần tử null bị bỏ.
- Chuẩn hóa đánh giá chất lượng (dòng 392-428): trim từng tên trong `itemNames`, **bỏ tên rỗng**, `itemNames` null thành danh sách rỗng, trim `rating`; **bỏ item rỗng hoàn toàn** (không tên và 4 trường số/đánh giá đều null); item null bị bỏ; ba số giữ nguyên.
- `syncToMainAssetCompare` (dòng 284-309): đích là dòng so sánh `type = 0` chưa xóa của nhóm; không có thì bỏ qua. `applyIfPresent` (dòng 341): chỉ ghi khi nguồn khác null. Bản đồ: `buildYear→shipBuildYear`, `buildPlace→shipBuildPlace`, `shipTypePurpose→shipUsagePurpose`, `shipClass→shipClass`, `deadweight→shipDeadweight`, `formatDimension(dimLengthMax, dimBreadthMax)→shipDimension`, `mainEnginePowerHp→shipMainEnginePowerHp`, `joinAssetAddresses(tài sản nhóm)→address`, CLCL→`shipRemainingQualityPct`.
- CLCL (dòng 313-345): mỗi item đủ `importanceWeight`, `relativeValue`, `standardValue > 0` đóng góp `w × (r ÷ s, scale 10, HALF_UP) × 100`; tổng ngoài [0, 100] hoặc không item hợp lệ thì **null (không ghi)**; ngược lại `setScale(2, HALF_UP)`. (Lớp thuần do B01 viết: `ShipQuality.remainingQualityPct`.)
- `formatDimension` (dòng 348-356): `"{L}m x {B}m"`, mỗi số `stripTrailingZeros().toPlainString()` rồi đổi `.` thành `,`; thiếu một số thì null. (B01: `ShipQuality.formatDimension`, được `applyShipSync` gọi.)
- `joinAssetAddresses` (L `AppraisalGroupCompareService.java:68`): ghép `asset` của tài sản `deleteFlag == 0` bằng `\n`, trim từng dòng, bỏ dòng rỗng, rỗng hết thì null.
- Validate: toàn bộ `UpdateAppraisalShipDescriptionRequest` (155 field, mục dưới).

## Khác hệ thống cũ (cố ý)

- **#2 của index:** PUT trả `ShipDescriptionData` (`id` mô tả, mọi field đã lưu, `remainingQualityPct` do server tính). Cũ trả `AppraisalAssetGroupShipSyncResponse {id, appraisalGroupCompare}` để vá DOM tab So sánh; ở FE mới tab So sánh bị hủy khi ẩn nên tự tải lại khi mở.
- **#3 của index:** lỗi qua `BusinessException`/validation 400; cũ `NoSuchElementException` chuyển 404 thủ công.
- Tra theo **nhóm** (`/groups/{groupId}/ship-description`, qua `ReportEditScope`) thay vì mã mô tả; nhóm không phải tàu thủy trả 400; mô tả chưa có hoặc đã xóa mềm trả 404 "Chưa có mô tả tàu thủy".
- Địa chỉ đồng bộ lấy từ repository tài sản của nhóm (`GroupAssetTypeReset.joinAssetAddresses`), không đi qua quan hệ JPA như cũ.
- `LandDescriptionService.requireLandGroup` **chặn SHIP** (cũ không có tầng này; nếu không chặn, SHIP lọt vào nhánh đất; Review Focus #1).

## Quy tắc ràng buộc của `ShipDescriptionInput` (copy từ L, 155 trường)

Nguồn duy nhất: `L entity/dto/request/UpdateAppraisalShipDescriptionRequest.java` @ `82193fe5`. Hằng của L: `NON_NEGATIVE_MESSAGE = "không được âm"`, `DECIMAL_MESSAGE = "tối đa 8 chữ số nguyên và 2 chữ số thập phân"`, `DECIMAL_LARGE_MESSAGE = "tối đa 10 chữ số nguyên và 2 chữ số thập phân"`.

Mẫu message (nhãn ở bảng dưới, ghép nguyên văn):
- `@Size(max = N)`: `"<nhãn> không được vượt quá N ký tự"`.
- `@PositiveOrZero`: `"<nhãn> không được âm"`.
- `@Digits(integer = 8|10, fraction = 2)`: `"<nhãn> tối đa 8 chữ số nguyên và 2 chữ số thập phân"` (hoặc `10 chữ số nguyên`). Mỗi `BigDecimal` mang **cả** `@PositiveOrZero` lẫn `@Digits`.
- Boolean (39 trường) và `LocalDate` (3 trường: `inspectionPeriodicDate`, `inspectionDockDate`, `inspectionAnnualDate`): không ràng buộc.
- Hai danh sách `conditionDetails`, `qualityEvaluationDetails`: `@Valid` (cascade vào ràng buộc trên record của B01: `itemName` `@Size(500)` "Hạng mục không được vượt quá 500 ký tự"; mỗi tên trong `itemNames` `@Size(500)` "Hạng mục thẩm định không được vượt quá 500 ký tự"; ba số `@PositiveOrZero` + `@Digits(8,2)`; `rating` `@Size(50)` "Đánh giá không được vượt quá 50 ký tự").
- Giới hạn khớp DDL cuối (xem `00-index.md`).

#### Bảng String

| Field | `@Size(max)` | Nhãn trong message |
|---|---|---|
| `shipName` | 500 | Tên phương tiện |
| `registrationNumber` | 255 | Số đăng ký |
| `shipTypePurpose` | 500 | Công dụng |
| `registryNumber` | 255 | Số đăng kiểm |
| `ownerName` | 500 | Chủ phương tiện |
| `buildPlace` | 1000 | Nơi đóng |
| `mooringLocation` | 255 | Nơi neo đậu |
| `material` | 255 | Vật liệu |
| `shipClass` | 255 | Cấp phương tiện |
| `operatingArea` | 255 | Vùng hoạt động |
| `operationNote` | 5000 | Ghi chú hoạt động |
| `designSymbol` | 255 | Ký hiệu thiết kế |
| `designCertificateNumber` | 255 | Số GCN |
| `appraisalNumber` | 255 | Số thẩm định |
| `conversionYearPlace` | 500 | Năm và nơi hoán cải |
| `anchorBowType` | 100 | Neo mũi - kiểu |
| `anchorSternType` | 100 | Neo đuôi - kiểu |
| `anchorSternTotalWeightKg` | 1000 | Neo đuôi - tổng khối lượng (kg) |
| `anchorChainCable` | 255 | Chiều dài/đường kính xích, cáp |
| `anchorWindlassType` | 100 | Kiểu tời neo |
| `steeringRudderType` | 100 | Bánh lái - kiểu |
| `steeringMainGearType` | 100 | Máy lái chính - kiểu |
| `steeringBackupGearType` | 100 | Máy lái dự phòng - kiểu |
| `mooringBollardType` | 100 | Cột buộc dây - kiểu |
| `mooringBollardQuantity` | 1000 | Cột buộc dây - số lượng (chiếc) |
| `towingBollardType` | 1000 | Kiểu cột kéo |
| `towingHookType` | 1000 | Kiểu mọc kéo |
| `towingWinchType` | 1000 | Kiểu tời thu dây |
| `towingRopeType` | 1000 | Loại dây kéo |
| `mainEngineBrand` | 255 | Máy chính - mác máy |
| `mainEngineSerialNumber` | 255 | Máy chính - số chế tạo |
| `auxEngineBrand` | 255 | Máy phụ - mác máy |
| `electricGenerator` | 100 | Máy phát |
| `electricGeneratorPowerKva` | 100 | Máy phát - công suất (KVA) |
| `electricGeneratorVoltage` | 255 | Máy phát - điện áp (V) |
| `electricBatteryTotalCapacity` | 255 | Ắc quy - tổng dung lượng (Ah) |
| `compressedAirTankQuantity` | 1000 | Bình khí nén (chiếc) |
| `compressedAirCompressorQuantity` | 1000 | Máy nén khí (chiếc) |
| `liftingSerialNumber` | 255 | Số chế tạo hoặc số quản lý |
| `liftingMaxReachCapacity` | 255 | Tầm với lớn nhất (m)/sức nâng (tấn) |
| `liftingMinReachCapacity` | 255 | Tầm với nhỏ nhất (m)/sức nâng (tấn) |
| `fireGasType` | 100 | Loại khí |
| `fireFoamType` | 100 | Loại bọt |
| `fireFoamTotalVolumeL` | 1000 | Bọt - tổng thể tích (lít) |
| `fireFoamPusherQuantity` | 1000 | Bình bọt đẩy - số lượng |
| `fireFoamPusherVolumeL` | 1000 | Bình bọt đẩy - dung tích (lít) |
| `firePortableFoam` | 100 | Thiết bị tạo bọt xách tay - số lượng |
| `fireFoamBottleQuantity` | 1000 | Bình bọt - số bình |
| `fireFoamBottleVolumeL` | 1000 | Bình bọt - dung tích (lít) |
| `lifesavingRingBuoy` | 100 | Phao tròn (chiếc): không dây; có dây |
| `lifesavingLifeJacket` | 100 | Phao áo (chiếc); phao áo trẻ em |
| `damageControlTools` | 100 | Bộ dụng cụ cứu đắm |
| `damageControlBilgeAlarm` | 100 | Thiết bị tự động báo mức nước đáy khoang |
| `signalSound` | 100 | Âm hiệu |
| `navigationRadio` | 100 | Thiết bị VTĐ (MF/HF; VHF) |
| `navigationAis` | 100 | AIS (cấp A; cấp B) |
| `navigationSart` | 100 | SART/AIS-SART |
| `pollutionOilyWaterSeparatorQuantity` | 1000 | Thiết bị phân ly dầu nước - số lượng (chiếc) |
| `pollutionOilyWaterSeparatorTotalOutput` | 1000 | Thiết bị phân ly dầu nước - tổng sản lượng |
| `pollutionSewageTankQuantity` | 1000 | Két nước thải - số lượng (chiếc) |
| `pollutionSewageTankTotalVolumeM3` | 1000 | Két nước thải - tổng thể tích (m3) |
| `pollutionSettlingTankQuantity` | 1000 | Két lắng - số lượng (chiếc) |
| `pollutionSettlingTankTotalVolumeM3` | 1000 | Két lắng - tổng thể tích (m3) |
| `pollutionSludgeTankQuantity` | 1000 | Két dầu bẩn - số lượng (chiếc) |
| `pollutionSludgeTankTotalVolumeM3` | 1000 | Két dầu bẩn - tổng thể tích (m3) |
| `pollutionStandardFlangeQuantity` | 1000 | Bích nối tiêu chuẩn - số lượng (chiếc) |
| `pollutionStandardFlangePosition` | 255 | Bích nối tiêu chuẩn - vị trí |
| `pollutionGarbageContainerQuantity` | 1000 | Thiết bị chứa rác - số lượng (chiếc) |
| `pollutionSewageTreatmentQuantity` | 1000 | Thiết bị xử lý nước thải - số lượng (chiếc) |
| `pollutionSewageTreatmentOutputM3` | 1000 | Thiết bị xử lý nước thải - sản lượng (m3) |

#### Bảng Integer

| Field | Nhãn trong message |
|---|---|
| `buildYear` | Năm đóng |
| `grossTonnage` | Tổng dung tích (GT) |
| `designYear` | Năm thiết kế |
| `anchorBowQuantity` | Neo mũi - số lượng (chiếc) |
| `anchorSternQuantity` | Neo đuôi - số lượng (chiếc) |
| `steeringRudderQuantity` | Bánh lái - số lượng (chiếc) |
| `mooringRopeQuantity` | Dây buộc - số lượng (chiếc) |
| `mainEngineQuantity` | Máy chính - số lượng (chiếc) |
| `mainEngineManufactureYear` | Máy chính - năm chế tạo |
| `auxEngineQuantity` | Máy phụ - số lượng (chiếc) |
| `electricBatteryQuantity` | Ắc quy - số bình |
| `firePumpQuantity` | Bơm chữa cháy - số lượng (chiếc) |
| `fireCo2BottleQuantity` | Bình CO2 - số bình |
| `firePowderBottleQuantity` | Bình bột - số bình |
| `lifesavingRaftBoatQuantity` | DC nổi/phao bè/Xuồng - số lượng (chiếc) |
| `lifesavingRaftBoatTotalCapacity` | DC nổi/phao bè/Xuồng - tổng sức chở (người) |
| `damageControlBilgePumpQuantity` | Bơm hút khô - số lượng (chiếc) |
| `pollutionOilyWaterTankQuantity` | Két thu hồi hỗn hợp dầu nước - số lượng (chiếc) |

#### Bảng BigDecimal

| Field | `@Digits(integer=…)` | Nhãn trong message |
|---|---|---|
| `dimLength` | 8 | L - chiều dài thiết kế (m) |
| `dimBreadth` | 8 | B - chiều rộng thiết kế (m) |
| `dimDepth` | 8 | D - chiều cao mạn (m) |
| `dimDraft` | 8 | d - chiều chìm thiết kế (m) |
| `dimLengthMax` | 8 | Lmax - chiều dài lớn nhất (m) |
| `dimBreadthMax` | 8 | Bmax - chiều rộng lớn nhất (m) |
| `freeBoard` | 8 | Mạn khô F (mm) |
| `cargoCapacity` | 10 | Khả năng khai thác - lượng hàng (tấn) |
| `deadweight` | 10 | Khả năng khai thác - TTTP (tấn) |
| `anchorBowTotalWeightKg` | 10 | Neo mũi - tổng khối lượng (kg) |
| `steeringRudderTotalAreaM2` | 10 | Bánh lái - tổng diện tích (m2) |
| `steeringMainGearTorqueTm` | 10 | Máy lái chính - mô men lái (tm) |
| `steeringBackupGearTorqueTm` | 10 | Máy lái dự phòng - mô men lái (tm) |
| `mooringRopeTotalLengthM` | 10 | Dây buộc - tổng chiều dài (m) |
| `mainEnginePowerHp` | 8 | Máy chính - công suất (sức ngựa) |
| `auxEnginePowerHp` | 8 | Máy phụ - tổng công suất (sức ngựa) |
| `electricBatteryVoltageV` | 8 | Ắc quy - điện áp (V) |
| `firePumpTotalFlowM3` | 10 | Bơm chữa cháy - tổng lưu lượng (m3) |
| `fireGasTotalWeightKg` | 10 | Khí - tổng khối lượng (kg) |
| `fireCo2BottleWeightKg` | 10 | Bình CO2 - khối lượng (kg) |
| `firePowderBottleWeightKg` | 10 | Bình bột - khối lượng (kg) |
| `damageControlBilgePumpTotalFlowM3` | 10 | Bơm hút khô - tổng lưu lượng (m3) |
| `pollutionOilyWaterTankTotalVolumeM3` | 10 | Két thu hồi hỗn hợp dầu nước - tổng thể tích (m3) |

#### Boolean (39)

`fireManualTools`, `fireLayoutPlan`, `signalLightSideRed`, `signalLightSideGreen`, `signalLightSternWhite`, `signalLightMastWhite`, `signalLightMastGreen`, `signalLightMastRed`, `signalLightAnchor`, `signalLightTowingRed`, `signalLightTowingGreen`, `signalLightYellow`, `signalLightFlashWhite`, `signalLightFlashYellow`, `signalLightFlashRed`, `signalLightOther`, `signalLightHalfGreenRed`, `signalLightSpare`, `signalShapeBlackTriangle`, `signalShapeWhiteTriangle`, `signalShapeBlackSquare`, `signalShapeBlackRectangle`, `signalShapeBlackCircle`, `signalShapeWhiteCircle`, `signalShapeBlackDiamond`, `signalShapeFlagSet`, `signalShapeParachuteFlare`, `signalShapeLineThrower`, `navigationBinoculars`, `navigationClock`, `navigationDepthSounder`, `navigationOrderBell`, `navigationInclinometer`, `navigationDraftGauge`, `navigationGps`, `navigationCompass`, `navigationCommandPa`, `navigationRadar`, `navigationEpirb`


### Công cụ sinh mã: `gen_ship_dto.py`

Để không gõ tay 155 trường, implementer tạo script **tạm** (không commit) dưới và chạy:
```bash
git -C td-project-legacy show 82193fe5:src/main/java/com/jvb/thanhdo/entity/dto/request/UpdateAppraisalShipDescriptionRequest.java | python gen_ship_dto.py <mode>
```
- `record`: các component của `ShipDescriptionInput` (giữ nguyên annotation và thứ tự của L, đã nối sẵn hằng thành chuỗi literal).
- `data-record`: các component của `ShipDescriptionData` (`long id`, rồi mọi trường không annotation, rồi `BigDecimal remainingQualityPct`).
- `data-from`: danh sách đối số getter cho `ShipDescriptionData.from(...)` (hai danh sách qua `orEmpty`).
- `tsv`: nội dung file `backend/src/test/resources/ship/ship-description-constraints.tsv`.

```python
#!/usr/bin/env python3
"""Sinh mã khung từ UpdateAppraisalShipDescriptionRequest (legacy @82193fe5) để tránh gõ tay 155 trường.
Dùng: git -C td-project-legacy show 82193fe5:src/main/java/com/jvb/thanhdo/entity/dto/request/UpdateAppraisalShipDescriptionRequest.java | python gen_ship_dto.py <mode>
mode: record | data-record | data-from | tsv
"""
import re
import sys

src = sys.stdin.read()
consts = dict(re.findall(r'private static final String (\w+) = "([^"]*)";', src))
body = src.split("public class")[1]
field = re.compile(r'((?:\s*@[^\n]*\n)*)\s*private\s+(?!static)([\w<>]+)\s+(\w+);')


def expand(annotation):
    """Thay `"abc " + CONST` bằng chuỗi literal đã nối (record không dùng được hằng của lớp legacy)."""
    def join(m):
        parts = re.findall(r'"([^"]*)"|(\w+)', m.group(1))
        text = "".join(lit if lit else consts[name] for lit, name in parts if lit or name in consts)
        return 'message = "' + text + '"'
    return re.sub(r'message = ((?:"[^"]*"|\w+)(?:\s*\+\s*(?:"[^"]*"|\w+))*)', join, annotation)


rows = []
for m in field.finditer(body):
    ann, typ, name = m.groups()
    anns = [expand(a.strip()) for a in ann.strip().splitlines() if a.strip()]
    rows.append((anns, typ, name))

mode = sys.argv[1]
SEP = ",\n"
if mode == "record":
    print(SEP.join("        " + " ".join(anns + [typ + " " + name]) for anns, typ, name in rows))
elif mode == "data-record":
    lines = ["        long id"] + ["        " + typ + " " + name for _, typ, name in rows]
    print(SEP.join(lines + ["        BigDecimal remainingQualityPct"]))
elif mode == "data-from":
    # Lombok @Getter sinh getXxx() cả với Boolean bọc; hai danh sách đi qua orEmpty
    lines = []
    for _, typ, name in rows:
        getter = "d.get" + name[0].upper() + name[1:] + "()"
        lines.append("                " + (f"orEmpty({getter})" if typ.startswith("List<") else getter))
    print(SEP.join(lines))
else:  # tsv (phân tách bằng tab, vì nhãn có dấu phẩy): name, type, max, digitsInteger, label
    print("name	type	max	digitsInteger	label")
    for anns, typ, name in rows:
        text = " ".join(anns)
        size = re.search(r'@Size\(max = (\d+), message = "(.*?) không được vượt quá', text)
        pz = re.search(r'@PositiveOrZero\(message = "(.*?) không được âm"', text)
        dg = re.search(r'@Digits\(integer = (\d+)', text)
        label = size.group(2) if size else (pz.group(1) if pz else "")
        print("	".join([name, typ, size.group(1) if size else "", dg.group(1) if dg else "", label]))
```

## Hợp đồng test dùng chung: `NT/model/dto/ShipDescriptionInputs.java`

Test helper `public final class` (B03 tạo, các test của B03 dùng). Dựa vào **reflection trên component của record** (không dùng Jackson để khỏi phụ thuộc phiên bản):
- `static ShipDescriptionInput empty()`: mọi component null.
- `static ShipDescriptionInput with(Map<String, Object> values)`: component vắng thì null; tên không tồn tại thì ném `IllegalArgumentException` (bắt lỗi gõ sai tên).
- `static ShipDescriptionInput sample()`: điền mọi component theo kiểu: `String` → `"  v-" + name + "  "` (có khoảng trắng hai đầu), `Integer` → `7`, `BigDecimal` → `new BigDecimal("12.34")`, `Boolean` → `true`, `LocalDate` → `2026-05-01`, `conditionDetails` → 2 dòng (`" Thân vỏ "`, `" Tốt "`) và (`"Khoang lái"`, null), `qualityEvaluationDetails` → 1 item (`itemNames [" Thân vỏ "]`, w `0.4`, s `10`, rating `" Tốt "`, r `8`).
- `static List<String> componentNames()`; `static Object expectedStored(String name)`: giá trị kỳ vọng trong entity sau `apply(sample())` (chuỗi đã trim, kiểu khác giữ nguyên).

## Task B03: API Mô tả tàu thủy

**Files:**
- Create: `N/model/dto/ShipDescriptionInput.java` (record, 155 component).
- Create: `N/model/dto/ShipDescriptionData.java` (record, `id` + 155 component + `remainingQualityPct`).
- Create: `N/service/ShipDescriptionService.java`.
- Create: `N/controller/ShipDescriptionController.java`.
- Modify: `N/model/entity/AppraisalShipDescription.java` (thêm `apply(ShipDescriptionInput)`; **chỉ thêm**, không đổi gì B01 đã viết).
- Modify: `N/service/LandDescriptionService.java` (`requireLandGroup` chặn SHIP).
- Create (test): `NT/model/dto/ShipDescriptionInputs.java` (helper), `NT/model/dto/ShipDescriptionInputValidationTest.java`, `NT/model/dto/ShipDescriptionDataTest.java`, `NT/model/entity/AppraisalShipDescriptionApplyTest.java`, `NT/repository/ShipDescriptionSaveRepositoryTest.java`, `NT/service/ShipDescriptionServiceTest.java`, `NT/controller/ShipDescriptionControllerTest.java`, `backend/src/test/resources/ship/ship-description-constraints.tsv`.
- Modify (test): `NT/service/LandDescriptionServiceTest.java`.
- Không đổi `backend/CLAUDE.md` (không thêm helper ở `common/`). Không sửa `SecurityConfig` (nhóm `/api/appraisal-records/**` đã bao phủ).

**Interfaces:**
- Produces (B07/B09/FE dùng):
  - `GET /api/appraisal-records/{profileId}/reports/{reportId}/groups/{groupId}/ship-description` → `ShipDescriptionData` (không bọc `ApiResponse`).
  - `PUT` cùng đường dẫn, body `ShipDescriptionInput` (`@Valid`) → `ApiResponse<ShipDescriptionData>` với message `"Lưu mô tả tàu thủy thành công"`.
  - `ShipDescriptionService`:
    - `@Transactional(readOnly = true) ShipDescriptionData get(long profileId, long reportId, long groupId)`
    - `@Transactional ShipDescriptionData save(long profileId, long reportId, long groupId, ShipDescriptionInput input)`
    - hằng package-private `NOT_SHIP_MESSAGE = "Chỉ nhóm tài sản tàu thủy có mô tả tàu thủy"`, `DESCRIPTION_NOT_FOUND_MESSAGE = "Chưa có mô tả tàu thủy"`.
    - Constructor: `ShipDescriptionService(ReportEditScope scope, AppraisalShipDescriptionRepository descriptions, AppraisalGroupCompareRepository compares, AppraisalReportAssetRepository assets)`.
  - `ShipDescriptionController` hằng `MSG_SAVED = "Lưu mô tả tàu thủy thành công"`; `@RequestMapping("/api/appraisal-records/{profileId}/reports/{reportId}/groups/{groupId}/ship-description")`; **không inject repository**.
  - `ShipDescriptionData.from(AppraisalShipDescription d, BigDecimal remainingQualityPct)`; hai danh sách null trong entity thành `List.of()`; Boolean giữ nguyên `Boolean` (có thể null) vì cột nullable.
  - `AppraisalShipDescription.apply(ShipDescriptionInput in)`: **ghi đè mọi field** (chuỗi `Texts.trimToNull`, các kiểu còn lại gán thẳng); `conditionDetails` / `qualityEvaluationDetails` chỉ ghi khi `in.xxx() != null`, qua chuẩn hóa private static trong entity (quy tắc ở "Hành vi hệ thống cũ"); **không đụng** `id`, `appraisalReportId`, `appraisalAssetGroupId`, `deleteFlag`, `created`.
  - `LandDescriptionService.requireLandGroup`: **danh sách cho phép** LAND và APARTMENT; CAR giữ `CAR_NOT_SUPPORTED_MESSAGE`; SHIP ném thêm hằng `SHIP_NOT_SUPPORTED_MESSAGE = "Nhóm tài sản tàu thủy không có mô tả đất"`; loại khác ném `VALIDATION_ERROR` "Loại tài sản này không có mô tả đất". Mọi lỗi đều `ErrorCode.VALIDATION_ERROR`.
- Consumes: mục "Điều kiện tiên quyết từ B01"; `Texts.trimToNull`; `ReportEditScope.requireGroup`; `AppraisalGroupCompareRepository.findFirstByAppraisalReportIdAndAppraisalAssetGroupIdAndTypeAndDeleteFlagOrderByIdAsc(reportId, groupId, AppraisalGroupCompare.TYPE_APPRAISED, AppraisalGroupCompare.DELETE_FLAG_ACTIVE)`; `AppraisalReportAssetRepository.findByAppraisalAssetGroupIdAndDeleteFlagOrderByIdAsc(groupId, AppraisalReportAsset.NOT_DELETED)`; **`GroupAssetTypeReset.joinAssetAddresses(List<AppraisalReportAsset>)`** (package-private static, cùng package `service`, đã lọc `deleteFlag == 0`, trim, bỏ rỗng, rỗng hết thì null; `ReportGroupResolver` cũng dùng).
- Luồng `save`: `requireShipGroup` → `requireDescription` (404 nếu không có hoặc `deleteFlag != NOT_DELETED`) → `description.apply(input)` → `descriptions.save` → tính `pct = ShipQuality.remainingQualityPct(description.getQualityEvaluationDetails())` → nếu có dòng so sánh type 0: `compare.applyShipSync(description, GroupAssetTypeReset.joinAssetAddresses(assets...), pct)` rồi `compares.save(compare)` → trả `ShipDescriptionData.from(description, pct)`. Không có dòng type 0 thì vẫn lưu bình thường. Không chạm dòng type 1–3.

- [ ] **Step 0: Kiểm tra điều kiện tiên quyết B01.**
  ```bash
  git -C backend grep -n "applyShipSync" HEAD -- src/main/java/com/thanhdo/tdprojectbe/model/entity/AppraisalGroupCompare.java
  git -C backend grep -n "remainingQualityPct\|formatDimension" HEAD -- src/main/java/com/thanhdo/tdprojectbe/service/ship/ShipQuality.java
  git -C backend ls-files src/main/java/com/thanhdo/tdprojectbe/model/entity/AppraisalShipDescription.java src/main/java/com/thanhdo/tdprojectbe/model/dto/ShipConditionDetail.java src/main/java/com/thanhdo/tdprojectbe/model/dto/ShipQualityEvaluationItem.java src/main/java/com/thanhdo/tdprojectbe/repository/AppraisalShipDescriptionRepository.java
  git -C backend grep -n "apply(" HEAD -- src/main/java/com/thanhdo/tdprojectbe/model/entity/AppraisalShipDescription.java
  ```
  Thiếu thứ nào thì **dừng, báo lại**. Nếu entity đã có `apply(…)` thì dừng báo (xung đột với giao việc B03). Đọc nguyên `AppraisalShipDescription.java` để chắc tên field Java trùng tên component của L (script `data-from` suy getter từ đó). Đọc `L service/AppraisalShipDescriptionService.java:86-255, 284-456` và toàn bộ `UpdateAppraisalShipDescriptionRequest.java`.

- [ ] **Step 1: Test ràng buộc FAIL.**
  - Tạo script `gen_ship_dto.py` (như trên, ngoài repo) và sinh `backend/src/test/resources/ship/ship-description-constraints.tsv` bằng mode `tsv`. Kiểm tra: `wc -l` ra 156 (1 dòng tiêu đề + 155).
  - Viết `NT/model/dto/ShipDescriptionInputs.java` và `NT/model/dto/ShipDescriptionInputValidationTest.java` (`Validation.buildDefaultValidatorFactory().getValidator()`, như `VcbReportFilterTest`; đọc TSV, tách bằng `\t`):
    - `tsv_khớp_đúng_component_của_input`: tập tên trong TSV == tập tên `ShipDescriptionInput.class.getRecordComponents()` == 155.
    - `chuỗi_vượt_giới_hạn_báo_đúng_message`: với mọi dòng `String` có `max`: giá trị dài `max + 1` → đúng 1 vi phạm, message `"<nhãn> không được vượt quá <max> ký tự"`; dài đúng `max` → không vi phạm.
    - `số_nguyên_âm_báo_không_được_âm`: mọi dòng `Integer`: `-1` → `"<nhãn> không được âm"`; `0` → hợp lệ.
    - `số_thập_phân_âm_báo_không_được_âm`: mọi dòng `BigDecimal`: `-0.01` → có vi phạm message `"<nhãn> không được âm"`.
    - `số_thập_phân_quá_chữ_số_báo_digits`: mọi dòng `BigDecimal`: giá trị có `digitsInteger + 1` chữ số nguyên → message `"<nhãn> tối đa <digitsInteger> chữ số nguyên và 2 chữ số thập phân"`; có 3 chữ số lẻ (`1.234`) → cùng message; `99999999.99` (cột 8) / `9999999999.99` (cột 10) → hợp lệ.
    - `ràng_buộc_lan_vào_hai_danh_sách`: `itemName` 501 ký tự, tên hạng mục thẩm định 501 ký tự, `rating` 51 ký tự, `importanceWeight = -1` → có vi phạm; danh sách null hoặc rỗng → hợp lệ.
  - Lệnh (FAIL vì chưa có `ShipDescriptionInput`):
    ```bash
    cd backend && ./mvnw -q test -Dtest=ShipDescriptionInputValidationTest
    ```

- [ ] **Step 2: Tạo `ShipDescriptionInput`.** Dán đầu ra mode `record` làm component, thêm import `jakarta.validation.Valid/Digits/PositiveOrZero/Size`, `BigDecimal`, `LocalDate`, `List`. Javadoc ghi: nguồn `UpdateAppraisalShipDescriptionRequest` của L; chuỗi cắt khoảng trắng và rỗng thành null khi lưu; hai danh sách null nghĩa là giữ nguyên. Chạy lại lệnh Step 1: PASS.

- [ ] **Step 3: Test entity `apply` + test repository FAIL.**
  - `NT/model/entity/AppraisalShipDescriptionApplyTest` (thuần, không Spring), nền `AppraisalShipDescription.empty(11L, 21L, NOW)`:
    - `mọi_trường_của_input_được_ghi_vào_entity`: `apply(sample())`; với mỗi tên (trừ hai danh sách) `ReflectionTestUtils.getField(entity, name)` bằng `expectedStored(name)` (BigDecimal so `compareTo`); kiểm cả hai danh sách.
    - `trim_rỗng_thành_null`: mọi chuỗi là `"   "` → toàn bộ field chuỗi null.
    - `ghi_đè_mọi_field_kể_cả_null`: `apply(sample())` rồi `apply(empty())` → mọi field (kể cả Boolean, ngày, số) về null.
    - `danh_sách_null_giữ_nguyên`: `apply(sample())` rồi `apply(empty())` → `getConditionDetails()` và `getQualityEvaluationDetails()` **không đổi**.
    - `danh_sách_rỗng_ghi_đè_thành_rỗng`: `apply(with(conditionDetails=[], qualityEvaluationDetails=[]))` → cả hai rỗng.
    - `bỏ_dòng_tình_trạng_rỗng`: `[("  ", null), (null, " Tốt "), (" Thân vỏ ", " Tốt "), (null, "  "), null]` → còn 2 dòng: `(null, "Tốt")` và `("Thân vỏ", "Tốt")`; dòng cả hai rỗng và phần tử null bị bỏ.
    - `bỏ_tên_hạng_mục_rỗng`: item `itemNames = [" Thân vỏ ", "  ", null, "Máy"]` → `["Thân vỏ", "Máy"]`.
    - `bỏ_item_đánh_giá_rỗng_hoàn_toàn`: item `(names [" "], null, null, " ", null)` và phần tử `null` bị bỏ; item chỉ có `importanceWeight = 0.5` được giữ (`itemNames` rỗng).
    - `itemNames_null_thành_danh_sách_rỗng_và_trim_rating`: `(null, 1, 2, " Khá ", 3)` → `itemNames == []`, `rating == "Khá"`.
    - `số_thập_phân_giữ_nguyên_scale`: `dimLength = 12.345678` không có ở bước validate nhưng `apply` không làm tròn (entity giữ đúng `BigDecimal` truyền vào).
    - `không_đổi_khóa_và_cờ_xóa`: `id` (set bằng reflection), `appraisalReportId`, `appraisalAssetGroupId`, `deleteFlag`, `created` không đổi sau `apply`.
  - `NT/repository/ShipDescriptionSaveRepositoryTest` (`@MysqlDataJpaTest`; dựng báo cáo, nhóm `asset_type = 4` bằng native insert như `AppraisalVehicleDescriptionRepositoryTest`, rồi `repository.save(AppraisalShipDescription.empty(...))`):
    - `ghi_đủ_155_trường_rồi_đọc_lại`: `apply(sample())`, `saveAndFlush`, `em.clear()`, `findById` → mọi component bằng `expectedStored` (BigDecimal `compareTo`); `conditionDetails` và `qualityEvaluationDetails` đọc lại đúng nội dung (JSON); các cột TEXT chấp nhận chuỗi dài đúng giới hạn (`operationNote` 5000, `buildPlace` 1000, `towingHookType` 1000).
    - `ghi_đè_null_xuống_cột_nhưng_giữ_json_khi_danh_sách_null`: `apply(sample())`, flush, `apply(empty())`, flush, clear → mọi cột scalar NULL, hai cột JSON còn nguyên.
    - `đồng_bộ_sang_dòng_so_sánh_type0_ghi_cột_ship`: `AppraisalGroupCompare.forGroup(reportId, groupId, TYPE_APPRAISED)`; `compare.applyShipSync(description, "Địa chỉ A", new BigDecimal("86.00"))`; save, flush, clear → `ship_build_year`, `ship_deadweight`, `ship_dimension` (`"118,35m x 19,4m"` khi `dimLengthMax 118.35`, `dimBreadthMax 19.40`), `ship_remaining_quality_pct = 86.00`, `address` đúng.
  - Lệnh (FAIL vì chưa có `apply`):
    ```bash
    cd backend && ./mvnw -q test -Dtest=AppraisalShipDescriptionApplyTest,ShipDescriptionSaveRepositoryTest
    ```

- [ ] **Step 4: Cài đặt `AppraisalShipDescription.apply`.** Sinh khối gán từ danh sách component (mỗi dòng `this.x = trimToNull(in.x())` cho `String`, `this.x = in.x()` cho kiểu khác; dùng `Texts.trimToNull`, **không** viết `trimToNull` private mới theo `backend/CLAUDE.md`). Hai danh sách: `if (in.conditionDetails() != null) this.conditionDetails = cleanConditions(in.conditionDetails());` tương tự quality; `cleanConditions` / `cleanQuality` là `private static` trong entity theo đúng quy tắc chuẩn hóa ở trên. Javadoc: port `AppraisalShipDescriptionService.update` của L; ghi đè mọi field; hai danh sách chỉ ghi khi khác null. Chạy lại lệnh Step 3: PASS (repository test cần MySQL theo `backend/.env.example`).

- [ ] **Step 5: Commit DTO + entity.**
  ```bash
  git -C backend add src/main/java/com/thanhdo/tdprojectbe/model/dto/ShipDescriptionInput.java src/main/java/com/thanhdo/tdprojectbe/model/entity/AppraisalShipDescription.java src/test
  git -C backend commit -m "feat: DTO và ghi trường mô tả tàu thủy"
  ```

- [ ] **Step 6: Test `ShipDescriptionData` FAIL, rồi cài đặt.**
  - `NT/model/dto/ShipDescriptionDataTest`:
    - `data_có_đủ_trường_của_input_cộng_id_và_clcl`: tên component của Data == `["id"] + componentNames() + ["remainingQualityPct"]`; kiểu của mọi component trùng Input (Boolean vẫn là `Boolean`).
    - `from_phản_ánh_từng_trường_của_entity`: entity `apply(sample())`, `ReflectionTestUtils.setField(entity, "id", 61L)`, `ShipDescriptionData.from(entity, new BigDecimal("86.00"))` → `id 61`, mỗi component bằng `expectedStored`, `remainingQualityPct 86.00`.
    - `from_danh_sách_null_thành_rỗng`: entity mới chưa có danh sách → `conditionDetails()` và `qualityEvaluationDetails()` là `[]` (không null).
    - `from_clcl_null_giữ_null`.
  - Tạo `ShipDescriptionData` bằng mode `data-record` và phương thức `from(d, remainingQualityPct)` dùng mode `data-from` (`new ShipDescriptionData(d.getId(), <data-from>, remainingQualityPct)`), cộng `private static <T> List<T> orEmpty(List<T>)`. Javadoc ghi Khác hệ thống cũ **#2**.
  - ```bash
    cd backend && ./mvnw -q test -Dtest=ShipDescriptionDataTest
    ```
    FAIL trước khi tạo lớp, PASS sau đó.

- [ ] **Step 7: Test service FAIL.** `NT/service/ShipDescriptionServiceTest` (Mockito như `VehicleDescriptionServiceTest`; `@Mock ReportEditScope`, `AppraisalShipDescriptionRepository`, `AppraisalGroupCompareRepository`, `AppraisalReportAssetRepository`; mô tả nền `AppraisalShipDescription.empty(REPORT_ID, GROUP_ID, NOW)` với `id 61`; so sánh nền là `AppraisalGroupCompare.forGroup(...)` **thật** để kiểm tra đồng bộ end-to-end):
  - `get_trả_dữ_liệu_kèm_clcl_tính_từ_bảng_đánh_giá`: mô tả có 2 item (w `0.4`, s `10`, r `8`) và (w `0.6`, s `10`, r `9`) → `remainingQualityPct == 86.00`, `id == 61`.
  - `nhóm_không_phải_tàu_trả_400`: lần lượt LAND, APARTMENT, CAR, 99 cho cả `get` lẫn `save` → `VALIDATION_ERROR`, message `NOT_SHIP_MESSAGE`; `verify(descriptions, never()).save(any())`, `verifyNoInteractions(compares)`.
  - `mô_tả_chưa_có_hoặc_đã_xóa_trả_404`: `Optional.empty()` và `deleteFlag = 1` → `NOT_FOUND`, message `DESCRIPTION_NOT_FOUND_MESSAGE`.
  - `trim_rỗng_thành_null`: `save(with(shipName="  Hải Âu ", ownerName="   "))` → `shipName "Hải Âu"`, `ownerName null`, `verify(descriptions).save(description)`.
  - `ghi_đè_mọi_field_kể_cả_null`: `save(sample())` rồi `save(empty())` → field scalar về null.
  - `danh_sách_null_giữ_nguyên`: `save(sample())` rồi `save(with(shipName="X"))` → hai danh sách còn nguyên; `Data` trả đủ.
  - `đồng_bộ_compare0_chỉ_khi_nguồn_khác_null`: compare nền có `shipClass "Cũ"`, `shipBuildPlace "Hải Phòng"`, `shipDeadweight 100`, `shipRemainingQualityPct 55.00`; input chỉ có `buildYear 2012`, `mainEnginePowerHp 1500` → compare: `shipBuildYear 2012`, `shipMainEnginePowerHp 1500`, còn `shipClass`, `shipBuildPlace`, `shipDeadweight`, `shipRemainingQualityPct` **giữ nguyên**; `verify(compares).save(compare)`.
  - `đồng_bộ_compare0_ghi_đủ_9_trường`: input có `buildYear 2012`, `buildPlace " Hải Phòng "`, `shipTypePurpose " Chở hàng khô "`, `shipClass " VR-SB "`, `deadweight 6500.50`, `dimLengthMax 118.35`, `dimBreadthMax 19.40`, `mainEnginePowerHp 1500`, bảng đánh giá cho CLCL 86.00, tài sản `"Tàu A"` → compare ghi đủ 9 trường (`shipBuildYear`, `shipBuildPlace "Hải Phòng"`, `shipUsagePurpose "Chở hàng khô"`, `shipClass "VR-SB"`, `shipDeadweight`, `shipDimension`, `shipMainEnginePowerHp`, `address "Tàu A"`, `shipRemainingQualityPct 86.00`).
  - `kích_thước_ghép_dấu_phẩy`: `dimLengthMax 118.35`, `dimBreadthMax 19.40` → `shipDimension == "118,35m x 19,4m"`; chỉ có `dimLengthMax` (thiếu B) → `shipDimension` giữ nguyên giá trị cũ.
  - `địa_chỉ_ghép_từ_tài_sản_chưa_xóa`: repository tài sản trả `" Tàu A "`, `"  "`, một tài sản `deleteFlag = 1`, `"Tàu B"` → `compare.getAddress() == "Tàu A\nTàu B"`; không còn tài sản hợp lệ thì `address` giữ nguyên.
  - `clcl_ngoài_0_100_không_ghi`: item (w `2`, s `1`, r `1`) → tổng 200; compare giữ `shipRemainingQualityPct 55.00`; `Data.remainingQualityPct()` null.
  - `clcl_không_có_item_hợp_lệ_không_ghi`: item có `standardValue = 0` hoặc thiếu `relativeValue` → giữ nguyên, `Data.remainingQualityPct()` null.
  - `không_có_compare0_vẫn_lưu`: finder trả `Optional.empty()` → `verify(descriptions).save`, `verify(compares, never()).save(any())`.
  - `chỉ_tìm_dòng_type0_chưa_xóa`: `verify(compares).findFirstBy…(REPORT_ID, GROUP_ID, TYPE_APPRAISED, DELETE_FLAG_ACTIVE)`, `verifyNoMoreInteractions(compares)` ngoài `save`.
  - Lệnh (FAIL, chưa có `ShipDescriptionService`):
    ```bash
    cd backend && ./mvnw -q test -Dtest=ShipDescriptionServiceTest
    ```

- [ ] **Step 8: Cài đặt `ShipDescriptionService`** theo Interfaces/Luồng `save`. `@Service @RequiredArgsConstructor`; `requireShipGroup` ném `BusinessException(VALIDATION_ERROR, NOT_SHIP_MESSAGE)` khi `group.getAssetType() != AppraisalAssetType.SHIP`; `requireDescription` lọc `deleteFlag`. Javadoc: tab "Mô tả tàu thủy" của nhóm Tàu thủy (`AppraisalShipDescriptionApiController` và `AppraisalShipDescriptionService.update` của L), nêu đồng bộ sang dòng so sánh type 0 qua `AppraisalGroupCompare#applyShipSync`, và **Khác hệ thống cũ #2, #3** cùng các điểm ở mục "Khác hệ thống cũ" trên. Chạy lại lệnh Step 7: PASS.

- [ ] **Step 9: Test controller FAIL, rồi `ShipDescriptionController`.** `NT/controller/ShipDescriptionControllerTest` (`@WebMvcTest(ShipDescriptionController.class)`, cùng khung `@Import({SecurityConfig.class, JwtConfig.class, TokenService.class})` và `bearer(...)` như `VehicleDescriptionControllerTest`; `@MockitoBean ShipDescriptionService`; `URL = "/api/appraisal-records/5/reports/11/groups/21/ship-description"`; `ShipDescriptionData` dựng bằng `from(entity, pct)`, không dùng constructor 157 đối số):
  - `get_trả_dữ_liệu_màn`: `$.data.id`, `$.data.shipName`, `$.data.remainingQualityPct`, `$.data.conditionDetails[0].itemName`, `$.data.fireManualTools` (boolean).
  - `put_nhận_json_và_trả_dữ_liệu_mới`: body gồm `shipName`, `buildYear 2012`, `dimLength 118.35`, `fireManualTools true`, `inspectionPeriodicDate "2026-05-01"`, `conditionDetails [{itemName, conditionDescription}]`, `qualityEvaluationDetails [{itemNames:["Thân vỏ"], importanceWeight:0.4, standardValue:10, rating:"Tốt", relativeValue:8}]`; kiểm `$.message == "Lưu mô tả tàu thủy thành công"`, `$.data.id`; `ArgumentCaptor<ShipDescriptionInput>` xác nhận từng giá trị đã map đúng kiểu (`LocalDate`, `BigDecimal`, `Boolean`, danh sách).
  - `put_dữ_liệu_sai_trả_400_và_không_gọi_service`: các body `{"buildYear":-1}`, `{"dimLength":-0.01}`, `{"dimLength":123456789.12}`, `{"dimLength":1.234}`, `{"cargoCapacity":12345678901.5}`, `{"shipName":"<501 x>"}`, `{"anchorSternTotalWeightKg":"<1001 x>"}`, `{"operationNote":"<5001 x>"}`, `{"conditionDetails":[{"itemName":"<501 x>"}]}`, `{"qualityEvaluationDetails":[{"rating":"<51 x>"}]}`, `{"qualityEvaluationDetails":[{"importanceWeight":-1}]}`, `{"inspectionDockDate":"không-phải-ngày"}` → 400; `verifyNoInteractions(service)`.
  - `biên_hợp_lệ_được_chấp_nhận`: `{}`, `shipName` 500 ký tự, `dimLength 99999999.99`, `cargoCapacity 9999999999.99`, `operationNote` 5000 ký tự, `buildYear 0` → 200.
  - `chức_vụ_ngoài_menu_bị_chặn_và_phải_đăng_nhập`: `GUEST` 403, không token 401, `verifyNoInteractions(service)`.
  - Lệnh (FAIL), sau đó tạo controller giống `VehicleDescriptionController` (GET trả DTO trực tiếp, PUT `@Valid @RequestBody` trả `ApiResponse.ok(service.save(...), MSG_SAVED)`; Javadoc nêu màn cũ `/api/appraisal-ship-descriptions` và Khác hệ thống cũ #2, #3):
    ```bash
    cd backend && ./mvnw -q test -Dtest=ShipDescriptionControllerTest,ShipDescriptionServiceTest
    ```
    PASS.

- [ ] **Step 10: Commit API.**
  ```bash
  git -C backend add src/main/java/com/thanhdo/tdprojectbe/model/dto/ShipDescriptionData.java src/main/java/com/thanhdo/tdprojectbe/service/ShipDescriptionService.java src/main/java/com/thanhdo/tdprojectbe/controller/ShipDescriptionController.java src/test
  git -C backend commit -m "feat: API mô tả tàu thủy và đồng bộ sang so sánh"
  ```

- [ ] **Step 11: Test chặn SHIP ở mô tả đất FAIL.** Trong `NT/service/LandDescriptionServiceTest` (giữ nguyên `ô_tô_và_mô_tả_chưa_có_bị_từ_chối`):
  - `tàu_bị_từ_chối_cả_get_và_save`: `givenGroup(AppraisalAssetType.SHIP)` → `get` và `save` ném `VALIDATION_ERROR` với message `LandDescriptionService.SHIP_NOT_SUPPORTED_MESSAGE`; `verify(descriptions, never()).save(any())`, `verify(descriptions, never()).findByAppraisalAssetGroupId(any())`.
  - `loại_lạ_bị_từ_chối`: `givenGroup(99)` → `VALIDATION_ERROR`.
  - `đất_và_căn_hộ_vẫn_đi_qua`: hai test hiện có cho LAND/APARTMENT vẫn PASS (không sửa).
  - Lệnh (FAIL do thiếu hằng):
    ```bash
    cd backend && ./mvnw -q test -Dtest=LandDescriptionServiceTest
    ```

- [ ] **Step 12: Cài đặt** `requireLandGroup` theo danh sách cho phép (Interfaces ở trên), cập nhật Javadoc lớp ("nhóm ô tô và tàu thủy trả 400"). Chạy lại Step 11: PASS.

- [ ] **Step 13: Kiểm tra toàn bộ phạm vi và commit.**
  ```bash
  cd backend && ./mvnw -q test -Dtest="ShipDescription*Test,AppraisalShipDescriptionApplyTest,AppraisalImageServiceTest,LandDescriptionServiceTest,LandDescriptionControllerTest,VehicleDescriptionServiceTest"
  git -C backend add src/main/java/com/thanhdo/tdprojectbe/service/LandDescriptionService.java src/test/java/com/thanhdo/tdprojectbe/service/LandDescriptionServiceTest.java
  git -C backend commit -m "fix: mô tả đất chặn nhóm tàu thủy"
  ```
  Rà soát cuối: grep `else` ngầm hiểu CAR/LAND trong hai file đã sửa; xác nhận không file nào ngoài danh sách **Files** bị đổi (`git -C backend status`).

---

## Vấn đề cần controller quyết định

1. **→ Đã chốt: `apply(ShipDescriptionInput)` thuộc B03 (00-index mục Quy ước tên; 01-be-foundation đã tuân theo, B01 không khai báo `apply`).** **`AppraisalShipDescription.apply(ShipDescriptionInput)` bị chia sai task.** `00-index.md` (mục "Quy ước tên") ghi `apply` là phương thức entity do B01 tạo, nhưng `ShipDescriptionInput` do B03 (đợt sau) tạo nên B01 không biên dịch được. Kế hoạch này giao `apply` cho B03. **Cần sửa `01-be-foundation.md`** để B01 không khai báo `apply`, và sửa `00-index.md` cho khớp. Hệ quả: B03 sửa file entity B01 đã tạo; các task cùng đợt B2 không được sửa file này.
2. **→ Đã đối chiếu với 01-be-foundation (khối Interfaces): B01 cung cấp đủ `ofShip`, `getAppraisalShipDescriptionId`, finder ảnh tàu trong `AppraisalImageRepository` (đã thêm vào Files của B01), thứ tự tham số `ShipQualityEvaluationItem`, `applyShipSync` tự gọi `ShipQuality.formatDimension`.** **Giả định về B01 cần đối chiếu:** `AppraisalImage.ofShip` và `getAppraisalShipDescriptionId`, `AppraisalImageRepository.findByAppraisalShipDescriptionIdAndDeletedOrderByCreatedAscIdAsc`, thứ tự tham số `ShipQualityEvaluationItem`, và `applyShipSync` **tự gọi `ShipQuality.formatDimension`** (service chỉ truyền địa chỉ và CLCL). Nếu B01 thiết kế khác (ví dụ nhận kích thước làm tham số) thì sửa test `kích_thước_ghép_dấu_phẩy` và luồng `save` của B03.
3. **Loại ảnh 21–30 chỉ cho nhóm SHIP** (B02) là quyết định bổ sung, không có trong hợp đồng; đồng thời `AppraisalImageService` ném 400 với loại tài sản lạ (trước đây ngầm là đất).
4. **→ Đã kiểm tra: hàm đã có sẵn trên `origin/develop`; B07 (đợt B3) sửa `GroupAssetTypeReset` phải giữ nguyên chữ ký này.** **`GroupAssetTypeReset.joinAssetAddresses` là package-private** ở `service`. B03 gọi trực tiếp (cùng package) và không chạm file đó. Nếu B05/B07 chuyển helper sang `common/` thì B03 phải đổi chỗ gọi.
5. **`ShipDescriptionData` giữ `Boolean` nullable** (cột nullable, `apply` cho phép ghi null), khác `VehicleDescriptionData` dùng `boolean`. Hai danh sách null được trả `[]`.
6. **`requireLandGroup` chuyển sang danh sách cho phép** (LAND, APARTMENT): loại tài sản lạ trước đây lọt vào nhánh đất nay trả 400.
