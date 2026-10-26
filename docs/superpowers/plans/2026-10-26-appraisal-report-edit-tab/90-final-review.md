# 90 — Kiểm tra cuối và review toàn nhánh (T35)

> Subagent reviewer: `model: "sonnet"`.

- [ ] **Step 1:** `cd backend && ./mvnw test` → toàn bộ PASS (cần MySQL). Ghi số test.
- [ ] **Step 2:** `cd frontend && npm run lint && npm run type-check && npm run test:unit && npm run build` → PASS.
- [ ] **Step 3:** `git -C backend diff main --stat`, `git -C frontend diff main --stat`: không có file ngoài các task; không có Flyway mới; `TabPlaceholder.vue` đã không còn được dùng (xóa nếu thừa).
- [ ] **Step 4: Kiểm tra tay trên app** (skill `run` hoặc dev server + backend nối DB thử nghiệm) với một báo cáo có đủ 3 loại nhóm: mỗi tab mở được, lưu được, sau khi lưu mô tả đất tab so sánh cột 0 có chi tiết mới; lưu so sánh → Bảng điều chỉnh có giá trị đồng bộ; lưu bảng điều chỉnh → Kết quả đổi theo `avgPriceGuide`. So JSON `GET evaluation-result` với `GET /api/evaluation-results/detail/{r}/{g}` của hệ thống cũ trên **cùng DB** cho ít nhất 1 nhóm mỗi loại/phương pháp có dữ liệu — phải giống hệt (trừ thứ tự khóa).
- [ ] **Step 5:** Gọi **một** subagent reviewer `model: "sonnet"` review cả hai nhánh so với `main` theo: Global Constraints, 9 điểm "Khác hệ thống cũ" có trong Javadoc, Review Focus (5 điểm, test tương ứng tồn tại và PASS), checklist cuối `.claude/rules/jpa-spring.md`, quy tắc UI `frontend/CLAUDE.md`. Sửa lỗi được xác nhận (mỗi lần sửa lại qua spec + code review sonnet), chạy lại Step 1–2.
- [ ] **Step 6:** Báo người dùng: số test, danh sách điểm khác hệ thống cũ, các phần ngoài phạm vi (xem trước, docx, phiếu TSSS, sửa thông tin cơ bản). Không push, không cập nhật con trỏ submodule ở repo gốc khi chưa được đồng ý.
