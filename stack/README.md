# SearXNG stack (custom/main branch)

Single-host SearXNG deployment packaged for our Firecrawl stack.
Sits next to `firecrawl-custom` and joins the same docker `backend`
network so Firecrawl can call `http://searxng-core:8080` without
exposing SearXNG to the host.

## Layout

- `docker-compose.yml` — `searxng-core` (image) + `searxng-valkey`
  (Valkey/Redis-fork for limiter/bot detection).
- `.env.example` — template; copy to `.env` and set `SEARXNG_SECRET`.
- `core-config/settings.yml` — `use_default_settings: true` plus 5
  curated engines (google, bing, duckduckgo, brave, startpage).

## Prerequisites

Firecrawl stack already running and managing a docker network named
`backend`. Verify:

```bash
docker network ls | grep backend
```

If absent, start Firecrawl first (`~/.services/firecrawl-custom`).

## First-time setup

```bash
cd ~/.services/searxng                # or wherever you put this stack
cp .env.example .env
sed -i "s/replace_with_openssl_rand_hex_32/$(openssl rand -hex 32)/" .env
docker compose up -d
docker compose logs -f searxng-core    # wait for "Server running on"
```

## Sanity tests

```bash
# From inside the Firecrawl api container:
docker exec firecrawl-api-1 \
  curl -sS 'http://searxng-core:8080/search?q=firecrawl&format=json' \
  | python3 -c 'import sys,json; print(len(json.load(sys.stdin)["results"]))'
# Expect: >= 5 results

# Trigger Firecrawl's /v2/search after restarting it with SEARXNG_ENDPOINT set:
cd ~/.services/firecrawl-custom
docker compose restart api
curl -sS -X POST http://localhost:3002/v2/search \
  -H 'Content-Type: application/json' \
  -d '{"query":"what is firecrawl","limit":5}' | jq '.data | length'
```

## Firecrawl side

In `~/.services/firecrawl-custom/.env`:

```
SEARXNG_ENDPOINT=http://searxng-core:8080
SEARXNG_ENGINES=google,bing,duckduckgo,brave,startpage
SEARXNG_CATEGORIES=general
```

Then `docker compose restart api playwright-service`.

## Notes

- SearXNG does not expose port 8080 on the host. Only containers on
  the `backend` network can reach it.
- Engines list mirrors `SEARXNG_ENGINES` in Firecrawl's `.env` —
  keep them in sync.
- Engines may rotate under heavy load to avoid IP rate-limits; this
  is upstream behaviour, not configurable here.
