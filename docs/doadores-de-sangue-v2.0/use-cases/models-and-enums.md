# Modelos e Enumerações — BloodDonation

[⌂ Módulo BloodDonation](../index.md) · [Visão Geral](./overview.md)

## Modelos de Domínio

Todas as entidades são `AggregateRoot<Guid>` e vivem em `Domain/Entities`. Os setters são privados: o estado só muda através de métodos de domínio que devolvem `Result`.

### Donor

Perfil de dador associado a um utilizador autenticado.

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | Identificador do perfil de dador (≠ `UserId`) |
| `UserId` | `Guid` | Utilizador dono do perfil; único por dador |
| `BloodType` | `BloodType` | Grupo ABO + fator Rh |
| `DonorMode` | `DonorMode` | Regime de dação |
| `Status` | `DonorStatus` | Estado de elegibilidade |
| `EmergencyContact` | `string?` | Contacto de emergência |
| `PreferredHospitalId` | `Guid?` | Hospital preferido |
| `UnavailableUntil` | `DateTime?` | Indisponibilidade declarada pelo próprio dador. Exposto por HTTP também como `availableUntil` (obsoleto, mesmo valor) |
| `DeferredUntil` | `DateTime?` | Adiamento imposto: rejeição clínica temporária ou intervalo entre dações |
| `DeferralReason` | `DonorDeferralReason?` | Origem do adiamento imposto |
| `LastDonationDate` | `DateTime?` | Data da última dação |
| `BlockedUntil` | `DateTime?` | Fim do bloqueio de moderação; `null` num bloqueio permanente |
| `IsPermanentlyBlocked` | `bool` | Bloqueio sem prazo |
| `BlockReason` | `string?` | Motivo do bloqueio de moderação |
| `BlockNotes` | `string?` | Notas internas do bloqueio |
| `HasMedicalRestrictions` | `bool` | Se tem restrições médicas |
| `MedicalRestrictionsDetails` | `string?` | Obrigatório quando `HasMedicalRestrictions = true` |
| `BloodTypeCorrectedAt` | `DateTime?` | Data da última correção do tipo sanguíneo |
| `BloodTypeCorrectedBy` | `Guid?` | Quem a fez |
| `BloodTypeCorrectionReason` | `string?` | Motivo indicado na correção |
| `IsVerified` | `bool` | Passa a `true` na aprovação da análise |
| `VerificationNotes` | `string?` | Notas do analista |
| `TotalDonations` | `int` | Contador de dações confirmadas |
| `CreatedAt` / `UpdatedAt` | `DateTime` | Auditoria temporal |

**Métodos de domínio:** `Create`, `ApproveAnalysis`, `RejectAnalysis`, `ReopenAnalysis`, `Block`, `Unblock`, `IsBlockedAt`, `MarkAvailable`, `MarkUnavailableUntil`, `Defer`, `RegisterDonation`, `CorrectBloodType`, `CanDonateAt`, `EffectiveStatusAt`, `SetMedicalRestrictions`, `UpdateProfile`.

#### Estado guardado e estado devolvido

O `Status` guardado descreve a **via clínica** do dador. As restrições que terminam por data — indisponibilidade declarada, intervalo entre dações e bloqueio com prazo — não são guardadas no `Status`: vivem nos seus próprios campos e deixam de valer quando a data passa.

O campo `status` das respostas HTTP é o **estado efetivo**, calculado no momento do pedido:

| Situação | `status` devolvido |
|---|---|
| Bloqueio ativo, com ou sem prazo | `Blocked` |
| `DeferredUntil` ou `UnavailableUntil` no futuro | `TemporarilyIneligible` |
| Rejeição clínica temporária | `TemporarilyIneligible` |
| Restante | o estado guardado |

Um dador só pode doar quando o estado guardado é `Eligible`, tem análise aprovada, não está bloqueado e não tem nenhuma das duas datas ativa.

**Ciclo de vida da via clínica:**

```
PendingAnalysis ──aprovação──> Eligible
       │                          │
       │                          ├── MarkUnavailableUntil(data) ──> continua Eligible,
       │                          │      com UnavailableUntil preenchido
       │                          └── dação confirmada ──> continua Eligible,
       │                                 com DeferredUntil = +56 dias
       │
       ├──rejeição temporária──> TemporarilyIneligible
       │        └── ReopenAnalysis() ──> PendingAnalysis  (após o prazo terminar)
       └──rejeição permanente──> PermanentlyIneligible
```

