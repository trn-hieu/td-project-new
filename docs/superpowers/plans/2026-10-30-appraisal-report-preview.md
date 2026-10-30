# Migrate tab "Xem trước báo cáo" — Implementation Plan

> **For agentic workers:** dùng superpowers:subagent-driven-development. **Mọi subagent chỉ dùng `model: "sonnet"`.** Mỗi task qua task review trước khi sang task sau. Steps dùng checkbox (`- [ ]`).

**Goal:** Port tab cuối "Xem trước báo cáo" của màn `/appraisal-report/edit-tab/{id}` (`td-project-legacy`) sang màn `/appraisal-reports/{reportId}` (`AppraisalReportEditView.vue`): iframe Office Online viewer, URL có đoạn ngẫu nhiên chống cache, bản preview bỏ bớt ảnh như code cũ, URL public có chữ ký HMAC có hạn, chỉ gọi khi người dùng bấm tab.

**Spec:** hành vi hệ thống cũ (tóm tắt ở mục "Bối cảnh chung" của từng task) + yêu cầu người dùng:
1. Vẫn dùng iframe với Office viewer (`https://view.officeapps.live.com/op/view.aspx?src=`).
2. Vẫn gán chuỗi ngẫu nhiên vào URL để tránh cache.
3. Bản preview không hiển thị toàn bộ ảnh, giữ đúng bộ lọc của code cũ.
4. URL xem preview là permitAll nên phải có chữ ký để kiểm tra.
5. Chỉ khi người dùng bấm tab "Xem trước báo cáo" mới gọi preview.

Viết tắt: **N** = `backend/src/main/java/com/thanhdo/tdprojectbe/`, **NT** = `backend/src/test/java/com/thanhdo/tdprojectbe/`, **W** = `N/common/util/export/word/appraisal/`, **F** = `frontend/src/`. Nhánh `feat/appraisal-report-edit-tab` trong cả `backend` và `frontend`. Commit trong submodule kèm dòng `Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>`; không push, không commit con trỏ submodule ở repo gốc. Tuân theo `.claude/rules/jpa-spring.md` và `.claude/rules/design-principles.md`. Comment, thông báo bằng tiếng Việt như code xung quanh.

---

### Task 1: BE — cờ preview trong exporter Word mới ✅ (backend 90a958c)

**Bối cảnh:** Code cũ dựng preview bằng exporter Word với cờ `isPreview=true`. Bản chép nguyên văn exporter cũ còn ở `N/common/util/export/word/AppraisalReportWordExporter.java` (giữ để đối chiếu): cờ ở dòng 97, kiểm ở 3 chỗ:
- dòng ~963-981: khi preview **bỏ** hai mục cuối "MỘT SỐ HÌNH ẢNH TÀI SẢN" và "MỘT SỐ HÌNH ẢNH VỀ TÀI SẢN SO SÁNH";
- dòng ~5726-5727 và ~6534-6535: khi preview **bỏ** khối ảnh ghi chú `NOTE_PHOTO` (mô tả đất / căn hộ).
Ảnh `ASSET_DIAGRAM` và `PLANNING_INFO` (`W/asset/LandParcelBlock.java:70-78`) vẫn giữ khi preview.

Exporter đang dùng là bản tách lớp `W/AppraisalWordExporter.java` (list `SECTIONS`, state chung `W/ReportWordContext.java`); nó đã **bỏ** cờ preview, chỉ còn comment "(ẩn khi preview)" ở `W/asset/LandDetailTable.java:~190` và `W/asset/ApartmentDetailWriter.java:~194`. `NT/common/util/export/word/appraisal/AppraisalWordExporterParityTest.java` so byte bản cũ với bản mới trên MySQL thật.

**Files:** sửa `W/ReportWordContext.java`, `W/AppraisalWordExporter.java`, `W/asset/LandDetailTable.java`, `W/asset/ApartmentDetailWriter.java`, `N/service/AppraisalReportWordService.java`, `NT/.../appraisal/AppraisalWordExporterParityTest.java`, và mọi chỗ gọi constructor `ReportWordContext`/`AppraisalWordExporter` khác (tìm bằng grep, vd `NT/.../appraisal/asset/AssetTypeWritersTest.java`).

