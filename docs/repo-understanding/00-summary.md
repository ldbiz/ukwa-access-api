# 00 — Repository Summary

## Purpose

The **UKWA Access API** is the access-time HTTP API for the [UK Web Archive](https://www.webarchive.org.uk/).  
It exposes archived web content to users and downstream systems through a set of REST endpoints:
looking up archived URLs in the CDX index, resolving URLs to the Wayback replay service, retrieving raw WARC records, and serving screenshots of archived pages via the IIIF Image API.

## Technology Stack

| Layer | Technology |
|---|---|
| Web framework | FastAPI 0.92 (ASGI) |
| Runtime server | Gunicorn + UvicornWorker |
| Data validation | Pydantic v1 |
| HTTP clients | `requests` (sync), `httpx` (async) |
| WARC parsing | `warcio` |
| Screenshot cache | `cachelib` (filesystem) |
| DB (nominations, disabled) | SQLAlchemy + SQLite |
| Metrics | Prometheus (`prometheus-fastapi-instrumentator`) |
| Containerisation | Docker + Docker Compose |

## Main Runtime Model

The app is an **ASGI application** (`ukwa_api.main:app`) run by Gunicorn using a custom `UvicornWorker`.  
Multiple worker processes serve requests concurrently. Prometheus metrics are collected in a shared multiprocess directory.

## Main External Services

| Service | Default URL | Purpose |
|---|---|---|
| CDX Server (OutbackCDX) | `http://cdx.api.wa.bl.uk/data-heritrix` | URL index lookups |
| Wayback / PyWB | `https://www.webarchive.org.uk/wayback/archive/` | Access-control checks |
| WARC Server (WebHDFS) | `http://warc-server.api.wa.bl.uk/webhdfs/v1/by-filename/` | Raw WARC retrieval |
| Web Render Server | `http://webrender:8010/render` | Screenshot generation |
| IIIF Server (Cantaloupe) | `http://iiif:8182` | Image manipulation |

## Main Entry Points

| Entry point | Description |
|---|---|
| `uvicorn ukwa_api.main:app --reload` | Development server |
| `gunicorn -k ukwa_api.worker.UkwaApiWorker ukwa_api.main:app` | Production (Docker CMD) |
| `GET /docs` | Swagger UI (auto-generated) |
| `GET /ping` | Health check |
| `GET /metrics` | Prometheus metrics |

## Architecture Summary

The app is structured around four active FastAPI routers, each mounted in `ukwa_api/main.py`:

- **Mementos** (`/mementos`) — CDX index queries, URL resolution to Wayback, raw WARC retrieval, and screenshot redirect.
- **IIIF** (`/iiif`) — IIIF Image API proxy; fetches screenshots from a rendering service and passes them through Cantaloupe for image manipulation.
- **Crawls** (`/crawls`) — Returns recent crawl statistics read from a static JSON file.
- **Collections** (`/collections`) — Serves pre-built collection JSON files from disk.

Every IIIF or WARC request performs an **access-control check** against the Wayback server before serving content. The access control check is a live HTTP call to the Wayback service; if the URL is blocked, the response status is propagated back to the caller.

A fifth router for **URL nominations** (user-submitted URLs to be crawled) exists in `ukwa_api/nominations/` but is **commented out** in `main.py` and not active.

There is also a legacy Flask app (`api.py` / `access_api/`) that pre-dates the FastAPI rewrite. It is not used at runtime and exists only for historical reference.

## Read These First

| File / Directory | Why |
|---|---|
| `ukwa_api/main.py` | App construction, router registration, CORS, Prometheus setup |
| `ukwa_api/mementos/router.py` | Core CDX lookup, URL resolution, WARC, screenshot endpoints |
| `ukwa_api/iiif/router.py` | IIIF proxy, screenshot rendering, access-control flow |
| `ukwa_api/cdx.py` | CDX and WARC access shared utilities (`can_access`, `lookup_in_cdx`, `get_warc_stream`) |
| `ukwa_api/pwid.py` | PWID encode/decode — essential to understanding IIIF identifiers |
| `Dockerfile` | Container build, env defaults, startup command |
| `docker-compose.yml` | Full local stack with all dependent services |
| `ukwa_api/worker.py` | Gunicorn/Uvicorn worker configuration |
| `ukwa_api/dependencies.py` | SQLAlchemy DB session setup (used only by nominations) |
| `requirements.txt` | Pinned Python dependencies |
