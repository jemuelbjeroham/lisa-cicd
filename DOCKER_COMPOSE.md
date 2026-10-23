# Local Docker Compose deployment

This deployment runs the LISA API, browser UI, network tools, PostgreSQL, and
Redis as local Linux containers. The API image contains LISA and
`lisa-mcp-server`; it starts the MCP server as a subprocess. There is no
separate MCP container.

Run the commands below from the `lisa-cicd` directory unless otherwise noted.
The Compose file uses sibling repositories as build contexts:

- API: `..` (the common parent containing `lisa-api`, `LISA`, and
  `lisa-mcp-server`)
- UI: `../lisa-ui`
- Network tools: `../net-tools-app`, with `../net-ops-kb-rag` supplied as the
  named BuildKit context `netops`

Docker Compose and BuildKit must support `additional_contexts` (Docker Compose
2.17 or newer is recommended).

## Prepare local configuration

```sh
cp .env.example .env
chmod 600 .env
```

Edit `.env` and replace every required placeholder. Compose reports an explicit
error if `POSTGRES_PASSWORD`, `LISA_DATABASE_URL`, `OPENAI_API_KEY`, or
`TAVILY_API_KEY` is missing or empty.

`POSTGRES_PASSWORD` is the raw password PostgreSQL uses. The password in
`LISA_DATABASE_URL` must be the same password, percent-encoded for URL use.
Compose does not URL-encode credentials for you. For example, encode `@` as
`%40`, `:` as `%3A`, `/` as `%2F`, and `%` as `%25` in the URL password
component. Quote `.env` values containing characters that Compose treats as
syntax (such as `$`, `#`, or whitespace); use Compose's supported `.env`
quoting rules.

Do not copy credentials from another repository's `.env` file. Use your own
authorized keys and keep `.env` private and untracked. The `.env.example`
contains placeholders only.

## Validate configuration and build images

First validate interpolation and Compose syntax without starting containers:

```sh
docker compose --env-file .env config --quiet
```

This checks the Compose model, not image builds, application behavior, database
migrations, or external credentials. Do not share the expanded output of
`docker compose config`, because it can contain interpolated secrets.

Build the local images without starting services:

```sh
docker compose --env-file .env build
```

The deployment builds locally and does not assume that application images are
published to GHCR.

## Storage and initial Chroma data

Compose creates four named volumes:

- `postgres-data` for PostgreSQL at `/var/lib/postgresql/data`
- `redis-data` for Redis at `/data`
- `net-tools-chroma` for Chroma at
  `/workspace/net-ops-kb-rag/storage/chroma`
- `huggingface-cache` for model files at `/home/app/.cache/huggingface`

No existing RAG dataset is copied or mounted into the network-tools service.
The Chroma volume starts empty unless it already contains restored data.
Network tools runs as UID/GID `10001:10001`; the image creates both storage
directories with that ownership before Compose mounts the named volumes.
Preserve writable ownership for this UID/GID when restoring data.

To initialize the collection from the checked-out `net-ops-kb-rag/sample_data`,
run the ingestion script as a one-off container after the images are built.
The source checkout is mounted read-only; the service's named Chroma and
Hugging Face volumes remain mounted at their configured paths:

```sh
docker compose --env-file .env run --rm --no-deps \
  --workdir /workspace/net-ops-kb-rag \
  --volume "$PWD/../net-ops-kb-rag:/workspace/net-ops-kb-rag:ro" \
  net-tools-app python scripts/ingest_documents.py
```

This command intentionally ingests the sample data from the local checkout.
For a deployment-specific corpus, follow the RAG project's ingestion process
with the approved source data. To restore a Chroma backup instead, restore it
into the `net-tools-chroma` volume using a trusted procedure and retain
ownership writable by UID/GID `10001:10001`. Do not remove or replace volumes
as a troubleshooting shortcut.

The first network-tools startup may download the embedding model from Hugging
Face and take several minutes. Its `/health` check becomes healthy after the
application has started. The API does not wait for network-tools health, so
model download does not block API startup.

## Start services and migrate the database

Start only PostgreSQL and Redis first, then wait for both health checks:

```sh
docker compose --env-file .env up -d postgres redis
docker compose --env-file .env ps
```

Apply the migrations before starting the API. Its image does not contain the
Alembic configuration or migration tree, so the one-off command mounts the
existing `LISA` checkout read-only at `/migration`. It uses the API image and
the API service's configured environment, including the required model,
Tavily, and database settings:

```sh
docker compose --env-file .env run --rm --no-deps \
  --volume "$PWD/../LISA:/migration:ro" \
  lisa-api alembic -c /migration/alembic.ini upgrade head
```

