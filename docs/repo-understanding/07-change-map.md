# 07 — Change Map

## If I Need to Change Startup or Runtime Behaviour

| File | Why |
|---|---|
| `ukwa_api/main.py` | Router registration, CORS policy, OpenAPI metadata, middleware. |
| `ukwa_api/worker.py` | Gunicorn/Uvicorn worker settings: `root_path`, proxy headers, trusted IPs. |
| `Dockerfile` | Production startup command, env var defaults (`SCRIPT_NAME`, `WORKERS`, `LOG_LEVEL`), `API_VERSION` build arg. |
| `gunicorn.ini` | Fallback Gunicorn settings (timeout, backlog, worker count formula). Note: Docker CMD flags take precedence. |

---

## If I Need to Change Configuration

| File | Why |
|---|---|
| `Dockerfile` | Env var defaults baked into the image. |
| `docker-compose.yml` | Local integration stack environment wiring. Also the canonical map of all service-to-service URLs. |
| `05-config-and-env.md` | Documentation of all env vars and their effects. Update this too. |

**Caveat**: All external service URLs are resolved at **module import time** (top of each module). Changing an env var after the app starts has no effect without a restart.

---

## If I Need to Change Request Handling or Core Behaviour

### Mementos (CDX lookup, URL resolve, WARC, screenshot redirect)

| File | Why |
|---|---|
| `ukwa_api/mementos/router.py` | All `/mementos/*` endpoints. |
| `ukwa_api/mementos/schemas.py` | Shared path/query parameter definitions (timestamp format, match types, sort orders, collapse). |
| `ukwa_api/cdx.py` | CDX XML query logic and WARC byte-range retrieval. |
| `ukwa_api/pwid.py` | PWID generation (used by screenshot redirect). |

### IIIF / Screenshots

| File | Why |
|---|---|
| `ukwa_api/iiif/router.py` | All `/iiif/*` endpoints, including the `render_raw` callback and PWID-with-slashes fallback routing. |
| `ukwa_api/pwid.py` | PWID parsing — all IIIF endpoints decode PWIDs from path parameters. |
| `ukwa_api/cdx.py` | Used by `render_raw` when `source=original` to look up a screenshot WARC record. |

**Caveat**: The IIIF rendering involves a **two-hop call chain**: the external client → Cantaloupe → `render_raw`. Changes to `render_raw` affect what Cantaloupe fetches, not what the client directly sees.

### Crawl Stats

| File | Why |
|---|---|
| `ukwa_api/crawls/router.py` | Single endpoint. Reads the JSON file path from `ANALYSIS_SOURCE_FILE`. |
| `test/data/fc.crawled.json` | The default data source. Update this fixture if the expected output format changes. |

### Collection Downloads

| File | Why |
|---|---|
| `ukwa_api/collections/router.py` | Single endpoint. Resolves files from `JSON_DIR`. |
| `test/data/collections/` | Default collection files served in development. |

---

## If I Need to Change External Service Integration

| Service | File | Notes |
|---|---|---|
| CDX Server | `ukwa_api/cdx.py` | `CDX_SERVER` env var. Both the streaming proxy (mementos) and the internal XML query use this. |
| Wayback (access check) | `ukwa_api/cdx.py` (`can_access`) | `WAYBACK_SERVER` env var. All WARC and IIIF endpoints call this first. |
| WARC Server | `ukwa_api/cdx.py` (`get_warc_stream`) | `WEBHDFS_PREFIX` + `WEBHDFS_USER` env vars. |
| IIIF Server (Cantaloupe) | `ukwa_api/iiif/router.py` (`proxy_call`) | `IIIF_SERVER` env var. |
| Web Render Server | `ukwa_api/iiif/router.py` (`render_raw`) | `WEBRENDER_ARCHIVE_SERVER` env var. |

---

## If I Need to Re-enable or Change Nominations

The nominations feature is fully implemented but disabled.

| File | Why |
|---|---|
| `ukwa_api/main.py` | Uncomment the `app.include_router(nominations.router, ...)` block. |
| `ukwa_api/nominations/router.py` | All nomination endpoints (create, list, get). |
| `ukwa_api/nominations/crud.py` | Database query functions. |
| `ukwa_api/nominations/models.py` | SQLAlchemy ORM models (`Nomination`, `Tag`). |
| `ukwa_api/nominations/schemas.py` | Pydantic schemas for public/private nomination fields. |
| `ukwa_api/nominations/rss.py` | RSS feed rendering for nominations. |
| `ukwa_api/dependencies.py` | DB session dependency. Ensure `SQLALCHEMY_DATABASE_URL` is set in production. |

---

## If I Need to Change Tests or Fixtures

| File | Why |
|---|---|
| `test/data/fc.crawled.json` | Update this to change the crawl stats returned in development. |
| `test/data/collections/` | Add or update collection JSON files for development/testing. |
| `integration-testing/webarchive/` | Add WARC files here and re-run `populate-ocdx.sh` to update the CDX index for integration testing. |
| `integration-testing/populate-ocdx.sh` | Update if the CDX server URL or index population approach changes. |
