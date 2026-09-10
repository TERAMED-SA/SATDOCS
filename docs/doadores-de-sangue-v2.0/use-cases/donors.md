# Casos de Uso de Dadores

[⌂ Módulo BloodDonation](../index.md) · [Visão Geral](./overview.md)

Grupo de rotas: `/api/v1/blood-donation/donors` · Requer autenticação.

## 1.1 Registar-se como dador

- Caso de uso: `CreateDonorCommand` / `CreateDonorCommandHandler`
- Endpoint: `POST /api/v1/blood-donation/donors`
- Finalidade: criar o perfil de dador associado ao utilizador autenticado
- Modelos principais: `Donor`, `DonorAnalysis`

### Regras de negócio

- um utilizador só pode ter um perfil de dador (`Donor.AlreadyExists`)
- o estado inicial é **`PendingAnalysis`** — o dador **não** fica logo elegível
- é criada automaticamente uma `DonorAnalysis` em estado `Pending` para o novo dador
- os campos opcionais só são aplicados se pelo menos um deles vier preenchido
- se `hasMedicalRestrictions = true`, `medicalRestrictionsDetails` é obrigatório (`Donor.InvalidMedicalRestrictions`)
- `preferredHospitalId` é um `Guid`; um valor malformado é recusado com `400` na desserialização

### Body

```json
{
  "bloodType": "O_Positive",
  "donorMode": "Regular",
  "emergencyContact": "Maria Silva - +244 923 456 789",
  "preferredHospitalId": "3f1a9c2e-5b6d-4a7e-9c8f-1d2e3a4b5c6d",
  "hasMedicalRestrictions": false,
  "medicalRestrictionsDetails": null
}
```

Apenas `bloodType` é obrigatório. `donorMode` assume `Regular` por omissão.

### Exemplo de chamada

```bash
curl -X POST http://localhost:5000/api/v1/blood-donation/donors \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <jwt>' \
  -d '{
    "bloodType": "O_Positive",
    "donorMode": "Regular",
    "emergencyContact": "Maria Silva - +244 923 456 789"
  }'
```

### Sucesso esperado

- `201 Created`, com `Location` a apontar para `GET /api/v1/blood-donation/donors/{id}`
- retorno: `DonorResponse`

```json
{
  "id": "f1ac3c5e-6fd2-4a7a-b64e-72f40be690cf",
  "userId": "9a7b6c5d-4e3f-2a1b-0c9d-8e7f6a5b4c3d",
  "bloodType": "O_Positive",
  "donorMode": "Regular",
  "status": "PendingAnalysis",
  "emergencyContact": "Maria Silva - +244 923 456 789",
  "preferredHospitalId": null,
  "unavailableUntil": null,
  "availableUntil": null,
  "deferredUntil": null,
  "deferralReason": null,
  "blockedUntil": null,
  "isPermanentlyBlocked": false,
  "lastDonationDate": null,
  "hasMedicalRestrictions": false,
  "medicalRestrictionsDetails": null,
  "bloodTypeCorrectedAt": null,
  "bloodTypeCorrectionReason": null,
  "isVerified": false,
  "totalDonations": 0,
  "createdAt": "2026-08-14T10:00:00Z"
}
```

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `409` | `Donor.AlreadyExists` | o utilizador já tem perfil de dador |
| `400` | `Donor.InvalidUserId` | `UserId` vazio |
| `400` | `Donor.InvalidMedicalRestrictions` | restrições marcadas sem detalhes |
| `401` | — | claim `NameIdentifier` ausente ou não é `Guid` |

## 1.2 Obter perfil de dador por id

- Caso de uso: `GetDonorProfileQuery`
- Endpoint: `GET /api/v1/blood-donation/donors/{id:guid}`
- Finalidade: consultar um perfil de dador pelo `Donor.Id`

### Regras de negócio

- qualquer utilizador autenticado pode consultar um perfil pelo seu id
- o `{id}` é o `Donor.Id`, não o `UserId`

### Sucesso esperado

- `200 OK`
- retorno: `DonorResponse`

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Donor.NotFound` | perfil inexistente |
| `401` | — | não autenticado |

## 1.3 Obter o meu perfil de dador

- Caso de uso: `GetMyDonorProfileQuery`
- Endpoint: `GET /api/v1/blood-donation/donors/me`
- Finalidade: recuperar o perfil do dador do utilizador autenticado

### Regras de negócio

- procura por `UserId` extraído do token
- devolve erro se o utilizador ainda não tiver criado o perfil

### Exemplo de chamada

```bash
curl http://localhost:5000/api/v1/blood-donation/donors/me \
  -H 'Authorization: Bearer <jwt>'
