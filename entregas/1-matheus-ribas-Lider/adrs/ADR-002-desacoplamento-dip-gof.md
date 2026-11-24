# ADR-002: Estratégia de Desacoplamento de Código com DIP e Padrões GoF

- **Status:** Aprovado
- **Data:** 2026-10-05
- **Responsável:** Matheus Eduardo Ribas

## Contexto
A auditoria métrica da Sprint 2 (Robert C. Martin) encontrou o pacote `PersistenciaInfra` na **Zona da Dor (D = 0,86)** e desvios em `StreamingIngestao` (D = 0,50) e `AnaliseRegrasNegocio` (D = 0,20). As regras de negócio dependiam diretamente de classes concretas como `ScadaConnector`, `KafkaEventBroker` e `TimeSeriesRepository`.

## Decisão

1. **DIP:** extrair as portas `ITelemetria`, `IPersistencia`, `IIntegracaoExterna` e `IAnaliseUseCase` para o pacote de regras (P1) e `IEventos` e `IStreamPartitioner` para o pacote de streaming (P2). A infraestrutura (P3) passa a implementá-las.
2. **OCP:** novos algoritmos e novos conectores entram por polimorfismo, sem alterar as classes mestres.
3. **Padrões GoF:**

| Padrão | Categoria | Onde | Problema resolvido |
|---|---|---|---|
| Factory Method | Criacional | `ingestao.ConectorFactory` | Criar conectores (MQTT, SCADA XML) sem acoplar a classes concretas |
| Adapter | Estrutural | `persistenciainfra.ScadaAdapter` / `LegadoAdapter` | Traduzir XML/EDI legado para os DTOs JSON do domínio |
| Strategy | Comportamental | `autonomia.EstrategiaRedistribuicao` | Trocar o algoritmo de redistribuição de carga sem if/else |

## Consequências e Trade-offs
Os três pacotes primários ficam dentro da Fitness Function **D ≤ 0,1**: P1 = 0,09, P2 = 0,00 e P3 = 0,00 (recontagem pelo desenho, ver relatório de métricas). O P3 sai da Zona da Dor (0,86 → 0,00).
O domínio fica testável sem Kafka, banco ou SCADA.
Mais interfaces e indireções, documentadas no C4 Nível 3.
As regras são protegidas pelas Fitness Functions (ArchUnit).
