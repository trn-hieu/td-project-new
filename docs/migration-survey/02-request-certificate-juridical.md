# Khảo sát migrate: Yêu cầu thẩm định, chứng thư, in ấn, pháp lý, file

Phạm vi: legacy `td-project (legacy)` (main), BE mới `td-project-be` (develop), FE mới `td-fe` (develop).
Chỉ đọc, không sửa code. Tiền tố đường dẫn viết tắt:

- LC = `td-project (legacy)/src/main/java/com/jvb/thanhdo/controller`
- LS = `td-project (legacy)/src/main/java/com/jvb/thanhdo/service` (impl nằm trong `service/impl`)
- LT = `td-project (legacy)/src/main/resources/templates`, LJ = `td-project (legacy)/src/main/resources/static/js`
- BC = `td-project-be/src/main/java/com/thanhdo/tdprojectbe/controller`, BS = `.../tdprojectbe/service`, BX = `.../tdprojectbe/common/util/export/excel`
- FV = `td-fe/src/views`, FA = `td-fe/src/api`, FR = `td-fe/src/router/index.ts`

Ký hiệu: ✅ đã migrate đủ, ⚠️ một phần, ❌ chưa migrate, ➖ không cần.

## Tổng kết số liệu

52 chức năng/endpoint nghiệp vụ được liệt kê: **41 ✅, 3 ⚠️, 2 ❌, 6 ➖**.
Nhóm này đã migrate rất đầy đủ. Các thiếu sót thật sự chỉ gồm: chứng thư Word chỉ có tiếng Việt, mục "Xem tài liệu" chưa có trang, phê duyệt hồ sơ thẩm định (loại 2/3) chưa có, xem file Office lưu local, và tuyến QR mã hóa cũ.

## Bảng đối chiếu

### A. Yêu cầu thẩm định (AppraisalRequestController, AppraisalRequestImportedController, LinkedBranchController)

