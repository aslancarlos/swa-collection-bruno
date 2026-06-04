# Security Policy

## No secrets in this repository

This collection is designed to contain **no secrets and no environment-specific data**:

- All tenant/endpoint values are **placeholders** (`YOUR_TENANT`, `YOUR_REGION`,
  `YOUR_EKS_OIDC_ID`, `YOUR_CLUSTER`, `YOUR_HOST`, `example.machineid`, `user@example.com`).
- Credentials (`TOKEN`, `IDENTITY_JWT`, `IDENTITY_PASSWORD`, `HOST_API_KEY`, `SVID`) are
  declared as **secret** variables in `environments/example.bru` with **no value**. Their
  values are entered locally in Bruno and are never committed.
- The [`.gitignore`](.gitignore) blocks common secret/runtime files (`*.env`, `*.key`,
  `*.pem`, `*-token*`, `*secret*.json`, …).

## Before you commit

Run the secret scan documented in [CONTRIBUTING.md](CONTRIBUTING.md#pre-commit-secret-check).
It must print nothing.

## Reporting a vulnerability or a leaked secret

If you discover a secret that was accidentally committed, or any other security issue:

1. **Do not** open a public issue with the sensitive value.
2. Contact the repository owner privately (GitHub profile / direct message).
3. If a real credential was exposed, **rotate it immediately** at the source
   (your SWA service / Identity provider), then purge it from git history.

## Purging a secret from git history

Editing a file does not remove a value already present in past commits. To fully remove it:

```bash
# Option A — rewrite history (e.g. with git filter-repo) and force-push
git filter-repo --replace-text <(echo 'THE_SECRET==>REDACTED')
git push --force --all

# Option B — for a small/private repo, delete and recreate from a clean single commit
```

After rewriting history, **rotate the exposed credential anyway** — assume it was captured.

## Scope

This is an independent, unofficial community collection of API requests. It is **not** an
official Palo Alto Networks product, is not endorsed by Palo Alto Networks, and ships no
executable services. "Secure Workload Access" and related product names are trademarks of
their respective owners and are used here for descriptive purposes only. Always consult the
latest official documentation.
