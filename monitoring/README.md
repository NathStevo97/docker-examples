# Monitoring Stack

This folder contains a small Prometheus + Grafana monitoring stack for local Docker-based observability.

## Included services

- Prometheus: metrics collection and queries at http://localhost:9090
- Node Exporter: host metrics at port 9100
- cAdvisor: container metrics at port 8080
- Grafana: dashboards and visualization at http://localhost:3000

## Prerequisites

- Docker Engine installed and running
- Docker Compose v2 (`docker compose`) or Docker Compose v1 (`docker-compose`)
- Ports 3000, 8080, 9090, and 9100 available locally

## Quick start

1. From this directory, create or review the environment file:

   ```bash
   cp .env.example .env 2>/dev/null || true
   ```

   The current environment file contains the Grafana admin credentials used by the stack.

2. Start the stack:

   ```bash
   docker compose up -d
   ```

   or, for legacy Compose:

   ```bash
   docker-compose up -d
   ```

3. Open the dashboards:

   - Grafana: http://localhost:3000
   - Prometheus: http://localhost:9090

4. Default Grafana login:

   - Username: `admin`
   - Password: from `.env` or the default value in the Compose file

5. The stack includes automatic Grafana provisioning for:

   - a Prometheus datasource
   - a starter dashboard: `Node Exporter Overview`

   These files live under `grafana/provisioning` and are loaded automatically when Grafana starts.

## Useful commands

```bash
# View container status
 docker compose ps

# View logs for one service
 docker compose logs -f prometheus
 docker compose logs -f grafana

# Restart a service
 docker compose restart prometheus

# Stop the stack
 docker compose down

# Stop and remove volumes
 docker compose down -v
```

## Troubleshooting

### 1. `ERROR: for prometheus 'ContainerConfig'`

This typically happens when a stale or malformed container is left behind by older Docker Compose versions. The fix is to remove the old Prometheus container and restart:

```bash
docker ps -a
docker rm -f <stale_prometheus_container>
docker compose up -d
```

If the container name is still present in the old Compose metadata, you can also remove all stopped containers:

```bash
docker container prune
```

### 2. Prometheus is not scraping targets

Check the config file in `prometheus/prometheus.yml` and confirm the service names match the Compose service names:

- `node-exporter:9100`
- `cadvisor:8080`

Then verify the targets in Prometheus:

- Open http://localhost:9090/targets
- Look for `UP` status and recent scrape times

### 3. Grafana shows blank or no data

Confirm the Prometheus datasource is configured in `grafana/provisioning/datasources` and that Prometheus is actually healthy:

```bash
curl http://localhost:9090/-/healthy
```

If the datasource is missing, you can add one manually under Grafana UI or by provisioning files in the `grafana/provisioning` directory.

Grafana will also auto-import dashboards from `grafana/provisioning/dashboards` when the container starts. Add a JSON file there and it will appear as a dashboard automatically.

### 4. Port already in use

Check for another process listening on the expected ports:

```bash
ss -tulpn | grep -E ':3000|:8080|:9090|:9100'
```

If needed, stop the conflicting service or change the published port mapping in the Compose file.

### 5. Health checks are failing

Check the container health state:

```bash
docker compose ps
```

Inspect logs for the unhealthy service:

```bash
docker compose logs --tail=200 prometheus
```

## Adding additional exporters or apps

You can add more targets as independent services under the same `monitoring` network.

### Example: Blackbox Exporter

```yaml
services:
  blackbox-exporter:
    image: prom/blackbox-exporter:latest
    container_name: blackbox-exporter
    ports:
      - "9115:9115"
    networks:
      - monitoring
    restart: unless-stopped
```

Then add a Prometheus job in `prometheus/prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://example.com
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

### Example: Postgres Exporter

```yaml
services:
  postgres-exporter:
    image: prometheuscommunity/postgres-exporter:latest
    container_name: postgres-exporter
    environment:
      DATA_SOURCE_NAME: "postgresql://postgres:postgres@postgres:5432/postgres?sslmode=disable"
    ports:
      - "9187:9187"
    networks:
      - monitoring
    restart: unless-stopped
```

Then add the scrape target:

```yaml
  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']
```

### Example: custom app exposing metrics

If your app exposes Prometheus metrics on `/metrics`, add it to the same network and scrape it directly:

```yaml
services:
  my-app:
    image: my-app:latest
    ports:
      - "8081:8081"
    networks:
      - monitoring
    restart: unless-stopped
```

```yaml
scrape_configs:
  - job_name: 'my-app'
    static_configs:
      - targets: ['my-app:8081']
```

### Example: adding a Grafana dashboard from a JSON file

Place the dashboard JSON into `grafana/provisioning/dashboards/` and restart Grafana:

```bash
cp my-dashboard.json monitoring/grafana/provisioning/dashboards/
docker compose restart grafana
```

Grafana will detect the file and load it automatically when the provisioning system runs.

## Notes

- The Compose file uses an explicit `monitoring` Docker bridge network so services can communicate by name.
- Prometheus uses Docker service names in `prometheus.yml` rather than localhost for internal scraping.
- Volumes are used so Prometheus data and Grafana state persist across restarts.
- Health checks provide a simple way to confirm services are alive before Grafana or other dependent services begin to rely on them.
