# Casos de Uso de Pedidos de Sangue

[⌂ Módulo BloodDonation](../index.md) · [Visão Geral](./overview.md)

Grupo de rotas: `/api/v1/blood-donation/requests` · Requer autenticação.

## 3.1 Criar pedido de sangue

- Caso de uso: `CreateBloodRequestCommand`
- Endpoint: `POST /api/v1/blood-donation/requests`
- Finalidade: publicar uma necessidade de sangue
- Modelo principal: `BloodRequest`

### Regras de negócio

- `quantityUnits` tem de ser maior que zero (`BloodRequest.InvalidQuantity`)
- `country` é obrigatório (`BloodRequest.CountryRequired`) — o valor por omissão do DTO é `"AO"`
- se `offersCompensation = true`, `compensationAmount` tem de existir e ser positivo (`BloodRequest.InvalidCompensationAmount`)
- `expiresAt`, se fornecido, tem de ser futuro (`BloodRequest.InvalidExpiresAt`)
- `hospitalId`, se fornecido, tem de corresponder a um hospital existente (`Hospital.NotFound`) — verifica-se **apenas a existência**, via `IHospitalContracts.ExistsAsync`. Um hospital inativo ou sem banco de sangue é aceite aqui: o pedido apenas indica onde o doente está. A aptidão para receber dações (`IsAllowedHospitalForDonationAsync`) só é exigida na confirmação da dação, onde o hospital deixa de ser referência e passa a ser destino
- se `expiresAt` não for fornecido, é calculado a partir da urgência:

| Urgência | Expiração por omissão |
|---|---|
| `Critical` | agora + 24 horas |
| `High` | agora + 3 dias |
| `Medium` | agora + 7 dias |
| `Low` | agora + 14 dias |

- o estado inicial é `Open`
- o requerente é sempre o utilizador autenticado (não é aceite no body)

### Body

```json
{
  "bloodType": "O_Negative",
  "quantityUnits": 2,
  "urgency": "Critical",
  "country": "AO",
  "hospitalId": "3f1a9c2e-5b6d-4a7e-9c8f-1d2e3a4b5c6d",
  "hospitalName": "Hospital Josina Machel",
  "address": "Rua Amilcar Cabral",
  "city": "Luanda",
  "patientCondition": "Hemorragia pos-operatoria",
  "requiresCrossmatch": true,
  "specialRequirements": "Sangue irradiado",
  "offersCompensation": false,
  "compensationAmount": null,
  "compensationCurrency": null,
  "expiresAt": null
}
```

Apenas `bloodType` é efetivamente obrigatório; os restantes campos têm valores por omissão (`quantityUnits = 1`, `urgency = Medium`, `country = "AO"`).

### Exemplo de chamada

```bash
curl -X POST http://localhost:5000/api/v1/blood-donation/requests \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <jwt>' \
  -d '{
    "bloodType": "O_Negative",
    "quantityUnits": 2,
    "urgency": "Critical",
    "country": "AO",
    "city": "Luanda"
  }'
```

### Sucesso esperado

- `201 Created`, com `Location` a apontar para `GET /api/v1/blood-donation/requests/{id}`
- retorno: `BloodRequestResponse`

```json
{
  "id": "8d5e1b2a-3c4f-4a5b-9c6d-7e8f9a0b1c2d",
  "requesterUserId": "9a7b6c5d-4e3f-2a1b-0c9d-8e7f6a5b4c3d",
  "bloodType": "O_Negative",
  "quantityUnits": 2,
  "urgency": "Critical",
  "status": "Open",
  "hospitalId": null,
  "hospitalName": null,
  "address": null,
  "city": "Luanda",
  "country": "AO",
  "patientCondition": null,
  "requiresCrossmatch": false,
  "offersCompensation": false,
  "compensationAmount": null,
  "expiresAt": "2026-08-15T10:00:00Z",
  "createdAt": "2026-08-14T10:00:00Z",
  "cancelledAt": null,
  "cancellationReason": null
}
```

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `400` | `BloodRequest.InvalidQuantity` | quantidade ≤ 0 |
| `400` | `BloodRequest.CountryRequired` | país em branco |
| `400` | `BloodRequest.InvalidCompensationAmount` | compensação oferecida sem montante positivo |
| `400` | `BloodRequest.InvalidExpiresAt` | data de expiração no passado |
| `404` | `Hospital.NotFound` | `hospitalId` não corresponde a nenhum hospital |
| `401` | — | não autenticado |

## 3.2 Listar os meus pedidos

- Caso de uso: `GetMyRequestsQuery`
- Endpoint: `GET /api/v1/blood-donation/requests/my`
- Finalidade: listar os pedidos criados pelo utilizador autenticado

### Query params

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `status` | string | filtra por `RequestStatus` (parse case-insensitive; valor inválido é ignorado) |
| `bloodType` | string | filtra por `BloodType` (idem) |
| `includeExpired` | bool | por omissão `false` |
| `page` | int | página (1-based), por omissão `1` |
| `limit` | int | itens por página, por omissão `20` (máximo `200`) |

### Regras de negócio

- filtragem e paginação feitas na base de dados, sobre os pedidos do utilizador
- ordenação por `createdAt` descendente
- com `includeExpired = false` (omissão), são excluídos os pedidos em `Expired` **e** em `Cancelled`
- valores inválidos em `status`/`bloodType` não geram erro: o filtro é simplesmente ignorado

### Exemplo de chamada

```bash
curl 'http://localhost:5000/api/v1/blood-donation/requests/my?status=Open&includeExpired=false' \
  -H 'Authorization: Bearer <jwt>'
```