```

### Sucesso esperado

- `200 OK`
- retorno: `DonorResponse`

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Donor.NotFound` | perfil ainda não criado |
| `401` | — | não autenticado |

## 1.4 Atualizar o meu perfil de dador

- Caso de uso: `UpdateMyDonorProfileCommand`
- Endpoint: `PATCH /api/v1/blood-donation/donors/me`
- Finalidade: atualizar campos editáveis do perfil

### Body

Todos os campos são opcionais; apenas os enviados são alterados.

```json
{
  "donorMode": "Occasional",
  "emergencyContact": "João - +244 912 000 111",
  "preferredHospitalId": "3f1a9c2e-5b6d-4a7e-9c8f-1d2e3a4b5c6d",
  "hasMedicalRestrictions": true,
  "medicalRestrictionsDetails": "Não pode doar em jejum"
}
```

### Regras de negócio

- o perfil tem de existir
- a edição do perfil é permitida em qualquer estado do dador
- a flag e o detalhe das restrições médicas formam um par:
  - `hasMedicalRestrictions = true` exige detalhe, enviado agora ou já guardado (`Donor.InvalidMedicalRestrictions`)
  - `hasMedicalRestrictions = false` limpa o detalhe; enviar detalhe no mesmo pedido é recusado (`Donor.MedicalRestrictionsWithoutFlag`)
  - enviar só o detalhe com as restrições desligadas é recusado (`Donor.MedicalRestrictionsWithoutFlag`)
- `preferredHospitalId` é um `Guid`; um valor malformado é recusado com `400` na desserialização
- o tipo sanguíneo **não** é editável por esta via; corrige-se em `PATCH /donors/me/blood-type`

### Sucesso esperado

- `200 OK`
- retorno: `DonorResponse` atualizado

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Donor.NotFound` | perfil não encontrado |
| `400` | `Donor.InvalidMedicalRestrictions` | restrições marcadas sem detalhe |
| `400` | `Donor.MedicalRestrictionsWithoutFlag` | detalhe enviado com as restrições desligadas |
| `401` | — | não autenticado |

## 1.5 Atualizar a minha disponibilidade

- Caso de uso: `UpdateDonorAvailabilityCommand`
- Endpoint: `PATCH /api/v1/blood-donation/donors/me/availability`
- Finalidade: marcar-se como disponível ou temporariamente indisponível

### Body

```json
{
  "isAvailable": false,
  "unavailableUntil": "2026-09-01T12:00:00Z"
}
```

| Campo | Estado | Significado |
|---|---|---|
| `isAvailable` | atual | `true` repõe o dador como disponível; `false` marca-o como indisponível |
| `unavailableUntil` | atual | data até à qual fica indisponível |
| `availableUntil` | **obsoleto** | nome anterior de `unavailableUntil`; continua a ser aceite |
| `temporaryUnavailable` | **obsoleto** | equivalente a `isAvailable = false`; continua a ser aceite |

Todos os campos são opcionais. Quando `unavailableUntil` e `availableUntil` vêm ambos preenchidos, prevalece `unavailableUntil`.

### Regras de negócio

1. `isAvailable = true` → limpa a indisponibilidade declarada
2. `isAvailable = false` **ou** `temporaryUnavailable = true`:
   - com data → marca indisponível até essa data
   - sem data → limpa a indisponibilidade declarada
3. apenas a data preenchida → marca indisponível até essa data
4. nenhum campo relevante → devolve o estado atual sem gravar

Para marcar indisponibilidade, envie sempre a data.

- este endpoint atua **apenas** sobre a indisponibilidade declarada pelo dador. Um adiamento imposto — rejeição clínica temporária ou intervalo entre dações — não é afetado e continua a valer até ao seu próprio prazo
- a operação é recusada em `PendingAnalysis`, `TemporarilyIneligible`, `PermanentlyIneligible` e durante um bloqueio (`Donor.InvalidStatusTransition`)
- uma data no passado ou igual a agora é recusada (`Donor.InvalidAvailabilityDate`)
- o `status` devolvido reflete o estado efetivo: com `unavailableUntil` no futuro, vem `TemporarilyIneligible`

O nome preferido é `unavailableUntil` porque é isso que o campo exprime: o fim da janela de **in**disponibilidade, não o fim da disponibilidade.

### Sucesso esperado

- `200 OK`
- retorno: `DonorResponse` atualizado

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Donor.NotFound` | perfil não encontrado |
| `400` | `Donor.InvalidStatusTransition` | dador em `PendingAnalysis`, `TemporarilyIneligible`, `PermanentlyIneligible` ou bloqueado |
| `400` | `Donor.InvalidAvailabilityDate` | data no passado |
| `401` | — | não autenticado |

