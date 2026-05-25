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
  `DB_HOST=postgres`;
- disables Spring Boot Docker Compose integration with
  `SPRING_DOCKER_COMPOSE_ENABLED=false`;
- pulls the image on refresh through `pull_policy: always`;
- exposes a container health check at
  `http://localhost:8080/actuator/health`.

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

## GHCR Publishing Workflow

The GitHub Actions workflow
`.github/workflows/docker-publish.yml` publishes the backend image to:

```text
ghcr.io/se2-machi-koro/server
```

Triggers:

- push to `main`;
- git tags matching `v*`;
- manual `workflow_dispatch`.

Workflow sequence:

1. `build-jar` checks out the backend repository, sets up JDK 21, and runs:

```bash
./gradlew bootJar -x test
mkdir -p build/docker
cp build/libs/*.jar build/docker/app.jar
```

2. The jar is uploaded as a short-lived artifact named `app-jar`.
3. `build-and-push` downloads the jar, sets up QEMU and Docker Buildx, logs in
   to GHCR using `GITHUB_TOKEN`, and builds the Docker image with:

```text
target: runtime-from-workspace
platforms: linux/amd64,linux/arm64
```

Published tags:

| Tag | Meaning |
| :--- | :--- |
| `latest` | Published for `main`. |
| `sha-<short-commit>` | Published for each workflow run and used for rollback. |
| `v*` | Published when a matching version tag is pushed. |
| branch ref tag | Produced by Docker metadata for branch builds. |

No custom GHCR secret is required for the publish workflow because it uses
GitHub's built-in `GITHUB_TOKEN` with `packages: write` permission.

## Automatic AAU Deployment

The intended automatic deployment path is:

1. Merge or push the backend change to `main`.
2. GitHub Actions publishes a new GHCR image.
3. doco-cd on the AAU server reconciles the stack from the backend
   repository's `compose.yaml`.
4. The backend service pulls
   `ghcr.io/se2-machi-koro/server:${IMAGE_TAG:-latest}` and restarts.
5. The deployment is healthy when both containers are healthy and the public
   health endpoint returns `UP`.

Verify the public service:

```bash
curl -s http://se2-demo.aau.at:53210/actuator/health
```

Expected response:

```json
{"status":"UP"}
```

## Manual Fallback Deployment

Use this only when doco-cd is not configured for group 6 or when a manual
refresh is needed after a GHCR image was published.

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

Local-only variables for `compose-dev.yaml`:

| Variable | Description |
| :--- | :--- |
| `PGADMIN_EMAIL` | pgAdmin login email for local development. |
| `PGADMIN_PASSWORD` | pgAdmin login password for local development. |

## Rollback

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
- Automatic doco-cd deployment and manual fallback deployment are separate.
- Required production secrets are listed and are not committed.