O bloqueio de moderação é uma dimensão à parte e não altera o estado clínico: um dador bloqueado em `PendingAnalysis` volta a `PendingAnalysis` quando o bloqueio termina.

Restrições de transição:

- `ApproveAnalysis` e `RejectAnalysis` só são válidos a partir de `PendingAnalysis` (senão: `Analysis.NotPendingAnalysis`)
- uma rejeição temporária exige `ineligibleUntil` (`Analysis.IneligibleUntilRequired`)
- `ReopenAnalysis` exige `TemporarilyIneligible`, prazo terminado e ausência de bloqueio (`Analysis.CannotReopen`, `Analysis.DeferralStillActive`)
- `MarkAvailable` e `MarkUnavailableUntil` são recusados em `PendingAnalysis`, `TemporarilyIneligible`, `PermanentlyIneligible` e durante um bloqueio (`Donor.InvalidStatusTransition`); limpam ou definem apenas a indisponibilidade declarada, nunca um adiamento imposto
- `Block` recusa se já houver bloqueio ativo (`Donor.AlreadyBlocked`); `Unblock` recusa se não houver (`Donor.NotBlocked`)
- não há transição de saída de `PermanentlyIneligible`

### DonorAnalysis

Registo da análise clínica que decide a elegibilidade do dador. É criada automaticamente com o perfil.

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `DonorId` | `Guid` | Referência ao `Donor.Id` |
| `ReviewedBy` | `Guid?` | Utilizador que reviu |
| `Status` | `AnalysisStatus` | |
| `Notes` | `string?` | Preenchido na aprovação |
| `RejectionReason` | `string?` | Preenchido na rejeição |
| `IsPermanentRejection` | `bool` | |
| `IneligibleUntil` | `DateTime?` | Só em rejeição temporária |
| `CreatedAt` / `ReviewedAt` / `UpdatedAt` | `DateTime` | |

Uma análise só pode ser revista uma vez (`Analysis.AlreadyReviewed`). Uma rejeição permanente exige motivo; uma rejeição temporária exige motivo e data futura.

Depois de o prazo de uma rejeição temporária terminar, o dador pode pedir nova análise em `POST /donors/me/reanalysis`, o que abre uma análise em `Pending` e devolve o dador a `PendingAnalysis`. Um dador só pode ter uma análise pendente de cada vez (`Analysis.AlreadyPending`).

### BloodRequest

Pedido de sangue criado por um requerente.

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `RequesterUserId` | `Guid` | Dono do pedido |
| `BloodType` | `BloodType` | |
| `QuantityUnits` | `int` | Tem de ser > 0 |
| `Urgency` | `RequestUrgency` | Define a expiração por omissão |
| `Status` | `RequestStatus` | |
| `HospitalId` / `HospitalName` | `Guid?` / `string?` | |
| `Address` / `City` / `Country` | `string?` / `string?` / `string` | `Country` é obrigatório |
| `PatientCondition` | `string?` | |
| `RequiresCrossmatch` | `bool` | |
| `SpecialRequirements` | `string?` | Persistido, **não devolvido** em `BloodRequestResponse` |
| `OffersCompensation` | `bool` | |
| `CompensationAmount` / `CompensationCurrency` | `decimal?` / `string?` | Montante tem de ser > 0 |
| `ExpiresAt` | `DateTime?` | |
| `CancelledAt` / `CancelledBy` / `CancellationReason` | | Preenchidos no cancelamento |
| `CreatedAt` / `UpdatedAt` | `DateTime` | |

**Métodos de domínio:** `Create`, `Cancel`, `Fulfill`, `Expire`, `StartMatching`, `RestoreToOpen`, `SetDetails`.

**Ciclo de vida do estado:**

```
Open ──StartMatching()──> Matching ──RestoreToOpen()──> Open
 │                            │
 │                            ├── Fulfill() ──> Fulfilled
 │                            └── Cancel() ──> Cancelled
 ├── Expire() ──> Expired
 ├── Fulfill() ──> Fulfilled
 └── Cancel() ──> Cancelled
```

Notas importantes:

