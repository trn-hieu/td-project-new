# 22 — Tab "Mô tả phương tiện" — CAR (T18–T19)

> Đọc `00-index.md` trước. Subagent: `model: "sonnet"`, có review.

## Hành vi hệ thống cũ

Nguồn: `LR templates/demo/car-description.html` (form, `buildPayload`, l.593–632 lưu), `L controller/api/AppraisalVehicleDescriptionApiController.java`, `L service/AppraisalVehicleDescriptionService.java` (l.58–149), `L entity/AppraisalVehicleDescription.java`.

- Trường (tên phẳng = cột): `vehicleLocation` (≤2000), `vehicleType`, `licensePlate` (≤50), `color`, `usesCleanEnergy`, `allowsPartialAutomation`, `allowsFullAutomation` (checkbox), `brand`, `modelType`, `engineNumber`, `chassisNumber`, `transmissionType`, `manufactureCountry`, `manufactureYear` (1900–2100), `usageTerm`, `wheelFormula`, `tireTrack`, `overallDimensions`, `wheelbase` (mm), `kerbWeight`, `grossWeight` (kg), `seatingCapacity`, `combustionEngineSymbol`, `fuelType`, `electricMotorType`, `motorQuantitySymbol`, `electricMotorVoltageTotalPower`, `batteryType`, `batteryVoltageCapacity`, `engineDisplacement`, `maxPowerRpm`, `tireDetails`, `odometerReading` (số nguyên ≥0, trống được), `remainingQualityPct` (0..100, 2 lẻ), `remarks` (≤5000). Ảnh 9, 16, 18, 19, 20, 10 (chỉ khi đã có mô tả).
- Lưu `PUT /api/appraisal-vehicle-descriptions/{id}` JSON; server `trimToNull`, năm 1900–2100, km ≥ 0, CLCL 0–100 (scale 2 HALF_UP), sai → 400 "Dữ liệu mô tả phương tiện không hợp lệ"; **đồng bộ dòng so sánh type 0** (l.116–149): `carColor ← color`, `carBrand ← brand`, `carTransmissionType ← transmissionType`, `carManufactureCountry ← manufactureCountry`, `carManufactureYear ← manufactureYear`, `carWheelFormula ← wheelFormula`, `carSeatingCapacity ← seatingCapacity`, `carFuelType ← fuelType`, `carEngineDisplacement ← engineDisplacement`, `carRemainingQualityPct ← remainingQualityPct`, `carOdometerReading ← odometerReading`. Cũ vá DOM tab so sánh; mới tab so sánh tự tải lại khi mở.

### Task T18: Backend Mô tả phương tiện

**Files:** Modify `N model/entity/AppraisalVehicleDescription.java` (map đủ cột + `apply(VehicleDescriptionInput)`), `N model/entity/AppraisalGroupCompare.java` (thêm 11 cột `car_*` ở trên + `applyVehicleSync(...)`, phần còn lại ở T22); Create `N model/dto/VehicleDescriptionData.java`, `VehicleDescriptionInput.java` (jakarta validation: `@Min(1900) @Max(2100)`, `@PositiveOrZero`, `@DecimalMin("0") @DecimalMax("100")`, `@Size`), `N service/VehicleCompareSync.java` (thuần), `N service/VehicleDescriptionService.java`, `N controller/VehicleDescriptionController.java`. Test repository, `VehicleCompareSyncTest`, service, controller.

**Interfaces:** `GET/PUT …/groups/{groupId}/vehicle-description` (chỉ nhóm CAR, khác → 400) → `VehicleDescriptionData`; PUT trả `ApiResponse.ok(VehicleDescriptionData, "Lưu mô tả phương tiện thành công")`.

- [ ] **Step 1–4:** test FAIL (`trim_rỗng_thành_null`, `clcl_làm_tròn_2_lẻ`, `năm_1899_trả_400`, `đồng_bộ_11_trường_sang_compare0`, `không_có_compare0_vẫn_lưu`) → cài đặt → PASS.
- [ ] **Step 5: Commit** `feat: API mô tả phương tiện`.

### Task T19: FE Mô tả phương tiện

**Files:** Create `F api/report-edit/vehicle-description.ts`, `F components/appraisal-report/edit/description/CarDescriptionTab.vue`; Modify `GroupWorkspace.vue` (map `land` cho CAR). Dùng `ImageSection`, `DescriptionCopyPaste` (`CAR_DESCRIPTION`), `NumberInput` (km, năm), `DecimalInput` (CLCL), `CheckboxOptions`/`Checkbox`. Test `CarDescriptionTab.spec.ts` (`năm_ngoài_khoảng_báo_lỗi`, `clcl_quá_100_báo_lỗi`, `lưu_gửi_json_đúng_trường`, `ảnh_ẩn_khi_chưa_có_mô_tả`).

- [ ] **Step 1–4:** FAIL → cài đặt (form lưới `grid gap-5 sm:grid-cols-2 xl:grid-cols-3`, tooltip đơn vị mm/kg) → PASS.
- [ ] **Step 5: Commit** `feat: tab mô tả phương tiện`.