The database must be healthy before running this command. `LISA/migrations/env.py`
loads `Settings`, so the one-off migration also needs `OPENAI_API_KEY` and
`TAVILY_API_KEY` even though the migration is not a model request.

Create the first administrator explicitly and interactively; no administrator
is created automatically:

```sh
docker compose --env-file .env run --rm --no-deps -it \
  lisa-api lisa-api-bootstrap-admin
```

Then start the API and UI. Add `net-tools-app` if you want its endpoint running;
it is included when starting the full project:

```sh
docker compose --env-file .env up -d
docker compose --env-file .env ps
```

## Service layout and URLs

| Service | Container port | Host access | Startup/readiness |
|---|---:|---|---|
| `postgres` | 5432 | Not published | `pg_isready` |
| `redis` | 6379 | Not published | `redis-cli ping`; AOF and periodic snapshots enabled |
| `net-tools-app` | 8000 | Not published | `GET /health`; may wait for the first model download |
| `lisa-api` | 8008 | `127.0.0.1:8008` | `GET /openapi.json`; basic process readiness only |
| `lisa-ui` | 3000 | `127.0.0.1:3000` | Depends on API process-readiness check |

Services use Compose's private default network. Inside it, the API uses
`postgres:5432`, `redis:6379`, and `http://net-tools-app:8000`. The browser
uses `LISA_API_URL=http://localhost:8008`; do not set it to the internal
`lisa-api` service name. The API CORS origin defaults to
`http://localhost:3000`.

The API health check only requests `/openapi.json`. A passing check does not
prove that the PostgreSQL schema is migrated or that database operations
succeed. The migration command and application-level checks are separate
requirements.

## Optional Langfuse

Langfuse is disabled when its public or secret key is absent. The Compose
service does not add or manage a Langfuse stack. To enable tracing, provide
both `LANGFUSE_PUBLIC_KEY` and `LANGFUSE_SECRET_KEY` and set
`LANGFUSE_BASE_URL` to a URL verified reachable **from the API container**.
The default `http://localhost:3001` points back to the API container and is not
a verified route to the existing local Langfuse instance. Do not modify or
restart that separate stack as part of this deployment.

## Validate the running deployment

After you have reviewed the commands and chosen to start the stack:

```sh
docker compose --env-file .env ps
docker compose --env-file .env logs --tail=100 postgres redis net-tools-app lisa-api lisa-ui
curl --fail http://localhost:8008/openapi.json
curl --fail http://localhost:3000/
curl --fail http://localhost:3000/api/runtime-config
```

The runtime-config response should contain `http://localhost:8008`. Check
network-tools readiness separately:

```sh
docker compose --env-file .env exec net-tools-app \
  python -c "import urllib.request; print(urllib.request.urlopen('http://127.0.0.1:8000/health').read().decode())"
```

These checks establish basic HTTP reachability, not successful chat completion,
database migration state, external model credentials, Chroma corpus contents,
or browser CORS behavior. Sign in and test the intended workflows after
confirming those prerequisites.

## Shutdown

Stop and remove the Compose containers and network while preserving all named
volumes:

```sh
docker compose --env-file .env down
```

Do not add `--volumes`/`-v` unless you explicitly intend to delete persistent
database, Redis, Chroma, and model-cache data.

## Troubleshooting

- **Missing-variable error:** Copy `.env.example` to `.env` and set the four
  required values. Run Compose from `lisa-cicd` with `--env-file .env`.
- **PostgreSQL authentication error:** Make `POSTGRES_PASSWORD` and the
  password in `LISA_DATABASE_URL` match. URL-encode reserved characters only
  in the URL password. Existing PostgreSQL data volumes retain the credentials
  they were initialized with; changing `.env` does not rewrite an existing
  database user's password.
- **API fails to start:** Check Redis health and API logs. Confirm migrations
  have been applied and the API keys and database URL are valid.
- **API returns an OpenAPI document but database requests fail:** The API
  health check is only process readiness. Run/check Alembic migration state
  and database connectivity separately.
- **Network tools is slow or unhealthy:** A cold cache requires outbound
  Hugging Face access. Inspect its logs and retain the `huggingface-cache`
  volume. The `net-tools-chroma` volume must contain indexed data for searches
  to return results.
- **Browser cannot reach the API:** Use `http://localhost:8008` for
  `LISA_API_URL` when the browser runs on the same machine, and ensure
  `LISA_CORS_ALLOWED_ORIGINS` matches the UI origin.
- **Build cannot resolve `netops`:** Check that the sibling
  `../net-ops-kb-rag` checkout exists and that Docker Compose/BuildKit supports
  named additional build contexts.