- [x] **Step 1:** `ReportWordContext`: thêm `private final boolean preview;` nhận qua constructor (tham số cuối), getter `public boolean preview()` có Javadoc "Bản xem trước: bỏ mục ảnh tài sản, ảnh so sánh và ảnh ghi chú (như {@code isPreview} của exporter cũ)".
- [x] **Step 2:** `AppraisalWordExporter`: thêm tham số constructor cuối `boolean preview`, lưu field; tách `SECTIONS` thành `CONTENT_SECTIONS` (9 mục đầu, giữ thứ tự) và `IMAGE_SECTIONS` (`AssetImagesSection`, `ComparativeImagesSection`). `writeDataLines` chạy `CONTENT_SECTIONS`, rồi `IMAGE_SECTIONS` chỉ khi `!preview` (comment: "Như exporter cũ: bản xem trước bỏ hai mục ảnh cuối cho đỡ nặng"). Cập nhật Javadoc lớp/hướng dẫn mở rộng cho đúng tên list mới.
- [x] **Step 3:** `LandDetailTable` và `ApartmentDetailWriter`: bọc nguyên khối ảnh `NOTE_PHOTO` bằng `if (!ctx.preview())` (giống dòng 5727/6535 của exporter cũ).
- [x] **Step 4:** `AppraisalReportWordService`: thêm `public ExportedReport export(long reportId, boolean preview)` (`@Transactional(readOnly = true)`) chứa logic hiện tại và truyền `preview` vào exporter; `export(long reportId)` giữ nguyên chữ ký, gọi `export(reportId, false)` (có `@Transactional(readOnly = true)` như cũ). Sửa Javadoc: bỏ câu "không có bản xem trước nên không dùng {@code isPreview}", thêm câu bản xem trước dùng cho tab "Xem trước báo cáo".
- [x] **Step 5:** Parity test: `compareReport` so cho cả hai chế độ `for (boolean preview : new boolean[] {false, true})` — bản cũ gọi `exporter.setIsPreview(preview)`, bản mới truyền `preview`; thông báo lệch ghi rõ chế độ (`"báo cáo " + id + (preview ? " [preview]" : "")`). Giữ `entityManager.flush()/clear()` giữa mỗi lần xuất. Sửa các chỗ gọi constructor khác (truyền `false`).
- [x] **Step 6:** Chạy từ `backend/`: `./mvnw.cmd test -Dtest="AppraisalWordExporterParityTest,AssetTypeWritersTest,AppraisalReportWorkspaceControllerTest"` — parity phải pass và in "Tổng số báo cáo đã so" > 0. Rồi `./mvnw.cmd test` toàn bộ. (Nếu gặp "Unresolved compilation problems" thì xóa `target/test-classes` rồi chạy lại.)
- [x] **Step 7:** Commit `feature: thêm chế độ xem trước cho exporter Word báo cáo thẩm định`.

---

### Task 2: BE — link xem trước có chữ ký + endpoint public ✅ (backend f48405b)

**Bối cảnh:** Code cũ: `GET /appraisal-report/preview/{id}/{randomPath}?isPreview=true` permitAll **không kiểm tra gì**, trả docx `Content-Disposition: inline`; `randomPath` chỉ để Office viewer không cache. Office viewer tải file từ máy chủ Microsoft nên không gửi được JWT → cần URL public có chữ ký.

Hiện có: `N/controller/AppraisalReportWorkspaceController.java` (`/api/appraisal-reports/{reportId}`, có `GET /word` dùng `AppraisalReportWordService.export(reportId)`), `N/service/AppraisalReportWordService.java` (Task 1 thêm `export(long reportId, boolean preview)`), `N/service/ReportEditScope.requireReport(reportId)` (404 nếu không có/đã xóa), `N/config/PublicUrlProperties.java` (`app.public.base-url`, có `qrUrl`, `withoutTrailingSlash`; API được Apache proxy cùng domain dưới `/api`), `N/config/PublicUrlConfig.java`, `N/config/security/JwtProperties.java` (mẫu properties record `@Validated`), `N/config/ClockConfig.java` (bean `Clock`), `N/service/OfficeViewerService.java` (hằng `VIEWER_URL` + `UriUtils.encode`), `N/common/util/OfficeFiles.java`, `N/config/SecurityConfig.java` (dòng `.requestMatchers(HttpMethod.GET, "/api/public/qr/*").permitAll()`), `N/controller/PublicQrController.java` (mẫu controller public). Test mẫu: `NT/controller/AppraisalReportWorkspaceControllerTest.java` (`@WebMvcTest` + `@Import({SecurityConfig.class, JwtConfig.class, TokenService.class})` + `@TestPropertySource`).

