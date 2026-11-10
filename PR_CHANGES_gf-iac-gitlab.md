Add a private GitLab path for the Layer 2 identity connection, and publish its address and pod security group for Layer 2 to use.

## Summary

- Add a new private hostname, `gitlab-identity-connection.<env>.gfgce-ha.brainpool.ai`, on an internal load balancer for the identity-connection workload.
- This new listener accepts traffic only from the new identity-connection pod security group, and only for the token exchange, the current-user lookup, and token revocation. Every other path returns 404.
- Publish the URL and the pod security group id on the Gitlab platform contract. The contract version stays 1.
- This can be applied before the GitLab application is registered. Users still approve the connection on the existing public GitLab site. The dev browser callback is `https://portal.dev.gfgce-ha.brainpool.ai/api/v1/identity/gitlab/callback`.

## Changes

The permissions and extraction listeners cannot limit a request to a path, so this path is a separate internal application load balancer (ALB) in front of the same GitLab Workhorse. It has its own certificate and a private DNS name. There is no public Cloudflare record for that name.

The load balancer rewrites the request host to the public GitLab name so GitLab accepts it. Callers still check the private certificate name. The pod security group has no rules of its own. Layer 2 attaches it only to the identity-connection pods, next to the existing node security group.

The GitLab OAuth application is not created here. Register it by hand as a confidential, untrusted application with scope `read_user` only. For dev, the redirect URI is `https://portal.dev.gfgce-ha.brainpool.ai/api/v1/identity/gitlab/callback`. Prod gets its own public hostname, application, and client secret.

## Linked Issues

#256 [AUTH-CONN-2] Provision Internal GitLab OAuth Application and Private Connectivity