# IAM/Authz skill

Repo này lưu kiến thức tái sử dụng để thiết kế đăng nhập và phân quyền dùng chung.

## Nội dung

- `SKILL.md`: quy trình, decision matrix và checklist.
- `examples/chinfon-kaizen.md`: phân tích kiến trúc mẫu thực tế từ Go + SQLite.

## Lựa chọn mặc định

Với một app nhỏ hoặc vài app nội bộ, bắt đầu bằng identity nhẹ và RBAC embedded. Chỉ centralize authorization nếu nhiều app bắt đầu sao chép hoặc lệch nhau.

MIT licensed.
