# Khảo sát migrate: Kế toán, nhân sự, xác thực & hạ tầng hệ thống

Phạm vi: nhóm 04. Chỉ đọc code, không sửa.
- Legacy (`main`): `td-project (legacy)/src/main/java/com/jvb/thanhdo`, templates `td-project (legacy)/src/main/resources/templates`, JS `td-project (legacy)/src/main/resources/static/js`.
- Backend mới (`develop`): `td-project-be/src/main/java/com/thanhdo/tdprojectbe`.
- Frontend mới (`develop`): `td-fe/src`.

Chú thích: ✅ đã migrate đủ | ⚠️ migrate một phần | ❌ chưa migrate | ➖ không cần.
Viết tắt đường dẫn: `L:` = legacy `.../com/jvb/thanhdo/`, `BE:` = `.../com/thanhdo/tdprojectbe/`, `FE:` = `td-fe/src/`.

---

## 0. Danh sách role / authority ở legacy

Authority sinh ra ở `L:entity/dto/CustomUserDetails.java#getAuthorities`, **backend mới giữ y nguyên** trong `BE:service/Authorities.java` (so khớp 1-1: position, `admin` khi `employees.role='admin'`, KSV được thêm `TPTD`, role trong bảng role, `BRANCH_HN` khi chi nhánh có HN, `BGD_HN` khi BGD có HN).

| Authority | Nguồn | Ghi chú |
|---|---|---|
| `BGD`, `TPTD`, `NVKD`, `NVTD`, `NVPI`, `NVKT`, `KSV` | chức vụ (`employees.position`) | `L:common/CommonConstants.java` dòng 23-39 |
| `admin` | `employees.role = 'admin'` | |
| `ADMIN_REPORT`, `REPORT_BOARD_OF_DIRECTOR` | bảng role (`RoleRepository.findRoleNamesByEmployeeId`) | |
| `BRANCH_HN`, `BGD_HN` | suy ra từ chi nhánh | |
| `isGlobalAdmin` (không phải authority) | position ∈ {BGD, NVKT, TPTD, KSV} và chi nhánh chứa HN (`L:common/CommonUtils.java:248`) | BE: `AssignAuth.isGlobalAdmin`, `EmployeeService.seesAllBranches` |

---

## 1. Bảng chức năng

### 1.1 Xác thực & tài khoản

| Chức năng | Legacy (file:endpoint) | Backend mới | Frontend mới | Trạng thái | Ghi chú thiếu sót |
|---|---|---|---|---|---|
| Đăng nhập | `L:controller/LoginController.java` GET `/login`, POST `/login_process`; `L:service/impl/LoginServiceImpl.java` (chặn email chưa đăng ký, tài khoản đã nghỉ, remember-me 30 ngày) | `BE:controller/AuthController.java` POST `/api/auth/login`, `/api/auth/refresh`; `BE:service/AuthService.java` (chặn resigned, JWT access 1h, refresh 14d) | `FE:views/auth/LoginView.vue`, `FE:api/auth.ts`, `FE:stores/auth.ts` | ✅ | Thay session bằng JWT (chủ ý). Thông báo lỗi tách "email chưa đăng ký" / "sai mật khẩu" ở cũ; cần xác minh BE mới có tách hay gộp (không ảnh hưởng chức năng). |
| Điều hướng sau đăng nhập theo chức vụ | `LoginController` GET `/` và GET `/login` (đã đăng nhập): BGD→`/admin`; NVKD/TPTD/NVTD/KSV→`/appraisal-request?requestType=1`; còn lại→`/appraisal-records` | Không cần | `FE:views/auth/LoginView.vue#redirectTarget` (về `?redirect=` hoặc `/`), `FE:views/home/HomeView.vue` | ⚠️ | Mới luôn về trang chủ `/` (dashboard lối tắt + thông báo), không redirect theo chức vụ. Có vẻ chủ ý nhưng cần xác nhận với nghiệp vụ. |
| Đăng xuất | `/logout` (Spring Security) | Không có endpoint logout; có `RefreshTokenService.revokeAll` nhưng không gắn endpoint | `FE:stores/auth.ts#logout`, `FE:components/layout/AppHeader.vue` | ⚠️ | Logout chỉ xóa token phía client, refresh token còn hiệu lực đến khi hết hạn 14 ngày (cần xác minh có endpoint khác gọi `revokeAll` không: grep không thấy `logout` trong BE). |
| Quên mật khẩu (gửi mail) | `LoginController` GET `/front-pages-password-reminder`, POST `/forgot-password/{email}`; `LoginServiceImpl#forgotPassword` (token 30 ký tự, hạn 24h); template `front-pages-password-reminder.html`, `email/forgot_password.html` | `BE:controller/AuthController.java` POST `/api/auth/forgot-password`; `BE:service/PasswordResetService.java`, `PasswordResetMailListener.java`, `MailService.java`; `BE:resources/templates/email/forgot_password.html` | `FE:views/auth/ForgotPasswordView.vue` | ✅ | Cải tiến: luôn 204 (không lộ email), giới hạn gửi lại 2 phút. |
| Đặt lại mật khẩu bằng token | `LoginController` GET `/reset-password/{token}`, POST `/reset-password`; template `reset_password_form.html` | `AuthController` GET `/api/auth/reset-password/validate`, POST `/api/auth/reset-password` | `FE:views/auth/ResetPasswordView.vue` | ✅ | Mới không nhận `employeeId` từ client (an toàn hơn). |
| Đổi mật khẩu (người đang đăng nhập) | `LoginController` GET/POST `/update-password`; template `update-password.html` | `AuthController` PUT `/api/auth/password` (yêu cầu mật khẩu hiện tại, trả token mới) | `FE:components/layout/ChangePasswordDialog.vue` (hộp thoại mở từ menu tài khoản) | ✅ | Mới bắt nhập mật khẩu hiện tại (cũ không). |
| Trang 403 `/access-denied` | `LoginController` GET `/access-denied` → `common/page-forbidden` | `BE:config/security/RestSecurityErrorHandler.java` trả JSON 401/403 | Guard `FE:router/index.ts` (cuối file) đẩy về `/` khi thiếu quyền; chỉ có `FE:views/NotFoundView.vue` | ⚠️ | Không có trang "Không có quyền truy cập"; người dùng gõ URL không đủ quyền bị đẩy im lặng về trang chủ. |
| Xử lý lỗi toàn cục | `L:controller/GlobalExceptionHandlerController.java` (log stack, trả `common/error`, fragment `tab-error` cho AJAX) | `BE:common/exception/GlobalExceptionHandler.java` (ApiResponse JSON, có xử lý lock, validation, access denied) | `FE:lib/http.ts` (`errorMessage`), toast; `FE:views/NotFoundView.vue` | ✅ | Kiến trúc khác (REST). Bỏ qua lỗi SockJS không còn cần. |

