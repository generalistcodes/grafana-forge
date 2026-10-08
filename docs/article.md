# One Command, Zero Port Conflicts: Building Grafana Forge

Every time I start work on a new server or client environment, I want the same thing: visibility.

I want to know whether the CPU is saturated, whether memory is disappearing, whether the disk is filling up, whether a container is restarting, and whether an application is producing suspicious logs.

The problem is not that Grafana is difficult. The problem is that the surrounding setup is repetitive.

Install Grafana. Install Prometheus. Add an exporter. Configure the datasource. Import a dashboard. Configure Docker metrics. Configure logs. Then, when something does not work, spend another twenty minutes finding out which part of the chain is broken.

I built **Grafana Forge** to remove that repetition.

## The goal

The goal is deliberately simple:

```bash
./grafana-forge
```

That command should turn a Linux/Docker machine into a useful local observability environment without requiring the developer to click through Grafana.

The stack automatically creates:

```text
                    Grafana
                       |
              +--------+--------+
              |                 |
         Prometheus           Loki
              |                 ^
       +------+-------+         |
       |              |       Alloy
 Node Exporter      cAdvisor    ^
       |              |         |
       +------+-------+---------+
              |
          Host / Docker
```

Node Exporter provides operating-system metrics. cAdvisor provides container metrics. Alloy forwards Docker logs to Loki. Prometheus stores metrics. Grafana presents the result.

## No manual datasource setup

One of the most important decisions was to treat Grafana configuration as code.

Forge generates a datasource provisioning file pointing Grafana at Prometheus and, when logs are enabled, Loki.

It also generates a dashboard provider and places dashboard JSON files into the provisioned directory.

The result is deterministic:

```text
script
  -> generated configuration
  -> Docker Compose
  -> Grafana provisioning
  -> dashboards
```

There is no requirement to import a dashboard ID from the Internet.

## The dashboard is the product

The default server dashboard is designed around the questions I actually ask when debugging a machine:

### Is the CPU the problem?

The dashboard shows total CPU busy time and CPU modes so I can distinguish normal application load from system time and I/O wait.

### Is memory the problem?

Total, available and cached memory are visible together instead of reducing everything to a single percentage.

### Is the disk the problem?

Filesystem capacity is only half the story. Disk throughput is also shown so a machine that has plenty of free space but is experiencing heavy I/O is still visible.

### Is the network the problem?

Receive and transmit throughput are available without building another dashboard.

### Is Docker the problem?

cAdvisor adds container-level CPU, memory, network and filesystem metrics.

## The shell script is also a diagnostic tool

This is the part that makes Forge more than a Compose wrapper.

```bash
./grafana-forge doctor
```

The command validates the environment rather than simply checking whether containers exist.

It checks the generated Compose configuration, Prometheus readiness, Grafana readiness, Prometheus targets, datasource provisioning, dashboards and alert rules.

The idea is that a developer should be able to answer:

> Is my observability stack actually working?

with one command.

## Debug mode

```bash
./grafana-forge debug
```

This produces a compact environment report that can be pasted into an issue or sent to another engineer.

For automation:

```bash
./grafana-forge debug --json
```

The human-readable output is for people. The JSON output is for tooling.

## Repair instead of reinstall

Another useful command is:

```bash
./grafana-forge repair grafana
```

If one service becomes unhealthy, Forge can recreate that service and run its diagnostics again.

That is much faster than deleting the whole environment.

## Plugins without manual installation

Grafana supports plugin installation through configuration. Forge exposes that through one environment variable:

```bash
FORGE_GRAFANA_PLUGINS=grafana-clock-panel ./grafana-forge
```

The important design choice is that plugins remain optional. Prometheus and Loki are already core Grafana data-source plugins, so Forge does not add unnecessary plugin dependencies.

## Why generate the configuration?

I want the infrastructure to explain itself.

After Forge runs, the directory contains the generated Compose file, Prometheus configuration, Loki configuration, provisioning manifests, dashboards and alert rules.

Nothing important is hidden inside a sequence of UI clicks.

If I want to inspect the configuration:

```bash
./grafana-forge config
```

If I want to archive it:

```bash
./grafana-forge backup
```

The runtime becomes inspectable and reproducible.

## Enterprise thinking, developer-sized

I call it a developer Swiss Army knife because the stack is intentionally small, but the workflow follows enterprise observability principles:

- configuration as code
- reproducible provisioning
- explicit health checks
- metrics and logs in one interface
- alert rules included by default
- predictable storage
- deterministic local state
- diagnostics before guesswork
- optional components instead of mandatory complexity

It is not trying to replace a large enterprise monitoring platform.

It is trying to give one developer the first 80% of useful observability in minutes.

## The hackathon challenge

The interesting question is not:

> How quickly can I start Grafana?

It is:

> How much operational knowledge can I encode into one shell command without making the command itself complicated?

That is the experiment behind Grafana Forge.

The shell script is the control plane.

Prometheus is the metrics engine.

Loki is the log store.

Grafana is the interface.

The generated configuration is the contract between them.

And `doctor` is the proof that the contract is still working.

## Final workflow

The ideal workflow is now:

```bash
./grafana-forge
```

```bash
./grafana-forge urls
```

Open Grafana.

Look at the Server dashboard.

Look at Docker.

Search the logs.

If something looks wrong:

```bash
./grafana-forge doctor
```

Then:

```bash
./grafana-forge debug
```

Then, if necessary:

```bash
./grafana-forge repair <service>
```

That is the whole idea: **one command to build the observability environment, and one command to prove that it works.**
