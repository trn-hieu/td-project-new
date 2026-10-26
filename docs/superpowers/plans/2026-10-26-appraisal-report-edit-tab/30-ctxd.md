# 30 — Tab "Mô tả CTXD" — chỉ LAND (T20–T21)

> Đọc `00-index.md` trước. Subagent: `model: "sonnet"`, có review.

## Hành vi hệ thống cũ

Nguồn: `LR templates/demo/construction-description.html` (pane l.3–69, modal l.75–385, mặc định l.480–610, lưu l.731–1020, công thức l.1218–1258, tên đánh giá l.1731–1763), `L controller/api/ConstructionDescriptionApiController.java`, `L service/ConstructionDescriptionService.java`, `L service/impl/AppraisalAssetGroupServiceImpl.java:95-117`, `L controller/AppraisalReportController.java:457-464`.

- Danh sách công trình của nhóm (chưa xóa): STT, Tên, ô diện tích (4 dòng "DT xây dựng theo GCN / Tổng DT sử dụng theo GCN / DT xây dựng thực tế / Tổng DT sử dụng thực tế"), Cấu tạo, Ghi chú, Sửa/Xóa. Bên dưới: bảng kết quả CTXD (từ API kết quả, phần `construction`). Form ghi chú: "Ghi chú mô tả CTXD" (`construction_note`), "Ghi chú kết quả thẩm định giá CTXD" (`construction_appraisal_result_note`) trên `appraisal_asset_groups`.
- Modal công trình: `constructionName` (bắt buộc, ≤500), `constructionType`, `valuationType` (NOT_INCLUDED / REFERENCE / **INCLUDE_IN_TOTAL** mặc định), `completionYear`, `renovationYear` (1900–2100), `constructionArea`, `totalUsageArea`, `actualConstructionArea`, `actualTotalUsageArea` (0..99.999.999,99), `valuationAreaBasis` (**0 thực tế** mặc định, 1 GCN), `structure`, `currentConstructionStatus`, `note`; kết cấu `structures[] {category (Móng, Nền sàn, Tường vách, Đỡ mái, Mái, Khác; "Trần" → Đỡ mái), structure, status, evaluation}`; bảng đánh giá `evaluations[] {mainStructureName (= category, chỉ đọc), remainQualityRate, mainStructureAgainstTotalValueRate, multiplication, sortOrder}` — dòng "Khác" ẩn và không gửi; thêm/xóa kết cấu thêm/xóa dòng đánh giá cùng chỉ số; tổng `totalValueRate`, `totalMultiplication`, `totalQualityRemainPercentage` (0..999,99); `unitPrice`; `circularInstruction` mặc định "Căn cứ nghị định số 95/2024/NĐ-CP ngày 24 tháng 7 năm 2024 của Chính Phủ về việc Quy định chi tiết một số điều của luật Nhà ở."; `decisionBasis` gợi ý từ `report_juridical` theo **tỉnh của báo cáo**, mặc định dòng `category == 2 && isDefault == 1`; ảnh đính kèm (jpg/png).
- Thêm mới: 6 dòng kết cấu Móng, Nền sàn, Tường vách, Đỡ mái, Mái, Khác; tỷ lệ mặc định Tường vách 15, Mái 16, còn lại 10.
- Công thức: `multi_i = round2(remain_i × rate_i / 100)`; `totalValueRate = Σ rate_i`; `totalMultiplication = Σ multi_i`; `totalQualityRemainPercentage = totalValueRate > 0 ? round2(totalMultiplication / totalValueRate × 100) : 0` (dòng hiển thị). Khác cũ: số thập phân dùng `DecimalInput` nên "12,5" không bị cắt thành 12.
- Lưu: tạo/sửa JSON; sửa **xóa cứng** kết cấu + đánh giá rồi chèn lại; ảnh upload sau khi lưu; xóa công trình = xóa mềm cả con; sau tạo/sửa tải lại danh sách và bảng kết quả (cũ: xóa không tải lại kết quả — mới tải lại cả hai). S3 ảnh `{assetFolder}construction/{reportId}/{constructionId}/{uuid}/{fileName}`; xóa ảnh mềm + S3.

### Task T20: Backend CTXD

**Files:** Modify `N model/entity/AppraisalConstructionDescription.java` (`@GeneratedValue(IDENTITY)`, map đủ cột, quan hệ con `@OneToMany(mappedBy)` hoặc repository riêng), `N model/entity/AppraisalAssetGroup.java` (`construction_note`, `construction_appraisal_result_note`, bỏ `updatable=false` ở `updated`); Create `AppraisalConstructionStructure`, `AppraisalConstructionEvaluation`, `AppraisalConstructionImage` + repositories; `N model/dto/ConstructionTabData.java` (danh sách + ghi chú + `decisionBasisOptions` + `defaultDecisionBasis` + `defaultCircularInstruction`), `ConstructionListItem.java`, `ConstructionDetail.java`, `ConstructionInput.java`, `ConstructionNotesInput.java`; `N service/ConstructionDescriptionService.java`; `N controller/ConstructionDescriptionController.java`. Test repository (đủ cột, xóa cứng con), service, controller; chạy lại `service/result` tests.

**Interfaces:** `GET …/groups/{g}/constructions` → `ConstructionTabData`; `GET …/constructions/{id}` → `ConstructionDetail` (kèm ảnh CloudFront); `POST …/groups/{g}/constructions`, `PUT …/constructions/{id}` body `ConstructionInput` → `ApiResponse.ok(ConstructionDetail, …)`; `DELETE …/constructions/{id}`; `POST …/constructions/{id}/images` (multipart `images`); `DELETE …/construction-images/{imageId}`; `PUT …/groups/{g}/construction-notes` body `ConstructionNotesInput(String constructionNote, String constructionAppraisalResultNote)`. Công trình/ảnh phải thuộc báo cáo trong đường dẫn.

- [ ] **Step 1–4:** test FAIL (`tạo_mặc_định_valuationType_và_basis`, `sửa_thay_toàn_bộ_kết_cấu`, `xóa_mềm_cả_con`, `ghi_chú_chỉ_ghi_hai_cột`, `quyết_định_mặc_định_theo_tỉnh_báo_cáo`) → cài đặt → PASS.
- [ ] **Step 5: Commit** `feat: API mô tả công trình xây dựng`.

### Task T21: FE tab Mô tả CTXD

**Files:** Create `F api/report-edit/constructions.ts`, `F lib/report-edit/construction-evaluation.ts`, `F components/appraisal-report/edit/ctxd/ConstructionTab.vue`, `ConstructionModal.vue`, `ConstructionStructureRows.vue`; Modify `GroupWorkspace.vue` (map `ctxd`). Dùng `EvaluationResultTables` (`sections=['construction']`), `SuggestInput` cho căn cứ quyết định. Test lib + component.

**Interfaces:** `construction-evaluation.ts`: `DEFAULT_STRUCTURES`, `defaultEvaluationRate(category): number`, `evaluationTotals(rows): { totalValueRate; totalMultiplication; totalQualityRemainPercentage }`, `rowMultiplication(remain, rate): number`.

- [ ] **Step 1–4:** test FAIL (`mặc_định_6_dòng_tỷ_lệ_15_16_10`, `tổng_bỏ_dòng_khác`, `tổng_tỷ_lệ_0_thì_0`, `lưu_xong_tải_lại_danh_sách_và_kết_quả`, `xóa_hỏi_xác_nhận`) → cài đặt (`AppModal size="xl" persistent`) → PASS.
- [ ] **Step 5: Commit** `feat: tab mô tả công trình xây dựng`.
