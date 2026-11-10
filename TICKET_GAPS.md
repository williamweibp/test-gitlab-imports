# Ticket gaps — Layer 2 GitLab identity connection

The five gaps in `IMPLEMENTATION_PLAN.md` are accepted. They are recorded here with the extra gaps this implementation still has. The dev public callback is `https://portal.dev.gfgce-ha.brainpool.ai/api/v1/identity/gitlab/callback`. Dev sets `gitlab_identity_connection_application_id` to the registered Application ID. The client secret and the Cloudflare tunnel are still outstanding.

## Accepted gaps from the plan

### 1. The OAuth application is registered by hand

The ticket asks for a confidential, untrusted GitLab application with the exact callback and `read_user`. This implementation documents that registration and does not create the application from Terraform.

There is no GitLab provider in these repos. A `gitlab_application` resource would return the client secret into Terraform state and would delete the application, and every user consent, on destroy. `oauth.tf` leaves Google and Azure outside Terraform for the same lifecycle reason. 1Password holds the operator copy. Secrets Manager holds the runtime copy.

### 2. Browser access stays on the existing GitLab allowlist

Users open `/oauth/authorize` on the public GitLab hostname. That site is already limited to `allowed_ip_cidrs`. This change does not add networks to that list and does not publish the identity listener on the internet. People who cannot already open GitLab cannot open the approval page.

### 3. The dev public callback is set; the tunnel is not

The dev public origin is `https://portal.dev.gfgce-ha.brainpool.ai`. GitLab must redirect the browser to:

`https://portal.dev.gfgce-ha.brainpool.ai/api/v1/identity/gitlab/callback`

`stacks/application/vars/dev.tfvars` sets `gitlab_identity_connection_redirect_uri` to that URL and `gitlab_identity_portal_url` to `https://portal.dev.gfgce-ha.brainpool.ai`. Stage and prod do not use this hostname. When `gitlab_identity_connection_redirect_uri` is null, Terraform still stores `https://public-layer2-origin.undefined/api/v1/identity/gitlab/callback`.

`gitlab_identity_connection_application_id` is the registered Application ID. `POST /api/v1/identity/gitlab/start` can build the approval URL after this apply. The identity process still refuses to exchange a code against an `undefined` host, and it refuses the exchange until the client secret is in Secrets Manager.

These names are private and are not the redirect URI:

- `layer2-api.dev.gfgce-ha.brainpool.ai`
- `layer2-bff.dev.gfgce-ha.brainpool.ai`
- `gitlab-identity-connection.dev.gfgce-ha.brainpool.ai`

The callback is not on the BFF load balancer. That load balancer stores every request address, including the query string, in S3. When the BFF hostname is set, Terraform creates a second internal load balancer for `/api/v1/identity/gitlab/callback` only, with access logs disabled. The Cloudflare tunnel sends that path there and sends every other `/api` path to the BFF. The private callback hostname is `layer2-identity-callback.<same suffix as the BFF name>`. It is not the GitLab redirect URI. The current dev tfvars do not set `bff_private_hostname`, so neither load balancer exists yet. GitLab can be registered with the public URL now. Browsers cannot complete the callback until the tunnel and that load balancer exist.

Kubelet probes for the identity pods come from the infra subnets. `gfgce-layer2-identity-connection-allow-ingress` allows those addresses on port 8000 on every apply. Private subnet addresses are added when an internal load balancer exists. The pods can become ready before the tunnel is enabled.

### 4. No registry UI control

`gf-registry-ui` is unchanged. It has no GitLab OAuth client, so rotating the secret does not rebuild the frontend. The live test starts from the authorize URL returned by `POST /api/v1/identity/gitlab/start`. A later Connect control only needs to call that route and send the browser to the returned URL.

### 5. Rotation is a documented procedure, read every time

The ticket asks for rotation and emergency revocation without rebuilding the frontend. GitLab application secrets are not on the 90-day bot-token schedule, and this workload must not call the GitLab admin API to rotate them. It is limited to `read_user`.

The identity process calls Secrets Manager on every callback and does not cache the client secret, the authorization code, or the user token. A restart is unnecessary. The READMEs still say to restart `deployment/gfgce-layer2-identity-connection` if a future change adds a cache. Do not restart `deployment/gfgce-layer2-api` for this secret.

Regenerating the secret in the GitLab admin UI revokes the previous secret immediately. Replace `client_secret` on the 1Password item, leave the URLs, `redirect_uri`, and `scope` unchanged, and write the item with `put-secret-value`. In-flight authorization codes fail at token exchange. No private GitLab scope is added.

## Additional gaps

