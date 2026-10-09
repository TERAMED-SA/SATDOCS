# Casos de Uso de Encaminhamentos

[⌂ Módulo Scheduling](../index.md) · [Visão Geral](./overview.md)

Grupo de rotas: `/api/v1/scheduling/forwardings` · Requer autenticação (sem role específica).

Um encaminhamento envia o paciente de uma marcação **concluída** para outra entidade (`EXTERNAL`) ou outro serviço do hospital (`INTERNAL`).

```
PENDING ──send──► SENT ──receive──► RECEIVED ──accept──► ACCEPTED ──complete──► COMPLETED
   │                                   │
 (PUT, DELETE)                       reject
                                       ▼
                                    REJECTED

cancel: a partir de qualquer estado exceto COMPLETED e CANCELLED
```

## 5.1 Criar encaminhamento

- Caso de uso: `CreateForwardingCommand`
- Endpoint: `POST /api/v1/scheduling/forwardings`
- Modelos principais: `Forwarding`, `Appointment`

### Regras de negócio, por ordem de validação

Validação do pedido (`400 ValidationFailed`):

1. `appointmentId` obrigatório
2. `senderTaxId` obrigatório, até 50 caracteres
3. `reason` obrigatório, até 1000 caracteres
4. `receiverTaxId` até 50, `medicalAreaId` e `specialityId` até 100, `obs` até 2000 caracteres

Regras de negócio:

5. a marcação existe (`404 AppointmentNotFound`)
6. a marcação está concluída — `COMPLETED` ou `isCompleted` (`400 AppointmentNotCompleted`)

O destinatário (`receiverTaxId`) **não é validado**: não se verifica se existe nem se pertence ao hospital.

### Efeitos

- o encaminhamento nasce `PENDING`, com `forwardedAt` = agora
- a marcação passa a `FORWARDED`, com `hasForwarding = true` e `forwardedAt`
- é publicado `ForwardingCreatedEvent`

### Body

```json
{
  "appointmentId": "e5f6a7b8-c9d0-4e1f-8a2b-3c4d5e6f7a8b",
  "destination": "INTERNAL",
  "medicalAreaId": null,
  "specialityId": "CARD",
  "receiverTaxId": "6b7c8d9e-0f1a-4b2c-9d3e-4f5a6b7c8d9e",
  "senderTaxId": "5a6b7c8d-9e0f-4a1b-8c2d-3e4f5a6b7c8d",
  "reason": "Avaliação cardiológica",
  "priority": "URGENT",
  "obs": null
}
```

| Campo | Obrigatório | Omissão |
|---|---|---|
| `appointmentId`, `senderTaxId`, `reason` | sim | — |
| `destination` | não | `EXTERNAL` |
| `priority` | não | `NORMAL` |
| `medicalAreaId`, `specialityId`, `receiverTaxId`, `obs` | não | `null` |

### Sucesso esperado

- `201 Created`, `Location` para `GET /api/v1/scheduling/forwardings/{id}`, corpo com o id (string JSON)

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `400` | `ValidationFailed` | pedido inválido |
| `404` | `AppointmentNotFound` | marcação inexistente |
| `400` | `AppointmentNotCompleted` | marcação ainda não concluída |

## 5.2 Listar encaminhamentos

- Caso de uso: `GetForwardingsQuery`
- Endpoints:
  - `GET /api/v1/scheduling/forwardings` — todos os filtros
  - `GET /api/v1/scheduling/forwardings/status/{status}?page=&limit=` — estado inválido devolve `400` com o texto `"Status inválido"`
  - `GET /api/v1/scheduling/forwardings/appointments/{appointmentId}?page=&limit=`
  - `GET /api/v1/scheduling/forwardings/receiver/{receiverTaxId}?page=&limit=`

Nas três últimas, `page` e `limit` são obrigatórios.

### Query params

