# 90 — Kiểm tra cuối và review toàn nhánh (Z01)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development hoặc superpowers:executing-plans. Đọc `00-index.md` (Review Focus, Khác hệ thống cũ, Global Constraints) trước khi làm. Subagent reviewer chỉ dùng `model: "sonnet"`.

## Task Z01: Review toàn nhánh, chạy toàn bộ test, E2E thủ công

**Phụ thuộc:** tất cả task B01–B10 và F01–F11 đã `[x]` (đã qua spec review và code review).

**Files:** không sửa file nào, trừ khi reviewer xác nhận lỗi (khi đó sửa theo quy trình: implementer, spec review, code review, rồi chạy lại Step 1–3).

**Quy ước:** `BE` = `C:/Users/ABC/Desktop/td-project-new/backend`, `FE` = `.../frontend`, cả hai đang ở nhánh `feat/ship-asset-type`. Không checkout nhánh khác, không push, không cập nhật con trỏ submodule ở repo gốc khi chưa được người dùng đồng ý.

### Step 1: Chạy toàn bộ test

- [ ] **1.1 Backend.** DB dev phải có schema tàu (24 migration trong `00-index.md`) và `.env` theo `.env.example`.
  ```bash
  git -C backend branch --show-current        # feat/ship-asset-type
  cd backend && ./mvnw test
  ```
  Kỳ vọng: `BUILD SUCCESS`, 0 failure, 0 error. Ghi số test (`Tests run: N`). Phải thấy PASS các lớp: `CurrencyCodeTest` (5 case), `AppraisalResultServiceSaveCurrencyTest` (9 case), test `ShipQuality`, test vòng đời nhóm (TC-10b/c/d và ca xóa nhóm), test kho TSSS tàu, test controller/service/repository của `ship-description`.
