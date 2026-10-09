# Visão Geral dos Casos de Uso — Scheduling

[⌂ Módulo Scheduling](../index.md)

## Objetivo

Esta documentação descreve os casos de uso do agendamento a partir da implementação real em `Application/Features` e da exposição HTTP em `Module/Endpoints`.

Para cada caso de uso explica-se:

- que problema resolve
- que modelos manipula
- como chamar o endpoint
- que resposta esperar em sucesso
- que erros de negócio podem ocorrer

## Visão Geral da API

- Prefixo global: `/api/v1` (`ModuleLoader`)
- Prefixo do módulo: `/scheduling`
- Stack HTTP: ASP.NET Core Minimal APIs + MediatR (`ISender`)
- Leituras: Dapper (`AdvancedQueryHelper`) sobre PostgreSQL; escritas: EF Core
- Autenticação: JWT Bearer, **obrigatória em todas as rotas** (garantido por `SchedulingRoutesTests`)
- **Autorização: nenhuma role é exigida.** Qualquer utilizador autenticado pode chamar qualquer rota do módulo

## Índice rápido de endpoints por domínio

Todas as rotas consideram o prefixo `/api/v1`.

### Agendas

| Método | Rota | Finalidade |
|---|---|---|
| `POST` | `/scheduling/schedules` | Criar agenda (exige hash de validação) |
| `GET` | `/scheduling/schedules` | Listar agendas |
| `GET` | `/scheduling/schedules/{id}` | Obter agenda por id |
| `GET` | `/scheduling/schedules/health-unit/{healthUnitTaxId}` | Listar agendas de um hospital |
| `GET` | `/scheduling/schedules/professional/{professionalTaxId}` | Listar agendas de um profissional |
| `GET` | `/scheduling/schedules/speciality/{categoryCode}` | Listar agendas de uma especialidade |
| `POST` | `/scheduling/schedules/{id}/clone` | Clonar agenda |
| `POST` | `/scheduling/schedules/{id}/restore` | Restaurar agenda apagada |
| `DELETE` | `/scheduling/schedules/{id}` | Apagar agenda (soft delete) |
| `PATCH` | `/scheduling/schedules/{id}/dates` | Alterar período |
| `PATCH` | `/scheduling/schedules/{id}/rrule` | Alterar RRULE de geração |
| `POST` / `DELETE` | `/scheduling/schedules/{id}/weekdays` | Adicionar / remover dia da semana |
| `POST` | `/scheduling/schedules/{id}/professionals` | Associar vários profissionais |
| `POST` / `DELETE` | `/scheduling/schedules/{id}/professionals/{taxId}` | Associar / remover um profissional |
| `POST` / `DELETE` | `/scheduling/schedules/{id}/locations` | Adicionar / remover localização |
| `PATCH` | `/scheduling/schedules/{id}/locations/status` | Ativar / desativar localização |
| `POST` / `DELETE` | `/scheduling/schedules/{scheduleId}/exclude-days/{excludeDayId}` | Associar / desassociar dia excluído |
| `POST` / `DELETE` | `/scheduling/schedules/{scheduleId}/exclude-ranges/{excludeRangeId}` | Associar / desassociar intervalo excluído |

Ver detalhes em [Agendas](./schedules.md).

### Slots

| Método | Rota | Finalidade |
|---|---|---|
| `GET` | `/scheduling/slots` | Listar slots |
| `GET` | `/scheduling/slots/available` | Listar slots abertos com horas disponíveis |
| `GET` | `/scheduling/slots/date/{date}` | Listar slots de um dia |
| `GET` | `/scheduling/slots/professional/{professionalTaxId}` | Listar slots de um profissional |
| `GET` | `/scheduling/schedules/{scheduleId}/slots` | Listar slots de uma agenda |
| `GET` | `/scheduling/slots/{id}` | Obter slot por id |
| `GET` | `/scheduling/slots/hours/{id}` | Obter grelha horária de um slot `TIMED` |
| `POST` / `DELETE` | `/scheduling/slots/{id}/professionals` | Associar / remover profissionais do slot |
| `PATCH` | `/scheduling/slots/{id}/capacity` | Alterar capacidade (só `CAPACITY`) |
| `PATCH` | `/scheduling/slots/{id}/open` | Abrir slot |
| `PATCH` | `/scheduling/slots/{id}/close` | Fechar slot |
| `POST` | `/scheduling/slots/schedules/{scheduleId}/slots/generate-async` | Gerar slots em background |
| `POST` | `/scheduling/slots/schedules/{scheduleId}/slots/generate-sync` | Gerar slots no pedido |
| `DELETE` | `/scheduling/slots/hard-delete/schedules/{scheduleId}/slots` | Apagar fisicamente os slots de uma agenda |

