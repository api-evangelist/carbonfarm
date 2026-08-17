---
name: Read CarbonFarm news posts from the CMS
description: >-
  Retrieve CarbonFarm's published news posts and their featured images from the Directus CMS
  content API, using filtering, sparse fieldsets and pagination correctly, and handling the
  Directus error envelope. Requires a CarbonFarm-issued static token — the collection returns
  403 anonymously.
api: openapi/carbonfarm-cms-openapi.json
base_url: https://cms.int.carbonfarm.app
operations:
- readItemsPost
- readSingleItemsPost
- getAsset
- ping
generated: '2026-08-17'
method: generated
source: >-
  Grounded in openapi/carbonfarm-cms-openapi.json (every operationId below was verified present in
  the harvested spec) plus conventions/carbonfarm-conventions.yml and
  errors/carbonfarm-problem-types.yml.
---

# Read CarbonFarm news posts

## Before you start

**You need a token.** The public Directus role has no read permission on the `post` collection:

```
GET /items/post?limit=1     -> 403 {"errors":[{"message":"You don't have permission to access this.",
                                               "extensions":{"code":"FORBIDDEN"}}]}
```

CarbonFarm publishes no developer portal, no sign-up and no documentation, so a token can only be
obtained by asking the company (https://carbonfarm.tech/contact). Do not attempt to work around
the 403.

Send the token in the header, never the query string:

```
Authorization: Bearer <token>
```

The spec also defines `KeyAuth` (`?access_token=…`). Do not use it — it leaks the credential into
access logs and Referer headers.

## Steps

**1. Confirm the instance is up — `ping`**

```
GET https://cms.int.carbonfarm.app/server/ping
-> 200  pong
```

This is anonymous and needs no token. Use it to separate "instance down" from "token rejected".

**2. List posts — `readItemsPost`**

```
GET /items/post
  ?fields=slug,title,description,type,datetime,is_pinned,featured_image.id
  &filter={"status":{"_eq":"published"}}
  &sort=-datetime
  &limit=20
  &meta=filter_count
```

- `fields` — always request a sparse fieldset. Dot notation traverses the relation
  (`featured_image.id`). Omitting `fields` returns the full record including audit columns you
  almost certainly do not want.
- `filter` — a Directus JSON filter object. Operators: `_eq`, `_neq`, `_in`, `_gt`, `_lt`,
  `_contains`, `_between`, `_null`, plus `_and`/`_or`.
- `sort` — `-datetime` for newest first. `datetime` is the editorial publication timestamp; do not
  sort on `date_created`, which is the audit column and can differ.
- `search` — full-text across string fields, if you need it instead of a filter.
- `meta=filter_count` — counts are **opt-in** and arrive in a sibling `meta` object, not in a
  header. Ask for them or you will not get them.

**3. Paginate**

Use `limit` with either `offset` or `page` (not both). Stop when a page returns fewer rows than
`limit`; compare against `meta.filter_count` if you requested it.

**4. Fetch one post — `readSingleItemsPost`**

```
GET /items/post/{slug}?fields=slug,title,content,datetime,featured_image.id
```

**The path parameter is the `slug`, not a numeric id or UUID** — this collection's primary key is
an ID slug. A renamed slug is a new identity, so cache by slug and expect 404 after a rename.

**5. Fetch the image — `getAsset`**

```
GET /assets/{featured_image.id}
```

Takes the file **UUID** from `featured_image.id`. This is the one operation that is anonymously
reachable — carbonfarm.tech serves its post images from it.

## Efficiency

Weak ETags are returned at runtime even though the spec does not declare them:

```
etag: W/"1d4-jtAm57rJXkGZKmjK2N0xRH3K4uw"
```

Store it and send `If-None-Match` on re-reads. Do not poll unconditionally — no rate limits are
documented and no `RateLimit-*` or `Retry-After` headers are returned, so you have **no runtime
back-off signal**. Be conservative: this is internal infrastructure for a marketing site, not a
metered product API.

## Errors

Not RFC 9457. Branch on `errors[].extensions.code`:

| Status | code | Meaning | Do |
|---|---|---|---|
| 403 | `FORBIDDEN` | Role lacks read permission | Stop. Get a token with collection read. Retrying will not help. |
| 401 | `INVALID_CREDENTIALS` | Token missing/expired/invalid | Reissue the static token, or `POST /auth/refresh` for a session token. |
| 400 | `INVALID_PAYLOAD` | Malformed request | Read `extensions.reason` — it names the failing constraint. Fix and retry once. |
| 404 | `ROUTE_NOT_FOUND` | No such route | `extensions.path` echoes what you asked for. Only `/items/post` and `/items/post/{id}` exist; a bare `/items` is not a route. |

There is **no idempotency mechanism** on this API, and no 5xx is declared anywhere in the
contract. All operations in this skill are `GET`, so retrying a read is safe.

## Scope warning

This API is the **marketing website's CMS**. It exposes exactly one content collection (`post`)
plus files. It contains **no** rice, paddy, emissions, MRV, project or carbon-credit data. If you
are trying to reach CarbonFarm's satellite MRV data, it is not here — that platform sits behind an
Auth0 organization login at `app.carbonfarm.tech` and publishes no contract.
