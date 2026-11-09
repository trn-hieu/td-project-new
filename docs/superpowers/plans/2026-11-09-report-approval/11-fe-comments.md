# F02: Ô nhắc người dùng chung và khung trao đổi trên báo cáo

> Đọc `00-index.md` trước (mục 10 trong "Quyết định đã chốt"), `frontend/CLAUDE.md` và `frontend/DESIGN_SYSTEM.md` (§10.7 IconButton, §14 toast). Chỉ làm Task F02. Cần F01 đã merge.

### Task F02: `MentionTextarea` + `ReportCommentPanel`

**Files:**
- Create:
  - `F components/chat/MentionTextarea.vue`
  - `F components/report-comment/ReportCommentPanel.vue`
- Modify: `F components/msb-exchange/ChatComposer.vue`. Thay phần gợi ý @nhắc (khoảng L59–120 và template tương ứng) bằng `MentionTextarea`. Giữ nguyên toàn bộ hành vi MSB: file đính kèm, tọa độ, Enter gửi, giới hạn ký tự.
- Test:
  - `F components/chat/__tests__/MentionTextarea.spec.ts`
  - `F components/report-comment/__tests__/ReportCommentPanel.spec.ts`
  - Chạy lại `F views/__tests__/MsbExchangeView.spec.ts`

**Interfaces:**
- Consumes:
  - F01: `listReportComments`, `postReportComment`, `searchReportMentions`, `ReportComment`.
  - Có sẵn: `findMentionQuery`, `insertMention`, `serializeMentions` (`@/lib/mention`), `AutoTextarea`, `MessageContent.vue`, `IconButton`, `toast`, `errorMessage`, `components/ui/collapsible`.
- Produces:
  - `MentionTextarea.vue`
    - Props: `{ search: (query: string, signal: AbortSignal) => Promise<MentionEmployee[]>; maxlength?: number; placeholder?: string; disabled?: boolean; enterSubmits?: boolean }`. `enterSubmits` mặc định `true`.
    - `v-model` là văn bản người dùng thấy (`@Họ Tên`).
    - Expose `serialize(): string`, gọi `serializeMentions` để trả văn bản có token `@[id]`. Expose thêm `reset()` và `focus()`.
    - Emits `submit` khi bấm Enter trên máy có chuột (`(pointer: fine)`) và không giữ Shift. Trên máy cảm ứng, Enter luôn xuống dòng, như `ChatComposer` hiện tại.
    - Gõ `@xx` thì sau 250 ms (`useDebounceFn`, giống code hiện có) gọi `search`, hủy request cũ bằng `AbortController`. Gợi ý hiện trong popover/listbox có điều hướng bàn phím ↑ ↓ Enter Esc, đúng như `ChatComposer` đang làm. Chọn một người thì `insertMention`.
  - `ReportCommentPanel.vue`
    - Props: `{ reportId: number }`.
    - Thu gọn/mở rộng bằng `Collapsible`. Trạng thái mở lưu trong `localStorage` khóa `report-comments-open`, mọi lần đọc/ghi bọc `try/catch`, mặc định mở.
    - Header gồm tiêu đề "Trao đổi", `IconButton label="Làm mới"` (icon `RefreshCw`) và `IconButton label="Thu gọn"`/`"Mở rộng"`.
    - Danh sách tin cũ ở trên, mới ở dưới. API trả `id DESC` nên đảo lại khi hiển thị. Mỗi tin gồm tên người viết, giờ (định dạng ngày giờ có sẵn trong `@/lib/date-format`) và `<MessageContent :html>`.
    - Còn trang thì có nút `outline` "Xem tin cũ hơn" ở đầu danh sách, tải `page + 1` rồi nối lên trên.
    - Ô soạn `MentionTextarea` (`search = searchReportMentions`, `maxlength = 10000`) và `Button` "Gửi" (icon `Send`). Đang gửi hoặc nội dung trống thì disabled. Gửi thành công thì nối tin vào cuối, `reset()` ô soạn và cuộn xuống cuối. Gửi lỗi thì `toast.error('Không gửi được tin nhắn', { description: errorMessage(e) })` và giữ nguyên nội dung.
    - Có trạng thái loading (skeleton), empty ("Chưa có trao đổi nào") và error (`Alert variant="destructive"` kèm nút "Thử lại").
    - Không poll. Tải lại khi mount, khi `reportId` đổi và khi bấm "Làm mới".

- [ ] **Step 1: Viết `MentionTextarea.spec.ts` (fail)**

  - `go_at_goi_search_sau_debounce`: dùng `vi.useFakeTimers()`. Gõ `"chào @ng"`, chạy timer 250 ms, kiểm `search` được gọi với `'ng'`.
  - `chon_goi_y_chen_ten`: chọn `{ employeeId: 10, fullName: 'Nguyễn A' }` thì v-model chứa `'@Nguyễn A '`, `serialize()` chứa `'@[10]'`.
  - `enter_emit_submit_shift_enter_khong`: mock `matchMedia` cho `(pointer: fine)` là true.
  - `go_tiep_huy_request_cu`: signal của lần gọi đầu có `aborted === true`.
  - `esc_dong_goi_y`

  Run: `cd frontend && npx vitest run src/components/chat/__tests__/MentionTextarea.spec.ts`. Expected: FAIL.

- [ ] **Step 2: Viết `MentionTextarea.vue`, tách logic từ `ChatComposer.vue`, chạy test**

  Expected: PASS.

- [ ] **Step 3: Sửa `ChatComposer.vue` dùng `MentionTextarea`, chạy test MSB**

  Run: `npx vitest run src/views/__tests__/MsbExchangeView.spec.ts src/lib/__tests__/mention.spec.ts`. Expected: PASS.

  Mở `/msb-profiles/{id}/exchange` bằng `npm run dev` và kiểm tay: gõ @ có gợi ý, gửi tin kèm file vẫn được. Không có dữ liệu dev thì ghi rõ trong báo cáo.

- [ ] **Step 4: Viết `ReportCommentPanel.spec.ts` (fail)**

  Mock `@/api/report-comments`.
  - `tai_trang_dau_hien_cu_tren_moi_duoi`: API trả `[3, 2, 1]`, DOM hiển thị theo thứ tự 1, 2, 3.
  - `gui_noi_tin_va_xoa_o`: `postReportComment(5, 'xin chào @[10]')` được gọi với nội dung đã serialize.
  - `gui_loi_toast_va_giu_noi_dung`
  - `xem_tin_cu_goi_page_1`: chỉ hiện nút khi `totalPages > 1`.
  - `rong_hien_chua_co_trao_doi`
  - `loi_tai_hien_alert_thu_lai`
  - `thu_gon_luu_local_storage`: `localStorage.setItem` ném lỗi thì component vẫn chạy.
  - `lam_moi_tai_lai_trang_0`

  Run: `npx vitest run src/components/report-comment/__tests__/ReportCommentPanel.spec.ts`. Expected: FAIL.

- [ ] **Step 5: Viết `ReportCommentPanel.vue`, chạy test**

  Expected: PASS.

- [ ] **Step 6: Lint, type-check, toàn bộ test**

  Run: `npm run lint && npm run type-check && npm run test:unit`. Expected: PASS.

- [ ] **Step 7: Commit**

  ```bash
  git -C frontend add -A
  git -C frontend commit -m "feat(report-approval): ô nhắc dùng chung và khung trao đổi trên báo cáo"
  ```
