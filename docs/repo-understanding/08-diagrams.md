# 08 — Diagrams

## 1. Runtime Flow

```mermaid
flowchart TD
    Client([Client])
    Gunicorn[Gunicorn]
    UW[UkwaApiWorker\nUvicorn ASGI]
    FastAPI[FastAPI App\nukwa_api/main.py]

    subgraph Routers
        M[/mementos]
        I[/iiif]
        C[/crawls]
        Co[/collections]
    end

    subgraph External Services
        CDX[(CDX Server\nOutbackCDX)]
        WB[(Wayback\nPyWB)]
        WS[(WARC Server\nWebHDFS)]
        WR[(Web Render\nPuppeteer)]
        IIIF[(IIIF Server\nCantaloupe)]
    end

    subgraph Disk
        CJ[fc.crawled.json]
        CL[collections/*.json]
        SC[screenshot_cache/]
    end

    Client --> Gunicorn --> UW --> FastAPI
    FastAPI --> M & I & C & Co

    M -->|access check| WB
    M -->|CDX query or proxy| CDX
    M -->|WARC byte-range| WS

    I -->|access check| WB
    I -->|image proxy| IIIF
    IIIF -->|render_raw callback| I
    I -->|render screenshot| WR
    I <-->|cache read/write| SC

    C --> CJ
    Co --> CL
```

---

## 2. Module Interaction Diagram

```mermaid
graph TD
    main["ukwa_api/main.py\n(app factory)"]

    main --> mementos["mementos/router.py"]
    main --> iiif["iiif/router.py"]
    main --> crawls["crawls/router.py"]
    main --> collections["collections/router.py"]
    main --> deps["dependencies.py\n(DB session)"]
    main --> worker["worker.py\n(Gunicorn config)"]

    mementos --> cdx["cdx.py\n(can_access, lookup_in_cdx,\nlist_from_cdx, get_warc_stream)"]
    mementos --> pwid["pwid.py\n(gen_pwid)"]
    mementos --> mschemas["mementos/schemas.py\n(path params, enums)"]

    iiif --> cdx
    iiif --> pwid
    iiif --> mschemas

    nominations["nominations/\n(router, crud, models,\nschemas, rss)"]
    nominations --> deps
    main -.->|commented out| nominations
```

---

## 3. IIIF Screenshot Request — Sequence Diagram

```mermaid
sequenceDiagram
    actor Client
    participant API as UKWA API\n/iiif/2/{pwid}/...
    participant Cantaloupe as IIIF Server\n(Cantaloupe)
    participant Wayback as Wayback\n(access check)
    participant Renderer as Web Render\n(Puppeteer)

    Client->>API: GET /iiif/2/{pwid}/{region}/{size}/...
    API->>API: parse_pwid(pwid)
    API->>Wayback: GET /wayback/archive/{url} (access check)
    Wayback-->>API: 200 OK (or 4xx/5xx)
    API->>Cantaloupe: GET /iiif/2/{pwid}/{region}/{size}/...
    Cantaloupe->>API: GET /iiif/render_raw?pwid=...
    API->>API: check screenshot_cache
    alt Cache miss
        API->>Renderer: GET /render?url={url}&show_screenshot=True&target_date={ts}
        Renderer-->>API: PNG image
        API->>API: store in screenshot_cache (1h TTL)
    end
    API-->>Cantaloupe: PNG image
    Cantaloupe-->>API: transformed image (cropped, resized, etc.)
    API-->>Client: image response
```
