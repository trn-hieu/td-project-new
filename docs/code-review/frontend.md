# Code review — frontend (nhánh `develop` so với `main`)

Phạm vi: `frontend/` (Vue 3 + TS), so sánh 22 commit trên `develop` chưa có trên `main`.
Phương pháp: 7 agent review độc lập (quy ước/altitude, hạ tầng/shell, nghiệp vụ appraisal, nghiệp vụ printing/msb, audit regression so với hành vi cũ, dọn dẹp simplification/efficiency, dò sự nhất quán cross-file). Có agent đã build thực tế (`vite build`, `vue-tsc`, `eslint`/`oxlint`, 928 vitest) để xác nhận một số lỗi.

Đường dẫn tương đối so với `frontend/`.

---

## 1. Lỗi UI xác nhận bằng build thực tế (độ tin cậy cao)

### 1.1 [Cao] Class `border-strong` không tồn tại trong Tailwind → mất màu border cho nút outline
`src/components/ui/button/index.ts:16`, `src/components/actions/icon-button.ts:46`.

Token đúng phải là `border-border-strong` (từ `--color-border-strong` khai báo trong `main.css`); `border-strong` không sinh ra CSS nào. Đã được **xác nhận bằng cách build CSS thực tế** (`@tailwindcss/node`): `.border-strong` không có trong output, `.border-border-strong` và `.border-control` thì có.

**Hậu quả:** mọi nút `variant="outline"` (Hủy, Quay lại, Thử lại, view/edit/upload...) và `IconButton tone="neutral" appearance="outline"` hiển thị với màu `--border` nhạt mặc định thay vì màu border đậm hơn như mục tiêu của commit (WCAG 1.4.11). Diff này cũng xóa luôn `dark:border-input` cũ nên dark mode mất border. `lint:tokens` không bắt được lỗi vì class không map tới token nào để check.

*(Phát hiện này được 3 agent độc lập xác nhận trùng nhau — độ tin cậy rất cao, nên sửa sớm.)*

### 1.2 [Trung bình] Sidebar hiển thị menu item mà user không có quyền mở
`src/components/layout/AppSidebar.vue:35` — chỉ `accountRestricted` items được filter theo quyền, còn "Phê duyệt yêu cầu" (BGD/TPTD/KSV), "In ấn" (BGD/NVPI), "Hồ sơ thẩm định" (staff only) hiện cho **mọi** user đã đăng nhập. Trước đây các mục này "trỏ đi đâu" (chưa gắn link) nên vô hại; nay đã gắn route nên bấm vào sẽ bị router guard redirect về `/` không có thông báo gì.

### 1.3 [Thấp-trung bình] Sidebar highlight sai mục khi ở màn exchange MSB
`src/components/layout/AppSidebar.vue:69` — dùng path-prefix match nên luôn highlight "Hồ sơ MSB" ở route `/msb-profiles/:id/exchange`, bất kể hồ sơ đang ở trạng thái nào, trong khi breadcrumb/nút "Quay lại" lại dẫn đúng theo trạng thái (đang xử lý/hoàn thành/đã hủy) → sidebar và breadcrumb không khớp nhau.

---

## 2. Lỗi nghiệp vụ / luồng xử lý

### 2.1 [Cao] Validate zod không cho phép `uuid: null` từ dữ liệu cũ
`src/components/appraisal-request/AppraisalRequestForm.vue:334` (copy) + schema dòng 173 (`uuid: z.string().optional()`). Backend có thể trả `uuid: null` cho asset cũ (`AppraisalRequestFormService.toAssets` lọc `Objects::nonNull` ngụ ý tồn tại uuid null). `optional()` chỉ cho phép `undefined`, không cho `null` → submit bị chặn vĩnh viễn, lỗi hiện ở error summary không rõ label field nào. **Sửa:** `uuid: row.uuid ?? undefined` hoặc `z.string().nullish()`.

