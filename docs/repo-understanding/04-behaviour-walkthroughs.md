# 04 — Behaviour Walkthroughs

## 1. CDX Index Lookup (`GET /mementos/cdx`)

**Trigger**: A client (user, script, or another service) wants to know which captures exist for a given URL in the archive.

**Files involved**: `ukwa_api/mementos/router.py`, `ukwa_api/cdx.py`

**Flow**:  
The request is validated (e.g. sort + closest timestamp must be consistent, collapse options are mutually exclusive), then proxied **as a streaming response** to the OutbackCDX server.  
The API passes through a filtered set of query parameters: URL, match type, sort, limit, output format, date range, and collapse options.  
The CDX server's response — either CDX11 plain-text or JSON — is streamed directly back to the client without buffering.

| | |
|---|---|
| **Inputs** | `url` (required), `matchType`, `sort`, `limit`, `output`, `closest`, `from`, `to`, `collapseToFirst/Last` |
| **Outputs** | CDX11 plain text or JSON, streamed |
| **External calls** | OutbackCDX server (`CDX_SERVER`) |
| **Side effects** | None |
| **Test coverage** | None (no unit tests exist) |

---

## 2. Resolve an Archived URL to Wayback (`GET /mementos/resolve/{timestamp}/{url}`)

**Trigger**: A client has a URL and a timestamp and wants to view the archived page in a browser.

**Files involved**: `ukwa_api/mementos/router.py`

**Flow**:  
The endpoint constructs a Wayback URL from the timestamp and URL path, then issues a **303 redirect** to `/wayback/archive/{timestamp}/{url}`.  
No CDX lookup, no access check — this is a pure redirect.  
The Wayback service itself handles access control and replay.

| | |
|---|---|
| **Inputs** | 14-digit timestamp (path), URL (path) |
| **Outputs** | HTTP 303 redirect to Wayback |
| **External calls** | None (client is redirected to Wayback) |
| **Side effects** | None |
| **Test coverage** | None |

---

## 3. Retrieve a Raw WARC Record (`GET /mementos/warc/{timestamp}/{url}`)

**Trigger**: A researcher or tool wants the raw WARC record for a specific capture.

**Files involved**: `ukwa_api/mementos/router.py`, `ukwa_api/cdx.py`

**Flow**:  
1. `can_access` checks the Wayback server to confirm the URL is open-access.
2. `lookup_in_cdx` queries the CDX server's XML API to find the closest capture to the given timestamp, returning the WARC filename, byte offset, and compressed record length.
3. `get_warc_stream` fetches a byte-range from the WARC server (WebHDFS), then uses `warcio`'s `DecompressingBufferedReader` to decompress the GZip chunk.
4. The WARC record is streamed back to the client with a `Content-Disposition` header for download.

| | |
|---|---|
| **Inputs** | 14-digit timestamp (path), URL (path) |
| **Outputs** | Streaming WARC or ARC record (`application/warc`) |
| **External calls** | Wayback (access check), CDX server (XML query), WARC server (WebHDFS byte-range) |
| **Side effects** | None |
| **Test coverage** | None |

---

## 4. Serve a Screenshot via IIIF (`GET /iiif/2/{pwid}/{region}/{size}/{rotation}/{quality}.{format}`)

**Trigger**: A client requests an image of an archived web page — for example a thumbnail in a collection UI.

**Files involved**: `ukwa_api/iiif/router.py`, `ukwa_api/pwid.py`, `ukwa_api/cdx.py`

**Flow**:  
This behaviour involves a **two-step callback chain**:

**Step 1 — Client calls the IIIF image endpoint:**
1. The PWID is decoded (Base64 or plain) to extract archive ID, timestamp, and URL.
2. `can_access` checks that the URL is open-access.
3. The API proxies the request to Cantaloupe (the IIIF server), adjusting the PWID path encoding.

**Step 2 — Cantaloupe calls back to `/iiif/render_raw`:**
4. Cantaloupe needs the raw image for the requested PWID.
5. It calls `/iiif/render_raw?pwid=...` on the API.
6. The API decodes the PWID, checks the filesystem screenshot cache (keyed by PWID).
7. If cached, returns the cached image immediately.
8. If not cached and `source=archive` (the default), calls the Web Render server with the archived URL and timestamp, receives a PNG screenshot.
9. If `source=original`, looks up the screenshot WARC record via CDX and streams it from the WARC server.
10. The image is stored in the cache (1-hour TTL) and returned to Cantaloupe.

**Step 3 — Cantaloupe applies image transformations** (crop region, resize, rotate, format conversion) and returns the result to the original client.

| | |
|---|---|
| **Inputs** | PWID (Base64 or plain), region, size, rotation, quality, format |
| **Outputs** | Image (PNG or JPEG) |
| **External calls** | Wayback (access check), Cantaloupe (image proxy), Web Render server (screenshot), optionally CDX + WARC server (original screenshot) |
| **Side effects** | Screenshot cached to filesystem |
| **Test coverage** | None |
