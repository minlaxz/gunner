# gunner

Self-hosted GitHub Actions runner as a Docker Compose service.

- **Ephemeral.** Each container start registers one runner, takes exactly one job,
  deregisters and exits; Compose restarts it and the cycle repeats. Nothing a job leaves
  behind survives into the next job.
- **Builds on the host daemon.** The host's `/var/run/docker.sock` is mounted into the
  runner, so `docker build` in a job uses the host's image layer cache. The second build of
  the same Dockerfile is fast.
- **One runner per repository.** Each runner directory registers to one repository (or one
  organization) with its own credential and labels.

The runner is root on the host for all practical purposes because it owns the Docker
socket. Run it on a dedicated CI host and register it only to private repositories whose
pushers you trust. Public repositories accept pull requests from anyone; never point a
runner with a Docker socket at one.

## Host requirements

- x86_64 Linux. The runner image is multi-arch, but jobs that build `linux/amd64` images on
  an arm64 host go through QEMU and are several times slower.
- Docker Engine 24 or newer with the Compose plugin.
- About 2 GB RAM free per runner, 20 GB free disk for image layers and build cache.
- Outbound HTTPS to `github.com`, `api.github.com`, Docker Hub and your image registry.

## Settings

All settings live in `.env` next to `docker-compose.yml`. Copy `.env.example` to start.

| Variable | Example | What it does and why it is set this way |
| --- | --- | --- |
| `REPO_URL` | `https://github.com/<OWNER>/<REPO>` | The repository the runner registers to. Only workflows in this repository can schedule jobs on it. One runner per repository keeps the blast radius of a leaked credential or a bad workflow to that repository. |
| `LABELS` | `myapp-build` | Comma-separated labels added on top of the defaults (`self-hosted`, `linux`, `x64`). Workflows select the runner with `runs-on: [self-hosted, myapp-build]`. Always use a custom label: `runs-on: self-hosted` alone matches every self-hosted runner the repository will ever have, so a job can land on a machine that lacks what it needs. Name the label after the job type, not the machine, so it keeps working when the runner moves hosts. |
| `RUNNER_NAME_PREFIX` | `myapp` | Prefix for the runner name GitHub shows under Settings → Actions → Runners. Ephemeral runners get a new random suffix on every registration; the prefix is how you tell runners apart when several repositories or hosts register to the same organization. |
| `RUNNER_SCOPE` | `repo` | `repo` registers to `REPO_URL`. `org` registers to the organization, so any repository in it can use the runner. Default is `repo` for the same blast-radius reason as above. |
| `ACCESS_TOKEN` | fine-grained PAT | Long-lived credential. The container exchanges it for a one-hour registration token on every start, which an ephemeral runner needs because it re-registers after every job. See *Credential* below. |
| `RUNNER_TOKEN` | registration token | Short-lived alternative to `ACCESS_TOKEN` for smoke tests. Dies after one hour, after which the runner cannot re-register. |

Fixed in `docker-compose.yml`, not meant to change per host:

| Setting | Why |
| --- | --- |
| `EPHEMERAL=1` | One job per container start. No state leaks between jobs, no stale workspaces, no long-lived process to babysit. |
| `DISABLE_AUTO_UPDATE=1` | The runner binary version comes from the pinned image tag. Dependabot bumps the tag in a pull request; the runner never updates itself in place. |
| Docker socket mount | Host daemon = shared layer cache. Docker-in-Docker would need a privileged sidecar and would lose the cache on every restart. |
| `restart: unless-stopped` | Replaces a systemd unit. The container restarts after each job and after host reboots. |
| Log rotation `10m × 3` | Runner logs are chatty; without a cap they fill the disk. |

## Credential

Create a fine-grained personal access token at
<https://github.com/settings/personal-access-tokens/new>:

- **Resource owner**: the account or organization that owns `<REPO>`. For an organization
  repository that must be the organization, and the organization must allow fine-grained
  tokens (Settings → Third-party Access → Personal access tokens).
- **Repository access**: *Only select repositories* → `<REPO>`.
- **Repository permissions** → *Administration*: Read and write. Nothing else.

Put it in `.env` as `ACCESS_TOKEN`. Give the token an expiry and a reminder: when it
expires the runner silently stops registering and jobs queue forever.

