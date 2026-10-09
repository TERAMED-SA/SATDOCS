# Casos de Uso de Horários de Trabalho

[⌂ Módulo Scheduling](../index.md) · [Visão Geral](./overview.md)

Grupo de rotas: `/api/v1/scheduling/working-hours` · Requer autenticação (sem role específica).

Um horário de trabalho diz que um profissional atende num hospital, numa especialidade e num tipo de serviço, num dia da semana, entre duas horas. É a base para:

- a **herança de profissionais** pelos slots na geração (`slotInheritProfessionals`) — ver [Slots 2.1](./slots.md#21-gerar-slots)
- a **atribuição do profissional** numa marcação — ver [Marcações 3.1](./appointments.md#atribuição-do-profissional)
- o aviso `PROFESSIONALS_MISSING_WORKING_HOURS` na [validação de publicação](./utilities.md#83-validar-publicação-de-slot)

Um profissional sem horário de trabalho compatível **não recebe marcações**, mesmo que esteja associado à agenda.

## 7.1 Criar horário

- Caso de uso: `CreateWorkingHourCommand`
- Endpoint: `POST /api/v1/scheduling/working-hours`
- Modelo principal: `WorkingHour`

### Regras de negócio

1. `professionalTaxId`, `healthUnitTaxId` e `specialityCode` obrigatórios (`400 ValidationFailed`)
2. `weekDay` e `typeOfService` válidos
3. `startAt` obrigatório e `endsAt > startAt`
4. `validTo ≥ validFrom`, quando há `validFrom`
5. não pode existir outro horário com o mesmo profissional, hospital, especialidade, dia, início e fim (`409 UniqueConstraintViolation`)

O horário nasce ativo e é publicado `WorkingHourCreatedEvent`. Não se verifica se o profissional ou o hospital existem, nem sobreposições parciais com outros horários.

### Body

```json
{
  "professionalTaxId": "5a6b7c8d-9e0f-4a1b-8c2d-3e4f5a6b7c8d",
  "healthUnitTaxId": "3f2b6c1e-8a4d-4f5e-9b7c-1d2e3f4a5b6c",
  "specialityCode": "CARD",
  "weekDay": "MONDAY",
  "startAt": "08:00",
  "endsAt": "12:00",
  "validFrom": "2026-11-01",
  "validTo": null,
  "typeOfService": "IN_PERSON"
}
```

`specialityCode` tem de ser igual ao `categoryCode` das agendas em que o profissional vai atender.

### Sucesso esperado

- `201 Created`, `Location` para `GET /api/v1/scheduling/working-hours/{id}`, **corpo vazio**

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `400` | `ValidationFailed` | pedido inválido |
| `409` | `UniqueConstraintViolation` | horário duplicado |

## 7.2 Consultar, alterar e apagar

- Casos de uso: `GetAllWorkingHoursQuery`, `GetWorkingHourByIdQuery`, `UpdateWorkingHourCommand`, `DeleteWorkingHourCommand`
- Endpoints:
  - `GET /api/v1/scheduling/working-hours` — filtros `professionalTaxId`, `healthUnitTaxId`, `specialityCode`, mais `page`, `limit`, `unitIds`, `departmentIds`, `sectorIds`
  - `GET /api/v1/scheduling/working-hours/{id}`
  - `PUT /api/v1/scheduling/working-hours/{id}`
  - `DELETE /api/v1/scheduling/working-hours/{id}`

### Body (alterar)

Todos opcionais; só os enviados mudam:

```json
{
  "startAt": "09:00",
  "endsAt": "13:00",
  "validFrom": null,
  "validTo": "2027-06-30",
  "isActive": true,
  "typeOfService": "IN_PERSON"
}
```

### Regras de negócio

- o horário existe (`404 WorkingHourNotFound`)
- profissional, hospital, especialidade e dia da semana **não** podem ser alterados
- depois de aplicar as alterações: `endsAt > startAt` (`400 InvalidWorkingHourTime`) e `validTo ≥ validFrom` (`400 InvalidWorkingHourValidity`)
- enviar `null` mantém o valor atual — não é possível limpar `validFrom`/`validTo`
- apagar é **físico** e publica `WorkingHourDeletedEvent`

### Sucesso esperado

- `GET`: `200 OK` com `WorkingHourDto` (os campos do body de criação mais `id` e `isActive`); a listagem vem em `PaginatedList`
- `PUT` e `DELETE`: `204 No Content`

## 7.3 Ausências do profissional

- Casos de uso: `AddExcludeDaysCommand`, `AddExcludeRangesCommand`, `RemoveExcludeDaysCommand`, `RemoveExcludeRangesCommand`
- Endpoints:
  - `POST /api/v1/scheduling/working-hours/{workingHourId}/exclude-days` — body de um [dia excluído](./exclusions.md#61-criar-exclusão)
  - `POST /api/v1/scheduling/working-hours/{workingHourId}/exclude-ranges` — body de um [intervalo excluído](./exclusions.md#61-criar-exclusão)
  - `DELETE /api/v1/scheduling/working-hours/exclude-days/{id}`
  - `DELETE /api/v1/scheduling/working-hours/exclude-ranges/{id}`

### Regras de negócio

- o horário existe (`404 WorkingHourNotFound`)
- cria a exclusão com `kind = WORKING` e `visibility = PARTICULAR` e associa-a ao horário; por isso **não** afeta as agendas, só a elegibilidade do profissional
- no intervalo, `endTime > startTime` (`400 ValidationFailed`)
- uma ausência que cubra a data (ou a hora) de uma marcação retira o profissional da atribuição

### Sucesso esperado

- criar: `201 Created`, sem `Location`, corpo `{ "id": "<id da associação>" }`
- **o `id` devolvido é o da associação** (horário ↔ exclusão), não o da exclusão — é este id que se usa no `DELETE`
- remover: `200 OK` sem corpo; remove a associação, a exclusão em si continua gravada

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `WorkingHourNotFound` | horário inexistente |
| `404` | `ExcludeDayNotFound` / `ExcludeRangeNotFound` | associação inexistente, no `DELETE` |

---

## Navegação

← [Exclusões](./exclusions.md) · [Índice do módulo](../index.md) · [Utilitários](./utilities.md) →

**Neste módulo:** [Visão Geral](./overview.md) · [Modelos e Enumerações](./models-and-enums.md) · [Agendas](./schedules.md) · [Slots](./slots.md) · [Marcações](./appointments.md) · [Reagendamentos](./reschedules.md) · [Encaminhamentos](./forwardings.md) · [Exclusões](./exclusions.md) · **Horários de Trabalho** · [Utilitários](./utilities.md) · [Casos Internos, Eventos e Notas](./internal-and-events.md)
