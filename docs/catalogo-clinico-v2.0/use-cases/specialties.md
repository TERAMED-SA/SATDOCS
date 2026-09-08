# Casos de Uso de Especialidades

[⌂ Módulo ClinicalCatalog](../index.md) · [Visão Geral](./overview.md)

Grupo de rotas: `/api/v1/clinical-catalog/specialties` · Requer autenticação (sem role específica).

A especialidade é a raiz do catálogo: itens clínicos só podem ser criados dentro de uma especialidade existente.

## 1.1 Criar especialidade

- Caso de uso: `CreateSpecialtyCommand`
- Endpoint: `POST /api/v1/clinical-catalog/specialties`
- Modelo principal: `MedicalSpecialty`

### Regras de negócio

- `snomedCode` é obrigatório (`Specialty.InvalidSnomedCode`) e único (`Specialty.AlreadyExists`)
- `name` é obrigatório (`Specialty.InvalidName`)
- `isActive` assume `true` por omissão
- `CreatedBy` e `UpdatedBy` são gravados com o id do utilizador autenticado, lido da claim `sub` do token

### Body

```json
{
  "snomedCode": "394814009",
  "name": "Clínica Geral",
  "isActive": true
}
```

### Exemplo de chamada

```bash
curl -X POST http://localhost:5000/api/v1/clinical-catalog/specialties \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <jwt>' \
  -d '{ "snomedCode": "394814009", "name": "Clinica Geral" }'
```

### Sucesso esperado

- `201 Created`, com `Location` a apontar para `GET /api/v1/clinical-catalog/specialties/{id}`
- retorno: `SpecialtyResponse`

```json
{
  "id": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
  "snomedCode": "394814009",
  "name": "Clinica Geral",
  "isActive": true,
  "createdBy": "9f1c7b2e-4a55-4c31-9d18-2b7a6e0f3c44",
  "updatedBy": "9f1c7b2e-4a55-4c31-9d18-2b7a6e0f3c44",
  "createdAt": "2026-08-14T10:00:00Z",
  "updatedAt": "2026-08-14T10:00:00Z",
  "itemCount": null
}
```

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `409` | `Specialty.AlreadyExists` | código SNOMED já existe |
| `400` | `Specialty.InvalidSnomedCode` | código em branco |
| `400` | `Specialty.InvalidName` | nome em branco |
| `401` | — | não autenticado |

## 1.2 Listar especialidades

- Caso de uso: `GetAllSpecialtiesQuery`
- Endpoint: `GET /api/v1/clinical-catalog/specialties`

### Query params

| Parâmetro | Tipo | Omissão | Descrição |
|---|---|---|---|
| `search` | string | — | filtra por `Name` **ou** `SnomedCode`, sem distinguir maiúsculas de minúsculas; `%` e `_` no termo são tratados como texto |
| `isActive` | bool | — | filtra por estado |
| `page` | int | `1` | página pedida (mínimo `1`) |
| `limit` | int | `20` | itens por página (máximo `200`) |

### Regras de negócio

- resultados ordenados por `Name`
- `page`/`limit` são aplicados como `Skip`/`Take`: `data` traz no máximo `limit` elementos
- cada especialidade traz `itemCount` — o número de itens clínicos associados, resolvido numa única consulta agregada para toda a página

### Exemplo de chamada

```bash
curl 'http://localhost:5000/api/v1/clinical-catalog/specialties?search=Clinica&isActive=true' \
  -H 'Authorization: Bearer <jwt>'
```

### Sucesso esperado

- `200 OK`
- retorno: `PagedResult<SpecialtyResponse>`

