# Casos de Uso de Regras Clínicas

[⌂ Módulo ClinicalCatalog](../index.md) · [Visão Geral](./overview.md)

Grupo de rotas: `/api/v1/clinical-catalog/rules` · Requer autenticação (sem role específica).

A regra clínica define os intervalos mínimos e contraindicações que condicionam a realização de um item clínico, com **vigência temporal**: cada regra vale de `validFrom` até `validTo` (ou indefinidamente, se `validTo` for `null`).

Para o mesmo item, as vigências **não podem sobrepor-se** — é isso que garante que existe no máximo uma regra em vigor em cada instante.

## 3.1 Criar regra

- Caso de uso: `CreateRuleCommand`
- Endpoint: `POST /api/v1/clinical-catalog/rules`
- Modelo principal: `ClinicalRule`
- Efeito colateral: **gera um registo de auditoria** de criação (`ClinicalRuleAudit`)

### Regras de negócio, por ordem de validação

1. o item clínico tem de existir (`Rule.ItemNotFound`)
2. `validFrom` e `validTo` são normalizados para UTC (`DateTimeKind.Utc`) antes de qualquer comparação
3. a janela não pode sobrepor-se a nenhuma regra existente do mesmo item (`Rule.OverlappingDates`)
4. se `validTo` tiver valor, `validFrom` tem de ser estritamente anterior (`Rule.InvalidDateRange`)

O teste de sobreposição trata `validTo = null` como `9999-12-31`, pelo que **uma regra de vigência aberta bloqueia a criação de qualquer regra posterior para o mesmo item**. Para versionar uma regra, feche primeiro a atual com `PATCH` a definir `validTo`, e só depois crie a nova.

### Body

```json
{
  "clinicalItemId": "b2c3d4e5-f6a7-4b8c-9d0e-1f2a3b4c5d6e",
  "minHoursAfterBloodDonation": 48,
  "minDaysAfterVaccination": 14,
  "minDaysAfterSurgery": 30,
  "contraindicatedAnticoagulants": true,
  "validFrom": "2026-09-01T00:00:00Z",
  "validTo": null
}
```

Todos os campos são obrigatórios no record (`validTo` aceita `null`). **O autor não vem no body**: `createdBy` é preenchido com o id do utilizador autenticado, lido da claim `sub` do token.

### Exemplo de chamada

```bash
curl -X POST http://localhost:5000/api/v1/clinical-catalog/rules \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <jwt>' \
  -d '{
    "clinicalItemId": "b2c3d4e5-f6a7-4b8c-9d0e-1f2a3b4c5d6e",
    "minHoursAfterBloodDonation": 48,
    "minDaysAfterVaccination": 14,
    "minDaysAfterSurgery": 30,
    "contraindicatedAnticoagulants": true,
    "validFrom": "2026-09-01T00:00:00Z",
    "validTo": null
  }'
```

### Sucesso esperado

- `201 Created`, com `Location` a apontar para `GET /api/v1/clinical-catalog/rules/{id}`
- retorno: `ClinicalRuleResponse`

```json
{
  "id": "d4e5f6a7-b8c9-4d0e-1f2a-3b4c5d6e7f80",
  "clinicalItemId": "b2c3d4e5-f6a7-4b8c-9d0e-1f2a3b4c5d6e",
  "minHoursAfterBloodDonation": 48,
  "minDaysAfterVaccination": 14,
  "minDaysAfterSurgery": 30,
  "contraindicatedAnticoagulants": true,
  "validFrom": "2026-09-01T00:00:00Z",
  "validTo": null,
  "createdBy": "9f1c7b2e-4a55-4c31-9d18-2b7a6e0f3c44",
  "createdAt": "2026-08-14T10:00:00Z",
  "updatedAt": "2026-08-14T10:00:00Z"
}
```

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `400` | `Rule.ItemNotFound` | item clínico inexistente |
| `409` | `Rule.OverlappingDates` | janela sobrepõe-se a regra existente do mesmo item |
| `400` | `Rule.InvalidDateRange` | `validFrom` ≥ `validTo` |
| `401` | — | não autenticado |

A não-sobreposição é garantida pela base de dados, além da verificação do handler: duas criações simultâneas para o mesmo item não conseguem ficar ambas gravadas.

## 3.2 Listar regras

- Caso de uso: `GetAllRulesQuery`
- Endpoint: `GET /api/v1/clinical-catalog/rules`

### Query params

| Parâmetro | Tipo | Omissão | Descrição |
|---|---|---|---|
| `clinicalItemId` | guid | — | filtra por item |
| `currentOnly` | bool | `false` | apenas as regras em vigor **agora** (`validFrom ≤ agora ≤ validTo`, ou `validTo = null`) |
| `page` | int | `1` | página pedida (mínimo `1`) |
| `limit` | int | `20` | itens por página (máximo `200`) |

### Regras de negócio

- resultados ordenados por `ValidFrom` ascendente
- `page`/`limit` são aplicados como `Skip`/`Take`: `data` traz no máximo `limit` elementos
- `total` respeita os mesmos filtros de `data`, `currentOnly` incluído

### Exemplo de chamada

```bash
curl 'http://localhost:5000/api/v1/clinical-catalog/rules?clinicalItemId=b2c3d4e5-f6a7-4b8c-9d0e-1f2a3b4c5d6e&currentOnly=true' \
  -H 'Authorization: Bearer <jwt>'
```

### Sucesso esperado

- `200 OK`
- retorno: `PagedResult<ClinicalRuleResponse>`

## 3.4 Obter regra por id

- Caso de uso: `GetRuleByIdQuery`
- Endpoint: `GET /api/v1/clinical-catalog/rules/{id:guid}`

