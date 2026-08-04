---
name: Retrieve JenaValve company news and press materials
description: >-
  Pull JenaValve Technology press releases, videos, corporate pages and downloadable PDFs
  (product brochure, reimbursement guide) from the anonymous WordPress REST API on
  jenavalve.com, without scraping HTML.
api: openapi/jenavalve-technology-site-openapi.yml
base_url: https://jenavalve.com/wp-json
auth: none
operations:
  - searchSite
  - listPosts
  - getPost
  - listCategories
  - listPages
  - getPage
  - listMedia
  - getMediaItem
---

# Retrieve JenaValve company news and press materials

## Before you start

JenaValve Technology publishes **no API programme**. This surface is the WordPress REST API of
their corporate site — an incidental CMS side effect, not a product. It is unversioned, has no
SLA, no status page and no support channel, and it can change or close without notice. Do not
describe it to a user as an official JenaValve integration.

No credential is required and none exists. Send no `Authorization` header.

Two accuracy rules when summarizing anything you retrieve:

- The Trilogy System is **CE marked** and is an **investigational device in the United States**,
  limited by federal law to investigational use. Never call it FDA approved.
- Content here is corporate and promotional material. Do not present it as clinical guidance.

## Step 1 — find content by topic

Start with `searchSite`, which searches every post type at once and is far cheaper than listing
collections.

```
GET /wp/v2/search?search=trilogy&per_page=20
```

Each hit is `{id, title, url, type, subtype}`. `subtype` tells you how to dereference: `post`
goes to `getPost`, `page` goes to `getPage`.

## Step 2 — segment the news feed

If you want a whole category rather than a search, first read the taxonomy with
`listCategories`:

```
GET /wp/v2/categories
```

As of 2026-08-04 there are four terms — `press` (id 13, 28 posts), `videos` (id 17, 10 posts),
`experts` (id 16, 0) and `uncategorized` (id 1, 0). Read the counts from the response rather than
trusting these numbers; they change.

Then list with `listPosts`, filtering by the id you found and trimming the payload:

```
GET /wp/v2/posts?categories=13&per_page=100&_fields=id,title,link,date,excerpt
```

Always pass `_fields`. Without it every post carries its full rendered HTML body.

## Step 3 — read one item

`getPost` returns the full object including `content.rendered`:

```
GET /wp/v2/posts/946
```

Title, content and excerpt are `{ "rendered": "<html>" }` objects — strip the markup yourself.
`author` and `featured_media` are bare integer IDs. If you need them resolved, do not make extra
calls; add `?_embed` to step 2 or 3 and read the `_embedded` block instead.

## Step 4 — corporate pages

`listPages` covers the Trilogy System, About Us, Investors, Careers, Contact and the ALIGN-AR /
ARTIST / JENA-VAD clinical study pages:

```
GET /wp/v2/pages?per_page=100&_fields=id,title,link,slug,parent,menu_order
```

`parent` and `menu_order` reconstruct the site hierarchy — `parent: 0` is top level. Fetch one
with `getPage`.

## Step 5 — downloadable documents

The product brochure, the 2026 reimbursement guide and the labeling symbols glossary are media
items. Use `listMedia`:

```
GET /wp/v2/media?mime_type=application/pdf&per_page=100&_fields=id,title,source_url,filesize
```

`source_url` is the direct file URL. `getMediaItem` returns one item with its full
`media_details`.

## Pagination

Totals are returned **only in headers** — `X-WP-Total` and `X-WP-TotalPages` — never in the body,
which is a bare JSON array. A `Link` header carries `rel="next"`. If you ignore the headers you
cannot tell whether you have everything.

`per_page` maxes at **100**, and it is not clamped: `per_page=200` returns HTTP 400
`rest_invalid_param`.

## Error handling

Errors are **not** RFC 9457 problem details. The envelope is
`{"code": "...", "message": "...", "data": {"status": 404}}` served as `application/json`.
Branch on `code`, not on the status alone:

- `rest_post_invalid_id` (404) — no object with that ID. Get IDs from a collection; never guess,
  the id space is shared across post types.
- `rest_invalid_param` (400) — read `data.params` for the offending parameter. Usually `per_page`.
- `rest_forbidden` / `rest_cannot_view` / `cookieyes_rest_cannot_view` (401), `wpcf7_forbidden`
  (403) — an administrative route. **Not remediable.** No public credential exists for these
  hosts, so do not retry and do not attempt to authenticate. Stop and report the route is closed.

Full catalog: `errors/jenavalve-technology-problem-types.yml`.

## Rate limits and courtesy

No rate-limit headers are published, which does not mean no limit exists. Keep concurrency low,
prefer one `_fields`-trimmed page-100 call over many small ones, and back off on any 429 or 5xx.

## Out of scope

Do not attempt write methods — anonymous responses advertise `Allow: GET` only. Do not enumerate
`wp/v2/users` to build profiles of individuals; the two author records are publishing bylines and
are not needed for any news-retrieval task.
