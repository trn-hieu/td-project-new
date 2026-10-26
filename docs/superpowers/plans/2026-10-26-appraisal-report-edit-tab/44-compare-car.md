# 44 — Thông tin so sánh — CAR (T30)

> Đọc `00-index.md`, `40-compare-backend.md`, `41-compare-common-fe.md` trước. Subagent: `model: "sonnet"`, có review.

## Hành vi hệ thống cũ

Nguồn: `LR templates/demo/comparative-information-car.html` (dòng; l.905–943 trường đồng bộ), `LR static/js/comparative-information-car-search.js` (l.311–391), `LR templates/demo/comparative-information-car-search-modals.html`.

Không có tọa độ, bản đồ, phân tích so sánh. Dòng: Tên xe `carName` (cột 1–3 có nút tìm); Nguồn; Liên hệ; Đồng bộ; Hình ảnh; Nhãn hiệu `carBrand`; Số người `carSeatingCapacity`; Dung tích động cơ `carEngineDisplacement` (hiển thị " (cm³)"); Kiểu hộp số `carTransmissionType`; Nhiên liệu `carFuelType`; Công thức bánh xe `carWheelFormula`; Xuất xứ `carOrigin` (mọi cột sửa được); Năm sản xuất `carManufactureYear` (1900–2100); Nước sản xuất `carManufactureCountry`; Màu `carColor`; Số km `carOdometerReading`; CLCL (%) `carRemainingQualityPct` (0..100); Giá rao bán/giao dịch/ước tính; Tỷ lệ ước tính; Ghi chú `carNote` (≤1000). Cột 0: các trường xe (trừ Xuất xứ) **chỉ đọc**, đồng bộ từ Mô tả phương tiện.
- Tìm theo tên xe: modal, từ khóa bắt buộc, cuộn vô hạn trang 20; chọn → điền trường xe, nguồn, liên hệ, ba giá, ghi chú; gán `compareInformationId`, khóa đồng bộ.

### Task T30: Thông tin so sánh — Ô tô

**Files:** Create `F components/appraisal-report/edit/compare/CarCompareTab.vue`, `CarCompareSearchModal.vue`; Modify `GroupWorkspace.vue` (map `compare` cho CAR), `lib/report-edit/compare-common.ts` (danh sách trường điền CAR nếu T27 chưa có). Test (`cột0_trường_xe_chỉ_đọc`, `xuất_xứ_cột0_sửa_được`, `tìm_không_từ_khóa_báo_lỗi`, `cuộn_tải_trang_tiếp`, `chọn_xe_điền_cột`).

- [ ] **Step 1–5:** FAIL → cài đặt → PASS → commit `feat: tab thông tin so sánh ô tô`.