### Sucesso esperado

- `200 OK`
- retorno: `ClinicalRuleResponse`

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Rule.NotFound` | regra inexistente |

## 3.5 Atualizar regra

- Caso de uso: `UpdateRuleCommand`
- Endpoint: `PATCH /api/v1/clinical-catalog/rules/{id:guid}`
- Efeito colateral: **gera um registo de auditoria** (`ClinicalRuleAudit`)

### Body

```json
{
  "minHoursAfterBloodDonation": 72,
  "minDaysAfterVaccination": 21,
  "minDaysAfterSurgery": null,
  "contraindicatedAnticoagulants": null,
  "validFrom": null,
  "validTo": "2027-01-01T00:00:00Z"
}
```

Todos os campos são opcionais. **O autor não vem no body**: o `changedBy` da auditoria é o id do utilizador autenticado.

### Regras de negócio, por ordem de validação

1. a regra tem de existir (`Rule.NotFound`)
2. a vigência resultante é calculada — o que o pedido não traz mantém-se — e normalizada para UTC
3. se `validTo` resultante não for nulo, `validFrom` resultante tem de ser anterior (`Rule.InvalidDateRange`)
4. a vigência resultante não pode sobrepor-se a outra regra do mesmo item, **excluindo a própria regra** da verificação (`Rule.OverlappingDates`)

Só depois destas validações a regra é alterada: um pedido rejeitado não deixa qualquer marca, nem na regra nem na auditoria.

- antes e depois da alteração, o `ClinicalRuleResponse` é serializado em JSON e guardado em `OldData`/`NewData` do registo de auditoria, junto com `ChangedBy` e `ChangedAt`
- auditoria e alteração são persistidas na mesma transação (um único `SaveChangesAsync`)

### Sucesso esperado

- `200 OK`
- retorno: `ClinicalRuleResponse` atualizado

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Rule.NotFound` | regra inexistente |
| `400` | `Rule.InvalidDateRange` | intervalo resultante inválido |
| `409` | `Rule.OverlappingDates` | vigência resultante sobrepõe-se a outra regra do mesmo item |
| `401` | — | não autenticado |

## 3.6 Apagar regra

- Caso de uso: `DeleteRuleCommand`
- Endpoint: `DELETE /api/v1/clinical-catalog/rules/{id:guid}`
- Efeito colateral: **gera um registo de auditoria** de eliminação (`ClinicalRuleAudit`)

### Regras de negócio

- a regra tem de existir (`Rule.NotFound`)
- a eliminação é **física**, mas fica auditada: o estado final da regra é serializado para `oldData` antes da remoção, na mesma transação
- **o histórico sobrevive à regra**, por desenho: os registos de auditoria não são apagados, porque são o único vestígio de que a regra existiu. `GET /rules/{id}/audits` continua a responder depois da eliminação, incluindo o registo da própria eliminação
- para encerrar uma regra mantendo-a consultável, prefira `PATCH` a definir `validTo`

### Sucesso esperado

- `204 No Content`

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Rule.NotFound` | regra inexistente |

## 3.7 Histórico de alterações de uma regra

- Caso de uso: `GetRuleAuditQuery`
- Endpoint: `GET /api/v1/clinical-catalog/rules/{id:guid}/audits`

### Query params

| Parâmetro | Tipo | Omissão |
|---|---|---|
| `page` | int | `1` |
| `limit` | int | `20` |

### Regras de negócio

- o histórico é devolvido mesmo depois de a regra ser eliminada; só é `404` quando não existe nem regra nem histórico para o id (`Rule.NotFound`)
- resultados ordenados por `ChangedAt` descendente (mais recente primeiro)
- `page`/`limit` são aplicados como `Skip`/`Take`, com a mesma normalização das restantes listagens
- o histórico cobre o ciclo de vida completo da regra. A operação distingue-se pelos lados preenchidos:

| Operação | `oldData` | `newData` |
|---|---|---|
| criação | `{}` | estado inicial |
| atualização | estado anterior | estado novo |
| eliminação | estado final | `{}` |

### Sucesso esperado

- `200 OK`
- retorno: `PagedResult<RuleAuditResponse>`

```json
{
  "data": [
    {
      "id": "e5f6a7b8-c9d0-4e1f-2a3b-4c5d6e7f8091",
      "clinicalRuleId": "d4e5f6a7-b8c9-4d0e-1f2a-3b4c5d6e7f80",
      "changedBy": "9f1c7b2e-4a55-4c31-9d18-2b7a6e0f3c44",
      "oldData": "{\"Id\":\"d4e5f6a7-...\",\"MinHoursAfterBloodDonation\":48}",
      "newData": "{\"Id\":\"d4e5f6a7-...\",\"MinHoursAfterBloodDonation\":72}",
      "changedAt": "2026-08-14T11:00:00Z"
    }
  ],
  "total": 1,
  "page": 1,
  "limit": 20,
  "totalPages": 1,
  "hasNext": false,
  "hasPrev": false
}
```

`oldData` e `newData` são strings com JSON serializado do `ClinicalRuleResponse` — o consumidor tem de fazer parse para comparar campos. `changedBy` é o id do utilizador autenticado no momento da operação.

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Rule.NotFound` | regra inexistente |

---

## Navegação

← [Itens Clínicos](./items.md) · [Índice do módulo](../index.md) · [Casos Internos, Eventos e Notas](./internal-and-events.md) →

**Neste módulo:** [Visão Geral](./overview.md) · [Modelos e Enumerações](./models-and-enums.md) · [Especialidades](./specialties.md) · [Itens Clínicos](./items.md) · **Regras Clínicas** · [Casos Internos, Eventos e Notas](./internal-and-events.md)
