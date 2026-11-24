# ADR-004: Mapeamento de Contextos Delimitados e Camada Anti-Corrupção

- **Status:** Aprovado
- **Data:** 2026-10-28
- **Responsável:** João Souza

## Contexto

O Gird the Grid integra sistemas legados com modelos incompatíveis com o nosso: o SCADA envia XML com códigos de RTU e energia em Wh brutos, e o Cadastro de Clientes mistura PII com dados técnicos. Se esses formatos chegarem ao núcleo, qualquer troca de fornecedor quebra as regras de negócio e há risco de violar a LGPD.

## Opções consideradas

| Opção | Desvantagem |
|---|---|
| Usar o modelo do legado direto no domínio (Conformist) | Acoplamento total; PII dentro do núcleo |
| Modelo canônico único para toda a empresa | Caro e lento de negociar |
| **Bounded Contexts + ACL na borda** | Exige manter os tradutores |

## Decisão

1. Dividir o sistema nos contextos **Análise de Regras da Rede**, **Autonomia (MAPE-K)**, **Telemetria / Streaming**, **Auditoria** e **Integração Legada** (ver `docs/ddd/mapa-de-contexto.md`).
2. Toda integração com legado passa por uma **ACL** feita com **Adapter** (`ScadaAdapter`, `MedidorAdapter`, `LegadoAdapter`, `SiemAdapter`).
3. A ACL do Cadastro descarta PII e é protegida por **Circuit Breaker** (ver ADR-003).

## Consequências e Trade-offs

- ✅ O núcleo fala apenas a Linguagem Ubíqua; trocar de fornecedor afeta só o Adapter.
- ✅ Conformidade com a LGPD (PII barrada na borda).
- ⚠️ Custo de manutenção dos tradutores e testes de contrato com cada fornecedor.
