# Environment Variables

## Agent

| Variable | Default | Description |
|----------|---------|-------------|
| `ASTRADNS_CONFIG_PATH` | `/etc/astradns/config/config.json` | Path to engine configuration file (or directory containing `config.json`) |
| `ASTRADNS_ENGINE_TYPE` | `unbound` | DNS engine to use: `unbound`, `coredns`, `powerdns`, or `bind` |
| `ASTRADNS_ENGINE_CONFIG_DIR` | `/var/run/astradns/engine` | Directory where the agent writes engine-specific config files |
| `ASTRADNS_LISTEN_ADDR` | `0.0.0.0:5353` | Address and port for the DNS proxy listener |
| `ASTRADNS_LOG_MODE` | `sampled` | Query logging mode: `full`, `sampled`, `errors-only`, or `off` |
| `ASTRADNS_LOG_SAMPLE_RATE` | `0.1` | Fraction of successful queries to log (when mode is `sampled`). Range: 0.0-1.0 |
| `ASTRADNS_HEALTH_ADDR` | `:8080` | Address for the health check HTTP server |
| `ASTRADNS_METRICS_ADDR` | `:9153` | Address for the Prometheus metrics HTTP server |

## Operator

| Variable | Default | Description |
|----------|---------|-------------|
| `ASTRADNS_ENGINE_TYPE` | `unbound` | Engine type for config rendering: `unbound`, `coredns`, `powerdns`, or `bind` |
| `ASTRADNS_AGENT_CONFIGMAP_NAME` | `astradns-agent-config` | Name of the ConfigMap written by the operator |
| `POD_NAMESPACE` | *(from downward API)* | Namespace where the operator is running. Used to determine ConfigMap location |
| `ASTRADNS_ENABLE_POOL_UNIQUENESS_WEBHOOK` | `false` | Enable the validating webhook that enforces one pool per namespace |

## Helm-Managed Variables

These variables are set automatically when deploying via Helm and generally should not be overridden manually:

| Variable | Set By | Context |
|----------|--------|---------|
| `ASTRADNS_ENGINE_TYPE` | `agent.engineType` | Both operator and agent |
| `POD_NAMESPACE` | Downward API | Operator deployment |
| `ASTRADNS_ENABLE_POOL_UNIQUENESS_WEBHOOK` | `webhook.enabled` | Operator deployment |
