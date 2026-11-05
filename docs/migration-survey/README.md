# Khảo sát mức độ migrate: legacy → backend/frontend mới

Ngày khảo sát: 2026-11-04

| Hệ thống | Repo | Nhánh | Commit |
|---|---|---|---|
| Legacy (Spring MVC + Thymeleaf) | `trn-hieu/td-project` | `main` | `cbcf81e` |
| Backend mới (Spring REST) | `trn-hieu/td-project-be` | `develop` | `1418745` |
| Frontend mới (Vue 3) | `trn-hieu/td-fe` | `develop` | `f2c6a09` |

Phương pháp: chỉ đọc code. Liệt kê 47 controller của legacy (29 MVC + 18 API, khoảng 277 endpoint), cùng exporter, schedule, runner, template và menu. Sau đó tìm phần tương đương ở BE mới và FE mới. Chưa chạy hai hệ thống để đối chiếu dữ liệu thực tế.

Ký hiệu: ✅ đã migrate đủ · ⚠️ migrate một phần · ❌ chưa migrate · ➖ không cần (code chết, demo, runner chạy một lần).

## Tổng quan

| Nhóm | Chi tiết | ✅ | ⚠️ | ❌ | ➖ |
|---|---|---|---|---|---|
| 1. Hồ sơ / báo cáo thẩm định | [01-appraisal-records.md](01-appraisal-records.md) | ~36 | 0 | 2 + luồng duyệt hồ sơ + 3 job ảnh | ~7 |
| 2. Yêu cầu, chứng thư, in ấn, pháp lý, file | [02-request-certificate-juridical.md](02-request-certificate-juridical.md) | 41 | 3 | 2 | 6 |
| 3. Báo cáo & xuất Excel | [03-reports.md](03-reports.md) | 19 | 0 | 1 | 3 |
| 4. Kế toán, nhân sự, xác thực, hệ thống | [04-accounting-hr-system.md](04-accounting-hr-system.md) | 36 | 7 | 5 | 7 |

**Kết luận:** phần lớn chức năng đang dùng đã migrate đủ cả BE lẫn FE, chiếm khoảng 85% số dòng khảo sát. Trong đó có toàn bộ 18 báo cáo đang dùng, màn 6 tab hồ sơ thẩm định, xuất Word (đã có test đối chiếu), phân quyền role và menu. Tuy vậy, **chưa thể tắt legacy** vì còn các thiếu sót ở mục dưới, đặc biệt là các job xử lý ảnh.

## Thiếu sót cần xử lý trước khi tắt legacy

### Ưu tiên cao

1. **Thiếu job định kỳ xử lý ảnh** (nhóm 1, 4). BE mới không có `@Scheduled` hay `@EnableScheduling` nào; điều này đã được kiểm lại. Ba job của legacy chưa được port:
   - `AppraisalImageResizeSchedule`
   - `AppraisalConstructionImageWordExportSchedule`
   - `ComparisonAssetImageWordExportSchedule`

   BE mới vẫn đọc hai cột `backup_resize_path` và `word_export_image_path` do các job này ghi. Khi tắt legacy, ảnh mới upload sẽ không có bản đã resize hay tối ưu cho Word. Word vẫn xuất được nhờ fallback về ảnh gốc, nhưng file sẽ rất nặng.
2. **Không tạo được Tiểu mục và Tài khoản ghi nợ/có mới** (nhóm 4). Legacy có `POST /accounting/subsection/create`, `/accounting/creditAccount/create` và `/accounting/creditAccount/detail`. BE mới và FE mới đều không có.
3. **Thiếu xuất Excel danh sách hồ sơ thẩm định** (nhóm 1). `AppraisalProfileExcelExporter` của legacy chưa có ở BE mới, FE mới cũng không có nút xuất.
4. **Thiếu chức năng "Trích xuất"** (nhóm 1). Đây là biểu đồ hồ sơ theo chi nhánh và tháng, chỉ chi nhánh HN dùng. Trong `AppraisalRecordListView.vue`, FE mới mới có nút ở trạng thái `disabled`.
5. **Runner migrate dữ liệu** (nhóm 1, 4). `LandDetailLandUseRowsMigrationRunner` và `LandDetailPlotsStructureMigrationRunner` mặc định bật trên legacy. Cần xác nhận hai runner đã chạy xong trên DB production trước khi tắt legacy, vì BE mới đọc cấu trúc `plots → mapSheets`.

