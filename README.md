# HealthAccess

HealthAccess is a patient-controlled emergency health summary and care
navigation service. A patient bsbdnchooses a small emergency profile, can share it by
a short code, and can find suitable care. SMS and USSD keep the same access
available on a basic phone without mobile data.

This repository is a TypeScript monorepo with three workspaces:

- `backend/online` - Express 5 + Prisma API for the patient portal, grants and
  consent, directory, symptom chat, medication tracking and reminders.
- `backend/offline` - independent SMS/USSD service for a basic phone. It shares
  the database and crypto contract but never calls the online API.
- `backend/shared` - shared crypto, access policy, projections and types.

PostgreSQL 16 is the single database. `backend/online/prisma/schema.prisma` is
the source of truth. The old SQL schemas, routines and FastAPI plan are retired.

## Status

The backend feature phases are implemented and verified: the Prisma schema and
three migrations exist, the online API boots with health checks, and the seed
creates reference data plus an optional sandbox patient (no demo facilities). The
online API covers auth/users, portal/grants, approved plus live Serper directory
results, emergency requests, symptom chat/AI with SSE streaming and medications
with reminders; the offline SMS/USSD service is Prisma-backed and uses live
Serper emergency/care search when the directory has no approved non-demo result.
Workspace typecheck is clean, the shared suite passes 4/4, offline unit tests
85/85, offline Prisma integration 20/20, and the online DB suite 108/108. Live
FCM delivery, real Africa's Talking provisioning and production monitoring remain
unverified. There is no frontend application yet. See
[`docs/README.md`](docs/README.md).

## Repository map

```text
backend/
  shared/          @healthaccess/shared crypto/policy/projection/types
  online/          @healthaccess/online Express 5 API and Prisma schema
    prisma/        schema, migrations, seed
    src/           app, routes, middleware, modules, services
  offline/         @healthaccess/offline SMS/USSD HTTP process and worker
    src/           callbacks, channels, repository, worker
    tests/         unit and Prisma DB integration tests
docs/              architecture, database, operations, integration plan
frontend/          placeholder only
plans/             retired HTML planning documents
docker-compose.yml deploys online, offline (server + worker), docs, PostgreSQL and Redis; S3/MinIO is external
.github/workflows/ unit/typecheck CI
```

## Quick start

Prerequisites: Node 24+, npm 11+. For direct npm runs you need a PostgreSQL 16
instance; Redis and S3/MinIO are optional. The Docker section below includes
PostgreSQL and Redis, so you only need those externally if you run the services
without Docker.

