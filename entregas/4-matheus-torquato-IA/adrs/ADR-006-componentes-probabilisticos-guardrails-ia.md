# ADR-006: Integração de Componentes Probabilísticos e Guardrails de IA

- **Status:** Aprovado
- **Data:** 2026-11-15
- **Responsável:** Matheus Sobolewski Torquato

## Contexto

O kata exige um Ciclo Autonômico MAPE-K **alimentado por IA** para redistribuir carga quando uma subestação falha. Modelos generativos são **probabilísticos**: podem inventar números, ignorar limites físicos ou ficar indisponíveis. Na rede elétrica, uma manobra errada pode causar **falha em cascata**.

## Opções consideradas

| Opção | Prós | Contras |
|---|---|---|
| Só regras determinísticas | Previsível | Pouca adaptação a cenários complexos |
| LLM em nuvem decidindo sozinho | Flexível | Latência, custo, dados saem da empresa, sem garantia de segurança |
| **IA sugere + guardrails decidem + Model Tiering** | Une adaptação e segurança | Mais componentes |

## Decisão

1. **Model Tiering:** SLM local (Tier 1), LLM em nuvem (Tier 2) e Strategy determinística como fallback.
2. **RAG** com Banco Vetorial (manuais, PRODIST, incidentes) como base de conhecimento (K do MAPE-K).
3. **Saída estruturada** validada por JSON Schema.
4. **Higienização de PII** antes de qualquer envio aos modelos.
5. **Guardrails de segurança física** G1 a G5 com a palavra final; manobras > 500 kW exigem aprovação humana.
6. Toda decisão (executada, bloqueada ou pendente) é registrada no Audit DB.

## Consequências e Trade-offs

- ✅ Falha da IA não para o ciclo.
- ✅ Nenhuma manobra viola os limites físicos, mesmo com alucinação.
- ✅ Conformidade com a LGPD.
- ⚠️ Limites (90%, 500 kW) precisam de calibração com engenheiros eletricistas.
- ⚠️ O SLM local exige capacidade computacional dedicada.
