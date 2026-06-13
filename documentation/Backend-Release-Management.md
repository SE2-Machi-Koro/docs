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
merge PR to main ──▶ CI (docker-publish.yml) ──▶ GHCR image published ──▶ auto deploy to AAU
                       build-jar                  latest + sha-<short>      (deploy job, SSH)
                       build-and-push             + main
```

1. A pull request is merged (or a commit is pushed) to `main`.
2. The `docker-publish.yml` workflow runs:
   - `build-jar` builds the Spring Boot jar (`./gradlew bootJar -x test`);
   - `build-and-push` builds the multi-arch Docker image
     (`linux/amd64,linux/arm64`) and pushes it to
     `ghcr.io/se2-machi-koro/server` with the computed tags.
3. The `deploy` job (runs only on `main`) SSHes into the AAU server and restarts
   the backend against the new image. See the
   [Automatic AAU Deployment](Backend-Deployment.md#automatic-aau-deployment)
   section for details.

The workflow also runs on git tags matching `v*` and on manual
`workflow_dispatch`. Tag builds publish a versioned image but do **not** run the
`deploy` job (which is gated on `refs/heads/main`).

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
| `latest` | Push to `main`. | Floating pointer to the newest `main` build. Default image used by `compose.yaml` (`${IMAGE_TAG:-latest}`). |
| `sha-<short-commit>` | Every workflow run. | Immutable, traceable handle for one exact commit. Use this to pin a release or roll back. |
| `main` | Push to `main` (branch ref). | Convenience alias for the current `main` build; equivalent to `latest`. |
| `v<x.y.z>` | A git tag matching `v*` is pushed. | Human-readable version marker for a milestone/presentation release. |

**Which tag to use:**

- **Production deployment** floats on `latest` by default, so every `main` merge
  ships the newest build. If `IMAGE_TAG` is pinned to an immutable tag,
  deployments continue using that pinned release until `IMAGE_TAG` is reset to
  `latest` or removed — the `deploy` job still runs, but `docker compose pull
  backend` keeps pulling the pinned tag.
- **Pinning / rollback** must use an immutable `sha-<short-commit>` (or a `v*`)
  tag. Never pin to `latest`, because it moves with every merge.

## Versioning and Promotion

Promotion to production is implicit while `IMAGE_TAG` is left at its default
(`latest`): any image that lands on `main` becomes the production release,
because the `deploy` job ships it automatically. The "tested" gate is therefore
the PR's CI and review — a change reaches `main` only after CI and SonarCloud
pass and the PR is approved. When `IMAGE_TAG` is pinned, new `main` builds are
published but production stays on the pinned release until it is reset.

To run a **specific release** instead of the floating `latest`, pin `IMAGE_TAG`
in the server-side `.env` (which lives next to `compose.yaml` on the AAU server)
to an immutable tag:

```env
# Pin to one exact build instead of the moving `latest` tag
IMAGE_TAG=sha-abc1234
```

Then refresh the backend service:

```bash
docker compose pull backend
docker compose up -d --no-deps backend
docker compose ps
```

To find the `sha-<short-commit>` for a known-good release, open the GitHub
Actions run for the merge commit, or inspect the package versions in GHCR.

To return to floating production behaviour, set `IMAGE_TAG=latest` (or remove
the line, since `compose.yaml` defaults to `latest`) and refresh again.

## Rollback

To roll back production to a previously published, known-good image:

1. Identify the target `sha-<short-commit>` tag from the GitHub Actions run of a
   known-good commit or from the GHCR package versions.
2. On the AAU server, edit the `.env` next to `compose.yaml`:

```env
IMAGE_TAG=sha-abc1234
```

3. Pull and restart only the backend (Postgres is left untouched):

```bash
docker compose pull backend
docker compose up -d --no-deps backend
docker compose ps
```

4. Verify the public health endpoint reports `UP`:

```bash
curl -s http://se2-demo.aau.at:53210/actuator/health
```

Expected response:

```json
{"status":"UP"}
```

Because every build is published as an immutable `sha-<short-commit>` image,
rollback never requires a rebuild — it only repoints `IMAGE_TAG`.

## Verification Checklist

- Merging to `main` publishes `latest`, `main`, and `sha-<short-commit>` images
  to `ghcr.io/se2-machi-koro/server`.
- Production floats on `latest`; pinning and rollback use immutable
  `sha-<short-commit>` tags.
- A specific release can be selected by setting `IMAGE_TAG` in the server `.env`.
- Rollback is achieved by repointing `IMAGE_TAG` and refreshing the backend,
  with no rebuild.
