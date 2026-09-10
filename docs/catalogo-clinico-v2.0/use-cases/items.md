# Casos de Uso de Itens Clínicos

[⌂ Módulo ClinicalCatalog](../index.md) · [Visão Geral](./overview.md)

Grupo de rotas: `/api/v1/clinical-catalog/items` · Requer autenticação (sem role específica).

O item clínico é a unidade agendável do catálogo: uma consulta, um exame, uma vacina ou um procedimento.

## 2.1 Criar item clínico

- Caso de uso: `CreateItemCommand`
- Endpoint: `POST /api/v1/clinical-catalog/items`
- Modelo principal: `ClinicalItem`

### Regras de negócio, por ordem de validação

1. a especialidade indicada tem de existir (`Item.SpecialtyNotFound`)
2. o `code` não pode já estar em uso (`Item.AlreadyExists`) — a unicidade é **global**, não por especialidade
3. `code` é obrigatório (`Item.InvalidCode`)
4. `name` é obrigatório (`Item.InvalidName`)

`CreatedBy`/`UpdatedBy` são gravados com o id do utilizador autenticado, lido da claim `sub` do token — não vêm no body.

### Body

```json
{
  "specialtyId": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
  "type": "Vaccine",
  "code": "VAC-HEPB",
  "name": "Vacina Hepatite B",
  "standardCode": "396429000",
  "description": "Esquema de três doses",
  "allowedScheduleType": "Both",
  "requiresPrescription": false,
  "requiresSafetyCheck": true,
  "isActive": true
}
```

| Campo | Obrigatório | Omissão |
|---|---|---|
| `specialtyId` | sim | — |
| `type` | sim | — |
| `code` | sim | — |
| `name` | sim | — |
| `standardCode` | não | `null` |
| `description` | não | `null` |
| `allowedScheduleType` | não | `Both` |
| `requiresPrescription` | não | `false` |
| `requiresSafetyCheck` | não | `false` |
| `isActive` | não | `true` |

`requiresSafetyCheck = true` sinaliza que o item exige avaliação de segurança clínica antes do agendamento — a avaliação em si é responsabilidade do módulo `ClinicalSafety`, ainda não implementado.

### Exemplo de chamada

```bash
curl -X POST http://localhost:5000/api/v1/clinical-catalog/items \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <jwt>' \
  -d '{
    "specialtyId": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
    "type": "Vaccine",
    "code": "VAC-HEPB",
    "name": "Vacina Hepatite B",
    "requiresSafetyCheck": true
  }'
```

### Sucesso esperado

- `201 Created`, com `Location` a apontar para `GET /api/v1/clinical-catalog/items/{id}`
- retorno: `ClinicalItemResponse`, já com `specialty` preenchida e `ruleCount` a `0`

```json
{
  "id": "b2c3d4e5-f6a7-4b8c-9d0e-1f2a3b4c5d6e",
  "specialtyId": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
  "type": "Vaccine",
  "code": "VAC-HEPB",
  "standardCode": null,
  "name": "Vacina Hepatite B",
  "description": null,
  "allowedScheduleType": "Both",
  "requiresPrescription": false,
  "requiresSafetyCheck": true,
  "isActive": true,
  "createdBy": "9f1c7b2e-4a55-4c31-9d18-2b7a6e0f3c44",
  "updatedBy": "9f1c7b2e-4a55-4c31-9d18-2b7a6e0f3c44",
  "createdAt": "2026-08-14T10:00:00Z",
  "updatedAt": "2026-08-14T10:00:00Z",
  "specialty": {
    "id": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
    "snomedCode": "394582007",
    "name": "Imunologia",
    "isActive": true
  },
  "ruleCount": 0
}
```

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `400` | `Item.SpecialtyNotFound` | especialidade inexistente |
| `409` | `Item.AlreadyExists` | código já usado por outro item |
| `400` | `Item.InvalidCode` | código em branco |
| `400` | `Item.InvalidName` | nome em branco |
| `401` | — | não autenticado |

## 2.2 Listar itens

- Caso de uso: `GetAllItemsQuery`
- Endpoint: `GET /api/v1/clinical-catalog/items`

### Query params