### 1.2 Nhân sự

| Chức năng | Legacy | Backend mới | Frontend mới | Trạng thái | Ghi chú thiếu sót |
|---|---|---|---|---|---|
| Danh sách nhân viên (tìm kiếm, phân trang, phạm vi chi nhánh) | `L:controller/EmployeeController.java` GET/POST `/admin`; `L:service/impl/EmployeeServiceImpl.java#getPaged` (global admin xem hết, còn lại theo chi nhánh); template `index.html` (thực chất là danh sách nhân viên, không phải dashboard) | `BE:controller/EmployeeController.java` GET `/api/employees`; `BE:service/EmployeeService.java#search` | `FE:views/employee/EmployeeListView.vue`, `FE:api/employees.ts` | ✅ | Quy tắc phạm vi khớp (`seesAllBranches`). |
| Xem / tạo / sửa nhân viên | `EmployeeController` GET `/admin/detail/{id}`, GET `/admin/create`, POST `/admin/create-update-employee/{mode}`; template `change-employee-info.html`; `EmployeeServiceImpl#createOrUpdateEmployee` (trùng email/SĐT, tạo tài khoản credit account cho nhân viên mới) | `EmployeeController` GET `/{id}`, POST, PUT `/{id}`; `BE:service/EmployeeFormService.java` (validate, `createCreditAccount`) | `FE:views/employee/EmployeeFormView.vue` | ✅ | Khớp trường: họ tên, ngày sinh, giới tính, địa chỉ, SĐT, email, mật khẩu, chức vụ, chi nhánh (nhiều), số thẻ thẩm định, ngày vào làm, ngày nghỉ việc. Lương đang bị ẩn ở cũ ("Tạm thời bỏ mức lương") và mới không đổi lương: khớp. Cờ `isBusinessSource` cũ chỉ giữ nguyên, không có UI sửa: khớp. |
| Xóa (mềm) nhân viên | POST `/admin/delete-employee/{id}` | `EmployeeController` DELETE `/{id}` | `FE:api/employees.ts#deleteEmployee`, `EmployeeListView.vue` | ✅ | |
| Bật/tắt quyền phân công hồ sơ (chỉ `admin`) | POST `/admin/allow-assign/{id}`; `SecurityConfig` yêu cầu `admin` | PUT `/api/employees/{id}/allow-assign` (`SecurityConfig` yêu cầu `admin`) | `EmployeeListView.vue` (`setEmployeeAllowAssign`) | ✅ | |
| Xuất Excel danh sách nhân viên | GET `/admin/export`; `L:controller/component/EmployeesExcelExporter.java` | GET `/api/employees/export`; `BE:common/util/export/excel/EmployeeExcelExporter.java` | `EmployeeListView.vue` (`exportEmployees`) | ✅ | Cùng 8 cột, cùng tên file. |
| Gợi ý @mention nhân viên | `L:controller/api/EmployeeApiController.java` GET `/api/employees/mentions` (dùng trong CKEditor trao đổi) | `BE:controller/MsbExchangeController.java` GET `/mention-employees` | thuộc màn trao đổi MSB (nhóm khác) | ✅ | Chỉ phục vụ trao đổi MSB. |
| Danh sách nhân viên theo chi nhánh | `EmployeeApiController` GET `/api/employees?branchCodes=` (dùng ở appraisal-request-detail.js) | `BE:controller/AppraisalRequestFormController.java` GET `/cash-collectors` | màn yêu cầu thẩm định (nhóm khác) | ✅ | Đổi tên endpoint theo ngữ cảnh. |

