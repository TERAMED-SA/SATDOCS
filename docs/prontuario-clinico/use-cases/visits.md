# Casos de Uso de Consultas

[ homepage Modulo ClinicalProntuary](../index.md) · [Visao Geral](./overview.md)

Grupo de rotas: `/prontuaries/{patientId}/visits` · Requer autenticacao.

## 2.1 Criar consulta

- Caso de uso: `CreateVisitCommand`
- Endpoint: `POST /api/v1/clinical-prontuary/prontuaries/{patientId}/visits`
- Finalidade: registar uma nova consulta num prontuario
- Modelo principal: `Visit`

### Body

```json
{
  "clinicalItemId": "clinical-item-uuid",
  "hospitalId": "hospital-uuid",
  "doctorId": "doctor-uuid",
  "date": "2026-08-18T10:30:00Z",
  "type": "Consultation",
  "notes": "Consulta de rotina"
}
```

Todos os campos exceto `notes` sao obrigatorios.

### Regras de negocio

- o prontuario do paciente tem de existir (`ClinicalProntuary.ProntuaryNotFound`)

### Sucesso esperado

- `201 Created`
- retorno: `Visit`

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `404` | `ClinicalProntuary.ProntuaryNotFound` | prontuario inexistente para o paciente |

## 2.2 Listar consultas do paciente

- Caso de uso: `GetVisitsByPatientQuery`
- Endpoint: `GET /api/v1/clinical-prontuary/prontuaries/{patientId}/visits`
- Finalidade: obter todas as consultas de um paciente

### Sucesso esperado

- `200 OK`
- retorno: array de `Visit`

---

## Navegacao

[ homepage Prontuarios](./prontuaries.md) · [Indice do modulo](../index.md) · [Registos Clinicos](./clinical-records.md)

**Neste modulo:** [Visao Geral](./overview.md) · [Modelos e Enumeracoes](./models-and-enums.md) · [Prontuarios](./prontuaries.md) · **Consultas** · [Registos Clinicos](./clinical-records.md) · [Casos Internos e Notas](./internal-and-events.md)