### Ưu tiên trung bình

6. **Mục menu "Xem tài liệu"** (`/download/list`, gồm Quy chế KSCL và khung giá đất) đã có trong `nav-items.ts`, nhưng không có route hay view nên bấm vào không có tác dụng.
7. **Xem file Office** (`/view-file`) chỉ hỗ trợ tài liệu có `s3FilePath`. File cũ chỉ có `filePath` lưu ở local sẽ trả 404. Cần xác minh dữ liệu cũ đã được chuyển lên S3.
8. **Xóa bút toán phí khai thác:** nút xóa đang `disabled` ở `ExploitationFeeListView.vue`.
9. **Bảo mật đăng xuất:** đăng xuất chỉ xóa token phía client, refresh token vẫn còn hiệu lực 14 ngày. Cần xác minh có gọi `RefreshTokenService.revokeAll` hay không.
10. **Không có trang 403:** khi thiếu quyền, người dùng bị đưa về `/` mà không có thông báo.
11. **Thông báo `ACCOUNTING_IMPORT`** không còn được tạo, vì import ở hệ thống mới chạy đồng bộ.
12. **Nhập Excel bảng so sánh** (`/apache-poi/excel`) chưa có ở hệ thống mới. Cần nghiệp vụ xác nhận chức năng này còn được dùng hay không.

### Cần nghiệp vụ xác nhận (khác biệt có chủ ý hoặc có thể bỏ)

- **Chứng thư Word** chỉ còn tiếng Việt. Legacy có thêm tiếng Anh và tiếng Trung qua tham số `printLanguage`.
- **Duyệt/từ chối hồ sơ thẩm định loại 2/3** (`/update-request/profile-accept|profile-reject`) vẫn chỉ có ở legacy. Hiện không còn UI nào tạo loại yêu cầu này, nên cần kiểm tra DB còn bản ghi `update_request` loại 2/3 tồn đọng hay không.
- **Báo cáo ngày:** menu ở legacy đã bị comment. QR dạng cũ `/appraisal-certificate/qr-code/detail` ít còn được dùng.
- **Sau khi đăng nhập**, hệ thống mới luôn về trang chủ, không điều hướng theo chức vụ như legacy.
- **Phân quyền ở bản mới chặt hơn**, nên ghi vào release note:
  - In ấn chỉ cho BGD và NVPI.
  - API phê duyệt và API báo cáo bị chặn ở server.
  - VCB/VCBNEO chặn BGD_HN.
  - Tải Word, xem trước và tải file MSB không còn `permitAll`.
  - Quyền tick "Đồng bộ lên CSDL" có thêm BGD.
- **Role `REPORT_BOARD_OF_DIRECTOR` và `ADMIN_REPORT`** không có giao diện để gán ở cả hai hệ thống.

### Việc dọn dẹp nhỏ

- Comment đã lỗi thời trong `td-fe/src/router/index.ts`: chức năng import imported đã được port.
- Javadoc `AccountingController` ghi "chưa port: duyệt yêu cầu sửa", nhưng chức năng này đã được port.

## Giới hạn của khảo sát

- Mới đối chiếu endpoint, bộ lọc, cột Excel và phân quyền. Công thức SQL, tổng và nhóm của 17/18 báo cáo chưa so từng dòng; riêng báo cáo Ban lãnh đạo đã so logic. Cần chạy đối chiếu dữ liệu thật giữa hai hệ thống.
- Chưa kiểm cấu hình `application*.properties` trên production của legacy, ví dụ trạng thái bật/tắt của các runner.