### The live acceptance test has not been run

A live development test against the deployed GitLab CE version is an acceptance criterion: authorization, token exchange, current-user lookup, revocation, and the numeric id on the next API request. That test waits on the client secret in Secrets Manager, the published image, and the Cloudflare tunnel for `https://portal.dev.gfgce-ha.brainpool.ai`. The Application ID is already in dev tfvars. Unit tests cover the client and the store. They do not call GitLab.

### Permission-resolver wiring is intentionally unconnected

Issue #75 resolves permissions with a service credential on `PERMISSION_RESOLVER_GITLAB_API_URL`. This ticket stores a user-approved numeric GitLab id. Nothing in the permission resolver reads that id yet. The connection point for the developer implementing that service is:

- Table `gitlab_identity_links`, primary key `clerk_user_id`, column `gitlab_user_id` (decimal string, no leading zeros).
- `AuthenticatedUser.gitlab_user_id`, filled by `_resolve_gitlab_user_id` in `gf-registry-api/services/api/middleware/auth.py` on each verified Clerk request.
- A missing row or a lookup failure leaves `gitlab_user_id` empty and leaves scopes as `layer2:public`.

That developer should not use this OAuth client secret, role `gfgce-layer2-<env>-identity-connection-irsa`, pod security group `identity_connection_pod_security_group_id`, or the identity origin `https://gitlab-identity-connection.<env-subdomain>`. Those exist only for the authorization-code flow. The permissions listener and the extraction listener stay on their own pod groups.

Comments marking that handoff:

- `gf-registry-api/services/api/db/models.py` — `GitlabIdentityLink` and `GitlabIdentityOAuthState`
- `gf-registry-api/services/api/config.py` — identity settings
- `gf-registry-api/services/api/middleware/auth.py` — `_resolve_gitlab_user_id` and the loop bind in `get_current_user`
- `gf-registry-api/services/api/httpclient/gitlab_identity.py` — module docstring; imported only by the callback process
- `gf-registry-api/services/api/identity_app.py` — `_post_complete`
- `gf-registry-api/services/api/app.py` — `start` and `complete`
- `gfc-iac/stacks/application/identity-connection-iam.tf` — secret and role header
- `gfc-iac/stacks/application/identity-connection-workload.tf` — `GITLAB_IDENTITY_CONNECTION_FORBIDDEN_ORIGINS` and `GITLAB_IDENTITY_COMPLETE_URL`
- `gfc-iac/stacks/application/bff-ingress.tf` — callback path comment
- `gfc-iac/k8s/api/deployment.yaml` and `k8s/api/network-policy.yaml` — port 8001
- `gfc-iac/k8s/identity-connection/deployment.yaml` and `network-policy.yaml` — IRSA token and the complete call

The in-cluster complete URL is `http://gfgce-layer2-api:8001/internal/identity/gitlab/complete`. During the live test, confirm an identity pod can open that port. The NetworkPolicy selects API pods by label. If the cluster's CNI does not associate that pod selector with the ClusterIP, the complete call is the place that will fail, and PostgreSQL or port 8000 should stay closed while it is fixed.

## Merge order, apply order, and manual steps

Do this for dev first. Repeat the same sequence for prod with the prod names below. Prod has no `dev.` label in DNS, uses secret `/gfgce-layer2/prod/gitlab/identity-connection`, role `gfgce-layer2-prod-identity-connection-irsa`, a separate 1Password item, and a separate GitLab application. The dev client secret is not copied. Prod secret recovery is 30 days and the prod identity ALB has deletion protection.

### 1. Merge and apply `gf-iac-gitlab` (dev)

Merge the GitLab pull request, then apply `terraform/gfgce-ha` for dev. That creates:

- Internal ALB `gfgce-ha-dev-alb-idc` and target group `gfgce-ha-dev-tg-idc`
- Pod security group for the identity-connection pods
- ACM certificate and its Cloudflare validation CNAME
- Private Route 53 CNAME `gitlab-identity-connection.dev.gfgce-ha.brainpool.ai`
- Platform parameter `/gfgce-ha/dev/platform` keys `gitlab_identity_connection_url` and `identity_connection_pod_security_group_id`

`contract_version` stays `1`. Do not add a public Cloudflare hostname for `gitlab-identity-connection`.

### 2. Register the GitLab application with the dev callback

The dev callback is `https://portal.dev.gfgce-ha.brainpool.ai/api/v1/identity/gitlab/callback`. That hostname is the public portal from the Pages project. `/api/*` on the same host is reserved for the same-origin Worker, so the callback is not a separate API domain. Applying the infrastructure is not blocked on that URL. Edit the registered GitLab application so its redirect URI is that exact string. Do not invent a callback from `layer2-api` or `layer2-bff`. GitLab stores the redirect URI exactly, and a hostname change means users approve again.