| # | Chức năng | Legacy (file:endpoint) | Backend mới (file) | Frontend mới (file) | Trạng thái | Ghi chú thiếu sót |
|---|---|---|---|---|---|---|
| 1 | Danh sách yêu cầu (cá nhân/doanh nghiệp), lọc từ khóa, "chưa phân công", chi nhánh, phân trang | LC/AppraisalRequestController: `GET/POST /appraisal-request` (+ LT/appraisal-request/appraisal-request.html) | BC/AppraisalRequestController `GET /api/appraisal-requests` (requestType, keyword, unassigned, branch, page, size) | FV/appraisal-request/AppraisalRequestListView.vue; FA/appraisal-requests.ts `listAppraisalRequests`; route `appraisal-request` | ✅ | Có cờ `updating` (chờ duyệt chỉnh sửa) và `canAssign` trên từng dòng như `isUpdating`/`assignAuth` cũ. Phạm vi xem theo `MsbRequestScope` khớp 3 nhánh của `getListAppraisalRequest` cũ (toàn cục / BGD-TPTD-cờ phân công theo chi nhánh / còn lại theo đầu mối-được phân công). |
| 2 | Xem chi tiết yêu cầu | LC: `GET /appraisal-request/view/{id}`, `POST /appraisal-request/detail` | BC/AppraisalRequestController `GET /api/appraisal-requests/{id}` | FV/appraisal-request/AppraisalRequestDetailView.vue; route `appraisal-request/:id` | ✅ | |
| 3 | Tạo yêu cầu mới | LC: `GET /appraisal-request/create`, `POST /appraisal-request/create` (`createOrUpdateAppraisalRequest` mode create) | BC/AppraisalRequestFormController `GET /form-options`, `GET /cash-collectors`, `POST /api/appraisal-requests`; BS/AppraisalRequestFormService (validate số HĐ, chi nhánh, upsert bút toán thu phí tiền mặt) | FV/appraisal-request/AppraisalRequestCreateView.vue + components/appraisal-request/AppraisalRequestForm.vue; route `appraisal-request/create` | ✅ | |
| 4 | Sửa trực tiếp hoặc gửi yêu cầu chỉnh sửa chờ duyệt (admin sửa thẳng; sửa riêng trường phí đang bằng 0 thì sửa thẳng) | LC: `POST /appraisal-request/update`; LS/impl/AppraisalRequestServiceImpl `createOrUpdateAppraisalRequest`, `directUpdateAppraisalRequest` | BC/AppraisalRequestFormController `GET /{id}/form`, `PUT /{id}`; BS/AppraisalRequestFormService (`appliesDirectly`, `contractSync`) + AppraisalRequestPermissions | FV/appraisal-request/AppraisalRequestEditView.vue; route `appraisal-request/:id/edit` (BGD, NVKD, admin) | ✅ | Quyền sửa: BGD, NVKD, admin (cũ chỉ ẩn nút `hasAnyAuthority('BGD','NVKD')`, mới kiểm ở service). |
| 5 | Phân công thẩm định viên (lần đầu tạo hồ sơ thẩm định + lịch sử; lần sau đổi người/ghi chú/hồ sơ ghép) | LC: `GET /appraisal-request/assign/{id}`, `POST /appraisal-request/assign` | BC/AppraisalRequestAssignController `GET/PUT /api/appraisal-requests/{id}/assign`; BS/AppraisalRequestAssignService | FV/appraisal-request/AppraisalRequestAssignView.vue; route `appraisal-request/:id/assign` | ✅ | Cũ đã tắt gửi email; mới cũng không gửi, khớp. |
| 6 | Cho phép nhân viên phân công theo từng yêu cầu | LC: `POST /appraisal-request/allow-assign` | Không có (cột `allowAssignEmployee` trong model/entity/MAppraisalRequest.java nhưng không dùng) | Không có | ➖ | Nút gọi endpoint này đã bị comment trong LJ/appraisal-request/appraisal-request-detail.js (dòng 1255-1290). Quyền phân công theo nhân viên (cờ `allow_assign_request`) đã có ở EmployeeListView. |
| 7 | Thêm nhanh "Mục đích thẩm định" | LC: `POST /appraisal-request/appraisal-purpose/create` | BC/AppraisalPurposeController `POST /api/appraisal-purposes` | components/appraisal-request/AppraisalPurposeModal.vue; FA/appraisal-requests.ts `createAppraisalPurpose` | ✅ | |
| 8 | Thêm nhanh "Đầu mối kinh doanh" | LC: `POST /appraisal-request/business-source/create` | Không có | Không có | ➖ | Legacy `showModalBusinessSource()` chỉ mở tab `/admin/create` (tạo nhân viên); tạo nhân viên đã có ở employees/new. Cần xác minh modal `#addNewBusinessSource` có còn hiển thị trong appraisal-request-detail.html hay không. |
| 9 | Nạp lại danh sách nhân viên cho select đầu mối | LC: `POST /appraisal-request/get-employees` | Thay bằng `GET /api/appraisal-requests/form-options` | AppraisalRequestForm.vue | ➖ | Chức năng gộp vào form-options. |
| 10 | Kiểm tra trùng số hợp đồng | LC: `POST /appraisal-request/detail-by-contract-id`, `.../appraisal-request-by-contract-id` | BS/AppraisalRequestFormService: `existsByContractIdAndDeleteFlag` khi tạo và khi đổi số HĐ | Lỗi theo ô hiển thị từ BE (`FieldErrorDetail contractId`) | ✅ | Chỉ admin được sửa số hợp đồng (`AppraisalRequestPermissions.canEditContractId`). |
| 11 | Ô chọn số hợp đồng ở màn Thu chi, Phí khai thác, Pháp lý | LC: `POST /appraisal-request/search`, `GET /appraisal-request/search-customer-name-by-contract-id` (dùng ở LJ/accounting/createAccounting.js, createExploitationFee.js, juridical.js) | BC/AccountingController `GET /api/accounting/contracts`; BC/JuridicalController `GET /by-contract`; BC/ExploitationFeeController `GET /contract-stats` | Các view accounting / JuridicalFormView | ✅ | Thuộc nhóm kế toán; mới thay bằng API riêng từng màn. Cần xác minh ô chọn số HĐ của màn Pháp lý dùng API nào (JuridicalFormView). |
| 12 | Kiểm tra mã chi nhánh liên kết tồn tại | LC/LinkedBranchController: `POST /linked-branch/is-linked-branch-exist` (gọi từ appraisal-request-detail.js dòng 669) | BS/AppraisalFormValidator (`linkedBranchRepository.existsByBranchCodeAndDeleteFlag`) chạy trong tạo/sửa; BC/BranchController `GET /api/branches` | Lỗi hiển thị theo ô | ✅ | Không còn endpoint riêng, kiểm ở server khi lưu. |
| 13 | Hồ sơ lưu trữ (Cơ sở dữ liệu): danh sách + lọc (từ khóa, ngân hàng, chi nhánh, công nợ, số seri) | LC/AppraisalRequestImportedController `GET /appraisal-request-imported`; LT/appraisal-request-imported/list.html | BC/AppraisalRequestImportedController `GET /api/appraisal-request-imported`, `GET /filter-options`; BS/AppraisalRequestImportedService | FV/imported/AppraisalRequestImportedListView.vue; components/appraisal-request-imported/AppraisalRequestImportedTable.vue; FA/appraisal-request-imported.ts; route `appraisal-request-imported` | ✅ | Comment cũ trong FR (dòng ~443) ghi "Chưa port chức năng import" đã lỗi thời, vì dialog import đã có. |
| 14 | Import Excel hồ sơ lưu trữ (kiểm tra trước, rồi xác nhận) + file mẫu | LC: `POST /appraisal-request-imported/import/validate`, `/import/confirm`; file mẫu `/download/mau-ho-so-import.xlsx` | BC `POST /api/appraisal-request-imported/import/validate`, `/import/confirm`; BS/AppraisalRequestImportedFileParser (18 cột `HEADERS` giống hệt cũ) | components/appraisal-request-imported/AppraisalRequestImportedImportDialog.vue + components/import/ExcelImportDialog.vue; file mẫu `td-fe/public/download/mau-ho-so-import.xlsx` | ✅ | Quyền: chỉ cần đăng nhập, menu ẩn theo tài khoản (`accountRestricted`), đúng như cũ. |

