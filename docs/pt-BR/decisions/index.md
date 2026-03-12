# Registros de Decisoes de Arquitetura

Decisoes tecnicas fundamentais por tras do AstraDNS, documentadas para transparencia e referencia futura.

## Indice de ADRs

| ADR | Titulo | Status | Resumo |
|-----|--------|--------|--------|
| [001](#adr-001-interceptacao-do-caminho-de-dados) | Interceptacao do Caminho de Dados | Aceito | Padrao NodeLocal DNS com encaminhamento CoreDNS |
| [002](#adr-002-escopo-de-politica-no-mvp) | Escopo de Politica no MVP | Aceito | Escopo apenas por namespace; escopos por workload/dominio adiados |
| [003](#adr-003-modelo-de-metricas) | Modelo de Metricas | Aceito | Metricas apenas por no no MVP; por namespace requer caminho de dados direto |
| [004](#adr-004-estrategia-de-failover) | Estrategia de Failover | Aceito | Failover nativo do engine no MVP; circuit breaker no agent pos-MVP |
| [005](#adr-005-nomenclatura-e-posicionamento) | Nomenclatura e Posicionamento | Aceito | "AstraDNS" com grupo de API `dns.astradns.com` |
| [006](#adr-006-enriquecimento-de-namespace) | Enriquecimento de Namespace | Aceito | Aceitar limitacao no MVP; pod informer pos-MVP |
| [007](#adr-007-estrategia-de-logging-de-consultas) | Estrategia de Logging de Consultas | Aceito | Logging estruturado no agent com amostragem |
| [008](#adr-008-estrategia-multi-engine) | Estrategia Multi-Engine | Aceito | Interface desde o dia 1; apenas Unbound no MVP |

---

## ADR-001: Interceptacao do Caminho de Dados

**Status:** Aceito

**Decisao:** Usar o padrao NodeLocal DNS -- o AstraDNS Agent se conecta a um endereco link-local (`169.254.20.11`) em cada no, e o CoreDNS encaminha consultas externas para ele.

**Opcoes avaliadas:**

| Opcao | Vantagens | Desvantagens |
|-------|-----------|--------------|
| Interceptacao iptables/nftables | Transparente, sem alteracoes no CoreDNS | Fragil, conflitos com CNI, dificil de depurar |
| Injecao de sidecar | Isolamento por pod | Alta sobrecarga, complexidade de mutating webhook |
| Modificar ConfigMap do CoreDNS | Simples, padrao comprovado | Requer job pos-instalacao |
| **NodeLocal DNS (escolhido)** | **Comprovado, isolamento por no, baixo risco** | **Requer configuracao link-local** |

**Consequencias:** Cada no tem seu proprio cache e resolver. Uma falha em um no nao afeta os outros.

---

## ADR-002: Escopo de Politica no MVP

**Status:** Aceito

**Decisao:** Suportar apenas escopo por namespace no MVP. Escopos por workload e por dominio sao adiados.

**Insight chave:** O namespace nao pode ser resolvido a partir do IP de origem no caminho de dados de encaminhamento do CoreDNS. CNIs alocam IPs por no, nao por namespace. A aplicacao real por namespace requer o caminho de dados direto (pods consultam o agent diretamente).

**Consequencias:** O CRD `ExternalDNSPolicy` esta pronto na API, mas a aplicacao e adiada para uma versao futura que implemente o caminho de dados direto.

---

## ADR-003: Modelo de Metricas

**Status:** Aceito

**Decisao:** As metricas do MVP sao apenas por no. Metricas por namespace requerem o caminho de dados direto.

**Consideracoes de cardinalidade:**

- Labels de `domain` sao perigosos (cardinalidade ilimitada)
- Labels de `pod` sao arriscados (milhares de series)
- Labels de `node` sao seguros (limitados pelo tamanho do cluster)

**Metricas do MVP:** total de consultas, acertos/falhas de cache, latencia/saude/falhas de upstream, contagens SERVFAIL/NXDOMAIN, ciclo de vida do agent.

---

## ADR-004: Estrategia de Failover

**Status:** Aceito

**Decisao:** Abordagem em duas fases:

1. **MVP:** Confiar no failover nativo do engine (Unbound lida com failover de upstream nativamente). O agent adiciona observabilidade via health checks e metricas.
2. **Pos-MVP:** Circuit breaker no nivel do agent com estados CLOSED / OPEN / HALF_OPEN.

---

## ADR-005: Nomenclatura e Posicionamento

**Status:** Aceito

**Decisao:** Nomear o projeto "AstraDNS" (metafora de navegacao). Grupo de API: `dns.astradns.com`.

**Posicionamento em relacao a projetos relacionados:**

| Projeto | Proposito | Sobreposicao |
|---------|-----------|--------------|
| ExternalDNS | Publicar registros DNS em provedores | Nenhuma -- problema diferente |
| CoreDNS | Servidor DNS do cluster | Complementar -- AstraDNS fica atras do CoreDNS |
| node-local-dns | Addon de cache DNS | Padrao similar -- AstraDNS adiciona observabilidade e politica |

---

## ADR-006: Enriquecimento de Namespace

**Status:** Aceito

**Decisao:** Aceitar a limitacao no MVP (namespace = "unknown" nas metricas e logs). Pos-MVP: pod informer com tabela de lookup IP-para-namespace.

**Design pos-MVP:** O agent mantem uma tabela em memoria mapeando IPs de pods para namespaces, populada via um pod informer do Kubernetes. Memoria estimada: ~100 bytes por pod, entao 1.000 pods = ~100KB.

---

## ADR-007: Estrategia de Logging de Consultas

**Status:** Aceito

**Decisao:** Logging JSON estruturado no nivel do agent para stdout. Usuarios integram com seu pipeline de logs existente (Loki, ELK, CloudWatch).

**Modos de log:**

| Modo | Comportamento |
|------|---------------|
| `full` | Registra todas as consultas |
| `sampled` | Registra N% das consultas bem-sucedidas, sempre registra erros |
| `errors-only` | Registra apenas NXDOMAIN e SERVFAIL |
| `off` | Desabilita o logging de consultas |

---

## ADR-008: Estrategia Multi-Engine

**Status:** Aceito

**Decisao:** Projetar a interface `Engine` desde o dia 1. Implementar apenas Unbound no MVP e depois adicionar CoreDNS e PowerDNS.

**Fases:**

1. MVP: apenas Unbound
2. Adicionar CoreDNS + selecao de engine via valor Helm
3. Adicionar PowerDNS

**Consequencia:** O agent usa a interface Engine desde o inicio, tornando trivial adicionar novos engines sem alterar as camadas de proxy, metricas ou logging.
