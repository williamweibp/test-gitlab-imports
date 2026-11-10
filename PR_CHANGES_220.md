Remember each connected user's GitLab project access for 60 seconds inside the API process.

## Summary

- The API keeps a user's GitLab project scopes in memory for 60 seconds. A later search in that window does not call GitLab.
- The memory is capped at 4,096 users. When it is full, the least recently used user is dropped. A hit does not extend the 60 seconds.
- A missing or expired entry is resolved again through the existing GitLab permission resolver. Several requests for the same user at the same moment share one GitLab call.
- If GitLab cannot be reached and there is no unexpired entry, the user sees public results only. An expired list is not kept alive because GitLab is down.
- A user with no GitLab projects, or a GitLab account that cannot be used, is remembered as an empty list. Search then returns public documents only.
- The cache key is built by the server from the environment, the stored GitLab user id, and a schema version in code. The browser cannot choose the key.
- Linking a different GitLab user, or a banned or locked Clerk account, drops that process's entry immediately.
- Hit, miss, expiry, eviction, refresh failure, invalidation, and fail-closed events record a count only. They do not include tokens, user ids, or the project list.
- No Redis, ElastiCache, DynamoDB, or session stickiness is added. Two API processes can disagree for at most the configured lifetime, which defaults to 60 seconds.

## Changes

Search for a connected user was asking GitLab, on every request, which projects that user can read. That walk is slow, and most of the answers can be reused for a short time. An answer that was removed, or that GitLab could not confirm, must not stay in force and must not turn into an unfiltered search.

The API now keeps those project scopes in the search process. Each entry is the list of `gitlab:project:<id>` scopes for one GitLab user, plus the timestamps that say when the entry expires. It is not the files inside those projects. Tokens, cookies, raw GitLab responses, and database passwords are not stored.

The key is not taken from the browser. The process copies its environment when it starts, reads the numeric GitLab user id from the identity link it already stored, and uses a schema version that lives in code. A valid entry is returned as-is. A missing or expired entry is resolved again. If several searches for that user arrive while the refresh is running, they wait for the same GitLab call instead of starting more. If the call fails, or a waiter gives up, private GitLab access is denied for that request and nothing is saved as a success.

An empty project list is a real answer. The user still receives public access, and a private document tagged only with a GitLab project does not match.

The API runs more than one process. Each process has its own memory. After a relink or a Clerk ban or lock, this process drops its entry immediately. Another process can still serve the previous scopes until they expire. That lag is at most `PERMISSION_CACHE_TTL_SECONDS`, which defaults to 60 seconds. Sticky sessions are not used to paper over the window. A shared cache is a later decision, only if replica count, GitLab load, memory, or a stricter revocation need makes 60 seconds not enough.

`PERMISSION_CACHE_TTL_SECONDS` and `PERMISSION_CACHE_MAX_ENTRIES` default to 60 and 4096. An unusable value stops the process at startup. The deployed ConfigMap does not need new keys for the pilot, and no cache service is created.

## Linked Issues

#220 In-memory GitLab permission cache

#114 Layer 2 Identity, Authentication & Authorization

#61 GitLab permission resolver, already in `gf-registry-api`. This change caches the scopes that resolver returns. It does not replace the resolver.
