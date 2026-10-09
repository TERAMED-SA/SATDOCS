# Casos de Uso de Reagendamentos

[⌂ Módulo Scheduling](../index.md) · [Visão Geral](./overview.md)

Grupo de rotas: `/api/v1/scheduling/reschedules` · Requer autenticação (sem role específica).

Um reagendamento move uma marcação para outro slot e/ou hora. Conforme a agenda de destino, é aplicado logo ou fica à espera de aprovação.

```
                      RescheduleStartPending = false
POST /reschedules ───────────────────────────────────────► APPROVED  (marcação movida, CONFIRMED)
        │
        │ RescheduleStartPending = true
        ▼
     PENDING ──approve──► APPROVED  (marcação movida)
        ├────reject─────► REJECTED  (marcação REJECTED, fica no slot antigo)
        └────cancel─────► CANCELLED (marcação CANCELLED, vaga libertada)
```

## 4.1 Pedir reagendamento

- Caso de uso: `CreateRescheduleCommand`
- Endpoint: `POST /api/v1/scheduling/reschedules`
- Modelos principais: `Reschedule`, `Appointment`, `Slot`

### Regras de negócio, por ordem de validação

1. a marcação existe (`404 AppointmentNotFound`)
2. está `CONFIRMED`, `PENDING_FOR_PAYMENT` ou `PENDING` (`400 AppointmentCannotBeRescheduled`)
3. `oldSlotId` é o slot atual da marcação (`400 OldSlotMismatch`)
4. o slot novo existe (`404 SlotNotFound`) e a sua agenda tem configurações (`404 ScheduleDoesNotHaveConfigurations`)
5. se o slot novo é `TIMED` e foi indicada `newSelectedHour`, esse horário está `AVAILABLE` (`400 SlotHourUnavailable`)
6. não existe outro reagendamento `PENDING` para a mesma marcação (`409 AppointmentRescheduled`)
7. o paciente cumpre os limites de idade da agenda nova (`409 PatientDoesNotMeetAgeRequirement`)

### Efeitos

Com `rescheduleStartPending = false` na agenda nova:

- o reagendamento fica `APPROVED`, com `approvedAt`
- a vaga é **tomada** no slot novo (horário `newSelectedHour` ou capacidade) e **devolvida** no slot antigo (horário `oldSelectedHour` ou capacidade)
- a marcação passa para o slot novo, com a nova `date` e `hour`, fica `CONFIRMED` e com `isReschedule = true`

Com `rescheduleStartPending = true`:

- o reagendamento fica `PENDING`; nenhuma vaga muda
- a marcação fica `PENDING` com `hasReschedule = true`

Em ambos os casos é publicado `RescheduleCreatedEvent`.

Não são verificados: o estado do slot novo (fechado, agenda apagada), o profissional da marcação no novo dia, a capacidade de um slot `CAPACITY` de destino, nem se `oldSelectedHour` corresponde à hora atual da marcação.

### Body

```json
{
  "appointmentId": "e5f6a7b8-c9d0-4e1f-8a2b-3c4d5e6f7a8b",
  "oldSlotId": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f",
  "oldSelectedHour": "08:30",
  "newSlotId": "d2e3f4a5-b6c7-4d8e-9f0a-1b2c3d4e5f6a",
  "newSelectedHour": "10:00",
  "reason": "Paciente indisponível na data original",
  "requestedBy": "PATIENT",
  "requestedByTaxId": "0f1e2d3c-4b5a-4968-8776-5a4b3c2d1e0f"
}
```

| Campo | Obrigatório | Notas |
|---|---|---|
| `appointmentId`, `oldSlotId`, `newSlotId` | sim | |
| `requestedBy`, `requestedByTaxId` | sim | aceites tal como enviados |
| `oldSelectedHour` | não | hora a libertar no slot antigo |
| `newSelectedHour` | em slots `TIMED` | hora a ocupar no slot novo |
| `reason` | não | |

### Sucesso esperado

- `201 Created`, `Location` para `GET /api/v1/scheduling/reschedules/{id}`, corpo com o id (string JSON)

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `AppointmentNotFound` | marcação inexistente |
| `400` | `AppointmentCannotBeRescheduled` | estado da marcação não permite |
| `400` | `OldSlotMismatch` | `oldSlotId` não é o slot da marcação |
| `404` | `SlotNotFound` / `OldSlotNotFound` | slot novo / antigo inexistente |
| `400` | `SlotHourUnavailable` | hora nova indisponível |
| `409` | `AppointmentRescheduled` | já há um reagendamento pendente |
| `409` | `PatientDoesNotMeetAgeRequirement` | idade fora dos limites da agenda nova |

## 4.2 Listar reagendamentos

- Caso de uso: `GetReschedulesQuery`
- Endpoints:
  - `GET /api/v1/scheduling/reschedules` — todos os filtros
  - `GET /api/v1/scheduling/reschedules/status/{status}?page=&limit=` — `page`/`limit` obrigatórios; estado inválido → `400 BadRequest`
  - `GET /api/v1/scheduling/reschedules/requester/{requestedByTaxId}?page=&limit=` — `page`/`limit` obrigatórios
  - `GET /api/v1/scheduling/reschedules/appointments/{appointmentId}` — até 1000 resultados, devolvidos como **array simples**

### Query params

