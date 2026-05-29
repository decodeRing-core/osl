<a name="top"></a>
[![decodeRing Core Server](https://org-web1.decodering.org/images/dcdr_banner.png)](https://decodering.org)
![Version](https://img.shields.io/badge/Version-v1--draft-blue) [![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0) ![Status](https://img.shields.io/badge/Status-Working_Draft-orange) [![Contributions Welcome](https://img.shields.io/badge/Contributions-Welcome-brightgreen)](https://github.com/decodeRing-core/dcdr-standard/blob/main/CONTRIBUTING.md)

# Open Secrets Language (OSL)

> One API standard for managing secrets across every major vault and secrets provider.

OSL is an open API standard that abstracts secrets management across providers - HashiCorp Vault, OpenBao, AWS Secrets Manager, Azure Key Vault, GCP Secret Manager, Kubernetes ESO, Doppler, CyberArk Conjur, and more - behind a single, consistent interface.

Instead of writing provider-specific integrations for every backend your team uses, OSL gives you one standard your apps, pipelines, and platform tooling can rely on - regardless of what's underneath.

---

## Why OSL exists

Secrets management is fragmented. Every provider has a different API, different auth model, different feature set, and different failure modes. Multi-cloud teams end up with:

- App code tightly coupled to specific vault SDKs
- Painful migrations when switching or consolidating providers
- Inconsistent secret lifecycle handling across environments
- No standard way to discover what a backend actually supports

OSL solves this by defining a **small required core** every compliant server must implement, plus **optional capability-gated modules** for advanced features like versioning, dynamic credentials, rotation, and sync - so clients never have to guess what a backend supports.

---

## How it works

```
Your App / CLI / SDK
        │
        ▼
  OSL-compliant server  (e.g. decodeRing core-server)
        │
        ├── HashiCorp Vault
        ├── OpenBao
        ├── AWS Secrets Manager
        ├── Azure Key Vault
        ├── GCP Secret Manager
        ├── Kubernetes ESO
        └── ... more backends
```

Clients call one standard API. The server handles provider-specific translation.

---

## Quick example

Write a secret - works the same regardless of backend:

```http
POST /osl/v1/secrets/put
Authorization: Bearer <token>

{
  "app_id": "billing-api",
  "secret_name": "database-creds",
  "store": {
    "backend_ref": "vault-1",
    "store_path": "prod/database-creds"
  },
  "data": {
    "username": "app_user",
    "password": "super-secret"
  }
}
```

Read it back:

```http
POST /osl/v1/secrets/get
Authorization: Bearer <token>

{
  "app_id": "billing-api",
  "secret_name": "database-creds"
}
```

Response:

```json
{
  "osl_version": "1.0.0",
  "status": "operation-completed",
  "message": "Operation completed",
  "data": {
    "username": "app_user",
    "password": "super-secret",
    "metadata": {
      "resolved_backend_ref": "vault-1",
      "provider_version_id": "1"
    }
  }
}
```

Same client code. Any supported backend.

---

## Who is OSL for?

- **Platform engineers** consolidating secrets infrastructure across clouds or vendors
- **Security teams** who need consistent lifecycle management and auditability
- **App developers** who want one integration that works everywhere
- **Vendors and OSS maintainers** building OSL-compatible servers or backend adapters

---

## Spec status

| Component | Status |
|---|---|
| OSL v1.0.0 spec | Alpha draft |
| Reference implementation | [decodeRing core-server](https://github.com/decodeRing-core/core-server) (alpha) |
| Go SDK | Available (alpha) |
| Python SDK | Available (alpha) |
| Breaking changes | Expected before stable release |

> ⚠️ OSL v1.0.0 is an alpha draft. The spec is open for feedback and contributions. Do not use in production.

---

## Ecosystem

| Repo | Description |
|---|---|
| [osl](https://github.com/decodeRing-core/osl) | This repo - the OSL API standard |
| [core-server](https://github.com/decodeRing-core/core-server) | Reference OSL server implementation (Go) |
| [dcdr-standard](https://github.com/decodeRing-core/dcdr-standard) | Underlying dcdr standard reference |

---

## Supported backends

| Backend | Core KV | Versioning | Dynamic Creds | Rotation | Sync |
|---|---|---|---|---|---|
| HashiCorp Vault | ✅ | ✅ | ✅ | ✅ | — |
| OpenBao | ✅ | ✅ | ✅ | ✅ | — |
| AWS Secrets Manager | ✅ | ✅ | — | ✅ | — |
| Azure Key Vault | ✅ | — | — | — | — |
| GCP Secret Manager | ✅ | — | — | — | — |
| Kubernetes ESO | — | — | — | — | ✅ |
| Doppler | ✅ | — | — | — | ✅ |
| CyberArk Conjur | ✅ | — | ✅ | — | — |

> Capability support is declared at runtime via `GET /osl/v1/capabilities/get`. Clients should always discover capabilities rather than assume them.

---

## Capability discovery

Clients SHOULD call this at startup and cache the response:

```http
GET /osl/v1/capabilities/get
Authorization: Bearer <token>
```

Response:

```json
{
  "osl_version": "1.0.0",
  "status": "operation-completed",
  "message": "Operation completed",
  "data": {
    "server_capabilities": ["kv.read", "kv.write", "kv.delete", "kv.taint", "sync.manage", "lease.issue"],
    "backends": [
      {
        "backend_ref": "vault-1",
        "type": "vault",
        "capabilities": ["kv.read", "kv.write", "kv.versioning", "lease.issue", "lease.renew", "lease.revoke"]
      },
      {
        "backend_ref": "aws-1",
        "type": "aws-secrets-manager",
        "capabilities": ["kv.read", "kv.write", "kv.versioning", "rotation.policy"]
      }
    ]
  }
}
```

---

## API reference

### Conventions

- **Major version in URL path**: `/osl/v1/...`
- **Spec version in responses**: `"osl_version": "1.0.0"`
- **Endpoint paths**: kebab-case
- **JSON fields**: snake_case
- **Auth**: Bearer token on every request

```http
Authorization: Bearer <your-token>
```

### Response envelopes

**Success (2xx):**

```json
{
  "osl_version": "1.0.0",
  "status": "operation-completed",
  "message": "Operation completed",
  "data": {}
}
```

**Error (non-2xx):**

```json
{
  "osl_version": "1.0.0",
  "error": {
    "code": "operation-failed",
    "message": "Operation failed",
    "detail": "Node is not initialized."
  }
}
```

### Common identifiers

| Field | Description |
|---|---|
| `app_id` | Application scope |
| `backend_ref` | Configured backend instance reference |
| `secret_name` | Logical name within an app |
| `store_path` | Provider-native secret identifier/path |

---

## Required core: KV secret lifecycle

These endpoints MUST be implemented by any OSL v1-compliant server.

### Put secret (create/update)
`POST /osl/v1/secrets/put`

### Get secret
`POST /osl/v1/secrets/get`

Supports optional `"version"` field. Defaults to latest if omitted.

### Delete secret (soft delete)
`POST /osl/v1/secrets/delete`

### Destroy secret (permanent)
`POST /osl/v1/secrets/destroy`

### List secrets
`POST /osl/v1/secrets/list`

### Describe secret
`POST /osl/v1/secrets/describe`

Returns provider-agnostic metadata plus provider-native hints (safe metadata only).

---

## Required core: Tainting

Tainting is a decodeRing-native concept that suspends access to a secret at the OSL server layer without deleting it from the backend. Useful for incident response and rotation workflows.

These endpoints MUST be implemented by any OSL v1-compliant server.

| Endpoint | Description |
|---|---|
| `POST /osl/v1/secrets/taint` | Suspend access to a secret |
| `POST /osl/v1/secrets/untaint` | Restore access to a secret |
| `POST /osl/v1/secrets/is-tainted` | Check taint status |

---

## Optional modules

Optional modules are capability-gated. Servers MUST return a structured `feature-not-supported` error when a client calls an optional endpoint against a backend that lacks the required capability.

### Secret versioning
Available when backend has `kv.versioning`.

- `POST /osl/v1/secrets/versions/list`
- `POST /osl/v1/secrets/versions/get`

### Dynamic credentials / leases
Available when backend has `lease.issue`.

- `POST /osl/v1/credentials/issue`
- `POST /osl/v1/credentials/renew`
- `POST /osl/v1/credentials/revoke`

### Rotation
Available when backend has `rotation.policy` and/or `rotation.rotate`.

- `POST /osl/v1/rotation-policies/put`
- `POST /osl/v1/secrets/rotate`

### Sync / materialization
Available when backend has `sync.manage`. Abstracts Kubernetes ESO and Doppler sync patterns.

- `POST /osl/v1/syncs/put`
- `POST /osl/v1/syncs/run`
- `POST /osl/v1/syncs/status/get`
- `POST /osl/v1/syncs/list`
- `POST /osl/v1/syncs/delete`

---

## Management API

- `GET /osl/v1/apps/list` - List registered applications
- `GET /osl/v1/backends/list` - List configured backends

---

## Migration from dcdr v0.1-draft

| Old endpoint | OSL v1 endpoint |
|---|---|
| `POST /api/dcdrCreateSecret` | `POST /osl/v1/secrets/put` |
| `POST /api/dcdrGet` | `POST /osl/v1/secrets/get` |
| `POST /api/dcdrDestroy` | `POST /osl/v1/secrets/destroy` |
| `POST /api/dcdrTaint` | `POST /osl/v1/secrets/taint` |
| `POST /api/dcdrUntaint` | `POST /osl/v1/secrets/untaint` |
| `POST /api/dcdrIsTainted` | `POST /osl/v1/secrets/is-tainted` |
| `POST /api/dcdrListSecrets` | `POST /osl/v1/secrets/list` |
| `GET /api/dcdrListApps` | `GET /osl/v1/apps/list` |
| `GET /api/dcdrListBackends` | `GET /osl/v1/backends/list` |

---

## Implementation notes

- Treat only the **required core** as universally supported.
- Gate everything else behind `capabilities/get`.
- Return structured `feature-not-supported` errors for unsupported optional module calls.
- Clients should never assume capabilities - always discover them.

---

## Contributing

OSL is an open standard. Contributions are welcome:

- 💬 [Open a discussion](https://github.com/decodeRing-core/osl/discussions) - propose changes, ask questions, share use cases
- 🐛 [File an issue](https://github.com/decodeRing-core/osl/issues) - report spec gaps, inconsistencies, or errors
- 🔌 Building an OSL-compatible server or backend adapter? Open a PR or discussion - we want to know.

---

## License

Licensed under the Apache License, Version 2.0.
