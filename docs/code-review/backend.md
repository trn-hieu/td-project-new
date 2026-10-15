# Code review — backend (nhánh `develop` so với `main`)

Phạm vi: `backend/` (Spring Boot), so sánh 19 commit trên `develop` chưa có trên `main`.
Phương pháp: 5 agent review độc lập, mỗi agent phụ trách một mảng nghiệp vụ, chỉ đọc code (không build/run), mức effort cao.

Mục lục theo mảng nghiệp vụ:
1. [In chứng thư / QR](#1-in-chứng-thư--qr)
2. [Kế toán / hồ sơ thẩm định (accounting, appraisal record)](#2-kế-toán--hồ-sơ-thẩm-định)
3. [Bảo mật / config / notification](#3-bảo-mật--config--notification)
4. [Yêu cầu thẩm định (appraisal request)](#4-yêu-cầu-thẩm-định)
5. [MSB / khảo sát / yêu cầu sửa đổi (update request)](#5-msb--khảo-sát--update-request)

Đường dẫn file là tương đối so với `backend/src/main/java/com/thanhdo/tdprojectbe/` trừ khi ghi chú khác.

---

## 1. In chứng thư / QR

### 1.1 [Nghiêm trọng] Truy cập chéo chi nhánh khi xem/lưu/export chứng thư
`service/PrintingCertificateSources.java:55`, dùng ở `PrintingCertificateFormService.java:61`, `PrintingCertificateSaveService.java:66`, `PrintingCertificateExportService.java:43`.

`list()` lọc theo chi nhánh, nhưng `GET /{requestId}`, `PUT /{requestId}`, `GET /{requestId}/word` chỉ load theo id, không kiểm tra phạm vi người gọi. `ExportService` không nhận caller.

**Hậu quả:** NVPI chi nhánh A gọi `PUT /api/printing-certificates/<id của chi nhánh B>` vẫn hoàn tất được yêu cầu, chuyển hồ sơ CONTROLLING sang COMPLETED, ghi/overwrite chứng thư của chi nhánh B. `GET .../word` cũng tải được dữ liệu khách hàng của chi nhánh B. Vi phạm rule CLAUDE.md về `MsbRequestScope`/`RequestScopeSql`. `AppraisalRequestSpecs.visibleById` đã có sẵn để dùng lại.

### 1.2 [Cao] NPE khi export chứng thư thiếu thẩm định viên
`service/PrintingCertificateExportService.java:50` — `List.of(a, b).stream().filter(Objects::nonNull)`: `List.of` đã throw NPE nếu `a`/`b` null trước khi filter chạy, trái với Javadoc "để trống số thẻ thay vì lỗi".

Lỗi tương tự: `CertificateContent.java:57` (`getCustName().toUpperCase()` NPE khi `cust_name` null), `CertificateDocx.java:179` (`text.split` NPE khi `custAddress`/`purpose`/... null) — đều do dữ liệu cũ (legacy row) không được validate khi tạo mới nhưng vẫn tồn tại trong DB.

### 1.3 [Trung bình] Query cảnh báo số seri có thể lỗi toàn bộ + thiếu loại trừ bản ghi đang sửa
`repository/AppraisalCertificateRepository.java:27`, `PrintingCertificateFormService.java:72-75`. `JSON_SEARCH(c.certificate_seri, ...)` chạy trên mọi row; nếu có 1 row dữ liệu cũ không phải JSON hợp lệ → MySQL throw "Invalid JSON text" → mọi lệnh gọi `/serials/contracts` trả 500. Đồng thời API không nhận `requestId` nên không loại trừ hợp đồng đang mở để sửa → mở lại chứng thư đã lưu sẽ cảnh báo serial "trùng" với chính nó.

### 1.4 [Trung bình] Form in lỗi khi asset không có `assetType`
`service/PrintingCertificateFormService.java:151` — `TEXT_ASSET_TYPES.contains(asset.assetType())` với `assetType == null` → NPE. `LegalDocumentAddresses.describeAsset` đã guard điều này nhưng chỗ này không dùng lại, lại hard-code `5`/`6`.

### 1.5 [Trung bình] Race condition khi lưu đồng thời + chứng thư đã soft-delete không lưu lại được
`service/PrintingCertificateSaveService.java:74-79` — `findFirstByAppraisalRequestIdAndDeleted` không lock, 2 request "In và Lưu" đồng thời đều thấy "chưa có" → double-click gây lỗi unique constraint hoặc tạo duplicate. Nếu unique key chỉ theo `appraisal_request_id`, chứng thư đã soft-delete (`deleted=1`) sẽ không bao giờ lưu lại được vì insert mới vẫn đụng unique key.

### 1.6 [Thấp-trung bình] Không validate id thẩm định viên
`model/dto/PrintingCertificateRequest.java` (`priceAppraiser`, `companyRepresentation` chỉ `@NotNull Long`) → có thể lưu id nhân viên đã nghỉ/không có thẻ thẩm định viên, Word in ra dòng "Số thẻ..." trống.

### 1.7 [Thấp] Word export lỗi với ký tự control, danh sách không giới hạn độ dài
`service/CertificateDocx.java` — ký tự control (<0x20) trong text người dùng nhập làm file Word hỏng (invalid XML). Các list (`certificateNos`, `certificateSeris`, `legalDocuments`) chỉ giới hạn độ dài từng item, không giới hạn số lượng item → có thể vượt giới hạn cột TEXT (65535 byte) của MySQL, gây lỗi 500 hoặc bị truncate âm thầm nếu MySQL không strict mode.

### 1.8 Các lỗi nhỏ khác
- `PrintingCertificateService.java:85,109` unbox `Integer status` (nullable) → NPE khi profile status null.
- `PublicUrlProperties` default `http://localhost:5173` khi thiếu env `PUBLIC_BASE_URL` → QR code production có thể trỏ về localhost mà không fail khi start.
- `GET .../word` insert dòng mới vào `appraisal_certificate_qr_code` mỗi lần gọi (GET nhưng đổi state) → bảng tăng vô hạn, không có throttle.
- Trang QR công khai còn phục vụ cả mã MD5-cũ (dễ đoán), không rate-limit → có thể dò ra thông tin khách hàng cũ.
- `roundingMoney` dùng magic number `0` thay vì hằng `AppraisalCertificate.NOT_DELETED`.

---

## 2. Kế toán / hồ sơ thẩm định

### 2.1 [Nghiêm trọng] Xóa hồ sơ (`AppraisalRecord`) không giới hạn phạm vi
`service/AppraisalRecordService.java:83-86` (`canDelete`) chỉ check `isAdmin || (BGD && status >= COMPLETED)`, không check `AppraisalRecordScope`/chi nhánh của profile.

**Hậu quả:** BGD chi nhánh khác (ngoài HN) — vốn chỉ được xem phạm vi chi nhánh mình ở màn `list` — vẫn `DELETE /api/appraisal-records/{id}` được hồ sơ bất kỳ chi nhánh khác, soft-delete toàn bộ profile + lịch sử.

### 2.2 [Cao] Validate import kế toán pass nhưng confirm vẫn fail do giới hạn độ dài cột
`service/AccountingImportedFileParser.java` (`parseRow`) không check độ dài 7 cột `length=500`. "Kiểm tra file" báo OK, "Xác nhận import" fail ở `saveAll` với lỗi truncation 500, rollback toàn bộ transaction, user không biết dòng/cột nào lỗi.

### 2.3 [Trung bình] Cell công thức (formula) trong Excel bị đọc sai qua định dạng hiển thị
`service/AccountingImportedFileParser.java:129-170` — `parseMoney`/`parseDate` chỉ xử lý `CellType.NUMERIC`, cell formula đi qua `text()` (chuỗi hiển thị theo format ô) → tiền dạng "1,500,000.00" bị regex `^-?\d+$` từ chối; ngày theo format `d/m/yyyy` fail strict parse `dd/MM/uuuu`. Nên dùng `getCachedFormulaResultType()`/`evaluator.evaluate`.

### 2.4 [Trung bình] Parse tiền có thể clamp/âm âm thầm
`service/AccountingImportedFileParser.java:133-140` — `Math.round(double)` clamp giá trị vượt range `long` thành `Long.MAX_VALUE` không báo lỗi (khác path text báo "vượt quá giới hạn"); số âm được chấp nhận; không check Thu/Chi đều trống hoặc đều có giá trị.

### 2.5 [Trung bình] Phân trang không ổn định khi nhiều profile cùng `contractId`
`repository/AppraisalRecordRepository.java:72-73` — order chỉ theo year, number, contractId, thiếu tie-breaker (ví dụ id) → với `LIMIT/OFFSET`, hàng trùng thứ tự có thể xuất hiện 2 lần hoặc bị mất khi phân trang.

### 2.6 [Thấp] Escape `LIKE` chưa đầy đủ
`repository/AppraisalRecordRepository.java:90` chỉ escape `%`, thiếu `_` và `\` (khác với `AccountingImportedSpecs.escapeLike` đã làm đúng) → search `A_1` match nhầm `AB1`.

### 2.7 [Trung bình] Race condition số dư tài khoản thu hộ + dùng tài khoản đã soft-delete
`service/CashCollectorAccountingService.java:61,94` — `CreditAccount.addBalance` read-modify-write không có `@Version`/lock → 2 request cùng lúc có thể mất 1 lần cộng tiền. `findFirstByEmployeeId` không filter `deleted` → tài khoản đã xóa vẫn nhận tiền.

### 2.8 [Thấp] Ngày kế toán theo lịch UTC
`model/entity/Accounting.java:128` — `now.toLocalDate()` dùng UTC Clock, request tạo 00:00–07:00 giờ VN bị ghi `accounting_date` lệch 1 ngày so với ngày VN thực tế (chưa xác nhận hệ thống cũ có cùng vấn đề).

---

## 3. Bảo mật / config / notification

### 3.1 [Nghiêm trọng] Đổi mật khẩu có thể mất dữ liệu mật khẩu mới âm thầm
`service/AuthService.java:92-94` + `repository/RefreshTokenRepository.java:21-23` — `employeeRepository.save(employee)` không flush (entity đã managed), sau đó `revokeAllActiveByEmployeeId` là `@Modifying(clearAutomatically = true)` **thiếu** `flushAutomatically`. Auto-flush trước bulk update chỉ cover table `refresh_tokens`; update `employees` (mật khẩu mới) không được flush và bị `em.clear()` loại bỏ.

**Hậu quả:** user đổi mật khẩu, nhận token mới, các refresh token cũ bị revoke — nhưng DB vẫn giữ hash mật khẩu cũ. Lần đăng nhập kế tiếp với mật khẩu mới sẽ fail. Test hiện tại mock repository nên không bắt được lỗi này. Mọi `@Modifying` khác trong repo đều có `flushAutomatically = true` trừ chỗ này.

### 3.2 [Cao] Link xem file Office (viewer) không kiểm tra quyền
`service/OfficeViewerService.java:35-40`, `controller/FileController.java:51-54` — `GET /api/files/documents/{documentId}/viewer` chỉ cần token hợp lệ, không check chi nhánh/phạm vi người gọi, trả presigned S3 URL 5 ngày cho bất kỳ `appraisal_profile_document` theo id tuần tự.

**Hậu quả:** nhân viên bất kỳ chi nhánh/vị trí nào có thể dò id liên tiếp để tải mọi file Office đã upload trên S3, bao gồm tài liệu khảo sát/kiểm soát ngoài phạm vi của họ.

### 3.3 [Trung bình] STOMP chỉ check quyền ở CONNECT, không check ở SEND/SUBSCRIBE
`config/StompJwtAuthInterceptor.java:37-39`, `config/WebSocketConfig.java:28,38` — broker `/queue` không có prefix và không có frame authorization → user có thể forge `SEND` tới `/user/{victimId}/queue/notifications` để giả noti cho người khác. Server chỉ cần push, nên nên chặn hẳn client `SEND`.

### 3.4 [Thấp-trung bình] Session WebSocket sống lâu hơn token
`config/StompJwtAuthInterceptor.java:48-49` — chỉ check JWT 1 lần lúc CONNECT, không re-check `exp` → session nhận noti tiếp dù token đã hết hạn, mật khẩu đã đổi (có intent revoke session khác), hoặc tài khoản đã bị disable/nghỉ việc.

### 3.5 [Thấp] NPE khi `coordinates` JSON là literal `null`
`common/util/UploadedFiles.java:51-53` — `parsed.addAll(null)` throw NPE thay vì trả lỗi validate có ý nghĩa; ảnh hưởng `MsbExchangeService.send`, `ProfileControlService`.

### 3.6 [Thấp] Access log: IP có thể giả, query string chứa PII, status sai khi lỗi
`config/AccessLoggingFilter.java:41-47,57-62` — tin tưởng `X-Forwarded-For` đầu tiên (caller tự set được); log cả query string (email/sđt trong `keyword=`) lưu 360 ngày; log status 200 mặc định trong `finally` nếu exception xảy ra trước khi set status.

### 3.7 [Thấp] QR base URL mặc định `localhost:5173` nếu thiếu env, không fail-fast ở production.

### 3.8 [Thấp] Vi phạm rule CLAUDE.md: literal `"admin"` thay vì dùng `AssignAuth`/hằng số chung
`config/SecurityConfig.java:56,77` — `"admin"` bị duplicate ở 4 nơi khác nhau (`Authorities.ROLE_ADMIN`, `AssignAuth.ROLE_ADMIN`, `EmployeeService.ROLE_ADMIN`).

---

## 4. Yêu cầu thẩm định

### 4.1 [Nghiêm trọng] API tạo yêu cầu (create) thiếu role check và thiếu gating field `collection`
`service/AppraisalRequestFormService.java` (`create` dòng 231-262) — khác với `update` (có `requireCanEdit` và chặn field `collection` trừ admin/BGD), `create` không có cả hai check này. `SecurityConfig` cho phép cả NVTD/NVPI/NVKT gọi `/api/appraisal-requests/**`.

**Hậu quả:** NVTD/NVPI có thể POST `cashCollectorEmployeeId` + `advanceFee` tùy ý → `cashCollectorAccountingService.upsert` tạo bút toán kế toán và cộng tiền vào tài khoản thu hộ của bất kỳ nhân viên nào. `formOptions` cũng trả toàn bộ company/collection account cho mọi caller, không lọc theo `canSeeCollection`.

### 4.2 [Cao] Màn phân công (assign) bỏ qua phạm vi chi nhánh mà list/detail đang áp dụng
`service/AppraisalRequestAssignService.java` (`assignData`, `assign`) — BGD/TPTD chi nhánh khác vẫn `GET/PUT /{id}/assign` được yêu cầu chi nhánh khác (dù Javadoc nói đây là chủ ý "no branch limit", nhưng mâu thuẫn với rule 404-out-of-scope ở nơi khác). `assign` cũng không check `assigneeId` có đúng `Positions.ASSIGNEE_POSITIONS` hay nhân viên đã nghỉ chưa — API trực tiếp có thể gán cho người đã nghỉ/vị trí sai, khiến hồ sơ "biến mất" khỏi scope của người liên quan.

### 4.3 [Cao] Lost-update số dư tài khoản thu hộ (tương tự 2.7, nhưng ở flow create/edit request)
`service/CashCollectorAccountingService.java:58-86`, gọi từ `AppraisalRequestFormService.java:260,385` — read-modify-write `balance` không version/lock, 2 request đồng thời có thể mất 1 khoản cộng.

### 4.4 [Trung bình] `contract_id` không giới hạn độ dài khớp cột DB
`service/AppraisalFormValidator.java:39-42` vs `model/entity/AppraisalRequest.java:62` (`length=45`) — DTO cho phép `@Size(max=100)`, insert MySQL lỗi "Data too long" (500) hoặc bị truncate âm thầm (non-strict mode), bypass luôn check trùng contract.

### 4.5 [Trung bình] `differsOutsideFees` false-positive, buộc sửa phí legacy phải qua luồng phê duyệt
`model/dto/AppraisalRequestFormFields.java` — so sánh `landParcelNo`/`assetType` thô với dữ liệu đã merge lại từ legacy luôn ra khác nhau → NVKD chỉ sửa phí trên request legacy vẫn bị đẩy vào `UpdateRequest` phê duyệt thay vì áp dụng trực tiếp như rule cũ.

### 4.6 [Trung bình] 2 hàm parse tiền không đồng nhất
`service/AppraisalRequestService.java:167-176` (`new BigDecimal(value).longValue()` → "500.000" = 500) vs `AppraisalRequestFormFields.java:68-77` (strip `.`,`,` → "500.000" = 500000) → cùng 1 phí hiển thị khác nhau giữa trang detail và form edit.

### 4.7 [Thấp-trung bình] Trùng lặp/thiếu nhất quán helper quyền & parse type
`AppraisalRequestPermissions`, `AppraisalRequestService.canSeeCollection` là 2-3 bản copy logic `isAdmin`/`canEditCollection`; `parseRequestType` có 3 cách triển khai khác nhau (default 2 ở 2 nơi, default 1 ở assign) → cùng 1 request có thể báo "cá nhân" ở màn assign nhưng "doanh nghiệp" ở detail/form nếu `request_type` null/dirty. `update()` còn gọi `.trim()` trên `getRequestType()` có thể null → NPE 500 (chỗ khác tolerant hơn).

### 4.8 [Thấp, chưa chắc chắn] Đổi contract_id không cập nhật lại `linked_branch`
`service/AppraisalRequestFormService.java` (`update`) — `create` tính `linkedBranch` từ hậu tố contract, `update` không recompute → đổi contract sang chi nhánh khác, `linked_branch` vẫn giữ chi nhánh cũ, ảnh hưởng scope filter.

---

## 5. MSB / khảo sát / update request

### 5.1 [Nghiêm trọng] Màn "Nhập yêu cầu" MSB bỏ qua rule edit contract + luồng phê duyệt + đồng bộ contract
`service/MsbRequestFormService.java:60-85` — `update()`/`applyMsbRequest` cho phép sửa `contractId`, tên khách hàng/công ty, `appraisalRequestCode`, business source **không** qua check `canEditContractId` (admin-only ở path thường), không qua `UpdateRequest` approval, và **không gọi** `contractSync.sync()`/`cashCollectorAccountingService.upsert()` như path `applyDirectly` bình thường.

**Hậu quả:** đổi contract qua màn MSB không đồng bộ `accounting`/`juridical` (vẫn giữ `contract_id` cũ); nếu đang có `UpdateRequest` pending, khi được duyệt sau đó sẽ ghi đè lại baseline sai.

### 5.2 [Cao] Race condition approve/reject `UpdateRequest`
`service/UpdateRequestService.java:168-200` (`decide`) — không `@Version`, không `PESSIMISTIC_WRITE` (khác `findByIdForUpdate` ở nơi khác) → 2 lệnh approve/reject đồng thời đều pass check `isPending()`, có thể vừa apply vừa set status rejected, hoặc apply 2 lần (chạy `contractSync`/`upsert` 2 lần).

### 5.3 [Trung bình] Request gốc bị soft-delete làm `UpdateRequest` pending vĩnh viễn
`service/UpdateRequestService.java:210-217` — `findOriginal` yêu cầu request gốc còn sống; nếu đã soft-delete, cả approve và reject trả `NOT_FOUND`, row kẹt ở status pending mãi, vẫn hiện trong `list` (JOIN không filter delete).

### 5.4 [Trung bình] NPE ở `detail()` khi `request_type` null
`service/UpdateRequestService.java:127` — `.getRequestType().trim()` không guard null (các nơi khác như `applyMsbRequest`/`isPersonal` đều guard).

### 5.5 [Thấp-trung bình, chưa chắc chắn] List/detail của UpdateRequest lệch nhau do case-sensitivity chi nhánh
`service/UpdateRequestService.java:213-215` — list filter SQL case-insensitive theo collation, nhưng `findOriginal` so sánh Java case-sensitive → request với `linked_branch="hn"` hiện trong list của chi nhánh "HN" nhưng detail/approve/reject trả 404.

### 5.6 [Thấp-trung bình, chưa chắc chắn] Validate tọa độ khảo sát chấp nhận chuỗi không phải số thập phân thuần
`service/SurveyService.java:113-139` — `Double.parseDouble` chấp nhận `"10.5f"`, `"1e1"`, `"0x1p3"`... và lưu chuỗi gốc chưa normalize; `CoordinateSearchRepository` dùng `CAST(... AS DECIMAL)` nên tìm kiếm trên map có thể ra tọa độ sai/0 dù khảo sát báo lưu thành công.

### 5.7 [Thấp] Filter thời lượng TAT dùng mốc kết thúc khác với TAT hiển thị ở màn "cancel"
`repository/MsbProfileRepository.java:45-46` — bucket "≤2 ngày/≤5 ngày/>5 ngày" dùng `COALESCE(draft_completed_at, status_complete_at, NOW)`, bỏ qua `cancelled_at`; cột TAT hiển thị (`MsbProfileService.sinceCreated`/`tat`) dừng ở `cancelledAt` → với hồ sơ đã cancel, filter và cột hiển thị lệch nhau.

### 5.8 [Thấp] Upload: check đuôi file có thể bị bypass + transaction DB mở suốt thời gian upload S3
`service/ProfileControlService.java:160-166,89` — file không có dấu `.` trong tên (ví dụ tên file literally là `pdf`) pass whitelist vì `substring(lastIndexOf('.')+1)` trả về cả tên file. `upload` là `@Transactional` nên giữ connection DB suốt khi upload file tới 300MB lên S3.

Lỗi nhỏ liên quan: `MsbExchangeService.searchMentions` chấp nhận keyword chỉ gồm khoảng trắng → `LIKE '%%'` trả ngẫu nhiên 10 nhân viên. `PROFILE_STATUS_*`/`STATUS_*` duplicate hằng số ở nhiều nơi.

---

## Tổng hợp mức độ ưu tiên đề xuất xử lý trước

| # | Mức độ | Vấn đề |
|---|--------|--------|
| 3.1 | Nghiêm trọng | Đổi mật khẩu không flush → không đăng nhập được bằng mật khẩu mới |
| 1.1 | Nghiêm trọng | Truy cập/sửa chéo chi nhánh ở printing-certificate |
| 2.1 | Nghiêm trọng | Xóa hồ sơ accounting/appraisal record không giới hạn phạm vi |
| 4.1 | Nghiêm trọng | Tạo yêu cầu thẩm định thiếu role check, có thể tự cộng tiền thu hộ |
| 5.1 | Nghiêm trọng | Màn MSB sửa contract bỏ qua rule + không đồng bộ |
| 3.2 | Cao | Viewer file Office không kiểm tra quyền, lộ file toàn hệ thống |
| 4.2 | Cao | Màn assign bỏ qua phạm vi chi nhánh |
| 5.2 | Cao | Race condition approve/reject update request |
| 1.2 / 2.7 / 4.3 | Cao | Nhiều NPE do dữ liệu null + race condition số dư tài khoản (lặp ở 2-3 chỗ, nên fix tập trung bằng khóa lạc quan/pessimistic lock) |
