<a name="top"></a>
[![decodeRing Core Server](https://decodering.org/wp-content/uploads/2025/11/Git-Banner-2-scaled.png)](https://decodering.org)
![Version](https://img.shields.io/badge/Version-v0.1--draft-blue) [![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0) ![Status](https://img.shields.io/badge/Status-Working_Draft-orange) [![Contributions Welcome](https://img.shields.io/badge/Contributions-Welcome-brightgreen)](https://github.com/decodeRing-core/dcdr-standard/blob/main/CONTRIBUTING.md)

# Open Secrets Language (OSL) — Abstraction API v1.0.0

Source reference: current OSL v0.1-draft in the DCDR standard README: https://github.com/decodeRing-core/dcdr-standard/blob/main/README.md

## 1) Goals

This version updates OSL into a provider-agnostic abstraction that can map cleanly onto:

- HashiCorp Vault
- OpenBao
- HCP Vault
- AWS Secrets Manager
- Azure Key Vault
- Google Cloud Secret Manager
- CyberArk Conjur
- Kubernetes External Secrets Operator
- Doppler
- Delinea Secret Server

### Key abstraction strategy

Different providers support different features (e.g., versioning, dynamic credentials, sync/injection). This API:

1. Defines a **small required core** that all backends can implement.
2. Adds optional modules (leases, rotation, sync) that are **capability-gated**.
3. Makes **capability discovery** mandatory so clients never guess.

## 2) Versioning and naming

- **Major version in the URL path**: `/osl/v1/...`
- **Spec version returned in responses**: `"osl_version": "1.0.0"`
- **Kebab-case** for endpoint paths.
- **Snake-case** for JSON fields.

## 3) Authentication

Clients MUST send a bearer token on every request:

```
Authorization: Bearer <your-token>
```

## 4) Standard response/error envelopes

### Success envelope (recommended)

All 2xx responses SHOULD include:

```json
{
  "osl_version": "1.0.0",
  "status": "operating-completed",
  "message": "Operation Completed",
  "data": {}
}
```

### Error envelope (required for non-2xx)

All non-2xx responses MUST return:

```json
{
  "osl_version": "1.0.0",
  "error": {
    "code": "operation-failed",
    "message": "Operation failed.",
    "detail": "Node is not initialized."
  }
}
```

## 5) Common identifiers

- `app_id`: application scope
- `backend_ref`: configured backend instance reference (maps to current `backend`)
- `secret_name`: logical name within an app
- `store_path`: provider-native secret identifier/path

## 6) Capability discovery (mandatory)

### `GET /osl/v1/capabilities/get`

Clients SHOULD call this at startup and cache responses.

**Response (example):**

```json
{
  "osl_version": "1.0.0",
  "status": "operating-completed",
  "message": "Operation Completed",
  "data": {
    "server_capabilities": [
      "kv.read",
      "kv.write",
      "kv.delete",
      "kv.taint",
      "sync.manage",
      "lease.issue"
    ],
    "backends": [
      {
        "backend_ref": "vault-1",
        "type": "vault",
        "capabilities": [
          "kv.read",
          "kv.write",
          "kv.versioning",
          "lease.issue",
          "lease.renew",
          "lease.revoke"
        ]
      },
      {
        "backend_ref": "aws-1",
        "type": "aws-secrets-manager",
        "capabilities": [
          "kv.read",
          "kv.write",
          "kv.versioning",
          "rotation.policy"
        ]
      },
      {
        "backend_ref": "eso-1",
        "type": "kubernetes-external-secrets-operator",
        "capabilities": ["sync.manage", "sync.run", "sync.status"]
      }
    ]
  }
}
```

## 7) Required core: KV secret lifecycle

These endpoints MUST be implemented by an OSL v1 server.

### 7.1) Put secret (create/update)

#### `POST /osl/v1/secrets/put`

Creates or updates a secret.

**Request:**

```json
{
  "app_id": "your-app-id",
  "secret_name": "my-database-credentials",
  "store": {
    "backend_ref": "aws-1",
    "store_path": "production/my-database-credentials"
  },
  "data": {
    "username": "db_user",
    "password": "super_secret_password"
  },
  "options": {
    "create_only": false
  }
}
```

**Response (example):**

```json
{
  "osl_version": "1.0.0",
  "status": "operation-completed",
  "message": "Operating completed",
  "data": {
    "secret_name": "my-database-credentials",
    "provider_version_id": "1"
  }
}
```

### 7.2) Get secret

#### `POST /osl/v1/secrets/get`

Retrieves secret data. Supports selecting a version when available.

**Request:**

```json
{
  "app_id": "your-app-id",
  "secret_name": "my-database-credentials",
  "version": 0 // Optional defaults to latest if not sent
}
```

**Response (example):**

```json
{
  "osl_version": "1.0.0",
  "status": "operation-completed",
  "message": "Operating completed",
  "data": {
    "password": "super_secret_password",
    "username": "db_user",
    "metadata": {
      "resolved_backend_ref": "openbao-rs",
      "provider_version_id": "3"
    }
  }
}
```

### 7.3) Destroy secret

#### `POST /osl/v1/secrets/destroy`

Permanently deletes a secret from the provider backend.

**Request:**

```json
{
  "app_id": "your-app-id",
  "secret_name": "my-database-credentials"
}
```

**Response (example):**

```json
{
  "osl_version": "1.0.0",
  "status": "operation-completed",
  "message": "Operating completed",
  "data": {
    "destroyed": true
  }
}
```

### 7.4) Delete secret

#### `POST /osl/v1/secrets/delete`

Soft deletes a secret from the provider backend if supported

**Request:**

```json
{
  "app_id": "your-app-id",
  "secret_name": "my-database-credentials"
}
```

**Response (example):**

```json
{
  "osl_version": "1.0.0",
  "status": "operation-completed",
  "message": "Operating completed",
  "data": {
    "deleted": true
  }
}
```

### 7.5) List secrets

#### `POST /osl/v1/secrets/list`

Lists secrets for an application.

**Request:**

```json
{
  "app_id": "your-app-id"
}
```

**Response (example):**

```json
{
  "osl_version": "1.0.0",
  "status": "operation-completed",
  "message": "Operating completed",
  "data": [
    {
      "secret_name": "my-database-credentials",
      "backend": "openbao-rs",
      "mount_path": "production-test/my-database-credentials",
      "tainted": false
    }
  ]
}
```

### 7.6) Describe secret

#### `POST /osl/v1/secrets/describe`

Returns provider-agnostic metadata plus provider-native hints (safe metadata only).

**Request:**

```json
{
  "app_id": "your-app-id",
  "secret_name": "my-database-credentials"
}
```

## 8) Required core: DCDR-style tainting (server-level)

These endpoints MUST be implemented by an OSL v1 server and operate at the OSL server layer (not necessarily the provider layer).

### 8.1) Taint

#### `POST /osl/v1/secrets/taint`

**Request:**

```json
{ "app_id": "your-app-id", "secret_name": "my-database-credentials" }
```

### 8.2) Untaint

#### `POST /osl/v1/secrets/untaint`

**Request:**

```json
{ "app_id": "your-app-id", "secret_name": "my-database-credentials" }
```

### 8.3) Is tainted

#### `POST /osl/v1/secrets/is-tainted`

**Request:**

```json
{ "app_id": "your-app-id", "secret_name": "my-database-credentials" }
```

**Response (example):**

```json
{
  "osl_version": "1.0.0",
  "data": { "tainted": true }
}
```

## 9) Optional module: Secret versioning

This module is available when backend has `kv.versioning`.

- `POST /osl/v1/secrets/versions/list`
- `POST /osl/v1/secrets/versions/get`

Servers MUST return `feature-not-supported` when a client calls these against a backend that lacks versioning.

## 10) Optional module: Dynamic credentials / checkout (leases)

This module abstracts:

- Vault/OpenBao/HCP Vault dynamic secrets (leases)
- “checkout”-style flows in enterprise vaults when available

Available when backend has `lease.issue`.

### 10.1) Issue credential

#### `POST /osl/v1/credentials/issue`

**Request:**

```json
{
  "app_id": "your-app-id",
  "credential_name": "db-readonly",
  "backend_ref": "vault-1",
  "parameters": {
    "role": "readonly",
    "ttl_seconds": 3600
  }
}
```

**Response (example):**

```json
{
  "osl_version": "1.0.0",
  "data": {
    "username": "v-generated-user",
    "password": "v-generated-pass",
    "lease": {
      "lease_id": "lease-abc",
      "expires_at": "2026-04-20T18:20:00Z",
      "renewable": true
    }
  }
}
```

### 10.2) Renew credential

#### `POST /osl/v1/credentials/renew`

```json
{ "lease_id": "lease-abc", "extend_ttl_seconds": 3600 }
```

### 10.3) Revoke credential

#### `POST /osl/v1/credentials/revoke`

```json
{ "lease_id": "lease-abc" }
```

## 11) Optional module: Rotation

Rotation differs significantly per provider. This abstraction supports:

- Policy definition (when supported)
- Manual rotate trigger (best-effort)

Available when backend has `rotation.policy` and/or `rotation.rotate`.

### 11.1) Put rotation policy

#### `POST /osl/v1/rotation-policies/put`

```json
{
  "app_id": "your-app-id",
  "secret_name": "my-database-credentials",
  "policy": {
    "mode": "scheduled",
    "interval_seconds": 604800
  }
}
```

### 11.2) Rotate secret

#### `POST /osl/v1/secrets/rotate`

```json
{ "app_id": "your-app-id", "secret_name": "my-database-credentials" }
```

## 12) Optional module: Sync / materialization

This module abstracts:

- Kubernetes External Secrets Operator (ESO)
- Doppler sync/injection patterns

Available when backend (or server) has `sync.manage`.

### 12.1) Put sync

#### `POST /osl/v1/syncs/put`

```json
{
  "app_id": "your-app-id",
  "sync_name": "prod-db-to-k8s",
  "source": {
    "secret_name": "my-database-credentials",
    "version": "latest"
  },
  "target": {
    "type": "kubernetes-secret",
    "cluster_ref": "cluster-1",
    "namespace": "production",
    "name": "db-credentials",
    "template": {
      "type": "opaque"
    }
  }
}
```

### 12.2) Run sync

#### `POST /osl/v1/syncs/run`

```json
{ "app_id": "your-app-id", "sync_name": "prod-db-to-k8s" }
```

### 12.3) Get sync status

#### `POST /osl/v1/syncs/status/get`

```json
{ "app_id": "your-app-id", "sync_name": "prod-db-to-k8s" }
```

### 12.4) List / delete syncs

- `POST /osl/v1/syncs/list`
- `POST /osl/v1/syncs/delete`

## 13) Management API (mapped from current draft)

### 13.1) List applications

#### `GET /osl/v1/apps/list`

### 13.2) List backends

#### `GET /osl/v1/backends/list`

## 14) Migration mapping (v0.1-draft → v1.0.0)

- `POST /api/dcdrCreateSecret` → `POST /osl/v1/secrets/put`
- `POST /api/dcdrGet` → `POST /osl/v1/secrets/get`
- `POST /api/dcdrDestroy` → `POST /osl/v1/secrets/delete`
- `POST /api/dcdrTaint` → `POST /osl/v1/secrets/taint`
- `POST /api/dcdrUntaint` → `POST /osl/v1/secrets/untaint`
- `POST /api/dcdrIsTainted` → `POST /osl/v1/secrets/is-tainted`
- `POST /api/dcdrListSecrets` → `POST /osl/v1/secrets/list`
- `GET /api/dcdrListApps` → `GET /osl/v1/apps/list`
- `GET /api/dcdrListBackends` → `GET /osl/v1/backends/list`

## 15) Implementation note

To keep the abstraction honest across all listed systems:

- Treat only the **core** as universally supported.
- Gate everything else behind `capabilities/get`.
- Return structured `feature-not-supported` errors for optional module calls.
