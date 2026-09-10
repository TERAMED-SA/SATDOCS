# Casos de Uso de Dações

[⌂ Módulo BloodDonation](../index.md) · [Visão Geral](./overview.md)

Grupo de rotas: `/api/v1/blood-donation/donations` · Requer autenticação.

A dação é o registo final do fluxo: só existe a partir de um match `Accepted`.

## 5.1 Confirmar dação

- Caso de uso: `ConfirmDonationCommand`
- Endpoint: `POST /api/v1/blood-donation/donations/confirm`
- Quem chama: **o requerente do pedido** (não o dador)
- Modelos afetados: `Donation` (criada), `BloodRequest`, `Donor`
- Integração: consulta o módulo HospitalManagement via `IHospitalContracts`

### Body

```json
{
  "matchId": "5a4b3c2d-1e0f-4a9b-8c7d-6e5f4a3b2c1d",
  "donationDate": "2026-08-14T09:30:00Z",
  "actualQuantityUnits": 1,
  "hospitalId": "3f1a9c2e-5b6d-4a7e-9c8f-1d2e3a4b5c6d",
  "notes": "Sem intercorrências."
}
```

| Campo | Obrigatório | Notas |
|---|---|---|
| `matchId` | sim | match em estado `Accepted` |
| `donationDate` | sim | não pode ser futura |
| `actualQuantityUnits` | sim | tem de ser > 0 |
| `hospitalId` | **sim, na prática** | apesar de ser `Guid?` no DTO, o handler rejeita `null` com `Hospital.NotFound`; fica gravado na dação |
| `notes` | não | gravado em `Donation.Notes` |

`confirmedByHospital` **já não é aceite** neste body: a confirmação hospitalar é do hospital, e faz-se em `POST /donations/{id}/hospital-confirmation`. Uma dação nasce sempre com `confirmedByHospital = false`.

### Regras de negócio, por ordem de validação

1. o match tem de existir (`Match.NotFound`)
2. o pedido associado tem de existir (`BloodRequest.NotFound`)
3. só o requerente do pedido pode confirmar (`Auth.Forbidden`)
4. o match tem de estar em `Accepted` (`Match.NotAccepted`)
5. não pode existir já uma dação para esse match (`Donation.AlreadyConfirmed`)
6. `donationDate` não pode estar no futuro (`Donation.DonationDateInFuture`)
7. `hospitalId` é obrigatório (`Hospital.NotFound`)
8. o hospital tem de ser elegível para dação, segundo o módulo HospitalManagement
9. o dador tem de existir (`Donor.NotFound`) e poder doar no momento (`Donor.NotEligible`): análise aprovada, sem bloqueio e sem adiamento nem indisponibilidade ativos
10. a compatibilidade ABO/Rh é reconfirmada entre o dador e o pedido (`Match.IncompatibleBloodType`)

Efeitos, na mesma transação:

- cria a `Donation` com `ConfirmedByHospital = false`, guardando `hospitalId` e `notes`
- `BloodRequest.Fulfill()` → o pedido passa a **`Fulfilled`**
- `Donor.TotalDonations` incrementado e `LastDonationDate` ← `donationDate`
- `Donor.DeferredUntil` ← agora + 56 dias, com `DeferralReason = PostDonationCooldown`. O estado guardado do dador continua `Eligible`; enquanto o prazo durar, o `status` devolvido é `TemporarilyIneligible` e o dador não pode voltar a doar

`Fulfill()` aceita `Open` e `Matching` — na prática o pedido está em `Matching`, para lá levado ao criar o primeiro match. Se o pedido estiver num estado que não admite ser satisfeito (cancelado, expirado, já satisfeito), o comando **falha** com `BloodRequest.CannotFulfill` e a dação não é gravada: nada é persistido parcialmente.

### Decisões do hospital

O handler traduz a decisão do módulo HospitalManagement:

| Decisão | Erro devolvido |
|---|---|
| `Allowed` | — prossegue |
| `NotFound` | `Hospital.NotFound` |
| `Inactive` | `Hospital.NotActive` |
| `NoBloodBank` | `Hospital.NoBloodBank` |
| `NotEligibleForDonation` | `Hospital.NotEligibleForDonation` |

### Exemplo de chamada

