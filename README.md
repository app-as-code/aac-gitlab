<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/icon-dark.svg">
    <source media="(prefers-color-scheme: light)" srcset="assets/icon-light.svg">
    <img src="assets/icon.svg" alt="GitLab logo" width="128">
  </picture>
</p>

<h1 align="center">aac-gitlab</h1>

<p align="center">Self-hosted deployment definition for GitLab CE, a source code management platform with a container registry and Pages.</p>

<p align="center">
  <a href="https://gitlab.com/gitlab-org/gitlab"><img alt="Upstream" src="https://img.shields.io/badge/upstream-gitlab--org%2Fgitlab-blue?logo=gitlab"></a>
  <a href="https://hub.docker.com/r/gitlab/gitlab-ce"><img alt="Image" src="https://img.shields.io/badge/image-gitlab%2Fgitlab--ce%3A18.9.1--ce.0-2496ED?logo=docker&logoColor=white"></a>
  <a href="https://gitlab.com/gitlab-org/gitlab/-/blob/master/LICENSE"><img alt="License" src="https://img.shields.io/badge/license-MIT%20(CE)-green"></a>
</p>

## About

GitLab is a web-based DevOps platform. It combines Git repository hosting with issue
tracking, merge requests, CI/CD pipelines, a container registry and static site hosting
(GitLab Pages) in one application. This repository describes how the Community Edition
(CE) is deployed as an App-as-Code service, together with its PostgreSQL database and
Redis cache.

Key features (as provided by upstream):

- Git repository hosting with merge requests and code review
- Built-in CI/CD pipelines with GitLab Runner support
- Integrated container registry
- GitLab Pages for static sites built from repositories
- Single sign-on via OpenID Connect (this deployment is prepared for Authentik)

## Upstream

- Project: <https://gitlab.com/gitlab-org/gitlab>
- Website: <https://about.gitlab.com>
- Container image: `gitlab/gitlab-ce:18.9.1-ce.0` (Docker Hub)
- Documentation: <https://docs.gitlab.com>
- Database: `postgres:16` (deployed as `aac-gitlab-db`)
- Cache: `redis:7.4` (deployed as `aac-gitlab-redis`)
- License: MIT for GitLab Community Edition (upstream project)

## Quick start (Docker Compose)

The Compose file is rendered from `service.yml` by the App-as-Code template engine. For a
manual run, the external networks used for routing must exist first:

```bash
docker network create secured
docker network create stack_internal

# Provide the secrets listed under "Configuration" in .env / stack.env
docker compose up -d
```

GitLab needs several minutes on first start while it initialises its database and
configuration. Follow progress with `docker compose logs -f aac-gitlab`.

Secrets are not stored in this repository. Provide them through the deployment pipeline
or create the `.env` and `stack.env` files yourself. Do not commit them.

## Configuration

Secrets (`CHANGE_ME` must be replaced for every deployment):

| Secret | Description |
|---|---|
| `GITLAB_ROOT_PASSWORD` | Initial password of the GitLab `root` account |
| `POSTGRES_PASSWORD` | Password of the GitLab database user (shared with the database container) |
| `AUTHENTIK_OIDC_CLIENT_ID` | OpenID Connect client ID for the single sign-on provider |
| `AUTHENTIK_OIDC_CLIENT_SECRET` | OpenID Connect client secret |

Environment variables passed to the container:

| Variable | Default | Description |
|---|---|---|
| `TZ` | `Europe/Berlin` | Time zone |
| `GITLAB_ROOT_PASSWORD` | `CHANGE_ME` (secret) | See above |
| `GITLAB_OMNIBUS_CONFIG` | generated | Omnibus configuration, rendered from `service.yml` (external URL, database and Redis hosts, OIDC login, registry and Pages settings) |

Template variables used in the rendered configuration (non-secret, set in `service.yml` `vars:`):

| Variable | Default | Description |
|---|---|---|
| `GITLAB_EXTERNAL_URL` | `https://gitlab.example.com` | Public URL of the GitLab instance; must match the ingress hostname |
| `GITLAB_REGISTRY_DOMAIN` | `registry.gitlab.example.com` | Public hostname of the container registry |
| `GITLAB_PAGES_DOMAIN` | `sites.gitlab.example.com` | Public hostname of GitLab Pages |
| `GITLAB_SSH_PORT` | `2222` | SSH port advertised to Git clients; must match the host port mapping |
| `DB_HOST` / `DB_NAME` / `DB_USER` | `aac-gitlab-db` / `gitlabhq_production` / `gitlab` | PostgreSQL connection |
| `REDIS_HOST` | `aac-gitlab-redis` | Redis host |
| `AUTHENTIK_OIDC_ISSUER` | `https://auth.example.com/application/o/gitlab/` | OpenID Connect issuer URL |
| `PUMA_WORKER_PROCESSES` | `8` | Rails worker processes (about 1 GB RSS each) |
| `GITLAB_DB_POOL` | `450` | Database connection pool size; must fit `PG_MAX_CONNECTIONS` |
| `SIDEKIQ_CONCURRENCY` | `100` | Background job concurrency |
| `PG_MAX_CONNECTIONS` | `500` | PostgreSQL `max_connections`; should exceed `GITLAB_DB_POOL` plus `SIDEKIQ_CONCURRENCY` |
| `PG_SHARED_BUFFERS` | `2GB` | PostgreSQL `shared_buffers`; a common rule of thumb is about 25 % of host RAM |

Use the sizing variables to fit a smaller host without changing the rest of `service.yml`.

Ports:

| Port | Protocol | Purpose |
|---|---|---|
| `80` (host `8888`) | TCP | Web interface |
| `22` (host `2222`) | TCP | Git over SSH |
| `5000` | TCP | Container registry |
| `8091` | TCP | GitLab Pages |

Volumes:

| Mount | Description |
|---|---|
| `gitlab_config` -> `/etc/gitlab` | GitLab Omnibus configuration |
| `gitlab_data` -> `/var/opt/gitlab` | Repositories, uploads, registry and application data |
| `gitlab_logs` -> `/var/log/gitlab` | GitLab logs |

Dependencies:

| Service | Image | Purpose |
|---|---|---|
| `aac-gitlab-db` | `postgres:16` | GitLab database (volume `db` -> `/var/lib/postgresql/data`) |
| `aac-gitlab-redis` | `redis:7.4` | Cache and Sidekiq queue (volume `redis-data` -> `/data`) |

## Deploy with App-as-Code

1. Edit `service.yml`. It is the single source of truth for image, ports, environment,
   volumes, dependencies and per-stage overrides. Do not edit the rendered output.
2. Pushing to `main` (or a configured stage branch) runs the service pipeline, which renders
   the manifests with `aac-template-engine` and triggers the deployment.
3. Releases of this definition are published under <https://github.com/app-as-code/aac-gitlab>.

## Known limitations

- The Traefik integration currently creates a router for the web interface only. The
  registry (port 5000) and Pages (port 8091) are published on the host and need an
  additional routing configuration to be reachable under their own hostnames.
- Moving existing GitLab data into the paths defined in `service.yml` is a manual step:
  stop the container, move the data directories, and only then roll out the definition.

## Credits and license

GitLab Community Edition is developed by GitLab Inc. and the GitLab community and is
licensed under the MIT license. This repository only contains the deployment definition
for it and is not a fork of the application. The logo is provided by GitLab; see
[`assets/ICON_SOURCE.md`](assets/ICON_SOURCE.md) for icon sources and licenses.