### 3. Merge and apply `gfc-iac` `stacks/application` (dev)

Apply this after step 1 so the platform parameter already contains the identity keys. The SecurityGroupPolicy is created only when that security group id is present. If this stack is applied first, re-apply it after the GitLab apply.

This apply creates the empty secret container, the IRSA role, the ServiceAccount, both ConfigMaps, the identity Service, the API port 8001, the probe NetworkPolicy, and, once the contract key exists, the identity `SecurityGroupPolicy`. The secret has no version yet. The role can read the identity secret and cannot write CloudWatch logs. With current dev tfvars the BFF hostname is unset, so the unlogged callback load balancer is not created yet. The probe rule is created, so the identity pods can pass health checks after the image deploy.

Confirm:

- `gfgce-layer2-dev-identity-connection-irsa` is allowed `GetSecretValue` and `DescribeSecret` on `/gfgce-layer2/dev/gitlab/identity-connection`. A read returns not found until step 5 creates the version
- The API role, the extraction-worker role, and the admin role cannot
- API pods are not in the identity pod security group, so they cannot open the identity listener
- Identity pods are not in the permissions or extraction pod security groups

### 4. Register the application

In GitLab: **Admin → Applications → New application**.

| Setting | Value |
|---|---|
| Name | `Layer 2 identity connection` |
| Redirect URI | `https://portal.dev.gfgce-ha.brainpool.ai/api/v1/identity/gitlab/callback` |
| Confidential | yes |
| Trusted | unchecked. Trusted skips the approval screen. |
| Scopes | `read_user` only |

Do not select `api`, `read_api`, `read_repository`, `write_repository`, `admin`, `sudo`, or any impersonation scope.

Copy the Application ID and Secret before leaving the page. GitLab does not show the secret again.

Dev `stacks/application/vars/dev.tfvars` already sets the redirect URI and `gitlab_identity_connection_application_id` to the registered Application ID. Do not put the client secret in tfvars.

That apply writes the Application ID onto the API ConfigMap and the identity ConfigMap. The redirect is `https://portal.dev.gfgce-ha.brainpool.ai/api/v1/identity/gitlab/callback`. It does not write a secret version. Roll `deployment/gfgce-layer2-api` and `deployment/gfgce-layer2-identity-connection` after those Deployments exist so both processes load the new env.

### 5. Add the 1Password item and copy it to Secrets Manager

Create a vault item named `gfgce-layer2/dev/gitlab/identity-connection`. Limit it to the operators who administer this connection. Do not put this item in the 1Password Environment that `terraform.yml` syncs to `/gfgce-layer2/dev/deployment`.

The item fields, and the Secrets Manager `SecretString`, are this JSON object:

```json
{
  "client_secret": "<secret shown once on the GitLab page>",
  "authorize_url": "https://gitlab.dev.gfgce-ha.brainpool.ai/oauth/authorize",
  "token_url": "https://gitlab-identity-connection.dev.gfgce-ha.brainpool.ai/oauth/token",
  "user_url": "https://gitlab-identity-connection.dev.gfgce-ha.brainpool.ai/api/v4/user",
  "revoke_url": "https://gitlab-identity-connection.dev.gfgce-ha.brainpool.ai/oauth/revoke",
  "redirect_uri": "https://portal.dev.gfgce-ha.brainpool.ai/api/v1/identity/gitlab/callback",
  "scope": "read_user"
}
```

The Application ID is the Terraform variable from step 4, on both ConfigMaps. `redirect_uri` matches `gitlab_identity_connection_redirect_uri` character for character. `scope` is the string `read_user`.

Wait until step 3 has created the secret container. On an administrator workstation, signed in to 1Password and AWS, write the JSON without printing it and without `TF_VAR_*`. This write creates the first version:

```shell
aws secretsmanager put-secret-value \
  --secret-id /gfgce-layer2/dev/gitlab/identity-connection \
  --secret-string "$(op read 'op://<operator-vault>/<item-id>/credential')"
```

Store the JSON above as the item's concealed `credential` field so `op read` returns that object and nothing else. The item title contains slashes, so the `op://` reference should use the item id rather than the title. Confirm a new version with `aws secretsmanager describe-secret`. Do not run `get-secret-value` in CI logs. Confirm the identity role can read it and the API, extraction, and admin roles cannot. `terraform state show aws_secretsmanager_secret.gitlab_identity_connection` shows the ARN and name. The secret value is not a Terraform attribute.

