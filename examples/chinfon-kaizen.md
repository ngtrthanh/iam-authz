# Example: Chinfon Kaizen 360

Ví dụ dựa trên [ngtrthanh/chinfon-kaizen](https://github.com/ngtrthanh/chinfon-kaizen).

## Kiến trúc hiện tại

- Go + `net/http`, single binary.
- SQLite WAL; master data, user, role, permission và session trong cùng database.
- Password: PBKDF2-SHA256, phiên đầu bằng PIN từ mobile.
- MFA TOTP bắt buộc cho mọi tài khoản.
- Session cookie `HttpOnly`, `SameSite=Strict`; session trước MFA bị giới hạn 10 phút.
- RBAC permission-based: role cộng dồn thành permissions.
- Scope theo `ALL` hoặc mã phòng ban.
- `VisibilityClause` và `CanView` chặn visibility.
- `CanAct` chặn action theo permission, scope và trạng thái hồ sơ.
- Audit log, workflow history append-only và trigger bất biến.
- Không có Docker/SSO/AD sync; một node, SQLite, master data nội bộ.

## Mô hình quyền

```text
User
  -> Employee
  -> Department
  -> dept_scope
  -> Roles
       -> Permissions

Action
  -> Permission check
  -> Department scope check
  -> Record state check
  -> Record ownership/participant check
```

## Permission mẫu

| Permission | Ý nghĩa |
|---|---|
| `idea:create` | Tạo sáng kiến |
| `idea:screen:dept` | Xác nhận bước screening |
| `idea:review:roi` | Thẩm định ROI |
| `idea:approve:final` | Phê duyệt cuối |
| `track:confidential:read` | Xem Luồng 3 confidential |
| `hr:payroll:export` | Xuất bảng lương |
| `rbac:admin` | Quản trị role/permission |

## Role mẫu

| Role | Permission chính | Scope |
|---|---|---|
| `WORKER` | create | Department + hồ sơ của mình |
| `SUPERVISOR` | create, screen | Department |
| `MANAGER` | create, screen, review | Department |
| `DIRECTOR` | create, screen, review, final approve | ALL |
| `COUNCIL` | screen, review, final approve | ALL |
| `HRD` | create, payroll export | ALL |
| `CEO_AUDIT` | toàn bộ và de-classify confidential | ALL |
| `ADMIN` | toàn bộ trong RBAC matrix | ALL |

## Quy tắc visibility

1. Ẩn `is_demo=1` khi demo mode tắt.
2. Người nộp vẫn xem hồ sơ của mình, kể cả confidential.
3. Confidential cần permission `track:confidential:read`.
4. Worker chưa có broad visibility chỉ thấy hồ sơ mình nộp hoặc là participant.
5. Role có broad visibility nhưng không có `ALL` chỉ thấy department.
6. Admin hoặc CEO scope `ALL` xem cả phạm vi doanh nghiệp theo permission.

## Áp dụng backend trong Go

```go
// Hành động phải kiểm tra cả visibility, permission, scope và trạng thái.
func (u *User) CanAct(i *Initiative, perm string) bool {
    if u == nil || i == nil {
        return false
    }
    if !u.CanView(i) || !u.HasPerm(perm) {
        return false
    }
    if i.RecordStatus != StatusOpen {
        return false
    }
    if u.ScopeAll() {
        return true
    }
    return i.DepartmentCode == u.Scope
}
```

Dùng cùng nguyên tắc cho query danh sách: SQL visibility phải nhận `dept_scope`, ownership và permission đã được validate từ session, không nhận scope từ request.

## Nếu chỉ scale một app

Giữ thiết kế hiện tại, nhưng nên tách ba phần:

- Identity service nhẹ: Pocket ID/Authelia, hoặc LDAP/AD adapter.
- Policy module: giữ trong Go hoặc extract thành package/testable policy engine.
- Application: SQLite một node, vẫn enforce quyền trong mỗi handler/query.

Đây là mức phức tạp hợp lý cho Chinfon Kaizen trong giai đoạn một app/single binary.

## Nếu thêm app thứ hai

```text
Pocket ID / Authelia
  -> OIDC login
  -> user employee_no
  -> roles từ application hoặc identity claims

Chinfon Kaizen / App khác
  -> verify OIDC token
  -> map employee_no -> department/scope
  -> local RBAC policy
  -> SQL/API resource check
```

Không chỉ copy JWT claims role nếu quyền thay đổi nhanh: role claim có thể stale. Dùng token ngắn hạn hoặc permission lookup/cache TTL ngắn.

## Khi nào thêm Cerbos

Thêm Cerbos khi:

- Hơn hai app bắt đầu sao chép policy.
- Policy thay đổi thường xuyên mà cần rollout/test/audit.
- Nhiều service cần cùng dạng permission và effect.
- Cần policy-as-code thay vì migration SQL riêng từng app.

Trong kiến trúc hiện tại, Cerbos không thay thế MFA/session; nó chỉ nhận principal, action và resource để quyết định allow/deny.

## Khi nào thêm OpenFGA

Chỉ thêm OpenFGA nếu mô hình quan hệ vượt role + department. Ví dụ:

- User thuộc team/project.
- Project có folder/document.
- User có quyền riêng theo participant/reviewer.
- Truy vấn “tôi thấy những resource nào?” phức tạp và lặp lại.

Với một initiative có submitter/participant/reviewer, vẫn có thể implement bằng ownership/participants trong SQL, không nhất thiết dùng ReBAC ngay.

## Migration đề xuất

### Phase 1

- Giữ password/session/MFA hiện tại.
- Chuẩn hóa permission code.
- Thêm policy test matrix.
- Thêm audit cho thay đổi role/permission.
- Giao diện RBAC chỉ cho phép scope hợp lệ.

### Phase 2

- Thêm OIDC identity provider.
- Liên kết employee identity bằng employee number.
- Giữ một user record canonical.
- Cho phép login SSO nhưng không đưa quyền hoàn toàn trong JWT.

### Phase 3

- Extract authorization policy thành module/service nếu nhiều app.
- Chỉ centralize khi lợi ích vượt chi phí vận hành.
- Không chuyển sang PostgreSQL/OpenFGA/Keycloak chỉ vì mục tiêu dài hạn.

## Bài test tối thiểu

- Worker xem hồ sơ của chính mình.
- Worker không xem hồ sơ khác cùng department.
- Manager không xem department khác.
- Confidential không lộ cho role không có permission.
- Submitter không tự approve/review.
- Closed/rejected record không nhận thêm action.
- Demo record bị ẩn trong SQL và UI.
- Admin role mất một permission thì handler/query trả deny ngay lập tức.
- Login thiếu MFA không truy cập business route.
- POST thiếu CSRF bị từ chối.