### 1.3 Kế toán (Hạch toán thu chi)

| Chức năng | Legacy | Backend mới | Frontend mới | Trạng thái | Ghi chú thiếu sót |
|---|---|---|---|---|---|
| Danh sách hạch toán + tổng PT/PC lũy kế + lọc (từ khóa, tiểu mục, tài khoản, khoảng ngày, "chưa có tiểu mục/tài khoản") | `L:controller/AccountingController.java` GET `/accounting`, POST `/accounting`; `L:service/impl/AccountingServiceImpl.java#getListAccountingByUser`, `getCumulativeTotals`; `accounting/accounting.html` | `BE:controller/AccountingController.java` GET `/api/accounting`, GET `/filter-options`; `BE:model/dto/AccountingScope.java` (OWN/ALL/BRANCHES khớp cũ) | `FE:views/accounting/AccountingListView.vue`, `FE:api/accounting.ts` | ✅ | Cờ `noSubsection`/`noCreditAccount` thay `__NULL__` (chủ ý). |
| Tạo hạch toán PT/PC | GET `/accounting/create`, POST `/accounting/create`; `create-accounting.html`; `AccountingServiceImpl#createOrUpdateAccounting` | `AccountingController` POST `/api/accounting`, GET `/form-options`, GET `/contracts`; `BE:service/AccountingCreateService.java` | `FE:views/accounting/AccountingFormView.vue` | ✅ | Logic số dư / bút toán con thu phí tạm ứng: đã port (`CreditAccountLedger`, `AccountingPosting`); độ khớp chi tiết từng nhánh nên có test đối chiếu (cần xác minh). |
| Xem chi tiết hạch toán | GET `/accounting/view/{id}`, POST `/accounting/detail` (NVKD chỉ xem bút toán của mình) | `AccountingController` GET `/{id}` (chặn NVKD xem của người khác, `AccountingService.java:105`) | `FE:views/accounting/AccountingDetailView.vue` | ✅ | |
| Sửa hạch toán (gửi yêu cầu chờ duyệt) | GET `/accounting/update/{id}`, POST `/accounting/create` (nhánh update tạo `UpdateRequest` + `MAccounting`) | `AccountingController` GET `/{id}/form`, PUT `/{id}`; `AccountingEditService` | `AccountingFormView.vue` (route `accounting/:id/edit`) | ✅ | |
| Duyệt / từ chối yêu cầu sửa hạch toán | GET `/accounting/view/{id}?approve=true` + `UpdateRequestController` | `BE:controller/UpdateRequestController.java` GET `/{id}/accounting`, POST `/{id}/approve`, `/{id}/reject`; `AccountingUpdateApproval` | `FE:views/update-request/AccountingUpdateRequestDetailView.vue` | ✅ | Javadoc đầu `AccountingController` còn ghi "chưa port: duyệt yêu cầu sửa" nhưng thực tế đã port (chỉ là comment lỗi thời). |
| Xóa hạch toán (trong 3 ngày, hoàn số dư, kể cả bút toán con) | DELETE `/api/accounting/{id}` (`AccountingApiController`; `AccountingServiceImpl#deleteAccounting`) | `AccountingController` DELETE `/{id}`; `BE:service/AccountingDeleteService.java` | `AccountingListView.vue` | ✅ | |
| Upload lô hạch toán (Excel) + file mẫu | POST `/accounting/upload` (đồng bộ cũ), GET `/accounting/download`; API mới của cũ `/api/accounting/import/validate` + `/confirm` (`AccountingApiController`); `AccountingAsyncService` xử lý bất đồng bộ rồi gửi notification `ACCOUNTING_IMPORT` | `BE:controller/AccountingImportController.java` GET `/template`, POST `/validate`, `/confirm` (BGD, NVKT) | `FE:components/accounting/AccountingBatchUploadDialog.vue` | ⚠️ | Mới xử lý đồng bộ trong một transaction (chủ ý, `AccountingImportService` javadoc). Notification `ACCOUNTING_IMPORT` có trong enum FE/BE nhưng **BE không tạo** loại thông báo này (chỉ `MsbExchangeService` tạo notification) → mất thông báo kết quả import; có thể chấp nhận vì đã đồng bộ, cần xác nhận. Chưa dùng `/accounting/upload` cũ (đã bỏ, hợp lý). |
| Thêm nhanh Tiểu mục (subsection) khi tạo hạch toán | POST `/accounting/subsection/create` (`SubsectionService#createOrUpdateSubsection`; nút "+" ở `create-accounting.html`, chỉ BRANCH_HN và BGD/NVKT) | Không có (grep `subsection` chỉ ở các controller báo cáo/hạch toán, không có endpoint tạo) | Không có | ❌ | Không thể tạo tiểu mục mới ở hệ thống mới. |
| Thêm nhanh Tài khoản ghi nợ / ghi có (credit account) | POST `/accounting/creditAccount/create`, POST `/accounting/creditAccount/detail` (`CreditAccountService`; modal ở `create-accounting.html`, BGD/NVKT; `static/js/accounting/createAccounting.js`) | Không có endpoint tạo/chi tiết; chỉ tự tạo tài khoản khi tạo nhân viên (`EmployeeFormService#createCreditAccount`) | Không có | ❌ | Không thể tạo tài khoản thu/chi công ty hoặc chi nhánh mới ở hệ thống mới. |

