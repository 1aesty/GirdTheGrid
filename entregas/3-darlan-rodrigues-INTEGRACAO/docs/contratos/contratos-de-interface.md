# Contratos de Interface

**Responsável:** Darlan Rodrigues Schmeller (Pesquisador & Engenheiro de Resiliência & Integração)

Atende ao item do checklist: *"As APIs expostas ou consumidas (REST/JSON ou gRPC/Protobuf) possuem contratos formais atualizados?"*

| Contrato | Formato | Quem expõe | Quem consome |
|---|---|---|---|
| API de Consulta do Dashboard | REST/JSON (OpenAPI 3) | Serviço de Consulta | Web App via API Gateway |
| Evento de Telemetria | JSON Schema (Kafka, tópico `telemetria.v1`) | Serviço de Ingestão | Worker de Regras, MAPE-K |
| Comando de Manobra | gRPC/Protobuf | API Gateway Industrial | Serviço MAPE-K |
| Cadastro de Clientes (legado) | REST/JSON (consumido via ACL) | Sistema legado | LegadoAdapter |

## 1. API de Consulta (REST, OpenAPI 3)

```yaml
openapi: 3.0.3
info:
  title: Gird the Grid - API de Consulta
  version: 1.0.0
paths:
  /tenants/{tenantId}/subestacoes:
    get:
      summary: Lista subestações com carga e utilização atual
      security: [{ oauth2: [] }]
      parameters:
        - { name: tenantId, in: path, required: true, schema: { type: string } }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                type: array
                items: { $ref: "#/components/schemas/Subestacao" }
        "401": { description: Não autenticado }
        "403": { description: Tenant não autorizado }
components:
  schemas:
    Subestacao:
      type: object
      required: [id, capacidadeKw, cargaKw, utilizacao, status]
      properties:
        id: { type: string, example: SE-CURITIBA-NORTE }
        capacidadeKw: { type: number }
        cargaKw: { type: number }
        utilizacao: { type: number, minimum: 0 }
        status: { type: string, enum: [OPERANDO, SOBRECARGA, FALHA] }
```

## 2. Evento de Telemetria (Kafka, JSON Schema)

```json
{
  "$id": "telemetria.v1",
  "type": "object",
  "required": ["tenantId", "medidorId", "subestacaoId", "potenciaKw", "tensaoV", "timestamp"],
  "properties": {
    "tenantId":     { "type": "string" },
    "medidorId":    { "type": "string" },
    "subestacaoId": { "type": "string" },
    "potenciaKw":   { "type": "number", "minimum": 0 },
    "tensaoV":      { "type": "number", "exclusiveMinimum": 0 },
    "timestamp":    { "type": "string", "format": "date-time" }
  },
  "additionalProperties": false
}
```

- **Chave da mensagem:** `tenantId + medidorId`.
- **Versionamento:** mudanças incompatíveis criam um novo tópico (`telemetria.v2`); o `v1` continua até todos os consumidores migrarem.

## 3. Comando de Manobra (gRPC / Protobuf)

```protobuf
syntax = "proto3";
package girdthegrid.manobra.v1;

service AtuadorRede {
  rpc ExecutarManobra (ManobraRequest) returns (ManobraResponse);
}

message ManobraRequest {
  string tenant_id   = 1;
  string origem_id   = 2;
  string destino_id  = 3;
  double carga_kw    = 4;
  string aprovado_por = 5; // "guardrail" ou id do operador (manobras > 500 kW)
}

message ManobraResponse {
  bool   executada = 1;
  string motivo    = 2;
}
```

## 4. Como funcionaria a validação dos contratos

> Explicação do funcionamento; não é código da entrega.

1. **Contract-first:** os arquivos acima ficam versionados no repositório. Mudança de contrato entra por PR, com aprovação do líder.
2. **Validação na borda:** o Serviço de Ingestão valida cada mensagem contra o JSON Schema `telemetria.v1` **antes** de publicar no Kafka. Se a mensagem não passar, vai direto para a DLQ com o motivo.
3. **Teste de contrato no CI:** para cada contrato, um teste compara o que o produtor gera com o schema publicado. Se alguém renomear um campo (ex.: `potenciaKw` para `kw`), o teste falha e o PR é bloqueado.
4. **Compatibilidade:** o CI compara o schema novo com o da `main`. Remover campo obrigatório ou mudar tipo é **quebra**, exige nova versão (`v2`) e um ADR.
