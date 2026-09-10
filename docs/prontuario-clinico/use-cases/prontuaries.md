# Casos de Uso de Prontuarios

[ homepage Modulo ClinicalProntuary](../index.md) · [Visao Geral](./overview.md)

Grupo de rotas: `/prontuaries` · Requer autenticacao.

## 1.1 Criar prontuario

- Caso de uso: `CreateProntuaryCommand`
- Endpoint: `POST /api/v1/clinical-prontuary/prontuaries`
- Finalidade: criar um prontuario para um paciente
- Modelo principal: `Prontuary`

### Body

```json
{
  "patientId": "patient-123",
  "bloodType": "O+"
}
```

`patientId` e obrigatorio. `bloodType` e opcional.

### Regras de negocio

- `patientId` e obrigatorio (`ClinicalProntuary.PatientIdRequired`)
- se ja existir prontuario para o paciente, retorna o existente sem criar duplicado (`ClinicalProntuary.AlreadyExists`)

### Exemplo de chamada

```bash
curl -X POST http://localhost:5000/api/v1/clinical-prontuary/prontuaries \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <jwt>' \
  -d '{
    "patientId": "patient-123",
    "bloodType": "O+"
  }'
```

### Sucesso esperado

- `201 Created`
- retorno: `Prontuary`

```json
{
  "id": "f1ac3c5e-6fd2-4a7a-b64e-72f40be690cf",
  "patientId": "patient-123",
  "bloodType": "O+",
  "createdAt": "2026-08-18T12:00:00Z",
  "updatedAt": "2026-08-18T12:00:00Z"
}
```

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `400` | `ClinicalProntuary.PatientIdRequired` | patientId vazio |

## 1.2 Obter prontuario por paciente

- Caso de uso: `FindProntuaryByPatientIdQuery`
- Endpoint: `GET /api/v1/clinical-prontuary/prontuaries/{patientId}`
- Finalidade: obter o prontuario completo com todas as consultas, alergias e imunizacoes

### Sucesso esperado

- `200 OK`
- retorno: `Prontuary` (com colecoes)

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `404` | `ClinicalProntuary.NotFound` | prontuario inexistente |

## 1.3 Obter resumo clinico (FHIR)

- Caso de uso: `FindProntuaryByPatientIdQuery` + `GetPatientClinicalSummaryQuery`
- Endpoint: `GET /api/v1/clinical-prontuary/prontuaries/{patientId}/fhir`
- Finalidade: bundle FHIR com prontuario e resumo clinico

### Sucesso esperado

- `200 OK`

```json
{
  "resourceType": "Bundle",
  "type": "collection",
  "patient": { ... },
  "summary": {
    "prontuary": { ... },
    "visits": [ ... ],
    "diagnoses": [ ... ],
    "prescriptions": [ ... ],
    "labResults": [ ... ],
    "allergies": [ ... ],
    "immunizations": [ ... ]
  }
}
```

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `404` | `ClinicalProntuary.NotFound` | prontuario inexistente |

---

## Navegacao

[ homepage Modelos e Enumeracoes](./models-and-enums.md) · [Indice do modulo](../index.md) · [Consultas](./visits.md)

**Neste modulo:** [Visao Geral](./overview.md) · [Modelos e Enumeracoes](./models-and-enums.md) · **Prontuarios** · [Consultas](./visits.md) · [Registos Clinicos](./clinical-records.md) · [Casos Internos e Notas](./internal-and-events.md)
