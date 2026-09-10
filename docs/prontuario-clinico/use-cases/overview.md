# Visao Geral da API - ClinicalProntuary

[ homepage Modulo ClinicalProntuary](../index.md)

Todas as rotas requerem autenticacao (JWT Bearer). Base path: `/api/v1/clinical-prontuary` (configurado no host).

## Resumo dos Endpoints

### Prontuarios

| Metodo | Rota | Comando/Query | Descricao |
|---|---|---|---|
| `GET` | `prontuaries/{patientId}` | `FindProntuaryByPatientIdQuery` | Obter prontuario completo do paciente |
| `GET` | `prontuaries/{patientId}/fhir` | `FindProntuaryByPatientIdQuery` + `GetPatientClinicalSummaryQuery` | Bundle FHIR com prontuario e resumo |
| `POST` | `prontuaries` | `CreateProntuaryCommand` | Criar novo prontuario |

### Consultas (Visits)

| Metodo | Rota | Comando/Query | Descricao |
|---|---|---|---|
| `POST` | `prontuaries/{patientId}/visits` | `CreateVisitCommand` | Registar nova consulta |
| `GET` | `prontuaries/{patientId}/visits` | `GetVisitsByPatientQuery` | Listar consultas do paciente |

### Registos Clinicos

| Metodo | Rota | Comando/Query | Descricao |
|---|---|---|---|
| `POST` | `visits/{visitId}/diagnoses` | `RegisterDiagnosisCommand` | Registar diagnostico |
| `POST` | `visits/{visitId}/prescriptions` | `RegisterPrescriptionCommand` | Registar prescricao |
| `POST` | `visits/{visitId}/lab-results` | `RegisterLabResultCommand` | Registar resultado laboratorial |
| `POST` | `prontuaries/{patientId}/immunizations` | `RegisterImmunizationCommand` | Registar imunizacao |
| `POST` | `prontuaries/{patientId}/allergies` | `RegisterAllergyCommand` | Registar alergia |

## Cabecalhos de Autenticacao

```
Authorization: Bearer <jwt-token>
```

## Convencoes de Resposta

- `200 OK` — operacao de leitura bem-sucedida
- `201 Created` — criacao bem-sucedida
- `400 Bad Request` — erro de validacao ou regra de negocio
- `404 Not Found` — entidade nao encontrada
- `403 Forbidden` — sem permissao

## Erro Envelope

```json
{
  "code": "ClinicalProntuary.NotFound",
  "message": "Prontuary not found."
}
```

## Autorizacao

Todos os endpoints usam politicas manuais via `ProntuaryPolicies`:
- **Leitura:** roles `SYSTEM_ADMIN`, `ORG_ADMIN`, `SERVICE_ACCOUNT`, `CLINICAL_ACTOR`, `VIEW_PATIENT_HISTORY`, ou `CUSTOM` com 2FA
- **Escrita:** requer 2FA + roles `SYSTEM_ADMIN`, `ORG_ADMIN`, `SERVICE_ACCOUNT`, `CLINICAL_ACTOR`, `PRESCRIBER`, `DIAGNOSER`

---

## Navegacao

[ homepage Modulo](../index.md) · [Modelos e Enumeracoes](./models-and-enums.md)

**Neste modulo:** **Visao Geral** · [Modelos e Enumeracoes](./models-and-enums.md) · [Prontuarios](./prontuaries.md) · [Consultas](./visits.md) · [Registos Clinicos](./clinical-records.md) · [Casos Internos e Notas](./internal-and-events.md)