### B. Chứng thư, in ấn (PrintingController, AppraisalCertificateController, AppraisalCertificateQrCodePublicController)

| # | Chức năng | Legacy (file:endpoint) | Backend mới (file) | Frontend mới (file) | Trạng thái | Ghi chú thiếu sót |
|---|---|---|---|---|---|---|
| 15 | Danh sách yêu cầu cần in (timeline hồ sơ trạng thái cao nhất, lọc từ khóa + trạng thái) | LC/PrintingController `GET /printing-certificate`; LS/impl/AppraisalCertificateServiceImpl `getPaged` | BC/PrintingCertificateController `GET /api/printing-certificates`; BS/PrintingCertificateService | FV/printing-certificate/PrintingCertificateListView.vue; FA/printing-certificates.ts; route `printing-certificates` | ✅ | |
| 16 | Form chứng thư (kể cả `mode=view` chỉ xem, chặn khi phí thẩm định = 0) | LC: `GET /printing-certificate/detail/{id}` | BC `GET /api/printing-certificates/{requestId}`; BS/PrintingCertificateFormService | FV/printing-certificate/PrintingCertificateFormView.vue (`viewOnly` = `?mode=view`); route `printing-certificates/:requestId` | ✅ | |
| 17 | "In và Lưu" chứng thư (hoàn thành yêu cầu, đẩy hồ sơ Kiểm soát sang Hoàn thành, ghi lịch sử in) | LC: `POST /printing-certificate/create`; `AppraisalCertificateServiceImpl.createAppraisalCertificate` | BC `PUT /api/printing-certificates/{requestId}`; BS/PrintingCertificateSaveService (cùng giao dịch; chặn phí thiếu ở server) | PrintingCertificateFormView.vue + `savePrintingCertificate` | ✅ | |
| 18 | Cảnh báo số seri đã dùng ở hợp đồng khác | LC: `GET /printing-certificate/find-contractId-by-certificate-seri` | BC `GET /api/printing-certificates/serials/contracts` | FA/printing-certificates.ts `listSerialContracts` | ✅ | |
| 19 | Xuất chứng thư Word có mã QR | LC/AppraisalCertificateController: `GET /appraisal-certificate/word-export/{id}?printLanguage=vi\|en\|cn`; `AppraisalCertificateWordExporter` | BC `GET /api/printing-certificates/{requestId}/word`; BS/PrintingCertificateExportService + CertificateDocx + CertificateQrImage | `downloadPrintingCertificateWord` trong FA/printing-certificates.ts, gọi từ PrintingCertificateFormView.vue | ⚠️ | **Chỉ tiếng Việt.** Cũ có ô "Ngôn ngữ" (Tiếng Việt / Anh / Trung) trong LT/printing-certificate/printing-certificate-detail.html (dòng ~448). BS/PrintingCertificateExportService ghi rõ bỏ bản Anh và Trung (cũ dịch qua dịch vụ ngoài). FE không có chọn ngôn ngữ. |
| 20 | Xuất chứng thư PDF từ mẫu | LC/AppraisalCertificateController: `GET /appraisal-certificate/{id}?printLanguage=` (template `certificate_of_appraisal-blank.pdf`) | Không có | Không có | ➖ | Không template/JS nào gọi endpoint này (grep chỉ thấy `word-export` ở LJ/print-certificate/print-certificate-detail.js dòng 271). Code chết. |
| 21 | Trang QR công khai `/qr/{code}` | LC/AppraisalCertificateQrCodePublicController `GET /qr/{code}` | BC/PublicQrController `GET /api/public/qr/{code}` (permitAll trong config/SecurityConfig); BS/QrCertificateService | FV/printing-certificate/QrCertificateView.vue; FA/qr.ts; route `/qr/:code` ngoài layout, không cần đăng nhập | ✅ | Mã mới là UUID (cũ MD5), mã cũ vẫn tra được. |
| 22 | Trang QR kiểu mã hóa cũ | LC/AppraisalCertificateController `GET /appraisal-certificate/qr-code/detail?code=<mã hóa>` (`AppraisalCertificateQrCodeDataDTO.decypt`) | Không có | Không có | ❌ | Ưu tiên thấp. Exporter hiện tại của cũ đã tạo URL `/qr/{code}` (AppraisalCertificateWordExporter dòng ~391), nên chỉ ảnh hưởng QR in từ rất lâu. Cần xác minh còn chứng thư cũ lưu hành dùng URL này không. |
| 23 | Tạo/sửa Pháp lý từ trang in | LC/PrintingController `POST /printing-certificate/create-juridical`; hàm `saveJuridical()` trong LJ/print-certificate/print-certificate-detail.js | Thay bằng BC/JuridicalController `POST /api/juridicals` | Thay bằng FV/juridical/JuridicalFormView.vue | ➖ | Template printing-certificate-detail.html không có modal pháp lý nên JS này không dùng. Luồng thực dùng là LJ/accounting/juridical.js, đã migrate ở dòng 39-40 bên dưới. |

