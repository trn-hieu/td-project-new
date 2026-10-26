# 42 — Thông tin so sánh — LAND (T28)

> Đọc `00-index.md`, `40-compare-backend.md`, `41-compare-common-fe.md` trước. Subagent: `model: "sonnet"`, có review.

## Hành vi hệ thống cũ

Nguồn: `LR templates/demo/comparative-information.html`; `LR static/js/comparative-information-calculator.js:139-160`, `area-outside-details-manager.js` (l.20–37, 226–326), `area-inside-details-manager.js` (l.28–56, 218), `structure-details-manager.js:264-331`, `plant-details-manager.js:145-160,287-306`, `land-use-purpose-updater.js:13-55`.

Dòng (cột 0 = TS thẩm định, 1–3 = TSSS): Địa chỉ (≤500); Số mặt tiền (≥0); Chiều rộng mặt tiền chính (m, thập phân); Loại đường; Chiều rộng đường (chuỗi, vd "6-8m"); Tọa độ (+ nút bản đồ cột 1–3); Đồng bộ (1–3); Hình ảnh (1–3); Nguồn (≤500); Tình trạng thông tin (≤255); Liên hệ (≤255); Tình trạng giao dịch (gợi ý "Đang rao bán", "Đã giao dịch"); Thời điểm giao dịch/chào bán; Tổng diện tích; DT ngoài QH (chỉ đọc = Σ chi tiết; modal chi tiết `{uuid, landTypeId, landType, area, areaConverted, areaNotConverted, price}` — cột 0 chỉ sửa diện tích, không thêm/xóa; cột 1–3 đủ CRUD; loại đất từ `/api/land-types`); DT trong QH (tương tự, `{uuid, landTypeId, landType, area, price}`); Diện tích sử dụng (= Σ cấu trúc.area); Kích thước (≤1000); Cấu trúc (modal `{buildingType, area, unitPrice, clclPercent}`); Cây trồng (modal `{uuid, plantName, quantity, unitPrice, size, totalPrice}`); Mục đích sử dụng đất (tự sinh); Thời hạn sử dụng đất (≤2000); Pháp lý (≤255); Giá rao bán / giao dịch / ước tính; Giá trị đất thuộc QH (tính); Thuế chuyển đổi (cột 0: **checkbox** lưu 1/0 vào `convertTax`; cột 1–3: tiền chỉ đọc tính); Giá trị CTXD (1–3, tính); Giá trị cây trồng (1–3, tính); Giá QSD/m² (1–3, tính); Tỷ lệ ước tính; ô "Phân tích so sánh" (`comparativeAnalysis`).

Công thức (cột i):
- `priceInPlanning = Σ round(area_j × price_j)` trên chi tiết trong QH (diện tích chuỗi châu Âu → số).
- `convertTax_i` (i=1..3): checkbox cột 0 bỏ → 0; ngược lại `maxPrice = max(price)` trên mọi chi tiết ngoài QH của cột i (kể cả diện tích 0), `tax = round(Σ_{area>0} (maxPrice − price_j) × area_j)`.
- `constructionPrice = round(Σ area × unitPrice × clamp(clcl,0..100)/100)`; `useArea = Σ area`.
- Cây: `totalPrice = round(qty × unitPrice)`; `plantPrice = Σ totalPrice`.
- `landUseRightPrice = areaOutside > 0 ? round(((trade + estimate + convertTax) − (priceInPlanning + constructionPrice + plantPrice)) / areaOutside) : 0` (cột 0 dùng convertTax = 0; trade và estimate **cộng**).
- `landUsePurpose = "Diện tích nằm ngoài quy hoạch: {type}: {area} m²; …; Diện tích nằm trong quy hoạch: …."` — chỉ dòng có loại đất và diện tích ≠ 0 (copy chính xác chuỗi từ `land-use-purpose-updater.js`).
- Cảnh báo (không chặn lưu): Tổng diện tích ≠ ngoài + trong QH (sai lệch > 0,01).
- Chọn từ kho (LAND): điền thêm areaOutside/Inside, useArea, landUsePurpose, landDimension, totalArea, numberOfFrontages, mainFrontageWidth, roadType, roadWidth, 4 chi tiết JSON, giá trị cây; sau đó **tính lại thuế chuyển đổi**, rồi **ghi lại `landUseRightPrice` bằng giá trị kho** (giữ quirk).

### Task T28: Thông tin so sánh — Đất

**Files:** Create `F lib/report-edit/compare-land-calc.ts`, `F components/appraisal-report/edit/compare/LandCompareTab.vue`, `AreaDetailsModal.vue` (dùng cho ngoài/trong QH, prop `mode: 'outside' | 'inside'`, `restricted: boolean` cho cột 0), `StructureDetailsModal.vue`, `PlantDetailsModal.vue`; Modify `GroupWorkspace.vue` (map `compare` cho LAND). Test lib + tab + modal.

**Interfaces:** `compare-land-calc.ts`: `priceInPlanning(inside)`, `convertTaxes(rows, flagChecked): [number, number, number]`, `constructionPrice(structures)`, `useArea(structures)`, `plantTotals(plants)`, `landUseRightPrice(row, convertTax)`, `landUsePurposeText(outside, inside)`, `recalcLandRow(row, ctx): row` (gọi theo thứ tự cũ).

- [ ] **Step 1: Test thất bại** `compare-land-calc.spec.ts`: mỗi công thức một test với số liệu tính tay, thêm `cột0_bỏ_checkbox_thuế_0`, `giá_qsd_areaOutside_0_là_0`, `mục_đích_bỏ_dòng_diện_tích_0`; tab: `cột0_không_thêm_xóa_chi_tiết`, `chọn_kho_tính_lại_thuế_rồi_giữ_giá_qsd_kho`, `cảnh_báo_tổng_diện_tích_không_chặn_lưu`, `lưu_gửi_convertTax_cột0_là_1`.
- [ ] **Step 2–4:** FAIL → cài đặt → PASS. **Step 5: Commit** `feat: tab thông tin so sánh đất`.
