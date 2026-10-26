# 03 — Tab "Kết quả thẩm định" (T06)

> Đọc `00-index.md` trước. Subagent: `model: "sonnet"`, có review.

## Hành vi hệ thống cũ

Nguồn: `LR templates/appraisal-records/evaluation-results.html` (fragment `evaluation-results-group-form` l.1451–1601, `loadEvaluationData` l.1043–1287, bảng l.710–1039, lưu l.1315–1441, boot script l.1614–1640).

- Mọi số liệu lấy từ API detail; client không tính (trừ bằng chữ: `new ReadMoney().doc(roundingMoney)` gửi khi lưu).
- Điều khiển: radio "Làm tròn đến hàng triệu" (1, mặc định) / "…hàng nghìn" (2) — đổi radio gọi lại API với `roundingMoneyType` (chưa lưu); textarea "Ghi chú"; nút "Lưu" chỉ hiện cho BGD, TPTD, NVTD, NVKD. Sau khi lưu không tải lại. Nút "Tải báo cáo" — **bỏ** (ngoài phạm vi).
- **Đất**: (1) "Bảng kết quả thẩm định giá đất" — cột Hạng mục / Loại đất / Diện tích / Đơn giá / Thành tiền; khi phương pháp AGRICULTURAL/LAND_USE_CONVERSION/STATE_PRICE và có `landOutsideDetails` → mỗi chi tiết một dòng dưới nhãn gộp "Giá trị quyền sử dụng đất nằm ngoài quy hoạch"; ngược lại một dòng từ `landOutside` (Loại đất "—"); khối "…nằm trong quy hoạch" từ `landInsideDetails`; "Tổng cộng" = `landAmount`, "Làm tròn" = `roundedLandAmount`. Ghi chú AGRICULTURAL (l.741–773, chỉ ở bảng nhúng trong Bảng điều chỉnh): tiêu đề + đoạn dùng `totalAreaBelowMaxPrice`, danh sách `landType: price đồng/m²` bỏ dòng `maxPriceRecord`. (2) "Bảng kết quả thẩm định giá trị CTXD" (l.871–949): 8 cột (Hạng mục, Diện tích, Đơn giá, Chi phí xây dựng mới, Chất lượng còn lại %, Tỷ lệ khấu hao %, Giá trị khấu hao, Thành tiền); phần "Công trình tính giá trị vào tổng…" (INCLUDE/null) + Tổng cộng/Làm tròn; phần "…để Quý ngân hàng tham khảo" (REFERENCE) + tổng; NOT_INCLUDED không hiện; ẩn bảng khi cả hai phần rỗng. (3) "Kết quả thẩm định giá tài sản" (l.977–1017): "Giá trị quyền sử dụng đất" = `roundedLandAmount`; "Giá trị công trình xây dựng" = `roundedConstructionAmount` (chỉ khi có công trình INCLUDE/null); "Tổng giá trị bất động sản" = `roundingMoney`; "Bằng chữ: … ./.".
- **Căn hộ** (l.710–729, 954–973): không có `directComparison` hoặc phương pháp khác DIRECT_COMPARISON → "Chưa có dữ liệu kết quả thẩm định căn hộ."; ngược lại một dòng "Giá trị quyền sở hữu " + chữ đầu thường(`assetName`), diện tích (hoặc "—"), đơn giá, thành tiền + "Làm tròn"; bảng kết quả: dòng 1 = `roundedAmount ?? amount ?? 0`, "Kết quả thẩm định" = `roundingMoney`, Bằng chữ.
- **Ô tô** (l.1020–1039): "Giá trị " + chữ đầu thường(`assetName`) = `car.amount`; "Làm tròn" = `car.roundedAmount ?? roundingMoney`; Bằng chữ.
- Định dạng: tiền làm tròn số nguyên, nhóm nghìn bằng "."; **0 hoặc null hiển thị trống**; diện tích là chuỗi từ server, hiển thị nguyên.

### Task T06: Tab Kết quả thẩm định

**Files:**
- Create: `F api/report-edit/evaluation-result.ts`, `F lib/report-edit/evaluation-result-format.ts`, `F components/appraisal-report/edit/results/EvaluationResultTab.vue`, `F components/appraisal-report/edit/results/EvaluationResultTables.vue` (chỉ hiển thị, dùng lại ở CTXD và Bảng điều chỉnh), `F components/appraisal-report/edit/results/LandResultTable.vue`, `ConstructionResultTable.vue`, `AssetResultTable.vue`
- Modify: `F components/appraisal-report/edit/GroupWorkspace.vue` (map `results`)
- Test: `F lib/report-edit/__tests__/evaluation-result-format.spec.ts`, `F components/appraisal-report/edit/results/__tests__/EvaluationResultTab.spec.ts`, `EvaluationResultTables.spec.ts`

**Interfaces:**
- Consumes: T05 API; `readMoney` (`F lib/read-money.ts`); `useAuthStore().authorities`.
- Produces:
  - `api/report-edit/evaluation-result.ts`: `interface EvaluationResult` (khớp bảng trường của `02-result-api.md`), `getEvaluationResult(profileId, reportId, groupId, roundingMoneyType?: 1 | 2, signal?)`, `saveEvaluationResult(profileId, reportId, groupId, body: { totalMoneyAppraisal: string; roundingMoney: number; roundingMoneyType: 1 | 2; totalPriceWordAppraisal: string; note: string | null })`.
  - `evaluation-result-format.ts`: `formatResultMoney(value: number | null | undefined): string` ("" cho 0/null), `formatResultPercent(value)`, `lowerFirst(text: string | null): string`.
  - `EvaluationResultTables.vue` props `{ result: EvaluationResult; sections?: ('land' | 'construction' | 'asset')[]; showAgriculturalNote?: boolean }`.
  - `EvaluationResultTab.vue` expose `reload()`.

- [ ] **Step 1: Test thất bại**: format (`0_hiển_thị_trống`, `nhóm_nghìn_bằng_dấu_chấm`, `lowerFirst`); Tables: `đất_mixed_một_dòng_loại_đất_gạch`, `đất_nông_nghiệp_nhiều_dòng`, `công_trình_ẩn_khi_không_có`, `căn_hộ_chưa_có_dữ_liệu`, `ô_tô_làm_tròn_fallback_roundingMoney`; Tab: `đổi_radio_gọi_api_với_roundingMoneyType`, `lưu_gửi_bằng_chữ_tính_bằng_readMoney`, `ẩn_nút_lưu_với_NVPI`, `ghi_chú_chưa_lưu_đánh_dấu_bẩn` (gọi `useDirtyTab`).
- [ ] **Step 2–4:** FAIL → cài đặt (bảng dùng `@/components/ui/table`, tiền `text-right tabular-nums`; radio `RadioOptions`; ghi chú `AutoTextarea`; `FormActions` với nút "Lưu") → `npm run lint && npm run type-check && npm run test:unit` PASS.
- [ ] **Step 5: Commit** `feat: tab kết quả thẩm định`.
