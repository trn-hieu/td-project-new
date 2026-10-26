# 43 — Thông tin so sánh — APARTMENT (T29)

> Đọc `00-index.md`, `40-compare-backend.md`, `41-compare-common-fe.md` trước. Subagent: `model: "sonnet"`, có review.

## Hành vi hệ thống cũ

Nguồn: `LR templates/demo/comparative-information-apartment.html` (dòng; l.1160–1179 vị trí tầng; l.1202–1226 công thức), `comparative-information-coordinate-map.js:319-368` (`fillApartmentCompareFields`).

Dòng: Địa chỉ; Loại căn hộ `apartmentUnitKind` (trống, 1 Căn góc, 2 Căn thường); Hướng cửa chính `mainDoorDirection`, Hướng ban công `balconyDirection`, Hướng logia `logiaDirection`, Nội thất `interiorDescription`; Chiều rộng đường; Tọa độ; Đồng bộ; Hình ảnh; Nguồn; Tình trạng thông tin; Liên hệ; Tình trạng giao dịch; Thời điểm; Diện tích tim tường `wallCenterAreaHundredths`, thông thủy `netFloorAreaHundredths`, cơi nới `extensionAreaHundredths` (m², số thực); Vị trí tầng `floorPosition` (1 Tầng cao, 2 Tầng trung, 3 Tầng thấp, 4 Khác) + `floorPositionOther` (chỉ sửa khi 4, khác thì xóa); Cấu trúc căn hộ `apartmentStructure`; Pháp lý (gợi ý: Giấy chứng nhận / HĐMB / Chưa có GCN (HĐMB)); Giá rao bán/giao dịch/ước tính; Cơ sở chia đơn giá `apartmentOwnershipAreaBasis` (1 tim tường, 2 thông thủy); Giá QSH căn hộ `apartmentOwnershipRightPrice` (chỉ đọc); Tỷ lệ ước tính; Phân tích so sánh. Cột 0: địa chỉ, hướng cửa chính, hướng logia được đồng bộ từ mô tả căn hộ (vẫn sửa được như cũ).
- `apartmentOwnershipRightPrice = area > 0 ? round((trade + estimate) / (basis == 2 ? netFloorArea : wallCenterArea)) : 0`, tính lại khi đổi basis/giá/diện tích.
- Chọn từ kho: loại căn, các hướng, nội thất, vị trí tầng (+khác), cấu trúc, 3 diện tích, basis; giá QSH lấy từ `land_use_right_price` rồi tính lại.

### Task T29: Thông tin so sánh — Căn hộ

**Files:** Create `F lib/report-edit/compare-apartment-calc.ts` (`apartmentOwnershipRightPrice(row): number`), `F components/appraisal-report/edit/compare/ApartmentCompareTab.vue`; Modify `GroupWorkspace.vue` (map `compare` cho APARTMENT). Test lib + tab (`basis_2_dùng_thông_thủy`, `diện_tích_0_giá_0`, `vị_trí_tầng_khác_4_xóa_text`, `chọn_kho_tính_lại_giá_qsh`).

- [ ] **Step 1–5:** FAIL → cài đặt → PASS → commit `feat: tab thông tin so sánh căn hộ`.
