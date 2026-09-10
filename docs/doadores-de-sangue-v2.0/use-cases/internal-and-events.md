# Casos Internos, Eventos e Notas — BloodDonation

[⌂ Módulo BloodDonation](../index.md) · [Visão Geral](./overview.md)

## Casos de uso não expostos por HTTP

Correm em background, sem rota associada.

### `ExpireOldRequestsCommand`

- Executado por: `ExpireRequestsBackgroundService`
- Periodicidade: **de hora a hora** (`TimeSpan.FromHours(1)`)
- O que faz: obtém os pedidos abertos cuja `ExpiresAt` já passou (`GetExpiredOpenAsync`) e chama `Expire()` em cada um
- Retorno: número de pedidos expirados; registado em log quando > 0
- Falhas: apanhadas e registadas em log; o serviço continua a correr

Como `Expire()` só é válido a partir de `Open`, pedidos que já tenham entrado em `Matching` **não são expirados por este job**, mesmo que a data tenha passado.

### `CleanupExpiredMatchesCommand`

- Executado por: `CleanupMatchesBackgroundService`
- Periodicidade: **de 30 em 30 minutos**
- O que faz: obtém os matches `Pending` associados a pedidos expirados (`GetPendingByExpiredRequestsAsync`) e rejeita-os com a razão `"Request expired."`
- Retorno: número de matches limpos

Ambos os serviços são registados em `InfrastructureExtensions.AddBloodDonationInfrastructure` via `AddHostedService` e partilham a base `PeriodicJob`, que trata do ritmo: a primeira passagem ocorre após um atraso aleatório de até cinco minutos, e as seguintes seguem o intervalo do trabalho. Uma passagem que falhe é registada em log e repetida no tique seguinte.

## Eventos de Domínio

Definidos em `Domain/Events`, todos herdando de `DomainEvent` (SharedKernel) e com sufixo de versão `V1`:

| Evento | Payload | Intenção |
|---|---|---|
| `DonorRegisteredV1` | `DonorId`, `UserId`, `BloodType` | novo dador registado |
| `BloodRequestCreatedV1` | `RequestId`, `RequesterUserId`, `BloodType`, `Urgency` | novo pedido publicado |
| `MatchAcceptedV1` | `MatchId`, `RequestId`, `DonorProfileId` | match aceite pelo requerente |
| `MatchRejectedV1` | `MatchId`, `RequestId`, `DonorProfileId`, `Reason` | match rejeitado |
| `DonationConfirmedV1` | `DonationId`, `MatchId`, `DonorProfileId`, `RequestId`, `ActualQuantityUnits` | dação confirmada |

Estes tipos definem o formato dos eventos para uso futuro. Não são publicados nem consumidos pela versão atual, pelo que não devem ser usados como mecanismo de integração.

## Comunicação entre módulos

### `IHospitalContracts` (porta ACL)

Definida em `Contracts/V1/IHospitalContracts.cs`, no vocabulário do próprio módulo BloodDonation:

```csharp
Task<HospitalDonationDecision> IsAllowedHospitalForDonationAsync(Guid hospitalId, CancellationToken ct);
```

`HospitalDonationDecision`: `Allowed`, `NotFound`, `Inactive`, `NoBloodBank`, `NotEligibleForDonation`.

### `HospitalContracts` (adaptador)

Em `Infrastructure/Acl/HospitalContracts.cs`. É o **único ponto do módulo autorizado a referenciar** `Sat.HospitalManagement.Contracts`, traduzindo a decisão do módulo de hospitais para o vocabulário local. A regra de elegibilidade do hospital vive no HospitalManagement; o BloodDonation apenas traduz a resposta.

Consumidores: `ConfirmDonationCommandHandler` e `ValidateDonationEligibilityQueryHandler`.

## Persistência

- `DbContext`: `BloodDonationDbContext`, schema PostgreSQL **`blood_donation`**
- `DbSet`s: `Donors`, `DonorAnalyses`, `BloodRequests`, `Donations`, `Matches`, `Reports`
- Unit of Work: o próprio `DbContext` implementa `IBloodDonationUnitOfWork`; os handlers gravam com um único `SaveChangesAsync` por caso de uso
- Repositórios registados como *scoped*: `IDonorRepository`, `IDonorAnalysisRepository`, `IBloodRequestRepository`, `IDonationRepository`, `IMatchRepository`, `IReportRepository`
- Migrações: `20260520203502_InitialCreate`, `20260528094439_AddDonorAnalysis`

### Aplicar migrações

```bash
make migrate-blood-donation
```

Equivalente a:

```bash
dotnet ef database update \
  --project modules/BloodDonation/Infrastructure \
  --startup-project src/Sat.Api \
  --context BloodDonationDbContext
```

## Composition root

`BloodDonationModule` implementa `IModule` (SharedKernel) e é descoberto automaticamente por reflexão pelo `ModuleScanner`. Regista:

- Application: MediatR sobre o assembly marcado por `BloodDonationApplicationMarker`
- Infrastructure: `DbContext` (Npgsql), UoW, repositórios, ACL de hospital, background services
- Endpoints: `Donors`, `Analysis`, `Donations`, `Requests`, `Matching`, `Moderation`, `Admin`

## Cobertura de testes

Em `Tests/`, 305 testes:

- **Domínio:** `DonorTests`, `DonorAnalysisTests`, `DonorSpecificationsTests`, `BloodRequestTests`, `MatchTests`, `DonationTests`, `ReportTests`, `BloodCompatibilityTests`
- **Aplicação:** registo e disponibilidade de dadores, análises (aprovar, rejeitar, re-análise), criação e aceitação de matches, confirmação de dação e confirmação hospitalar, elegibilidade, denúncias e bloqueio de dadores, paginação e normalização de datas
- **Infraestrutura:** `PeriodicJobTests`

---

## Navegação

← [Administração](./admin.md) · [Índice do módulo](../index.md)

**Neste módulo:** [Visão Geral](./overview.md) · [Modelos e Enumerações](./models-and-enums.md) · [Dadores](./donors.md) · [Análises Clínicas](./analysis.md) · [Pedidos de Sangue](./requests.md) · [Matching](./matching.md) · [Dações](./donations.md) · [Moderação](./moderation.md) · [Administração](./admin.md) · **Casos Internos, Eventos e Notas**
