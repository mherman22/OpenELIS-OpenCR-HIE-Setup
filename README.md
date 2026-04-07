# OpenELIS + OpenCR Health Information Exchange

[![OpenELIS-OpenCR HIE Integration](https://github.com/mherman22/OpenELIS-OpenCR-HIE-Setup/actions/workflows/main.yml/badge.svg)](https://github.com/mherman22/OpenELIS-OpenCR-HIE-Setup/actions/workflows/main.yml)

A Docker-based Health Information Exchange (HIE) that integrates [OpenELIS Global](https://github.com/I-TECH-UW/OpenELIS-Global-2) (Laboratory Information System), [OpenCR](https://github.com/mherman22/client-registry) (Client Registry / Master Patient Index), and [OpenHIM](http://openhim.org/) (Interoperability Layer).

## What does this do?

This setup enables **patient identity lookup across systems**:

1. A lab technician in OpenELIS searches for a patient
2. If the patient isn't found locally, OpenELIS queries the Client Registry (OpenCR) via OpenHIM
3. The patient's demographics and identifiers are imported from the central registry into OpenELIS
4. Lab results are linked to the correct patient across all facilities

This is a reference implementation for laboratory integration in an OpenHIE-based architecture.

---

## Architecture

```
┌──────────────────┐     ┌──────────────┐     ┌──────────────┐
│   OpenELIS       │     │   OpenHIM     │     │   OpenCR     │
│  (Lab System)    │────▶│ (Mediator)    │────▶│ (Client      │
│                  │     │              │     │  Registry)   │
│  Port: 443       │     │  Port: 5001   │     │  Port: 3001   │
└──────────────────┘     └──────┬───────┘     └──────┬───────┘
                                │                     │
                         ┌──────▼───────┐     ┌──────▼───────┐
                         │   MongoDB    │     │  HAPI FHIR   │
                         │  (OpenHIM)   │     │  (patients)  │
                         └──────────────┘     └──────────────┘
                                              ┌──────────────┐
                                              │Elasticsearch │
                                              │  (matching)  │
                                              └──────────────┘
```

### Services

| Service | Image | Port | Purpose |
|---------|-------|------|---------|
| OpenELIS Frontend | `itechuw/openelis-global-2-frontend-dev:develop` | 443 | Lab system web UI |
| OpenELIS Backend | `itechuw/openelis-global-2-dev:develop` | 8443 | Lab system API |
| OpenELIS Database | `postgres:14.4` | 5432 | Lab data |
| OpenCR | `ghcr.io/mherman22/client-registry:ui-rewrite` | 3001 | Patient matching and golden records |
| OpenCR HAPI FHIR | `hapiproject/hapi:v5.5.1` | 8087 | Patient storage for OpenCR |
| Elasticsearch | `intrahealth/elasticsearch:latest` | 9200 | Patient matching index |
| OpenHIM Core | `jembi/openhim-core:v7.1.0` | 5001 | Interoperability layer |
| OpenHIM Console | `jembi/openhim-console:v1.15.0` | 9000 | OpenHIM admin UI |
| External FHIR API | `hapiproject/hapi:v6.6.0-tomcat` | 8444 | Shared Health Record |
| MongoDB | `mongo:3.4` | — | OpenHIM data store |
| Nginx Proxy | `nginx:1.15-alpine` | 80/443 | Reverse proxy for OpenELIS |

---

## Prerequisites

- **Docker** (v20+) with Docker Compose v2: [Install Docker](https://docs.docker.com/engine/install/)
- **Git** with Git LFS: [Install Git LFS](https://docs.github.com/en/repositories/working-with-files/managing-large-files/installing-git-large-file-storage)
- **8 GB RAM** minimum (16 GB recommended)
- **Ports 80, 443, 3001, 5001, 9000** available

---

## Quick Start

```bash
# 1. Clone with LFS support
git clone https://github.com/mherman22/OpenELIS-OpenCR-HIE-Setup.git
cd OpenELIS-OpenCR-HIE-Setup
git lfs pull

# 2. Set OpenCR to fresh install mode
# Edit configs/opencr/config.json — set "installed": false under "app"

# 3. Start all services
docker compose -f openelis-opencr-hie-docker-compose.yml up -d

# 4. Wait for services to be ready (~3-5 minutes)
docker compose -f openelis-opencr-hie-docker-compose.yml ps
```

### Access the services

| Service | URL | Credentials |
|---------|-----|-------------|
| OpenELIS | https://localhost/login | `admin` / `adminADMIN!` |
| OpenHIM Console | http://localhost:9000 | `root@openhim.org` / `openhim` |
| OpenCR (CRUX UI) | https://localhost:3001/crux/#/login | `root@intrahealth.org` / `intrahealth` |
| External FHIR API | https://localhost:8444/fhir | — |

> **Note:** After first startup, restart the streaming pipeline to ensure data flows correctly:
> ```bash
> docker restart streaming-pipeline
> ```

---

## Configuration

### Environment Variables (`.env`)

```env
TAG=nightly
HOST_URL=localhost
COMPOSE_PROJECT_NAME=sigdep3

# Client Registry connection (used by OpenMRS/SigDep3)
CLIENTREGISTRY_SERVERURL=http://openhim-core:5001/CR/fhir
CLIENTREGISTRY_USERNAME=sigdep3
CLIENTREGISTRY_PASSWORD=sigdep3
CLIENTREGISTRY_IDENTIFIERROOT=http://clientregistry.org/openmrs
```

For remote deployment, set `HOST_URL` to your server's domain.

### Configuration Files

```
configs/
├── opencr/
│   ├── config.json              # OpenCR main config (FHIR, ES, mediator, clients)
│   ├── decisionRules.json       # Patient matching rules
│   ├── mediator.json            # OpenHIM mediator registration
│   └── PatientRelationship.json # Elasticsearch field mapping
├── openelis/                    # OpenELIS properties and SSL certs
├── openhim/                     # OpenHIM channel/client import
├── openhim-console/             # OpenHIM Console config
├── hapi/                        # HAPI FHIR application.yaml
├── traefik/                     # Traefik reverse proxy (for remote deploy)
├── nginx/                       # Nginx proxy for OpenELIS
└── streaming-pipeline/          # FHIR data pipeline config
```

### Key Configuration Notes

**OpenCR:** Set `app.installed: false` on first run to trigger initial setup. After first run, it auto-sets to `true`.

**OpenHIM:** The config importer (`openhim-config`) auto-loads channels and clients on startup. If you need to re-import, remove the config container and restart.

**OpenELIS:** SSL certificates are generated by the `certs` init container. The database is initialized from the `init-scripts/` directory.

---

## Development

### Starting specific services

```bash
# Start only OpenCR stack (for testing client registry)
docker compose -f openelis-opencr-hie-docker-compose.yml up -d opencr opencr-fhir es

# Start only OpenELIS stack
docker compose -f openelis-opencr-hie-docker-compose.yml up -d oe.openeliss.org database certs proxy

# Start the full HIE
docker compose -f openelis-opencr-hie-docker-compose.yml up -d
```

### Viewing logs

```bash
# All services
docker compose -f openelis-opencr-hie-docker-compose.yml logs -f

# Specific service
docker compose -f openelis-opencr-hie-docker-compose.yml logs -f opencr

# Check service health
docker compose -f openelis-opencr-hie-docker-compose.yml ps
```

### Resetting OpenCR

```bash
docker stop opencr opencr-fhir es
docker rm opencr opencr-fhir es
docker volume rm sigdep3_opencr-data sigdep3_es-data sigdep3_opencr-fhir-data
# Set "installed": false in configs/opencr/config.json
docker compose -f openelis-opencr-hie-docker-compose.yml up -d opencr opencr-fhir es
```

### Resetting everything

```bash
docker compose -f openelis-opencr-hie-docker-compose.yml down -v
# Set "installed": false in configs/opencr/config.json
docker compose -f openelis-opencr-hie-docker-compose.yml up -d
```

---

## Testing

### Postman Tests

The `.postman/` directory contains test collections for validating the integration:

```bash
# Install Newman (Postman CLI)
npm install -g newman

# Run the general test suite
newman run .postman/1-general-tests.json --insecure
```

### Preloading Test Data

```bash
cd test
newman run postman_collection.json \
  -e postman_environment.json \
  --iteration-data pims_rule_test_dataset.csv \
  --insecure
```

---

## Remote Deployment (Ansible)

For deploying to a remote server:

```bash
# 1. Install Ansible
# https://docs.ansible.com/ansible/latest/installation_guide/installation_distros.html

# 2. Configure inventory
# Edit deployment/inventory.ini with your server addresses

# 3. Deploy
cd deployment
ansible-playbook -i inventory.ini deployment.yml
```

Ensure your SSH public key is on the remote server and update `ansible_ssh_private_key_file` in the inventory.

### SSL Certificates (Remote)

For production deployments with a domain:

```bash
# Generate Let's Encrypt certificates
docker compose -f certbot-compose.yml up
```

---

## Elasticsearch Plugin

OpenCR uses a custom Elasticsearch string-similarity plugin for probabilistic matching (Jaro-Winkler, Damerau-Levenshtein). The plugin is pre-installed in the `intrahealth/elasticsearch` image.

For local builds:

```bash
cd esplugin/string-similarity
unzip string-similarity-scoring-0.0.6-es7.9.1.zip
```

---

## Troubleshooting

### OpenELIS shows connection errors to Client Registry
Check that OpenHIM is running and the client registry channel is configured:
```bash
docker logs openhim-core 2>&1 | tail -20
docker logs opencr 2>&1 | tail -20
```

### OpenCR shows empty patient list
Verify Elasticsearch is healthy:
```bash
curl http://localhost:9200/_cluster/health?pretty
```

### Streaming pipeline fails
The pipeline depends on OpenMRS (SigDep3) being fully started. Restart it:
```bash
docker restart streaming-pipeline
```

### Port conflicts
If ports 80/443 are in use, stop conflicting services or modify port mappings in the compose file.

### Permissions error on `.db` folder
```bash
sudo chmod -R 777 .db
```

---

## CI/CD

GitHub Actions runs on every push and PR to `main`:

1. **Pulls all container images**
2. **Starts the core HIE stack** (OpenMRS, OpenHIM, OpenCR, Elasticsearch)
3. **Waits for services to be healthy**
4. **Runs Postman integration tests** via Newman
5. **Reports results** and tears down

See [`.github/workflows/main.yml`](.github/workflows/main.yml).

---

## Project Structure

```
OpenELIS-OpenCR-HIE-Setup/
├── openelis-opencr-hie-docker-compose.yml  # Main compose file
├── certbot-compose.yml                      # SSL cert generation
├── .env                                     # Environment variables
├── configs/                                 # Service configurations
│   ├── opencr/                              # OpenCR config, rules, mappings
│   ├── openelis/                            # OpenELIS properties
│   ├── openhim/                             # OpenHIM channels/clients
│   └── ...                                  # Other service configs
├── deployment/                              # Ansible playbooks
├── esplugin/                                # ES string-similarity plugin
├── init-scripts/                            # Database init SQL
├── .postman/                                # Integration test collections
├── test/                                    # Test data and scripts
└── .github/workflows/                       # CI configuration
```

---

## Related Projects

- [OpenCR (Client Registry)](https://github.com/mherman22/client-registry) — Forked with bug fixes and UI modernization
- [OpenELIS Global](https://github.com/I-TECH-UW/OpenELIS-Global-2) — Laboratory Information System
- [OpenHIM](http://openhim.org/) — Health Information Mediator
- [SEDISH Haiti HIE](https://github.com/charess-org/sedish) — Haiti HIE deployment using OpenCR

---

## License

[Apache License 2.0](LICENSE)