### 1.4 Hạch toán phí khai thác & Pháp lý (accounting/*)

| Chức năng | Legacy | Backend mới | Frontend mới | Trạng thái | Ghi chú thiếu sót |
|---|---|---|---|---|---|
| Danh sách phí khai thác | GET `/accounting/exploitation-fee`; `AccountingServiceImpl#getListExploitationFeeByUser`; `exploitation-fee.html` | `BE:controller/ExploitationFeeController.java` GET `/api/accounting/exploitation-fees`, `/filter-options` | `FE:views/exploitation-fee/ExploitationFeeListView.vue`, `FE:api/exploitation-fee.ts` | ✅ | Mới siết quyền: BRANCH_HN và BGD/NVKT cả ở API (cũ chỉ ẩn menu; NVKD và chi nhánh khác gõ URL vẫn vào được). Chủ ý, có ghi trong `SecurityConfig`. |
| Tạo phí khai thác | GET `/accounting/create-exploitation-fee`, POST `/api/accounting/create-exploitation-fee`; `create-exploitation-fee.html` | `ExploitationFeeController` POST, GET `/{id}/form`, GET `/contract-stats`; `ExploitationFeeEditService` | `FE:views/exploitation-fee/ExploitationFeeFormView.vue` | ✅ | |
| Xem / sửa phí khai thác | GET `/accounting/exploitation-fee/{mode}/{id}`, POST `/api/accounting/update-exploitation-fee` | `ExploitationFeeController` GET `/{id}`, PUT `/{id}` | `ExploitationFeeDetailView.vue`, `ExploitationFeeFormView.vue`, `components/accounting/ExploitationFeeHistory.vue` | ✅ | |
| Xóa phí khai thác | DELETE `/api/accounting/{id}` (dùng chung) | `ExploitationFeeController` không có DELETE; `AccountingController` DELETE `/api/accounting/{id}` có thể dùng chung (chưa chắc áp dụng cho bút toán phí khai thác: cần xác minh `Accounting.isUserEntered()`) | `ExploitationFeeListView.vue:401` nút "Xóa hạch toán (chưa có)" bị `disabled`; javadoc dòng 47 "Chưa port: Xóa" | ⚠️ | UI chưa nối; nếu API dùng chung đủ thì chỉ cần nối UI. |
| Import phí khai thác (validate + confirm) | `AccountingApiController` POST `/api/accounting/exploitation-fee/import/validate`, `/confirm` | `BE:controller/ExploitationFeeImportController.java` GET `/template`, POST `/validate`, `/confirm` | `FE:components/accounting/ExploitationFeeImportDialog.vue` | ✅ | |
| Pháp lý (accounting/juridical) | GET `/accounting/juridical` → `accounting/juridical.html`; API do `JuridicalController` legacy (ngoài danh sách, nhóm khác) | `BE:controller/JuridicalController.java` GET `/by-contract`, `/bill-exists`, POST | `FE:views/juridical/JuridicalFormView.vue`, `FE:api/juridical.ts`, `FE:lib/juridical.ts` | ✅ | Quyền BGD, NVKT khớp menu cũ. |
| Xuất Excel hạch toán (`AccountingExcelExporter`) | Chỉ dùng trong `L:controller/ReportController.java:391` (báo cáo thu chi, không phải màn hạch toán) | `BE:controller/AccountingReportController.java` | `FE:views/report/AccountingReportView.vue` | ✅ | Thuộc nhóm báo cáo (đã có kế hoạch migrate riêng `plans/2026-10-26-migrate-accounting-report.md`). |

