# Setup Guide — SHPF

## Prerequisites
- Docker & Docker Compose
- Python 3.11+
- Git

## Quick Start

```bash
# Clone and enter the project
git clone <repo-url> shpf
cd shpf

# Start all services
docker compose up -d

# Verify all containers are healthy
docker compose ps
```

## Services

| Service | Port | URL |
|---|---|---|
| Airflow Webserver | 8080 | http://localhost:8080 |
| Streamlit UI | 8501 | http://localhost:8501 |
| PostgreSQL | 5432 | localhost:5432 |
| Redis | 6379 | localhost:6379 |
| Grafana | 3000 | http://localhost:3000 |
| Prometheus | 9090 | http://localhost:9090 |

## Development

```bash
# Run tests
pytest shpf/tests/

# Run specific test
pytest shpf/tests/test_detection/test_schema_drift.py -v

# Run demo
bash demo.sh
```

## Configuration
- Airflow: `LocalExecutor` (not CeleryExecutor)
- Pipeline config repo: separate local git repo at `../shpf-pipeline-config/`
- All state persisted in Docker volumes
