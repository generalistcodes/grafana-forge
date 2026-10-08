# Grafana Forge

**One command. Zero manual Grafana configuration. A developer's observability Swiss Army knife.**

Grafana Forge turns a Linux/Docker machine into a self-configuring standalone observability environment.

It generates the stack, detects free ports, provisions Grafana data sources and dashboards, installs optional Grafana plugins, validates the generated configuration, starts the services, probes their health, and provides a built-in diagnostic and repair toolkit.

The goal is simple:

> **Don't make the developer configure observability. Make the tool configure itself and explain what to do when something breaks.**

## The idea

A normal Grafana deployment leaves a surprising amount of repetitive work to the developer:

1. Install Grafana
2. Install Prometheus
3. Install exporters
4. Connect the datasource
5. Find or import dashboards
6. Configure Docker metrics
7. Configure logs
8. Configure alert rules
9. Debug targets when something is missing
10. Recreate broken configuration
11. Figure out which service is actually failing

Forge compresses that workflow into one reproducible shell utility.

```bash
./grafana-forge
```

On a Linux machine with Docker running, the default profile detects and enables:

* Host metrics
* Docker metrics
* Docker logs
* Grafana provisioning
* Prometheus alert rules

The user opens Grafana and the dashboards are already provisioned.

## What it installs

| Component     | Purpose                                                   |
| ------------- | --------------------------------------------------------- |
| Grafana       | Visualization and exploration                             |
| Prometheus    | Metrics collection and time-series storage                |
| Node Exporter | Linux CPU, RAM, filesystem, disk I/O, network and load    |
| cAdvisor      | Docker container CPU, RAM, filesystem and network metrics |
| Loki          | Local log storage                                         |
| Grafana Alloy | Docker log collection and forwarding                      |

Prometheus and Loki are configured as Grafana data sources automatically.

Dashboards, data sources and alert rules are provisioned from files rather than configured manually through the Grafana UI.

## Dashboards

### Home

The Home dashboard provides a high-level overview of the entire environment:

* Observability stack health
* Host status
* Docker status
* CPU utilization
* Memory utilization
* Disk usage
* Running containers
* Log availability

### Server

The Server dashboard covers:

* CPU busy percentage
* CPU modes
* Memory total
* Memory available
* Memory cache
* Load average
* Root filesystem usage
* Filesystem free space
* Disk read/write throughput
* Network receive/transmit throughput

### Docker

The Docker dashboard covers:

* Running containers
* Container CPU
* Container memory
* Container network I/O
* Container disk I/O
* Container lifecycle information

### Logs

The Logs dashboard provides Docker logs through:

```text
Docker socket
      |
      v
   Alloy
      |
      v
    Loki
      |
      v
   Grafana
```

## One-command workflow

Start everything:

```bash
./grafana-forge
```

Show service URLs:

```bash
./grafana-forge urls
```

Run the health check:

```bash
./grafana-forge doctor
```

A typical healthy installation looks like:

```text
Grafana Forge Doctor 0.3.1
────────────────────────────────────────
[PASS] Docker available
[PASS] Docker Compose
[PASS] Compose syntax
[PASS] Prometheus API
[PASS] Grafana API

Prometheus targets:
  cadvisor: up
  node: up
  prometheus: up

[PASS] Prometheus targets
[PASS] Datasource provisioning
[PASS] Dashboard provisioning
[PASS] Alert rules

────────────────────────────────────────
RESULT: HEALTHY
```

## Profiles

Explicit profiles are useful when you want a smaller installation.

Host metrics only:

```bash
./grafana-forge up host
```

Host and Docker metrics:

```bash
./grafana-forge up host docker
```

Full observability stack:

```bash
./grafana-forge up host docker logs
```

The default command automatically detects Docker availability and selects the appropriate profile.

### Profile dependencies

```text
host
 └── Node Exporter

docker
 └── cAdvisor

logs
 └── Alloy
      └── Loki
```

The `logs` profile requires the `docker` profile because Alloy collects Docker logs through the Docker socket.

## Automatic port detection

Forge does not assume that the standard ports are available.

For example:

```text
Grafana     3000
Prometheus  9090
Loki        3100
```

If a port is already occupied, Forge searches for an available port and generates the corresponding configuration automatically.

This means another Grafana, Prometheus, development server or unrelated application does not have to be stopped just to start Forge.

The selected ports are stored in the generated environment configuration and are reused by the generated Compose stack.

