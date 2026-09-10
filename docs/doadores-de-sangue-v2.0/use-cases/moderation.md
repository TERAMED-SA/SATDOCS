# Casos de Uso de Moderação

[⌂ Módulo BloodDonation](../index.md) · [Visão Geral](./overview.md)

Grupo de rotas: `/api/v1/blood-donation/moderation` · Requer autenticação; a maioria exige a role `SystemAdmin`.

## Rotas para todos os utilizadores autenticados

### 6.1 Criar denúncia

- Caso de uso: `CreateReportCommand`
- Endpoint: `POST /api/v1/blood-donation/moderation/reports`
- Modelo principal: `Report`

#### Body

```json
{
  "reportType": "Fraud",
  "targetType": "Donor",
  "targetId": "f1ac3c5e-6fd2-4a7a-b64e-72f40be690cf",
  "reason": "Fraud",
  "description": "Perfil apresenta dados clínicos inconsistentes.",
  "urgency": "High",
  "evidenceUrl": "https://exemplo.ao/prova.png"
}
```

`urgency` assume `Medium` por omissão; `evidenceUrl` é opcional.

#### Regras de negócio

- `description` é obrigatória (`Report.DescriptionRequired`)
- validação do alvo, consoante `targetType`:
  - `Donor` → o dador tem de existir (`Donor.NotFound`) e não pode ser o próprio (`Report.SelfReport`)
  - `BloodRequest` → o pedido tem de existir (`BloodRequest.NotFound`) e não pode ser do próprio (`Report.SelfReport`)
  - `Match` → o match tem de existir (`Match.NotFound`) e o denunciante não pode ser o dador desse match (`Report.SelfReport`). Denunciar a contraparte é legítimo: quem fez o pedido pode denunciar o match em que o dador falhou
- anti-duplicação: não é possível denunciar o mesmo alvo mais do que uma vez em **7 dias** (`Report.DuplicateReport`)
- o estado inicial é `Pending`

#### Sucesso esperado

- `200 OK`
- retorno: `ReportResponse`

#### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `400` | `Report.DescriptionRequired` | descrição vazia |
| `404` | `Donor.NotFound` / `BloodRequest.NotFound` / `Match.NotFound` | alvo inexistente |
| `400` | `Report.SelfReport` | tentativa de auto-denúncia |
| `409` | `Report.DuplicateReport` | denúncia repetida em menos de 7 dias |
| `401` | — | não autenticado |

### 6.2 Listar as minhas denúncias

- Caso de uso: `GetMyReportsQuery`
- Endpoint: `GET /api/v1/blood-donation/moderation/reports/my`

#### Query params

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `status` | string | filtra por `ReportStatus`; valor inválido é ignorado |
| `targetType` | string | filtra por `ReportTargetType`; idem |
| `recentOnly` | bool | restringe às denúncias dos últimos 30 dias |
| `page` | int | página (1-based), por omissão `1` |
| `limit` | int | itens por página, por omissão `20` (máximo `200`) |

#### Regras de negócio

- filtragem e paginação feitas na base de dados, sobre as denúncias do chamador
- ordenação por `createdAt` descendente

#### Sucesso esperado

