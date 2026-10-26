# 51 — Tab "Bảng điều chỉnh" — Frontend (T32–T34)

> Đọc `00-index.md` và `50-panel-backend.md` trước. Subagent: `model: "sonnet"`, có review.

## Hành vi hệ thống cũ (tính trên client, server lưu nguyên)

Nguồn: `LR static/js/appraisal-panel-tabbed.js` (`tabbedControlCompareFactoryCalculatorForGroup` l.117–504, ràng buộc l.511–552, thêm dòng l.589–691, thu thập l.823–993, đổi nhãn l.1167–1264), `LR static/js/appraisal.js:2035-2049` (`amplitudeControlCalculator`), `LR templates/demo/appraisal-panel.html`.

Bảng: SỐ TT | YẾU TỐ | ĐƠN VỊ | TS TĐG | SS1 | SS2 | SS3.
- A "Giá thị trường" (CAR: "Loại xe"), B "Giá quy đổi về đơn vị so sánh chuẩn" (LAND/APARTMENT, VND/m²) hoặc "Đơn giá" (CAR, VND) — chỉ đọc từ server.
- Mỗi factor: C{n}; chọn nhãn (Khác/null → nhập `label` bắt buộc; nhãn khác → `label` = tên nhãn); đơn vị; `value`, `valueCp1..3` (bắt buộc, ≤255); dòng "Tỷ lệ đất ở trên tổng diện tích" (%) chỉ đọc, chỉ khi nhãn 8; "Đánh giá" (0/trống "Tương đồng với tài sản thẩm định", < 0 "Tốt hơn tài sản thẩm định", > 0 "Kém hơn tài sản thẩm định"); "Tỷ lệ điều chỉnh" (%) mặt nạ `-?\d*\.?\d{0,2}`, trống khi blur → "0"; "Mức điều chỉnh"; "Mức giá sau điều chỉnh". Dòng đầu không xóa; thêm dòng = nhãn Khác, `rate*` = 100, `isDefault = 0`. Đổi nhãn → gọi sync-preview điền `value*` (+ tỷ lệ khi nhãn 8; đổi khỏi 8 xóa tỷ lệ) rồi tính lại.
- Công thức cột i (1..3), B_i = landUseRightPrice cột i; duyệt factor theo thứ tự, `p` = tỷ lệ điều chỉnh: rỗng/không phải số → `controlRate = 0`, `controlPrice = 0`, giá sau không đổi; ngược lại `controlPrice = round(B_i × p / 100)` (**luôn theo B**), `priceAfter = B_i + Σ_{j≤k} controlPrice_j` (lưu `Math.round` và bản thực), `p ≠ 0` → đếm + đưa |p| vào danh sách biên độ.
- D `priceGuide_i` = giá sau cùng (hiển thị `floor`, lưu bản thực `priceGuideReal`). E1 = Σ|controlPrice| (floor). E2 = số lần điều chỉnh. E3 = biên độ: rỗng → "", min = max → "x%", ngược lại "min% - max%"; hiển thị `toFixed(2)` đổi "." → ","; bản thực không làm tròn. E4 = |Σ controlPrice| (floor).
- D1 `avg = (pg1 + pg2 + pg3) / 3` (**luôn chia 3**); hiển thị: chế độ 0 `floor(avg)`, chế độ 1 `round(avg/1000)×1000`; **`avgPriceGuide` lưu = giá trị hiển thị**; `avgPriceGuideReal` = avg thực. D2 `gap_i = (pg_i − avg)/avg × 100` (2 lẻ + " %"; không chặn chia 0 — bản mới: avg = 0 → hiển thị trống, ghi chú Khác hệ thống cũ trong component).
- "Nguyên tắc khống chế" = mẫu + `priceMain` + " " + đơn vị (+ ", làm tròn {D1 hiển thị} {đơn vị}." khi chế độ 1, `priceMain = floor(avgReal)`); đơn vị `đồng/xe` (CAR) hoặc `đồng/m²`; sinh lại mỗi lần tính. Không có kiểm tra 15% bằng số.
- Thẻ 2: radio làm tròn (0 không, 1 nghìn); "Nguyên tắc khống chế" chỉ đọc; `landUseRightPlanningValue` khi `hasPlanningLand`; phương pháp định giá: LAND 4 lựa chọn (MIXED / AGRICULTURAL_STATE_PRICE / LAND_USE_CONVERSION / STATE_PRICE), APARTMENT chỉ DIRECT_COMPARISON, CAR không có; `statePriceNote` khi STATE_PRICE (LAND); nút "Chi tiết loại đất ngoài QH" khi LAND_USE_CONVERSION (LAND) → modal: ẩn dòng `landTypeId = 1`, sửa `areaConverted` (0 ≤ x ≤ area), `areaNotConverted = max(0, area − converted)`, ghi chú phương pháp chuyển đổi + ghi chú giá nhà nước nông nghiệp; lưu xong tải lại tab (cũ reload trang — mất sửa dở: bản mới hỏi xác nhận nếu bảng đang bẩn). Bảng kết quả nhúng (đất: có ghi chú AGRICULTURAL) — tải lại sau khi lưu bảng điều chỉnh / modal.

