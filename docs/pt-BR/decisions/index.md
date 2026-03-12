# Registros de Decisões de Arquitetura

Decisões técnicas fundamentais por trás do AstraDNS, documentadas para transparência e referência futura.

## Índice de ADRs

| ADR | Título | Status | Resumo |
|-----|--------|--------|--------|
| [001](adr-001.md) | Interceptação do Caminho de Dados | Aceito | Padrão NodeLocal DNS com encaminhamento CoreDNS |
| [002](adr-002.md) | Escopo de Política no MVP | Aceito | Escopo apenas por namespace; escopos por workload/domínio adiados |
| [003](adr-003.md) | Modelo de Métricas | Aceito | Métricas apenas por nó no MVP; por namespace requer caminho de dados direto |
| [004](adr-004.md) | Estratégia de Failover | Aceito | Failover nativo do engine no MVP; circuit breaker no agent pós-MVP |
| [005](adr-005.md) | Nomenclatura e Posicionamento | Aceito | "AstraDNS" com grupo de API `dns.astradns.com` |
| [006](adr-006.md) | Enriquecimento de Namespace | Aceito | Aceitar limitação no MVP; pod informer pós-MVP |
| [007](adr-007.md) | Estratégia de Logging de Consultas | Aceito | Logging estruturado no agent com amostragem |
| [008](adr-008.md) | Estratégia Multi-Engine | Aceito | Interface desde o dia 1; apenas Unbound no MVP |

Os ADRs de origem ficam em `astradns-docs/adrs/` e são publicados aqui como páginas versionadas.