### 2.2 [Trung bình] Reset form "mất trắng" dữ liệu khi đổi loại hồ sơ, không hỏi xác nhận
`src/components/appraisal-request/AppraisalRequestForm.vue:400` — watcher đổi `requestType` gọi `resetForm({values: emptyValues()})` không check `meta.dirty`/`confirmDiscard`. Đổi qua query param trên cùng route không trigger `onBeforeRouteLeave` nên guard "mất dữ liệu chưa lưu" bị bỏ qua. User điền nửa form rồi bấm sang "Hồ sơ doanh nghiệp" sẽ mất hết, không cảnh báo.

### 2.3 [Thấp-trung bình] Danh sách "Thu phí tiền mặt" không reset khi đổi sang tài khoản không có chi nhánh
`AppraisalRequestForm.vue:426` — thứ tự check `accountId === loadedAccountId` trước `if (!branch) return` khiến đổi từ tài khoản chi nhánh A sang tài khoản công ty (không có branch) để lại danh sách nhân viên cũ của chi nhánh A.

### 2.4 [Thấp] `DateTimePicker` ra `NaN` nếu model không đúng format `YYYY-MM-DDTHH:mm`
`src/components/form/DateTimePicker.vue:45` — `Number(model.value.slice(11,13))` không validate, string sai format cho giờ/phút NaN.

### 2.5 [Thấp] Duplicate key trong `v-for` của timeline hồ sơ
`src/components/appraisal-record/AppraisalTimeline.vue:58` — `:key="time"` trùng khi 1 bước có 2 timestamp giống nhau (ví dụ hồ sơ bị trả lại rồi làm lại).

### 2.6 [Trung bình] Nút "Xác nhận import" biến mất ngay khi đang xử lý
`src/components/accounting-imported/AccountingImportedImportDialog.vue:74` — `clearResult()` chạy trước khi gọi API confirm → `canConfirm` false ngay lập tức → nút + bảng preview biến mất khỏi DOM trong lúc đang xử lý (không thấy spinner), nếu lỗi thì phải "Kiểm tra file" lại từ đầu.

### 2.7 [Trung bình] Mở file đính kèm local có thể bị popup blocker chặn âm thầm
`src/composables/useAttachmentOpener.ts:168` — `window.open` gọi sau `await downloadFile(...)`, ra ngoài user-activation window (đặc biệt Safari/iOS hoặc mạng chậm >5s) → popup bị block, không toast, không fallback.

### 2.8 [Trung bình] Loading "kẹt" mãi ở màn chat trao đổi MSB khi load im lặng (silent) bị abort bởi load khác
`src/components/msb-exchange/ExchangeChat.vue:62` — mọi `load()` abort request trước, kể cả silent poll abort load đầu tiên; `finally` chỉ clear `loading` cho load non-silent và còn "current" → trường hợp silent load thắng thì `loading` không bao giờ về false, skeleton hiện vĩnh viễn tới khi đổi `requestId`.

### 2.9 [Trung bình] Cảnh báo trùng serial chứng thư: so sánh chưa trim, không có abort, dễ bị stale
`src/views/PrintingCertificateFormView.vue:187` — so sánh `contractId` chưa trim khoảng trắng gây false positive; không guard request cũ trả về sau khi `load(true)` đã clear cảnh báo → ghi lại cảnh báo cũ không còn đúng.

### 2.10 [Thấp-trung bình] Save chứng thư thành công nhưng reload lỗi làm mất cả form hiển thị
`src/views/PrintingCertificateFormView.vue:273` — `await load(true)` lỗi (silent reload) thì vẫn hiện toàn màn hình lỗi "Không tải được dữ liệu chứng thư" che mất form, dù dữ liệu đã lưu thành công → user tưởng lưu thất bại và có thể bấm lưu lại.

### 2.11 [Thấp] Số tiền bằng chữ ra rỗng/gây hiểu nhầm với giá trị thập phân <1
`src/lib/read-money.ts:72` — giá trị như 0.5 hoặc -0.5 cho `text=''`, kết quả "` đồng`" hoặc "`Âm  đồng`" thay vì "Không đồng", vẫn pass validate vì chuỗi không rỗng hoàn toàn.

