# mdnest image builder

A thin build repo that publishes [mdnest](https://github.com/mahsanamin/mdnest)
images to Docker Hub under [`dualfrostbeams`](https://hub.docker.com/u/dualfrostbeams).
Upstream mdnest is distributed as source only and publishes no container images
of its own — this repo exists to build them and publish the results automatically.

**Unofficial build.** This project is not affiliated with mdnest or its author.

## Images produced

| Image | Source | Notes |
|---|---|---|
| `dualfrostbeams/mdnest-backend:<tag>` | upstream `backend/Dockerfile` | Go + alpine; `GIT_COMMIT` / `BUILD_TIME` baked in so `/api/config` reports the exact build |
| `dualfrostbeams/mdnest-frontend:<tag>` | upstream `frontend/Dockerfile` | node build → nginx |
| `...:latest` | — | always mirrors the newest release built |

Tags mirror upstream release tags (e.g. `v4.5.1`). Platform: **linux/amd64 only**.


## How it works

- **Daily schedule** resolves the latest upstream release via the GitHub API and
  checks whether that tag already exists on Docker Hub (both images). If both
  exist → skip; if either is missing → build both (the pair moves in lockstep).
- **Manual dispatch** builds any specific upstream tag; the `force` input
  rebuilds a tag that already exists (e.g. upstream re-tags or you suspect a
  bad cached layer).
- Builds use GitHub Actions layer caching, so repeat builds of the Go/npm
  parts are fast.
- Every successful build commits the version to a `VERSION` file — a visible
  build history, and it keeps the repo "active" (see caveat below).
- Images are pushed with **no provenance/SBOM attestations** (plain manifests),
  matching upstream's own build behaviour and avoiding attestation-manifest
  quirks in older Podman — which is what consumes these images.

## Required secrets

The workflow needs two repo secrets (**Settings → Secrets and variables →
Actions**):

| Secret | Value |
|---|---|
| `DOCKERHUB_USERNAME` | Docker Hub username to publish under |
| `DOCKERHUB_TOKEN` | Docker Hub [access token](https://hub.docker.com/settings/personal-access-tokens) with read/write access — never the account password |

## Caveats

- **GitHub disables scheduled workflows after 60 days of repo inactivity.**
  Each build's `VERSION` commit resets that clock, so as long as mdnest keeps
  releasing (it's active), the schedule stays alive. If upstream goes quiet for
  two months, GitHub emails you and disables the schedule — nothing to build
  anyway; re-enable with one click when a release lands.
- **The skip-check queries Docker Hub anonymously**, which only works while the
  Hub repos are **public**. If you ever flip them private, the check sees 404
  and the pipeline would rebuild daily — keep the repos public or adjust the
  check.
