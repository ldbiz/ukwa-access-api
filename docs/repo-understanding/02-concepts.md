# 02 — Core Concepts

## Memento / Archived URL

A **Memento** (per [RFC 7089](https://datatracker.ietf.org/doc/html/rfc7089)) is a snapshot of a web resource captured at a specific point in time.

- In this repo, a memento is identified by a **URL + 14-digit timestamp** pair (e.g. `19950630120000` for 30 June 1995 at noon).
- The mementos module is the main access path: it exposes CDX lookup, URL resolution to Wayback, and raw WARC retrieval.
- Relevant files: `ukwa_api/mementos/router.py`, `ukwa_api/mementos/schemas.py`

## CDX Index

The **CDX** (Capture inDeX) is the searchable index of all archived URLs.

- Each CDX record maps a canonicalised URL + timestamp to the WARC filename and byte offset where the capture lives.
- The CDX server used here is [OutbackCDX](https://github.com/nla/outbackcdx), queried via HTTP.
- The API exposes CDX queries directly at `GET /mementos/cdx`, proxying requests to the CDX server.
- Internal lookups (for WARC retrieval and screenshots) use the XML-based query API from `ukwa_api/cdx.py`.
- Relevant files: `ukwa_api/cdx.py`, `ukwa_api/mementos/router.py`

## WARC

A **WARC** (Web ARChive) file is the standard container format for archived web content.

- Each WARC file stores compressed HTTP request/response records.
- The API retrieves individual WARC records by locating the file via CDX, then fetching a byte-range from the WARC server (via WebHDFS), and streaming the decompressed record back to the caller.
- Relevant files: `ukwa_api/cdx.py` (`get_warc_stream`), `ukwa_api/mementos/router.py` (`/mementos/warc`)

## PWID (Persistent Web IDentifier)

A **PWID** is a URN that uniquely identifies a specific archived web page.

Its canonical form is:
```
urn:pwid:<archive-id>:<ISO-timestamp>:page:<url>
```
Example:
```
urn:pwid:webarchive.org.uk:1995-04-18T15:56:00Z:page:http://portico.bl.uk/
```

- PWIDs are used as IIIF image identifiers throughout the IIIF module.
- Because IIIF URL paths cannot contain raw forward-slashes, PWIDs are **Base64-encoded** (URL-safe) when used as path parameters.
- `ukwa_api/pwid.py` handles encoding and decoding.
- A fallback route in `ukwa_api/iiif/router.py` handles PWIDs with unescaped slashes using regex matching.
- Relevant files: `ukwa_api/pwid.py`, `ukwa_api/iiif/router.py`

## IIIF Image API

The [IIIF Image API](https://iiif.io/api/image/2.1/) is a standard for serving and manipulating images.

- This repo implements IIIF as a **proxy** in front of a [Cantaloupe](https://cantaloupe-project.github.io/) image server.
- The IIIF identifier used here is a PWID, encoding the archive, timestamp, and URL.
- Cantaloupe fetches raw image data by calling back to `/iiif/render_raw` on this API, which either pulls a stored screenshot from a WARC or requests a live rendering from the web rendering service.
- Relevant files: `ukwa_api/iiif/router.py`

## Access Control (`can_access`)

Before serving any WARC record or screenshot, the API checks whether the requested URL is **open access**.

- This check is a live HTTP request to the Wayback server.
- A non-2xx/3xx response from Wayback causes the API to propagate that status code back to the caller (e.g. 403 for restricted content).
- Relevant files: `ukwa_api/cdx.py` (`can_access`), called from `ukwa_api/mementos/router.py` and `ukwa_api/iiif/router.py`

## Wayback / PyWB

The **Wayback** service ([PyWB](https://github.com/webrecorder/pywb)) is the replay layer that serves archived web pages to users in a browser.

- The mementos `/resolve` endpoint redirects users to the Wayback URL rather than serving content directly.
- The Wayback server also acts as the **access-control oracle** (via `can_access`).
- In integration testing, PyWB is run locally and configured with test WARC collections.

## Nomination (Disabled)

A **Nomination** is a user-submitted URL requesting that the UK Web Archive capture a given site.

- The nominations module (`ukwa_api/nominations/`) is **fully implemented** (SQLite database, CRUD, RSS feed, Prometheus counter) but **commented out** in `ukwa_api/main.py` and not active.
- Relevant files: `ukwa_api/nominations/` (router, crud, models, schemas, rss)

## SCRIPT_NAME / Root Path

The API is designed to be **mounted at a URL prefix** (e.g. `/api`) when deployed behind a reverse proxy.

- `SCRIPT_NAME` is passed to the UvicornWorker as `root_path`, which makes FastAPI generate correct absolute URLs in OpenAPI metadata.
- The default in Docker is `SCRIPT_NAME=/api`.
- Relevant files: `ukwa_api/worker.py`, `ukwa_api/main.py`, `Dockerfile`

## Screenshot Cache

Screenshots fetched from the web rendering service are **cached on the filesystem** to avoid duplicate rendering calls.

- The cache is keyed by PWID and has a 1-hour TTL.
- The cache directory is configured via `CACHE_FOLDER` (default: `.`).
- Relevant files: `ukwa_api/iiif/router.py`

## Prometheus Metrics

The API exposes Prometheus metrics at `/metrics` (not shown in the Swagger UI).

- A custom `UkwaApiWorker` and `PROMETHEUS_MULTIPROC_DIR` env var enable **multi-process metric aggregation** across Gunicorn workers.
- Relevant files: `ukwa_api/main.py`, `ukwa_api/worker.py`, `Dockerfile`
