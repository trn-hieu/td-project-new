# td-project-new

## Cập nhật submodule

Cập nhật các submodule lên commit mới nhất của remote:

```bash
git submodule update --remote --merge
```

Các biến thể hay dùng:

- Lần đầu (submodule chưa được clone/khởi tạo):
  ```bash
  git submodule update --init --recursive --remote
  ```
- Chỉ cập nhật một submodule:
  ```bash
  git submodule update --remote <đường-dẫn-submodule>
  ```
- Cập nhật theo commit mà repo chính đang trỏ tới (không lấy bản mới nhất):
  ```bash
  git submodule update --init --recursive
  ```
- Chạy pull trong mọi submodule (thay `main` bằng branch của submodule):
  ```bash
  git submodule foreach 'git pull origin main'
  ```

Lưu ý:

- `--remote` lấy commit mới nhất của branch khai báo trong `.gitmodules` (`branch = ...`); nếu không có thì dùng HEAD mặc định của remote.
- Sau khi cập nhật, repo chính sẽ thấy con trỏ commit của submodule thay đổi, cần commit lại:
  ```bash
  git add <đường-dẫn-submodule>
  git commit -m "Update submodule"
  ```

## Kết nối Zalo OA

- Trang FE (không có menu, truy cập trực tiếp bằng URL, cần đăng nhập, không cần quyền admin): `<PUBLIC_BASE_URL>/admin/integrations/zalo`. Ví dụ local: `http://localhost:5173/admin/integrations/zalo`.
- API BE (cần đăng nhập):
  - `POST /api/integrations/zalo/connect` → `{ authorizeUrl }`
  - `GET /api/integrations/zalo/status` → `{ connected, oaId, accessTokenExpiresAt, refreshTokenExpiresAt, status, lastError }` (không trả token)
  - `DELETE /api/integrations/zalo`
- Callback công khai cho Zalo: `GET /api/public/zalo/callback`. Đăng ký `https://<domain-backend>/api/public/zalo/callback` trên developers.zalo.me, trùng với `ZALO_REDIRECT_URI`. Callback luôn redirect (302) về trang FE ở trên kèm `?result=success` hoặc `?result=error&reason=<mã>`.
- Biến môi trường (backend): `ZALO_ENABLED` (mặc định `false`, tắt thì các endpoint trả 404), `ZALO_APP_ID`, `ZALO_SECRET_KEY`, `ZALO_REDIRECT_URI`, `ZALO_OA_ID`, `ZALO_TOKEN_ENCRYPTION_KEY`
