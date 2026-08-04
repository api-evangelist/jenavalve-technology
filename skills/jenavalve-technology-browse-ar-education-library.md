---
name: Browse the JenaValve Discover AR education library
description: >-
  Retrieve JenaValve's aortic-regurgitation education library from discover-ar.com by media
  format — articles, videos, audio, presentations — using the anonymous WordPress REST API and
  the Yoast head endpoint for structured metadata.
api: openapi/jenavalve-technology-discover-ar-openapi.yml
base_url: https://discover-ar.com/wp-json
auth: none
operations:
  - listDiscoverArCategories
  - listDiscoverArPosts
  - getDiscoverArPost
  - searchDiscoverAr
  - listDiscoverArPages
  - listDiscoverArMedia
  - getYoastHead
---

# Browse the JenaValve Discover AR education library

## Before you start

Discover AR (discover-ar.com) is JenaValve Technology's aortic-regurgitation education site,
linked from the jenavalve.com navigation as "Resources". This is its WordPress REST API — an
incidental CMS surface, not a published API product. No credential is required and none exists;
send no `Authorization` header. There is no versioning, SLA, status page or support channel.

This is a **separate WordPress installation** from jenavalve.com. The two share no identifier
space: id 409 here has nothing to do with id 409 there. Never join across the two by id.

Accuracy rules when summarizing what you retrieve:

- This is **patient and clinician education material published by a device manufacturer**. It is
  not independent clinical guidance and not a substitute for medical advice. Say so if you
  surface it to a user asking a health question.
- The Trilogy System is CE marked and is an **investigational device in the United States**.
  Never describe it as FDA approved.

## Step 1 — read the format taxonomy first

This is the one endpoint that makes this site worth calling. The category taxonomy is
purpose-built to segment the library by media format, so you can retrieve every AR video or every
AR audio item directly instead of scraping the resources page.

```
GET /wp/v2/categories
```

As of 2026-08-04: `articles` (id 12, 4 items), `videos` (id 13, 6), `audio` (id 15, 3),
`presentations` (id 14, 0), `uncategorized` (id 1, 0). **Read the ids and counts from the live
response** — do not hardcode these. Note that `videos` is id 13 here but id 17 on jenavalve.com.

## Step 2 — list the library by format

`listDiscoverArPosts` with the id from step 1:

```
GET /wp/v2/posts?categories=13&per_page=100&_fields=id,title,link,date,excerpt,categories
```

Omit `categories` to get the whole library — 13 items as of 2026-08-04, small enough to pull in
one page.

Always pass `_fields`; a full post object carries its rendered HTML body.

## Step 3 — search instead, when the user has a topic

```
GET /wp/v2/search?search=regurgitation&per_page=20
```

Returns `{id, title, url, type, subtype}`. Dereference `subtype: post` with
`getDiscoverArPost`, `subtype: page` with `listDiscoverArPages` / the page id.

## Step 4 — read one item

```
GET /wp/v2/posts/409
```

`title`, `content` and `excerpt` are `{ "rendered": "<html>" }` — strip the markup. `categories`
is an array of term ids you can map back to the format names from step 1. Add `?_embed` if you
want the featured media and terms inlined rather than making follow-up calls.

## Step 5 — structured metadata without parsing HTML

Unique to this host: the Yoast SEO endpoint returns rendered head metadata, including
**schema.org JSON-LD**, for any discover-ar.com URL.

```
GET /yoast/v1/get_head?url=https%3A%2F%2Fdiscover-ar.com%2Fcontemporary-treatment-of-aortic-regurgitation%2F
```

Use this when you want canonical URL, meta description, Open Graph fields or JSON-LD for a
library item. It is the cheapest route to structured metadata here. This endpoint does **not**
exist on jenavalve.com.

## Step 6 — media assets

```
GET /wp/v2/media?per_page=100&_fields=id,title,source_url,mime_type,media_type
```

63 items as of 2026-08-04. `source_url` is the direct file URL. Filter with `media_type` (image,
video, audio, application) when you want one class of asset.

## Pagination

Totals live **only** in the `X-WP-Total` and `X-WP-TotalPages` headers; the body is a bare array.
A `Link` header carries `rel="next"`. `per_page` maxes at 100 and is not clamped — `per_page=200`
returns HTTP 400 `rest_invalid_param`.

## Error handling

Not RFC 9457. The envelope is `{"code", "message", "data": {"status"}}` as `application/json`.
Branch on `code`:

- `rest_post_invalid_id` (404) — no object with that ID.
- `rest_invalid_param` (400) — inspect `data.params`.
- `rest_forbidden` (401) — administrative route, e.g. `wp/v2/settings`. **Not remediable**; no
  public credential exists. Do not retry, do not try to authenticate.

Full catalog: `errors/jenavalve-technology-problem-types.yml`.

## Courtesy

No rate-limit headers are published. The whole library is 13 posts and 63 media items — pull it
in a handful of `_fields`-trimmed calls and cache the result rather than polling. Back off on any
429 or 5xx. Read-only: anonymous responses advertise `Allow: GET`.
