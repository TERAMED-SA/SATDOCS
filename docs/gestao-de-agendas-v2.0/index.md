# Módulo Scheduling

## Visão Geral

O módulo `Scheduling` gere o **agendamento clínico** da plataforma: as **agendas** de cada hospital e item clínico, os **slots** (dias de atendimento) gerados a partir delas, as **marcações** de pacientes, os **reagendamentos** e **encaminhamentos** dessas marcações, as **exclusões** (dias e intervalos sem atendimento) e os **horários de trabalho** dos profissionais.

Consome os módulos [Gestão Hospitalar](../hospital/index.md) (hospital ativo, ofertas, estrutura física) e [Catálogo Clínico (v2.0)](../catalogo-clinico-v2.0/index.md) (item clínico e especialidade) através de adaptadores ACL. A geração de slots corre em background com Hangfire.

## Estrutura de projetos

| Projeto | Responsabilidade |
|---|---|
| `Sat.Scheduling.Domain` | Entidades, enumerações, `ErrorCode`, regras de negócio puras (`ScheduleLogic`, `SlotLogic`, `AppointmentLogic`, `RecurrenceLogic`, `RruleLogic`) |
| `Sat.Scheduling.Application` | Commands, Queries, Handlers (MediatR), validadores FluentValidation, DTOs, jobs de geração de slots |
| `Sat.Scheduling.Contracts` | Interfaces ACL para outros módulos (`IHospitalCatalogService`, `IClinicalCatalogService`) e eventos `V1` |
| `Sat.Scheduling.Infrastructure` | `SchedulingDbContext`, configurações EF e Dapper, repositórios, Hangfire, adaptadores ACL, migrações |
| `Sat.Scheduling.Module` | Composition root: registo de serviços, `SchedulingExceptionHandler` e mapeamento de endpoints |
| `Sat.Scheduling.Tests` | Testes de registo de rotas |

## Modelo mental

```
Schedule 1 ── 1 ScheduleConfigurations
    │
    ├── N Slot (um por dia)  1 ── N Appointment  1 ── N Reschedule
    │      └── Hours[] (TimeBox, só em modo TIMED)       └── N Forwarding
    │
    └── N ExcludeDay / ExcludeRange  (associação ou âmbito)

WorkingHour (profissional × hospital × especialidade × dia da semana)
    └── N ExcludeDay / ExcludeRange  (Kind = WORKING)
```

- uma **agenda** define período, recorrência, dias da semana, janela horária e regras (capacidade, idade, género, pagamento, aprovação)
- os **slots** são gerados por período de recorrência; cada slot opera em modo `TIMED` (grelha de horários) ou `CAPACITY` (vagas sem hora)
- uma **marcação** ocupa uma vaga de um slot e é atribuída a um profissional com horário de trabalho compatível
- as **exclusões** retiram dias ou intervalos horários à geração e à disponibilidade

## Navegação

- [Visão Geral dos Casos de Uso](./use-cases/overview.md)
- [Modelos e Enumerações](./use-cases/models-and-enums.md)
- [Agendas](./use-cases/schedules.md)
- [Slots](./use-cases/slots.md)
- [Marcações](./use-cases/appointments.md)
- [Reagendamentos](./use-cases/reschedules.md)
- [Encaminhamentos](./use-cases/forwardings.md)
- [Exclusões](./use-cases/exclusions.md)
- [Horários de Trabalho](./use-cases/working-hours.md)
- [Utilitários](./use-cases/utilities.md)
- [Casos Internos, Eventos e Notas](./use-cases/internal-and-events.md)

## Estrutura sugerida de leitura

1. Comece por [Visão Geral](./use-cases/overview.md)
2. Consulte [Modelos e Enumerações](./use-cases/models-and-enums.md)
3. Siga o fluxo natural: [Utilitários](./use-cases/utilities.md) (validar) → [Agendas](./use-cases/schedules.md) → [Slots](./use-cases/slots.md) → [Marcações](./use-cases/appointments.md) → [Reagendamentos](./use-cases/reschedules.md) / [Encaminhamentos](./use-cases/forwardings.md)
4. Configure a disponibilidade em [Horários de Trabalho](./use-cases/working-hours.md) e [Exclusões](./use-cases/exclusions.md)
5. Feche em [Casos Internos, Eventos e Notas](./use-cases/internal-and-events.md) — jobs, eventos, persistência e limitações conhecidas
