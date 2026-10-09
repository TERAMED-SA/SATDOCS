# Casos de Uso de Slots

[⌂ Módulo Scheduling](../index.md) · [Visão Geral](./overview.md)

Grupo de rotas: `/api/v1/scheduling/slots` (e `/api/v1/scheduling/schedules/{scheduleId}/slots`) · Requer autenticação (sem role específica).

Um slot é um dia de atendimento de uma agenda. Opera num de dois modos, decidido na geração:

| Modo | Quando | Vagas |
|---|---|---|
| `TIMED` | a agenda tem `startTime`, `endTime` e `appointmentDurationInMinutes` | grelha `Hours[]`: um `TimeBox` por intervalo de `appointmentDurationInMinutes`, cada um com `capacityPerInterval` vagas; `markingLimit` do slot = soma das vagas |
| `CAPACITY` | caso contrário | `markingLimit` = `maxAppointmentsPerSlot` vagas no dia, sem hora |

`markingLimit` representa sempre as **vagas restantes**: desce a cada marcação, sobe a cada cancelamento. O slot fecha sozinho quando chega a zero.

## 2.1 Gerar slots

- Casos de uso: `GenerateSlotsCommand` (síncrono), `GenerateSlotsAsyncCommand` (enfileira `TriggerNowSlotGenerationJob`)
- Endpoints:
  - `POST /api/v1/scheduling/slots/schedules/{scheduleId}/slots/generate-sync`
  - `POST /api/v1/scheduling/slots/schedules/{scheduleId}/slots/generate-async`

Sem body. Cada chamada gera **o próximo período** da agenda.

### Regras de negócio

1. a agenda existe e não está apagada (`ScheduleNotFound`) e tem configurações (`ScheduleDoesNotHaveConfigurations`)
2. o período a gerar é o ciclo número `timesItGenerated` a contar de `startDate`: uma semana em `WEEKLY`, um mês em `MONTHLY`, três em `QUARTERLY`, seis em `SEMIANNUALLY`, um ano em `ANNUALLY`
3. para cada dia do período, **não** é gerado slot se:
   - o dia da semana não está em `weekDays`
   - o dia está fora de `startDate`–`endDate`
   - uma exclusão de dia **associada à agenda** o exclui (data, dia da semana ou RRULE)
4. em `TIMED`, os horários abrangidos por um intervalo excluído **associado à agenda** são retirados da grelha
5. com `slotInheritProfessionals`, o slot recebe os profissionais da agenda que têm horário de trabalho ativo e válido nesse dia, para o hospital e a especialidade (`categoryCode`) da agenda
6. o slot nasce `PUBLISHED` com `autoPublishSlots`, senão `DRAFT`
7. se já existir slot para esse dia, é **atualizado**: os horários com marcações (não `AVAILABLE`) mantêm-se, os restantes são substituídos, e `markingLimit` é reposto pelo valor recalculado
8. no fim, `timesItGenerated` aumenta 1

Gerações concorrentes da mesma agenda (jobs, `generate-sync` e `generate-async` ao mesmo tempo) são serializadas por um advisory lock no PostgreSQL: a segunda espera que a primeira termine e gera o período seguinte.