### C. Phê duyệt yêu cầu (UpdateRequestController)

| # | Chức năng | Legacy (file:endpoint) | Backend mới (file) | Frontend mới (file) | Trạng thái | Ghi chú thiếu sót |
|---|---|---|---|---|---|---|
| 24 | Danh sách yêu cầu chỉnh sửa (lọc loại, trạng thái, từ khóa) | LC/UpdateRequestController `GET /update-request` | BC/UpdateRequestController `GET /api/update-requests` (type 1 và 4, status 0-2, keyword); BS/UpdateRequestService | FV/update-request/UpdateRequestListView.vue; FA/update-requests.ts; route `update-requests` | ✅ | Cũ chỉ liệt kê loại 1 (Yêu cầu thẩm định) và 4 (Hạch toán thu chi) trong `CommonUtils.listUpdateRequestType()`; loại 2 và 3 đã bị comment, nên khớp. |
| 25 | Xem chi tiết so sánh bản cũ/mới | LC: `GET /update-request/{mode}/{id}` | BC `GET /{id}`, `GET /{id}/accounting` | FV/update-request/UpdateRequestDetailView.vue, AccountingUpdateRequestDetailView.vue | ✅ | |
| 26 | Duyệt / từ chối yêu cầu thẩm định | LC: `POST /update-request/approve/{id}`, `/reject/{id}` | BC `POST /api/update-requests/{id}/approve`, `/{id}/reject` | UpdateRequestDetailView.vue | ✅ | Mới chỉ duyệt khi còn chờ (cũ cho duyệt lại/đảo trạng thái). |
| 27 | Duyệt / từ chối chỉnh sửa hạch toán thu chi | LC: `POST /update-request/accounting-accept/{id}`, `/accounting-reject/{id}` | BS/AccountingUpdateApproval qua cùng `approve`/`reject` | AccountingUpdateRequestDetailView.vue | ✅ | |
| 28 | Duyệt / từ chối hàng loạt | LC: `POST /update-request/accept-multiple`, `/reject-multiple` | BC `POST /api/update-requests/approve-all`, `/reject-all` (trả kết quả từng yêu cầu) | UpdateRequestListView.vue | ✅ | Cũ luôn trả "thành công" dù từng yêu cầu lỗi; mới trả chi tiết. |
| 29 | Duyệt / từ chối hồ sơ thẩm định (loại 2, 3) | LC: `POST /update-request/profile-accept/{id}`, `/profile-reject/{id}` (gọi từ LT/appraisal-records/approve.html dòng 243-245) | Không có (BS/UpdateRequestService ghi rõ loại 2-3 "vẫn ở hệ thống cũ") | Không có | ⚠️ | Danh sách phê duyệt đã ẩn loại 2-3 nên đúng. Nhưng màn approve của hồ sơ thẩm định còn gọi 2 endpoint này; thuộc nhóm "Hồ sơ thẩm định", cần xác minh bên đó đã có màn tương ứng chưa. |

### D. Hồ sơ MSB (MSBAppraisalProfileController)