### 1.5 Hạch toán đã lưu trữ (accounting-imported)

| Chức năng | Legacy | Backend mới | Frontend mới | Trạng thái | Ghi chú thiếu sót |
|---|---|---|---|---|---|
| Danh sách + tìm kiếm thu chi lưu trữ | `L:controller/AccountingImportedController.java` GET `/accounting-imported`; `accounting-imported/list.html` | `BE:controller/AccountingImportedController.java` GET `/api/accounting-imported` | `FE:views/imported/AccountingImportedListView.vue`, `FE:api/accounting-imported.ts` | ✅ | Chỉ authenticated, menu ẩn theo 4 email: khớp (`FE:lib/accounting-imported.ts`). |
| Import file (validate + confirm) | POST `/accounting-imported/import/validate`, `/confirm` | POST `/api/accounting-imported/import/validate`, `/confirm` | `FE:components/accounting-imported/AccountingImportedImportDialog.vue` | ✅ | Comment ở `FE:router/index.ts:442` "Chưa port chức năng import" là của route `appraisal-request-imported` (nhóm khác), không phải accounting-imported. |

### 1.6 Thông báo, bản đồ, địa giới hành chính

| Chức năng | Legacy | Backend mới | Frontend mới | Trạng thái | Ghi chú thiếu sót |
|---|---|---|---|---|---|
| Danh sách thông báo + đánh dấu đã đọc tất cả | `L:controller/api/NotificationApiController.java` GET `/api/notifications`, POST `/read-all` | `BE:controller/NotificationController.java` GET, GET `/unread-count`, POST `/read-all` | `FE:components/layout/NotificationPopover.vue`, `FE:stores/notifications.ts`, `FE:views/home/HomeNotifications.vue`, `FE:api/notifications.ts` | ✅ | |
| Đẩy thông báo realtime (WebSocket/STOMP) | `L:config/WebSocketBrokerConfig.java` (`/ws`, SockJS, broker `/specific`, origin cố định `thamdinhgiathanhdo.net`) | `BE:config/WebSocketConfig.java` (`/ws`, broker `/queue`, origin từ cấu hình), `StompJwtAuthInterceptor`, `NotificationPushListener` | `FE:stores/notifications.ts` (nối lại WebSocket) | ✅ | Đổi giao thức (bỏ SockJS, destination `/queue`), FE/BE mới đồng bộ. |
| Nguồn phát sinh thông báo | Legacy phát: (a) @mention ở trao đổi thông tin (`ExchangeMessageServiceImpl:110`), (b) kết quả import lô hạch toán (`AccountingAsyncService`) | Mới chỉ phát: @mention ở `MsbExchangeService:232` | | ⚠️ | Thiếu thông báo import lô hạch toán (xem mục 1.3). Còn trao đổi thông tin của yêu cầu thẩm định thường (không phải MSB) thuộc nhóm khác, cần xác minh có tạo thông báo hay không. |
| API key Google Maps theo chi nhánh | `L:controller/api/GoogleMapApiController.java` GET `/api/google-map/key` | `BE:controller/MapController.java` GET `/api/maps/key`; `BE:service/MapKeyService.java` | `FE:api/maps.ts`, `FE:lib/google-maps.ts`, `FE:lib/google-routes.ts` | ✅ | |
| Tỉnh / quận-huyện / phường-xã | `L:controller/api/ProvinceDistrictWardApiController.java` GET `/api/provinces`, `/districts`, `/wards`, `/wards-by-province` (có gộp tỉnh sáp nhập `findMergedProvinceIds`) | `BE:controller/AdministrativeUnitController.java` GET `/api/provinces`, `/districts`, `/wards` (hỗ trợ `provinceId`) | Dùng trực tiếp trong các form (LocationFields.vue, ReportJuridicalListView.vue, ...) | ✅ | Hợp nhất `wards-by-province` vào `/wards?provinceId=`. Việc gộp tỉnh sáp nhập: cần xác minh trong `AdministrativeUnitService`. |
| Email phân công yêu cầu (`email/assignment.html`) | Chỉ khai báo, code gửi đã bị comment (`AppraisalRequestServiceImpl:571-590`) | Không có | Không có | ➖ | Đã tắt ở legacy, không cần migrate. |
| Email duyệt (`email/send_approve.html`) | `AppraisalRecordsServiceImpl#sendMailApprove` có nhưng mọi chỗ gọi đều bị comment (dòng 1572, 1945) | Không có | Không có | ➖ | Dead code ở legacy. |
| Email thông báo cập nhật báo cáo (`email/update_report_notice.html`) | `EmailServiceImpl#sendMailNoticeWithFiles`, chỗ gọi bị comment (`UploadServiceImpl:178`) | Không có | Không có | ➖ | Dead code ở legacy. |
| Email quên mật khẩu | `email/forgot_password.html` | `BE:resources/templates/email/forgot_password.html`, `footer.html` | | ✅ | Xem mục 1.1. |

