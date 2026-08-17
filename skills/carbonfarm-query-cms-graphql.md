---
name: Query the CarbonFarm CMS over GraphQL
description: >-
  Use CarbonFarm's Directus GraphQL endpoint for the capabilities the REST surface does not
  expose — post aggregation, content versions, and realtime create/update/delete subscriptions —
  without sending an introspection query, because the full SDL is already published.
api: graphql/carbonfarm-cms-schema.graphql
endpoint: https://cms.int.carbonfarm.app/graphql
operations:
- post
- post_by_id
- post_aggregated
- post_by_version
- post_mutated
- directus_files_mutated
generated: '2026-08-17'
method: generated
source: >-
  Grounded in graphql/carbonfarm-cms-schema.graphql, published verbatim by CarbonFarm at
  https://cms.int.carbonfarm.app/server/specs/graphql, plus
  mcp/carbonfarm-tool-crosswalk.yml.
---

# Query the CarbonFarm CMS over GraphQL

## When to use this instead of REST

Reach for GraphQL only for the four capabilities REST cannot do (see
`mcp/carbonfarm-tool-crosswalk.yml`):

| Need | GraphQL field | REST equivalent |
|---|---|---|
| Group/count posts | `post_aggregated` | none — REST gives only `meta.total_count`/`filter_count` |
| A named content version | `post_by_version` → `version_post` | partial (a `version` query param, no typed return) |
| React to changes | `Subscription.post_mutated` | none |
| React to file changes | `Subscription.directus_files_mutated` | none |

For plain reads, prefer REST — it also gives you `export`, asset delivery and ETags.

## Do not send an introspection query

CarbonFarm publishes the complete SDL anonymously:

```
GET https://cms.int.carbonfarm.app/server/specs/graphql   -> 200, 6,583 bytes
```

It is saved at `graphql/carbonfarm-cms-schema.graphql`. Read the schema from there. Sending a
full introspection query to a production endpoint you do not own is unnecessary work.

## Authentication

Same credential as REST — `Authorization: Bearer <token>`. Queries fail without it. A request with
no body at all returns:

```
400 {"errors":[{"message":"Invalid payload. Must provide query string.",
                "extensions":{"code":"INVALID_PAYLOAD","reason":"Must provide query string"}}]}
```

## The `post` type

`slug: ID!` is the primary key. Fields: `title`, `description!`, `content!`, `type!`, `status`,
`datetime!`, `is_pinned!`, `featured_image` (→ `directus_files`), plus audit columns
(`user_created`, `date_created`, `user_updated`, `date_updated`).

No enum is published for `type` or `status`, so **valid values are not discoverable from the
schema**. Do not hard-code a status string; read the distinct values with `post_aggregated`
first (step 2).

## Steps

**1. Read posts — `post`**

```graphql
query Recent($limit: Int!) {
  post(
    filter: { status: { _eq: "published" } }
    sort: ["-datetime"]
    limit: $limit
  ) {
    slug
    title
    description
    type
    datetime
    is_pinned
    featured_image { id }
  }
}
```

Arguments mirror REST: `filter`, `sort`, `limit`, `offset`, `page`, `search`. The typed filter
inputs are `post_filter` composed of `string_filter_operators`, `number_filter_operators`,
`date_filter_operators`, `boolean_filter_operators`, `big_int_filter_operators` and
`count_function_filter_operators`.

**2. Discover the real category/status values — `post_aggregated`**

```graphql
query Types {
  post_aggregated(groupBy: ["type"]) {
    group
    count { slug }
  }
}
```

This is the capability that justifies using GraphQL at all. Do the same with
`groupBy: ["status"]` to learn the publish-workflow values the schema does not enumerate.

**3. One post — `post_by_id`**

```graphql
query One($slug: ID!) { post_by_id(id: $slug) { slug title content datetime } }
```

The argument is named `id` but the value is the **slug**.

**4. A content version — `post_by_version`**

```graphql
query Draft($version: String!, $slug: ID!) {
  post_by_version(version: $version, id: $slug) { slug title content }
}
```

**5. Subscribe to changes — `post_mutated`**

```graphql
subscription { post_mutated(event: update) { key event } }
```

`event` is `EventEnum`: `create | update | delete`. Transport is a Directus WebSocket connection.

Because CarbonFarm publishes **no AsyncAPI and no webhooks**, this subscription root is the only
event surface that exists, and it is discoverable only by reading the SDL. Do not assume a webhook
endpoint exists — none is documented. If you cannot hold a WebSocket, fall back to conditional
REST polling with `If-None-Match` rather than tight polling; there are no rate-limit headers to
guide you.

## Errors

The GraphQL endpoint returns the same Directus envelope as REST —
`{"errors":[{"message","extensions":{"code"}}]}`. Branch on `extensions.code`
(`FORBIDDEN`, `INVALID_CREDENTIALS`, `INVALID_PAYLOAD`). See
`errors/carbonfarm-problem-types.yml`.

## Scope warning

The whole schema is two entities — `post` and `directus_files`. This is the website CMS, not
CarbonFarm's rice MRV platform. No emissions, paddy, project or credit data is reachable here.
