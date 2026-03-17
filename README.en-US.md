# nextcloud-notify-push-docker

Docker packaging for `notify_push`, with the upstream source vendored in `./notify_push` via `git subtree`.

This repository adds:

- a Portuguese README for the vendored `notify_push` project
- an example deployment behind `nginx-proxy`
- GitHub Actions configuration to publish the image to GHCR

## Repository layout

- `notify_push/`: upstream `nextcloud/notify_push` source tree
- `docker-compose.example.yml`: example deployment with `nginx-proxy`
- `.github/workflows/publish-ghcr.yml`: image publishing workflow for GHCR

## Deploy behind nginx-proxy

Use the same host for Nextcloud and `notify_push`, and route `/push/` to the push service.

Example using the generic domain `cloud.example.com`:

```yaml
services:
  nextcloud:
    image: nextcloud:apache
    environment:
      VIRTUAL_HOST: cloud.example.com
      VIRTUAL_PATH: /
    networks:
      - reverse-proxy
      - internal

  notify_push:
    image: ghcr.io/librecodecoop/nextcloud-notify-push-docker:latest
    environment:
      PORT: "7867"
      NEXTCLOUD_URL: "https://cloud.example.com"
      REDIS_URL: "redis://redis:6379/0"
      DATABASE_URL: "postgres://nextcloud:secret@postgres/nextcloud"
      DATABASE_PREFIX: ""
      LOG: "info"
      VIRTUAL_HOST: cloud.example.com
      VIRTUAL_PATH: /push/
      VIRTUAL_DEST: /
      VIRTUAL_PORT: "7867"
    expose:
      - "7867"
    networks:
      - reverse-proxy
      - internal
```

Important details:

- `notify_push` must be attached to the same Docker network as `nginx-proxy`
- use `VIRTUAL_PATH=/push/` with the trailing slash
- use `VIRTUAL_DEST=/` so the `/push/` prefix is stripped before forwarding the request
- with `nginx-proxy`, `expose` is usually enough and you do not need to publish port `7867` on the host

## Configure in Nextcloud

Once the service is reachable through the reverse proxy:

```bash
occ app:enable notify_push
occ notify_push:setup https://cloud.example.com/push
occ notify_push:self-test
```

## Test the setup

Verify the proxy responses:

```bash
curl -I https://cloud.example.com/push
curl -I https://cloud.example.com/push/
```

Expected behavior:

- `/push` returns `301` redirecting to `/push/`
- `/push/` does not return `502` or another proxy upstream error

Watch the service logs while generating a Nextcloud notification, Talk message, or file change:

```bash
docker compose logs -f notify_push
```

## Publish to GHCR

The workflow in `.github/workflows/publish-ghcr.yml` builds the image from `notify_push/Dockerfile` and publishes it to
GitHub Container Registry.

It runs on:

- pushes to `main`
- version tags matching `v*`
- manual dispatch

Required repository settings:

- GitHub Actions must be enabled
- the workflow must have `packages: write` permission

The published image name is derived from the repository path, for example:

```text
ghcr.io/librecodecoop/nextcloud-notify-push-docker:latest
ghcr.io/librecodecoop/nextcloud-notify-push-docker:main
```

## Portuguese docs

The Portuguese translation for the vendored upstream README is available at
[`notify_push/README.pt-BR.md`](/home/mohr/git/librecode/nextcloud-notify-push-docker/notify_push/README.pt-BR.md).
