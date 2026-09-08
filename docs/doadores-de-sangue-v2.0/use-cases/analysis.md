# Casos de Uso de Análises Clínicas

[⌂ Módulo BloodDonation](../index.md) · [Visão Geral](./overview.md)

Grupo de rotas: `/api/v1/blood-donation/analyses` · Requer autenticação.

A análise é o mecanismo que decide a elegibilidade de um dador. É criada automaticamente em estado `Pending` quando o perfil de dador é registado (ver [Dadores](./donors.md#11-registar-se-como-dador)), e uma nova pode ser aberta depois de uma rejeição temporária terminar (ver [Pedir nova análise clínica](./donors.md#17-pedir-nova-analise-clinica)).

## 2.1 Listar análises pendentes

- Caso de uso: `GetPendingAnalysesQuery`
- Endpoint: `GET /api/v1/blood-donation/analyses/pending`
- Finalidade: obter a fila de trabalho das análises por rever

### Query params

| Parâmetro | Tipo | Omissão | Descrição |
|---|---|---|---|
| `page` | int | `1` | página (1-based) |
| `limit` | int | `20` | itens por página (máximo `200`) |

### Regras de negócio

- ordenação por `createdAt` ascendente — a fila é servida pela ordem de chegada

### Sucesso esperado

- `200 OK`
- retorno: `PagedResult<AnalysisResponse>` (`AnalysisStatus = Pending`) — ver [Envelope de paginação](./overview.md#envelope-de-paginacao)

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `400` | vário | falha do handler |
| `401` | — | não autenticado |

## 2.2 Listar análises de um dador

- Caso de uso: `GetDonorAnalysesQuery`
- Endpoint: `GET /api/v1/blood-donation/analyses/donor/{donorId:guid}`
- Finalidade: histórico de análises de um dador

### Query params

| Parâmetro | Tipo | Omissão | Descrição |
|---|---|---|---|
| `page` | int | `1` | página (1-based) |
| `limit` | int | `20` | itens por página (máximo `200`) |

### Regras de negócio

- `{donorId}` é o `Donor.Id`
- devolve as análises do dador independentemente do estado, das mais recentes para as mais antigas

### Exemplo de chamada

```bash
curl http://localhost:5000/api/v1/blood-donation/analyses/donor/f1ac3c5e-6fd2-4a7a-b64e-72f40be690cf \
  -H 'Authorization: Bearer <jwt>'
```

### Sucesso esperado

- `200 OK`
- retorno: `PagedResult<AnalysisResponse>` — ver [Envelope de paginação](./overview.md#envelope-de-paginacao)

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | Donor.NotFound | Donor was not found |
| `401` | — | não autenticado |

## 2.3 Aprovar análise

- Caso de uso: `ApproveAnalysisCommand`
- Endpoint: `POST /api/v1/blood-donation/analyses/{id:guid}/approve`
- Finalidade: validar clinicamente o dador e torná-lo elegível
- Modelos afetados: `DonorAnalysis` **e** `Donor`

### Body

```json
{
  "notes": "Hemograma dentro dos parâmetros."
}
```

`notes` é opcional.

### Regras de negócio

- a análise tem de existir (`Analysis.NotFound`)
- a análise tem de estar em `Pending` (`Analysis.AlreadyReviewed`)
- o dador associado tem de existir (`Donor.NotFound`)
- o dador tem de estar em `PendingAnalysis` (`Analysis.NotPendingAnalysis`)
- efeitos, na mesma transação:
  - `DonorAnalysis.Status` → `Approved`, com `ReviewedBy` = id do utilizador autenticado e `ReviewedAt` = agora
  - `Donor.Status` → `Eligible`
  - `Donor.IsVerified` → `true`
  - `Donor.VerificationNotes` ← `notes`

### Exemplo de chamada

```bash
curl -X POST http://localhost:5000/api/v1/blood-donation/analyses/7c9e6679-7425-40de-944b-e07fc1f90ae7/approve \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <jwt>' \
  -d '{ "notes": "Hemograma dentro dos parametros." }'
```

### Sucesso esperado

- `200 OK`
- retorno: `AnalysisResponse`

```json
{
  "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "donorId": "f1ac3c5e-6fd2-4a7a-b64e-72f40be690cf",
  "reviewedBy": "2b1c3d4e-5f60-4718-8293-a4b5c6d7e8f9",
  "status": "Approved",
  "notes": "Hemograma dentro dos parametros.",
  "rejectionReason": null,
  "isPermanentRejection": false,
  "ineligibleUntil": null,
  "createdAt": "2026-08-14T10:00:00Z",
  "reviewedAt": "2026-08-14T11:30:00Z"
}
```

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Analysis.NotFound` | análise inexistente |
| `409` | `Analysis.AlreadyReviewed` | análise já revista |
| `400` | `Analysis.NotPendingAnalysis` | dador já não está em `PendingAnalysis` |
| `404` | `Donor.NotFound` | dador associado inexistente |
| `401` | — | não autenticado |

## 2.4 Rejeitar análise

- Caso de uso: `RejectAnalysisCommand`
- Endpoint: `POST /api/v1/blood-donation/analyses/{id:guid}/reject`
- Finalidade: recusar a elegibilidade do dador, de forma temporária ou permanente

### Body

```json
{
  "reason": "Hemoglobina abaixo do limite.",
  "isPermanent": false,
  "ineligibleUntil": "2026-10-01T00:00:00Z"
}
```

| Campo | Obrigatório | Notas |
|---|---|---|
| `reason` | sim | motivo da rejeição |
| `isPermanent` | sim | `true` → rejeição definitiva |
| `ineligibleUntil` | em rejeição temporária | data futura até à qual o dador fica inelegível; não se aplica a rejeições permanentes |

### Regras de negócio

- a análise tem de existir e estar em `Pending`
- `reason` é obrigatório (`Analysis.RejectionReasonRequired`)
- numa rejeição temporária, `ineligibleUntil` é **obrigatório** e tem de ser futuro (`Analysis.IneligibleUntilRequired`, `Donor.InvalidAvailabilityDate`)
- numa rejeição permanente, `ineligibleUntil` não se aplica
- efeitos, na mesma transação:
  - `DonorAnalysis.Status` → `RejectedTemporary` ou `RejectedPermanent`
  - `RejectionReason`, `IsPermanentRejection`, `IneligibleUntil`, `ReviewedBy`, `ReviewedAt` preenchidos
  - `Donor.Status` → `TemporarilyIneligible` ou `PermanentlyIneligible`
  - numa rejeição temporária, `Donor.DeferredUntil` ← `ineligibleUntil` com `DeferralReason = ClinicalDeferral`

Depois de o prazo terminar, o dador pode pedir nova análise em `POST /donors/me/reanalysis`. `PermanentlyIneligible` não tem saída.

### Sucesso esperado

- `200 OK`
- retorno: `AnalysisResponse`

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Analysis.NotFound` | análise inexistente |
| `409` | `Analysis.AlreadyReviewed` | análise já revista |
| `400` | `Analysis.RejectionReasonRequired` | `reason` vazio |
| `400` | `Analysis.IneligibleUntilRequired` | rejeição temporária sem `ineligibleUntil` |
| `400` | `Donor.InvalidAvailabilityDate` | `ineligibleUntil` no passado numa rejeição temporária |
| `400` | `Analysis.NotPendingAnalysis` | dador já não está em `PendingAnalysis` |
| `404` | `Donor.NotFound` | dador associado inexistente |
| `401` | — | não autenticado |

## Contrato `AnalysisResponse`

```
Id, DonorId, ReviewedBy, Status, Notes, RejectionReason,
IsPermanentRejection, IneligibleUntil, CreatedAt, ReviewedAt
```

---

## Navegação

← [Dadores](./donors.md) · [Índice do módulo](../index.md) · [Pedidos de Sangue](./requests.md) →

**Neste módulo:** [Visão Geral](./overview.md) · [Modelos e Enumerações](./models-and-enums.md) · [Dadores](./donors.md) · **Análises Clínicas** · [Pedidos de Sangue](./requests.md) · [Matching](./matching.md) · [Dações](./donations.md) · [Moderação](./moderation.md) · [Administração](./admin.md) · [Casos Internos, Eventos e Notas](./internal-and-events.md)
