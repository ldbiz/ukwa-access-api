# 01 — Important Files

## Entrypoints

| File / Directory | Role | Why it matters |
|---|---|---|
| `ukwa_api/main.py` | FastAPI application factory | Registers all routers, configures CORS, Prometheus, OpenAPI metadata, and static file serving. The single authoritative place to see what the app exposes. |
| `ukwa_api/worker.py` | Gunicorn/Uvicorn worker class | Sets `root_path`, proxy headers, and trusted IP forwarding for production deployment. Without this, reverse-proxy path prefixes break. |
| `Dockerfile` | Container build + CMD | Defines the production startup command, env var defaults (`SCRIPT_NAME`, `WORKERS`, `LOG_LEVEL`), and bakes in `API_VERSION` at build time. |

## Configuration

| File / Directory | Role | Why it matters |
|---|---|---|
| `docker-compose.yml` | Local integration stack | Defines all dependent services (CDX, PyWB, WARC server, renderer, IIIF, Kafka) and their environment wiring. The definitive map of how services connect. |
| `requirements.txt` | Pinned Python dependencies | All versions are pinned; changes here have direct runtime impact. |
| `gunicorn.ini` | Gunicorn configuration file | Sets worker count formula, timeouts, backlog, and log targets. Superseded by the Docker CMD flags in practice, but still loaded. |

## Core Application Logic

| File / Directory | Role | Why it matters |
|---|---|---|
| `ukwa_api/cdx.py` | CDX and WARC utilities | Contains `can_access` (access-control gate), `lookup_in_cdx` (closest-match lookup), `list_from_cdx` (index query), and `get_warc_stream` (WebHDFS retrieval). Used by both mementos and IIIF modules. |
| `ukwa_api/pwid.py` | PWID encode/decode | Converts between 14-digit Wayback timestamps + URLs and Base64-encoded PWID URNs. All IIIF endpoints depend on this. |
| `ukwa_api/mementos/router.py` | Mementos endpoints | Implements `/mementos/cdx`, `/mementos/resolve`, `/mementos/warc`, and `/mementos/screenshot`. The primary access endpoints. |
| `ukwa_api/iiif/router.py` | IIIF Image API proxy | Implements IIIF info and image endpoints, the raw renderer callback (`/iiif/render_raw`), and handles PWID-with-slashes fallback routing. |
| `ukwa_api/crawls/router.py` | Crawl stats endpoint | Reads a JSON file from disk and returns it as crawl activity data. |
| `ukwa_api/collections/router.py` | Collection download endpoint | Serves per-collection JSON files from a configurable directory. |
| `ukwa_api/mementos/schemas.py` | Shared request schemas | Defines timestamp/URL path parameters and query enums used across mementos and IIIF routes. |
| `ukwa_api/dependencies.py` | DB session dependency | SQLAlchemy session factory injected as a FastAPI dependency. Currently only relevant to the disabled nominations router. |

## Tests and Fixtures

| File / Directory | Role | Why it matters |
|---|---|---|
| `test/data/fc.crawled.json` | Crawl stats fixture | Served directly by `GET /crawls/fc/recent-activity` when `ANALYSIS_SOURCE_FILE` is not overridden. Defines what the endpoint returns in development. |
| `test/data/collections/4388.json` | Collection fixture | Served by `GET /collections/download/4388` when `JSON_DIR` is not overridden. Represents a minimal real collection record. |
| `integration-testing/` | Local integration test stack | Contains WARC files, PyWB config, and a CDX-population script for end-to-end testing with `docker-compose`. |
