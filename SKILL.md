# IAM / Authorization Design Skill

Dùng skill này để thiết kế hoặc mở rộng xác thực và phân quyền cho ứng dụng. Ưu tiên giải pháp đơn giản nhất đáp ứng nhu cầu, có thể kiểm thử và có ranh giới bảo mật rõ ràng.

## Khi nào dùng skill

- Thêm đăng nhập dùng chung cho nhiều ứng dụng.
- Chuyển từ RBAC nhúng trong app sang identity hoặc authorization service.
- Thiết kế Docker image/binary, YAML và environment config.
- Tạo ma trận role, permission, scope và luồng kiểm tra quyền.
- Ước lượng CAPEX/OPEX hoặc viết decision record.

## Quy tắc thiết kế

1. Tách Authentication khỏi Authorization.
2. Backend là enforcement point; frontend ẩn nút không phải cơ chế bảo vệ.
3. Permission phải đặt ở mức action + resource, ví dụ `initiative:approve:final`.
4. Scope dữ liệu phải chặn ở SQL/query, không chỉ ở handler hoặc UI.
5. Bắt đầu bằng RBAC. Chỉ thêm ABAC/ReBAC khi mô hình quyền thực sự yêu cầu.
6. Không tự xây OAuth/OIDC server khi nhu cầu chỉ cần IdP nhẹ và thương mại hóa đơn giản.
7. Token ngắn hạn; mất quyền hoặc vai trò phải có cơ chế revoke/cached policy TTL.
8. Mặc định deny; kiểm tra tài nguyên cụ thể thay vì chỉ kiểm tra endpoint.
9. Secret không nằm trong image, Git hoặc YAML public.
10. Audit mọi thay đổi vai trò, quyền, đăng nhập, từ chối và hành động nhạy cảm.

## Phân loại giải pháp

| Cấp độ | Khi dùng | Ví dụ |
|---|---|---|
| Embedded | Một app hoặc ít app, không cần SSO | Session + RBAC trong backend |
| Identity nhẹ | SSO/OIDC/MFA dùng chung | Pocket ID, Authelia, Dex |
| Policy centralized | Nhiều app dùng chung policy | Cerbos, OPA |
| Relationship-based | User/team/project/resource phức tạp | OpenFGA, SpiceDB |
| Enterprise IAM | Federation, SAML, SCIM, compliance | Keycloak, Okta, Entra |

## Quy trình đề xuất

1. Liệt kê app, user, tenant, role, permission và resource.
2. Phân biệt identity, session, role, scope và ownership.
3. Vẽ các quyết định: đăng nhập, đọc, tạo, duyệt, xuất dữ liệu, quản trị RBAC.
4. Chọn mô hình phân quyền trước; chọn công nghệ sau.
5. Viết ma trận role × permission.
6. Viết ma trận role/permission × hành động dữ liệu.
7. Ghi rõ repository, SQL/API enforcement và audit.
8. Ước lượng CAPEX, OPEX và migration risk.
9. Viết test cho allow/deny và scope.

## Mẫu decision record

```markdown
# ADR: Shared Authentication and Authorization

## Context
- Number of apps:
- Number of users:
- Tenant model:
- Existing stack:
- Offline or network constraints:

## Decision
- Authentication:
- Authorization:
- Session model:
- Scope model:
- Deployment: binary, Docker, service, or library

## Security invariants
- Backend enforces every operation.
- Tenant/scope is server-derived.
- Policy deny-by-default.
- Secrets excluded from Git/images.
- Audit covers sensitive operations.

## Consequences
- CAPEX:
- OPEX:
- Migration effort:
- Remaining risks:
```

## Ma trận CAPEX/OPEX tương đối

| Giải pháp | CAPEX | OPEX | Phù hợp |
|---|---:|---:|---|
| RBAC embedded | 1 | 1 | 1 app, SQLite/small team |
| Pocket ID / Authelia | 2 | 1–2 | Vài app cần SSO/MFA |
| Dex + external directory | 2–3 | 1–2 | Đã có identity source |
| Keycloak | 3–4 | 2–3 | Federation/SAML/SCIM nhiều ứng dụng |
| Cerbos | 2–3 | 1–2 | Policy chung cho nhiều backend |
| OpenFGA / SpiceDB | 4 | 2–3 | Quan hệ team/project/resource phức tạp |
| Custom IAM | 5 | 5 | Chỉ có requirement pháp lý/đặc thù bắt buộc |

Thang điểm 1–5. Không quy đổi thành tiền cho đến khi có MAU, số app, availability, regions và SLA cụ thể.

## Docker image hay binary?

- Chọn **binary nhúng** nếu app là monolith và không cần chính sách giữa các dịch vụ.
- Chọn **Docker service** khi nhiều app, nhiều ngôn ngữ cần quyết định hoặc policy chung.
- Chọn **SDK/thư viện** khi app cùng ngôn ngữ, policy ít và cập nhật đồng bộ được.
- Cấu hình YAML cho cấu hình không bí mật.
- Dùng environment hoặc secret manager cho mật khẩu/key/client secret.
- Ghim image tag/digest; không dùng `latest` ở production.
- Có health/readiness, timeout, circuit breaker và deny-safe behavior.

## Template cấu hình

```yaml
server:
  address: 0.0.0.0:8080

identity:
  issuer: ${OIDC_ISSUER}
  audience: [kaizen-api]

authorization:
  default: deny
  cache_ttl_seconds: 30
  fail_mode: closed

observability:
  audit_enabled: true
  log_level: info
```

Không commit các giá trị như `DB_PASSWORD`, private key, MFA key hoặc API secret vào file này.

## Policy checklist

- [ ] Định danh người dùng canonical.
- [ ] Xác định tenant/department/group scope.
- [ ] Map role sang permission.
- [ ] Map permission sang handler/query/workflow action.
- [ ] Kiểm tra owner/participant trong resource-level check.
- [ ] Loại confidential/demo nội dung khỏi truy vấn.
- [ ] Kiểm tra trạng thái record trước khi action.
- [ ] Bảo vệ POST bằng CSRF nếu dùng cookie session.
- [ ] Có MFA/step-up cho quyền nhạy cảm.
- [ ] Có revoke session và role change.
- [ ] Có test allow/deny/scope.
- [ ] Có audit log append-only.
- [ ] Có backup/rotation cho MFA key và session secret.
