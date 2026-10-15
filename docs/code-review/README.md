# Tổng hợp code review — backend & frontend

Ngày thực hiện: 2026-10-15
Phạm vi: so sánh nhánh `develop` với `main` của 2 submodule, dùng skill `/code-review` (effort `high`), chạy song song nhiều agent độc lập theo từng mảng nghiệp vụ/góc nhìn.

- Backend (`backend/`): 19 commit chưa merge vào `main`, 5 agent review → chi tiết tại [backend.md](backend.md)
- Frontend (`frontend/`): 22 commit chưa merge vào `main`, 7 agent review (1 agent có build/test thực tế để xác nhận) → chi tiết tại [frontend.md](frontend.md)

Lưu ý: đây là review tĩnh (đọc code, không chạy ứng dụng/test tích hợp, trừ phần frontend có build/lint/vitest xác nhận một số lỗi), nên các mục đánh dấu "chưa chắc chắn" cần người có domain knowledge (schema DB, dữ liệu legacy thật) xác nhận lại trước khi fix.

## Số liệu tổng quan

| | Số agent | Số phát hiện chính |
|---|---|---|
| Backend | 5 | ~40 (8 mảng × 5 agent, đã gom theo nghiệp vụ) |
| Frontend | 7 | ~23 lỗi/nghiệp vụ + ~10 cơ hội đơn giản hóa code |

## Các vấn đề nghiêm trọng nhất cần xử lý trước (toàn hệ thống)

1. **Backend — Đổi mật khẩu không flush DB** (`AuthService.changePassword`): user đổi mật khẩu xong không đăng nhập lại được bằng mật khẩu mới. Xem [backend.md §3.1](backend.md#31-nghiêm-trọng-đổi-mật-khẩu-có-thể-mất-dữ-liệu-mật-khẩu-mới-âm-thầm).
2. **Backend — Truy cập/sửa chéo chi nhánh** ở nhiều API (in chứng thư, phân công yêu cầu, viewer file Office): thiếu kiểm tra phạm vi (`MsbRequestScope`/`RequestScopeSql`) ở các API theo id, chỉ có ở API list. Xem [backend.md §1.1](backend.md#1-in-chứng-thư--qr), [§3.2](backend.md#32-cao-link-xem-file-office-viewer-không-kiểm-tra-quyền), [§4.2](backend.md#42-cao-màn-phân-công-assign-bỏ-qua-phạm-vi-chi-nhánh-mà-listdetail-đang-áp-dụng).
3. **Backend — Thiếu role check ở API tạo yêu cầu thẩm định** (`AppraisalRequestFormService.create`): có thể bị lợi dụng để tự cộng tiền vào tài khoản thu hộ. Xem [backend.md §4.1](backend.md#41-nghiêm-trọng-api-tạo-yêu-cầu-create-thiếu-role-check-và-thiếu-gating-field-collection).
4. **Backend — Race condition số dư tài khoản thu hộ** lặp lại ở 2-3 nơi (`CashCollectorAccountingService.addBalance` không có lock/version) — nên sửa tập trung một lần. Xem [backend.md §2.7 / §4.3](backend.md#27-trung-bình-race-condition-số-dư-tài-khoản-thu-hộ--dùng-tài-khoản-đã-soft-delete).
5. **Frontend — Class `border-strong` không tồn tại**, làm mất màu viền cho mọi nút outline (đã xác nhận bằng build CSS thật, 3 agent độc lập cùng phát hiện). Sửa 2 dòng là xong. Xem [frontend.md §1.1](frontend.md#11-cao-class-border-strong-không-tồn-tại-trong-tailwind--mất-màu-border-cho-nút-outline).
6. **Frontend — Zod schema chặn `uuid: null`** khiến không sửa được một số request cũ (dữ liệu legacy). Xem [frontend.md §2.1](frontend.md#21-cao-validate-zod-không-cho-phép-uuid-null-từ-dữ-liệu-cũ).

## Gợi ý thứ tự xử lý

1. Vá ngay các lỗi bảo mật/phân quyền chéo chi nhánh ở backend (mục 2-3 trên) vì ảnh hưởng dữ liệu khách hàng và tiền.
2. Fix lỗi đổi mật khẩu (mục 1) — ảnh hưởng trực tiếp trải nghiệm đăng nhập.
3. Fix `border-strong` ở frontend — 1-dòng, rủi ro thấp, ảnh hưởng UI toàn app.
4. Fix zod `uuid` — chặn luồng sửa request cũ.
5. Dọn race condition số dư tài khoản thu hộ — cần thêm `@Version`/lock, nên làm 1 lần cho mọi chỗ gọi `CashCollectorAccountingService`.
6. Các mục còn lại (trung bình/thấp) xử lý theo sprint, ưu tiên theo mức độ đã gắn trong từng file chi tiết.
