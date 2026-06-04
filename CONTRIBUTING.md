# Contributing

Thanks for your interest in improving this Bruno collection! Contributions of all sizes
are welcome — fixes, new requests, clearer docs, or better examples.

## Ground rules

1. **Never commit secrets or environment-specific data.** This is the single most important
   rule. See [SECURITY.md](SECURITY.md). Use the placeholder conventions below for anything
   tenant-specific.
2. **Keep everything in English** — request names, `docs` blocks, comments, and docs files.
3. **Preserve the Bruno format.** Don't hand-edit structural keys in ways that break the app;
   prefer editing requests in Bruno itself when possible.

## Placeholder conventions

Replace any real value with a placeholder so the collection stays generic:

| Real thing | Placeholder |
|---|---|
| Tenant subdomain | `YOUR_TENANT` |
| Tenant / Identity id | `YOUR_TENANT_ID` |
| Cloud region | `YOUR_REGION` |
| EKS OIDC provider id | `YOUR_EKS_OIDC_ID` |
| Cluster id | `YOUR_CLUSTER` |
| Host id | `YOUR_HOST` |
| Container registry | `YOUR_REGISTRY` |
| Trust domain name | `example.machineid` |
| User / email | `user@example.com` |
| Example secret value | `example-secret-value` |

Credentials (`TOKEN`, `IDENTITY_JWT`, `IDENTITY_PASSWORD`, `HOST_API_KEY`, `SVID`) must remain
declared as **secret** variables with **no value** in the committed files.

## Workflow

1. Fork and create a feature branch: `git checkout -b feat/short-description`.
2. Make your change. If you add a request, give it a clear name and a `docs` block.
3. Run the pre-commit secret check (below) and confirm it is clean.
4. Commit with a descriptive message and open a pull request using the template.

## Pre-commit secret check

Before every commit, scan the working tree for leaked values:

```bash
grep -rIn -E \
  "eyJ[A-Za-z0-9_-]{20,}|-----BEGIN|AKIA[0-9A-Z]{16}|[a-z0-9-]+\.amazonaws\.com/id/[A-Z0-9]{20,}" \
  . --exclude-dir=.git
```

The check should print nothing. If it finds a real value, replace it with a placeholder and
re-run before committing.

## Style

- Folder numbering (`00`, `01`, …) reflects the intended run order — keep it consistent.
- Prefer environment variables (`{{VAR}}`) over hardcoded values in request URLs and bodies.
- Document **why**, not just **what**, in `docs` blocks and response-code notes.
