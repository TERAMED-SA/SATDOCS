# Casos de Uso de Matching

[⌂ Módulo BloodDonation](../index.md) · [Visão Geral](./overview.md)

Grupo de rotas: `/api/v1/blood-donation` (rotas de `requests` e `matches` misturadas) · Requer autenticação.

O match é a proposta de um dador para responder a um pedido concreto. O fluxo típico:

```
dador propõe match           requerente aceita           requerente confirma dação
      │                            │                              │
   Pending  ──────────────────> Accepted  ────────────────────> Donation
      │                            │
      ├── Rejected                 └── Withdrawn (pelo dador)
      └── Withdrawn
```

## 4.1 Propor match a um pedido

- Caso de uso: `CreateMatchCommand`
- Endpoint: `POST /api/v1/blood-donation/requests/{id:guid}/match`
- Finalidade: um dador oferece-se para um pedido específico
- Quem chama: o **dador** (o perfil é resolvido a partir do `UserId` do token)

### Body

```json
{ "proposedDonationDate": "2026-08-20T09:00:00Z" }
```

`proposedDonationDate` é opcional.

### Regras de negócio, por ordem de validação

1. o pedido tem de existir (`BloodRequest.NotFound`)
2. o pedido tem de estar em `Open` ou `Matching` (`BloodRequest.NotOpen`)
3. o pedido não pode estar expirado por data (`BloodRequest.Expired`)
4. o chamador tem de ter perfil de dador (`Donor.NotFound`)
5. o dador tem de poder doar no momento (`Donor.NotEligible`): análise aprovada, sem bloqueio e sem adiamento nem indisponibilidade ativos
6. o `BloodType` do dador tem de ser compatível com o do pedido (`Match.IncompatibleBloodType`)
7. o dador não pode ter outro match ativo (`Match.DonorHasActiveMatch`)
8. não pode existir já um match deste dador para este pedido (`Match.DuplicateMatch`)
9. `proposedDonationDate`, se fornecida, tem de ser futura (`Match.InvalidProposedDate`) e é gravada no match

Efeitos: cria o `Match` em `Pending` e, se o pedido estava em `Open`, transita-o para `Matching`.

### Compatibilidade ABO/Rh

A verificação é direcional — quem pode doar para quem — e segue a tabela de concentrado eritrocitário:

| Pedido (recetor) | Dadores aceites |
|---|---|
| `O_Negative` | O− |
| `O_Positive` | O−, O+ |
| `A_Negative` | O−, A− |
| `A_Positive` | O−, O+, A−, A+ |
| `B_Negative` | O−, B− |
| `B_Positive` | O−, O+, B−, B+ |
| `AB_Negative` | O−, A−, B−, AB− |
| `AB_Positive` | todos |

Implementada em `BloodCompatibility` (`Domain/Services`), aplicada apenas na criação do match.

`MatchScore` está reservado para uma pontuação futura e vem sempre `null`.

### Exemplo de chamada

```bash
curl -X POST http://localhost:5000/api/v1/blood-donation/requests/8d5e1b2a-3c4f-4a5b-9c6d-7e8f9a0b1c2d/match \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <jwt-do-dador>' \
  -d '{ "proposedDonationDate": "2026-08-20T09:00:00Z" }'
```

### Sucesso esperado

- `201 Created`
- retorno: `MatchResponse`

