# Modelos e Enumerações — Scheduling

[⌂ Módulo Scheduling](../index.md) · [Visão Geral](./overview.md)

## Modelos de Domínio

As entidades são classes anémicas que herdam de `Entity`: as regras vivem em classes estáticas de `Domain/Logic` (`ScheduleLogic`, `SlotLogic`, `AppointmentLogic`, `RecurrenceLogic`, `RruleLogic`, `ProfessionalAssignmentLogic`) e nos handlers.

### Campos comuns (`Entity`)

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | Gerado na criação |
| `CreatedAt` / `UpdatedAt` | `DateTime` | UTC |
| `DeletedAt` | `DateTime?` | Preenchido no soft delete |
| `CreatedBy` / `UpdatedBy` / `DeletedBy` | `string?` | **Não preenchidos** pelos handlers atuais |
| `RowVersion` | `byte[]` | Token de concorrência otimista, renovado em cada gravação |

### Schedule

Agenda de atendimento de um item clínico num hospital.

| Campo | Tipo | Notas |
|---|---|---|
| `HealthUnitTaxId` | `string` | Id do hospital (GUID em texto) |
| `ClinicalItemId` | `Guid` | Item do [Catálogo Clínico (v2.0)](../../catalogo-clinico-v2.0/index.md) |
| `WeekDays` | `List<WeekDay>` | Dias de atendimento; obrigatório se houver recorrência |
| `StartTime` / `EndTime` | `TimeOnly?` | Janela horária; ambos presentes ⇒ slots `TIMED` |
| `TypeOfRecurrence` | `TypeOfRecurrence` | `DAILY` e `CUSTOM` não são aceites na criação |
| `TypeOfSchedule` | `TypeOfSchedule` | |
| `ScheduleContext` | `ScheduleContext` | |
| `Category` | `CategoryOfSchedule` | |
| `CategoryCode` | `string` | Código da especialidade; usado para casar com `WorkingHour.SpecialityCode` |
| `TypeOfService` | `TypeOfService` | |
| `StartDate` / `EndDate` | `DateOnly` / `DateOnly?` | `EndDate` nulo = agenda sem fim |
| `UnitId` / `DepartmentId` / `SectorId` | `Guid?` | Âmbito organizacional, herdado pelos slots |

### ScheduleConfigurations

Uma por agenda (`ScheduleId`). Agrupa as regras de geração, capacidade e marcação.

| Campo | Tipo | Omissão | Notas |
|---|---|---|---|
| `AppointmentDurationInMinutes` | `int?` | — | Com `StartTime`/`EndTime` ⇒ modo `TIMED` |
| `CapacityPerInterval` | `int?` | — | Vagas por horário no modo `TIMED` |
| `MaxAppointmentsPerSlot` | `int?` | — | Vagas por dia no modo `CAPACITY` |
| `UnlimitedSlotCapacity` | `bool` | `false` | |
| `AutomaticallyGenerateSlots` | `bool` | `false` no body | Agenda o job de geração na criação |
| `RruleDateToGenerate` | `string?` | — | RRULE que dita quando gerar; sem ela usa-se cron pela recorrência |
| `TimesItGenerated` | `int` | `0` | Quantos períodos já foram gerados |
| `AvailableProfessionalTaxIds` | `List<string>` | `[]` | Profissionais da agenda |
| `SlotInheritProfessionals` | `bool` | `false` no body | Slots herdam os profissionais com horário compatível |
| `AutoAssignProfessionals` | `bool` | `true` | |
| `AutoPublishSlots` | `bool` | `false` | Slots nascem `PUBLISHED` em vez de `DRAFT` |
| `RequireSupervisorApproval` | `bool` | `false` no body | |
| `AppointmentStartPending` | `bool` | `false` | Marcações `TIMED` nascem `PENDING` |
| `RescheduleStartPending` | `bool` | `false` | Reagendamentos ficam à espera de aprovação |
| `ForwardingStartPending` | `bool` | `false` | Sem efeito na implementação atual |
| `GenderRestrictions` | `GenderRestrictions` | `NONE` | |
| `MinimumAgeInDays` / `MaximumAgeInDays` | `int?` | — | Idade do paciente em dias |
| `PaymentRequired` | `bool` | `false` | Marcações nascem `PENDING_FOR_PAYMENT` |
| `ProofIsRequired` | `bool` | `false` | Sem efeito na implementação atual |
| `DeadlineForSlotBookingInHours` | `int?` | — | Antecedência mínima; usada também como prazo de cancelamento |
| `AdvanceCancellationInHours` | `int?` | — | Guardado mas **não usado** |
| `UseLocation` | `bool` | `false` | Exige `LocationMetadata` |
| `LocationMetadata` | `List<LocationMetadata>?` | — | Locais de atendimento |
| `DefaultPublishBufferHours` / `DefaultReviewTTLHours` | `int?` | — | Usados apenas nos avisos de validação |

