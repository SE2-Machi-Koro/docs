# Backend Container Deployment

This document verifies the current backend container build and deployment path
for the AAU shared server. It is based on the `SE2-Machi-Koro/Server`
repository files:

- `Dockerfile`
- `compose.yaml`
- `compose-dev.yaml`
- `.github/workflows/docker-publish.yml`
- `.env.example`
- `src/main/resources/application.properties`

## Production Endpoints

| Resource | URL |
| :--- | :--- |
| Backend HTTP | `http://se2-demo.aau.at:53210` |
| Health check | `http://se2-demo.aau.at:53210/actuator/health` |
| WebSocket | `ws://se2-demo.aau.at:53210/ws` |
| Swagger UI | `http://se2-demo.aau.at:53210/swagger-ui.html` |

Expected health response:

```json
{
  "status": "UP"
}
```

## Dockerfile Build Path

The backend `Dockerfile` supports two build paths:

| Target | Purpose |
| :--- | :--- |
| `runtime-from-builder` | Source-based build path. The Docker build runs `./gradlew bootJar -x test` inside the builder stage. |
| `runtime-from-workspace` | CI publish path. The GitHub workflow builds the jar first, copies it to `build/docker/app.jar`, and Docker packages that jar. |

The final default target is `final`, which currently resolves to
`runtime-from-builder`. The GHCR workflow explicitly uses
`target: runtime-from-workspace` to avoid rebuilding the jar once per Docker
architecture.

Manual local image build from the backend repository:

```bash
docker build -t machikoro-server:local .
```

## Compose Files

### `compose.yaml`

`compose.yaml` is the production stack definition. It runs:

- `postgres`, using `postgres:18.0`;
- `backend`, using `ghcr.io/se2-machi-koro/server:${IMAGE_TAG:-latest}`.

The backend service:

- publishes `${PUBLIC_PORT:-53210}:8080`;
- sets `SERVER_PORT=8080` inside the container;
- connects to Postgres through the internal Compose network with
  `DB_HOST=postgres` and `DB_PORT=5432`;
- disables Spring Boot Docker Compose integration with
  `SPRING_DOCKER_COMPOSE_ENABLED=false`;
- defaults `WEBSOCKET_ALLOWED_ORIGINS` to the public AAU origins
  (`http://se2-demo.aau.at:53210,https://se2-demo.aau.at:53210`);
- keeps the debug admin-seeding switches off by default
  (`DEBUG_ENABLED=false`, empty `ADMIN_PASSWORD`);
- bounds the JVM heap to the container limit with
  `JAVA_TOOL_OPTIONS=-XX:MaxRAMPercentage=75.0`;
- pulls the image on refresh through `pull_policy: always`;
- restarts automatically with `restart: unless-stopped`;
- exposes a container health check at
  `http://localhost:8080/actuator/health`.

Both services declare CPU and memory limits under `deploy.resources` (the
backend is capped at 1 CPU / 512 MB, Postgres at 0.5 CPU / 256 MB) so
the stack stays within the shared-server quota.

The Postgres service is not exposed on the production host. It is only reachable
inside the Compose network.

### `compose-dev.yaml`

`compose-dev.yaml` is for local development. It starts Postgres and pgAdmin, and
the backend can then run from source with Gradle.

Local development startup from the backend repository:

```bash
cp .env.example .env
docker compose -f compose-dev.yaml --env-file .env up -d postgres
./gradlew bootRun
```

The old docs command using `compose.local-test.yaml` is outdated for the current
backend repository because the current Compose files are `compose.yaml` and
`compose-dev.yaml`.

## GHCR Publishing and Deployment Workflow

The GitHub Actions workflow `.github/workflows/docker-publish.yml` builds the
backend image, publishes it to GHCR, and deploys it to the AAU server. The image
is published to:

```text
ghcr.io/se2-machi-koro/server
```

Triggers:

- push to `main`;
- git tags matching `v*`;
- manual `workflow_dispatch`.

The workflow runs three sequential jobs:

1. **`build-jar`** checks out the backend repository, sets up JDK 21, and runs:

```bash
./gradlew bootJar -x test
mkdir -p build/docker
cp build/libs/*.jar build/docker/app.jar
```

   The jar is uploaded as a short-lived artifact named `app-jar`.

2. **`build-and-push`** downloads the jar, sets up QEMU and Docker Buildx, logs
   in to GHCR using `GITHUB_TOKEN`, and builds and pushes the Docker image with:

```text
target: runtime-from-workspace
platforms: linux/amd64,linux/arm64
```

3. **`deploy`** runs only on `main`. It SSHes into the AAU server with
   `appleboy/ssh-action` (port `53200`) and refreshes the running backend:

```bash
cd ~/machi-koro-server-deploy
docker compose pull backend
docker compose up -d --no-deps backend
```

Published tags:

| Tag | Meaning |
| :--- | :--- |
| `latest` | Published for `main`. |
| `sha-<short-commit>` | Published for each workflow run and used for rollback. |
| `v*` | Published when a matching version tag is pushed. |
| branch ref tag | Produced by Docker metadata for branch builds. |

The build-and-push job needs no custom GHCR secret: it uses GitHub's built-in
`GITHUB_TOKEN` with `packages: write` permission. The `deploy` job relies on
three repository secrets for the SSH connection:

