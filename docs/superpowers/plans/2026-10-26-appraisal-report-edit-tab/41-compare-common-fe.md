# 41 — Tab "Thông tin so sánh" — FE dùng chung (T27)

> Đọc `00-index.md` và `40-compare-backend.md` (mục Hành vi) trước. Subagent: `model: "sonnet"`, có review.

## Hành vi hệ thống cũ (phần chung)

Nguồn: `LR templates/demo/comparative-information.html` (serializer l.2080–2299, đồng bộ kho l.357–442 & 1966–1997, ảnh l.3086–3226), `LR static/js/comparative-information-coordinate-map.js` (tìm gần l.779–814, điền l.377–552, xóa cột l.572–654).

- Bảng 4 cột (TS thẩm định, TSSS 1, 2, 3) × nhiều dòng trường; một form mỗi nhóm; lưu xong tải lại.
- Dòng "Đồng bộ lên cơ sở dữ liệu" (TSSS 1–3): checkbox, đã tick khi `syncToCompareInformationId != null`; khóa khi cột chọn từ kho (`compareInformationId != null`) hoặc người dùng không có quyền (KSV/TPTD); người không có quyền mà dòng đã đồng bộ → vẫn gửi `syncToCompareInfo = 1`. Cột chọn từ kho có nút "Bỏ chọn tài sản so sánh" (xóa toàn bộ cột, chi tiết JSON thành `[]`, mở khóa checkbox).
- Ảnh TSSS (cột 1–3): xem/thêm/xóa ngay qua API ảnh TSSS, JPG/PNG.
- Tìm TSSS gần: nút bản đồ cạnh ô tọa độ cột 1–3 → bản đồ quanh tọa độ cột đó, danh sách TSSS trong 3 km cùng loại tài sản → chọn → điền cột, gán `compareInformationId`, khóa đồng bộ.
- "Tỷ lệ ước tính" (chỉ hiển thị) = `estimate / forSale × 100`, 2 lẻ, dấu phẩy.

### Task T27: Thành phần dùng chung

**Files:** Create `F api/report-edit/compare.ts` (GET/PUT so sánh + ảnh TSSS), `F api/report-edit/compare-informations.ts` (nearby, search), `F lib/report-edit/compare-common.ts` (`estimateRatio(forSale, estimate): string`, `emptyColumn(row, assetType)`, `fillFromKho(row, detail, assetType)` — danh sách trường điền theo loại đúng như cũ, kết quả gán `compareInformationId`), `F components/appraisal-report/edit/compare/CompareGrid.vue` (khung bảng 4 cột: nhãn dòng + slot ô theo cột; `ScrollableTable`), `KhoSyncCell.vue`, `CompareImagesModal.vue`, `NearbyCompareMapModal.vue` (Google Maps qua `lib/google-maps.ts`, marker + danh sách), `F composables/useCompareForm.ts` (nạp, trạng thái 4 dòng, bẩn, lưu, gọi `useDirtyTab`). Test lib + từng component.

**Interfaces:** `useCompareForm(props): { data: Ref<GroupCompareData|null>; rows: Ref<GroupCompareRowInput[]>; isDirty; loading; error; save(): Promise<void>; pickFromKho(col: 1|2|3, detail): void; clearColumn(col): void }`.

- [ ] **Step 1–4:** test FAIL (`tỷ_lệ_ước_tính`, `điền_từ_kho_gán_id_và_khóa_đồng_bộ`, `bỏ_chọn_xóa_cột_về_mảng_rỗng`, `không_quyền_nhưng_đã_đồng_bộ_vẫn_gửi_1`, `bản_đồ_chọn_tsss_emit_detail`) → cài đặt → PASS.
- [ ] **Step 5: Commit** `feat: thành phần dùng chung tab thông tin so sánh`.
