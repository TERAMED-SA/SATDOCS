# Casos Internos, Eventos e Notas — Scheduling

[⌂ Módulo Scheduling](../index.md) · [Visão Geral](./overview.md)

## Casos de uso não expostos por HTTP

| Job (Hangfire) | Disparado por | O que faz |
|---|---|---|
| `RruleSlotGenerationJob` | criação de agenda com `automaticallyGenerateSlots`; ele próprio | gera o próximo período e agenda a execução seguinte |
| `TriggerNowSlotGenerationJob` | `POST /slots/.../generate-async` | gera o próximo período uma vez |
| `HardDeleteScheduleSlotsJob` | `DELETE /schedules/{id}` | apaga fisicamente os slots da agenda |
| `RecurrenceSlotGenerationJob` | — | registado no DI mas **nunca agendado** |

## Geração de slots em background

Ao criar uma agenda com `automaticallyGenerateSlots = true`, o `ScheduleJobManager` escolhe uma estratégia pelas configurações:

| Estratégia | Quando | Registo |
|---|---|---|
| `RruleScheduleJobStrategy` | `rruleDateToGenerate` preenchida | enfileira `RruleSlotGenerationJob` de imediato |
| `CronScheduleJobStrategy` | sem `rruleDateToGenerate` | recurring job `generate-slots-{scheduleId}` com cron pela recorrência (`WEEKLY` → domingo 00:00, `MONTHLY` → dia 1, `QUARTERLY`, `SEMIANNUALLY`, `ANNUALLY`) |

Em cada execução, o `RruleSlotGenerationJob`:

