# ADR-003: Adoção de Ingestão Assíncrona com Apache Kafka para Telemetria IoT

- **Status:** Aprovado
- **Data:** 2026-10-20 (Revisado em 2026-11-23)
- **Responsável:** Darlan Rodrigues Schmeller (Pesquisador & Engenheiro de Resiliência e Integração)

> Este ADR formaliza a decisão arquitetural chave para a ingestão massiva de telemetria IoT do projeto **Gird the Grid**, estabelecendo os protocolos de integração entre medidores inteligentes, SCADA, camada de ingestão e os consumidores analíticos e autonômicos.

---

## 1. Contexto e Problema

O sistema **Gird the Grid** é uma plataforma multi-tenant para concessionárias de distribuição de energia elétrica com bases de clientes variando entre 100.000 e 1.900.000 unidades consumidoras (medidores inteligentes IoT) e dezenas de subestações monitoradas via SCADA.

### Requisitos Arquiteturais Críticos:
1. **Alto Throughput e Nivelamento de Carga (Load Leveling):** Dimensionamento BOTE (*Back-of-the-Envelope*) aponta um fluxo contínuo de regime nominal de ~19.000 eventos/s, com picos súbitos de até 3x (~57.000 eventos/s) em eventos de contingência climática ou oscilações de rede.
2. **Alta Disponibilidade e Desacoplamento:** Disponibilidade global de 99,99%. A indisponibilidade de bancos de dados analíticos ou serviços downstream não pode travar a ingestão nem provocar perda de pacotes dos medidores.
3. **Isolamento Multi-tenant e Ordenação Estrita:** Garantia de ordem cronológica dos eventos por medidor/subestação e isolamento por concessionária.
4. **Retenção e Reprocessamento (Replay):** Capacidade de reprocessar eventos passados para reanálise do ciclo autonômico MAPE-K, recomputação de métricas e conformidade regulatória (PRODIST/ANEEL).
5. **Comunicação Heterogênea:** Suporte a medidores IoT modernos (MQTT/TLS), sistemas SCADA/RTU legados (XML/EDI sobre TCP/HTTPS) e interfaces com operadores e serviços internos (REST/JSON e gRPC).

Uma abordagem síncrona tradicional (Medidor → API REST → Banco de Dados Time-Series) tornaria o banco o gargalo do sistema e violaria o requisito de disponibilidade (uma queda na base de dados derrubaria toda a rede de coleta).

---

## 2. Alternativas Consideradas

Três alternativas principais foram avaliadas para o núcleo de ingestão:

### Opção 1: Ingestão Síncrona via HTTP/REST direto em Banco de Dados
- **Descrição:** O API Gateway recebe as requisições HTTP dos medidores e realiza `INSERT` direto no banco Time-Series (TimescaleDB).
- **Prós:** Arquitetura simples, sem necessidade de middlewares adicionais; menor complexidade operacional imediata.
- **Contras:** Acoplamento temporal rígido; ausência de buffer para absorção de picos; esgotamento do pool de conexões do banco de dados sob carga de 19k+ req/s; falha no banco acarreta perda massiva de dados de campo e timeout nos medidores; violação dos requisitos de disponibilidade e RPO.

### Opção 2: Message Broker Baseado em Filas (RabbitMQ / AMQP)
- **Descrição:** O Gateway publica os eventos em exchanges do RabbitMQ, que roteia as mensagens para filas dedicadas dos workers.
- **Prós:** Mecanismo maduro e flexível de roteamento (exchanges direct, topic, fanout); suporte nativo a Dead Letter Exchanges (DLX) e acknowledgment por mensagem; baixo consumo de memória para filas curtas; excelente para mensageria transacional e RPC.
- **Contras:** Modelo destrutivo de leitura (a mensagem é removida da fila após o `ACK`); incapacidade nativa de replay histórico para reprocessamento; degradação de desempenho quando as filas acumulam milhões de mensagens em disco; gargalo de escalabilidade vertical sob throughput extremo de streaming contínuo.

### Opção 3: Log Distribuído de Eventos com Apache Kafka (Adotada)
- **Descrição:** O API Gateway Industrial (EMQX/Envoy) valida os dados e o Serviço de Ingestão publica em tópicos particionados do Apache Kafka, de onde múltiplos consumer groups leem de forma independente.
- **Prós:** Altíssimo throughput de gravação sequencial em disco com zero-copy; armazenamento imutável baseado em log de commit; retenção configurável por tempo ou tamanho permitindo replay de eventos; particionamento distribuído que permite escalabilidade horizontal ilimitada; suporte a múltiplos consumidores independentes lendo os mesmos eventos em ritmos próprios; garantias de ordenação por chave (`tenantId + medidorId`).
- **Contras:** Maior complexidade de implantação e operação (necessidade de KRaft / orquestração de cluster); overhead de gerenciamento de offsets e partições; modelo de consistência eventual na visualização final.

---

## 3. Matriz Comparativa Detalhada

