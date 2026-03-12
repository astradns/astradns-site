# Registros de Decisiones de Arquitectura

Decisiones técnicas clave detrás de AstraDNS, documentadas por transparencia y referencia futura.

## Índice de ADRs

| ADR | Título | Estado | Resumen |
|-----|--------|--------|---------|
| [001](#adr-001-intercepcion-de-la-ruta-de-datos) | Intercepción de la Ruta de Datos | Aceptado | Patrón NodeLocal DNS con reenvío de CoreDNS |
| [002](#adr-002-alcance-de-politicas-en-mvp) | Alcance de Políticas en MVP | Aceptado | Solo alcance de namespace; alcances de carga de trabajo/dominio diferidos |
| [003](#adr-003-modelo-de-metricas) | Modelo de Métricas | Aceptado | Solo métricas a nivel de nodo en MVP; namespace requiere ruta de datos directa |
| [004](#adr-004-estrategia-de-failover) | Estrategia de Failover | Aceptado | Failover nativo del motor en MVP; circuit breaker del agent post-MVP |
| [005](#adr-005-nombre-y-posicionamiento) | Nombre y Posicionamiento | Aceptado | "AstraDNS" con grupo de API `dns.astradns.com` |
| [006](#adr-006-enriquecimiento-de-namespace) | Enriquecimiento de Namespace | Aceptado | Aceptar limitación en MVP; pod informer post-MVP |
| [007](#adr-007-estrategia-de-registro-de-consultas) | Estrategia de Registro de Consultas | Aceptado | Registro estructurado a nivel de agent con muestreo |
| [008](#adr-008-estrategia-multi-motor) | Estrategia Multi-Motor | Aceptado | Interfaz desde el día 1; solo Unbound en MVP |

---

## ADR-001: Intercepción de la Ruta de Datos

**Estado:** Aceptado

**Decisión:** Usar el patrón NodeLocal DNS -- el AstraDNS Agent se enlaza a una dirección link-local (`169.254.20.11`) en cada nodo, y CoreDNS reenvía las consultas externas hacia él.

**Opciones evaluadas:**

| Opción | Ventajas | Desventajas |
|--------|----------|-------------|
| Intercepción iptables/nftables | Transparente, sin cambios en CoreDNS | Frágil, conflictos con CNI, difícil de depurar |
| Inyección de sidecar | Aislamiento por pod | Alto overhead, complejidad del mutating webhook |
| Modificar ConfigMap de CoreDNS | Simple, patrón probado | Requiere job post-instalación |
| **NodeLocal DNS (elegido)** | **Probado, aislamiento por nodo, bajo riesgo** | **Requiere configuración link-local** |

**Consecuencias:** Cada nodo tiene su propio caché y resolver. Una falla en un nodo no afecta a los demás.

---

## ADR-002: Alcance de Políticas en MVP

**Estado:** Aceptado

**Decisión:** Soportar solo alcance de namespace en MVP. Los alcances a nivel de carga de trabajo y dominio se difieren.

**Hallazgo clave:** El namespace no se puede resolver desde la IP de origen en la ruta de datos de reenvío de CoreDNS. Los CNIs asignan IPs por nodo, no por namespace. La imposición real a nivel de namespace requiere la ruta de datos directa (los pods consultan al agent directamente).

**Consecuencias:** El CRD `ExternalDNSPolicy` está listo a nivel de API, pero la imposición se difiere a una versión futura que implemente la ruta de datos directa.

---

## ADR-003: Modelo de Métricas

**Estado:** Aceptado

**Decisión:** Las métricas del MVP son solo a nivel de nodo. Las métricas a nivel de namespace requieren la ruta de datos directa.

**Consideraciones de cardinalidad:**

- Las etiquetas `domain` son peligrosas (cardinalidad ilimitada)
- Las etiquetas `pod` son riesgosas (miles de series)
- Las etiquetas `node` son seguras (limitadas por el tamaño del clúster)

**Métricas del MVP:** total de consultas, aciertos/fallos de caché, latencia/salud/fallas de upstreams, conteos de SERVFAIL/NXDOMAIN, ciclo de vida del agent.

---

## ADR-004: Estrategia de Failover

**Estado:** Aceptado

**Decisión:** Enfoque en dos fases:

1. **MVP:** Depender del failover nativo del motor (Unbound maneja el failover de upstreams de forma nativa). El agent agrega observabilidad mediante verificaciones de salud y métricas.
2. **Post-MVP:** Circuit breaker a nivel de agent con estados CLOSED / OPEN / HALF_OPEN.

---

## ADR-005: Nombre y Posicionamiento

**Estado:** Aceptado

**Decisión:** Nombrar el proyecto "AstraDNS" (metáfora de navegación). Grupo de API: `dns.astradns.com`.

**Posicionamiento frente a proyectos relacionados:**

| Proyecto | Propósito | Solapamiento |
|----------|-----------|--------------|
| ExternalDNS | Publicar registros DNS en proveedores | Ninguno -- problema diferente |
| CoreDNS | Servidor DNS del clúster | Complementario -- AstraDNS se ubica detrás de CoreDNS |
| node-local-dns | Addon de caché DNS | Patrón similar -- AstraDNS agrega observabilidad y políticas |

---

## ADR-006: Enriquecimiento de Namespace

**Estado:** Aceptado

**Decisión:** Aceptar la limitación en MVP (namespace = "unknown" en métricas y logs). Post-MVP: pod informer con tabla de búsqueda IP-a-namespace.

**Diseño post-MVP:** El agent mantiene una tabla en memoria mapeando IPs de pods a namespaces, poblada mediante un pod informer de Kubernetes. Memoria estimada: ~100 bytes por pod, entonces 1,000 pods = ~100KB.

---

## ADR-007: Estrategia de Registro de Consultas

**Estado:** Aceptado

**Decisión:** Registro JSON estructurado a nivel de agent hacia stdout. Los usuarios integran con su pipeline de logs existente (Loki, ELK, CloudWatch).

**Modos de registro:**

| Modo | Comportamiento |
|------|----------------|
| `full` | Registra cada consulta |
| `sampled` | Registra N% de las consultas exitosas, siempre registra errores |
| `errors-only` | Registra solo NXDOMAIN y SERVFAIL |
| `off` | Deshabilita el registro de consultas |

---

## ADR-008: Estrategia Multi-Motor

**Estado:** Aceptado

**Decisión:** Diseñar la interfaz `Engine` desde el día 1. Implementar solo Unbound en MVP, luego agregar CoreDNS y PowerDNS.

**Fases:**

1. MVP: Solo Unbound
2. Agregar CoreDNS + selección de motor mediante valor de Helm
3. Agregar PowerDNS

**Consecuencia:** El agent usa la interfaz Engine desde el inicio, haciendo trivial agregar nuevos motores sin cambiar las capas de proxy, métricas o registro.
