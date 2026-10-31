# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

docker-monero-node: a Docker Compose stack running a Monero node (`monerod`) with monitoring (Prometheus, Grafana, exporter, nodemapper), anonymity tooling (tor, i2p), and a block explorer. Images are published to Docker Hub as `lalanza808/*`.

## Commands

There is no test or lint suite. Verify changes by building images and running the stack.

```bash
make up          # docker-compose.yaml — default; pulls prebuilt lalanza808/* images
make up-full     # docker-compose.full.yaml — builds everything locally, monerod compiled from source
make build       # build default stack images locally
make build-full  # build full stack images (monerod compiled from source)
make down / down-full
make logs        # follow monerod logs (logs-full for full stack)

# Equivalent without make:
docker compose up -d
docker compose build
docker compose logs -f monerod
```

### Releasing images

`./release.sh <image> [dh_user]` builds a multi-arch (amd64+arm64) image with buildx and pushes it to Docker Hub. `<image>` is one of: `monero`, `exporter`, `nodemapper`, `tor`, `i2p`. The `DH_USER` defaults to `lalanza808`.

CI (`.github/workflows/publish.yaml`) runs this automatically: a push to `master` whose commit message contains `release(<image>)` (e.g. `release(monero): bump to v0.18.5.3`) triggers the matching publish job. Version numbers that CI builds come from variables at the top of `release.sh`.

## Architecture

### Two compose files

- `docker-compose.yaml` (default): services carry both `image:` (pulled prebuilt) and `build:` sections. Monerod uses `dockerfiles/monero`.
- `docker-compose.full.yaml`: no `image:` pins for the custom services, so everything is built locally. Monerod uses `dockerfiles/monero_compile`.

**The monerod Dockerfile names are easy to misread:**
- `dockerfiles/monero` — fetches official prebuilt binaries via `dockerfiles/files/monerod/install.sh`, which GPG-verifies `hashes.txt` against `binaryfate.asc` and checks SHA256 sums.
- `dockerfiles/monero_compile` — clones monero-project/monero at `ARG MONERO_RELEASE` and runs `make -j$THREADS` (build args passed from `docker-compose.full.yaml`).

Both copy in `monerod_entrypoint.sh` (as `/entrypoint.sh`) and `ban_list.txt`.

### Networking and the monerod entrypoint

- Custom bridge `tor_net` with hardcoded subnet `172.31.255.0/24`; tor is pinned to `172.31.255.250`, i2p to `172.31.255.251`.
- These IPs are **hardcoded a second time** in `dockerfiles/files/monerod/monerod_entrypoint.sh` (`--tx-proxy=tor,...` / `--tx-proxy=i2p,...`) — monerod requires literal IPs. If you change the subnet or container addresses in compose, update the entrypoint too.
- The entrypoint waits for onion hostnames in `/var/lib/tor/{monerod,monerod-rpc}/hostname` (tor volume mounted read-only into monerod), exports them, then launches `monerod` with a fixed flag set: `--anonymous-inbound` on 18084 and the tx-proxies above.
- Port scheme (monerod): 18080 p2p, 18081 restricted RPC, 18082 zmq, 18083 unrestricted RPC, 18084 tor anonymous-inbound. Only 18080/18081 bind publicly by default; the rest are localhost-bound.
- `dockerfiles/tor-config` defines two hidden services: `monerod-rpc` → `monerod:18081`, `monerod` → `monerod:18084`.

### Monitoring stack

- `exporter` (upstream Go `monero-exporter`) → scrapes monerod's unrestricted RPC (`monerod:18083`).
- `nodemapper` (`dockerfiles/nodemapper.py`, Flask/gunicorn) → peer GeoIP metrics using a downloaded GeoLite2 mmdb; configured via `NODE_HOST`/`NODE_PORT`.
- Prometheus scrapes `exporter:9000` and `nodemapper:5000` per `files/prometheus/config.yaml`.
- Grafana is provisioned from `files/grafana/`: datasource + dashboard providers under `provisioning/`, dashboard JSON under `dashboards/` (`node_stats.json`).

### VPS provisioning

`cloud-init.sh` is a one-shot VPS bootstrap: installs docker/nginx/certbot/tor, sets up a tor hidden service and nginx TLS (with `nginx-example.conf`), clones this repo, writes a `.env`, runs `make up` as a dedicated user, and registers the node with monero.fail. Configuration for local use comes from `env-example` → `.env`.

## Bumping the Monero version

This touches several files that must stay in sync (as of v0.18.5.3):

1. `dockerfiles/files/monerod/hashes.txt` — replace with the new GPG-signed hashes file from downloads.getmonero.org (verified at build time against `binaryfate.asc`). Versioning is driven by this file, not by a tag in the binary-fetch Dockerfile.
2. `release.sh` — `MONERO_VERSION` (what CI publishes).
3. `docker-compose.yaml` — `image: lalanza808/monero:vX.Y.Z.W`.
4. `docker-compose.full.yaml` — `MONERO_RELEASE` build-arg default.
5. `dockerfiles/monero_compile` — `ARG MONERO_RELEASE` default.
6. `env-example` — `MONERO_RELEASE`; `README.md` — image tag mentions.

Commit as `release(monero): ...` to trigger the CI publish job. Other images (`exporter`, `nodemapper`, `tor`, `i2p`) have their version variables in `release.sh` and pinned tags in `docker-compose.yaml`; keep README tags matching as well.