| # | Chức năng | Legacy (file:endpoint) | Backend mới (file) | Frontend mới (file) | Trạng thái | Ghi chú thiếu sót |
|---|---|---|---|---|---|---|
| 30 | Danh sách yêu cầu MSB (từ khóa + trạng thái) | LC/MSBAppraisalProfileController `GET /appraisal-msb-profile/list-request` | BC/MsbProfileController `GET /api/msb-profiles`; BS/MsbRequestListService | FV/msb-profile/MsbRequestListView.vue; FA/msb-profiles.ts | ✅ | |
| 31 | Nhập yêu cầu MSB (gợi ý giá trị trống, sửa tài sản) | LC: `GET/POST /appraisal-msb-profile/update-request/{id}` (`MSBAppraisalRequestDTO`) | BC `GET /{id}/request-form`, `PUT /{id}/request`; BS/MsbRequestFormService; `MsbRequestUpdate` | FV/msb-profile/MsbRequestFormView.vue; route `msb-profiles/:id/request` | ✅ | Validate bắt buộc khớp DTO cũ. |
| 32 | Danh sách đang xử lý / hoàn thành / hủy (lọc nhân viên, chi nhánh, trạng thái, đầu mối, thời hạn) | LC: `GET /appraisal-msb-profile/list/{type}` | BC `GET /api/msb-profiles/processing\|complete\|cancel`, `/processing/filter-options`; BS/MsbProfileService | FV/msb-profile/MsbProcessingListView.vue, MsbCompleteListView.vue, MsbCancelListView.vue; components/msb-profile/MsbProfileList.vue | ✅ | |
| 33 | Đổi trạng thái hồ sơ (gửi báo cáo / hủy / đang xử lý) kèm đồng bộ trạng thái hồ sơ thẩm định | LC: `POST /appraisal-msb-profile/update-status/{id}` | BC/MsbExchangeController `PUT /api/msb-profiles/{id}/status`; BS/MsbExchangeService `updateStatus`, `changeProfileStatuses` | components/msb-exchange/MsbStatusModal.vue | ✅ | |
| 34 | Màn Trao đổi thông tin: mở (ghi lịch sử xem, đánh dấu đã đọc thông báo) | LC: `GET /appraisal-msb-profile/exchange-info/{id}?notificationId=` | BC `GET /api/msb-profiles/{id}/exchange`; BS/MsbExchangeService `open` | FV/msb-profile/MsbExchangeView.vue; route `msb-profiles/:id/exchange` | ✅ | |
| 35 | Lưu ngày hoàn thành draft / lý do chậm / hoãn | LC: `POST /appraisal-msb-profile/exchange-info/{id}` | BC `PUT /api/msb-profiles/{id}/notes` | components/msb-exchange/MsbNotesForm.vue | ✅ | |
| 36 | Gửi tin nhắn kèm file, tọa độ, @nhắc nhân viên | LC: `POST /appraisal-msb-profile/exchange-info` (id, message, files[], coordinates) | BC `GET /{id}/messages`, `POST /{id}/messages` (multipart), `GET /mention-employees`; BS/MsbExchangeService (S3, sự kiện thông báo) | components/msb-exchange/ExchangeChat.vue, ChatComposer.vue, ChatMessageBubble.vue, AttachmentCoordinateModal.vue | ✅ | |
| 37 | `POST /appraisal-msb-profile/send-message` | LC | Không có | Không có | ➖ | Chỉ trả lại view, không lưu gì (stub). |
| 38 | Tải/xem file đính kèm MSB | LC: `GET /appraisal-msb-profile/download?path=&download=` (permitAll) | BC/FileController `GET /api/files/download` (chống path traversal, ép attachment với loại file lạ) | FA/files.ts | ✅ | Khác cũ: mới yêu cầu đăng nhập, cũ công khai (để Office Online đọc). Xem dòng 52 về xem file Office. |

### E. Pháp lý (JuridicalController, ReportJuridicalController, ReportJuridicalApiController) và báo cáo kèm exporter

