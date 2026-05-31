# 05 — Configuration and Environment

All runtime configuration is via **environment variables**.  
There are no config files read by the application at startup (the `gunicorn.ini` is available but the Docker CMD flags take precedence for most settings).

## Environment Variables

### API Identity

| Variable | Default | Required | Effect |
|---|---|---|---|
| `API_VERSION` | `0.0.0-dev` | No | Version string shown in OpenAPI docs. Set at build time via Docker `ARG`. |
| `SCRIPT_NAME` | `""` (dev) / `/api` (Docker) | No | URL prefix when mounted behind a reverse proxy. Passed as `root_path` to Uvicorn. |

### External Services

| Variable | Default | Required | Effect |
|---|---|---|---|
| `CDX_SERVER` | `http://cdx.api.wa.bl.uk/data-heritrix` | Yes (for WARC/screenshot lookup) | OutbackCDX endpoint. Used for CDX proxy and internal lookups. |
| `WAYBACK_SERVER` | `https://www.webarchive.org.uk/wayback/archive/` | Yes | Wayback endpoint used for access-control checks. Must be open-access Wayback. |
| `WEBHDFS_PREFIX` | `http://warc-server.api.wa.bl.uk/webhdfs/v1/by-filename/` | Yes (for WARC retrieval) | WebHDFS base URL for raw WARC byte-range reads. |
| `WEBHDFS_USER` | `access` | No | Username passed to WebHDFS in `user.name` query param. |
| `WEBRENDER_ARCHIVE_SERVER` | `http://webrender:8010/render` | Yes (for screenshot rendering) | Web rendering service endpoint (Puppeteer-based). |
| `IIIF_SERVER` | `http://iiif:8182` | Yes (for IIIF) | Cantaloupe IIIF server. Receives proxied image requests. |

### Runtime and Serving

| Variable | Default | Required | Effect |
|---|---|---|---|
| `WORKERS` | `2` | No | Number of Gunicorn worker processes. |
| `LOG_LEVEL` | `info` | No | Gunicorn/Uvicorn log level (`debug`, `info`, `warning`, `error`). |
| `PROMETHEUS_MULTIPROC_DIR` | `/tmp_multiproc` (Docker) | Yes (multi-process) | Shared directory for Prometheus multi-process metric files. Must exist. |

### File Paths

| Variable | Default | Required | Effect |
|---|---|---|---|
| `CACHE_FOLDER` | `.` | No | Root directory for the filesystem screenshot cache (`<CACHE_FOLDER>/screenshot_cache/`). |
| `ANALYSIS_SOURCE_FILE` | `test/data/fc.crawled.json` | No | JSON file read by `GET /crawls/fc/recent-activity`. |
| `JSON_DIR` | `test/data/collections/` | No | Directory from which collection JSON files are served. |

### Database (Nominations — Disabled)

| Variable | Default | Required | Effect |
|---|---|---|---|
| `SQLALCHEMY_DATABASE_URL` | `sqlite:///./sql_app.db` | No | DB connection string. Used only if the nominations router is re-enabled. |

### Legacy (Flask app — Not Active)

| Variable | Default | Notes |
|---|---|---|
| `APP_SECRET` | `dev-mode-key` | Flask `SECRET_KEY`. Only relevant to the old `api.py`. |
| `KAFKA_LAUNCH_BROKER` | None | Kafka broker for crawl queue. Only used in `api.py`. |
| `KAFKA_LAUNCH_TOPIC` | None | Kafka topic for crawl queue. Only used in `api.py`. |

## Integration Testing Configuration (docker-compose.yml)

The compose file overrides the following for local development:

```yaml
CDX_SERVER: http://cdxserver:8080/tc
WAYBACK_SERVER: http://pywb:8080/test/
WEBHDFS_PREFIX: http://warc-server:8000/by-filename/
WEBRENDER_ARCHIVE_SERVER: http://webrender:8010/render
IIIF_SERVER: http://iiif:8182
LOG_LEVEL: debug
SCRIPT_NAME: ""
```

The CDX server needs to be populated before integration testing by running `integration-testing/populate-ocdx.sh`, which POSTs the `integration-testing/test.cdx` file to the local OutbackCDX instance.

## Docker Build Args

| Arg | Effect |
|---|---|
| `API_VERSION` | Passed as `ENV API_VERSION` into the container. Set in `hooks/build` from `$DOCKER_TAG`. |
| `http_proxy` / `https_proxy` | Used during pip install inside the container (for corporate proxy environments). |