**Giá trị cố định:**
- Property: `app.report-preview.secret` (`@NotBlank`), `app.report-preview.ttl` (`Duration`, mặc định `PT10M`). `application.properties`: `app.report-preview.secret=${REPORT_PREVIEW_SECRET:dev-report-preview-secret-change-me-0123456789}`.
- Chuỗi ký: `reportId + ":" + nonce + ":" + expires` (UTF-8), `HmacSHA256`, chữ ký = base64url không padding.
- `nonce`: 16 byte `SecureRandom`, base64url không padding.
- `expires`: epoch second = `clock.instant() + ttl`.
- URL file: `{base-url bỏ / cuối}/api/public/appraisal-reports/{reportId}/preview/{nonce}?expires={expires}&signature={signature}`.
- `viewerUrl` = `https://view.officeapps.live.com/op/view.aspx?src=` + `UriUtils.encode(fileUrl, UTF_8)`.
- Lỗi chữ ký: `BusinessException(ErrorCode.FORBIDDEN, "Liên kết xem trước không hợp lệ hoặc đã hết hạn")`.

**Files:**
- Tạo `N/config/ReportPreviewProperties.java`, `N/config/ReportPreviewConfig.java` (`@Configuration @EnableConfigurationProperties(ReportPreviewProperties.class)`), `N/service/ReportPreviewSigner.java`, `N/service/AppraisalReportPreviewService.java`, `N/model/dto/ReportPreviewLink.java` (`record ReportPreviewLink(String viewerUrl)`), `N/controller/PublicReportPreviewController.java`.
- Sửa `N/common/util/OfficeFiles.java`, `N/service/OfficeViewerService.java`, `N/config/PublicUrlProperties.java`, `N/controller/AppraisalReportWorkspaceController.java`, `N/config/SecurityConfig.java`, `backend/src/main/resources/application.properties`.
- Test: `NT/service/ReportPreviewSignerTest.java`, `NT/service/AppraisalReportPreviewServiceTest.java`, `NT/controller/PublicReportPreviewControllerTest.java`, sửa `NT/controller/AppraisalReportWorkspaceControllerTest.java`, test `OfficeViewerService`/`OfficeFiles` nếu có.

