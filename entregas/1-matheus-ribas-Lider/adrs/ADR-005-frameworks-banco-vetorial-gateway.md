# ADR-005: Seleção de Frameworks, Banco Vetorial e Stack do API Gateway

- **Status:** Aprovado
- **Data:** 2026-11-05
- **Responsável:** Matheus Eduardo Ribas

## Contexto

Precisamos fechar a stack que sustenta as decisões anteriores: resiliência (Circuit Breaker), testes de arquitetura (ArchUnit), gateways (industrial e de consulta), persistência de telemetria e uma base de conhecimento vetorial para a IA do ciclo MAPE-K.

## Decisão

| Necessidade | Escolha | Alternativas avaliadas | Justificativa |
|---|---|---|---|
| Backend | **.NET 8 (C#)** | Java, Python | Ecossistema com Polly e ArchUnitNET; bom desempenho em I/O |
| Resiliência | **Polly** (previsto) | Implementação própria | Circuit Breaker, Retry e Fallback prontos e testados |
| Testes de arquitetura | **ArchUnitNET** | NetArchTest | Mesma semântica do ArchUnit (Java) |
| API Gateway Industrial | **EMQX (MQTT) + Envoy** | Só gateway HTTP | Medidores falam MQTT; mTLS por dispositivo |
| API Gateway de Consulta | **YARP** | Ocelot, Kong | Nativo .NET, roteamento por tenant, rate limit |
| Barramento | **Apache Kafka** | RabbitMQ (incl. Streams) | Particionamento por chave com ordem por medidor, consumer groups e ecossistema (ver ADR-003) |
| Telemetria | **TimescaleDB** | InfluxDB, Cassandra | SQL padrão, compressão, agregações contínuas |
| Auditoria | **PostgreSQL append-only** | Mesmo banco da telemetria | Imutabilidade e isolamento (LGPD) |
| **Banco Vetorial** | **Qdrant** | pgvector, Pinecone | Self-hosted (dados ficam na concessionária), filtros por tenant, baixa latência |

## Consequências e Trade-offs

-Stack coerente com os padrões do template (Polly, ArchUnit).
-O banco vetorial self-hosted mantém a soberania dos dados.
-Mais componentes para operar (Kafka, Qdrant, TimescaleDB).