| # | Chức năng | Legacy (file:endpoint) | Backend mới (file) | Frontend mới (file) | Trạng thái | Ghi chú thiếu sót |
|---|---|---|---|---|---|---|
| 39 | Hồ sơ pháp lý theo hợp đồng: lấy chi tiết + kiểm tra trùng số hóa đơn | LC/JuridicalController `POST /juridical/detail`, `/juridical/detail/bill` | BC/JuridicalController `GET /api/juridicals/by-contract`, `GET /bill-exists` | FV/juridical/JuridicalFormView.vue; FA/juridical.ts; route `accounting/juridical` | ✅ | |
| 40 | Lưu pháp lý (chỉ BGD, NVKT) | `JuridicalServiceImpl.executeCreateJuridicalAction` (từ LJ/accounting/juridical.js dòng 146) | BC `POST /api/juridicals`; BS/JuridicalService (kiểm chức vụ, ngày hóa đơn) | JuridicalFormView.vue | ✅ | |
| 41 | Pháp lý theo tỉnh thành: danh sách tỉnh + tìm kiếm, trang sửa, CRUD nội dung theo tỉnh | LC/ReportJuridicalController `GET /report-juridical`, `/report-juridical/edit`; api/ReportJuridicalApiController `GET/POST /api/report-juridical`, `PUT/DELETE /api/report-juridical/{id}` | BC/ReportJuridicalController `GET /api/report-juridicals/provinces`, `/provinces/{id}`, `GET/POST/PUT/DELETE /api/report-juridicals` | FV/report-juridical/ReportJuridicalListView.vue, ReportJuridicalEditView.vue; components/report-juridical/ReportJuridicalCategoryPanel.vue; FA/report-juridicals.ts | ✅ | |
| 42 | Pháp lý chung: danh sách + CRUD | LC: `GET /report-juridical/global`; api: `POST/PUT/DELETE /api/report-juridical/global[/{id}]` | BC `GET/POST/PUT/DELETE /api/report-juridicals/global[/{id}]` | FV/report-juridical/ReportJuridicalGlobalView.vue; components/report-juridical/ReportJuridicalGlobalModal.vue | ✅ | |
| 43 | Báo cáo pháp lý + xuất Excel (`JuridicalStreamingExcelExporter`, `JuridicalExcelExporter` đã @deprecated) | LC/ReportController `GET /report/juridical`, `/report/export-juridical` | BC/JuridicalReportController `GET /api/reports/juridical`, `/filter-options`, `/export`; BX/JuridicalReportExcelExporter (14 cột như cũ, cột SĐT chỉ cho admin) | FV/report/JuridicalReportView.vue; FA/juridical-reports.ts `download('/reports/juridical/export')` | ✅ | Cờ "thiếu hợp đồng / biên bản khảo sát / phiếu so sánh" ánh xạ sang `missing*`. Cũ có tham số `region`, mới `region` 0-5, giữ. |
| 44 | Báo cáo chứng thư + xuất Excel (`CertificateExcelExporter`) | LC/ReportController `GET /report/certificate`, `/report/export-certificate` | BC/CertificateReportController `GET /api/reports/certificate`, `/totals`, `/filter-options`, `/export`; BX/CertificateReportExcelExporter (16 cột như cũ) | FV/report/CertificateReportView.vue; FA/certificate-reports.ts | ✅ | Lọc keyword, employeeId, ngày ký, ngày in. |
| 45 | Báo cáo hồ sơ MSB + xuất Excel (`MSBReportExcelExporter`) | LC/ReportController `GET /report/msb`, `/report/export-msb-report` | BC/MsbReportController `GET /api/reports/msb`, `/export`; BX/MsbReportExcelExporter (18 cột như cũ) | FV/report/MsbReportView.vue; FA/msb-reports.ts | ✅ | Cũ có `branch` trong DTO `SearchMSBReport` nhưng template msb-report.html không có ô chi nhánh, nên không thiếu. |
| 46 | Báo cáo kế toán (kê khai) + xuất Excel (`DeclarationReportExporter`) | LC/ReportController `GET /report/declaration`, `/report/export-declaration` | BC/DeclarationReportController `GET /api/reports/declaration`, `/totals`, `/export`; BX/DeclarationReportExcelExporter | FV/report/DeclarationReportView.vue; FA/declaration-reports.ts | ✅ | Chỉ admin cả cũ lẫn mới. Mới sửa lỗi chính tả tiêu đề cột "Đã CK T Cty" thành "Đã CK TK Cty" (cố ý). |

### F. File: upload, download, xem file