- `Fulfill()` é aceite a partir de `Open` e de `Matching` — na prática o pedido está em `Matching` quando a dação é confirmada; noutros estados devolve `BloodRequest.CannotFulfill`
- `Expire()` só é aceite a partir de `Open`
- `RestoreToOpen()` só atua se o estado for `Open` ou `Matching`; noutros estados devolve sucesso sem alterar nada
- `RequestStatus.Fulfilled` é o estado final de um pedido satisfeito, atribuído ao confirmar a dação

### Match

Vínculo entre um dador e um pedido.

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `RequestId` | `Guid` | |
| `DonorProfileId` | `Guid` | `Donor.Id`, não `UserId` |
| `Status` | `MatchStatus` | |
| `MatchScore` | `double?` | Campo existe mas nunca é calculado nem atribuído |
| `ProposedDonationDate` | `DateTime?` | Data proposta pelo dador; tem de ser futura |
| `AcceptedAt` / `RejectedAt` / `WithdrawnAt` | `DateTime?` | |
| `RejectionReason` / `WithdrawalReason` | `string?` | |
| `CreatedAt` / `UpdatedAt` | `DateTime` | |

**Ciclo de vida:**

```
Pending ──Accept()──> Accepted ──Withdraw()──> Withdrawn
   │                     
   ├── Reject() ──> Rejected  (terminal)
   └── Withdraw() ──> Withdrawn  (terminal)
```

### Donation

Dação efetivamente confirmada.

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `MatchId` | `Guid` | Um match só pode gerar uma dação |
| `DonorProfileId` | `Guid` | |
| `RequestId` | `Guid` | |
| `DonationDate` | `DateTime` | Não pode estar no futuro |
| `ActualQuantityUnits` | `int` | Tem de ser > 0 |
| `ConfirmedByHospital` | `bool` | Só o passo de confirmação hospitalar o coloca a `true` |
| `HospitalId` | `Guid?` | Gravado na criação da dação |
| `HospitalConfirmationDate` | `DateTime?` | Data indicada pelo hospital, ou a da confirmação |
| `HospitalConfirmationNotes` | `string?` | Notas do hospital na confirmação |
| `ConfirmedByUserId` | `Guid?` | Utilizador que registou a confirmação hospitalar |
| `Notes` | `string?` | Notas do requerente ao confirmar a dação |
| `CreatedAt` / `UpdatedAt` | `DateTime` | |

### Report

Denúncia submetida por um utilizador.

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `ReporterUserId` | `Guid` | |
| `ReportType` | `ReportType` | |
| `TargetType` | `ReportTargetType` | |
| `TargetId` | `Guid` | |
| `Reason` | `ReportReason` | |
| `Description` | `string` | Obrigatória |
| `Urgency` | `ReportUrgency` | |
| `EvidenceUrl` | `string?` | |
| `Status` | `ReportStatus` | |
| `ResolvedAt` / `ResolvedBy` / `ResolutionNotes` | | |
| `CreatedAt` / `UpdatedAt` | `DateTime` | |

**Métodos de domínio:** `Create`, `MoveToReview`, `Resolve`, `Dismiss`, `Update`.

**Ciclo de vida:**

```
Pending ──> UnderReview ──> Resolved   (terminal)
   │             └───────> Dismissed  (terminal)
   └──────────────────────> Resolved / Dismissed
```

`Update` aplica a alteração através destas transições: `Resolved` e `Dismissed` são finais, e uma denúncia encerrada não volta atrás (`Report.CannotReopen`) nem é encerrada uma segunda vez (`Report.AlreadyResolved`).

## Enumerações de Negócio

Todas em `Domain/Enums`. Os valores são serializados conforme a configuração JSON global (`AddJsonConfiguration`); os nomes abaixo são os nomes C#.

### BloodType

- `A_Positive`
- `A_Negative`
- `B_Positive`
- `B_Negative`
- `AB_Positive`
- `AB_Negative`
- `O_Positive`
- `O_Negative`

### DonorMode

- `UrgentOnly` — apenas em situações urgentes
- `Occasional` — dação ocasional
- `Regular` — dação regular (valor por omissão em `CreateDonorRequest`)

### DonorStatus

- `Eligible` (0)
- `TemporarilyIneligible` (1)
- `PermanentlyIneligible` (2)
- `Blocked` (3)
- `PendingAnalysis` (4) — estado inicial de qualquer dador