### 1.7 Dashboard trang chủ & menu

| Chức năng | Legacy | Backend mới | Frontend mới | Trạng thái | Ghi chú thiếu sót |
|---|---|---|---|---|---|
| Trang chủ | Không có dashboard thật; `/` redirect theo chức vụ. `index.html` thực chất là danh sách nhân viên `/admin` | | `FE:views/home/HomeView.vue`, `HomeShortcuts.vue`, `HomeNotifications.vue` | ➖ | Trang mới (lối tắt theo quyền + thông báo gần đây), không có tương đương cũ. |
| Menu trái theo role | `L:../resources/templates/fragment/mainnav-container.html` | | `FE:components/layout/nav-items.ts`, `nav-visibility.ts`, `AppSidebar.vue` | ✅ | Xem bảng so sánh mục 2. Mục "Xem tài liệu" chưa có `to` (nhóm tài liệu, không thuộc nhóm này). |
| Header: chuông thông báo, đổi mật khẩu, đăng xuất | `fragment/header.html` | | `FE:components/layout/AppHeader.vue`, `NotificationPopover.vue`, `ChangePasswordDialog.vue` | ✅ | |

### 1.8 Hạ tầng hệ thống

| Hạng mục | Legacy | Backend mới | Trạng thái | Ghi chú |
|---|---|---|---|---|
| `SecurityConfig` (phân quyền URL, BCrypt, remember-me, session) | `L:config/SecurityConfig.java` | `BE:config/SecurityConfig.java` (stateless JWT, phân quyền theo `/api/**`, nhiều chỗ siết chặt hơn cũ có chú thích) | ✅ | Xem mục 3 so sánh quyền. `/ws/**` mở ở HTTP, xác thực ở frame STOMP CONNECT. |
| `AccessLoggingFilter` | `L:config/AccessLoggingFilter.java` | `BE:config/AccessLoggingFilter.java` (port, bỏ phần đọc SavedRequest) | ✅ | Request bị 401/403 từ filter chain không được ghi (chủ ý). |
| `S3Config` | `L:config/S3Config.java` (S3Client, thư mục `report/`, `profile/`) | `BE:config/S3Properties.java` + `BE:service/S3LinkService` (presigned URL tải file) | ⚠️ | Mới chỉ cấu hình tạo presigned URL tải file. Phần upload S3 (nếu cần) thuộc các nhóm upload/báo cáo, cần xác minh khi khảo sát nhóm đó. |
| `GsonConfig`, `LocalDateAdapter`, `ModelMapperConfig` | `L:config/*` | Không cần (Jackson/record DTO) | ➖ | |
| `schedule/AppraisalImageResizeSchedule` (mỗi phút: resize ảnh hồ sơ, tạo `backup_resize_path`, `word_export_image_path` 220 ppi) | `L:schedule/AppraisalImageResizeSchedule.java` (`@Scheduled "0 * * * * ?"`) | **Không có**: BE mới không có `@Scheduled` / `@EnableScheduling` nào. Entity `AppraisalImage` ghi chú "do lịch resize của hệ thống cũ ghi, đây chỉ đọc" | ❌ | Mới phụ thuộc legacy chạy job nền. Khi tắt legacy, ảnh upload mới sẽ không có ảnh resize / ảnh xuất Word. Cần quyết định: port job hoặc xử lý ngay lúc upload. |
| `schedule/AppraisalConstructionImageWordExportSchedule` (mỗi phút, giây 30) | `L:schedule/...` | Không có | ❌ | Như trên, ảnh công trình. |
| `schedule/ComparisonAssetImageWordExportSchedule` (mỗi phút, giây 40) | `L:schedule/...` | Không có | ❌ | Như trên, ảnh tài sản so sánh. |
| `validation/LegalDocumentAssetTypeValidator` | `L:validation/` (dùng ở `AppraisalReportServiceImpl:146,187`) | `BE:service/ReportLegalDocumentValidator.java` | ✅ | Đã port (javadoc ghi rõ). |
| `config/*Runner` — migrate dữ liệu một lần | `AppraisalProfileDocumentS3MigrateRunner`, `ComparisonAssetImageSyncBackfillRunner`, `AppraisalImageWordExportBackfillRunner`, `LandDetailLandUseRowsMigrationRunner` (mặc định bật), `LandDetailPlotsStructureMigrationRunner` (mặc định bật), `LandPriceAreaInsideSyncRunner`, `LandPriceAreaInsideValidationRunner`, `RoadFrontageSizeTxtMigrationRunner` | Không có | ➖ | Đều là backfill / migrate dữ liệu một lần, bật bằng cờ `thanhdo.*.enabled` (hai runner LandDetail bật mặc định `matchIfMissing=true`). Không cần port, nhưng phải chạy xong trên DB trước khi cắt legacy. |
| `BidvAppraisalExcelExportRunner` | `L:config/BidvAppraisalExcelExportRunner.java` (xuất file `bidv_appraisal_requests_2026.xlsx` khi bật cờ, mặc định tắt) | Không có | ➖ | Xuất một lần theo yêu cầu; nếu còn cần xuất định kỳ thì phải làm màn riêng (cần xác minh với nghiệp vụ). |