Dev URLs, for reference:

| Use | URL |
|---|---|
| Browser authorize | `https://gitlab.dev.gfgce-ha.brainpool.ai/oauth/authorize` |
| Private origin | `https://gitlab-identity-connection.dev.gfgce-ha.brainpool.ai` |
| Token | `https://gitlab-identity-connection.dev.gfgce-ha.brainpool.ai/oauth/token` |
| Current user | `https://gitlab-identity-connection.dev.gfgce-ha.brainpool.ai/api/v4/user` |
| Revoke | `https://gitlab-identity-connection.dev.gfgce-ha.brainpool.ai/oauth/revoke` |
| Callback path | `/api/v1/identity/gitlab/callback` |
| Dev public origin | `https://portal.dev.gfgce-ha.brainpool.ai` |
| Dev GitLab redirect | `https://portal.dev.gfgce-ha.brainpool.ai/api/v1/identity/gitlab/callback` |
| Placeholder redirect when the variable is null | `https://public-layer2-origin.undefined/api/v1/identity/gitlab/callback` |
| Secrets Manager | `/gfgce-layer2/dev/gitlab/identity-connection` |
| 1Password item | `gfgce-layer2/dev/gitlab/identity-connection` |
| Complete call inside the cluster | `http://gfgce-layer2-api:8001/internal/identity/gitlab/complete` |

### 6. Merge `gf-registry-api` and deploy the image

Merge the API pull request, publish `gfgce-layer2/api`, and run **Layer 2 CD - Deploy ECR Images to EKS** with `deploy_api`. One digest rolls `deployment/gfgce-layer2-api` and `deployment/gfgce-layer2-identity-connection`. There is no separate image and no Terraform apply in this repo.

### 7. Allow the public origin on the portal

The search API allowlist is `api_clerk_authorized_parties` in `stacks/application/vars/dev.tfvars`. Step 3 writes it to `CLERK_AUTHORIZED_PARTIES` on ConfigMap `gfgce-layer2-api-config`. Dev already includes `https://portal.dev.gfgce-ha.brainpool.ai`. The API pods load it when step 6 rolls `deployment/gfgce-layer2-api`.

The portal does not read that ConfigMap. `gf-registry-ui` verifies the token `azp` from Kubernetes secret `registry-portal-env`, key `CLERK_AUTHORIZED_PARTIES`. A sign-in at `https://portal.dev.gfgce-ha.brainpool.ai` is rejected until that key includes the same origin. The portal deploy workflow refuses to roll while the key is empty. The secret is not in Terraform, and this value is not baked into the portal image.

On an administrator workstation, update only that key. Keep every other HTTPS origin already in the comma-separated value. Do not print `CLERK_SECRET_KEY` or `CLERK_JWT_KEY`. The namespace and deployment name are `K8S_NAMESPACE` and `K8S_DEPLOYMENT` from the portal deploy workflow.

```shell
kubectl patch secret registry-portal-env -n <portal-namespace> --type merge \
  -p '{"stringData":{"CLERK_AUTHORIZED_PARTIES":"https://portal.dev.gfgce-ha.brainpool.ai"}}'
kubectl rollout restart deployment/<portal-deployment> -n <portal-namespace>
kubectl rollout status deployment/<portal-deployment> -n <portal-namespace> --timeout=300s
```

A merge patch replaces the key. If the current value has other origins, set the new value to those origins plus `https://portal.dev.gfgce-ha.brainpool.ai`, separated by commas. Restart the portal after the patch. `secretKeyRef` is read when the pod starts.

### 8. Live dev test, after the callback, the application, the secret, the image, and the portal allowlist exist

Use one browser that is already on `allowed_ip_cidrs`.

1. Sign in to Layer 2 and call `POST /api/v1/identity/gitlab/start` with the Clerk session. Open the returned `https://gitlab.dev.gfgce-ha.brainpool.ai/oauth/authorize?...&scope=read_user` URL.
2. Confirm GitLab shows the approval screen and names `read_user`. A trusted application would skip that screen.
3. Approve. The browser should land on the portal with `gitlab_connection=connected` and without the code, the token, or the GitLab id in the URL.
4. The next authenticated API request should carry numeric `gitlab_user_id` from `GET /api/v4/user`.
5. Logs should contain the path and status only. Reusing the authorization code should fail. The revoked token should get 401 from GitLab.
6. From an identity pod, `GET /api/v4/projects` on the identity host should be the ALB 404. The three approved paths should reach Workhorse. An API pod should be unable to open the identity host.