```json
{
  "data": [
    {
      "id": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
      "snomedCode": "394814009",
      "name": "Clinica Geral",
      "isActive": true,
      "createdBy": "9f1c7b2e-4a55-4c31-9d18-2b7a6e0f3c44",
      "updatedBy": "9f1c7b2e-4a55-4c31-9d18-2b7a6e0f3c44",
      "createdAt": "2026-08-14T10:00:00Z",
      "updatedAt": "2026-08-14T10:00:00Z",
      "itemCount": 12
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

## 1.3 Obter especialidade por id

- Caso de uso: `GetSpecialtyByIdQuery`
- Endpoint: `GET /api/v1/clinical-catalog/specialties/{id:guid}`

### Sucesso esperado

- `200 OK`
- retorno: `SpecialtyResponse` com `itemCount` preenchido

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Specialty.NotFound` | especialidade inexistente |
| `401` | — | não autenticado |

## 1.4 Verificar se pode ser apagada

- Caso de uso: `CanDeleteSpecialtyQuery`
- Endpoint: `GET /api/v1/clinical-catalog/specialties/{id:guid}/can-delete`
- Finalidade: consulta prévia, para a UI poder desativar o botão de eliminação

### Regras de negócio

- a especialidade tem de existir (`Specialty.NotFound`)
- se tiver itens associados: `canDelete = false` e `reason = "Specialty has N associated item(s)"`
- caso contrário: `canDelete = true`, `reason = null`

### Sucesso esperado

- `200 OK`

```json
{
  "canDelete": false,
  "reason": "Specialty has 12 associated item(s)"
}
```

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Specialty.NotFound` | especialidade inexistente |

## 1.5 Atualizar especialidade

- Caso de uso: `UpdateSpecialtyCommand`
- Endpoint: `PATCH /api/v1/clinical-catalog/specialties/{id:guid}`

### Body

```json
{
  "name": "Medicina Geral e Familiar",
  "isActive": false
}
```

Ambos os campos são anuláveis; só os enviados são aplicados. `snomedCode` **não é editável**.

### Regras de negócio

- a especialidade tem de existir (`Specialty.NotFound`)
- se `name` for enviado, não pode ser vazio ou só espaços (`Specialty.InvalidName`)
- `UpdatedBy` passa a ser o id de quem fez a alteração e `UpdatedAt` a data atual
- `isActive` aplica-se apenas à especialidade: os itens clínicos associados mantêm o seu próprio `isActive`, que se altera item a item

### Sucesso esperado

- `200 OK`
- retorno: `SpecialtyResponse` (com `itemCount` a `null`)

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Specialty.NotFound` | especialidade inexistente (este endpoint devolve `400`, não `404`) |
| `400` | `Specialty.InvalidName` | nome enviado em branco |
| `401` | — | não autenticado |

## 1.6 Apagar especialidade

- Caso de uso: `DeleteSpecialtyCommand`
- Endpoint: `DELETE /api/v1/clinical-catalog/specialties/{id:guid}`

### Regras de negócio

- a especialidade tem de existir (`Specialty.NotFound`)
- não pode ter itens clínicos associados (`Specialty.HasAssociatedItems`)
- a eliminação é **física** (`Remove`), não um soft delete — para retirar de circulação sem perder histórico, use `PATCH` com `isActive = false`

### Exemplo de chamada

```bash
curl -X DELETE http://localhost:5000/api/v1/clinical-catalog/specialties/a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d \
  -H 'Authorization: Bearer <jwt>'
```

### Sucesso esperado

- `204 No Content` (sem corpo)

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Specialty.NotFound` | especialidade inexistente |
| `409` | `Specialty.HasAssociatedItems` | ainda tem itens clínicos |
| `401` | — | não autenticado |

---

## Navegação

← [Modelos e Enumerações](./models-and-enums.md) · [Índice do módulo](../index.md) · [Itens Clínicos](./items.md) →

**Neste módulo:** [Visão Geral](./overview.md) · [Modelos e Enumerações](./models-and-enums.md) · **Especialidades** · [Itens Clínicos](./items.md) · [Regras Clínicas](./rules.md) · [Casos Internos, Eventos e Notas](./internal-and-events.md)