"`false` no body" indica que a entidade tem outro valor por omissão, mas o DTO de criação (`ScheduleConfigurationsDto`) envia `false` quando o campo é omitido.

### LocationMetadata

Local de atendimento dentro do hospital. Pelo menos um nível tem de estar preenchido.

| Campo | Tipo |
|---|---|
| `UnitId`, `DepartmentId`, `BuildingId`, `FloorId`, `SectorId`, `RoomId`, `BedId` | `Guid?` |
| `IsActive` | `bool` (omissão `true`) |

Dois locais são "o mesmo" quando os sete níveis coincidem (`IsSamePlaceAs`).

### Slot

Um dia de atendimento de uma agenda. A geração reaproveita o slot existente para o mesmo `(ScheduleId, Date)` em vez de criar outro; não há índice único na base de dados a garanti-lo.

| Campo | Tipo | Notas |
|---|---|---|
| `ScheduleId` | `Guid` | |
| `Date` / `WeekDay` | `DateOnly` / `WeekDay` | |
| `Mode` | `OperationMode?` | `TIMED` se a agenda tem duração e janela horária; senão `CAPACITY` |
| `Hours` | `List<TimeBox>?` | Grelha horária, só em `TIMED` |
| `MarkingLimit` | `int` | **Vagas restantes**: diminui a cada marcação e aumenta a cada cancelamento. Em `TIMED` é a soma das vagas dos horários |
| `IsClosed` | `bool` | Fecha automaticamente quando as vagas chegam a zero (ou, em `TIMED`, quando nenhum horário fica `AVAILABLE`); reabre ao libertar uma vaga |
| `AvailableProfessionalTaxIds` | `List<string>` | Profissionais autorizados no slot |
| `Visibility` | `SlotVisibility` | |
| `ReviewedByTaxId`, `ReviewedAt`, `ReviewNotes` | | Revisão (sem endpoint que os altere) |
| `PublishAt`, `PublishNotBefore`, `PublishExpiresAt` | `DateTime?` | Publicação (sem endpoint que os altere) |
| `UnitId` / `DepartmentId` / `SectorId` | `Guid?` | Herdados da agenda |

### TimeBox

Um horário dentro de um slot `TIMED`.

| Campo | Tipo | Notas |
|---|---|---|
| `TimeFrom` / `TimeTo` | `TimeOnly` | |
| `MarkingLimit` | `int` | Vagas restantes neste horário (inicia em `CapacityPerInterval`) |
| `Status` | `TimeBoxStatus` | `AVAILABLE`, `BLOCKED`, `UNAVAILABLE` (sem vagas) |

### Appointment

Marcação de um paciente num slot.

| Campo | Tipo | Notas |
|---|---|---|
| `SlotId` | `Guid` | |
| `HealthUnitTaxId`, `ClinicalItemId` | | Copiados da agenda |
| `PatientTaxId` | `string` | GUID em texto |
| `BirthDate`, `Gender`, `PhoneNumber` | | Dados do paciente no momento da marcação |
| `ProfessionalTaxId` | `string?` | Profissional atribuído |
| `Date` / `Hour` | `DateOnly` / `TimeOnly?` | `Hour` só em slots `TIMED` |
| `DurationInMinutes` | `int?` | Da agenda, só em `TIMED` |
| `Status` | `AppointmentStatus` | |
| `StatusChangedAt` | `DateTime` | |
| `Category`, `TypeOfService`, `TypeOfAppointment` | | Copiados da agenda |
| `RoomLink` | `string?` | Link Jitsi, só em `TELEMEDICINE` |
| `Notes`, `PaymentId` | | Sem endpoint que os preencha |
| `CancelledAt`, `CancelledBy`, `CancelledByTaxId` | | Preenchidos no cancelamento |
| `IsCompleted` | `bool` | |
| `HasForwarding`, `ForwardedAt` | | |
| `HasReschedule`, `IsReschedule` | `bool` | |
| `UnitId` / `DepartmentId` / `SectorId` | `Guid?` | Do body ou, em falta, do slot |