- [ ] **1.1b Parity Word (Quyết định #18).** B09 thêm hook vào `AssetTypeWriter` và sửa bốn writer cũ; phải chứng minh output loại 1–3 không đổi, một lần ở đây (cần MySQL như 1.1):
  ```bash
  cd backend && ./mvnw -q test -Dtest=AppraisalWordExporterParityTest -Dparity=true
  ```
  Kỳ vọng PASS; ghi kết quả vào báo cáo. Không có DB thì ghi rõ "chưa chạy parity" và nêu lý do.
- [ ] **1.2 Frontend.**
  ```bash
  git -C frontend branch --show-current       # feat/ship-asset-type
  cd frontend && npm run lint && npm run type-check && npm run test:unit -- --run && npm run build
  ```
  Kỳ vọng: cả bốn lệnh thoát 0 (`lint` gồm oxlint, eslint, `lint:tokens`). Ghi số test vitest. Phải có test của `currency.ts`, `ship-quality.ts`, `ship-description-fields.ts`.
- [ ] **1.3** Lệnh nào fail: dừng, ghi nguyên văn lỗi, giao cho implementer sửa. Không bỏ qua, không đánh dấu test `@Disabled`/`skip`.

### Step 2: Rà soát bằng grep

Chạy từng lệnh, ghi kết quả vào báo cáo. Mỗi kết quả trả về phải được giải thích (đã có nhánh SHIP riêng, hoặc là CAR thật sự).

- [ ] **2.1 BE, không còn `else` ngầm là CAR.**
  ```bash
  cd backend/src/main/java/com/thanhdo/tdprojectbe
  grep -rnE "AppraisalAssetType\.CAR|CommonConstants\.AppraisalAssetType\.CAR" . 
  grep -rnE "\bisCar\b|isCar\(" .
  grep -rnE "AppraisalAssetType\.(LAND|APARTMENT|CAR)" . | grep -vE "SHIP"
  grep -rnE "case +[0-9]|1 *\| *2 *\| *3|\b[123] *, *[123] *, *[123]\b" . | grep -iE "asset|type"
  grep -rn "SHIP" common/constant/AppraisalAssetType.java common/constant/CommonConstants.java
  ```
  Với mỗi dòng có `CAR`, mở file và xác nhận ngay cạnh có nhánh `SHIP` tương ứng. Bắt buộc rà các điểm của Review Focus 1: `AppraisalResultCalculator`, `ControlFactorDefaults` (Quyết định #11: `case SHIP` riêng, `default` giữ nhãn đất như hiện tại), `LandDescriptionService.requireLandGroup` (chặn SHIP), `ReportLegalService`, `AppraisalImageService.isCar` (đã thay bằng map theo loại hoặc có `isShip`). Mỗi điểm phải có test chứng minh nhóm SHIP đi đúng nhánh: ghi tên test.
- [ ] **2.2 FE, không còn `else` ngầm là CAR.**
  ```bash
  cd frontend/src
  grep -rnE "ASSET_TYPE\.CAR" . --include=*.ts --include=*.vue | grep -v __tests__
  grep -rnE "\bisCar\b" . --include=*.ts --include=*.vue
  grep -rnE "1 *\| *2 *\| *3" . --include=*.ts --include=*.vue
  grep -rnE "assetType *(===|!==|==|!=) *[0-9]" . --include=*.ts --include=*.vue
  grep -rn "ASSET_TYPE.SHIP\|SHIP:" lib/report-edit/tabs.ts
  ```
  Điểm bắt buộc rà (Review Focus 1): `GroupWorkspace.byAssetType`, `AssetResultTable`, `PanelSummaryRows`, `compare-common.ts`. Kiểu union `1 | 2 | 3` phải đã mở rộng thành gồm `4` hoặc dùng `AssetType`. Không có số `4` viết cứng ngoài `ASSET_TYPE.SHIP`.
- [ ] **2.2b Khóa JSON kết quả tàu (Quyết định #1).**
  ```bash
  grep -rn "shipDetail" backend/src frontend/src
  ```
  Phải **rỗng**: khóa là `ship` ở BE (`EvaluationResultResponse`, `EvaluationResultDetailDto`) và FE (`EvaluationResult.ship`).
- [ ] **2.3 Hai bản hằng số SHIP.**
  ```bash
  grep -rn "SHIP *= *4" backend/src/main/java
  ```
  Phải thấy `SHIP = 4` ở **cả** `common/constant/AppraisalAssetType` **và** `CommonConstants.AppraisalAssetType`, cộng `ValuationBasis.ASSET_TYPE_SHIP = 4`. Ở FE: `grep -rn "SHIP: *4\|SHIP *= *4" frontend/src/lib/report-edit/tabs.ts`. Loại ảnh 21–30 khớp bảng trong `00-index.md` ở BE và FE.
- [ ] **2.4 Không có migration mới trong backend.**
  ```bash
  git -C backend diff origin/develop --name-status | grep -iE "db/migration|\.sql$|flyway"      # phải rỗng
  git -C backend diff origin/develop --stat -- src/main/resources                                  # chỉ cho phép đổi cấu hình nếu có lý do ghi trong task
  ```
- [ ] **2.5 Exporter Word cũ không bị sửa.**
  ```bash
  git -C backend diff origin/develop -- src/main/java/com/thanhdo/tdprojectbe/common/util/export/word/AppraisalReportWordExporter.java
  ```
  Kết quả phải **rỗng**. Không có `if (assetType == …)` mới trong các section Word (`grep -rn "SHIP" backend/src/main/java/**/export/word/appraisal/section` phải rỗng; SHIP chỉ qua `ShipAssetWriter` + `AssetTypeWriters`).
- [ ] **2.6 Phạm vi diff.**
  ```bash
  git -C backend diff origin/develop --stat
  git -C frontend diff origin/develop --stat
  ```
  Không có file ngoài các task; không có file rác (`*.orig`, ảnh thử, `.env`); không chạm nhánh `report/ship_market_overview`, role Ban lãnh đạo, báo cáo công nợ/kê khai, tab Xem trước.
- [ ] **2.7 Javadoc "Khác hệ thống cũ".**
  ```bash
  grep -rn "Khác hệ thống cũ" backend/src/main/java | grep -i ship
  ```
  Mỗi mục 1, 2, 3, 4, 5 trong `00-index.md` phải có ghi chú ở class/phương thức liên quan (xóa nhóm xóa mềm mô tả tàu và giữ ảnh; PUT trả `ShipDescriptionData`; lỗi 400 qua `BusinessException`; Word dòng A "Giá bán"; JSON thay fragment HTML).
- [ ] **2.8 Bảng helper.** `backend/CLAUDE.md` có dòng cho `CurrencyCodes` và `ShipQuality` (B01 thêm) và `ShipValuationMath` (B06 không sửa file này; controller gộp dòng đó sau đợt B2 theo ghi chú cuối task B06). Thiếu dòng nào thì bổ sung ở bước này.

### Step 3: Final reviewer (một subagent, sonnet)

- [ ] **3.1** Gọi **một** subagent: `Agent({ subagent_type: "general-purpose", model: "sonnet", prompt })`. Prompt gồm:
  - đường dẫn `00-index.md` và `surveys/*.md`, yêu cầu đọc trước;
  - đọc diff hai nhánh bằng `git -C backend diff origin/develop` và `git -C frontend diff origin/develop` (không checkout nhánh khác, không sửa file);
  - đối chiếu code legacy bằng `git -C td-project-legacy show 82193fe5:<path>`;
  - tiêu chí: Global Constraints; **Review Focus 1–5**; **Khác hệ thống cũ 1–5** (có trong Javadoc và có test); `backend/CLAUDE.md` + `backend/.claude/rules/*.md` (đặc biệt `jpa-spring.md`: LAZY, không `@Data`, `@Transactional`, `BusinessException`); `frontend/CLAUDE.md` + `frontend/DESIGN_SYSTEM.md` (token màu, không control native, tab tối đa 2 cấp).
  - Yêu cầu kiểm cụ thể:
    1. Review Focus 1: mọi chỗ `else`/`default` đã tách SHIP; mỗi chỗ có test.
    2. Review Focus 2: đủ 9 case `AppraisalResultServiceSaveCurrencyTest` và 5 case `CurrencyCodeTest`; BE và FE `resolveCurrencyCode` giống hệt.
    3. Review Focus 3: ba nơi tính CLCL (`ShipQuality`, `ship-quality.ts`, Word) cùng kết quả trên cùng bộ dữ liệu; (4) làm tròn 1 chữ số chỉ để hiển thị, (5) dùng giá trị chưa làm tròn; tổng ngoài 0–100 không ghi sang so sánh.
    4. Review Focus 4: ô tô→tàu, tàu→đất, đất→tàu dùng lại bản ghi xóa mềm không vi phạm unique; xóa/khôi phục nhóm; TC-10b/c/d.
    5. Review Focus 5: TSSS khác tiền tệ 400 trước mọi side-effect ảnh/S3; đồng bộ lên kho ghi `currencyCode` đã resolve, không ghi `shipForSalePriceUsd`.
    6. DB chung với legacy: tên khóa JSON converter khớp DTO của L; kiểu `delete_flag`/`deleted` đúng từng bảng; converter đọc lỗi trả rỗng; record JSON có `ignoreUnknown`.
    7. Hồi quy LAND/APARTMENT/CAR: không test cũ nào bị sửa kỳ vọng để cho qua.
  - Định dạng trả lời: APPROVED hoặc danh sách lỗi `file:dòng`, mức độ (chặn/nên sửa), lý do.
- [ ] **3.2** Lỗi được xác nhận (sau khi controller tự kiểm chứng, không tin mù): sửa theo quy trình implementer + spec review + code review (sonnet), rồi chạy lại Step 1 và Step 2.

### Step 4: Checklist E2E thủ công

Chạy app trên **DB dev đã có schema tàu**. Không dùng DB production. Lệnh khởi chạy (xác nhận lại với `backend/CLAUDE.md`, `frontend/package.json`, `.claude/launch.json` nếu có):
```bash
cd backend && ./mvnw spring-boot:run        # cần .env theo .env.example
cd frontend && npm run dev                  # Vite, trỏ proxy vào BE
```
Tài khoản test do người dùng cung cấp hoặc lấy từ dữ liệu seed của DB dev. Không tự tạo tài khoản thật. Mỗi mục đánh dấu `[x]` khi đạt; mục lỗi ghi vào báo cáo kèm bước tái hiện.

**4.1 Tạo báo cáo**
- [ ] Tạo hồ sơ có pháp lý loại **Tàu thủy** với cơ sở thẩm định tàu (`ValuationBasis` loại 4 hiện trong danh sách). Báo cáo có nhóm tài sản SHIP.
- [ ] Mở báo cáo: có đủ 5 tab của nhóm tàu (Pháp lý, Mô tả tàu thủy, Thông tin so sánh, Bảng điều chỉnh, Kết quả thẩm định); tab mở được, không lỗi console.

**4.2 Tab Pháp lý**
- [ ] Nhập thông tin GCN đăng ký (số, nơi cấp, ngày cấp, cho ai) và GCN kiểm định (số, nơi cấp, ngày cấp, hết hạn, cho ai): lưu, tải lại giữ nguyên.
- [ ] Sinh văn bản pháp lý: ra **2 dòng GCN** (đăng ký và kiểm định) đúng văn phong legacy.

**4.3 Tab Mô tả tàu**
- [ ] Nhập đủ các khối (thông số, kích thước, máy chính/phụ, thiết bị kéo, neo, tín hiệu, cứu hỏa, mooring...). Các ô số dạng TEXT nhận chuỗi, ô DECIMAL nhận số thập phân.
- [ ] Bảng **tình trạng** hiện sẵn **18 mục mặc định**; sửa mô tả, lưu, tải lại giữ nguyên.
- [ ] Bảng **chất lượng**: nhập hệ số, giá trị chuẩn, đánh giá; cột (4) hiển thị làm tròn 1 chữ số, tổng CLCL khớp công thức; tổng ngoài 0–100 thì hiện lỗi/không đồng bộ.
- [ ] Ảnh: đủ **11 mục** (21–30 và "Khác" mã 10); tải lên, xem lại, xóa được.
- [ ] Lưu thành công (toast "Lưu mô tả tàu thủy thành công"). Chuyển sang tab So sánh: cột **thẩm định (type 0) đã đồng bộ** (chủng loại, năm đóng, nơi đóng, trọng tải, kích thước Lmax x Bmax, công suất...) và **CLCL**.

**4.4 Tab So sánh**
- [ ] Mở modal tìm TSSS từ kho: tìm theo **chủng loại** (từ khóa) ra kết quả phân trang, sắp theo cập nhật mới nhất; chỉ có TSSS loại tàu.
- [ ] Chọn TSSS vào cột 1–3; **bỏ chọn** hoạt động; các dòng ngoài đúng thứ tự 14 hàng của Word (xem 4.9).
- [ ] Tick **đồng bộ lên kho** rồi lưu: bản ghi kho cập nhật, `currencyCode` là mã đã resolve; `shipForSalePriceUsd` không bị ghi.
- [ ] TSSS có tiền tệ **khác** tiền tệ của nhóm: modal **chặn** chọn (có thông báo); gọi thẳng API (DevTools hoặc curl) thì server trả **400** và không có ảnh/S3 nào được tạo (kiểm S3/bảng ảnh).

**4.5 Bảng điều chỉnh**
- [ ] Có **8 nhãn** đúng thứ tự: Năm đóng, Nơi đóng, Ngoại quan tổng thể, Khả năng vận hành, Trọng tải toàn phần, Chất lượng còn lại, Công suất (CV), Cấp phương tiện.
- [ ] Giá trị `c_value*` được đồng bộ từ so sánh: năm đóng, nơi đóng, trọng tải, công suất, CLCL, cấp; **Ngoại quan** và **Khả năng vận hành** không đồng bộ (nhập tay).
- [ ] Dòng A là **"Giá bán"**; không có dòng "Giá thị trường (giá trước điều chỉnh)" và "Giá quy đổi về đơn vị so sánh chuẩn".
- [ ] Đơn vị đúng: VND là "đồng" / "VND" theo ô; USD là "USD". Không có "/tàu", "/m²".

**4.6 Kết quả thẩm định**
- [ ] Mặc định VND: bảng kết quả hiện đúng; lưu; tổng, làm tròn, bằng chữ ("... đồng") đúng.
- [ ] Đổi sang **USD**, tỷ giá **25.450**, nhập ghi chú: lưu. Bảng web chuyển thành **3 cột** (Hạng mục, Thành tiền (USD), Thành tiền (VNĐ)), không có cột STT; `vndAmount = amount x 25450`; làm tròn theo kiểu làm tròn đã chọn; **bằng chữ theo VNĐ** (hậu tố "đồng", không phải "đô la Mỹ").
- [ ] Tỷ giá ngoài 1..999.999.999 hoặc ghi chú > 1000 ký tự: báo lỗi 400 rõ ràng. Quy đổi tràn số: message "Thành tiền quy đổi sang VNĐ quá lớn...".
- [ ] Quay lại tab So sánh (tải lại khi mở): nhãn giá hiện **(USD)**; Bảng điều chỉnh đơn vị USD.
- [ ] Đổi lại VND: tỷ giá về 1.

**4.7 Xuất Word** (đối chiếu `surveys/legacy-ui-word.md` mục 8 và Khác hệ thống cũ #4). Mở file bằng Word, kiểm từng mục:
- [ ] Một nhóm tàu: tiêu đề mục 6 "THẨM ĐỊNH GIÁ TRỊ TÀI SẢN"; nhiều nhóm: `6.i.1` "Mô tả tài sản thẩm định". Mục con `.1` Mô tả, `.2` Tình trạng, `.3` Đánh giá chất lượng, `.4` phần chung. Tiêu đề "Thẩm định giá trị tài sản".
- [ ] Mục 2.1.2 (pháp lý liên quan, chỉ có tàu) có đúng 4 bullet: TT 45/2013/TT-BTC, TT 147/2016/TT-BTC, TT 28/2017/TT-BTC, QĐ 57/1999/QĐ-TĐC. Cụm chủ thể "giá trị tàu chở hàng khô".
- [ ] Mô tả tàu: dạng danh sách bullet, không bảng; khối rỗng bị bỏ tiêu đề; dòng tự thêm dấu "."; khối **Thiết bị kéo** (4 dòng) đặt trước Máy chính; các dòng ghép (Kích thước cơ bản, Lmax x Bmax, Số lượng máy a/b, Tổng công suất, Khả năng khai thác); Năm đóng và Nơi đóng hai dòng riêng; mỗi dòng `operationNote` một bullet; bơm chữa cháy "m³"; máy chính "Công suất (HP/KW):".
- [ ] Trang bị tín hiệu: "Đèn hiệu:" + bảng lưới ☑/☐ (8 cột, 4 cặp/hàng, không viền), "Âm hiệu: …", "Vật hiệu:" + bảng lưới.
- [ ] Tình trạng: đoạn mở đầu (subject, số đăng ký, năm đóng, nơi đóng, neo đậu tại ...) + câu cố định "Tiếp cận kiểm tra ..."; bảng 3 cột 8/32/60%; chỉ dòng có dữ liệu.
- [ ] Đánh giá chất lượng: đoạn dẫn nguyên văn (QĐ–TĐC gạch en-dash); bảng 8 cột, header 2 hàng với (1)...(5); mỗi item `max(1, số tên)` hàng và gộp dọc; (4) làm tròn 1 chữ số, (3)(5) dạng `#,##0.00` và "%" ở (5); dòng cuối "Chất lượng còn lại của tàu"; câu "Vậy chất lượng còn lại của ... là: x%." Số liệu **khớp với tab web**.
- [ ] Bảng so sánh: 5 cột (18,5% + 4x20,375%), **không hàng ảnh**, không hàng "Tỷ lệ ước tính", hàng cả 4 ô rỗng bị bỏ; thứ tự 14 hàng (Chủng loại ... Giá ước tính); đơn vị giá theo tiền tệ.
- [ ] Thông tin TSSS: mỗi TSSS có dòng in đậm "Tài sản so sánh k: là ...", "Link tham khảo:" từng link một bullet, bảng ảnh TSSS nếu có.
- [ ] **Bảng điều chỉnh (Khác hệ thống cũ #4):** dòng A "Giá bán"; **không** có "Giá thị trường (giá trước điều chỉnh)" và "Giá quy đổi về đơn vị so sánh chuẩn"; ô đơn vị theo tiền tệ; nguyên tắc khống chế dùng đúng đơn vị.
- [ ] Bảng kết quả: VND 3 cột (8/50/42%); USD 4 cột (8/40/26/26%) có cột "Thành tiền (đồng)"; dòng "Làm tròn" đúng theo bảng ở mục 8; ghi chú chuyển đổi in nghiêng, giãn dòng 1,5, chỉ khi USD và có ghi chú. Nhiều nhóm: bảng tổng hợp dùng `roundedAmount` VNĐ, "Tổng giá trị tài sản".
- [ ] Ảnh tàu: thứ tự 12 nhóm ảnh (Mặt trước ... Khác), 2 ảnh/hàng, hàng lẻ gộp 2 ô; không có ảnh thì bỏ mục.
- [ ] Pháp lý tài sản: văn bản tự sinh từng dòng bullet (gồm 2 dòng GCN).
- [ ] So sánh với bản Word xuất từ **legacy** trên cùng dữ liệu: các điểm khác chỉ là điểm đã khai ở Khác hệ thống cũ #4.

**4.8 Vòng đời nhóm**
- [ ] Đổi loại **Tàu → Đất → Tàu**: bản ghi mô tả tàu **dùng lại** (cùng id, không lỗi unique), nội dung được reset về mặc định hợp lý; **tiền tệ về VND**, tỷ giá 1, ghi chú null.
- [ ] Ô tô → Tàu và Tàu → Ô tô: không lỗi; dữ liệu CAR cũ không rò sang tàu.
- [ ] **Xóa** nhóm tàu: mô tả tàu bị xóa mềm, **ảnh vẫn được giữ** (Khác hệ thống cũ #1, Quyết định #8), nhóm biến mất khỏi báo cáo.
- [ ] **Khôi phục** nhóm: mô tả tàu **và ảnh** hiện lại đúng dữ liệu trước khi xóa.
- [ ] Đổi loại rời tàu (Tàu → Đất): ảnh của mô tả tàu bị xóa (như legacy), khác với xóa nhóm.

**4.9 Hồi quy LAND / APARTMENT / CAR**
- [ ] Báo cáo có 3 loại nhóm cũ: mỗi tab mở, lưu, đồng bộ như trước; Bảng điều chỉnh, Kết quả (không có khối tiền tệ), đơn vị "đồng/m²" (đất, căn hộ), "đồng/xe" (ô tô).
- [ ] Xuất Word báo cáo chỉ có LAND/CAR: giống bản xuất trước khi có nhánh tàu (so với bản đã lưu hoặc xuất từ `origin/develop`).
- [ ] Gọi `PUT evaluation-result` với `currencyCode=USD` cho nhóm không phải tàu: bị bỏ qua, tiền tệ vẫn VND.
- [ ] Tab Mô tả đất gọi cho nhóm SHIP: 400 (`requireLandGroup`).

### Step 5: Báo cáo cho người dùng

- [ ] Báo cáo ngắn gồm: số test BE và FE, kết quả từng lệnh grep Step 2 (đặc biệt: không còn `else` ngầm CAR, hai bản hằng số, không migration, exporter cũ không đổi), kết luận của final reviewer và các lỗi đã sửa, kết quả E2E (mục nào đạt, mục nào chưa kiểm được và vì sao), danh sách điểm **Khác hệ thống cũ 1–5**, các phần ngoài phạm vi (tab Xem trước, `report/ship_market_overview`, role Ban lãnh đạo, báo cáo công nợ/kê khai).
- [ ] **Không push**, không cập nhật con trỏ submodule ở repo gốc. Hỏi người dùng có đồng ý push `feat/ship-asset-type` của từng submodule và cập nhật con trỏ không.
