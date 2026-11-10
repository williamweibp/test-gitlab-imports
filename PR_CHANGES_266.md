Give the Layer 2 API the Clerk settings the completed trusted principal already reads.

## Summary

- When Clerk is enabled, ConfigMap `gfgce-layer2-api-config` receives `CLERK_ISSUER_URL`, `CLERK_AUTHORIZED_PARTIES`, and `CLERK_ORGANIZATION_ID`. When the issuer is unset, those keys are omitted.
- `CLERK_SECRET_KEY` is written only to Kubernetes secret `gfgce-layer2-api-clerk`, from the existing 1Password variable `clerk_secret_key`. It is not in the ConfigMap and not in Terraform outputs.
- Every issuer and authorized party must be a strict HTTPS origin. `http://localhost` and `http://127.0.0.1` are rejected. Dev sets `api_clerk_authorized_parties` to `https://portal.dev.gfgce-ha.brainpool.ai`. `http://localhost` is removed from that allowlist.
- If the issuer is set, plan fails without an organization id or without `clerk_secret_key`. If Admin and the API both set an issuer or an organization id, the values must match.
- [#51](https://github.com/gates-foundation/gfc-iac/issues/51) is already complete in `gf-registry-api`. This change only supplies the four environment variables that principal reads. It does not close #51.

## Changes

The Layer 2 API was the workload without Terraform-managed Clerk settings. Registry Admin already had an issuer, an organization id, and the shared backend key. The completed principal from #51 verifies a session from `CLERK_ISSUER_URL`, checks `CLERK_AUTHORIZED_PARTIES`, restricts the caller to `CLERK_ORGANIZATION_ID`, and uses `CLERK_SECRET_KEY` for account-status checks. Those four values are the connection between this stack and that principal. The contract is `gfc-iac/docs/auth/clerk-principal.md`.

The issuer, the allowlist, and the organization id are public verifier settings, so they belong in the ConfigMap. The backend key can call Clerk account APIs, so it stays in its own Kubernetes secret and is mounted with `secretKeyRef`. The deployment reference is optional so a pod can start before the secret exists; a missing key must fail closed in the API. Rotating the 1Password value updates the secret and does not restart running pods. Terraform state holds the key because Terraform manages the secret, the same way it already does for Registry Admin. The key is not committed and is not an output.

#85 is complete: the portal verifies the bearer token and forwards it to this API, so both sides must use the same `azp` origins. Dev sets `api_clerk_authorized_parties` to `https://portal.dev.gfgce-ha.brainpool.ai`, the public portal origin used by the GitLab identity callback. `http://localhost` and `http://localhost:3000` are removed from that allowlist. The portal reads `CLERK_AUTHORIZED_PARTIES` from the `registry-portal-env` secret and must include the same origin. An empty allowlist remains valid and fail-closed: the API rejects every token until an approved HTTPS origin is configured. Plain HTTP, including loopback, is rejected because `http://localhost:3000` is not a unique application origin, so another local site on that port against the same Clerk instance could mint a token the API would accept. Dev enables the issuer, the organization id, and this portal origin.

When both Admin and the API are configured, their issuer and organization id must be the same. A plan that pointed the API at a different Clerk instance or organization would verify a different population than Registry Admin.

## Linked Issues

#266 Layer 2 API Clerk verification configuration

#51 Trusted application principal in `gf-registry-api` (complete). This change supplies its runtime configuration and does not close #51.