### Reschedule

Pedido de mudança de uma marcação para outro slot/hora.

| Campo | Tipo |
|---|---|
| `AppointmentId` | `Guid` |
| `OldSlotId` / `OldSelectedHour` | `Guid` / `TimeOnly?` |
| `NewSlotId` / `NewSelectedHour` | `Guid` / `TimeOnly?` |
| `Reason` | `string?` (motivos de rejeição/cancelamento são acrescentados aqui) |
| `RequestedBy` / `RequestedByTaxId` | `RequestedBy` / `string` |
| `RequestedAt` / `ApprovedAt` | `DateTime` / `DateTime?` |
| `Status` | `RescheduleStatus` |

### Forwarding

Encaminhamento de uma marcação concluída para outra entidade ou serviço.

| Campo | Tipo |
|---|---|
| `AppointmentId` | `Guid` |
| `Destination` | `ForwardingDestination` |
| `MedicalAreaId`, `SpecialityId`, `ReceiverTaxId` | `string?` |
| `SenderTaxId` | `string` |
| `Reason` | `string?` |
| `Priority` | `Priority` |
| `Status` | `ForwardingStatus` |
| `ForwardedAt` | `DateTime` |
| `Obs` | `string?` (motivos de rejeição/cancelamento são acrescentados aqui) |

### ExcludeDay

Dia sem atendimento.