Ver detalhes em [Slots](./slots.md).

### Marcações

| Método | Rota | Finalidade |
|---|---|---|
| `POST` | `/scheduling/appointments` | Criar marcação |
| `GET` | `/scheduling/appointments` | Listar marcações |
| `GET` | `/scheduling/appointments/{id}` | Obter marcação por id |
| `GET` | `/scheduling/appointments/patient/{patientTaxId}` | Marcações de um paciente |
| `GET` | `/scheduling/appointments/professional/{professionalTaxId}` | Marcações de um profissional |
| `GET` | `/scheduling/slots/{slotId}/appointments` | Marcações de um slot |
| `GET` | `/scheduling/schedules/{scheduleId}/appointments` | Marcações de uma agenda |
| `PUT` | `/scheduling/appointments/{id}` | Trocar o profissional |
| `PATCH` | `/scheduling/appointments/{id}/confirm` | `PENDING` → `CONFIRMED` |
| `PATCH` | `/scheduling/appointments/{id}/check-in` | `CONFIRMED` → `CHECKED_IN` |
| `PATCH` | `/scheduling/appointments/{id}/start-session` | `CHECKED_IN` → `IN_SESSION` |
| `PATCH` | `/scheduling/appointments/{id}/complete` | `IN_SESSION` → `COMPLETED` |
| `PATCH` | `/scheduling/appointments/{id}/no-show` | → `NO_SHOW` |
| `PATCH` | `/scheduling/appointments/{id}/expire` | `PENDING` → `EXPIRED` |
| `PATCH` | `/scheduling/appointments/{id}/cancel-by-patient` | Cancelar (paciente) |
| `PATCH` | `/scheduling/appointments/{id}/cancel-by-professional` | Cancelar (profissional) |
| `PATCH` | `/scheduling/appointments/{id}/cancel-by-system` | Cancelar (sistema) |

Ver detalhes em [Marcações](./appointments.md).

### Reagendamentos

| Método | Rota | Finalidade |
|---|---|---|
| `POST` | `/scheduling/reschedules` | Pedir reagendamento |
| `GET` | `/scheduling/reschedules` | Listar reagendamentos |
| `GET` | `/scheduling/reschedules/{id}` | Obter por id |
| `GET` | `/scheduling/reschedules/status/{status}` | Listar por estado |
| `GET` | `/scheduling/reschedules/appointments/{appointmentId}` | Reagendamentos de uma marcação |
| `GET` | `/scheduling/reschedules/requester/{requestedByTaxId}` | Reagendamentos de um requerente |
| `PUT` | `/scheduling/reschedules/{id}` | Alterar o motivo |
| `PATCH` | `/scheduling/reschedules/{id}/approve` | Aprovar |
| `PATCH` | `/scheduling/reschedules/{id}/reject` | Rejeitar |
| `PATCH` | `/scheduling/reschedules/{id}/cancel` | Cancelar |

Ver detalhes em [Reagendamentos](./reschedules.md).

### Encaminhamentos

| Método | Rota | Finalidade |
|---|---|---|
| `POST` | `/scheduling/forwardings` | Criar encaminhamento |
| `GET` | `/scheduling/forwardings` | Listar encaminhamentos |
| `GET` | `/scheduling/forwardings/{id}` | Obter por id |
| `GET` | `/scheduling/forwardings/status/{status}` | Listar por estado |
| `GET` | `/scheduling/forwardings/appointments/{appointmentId}` | Encaminhamentos de uma marcação |
| `GET` | `/scheduling/forwardings/receiver/{receiverTaxId}` | Encaminhamentos de um destinatário |
| `PUT` | `/scheduling/forwardings/{id}` | Alterar (só `PENDING`) |
| `DELETE` | `/scheduling/forwardings/{id}` | Apagar (só `PENDING`) |
| `PATCH` | `/scheduling/forwardings/{id}/send` · `receive` · `accept` · `reject` · `complete` · `cancel` | Transições de estado |

Ver detalhes em [Encaminhamentos](./forwardings.md).

### Exclusões

