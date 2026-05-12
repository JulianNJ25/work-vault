Openbao has no built-in admin role. When creating a policy it should sit between:
- **Root token** → unrestricted, dangerous
- **App / service tokens** → highly restricted
- **Admin token** → operational control, but **not root-only internals**
# Should:

- Enable/disable auth methods (`sys/auth/*`)
- Enable/disable secret engines (`sys/mounts/*`)
- Manage policies (`sys/policies/*`)
- Manage roles (database, approle, etc.)
- Manage tokens
- Read system health and metadata
- Handle leases (revoke, lookup)
# Should NOT:

Restricted to root:

- `sys/raw/*` (low-level storage)
- `sys/init`
- `sys/generate-root/*`
- `sys/audit/*` (often restricted)

# Full List of relevant admin endpoints
## System / Core

- `sys/seal`, `sys/unseal` (can be assigned to admin, it allowed)
- `sys/health`
- `sys/capabilities-self`
- `sys/leases/*`
- `sys/internal/ui/*`

## Auth Management

- `sys/auth/*` 
- `auth/*` (login flows, users, approle, etc.)

## Secrets Engines

- `sys/mounts/*`

## Policies

- `sys/policies/acl/*`

## Tokens

- `auth/token/*`
- `auth/token/lookup/*`

## Secret Engines (example: KV, DB)

- `secret/*`
- `database/*`

# Final admin config

```hcl
# =========================================
# ADMIN POLICY — OPENBAO (hardened)
# =========================================

# -------------------------
# SYSTEM
# -------------------------
path "sys/health" {
  capabilities = ["read"]
}
path "sys/capabilities" {
  capabilities = ["update"]
}
path "sys/capabilities-self" {
  capabilities = ["update"]
}
path "sys/capabilities-accessor" {
  capabilities = ["update"]
}
path "sys/internal/ui/*" {
  capabilities = ["read"]
}
path "sys/step-down" {
  capabilities = ["update", "sudo"]
}
path "sys/seal" {
  capabilities = ["update", "sudo"]
}
path "sys/unseal" {
  capabilities = ["update", "sudo"]
}
path "sys/rotate" {
  capabilities = ["update", "sudo"]
}

# -------------------------
# MOUNTS (SECRETS ENGINES)
# -------------------------
path "sys/mounts" {
  capabilities = ["read", "list"]
}
path "sys/mounts/*" {
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}

# -------------------------
# AUTH METHODS
# -------------------------
path "sys/auth" {
  capabilities = ["read", "list"]
}
path "sys/auth/*" {
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}
path "auth/approle/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
path "auth/userpass/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# -------------------------
# TOKENS
# -------------------------
path "auth/token/create" {
  capabilities = ["create", "update", "sudo"]
}
path "auth/token/create/*" {
  capabilities = ["create", "update", "sudo"]
}
path "auth/token/lookup" {
  capabilities = ["update"]
}
path "auth/token/lookup/*" {
  capabilities = ["read"]
}
path "auth/token/lookup-self" {
  capabilities = ["read"]
}
path "auth/token/renew" {
  capabilities = ["update"]
}
path "auth/token/renew-self" {
  capabilities = ["update"]
}
path "auth/token/revoke" {
  capabilities = ["update"]
}
path "auth/token/revoke-accessor" {
  capabilities = ["update"]
}
path "auth/token/revoke-self" {
  capabilities = ["update"]
}
path "auth/token/roles/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# -------------------------
# POLICIES
# -------------------------
path "sys/policies/acl" {
  capabilities = ["list"]
}
path "sys/policies/acl/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# -------------------------
# LEASES
# -------------------------
path "sys/leases/lookup" {
  capabilities = ["update"]
}
path "sys/leases/renew" {
  capabilities = ["update"]
}
path "sys/leases/revoke" {
  capabilities = ["update"]
}
path "sys/leases/revoke-prefix/*" {
  capabilities = ["update", "sudo"]
}
path "sys/leases/*" {
  capabilities = ["read", "update", "list"]
}

# -------------------------
# IDENTITY
# -------------------------
path "identity/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# -------------------------
# KV SECRETS
# -------------------------
path "secret/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# -------------------------
# DATABASE (DYNAMIC CREDS)
# -------------------------
path "database/config/*" {
  capabilities = ["create", "read", "update", "delete"]
}
path "database/roles/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
path "database/creds/*" {
  capabilities = ["read"]
}

```

