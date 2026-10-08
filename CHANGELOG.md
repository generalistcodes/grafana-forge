# Changelog

## 0.3.0 - Observability Swiss Army Knife

- Automatic profile detection for Linux + Docker.
- Automatic Node Exporter host monitoring.
- Automatic cAdvisor Docker monitoring using GHCR.
- Automatic Loki + Alloy Docker log pipeline.
- Automatic Grafana Prometheus/Loki datasource provisioning.
- Auto-provisioned Home, Server, Docker and Logs dashboards.
- Built-in Prometheus alert rules for target health, CPU, memory, disk and containers.
- `doctor` validates APIs, targets, provisioning and dashboards.
- `debug` and `debug --json` diagnostic output.
- `config` configuration inspection.
- `repair [service]` service recovery workflow.
- `logs [service]` log following shortcut.
- `backup` generated configuration export.
- `FORGE_GRAFANA_PLUGINS` plugin installation support.
- Configurable metric/log retention and scrape/dashboard refresh intervals.
- Grafana and Prometheus container health checks.
- Explicit `docker compose config` preflight before startup.