The token is tied to the user who created it. If that user leaves the organization the
runner stops. Prefer a service account for organization runners.

For a smoke test without a PAT, use a raw registration token instead. It expires after
one hour:

```sh
gh api -X POST repos/<OWNER>/<REPO>/actions/runners/registration-token --jq .token
```

## Set up a runner

```sh
git clone https://github.com/minlaxz/gunner.git ~/gunner
cd ~/gunner
cp .env.example .env
chmod 600 .env              # the file holds the PAT
# edit .env: REPO_URL, LABELS, RUNNER_NAME_PREFIX, ACCESS_TOKEN
docker compose up -d
docker compose logs -f runner   # wait for "Listening for Jobs"
```

The runner appears under the repository's Settings → Actions → Runners with status
*Idle*. After each job it briefly shows *Offline* while the container restarts.

## Set up another runner

### Another repository on the same host

One directory per repository. Compose derives the project name from the directory, so the
containers do not clash.

```sh
git clone https://github.com/minlaxz/gunner.git ~/gunner-<REPO>
cd ~/gunner-<REPO>
cp .env.example .env && chmod 600 .env
# edit .env: REPO_URL for the other repository, its own LABELS, RUNNER_NAME_PREFIX, ACCESS_TOKEN
docker compose up -d
```

Both runners share the host Docker daemon and its layer cache. Jobs from the two
repositories can see each other's images; that is fine for one team, not for tenants that
must not see each other.

### More parallelism for the same repository

A matrix job with N entries runs serially on one runner. Scale the service:

```sh
docker compose up -d --scale runner=2
```

Each replica registers its own runner name (`RUNNER_NAME_PREFIX` plus a random suffix).
Budget RAM per replica; two concurrent image builds on a 4 GB host will swap.

### Another host

Repeat *Set up a runner* on the new host with the same `.env`. Both runners pick up jobs
for the same labels. Stop the old one with `docker compose down` once the new one shows
*Idle*; `down` deregisters it.

## Use from a project

In the workflow, target the custom label and build on the host daemon:

```yaml
jobs:
  build:
    runs-on: [self-hosted, myapp-build]   # second label = LABELS in .env
    steps:
      - uses: actions/checkout@v4

      # driver: docker sends builds to the host daemon, so its layer cache is reused.
      # The default docker-container driver has its own empty cache in every job.
      - uses: docker/setup-buildx-action@v3
        with:
          driver: docker

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ghcr.io/<OWNER>/<IMAGE>:${{ github.sha }}

      # Ephemeral runners do not clean the host daemon. Prune week-old layers and build
      # cache so the disk does not fill; one week keeps the cache warm between releases.
      - if: always()
        run: |
          docker image prune -f --filter until=168h
          docker builder prune -f --filter until=168h
```

Things that change when moving a workflow from GitHub-hosted runners:

- Drop `cache-from: type=gha` / `cache-to: type=gha`. The host daemon is the cache.
- `driver: docker` cannot build multi-platform images. Keep the default driver and add a
  `type=registry` cache if you need `linux/arm64` too.
- `services:` containers (for example a database for tests) work, because the runner can
  talk to the host daemon. They are started on the host and removed after the job.
- `setup-python` / `setup-node` tool caches are not persisted across jobs. Add a named
  volume for `/opt/hostedtoolcache` to the compose file if that becomes slow.
- Only trigger self-hosted jobs from pushes to protected branches or from
  `workflow_dispatch`. Pull requests from forks must not reach this runner.

Runner offline means the job sits in *Queued* until the runner returns. GitHub does not
alert on this. The fallback is a one-line change to `runs-on: ubuntu-latest`.

## Operate

```sh
docker compose logs --tail 100 runner
docker compose pull && docker compose up -d   # after a Dependabot image bump
docker compose down                           # deregisters the runner on shutdown
```

To rotate the credential: edit `ACCESS_TOKEN` in `.env`, then `docker compose up -d`.
The next registration uses the new token.

## Smoke test

`.github/workflows/smoke.yml` in this repository runs on label `gunner-smoke` and checks
that the runner can talk to the host daemon and build an image. Register a runner to this
repository with `LABELS=gunner-smoke`, then:

```sh
gh workflow run smoke.yml
gh run watch
```
