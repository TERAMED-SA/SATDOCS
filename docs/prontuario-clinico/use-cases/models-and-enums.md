# Modelos e Enumeracoes - ClinicalProntuary

[ homepage Modulo ClinicalProntuary](../index.md) · [Visao Geral](./overview.md)

## Modelos de Dominio

Todas as entidades vivem em `Domain/Entities`. Os constructores sao privados: o estado so muda atraves de metodos de dominio.

### Prontuary (Aggregate Root)

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `PatientId` | `string` | Obrigatorio, unico por paciente |
| `BloodType` | `string?` | Grupo sanguineo |
| `CreatedAt` | `DateTime` | |
| `UpdatedAt` | `DateTime` | |

**Colecoes:** `Visits`, `Allergies`, `Immunizations`

**Metodos de dominio:** `Create`, `AddVisit`, `AddAllergy`, `AddImmunization`, `GetDiagnoses`, `GetActiveMedications`

### Visit (Aggregate Root)

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `ProntuaryId` | `Guid` | FK para Prontuary |
| `ClinicalItemId` | `string` | Item clinico associado |
| `HospitalId` | `string` | Hospital onde ocorreu |
| `DoctorId` | `string` | Medico responsable |
| `Date` | `DateTime` | Data da consulta |
| `Type` | `VisitType` | Consultation, Exam, Vaccine, Procedure |
| `Notes` | `string?` | Observacoes |

**Colecoes:** `Diagnoses`, `Prescriptions`, `LabResults`, `MedicalNotes`

**Metodos de dominio:** `Create`, `AddDiagnosis`, `AddPrescription`, `AddLabResult`, `AddMedicalNote`

### Diagnosis

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `VisitId` | `Guid` | FK para Visit |
| `DiseaseId` | `string` | ID da doenca externa |
| `DiagnosedAt` | `DateTime` | |
| `Notes` | `string?` | |

### Prescription

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `VisitId` | `Guid` | FK para Visit |
| `MedicationId` | `string` | ID do medicamento externo |
| `Dosage` | `string?` | Dosagem |
| `StartDate` | `DateTime` | Data de inicio |
| `EndDate` | `DateTime?` | Data de fim (null = ativo) |
| `Notes` | `string?` | |

### LabResult

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `VisitId` | `Guid` | FK para Visit |
| `ItemId` | `string` | ID do item de exame |
| `ResultData` | `string` | JSON com resultados |
| `Date` | `DateTime` | |
| `Notes` | `string?` | |

### AllergyRecord

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `ProntuaryId` | `Guid` | FK para Prontuary |
| `Substance` | `string` | Substancia alergenica |
| `Reaction` | `string` | Reacao |
| `Severity` | `Severity` | Mild, Moderate, Severe, LifeThreatening |
| `CreatedAt` | `DateTime` | |

### ImmunizationRecord

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `ProntuaryId` | `Guid` | FK para Prontuary |
| `VaccineId` | `string` | ID da vacina externa |
| `DateAdministered` | `DateTime` | |
| `BatchNumber` | `string?` | Numero do lote |
| `Notes` | `string?` | |

### MedicalNote

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `VisitId` | `Guid` | FK para Visit |
| `DoctorId` | `string` | Medico que registou |
| `NoteText` | `string` | Texto da nota |
| `CreatedAt` | `DateTime` | |

## Enumeracoes

### VisitType

- `Consultation`
- `Exam`
- `Vaccine`
- `Procedure`

### Severity

- `Mild`
- `Moderate`
- `Severe`
- `LifeThreatening`

## Catalogo de Erros

Definidos em `Domain/Errors/ClinicalProntuaryErrors.cs`.

### Prontuario

| Code | Tipo | Mensagem |
|---|---|---|
| `ClinicalProntuary.NotFound` | NotFound | Prontuario not found. |
| `ClinicalProntuary.PatientIdRequired` | Validation | PatientId is required. |
| `ClinicalProntuary.AlreadyExists` | Conflict | A prontuary already exists for this patient. |

### Consulta

| Code | Tipo | Mensagem |
|---|---|---|
| `ClinicalProntuary.ProntuaryNotFound` | NotFound | Prontuary not found for patient. |
| `ClinicalProntuary.VisitNotFound` | NotFound | Visit not found. |

### Registos Clinicos

| Code | Tipo | Mensagem |
|---|---|---|
| `ClinicalProntuary.Diagnosis ValidationError` | Validation | Diagnosis data is invalid. |
| `ClinicalProntuary.Prescription ValidationError` | Validation | Prescription data is invalid. |
| `ClinicalProntuary.LabResult ValidationError` | Validation | Lab result data is invalid. |
| `ClinicalProntuary.Allergy ValidationError` | Validation | Allergy data is invalid. |
| `ClinicalProntuary.Immunization ValidationError` | Validation | Immunization data is invalid. |

---

## Navegacao

[ homepage Visao Geral](./overview.md) · [Indice do modulo](../index.md) · [Prontuarios](./prontuaries.md)

**Neste modulo:** [Visao Geral](./overview.md) · **Modelos e Enumeracoes** · [Prontuarios](./prontuaries.md) · [Consultas](./visits.md) · [Registos Clinicos](./clinical-records.md) · [Casos Internos e Notas](./internal-and-events.md)