### Task T32: Hàm tính bảng điều chỉnh

**Files:** Create `F lib/report-edit/adjustment-panel-calc.ts`; Test `F lib/report-edit/__tests__/adjustment-panel-calc.spec.ts`.

**Interfaces:** `calculatePanel(input: { b: [number | null, number | null, number | null]; rates: Array<[string, string, string]>; roundingType: 0 | 1; assetType: 1 | 2 | 3; template: string }): PanelResult` với `PanelResult = { factors: Array<{ controlRate: [number, number, number]; controlPrice: [number, number, number]; priceAfter: [number, number, number]; priceAfterReal: [number, number, number] }>; priceGuide; priceGuideReal; avgDisplay: number; avgReal: number; gapReal: [number, number, number]; gapText: [string, string, string]; totalControlPrice /*E1*/; totalControlCount /*E2*/; amplitudeText; amplitudeReal; netControlPrice /*E4*/; regulatoryConstraints: string }`; `evaluationLabel(rate: string): string`; `amplitudeText(values: number[]): string`.

- [ ] **Step 1: Test thất bại** với ≥ 3 bộ số liệu tính tay từ công thức cũ: `điều_chỉnh_luôn_theo_B_không_cộng_dồn_cơ_sở`, `tỷ_lệ_rỗng_không_đếm`, `biên_độ_một_giá_trị`, `biên_độ_min_max_dấu_phẩy`, `trung_bình_luôn_chia_3_khi_tsss_trống`, `làm_tròn_nghìn_avgDisplay`, `gap_avg_0_trống`, `ràng_buộc_ô_tô_đồng_xe`, `đánh_giá_ba_nhãn`.
- [ ] **Step 2–4:** FAIL → cài đặt → PASS. **Step 5: Commit** `feat: hàm tính bảng điều chỉnh`.

### Task T33: Bảng điều chỉnh — Đất

**Files:** Create `F api/report-edit/panel.ts`, `F components/appraisal-report/edit/panel/AdjustmentPanelTab.vue`, `FactorRows.vue`, `PanelSummaryRows.vue` (D/E), `ValuationMethodCard.vue`, `AreaOutsideConversionModal.vue`; Modify `GroupWorkspace.vue` (map `panel`). Dùng `EvaluationResultTables` (`sections=['land']`, `showAgriculturalNote`). Test `AdjustmentPanelTab.spec.ts` (`tải_lại_khi_mở_tab`, `đổi_nhãn_gọi_preview_và_điền`, `nhãn_khác_bắt_buộc_nhập_tên`, `thêm_dòng_mặc_định_khác`, `không_xóa_dòng_đầu`, `lưu_gửi_avgPriceGuide_hiển_thị`, `chuyển_đổi_hiện_nút_chi_tiết`, `lưu_modal_tải_lại_kết_quả`), `AreaOutsideConversionModal.spec.ts` (`ẩn_đất_ở`, `chuyển_đổi_vượt_diện_tích_báo_lỗi`).

- [ ] **Step 1–5:** FAIL → cài đặt (bảng `@/components/ui/table` trong `ScrollableTable`, số `text-right tabular-nums`) → PASS → commit `feat: tab bảng điều chỉnh đất`.

### Task T34: Bảng điều chỉnh — Căn hộ, Ô tô

**Files:** Modify `AdjustmentPanelTab.vue`, `ValuationMethodCard.vue` (APARTMENT: chỉ DIRECT_COMPARISON; CAR: ẩn khối phương pháp, ẩn ghi chú/nút đất), `PanelSummaryRows.vue` (nhãn B "Đơn giá", đơn vị VND cho CAR; A "Loại xe"); Test thêm case `căn_hộ_chỉ_so_sánh_trực_tiếp`, `ô_tô_nhãn_loại_xe_và_đơn_giá`, `ô_tô_không_có_phương_pháp`, `kết_quả_nhúng_căn_hộ_và_ô_tô` (`sections=['asset']`).

- [ ] **Step 1–5:** FAIL → cài đặt → PASS → commit `feat: bảng điều chỉnh căn hộ và ô tô`.
