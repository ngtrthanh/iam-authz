# Policies and platform products

## Self-hosted identity options

- Pocket ID: OpenID Connect/passkeys, lightweight.
- Authelia: SSO/MFA/gateway-oriented.
- Dex: OIDC broker; use with an external identity source.
- LLDAP: LDAP directory, often needs an OIDC layer for modern apps.

## Authorization engines

- Cerbos: policy-as-code for RBAC/ABAC; no identity-provider responsibilities.
- OPA: flexible policy engine; more code/runtime integration is needed.
- OpenFGA: Zanzibar-style relationship authorization.
- SpiceDB: relationship graph and consistency-focused architecture.

## Production note

Liteweight does not mean low operational risk. Verify upstream license, Docker image signing, supported versions, backup/restore, upgrades, SLO, MFA, audit and fail-open/fail-closed behavior before deployment.
