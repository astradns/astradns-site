# Variables de Entorno

## Agent

| Variable | Por Defecto | Descripción |
|----------|-------------|-------------|
| `ASTRADNS_CONFIG_PATH` | `/etc/astradns/config/config.json` | Ruta al archivo de configuración del motor (o directorio que contiene `config.json`) |
| `ASTRADNS_ENGINE_TYPE` | `unbound` | Motor DNS a utilizar: `unbound`, `coredns`, `powerdns` o `bind` |
| `ASTRADNS_ENGINE_CONFIG_DIR` | `/var/run/astradns/engine` | Directorio donde el agent escribe los archivos de configuración específicos del motor |
| `ASTRADNS_LISTEN_ADDR` | `0.0.0.0:5353` | Dirección y puerto para el listener del proxy DNS |
| `ASTRADNS_LOG_MODE` | `sampled` | Modo de registro de consultas: `full`, `sampled`, `errors-only` u `off` |
| `ASTRADNS_LOG_SAMPLE_RATE` | `0.1` | Fracción de consultas exitosas a registrar (cuando el modo es `sampled`). Rango: 0.0-1.0 |
| `ASTRADNS_HEALTH_ADDR` | `:8080` | Dirección para el servidor HTTP de verificación de salud |
| `ASTRADNS_METRICS_ADDR` | `:9153` | Dirección para el servidor HTTP de métricas de Prometheus |

## Operator

| Variable | Por Defecto | Descripción |
|----------|-------------|-------------|
| `ASTRADNS_ENGINE_TYPE` | `unbound` | Tipo de motor para el renderizado de configuración: `unbound`, `coredns`, `powerdns` o `bind` |
| `ASTRADNS_AGENT_CONFIGMAP_NAME` | `astradns-agent-config` | Nombre del ConfigMap escrito por el operator |
| `POD_NAMESPACE` | *(desde la Downward API)* | Namespace donde se ejecuta el operator. Usado para determinar la ubicación del ConfigMap |
| `ASTRADNS_ENABLE_POOL_UNIQUENESS_WEBHOOK` | `false` | Habilitar el webhook de validación que impone un pool por namespace |

## Variables Administradas por Helm

Estas variables se establecen automáticamente al desplegar mediante Helm y generalmente no deben ser sobreescritas manualmente:

| Variable | Establecida Por | Contexto |
|----------|----------------|----------|
| `ASTRADNS_ENGINE_TYPE` | `agent.engineType` | Tanto en operator como en agent |
| `POD_NAMESPACE` | Downward API | Deployment del operator |
| `ASTRADNS_ENABLE_POOL_UNIQUENESS_WEBHOOK` | `webhook.enabled` | Deployment del operator |