- [x] **Step 1 (TDD):** Viết `ReportPreviewSignerTest` (Clock cố định): ký rồi verify → true; đổi reportId / nonce / expires / 1 ký tự signature → false; `expires` < now → false; signature không phải base64url (vd `"%%%"`) hoặc null → false (không ném). Chạy thấy đỏ.
- [x] **Step 2:** `ReportPreviewProperties` (record `@Validated @ConfigurationProperties("app.report-preview")`, `@NotBlank String secret`, `@DefaultValue("PT10M") Duration ttl`) + `ReportPreviewConfig` + property trong `application.properties`. `ReportPreviewSigner` (`@Component`, `@RequiredArgsConstructor`, phụ thuộc `ReportPreviewProperties`, `Clock`): `long expiresAt()` (epoch second now+ttl), `String sign(long reportId, String nonce, long expires)`, `boolean verify(long reportId, String nonce, long expires, String signature)` — hết hạn hoặc decode lỗi → false, so sánh `MessageDigest.isEqual` trên byte. Test xanh.
- [x] **Step 3:** `OfficeFiles.viewerUrl(String fileUrl)` (public static, chứa hằng URL viewer + encode UTF-8); `OfficeViewerService` dùng nó (bỏ hằng riêng). `PublicUrlProperties.reportPreviewUrl(long reportId, String nonce, long expires, String signature)` theo định dạng ở trên, cạnh `qrUrl`.
- [x] **Step 4 (TDD):** `AppraisalReportPreviewServiceTest` (Mockito, mock `ReportEditScope`, `AppraisalReportWordService`; signer thật với Clock cố định; `PublicUrlProperties("https://td.example.com/")`): `previewLink` gọi `scope.requireReport(id)`, `viewerUrl` bắt đầu bằng `https://view.officeapps.live.com/op/view.aspx?src=` và phần `src` sau decode khớp `https://td.example.com/api/public/appraisal-reports/{id}/preview/{nonce}?expires=..&signature=..` (một `/`), chữ ký trong URL verify được; hai lần gọi cho nonce khác nhau; báo cáo không có → ném lại `BusinessException(NOT_FOUND)` từ scope. `preview` chữ ký sai → `BusinessException` FORBIDDEN và `verifyNoInteractions(wordService)`; đúng → trả kết quả `wordService.export(id, true)`.
- [x] **Step 5:** `AppraisalReportPreviewService` (`@Service`): `@Transactional(readOnly = true) ReportPreviewLink previewLink(long reportId)` và `ExportedReport preview(long reportId, String nonce, long expires, String signature)` — kiểm chữ ký **trước**, rồi `wordService.export(reportId, true)`. Javadoc tiếng Việt nêu khác hệ thống cũ: URL public có chữ ký HMAC hết hạn sau TTL thay vì mở hoàn toàn; chuỗi ngẫu nhiên (nonce) do server sinh và nằm trong chữ ký. Test xanh.
- [x] **Step 6:** Controller:
  - `AppraisalReportWorkspaceController`: `@GetMapping("/preview-link") public ReportPreviewLink previewLink(@PathVariable long reportId)` (theo kiểu trả về của `workspace` trong cùng controller).
  - `PublicReportPreviewController` `@RequestMapping("/api/public/appraisal-reports/{reportId}/preview")`, `@GetMapping("/{nonce}")` với `@RequestParam long expires`, `@RequestParam String signature` → `ResponseEntity<byte[]>`: content type docx, `Content-Disposition` `ContentDisposition.inline().filename(fileName, UTF_8)`, `Cache-Control: no-store` (`CacheControl.noStore()`).
  - `SecurityConfig`: ngay sau dòng `/api/public/qr/*` thêm comment "Như hệ thống cũ /appraisal-report/preview/** công khai cho Office Online tải file, nhưng có chữ ký hết hạn (AppraisalReportPreviewService)." và `.requestMatchers(HttpMethod.GET, "/api/public/appraisal-reports/*/preview/*").permitAll()`.
- [x] **Step 7:** Test controller: `AppraisalReportWorkspaceControllerTest` thêm `/preview-link` → 401 không token, 200 với `bearer("NVTD")` và trả `viewerUrl` (mock `AppraisalReportPreviewService`). `PublicReportPreviewControllerTest` (`@WebMvcTest` cùng cách import security như test mẫu): không token + service trả file → 200, header `Content-Disposition` bắt đầu `inline`, `Cache-Control` chứa `no-store`, content type docx; service ném FORBIDDEN → 403. Bổ sung property `app.report-preview.secret` vào `@TestPropertySource` của mọi test slice cần nạp context có `ReportPreviewConfig` (chạy toàn bộ test để phát hiện).
- [x] **Step 8:** `./mvnw.cmd test` toàn bộ từ `backend/` xanh.
- [x] **Step 9:** Commit `feature: link xem trước báo cáo thẩm định có chữ ký cho Office Online`.

---

### Task 3: FE — tab "Xem trước báo cáo"

**Bối cảnh:** Code cũ (`demo-report-step-2.html:143-147, 254-279, 443-467`): tab cuối "Xem trước báo cáo" gồm cảnh báo "Bản xem trước sẽ không hiển thị toàn bộ hình ảnh, vui lòng tải báo cáo về máy để xem bản đầy đủ.", nút "Tải báo cáo", nút "Làm mới" (nạp lại iframe với URL ngẫu nhiên mới), iframe Office viewer cao `min-height: 75vh`. Cũ nạp iframe ngay khi vào trang — **khác cố ý**: chỉ nạp khi bấm tab.

Backend (Task 2) có `GET /api/appraisal-reports/{reportId}/preview-link` (cần đăng nhập) trả `{ viewerUrl: string }` — mỗi lần gọi là link mới (nonce mới, ký có hạn). Kiểm tra dạng phản hồi thật của controller (`AppraisalReportWorkspaceController.workspace` trả thế nào thì `preview-link` trả y vậy) và cách `api.get` bóc dữ liệu trong `F/lib/api.ts`.

