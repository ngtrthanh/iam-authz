# Scope: Chinfon Kaizen 360

> Repository: [ngtrthanh/chinfon-kaizen](https://github.com/ngtrthanh/chinfon-kaizen)
>
> Trạng thái: internal pilot / trusted LAN-Tailscale. Chưa coi là sole authoritative payroll or public whistleblowing system.

## Boundaries

### In scope

- Go single binary, embedded templates/CSS/JS và SQL migrations.
- SQLite single-node; dữ liệu app, master data, sessions, RBAC và audit nằm trong cùng database.
- Password bootstrap bằng mobile PIN, bắt buộc đổi password và MFA TOTP.
- RBAC theo role/permission; scope theo `ALL` hoặc department code.
- Ownership/participant check cho hồ sơ do người dùng nộp hoặc tham gia.
- Visibility ở SQL (`VisibilityClause`) và object/action check (`CanView`, `CanAct`).
- Workflow state, record status, review history, confidential track và audit append-only.
- Demo mode off theo migration/setting; không seed dữ liệu demo cho production.
- Staff import từ CSV; upsert department, employee, account và role trong transaction.
- Backup SQLite + uploads + MFA key; restore/recovery test.
- LAN/Tailscale-only pilot, hoặc HTTPS reverse proxy nếu cần public access.

### Out of scope for the first release

- PostgreSQL hoặc horizontal scaling.
- Central identity provider bắt buộc cho một app.
- SAML, SCIM hoặc enterprise federation.
- Keycloak chỉ để triển khai một app.
- OpenFGA/SpiceDB nếu quyền vẫn biểu diễn đủ bằng role, department, ownership và participant.
- Central Cerbos nếu chưa có ít nhất hai app sao chép policy hoặc policy quá phức tạp để quản lý local.
- Public identity provider hoặc public rate-limit service.
- E-signature hoặc mobile native app.
- Email/push notification; SLA escalation hiện chỉ ở dashboard.
- HR system real-time sync.
- Native XLSX parser.

## Current IAM model

```text
Employee
  -> Department
  -> app_user
       -> app_user_role
            -> rbac_role_permission
                 -> permission map

Session cookie
  -> app_session
  -> mfa_passed
  -> currentUser permissions
  -> dept_scope

Resource
  -> submitter / participant
  -> department
  -> track
  -> record status / stage
  -> visibility/action check
```

## Security invariants

1. Browser chỉ render UI; mọi quyết định allow/deny thuộc backend.
2. `dept_scope` không được lấy từ request hoặc URL parameter.
3. SQL danh sách phải áp dụng visibility clause.
4. Confidential record luôn cần permission `track:confidential:read` trừ submitter xem chính hồ sơ của mình.
5. Record closed/rejected không được transition thêm.
6. Submitter không tự screen/review/approve hồ sơ của mình.
7. MFA phải hoàn thành trước business route; partial session chỉ truy cập MFA/setup/logout.
8. POST phải có CSRF; login/setup có login CSRF.
9. Session token lưu dạng SHA-256 hash; cookie có HttpOnly/SameSite/optional Secure.
10. Role/permission change có hiệu lực cho user load tại request, nhưng phải có audit và test.
11. Secret, MFA key, session cookie và password hash không public hoặc không commit vào repository.

## Deployment criteria

Không mở public hoặc chỉ coi là production khi tất cả điều kiện sau đã hoàn tất:

- Thay đổi hoặc nâng cường password policy; xem xét SSO/AD thay local password.
- Rate limit lockout dùng persistent/shared store; bỏ giới hạn in-memory nếu chạy nhiều instance.
- Xác nhận PIN bootstrap có được phép ở quy mô nhân sự thực tế.
- MFA bắt buộc cho mọi role và có break-glass recovery đã test.
- Backup và restore được test định kỳ.
- MFA key được backup riêng và rotation/recovery procedure được ghi.
- HTTPS/reverse proxy; `COOKIE_SECURE=true`; CSP/HSTS/TLS kiểm tra end-to-end.
- Staff import được dry-run và reconcile trước mỗi import lớn.
- HR review ma trận role/scope, đặc biệt `ALL`, `CEO_AUDIT`, `ADMIN`.
- Luồng confidential submitter visibility được xác nhận chính sách.
- Payroll export và reward override được review độc lập.
- Audit log retention và incident response được xác định.

## Next decision points

### Keep local when

- Chỉ một app.
- User identity tương đương employee master trong SQLite.
- Role/permission thay đổi chậm.
- Department scope là ranh giới chính.
- Single-node là yêu cầu thực tế và không cần federation.

### Introduce Pocket ID or Authelia when

- Có nhiều ứng dụng cần cùng SSO.
- Muốn MFA/quy trình password tập trung.
- Cần single sign-on nhưng vẫn giữ policy nghiệp vụ local.

Pocket ID/Authelia không bỏ policy ở `VisibilityClause`/`CanAct`; identity chỉ thay thế nguồn xác thực và session identity.

### Introduce Cerbos when

- Có ít nhất hai app bắt đầu sao chép policy.
- Policy cần test/review/rollout độc lập nhiều team.
- Permission và resource action cần policy-as-code dùng chung.

Không thêm Cerbos chỉ vì số role tăng nhẹ.

### Introduce ReBAC when

- Mô hình quyền phát triển thành nhiều quan hệ: team, project, folder, reviewer, participant.
- Cần truy vấn “user có thể xem resource nào” theo quan hệ phức tạp.
- Role + department + ownership không còn đủ để mô tả quan hệ.

Từ role/department/owner của Chinfon Kaizen không phải bằng chứng cần ReBAC ngay.

## Success criteria

Mức scope này được xem là ổn khi:

- Người dùng chỉ truy cập record được phép.
- Action tuân thủ permission, scope, trạng thái và ownership.
- Confidential không suy đoán hoặc bị list leak.
- Admin có thể quản trị role/permission nhưng không tự bypass policy ngoài matrix.
- Test tự động che tất cả allow/deny/scope cơ bản.
- Mỗi thay đổi role/permission và action nhạy cảm được audit.
- App có thể vận hành offline hoặc LAN mà không phụ thuộc network identity ngoài.