| Método | Rota | Finalidade |
|---|---|---|
| `POST` | `/scheduling/exclude-days` | Criar dia excluído |
| `GET` | `/scheduling/exclude-days` | Listar dias excluídos |
| `GET` | `/scheduling/exclude-days/{id}` | Obter por id |
| `GET` | `/scheduling/exclude-days/schedules/{scheduleId}/exclude-days` | Dias excluídos associados a uma agenda |
| `PUT` | `/scheduling/exclude-days/{id}` | Alterar título e motivo |
| `PATCH` | `/scheduling/exclude-days/{id}/toggle` | Ativar / desativar |
| `DELETE` | `/scheduling/exclude-days/{id}` | Apagar (soft delete) |
| — | `/scheduling/exclude-ranges/...` | Mesmas sete rotas para intervalos excluídos |

Ver detalhes em [Exclusões](./exclusions.md).

### Horários de Trabalho

| Método | Rota | Finalidade |
|---|---|---|
| `POST` | `/scheduling/working-hours` | Criar horário |
| `GET` | `/scheduling/working-hours` | Listar horários |
| `GET` | `/scheduling/working-hours/{id}` | Obter por id |
| `PUT` | `/scheduling/working-hours/{id}` | Alterar horário |
| `DELETE` | `/scheduling/working-hours/{id}` | Apagar horário |
| `POST` | `/scheduling/working-hours/{workingHourId}/exclude-days` | Criar ausência de dia |
| `POST` | `/scheduling/working-hours/{workingHourId}/exclude-ranges` | Criar ausência de intervalo |
| `DELETE` | `/scheduling/working-hours/exclude-days/{id}` | Remover ausência de dia |
| `DELETE` | `/scheduling/working-hours/exclude-ranges/{id}` | Remover ausência de intervalo |

Ver detalhes em [Horários de Trabalho](./working-hours.md).

### Utilitários

| Método | Rota | Finalidade |
|---|---|---|
| `POST` | `/scheduling/utilities/validate/schedule` | Validar agenda e emitir hash de criação |
| `POST` | `/scheduling/utilities/validate/slot-availability` | Validar disponibilidade de um slot |
| `POST` | `/scheduling/utilities/validate/slot-publication` | Validar publicação de um slot |
| `POST` | `/scheduling/utilities/validate/appointment` | Pré-validar uma marcação |
| `POST` | `/scheduling/utilities/validate/rrule` | Expandir e validar uma RRULE |
| `GET` | `/scheduling/utilities/enums/...` | Valores das enumerações (19 rotas) |

Ver detalhes em [Utilitários](./utilities.md).

## Headers

```http
Authorization: Bearer <jwt>
Content-Type: application/json
```

