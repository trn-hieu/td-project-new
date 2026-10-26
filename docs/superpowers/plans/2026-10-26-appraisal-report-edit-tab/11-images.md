# 11 — Ảnh mô tả dùng chung (T10–T11)

> Đọc `00-index.md` trước. Subagent: `model: "sonnet"`, có review.

## Hành vi hệ thống cũ

Nguồn: `L controller/api/AppraisalImageApiController.java`, `L service/AppraisalImageService.java`, `LR templates/demo/demo-report-step-2.html:312-405` (modal), `LR templates/demo/land-description.html:1387-1801` (luồng ảnh), `L common/ReportConstants.java` (`AppraisalImageType`).

- Bảng `appraisal_image`: ảnh thuộc **một** mô tả đất (`appraisal_land_description_id`) hoặc mô tả phương tiện (`appraisal_vehicle_description_id`); cột `type, note, name, path, group_uuid, deleted, synced_from_apartment_building_image_id`.
- Loại ảnh: 1 sơ đồ tài sản, 2 hiện trạng, 3 thông tin quy hoạch, 4 ghi chú, 5 vị trí-hướng, 6 đường trước tài sản, 7 mặt tiền tổng quan, 8 hiện trạng tài sản, 9 người hướng dẫn và người khảo sát, 10 khác, 11 thang máy-thang bộ, 12 hầm-tầng để xe, 13 tiện ích tòa chung cư, 14 hành lang trước căn hộ, 15 đối diện tài sản, 16 mặt trước-sau xe, 18 hai bên thân xe, 19 bên trong xe, 20 số khung và số km.
- Thêm ảnh: modal chọn sẵn loại (khóa ô chọn), có thể đổi nhãn hiển thị; JPEG/PNG, nhiều file, ghi chú tối đa 500; mỗi lần upload một `group_uuid`, ghi chú lưu trên mọi ảnh của lô. S3 key `{assetFolder}{land|vehicle}/{reportId}/{descId}/{uuid}/{fileName}`; URL hiển thị CloudFront.
- Hiển thị: lọc theo loại, gom theo `group_uuid`, lưới 2 cột, ghi chú của ảnh đầu lô bên dưới. Xóa: xác nhận → xóa mềm (`deleted = 1`) + xóa S3.
- **Lỗi cũ**: không chọn file thì nút Lưu kẹt trạng thái loading — bản mới validate "Vui lòng chọn ảnh".

### Task T10: Backend ảnh mô tả

**Files:** Create `N model/entity/AppraisalImage.java`, `N repository/AppraisalImageRepository.java`, `N model/dto/AppraisalImageView.java`, `N service/AppraisalImageService.java`, `N controller/AppraisalImageController.java`; Test `NT repository/AppraisalImageRepositoryTest.java`, `NT service/AppraisalImageServiceTest.java`, `NT controller/AppraisalImageControllerTest.java`.

**Interfaces:**
- Produces:
  - `record AppraisalImageView(long id, int type, String note, String url, String groupUuid, LocalDateTime created)`.
  - `GET …/groups/{groupId}/images` → `List<AppraisalImageView>` (theo mô tả của nhóm: LAND/APARTMENT → mô tả đất, CAR → mô tả phương tiện; chưa có mô tả → rỗng).
  - `POST …/groups/{groupId}/images` multipart `type` (1–20 trừ 17), `note` (≤500), `images` (≥1, JPEG/PNG — dùng `SafeContentType` nếu phù hợp) → `ApiResponse.ok(List<AppraisalImageView>, "Thêm ảnh thành công")`; mô tả chưa tồn tại → 400 "Vui lòng lưu mô tả trước khi thêm ảnh".
  - `DELETE …/images/{imageId}` → xóa mềm + S3 (ảnh phải thuộc mô tả của báo cáo).
  - `AppraisalImageService` public cho T16: `List<AppraisalImage> activeOfLandDescription(long landDescriptionId, Set<Integer> types)`.

- [ ] **Step 1–4:** test FAIL (`cùng_lô_cùng_group_uuid_và_ghi_chú`, `ô_tô_gắn_vehicle_description`, `xóa_mềm_và_xóa_s3`, `loại_17_bị_từ_chối`, `ảnh_báo_cáo_khác_404`) → cài đặt → PASS.
- [ ] **Step 5: Commit** `feat: API ảnh mô tả tài sản`.

### Task T11: FE khối ảnh + modal thêm ảnh

**Files:** Create `F api/report-edit/images.ts`, `F lib/report-edit/image-types.ts` (hằng loại + nhãn), `F components/appraisal-report/edit/images/ImageSection.vue`, `ImageUploadModal.vue`; Test `images/__tests__/ImageSection.spec.ts`, `ImageUploadModal.spec.ts`.

**Interfaces:** Produces `ImageSection.vue` props `{ profileId, reportId, groupId, type: number, title: string, images: AppraisalImageView[], disabled?: boolean }`, emit `changed`; `ImageUploadModal.vue` props `{ type, typeLabel, … }`. Danh sách ảnh nạp **một lần** ở tab mô tả (T14/T17/T19) rồi truyền xuống từng `ImageSection` (lọc theo type), không mỗi khối một request.

- [ ] **Step 1–4:** test FAIL (`gom_theo_group_uuid`, `không_chọn_file_báo_lỗi`, `xóa_hỏi_xác_nhận_rồi_emit_changed`) → cài đặt (`AppModal` `persistent`; input file theo mẫu sẵn có trong repo — tìm `accept="image` để dùng lại; ảnh `loading="lazy"`) → PASS.
- [ ] **Step 5: Commit** `feat: khối ảnh mô tả dùng chung`.
