# 09 — Caveats and Unknowns

## Legacy Code (`api.py` / `access_api/`)

- `api.py` is a full Flask/flask-restx application with overlapping functionality (CDX lookup, Kafka-based URL saving, screenshot serving).
- `access_api/` contains its templates, static assets, and helper modules.
- These files are **not imported by anything in `ukwa_api/`** and appear to be the previous implementation.
- It is unclear whether `api.py` is used in any other deployment configuration, or whether it is safe to remove.

## Nominations Module

- The nominations router is fully implemented with a SQLite-backed database, CRUD endpoints, RSS feed, and Prometheus counter.
- It is commented out in `ukwa_api/main.py` with the comment "needs a bit more work before going live".
- It is not clear what specific work remains, whether it has been partially deployed anywhere, or whether it is being actively developed.

## Access Control Semantics

- `can_access` calls the Wayback server with the raw URL and treats any 4xx/5xx response as "access denied."
- It is not clear what the Wayback server checks (IP-based, rights metadata, embargo date, etc.) or how it signals restricted vs. blocked vs. not-found content.
- The error propagation means that a 404 from Wayback (URL not in the archive) and a 403 (access restricted) are both propagated to the API caller as-is.

## CDX Query Protocol (Legacy XML vs. CDX Proxy)

- `ukwa_api/mementos/router.py` proxies CDX queries using the standard CDX HTTP API (CDX11 or JSON output).
- `ukwa_api/cdx.py` uses an older XML-based query format (`?q=type:urlquery+url:...`) for internal WARC lookups.
- These appear to target different endpoint paths on the same CDX server (`/data-heritrix` vs. the XML endpoint). It is not confirmed whether both paths are always available on the same OutbackCDX instance.

## Screenshot Cache Persistence

- The filesystem screenshot cache is stored in `CACHE_FOLDER/screenshot_cache/`.
- The cache is not mounted as a persistent Docker volume in `docker-compose.yml`.
- It is unclear whether the production deployment uses a persistent volume for the cache, and whether cache loss on container restart is acceptable.

## Cantaloupe Callback Architecture

- Cantaloupe calls back to the API's `/iiif/render_raw` endpoint using its `HTTPSOURCE_BASICLOOKUPSTRATEGY_URL_PREFIX`.
- This callback URL is configured in `docker-compose.yml` as `http://api:8000/iiif/render_raw?pwid=`.
- In production, the Cantaloupe configuration is not present in this repo. It is unknown whether the callback URL is correctly configured in the production environment or how it handles HTTPS and path prefixes.

## PWID Timestamp Bug

- `ukwa_api/pwid.py` (`gen_pwid`) converts the 14-digit Wayback timestamp to an ISO timestamp but appears to use the hours digit twice for both hours and minutes:
  ```python
  iso_ts = f"{yy1}{yy2}-{MM}-{dd}T{hh}:{hh}:{ss}Z"
  ```
  The same is present in the legacy `api.py`. This may be intentional (seconds precision only) or may be a latent bug, but it has not been confirmed by a maintainer.

## `api_out_of_use.py`

- This file is referenced by its name but not listed in the main directory view. It likely corresponds to `api.py` which is marked in the README as `api_out_of_use.py` by convention.
- Its presence in the repository is ambiguous — it may have been retained for reference or migration purposes.

## No Automated Tests

- There are no unit tests, integration test scripts, or CI test steps in this repository.
- The CI workflow (`.github/workflows/push-to-docker-hub.yml`) only builds and pushes the Docker image — no tests are run.
- All correctness guarantees depend on manual integration testing with the Docker Compose stack.

## Database in Production

- `SQLALCHEMY_DATABASE_URL` defaults to a local SQLite file (`./sql_app.db`).
- The `get_db` dependency is injected into the FastAPI app globally, meaning a DB session is created for every request even though no active router uses it.
- Whether a persistent database (e.g. PostgreSQL) is used in production is unknown from this repo alone.

## `api-data/arks.txt`

- The `api-data/` directory contains a single file `arks.txt`.
- Its purpose and how it is used by the application (if at all) is not clear from the code.