```json
{
  "id": "5a4b3c2d-1e0f-4a9b-8c7d-6e5f4a3b2c1d",
  "requestId": "8d5e1b2a-3c4f-4a5b-9c6d-7e8f9a0b1c2d",
  "donorProfileId": "f1ac3c5e-6fd2-4a7a-b64e-72f40be690cf",
  "status": "Pending",
  "matchScore": null,
  "proposedDonationDate": "2026-08-20T09:00:00Z",
  "acceptedAt": null,
  "rejectedAt": null,
  "withdrawnAt": null,
  "rejectionReason": null,
  "withdrawalReason": null,
  "createdAt": "2026-08-14T12:00:00Z"
}
```

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `BloodRequest.NotFound` | pedido inexistente |
| `400` | `BloodRequest.NotOpen` | pedido fechado, cancelado ou satisfeito |
| `400` | `BloodRequest.Expired` | pedido ultrapassou `expiresAt` |
| `404` | `Donor.NotFound` | o chamador não tem perfil de dador |
| `400` | `Donor.NotEligible` | dador não pode doar no momento |
| `400` | `Match.IncompatibleBloodType` | grupo do dador incompatível com o do pedido |
| `409` | `Match.DonorHasActiveMatch` | dador já tem um match ativo noutro pedido |
| `409` | `Match.DuplicateMatch` | já existe match deste dador para este pedido |
| `400` | `Match.InvalidProposedDate` | data proposta no passado |
| `401` | — | não autenticado |

## 4.2 Aceitar match

- Caso de uso: `AcceptMatchCommand`
- Endpoint: `POST /api/v1/blood-donation/matches/{id:guid}/accept`
- Quem chama: **apenas o requerente do pedido**
- Sem body

### Regras de negócio

1. o match tem de existir (`Match.NotFound`)
2. o pedido associado tem de existir (`BloodRequest.NotFound`)
3. só o requerente pode aceitar (`Auth.Forbidden`)
4. o match tem de estar em `Pending` (`Match.CannotAccept`)
5. o dador tem de continuar a poder doar (`Donor.NotActive`)
6. o dador não pode já ter um match aceite noutro sítio (`Match.DonorHasAcceptedMatch`)

Efeitos:

- o match passa a `Accepted`, com `AcceptedAt`
- **todos os outros matches `Pending` do mesmo pedido são automaticamente rejeitados** com a razão `"Another match was accepted."`

### Sucesso esperado

- `200 OK`
- retorno: `MatchResponse` com `status = "Accepted"`

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Match.NotFound` | match inexistente |
| `404` | `BloodRequest.NotFound` | pedido associado inexistente |
| `403` | `Auth.Forbidden` | o chamador não é o requerente |
| `400` | `Match.CannotAccept` | match não está em `Pending` |
| `400` | `Donor.NotActive` | dador deixou de estar elegível |
| `409` | `Match.DonorHasAcceptedMatch` | dador já tem match aceite |
| `401` | — | não autenticado |

## 4.3 Rejeitar match

- Caso de uso: `RejectMatchCommand`
- Endpoint: `POST /api/v1/blood-donation/matches/{id:guid}/reject`
- Quem chama: **requerente ou dador** do match

### Body

```json
{ "reason": "Indisponível na data proposta." }
```

`reason` é opcional; o body é esperado pelo binding.

### Regras de negócio

- o match tem de existir
- o chamador tem de ser o requerente do pedido ou o dador do match (`Auth.Forbidden`)
- o match tem de estar em `Pending` (`Match.CannotReject`)
- se o pedido estava em `Matching` e deixou de haver matches pendentes, o pedido volta a `Open`

### Sucesso esperado

- `200 OK`
- retorno: `MatchResponse` com `status = "Rejected"`, `rejectedAt` e `rejectionReason` preenchidos

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Match.NotFound` | match inexistente |
| `403` | `Auth.Forbidden` | chamador não é parte do match |
| `400` | `Match.CannotReject` | match não está em `Pending` |
| `401` | — | não autenticado |

## 4.4 Desistir do match

- Caso de uso: `WithdrawMatchCommand`
- Endpoint: `POST /api/v1/blood-donation/matches/{id:guid}/withdraw`
- Quem chama: **apenas o dador**

### Body

```json
{ "reason": "Motivo de saúde." }
```

### Regras de negócio

- o match tem de existir
- só o dador do match pode desistir (`Auth.Forbidden`)
- o match tem de estar em `Pending` **ou** `Accepted` (`Match.CannotWithdraw`)
- se o match estava `Accepted`, o pedido associado volta a `Open` (`RestoreToOpen`)