1. gera o próximo período (o mesmo que [`generate-sync`](./slots.md#21-gerar-slots)), se a agenda tiver `automaticallyGenerateSlots` **e** `rruleDateToGenerate`
2. calcula a próxima ocorrência da RRULE a partir de agora e agenda-se para essa data; sem RRULE, não agenda nada

### Concorrência

Várias gerações da mesma agenda podem correr ao mesmo tempo (o Hangfire tem 10 workers, e o mesmo período pode ser pedido por job, `generate-sync` e `generate-async`). O `GenerateSlotsCommandHandler` corre dentro de `IUnitOfWork.ExecuteWithLockAsync`, que abre uma transação e obtém `pg_advisory_xact_lock` sobre `scheduling:generate-slots:{scheduleId}`. A segunda geração espera pela primeira e lê o `timesItGenerated` já atualizado, em vez de falhar com `DbUpdateConcurrencyException`. O lock é do PostgreSQL, por isso funciona com várias instâncias da API.

### Configuração do Hangfire

- armazenamento: **SQL Server**, connection string `SQLServer`, schema `Hangfire`
- 10 workers, polling de 30 s
- tentativas automáticas por omissão do Hangfire (10) em caso de exceção

## Eventos

Definidos em `Contracts/V1/Events` como `INotification` do MediatR e publicados em processo com `IPublisher`:

| Evento | Publicado em | Payload |
|---|---|---|
| `ScheduleCreatedEvent` | criação de agenda | `ScheduleId`, `HealthUnitTaxId`, `ClinicalItemId`, `StartDate`, `EndDate` |
| `ScheduleDeletedEvent` | apagar agenda | `ScheduleId`, `DeletedAt` |
| `AppointmentScheduledEvent` | criação de marcação | `AppointmentId`, `PatientTaxId`, `HealthUnitTaxId`, `ClinicalItemId`, `Date`, `Status` |
| `AppointmentUpdatedEvent` | troca de profissional | `AppointmentId`, `ProfessionalTaxId`, `Status` |
| `AppointmentCancelledEvent` | cancelamento | `AppointmentId`, `Status`, `CancelledAt` |
| `RescheduleCreatedEvent` | pedido de reagendamento | `RescheduleId`, `AppointmentId`, `OldSlotId`, `NewSlotId`, `Status` |
| `ForwardingCreatedEvent` | criação de encaminhamento | `ForwardingId`, `AppointmentId`, `SenderTaxId`, `ReceiverTaxId`, `Status` |
| `WorkingHourCreatedEvent` / `WorkingHourDeletedEvent` | criar / apagar horário | `WorkingHourId`, … |

`RescheduleStatusChangedEvent` e `ForwardingStatusChangedEvent` estão definidos mas não são publicados.

**Nenhum módulo da solução subscreve estes eventos** (não há `INotificationHandler` para eles). Não há outbox: as tabelas `EventOutbox` e `IdempotentRequests` existem no `DbContext` mas não são usadas.

## Comunicação entre módulos

O Scheduling consome dois módulos através de portas definidas em `Contracts/V1/ExternalCatalogServices.cs` e implementadas por adaptadores ACL em `Infrastructure/Acl`:

| Porta | Adaptador | Módulo consumido | Usado para |
|---|---|---|---|
| `IHospitalCatalogService` | `HospitalCatalogService` | [Gestão Hospitalar](../../hospital/index.md) (`IHospitalLookupApi`, `IHospitalStructureApi`) | hospital ativo, ofertas de item e especialidade, níveis da estrutura física |
| `IClinicalCatalogService` | `ClinicalCatalogService` | [Catálogo Clínico (v2.0)](../../catalogo-clinico-v2.0/index.md) (`IClinicalCatalogApi`) | item clínico (especialidade, ativo) e especialidade ativa |

As chamadas são em processo; o Scheduling não expõe uma API pública para outros módulos.

## Persistência

- `DbContext`: `SchedulingDbContext`, schema PostgreSQL **`scheduling`**
- `DbSet`s: `Schedules`, `ScheduleConfigurations`, `Slots`, `Appointments`, `ExcludeDays`, `ExcludeRanges`, `WorkingHours`, `Forwardings`, `Reschedules`, `EventOutbox`, `IdempotentRequests` (mais as tabelas de associação)
- Unit of Work: o próprio `DbContext` implementa `IUnitOfWork`; renova o `RowVersion` de cada entidade alterada em `SaveChanges`
- Leituras de listagem: Dapper (`AdvancedQueryHelper`) com type handlers para `jsonb`, `DateOnly` e `TimeOnly`
- Enumerações gravadas como texto; listas (`WeekDays`, `Hours`, `AvailableProfessionalTaxIds`, `LocationMetadata`, ...) como `jsonb`

### Índices únicos

| Tabela | Índice |
|---|---|
| `ScheduleExcludedDays` | `(ScheduleId, ExcludeDayId)` |
| `ScheduleExcludedRanges` | `(ScheduleId, ExcludeRangeId)` |
| `WorkingHours` | `(ProfessionalTaxId, HealthUnitTaxId, SpecialityCode, WeekDay, StartAt, EndsAt)` |

Uma violação destes índices devolve `409 UniqueConstraintViolation`.

### Migrações

| Migração | Efeito |
|---|---|
| `20260827160118_InitialCreate` | criação do schema e das tabelas |
| `20260831155003_RemoveShadowTables` | remoção das tabelas sombra de hospitais/utilizadores |
| `20260922134642_AddAppointmentCancelledBy` | coluna `CancelledBy` em `Appointments` |

### Aplicar migrações

```bash
make migrate-scheduling
```

Equivalente a:

```bash
dotnet ef database update \
  --project modules/Scheduling/Infrastructure \
  --startup-project src/Sat.Api \
  --context SchedulingDbContext
```

## Composition root

`SchedulingModule` implementa `IModule` e é descoberto pelo `ModuleScanner`. Regista:

- Application: MediatR, validadores e behaviors
- Infrastructure: `DbContext` (Npgsql com retry), Dapper, repositórios, Hangfire, publishers de eventos, cache (`IMemoryCache` + `AddDistributedMemoryCache`), adaptadores ACL, cliente Jitsi
- `SchedulingExceptionHandler`: traduz `DomainException` e violações de índice único
- Endpoints: `Schedules`, `Appointments`, `Slots`, `Forwardings`, `Reschedules`, `ExcludeDays`, `ExcludeRanges`, `WorkingHours`, `Utilities`

## Cobertura de testes

Em `Tests/`, 128 testes, todos em `SchedulingRoutesTests`:

- cada rota esperada está registada com o verbo e o padrão certos
- não existem rotas a mais
- todas as rotas exigem autenticação

**Não há testes de domínio nem de handlers**: as regras descritas nesta documentação não estão cobertas por testes automáticos.

## Limitações conhecidas

Comportamentos da implementação atual a ter em conta, por ordem de impacto:

1. **A geração automática sem RRULE não gera slots.** O recurring job de cron chama o `RruleSlotGenerationJob`, que só gera quando há `rruleDateToGenerate`. Agendas com `automaticallyGenerateSlots` e sem RRULE têm de ser geradas com `generate-sync`/`generate-async`.
2. **Agenda com `typeOfRecurrence = NONE`:** a geração falha com `500` (o período não é calculável) e, com `automaticallyGenerateSlots`, a criação devolve `500` **depois** de gravar a agenda (o cron não aceita `NONE`); o hash de validação não chega a ser consumido.
3. **Apagar uma agenda não apaga os slots.** O job de remoção corre depois do soft delete e falha com `ScheduleNotFound` porque a agenda já está apagada; o Hangfire repete-o 10 vezes.
4. **Cadeias de jobs duplicadas.** Definir uma RRULE (`PATCH /rrule`) numa agenda criada sem ela não remove o recurring job de cron; cada execução deste passa a agendar também a próxima execução da RRULE. O advisory lock impede erros de concorrência, mas cada execução extra gera um período adiantado.
5. **Hash de validação em memória.** A cache usa `AddDistributedMemoryCache`: perde-se num reinício e não é partilhada entre instâncias. Com várias instâncias, a criação pode falhar com `ScheduleValidationRequired` se cair noutra instância. O hash também não está ligado ao utilizador.
6. **Exclusões criadas sempre com `visibility = SYSTEM`** — ver [Exclusões](./exclusions.md#âmbito-de-uma-exclusão).
7. **Exclusões apagadas continuam a contar na geração** enquanto estiverem associadas à agenda.
8. **A criação de marcação não verifica:** visibilidade do slot (`DRAFT` aceita marcações), exclusões, prazo de reserva (`deadlineForSlotBookingInHours`) nem marcações do próprio paciente à mesma hora. Estas verificações existem só nos [utilitários](./utilities.md).
9. **Prazo de cancelamento** usa `deadlineForSlotBookingInHours`; `advanceCancellationInHours` não é usado.
10. **Sem autorização nem identidade do token:** qualquer utilizador autenticado pode fazer tudo, e os atores (`requestedByTaxId`, `senderTaxId`, ...) vêm do body.
11. **`AppointmentDto` não inclui** `patientTaxId`, `professionalTaxId`, `slotId` nem `phoneNumber`.
12. **Paginação de reagendamentos e encaminhamentos** não é normalizada: `page < 1` ou `limit < 1` podem dar `500`, e `limit` não tem máximo.
13. **`search` nos encaminhamentos** usa uma comparação que o EF Core não traduz para SQL e pode devolver `500`; use `reasonContains`/`obsContains`.
14. **Mensagens dos avisos** (`warnings[].message`) estão gravadas no código-fonte com dupla codificação UTF-8 e podem chegar ao cliente com caracteres trocados (`nÃ£o`). Use o `code` para mostrar mensagens.
15. **Erros de domínio sem mensagem legível:** `message` repete o `code`.

---

## Navegação

← [Utilitários](./utilities.md) · [Índice do módulo](../index.md)

**Neste módulo:** [Visão Geral](./overview.md) · [Modelos e Enumerações](./models-and-enums.md) · [Agendas](./schedules.md) · [Slots](./slots.md) · [Marcações](./appointments.md) · [Reagendamentos](./reschedules.md) · [Encaminhamentos](./forwardings.md) · [Exclusões](./exclusions.md) · [Horários de Trabalho](./working-hours.md) · [Utilitários](./utilities.md) · **Casos Internos, Eventos e Notas**
