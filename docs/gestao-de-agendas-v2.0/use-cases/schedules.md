# Casos de Uso de Agendas

[⌂ Módulo Scheduling](../index.md) · [Visão Geral](./overview.md)

Grupo de rotas: `/api/v1/scheduling/schedules` · Requer autenticação (sem role específica).

A agenda define quando, onde e como um item clínico é atendido num hospital. Os slots são gerados a partir dela — ver [Slots](./slots.md).

## 1.1 Criar agenda

- Caso de uso: `CreateScheduleCommand`
- Endpoint: `POST /api/v1/scheduling/schedules`
- Modelos principais: `Schedule`, `ScheduleConfigurations`

### Fluxo obrigatório: validar antes de criar

Uma agenda só pode ser criada com dados que **já passaram** por [`POST /utilities/validate/schedule`](./utilities.md#81-validar-agenda):

1. o cliente envia o body ao validador
2. se a agenda for válida (sem avisos bloqueantes), a resposta traz `validationHash` e `validationHashExpiresAt` (30 minutos)
3. o cliente envia **exatamente o mesmo body** para `POST /schedules`, acrescentando `validationHash`
4. o handler volta a calcular o hash (SHA-256 do body sem `validationHash`) e compara-o com o enviado
5. depois de a agenda ser criada, o hash é consumido e não pode ser reutilizado

Qualquer alteração ao body depois da validação — incluindo a ordem dos elementos de uma lista — produz outro hash e devolve `ScheduleValidationHashMismatch`.

### Regras de negócio, por ordem de validação

Validação do pedido (`400 ValidationFailed`):

1. `healthUnitTaxId` obrigatório e um GUID válido
2. `clinicalItemId` e `categoryCode` obrigatórios
3. `startDate ≤ endDate`, quando há `endDate`
4. `endTime > startTime`, quando ambos existem
5. `weekDays` não vazio quando `typeOfRecurrence ≠ NONE`
6. `typeOfRecurrence` não pode ser `DAILY` nem `CUSTOM`
7. categoria compatível com o serviço: `CLINIC` não é aceite em `HOME` nem em `FOLLOW_UP`; `VACCINE` e `EXAM` não são aceites em `TELEMEDICINE`
8. com janela horária (`startTime` e `endTime`): `appointmentDurationInMinutes > 0`, `capacityPerInterval > 0` e `maxAppointmentsPerSlot` vazio
9. pelo menos um de `appointmentDurationInMinutes` ou `maxAppointmentsPerSlot`
10. `locationMetadata` com pelo menos um local quando `useLocation = true`
11. `scheduleContext` válido

Regras de negócio (no handler):

12. `validationHash` presente e emitido para estes dados (`ScheduleValidationRequired`, `ScheduleValidationHashMismatch`)
13. o item clínico existe no catálogo (`ItemNotFound`)
14. o hospital existe e está ativo (`HospitalNotFound`)
15. o hospital oferece o item clínico (`HospitalDoesNotOfferItem`) e a sua especialidade (`HospitalDoesNotHaveSpecialty`)
16. com `useLocation`, cada nível de cada local tem de existir, estar ativo e pertencer ao nível acima (unidade e departamento ao hospital) — `LocationIncompatibleWithService`
17. o período `startDate → endDate` cobre pelo menos um ciclo de recorrência (`EndDateBeforeMinimumRecurrence`): 6 dias em `WEEKLY`, 27 em `MONTHLY`, 84 em `QUARTERLY`, 182 em `SEMIANNUALLY`, 364 em `ANNUALLY`

Depois de gravar:

- os `excludeDayIds`/`excludeRangeIds` que existem são associados à agenda; **ids inexistentes são ignorados em silêncio**
- é publicado `ScheduleCreatedEvent`
- com `automaticallyGenerateSlots = true`, é agendado o job de geração — ver [Casos Internos](./internal-and-events.md#geração-de-slots-em-background)

### Body

```json
{
  "healthUnitTaxId": "3f2b6c1e-8a4d-4f5e-9b7c-1d2e3f4a5b6c",
  "clinicalItemId": "b2c3d4e5-f6a7-4b8c-9d0e-1f2a3b4c5d6e",
  "weekDays": ["MONDAY", "WEDNESDAY", "FRIDAY"],
  "startTime": "08:00",
  "endTime": "12:00",
  "typeOfRecurrence": "MONTHLY",
  "typeOfSchedule": "INITIAL",
  "scheduleContext": "PUBLIC",
  "category": "SPECIALITY",
  "categoryCode": "CARD",
  "typeOfService": "IN_PERSON",
  "startDate": "2026-11-01",
  "endDate": "2027-04-30",
  "unitId": null,
  "departmentId": null,
  "sectorId": null,
  "configurations": {
    "appointmentDurationInMinutes": 30,
    "capacityPerInterval": 1,
    "automaticallyGenerateSlots": true,
    "slotInheritProfessionals": true,
    "availableProfessionalTaxIds": ["5a6b7c8d-9e0f-4a1b-8c2d-3e4f5a6b7c8d"],
    "autoPublishSlots": false,
    "requireSupervisorApproval": false,
    "genderRestrictions": "NONE",
    "minimumAgeInDays": null,
    "maximumAgeInDays": null,
    "paymentRequired": false,
    "deadlineForSlotBookingInHours": 24,
    "useLocation": false,
    "locationMetadata": null
  },
  "excludeDayIds": [],
  "excludeRangeIds": [],
  "validationHash": "9f2c...e41a"
}
```

| Campo | Obrigatório | Omissão |
|---|---|---|
| `healthUnitTaxId` | sim | — |
| `clinicalItemId` | sim | — |
| `typeOfRecurrence`, `typeOfSchedule`, `scheduleContext`, `category`, `categoryCode`, `typeOfService` | sim | — |
| `startDate` | sim | — |
| `configurations` | sim | — |
| `validationHash` | sim | — |
| `weekDays` | se houver recorrência | `[]` |
| `startTime` / `endTime` | não | `null` (slots `CAPACITY`) |
| `endDate` | não | `null` (sem fim) |
| `unitId`, `departmentId`, `sectorId` | não | `null` |
| `excludeDayIds`, `excludeRangeIds` | não | `[]` |

Os campos de `configurations` estão descritos em [Modelos e Enumerações](./models-and-enums.md#scheduleconfigurations).

**Modo dos slots:** com `startTime`, `endTime` e `appointmentDurationInMinutes`, os slots são `TIMED` (grelha de horários de `appointmentDurationInMinutes`, cada um com `capacityPerInterval` vagas). Sem janela horária, são `CAPACITY` com `maxAppointmentsPerSlot` vagas por dia.

### Exemplo de chamada

```bash
curl -X POST http://localhost:5000/api/v1/scheduling/schedules \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <jwt>' \
  -d @agenda.json
```

### Sucesso esperado

- `201 Created`, com `Location` a apontar para `GET /api/v1/scheduling/schedules/{id}`

```json
{ "id": "7d8e9f0a-1b2c-4d3e-8f4a-5b6c7d8e9f0a" }
```

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `400` | `ValidationFailed` | regra de validação do pedido violada |
| `422` | `ScheduleValidationRequired` | `validationHash` em falta, expirado ou já usado |
| `422` | `ScheduleValidationHashMismatch` | o body não é o que foi validado |
| `404` | `ItemNotFound` | item clínico inexistente |
| `404` | `HospitalNotFound` | hospital inexistente ou inativo |
| `422` | `HospitalDoesNotOfferItem` | hospital não oferece o item |
| `422` | `HospitalDoesNotHaveSpecialty` | hospital não tem a especialidade do item |
| `422` | `LocationIncompatibleWithService` | local inexistente, inativo ou fora do hospital |
| `422` | `EndDateBeforeMinimumRecurrence` | período mais curto que um ciclo de recorrência |
| `401` | — | não autenticado |

## 1.2 Listar agendas

- Caso de uso: `ListSchedulesQuery`
- Endpoints:
  - `GET /api/v1/scheduling/schedules`
  - `GET /api/v1/scheduling/schedules/health-unit/{healthUnitTaxId}` — fixa `healthUnitTaxId`, aceita os restantes filtros
  - `GET /api/v1/scheduling/schedules/professional/{professionalTaxId}?page=&limit=` — só `page`/`limit`, ambos obrigatórios
  - `GET /api/v1/scheduling/schedules/speciality/{categoryCode}?page=&limit=` — só `page`/`limit`, ambos obrigatórios

### Query params

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `healthUnitTaxId`, `categoryCode`, `createdBy` | string | igualdade |
| `unitId`, `departmentId`, `sectorId` | guid | igualdade |
| `categoryOfSchedule` | `CategoryOfSchedule` | |
| `typeOfService`, `typeOfSchedule`, `typeOfRecurrence` | enumeração | |
| `startDateFrom`, `startDateTo`, `endDateFrom`, `endDateTo` | date | intervalos sobre o período |
| `activeOn` | date | agendas cujo período contém a data |
| `weekDays` | lista de `WeekDay` | tem **algum** dos dias |
| `hasRecurrence` | bool | `typeOfRecurrence ≠ NONE` |
| `mode` | `OperationMode` | `TIMED` = tem duração; `CAPACITY` = sem duração; `UNLIMITED` = capacidade ilimitada |
| `hasAppointmentDuration`, `hasMaxAppointments`, `paymentRequired`, `automaticallyGenerateSlots`, `eachSlotInheritAllScheduleProfessionals` | bool | configurações |
| `professionalTaxId` | string | o profissional está na agenda |
| `availableProfessionalTaxIds` | lista | tem **algum** destes profissionais |
| `hasProfessionals`, `hasSlots`, `hasExcludeDays`, `hasExcludeRanges`, `hasExclusions` | bool | existência |
| `createdAtFrom/To`, `updatedAtFrom/To` | datetime | |
| `search` | string | `categoryCode` contém o termo (sem distinguir maiúsculas) |
| `isActive` | bool | não apagada |
| `includeDeleted` / `onlyDeleted` | bool | por omissão as apagadas ficam de fora |
| `sortBy` | `startDate` \| `createdAt` | omissão: `createdAt` |
| `sortOrder` | `asc` \| `desc` | omissão: `desc` |
| `page`, `limit`, `unitIds`, `departmentIds`, `sectorIds` | | ver [Visão Geral](./overview.md#envelopes-de-paginação) |

### Exemplo de chamada

```bash
curl 'http://localhost:5000/api/v1/scheduling/schedules?activeOn=2026-11-15&typeOfService=IN_PERSON&limit=50' \
  -H 'Authorization: Bearer <jwt>'
```

### Sucesso esperado

- `200 OK` com `PaginatedList<ScheduleDto>`; cada agenda traz as `configurations` embutidas, no formato de [1.3](#13-obter-agenda-por-id)

## 1.3 Obter agenda por id

- Caso de uso: `GetScheduleByIdQuery`
- Endpoint: `GET /api/v1/scheduling/schedules/{id}`

### Sucesso esperado

- `200 OK`

```json
{
  "id": "7d8e9f0a-1b2c-4d3e-8f4a-5b6c7d8e9f0a",
  "healthUnitTaxId": "3f2b6c1e-8a4d-4f5e-9b7c-1d2e3f4a5b6c",
  "clinicalItemId": "b2c3d4e5-f6a7-4b8c-9d0e-1f2a3b4c5d6e",
  "weekDays": ["MONDAY", "WEDNESDAY", "FRIDAY"],
  "startTime": "08:00:00",
  "endTime": "12:00:00",
  "typeOfRecurrence": "MONTHLY",
  "typeOfSchedule": "INITIAL",
  "scheduleContext": "PUBLIC",
  "category": "SPECIALITY",
  "categoryCode": "CARD",
  "typeOfService": "IN_PERSON",
  "startDate": "2026-11-01",
  "endDate": "2027-04-30",
  "unitId": null,
  "departmentId": null,
  "sectorId": null,
  "configurations": {
    "appointmentDurationInMinutes": 30,
    "capacityPerInterval": 1,
    "automaticallyGenerateSlots": true,
    "timesItGenerated": 1,
    "availableProfessionalTaxIds": ["5a6b7c8d-9e0f-4a1b-8c2d-3e4f5a6b7c8d"]
  }
}
```

(`configurations` abreviado — traz todos os campos de `ScheduleConfigurationsDto`.)

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `ScheduleNotFound` | agenda inexistente **ou apagada** |

## 1.4 Clonar agenda

- Caso de uso: `CloneScheduleCommand`
- Endpoint: `POST /api/v1/scheduling/schedules/{id}/clone` (sem body)

### Regras de negócio

- a agenda de origem tem de existir e não estar apagada (`ScheduleNotFound`)
- copia a agenda e as configurações com novos ids, **incluindo `timesItGenerated`**
- **não** copia as exclusões associadas, **não** gera slots, **não** agenda jobs nem passa pela validação

### Sucesso esperado

- `201 Created`, `Location` para a nova agenda, corpo `{ "id": "<nova agenda>" }`

## 1.5 Apagar e restaurar agenda

- Casos de uso: `DeleteScheduleCommand`, `RestoreScheduleCommand`
- Endpoints: `DELETE /api/v1/scheduling/schedules/{id}` · `POST /api/v1/scheduling/schedules/{id}/restore`

### Regras de negócio

- apagar é **soft delete** (`DeletedAt`); falha se algum slot da agenda tiver marcações (`ScheduleHasSlotsWithAppointments`)
- apagar enfileira a remoção física dos slots e publica `ScheduleDeletedEvent` — ver a nota sobre este job em [Casos Internos](./internal-and-events.md#limitações-conhecidas)
- restaurar limpa `DeletedAt`; numa agenda que não está apagada não faz nada. Os slots já removidos não voltam

### Sucesso esperado

- `204 No Content` em ambos

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `ScheduleNotFound` | agenda inexistente (ou já apagada, no `DELETE`) |
| `422` | `ScheduleHasSlotsWithAppointments` | há marcações nos slots |

## 1.6 Alterar o período

- Caso de uso: `UpdateScheduleDatesCommand`
- Endpoint: `PATCH /api/v1/scheduling/schedules/{id}/dates`

### Body

```json
{ "startDate": "2026-12-01", "endDate": "2027-05-31" }
```

### Regras de negócio

1. `startDate` obrigatório (`400 ValidationFailed`)
2. a agenda existe e não está apagada (`ScheduleNotFound`)
3. `endDate ≥ startDate` (`EndDateTooEarly`)
4. o período cobre pelo menos um ciclo de recorrência (`EndDateBeforeMinimumRecurrence`)

Os slots já gerados **não** são ajustados ao novo período.

### Sucesso esperado

- `204 No Content`

## 1.7 Alterar a RRULE de geração

- Caso de uso: `UpdateRruleCommand`
- Endpoint: `PATCH /api/v1/scheduling/schedules/{id}/rrule`

### Body

Uma string JSON (ou `null` para remover):

```json
"FREQ=MONTHLY;BYMONTHDAY=25"
```

### Regras de negócio

- a agenda tem de ter configurações (`ScheduleNotFound`)
- a RRULE **não é validada** aqui — use [`/utilities/validate/rrule`](./utilities.md#85-validar-rrule) antes
- a alteração **não** reagenda o job de geração já existente — ver [Casos Internos](./internal-and-events.md#geração-de-slots-em-background)

### Sucesso esperado

- `204 No Content`

## 1.8 Dias da semana

- Casos de uso: `AddWeekDayCommand`, `RemoveWeekDayCommand`
- Endpoints: `POST` / `DELETE /api/v1/scheduling/schedules/{id}/weekdays`

### Body

Uma string JSON com o dia:

```json
"TUESDAY"
```

### Regras de negócio

- adicionar um dia que já existe, ou remover um que não existe, não faz nada
- não é possível remover o último dia de uma agenda com recorrência (`422 ScheduleWeekDaysRequired`)
- os slots já gerados não são alterados

### Sucesso esperado

- `204 No Content`

## 1.9 Profissionais da agenda

- Casos de uso: `AddProfessionalCommand`, `RemoveProfessionalCommand`
- Endpoints:
  - `POST /api/v1/scheduling/schedules/{id}/professionals` — vários
  - `POST /api/v1/scheduling/schedules/{id}/professionals/{taxId}` — um
  - `DELETE /api/v1/scheduling/schedules/{id}/professionals/{taxId}` — remover um

### Body (associar vários)

```json
{ "professionalTaxIds": ["5a6b7c8d-...", "6b7c8d9e-..."] }
```

### Regras de negócio

- a lista não pode ser vazia nem ter elementos vazios (`400 ValidationFailed`)
- os ids são aparados, duplicados e já associados são ignorados
- remover um profissional que não está associado não faz nada
- a alteração afeta a atribuição em novas marcações e as próximas gerações de slots; os slots existentes mantêm a sua lista — ver [Slots 2.5](./slots.md#25-profissionais-do-slot)
- não é verificado se o profissional existe nem se tem horário de trabalho

### Sucesso esperado

- associar: `201 Created`, `Location` para a agenda

```json
{
  "scheduleId": "7d8e9f0a-1b2c-4d3e-8f4a-5b6c7d8e9f0a",
  "professionalTaxIds": ["5a6b7c8d-...", "6b7c8d9e-..."]
}
```

`professionalTaxIds` é a lista **completa** de profissionais da agenda depois da operação.

- remover: `204 No Content`

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `400` | `ValidationFailed` | lista vazia ou com elementos vazios |
| `404` | `ScheduleNotFound` | agenda sem configurações |

## 1.10 Localizações

- Casos de uso: `AddLocationCommand`, `RemoveLocationCommand`, `UpdateLocationStatusCommand`
- Endpoints:
  - `POST /api/v1/scheduling/schedules/{id}/locations`
  - `DELETE /api/v1/scheduling/schedules/{id}/locations`
  - `PATCH /api/v1/scheduling/schedules/{id}/locations/status`

### Body

Um `LocationMetadata`:

```json
{
  "unitId": "1a2b3c4d-...",
  "departmentId": null,
  "buildingId": null,
  "floorId": null,
  "sectorId": null,
  "roomId": "9e8d7c6b-...",
  "bedId": null,
  "isActive": true
}
```

### Regras de negócio

- **adicionar:** pelo menos um nível preenchido (`400 ValidationFailed`); cada nível tem de existir, estar ativo e pertencer ao nível acima — unidade e departamento ao hospital da agenda (`422 LocationIncompatibleWithService`); um local igual a um já existente é ignorado
- **remover:** remove o local com os mesmos sete níveis; se não existir, não faz nada
- **estado:** procura o local com os mesmos níveis e grava o `isActive` enviado (`404 LocationNotFound` se não existir)

### Sucesso esperado

- `204 No Content`

## 1.11 Associar exclusões

- Casos de uso: `AssociateExcludeDayCommand`, `DisassociateExcludeDayCommand`, `AssociateExcludeRangeCommand`, `DisassociateExcludeRangeCommand`
- Endpoints:
  - `POST` / `DELETE /api/v1/scheduling/schedules/{scheduleId}/exclude-days/{excludeDayId}`
  - `POST` / `DELETE /api/v1/scheduling/schedules/{scheduleId}/exclude-ranges/{excludeRangeId}`

### Regras de negócio

- a agenda e a exclusão têm de existir e não estar apagadas
- associar duas vezes devolve `409`
- desassociar uma exclusão que não está associada não faz nada
- as associações contam para a geração de slots e para as validações; ver [Exclusões](./exclusions.md#âmbito-de-uma-exclusão)

### Sucesso esperado

- `204 No Content`

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `ScheduleNotFound` | agenda inexistente ou apagada |
| `404` | `ExcludeDayNotFound` / `ExcludeRangeNotFound` | exclusão inexistente ou apagada |
| `409` | `ExcludeDayAlreadyAssociated` / `ExcludeRangeAlreadyAssociated` | já associada |

---

## Navegação

← [Modelos e Enumerações](./models-and-enums.md) · [Índice do módulo](../index.md) · [Slots](./slots.md) →

**Neste módulo:** [Visão Geral](./overview.md) · [Modelos e Enumerações](./models-and-enums.md) · **Agendas** · [Slots](./slots.md) · [Marcações](./appointments.md) · [Reagendamentos](./reschedules.md) · [Encaminhamentos](./forwardings.md) · [Exclusões](./exclusions.md) · [Horários de Trabalho](./working-hours.md) · [Utilitários](./utilities.md) · [Casos Internos, Eventos e Notas](./internal-and-events.md)