Esta é a operação a usar antes de cancelar um pedido que tenha matches aceites (ver [Pedidos](./requests.md#35-cancelar-pedido)).

### Sucesso esperado

- `200 OK`
- retorno: `MatchResponse` com `status = "Withdrawn"`

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Match.NotFound` | match inexistente |
| `403` | `Auth.Forbidden` | o chamador não é o dador |
| `400` | `Match.CannotWithdraw` | match em `Rejected` ou `Withdrawn` |
| `401` | — | não autenticado |

## 4.5 Listar os meus matches

- Caso de uso: `GetMyMatchesQuery`
- Endpoint: `GET /api/v1/blood-donation/matches/my`

### Query params

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `status` | string | filtra por `MatchStatus`; valor inválido é ignorado |
| `asDonor` | bool | incluir matches em que sou o dador |
| `asRequester` | bool | incluir matches em que sou o requerente |
| `page` | int | página (1-based), por omissão `1` |
| `limit` | int | itens por página, por omissão `20` (máximo `200`) |

### Regras de negócio

- os dois lados (dador e requerente) são resolvidos numa única consulta, pelo que um match alcançável pelos dois lados aparece — e é contado — uma só vez
- `asDonor` por omissão segue a existência de perfil de dador; `asRequester` por omissão é `true`
- com ambos os lados excluídos, a resposta é uma página vazia (`total = 0`), sem consulta à base de dados
- ordenação por `createdAt` descendente

### Exemplo de chamada

```bash
curl 'http://localhost:5000/api/v1/blood-donation/matches/my?status=Pending&asDonor=true&page=1&limit=20' \
  -H 'Authorization: Bearer <jwt>'
```

### Sucesso esperado

- `200 OK`
- retorno: `PagedResult<MatchResponse>` — ver [Envelope de paginação](./overview.md#envelope-de-paginacao)

## 4.6 Listar matches de um pedido

- Caso de uso: `GetRequestMatchesQuery`
- Endpoint: `GET /api/v1/blood-donation/requests/{id:guid}/matches`
- Quem chama: **apenas o dono do pedido**

### Query params

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `page` | int | página (1-based), por omissão `1` |
| `limit` | int | itens por página, por omissão `20` (máximo `200`) |

### Regras de negócio

1. o pedido tem de existir (`BloodRequest.NotFound`)
2. só o requerente pode listar os matches do seu pedido (`Auth.Forbidden`)
3. ordenação por `createdAt` descendente

### Sucesso esperado

- `200 OK`
- retorno: `PagedResult<MatchResponse>` (todos os estados) — ver [Envelope de paginação](./overview.md#envelope-de-paginacao)

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `BloodRequest.NotFound` | pedido inexistente |
| `403` | `Auth.Forbidden` | o chamador não é o requerente |
| `401` | — | não autenticado |

## Limpeza automática

Matches `Pending` associados a pedidos expirados são rejeitados automaticamente de 30 em 30 minutos pelo `CleanupMatchesBackgroundService`. Ver [Casos Internos e Eventos](./internal-and-events.md).

## Contrato `MatchResponse`

```
Id, RequestId, DonorProfileId, Status, MatchScore, ProposedDonationDate,
AcceptedAt, RejectedAt, WithdrawnAt, RejectionReason, WithdrawalReason, CreatedAt
```

---

## Navegação

← [Pedidos de Sangue](./requests.md) · [Índice do módulo](../index.md) · [Dações](./donations.md) →

**Neste módulo:** [Visão Geral](./overview.md) · [Modelos e Enumerações](./models-and-enums.md) · [Dadores](./donors.md) · [Análises Clínicas](./analysis.md) · [Pedidos de Sangue](./requests.md) · **Matching** · [Dações](./donations.md) · [Moderação](./moderation.md) · [Administração](./admin.md) · [Casos Internos, Eventos e Notas](./internal-and-events.md)