```bash
curl -X POST http://localhost:5000/api/v1/blood-donation/donations/confirm \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <jwt-do-requerente>' \
  -d '{
    "matchId": "5a4b3c2d-1e0f-4a9b-8c7d-6e5f4a3b2c1d",
    "donationDate": "2026-08-14T09:30:00Z",
    "actualQuantityUnits": 1,
    "hospitalId": "3f1a9c2e-5b6d-4a7e-9c8f-1d2e3a4b5c6d"
  }'
```

### Sucesso esperado

- `201 Created`
- retorno: `DonationResponse`

```json
{
  "id": "c3d4e5f6-a7b8-4c9d-8e0f-1a2b3c4d5e6f",
  "matchId": "5a4b3c2d-1e0f-4a9b-8c7d-6e5f4a3b2c1d",
  "donorProfileId": "f1ac3c5e-6fd2-4a7a-b64e-72f40be690cf",
  "requestId": "8d5e1b2a-3c4f-4a5b-9c6d-7e8f9a0b1c2d",
  "donationDate": "2026-08-14T09:30:00Z",
  "actualQuantityUnits": 1,
  "confirmedByHospital": false,
  "confirmedByUserId": null,
  "hospitalId": "3f1a9c2e-5b6d-4a7e-9c8f-1d2e3a4b5c6d",
  "hospitalConfirmationDate": null,
  "hospitalConfirmationNotes": null,
  "notes": "Sem intercorrências.",
  "createdAt": "2026-08-14T10:05:00Z"
}
```

`hospitalId` é gravado logo na criação, mas `confirmedByHospital` só passa a `true` no passo de confirmação hospitalar (5.2).

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Match.NotFound` | match inexistente |
| `404` | `BloodRequest.NotFound` | pedido associado inexistente |
| `403` | `Auth.Forbidden` | o chamador não é o requerente |
| `400` | `Match.NotAccepted` | match não está em `Accepted` |
| `409` | `Donation.AlreadyConfirmed` | já existe dação para esse match |
| `400` | `Donation.DonationDateInFuture` | data no futuro |
| `400` | `Donation.InvalidQuantityUnits` | unidades ≤ 0 |
| `404` | `Hospital.NotFound` | `hospitalId` ausente ou desconhecido |
| `400` | `Hospital.NotActive` / `Hospital.NoBloodBank` / `Hospital.NotEligibleForDonation` | hospital não elegível |
| `404` | `Donor.NotFound` / `Donor.NotEligible` | dador inexistente ou não elegível |
| `401` | — | não autenticado |

## 5.2 Confirmação hospitalar

- Caso de uso: `AddHospitalConfirmationCommand`
- Endpoint: `POST /api/v1/blood-donation/donations/{id:guid}/hospital-confirmation`
- Finalidade: o hospital atesta que a dação ocorreu

### Body

```json
{
  "confirmed": true,
  "confirmationDate": "2026-08-14T11:00:00Z",
  "confirmationNotes": "Registo interno #4471"
}
```

O hospital não é indicado aqui: a confirmação aplica-se ao hospital já gravado na dação.

### Regras de negócio

- a dação tem de existir (`Donation.NotFound`)
- `confirmed` tem de ser `true`; este endpoint regista uma confirmação (`Donation.ConfirmationNotRequested`)
- a dação tem de ter hospital gravado (`Donation.NoHospitalOnRecord`)
- o hospital tem de continuar a existir no HospitalManagement (`Hospital.NotFound`). Alterações posteriores do estado do hospital — inativo, sem banco de sangue — não impedem o registo, porque a confirmação atesta um facto passado
- a dação não pode já estar confirmada (`Donation.AlreadyConfirmed`)
- efeitos:
  - `ConfirmedByHospital` → `true`
  - `ConfirmedByUserId` ← utilizador autenticado
  - `HospitalConfirmationDate` ← `confirmationDate`, ou agora se omitida. Não pode ser futura (`Donation.ConfirmationDateInFuture`) nem anterior à data da dação (`Donation.ConfirmationBeforeDonation`)
  - `HospitalConfirmationNotes` ← `confirmationNotes`

### Sucesso esperado

- `200 OK`
- retorno: `DonationResponse` atualizado

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Donation.NotFound` | dação inexistente |
| `400` | `Donation.ConfirmationNotRequested` | `confirmed` diferente de `true` |
| `400` | `Donation.NoHospitalOnRecord` | a dação não tem hospital gravado |
| `404` | `Hospital.NotFound` | o hospital da dação já não existe |
| `409` | `Donation.AlreadyConfirmed` | já confirmada anteriormente |
| `400` | `Donation.ConfirmationDateInFuture` | `confirmationDate` no futuro |
| `400` | `Donation.ConfirmationBeforeDonation` | `confirmationDate` anterior à dação |
| `401` | — | não autenticado |

