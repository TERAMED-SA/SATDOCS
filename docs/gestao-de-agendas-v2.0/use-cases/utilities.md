# Casos de Uso de Utilitários

[⌂ Módulo Scheduling](../index.md) · [Visão Geral](./overview.md)

Grupo de rotas: `/api/v1/scheduling/utilities` · Requer autenticação (sem role específica).

Os utilitários **não gravam dados** (com uma exceção: o hash de validação de agenda, guardado em cache). Servem para o cliente verificar um pedido antes de o submeter e para obter os valores das enumerações.

## Formato dos resultados de validação

As validações de agenda, disponibilidade e publicação devolvem `ValidationResult<TCode>`:

```json
{
  "valid": false,
  "warnings": [
    { "code": "SCHEDULE_OPEN_ENDED_RECURRENCE", "message": "Recorrência definida sem data de término." },
    { "code": "SCHEDULE_WITHOUT_AVAILABLE_PROFESSIONALS", "message": "A agenda não tem profissionais disponíveis." }
  ],
  "blockingWarnings": ["SCHEDULE_WITHOUT_AVAILABLE_PROFESSIONALS"],
  "effectiveRules": null
}
```

- `warnings`: todos os avisos encontrados, com mensagem em português
- `blockingWarnings`: os códigos **bloqueantes** (marcados com `[BlockingWarning]` na enumeração)
- `valid`: `true` se não houver avisos bloqueantes. Avisos não bloqueantes são informativos
- a lista de códigos bloqueantes pode ser obtida em `GET /utilities/enums/*-blocking-warnings`

Estas rotas devolvem sempre `200 OK` quando conseguem avaliar o pedido — um resultado inválido **não** é um erro HTTP.

## 8.1 Validar agenda

- Caso de uso: `ValidateScheduleCommand`
- Endpoint: `POST /api/v1/scheduling/utilities/validate/schedule`