## Plugin support

Grafana plugins can be requested without editing Compose:

```bash
FORGE_GRAFANA_PLUGINS=grafana-clock-panel ./grafana-forge
```

Multiple plugins can be passed using Grafana's plugin-list syntax:

```bash
FORGE_GRAFANA_PLUGINS=grafana-clock-panel,grafana-piechart-panel ./grafana-forge
```

Forge does not require an additional plugin for Prometheus or Loki because they are core Grafana data-source plugins.

Plugins are installed through Grafana's container configuration rather than through manual UI interaction.

## Configuration

Forge writes generated runtime state locally:

```text
.env

compose.yml

prometheus.yml
loki-config.yml

config/
  config.alloy

dashboards/
  forge-home.json
  forge-server.json
  forge-docker.json
  forge-logs.json

provisioning/
  datasources/
  dashboards/

alerts/
  forge.yml
```

Useful environment variables:

```bash
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=change-me

FORGE_GRAFANA_PLUGINS=

FORGE_RETENTION=15d
FORGE_LOG_RETENTION=168h

FORGE_SCRAPE_INTERVAL=15s
FORGE_DASHBOARD_REFRESH=30s
```

Generated files are runtime artifacts. The source templates remain under `templates/`.

This separation allows Forge to regenerate the runtime configuration if files are deleted or become inconsistent.

## Diagnostics

Forge is designed to be useful when things go wrong.

The diagnostic system follows this model:

```text
Detect
  ↓
Identify
  ↓
Explain
  ↓
Suggest
  ↓
Repair
  ↓
Verify
```

The goal is not simply to report:

```text
[FAIL] Prometheus targets
```

It should explain which target failed and suggest the appropriate next command.

For example:

```text
[FAIL] Prometheus targets

cadvisor: down
node: up
prometheus: up

Suggested actions:

  ./grafana-forge repair cadvisor
  ./grafana-forge logs cadvisor

After repairing:

  ./grafana-forge doctor
```

### Doctor

```bash
./grafana-forge doctor
```

Validates:

* Docker availability
* Docker Compose
* Generated Compose configuration
* Prometheus API
* Grafana API
* Prometheus targets
* Datasource provisioning
* Dashboard files
* Provisioned dashboards
* Alert rules
* Generated configuration

The doctor command should be the first command to run when something is not working correctly.

### Debug

```bash
./grafana-forge debug
```

The debug command provides a deeper inspection of:

* Operating system
* Docker
* Docker Compose
* Generated services
* Generated configuration
* Dashboards
* Provisioning
* Alert configuration

Machine-readable output:

```bash
./grafana-forge debug --json
```

This is intended for automation and future integration with CI or external diagnostic tooling.

### Configuration inspection

```bash
./grafana-forge config
```

Shows:

* Active profiles
* Generated ports
* Grafana configuration
* Requested plugins
* Metrics retention
* Log retention
* Scrape interval
* Dashboard refresh interval
* Generated configuration locations

## Repair

Forge can recreate the entire stack:

```bash
./grafana-forge repair
```

Or repair a specific service:

```bash
./grafana-forge repair grafana
```

```bash
./grafana-forge repair prometheus
```

```bash
./grafana-forge repair cadvisor
```

```bash
./grafana-forge repair node-exporter
```

The repair workflow is intended to be more than a simple container restart.

The expected repair flow is:

```text
Check generated configuration
        ↓
Regenerate missing configuration
        ↓
Validate Compose
        ↓
Recreate requested service
        ↓
Probe service
        ↓
Check Prometheus targets
        ↓
Report remaining problems
```

For example, if:

```text
alerts/forge.yml
```

is missing, Forge should detect that condition and regenerate the required configuration rather than producing a confusing `grep: No such file or directory` error.

## Logs

Inspect a service:

```bash
./grafana-forge logs grafana
```

```bash
./grafana-forge logs prometheus
```

```bash
./grafana-forge logs cadvisor
```

```bash
./grafana-forge logs alloy
```

```bash
./grafana-forge logs loki
```

This is especially useful when a target is reported as `down`.

For example:

```text
cadvisor: down
```

should be followed by:

```bash
./grafana-forge logs cadvisor
```

rather than requiring the developer to manually discover the container name.

## Backup

Create a backup:

```bash
./grafana-forge backup
```

This creates a compressed backup of generated configuration, dashboards, provisioning and alert rules.

Because the generated `.env` can contain credentials, backups should be treated as sensitive files.