`PUT /appointments/{id}` aceita também `If-Match: "<rowVersion em base64>"` para controlo de concorrência otimista — ver [Marcações](./appointments.md#36-trocar-o-profissional).

### Autoria dos registos

Ao contrário de ClinicalCatalog e BloodDonation, **o Scheduling não lê o utilizador do token**. Os campos `CreatedBy`/`UpdatedBy` das entidades não são preenchidos pelos handlers, e os identificadores de atores (`RequestedByTaxId`, `SenderTaxId`, `ReceiverTaxId`, `ProfessionalTaxId`, `PatientTaxId`) vêm **do body** e são aceites tal como enviados.

## Padrão de Resposta e Erro

### Sucesso

- `201 Created` nas criações, com `Location` a apontar para o `GET` do recurso:
  - agenda, clone, marcação, reagendamento, encaminhamento: corpo com o id
  - dia/intervalo excluído e horário de trabalho: **corpo vazio**, só `Location`
  - associar profissionais à agenda: `{ scheduleId, professionalTaxIds }`
- `200 OK` nas leituras e nos utilitários
- `204 No Content` nas alterações e eliminações
- `202 Accepted` nas duas gerações de slots
- **Exceções:** `DELETE /working-hours/exclude-days/{id}` e `DELETE /working-hours/exclude-ranges/{id}` devolvem `200 OK` sem corpo

### Envelope de erro

Em falha, o corpo é o record `Error` do SharedKernel:

```json
{
  "code": "ScheduleNotFound",
  "message": "ScheduleNotFound"
}
```

Os erros de negócio do módulo são exceções `DomainException` com um `ErrorCode` (`Domain/Enums/ErrorCode.cs`). O `SchedulingExceptionHandler` traduz cada uma no status que a própria exceção declara. **A `message` é sempre igual ao `code`** — o módulo não tem mensagens legíveis para os seus erros; o cliente deve mapear o `code`.

| Exceção | Status | Exemplo de `code` |
|---|---|---|
| `BadRequestException` | `400 Bad Request` | `SelectedHourRequired` |
| `ResourceNotFoundException` | `404 Not Found` | `SlotNotFound` |
| `ResourceConflictException` / `ResourceAlreadyModifiedException` | `409 Conflict` | `AppointmentAlreadyCancelled` |
| `UnprocessableActionException` | `422 Unprocessable Entity` | `ScheduleHasSlotsWithAppointments` |

Regra geral: **`422` quando um pedido bem formado viola uma regra de negócio** (estado, capacidade, prazo), `400` quando o pedido em si está errado, `409` quando colide com o estado atual de outro recurso.

Outros erros transversais:

| Situação | Status | `code` |
|---|---|---|
| Falha de FluentValidation | `400` | `ValidationFailed` — a `message` junta `Propriedade: erro` separados por `; ` |
| Body ilegível, parâmetro de rota/query inválido, parâmetro obrigatório em falta | `400` | `BadRequest` |
| Violação de índice único no `SchedulingDbContext` | `409` | `UniqueConstraintViolation` |
| Sem token válido | `401` | corpo vazio |

### Envelopes de paginação

A maioria das listagens devolve `PaginatedList<T>`:

```json
{
  "data": [],
  "pagination": {
    "page": 1,
    "limit": 20,
    "totalItems": 0,
    "totalPages": 0,
    "hasNext": false,
    "hasPrev": false
  },
  "filters": {
    "applied": { "Page": 1, "Limit": 20, "IsClosed": true },
    "available": {
      "IsClosed": [true, false],
      "Mode": [{ "name": "TIMED", "value": 0 }, { "name": "CAPACITY", "value": 1 }, { "name": "UNLIMITED", "value": 2 }],
      "DateFrom": { "type": "DateOnly" },
      "HealthUnitTaxId": { "type": "text" }
    }
  }
}
```

- `filters.applied` lista os filtros enviados (ignora `null`, `false`, `0` e texto vazio)
- `filters.available` lista todos os filtros possíveis: `[true, false]` para booleanos, `{ name, value }` para enumerações, `{ type }` para os restantes
- as chaves destes dois dicionários vêm com o nome C# da propriedade (PascalCase), ao contrário do resto do JSON
- `page` e `limit` são normalizados por `PageLimits`: `page < 1` → `1`; `limit < 1` → `20`; `limit > 200` → `200`

Os **encaminhamentos** usam um envelope próprio (`PagedResult<T>`), com `pagination.total` e um bloco `stats` — ver [Encaminhamentos](./forwardings.md#52-listar-encaminhamentos).

`GET /reschedules/appointments/{appointmentId}` devolve **um array simples**, sem envelope.

### Binding dos filtros de listagem

As listagens com filtros (`GET /schedules`, `/slots`, `/appointments`, `/reschedules`, `/forwardings`, `/exclude-*`, `/working-hours`) usam o `QueryObjectBinder`:

- o nome do parâmetro é o nome da propriedade, **sem distinguir maiúsculas** (`?isClosed=true` ou `?IsClosed=true`)
- listas aceitam valores separados por vírgula ou o parâmetro repetido (`?statusIn=PENDING,SENT`)
- enumerações aceitam o nome sem distinguir maiúsculas; datas em ISO (`2026-10-09`), horas `HH:mm`
- um valor que não converte devolve `400 BadRequest`

As rotas de atalho `GET /schedules/professional/{taxId}`, `/schedules/speciality/{code}`, `/forwardings/status|appointments|receiver/...` e `/reschedules/status|requester/...` só leem `page` e `limit`, e estes são **obrigatórios**: omiti-los devolve `400`.

### Âmbito organizacional

As listagens que herdam de `ScopedQuery` (agendas, slots, marcações, exclusões, horários de trabalho) aceitam três filtros extra, combinados com `AND`:

| Parâmetro | Tipo | Efeito |
|---|---|---|
| `unitIds` | `guid[]` | `UnitId` pertence à lista |
| `departmentIds` | `guid[]` | `DepartmentId` pertence à lista |
| `sectorIds` | `guid[]` | `SectorId` pertence à lista |

São filtros opcionais enviados pelo cliente — **não** são impostos a partir do token.

---

## Navegação

[Índice do módulo](../index.md) · [Modelos e Enumerações](./models-and-enums.md) →

**Neste módulo:** **Visão Geral** · [Modelos e Enumerações](./models-and-enums.md) · [Agendas](./schedules.md) · [Slots](./slots.md) · [Marcações](./appointments.md) · [Reagendamentos](./reschedules.md) · [Encaminhamentos](./forwardings.md) · [Exclusões](./exclusions.md) · [Horários de Trabalho](./working-hours.md) · [Utilitários](./utilities.md) · [Casos Internos, Eventos e Notas](./internal-and-events.md)
