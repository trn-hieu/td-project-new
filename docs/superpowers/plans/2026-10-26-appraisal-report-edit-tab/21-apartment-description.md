# 21 — Tab "Mô tả đất" — APARTMENT (T16–T17)

> Đọc `00-index.md` và `20-land-description.md` trước. Subagent: `model: "sonnet"`, có review.

## Hành vi hệ thống cũ

Nguồn: `LR templates/demo/land-description-apartment.html` (form l.1–570, sao chép l.574–708), `LR static/js/demo-apartment-building-search.js`, `LR templates/demo/apartment-building-search-modals.html`, `L service/AppraisalLandDescriptionService.java:262-404` (đặc biệt 362–398), `L service/ApartmentBuildingService.java` (`upsertFromLandDescriptionRequest`), `L service/ApartmentBuildingImageSyncService.java:43-110`, `L controller/api/ApartmentBuilding*Controller.java`.

- Trường riêng căn hộ: `apartmentBuildingName` (+ nút tìm tòa), `syncFromApartmentBuilding`, `syncToApartmentBuilding` (ẩn), checkbox "Đồng bộ thông tin chung cư vào CSDL" `syncToApartmentBuildingEnabled` (khóa khi `syncFrom` có giá trị; "Bỏ chọn tòa" xóa `syncFrom`; checkbox bật thì khóa nút tìm), `apartmentCommercialName`, `usageForm` (1 chung, 2 riêng), `landUsePurpose`, `landUseTerm`, `landUseOrigin`, `apartmentBuildingStatus`, `constructionCompletionYear`, `remainingQuality`; mục 2 căn hộ: `apartmentForm`, tên tòa/tên thương mại (chỉ đọc, phản chiếu mục 1), `ownershipForm` (1 chung, 2 riêng), `ownershipTerm`, `sharedOwnershipOutsideApartment`, `floorArea`, `extensionArea`, `totalUsableArea` (**không tự cộng**), `livingRoomCount, kitchenCount, bedroomCount, bathroomCount, balconyCount, logiaCount` (≥0), `mainDoorDirection`, `logiaDirection` (≤500), `apartmentOther`, `buildingConditionAssessment` (1 Tốt, 2 Khá tốt, 3 Khá, 4 Trung bình khá, 5 Kém), `apartmentAmenities`, `structureDescription`, `physicalCondition`, `note`. Trường chung với đất: `categoryType`, tỉnh/quận/phường, tọa độ, `landPosition` (không mặc định), `positionDescription`, `landArea`.
- Ảnh: 1, 3 trên cùng; 5, 9, 6 (nhãn "Đường trước tòa chung cư"), 7 ("Mặt tiền tổng quan tòa nhà"), 11, 12, 13, 14, 8 ("Hiện trạng căn hộ"), 10; ghi chú (4).
- Tìm tòa: `GET /api/apartment-buildings/search?keyword&limit=20&provinceId&districtId&wardId` → chọn → `GET /api/apartment-buildings/{id}` → điền: categoryType, tỉnh/quận/phường, tọa độ, landPosition, positionDescription, landArea, usageForm, landUse*, buildingStatus, completionYear, remainingQuality, tên, tên thương mại; `syncFromApartmentBuilding = id`; khóa checkbox đồng bộ.
- Lưu: `syncFrom == null && syncToEnabled == 1` → `upsertFromLandDescriptionRequest` (tìm theo `syncTo`, rồi theo tên + tỉnh, không có thì tạo; copy các trường tòa/căn hộ sang `apartment_building`), `syncTo = id`, rồi `replaceBuildingImagesFromLandDescription` (ảnh loại {1,3} của mô tả → `apartment_building_image`, thay ảnh đồng bộ trước, an toàn S3). Ngược lại nếu `syncFrom != null` → `replaceLandImagesFromBuilding` (ảnh {1,3} của tòa → `appraisal_image` với `synced_from_apartment_building_image_id`). Đồng bộ dòng so sánh type 0: tọa độ + `address = join("\n", địa chỉ tài sản không rỗng)`, `mainDoorDirection`, `logiaDirection`.
- **Sửa (Khác hệ thống cũ 7)**: lưu căn hộ chỉ ghi cột chung + cột căn hộ; **không** đụng `landPriceJson`, `landDetailsJson`, hướng, `notPlanningLand`… và **không** chạy đồng bộ chi tiết DT ngoài/trong QH (cũ làm rỗng chi tiết).

### Task T16: Backend Mô tả căn hộ + tòa chung cư

**Files:** Create `N model/entity/ApartmentBuilding.java`, `ApartmentBuildingImage.java` + repositories; `N service/ApartmentBuildingService.java` (search, detail, `upsertFromLandDescription`), `N service/ApartmentBuildingImageSync.java`; `N controller/ApartmentBuildingController.java`; `N model/dto/ApartmentBuildingOption.java`, `ApartmentBuildingDetail.java`; Modify `N service/LandDescriptionService.java` (nhánh APARTMENT), `N model/entity/AppraisalGroupCompare.java` (thêm `address`, `main_door_direction`, `logia_direction` nếu T13 chưa có), `N model/entity/AppraisalImage.java` (cột `synced_from_apartment_building_image_id`). Test: repository tòa/ảnh, `ApartmentBuildingServiceTest`, `ApartmentBuildingImageSyncTest`, thêm case `LandDescriptionServiceTest`, controller.

**Interfaces:** `GET /api/apartment-buildings/search?keyword&provinceId&districtId&wardId&limit=20` → `List<ApartmentBuildingOption(long id, String name, String commercialName, String address)>`; `GET /api/apartment-buildings/{id}` → `ApartmentBuildingDetail` (các trường điền ở trên). Tìm theo từ khóa dùng `LikePattern.contains`.

- [ ] **Step 1–4:** test FAIL (`lưu_căn_hộ_không_đụng_chi_tiết_đất`, `đồng_bộ_compare_type0_địa_chỉ_và_hướng`, `bật_đồng_bộ_tạo_tòa_mới_và_gán_syncTo`, `đã_có_syncTo_cập_nhật_tòa_cũ`, `chọn_từ_tòa_sao_chép_ảnh_1_3`, `ảnh_không_đổi_không_sao_chép_lại`) → cài đặt → PASS.
- [ ] **Step 5: Commit** `feat: mô tả căn hộ và đồng bộ tòa chung cư`.

### Task T17: FE Mô tả căn hộ

**Files:** Create `F api/report-edit/apartment-buildings.ts`, `F components/appraisal-report/edit/description/ApartmentDescriptionTab.vue`, `ApartmentBuildingSearchModal.vue`; Modify `GroupWorkspace.vue` (map `land` cho APARTMENT). Dùng lại `LocationFields`, `ImageSection`, `DescriptionCopyPaste` (`LAND_APARTMENT_DESCRIPTION`). Test tương ứng.

- [ ] **Step 1–4:** test FAIL (`chọn_tòa_điền_trường_và_khóa_đồng_bộ`, `bỏ_chọn_tòa_mở_khóa`, `bật_đồng_bộ_khóa_nút_tìm`, `tên_tòa_phản_chiếu_mục_2`, `số_phòng_không_âm`) → cài đặt → PASS.
- [ ] **Step 5: Commit** `feat: tab mô tả căn hộ`.