| Secret | Purpose |
| :--- | :--- |
| `DEPLOY_HOST` | AAU server hostname (`se2-demo.aau.at`). |
| `DEPLOY_USER` | SSH user (`grp-6`). |
| `DEPLOY_SSH_KEY` | Private key authorized for that user. |

## Automatic AAU Deployment

Deployment to the AAU server is fully automated by the `deploy` job in
`.github/workflows/docker-publish.yml`. The end-to-end path is:

1. Merge or push the backend change to `main`.
2. GitHub Actions builds the jar and publishes a new GHCR image
   (`latest` and `sha-<short-commit>`).
3. The `deploy` job SSHes into the AAU server and, from
   `~/machi-koro-server-deploy`, runs `docker compose pull backend` followed by
   `docker compose up -d --no-deps backend`.
4. The backend service pulls
   `ghcr.io/se2-machi-koro/server:${IMAGE_TAG:-latest}` and restarts; Postgres
   is left untouched (`--no-deps`).
5. The deployment is healthy when both containers are healthy and the public
   health endpoint returns `UP`.

> **Note:** Earlier revisions of this stack were reconciled by doco-cd. The
> server `main` branch now deploys via the GitHub Actions SSH job described
> above; doco-cd is no longer part of the active path.

Verify the public service:

```bash
curl -s http://se2-demo.aau.at:53210/actuator/health
```

Expected response:

```json
{"status":"UP"}
```

## Manual Fallback Deployment

Use this when the automated `deploy` job is unavailable (for example, a missing
or rotated SSH secret) or when a manual refresh is needed after a GHCR image was
published.

Connect to the AAU server:

```bash
ssh grp-6@se2-demo.aau.at -p 53200
```

Prepare the deployment directory if it does not already exist:

```bash
mkdir -p /home/grp-6/machi-koro-server-deploy
cp compose.yaml /home/grp-6/machi-koro-server-deploy/compose.yaml
cd /home/grp-6/machi-koro-server-deploy
```

Create or update the production `.env` next to `compose.yaml` and restrict its
permissions:

```bash
chmod 600 .env
```

Minimal production `.env`:

```env
DB_NAME=machikoro
DB_USERNAME=machikoro
DB_PASSWORD=<production-password>
PUBLIC_PORT=53210
WEBSOCKET_ALLOWED_ORIGINS=http://se2-demo.aau.at:53210,https://se2-demo.aau.at:53210
IMAGE_TAG=latest
```

Start or refresh the stack:

```bash
docker compose pull
docker compose up -d
docker compose ps
```

Expected result:

```text
machikoro-db       healthy
machikoro-server   healthy
```

Confirm the public health endpoint:

```bash
curl -s http://se2-demo.aau.at:53210/actuator/health
```

## Required Environment Variables

| Variable | Required in production | Description |
| :--- | :--- | :--- |
| `DB_NAME` | Yes | PostgreSQL database name. |
| `DB_USERNAME` | Yes | PostgreSQL database user. |
| `DB_PASSWORD` | Yes | PostgreSQL password. Must never be committed. |
| `PUBLIC_PORT` | Yes | Host port assigned by AAU. Group 6 uses `53210`. |
| `WEBSOCKET_ALLOWED_ORIGINS` | Yes | Comma-separated allowed browser/client origins. |
| `IMAGE_TAG` | No | GHCR tag to deploy. Defaults to `latest`; set to `sha-<short-commit>` for rollback. |
| `SERVER_PORT` | No | Container server port. Production Compose sets it to `8080`. |
| `DB_HOST` | No | Production Compose sets it to `postgres`. |
| `DB_PORT` | No | Production Compose sets it to `5432`. |
| `DEBUG_ENABLED` | No | Enables debug endpoints and admin-account seeding. Defaults to `false`; keep off in production. |
| `ADMIN_PASSWORD` | No | Password for the seeded admin accounts. Required only when `DEBUG_ENABLED=true`. Must never be committed. |

Local-only variables for `compose-dev.yaml`:

| Variable | Description |
| :--- | :--- |
| `PGADMIN_EMAIL` | pgAdmin login email for local development. |
| `PGADMIN_PASSWORD` | pgAdmin login password for local development. |

## Rollback

For release tagging conventions and how production releases are promoted, see
[Backend-Release-Management.md](Backend-Release-Management.md).

To roll back to a previous published image:

1. Identify a known-good `sha-<short-commit>` tag from GHCR or the GitHub Actions
   run.
2. On the AAU server, update `.env`:

```env
IMAGE_TAG=sha-abc1234
```

3. Refresh the stack:

```bash
docker compose pull backend
docker compose up -d backend
docker compose ps
```

4. Verify:

```bash
curl -s http://se2-demo.aau.at:53210/actuator/health
```

## Verification Checklist

- `Dockerfile` has both source-build and CI-packaged-jar paths.
- `compose.yaml` uses the GHCR image and publishes `${PUBLIC_PORT:-53210}:8080`.
- `compose.yaml` health checks `http://localhost:8080/actuator/health`.
- `compose-dev.yaml` is documented as local-only.
- GHCR publish workflow targets `runtime-from-workspace`.
- Health URL is documented as
  `http://se2-demo.aau.at:53210/actuator/health`.
- WebSocket URL is documented as `ws://se2-demo.aau.at:53210/ws`.
- Automatic GitHub Actions SSH deployment and the manual fallback path are
  documented separately.
- Admin-seeding switches (`DEBUG_ENABLED`, `ADMIN_PASSWORD`) are documented and
  default to off.
- Required production secrets are listed and are not committed.

