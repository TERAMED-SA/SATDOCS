# Casos de Uso de Hospitais

[ homepage Modulo HospitalManagement](../index.md) · [Visao Geral](./overview.md)

Grupo de rotas: `/api/v1/hospital-management/hospitals` · Requer autenticacao.

## 1.1 Criar hospital

- Caso de uso: `CreateHospitalCommand`
- Endpoint: `POST /api/v1/hospital-management/hospitals`
- Finalidade: registar um novo hospital no sistema
- Modelo principal: `Hospital`

### Regras de negocio

- `legalName` e obrigatorio (`Hospital.LegalNameRequired`)
- `commercialName` e obrigatorio (`Hospital.CommercialNameRequired`)
- `country` e obrigatorio (`Hospital.CountryRequired`)
- nao pode existir outro hospital com o mesmo NIF ou Legal Name (`Hospital.AlreadyExists`)
- o estado inicial e `Active`
- `emergencyEnabled`, `intensiveCare`, `hasBloodBank`, `eligibleForDonation` assumem `false` por omissao

### Body

```json
{
  "legalName": "Hospital Central de Luanda",
  "commercialName": "HCL",
  "hospitalType": "Public",
  "hospitalKind": "General",
  "country": "AO",
  "nif": "5410001234",
  "email": "contact@hcl.co.ao",
  "phone": "+244923456789",
  "website": "https://hcl.co.ao",
  "province": "Luanda",
  "city": "Luanda",
  "address": "Av. 4 de Fevereiro, 100",
  "openingDate": "2010-01-15T00:00:00Z",
  "emergencyEnabled": true,
  "intensiveCare": true,
  "hasBloodBank": true,
  "eligibleForDonation": true
}
```

Apenas `legalName`, `commercialName`, `hospitalType`, `hospitalKind` e `country` sao obrigatórios. Os restantes campos sao opcionais.

### Exemplo de chamada

```bash
curl -X POST http://localhost:5000/api/v1/hospital-management/hospitals \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <jwt>' \
  -d '{
    "legalName": "Hospital Central de Luanda",
    "commercialName": "HCL",
    "hospitalType": "Public",
    "hospitalKind": "General",
    "country": "AO"
  }'
```

### Sucesso esperado

- `201 Created`
- retorno: `HospitalResponse`

```json
{
  "id": "f1ac3c5e-6fd2-4a7a-b64e-72f40be690cf",
  "legalName": "Hospital Central de Luanda",
  "commercialName": "HCL",
  "hospitalType": "Public",
  "hospitalKind": "General",
  "status": "Active",
  "country": "AO",
  "province": null,
  "city": null,
  "email": null,
  "phone": null,
  "emergencyEnabled": false,
  "intensiveCare": false,
  "hasBloodBank": false,
  "eligibleForDonation": false,
  "createdAt": "2026-08-18T12:00:00Z",
  "updatedAt": "2026-08-18T12:00:00Z"
}
```

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `400` | `Hospital.LegalNameRequired` | legalName vazio |
| `400` | `Hospital.CommercialNameRequired` | commercialName vazio |
| `400` | `Hospital.CountryRequired` | country vazio |
| `400` | `Hospital.AlreadyExists` | hospital com mesmo NIF ou Legal Name |

## 1.2 Listar hospitais

- Caso de uso: `GetHospitalsQuery`
- Endpoint: `GET /api/v1/hospital-management/hospitals`
- Finalidade: obter todos os hospitais registados

### Sucesso esperado

- `200 OK`
- retorno: array de `HospitalResponse`

## 1.3 Obter hospital por id

- Caso de uso: `GetHospitalByIdQuery`
- Endpoint: `GET /api/v1/hospital-management/hospitals/{id:guid}`

### Sucesso esperado

- `200 OK`
- retorno: `HospitalResponse`

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `404` | `Hospital.NotFound` | hospital inexistente |

## 1.4 Atualizar hospital

- Caso de uso: `UpdateHospitalCommand`
- Endpoint: `PUT /api/v1/hospital-management/hospitals/{id:guid}`

### Body

Mesmo shape do `POST /hospitals`.

### Regras de negocio

- o hospital tem de existir
- hospitais em estado `Inactive` ou `Suspended` sao read-only (`Hospital.ReadOnly`)

### Sucesso esperado

- `200 OK`
- retorno: `HospitalResponse` atualizado

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `400` | `Hospital.ReadOnly` | hospital inativo ou suspenso |
| `400` | `Hospital.LegalNameRequired` | legalName vazio |
| `400` | `Hospital.CommercialNameRequired` | commercialName vazio |
| `400` | `Hospital.CountryRequired` | country vazio |

## 1.5 Eliminar hospital

- Caso de uso: `DeleteHospitalCommand`
- Endpoint: `DELETE /api/v1/hospital-management/hospitals/{id:guid}`
- Finalidade: soft delete (define `Status = Inactive` e preenche `DeletedAt`)

### Sucesso esperado

- `204 No Content`

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `400` | `Hospital.NotFound` | hospital inexistente |

## Contrato `HospitalResponse`

```
Id, LegalName, CommercialName, HospitalType, HospitalKind, Status,
Country, Province, City, Email, Phone,
EmergencyEnabled, IntensiveCare, HasBloodBank, EligibleForDonation,
CreatedAt, UpdatedAt
```

---

## Navegacao

[ homepage Modelos e Enumeracoes](./models-and-enums.md) · [Indice do modulo](../index.md) · [Unidades](./units.md)

**Neste modulo:** [Visao Geral](./overview.md) · [Modelos e Enumeracoes](./models-and-enums.md) · **Hospitais** · [Unidades](./units.md) · [Departamentos](./departments.md) · [Estrutura Fisica](./physical-structure.md) · [Ofertas](./offers.md) · [Contas](./accounts.md) · [Seguranca](./security.md) · [Verificacoes](./checks.md) · [Casos Internos e Notas](./internal-and-events.md)
