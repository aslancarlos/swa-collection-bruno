# Secure Workload Access (SWA) · Bruno Collection

A [Bruno](https://www.usebruno.com/) API collection for **Secure Workload Access (SWA)**.
It documents and exercises the full lifecycle end to end: authentication, building the SWA
trust hierarchy (**Trust Domain → Server Group → Node Group → Server**), the public
discovery endpoints, validating a workload **SVID**, and finally wiring **`authn-jwt`** so
SPIFFE workloads can authenticate and read secrets.

> [!WARNING]
> **Unofficial / community collection.** This repository is **not** an official
> **Palo Alto Networks** product and is not endorsed by Palo Alto Networks. APIs, endpoints,
> field names, and behavior **may differ** from the actual service and can change at any time.
> Always refer to the **latest official Secure Workload Access documentation** as the source
> of truth. Use at your own risk.

> [!IMPORTANT]
> **No secrets and no environment data are stored in this repository.** Every
> tenant/endpoint value is a **placeholder** (`YOUR_SWA_API_HOST`, `YOUR_IDENTITY_POD_HOST`,
> `YOUR_REGION`, `YOUR_EKS_OIDC_ID`, `YOUR_CLUSTER`, `YOUR_HOST`, `example.machineid`,
> `user@example.com`). Credentials (`TOKEN`, `IDENTITY_JWT`, `IDENTITY_PASSWORD`,
> `HOST_API_KEY`, `SVID`) are declared as **secret** variables and must be filled in
> locally in Bruno — they are never committed. See [SECURITY.md](SECURITY.md).

---

## Table of contents

- [What is Secure Workload Access?](#what-is-secure-workload-access)
- [Prerequisites](#prerequisites)
- [Quick start](#quick-start)
- [Authentication](#authentication)
- [The SWA hierarchy](#the-swa-hierarchy)
- [Step-by-step walkthrough (00 → 06)](#step-by-step-walkthrough-00--06)
- [Environment variables](#environment-variables)
- [Repository structure](#repository-structure)
- [Contributing & support](#contributing--support)
- [License](#license)

---

## What is Secure Workload Access?

**Secure Workload Access (SWA)** gives workloads a short-lived, cryptographically verifiable
**SPIFFE identity** (an **SVID**, in JWT or X.509 form). Instead of a long-lived API key, a
workload proves *what it is* (e.g. a Kubernetes pod running as a specific service account)
and receives an SVID it can exchange for an access token to read secrets.

This collection covers both sides:

1. **Control plane (SWA admin API)** — create and operate the trust hierarchy via
   `https://YOUR_SWA_API_HOST/api/swa/...`.
2. **Consumption (`authn-jwt`)** — register the SWA trust domain as a JWT authenticator so
   workload SVIDs become logins that can fetch secrets.

---

## Prerequisites

| Requirement | Notes |
|---|---|
| [Bruno](https://www.usebruno.com/) | Desktop app or CLI. Open this folder as a collection. |
| An SWA-enabled tenant | A Secure Workload Access service endpoint you can reach. |
| Admin role | SWA admin endpoints require an administrator role with SWA permissions. |
| Identity credentials | An Identity-provider user (for the OIDC login flow in folder `00`). |

---

## Quick start

1. **Open the collection** — Bruno → *Open Collection* → select this folder.
2. **Configure the environment** — select the **example** environment (top-right), then edit
   [`environments/example.bru`](environments/example.bru) and replace every `YOUR_*` /
   `example.*` placeholder with your real values.
3. **Fill the secrets** — in Bruno's *Variables* panel set the secret values you need
   (`IDENTITY_PASSWORD` or `IDENTITY_JWT`/`TOKEN`). These stay local.
4. **Get a token** — run the requests in folder **`00 - Auth`** (see below).
5. **Run the folders in order** — `01` → `02` → `03` → `04`, then optionally `05` and `06`.

---

## Authentication

Every admin call sends a collection-level header (defined in `collection.bru`):

```
Authorization: Token token="{{TOKEN}}"
Accept: application/x.secretsmgr.v2+json
Content-Type: application/json
```

`TOKEN` is the **access token** (base64 of the token JSON), **not** a Bearer token. SWA admin
endpoints require an administrator role with SWA permissions.

Two ways to obtain it (folder `00`):

- **Identity login flow** (`00 - Auth/Identity Login/`): fill `IDENTITY_PASSWORD` (and
  `MFA_CODE` when prompted) and run `1. StartAuthentication` →
  `2. AdvanceAuthentication (password)` → (`3. StartOOB` for Email/SMS) →
  `4. AdvanceAuthentication (MFA answer)`. The chained scripts persist `IDENTITY_JWT`.
  Then run **Exchange Identity JWT for Access Token** → it stores `TOKEN`.
- **Host login** (`00 - Auth/Get Token (host)`): authenticates a host with `HOST_API_KEY`.
  ⚠️ A host token is admin over `data/*` but is **not** an SWA admin, so SWA admin calls may
  return `403`. Useful for general API calls only.

> The Identity JWT expires in ~15 min; the access token in ~1 h.

---

## The SWA hierarchy

The admin objects are created top-down. Each level is the parent of the next:

```
Trust Domain            (root of SPIFFE trust: signing keys, TTLs, discovery)
└── Server Group        (a set of SWA servers + node attestation policy, e.g. k8s PSAT)
    └── Node Group      (workload selectors inside the server group; issues SVIDs)
        └── Server       (a registered SWA server component, with JWT auth)
            └── Workloads (pods/SAs that receive SVIDs and authenticate to read secrets)
```

Folders `01 → 04` follow exactly this creation order. Re-running a **Create/Register**
request for an object that already exists returns **`409 Conflict`** — use **List/Get** to
inspect, or change the name in the environment to create a new one.

---

## Step-by-step walkthrough (00 → 06)

Each folder is numbered to convey the intended run order. Every request also has an
inline `docs` block (visible in Bruno) describing its parameters and response codes.

### `00 - Auth` — get a usable token
Establishes the `TOKEN` used by every other request.
- **Exchange Identity JWT for Access Token** — trades an Identity OIDC JWT (`IDENTITY_JWT`)
  for an access token via `authn-oidc`; stores it in `TOKEN`.
- **Get Token (host)** — alternative host-based login with `HOST_API_KEY`.
- **Identity Login/** — the raw Identity challenge flow (StartAuthentication → password →
  optional MFA) that produces `IDENTITY_JWT` without leaving Bruno.

### `01 - Trust Domains` — the root of trust
Creates and inspects the SPIFFE trust domain (signing algorithm, key type/TTL, token TTL).
- **Create / List / Get Trust Domain** — lifecycle of the trust domain object.
- **Get CA Bundles** — the X.509 bundle used as `bootstrap.bundleSourceUrl` by SWA agents.
- **Get OIDC Discovery / Get JWKS** — the public `.well-known/*` endpoints (no token) that
  relying parties (and the `authn-jwt` authenticator) use to validate SVIDs.

### `02 - Server Groups` — where servers live + node attestation
A server group bundles SWA servers and defines **how nodes are attested** (e.g. Kubernetes
PSAT: cluster id, audience, allowed service accounts).
- **Create / List / Get** and **Update (PATCH)** the server group.

### `03 - Node Groups` — which workloads get an identity
A node group sits inside a server group and defines the **workload selectors** (the
registration policy, e.g. `k8s.ns == '...' && k8s.sa == '...'`) that decide which workloads
receive an SVID and under which SPIFFE ID.
- **Create / List / Get** and **Update (PATCH)** the node group.

### `04 - Servers (Components)` — register an SWA server
Registers an SWA **server component** under a server group and configures its **JWT
authenticator** (issuer, JWKS, audience). The **Register** response returns the
`login_url` (base64) used by the server/agent Helm charts — use it **as-is**, do not decode.
- **Register / List / Get** the server.

### `05 - Workload SVID (test)` — prove it works
Validates that a workload can fetch an SVID and that the SVID verifies against the trust
domain's published JWKS.
- **Get Trust Bundle (validate SVID JWKS)** — fetches the JWKS and walks through decoding a
  sample JWT-SVID's claims (`sub` = SPIFFE ID, `aud`, TTL).

### `06 - authn-jwt (SWA)` — let workloads read secrets
Wires the SWA trust domain in as a JWT authenticator so SVIDs become logins. Run the
numbered requests in order:
1. **Load Authenticator Policy** — creates the `authn-jwt/secureWorkloadAccess` webservice.
2–6. **Set vars** — `jwks-uri`, `token-app-property=sub`, `identity-path`, `issuer`,
   `audience`.
7. **Enable Authenticator** — `PATCH enabled=true`.
8. **Load Workloads Group** — creates the `workloads` group + hosts whose IDs are SPIFFE IDs
   (annotated `authn-jwt/secureWorkloadAccess/sub`).
9. **Grant workloads to apps** — grants the `workloads` group access to the consuming role.
10–11. **Load Secrets and Permissions** + **Set Secret** (`{{SECRET_ID}}`).
12. **Authenticate (SVID → access token)** — exchanges a JWT-SVID (`SVID`) at the
    `.../authn-jwt/secureWorkloadAccess/.../authenticate` endpoint and stores the `TOKEN`.
13. **Get Secret** — reads the secret as the now-authenticated workload.

---

## Environment variables

Defined in [`environments/example.bru`](environments/example.bru). Replace placeholders with
your real values; fill secrets locally.

| Variable | Example / placeholder |
|---|---|
| `SWA_API_BASE` | `https://YOUR_SWA_API_HOST` |
| `OIDC_SERVICE_ID` / `CONJUR_ACCOUNT` | service-id / account values for your tenant |
| `TRUST_DOMAIN_NAME` | `example.machineid` |
| `SERVER_GROUP_NAME` / `NODE_GROUP_NAME` | `k8s-prod` / `k8s-prod-ng` |
| `SERVER_NAME` | `swa-server-a` |
| `K8S_CLUSTER_ID` | `YOUR_CLUSTER` |
| `EKS_ISSUER` | `https://oidc.eks.YOUR_REGION.amazonaws.com/id/YOUR_EKS_OIDC_ID` |
| `IDENTITY_INIT_URL` / `IDENTITY_POD_URL` | `https://YOUR_IDENTITY_INIT_HOST` / `https://YOUR_IDENTITY_POD_HOST` |
| `IDENTITY_USER` | `user@example.com` |
| `AUTHN_JWT_ID` | `secureWorkloadAccess` |
| `SECRET_ID` | `data/swa/secrets/myapp/api-key` |
| `TOKEN`, `IDENTITY_JWT`, `IDENTITY_PASSWORD`, `HOST_API_KEY`, `SVID` | **secret** — fill in Bruno |

> `OIDC_SERVICE_ID`, `CONJUR_ACCOUNT`, the `application/x.secretsmgr.v2+json` media type, and
> the `authn-*` URL paths are **API contract values** — keep them as the service expects.

---

## Repository structure

```
.
├── 00 - Auth/                     # Get an access token (Identity OIDC + host)
│   └── Identity Login/            # Raw Identity challenge flow
├── 01 - Trust Domains/            # Trust domain lifecycle + discovery (.well-known)
├── 02 - Server Groups/            # Server groups + node attestation
├── 03 - Node Groups/              # Workload selectors / registration policy
├── 04 - Servers (Components)/     # Register an SWA server (JWT auth)
├── 05 - Workload SVID (test)/     # Validate an SVID against the JWKS
├── 06 - authn-jwt (SWA)/          # Wire SWA in via authn-jwt (1 → 13)
├── environments/example.bru       # Environment template (placeholders only)
├── collection.bru                 # Collection-level headers + docs
├── bruno.json                     # Bruno collection manifest
├── README.md
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
├── SECURITY.md
├── CHANGELOG.md
└── LICENSE
```

---

## Contributing & support

Contributions are welcome — see [CONTRIBUTING.md](CONTRIBUTING.md) and the
[Code of Conduct](CODE_OF_CONDUCT.md). To report a security concern (including an
accidentally committed secret), read [SECURITY.md](SECURITY.md).

## License

Released under the [MIT License](LICENSE).

---

*Disclaimer: "Secure Workload Access" and related product names are referenced for
descriptive purposes only. This is an independent, unofficial collection and is not
affiliated with, endorsed by, or supported by Palo Alto Networks. Always consult the latest
official documentation.*
