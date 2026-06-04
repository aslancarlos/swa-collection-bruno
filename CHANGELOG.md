# Changelog

All notable changes to this collection are documented here. The format is based on
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to
[Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0]

### Added
- Initial public release of the Bruno collection for **Secure Workload Access (SWA)**.
- Request folders `00 - Auth` through `06 - authn-jwt (SWA)` covering authentication,
  the trust hierarchy (Trust Domain → Server Group → Node Group → Server), discovery
  endpoints, SVID validation, and the `authn-jwt` flow.
- `environments/example.bru` environment template with placeholders only.
- Repository documentation: `README.md` with a step-by-step (00 → 06) walkthrough,
  `CONTRIBUTING.md`, `SECURITY.md`, `CODE_OF_CONDUCT.md`, GitHub issue/PR templates,
  `.editorconfig`, and an MIT `LICENSE`.

### Security
- All requests use placeholders and `{{variable}}` references — no tenant, endpoint, or
  secret data is included. Credentials are declared as **secret** variables without values.