| Parâmetro | Tipo |
|---|---|
| `id`, `appointmentId`, `oldSlotId`, `newSlotId` | guid |
| `requestedByTaxId` | string |
| `status` / `statusIn` | `RescheduleStatus` / lista |
| `requestedBy` | `RequestedBy` |
| `requestedAtFrom/To`, `approvedAtFrom/To`, `createdAtFrom/To` | datetime |
| `sortBy` | `requestedAt` (omissão), `approvedAt`, `createdAt`, `updatedAt`, `status` |
| `sortOrder` | `asc` \| `desc` (omissão) |
| `page`, `limit` | omissão `1` e `20`; **não são normalizados** — use `page ≥ 1` e `limit ≥ 1` |

### Sucesso esperado

- `200 OK` com `PaginatedList<RescheduleDto>`

```json
{
  "data": [
    {
      "id": "a7b8c9d0-e1f2-4a3b-8c4d-5e6f7a8b9c0d",
      "reason": "Paciente indisponível na data original",
      "requestedBy": "PATIENT",
      "requestedByTaxId": "0f1e2d3c-4b5a-4968-8776-5a4b3c2d1e0f",
      "requestedAt": "2026-10-09T10:00:00Z",
      "approvedAt": null,
      "status": "PENDING",
      "createdAt": "2026-10-09T10:00:00Z",
      "updatedAt": "2026-10-09T10:00:00Z",
      "oldSlot": { "weekDay": "MONDAY", "date": "2026-11-02", "mode": "TIMED", "hour": { "timeFrom": "08:30", "timeTo": "09:00" } },
      "newSlot": { "weekDay": "WEDNESDAY", "date": "2026-11-04", "mode": "TIMED", "hour": { "timeFrom": "10:00", "timeTo": "10:30" } }
    }
  ],
  "pagination": { "page": 1, "limit": 20, "totalItems": 1, "totalPages": 1, "hasNext": false, "hasPrev": false },
  "filters": { "applied": {}, "available": {} }
}
```

## 4.3 Obter reagendamento por id

- Caso de uso: `GetRescheduleByIdQuery`
- Endpoint: `GET /api/v1/scheduling/reschedules/{id}`

### Sucesso esperado

- `200 OK` com um `RescheduleDto` (forma de [4.2](#42-listar-reagendamentos))

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `RescheduleNotFound` | inexistente |

## 4.4 Alterar o motivo

- Caso de uso: `UpdateRescheduleCommand`
- Endpoint: `PUT /api/v1/scheduling/reschedules/{id}`

### Body

```json
{ "reason": "Novo motivo" }
```

### Regras de negócio

- `reason` obrigatório (`400 ValidationFailed`)
- só reagendamentos `PENDING` (`400 RescheduleCannotBeUpdated`); o motivo é **substituído**

### Sucesso esperado

- `204 No Content`

## 4.5 Aprovar, rejeitar e cancelar

- Casos de uso: `ApproveRescheduleCommand`, `RejectRescheduleCommand`, `CancelRescheduleCommand`
- Endpoints:
  - `PATCH /api/v1/scheduling/reschedules/{id}/approve` (sem body)
  - `PATCH /api/v1/scheduling/reschedules/{id}/reject` — body `{ "reason": "..." }`
  - `PATCH /api/v1/scheduling/reschedules/{id}/cancel` — body `{ "reason": "..." }`

Todas exigem o reagendamento em `PENDING`.

| Ação | Efeito no reagendamento | Efeito na marcação e nas vagas |
|---|---|---|
| `approve` | `APPROVED`, `approvedAt` | toma a vaga no slot novo, devolve a do antigo, move a marcação (slot, `date`, `hour`) e limpa `hasReschedule`. **O estado da marcação não muda** (continua `PENDING`) |
| `reject` | `REJECTED`; motivo acrescentado a `reason` | marcação fica `REJECTED` no slot antigo, com `hasReschedule = false`; **a vaga antiga continua ocupada** |
| `cancel` | `CANCELLED`; motivo acrescentado a `reason` | **a marcação é cancelada** (`CANCELLED`, `cancelledBy` = quem pediu o reagendamento) e a sua vaga é libertada |

Em `reject` e `cancel`, `reason` é obrigatório (`400 ValidationFailed`).

### Sucesso esperado

- `204 No Content`

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `RescheduleNotFound` | reagendamento inexistente |
| `400` | `RescheduleCannotBeApproved` / `RescheduleCannotBeRejected` / `RescheduleCannotBeCancelled` | não está `PENDING` |
| `404` | `AppointmentNotFound`, `SlotNotFound`, `OldSlotNotFound` | dados relacionados em falta |
| `400` | `SlotHourUnavailable` | ao aprovar, a hora nova já não está disponível |

---

## Navegação

← [Marcações](./appointments.md) · [Índice do módulo](../index.md) · [Encaminhamentos](./forwardings.md) →

**Neste módulo:** [Visão Geral](./overview.md) · [Modelos e Enumerações](./models-and-enums.md) · [Agendas](./schedules.md) · [Slots](./slots.md) · [Marcações](./appointments.md) · **Reagendamentos** · [Encaminhamentos](./forwardings.md) · [Exclusões](./exclusions.md) · [Horários de Trabalho](./working-hours.md) · [Utilitários](./utilities.md) · [Casos Internos, Eventos e Notas](./internal-and-events.md)