## 5.3 Histórico das minhas dações

- Caso de uso: `GetMyDonationHistoryQuery`
- Endpoint: `GET /api/v1/blood-donation/donations/my-history`

### Query params

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `page` | int | página (1-based), por omissão `1` |
| `limit` | int | itens por página, por omissão `20` (máximo `200`) |

### Regras de negócio

- resolve o perfil de dador a partir do `UserId` do token
- **se o utilizador não tiver perfil de dador, devolve `200 OK` com uma página vazia (`total = 0`)** (não é erro)
- ordenação por `donationDate` descendente

### Sucesso esperado

- `200 OK`
- retorno: `PagedResult<DonationResponse>` — ver [Envelope de paginação](./overview.md#envelope-de-paginacao)

## 5.4 Verificar elegibilidade para dação

- Caso de uso: `GetMyDonorProfileQuery` + `ValidateDonationEligibilityQuery`
- Endpoint: `GET /api/v1/blood-donation/donations/eligibility/check`
- Finalidade: saber se o utilizador autenticado pode doar agora

### Query params

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `hospitalId` | guid | opcional; valida também o hospital de destino |

### Regras de negócio

O endpoint executa dois passos:

1. resolve o perfil de dador do chamador; se não existir, devolve `404` com `Donor.NotFound`
2. avalia a elegibilidade:
   - adiamento imposto ativo → `reason = "Cooldown period active."` (intervalo entre dações) ou `"Temporary clinical deferral in force."`, com `nextEligibleDate` preenchido
   - indisponibilidade declarada ativa → `reason = "Donor marked themselves unavailable."`, com `nextEligibleDate`
   - restantes casos de inelegibilidade → `reason = "Donor status: <estado efetivo>"`
   - se `hospitalId` for indicado e a decisão do HospitalManagement não for `Allowed` → não elegível, com a razão correspondente
   - caso contrário → elegível

A resposta de "não elegível" é um **sucesso** (`200 OK`) com `eligible: false`, não um erro. Esta rota aplica a mesma regra de elegibilidade dos caminhos de escrita; o intervalo mínimo entre dações é de 56 dias.

### Exemplo de chamada

```bash
curl 'http://localhost:5000/api/v1/blood-donation/donations/eligibility/check?hospitalId=3f1a9c2e-5b6d-4a7e-9c8f-1d2e3a4b5c6d' \
  -H 'Authorization: Bearer <jwt>'
```

### Sucesso esperado

- `200 OK`
- retorno: `EligibilityResponse`

```json
{
  "eligible": false,
  "reason": "Cooldown period active.",
  "nextEligibleDate": "2026-10-09T09:30:00Z"
}
```

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Donor.NotFound` | o chamador não tem perfil de dador |
| `400` | vário | falha na avaliação |
| `401` | — | não autenticado |

## Contratos

### `DonationResponse`

```
Id, MatchId, DonorProfileId, RequestId, DonationDate, ActualQuantityUnits,
ConfirmedByHospital, HospitalId, HospitalConfirmationDate, ConfirmedByUserId,
HospitalConfirmationNotes, Notes, CreatedAt
```

### `EligibilityResponse`

```
Eligible, Reason, NextEligibleDate
```

---

## Navegação

← [Matching](./matching.md) · [Índice do módulo](../index.md) · [Moderação](./moderation.md) →

**Neste módulo:** [Visão Geral](./overview.md) · [Modelos e Enumerações](./models-and-enums.md) · [Dadores](./donors.md) · [Análises Clínicas](./analysis.md) · [Pedidos de Sangue](./requests.md) · [Matching](./matching.md) · **Dações** · [Moderação](./moderation.md) · [Administração](./admin.md) · [Casos Internos, Eventos e Notas](./internal-and-events.md)
