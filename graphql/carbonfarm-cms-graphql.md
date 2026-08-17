---
generated: '2026-08-17'
method: probed
source: https://cms.int.carbonfarm.app/server/specs/graphql
schema: carbonfarm-cms-schema.graphql
endpoint: https://cms.int.carbonfarm.app/graphql
---

# CarbonFarm CMS — GraphQL schema

CarbonFarm's Directus 10.10.7 headless CMS publishes its full GraphQL SDL anonymously at
`https://cms.int.carbonfarm.app/server/specs/graphql` (HTTP 200, 6,583 bytes). The schema was
saved verbatim to `carbonfarm-cms-schema.graphql`.

**Ownership.** The host is a subdomain of `carbonfarm.app`, the same domain that serves
CarbonFarm's client login (`web-login.carbonfarm.app`) and portal (`portal.carbonfarm.app`), and
`carbonfarm.tech` loads its news images from `cms.int.carbonfarm.app/assets/`. The instance
identifies itself as `project_name: "CarbonFarm CMS"` at `/server/info`.

## What the schema exposes

One content collection — `post` — plus Directus' built-in file type.

| Root | Field | Returns |
|---|---|---|
| Query | `post(filter, sort, limit, offset, page, search)` | `[post!]!` |
| Query | `post_by_id(id, version)` | `post` |
| Query | `post_aggregated(groupBy, filter, limit, offset, page, search, sort)` | `[post_aggregated!]!` |
| Query | `post_by_version(version, id)` | `version_post` |
| Subscription | `post_mutated(event: create\|update\|delete)` | `post_mutated` |
| Subscription | `directus_files_mutated(event: create\|update\|delete)` | `directus_files_mutated` |

`post` fields: `slug` (ID), `status`, `title`, `description`, `content`, `type`, `datetime`,
`is_pinned`, `featured_image` (→ `directus_files`), plus Directus audit fields
(`user_created`, `date_created`, `user_updated`, `date_updated`).

## Access

The **schema** is anonymous. The **data** is not:

```
GET /items/post?limit=1  -> 403 {"errors":[{"message":"You don't have permission to access this.",
                                            "extensions":{"code":"FORBIDDEN"}}]}
GET /users/me            -> 401 {"errors":[{"message":"Invalid user credentials.",
                                            "extensions":{"code":"INVALID_CREDENTIALS"}}]}
POST /graphql (no query) -> 400 {"errors":[{"message":"Invalid payload. Must provide query string.",
                                            "extensions":{"code":"INVALID_PAYLOAD"}}]}
```

No introspection or data query was executed against the endpoint beyond the anonymous
`/server/specs/graphql` document the instance publishes for that purpose.

## Realtime surface

The `Subscription` root is a genuine event surface (Directus WebSocket subscriptions with a
`create | update | delete` event enum). CarbonFarm publishes **no AsyncAPI** and documents **no
webhooks**, so no `AsyncAPI` or `Webhooks` pointer is emitted — the subscription root is recorded
here as evidence, not promoted to an event contract the company has not published.
