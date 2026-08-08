---
name: Harvest the Belharra Therapeutics media library
description: >-
  Enumerate the 430-item media library, filter by media type, MIME type, parent and modification
  date, resolve featured images from any content record, and select the right generated size
  variant instead of downloading originals.
api: openapi/belharra-therapeutics-content-openapi.yml
base_url: https://belharratx.com/wp-json
auth: none
operations:
  - listMedia
  - getMediaItem
  - listMultimediaFiles
  - getMultimediaFile
  - getOembed
generated: '2026-08-06'
method: generated
---

# Harvest the Belharra Therapeutics media library

Two different things on this site are called "media", and confusing them is the main failure mode:

| | Route | Items | What it is |
|---|---|---|---|
| **Attachments** | `/wp/v2/media` | 430 | The raw WordPress upload library — every image file, logo and document |
| **Multimedia files** | `/wp/v2/multimedia-file` | 10 | A *curated editorial* custom post type backing https://belharratx.com/multimedia-library/ — video and the "Birth of a Biotech" podcast series |

If you want files, use `media`. If you want Belharra's curated video and podcast programming, use
`multimedia-file`. **Send no credentials.**

## Step 1 — enumerate attachments

```
GET /wp/v2/media?per_page=100&orderby=date&order=desc
```

`listMedia`. 430 items at harvest, so ~5 pages at `per_page=100`. Read `X-WP-Total` and
`X-WP-TotalPages`; follow the `Link` header's `rel="next"`.

Media records are the heaviest on the surface — trim hard:

```
GET /wp/v2/media?per_page=100&_fields=id,slug,source_url,mime_type,media_type,alt_text,filesize,media_details
```

## Step 2 — filter

- `media_type` — `image`, `video`, `text`, `application`, `audio`
- `mime_type` — e.g. `image/png`, `image/svg+xml`, `image/webp`
- `parent` / `parent_exclude` — the content object an attachment was uploaded to
- `after` / `before` / `modified_after` / `modified_before` — ISO 8601 windows
- `search` — matches title, caption and alt text

```
GET /wp/v2/media?media_type=image&mime_type=image/svg%2Bxml&per_page=100
```

Attachments uploaded directly to the library rather than to a post have `post: null`, so
`parent`-based filtering silently misses them. Do not use it as your only pass.

## Step 3 — pick the right size, do not download originals

Each record's `media_details.sizes` holds the generated variants — typically `thumbnail`,
`medium`, `medium_large`, `large` and `full`, each with its own `source_url`, `width`, `height` and
`mime_type`. Select the smallest variant that meets your need. The top-level `source_url` is always
the **original**, which for the team photography is a multi-megabyte file. `filesize` (bytes) is on
the record, so you can decide before you fetch.

Uploads live under `https://belharratx.com/wp-content/uploads/YYYY/MM/`. The path is stable, but
read it from `source_url` rather than constructing it — `media_details.file` is relative to the
uploads root.

## Step 4 — resolve a featured image from any content record

Every post, page, press release, company-news item and multimedia-file carries `featured_media` —
an attachment ID, or `0` when unset.

```
GET /wp/v2/media/{featured_media}
```

`getMediaItem`. Or inline it in one request:

```
GET /wp/v2/press-release?per_page=100&_embed
```

which puts the full attachment under `_embedded['wp:featuredmedia'][0]`. One request instead of
N+1. Guard for `featured_media: 0` — the `_embedded` key is simply absent, not null.

## Step 5 — the curated multimedia collection

```
GET /wp/v2/multimedia-file?per_page=100
```

`listMultimediaFiles` / `getMultimediaFile`. These are editorial records, not files. Each has
`title`, `content.rendered`, `menu_order` (which controls display order on the Multimedia Library
page, independent of date) and — critically — an **`acf` object where the actual video and podcast
references live**. The ACF field definitions are behind the 401-gated admin, so inspect the object
rather than assuming a shape. The embedded player markup may also be present inside
`content.rendered`.

`multimedia-file` declares no taxonomy, so filter by `search`, date window, `slug` or `menu_order`
only.

## Step 6 — oEmbed for a page

```
GET /oembed/1.0/embed?url=https%3A%2F%2Fbelharratx.com%2F&maxwidth=600
```

`getOembed` returns a standard oEmbed 1.0 rich response — `version`, `provider_name`,
`provider_url`, `author_name`, `author_url`, `title`, `type`, `width`, `height` and iframe `html`.
It resolves **belharratx.com permalinks only**; a foreign URL returns 404 and omitting `url`
entirely returns 400 `rest_missing_callback_param`.

## Rights

These files are Belharra Therapeutics' property. The library includes the Belharra and Searchlight
logos, team photography, podcast artwork and third-party press-outlet logos that are those outlets'
marks, not Belharra's to sublicense. The site's Terms of Use are at
https://belharratx.com/terms/. Enumerating the library through the public API does not grant a
licence to republish it.

## Etiquette

430 attachments across 5 paged requests is trivial; downloading 430 originals is not. Fetch
metadata first, decide with `filesize`, then pull only what you need. `robots.txt` advertises
`Crawl-delay: 10`. No `RateLimit-*` headers exist. Errors are `{code, message, data:{status}}` —
branch on `code`.