## 1.6 Corrigir o tipo sanguíneo

- Caso de uso: `CorrectBloodTypeCommand`
- Endpoint: `PATCH /api/v1/blood-donation/donors/me/blood-type`
- Finalidade: corrigir o tipo sanguíneo registado no perfil

### Body

```json
{
  "bloodType": "O_Negative",
  "reason": "Resultado laboratorial de 12/08"
}
```

Ambos os campos são obrigatórios.

### Regras de negócio

O tipo sanguíneo é auto-declarado no registo. Corrigi-lo não é uma edição de perfil: invalida tudo o que foi decidido com base no valor anterior.

- `reason` é obrigatório (`Donor.BloodTypeCorrectionReasonRequired`) e fica gravado no perfil
- enviar o **mesmo** tipo não produz efeito nenhum
- numa alteração real:
  - todos os matches `Pending` do dador são rejeitados e os `Accepted` retirados, com a razão `"Blood type corrected."`; os pedidos afetados voltam a `Open`
  - um dador `Eligible` volta a `PendingAnalysis`, perde `IsVerified` e é aberta nova análise clínica
  - `TemporarilyIneligible`, `PermanentlyIneligible` e bloqueios em curso **mantêm-se**: a correção não levanta decisões clínicas nem sanções

### Sucesso esperado

- `200 OK`
- retorno: `DonorResponse` atualizado

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Donor.NotFound` | o utilizador não tem perfil de dador |
| `400` | `Donor.BloodTypeCorrectionReasonRequired` | `reason` vazio |
| `401` | — | não autenticado |

## 1.7 Pedir nova análise clínica

- Caso de uso: `RequestReanalysisCommand`
- Endpoint: `POST /api/v1/blood-donation/donors/me/reanalysis`
- Finalidade: reabrir o rastreio clínico depois de uma rejeição temporária ter terminado
- Modelos principais: `Donor`, `DonorAnalysis`

### Regras de negócio

- o dador tem de estar em `TemporarilyIneligible` por **rejeição clínica**: a análise mais recente tem de ser `RejectedTemporary` (`Analysis.CannotReopen`)
- o prazo da rejeição tem de ter terminado (`Analysis.DeferralStillActive`)
- não pode existir já uma análise pendente (`Analysis.AlreadyPending`)
- não é permitido durante um bloqueio de moderação
- uma rejeição permanente não é reaberta por esta via
- o efeito é abrir uma `DonorAnalysis` em `Pending` e devolver o dador a `PendingAnalysis`; a elegibilidade volta a depender da revisão

### Exemplo de chamada

```bash
curl -X POST http://localhost:5000/api/v1/blood-donation/donors/me/reanalysis \
  -H 'Authorization: Bearer <jwt>'
```

### Sucesso esperado

- `201 Created`
- retorno: `AnalysisResponse` da análise criada, em `Pending`

### Erros esperados

| HTTP | Code | Situação |
|---|---|---|
| `404` | `Donor.NotFound` | o utilizador não tem perfil de dador |
| `400` | `Analysis.CannotReopen` | estado não elegível para reabertura |
| `400` | `Analysis.DeferralStillActive` | o prazo da rejeição ainda não terminou |
| `409` | `Analysis.AlreadyPending` | já existe uma análise por rever |
| `401` | — | não autenticado |

## Contrato `DonorResponse`

```
Id, UserId, BloodType, DonorMode, Status, EmergencyContact,
PreferredHospitalId, UnavailableUntil, AvailableUntil (obsoleto, mesmo valor),
DeferredUntil, DeferralReason, BlockedUntil, IsPermanentlyBlocked,
LastDonationDate,
HasMedicalRestrictions, MedicalRestrictionsDetails,
BloodTypeCorrectedAt, BloodTypeCorrectionReason,
IsVerified, TotalDonations, CreatedAt
```

`Status` é o estado efetivo no momento do pedido — ver [Estado guardado e estado devolvido](./models-and-enums.md#estado-guardado-e-estado-devolvido).

Campos persistidos mas **não** expostos nesta resposta: `VerificationNotes`, `BlockReason`, `BlockNotes`, `UpdatedAt`.

---

## Navegação

← [Modelos e Enumerações](./models-and-enums.md) · [Índice do módulo](../index.md) · [Análises Clínicas](./analysis.md) →

**Neste módulo:** [Visão Geral](./overview.md) · [Modelos e Enumerações](./models-and-enums.md) · **Dadores** · [Análises Clínicas](./analysis.md) · [Pedidos de Sangue](./requests.md) · [Matching](./matching.md) · [Dações](./donations.md) · [Moderação](./moderation.md) · [Administração](./admin.md) · [Casos Internos, Eventos e Notas](./internal-and-events.md)
