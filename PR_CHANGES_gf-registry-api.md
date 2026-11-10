Let the existing API image to start a GitLab connection and, in a separate process, finish it by calling the API.

## Summary

- The search API adds `POST /api/v1/identity/gitlab/start` to begin identity connection. A signed-in Clerk user gets the GitLab approval URL. The route returns 503 until the real callback URL and Application ID are set.
- A second process, used only by Deployment `gfgce-layer2-identity-connection`, handles `GET /api/v1/identity/gitlab/callback`. It exchanges the code, reads the numeric GitLab user id, and revokes the token.
- That process then calls the search API at `POST /internal/identity/gitlab/complete` on port 8001. The API stores the id for the Clerk user. The callback process does not write the database itself.
- Later API requests read that stored id. If the user has not connected, they stay on public access.
- The image command is unchanged. The API Deployment is what starts both the public API and the port 8001 route.

## Changes

The identity connection is a separate deployment because the identity-connection workload is the only place that should see the client secret, the authorization code, and the private GitLab path. Adding a new service to the search API would allow the permission resolver to access the private identity-connection path and read the one-time authorization code.

The search API keeps Clerk and the database. It starts the flow and stores the result. It does not exchange the code.

The two processes share one image. `gfgce-layer2-identity-connection` runs the callback. The API runs `start` on port 8000 and the complete call on port 8001. The public app does not serve the complete call. One published digest updates both deployments.

`start` does not call GitLab and does not read the client secret. The callback reads the secret from Secrets Manager on every request, allows only the token, current-user, and revoke calls, and drops the token after revoke. The browser is sent back with `gitlab_connection=connected` or `gitlab_connection=error`, and without the code or the GitLab id.