For production environments, credentials should be supplied through an appropriate secret-management mechanism rather than relying on the generated default.

## Reset

Remove the generated runtime environment:

```bash
./grafana-forge reset
```

This is useful when you want to start with a clean configuration.

Reset is intentionally destructive. Use it when you want Forge to regenerate the environment from scratch.

## Alert rules

Forge provisions Prometheus alert rules automatically.

The initial rules cover conditions such as:

* Prometheus targets becoming unavailable
* High CPU utilization
* High memory utilization
* Low filesystem space
* Container restart activity

The alert rules are generated from tracked templates:

```text
templates/
└── alerts/
    └── forge.yml
```

and written into the runtime configuration:

```text
alerts/
└── forge.yml
```

This keeps alert configuration version-controlled while allowing the generated runtime environment to be recreated.

## Configuration as code

Forge deliberately separates source templates from generated runtime state.

```text
Source
  |
  +-- templates/
  |     +-- dashboards/
  |     +-- alerts/
  |
  +-- grafana-forge
          |
          v
      Generated state
          |
          +-- compose.yml
          +-- prometheus.yml
          +-- loki-config.yml
          +-- config/
          +-- dashboards/
          +-- provisioning/
          +-- alerts/
```

This provides two useful properties:

1. The repository stays clean and reproducible.
2. The runtime environment can be regenerated when files are missing or corrupted.

## Design principles

### Configuration as code

Grafana provisioning files are generated instead of relying on UI state.

This makes the stack reproducible and version-controllable.

### Fail before startup

Forge validates:

```bash
docker compose config
```

before starting services.

A broken generated Compose file should fail during configuration rather than after a partially started stack.

### Self-description

The shell utility can explain the generated environment with:

```bash
./grafana-forge config
```

```bash
./grafana-forge debug
```

```bash
./grafana-forge doctor
```

### Self-diagnosis

Forge does not only ask:

> Is the stack running?

It asks:

> Is every important component reachable, configured and producing the expected data?

This distinction matters because a running Docker container does not necessarily mean that observability is working.

### Guided repair

A failed check should provide an actionable next step.

```text
FAIL
 ↓
Explain the failure
 ↓
Suggest command
 ↓
Repair
 ↓
Verify
```

### Portable defaults

The core stack is local and standalone.

No Grafana Cloud account is required.

### Optional complexity

Docker metrics and logs are enabled automatically when Docker is available, but can also be explicitly selected by profile.

The developer can therefore choose between a minimal stack and a complete local observability environment.

## Security notes

The default stack is intended for local/developer and trusted internal environments.

* Grafana and Prometheus are exposed on localhost through generated host ports.
* cAdvisor requires privileged access and host mounts to inspect containers.
* Alloy reads the Docker socket to collect logs.
* The generated Grafana password is deterministic for the installation directory unless explicitly supplied.
* Generated `.env` files contain configuration and may contain credentials.
* Backup archives may therefore contain credentials.
* For shared or production environments, place the stack behind an authenticated reverse proxy and use managed secrets.
* Review Docker socket and privileged-container access before using the stack on security-sensitive machines.

## Project structure

```text
grafana-forge/

├── grafana-forge
├── templates/
│   ├── dashboards/
│   └── alerts/
├── docs/
│   └── article.md
├── LICENSE
└── README.md
```

Runtime configuration is generated beside the script:

```text
grafana-forge/

├── grafana-forge
├── templates/
├── docs/
│
├── .env
├── compose.yml
├── prometheus.yml
├── loki-config.yml
├── config/
├── dashboards/
├── provisioning/
└── alerts/
```

The generated state can be inspected, copied, archived or removed independently from the source templates.

## Why this is different

Forge is intentionally not another dashboard collection.

It treats observability as an **executable environment**.

```text
Detect
  ↓
Generate
  ↓
Validate
  ↓
Start
  ↓
Probe
  ↓
Provision
  ↓
Diagnose
  ↓
Suggest
  ↓
Repair
  ↓
Verify
```

The shell script becomes the control plane.

Grafana becomes the interface.

Prometheus becomes the metrics engine.

Loki becomes the log engine.

Alloy becomes the collection layer.

And the diagnostic system connects everything together.

The result is a developer tool that tries to answer not only:

> **"How do I see my system?"**

but also:

> **"What is broken, why is it broken, and what command should I run next?"**

That is the core idea behind Grafana Forge.
