# Casos de Uso de Exclusões

[⌂ Módulo Scheduling](../index.md) · [Visão Geral](./overview.md)

Grupos de rotas: `/api/v1/scheduling/exclude-days` e `/api/v1/scheduling/exclude-ranges` · Requer autenticação (sem role específica).

Uma exclusão retira disponibilidade:

- **dia excluído** (`ExcludeDay`): um dia inteiro — data concreta, dia da semana recorrente ou RRULE
- **intervalo excluído** (`ExcludeRange`): um intervalo horário (`startTime`–`endTime`), opcionalmente limitado a datas ou dias da semana

Este grupo gere exclusões de **agenda** (`Kind = SCHEDULE`). As ausências de um profissional (`Kind = WORKING`) são criadas em [Horários de Trabalho](./working-hours.md#73-ausências-do-profissional).

## Âmbito de uma exclusão

Uma exclusão de agenda alcança uma agenda de duas formas:

| Forma | Como | Onde conta |
|---|---|---|
| **Associação** | [`POST /schedules/{id}/exclude-days/{excludeDayId}`](./schedules.md#111-associar-exclusões) ou `excludeDayIds` na criação da agenda | geração de slots **e** validações |
| **Âmbito** (`visibility`) | `SYSTEM` → todas as agendas; `HEALTH_UNIT` → agendas com o mesmo `healthUnitTaxId`; `SPECIALITY` → agendas com o mesmo `categoryCode` | só nas validações de [disponibilidade](./utilities.md#82-validar-disponibilidade-de-slot) e [publicação](./utilities.md#83-validar-publicação-de-slot) |

Só exclusões ativas e não apagadas contam.

**Atenção:** a criação força `visibility = SYSTEM`, independentemente do que for enviado. Na prática, **toda a exclusão criada por estas rotas aplica-se às validações de todas as agendas da plataforma**, e só afeta a geração das agendas a que for associada.

## 6.1 Criar exclusão

- Casos de uso: `CreateExcludeDayCommand`, `CreateExcludeRangeCommand`
- Endpoints: `POST /api/v1/scheduling/exclude-days` · `POST /api/v1/scheduling/exclude-ranges`

### Body — dia excluído

```json
{
  "title": "Feriado nacional",
  "reason": "Dia da Independência",
  "specificDate": "2026-11-11",
  "weekDays": [],
  "typeOfRecurrence": null,
  "rrule": null,
  "exclusionVisibility": "SYSTEM",
  "healthUnitTaxId": null,
  "isActive": true
}
```

| Campo | Obrigatório | Notas |
|---|---|---|
| `title` | sim | |
| `exclusionVisibility` | sim | **ignorado**: gravado sempre como `SYSTEM` |
| `specificDate` | não | exclui esta data |
| `weekDays` | não | exclui estes dias da semana, todas as semanas |
| `rrule` | não | exclui as datas da RRULE |
| `typeOfRecurrence`, `reason`, `healthUnitTaxId`, `isActive` | não | |

Um dia é excluído se coincidir com **qualquer** um de `specificDate`, `weekDays` ou `rrule`.

### Body — intervalo excluído

```json
{
  "title": "Pausa de almoço",
  "reason": null,
  "startTime": "12:00",
  "endTime": "13:00",
  "startDate": "2026-11-01",
  "endDate": "2027-04-30",
  "weekDays": ["MONDAY", "TUESDAY", "WEDNESDAY", "THURSDAY", "FRIDAY"],
  "specificDates": [],
  "typeOfRecurrence": null,
  "rrule": null,
  "visibility": "SYSTEM",
  "healthUnitTaxId": null,
  "medicalAreaId": null,
  "specialityCode": null,
  "definedBy": null,
  "isActive": true
}
```

| Campo | Obrigatório | Notas |
|---|---|---|
| `startTime`, `endTime` | sim | |
| `visibility` | sim | **ignorado**: gravado sempre como `SYSTEM` |
| restantes | não | |

### Regras de negócio

- não há validação de conteúdo: um dia excluído sem data, dia da semana nem RRULE é aceite (e não exclui nada); um intervalo com `endTime ≤ startTime` também é aceite
- `kind` é sempre `SCHEDULE`
- a exclusão **não** é associada a nenhuma agenda — faça-o em [Agendas 1.11](./schedules.md#111-associar-exclusões)

### Sucesso esperado

- `201 Created`, `Location` para `GET .../exclude-days/{id}` ou `.../exclude-ranges/{id}`, **corpo vazio**

## 6.2 Listar exclusões

- Casos de uso: `ListExcludeDaysQuery`, `ListExcludeRangesQuery`
- Endpoints:
  - `GET /api/v1/scheduling/exclude-days` · `GET /api/v1/scheduling/exclude-ranges`
  - `GET /api/v1/scheduling/exclude-days/schedules/{scheduleId}/exclude-days` — só as **associadas** à agenda
  - `GET /api/v1/scheduling/exclude-ranges/schedules/{scheduleId}/exclude-ranges` — só as **associadas** à agenda

### Query params — dias excluídos

| Parâmetro | Tipo |
|---|---|
| `scheduleId` | guid — associadas à agenda |
| `healthUnitTaxId`, `reason`, `typeOfRecurrence`, `kind`, `rrule` | string |
| `isActive`, `hasRrule`, `hasSpecificDate`, `hasWeekDays` | bool |
| `specificDateFrom`, `specificDateTo`, `specificDateOn`, `specificDateBetweenFrom`, `specificDateBetweenTo` | datetime |
| `weekDays` | lista |
| `month`, `year`, `dayOfMonth` | int — sobre `specificDate` |

### Query params — intervalos excluídos

| Parâmetro | Tipo |
|---|---|
| `scheduleId` | guid — associados à agenda |
| `includeForAllUnitSchedules` | bool — só com `scheduleId` |
| `healthUnitTaxId`, `title`, `reason`, `typeOfRecurrence`, `kind`, `rrule` | string |
| `isActive`, `hasSpecificDates`, `hasRrule` | bool |
| `startDateFrom/To`, `endDateFrom/To`, `activeOnDate`, `activeBetweenFrom`, `activeBetweenTo` | datetime |
| `startTimeFrom/To`, `endTimeFrom/To` | time |

### Comuns

| Parâmetro | Notas |
|---|---|
| `search` | `title` ou `reason` contêm o termo (sem distinguir maiúsculas) |
| `createdAtFrom/To`, `updatedAtFrom/To` | |
| `includeDeleted` / `onlyDeleted` | por omissão as apagadas ficam de fora |
| `sortBy` | `title` \| `createdAt` (omissão) |
| `sortOrder` | `asc` \| `desc` |
| `page`, `limit`, `unitIds`, `departmentIds`, `sectorIds` | ver [Visão Geral](./overview.md#envelopes-de-paginação) |

### Sucesso esperado

- `200 OK` com `PaginatedList<ExcludeDayDto>` / `PaginatedList<ExcludeRangeDto>` — os campos do body de criação mais `id` e `kind` (e, nos intervalos, `includeForAllUnitSchedules`, `createdAt`, `updatedAt`, `deletedAt`)

## 6.3 Obter exclusão por id

- Casos de uso: `GetExcludeDayByIdQuery`, `GetExcludeRangeByIdQuery`
- Endpoints: `GET /api/v1/scheduling/exclude-days/{id}` · `GET /api/v1/scheduling/exclude-ranges/{id}`

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `ExcludeDayNotFound` / `ExcludeRangeNotFound` | inexistente |

## 6.4 Alterar, ativar/desativar e apagar

- Casos de uso: `UpdateExclude*Command`, `ToggleExclude*Command`, `DeleteExclude*Command`
- Endpoints (para `exclude-days` e `exclude-ranges`):
  - `PUT /{id}` — body `{ "title": "...", "reason": "..." }`
  - `PATCH /{id}/toggle` — sem body
  - `DELETE /{id}`

### Regras de negócio

- todas exigem a exclusão existente e não apagada (`404 ExcludeDayNotFound` / `ExcludeRangeNotFound`)
- **alterar** só muda `title` e `reason`: `title` nulo mantém o atual. Em dias excluídos, `reason` nulo **apaga** o motivo; em intervalos, mantém-no. Datas, horas e recorrência não podem ser alteradas — apague e crie outra
- **toggle** inverte `isActive`; uma exclusão inativa deixa de contar em todo o lado
- **apagar** é soft delete e deixa de contar nas validações. As associações às agendas **ficam**, e a geração de slots só verifica `isActive` — por isso uma exclusão apagada mas ainda associada **continua a excluir dias na geração**. Para a retirar de uma agenda, desative-a (`toggle`) ou desassocie-a antes de apagar
- um intervalo sem `weekDays`, `specificDates` nem `rrule` aplica-se a todos os dias entre `startDate` e `endDate`

### Sucesso esperado

- `204 No Content`

---

## Navegação

← [Encaminhamentos](./forwardings.md) · [Índice do módulo](../index.md) · [Horários de Trabalho](./working-hours.md) →

**Neste módulo:** [Visão Geral](./overview.md) · [Modelos e Enumerações](./models-and-enums.md) · [Agendas](./schedules.md) · [Slots](./slots.md) · [Marcações](./appointments.md) · [Reagendamentos](./reschedules.md) · [Encaminhamentos](./forwardings.md) · **Exclusões** · [Horários de Trabalho](./working-hours.md) · [Utilitários](./utilities.md) · [Casos Internos, Eventos e Notas](./internal-and-events.md)
