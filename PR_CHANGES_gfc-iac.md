Add a separate identity-connection deployment that holds the GitLab client secret and calls the search API when a user finishes connecting.

## Summary

- Add Deployment `gfgce-layer2-identity-connection`. It runs the existing API image as its own process, with its own role, secret, and pod security group.
- The separate identity-connection deployment is the only workload that can read the GitLab client secret and the only one that can open the private GitLab identity path.
- Since identity-connection is a separate deployment, it talks to the search API with an HTTP call on port 8001, `POST /internal/identity/gitlab/complete`. The search API stores the numeric GitLab user id. The identity pods do not use the database.
- The browser callback goes to the identity-connection deployment on a new load balancer. The existing BFF load balancer writes access logs, which would store the one-time authorization code. The new load balancer does not write access logs.
- The dev GitLab callback is `https://portal.dev.gfgce-ha.brainpool.ai/api/v1/identity/gitlab/callback`. This apply writes that URL and the registered Application ID onto the API ConfigMap and the identity ConfigMap. `start` can return the approval URL. The client secret is still written with `put-secret-value`.

## Changes

The identity-connection deployment only sends the numeric GitLab user id to the search API with HTTP, which writes the link for that Clerk user.

The HTTP call stays inside the cluster and only contains the one-time state and the numeric GitLab user id. The authorization code and the GitLab token remain only in the identity-connection pod and is not sent. The Service is a ClusterIP, and port 8001 is not on the BFF or the private API load balancer. The network policy admits only the identity-connection pods. Other workloads are not on that path, and the log records the path and status only.

Additionally, keeping the identity connection workload inside the search API would let the permission resolver and the identity connection share one pod, one IAM role, and one pod security group. This would let both open the same GitLab path and let the permission resolver read the user OAuth client secret, so a separate deployment is used.

The secret container is `/<project>/<env>/gitlab/identity-connection`. Terraform creates the container. An operator writes the secret value later. Only role `gfgce-layer2-<env>-identity-connection-irsa` can read it. The same image deploy rolls the API and the identity-connection Deployment together. There is no new image repository.

Dev does not set the BFF hostname yet, so the callback load balancer and the Cloudflare tunnel are not created by this apply. The GitLab application is registered with `https://portal.dev.gfgce-ha.brainpool.ai/api/v1/identity/gitlab/callback`, and dev tfvars hold that Application ID. The browser returns to `https://portal.dev.gfgce-ha.brainpool.ai`. Prod uses its own public hostname, secret, role, and GitLab application. The dev client secret is not copied.