| Parâmetro | Tipo | Omissão | Descrição |
|---|---|---|---|
| `specialtyId` | guid | — | filtra por especialidade |
| `type` | `ClinicalItemType` | — | filtra por tipo |
| `search` | string | — | filtra por `Name` **ou** `Code`, sem distinguir maiúsculas de minúsculas; `%` e `_` no termo são tratados como texto |
| `isActive` | bool | — | filtra por estado |
| `page` | int | `1` | página pedida (mínimo `1`) |
| `limit` | int | `20` | itens por página (máximo `200`) |

### Regras de negócio

- resultados ordenados por `Name`
- `page`/`limit` são aplicados como `Skip`/`Take`: `data` traz no máximo `limit` elementos
- omitir `isActive` devolve ativos e desativados; para o catálogo agendável use `?isActive=true`
- cada item traz a `specialty` aninhada e o `ruleCount`, resolvidos em duas consultas em lote para toda a página

### Exemplo de chamada

```bash
curl 'http://localhost:5000/api/v1/clinical-catalog/items?type=Vaccine&search=Hepatite' \
  -H 'Authorization: Bearer <jwt>'
```

### Sucesso esperado

- `200 OK`
- retorno: `PagedResult<ClinicalItemResponse>`, com `specialty` e `ruleCount` preenchidos

## 2.4 Obter item por id

- Caso de uso: `GetItemByIdQuery`
- Endpoint: `GET /api/v1/clinical-catalog/items/{id:guid}`

### Sucesso esperado

- `200 OK`
- retorno: `ClinicalItemResponse` com `specialty` aninhada e `ruleCount`

Se a especialidade referenciada não for encontrada, `specialty` vem `null` sem erro.

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Item.NotFound` | item inexistente |
| `401` | — | não autenticado |

## 2.5 Atualizar item

- Caso de uso: `UpdateItemCommand`
- Endpoint: `PATCH /api/v1/clinical-catalog/items/{id:guid}`

### Body

```json
{
  "name": "Vacina Hepatite B (adulto)",
  "standardCode": "LOINC-58410-2",
  "description": "Esquema 0-1-6 meses",
  "allowedScheduleType": "Internal",
  "requiresPrescription": true,
  "requiresSafetyCheck": true,
  "isActive": true
}
```

Todos os campos são opcionais; só os enviados são aplicados.

### Regras de negócio

- o item tem de existir (`Item.NotFound`)
- `name`, se enviado, não pode ser vazio (`Item.InvalidName`)
- **não são editáveis:** `specialtyId`, `type` e `code` — um item não pode mudar de especialidade nem de código
- `UpdatedBy` passa a ser o id de quem fez a alteração

### Sucesso esperado

- `200 OK`
- retorno: `ClinicalItemResponse`, com `specialty` e `ruleCount` preenchidos — a mesma forma devolvida por `GET /items/{id}`

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Item.NotFound` | item inexistente |
| `400` | `Item.InvalidName` | nome enviado em branco |
| `401` | — | não autenticado |

## 2.6 Apagar item

- Caso de uso: `DeleteItemCommand`
- Endpoint: `DELETE /api/v1/clinical-catalog/items/{id:guid}`

### Regras de negócio

- o item tem de existir (`Item.NotFound`)
- o item **não pode ter regras clínicas associadas** (`Item.HasAssociatedRules`) — a proteção espelha a que existe entre especialidade e itens. Como o schema não define chaves estrangeiras (`ClinicalItemId` é apenas uma coluna `uuid` indexada), esta verificação é o único obstáculo a regras órfãs
- a eliminação é **física**

Para apagar um item com regras, apague primeiro as regras (`DELETE /rules/{id}`), o que deixa cada eliminação registada na auditoria.

### Sucesso esperado

- `204 No Content`

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Item.NotFound` | item inexistente |
| `409` | `Item.HasAssociatedRules` | item tem regras clínicas associadas |
| `401` | — | não autenticado |

---

## Navegação

← [Especialidades](./specialties.md) · [Índice do módulo](../index.md) · [Regras Clínicas](./rules.md) →

**Neste módulo:** [Visão Geral](./overview.md) · [Modelos e Enumerações](./models-and-enums.md) · [Especialidades](./specialties.md) · **Itens Clínicos** · [Regras Clínicas](./rules.md) · [Casos Internos, Eventos e Notas](./internal-and-events.md)