| Critério de Avaliação | HTTP Síncrono Direto | RabbitMQ (Filas) | Apache Kafka (Log Distribuído) |
|---|---|---|---|
| **Capacidade de Throughput** | Baixa (< 5.000 req/s) | Média (~10.000 a 20.000 msg/s) | Altíssima (> 100.000 msg/s por cluster) |
| **Nivelamento de Carga (Load Leveling)** | Inexistente (pressão direta no BD) | Bom (buffer transitório em fila) | Excelente (log persistente de alta capacidade) |
| **Capacidade de Replay / Auditoria** | Inexistente no transporte | Inexistente (mensagens são apagadas) | Nativo (retenção por dias/semanas e reset de offset) |
| **Ordenação por Dispositivo** | Não garantida | Difícil sob consumidores concorrentes | Garantida estritamente por chave de partição |
| **Consumo Multiuso (Fan-out)** | Duplicação de chamadas HTTP | Réplica por fila (overhead de memória) | Múltiplos Consumer Groups sem duplicar dados |
| **Isolamento de Falhas** | Nulo (falha em cascata) | Alto (isolamento por fila) | Altíssimo (isolamento por broker e partição) |
| **Complexidade Operacional** | Baixa | Média | Média-Alta (requer monitoramento e tuning) |

---

## 4. Decisão Adotada

1. **Ingestão Primária Assíncrona via Apache Kafka:**
   - Adotar o Apache Kafka como espinha dorsal de eventos do Gird the Grid.
   - O tráfego de campo é recebido pelo **API Gateway Industrial** (EMQX para MQTT/TLS com mTLS por dispositivo, e Envoy para SCADA/HTTPS).
   - O **Serviço de Ingestão** valida os payloads contra contratos formais JSON Schema (`telemetria.v1`), normaliza a estrutura e publica os eventos no tópico particionado `telemetria.v1`.
2. **Estratégia de Chave de Particionamento:**
   - A chave de mensagem é obrigatoriamente composta por `tenantId:medidorId`. Isso garante que todos os dados de um mesmo medidor caiam na mesma partição (ordenação estrita por dispositivo) e distribui uniformemente a carga entre os nós.
3. **Tratamento de Falhas e Dead Letter Queue (DLQ):**
   - Criação do tópico `telemetria.dlq`.
   - Mensagens estruturalmente corrompidas (*poison pills*) são encaminhadas imediatamente à DLQ na borda.
   - Erros transitórios de processamento em workers passam por retry com backoff exponencial antes do descarte na DLQ.
4. **Matriz Complementar de Protocolos:**
   - **REST/JSON (OpenAPI 3):** Exclusivo para consultas síncronas do Dashboard/Web App via API Gateway de Consulta (YARP).
   - **gRPC / Protobuf:** Utilizado para comunicação síncrona interna de alto desempenho e comandos de atuação de rede (`ExecutarManobra`).
   - **RabbitMQ:** Fica reservado para integrações pontuais que exijam roteamento AMQP complexo e transações de curta duração fora do streaming de telemetria.
   - **Polly (Circuit Breaker + Retry + Fallback):** Obrigatório em todas as integrações síncronas com sistemas legados (ex.: Cadastro de Clientes).

---

## 5. Consequências e Trade-offs

### Consequências Positivas:
- **Resiliência a Picos:** A plataforma tolera elevações de carga nominais de 19.000 eventos/s para mais de 57.000 eventos/s sem degradação do gateway nem esgotamento do banco.
- **RPO ≈ 0 e Tolerância a Desastres:** Com fator de replicação $RF=3$ e `min.insync.replicas=2` com `acks=all`, garante-se durabilidade total dos dados mesmo com perda de broker.
- **Reprocessamento Independente:** O ciclo autonômico MAPE-K e o pipeline analítico podem reler semanas de dados sem interferir na operação em tempo real.
- **Consumo Concorrente e Desacoplado:** O Worker de Regras, o MAPE-K e o arquivador de auditoria leem na velocidade de seus respectivos recursos computacionais.

### Consequências Negativas e Mitigações:
- **Consistência Eventual:** Há uma latência entre a publicação no Kafka e a gravação no TimescaleDB. *Mitigação:* A latência ponta a ponta do consumidor é monitorada via SLO (meta < 1s).
- **Complexidade Operacional:** Necessidade de gerenciar cluster Kafka, monitorar lag de consumidores, retenção em disco e balanceamento de partições. *Mitigação:* Uso do KRaft (sem Zookeeper), alertas de consumer lag e políticas automáticas de limpeza de log por tempo (retenção de 7 dias).

---

## 6. Referências Técnicas

- KLEPPMANN, Martin. *Designing Data-Intensive Applications: The Big Ideas Behind Reliable, Scalable, and Maintainable Systems*. O'Reilly Media, 2017.
- APACHE KAFKA. *Documentation: Design, Performance, and Implementation*. Disponível em: <https://kafka.apache.org/documentation/>.
- NYGARD, Michael T. *Release It!: Design and Deploy Production-Ready Software*. Pragmatic Bookshelf, 2018.
- ONS / ANEEL. *Procedimentos de Distribuição de Energia Elétrica no Sistema Elétrico Nacional (PRODIST)*, Módulo 8 — Qualidade da Energia Elétrica.
