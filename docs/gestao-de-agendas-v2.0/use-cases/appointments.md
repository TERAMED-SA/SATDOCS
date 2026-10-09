# Casos de Uso de Marcações

[⌂ Módulo Scheduling](../index.md) · [Visão Geral](./overview.md)

Grupo de rotas: `/api/v1/scheduling/appointments` · Requer autenticação (sem role específica).

Uma marcação ocupa uma vaga de um slot para um paciente e é atribuída a um profissional. O diagrama de estados está em [Modelos e Enumerações](./models-and-enums.md#estados-de-uma-marcação).

## 3.1 Criar marcação

- Caso de uso: `CreateAppointmentCommand`
- Endpoint: `POST /api/v1/scheduling/appointments`
- Modelos principais: `Appointment`, `Slot`

Para verificar uma marcação sem a gravar, use primeiro [`/utilities/validate/appointment`](./utilities.md#84-pré-validar-marcação) ou [`/utilities/validate/slot-availability`](./utilities.md#82-validar-disponibilidade-de-slot).

### Regras de negócio, por ordem de validação

Validação do pedido (`400 ValidationFailed`):

1. `slotId`, `phoneNumber` e `birthDate` obrigatórios
2. `patientTaxId` obrigatório, até 50 caracteres e um GUID válido
3. `gender` válido

Regras de negócio:

4. o slot existe (`SlotNotFound`)
5. vaga:
   - slot `CAPACITY`: tem de ter vagas restantes (`409 SlotCapacityReached`)
   - slot `TIMED`: `hour` é obrigatória (`400 SelectedHourRequired`), tem de existir na grelha (`404 SlotHourUnavailable`) e estar `AVAILABLE` (`400 SlotHourFullyBooked`)
6. o slot não está fechado e a agenda não está apagada (`404 SlotUnavailable`)
7. o hospital existe (`404 HospitalNotFound`) e está ativo (`422 HospitalInactive`)
8. o item clínico existe (`404 ClinicalItemNotFound`) e está ativo (`422 ClinicalItemInactive`); a especialidade está ativa (`422 InvalidSpecialty`) e é oferecida pelo hospital (`422 HospitalDoesNotHaveSpecialty`)
9. o hospital oferece o item (`422 HospitalCannotProvideClinicalItem`)
10. a idade do paciente, em dias, está dentro de `minimumAgeInDays`–`maximumAgeInDays` (`409 PatientDoesNotMeetAgeRequirement`)
11. o género respeita `genderRestrictions` (`422 GenderNotPermited`)
12. **atribuição do profissional** — ver abaixo

### Atribuição do profissional

1. **candidatos**: os profissionais do slot se a agenda tiver `slotInheritProfessionals`; senão os da agenda. Se a lista do slot estiver vazia, usa-se a da agenda. Sem candidatos → `422 ProfessionalNotFound`
2. se o body indicar `professionalTaxId`, tem de ser candidato (`422 ProfessionalNotAllowedToThisSlot`) e passa a ser o único considerado
3. só ficam os candidatos com **horário de trabalho** ativo para o hospital, a especialidade (`categoryCode`), o tipo de serviço e o dia da semana do slot, válido na data, que inclua a `hour` pedida e sem ausências (dias ou intervalos) que a cubram
4. em `TIMED`, saem os que já têm marcação não cancelada à mesma data e hora; se o profissional pedido estiver nessa situação → `422 ProfessionalHasAppoitmentOnThisTime`
5. entre os restantes, escolhe-se o com **menos marcações não canceladas** (todas as datas); empate resolvido ao acaso
6. sem nenhum elegível → `422 ProfessionalUnavailable`

### Efeitos

- `date`, `category`, `typeOfService`, `typeOfAppointment`, `healthUnitTaxId` e `clinicalItemId` são copiados do slot/agenda
- `unitId`/`departmentId`/`sectorId` vêm do body ou, em falta, do slot
- em `TELEMEDICINE`, é gerado um `roomLink` (Jitsi)
- em `TIMED`: grava `hour` e `durationInMinutes`, consome uma vaga do horário e do slot
- em `CAPACITY`: `hour` fica `null`, consome uma vaga do slot
- estado inicial: `PENDING_FOR_PAYMENT` com `paymentRequired`; senão `PENDING` em `TIMED` com `appointmentStartPending`; senão `CONFIRMED`
- é publicado `AppointmentScheduledEvent`

### Body

```json
{
  "slotId": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f",
  "patientTaxId": "0f1e2d3c-4b5a-4968-8776-5a4b3c2d1e0f",
  "birthDate": "1990-05-14",
  "gender": "FEMALE",
  "phoneNumber": "+244923000000",
  "hour": "08:30",
  "professionalTaxId": null,
  "clinicalSafetyConfirmed": false,
  "unitId": null,
  "departmentId": null,
  "sectorId": null
}
```

| Campo | Obrigatório | Omissão |
|---|---|---|
| `slotId`, `patientTaxId`, `birthDate`, `gender`, `phoneNumber` | sim | — |
| `hour` | em slots `TIMED` | `null` |
| `professionalTaxId` | não | atribuição automática |
| `clinicalSafetyConfirmed` | não | `false` — **não é usado** |
| `unitId`, `departmentId`, `sectorId` | não | os do slot |

### Exemplo de chamada

```bash
curl -X POST http://localhost:5000/api/v1/scheduling/appointments \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <jwt>' \
  -d '{
    "slotId": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f",
    "patientTaxId": "0f1e2d3c-4b5a-4968-8776-5a4b3c2d1e0f",
    "birthDate": "1990-05-14",
    "gender": "FEMALE",
    "phoneNumber": "+244923000000",
    "hour": "08:30"
  }'
```

### Sucesso esperado

- `201 Created`, com `Location` a apontar para `GET /api/v1/scheduling/appointments/{id}`
- corpo: o id da marcação, como string JSON

```json
"e5f6a7b8-c9d0-4e1f-8a2b-3c4d5e6f7a8b"
```

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `400` | `ValidationFailed` | pedido inválido |
| `404` | `SlotNotFound` | slot inexistente |
| `409` | `SlotCapacityReached` | slot `CAPACITY` sem vagas |
| `400` | `SelectedHourRequired` | slot `TIMED` sem `hour` |
| `404` | `SlotHourUnavailable` | `hour` fora da grelha |
| `400` | `SlotHourFullyBooked` | horário sem vagas ou bloqueado |
| `404` | `SlotUnavailable` | slot fechado ou agenda apagada |
| `404` / `422` | `HospitalNotFound` / `HospitalInactive` | hospital |
| `404` / `422` | `ClinicalItemNotFound` / `ClinicalItemInactive` | item clínico |
| `422` | `InvalidSpecialty`, `HospitalDoesNotHaveSpecialty`, `HospitalCannotProvideClinicalItem` | oferta do hospital |
| `409` | `PatientDoesNotMeetAgeRequirement` | idade fora dos limites |
| `422` | `GenderNotPermited` | género não permitido |
| `422` | `ProfessionalNotFound` | agenda e slot sem profissionais |
| `422` | `ProfessionalNotAllowedToThisSlot` | profissional pedido não é candidato |
| `422` | `ProfessionalHasAppoitmentOnThisTime` | profissional pedido já ocupado nessa hora |
| `422` | `ProfessionalUnavailable` | nenhum candidato com horário de trabalho compatível |

## 3.2 Listar marcações

- Caso de uso: `ListAppointmentsQuery`
- Endpoints (todos aceitam os mesmos filtros):
  - `GET /api/v1/scheduling/appointments`
  - `GET /api/v1/scheduling/appointments/patient/{patientTaxId}`
  - `GET /api/v1/scheduling/appointments/professional/{professionalTaxId}`
  - `GET /api/v1/scheduling/slots/{slotId}/appointments`
  - `GET /api/v1/scheduling/schedules/{scheduleId}/appointments`

### Query params

| Grupo | Parâmetros |
|---|---|
| Identificação | `id`, `slotId`, `scheduleId`, `patientTaxId`, `patientTaxIds`, `professionalTaxId`, `professionalTaxIds`, `hasProfessional`, `paymentId`, `hasPayment` |
| Data e hora | `date`, `hourFrom`, `hourTo`, `today`, `tomorrow`, `thisWeek`, `nextWeek`, `past`, `future`, `upcoming`, `timeOfDay` |
| Estado | `status`, `statusIn`, `isActive`, `isCancelled`, `isCompleted`, `cancelledByTaxId`, `hasCancellationInfo` |
| Serviço | `typeOfService`, `typeOfServiceIn`, `category`, `categoryCode`, `typeOfSchedule`, `hasRoomLink`, `hasNotes` |
| Hospital | `healthUnitTaxId`, `unitId`, `departmentId`, `sectorId` |
| Slot | `slotSpecificDate`, `slotWeekDay`, `slotIsClosed`, `slotInheritAllProfessionals` |
| Encaminhamento | `hasForwarding`, `forwardingStatus`, `forwardingDestination`, `forwardingPriority` |
| Reagendamento | `hasReschedule`, `rescheduleStatus`, `wasRescheduled`, `rescheduleRequestedBy` |
| Registo | `createdAtFrom/To`, `updatedAtFrom/To`, `statusChangedAtFrom/To`, `cancelledAtFrom/To` |
| Ordenação | `sortBy`: `hour` \| `createdAt` (qualquer outro valor, incluindo a omissão `selectedHour`, ordena por `createdAt`) · `sortOrder`: omissão `desc` |
| Paginação e âmbito | `page`, `limit`, `unitIds`, `departmentIds`, `sectorIds` |

### Sucesso esperado

- `200 OK` com `PaginatedList<AppointmentDto>` — ver a forma em [3.3](#33-obter-marcação-por-id)

## 3.3 Obter marcação por id

- Caso de uso: `GetAppointmentByIdQuery`
- Endpoint: `GET /api/v1/scheduling/appointments/{id}`

### Sucesso esperado

- `200 OK` com `AppointmentDto`

```json
{
  "id": "e5f6a7b8-c9d0-4e1f-8a2b-3c4d5e6f7a8b",
  "age": { "years": 36, "months": 4, "days": 25, "isBaby": false },
  "durationInMinutes": 30,
  "status": "CONFIRMED",
  "typeOfService": "IN_PERSON",
  "typeOfAppointment": "INITIAL",
  "gender": "FEMALE",
  "roomLink": null,
  "notes": null,
  "isCancelled": false,
  "isActive": true,
  "isCompleted": false,
  "hasForwarding": false,
  "hasReschedule": false,
  "isReschedule": false,
  "unitId": null,
  "departmentId": null,
  "sectorId": null,
  "rowVersion": "AAAAAAAAB9E=",
  "date": "2026-11-02",
  "weekDay": "MONDAY",
  "hour": "08:30:00"
}
```

- `age` é calculada a partir de `birthDate`
- `rowVersion` é o valor a enviar em `If-Match` para [trocar o profissional](#36-trocar-o-profissional)
- a resposta **não inclui** `patientTaxId`, `professionalTaxId`, `slotId` nem `phoneNumber` — para os obter filtre a listagem por esses campos

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `AppointmentNotFound` | marcação inexistente |

## 3.4 Transições de estado

- Casos de uso: `ConfirmAppointmentCommand`, `CheckInAppointmentCommand`, `StartSessionAppointmentCommand`, `CompleteAppointmentCommand`, `NoShowAppointmentCommand`, `ExpireAppointmentCommand`
- Endpoints: `PATCH /api/v1/scheduling/appointments/{id}/<ação>` (sem body)

| Ação | Estado exigido | Novo estado | Erro se o estado não servir |
|---|---|---|---|
| `confirm` | `PENDING` | `CONFIRMED` | `422 AppointmentMustBePending` |
| `check-in` | `CONFIRMED` | `CHECKED_IN` | `422 AppointmentConfirmed` |
| `start-session` | `CHECKED_IN` | `IN_SESSION` | `422 AppointmentMustBeCheckInd` |
| `complete` | `IN_SESSION` | `COMPLETED` (e `isCompleted = true`) | `422 AppointmentMustBeInSession` |
| `no-show` | `CONFIRMED` ou `CHECKED_IN` | `NO_SHOW` | `422 AppointmentMustBeCheckedInOrConfirmed` |
| `expire` | `PENDING`, com data/hora já passada | `EXPIRED` | `409 AppointmentMustBePending`; `422 AppointmentDateHasNtPassedYet` |

Regras comuns:

- todas atualizam `statusChangedAt`
- `confirm` não verifica o pagamento: uma marcação `PENDING_FOR_PAYMENT` não pode ser confirmada por esta rota
- `expire`: uma marcação sem hora só é considerada passada a partir do dia seguinte
- nenhuma destas transições liberta a vaga do slot — só o [cancelamento](#35-cancelar-marcação) o faz

### Sucesso esperado

- `204 No Content`

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `AppointmentNotFound` | marcação inexistente |
| `409` / `422` | ver tabela | estado não permite a transição |

## 3.5 Cancelar marcação

- Caso de uso: `CancelAppointmentCommand`
- Endpoints (sem body):
  - `PATCH /api/v1/scheduling/appointments/{id}/cancel-by-patient` → `cancelledBy = PATIENT`
  - `PATCH /api/v1/scheduling/appointments/{id}/cancel-by-professional` → `cancelledBy = PROFESSIONAL`
  - `PATCH /api/v1/scheduling/appointments/{id}/cancel-by-system` → `cancelledBy = SYSTEM`

### Regras de negócio, por ordem de validação

1. a marcação existe (`404 AppointmentNotFound`)
2. não está já cancelada (`409 AppointmentAlreadyCancelled`)
3. não está `COMPLETED`, `NO_SHOW` nem `EXPIRED` (`422 AppointmentCannotBeCancelled`)
4. o slot existe (`404 SlotNotFound`) e a agenda tem configurações (`404 ScheduleNotFound`)
5. ainda está dentro do prazo: a data/hora da marcação menos `deadlineForSlotBookingInHours` (marcações sem hora contam a partir das 00:00 do dia) — `422 DeadlineToCancelAppointmentHasAlreadyPassed`

### Efeitos

- `status = CANCELLED`, `cancelledAt` e `cancelledBy` preenchidos; `cancelledByTaxId` **não** é preenchido por esta rota
- a vaga é devolvida: ao horário (que volta a `AVAILABLE`) e ao slot, que reabre se estava fechado
- é publicado `AppointmentCancelledEvent`

### Sucesso esperado

- `204 No Content`

## 3.6 Trocar o profissional

- Caso de uso: `ChangeProfessionalCommand`
- Endpoint: `PUT /api/v1/scheduling/appointments/{id}`

### Headers

```http
If-Match: "AAAAAAAAB9E="
```

Opcional. Quando presente, é o `rowVersion` (base64) lido em [3.3](#33-obter-marcação-por-id); se a marcação tiver sido alterada entretanto, o pedido falha com `409`.

### Body

```json
{
  "id": "e5f6a7b8-c9d0-4e1f-8a2b-3c4d5e6f7a8b",
  "professional": { "professionalTaxId": "6b7c8d9e-0f1a-4b2c-9d3e-4f5a6b7c8d9e" }
}
```

### Regras de negócio

1. `id` do body igual ao da rota (senão `400` sem corpo)
2. `professional.professionalTaxId` obrigatório (`400 ValidationFailed`)
3. a marcação existe (`404 AppointmentNotFound`)
4. `If-Match`, se enviado, coincide com o `rowVersion` atual (`409 AppointmentAlreadyModified`)
5. o slot existe (`404 SlotNotFound`) e a agenda tem configurações (`404 ScheduleDoesNotHaveConfigurations`)
6. o novo profissional é candidato do slot (`422 ProfessionalNotAllowedToThisSlot`)

Não é verificado o horário de trabalho nem conflitos de hora do novo profissional, nem o estado da marcação.

### Sucesso esperado

- `204 No Content`; é publicado `AppointmentUpdatedEvent`

---

## Navegação

← [Slots](./slots.md) · [Índice do módulo](../index.md) · [Reagendamentos](./reschedules.md) →

**Neste módulo:** [Visão Geral](./overview.md) · [Modelos e Enumerações](./models-and-enums.md) · [Agendas](./schedules.md) · [Slots](./slots.md) · **Marcações** · [Reagendamentos](./reschedules.md) · [Encaminhamentos](./forwardings.md) · [Exclusões](./exclusions.md) · [Horários de Trabalho](./working-hours.md) · [Utilitários](./utilities.md) · [Casos Internos, Eventos e Notas](./internal-and-events.md)
