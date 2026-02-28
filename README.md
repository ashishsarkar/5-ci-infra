# CI Security Tooling Stack

Standalone Docker Compose stack providing security and artifact management services for the CI pipeline (`2-web-app/.github/workflows/frontend-ci.yml`).

## Services

| Service | Purpose | Image |
|---|---|---|
| **DefectDojo** | Vulnerability management — aggregates scan results (SAST, SCA, container scans) | `defectdojo/defectdojo-django:2.38.1` |
| **Dependency-Track** | SCA & SBOM analysis — tracks component vulnerabilities | `dependencytrack/apiserver:4.11.4` |
| **MinIO** | S3-compatible object storage — stores CI reports and artifacts | `minio/minio:RELEASE.2024-06-13T22-53-53Z` |
| **Docker Registry** | Container image registry for CI builds | `registry:2` |
| **Registry UI** | Web UI for browsing the container registry | `joxit/docker-registry-ui:2.5.7` |

## Prerequisites

- Docker Desktop (or Docker Engine + Docker Compose v2)
- On Apple Silicon (ARM64): Rosetta emulation is used automatically for x86-only images (`platform: linux/amd64`)
- Minimum recommended resources: 4 CPU / 8 GB RAM allocated to Docker

## Quick Start

```bash
# From workspace root
cd 5-ci-infra

# Start all services
docker compose up -d

# Check status (wait ~90s for all healthchecks to pass)
docker ps -a --filter "name=5-ci-infra"

# Stop all services
docker compose down

# Stop and remove all data volumes
docker compose down -v
```

## Access & Credentials

| Service | URL | Username | Password |
|---|---|---|---|
| DefectDojo | http://localhost:8080 | `admin` | `admin` |
| Dependency-Track UI | http://localhost:8082 | `admin` | `admin` |
| Dependency-Track API | http://localhost:8081 | — | API key (generate from UI) |
| MinIO Console | http://localhost:9001 | `minioadmin` | `minioadmin` |
| MinIO S3 API | http://localhost:9000 | `minioadmin` | `minioadmin` |
| Registry API | http://localhost:5050 | — | No auth (local dev) |
| Registry UI | http://localhost:8443 | — | No auth (local dev) |

## Container Overview

After `docker compose up -d`, you'll see 13 containers. **Two will show `Exited (0)` — this is expected:**

| Container | Type | Expected Status |
|---|---|---|
| `defectdojo-web` | Long-running | `Up (healthy)` |
| `defectdojo-celery-worker` | Long-running | `Up` |
| `defectdojo-celery-beat` | Long-running | `Up` |
| `defectdojo-db` | Long-running | `Up (healthy)` |
| `defectdojo-rabbitmq` | Long-running | `Up (healthy)` |
| `dtrack-apiserver` | Long-running | `Up (healthy)` |
| `dtrack-frontend` | Long-running | `Up` |
| `dtrack-db` | Long-running | `Up (healthy)` |
| `minio` | Long-running | `Up (healthy)` |
| `registry` | Long-running | `Up (healthy)` |
| `registry-ui` | Long-running | `Up` |
| `defectdojo-initializer` | **Init job** | **`Exited (0)`** — runs migrations + creates admin user, then stops |
| `minio-init` | **Init job** | **`Exited (0)`** — creates `ci-reports` bucket, then stops |

> `Exited (0)` = completed successfully. Only non-zero exit codes indicate failure.

## Port Map

```
8080  →  DefectDojo Web UI
8081  →  Dependency-Track API
8082  →  Dependency-Track Frontend
9000  →  MinIO S3 API
9001  →  MinIO Console
5050  →  Docker Registry API (v2)
8443  →  Registry UI
```

## Pushing Images to the Local Registry

```bash
# Tag an image for the local registry
docker tag my-app:latest localhost:5050/my-app:latest

# Push to local registry
docker push localhost:5050/my-app:latest

# Verify
curl http://localhost:5050/v2/_catalog
```

## CI Pipeline Integration

The `frontend-ci.yml` pipeline connects to these services via secrets:

| Secret | Value (local dev) |
|---|---|
| `MINIO_ENDPOINT` | `http://localhost:9000` |
| `MINIO_ACCESS_KEY` | `minioadmin` |
| `MINIO_SECRET_KEY` | `minioadmin` |
| `MINIO_BUCKET` | `ci-reports` |
| `DEFECTDOJO_URL` | `http://localhost:8080` |
| `DEFECTDOJO_API_KEY` | Generate from DefectDojo UI: API v2 > API Key |
| `DTRACK_URL` | `http://localhost:8081` |
| `DTRACK_API_KEY` | Generate from Dependency-Track UI: Administration > Access Management > Teams > API Keys |
| `REGISTRY_URL` | `localhost:5050` |

## Data Persistence

All data is stored in named Docker volumes:

| Volume | Service | Data |
|---|---|---|
| `defectdojo-pgdata` | DefectDojo | Vulnerability findings, users, products |
| `defectdojo-rmq` | DefectDojo | Celery message queue |
| `dtrack-data` | Dependency-Track | SBOM data, internal caches |
| `dtrack-pgdata` | Dependency-Track | Component and vulnerability database |
| `minio-data` | MinIO | CI reports and scan artifacts |
| `registry-data` | Docker Registry | Container images |

To reset all data: `docker compose down -v`

## Upgrading to Harbor

The local Docker Registry is a lightweight stand-in. To switch to Harbor for production:

1. Download the Harbor offline installer from https://goharbor.io/docs/latest/install-config/
2. Run `./install.sh` which generates TLS certs, configs, and inter-service secrets
3. Update the CI pipeline's `REGISTRY_URL` secret to point to Harbor

> Harbor cannot run as individual standalone containers — it requires its official installer to generate shared configuration.

## Troubleshooting

**DefectDojo shows `unhealthy`:**
- Wait 90 seconds after startup — it has a 60s start period
- Check logs: `docker logs 5-ci-infra-defectdojo-web-1`

**Dependency-Track API is slow on first start:**
- Initial NVD mirror sync takes 10-15 minutes
- Check progress: `docker logs -f 5-ci-infra-dtrack-apiserver-1`

**ARM64 / Apple Silicon issues:**
- Images with `platform: linux/amd64` run under Rosetta emulation
- Ensure Rosetta is enabled in Docker Desktop: Settings > General > Use Rosetta

**Port conflicts:**
- If a port is already in use, edit `docker-compose.yml` and change the host port (left side of `:`):
  ```yaml
  ports:
    - "9080:8081"  # Change 8080 to 9080
  ```
