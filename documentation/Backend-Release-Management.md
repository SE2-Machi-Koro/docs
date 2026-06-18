# Backend Release Management

This document describes how backend releases are produced today: versioned
container images published to GitHub Container Registry (GHCR) by the
`SE2-Machi-Koro/Server` repository, the image tagging conventions, and how a
specific release is promoted to production or rolled back.

It is based on the server repository files:

- `.github/workflows/docker-publish.yml`
- `compose.yaml`

For the full production stack, environment variables, and the manual fallback
deployment, see [Backend-Deployment.md](Backend-Deployment.md).

## Release Flow

A release is an immutable container image in GHCR, produced automatically by CI.
There is no separate manual build step — merging to `main` is the trigger.

```text
merge PR to main ──▶ CI (docker-publish.yml) ──▶ GHCR image published ──▶ Railway auto-redeploys
                       build-jar                  latest + sha-<short>      (detects new `latest`)
                       build-and-push             + main
```

1. A pull request is merged (or a commit is pushed) to `main`.
2. The `docker-publish.yml` workflow runs:
   - `build-jar` builds the Spring Boot jar (`./gradlew bootJar -x test`);
   - `build-and-push` builds the multi-arch Docker image
     (`linux/amd64,linux/arm64`) and pushes it to
     `ghcr.io/se2-machi-koro/server` with the computed tags.
3. Railway detects the updated `latest` image (via `pull_policy: always`) and
   automatically redeploys the backend service against the new image. See the
   [Automatic Railway Deployment](Backend-Deployment.md#automatic-railway-deployment)
   section for details.

The workflow also runs on git tags matching `v*` and on manual
`workflow_dispatch`. Tag builds publish a versioned image but do not change the
`latest` tag, so they do not trigger a Railway redeploy on their own.

## Image Tagging Conventions

Tags are computed by `docker/metadata-action` from this configuration:

```yaml
tags: |
  type=ref,event=branch
  type=ref,event=tag
  type=sha,prefix=sha-,format=short
  type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}
```

| Tag | Produced when | Purpose |
| :--- | :--- | :--- |
| `latest` | Push to `main`. | Floating pointer to the newest `main` build. Image deployed by Railway and used by `compose.yaml` (`${IMAGE_TAG:-latest}`). |
| `sha-<short-commit>` | Every workflow run. | Immutable, traceable handle for one exact commit. Use this to pin a release or roll back. |
| `main` | Push to `main` (branch ref). | Convenience alias for the current `main` build; equivalent to `latest`. |
| `v<x.y.z>` | A git tag matching `v*` is pushed. | Human-readable version marker for a milestone/presentation release. |

**Which tag to use:**

- **Production deployment** floats on `latest` by default, so every `main` merge
  ships the newest build. On Railway, deploying from `ghcr.io/se2-machi-koro/server:latest`
  means each new `main` build is picked up automatically. To pin a specific
  release on Railway, set the `IMAGE_TAG` environment variable (or change the
  image reference) to an immutable tag.
- **Pinning / rollback** must use an immutable `sha-<short-commit>` (or a `v*`)
  tag. Never pin to `latest`, because it moves with every merge.

## Versioning and Promotion

Promotion to production is implicit while Railway deploys the floating `latest`
tag: any image that lands on `main` becomes the production release, because
Railway redeploys it automatically. The "tested" gate is therefore the PR's CI
and review — a change reaches `main` only after CI and SonarCloud pass and the PR
is approved. When a specific `IMAGE_TAG` is pinned on Railway, new `main` builds
are published but production stays on the pinned release until it is reset.

To run a **specific release** instead of the floating `latest`, pin the
`IMAGE_TAG` environment variable in the Railway backend service settings to an
immutable tag:

```env
# Pin to one exact build instead of the moving `latest` tag
IMAGE_TAG=sha-abc1234
```

Railway redeploys automatically when the variable is saved.

To find the `sha-<short-commit>` for a known-good release, open the GitHub
Actions run for the merge commit, or inspect the package versions in GHCR.

To return to floating production behaviour, set `IMAGE_TAG=latest` (or remove
the variable, since the image defaults to `latest`).

## Rollback

To roll back production to a previously published, known-good image on Railway:

1. Identify the target `sha-<short-commit>` tag from the GitHub Actions run of a
   known-good commit or from the GHCR package versions.
2. In the Railway backend service settings, set the `IMAGE_TAG` environment
   variable:

```env
IMAGE_TAG=sha-abc1234
```

3. Save the change. Railway automatically redeploys the backend against the
   pinned image (the managed PostgreSQL service is left untouched).
4. Verify the public health endpoint reports `UP`:

```bash
curl -s https://machi-koro.up.railway.app/actuator/health
```

Expected response:

```json
{"status":"UP"}
```

Because every build is published as an immutable `sha-<short-commit>` image,
rollback never requires a rebuild — it only repoints `IMAGE_TAG`.

> **Legacy (AAU):** On the former AAU server, rollback was done by editing the
> `.env` next to `compose.yaml` and running `docker compose pull backend &&
> docker compose up -d --no-deps backend`. See
> [Backend-Deployment.md](Backend-Deployment.md#legacy-aau-deployment).

## Verification Checklist

- Merging to `main` publishes `latest`, `main`, and `sha-<short-commit>` images
  to `ghcr.io/se2-machi-koro/server`.
- Production floats on `latest`; pinning and rollback use immutable
  `sha-<short-commit>` tags.
- A specific release can be selected by setting `IMAGE_TAG` in the Railway
  service settings.
- Rollback is achieved by repointing `IMAGE_TAG`, with no rebuild; Railway
  redeploys automatically.