### 2.12 [Trung bình] Notification đọc trên trang chat không đồng bộ lại badge chưa đọc
`src/components/layout/NotificationPopover.vue:99` + `src/views/MsbExchangeView.vue:71` — backend chỉ mark-read khi gọi `GET .../exchange?notificationId=`, nhưng nếu user đang mở sẵn trang exchange và nhận link mới chỉ đổi `notificationId` (query) thì `watch(requestId)` không trigger lại `load()` → noti không được mark read. Ngoài ra sau khi đọc, frontend không tự `refreshCount()` nên badge số chưa đọc trên chuông vẫn sai cho tới lần socket push/reload kế tiếp.

### 2.13 [Thấp] Route `/appraisal-request/:id/edit` cho phép `admin` nhưng backend từ chối
`src/router/index.ts:114` — guard FE cho `admin` qua được route, nhưng `SecurityConfig` BE không cho `admin` gọi các API liên quan → tài khoản admin-only vào thẳng trang lỗi tải dữ liệu (403) thay vì bị redirect như các route khác.

### 2.14 [Thấp] `tatStartAt` bị cắt mất giây khi lưu MSB request
`src/views/MsbRequestFormView.vue:142` — slice 16 ký tự làm giây luôn bị ghi `:00` dù user không sửa trường này.

### 2.15 [Thấp] Form MSB không reload khi đổi `requestId` qua router-link (chỉ cùng component, khác param)
`src/views/MsbRequestFormView.vue:177` — `load()` gọi 1 lần, không `watch(requestId)`; chuyển giữa 2 id dùng lại component cũ, submit có thể PUT dữ liệu cũ vào id mới.

### 2.16 [Thấp] Field ẩn thiếu giá trị mặc định khi dữ liệu legacy thiếu key
`src/lib/printing-certificate.ts:130-135` — `is_print`, `appraisal_request_id`, `appraisal_profile_id` pass raw vào schema strict `z.boolean()`/`z.number().nullable()`; JSON cũ thiếu key sẽ gây lỗi khó hiểu khi submit.

### 2.17 [Thấp] Map khảo sát không redraw sau khi "Thử lại" load Google Maps thành công
`src/components/survey/SurveyForm.vue:378` — khác `ControlMap` (có `.then(render)`), retry ở đây không vẽ lại route đã lưu, còn gắn thêm Autocomplete mới mỗi lần retry mà không dọn cái cũ.

### 2.18 [Thấp] `picking` mode bị kẹt `true` khi đổi hồ sơ control
`src/views/ControlView.vue:32` — load không-quiet set `control=null` unmount panel mà không emit `pickingChange(false)`, map mới mount vẫn ở pick-mode nhưng panel mới không khớp key, click bị bỏ qua.

### 2.19 [Thấp] Route quãng đường ưu tiên tọa độ cũ, bỏ qua địa chỉ mới chọn từ Autocomplete
`src/components/survey/SurveyForm.vue:162` — chọn địa chỉ mới từ gợi ý Autocomplete không cập nhật lat/lng, "Tính quãng đường" vẫn tính theo tọa độ cũ dù hiển thị địa chỉ mới.

### 2.20 [Thấp] Geocode địa chỉ đích theo tọa độ không chống stale response
`src/components/survey/SurveyForm.vue:125` — không có guard request-id, nếu người dùng sửa lat rồi lẹ sang lng, response geocode cũ về sau có thể ghi đè kết quả mới.

### 2.21 [Thấp] Deploy thiếu proxy WebSocket cho `/ws`
`frontend/deploy/README.md:79` — vhost Apache mẫu chỉ proxy `/api`, không proxy `/ws` và không bật `proxy_wstunnel_module` → lên môi trường theo đúng hướng dẫn deploy thì realtime notification (toast, bell, badge) sẽ không hoạt động (chỉ chạy được ở dev nhờ vite proxy).