Hiện có: `F/views/appraisal-record/AppraisalReportEditView.vue` (tab cấp 1 theo nhóm: `groupIds` + `useRouteTab('group', groupIds, { resets: ['tab'] })`, `changeGroup` qua `guard.confirmLeave()`, `TabsContent` không force-mount nên tab ẩn bị hủy), `F/api/report-edit/workspace.ts` (`getReportWorkspace`, `downloadReportWord`), `F/components/appraisal-report/edit/DownloadReportButton.vue` (+ `__tests__/DownloadReportButton.spec.ts`), `F/views/document/OfficeViewerView.vue` (mẫu iframe Office viewer, + `F/views/__tests__/OfficeViewerView.spec.ts`), test view `F/views/__tests__/AppraisalReportEditView.spec.ts` (mock `@/api/report-edit/workspace`).

**Files:** sửa `F/api/report-edit/workspace.ts`, `F/views/appraisal-record/AppraisalReportEditView.vue`, `F/views/__tests__/AppraisalReportEditView.spec.ts`; tạo `F/components/appraisal-report/edit/ReportPreviewTab.vue`, `F/components/appraisal-report/edit/__tests__/ReportPreviewTab.spec.ts`.

- [ ] **Step 1:** `workspace.ts`: `export interface ReportPreviewLink { viewerUrl: string }` và `getReportPreviewLink(reportId: number, signal?: AbortSignal)` gọi `/appraisal-reports/${reportId}/preview-link`, JSDoc tiếng Việt (link Office Online có chữ ký, hết hạn sau ít phút; mỗi lần gọi là URL mới để viewer không dùng bản cache).
- [ ] **Step 2 (TDD):** `ReportPreviewTab.spec.ts` (mock `@/api/report-edit/workspace`): mount → gọi `getReportPreviewLink(reportId)` 1 lần, render `iframe[title="Xem trước báo cáo"]` có `src` = `viewerUrl`; hiện câu cảnh báo; bấm "Làm mới" → gọi lần 2 và `src` đổi sang URL mới; API lỗi → hiện thông báo lỗi + nút "Thử lại", bấm thì gọi lại. Chạy thấy đỏ.
- [ ] **Step 3:** `ReportPreviewTab.vue` (props `reportId: number`): khi mount tải link (AbortController, hủy khi unmount và khi tải lại — theo mẫu `load()` trong `AppraisalReportEditView.vue`); thanh trên gồm Alert cảnh báo (nguyên văn câu cũ), nút "Làm mới" (`RefreshCw`, disabled khi đang tải) và `<DownloadReportButton :report-id="reportId" />`; đang tải: `Skeleton` cao `75vh`; lỗi: Alert destructive + "Thử lại" (thông điệp từ `errorMessage(failure, 'Không tải được bản xem trước')`); có link: `<iframe :src="viewerUrl" title="Xem trước báo cáo" class="min-h-[75vh] w-full rounded-lg border" />`. Comment đầu file nêu nguồn cũ và việc chỉ tải khi tab được mở. Test xanh.
- [ ] **Step 4 (TDD):** `AppraisalReportEditView.spec.ts`: thêm mock `getReportPreviewLink`; test vào màn có nhóm → hiện tab "Xem trước báo cáo" (cuối danh sách) và **không** gọi `getReportPreviewLink`; bấm tab → gọi đúng 1 lần với reportId và query `group=preview`. Chạy thấy đỏ.
- [ ] **Step 5:** `AppraisalReportEditView.vue`: hằng `const PREVIEW_TAB = 'preview'`; `groupIds` = id các nhóm + `PREVIEW_TAB`; thêm `<TabsTrigger :value="PREVIEW_TAB">Xem trước báo cáo</TabsTrigger>` cuối `TabsList` và `<TabsContent :value="PREVIEW_TAB"><ReportPreviewTab :report-id="reportId" /></TabsContent>`. Sửa comment đầu file: bỏ "tab Xem trước báo cáo" khỏi danh sách "Chưa có", thêm một câu: tab xem trước chỉ mount (và gọi API) khi được chọn. Test xanh.
- [ ] **Step 6:** Từ `frontend/`: `npx vitest run`, `npx vue-tsc --noEmit` (hoặc script type-check trong `package.json`), lint theo script của `package.json`. Test `AppraisalRequestListView.spec.ts › moves with "Sửa"...` fail sẵn, bỏ qua.
- [ ] **Step 7:** Commit `feature: tab xem trước báo cáo thẩm định`.
