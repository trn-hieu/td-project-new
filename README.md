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