---

## 2. So sánh ma trận role / menu (fragment `mainnav-container.html` ↔ `nav-items.ts`)

| Mục menu | Legacy (`sec:authorize`) | Mới (`FE:components/layout/nav-items.ts`) | Khớp |
|---|---|---|---|
| Danh sách nhân viên | BGD | `['BGD']` → `/employees` | ✅ |
| Yêu cầu thẩm định (cá nhân / doanh nghiệp) | mọi người | không giới hạn | ✅ |
| Hồ sơ MSB (4 mục) | mọi người | không giới hạn | ✅ |
| Hồ sơ thẩm định | BGD, NVKT, TPTD, NVTD, NVPI, NVKD, KSV | `ALL_STAFF` (cùng 7) | ✅ |
| Phê duyệt yêu cầu | BGD, TPTD, KSV | `['BGD','TPTD','KSV']` | ✅ (BE thêm `admin`) |
| Hạch toán (cha) | BGD, NVKT, NVKD | `['BGD','NVKT','NVKD']` | ✅ |
| · Hạch toán thu chi | (theo cha) | không giới hạn thêm | ✅ |
| · Hạch toán phí khai thác | `BRANCH_HN` và (BGD, NVKT) | `['BGD','NVKT']` + `branchHnOnly` | ✅ |
| · Pháp lý | BGD, NVKT | `['BGD','NVKT']` | ✅ |
| In ấn | BGD, NVPI | `['BGD','NVPI']` | ✅ |
| Báo cáo (cha) | BGD, NVKT, NVKD, NVTD, NVPI, TPTD, KSV, admin | `[...ALL_STAFF,'admin']` | ✅ |
| · Pháp lý / BTC / Ngân hàng / Công nợ / Thẩm định / Kinh doanh / Chứng thư / Thu chi / Số dư / Theo dõi HĐ / Kế toán / Tháng / MSB / Thu tạm ứng / VCB / VCBNEO / TSSS / Ban lãnh đạo | như `mainnav-container.html` dòng 142-160 | giống từng mục | ✅ (ngoại lệ: "Báo cáo công nợ" mới thêm `ADMIN_REPORT`, cũ menu không có, nhưng `CommonUtils.validatePermission` cũ cho phép role này; thuộc nhóm báo cáo) |
| Cơ sở dữ liệu (cha) | BGD, NVKT, NVKD, NVTD, NVPI, TPTD, KSV | `ALL_STAFF` | ✅ |
| · Tìm kiếm hồ sơ | mọi người (theo cha) | `/coordinate-search`, route `['BGD','NVKT','NVKD','NVTD','NVPI','TPTD','KSV']` | ✅ |
| · Xem tài liệu | mọi người (theo cha) | chưa có `to` | ❌ (nhóm khác) |
| · Hồ sơ lưu trữ, Thu chi | chỉ 4 email: loan.lt1, tam.nm1, doan.bt, huong.ltn @tdvc.com.vn | `accountRestricted` + `ACCOUNTING_IMPORTED_EMAILS` (cùng 4 email) | ✅ |
| · Pháp lý theo tỉnh thành / Pháp lý chung | BGD, TPTD, KSV | `['BGD','TPTD','KSV']` | ✅ |
| Mục ẩn ở legacy (Menu sơ bộ, Menu báo cáo, DEMO) | `display:none` | đã bỏ | ➖ |

### Ma trận quyền URL/API (SecurityConfig)