É o **primeiro passo obrigatório** para [criar uma agenda](./schedules.md#11-criar-agenda).

### Body

O mesmo body de [`POST /schedules`](./schedules.md#body), sem `validationHash`.

### Regras de negócio

- carrega as exclusões indicadas em `excludeDayIds`/`excludeRangeIds` (ids inexistentes são ignorados)
- aplica `ScheduleLogic.ValidateBussinessRule` e devolve os avisos abaixo
- **não** aplica as validações de pedido de `POST /schedules` nem consulta o hospital ou o catálogo: uma agenda válida aqui ainda pode ser rejeitada na criação
- se o resultado for válido, calcula o SHA-256 do body, guarda-o em cache durante **30 minutos** e devolve-o

### Avisos

| Código | Bloqueante | Significado |
|---|---|---|
| `SCHEDULE_UNLIMITED_WITH_MAX_CAPACITY` | sim | capacidade ilimitada e máximo por dia ao mesmo tempo |
| `SCHEDULE_MAX_CAPACITY_INVALID` | sim | capacidade máxima por dia inválida |
| `SCHEDULE_SLOTS_HOURS_CAPACITY_REQUIRED` | sim | falta o número de pacientes por horário |
| `SCHEDULE_NO_CAPACITY_DEFINED` | sim | sem duração de consulta nem limite de marcações |
| `SCHEDULE_INVALID_MAX_APPOINTMENTS` | sim | máximo de marcações por dia não é superior a zero |
| `SCHEDULE_HOME_CLINIC_INVALID` | sim | categoria `CLINIC` em serviço ao domicílio |
| `SCHEDULE_FOLLOWUP_CLINIC_INVALID` | não | categoria `CLINIC` em agenda de seguimento |
| `SCHEDULE_TELEMEDICINE_INVALID_CATEGORY` | não | `EXAM` ou `VACCINE` em telemedicina |
| `SCHEDULE_DATE_RANGE_INVALID` | sim | data de fim anterior à de início |
| `SCHEDULE_OPEN_ENDED_RECURRENCE` | não | recorrência sem data de fim |
| `SCHEDULE_EXCLUDES_OVERLAP` | sim | as exclusões cobrem todas as datas ou horários |
| `SCHEDULE_PAYMENT_REQUIRED_WITHOUT_BOOKING_DEADLINE` | não | pagamento obrigatório sem prazo de reserva |
| `SCHEDULE_WITHOUT_AVAILABLE_PROFESSIONALS` | sim | sem profissionais |
| `SCHEDULE_PROFESSIONALS_DEFINED_NOT_INHERITED` | não | há profissionais mas os slots não os herdam |
| `SCHEDULE_AUTO_PUBLISH_WITHOUT_BUFFER` | não | publicação automática sem `defaultPublishBufferHours` |
| `SCHEDULE_APPROVAL_REQUIRED_WITHOUT_TTL` | não | aprovação de supervisor sem `defaultReviewTTLHours` |
| `SCHEDULE_WEEKLY_RECURRENCE_WITHOUT_WEEKDAYS` | sim | recorrência semanal sem dias da semana |
| `SCHEDULE_WEEKDAYS_DEFINED_WITHOUT_RECURRENCE` | não | dias da semana sem recorrência |
| `SCHEDULE_TIME_WINDOW_INVALID` | sim | hora de início ≥ hora de fim |
| `SCHEDULE_INVALID_DURATION` | sim | duração da consulta não é superior a zero |
| `SCHEDULE_WINDOW_NOT_MULTIPLE_OF_APPOINTMENT_DURATION` | não | a janela não é múltipla da duração; os minutos finais são ignorados |

### Sucesso esperado

- `200 OK`

```json
{
  "valid": true,
  "warnings": [
    { "code": "SCHEDULE_PAYMENT_REQUIRED_WITHOUT_BOOKING_DEADLINE", "message": "O pagamento é obrigatório, mas o prazo para reserva de vagas não foi definido." }
  ],
  "blockingWarnings": [],
  "effectiveRules": { "recurrence": "MONTHLY", "weekDays": ["MONDAY", "WEDNESDAY", "FRIDAY"], "mode": "TIMED" },
  "validationHash": "9f2c7b1e0d4a...e41a",
  "validationHashExpiresAt": "2026-10-09T15:30:00Z"
}
```

- `effectiveRules`: recorrência, dias da semana e modo de operação que a agenda vai usar
- `validationHash` e `validationHashExpiresAt` só vêm preenchidos quando `valid = true`
- validar o mesmo body de novo renova o prazo do mesmo hash

## 8.2 Validar disponibilidade de slot

- Caso de uso: `ValidateSlotAvailabilityQuery`
- Endpoint: `POST /api/v1/scheduling/utilities/validate/slot-availability`

Diz se um slot (e uma hora) pode receber uma marcação de um profissional.

### Body

```json
{
  "slotId": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f",
  "selectedHour": "08:30",
  "professionalTaxId": "5a6b7c8d-9e0f-4a1b-8c2d-3e4f5a6b7c8d",
  "patientTaxId": "0f1e2d3c-4b5a-4968-8776-5a4b3c2d1e0f"
}
```

### Regras de negócio

- considera as exclusões **associadas** à agenda e as de **âmbito** — ver [Exclusões](./exclusions.md#âmbito-de-uma-exclusão)
- em todos os modos: o dia não pode estar excluído (`SLOT_DAY_EXCLUDED`)
- slot `TIMED`: `selectedHour` obrigatória (`SELECTED_HOUR_REQUIRED`), disponível (`SLOT_HOUR_UNAVAILABLE`), fora de intervalos excluídos (`SLOT_HOUR_EXCLUDED`), e o profissional autorizado no slot (`PROFESSIONAL_NOT_ALLOWED`)
- slot `CAPACITY`: aberto (`SLOT_CLOSED`) e com capacidade (`SLOT_CAPACITY_REACHED`)
- `patientTaxId` não é usado
- **todos** os avisos de disponibilidade são bloqueantes

### Sucesso esperado

- `200 OK` com `ValidationResult<SlotAvailabilityWarningCode>`

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `SlotNotFound` | slot inexistente |
| `404` | `ScheduleNotFound` / `ScheduleDoesNotHaveConfigurations` | agenda do slot em falta |

## 8.3 Validar publicação de slot

- Caso de uso: `ValidateSlotPublicationQuery`
- Endpoint: `POST /api/v1/scheduling/utilities/validate/slot-publication`

Diz se um slot está em condições de ser publicado.

### Body

```json
{ "slotId": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f" }
```

### Avisos (todos bloqueantes)

| Código | Significado |
|---|---|
| `SLOT_OUTSIDE_SCHEDULE_PERIOD` | a data do slot está fora do período da agenda |
| `SLOT_MARKING_LIMIT_LESS_THAN_AVAILABLE_VACANCIES` | o limite de marcações é menor que as vagas da grelha |
| `SLOT_DAY_EXCLUDED` | o dia está excluído |
| `SLOT_HOURS_COLLIDE_WITH_EXCLUDES` | algum horário colide com um intervalo excluído |
| `PROFESSIONALS_MISSING_WORKING_HOURS` | nem todos os profissionais do slot têm horário de trabalho |

Não existe endpoint para publicar: esta validação é informativa. A visibilidade do slot é definida na geração (`autoPublishSlots`).

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `SlotNotFound` | slot inexistente |
| `404` | `ScheduleNotFound` / `ScheduleDoesNotHaveConfigurations` | agenda do slot em falta |

## 8.4 Pré-validar marcação

- Caso de uso: `ValidateAppointmentQuery`
- Endpoint: `POST /api/v1/scheduling/utilities/validate/appointment`

Ao contrário das anteriores, **falha com erro HTTP** na primeira regra violada.

### Body

```json
{
  "slotId": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f",
  "selectedHour": "08:30",
  "patientTaxId": "0f1e2d3c-4b5a-4968-8776-5a4b3c2d1e0f",
  "professionalTaxId": "5a6b7c8d-9e0f-4a1b-8c2d-3e4f5a6b7c8d",
  "typeOfService": "IN_PERSON"
}
```

### Regras de negócio, por ordem de validação

1. `slotId` e `patientTaxId` obrigatórios (`400 ValidationFailed`)
2. o slot, a agenda e as configurações existem (`404 SlotNotFound` / `ScheduleNotFound`)
3. com `deadlineForSlotBookingInHours`, a reserva ainda está dentro do prazo (`409 BookingDeadlineExceeded`)
4. slot `TIMED`: `selectedHour` obrigatória (`409 SelectedHourRequired`) e existente na grelha (`409 SlotHourUnavailable`) — **não** verifica se o horário ainda tem vagas
5. slot `CAPACITY` sem capacidade ilimitada: ainda há vagas (`409 SlotCapacityReached`)
6. o profissional não tem outra marcação à mesma data e hora (`409 ProfessionalHasAppoitmentOnThisTime`)
7. o paciente não tem outra marcação à mesma data e hora (`409 PatientHasAppoitmentOnThisTime`)

`typeOfService` não é usado.

### Sucesso esperado

- `200 OK`

```json
{
  "valid": true,
  "normalized": {
    "slotId": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f",
    "selectedHour": "08:30:00",
    "status": "PENDING"
  }
}
```

`normalized.status` é sempre `PENDING`; o estado real da marcação é decidido na criação — ver [Marcações 3.1](./appointments.md#efeitos).

## 8.5 Validar RRULE

- Caso de uso: `ValidateRRuleQuery`
- Endpoint: `POST /api/v1/scheduling/utilities/validate/rrule`

Expande uma RRULE (RFC 5545) entre duas datas. Útil antes de [alterar a RRULE de uma agenda](./schedules.md#17-alterar-a-rrule-de-geração) ou criar uma exclusão com `rrule`.

### Body

```json
{
  "rrule": "FREQ=WEEKLY;BYDAY=MO,WE",
  "from": "2026-11-01",
  "to": "2026-11-30",
  "maxInstances": 5
}
```

| Campo | Obrigatório | Notas |
|---|---|---|
| `rrule` | sim | sem o prefixo `RRULE:` |
| `from`, `to` | sim | intervalo de expansão |
| `maxInstances` | não | limita o número de datas devolvidas |

### Sucesso esperado

- `200 OK`

```json
{
  "valid": true,
  "occurrences": ["2026-11-02", "2026-11-04", "2026-11-09", "2026-11-11", "2026-11-16"],
  "truncated": true
}
```

- uma RRULE inválida devolve `valid = false` e `occurrences` vazio — continua a ser `200`
- frequência `SECONDLY` com intervalo > 1 dia, ou `MINUTELY` com intervalo > 7 dias, também dá `valid = false`
- `truncated = true` quando havia mais ocorrências que `maxInstances`

## 8.6 Fluxo recomendado

```
POST /utilities/validate/schedule  ──valid + validationHash──►  POST /schedules
                                                                    │
                                                    (automático ou generate-sync/async)
                                                                    ▼
POST /utilities/validate/slot-publication ◄────────────────────── slots
POST /utilities/validate/slot-availability  ─┐
POST /utilities/validate/appointment        ─┴──────────────────►  POST /appointments
```

## 8.7 Enumerações

- Endpoints: `GET /api/v1/scheduling/utilities/enums/<nome>`

| Rota | Enumeração | Resposta |
|---|---|---|
| `type-of-recurrence` | `TypeOfRecurrence` | `{ "values": ["NONE", ...] }` |
| `type-of-schedule` | `TypeOfSchedule` | idem |
| `category-of-schedule` | `CategoryOfSchedule` | idem |
| `type-of-service` | `TypeOfService` | idem |
| `week-days` | `WeekDay` | idem |
| `appointment-status` | `AppointmentStatus` | idem |
| `forwarding-status` | `ForwardingStatus` | idem |
| `forwarding-destination` | `ForwardingDestination` | idem |
| `priority` | `Priority` | idem |
| `gender-restrictions` | `GenderRestrictions` | idem |
| `gender` | `Gender` | idem |
| `operation-mode` | `OperationMode` | idem |
| `slot-visibility` | `SlotVisibility` | idem |
| `schedule-warning-codes` | `ScheduleWarningCode` | `{ "values": [{ "name": "...", "isBlocking": true }] }` |
| `slot-publication-warning-codes` | `SlotPublicationWarningCode` | idem |
| `slot-availability-warning-codes` | `SlotAvailabilityWarningCode` | idem |
| `schedule-blocking-warnings` | só os bloqueantes | `{ "values": [{ "name": "...", "value": 0 }] }` |
| `slot-publication-blocking-warnings` | só os bloqueantes | idem |
| `slot-availability-blocking-warnings` | só os bloqueantes | idem |

Todas devolvem `200 OK`.

---

## Navegação

← [Horários de Trabalho](./working-hours.md) · [Índice do módulo](../index.md) · [Casos Internos, Eventos e Notas](./internal-and-events.md) →

**Neste módulo:** [Visão Geral](./overview.md) · [Modelos e Enumerações](./models-and-enums.md) · [Agendas](./schedules.md) · [Slots](./slots.md) · [Marcações](./appointments.md) · [Reagendamentos](./reschedules.md) · [Encaminhamentos](./forwardings.md) · [Exclusões](./exclusions.md) · [Horários de Trabalho](./working-hours.md) · **Utilitários** · [Casos Internos, Eventos e Notas](./internal-and-events.md)