- `200 OK`
- retorno: `PagedResult<ReportResponse>` — ver [Envelope de paginação](./overview.md#envelope-de-paginacao)

## Rotas administrativas (`SystemAdmin`)

### 6.3 Listar denúncias pendentes

- Caso de uso: `GetPendingReportsQuery`
- Endpoint: `GET /api/v1/blood-donation/moderation/reports/pending`
- Query params: `page` (por omissão `1`), `limit` (por omissão `20`, máximo `200`)

Devolve `200 OK` com `PagedResult<ReportResponse>` em estado pendente, das mais recentes para as mais antigas — ver [Envelope de paginação](./overview.md#envelope-de-paginacao).

### 6.4 Obter denúncia por id

- Caso de uso: `GetReportByIdQuery`
- Endpoint: `GET /api/v1/blood-donation/moderation/reports/{id:guid}`

| HTTP | Code | Situação |
|---|---|---|
| `200` | — | `ReportResponse` |
| `404` | `Report.NotFound` | denúncia inexistente |
| `403` | — | sem role `SystemAdmin` |

### 6.5 Atualizar denúncia

- Caso de uso: `UpdateReportCommand`
- Endpoint: `PATCH /api/v1/blood-donation/moderation/reports/{id:guid}`
- Finalidade: mover a denúncia no fluxo de moderação (rever, resolver, arquivar)

#### Body

```json
{
  "status": "Resolved",
  "resolutionNotes": "Confirmado; dador bloqueado por 30 dias."
}
```

#### Regras de negócio

- a denúncia tem de existir (`Report.NotFound`)
- as transições válidas são:
  - `Pending` → `UnderReview` (`Report.CannotReview` a partir de outro estado)
  - `Pending` ou `UnderReview` → `Resolved` / `Dismissed`
- `Resolved` e `Dismissed` são finais: uma denúncia encerrada não é reaberta (`Report.CannotReopen`) nem encerrada de novo (`Report.AlreadyResolved`), o que preserva quem a fechou e quando
- quando o novo estado é `Resolved` ou `Dismissed`, são preenchidos `ResolvedAt` (agora), `ResolvedBy` (moderador autenticado) e `ResolutionNotes`
- `resolutionNotes` é gravado em qualquer transição, e também quando `status` não é enviado

#### Sucesso esperado

- `200 OK`
- retorno: `ReportResponse` atualizado

#### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Report.NotFound` | denúncia inexistente |
| `400` | `Report.CannotReview` | `UnderReview` pedido a partir de um estado que não `Pending` |
| `409` | `Report.CannotReopen` | tentativa de reabrir uma denúncia encerrada |
| `409` | `Report.AlreadyResolved` | denúncia já resolvida ou arquivada |
| `403` | — | sem role `SystemAdmin` |

### 6.6 Bloquear dador

- Caso de uso: `BlockDonorCommand`
- Endpoint: `POST /api/v1/blood-donation/moderation/block-donor/{id:guid}`
- `{id}` é o `Donor.Id`

#### Body

```json
{
  "reason": "Fraude confirmada",
  "notes": "Reincidente",
  "blockDuration": "permanent"
}
```

`blockDuration` aceita `"permanent"`, a ausência de valor (equivalente a permanente), ou uma quantidade positiva seguida de `d` (dias), `m` (meses) ou `y` (anos) — por exemplo `"30d"`, `"6m"`, `"1y"`. Qualquer outro valor é recusado com `Donor.InvalidBlockDuration`; não há interpretação por omissão.

#### Regras de negócio

- o dador tem de existir (`Donor.NotFound`)
- não pode já ter um bloqueio ativo (`Donor.AlreadyBlocked`)
- `blockDuration` tem de ser interpretável (`Donor.InvalidBlockDuration`)
- efeitos em cascata:
  - com prazo, `BlockedUntil` fica com a data de fim; sem prazo, `IsPermanentlyBlocked` fica `true`
  - `reason` e `notes` são gravados em `BlockReason`/`BlockNotes`
  - todos os matches `Pending` do dador são rejeitados com a razão `"Donor blocked"`
  - todos os matches `Accepted` são retirados com a razão `"Donor blocked"` e os pedidos correspondentes voltam a `Open`

O bloqueio não altera o estado clínico do dador: enquanto durar, o `status` devolvido é `Blocked`; quando termina, o dador volta ao estado clínico que tinha. Um bloqueio com prazo termina sozinho quando a data passa; um permanente só sai por desbloqueio.

#### Sucesso esperado

- `200 OK`
- retorno: `BlockDonorResultResponse`

```json
{
  "donorId": "f1ac3c5e-6fd2-4a7a-b64e-72f40be690cf",
  "status": "Blocked",
  "blockedUntil": null
}
```

#### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Donor.NotFound` | dador inexistente |
| `400` | `Donor.InvalidBlockDuration` | `blockDuration` não interpretável |
| `409` | `Donor.AlreadyBlocked` | já tem bloqueio ativo |
| `403` | — | sem role `SystemAdmin` |

### 6.7 Desbloquear dador

- Caso de uso: `UnblockDonorCommand`
- Endpoint: `POST /api/v1/blood-donation/moderation/unblock-donor/{id:guid}`
- Sem body

#### Regras de negócio

- o dador tem de ter um bloqueio ativo (`Donor.NotBlocked`)
- efeitos: `BlockedUntil`, `IsPermanentlyBlocked`, `BlockReason` e `BlockNotes` limpos

O desbloqueio levanta a sanção e o dador volta ao estado clínico que tinha antes — quem estava em `PendingAnalysis` continua a precisar de análise, e um adiamento em curso mantém-se.

#### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Donor.NotFound` | dador inexistente |
| `400` | `Donor.NotBlocked` | dador não tem bloqueio ativo |
| `403` | — | sem role `SystemAdmin` |

### 6.8 Estatísticas de moderação

- Caso de uso: `GetModerationStatsQuery`
- Endpoint: `GET /api/v1/blood-donation/moderation/stats`
- Query params: `timeframeDays` (por omissão `30`)

#### Sucesso esperado

- `200 OK`
- retorno: `ModerationStatsResponse`

```json
{
  "timeframeDays": 30,
  "totalReports": 42,
  "pendingReports": 7,
  "resolvedReports": 31,
  "blockedDonors": 3
}
```

### 6.9 Avaliação de risco do dador

- Caso de uso: `GetDonorRiskAssessmentQuery`
- Endpoint: `GET /api/v1/blood-donation/moderation/risk-assessment/{donorId:guid}`

#### Regras de negócio

- o dador tem de existir (`Donor.NotFound`)
- o `riskScore` é calculado como **`número de denúncias resolvidas contra o dador × 20`** — sem teto, sem ponderação por urgência, tipo de denúncia ou histórico de dações

#### Sucesso esperado

- `200 OK`
- retorno: `DonorRiskAssessmentResponse`

```json
{
  "donorId": "f1ac3c5e-6fd2-4a7a-b64e-72f40be690cf",
  "userId": "9a7b6c5d-4e3f-2a1b-0c9d-8e7f6a5b4c3d",
  "riskScore": 40,
  "resolvedReportsCount": 2,
  "totalDonations": 5
}
```

#### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Donor.NotFound` | dador inexistente |
| `403` | — | sem role `SystemAdmin` |

### 6.10 Denúncias por alvo

- Caso de uso: `GetReportsByTargetQuery`
- Endpoint: `GET /api/v1/blood-donation/moderation/target-reports/{targetType}/{targetId:guid}`

#### Parâmetros

| Parâmetro | Local | Descrição |
|---|---|---|
| `targetType` | rota | `Donor`, `BloodRequest` ou `Match` (case-insensitive) |
| `targetId` | rota | id do alvo |
| `includeResolved` | query | incluir denúncias já encerradas |
| `page` | query | página (1-based), por omissão `1` |
| `limit` | query | itens por página, por omissão `20` (máximo `200`) |

#### Regras de negócio

- `targetType` inválido devolve `400` com o texto `"Invalid targetType."` (string simples, **não** o envelope `Error`)
- o filtro `includeResolved` é aplicado na base de dados, pelo que `total` conta apenas as denúncias visíveis com o filtro em vigor
- ordenação por `createdAt` descendente

#### Sucesso esperado

- `200 OK`
- retorno: `PagedResult<ReportResponse>` — ver [Envelope de paginação](./overview.md#envelope-de-paginacao)

## Contrato `ReportResponse`

```
Id, ReporterUserId, ReportType, TargetType, TargetId, Reason, Description,
Urgency, EvidenceUrl, Status, ResolvedAt, ResolvedBy, ResolutionNotes, CreatedAt
```

---

## Navegação

← [Dações](./donations.md) · [Índice do módulo](../index.md) · [Administração](./admin.md) →

**Neste módulo:** [Visão Geral](./overview.md) · [Modelos e Enumerações](./models-and-enums.md) · [Dadores](./donors.md) · [Análises Clínicas](./analysis.md) · [Pedidos de Sangue](./requests.md) · [Matching](./matching.md) · [Dações](./donations.md) · **Moderação** · [Administração](./admin.md) · [Casos Internos, Eventos e Notas](./internal-and-events.md)