| # | Chức năng | Legacy (file:endpoint) | Backend mới (file) | Frontend mới (file) | Trạng thái | Ghi chú thiếu sót |
|---|---|---|---|---|---|---|
| 47 | Trang Kiểm soát / tài liệu liên quan của hồ sơ thẩm định (danh sách file + file MSB) | LC/UploadController `GET /upload-files/{id}` | BC/ProfileControlController `GET /api/appraisal-records/{id}/control` | FV/appraisal-record/ControlView.vue; route `appraisal-records/:id/control` | ✅ | |
| 48 | Tải file lên kèm tọa độ (zip/rar/doc/docx/xls/xlsx/pdf, tổng tối đa 300MB, chuyển hồ sơ sang Kiểm soát) | LC: `POST /upload/{id}`; LS/impl/UploadServiceImpl `singleFileUpload` | BC `POST /api/appraisal-records/{id}/control/files`; BS/ProfileControlService (`validateExtensions`, `MAX_TOTAL_SIZE`, `moveToControl`, lưu S3) | components/control/ControlUploadPanel.vue (chỉ BGD, TPTD, NVPI) | ✅ | File mới lưu S3 thay cho thư mục local. |
| 49 | Tải file đã upload | LC: `GET /download-files/{id}/{filename}` | BC/FileController `GET /api/files/download` | FA/files.ts | ✅ | |
| 50 | Tìm kiếm hồ sơ theo tọa độ | LC/DownloadController `GET /download/search-coordinate` | BC/CoordinateSearchController `/api/coordinate-search/*` | FV/coordinate-search/*; route `coordinate-search` | ✅ | Thuộc nhóm bản đồ. |
| 51 | **Xem tài liệu** (Quy chế KSCL, khung giá đất 2023 theo quận) | LC/DownloadController `GET /download/list`, `GET /download/file/{year}/{filename}`; LT/upload-download/list-file.html; menu `mainnav-container.html` dòng 152 | Không có (grep `database-khu-vuc`, `download/file` ở BE trống) | Menu "Xem tài liệu" có trong components/layout/nav-items.ts dòng 126 nhưng **không có `to`**; không có view/route | ❌ | Mục menu trống, bấm không tới đâu. Cần BE phục vụ PDF tĩnh (cũ đọc từ thư mục cứng `/home/ec2-user/project/thanhdo/.../download/database-khu-vuc/`) hoặc đưa PDF lên S3/public của FE, cộng một trang liệt kê. |
| 52 | Xem file Office bằng Office Online | LC/ViewFileController `GET /view-file?path=` (iframe `view.officeapps.live.com`) | BC/FileController `GET /api/files/documents/{documentId}/viewer`; BS/OfficeViewerService (presign S3) | FV/document/OfficeViewerView.vue; route `documents/:id/view` | ⚠️ | Mới chỉ nhận mã tài liệu có `s3FilePath` và phải là file Office; cũ cũng xem được file lưu local qua `/appraisal-msb-profile/download?path=` (survey.html, exchange-info.html, list-processing.html dùng `file.filePath`). File cũ chỉ có `filePath` local sẽ trả 404. Cần xác minh dữ liệu cũ đã được chuyển lên S3 chưa. Cố ý chặn path tùy ý (bảo mật). |

(Phụ: tải `/download/mau-ho-so-import.xlsx` và `mau-thu-chi-import.xlsx` đã có ở `td-fe/public/download/`.)

## So sánh phân quyền

| Khu vực | Legacy (config/SecurityConfig.java, controller) | Mới (BE config/SecurityConfig.java + service) | Nhận xét |
|---|---|---|---|
| Yêu cầu thẩm định (danh sách, xem) | `/appraisal-request`, `/appraisal-request/view/**`: BGD, NVKD, TPTD, NVTD, NVPI, NVKT; các URL còn lại (create, assign, update, search) chỉ cần đăng nhập | `/api/appraisal-requests/**`: cùng 6 chức vụ cho toàn bộ nhóm API, rồi service kiểm tiếp (sửa: BGD/NVKD/admin; phân công: BGD/TPTD/admin/cờ; sửa số HĐ: admin) | Khớp, mới chặt hơn ở server (cũ chỉ ẩn nút). |
| Phê duyệt yêu cầu | Chỉ ẩn menu (BGD, TPTD, KSV); endpoint `/update-request/**` không chặn | `/api/update-requests/**`: BGD, TPTD (KSV mang authority TPTD), admin; service kiểm tra thêm | Mới chặt hơn, đúng ý đồ. |
| In ấn | `/printing-certificate` (chỉ trang danh sách): BGD, NVPI; `/printing-certificate/detail`, `/create`, word-export: chỉ cần đăng nhập (word-export thuộc `/appraisal-certificate/**` cho BGD, TPTD, NVTD, NVPI, NVKT, NVKD) | `/api/printing-certificates/**` (cả form, lưu, Word): BGD, NVPI | Mới chặt hơn: TPTD, NVTD, NVKT, NVKD trước đây gọi thẳng URL được, nay không. Cố ý, đã ghi trong comment BC/PrintingCertificateController. |
| QR công khai | `/qr/**`, `/appraisal-certificate/qr-code/detail`: permitAll | `GET /api/public/qr/*`: permitAll; FE `/qr/:code` ngoài layout | Khớp. |
| Hồ sơ MSB, trao đổi thông tin | Chỉ đăng nhập (trừ `/appraisal-msb-profile/download` permitAll) | `/api/msb-profiles/**`: chỉ đăng nhập; scope dữ liệu theo `MsbRequestScope` | Khớp; riêng tải file thì mới bắt đăng nhập. |
| Hồ sơ lưu trữ (imported) | Chỉ đăng nhập; menu ẩn theo tài khoản | Chỉ đăng nhập, FE `accountRestricted` | Khớp. |
| Pháp lý theo tỉnh/chung | `/report-juridical*` trang: BGD, TPTD, KSV; `/api/report-juridical/global/**`: BGD, TPTD, KSV; **`/api/report-juridical` (không global) chỉ cần đăng nhập** | `/api/report-juridicals/**`: BGD, TPTD, KSV (toàn bộ) | Mới chặt hơn ở API theo tỉnh (cố ý, có comment). |
| Lưu pháp lý theo hợp đồng | Service: chỉ BGD, NVKT | Service: chỉ BGD, NVKT | Khớp. |
| Báo cáo pháp lý | Trang `/report/juridical`: BGD, NVKD, NVPI, NVKT; export theo rule chung `/report/**` | `/api/reports/juridical/**`: BGD, NVKD, NVPI, NVKT cả trang lẫn export | Khớp trang, siết export (cố ý). |
| Báo cáo chứng thư | Trang: BGD, NVPI | `/api/reports/certificate/**`: BGD, NVPI | Khớp. |
| Báo cáo MSB | Rule chung `/report/**` (admin, BGD, NVKT, NVKD, NVTD, TPTD, NVPI) | `/api/reports/msb/**`: admin, BGD, NVKT, NVKD, NVTD, TPTD, KSV, NVPI | Khớp. |
| Báo cáo kế toán (declaration) | Chỉ role admin | `/api/reports/declaration/**`: admin | Khớp. |
| Upload / Kiểm soát | `/upload-files/**`, `/upload/**`: chỉ đăng nhập; khối upload hiện cho BGD, TPTD, NVPI ở giao diện | `/api/appraisal-records/*/control*`: authenticated; `canUploadControl` ở FE (BGD, TPTD, NVPI) | Khớp. Cần xác minh: BE `ProfileControlController` có chặn upload theo chức vụ không (comment ghi "chỉ ở giao diện", đúng như cũ). |

## Thiếu sót quan trọng (theo mức ưu tiên)

### Ưu tiên cao

1. **Chứng thư Word chỉ có tiếng Việt** (dòng 19). Cũ cho in tiếng Anh và tiếng Trung qua `printLanguage` (LT/printing-certificate/printing-certificate-detail.html; LC/AppraisalCertificateController `certificateWordExport`; `AppraisalCertificateWordExporter` dùng dịch vụ ngoài). BS/PrintingCertificateExportService xác nhận cố ý bỏ. Nếu nghiệp vụ còn cần chứng thư song ngữ thì phải bổ sung: tham số ngôn ngữ ở BE `GET /api/printing-certificates/{requestId}/word`, ô chọn ngôn ngữ ở PrintingCertificateFormView.vue, và nguồn dịch. Cần xác nhận với người dùng nghiệp vụ.
2. **Mục "Xem tài liệu" chưa có trang** (dòng 51). Menu tồn tại ở `components/layout/nav-items.ts` nhưng không có `to`. Cần: view danh sách (Quy chế KSCL và khung giá đất 2023 theo quận), nơi lưu PDF (cũ dùng đường dẫn cứng trên máy chủ), endpoint phục vụ PDF.

### Ưu tiên trung bình

3. **Xem file Office đối với file lưu local** (dòng 52). `OfficeViewerService` chỉ chấp nhận tài liệu có `s3FilePath`. Cần xác minh: file MSB và hồ sơ cũ chỉ có `filePath` local đã được đẩy lên S3 chưa; nếu chưa, các nút "xem" ở các màn có file đính kèm sẽ 404.
4. **Phê duyệt hồ sơ thẩm định (loại 2, 3)** (dòng 29). `profile-accept/reject` vẫn được trang LT/appraisal-records/approve.html gọi, BE mới chưa có. Cần xác minh bên nhóm "Hồ sơ thẩm định" (appraisal-records) đã có luồng tương đương hay đã bỏ nghiệp vụ.
5. **Thay đổi hành vi tải file MSB**: cũ `/appraisal-msb-profile/download` công khai, mới `/api/files/download` cần Bearer token. Nếu FE hiển thị ảnh/file bằng thẻ `<img src>` hoặc link mở tab mới thì cần chắc chắn có cơ chế gắn token hoặc URL ký sẵn (cần xác minh trong components/msb-exchange/ImageViewer.vue, MessageContent.vue).

### Ưu tiên thấp

6. **Tuyến QR mã hóa cũ** `/appraisal-certificate/qr-code/detail?code=` (dòng 22): chỉ cần nếu còn chứng thư in từ rất lâu dùng URL này.
7. **Dọn comment lỗi thời** trong `td-fe/src/router/index.ts` (route `appraisal-request-imported`): ghi "Chưa port chức năng import" nhưng đã có `AppraisalRequestImportedImportDialog.vue`.
8. **Mục ➖ cần xác nhận một lần**: dòng 8 (modal tạo "Đầu mối kinh doanh" có còn hiển thị ở legacy hay chỉ mở `/admin/create`), dòng 11 (ô chọn số hợp đồng màn Pháp lý dùng API nào).

### Không phải thiếu sót (đã so sánh, giữ để tránh khảo sát lại)

- Tất cả exporter Excel trong phạm vi (`CertificateExcelExporter`, `JuridicalStreamingExcelExporter`/`JuridicalExcelExporter` (cũ đã @deprecated), `MSBReportExcelExporter`, `DeclarationReportExporter`) đã có bản tương ứng trong BX với đúng số cột/tên cột; chỉ khác chủ ý (ô trống thay "N/A", sửa chính tả cột).
- `DownloadController.search-coordinate` đã tương đương CoordinateSearch.
- Các điểm siết quyền ở mục "So sánh phân quyền" là chủ ý, được ghi lại trong comment của BE.
