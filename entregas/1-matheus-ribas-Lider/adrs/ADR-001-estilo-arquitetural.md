# ADR-001: Arquitetura modular orientada a eventos

- **Status:** Aprovado
- **Data:** 2026-09-10
- **Responsável:** Matheus Eduardo Ribas

## Contexto

O Gird the Grid precisa ingerir até 19.000 eventos/s (1,9M clientes), manter 99,99% de disponibilidade e atualizar dashboards em tempo quase real. A Análise de Nexo apontou a **topologia de dados** como risco central: em um banco relacional monolítico, a escrita da telemetria bloquearia a leitura dos dashboards e dos relatórios de intrusão. A equipe tem 5 pessoas, então o custo operacional também pesa.

## Opções consideradas

| Opção | Vantagens | Desvantagens |
|---|---|---|
| Monólito em camadas (request-response) | Simples de construir e implantar | Escrita e leitura disputam o mesmo banco; sem absorção de picos |
| Microsserviços | Escala e implantação independentes | Alta complexidade operacional; risco de "monólito distribuído" |
| **Arquitetura modular orientada a eventos** | Fronteiras claras (DDD), Kafka absorve picos, módulos podem virar serviços depois | Exige disciplina nas fronteiras e observabilidade |

## Decisão

Adotar uma **arquitetura modular orientada a eventos**: módulos alinhados aos Bounded Contexts e comunicação assíncrona via **Apache Kafka particionado por tenant**. Ingestão, Worker de Regras, Consulta e MAPE-K são unidades de implantação distintas. No Worker, o consumidor de eventos, o motor de regras e seus adaptadores são implantados juntos, conforme o C4 nível 3.

## Consequências e Trade-offs

- ✅ Desacoplamento temporal: picos ficam retidos no barramento.
- ✅ Isolamento de falhas por tenant e replay para auditoria.
- ✅ Evolução gradual: um módulo pode ser extraído para microsserviço sem reescrita.
- ⚠️ Consistência eventual entre ingestão e dashboard.
- ⚠️ Necessidade de monitorar lag de consumidores e DLQ.