| Campo | Tipo | Notas |
|---|---|---|
| `Title` | `string` | |
| `Reason` | `string?` | |
| `SpecificDate` | `DateOnly?` | Data concreta |
| `WeekDays` | `List<WeekDay>` | Dias da semana recorrentes |
| `TypeOfRecurrence`, `Rrule` | | Recorrência |
| `Kind` | `ExclusionKind` | `SCHEDULE` (criado em `/exclude-days`) ou `WORKING` (ausência de um horário de trabalho) |
| `Visibility` | `ExclusionVisibility` | Âmbito — ver [Exclusões](./exclusions.md#âmbito-de-uma-exclusão) |
| `IsActive` | `bool` | |
| `HealthUnitTaxId`, `MedicalAreaId`, `SpecialityCode`, `DefinedBy` | `string?` | |

### ExcludeRange

Intervalo horário sem atendimento.

| Campo | Tipo |
|---|---|
| `StartTime` / `EndTime` | `TimeOnly` |
| `StartDate` / `EndDate` | `DateOnly?` |
| `SpecificDates` | `List<DateOnly>` |
| `WeekDays` | `List<WeekDay>` |
| `TypeOfRecurrence`, `Rrule` | |
| `Kind`, `Visibility`, `IsActive` | como em `ExcludeDay` |
| `Title`, `Reason`, `HealthUnitTaxId`, `MedicalAreaId`, `SpecialityCode`, `DefinedBy` | `string?` |

### WorkingHour

Horário de trabalho de um profissional num hospital, para uma especialidade e dia da semana.

| Campo | Tipo |
|---|---|
| `ProfessionalTaxId`, `HealthUnitTaxId`, `SpecialityCode` | `string` |
| `WeekDay` | `WeekDay` |
| `StartAt` / `EndsAt` | `TimeOnly` |
| `ValidFrom` / `ValidTo` | `DateOnly?` |
| `TypeOfService` | `TypeOfService` |
| `IsActive` | `bool` |

### Tabelas de associação

| Entidade | Liga |
|---|---|
| `ScheduleExcludedDay` | `Schedule` ↔ `ExcludeDay` |
| `ScheduleExcludedRange` | `Schedule` ↔ `ExcludeRange` (com `IncludeForAllUnitSchedules`) |
| `WorkingHourExcludedDay` | `WorkingHour` ↔ `ExcludeDay` |
| `WorkingHourExcludedRange` | `WorkingHour` ↔ `ExcludeRange` |

## Enumerações

Todas as enumerações viajam em JSON **pelo nome** (`JsonStringEnumConverter`) e são persistidas como texto.

| Enumeração | Valores |
|---|---|
| `TypeOfRecurrence` | `NONE`, `DAILY`, `WEEKLY`, `MONTHLY`, `QUARTERLY`, `SEMIANNUALLY`, `ANNUALLY`, `CUSTOM` |
| `TypeOfSchedule` | `INITIAL`, `FOLLOW_UP` |
| `ScheduleContext` | `PUBLIC`, `INTERNAL` |
| `CategoryOfSchedule` | `CLINIC`, `SPECIALITY`, `VACCINE`, `EXAM`, `SURGERY`, `THERAPY`, `NURSING` |
| `TypeOfService` | `IN_PERSON`, `TELEMEDICINE`, `HOME` |
| `WeekDay` | `SUNDAY`, `MONDAY`, `TUESDAY`, `WEDNESDAY`, `THURSDAY`, `FRIDAY`, `SATURDAY` |
| `OperationMode` | `TIMED`, `CAPACITY`, `UNLIMITED` |
| `SlotVisibility` | `DRAFT`, `PUBLISHED`, `REJECTED`, `ARCHIVED` |
| `TimeBoxStatus` | `AVAILABLE`, `BLOCKED`, `UNAVAILABLE` |
| `AppointmentStatus` | `PENDING`, `RESCHEDULED`, `REJECTED`, `FORWARDED`, `PENDING_FOR_PAYMENT`, `PAYMENT_EXPIRED`, `CONFIRMED`, `CHECKED_IN`, `IN_SESSION`, `COMPLETED`, `CANCELLED`, `NO_SHOW`, `FOLLOW_UP_REQUIRED`, `DOCUMENT_PENDING`, `EXPIRED` |
| `AppointmentCancelledBy` | `PATIENT`, `PROFESSIONAL`, `SYSTEM` |
| `Gender` | `MALE`, `FEMALE` |
| `GenderRestrictions` | `NONE`, `MALE_ONLY`, `FEMALE_ONLY` |
| `RescheduleStatus` | `REJECTED`, `CANCELLED`, `APPROVED`, `PENDING` |
| `RequestedBy` | `PROFESSIONAL`, `PATIENT` |
| `ForwardingStatus` | `PENDING`, `SENT`, `RECEIVED`, `ACCEPTED`, `REJECTED`, `COMPLETED`, `CANCELLED` |
| `ForwardingDestination` | `EXTERNAL` (entidade externa), `INTERNAL` (outro serviço do hospital) |
| `Priority` | `NORMAL`, `URGENT`, `SERIOUS` |
| `ExclusionKind` | `SCHEDULE`, `WORKING` |
| `ExclusionVisibility` | `SYSTEM`, `HEALTH_UNIT`, `MEDICAL_AREA`, `SPECIALITY`, `PARTICULAR` |

Os valores atuais de cada enumeração podem ser consultados em `GET /scheduling/utilities/enums/...` — ver [Utilitários](./utilities.md#87-enumerações).

### Estados de uma marcação

Só uma parte de `AppointmentStatus` é usada pelos fluxos atuais:

```
                ┌──────────── cancel-* ────────────┐
                │                                  ▼
PENDING_FOR_PAYMENT          PENDING ──confirm──► CONFIRMED ──check-in──► CHECKED_IN ──start-session──► IN_SESSION ──complete──► COMPLETED
                               │                    │                         │                                                  │
                            expire                no-show ◄───────────────────┘                                          forwarding
                               ▼                    ▼                                                                           ▼
                            EXPIRED              NO_SHOW                                                                    FORWARDED
```

- `CANCELLED`: a partir de qualquer estado exceto `COMPLETED`, `NO_SHOW`, `EXPIRED` e o próprio `CANCELLED`
- `REJECTED`: quando um reagendamento pendente é rejeitado
- `RESCHEDULED`, `PAYMENT_EXPIRED`, `FOLLOW_UP_REQUIRED`, `DOCUMENT_PENDING`: existem na enumeração mas **nenhum fluxo os atribui**

---

## Navegação

← [Visão Geral](./overview.md) · [Índice do módulo](../index.md) · [Agendas](./schedules.md) →

**Neste módulo:** [Visão Geral](./overview.md) · **Modelos e Enumerações** · [Agendas](./schedules.md) · [Slots](./slots.md) · [Marcações](./appointments.md) · [Reagendamentos](./reschedules.md) · [Encaminhamentos](./forwardings.md) · [Exclusões](./exclusions.md) · [Horários de Trabalho](./working-hours.md) · [Utilitários](./utilities.md) · [Casos Internos, Eventos e Notas](./internal-and-events.md)