| Parâmetro | Tipo |
|---|---|
| `id`, `appointmentId` | guid |
| `senderTaxId`, `receiverTaxId`, `medicalAreaId`, `specialityId` | string |
| `status` / `statusIn` | `ForwardingStatus` / lista |
| `isActive`, `isPending`, `isCompleted`, `hasReceiver`, `isUrgent` | bool |
| `destination` | `ForwardingDestination` |
| `priority` / `priorityIn` | `Priority` / lista |
| `forwardedAtFrom/To`, `createdAtFrom/To` | datetime |
| `reasonContains`, `obsContains` | string (distingue maiúsculas) |
| `search` | string — ver [limitações](./internal-and-events.md#limitações-conhecidas) |
| `sortBy` | `forwardedAt` (omissão), `createdAt`, `updatedAt`, `priority`, `status` |
| `sortOrder` | `asc` \| `desc` (omissão) |
| `page`, `limit` | omissão `1` e `20`; **não são normalizados** |

### Sucesso esperado

- `200 OK` com o envelope próprio `PagedResult<ForwardingDto>`:

```json
{
  "data": [
    {
      "id": "f6a7b8c9-d0e1-4f2a-8b3c-4d5e6f7a8b9c",
      "appointmentId": "e5f6a7b8-c9d0-4e1f-8a2b-3c4d5e6f7a8b",
      "destination": "INTERNAL",
      "medicalAreaId": null,
      "specialityId": "CARD",
      "receiverTaxId": "6b7c8d9e-0f1a-4b2c-9d3e-4f5a6b7c8d9e",
      "senderTaxId": "5a6b7c8d-9e0f-4a1b-8c2d-3e4f5a6b7c8d",
      "reason": "Avaliação cardiológica",
      "priority": "URGENT",
      "status": "PENDING",
      "forwardedAt": "2026-10-09T15:00:00Z",
      "obs": null,
      "createdAt": "2026-10-09T15:00:00Z",
      "updatedAt": "2026-10-09T15:00:00Z",
      "appointment": { "id": "e5f6a7b8-c9d0-4e1f-8a2b-3c4d5e6f7a8b", "status": "FORWARDED" }
    }
  ],
  "pagination": { "page": 1, "limit": 20, "total": 1, "totalPages": 1, "hasNext": false, "hasPrev": false },
  "stats": {
    "totalForwardings": 1,
    "byStatus": { "PENDING": 1 },
    "byPriority": { "URGENT": 1 },
    "byDestination": { "INTERNAL": 1 }
  }
}
```

- `appointment` traz o `AppointmentDto` completo (abreviado acima)
- `stats` é calculado sobre **todos** os resultados filtrados, não só a página

## 5.3 Obter encaminhamento por id

- Caso de uso: `GetForwardingByIdQuery`
- Endpoint: `GET /api/v1/scheduling/forwardings/{id}`

### Sucesso esperado

- `200 OK` com um `ForwardingDto`

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `ForwardingNotFound` | inexistente |

## 5.4 Alterar e apagar

- Casos de uso: `UpdateForwardingCommand`, `DeleteForwardingCommand`
- Endpoints: `PUT /api/v1/scheduling/forwardings/{id}` · `DELETE /api/v1/scheduling/forwardings/{id}`

### Body (alterar)

Todos os campos são opcionais; só os enviados são alterados:

```json
{
  "destination": "EXTERNAL",
  "medicalAreaId": null,
  "specialityId": null,
  "receiverTaxId": "7c8d9e0f-...",
  "reason": "Encaminhar para hospital parceiro",
  "priority": "SERIOUS",
  "obs": "Levar exames anteriores"
}
```

### Regras de negócio

- só encaminhamentos `PENDING` (`400 ForwardingCannotBeUpdated` / `400 ForwardingCannotBeDeleted`)
- ao alterar, campos de texto vazios são ignorados; `obs` é substituído sempre que enviado (mesmo vazio)
- `reason` até 1000 e `obs` até 2000 caracteres (`400 ValidationFailed`)
- apagar é **físico**; a marcação continua `FORWARDED` com `hasForwarding = true`

### Sucesso esperado

- `204 No Content`

## 5.5 Transições de estado

- Casos de uso: `SendForwardingCommand`, `ReceiveForwardingCommand`, `AcceptForwardingCommand`, `RejectForwardingCommand`, `CompleteForwardingCommand`, `CancelForwardingCommand`
- Endpoints: `PATCH /api/v1/scheduling/forwardings/{id}/<ação>`

| Ação | Estado exigido | Novo estado | Body | Erro |
|---|---|---|---|---|
| `send` | `PENDING` | `SENT` | — | `400 ForwardingCannotBeSent` |
| `receive` | `SENT` | `RECEIVED` | — | `400 ForwardingCannotBeReceived` |
| `accept` | `RECEIVED` | `ACCEPTED` | — | `400 ForwardingCannotBeAccepted` |
| `reject` | `RECEIVED` | `REJECTED` | `{ "reason": "..." }` | `400 ForwardingCannotBeRejected` |
| `complete` | `ACCEPTED` | `COMPLETED` | — | `400 ForwardingCannotBeCompleted` |
| `cancel` | qualquer, exceto `COMPLETED` e `CANCELLED` | `CANCELLED` | `{ "reason": "..." }` | `400 ForwardingCannotBeCancelled` |

- em `reject` e `cancel`, `reason` é opcional; quando enviado, é acrescentado a `obs`
- nenhuma transição altera a marcação nem publica eventos

### Sucesso esperado

- `204 No Content`

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `ForwardingNotFound` | inexistente |
| `400` | ver tabela | estado não permite a transição |

---

## Navegação

← [Reagendamentos](./reschedules.md) · [Índice do módulo](../index.md) · [Exclusões](./exclusions.md) →

**Neste módulo:** [Visão Geral](./overview.md) · [Modelos e Enumerações](./models-and-enums.md) · [Agendas](./schedules.md) · [Slots](./slots.md) · [Marcações](./appointments.md) · [Reagendamentos](./reschedules.md) · **Encaminhamentos** · [Exclusões](./exclusions.md) · [Horários de Trabalho](./working-hours.md) · [Utilitários](./utilities.md) · [Casos Internos, Eventos e Notas](./internal-and-events.md)