The commands below run the services directly with npm. For the compose stack,
skip to [Docker deployment](#docker-deployment).

```sh
npm install

cp backend/online/.env.example backend/online/.env
cp backend/offline/.env.example backend/offline/.env
# Fill required values and point them at your PostgreSQL/Redis/S3 instances.
# Never commit .env. JWT_SECRET, CODE_DIGEST_KEY and DATA_ENCRYPTION_KEY are
# required, and CODE_DIGEST_KEY / DATA_ENCRYPTION_KEY must match in both files.

npm run db:migrate
npm run db:seed
npm run dev:online
```

The online API listens on `http://localhost:3000`; check `/health/live` and
`/health/ready`. The versioned route prefix is `/v1`.

The offline service runs two processes:

```sh
npm run dev:offline
npm run dev:worker
```

The offline service uses the same Prisma schema and migrations as online, so no
second schema or `db:apply` step is needed. Its full DB integration run needs a
disposable database; see [`docs/operations.md`](docs/operations.md).

## Environment templates

Copy each block into the matching file. The root `.env` is optional and only
customizes the compose stack; the two backend files are required.

### Root `.env` (optional, compose only)

```dotenv
# Change before the first real deployment.
POSTGRES_PASSWORD=healthaccess
TZ=Africa/Lagos
ONLINE_PORT=3000
OFFLINE_PORT=3001
DOCS_PORT=8080
```

### `backend/online/.env`

```dotenv
# Leave empty for Docker: compose supplies the included Postgres.
# Set only for direct npm runs.
DATABASE_URL=
CODE_DIGEST_KEY=
DATA_ENCRYPTION_KEY=
PHONE_DIGEST_KEY=
INPUT_FINGERPRINT_KEY=
ENCRYPTION_KEY_VERSION=1

JWT_SECRET=
JWT_ACCESS_TTL_SECONDS=900
REFRESH_TTL_DAYS=30

GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
BASE_URL=http://localhost:3000
FRONTEND_URL=http://localhost:5173

AWS_REGION=
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_BUCKET_NAME=
S3_ENDPOINT=
S3_FORCE_PATH_STYLE=false

# Leave empty for Docker: compose supplies redis://redis:6379.
# Set only for direct npm runs.
REDIS_URL=redis://localhost:6379

OPENROUTER_API_KEY=
OPENROUTER_MODEL=deepseek/deepseek-chat
DEEPSEEK_API_KEY=
GOOGLE_API_KEY=
ANTHROPIC_API_KEY=
GROQ_API_KEY=
SERPER_API_KEY=
CHAT_MODEL=openrouter
PRESCRIPTION_MODEL=haiku
MEDICATION_SCHEDULER_MODEL=openrouter

FIREBASE_SERVICE_ACCOUNT=
FCM_ENABLED=false

AT_USERNAME=sandbox
AT_API_KEY=
AT_SMS_FROM=

NODE_ENV=development
PORT=3000
LOG_LEVEL=info
```

### `backend/offline/.env`

```dotenv
# Leave empty for Docker: compose supplies the included Postgres.
# Set only for direct npm runs.
DATABASE_URL=
CODE_DIGEST_KEY=
DATA_ENCRYPTION_KEY=
ENCRYPTION_KEY_VERSION=1

CALLBACK_ALLOWED_IPS=
CALLBACK_TOKEN=

AT_USERNAME=sandbox
AT_API_KEY=
AT_SMS_FROM=
AT_USSD_SERVICE_CODE=

OPENROUTER_API_KEY=
OPENROUTER_MODEL=
SERPER_API_KEY=

NODE_ENV=development
PORT=3001
# off | redacted | full. Full logs callback bodies; local debugging only.
LOG_HTTP_BODIES=full
```

Shared rules:

- `CODE_DIGEST_KEY`, `DATA_ENCRYPTION_KEY` and `ENCRYPTION_KEY_VERSION` must
  match across both backend files. Either set `PHONE_DIGEST_KEY` and
  `INPUT_FINGERPRINT_KEY` identically in both, or leave them empty in both.
- `JWT_SECRET` is online only.
- For Docker, leave `DATABASE_URL` and `REDIS_URL` empty. Compose sets
  `postgresql://healthaccess:<POSTGRES_PASSWORD>@postgres:5432/healthaccess` and
  `redis://redis:6379`. Their values only matter for direct `npm` runs.
- `NODE_ENV=development` uses sandbox/mock SMS. `production` requires live
  Africa's Talking credentials, `AT_SMS_FROM`, `AT_USSD_SERVICE_CODE` and
  `CALLBACK_ALLOWED_IPS`.
- Generate keys with `cd backend/offline && npm run keys:generate`.

## Docker deployment

`docker-compose.yml` builds one backend image and runs:

| Service | Default port | Purpose |
| --- | --- | --- |
| `postgres` | internal 5432 | PostgreSQL 16 with the `healthaccess_pgdata` volume |
| `redis` | internal 6379 | Redis 7 with AOF persistence |
| `migrate` | - | one-shot `prisma migrate deploy`, runs first |
| `online` | 3000 | Express API and `/docs` |
| `offline` | 3001 | SMS/USSD callback server |
| `offline-worker` | - | SMS outbox and conversation worker |
| `docs` | 8080 | standalone API reference with a same-origin `/v1` proxy |

PostgreSQL and Redis are included and are not published on the host by default.
S3/MinIO is not included; set its details in `backend/online/.env` if you use
uploads. The compose sets `DATABASE_URL` and `REDIS_URL` for the backend
containers, so the `localhost` values in the env files only matter when you run a
service directly with npm. The backend env files must still agree on
`CODE_DIGEST_KEY`, `DATA_ENCRYPTION_KEY` and `ENCRYPTION_KEY_VERSION`.

The default Postgres password is `healthaccess`. For anything beyond local
testing, copy `.env.example` to `.env` and set `POSTGRES_PASSWORD` before the
first start. Changing it after the database volume exists requires resetting
that volume. Use a URL-safe password or URL-encode it in `DATABASE_URL`. The
included Postgres uses its owner account; a production deployment should use a
least-privilege role or an external managed database.

### Which env file is used?

| Container | Reads | Main values |
| --- | --- | --- |
| `online`, `migrate` | `backend/online/.env` | JWT, S3, AI, `FRONTEND_URL`; DB/Redis come from compose |
| `offline`, `offline-worker` | `backend/offline/.env` | callback token, Africa's Talking, AI; DB comes from compose |
| `postgres` | compose | `POSTGRES_PASSWORD` |
| `redis` | compose | no password; internal network only |
| `docs` | none | static page plus the `/v1` proxy |
| Compose itself | optional root `.env` | `POSTGRES_PASSWORD`, `${TZ}`, ports |

A root `.env` is not passed wholesale into the containers; it only supplies the
values above for Compose. The two backend env files still hold the JWT, crypto,
Africa's Talking, S3 and AI settings. If you replace the included Postgres/Redis
with external services, change the compose `DATABASE_URL`/`REDIS_URL` to a host
the containers can reach.

Each backend container reads `NODE_ENV` from its own env file. The examples ship
with `development` (sandbox/mock SMS). For a live deployment, set
`NODE_ENV=production` in both env files; that requires live Africa's Talking
credentials (`AT_USERNAME` not `sandbox`, `AT_API_KEY`), the provisioned
`AT_SMS_FROM` and `AT_USSD_SERVICE_CODE`, and `CALLBACK_ALLOWED_IPS`.

```sh
docker compose up -d --build
docker compose run --rm online npm run db:seed   # optional seed and sandbox patient
docker compose ps
docker compose logs -f online
```

Then open:

- `http://localhost:3000/v1` for the API
- `http://localhost:3000/docs` for the reference served by the API
- `http://localhost:8080` for the standalone docs site (Try It proxies `/v1` to `online`)
- `http://localhost:3001` for the Africa's Talking callbacks

From another machine, replace `localhost` with the server IP or domain. The
compose publishes plain HTTP; put your reverse proxy/TLS in front for production.

Override published ports with `ONLINE_PORT`, `OFFLINE_PORT`, `DOCS_PORT` and the
timezone with `TZ` (default `Africa/Lagos`). A new migration rebuilds the image
and re-runs the `migrate` service on `up --build`.

`docker compose down` stops the stack; `docker compose down -v` also deletes the
Postgres and Redis volumes.

### On a cloud server

```sh
# If Docker is missing:
# curl -fsSL https://get.docker.com | sh
# sudo usermod -aG docker "$USER"   # then log out and back in

git clone <your-repo-url> healthaccess
cd healthaccess

cp .env.example .env
cp backend/online/.env.example backend/online/.env
cp backend/offline/.env.example backend/offline/.env
# Edit the three files. Leave DATABASE_URL and REDIS_URL empty for Docker.
```

Generate the secrets on the server:

```sh
openssl rand -hex 24     # POSTGRES_PASSWORD (URL-safe)
openssl rand -base64 32  # CODE_DIGEST_KEY
openssl rand -base64 32  # DATA_ENCRYPTION_KEY (must differ)
openssl rand -hex 32     # JWT_SECRET
openssl rand -hex 32     # CALLBACK_TOKEN
```

Put the same `CODE_DIGEST_KEY`, `DATA_ENCRYPTION_KEY` and
`ENCRYPTION_KEY_VERSION` in both backend env files. Then start:

```sh
docker compose up -d --build
docker compose run --rm online npm run db:seed   # optional demo data
docker compose ps
docker compose logs -f online
```

Open the ports in the cloud security group and the host firewall:

```sh
sudo ufw allow 3000/tcp   # API
sudo ufw allow 3001/tcp   # Africa's Talking callbacks
sudo ufw allow 8080/tcp   # docs plus same-origin /v1
```

- API: `http://SERVER_IP:3000/v1`
- Docs: `http://SERVER_IP:8080`
- Offline callbacks: `http://SERVER_IP:3001`

For production, put Nginx or Caddy on 80/443 and set `BASE_URL` and
`FRONTEND_URL` to the public domains. The included Postgres and Redis are not
published; connect with
`docker compose exec postgres psql -U healthaccess -d healthaccess`.

Back up the database and keep the env files and crypto keys safe:

```sh
docker compose exec -T postgres pg_dump -U healthaccess healthaccess > healthaccess.sql
```

`docker compose down` stops the stack; `down -v` deletes the Postgres and Redis
volumes. If `npm ci`/the build runs out of memory on a small VPS, add swap or
build the images on a larger machine and push them to a registry.

For keys, environment details, sandbox callbacks and troubleshooting, read
[`docs/operations.md`](docs/operations.md).

## Seed and sandbox data

`npm run db:seed` is idempotent and creates reference data plus an optional
sandbox patient:

- 15 field definitions, including emergency-eligible name, date of birth,
  allergies, current medications, conditions, blood group and emergency contact.
- 3 services: `general`, `pediatrics`, `maternity`.
- Real area `Ikeja, Lagos` with `is_demo = false`.
- Sandbox patient `demo.patient@example.test` with password `DemoPass123!`,
  patient code `123456789012`, five profile values and emergency fields for
  allergies and emergency contact.
- No demo facilities. The seed removes legacy demo facility rows. Directory and
  care-search results come from approved non-demo facilities plus live Serper
  Maps leads; live leads are labelled unverified and never presented as approved.

The sandbox credentials are fixtures, not production credentials.
`backend/offline/scripts/seed-demo.ts` writes the same reference area and sandbox
patient for offline testing; the online seed is the canonical step.

## Tests

```sh
npm run typecheck
npm run build

# unit tests that do not need a database
npm run test --workspace @healthaccess/shared
npm run test --workspace @healthaccess/offline
```

Current results: typecheck clean, shared 4/4, offline unit 85/85. The offline
Prisma integration suite passes 20/20 against a disposable database. `npm test`
at the root runs every workspace, including the online DB integration suite
(108/108), so it needs a configured database; use the two workspace commands
above for a database-free run.

The online tests under `backend/online/tests/` exercise PostgreSQL and are DB
integration tests, so CI does not run them. Run them locally against a
disposable database:

```sh
cd backend/online
node --env-file-if-exists=.env --test --import tsx tests/*.test.ts
```

The offline DB integration suite needs a disposable database:

```sh
cd backend/offline
TEST_DATABASE_URL='postgresql://<admin>:<password>@localhost:5432/healthaccess_test' npm run test:db
```

Never point integration tests at a live database. See
[`docs/operations.md`](docs/operations.md) for the full command set.

## Documentation

- [`docs/README.md`](docs/README.md) - product, current status, repo map and
  quick start.
- [`docs/architecture.md`](docs/architecture.md) - components, auth, AI,
  reminders, offline flows and security.
- [`docs/database.md`](docs/database.md) - Prisma schema, table groups, queue
  semantics and crypto contract.
- [`docs/operations.md`](docs/operations.md) - environment, local setup, sandbox
  testing, tests and production checklist.
- [`docs/api/index.html`](docs/api/index.html) - Postman-style API reference with
  captured request and response examples. `npm run docs:api` rebuilds it.
- [`docs/integration-plan.md`](docs/integration-plan.md) - parallel-work contract
  and ownership map.