### 2.22 [Trung bình] Reconnect STOMP có thể gây logout domino hoặc tràn request refresh token
`src/composables/useNotificationSocket.ts:48` + `src/stores/auth.ts:122-125` — `onStompError` gọi `auth.refresh()` mỗi lần có lỗi STOMP; `doRefresh` xóa session khi refresh fail bất kỳ lý do gì (kể cả lỗi mạng thoáng qua hoặc do nhiều tab share refresh token 1-lần-dùng) → có thể logout domino nhiều tab. Ngoài ra không có backoff (`reconnectDelay` cố định 5s), backend down sẽ bị retry + gọi refresh liên tục vô hạn.

### 2.23 [Thấp] Unread count có thể bị ghi đè bởi response trả chậm của user cũ sau khi logout
`src/stores/notifications.ts:69` — `refreshCount` không có generation guard như `reload`/`loadMore`, request của user A trả về sau khi đã `reset()`/login user B có thể ghi nhầm badge.

---

## 3. Trùng lặp code / cơ hội đơn giản hóa (không phải bug nhưng ảnh hưởng chi phí bảo trì)

- **Khung danh sách có phân trang + sync URL** lặp lại gần như nguyên vẹn (~100 dòng/màn) ở 8+ view: `AppraisalRequestListView`, `MsbProfileList`, `AppraisalRecordListView`, `MsbRequestListView`, `PrintingCertificateListView`, `AccountingImportedListView`, `UpdateRequestListView`, `EmployeeListView`. Đề xuất: composable `useUrlSyncedList(...)`.
- **Pattern load có abort controller** (`AbortController`, loading/error ref, check `signal.aborted`, cleanup unmount) lặp ở ~44 vị trí. Đề xuất: composable `useAbortableLoader`/`useAsyncData`.
- **3 composable bản đồ Google Maps** (`useControlMap`, `useSurveyMap`, `useCoordinateSearchMap`) lặp `textContent()`, icon marker, `init()` gần như y hệt.
- **`tokenColor()`** (`src/lib/map-theme.ts:19`) tái triển khai lại đúng logic resolve-CSS-color-to-hex đã có ở `src/lib/color.ts`, nhưng không cache canvas context → tốn hiệu năng mỗi lần gọi (khi recenter map/vẽ route).
- **`showServerError`/`onInvalid`/error-summary wiring** lặp ở 6 file form, mỗi nơi lệch nhau nhỏ (dễ drift khi sửa 1 nơi quên nơi khác).
- **`toOptions` map `{value, label}`** viết lại rời rạc ở 5 nơi; nên đưa vào `lib/options.ts`.
- **Format tiền/ngày** viết lại 3-5 nơi khác nhau với output không đồng nhất (rỗng vs "N/A" vs "x đ").
- **Mapping status → label/badge** của `UpdateRequest` định nghĩa 3 lần ở 2 file, dễ lệch khi đổi tên trạng thái.
- **Branch list tĩnh** (`/branches`) fetch lại mỗi lần mount view thay vì cache module-level như `loadGoogleMaps()` đã làm.
- **`AppSidebar.isActive`** có độ phức tạp ~O(n²) theo số menu item (hiện ~30 item ⇒ ~900 lần gọi `matchesRoute`/dựng `URLSearchParams` mỗi lần re-render) — chưa phải vấn đề ở quy mô hiện tại nhưng nên tính trước khi menu phình to.

---

## Tổng hợp mức độ ưu tiên đề xuất xử lý trước

| # | Mức độ | Vấn đề |
|---|--------|--------|
| 1.1 | Cao (đã xác nhận bằng build) | `border-strong` không tồn tại → mất màu viền nút outline toàn app |
| 2.1 | Cao | Zod chặn `uuid: null` → không sửa được request cũ, submit luôn fail |
| 1.2 | Trung bình | Sidebar hiện menu không đúng quyền, bấm vào bị redirect không rõ lý do |
| 2.2 | Trung bình | Mất dữ liệu form khi đổi loại hồ sơ, không cảnh báo |
| 2.22 | Trung bình | Risk logout domino nhiều tab khi refresh token lỗi |
| 2.12 | Trung bình | Badge thông báo chưa đọc không đồng bộ |
| 2.6 / 2.8 / 2.9 / 2.10 | Trung bình | Một nhóm lỗi UX quanh "trạng thái loading/kết quả" ở các màn import, chat MSB, in chứng thư |
