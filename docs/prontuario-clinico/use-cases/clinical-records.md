# Casos de Uso de Registos Clinicos

[ homepage Modulo ClinicalProntuary](../index.md) · [Visao Geral](./overview.md)

Grupo de rotas: `/visits/{visitId}` e `/prontuaries/{patientId}` · Requer autenticacao.

## Diagnosticos

### 3.1 Registar diagnostico

- Caso de uso: `RegisterDiagnosisCommand`
- Endpoint: `POST /api/v1/clinical-prontuary/visits/{visitId}/diagnoses`
- Modelo principal: `Diagnosis`

### Body

```json
{
  "diseaseId": "disease-uuid",
  "notes": "Diagnostico clinico"
}
```

### Regras de negocio

- a consulta tem de existir (`ClinicalProntuary.VisitNotFound`)

### Sucesso esperado

- `201 Created`
- retorno: `Diagnosis`

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `404` | `ClinicalProntuary.VisitNotFound` | consulta inexistente |

---

## Prescricoes

### 3.2 Registar prescricao

- Caso de uso: `RegisterPrescriptionCommand`
- Endpoint: `POST /api/v1/clinical-prontuary/visits/{visitId}/prescriptions`
- Modelo principal: `Prescription`

### Body

```json
{
  "medicationId": "medication-uuid",
  "dosage": "500mg 2x/dia",
  "startDate": "2026-08-18T00:00:00Z",
  "endDate": "2026-08-25T00:00:00Z",
  "notes": "Tomar com alimentos"
}
```

`medicationId` e `startDate` sao obrigatorios. `endDate` e opcional (null = medicacao ativa).

### Regras de negocio

- a consulta tem de existir (`ClinicalProntuary.VisitNotFound`)

### Sucesso esperado

- `201 Created`
- retorno: `Prescription`

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `404` | `ClinicalProntuary.VisitNotFound` | consulta inexistente |

---

## Resultados Laboratoriais

### 3.3 Registar resultado laboratorial

- Caso de uso: `RegisterLabResultCommand`
- Endpoint: `POST /api/v1/clinical-prontuary/visits/{visitId}/lab-results`
- Modelo principal: `LabResult`

### Body

```json
{
  "itemId": "lab-item-uuid",
  "resultData": "{\"hemoglobin\": 14.2, \"glucose\": 95}",
  "date": "2026-08-18T00:00:00Z",
  "notes": "Resultados dentro dos valores normais"
}
```

`itemId` e `resultData` sao obrigatorios. `resultData` e armazenado como JSON.

### Regras de negocio

- a consulta tem de existir (`ClinicalProntuary.VisitNotFound`)

### Sucesso esperado

- `201 Created`
- retorno: `LabResult`

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `404` | `ClinicalProntuary.VisitNotFound` | consulta inexistente |

---

## Imunizacoes

### 3.4 Registar imunizacao

- Caso de uso: `RegisterImmunizationCommand`
- Endpoint: `POST /api/v1/clinical-prontuary/prontuaries/{patientId}/immunizations`
- Modelo principal: `ImmunizationRecord`

### Body

```json
{
  "vaccineId": "vaccine-uuid",
  "dateAdministered": "2026-08-18T00:00:00Z",
  "batchNumber": "LOT-2026-001",
  "notes": "Sem reacoes adversas"
}
```

`vaccineId` e obrigatorio. Restantes campos sao opcionais.

### Regras de negocio

- o prontuario do paciente tem de existir (`ClinicalProntuary.ProntuaryNotFound`)

### Sucesso esperado

- `201 Created`
- retorno: `ImmunizationRecord`

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `404` | `ClinicalProntuary.ProntuaryNotFound` | prontuario inexistente |

---

## Alergias

### 3.5 Registar alergia

- Caso de uso: `RegisterAllergyCommand`
- Endpoint: `POST /api/v1/clinical-prontuary/prontuaries/{patientId}/allergies`
- Modelo principal: `AllergyRecord`

### Body

```json
{
  "substance": "Penicilina",
  "reaction": "Urticaria",
  "severity": "Severe"
}
```

Todos os campos sao obrigatorios.

### Regras de negocio

- o prontuario do paciente tem de existir (`ClinicalProntuary.ProntuaryNotFound`)

### Sucesso esperado

- `201 Created`
- retorno: `AllergyRecord`

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `404` | `ClinicalProntuary.ProntuaryNotFound` | prontuario inexistente |

---

## Navegacao

[ homepage Consultas](./visits.md) · [Indice do modulo](../index.md) · [Casos Internos e Notas](./internal-and-events.md)

**Neste modulo:** [Visao Geral](./overview.md) · [Modelos e Enumeracoes](./models-and-enums.md) · [Prontuarios](./prontuaries.md) · [Consultas](./visits.md) · **Registos Clinicos** · [Casos Internos e Notas](./internal-and-events.md)
