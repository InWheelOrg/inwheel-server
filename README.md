# inwheel-server

Backend for the **InWheel** accessibility platform. A global registry of wheelchair accessibility data for physical places.

Licensed under [AGPL-3.0](./LICENSE).

## Overview

inwheel-server is a Go backend composed of three services:

| Service | Path | Description |
|---|---|---|
| `api` | `cmd/api` | Public REST API for places and accessibility profiles |
| `auditor` | `cmd/auditor` | Background AI worker that validates accessibility data |
| `ingestion` | `cmd/ingestion` | OSM data ingestion worker (in development) |

Data comes from two sources: **OpenStreetMap (OSM)** sync and **direct user contributions**. Because these sources can contradict each other, an asynchronous AI auditor flags logical conflicts without blocking writes or modifying user data.

## Architecture

```
Client
  |
  v
[API server]  ──write──>  [PostgreSQL + PostGIS]
                                    |
                          needs_audit = true
                                    |
                                    v
                          [Auditor worker]  <──>  [Ollama (local LLM)]
```

The API is intentionally kept low-latency: all writes are saved immediately and marked `needs_audit = true`. The auditor worker picks up tasks asynchronously, queries a local Small Language Model (SLM) via Ollama, and writes the result back without touching the original data.

## Data Model

**Place** — a physical location with coordinates, category, OSM metadata, and a hierarchy (e.g. a shop can have a mall as its parent).

**AccessibilityProfile** — attached to a place; contains:
- `overall_status`: `accessible` | `limited` | `inaccessible` | `unknown`
- `components`: structured data per feature type — `entrance`, `restroom`, `parking`, `elevator`
- `audit`: AI findings (`has_conflict`, `reasoning`, `confidence`)

Child places inherit accessibility components from their parent for any component they don't own directly.

## API Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/places` | List places. Supports proximity (`lat`, `lng`, `radius`) and bounding box (`min_lng`, `min_lat`, `max_lng`, `max_lat`) queries |
| `GET` | `/places/{id}` | Get a single place with its accessibility profile |
| `POST` | `/places` | Create a place (with optional accessibility data) |
| `PATCH` | `/places/{id}/accessibility` | Update or create an accessibility profile |

## Running with Docker Compose

Copy and fill in the required environment variables:

```sh
cp .env.example .env  # set DB_USER, DB_PASSWORD, DB_NAME
```

Start all services:

```sh
docker compose up
```

The API will be available at `http://localhost:8080`. Ollama will automatically pull the configured model on first start.

To use a different model:

```sh
AUDIT_MODEL=qwen2.5:1.5b docker compose up
```

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `PORT` | `8080` | API server port |
| `DB_HOST` | `localhost` | PostgreSQL host |
| `DB_PORT` | `5432` | PostgreSQL port |
| `DB_USER` | `postgres` | Database user |
| `DB_PASSWORD` | `postgres` | Database password |
| `DB_NAME` | `inwheel` | Database name |
| `DB_SSLMODE` | `disable` | PostgreSQL SSL mode |
| `OLLAMA_URL` | `http://ollama:11434` | Ollama API URL |
| `AUDIT_MODEL` | `deepseek-r1:1.5b` | Ollama model for the auditor |

## Development

```sh
go test ./...
```

Requires a running PostgreSQL instance with the PostGIS extension enabled.
