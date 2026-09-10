# Casos de Uso de Verificacoes

[ homepage Modulo HospitalManagement](../index.md) · [Visao Geral](./overview.md)

Grupo de rotas: `/api/v1/hospital-management/hospitals/{hospitalId}/checks` · Requer autenticacao.

Estes endpoints sao consultas de validacao usadas por outros modulos (ex.: BloodDonation) para verificar o estado de um hospital.

## 8.1 Verificar banco de sangue

- Caso de uso: `GetBloodBankCheckQuery`
- Endpoint: `GET /api/v1/hospital-management/hospitals/{hospitalId:guid}/checks/blood-bank`
- Finalidade: saber se o hospital possui banco de sangue

### Sucesso esperado

- `200 OK`

```json
{
  "hospitalId": "f1ac3c5e-6fd2-4a7a-b64e-72f40be690cf",
  "hasBloodBank": true
}
```

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `404` | `Hospital.NotFound` | hospital inexistente |

## 8.2 Verificar elegibilidade para dacao

- Caso de uso: `GetDonationEligibilityCheckQuery`
- Endpoint: `GET /api/v1/hospital-management/hospitals/{hospitalId:guid}/checks/donation-eligibility`
- Finalidade: saber se o hospital e elegivel para receber doacoes de sangue

### Sucesso esperado

- `200 OK`

```json
{
  "hospitalId": "f1ac3c5e-6fd2-4a7a-b64e-72f40be690cf",
  "eligibleForDonation": true
}
```

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `404` | `Hospital.NotFound` | hospital inexistente |

## 8.3 Verificar compatibilidade de localizacao

- Caso de uso: `GetLocationCompatibilityQuery`
- Endpoint: `GET /api/v1/hospital-management/hospitals/{hospitalId:guid}/checks/location-compatibility`
- Finalidade: verificar se a localizacao do hospital e compativel

### Query params

| Parametro | Tipo | Obrigatorio | Descricao |
|---|---|---|---|
| `country` | string | sim | Codigo do pais |
| `province` | string | nao | Provincia |
| `city` | string | nao | Cidade |

### Sucesso esperado

- `200 OK`

```json
{
  "hospitalId": "f1ac3c5e-6fd2-4a7a-b64e-72f40be690cf",
  "compatible": true
}
```

## 8.4 Verificar especialidade

- Caso de uso: `GetHasSpecialtyCheckQuery`
- Endpoint: `GET /api/v1/hospital-management/hospitals/{hospitalId:guid}/checks/has-specialty/{specialtyId}`
- Finalidade: saber se o hospital oferece uma especialidade especifica

### Sucesso esperado

- `200 OK`

```json
{
  "hospitalId": "f1ac3c5e-6fd2-4a7a-b64e-72f40be690cf",
  "specialtyId": "uuid-string",
  "hasSpecialty": true
}
```

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `400` | — | formato de specialtyId invalido |

## 8.5 Verificar item clinico

- Caso de uso: `GetOffersClinicalItemCheckQuery`
- Endpoint: `GET /api/v1/hospital-management/hospitals/{hospitalId:guid}/checks/offers-clinical-item/{clinicalItemId}`
- Finalidade: saber se o hospital oferece um item clinico especifico

### Sucesso esperado

- `200 OK`

```json
{
  "hospitalId": "f1ac3c5e-6fd2-4a7a-b64e-72f40be690cf",
  "clinicalItemId": "uuid-string",
  "offersClinicalItem": true
}
```

---

## Navegacao

[ homepage Seguranca](./security.md) · [Indice do modulo](../index.md) · [Casos Internos e Notas](./internal-and-events.md)

**Neste modulo:** [Visao Geral](./overview.md) · [Modelos e Enumeracoes](./models-and-enums.md) · [Hospitais](./hospitals.md) · [Unidades](./units.md) · [Departamentos](./departments.md) · [Estrutura Fisica](./physical-structure.md) · [Ofertas](./offers.md) · [Contas](./accounts.md) · [Seguranca](./security.md) · **Verificacoes** · [Casos Internos e Notas](./internal-and-events.md)