### Sucesso esperado

- `200 OK`
- retorno: `PagedResult<BloodRequestResponse>` — ver [Envelope de paginação](./overview.md#envelope-de-paginacao)

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `401` | — | não autenticado |

## 3.3 Procurar pedidos públicos

- Caso de uso: `SearchRequestsQuery`
- Endpoint: `GET /api/v1/blood-donation/requests/public/search`
- Finalidade: descoberta de pedidos abertos por potenciais dadores

### Query params

| Parâmetro | Tipo | Omissão | Descrição |
|---|---|---|---|
| `bloodType` | string | — | filtra por grupo sanguíneo |
| `city` | string | — | filtra por cidade |
| `hospitalId` | guid | — | filtra por hospital |
| `minUrgency` | string | — | urgência mínima |
| `page` | int | `1` | página (1-based) |
| `limit` | int | `20` | itens por página (máximo `200`) |

### Regras de negócio

- a pesquisa exclui os pedidos do próprio chamador (`excludeUserId`)
- a extração do `userId` é *best-effort*: se a claim não for válida, `excludeUserId` fica `Guid.Empty` e nada é excluído — mas o grupo continua a exigir autenticação, pelo que a rota **não é anónima** apesar do nome `public`
- a filtragem e a paginação são delegadas ao repositório (`SearchOpenAsync` / `CountOpenAsync`), que restringe a pedidos em aberto
- ordenação por `createdAt` descendente

### Exemplo de chamada

```bash
curl 'http://localhost:5000/api/v1/blood-donation/requests/public/search?bloodType=O_Negative&city=Luanda&page=1&limit=10' \
  -H 'Authorization: Bearer <jwt>'
```

### Sucesso esperado

- `200 OK`
- retorno: `PagedResult<BloodRequestResponse>` — ver [Envelope de paginação](./overview.md#envelope-de-paginacao)

## 3.4 Obter pedido por id

- Caso de uso: `GetRequestByIdQuery`
- Endpoint: `GET /api/v1/blood-donation/requests/{id:guid}`

### Regras de negócio

- sem verificação de ownership: qualquer utilizador autenticado pode ler qualquer pedido pelo id

### Sucesso esperado

- `200 OK`
- retorno: `BloodRequestResponse`

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `BloodRequest.NotFound` | pedido inexistente |
| `401` | — | não autenticado |

## 3.5 Cancelar pedido

- Caso de uso: `CancelBloodRequestCommand`
- Endpoint: `PATCH /api/v1/blood-donation/requests/{id:guid}/cancel`
- Finalidade: encerrar um pedido por decisão do requerente

### Body

```json
{ "reason": "Paciente já recebeu transfusão." }
```

`reason` é opcional, mas **o body é obrigatório** — o endpoint faz binding de `CancelBloodRequestBody`. Envie no mínimo `{}`.

### Regras de negócio

- só o requerente pode cancelar (`Auth.Forbidden`)
- não é possível cancelar se existirem matches já aceites (`BloodRequest.HasAcceptedMatches`) — é preciso desistir desses matches primeiro
- não é possível cancelar um pedido já cancelado (`BloodRequest.AlreadyCancelled`) ou já satisfeito (`BloodRequest.AlreadyFulfilled`)
- efeitos: `Status` → `Cancelled`, com `CancelledAt`, `CancelledBy` e `CancellationReason` preenchidos

### Exemplo de chamada

```bash
curl -X PATCH http://localhost:5000/api/v1/blood-donation/requests/8d5e1b2a-3c4f-4a5b-9c6d-7e8f9a0b1c2d/cancel \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <jwt>' \
  -d '{ "reason": "Paciente ja recebeu transfusao." }'
```

### Sucesso esperado

- `200 OK`
- retorno: `BloodRequestResponse` com `status = "Cancelled"`

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `BloodRequest.NotFound` | pedido inexistente |
| `403` | `Auth.Forbidden` | o chamador não é o requerente |
| `400` | `BloodRequest.HasAcceptedMatches` | há matches aceites por retirar |
| `409` | `BloodRequest.AlreadyCancelled` | pedido já cancelado |
| `409` | `BloodRequest.AlreadyFulfilled` | pedido já satisfeito |
| `401` | — | não autenticado |

## Expiração automática

Não existe endpoint de expiração. O `ExpireOldRequestsCommand` corre de hora a hora através do `ExpireRequestsBackgroundService`. Ver [Casos Internos e Eventos](./internal-and-events.md).

## Contrato `BloodRequestResponse`

```
Id, RequesterUserId, BloodType, QuantityUnits, Urgency, Status,
HospitalId, HospitalName, Address, City, Country, PatientCondition,
RequiresCrossmatch, OffersCompensation, CompensationAmount,
ExpiresAt, CreatedAt, CancelledAt, CancellationReason
```

Persistidos mas **não** expostos: `SpecialRequirements`, `CompensationCurrency`, `CancelledBy`, `UpdatedAt`.

---

## Navegação

← [Análises Clínicas](./analysis.md) · [Índice do módulo](../index.md) · [Matching](./matching.md) →

**Neste módulo:** [Visão Geral](./overview.md) · [Modelos e Enumerações](./models-and-enums.md) · [Dadores](./donors.md) · [Análises Clínicas](./analysis.md) · **Pedidos de Sangue** · [Matching](./matching.md) · [Dações](./donations.md) · [Moderação](./moderation.md) · [Administração](./admin.md) · [Casos Internos, Eventos e Notas](./internal-and-events.md)
