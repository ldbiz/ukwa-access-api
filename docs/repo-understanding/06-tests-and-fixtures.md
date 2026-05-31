# 06 — Tests and Fixtures

## Test Framework

There is **no unit or integration test suite** in this repository.  
No test runner (pytest, unittest, etc.) is configured, and no test files exist in `test/`.

The `test/` directory contains only **fixture data files** that the application reads directly at runtime.

## How the App Is Tested (Integration Testing)

Integration testing is done using `docker-compose.yml`, which spins up the full stack:
the API, OutbackCDX, PyWB, a WARC server, a web rendering service (Puppeteer), Cantaloupe (IIIF), and Kafka.

The workflow is:

1. Start the stack with `docker-compose up`.
2. Populate the CDX server with WARC index data using `integration-testing/populate-ocdx.sh`.
3. Exercise endpoints manually (e.g. via `curl`, a browser, or Swagger UI at `http://localhost:8000/docs`).

## Fixtures and Sample Data

### `test/data/fc.crawled.json`

- **Used by**: `GET /crawls/fc/recent-activity`
- **Content**: A crawl statistics summary including HTTP status code distribution, MIME types, hosts crawled, and the last-crawl timestamp.
- **At runtime**: Read directly from disk by the crawls router. If `ANALYSIS_SOURCE_FILE` is not set, this file is used in both development and production.
- **Represents**: A real (anonymised) snapshot of frequent-crawl statistics from the UKWA crawl system.

### `test/data/collections/4388.json`

- **Used by**: `GET /collections/download/4388`
- **Content**: A minimal collection record with fields including `id`, `name`, `description`, `start_date`, `children`, and `publish` status.
- **At runtime**: Read from disk by the collections router. The directory is configurable via `JSON_DIR`.
- **Represents**: The "American Football" collection — a single real collection entry from the UKWA collections catalogue.

### `integration-testing/test.cdx`

- **Used by**: `integration-testing/populate-ocdx.sh`
- **Content**: A CDX file generated from the WARC files in `integration-testing/webarchive/`.
- **Represents**: Index entries for WARC captures used in end-to-end testing of memento resolution, WARC retrieval, and screenshot generation.

### `integration-testing/webarchive/`

- **Content**: A PyWB-compatible web archive directory including WARC files and PyWB configuration (`config.yaml`).
- **Used by**: The local PyWB container (access control) and WARC server (WARC streaming) in the compose stack.

## Coverage Gaps

- **No unit tests** exist for any module, endpoint, utility function, or PWID parsing logic.
- The access-control logic (`can_access`), CDX lookup (`lookup_in_cdx`, `list_from_cdx`), WARC streaming (`get_warc_stream`), and PWID encode/decode (`gen_pwid`, `parse_pwid`) have no automated test coverage.
- The IIIF module's fallback regex routing and callback chain are only testable through a full running stack.
- Prometheus metrics behaviour is not tested.
