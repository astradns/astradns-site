# Registros de Decisiones de Arquitectura

Decisiones técnicas clave detrás de AstraDNS, documentadas por transparencia y referencia futura.

## Índice de ADRs

| ADR | Título | Estado | Resumen |
|-----|--------|--------|---------|
| [001](adr-001.md) | Intercepción de la Ruta de Datos | Aceptado | Patrón NodeLocal DNS con reenvío de CoreDNS |
| [002](adr-002.md) | Alcance de Políticas en MVP | Aceptado | Solo alcance de namespace; alcances de carga de trabajo/dominio diferidos |
| [003](adr-003.md) | Modelo de Métricas | Aceptado | Solo métricas a nivel de nodo en MVP; namespace requiere ruta de datos directa |
| [004](adr-004.md) | Estrategia de Failover | Aceptado | Failover nativo del motor en MVP; circuit breaker del agent post-MVP |
| [005](adr-005.md) | Nombre y Posicionamiento | Aceptado | "AstraDNS" con grupo de API `dns.astradns.com` |
| [006](adr-006.md) | Enriquecimiento de Namespace | Aceptado | Aceptar limitación en MVP; pod informer post-MVP |
| [007](adr-007.md) | Estrategia de Registro de Consultas | Aceptado | Registro estructurado a nivel de agent con muestreo |
| [008](adr-008.md) | Estrategia Multi-Motor | Aceptado | Interfaz desde el día 1; solo Unbound en MVP |
| [009](adr-009.md) | Perfiles de Topología del Agent | Aceptado | Dos perfiles (node-local, central) para trade-off costo vs latencia |

Los ADRs de origen viven en `astradns-docs/adrs/` y se publican aquí como páginas versionadas.