Só as exclusões **associadas** à agenda afetam a geração; as exclusões de âmbito (`SYSTEM`, `HEALTH_UNIT`, `SPECIALITY`) só contam nas validações de disponibilidade e publicação — ver [Exclusões](./exclusions.md#âmbito-de-uma-exclusão).

### Sucesso esperado

- `202 Accepted` em ambos, sem corpo
- no síncrono, os slots já existem quando a resposta chega; no assíncrono, são criados pelo job

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `ScheduleNotFound` | agenda inexistente ou apagada |
| `404` | `ScheduleDoesNotHaveConfigurations` | agenda sem configurações (só no síncrono) |
| `422` | `EndDateBeforeMinimumRecurrence` | período calculado inválido (só no síncrono) |
| `500` | — | agenda com `typeOfRecurrence = NONE` — ver [Casos Internos](./internal-and-events.md#limitações-conhecidas) |

## 2.2 Listar slots

- Caso de uso: `ListSlotsQuery`
- Endpoints:
  - `GET /api/v1/scheduling/slots`
  - `GET /api/v1/scheduling/slots/available` — fixa `isClosed=false` e `hasAvailableHours=true`
  - `GET /api/v1/scheduling/slots/date/{date}` — fixa `dateFrom` = `dateTo` = `date`
  - `GET /api/v1/scheduling/slots/professional/{professionalTaxId}` — fixa `professionalTaxId`
  - `GET /api/v1/scheduling/schedules/{scheduleId}/slots` — fixa `scheduleId`

Todas aceitam os mesmos filtros.

### Query params

| Grupo | Parâmetros |
|---|---|
| Data | `scheduleId`, `scheduleIds`, `date`, `dateFrom`, `dateTo`, `weekDay`, `today`, `tomorrow`, `thisWeek`, `nextWeek`, `weekend` |
| Estado | `isClosed`, `mode`, `isActive`, `includeDeleted`, `onlyDeleted` |
| Horários | `hasHours`, `hoursCountMin`, `hoursCountMax`, `hasAvailableHours`, `hasBookedHours`, `hasBlockedHours`, `hourAvailable` (texto igual ao gravado na grelha, ex.: `08:30:00`) |
| Capacidade | `markingLimit`, `markingLimitMin`, `markingLimitMax`, `availableCapacityMin`, `availableCapacityMax`, `isFull`, `hasAvailableCapacity`, `occupancyRateMin`, `occupancyRateMax` |
| Profissionais | `professionalTaxId`, `availableProfessionalTaxIds`, `hasProfessionals`, `professionalsCountMin`, `professionalsCountMax`, `inheritAllProfessionals` |
| Agenda | `healthUnitTaxId`, `specialityId` (= `categoryCode`), `typeOfService`, `typeOfSchedule`, `scheduleTypeOfRecurrence`, `automaticallyGenerated` |
| Marcações | `hasAppointments`, `appointmentsCountMin`, `appointmentsCountMax`, `appointmentStatus` (marcações canceladas não contam) |
| Registo | `createdAtFrom`, `createdAtTo`, `updatedAtFrom`, `updatedAtTo` |
| Ordenação | `sortBy`: `date` (omissão), `createdAt`, `updatedAt`, `markingLimit`, `weekDay`, `availableCapacity` · `sortOrder`: `asc` (omissão) \| `desc` |
| Paginação e âmbito | `page`, `limit`, `unitIds`, `departmentIds`, `sectorIds` |

Os filtros `today`/`tomorrow`/`thisWeek`/`nextWeek`/`weekend` usam a data do servidor da API e só têm efeito com `true` (`false` é ignorado); a semana começa à segunda-feira.

`dateRange`, `timeRangeAvailable`, `updatedRecently`, `fields` e `include` existem na query mas **não têm efeito**.

### Exemplo de chamada

```bash
curl 'http://localhost:5000/api/v1/scheduling/slots/available?healthUnitTaxId=3f2b6c1e-8a4d-4f5e-9b7c-1d2e3f4a5b6c&dateFrom=2026-11-01&dateTo=2026-11-30' \
  -H 'Authorization: Bearer <jwt>'
```

### Sucesso esperado

- `200 OK` com `PaginatedList<SlotSummaryDto>`

```json
{
  "data": [
    {
      "id": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f",
      "scheduleId": "7d8e9f0a-1b2c-4d3e-8f4a-5b6c7d8e9f0a",
      "date": "2026-11-02",
      "weekDay": "MONDAY",
      "mode": "TIMED",
      "markingLimit": 7,
      "isClosed": false,
      "visibility": "PUBLISHED",
      "availableProfessionalTaxIds": ["5a6b7c8d-9e0f-4a1b-8c2d-3e4f5a6b7c8d"],
      "hours": [
        { "timeFrom": "08:00:00", "timeTo": "08:30:00", "markingLimit": 0, "status": "UNAVAILABLE" },
        { "timeFrom": "08:30:00", "timeTo": "09:00:00", "markingLimit": 1, "status": "AVAILABLE" }
      ],
      "reviewedByTaxId": null,
      "reviewedAt": null,
      "publishAt": null,
      "reviewNotes": null,
      "publishNotBefore": null,
      "publishExpiresAt": null,
      "unitId": null,
      "departmentId": null,
      "sectorId": null,
      "createdAt": "2026-10-09T10:00:00Z",
      "updatedAt": "2026-10-09T11:20:00Z",
      "bookedAppointments": 1,
      "totalHours": 8,
      "availableHours": 7,
      "healthUnitTaxId": "3f2b6c1e-8a4d-4f5e-9b7c-1d2e3f4a5b6c",
      "clinicalItemId": "b2c3d4e5-f6a7-4b8c-9d0e-1f2a3b4c5d6e",
      "category": "SPECIALITY",
      "categoryCode": "CARD",
      "typeOfService": "IN_PERSON",
      "typeOfSchedule": "INITIAL",
      "typeOfRecurrence": "MONTHLY"
    }
  ],
  "pagination": { "page": 1, "limit": 20, "totalItems": 1, "totalPages": 1, "hasNext": false, "hasPrev": false },
  "filters": { "applied": {}, "available": {} }
}
```

| Campo calculado | Significado |
|---|---|
| `bookedAppointments` | marcações não canceladas no slot |
| `totalHours` | número de horários da grelha (`0` em `CAPACITY`) |
| `availableHours` | horários `AVAILABLE` com vagas |
| `healthUnitTaxId` … `typeOfRecurrence` | resumo da agenda do slot |

## 2.3 Obter slot por id

- Caso de uso: `GetSlotByIdQuery`
- Endpoint: `GET /api/v1/scheduling/slots/{id}`

### Sucesso esperado

- `200 OK` com a forma resumida `SlotMinimalDto`:

```json
{
  "id": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f",
  "date": "2026-11-02",
  "mode": "TIMED",
  "weekDay": "MONDAY",
  "markingLimit": 7
}
```

Para a informação completa (grelha, profissionais, agenda), use a listagem com `?scheduleId=` ou `GET /slots/hours/{id}`.

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `SlotNotFound` | slot inexistente ou apagado |

## 2.4 Obter grelha horária

- Caso de uso: `GetSlotHoursQuery`
- Endpoint: `GET /api/v1/scheduling/slots/hours/{id}`

### Sucesso esperado

- `200 OK`

```json
{
  "totalHours": 8,
  "totalVacancies": 7,
  "mode": "TIMED",
  "hours": [
    { "status": "UNAVAILABLE", "timeFrom": "08:00:00", "timeTo": "08:30:00", "markingLimit": 0 },
    { "status": "AVAILABLE", "timeFrom": "08:30:00", "timeTo": "09:00:00", "markingLimit": 1 }
  ]
}
```

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `SlotUnavailable` | slot inexistente ou apagado |
| `422` | `SlotOperationModeIsNotTimed` | slot `CAPACITY` (sem grelha) |

## 2.5 Profissionais do slot

- Casos de uso: `AssignProfessionalCommand`, `RemoveProfessionalCommand`
- Endpoints: `POST` / `DELETE /api/v1/scheduling/slots/{id}/professionals`

### Body

```json
{ "professionalTaxIds": ["5a6b7c8d-9e0f-4a1b-8c2d-3e4f5a6b7c8d"] }
```

### Regras de negócio

- **associar:** lista não vazia, elementos não vazios e com no máximo 50 caracteres (`400 ValidationFailed`); os já associados são ignorados
- **remover:** remove os indicados; os que não estão associados são ignorados
- a lista do slot define quem pode receber marcações nele — ver [Marcações 3.1](./appointments.md#31-criar-marcação)

### Sucesso esperado

- `204 No Content`

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `SlotUnavailable` | slot inexistente |

## 2.6 Alterar capacidade

- Caso de uso: `UpdateSlotCapacityCommand`
- Endpoint: `PATCH /api/v1/scheduling/slots/{id}/capacity`

### Body

```json
{ "capacity": 15 }
```

### Regras de negócio

1. `capacity ≥ 0` (`400 ValidationFailed`)
2. o slot existe (`404 SlotUnavailable`)
3. só slots `CAPACITY`: num slot `TIMED` a capacidade é a soma da grelha e não pode ser escrita diretamente (`422 TimedSlotMarkingLimitViolation`)
4. `capacity` substitui as **vagas restantes** (`markingLimit`), não a capacidade total

### Sucesso esperado

- `204 No Content`

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `400` | `ValidationFailed` | capacidade negativa |
| `404` | `SlotUnavailable` | slot inexistente |
| `422` | `TimedSlotMarkingLimitViolation` | slot `TIMED` |

## 2.7 Abrir e fechar slot

- Casos de uso: `OpenSlotCommand`, `CloseSlotCommand`
- Endpoints: `PATCH /api/v1/scheduling/slots/{id}/open` · `PATCH /api/v1/scheduling/slots/{id}/close`

### Regras de negócio

- fechar impede novas marcações (`SlotUnavailable` em [3.1](./appointments.md#31-criar-marcação)); é sempre permitido
- abrir só é permitido se o slot ainda tiver vagas (`422 SlotCapacityReached`)

### Sucesso esperado

- `204 No Content`

## 2.8 Apagar os slots de uma agenda

- Caso de uso: `HardDeleteScheduleSlotsCommand`
- Endpoint: `DELETE /api/v1/scheduling/slots/hard-delete/schedules/{scheduleId}/slots`

### Regras de negócio

- a agenda existe e não está apagada (`404 ScheduleNotFound`)
- se algum slot tiver marcações, **nada é apagado** e a resposta continua a ser `204`
- caso contrário, todos os slots da agenda são removidos fisicamente
- `timesItGenerated` **não** é reposto: a próxima geração continua no período seguinte

### Sucesso esperado

- `204 No Content`

---

## Navegação

← [Agendas](./schedules.md) · [Índice do módulo](../index.md) · [Marcações](./appointments.md) →

**Neste módulo:** [Visão Geral](./overview.md) · [Modelos e Enumerações](./models-and-enums.md) · [Agendas](./schedules.md) · **Slots** · [Marcações](./appointments.md) · [Reagendamentos](./reschedules.md) · [Encaminhamentos](./forwardings.md) · [Exclusões](./exclusions.md) · [Horários de Trabalho](./working-hours.md) · [Utilitários](./utilities.md) · [Casos Internos, Eventos e Notas](./internal-and-events.md)