`Blocked` e, nos casos por data, `TemporarilyIneligible` são calculados no momento da resposta a partir dos campos de prazo. Ver [Estado guardado e estado devolvido](#estado-guardado-e-estado-devolvido).

### DonorDeferralReason

Origem de um adiamento imposto ao dador, em `DeferralReason`.

- `ClinicalDeferral` — rejeição clínica temporária; termina com nova análise
- `PostDonationCooldown` — intervalo mínimo entre dações; termina quando `DeferredUntil` passa

### AnalysisStatus

- `Pending` (0)
- `Approved` (1)
- `RejectedTemporary` (2)
- `RejectedPermanent` (3)

### RequestStatus

- `Open`
- `Matching`
- `Cancelled`
- `Fulfilled` — estado final de um pedido satisfeito, atribuído ao confirmar a dação
- `Expired`

Persistido como texto, não como inteiro (`HasConversion<string>`), pelo que a ordem dos membros não é significativa.

### RequestUrgency

- `Critical` — expira por omissão em 24 horas
- `High` — 3 dias
- `Medium` — 7 dias (valor por omissão)
- `Low` — 14 dias

### MatchStatus

- `Pending`
- `Accepted`
- `Rejected`
- `Withdrawn`

### ReportType

- `Safety`
- `Fraud`
- `Harassment`
- `InappropriateBehavior`
- `FalseInformation`
- `Other`

### ReportTargetType

- `Donor`
- `BloodRequest`
- `Match`

### ReportReason

- `Fraud`
- `Harassment`
- `SafetyConcern`
- `InappropriateBehavior`
- `FalseInformation`
- `Other`

### ReportUrgency

- `Critical`
- `High`
- `Medium` (valor por omissão)
- `Low`

### ReportStatus

- `Pending`
- `UnderReview`
- `Resolved`
- `Dismissed`

## Catálogo de Erros

Definidos em `Domain/Errors/BloodDonationErrors.cs`. O campo `code` da resposta corresponde exatamente a estes identificadores.

### Donor

| Code | Categoria | Mensagem |
|---|---|---|
| `Donor.NotFound` | NotFound | Donor was not found. |
| `Donor.AlreadyExists` | Conflict | A donor with the same identity already exists. |
| `Donor.NotEligible` | Validation | Donor is not eligible to donate at this time. |
| `Donor.InvalidUserId` | Validation | User ID is required. |
| `Donor.AlreadyVerified` | Conflict | Donor is already verified. |
| `Donor.AlreadyBlocked` | Conflict | Donor is already blocked. |
| `Donor.Blocked` | Validation | Donor is blocked and cannot perform this action. |
| `Donor.InvalidAvailabilityDate` | Validation | Availability date must be in the future. |
| `Donor.InvalidMedicalRestrictions` | Validation | Medical restriction details are required. |
| `Donor.AlreadyDeleted` | Conflict | Donor has already been deleted. |
| `Donor.NotBlocked` | Validation | Donor is not blocked. |
| `Donor.InvalidStatusTransition` | Validation | This operation is not permitted in the donor's current status. |

### Analysis

| Code | Categoria | Mensagem |
|---|---|---|
| `Analysis.NotFound` | NotFound | Donor analysis was not found. |
| `Analysis.AlreadyReviewed` | Conflict | This analysis has already been reviewed. |
| `Analysis.NotPendingAnalysis` | Validation | Donor is not in PendingAnalysis status. |

### BloodRequest

| Code | Categoria | Mensagem |
|---|---|---|
| `BloodRequest.NotFound` | NotFound | Blood request was not found. |
| `BloodRequest.InvalidRequesterId` | Validation | Requester ID is required. |
| `BloodRequest.InvalidQuantity` | Validation | Quantity must be greater than zero. |
| `BloodRequest.CountryRequired` | Validation | Country is required. |
| `BloodRequest.AlreadyCancelled` | Conflict | Blood request is already cancelled. |
| `BloodRequest.AlreadyFulfilled` | Conflict | Blood request is already fulfilled. |
| `BloodRequest.CannotFulfill` | Validation | Blood request cannot be fulfilled in its current status. |
| `BloodRequest.NotOpen` | Validation | Blood request is not open. |
| `BloodRequest.CannotExpire` | Validation | Blood request cannot be expired in its current status. |
| `BloodRequest.InvalidCompensationAmount` | Validation | Compensation amount must be greater than zero. |
| `BloodRequest.InvalidExpiresAt` | Validation | Expiry date must be in the future. *(definido no handler)* |
| `BloodRequest.Expired` | Validation | Blood request has expired. *(definido no handler)* |
| `BloodRequest.HasAcceptedMatches` | Validation | Cannot cancel: withdraw all accepted matches first. *(definido no handler)* |

### Match

| Code | Categoria | Mensagem |
|---|---|---|
| `Match.NotFound` | NotFound | Match was not found. |
| `Match.InvalidRequestId` | Validation | Request ID is required. |
| `Match.InvalidDonorId` | Validation | Donor ID is required. |
| `Match.CannotAccept` | Validation | Match can only be accepted when in Pending status. |
| `Match.CannotReject` | Validation | Match can only be rejected when in Pending status. |
| `Match.CannotWithdraw` | Validation | Match cannot be withdrawn in its current status. |
| `Match.DonorHasActiveMatch` | Conflict | Donor already has an active match. *(handler)* |
| `Match.DonorHasAcceptedMatch` | Conflict | Donor already has an accepted match. *(handler)* |
| `Match.DuplicateMatch` | Conflict | A match already exists for this donor and request. *(handler)* |
| `Match.InvalidProposedDate` | Validation | Proposed donation date must be in the future. *(handler)* |
| `Match.NotAccepted` | Validation | Match must be in Accepted status to confirm donation. *(handler)* |

### Donation

| Code | Categoria | Mensagem |
|---|---|---|
| `Donation.NotFound` | NotFound | Donation record was not found. |
| `Donation.AlreadyApproved` | Conflict | This donation has already been approved. |
| `Donation.InvalidBloodVolume` | Validation | Blood volume is outside the acceptable range. |
| `Donation.InvalidMatchId` | Validation | Match ID is required. |
| `Donation.InvalidDonorId` | Validation | Donor ID is required. |
| `Donation.InvalidQuantityUnits` | Validation | Quantity units must be greater than zero. |
| `Donation.AlreadyConfirmed` | Conflict | Donation has already been confirmed by hospital. |
| `Donation.DonationDateInFuture` | Validation | Donation date cannot be in the future. *(handler)* |

### Hospital

| Code | Categoria | Mensagem |
|---|---|---|
| `Hospital.NotFound` | NotFound | Hospital was not found in local records. |
| `Hospital.NotActive` | Validation | Hospital is not active. |
| `Hospital.NoBloodBank` | Validation | Hospital does not have a blood bank. |
| `Hospital.NotEligibleForDonation` | Validation | Hospital is not eligible for blood donation or has no blood bank. |

### Report

| Code | Categoria | Mensagem |
|---|---|---|
| `Report.NotFound` | NotFound | Report was not found. |
| `Report.InvalidReporterId` | Validation | Reporter ID is required. |
| `Report.DescriptionRequired` | Validation | Report description is required. |
| `Report.AlreadyResolved` | Conflict | Report has already been resolved or dismissed. |
| `Report.CannotReview` | Validation | Only pending reports can be moved to review. |
| `Report.InvalidTargetType` | Validation | Report target type is not a recognised value. |
| `Report.SelfReport` | Validation | Cannot report yourself / your own request. *(handler)* |
| `Report.DuplicateReport` | Conflict | You have already reported this target recently. *(handler)* |

### Autorização

| Code | Categoria | Mensagem |
|---|---|---|
| `Auth.Forbidden` | Forbidden | Mensagem específica do caso de uso (ex.: "Only the requester can accept a match.") |

### BloodStock

| Code | Categoria | Mensagem |
|---|---|---|
| `BloodStock.InsufficientStock` | Validation | Insufficient blood stock for this blood type. *(definido, sem uso atual)* |

---

## Navegação

← [Visão Geral](./overview.md) · [Índice do módulo](../index.md) · [Dadores](./donors.md) →

**Neste módulo:** [Visão Geral](./overview.md) · **Modelos e Enumerações** · [Dadores](./donors.md) · [Análises Clínicas](./analysis.md) · [Pedidos de Sangue](./requests.md) · [Matching](./matching.md) · [Dações](./donations.md) · [Moderação](./moderation.md) · [Administração](./admin.md) · [Casos Internos, Eventos e Notas](./internal-and-events.md)