| Vùng | Legacy | Mới | Nhận xét |
|---|---|---|---|
| `/admin/**` ↔ `/api/employees/**` | BGD; `allow-assign` chỉ `admin` | BGD; PUT `allow-assign` chỉ `admin` | Khớp |
| `/accounting/upload,download,subsection/create,creditAccount/create` ↔ `/api/accounting/import/**` | BGD, NVKT | BGD, NVKT | Khớp (hai endpoint tạo subsection/creditAccount không có ở mới) |
| `/accounting/**`, `/api/accounting/**` | BGD, NVKT, NVKD | BGD, NVKT, NVKD | Khớp |
| `/api/accounting/exploitation-fees/**` | (không có quy tắc riêng, rơi vào BGD/NVKT/NVKD) | `BRANCH_HN` và (BGD, NVKT) | Mới chặt hơn, chủ ý |
| `/accounting-imported` | authenticated | authenticated | Khớp |
| `/update-request` | authenticated (chỉ menu BGD/TPTD/KSV) | BGD, TPTD (KSV có TPTD), admin | Mới chặt hơn |
| `/api/notifications`, `/api/provinces`..., `/api/google-map/key` ↔ `/api/maps/key` | authenticated | authenticated | Khớp |
| `/login`, `/forgot-password`, `/reset-password`, `/qr/**`, preview báo cáo | permitAll | `/api/auth/*` (4 endpoint), `/ws/**`, `/api/public/qr/*`, `/api/public/appraisal-reports/*/preview/*` permitAll | Khớp ý nghĩa |

---

## 3. Thiếu sót quan trọng (xếp theo ưu tiên)

1. **Cao — Job nền xử lý ảnh chưa migrate** (`L:schedule/AppraisalImageResizeSchedule`, `AppraisalConstructionImageWordExportSchedule`, `ComparisonAssetImageWordExportSchedule`). BE mới không có `@Scheduled`. Entity mới chỉ đọc các cột `backup_resize_path` / `word_export_image_path` và ghi chú "do lịch resize của hệ thống cũ ghi". Nếu tắt legacy, ảnh mới upload sẽ không có ảnh resize/ảnh xuất Word, ảnh hưởng xuất báo cáo Word. Cần chọn: port job vào BE mới hoặc resize ngay lúc upload.
2. **Cao — Không tạo được Tiểu mục và Tài khoản ghi nợ/ghi có mới** (`POST /accounting/subsection/create`, `/accounting/creditAccount/create`, `/accounting/creditAccount/detail`). Chỉ BGD/NVKT dùng nhưng là dữ liệu danh mục nền cho hạch toán; thiếu thì phải sửa DB tay.
3. **Trung bình — Xóa hạch toán phí khai thác chưa nối UI** (`FE:views/exploitation-fee/ExploitationFeeListView.vue:47,401`); cần xác minh `DELETE /api/accounting/{id}` dùng chung được.
4. **Trung bình — Không có trang 403** và logout không thu hồi refresh token. Gõ URL không đủ quyền bị đẩy về `/` mà không báo lý do; token refresh còn hiệu lực 14 ngày sau khi bấm Đăng xuất (xác minh có chỗ gọi `RefreshTokenService.revokeAll` không).
5. **Trung bình — Thông báo kết quả import lô hạch toán** (`ACCOUNTING_IMPORT`) không còn được tạo (mới import đồng bộ). Nếu import file lớn bị timeout HTTP thì cần phương án bất đồng bộ + thông báo như cũ.
6. **Thấp — Điều hướng sau đăng nhập** không theo chức vụ (BGD→danh sách nhân viên, nhóm kinh doanh/thẩm định→yêu cầu thẩm định). Cần xác nhận với nghiệp vụ là chủ ý.
7. **Thấp — Nguồn phát sinh thông báo**: mới chỉ có @mention ở trao đổi MSB; cần xác minh trao đổi thông tin của yêu cầu thẩm định thường (nhóm khác) có phát thông báo hay không.
8. **Thấp — S3Config**: mới chỉ có presigned URL tải file; cần xác minh phần upload S3 ở nhóm upload/báo cáo.
9. **Thấp — Comment lỗi thời** ở `BE:controller/AccountingController.java` (javadoc "chưa port: duyệt yêu cầu sửa" nhưng đã port ở `UpdateRequestController`).
10. **Lưu ý khi cắt legacy**: các runner migrate dữ liệu (xem mục 1.8) không cần port nhưng `LandDetailLandUseRowsMigrationRunner` và `LandDetailPlotsStructureMigrationRunner` bật mặc định, phải bảo đảm đã chạy xong trên DB production.

## 4. Thống kê

Tổng 55 dòng trong các bảng mục 1.1 đến 1.8 (một số dòng gộp nhiều endpoint):
- ✅ Đã migrate đủ: 36
- ⚠️ Migrate một phần: 7 (điều hướng sau đăng nhập, đăng xuất, trang 403, upload lô hạch toán, xóa phí khai thác, nguồn phát thông báo, S3Config)
- ❌ Chưa migrate: 5 (thêm Tiểu mục, thêm/chi tiết Tài khoản, 3 job schedule: tính 3 dòng; riêng tiểu mục và tài khoản là 2 dòng)
- ➖ Không cần: 7 (3 email dead code, trang chủ mới, Gson/ModelMapper, nhóm runner migrate một lần, BidvRunner)
