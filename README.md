# gunner

Self-hosted GitHub Actions runner as a Docker Compose service. Ephemeral: each
container start registers, runs exactly one job, deregisters and restarts. Jobs
talk to the host Docker daemon through the mounted socket, so `docker build`
inside a job reuses the host's layer cache.

The runner is root on the host for all practical purposes (it owns the Docker
socket). Run it on a dedicated CI host, not next to production, and only
register it to private repositories whose pushers you trust.

## Host requirements

- x86_64 Linux, Docker Engine 24 or newer with the Compose plugin
- 2 GB RAM free per runner replica, 20 GB free disk for image layers
- Outbound HTTPS to github.com, api.github.com, Docker Hub and ghcr.io

## Install

```sh
git clone git@github.com:minlaxz/gunner.git ~/gunner
cd ~/gunner
cp .env.example .env
chmod 600 .env
# edit .env: REPO_URL, LABELS, ACCESS_TOKEN
docker compose up -d
docker compose logs -f runner   # wait for "Listening for Jobs"
```

The runner appears under the repository's Settings -> Actions -> Runners.

## Credential

Create a fine-grained personal access token at
https://github.com/settings/personal-access-tokens/new:

- Resource owner: the account or organization that owns the repository
- Repository access: only select repositories -> the one repository
- Repository permissions -> Administration: Read and write

Put it in `.env` as `ACCESS_TOKEN`. The container exchanges it for a one-hour
registration token on every start. Set an expiry and a reminder; when the token
expires the runner silently stops registering and jobs queue forever.

For a smoke test without a PAT, use a raw registration token instead
(`RUNNER_TOKEN`). It expires after one hour:

```sh
gh api -X POST repos/<owner>/<repo>/actions/runners/registration-token --jq .token
```

## Use from a workflow

```yaml
jobs:
  build:
    runs-on: [self-hosted, spai-build]   # second label = LABELS in .env
```

Use `docker/setup-buildx-action` with `driver: docker` so builds go to the host
daemon and its cache. Add a final cleanup step so the host disk does not fill:

```yaml
      - if: always()
        run: |
          docker image prune -f --filter until=168h
          docker builder prune -f --filter until=168h
```

## Operate

```sh
docker compose pull && docker compose up -d   # upgrade after a Dependabot bump
docker compose logs --tail 100 runner
docker compose down                           # deregisters on shutdown
```

If a runner shows offline in GitHub, jobs targeting it wait indefinitely. There
is no alert. Flip the workflow's `runs-on` to a GitHub-hosted label as the
fallback.

## Smoke test

`.github/workflows/smoke.yml` runs on label `gunner-smoke`. Register a runner to
this repository with `LABELS=gunner-smoke`, then:

```sh
gh workflow run smoke.yml
gh run watch
```
