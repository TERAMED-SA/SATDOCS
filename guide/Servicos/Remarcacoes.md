# API Hospitalar — Endpoints (Remarcações)

> Todas as datas em **UTC** (ISO 8601). Padrões: `page=1`, `limit=20`, `sortOrder=desc`.  
> Autenticação: Bearer Token. Autorização: RBAC/ABAC conforme perfis.

---

## 6. REMARCAÇÕES (RESCHEDULE)

### 6.1. Gestão de Remarcações
- **POST** `/reschedules` - Solicitar remarcação
- **GET** `/reschedules/{id}` - Obter solicitação de remarcação
- **PATCH** `/reschedules/{id}/approve` - Aprovar remarcação
- **PATCH** `/reschedules/{id}/reject` - Rejeitar remarcação
- **DELETE** `/reschedules/{id}` - Cancelar solicitação

### 6.2. Consultas
- **GET** `/reschedules` - Listar remarcações
- **GET** `/appointments/{appointmentId}/reschedules` - Remarcações da marcação
- **GET** `/reschedules/pending` - Solicitações pendentes

---

# Filtros para Listagem de Remarcações (`GET /reschedules`)

## 1. FILTROS BÁSICOS E IDENTIFICAÇÃO

### 1.1. Identificação Direta
- **id** – ID específico da remarcação  
- **appointmentId** – Marcação específica  
- **requestedByTaxId** – Usuário que solicitou  
- **approvedByTaxId** – Usuário que aprovou  
- **oldSlotId** – Slot original  
- **newSlotId** – Novo slot  

---

## 2. FILTROS DE STATUS E ESTADO

### 2.1. Status da Remarcação
- **hasApproval** – Com aprovação (`approvedByTaxId != null`)  
- **isApproved** – Aprovadas (`approvedByTaxId != null && approvedAt != null`)  
- **isPending** – Pendentes de aprovação (`approvedByTaxId == null`)  
- **hasOldSlot** – Com slot original definido  
- **hasNewSlot** – Com novo slot definido  

### 2.2. Estados de Fluxo
- **awaitingApproval** – Aguardando aprovação (`approvedByTaxId == null`)  
- **completed** – Processadas (`approvedByTaxId != null`)  
- **hasChanges** – Com alterações de slot/horário  

---

## 3. FILTROS TEMPORAIS

### 3.1. Períodos de Solicitação
- **requestedAtFrom** – Solicitadas a partir de  
- **requestedAtTo** – Solicitadas até  
- **requestedDate** – Data específica de solicitação  
- **createdAtFrom** – Criadas a partir de  
- **createdAtTo** – Criadas até  
- **updatedAtFrom** – Atualizadas a partir de  
- **updatedAtTo** – Atualizadas até  

### 3.2. Períodos de Aprovação
- **approvedAtFrom** – Aprovadas a partir de  
- **approvedAtTo** – Aprovadas até  
- **hasApprovalDate** – Com data de aprovação  
- **approvalDelay** – Tempo até aprovação (em horas)  

### 3.3. Períodos Especiais
- **today** – Solicitadas hoje  
- **thisWeek** – Solicitadas esta semana  
- **lastWeek** – Solicitadas na semana passada  
- **thisMonth** – Solicitadas este mês  
- **recent** – Remarcações recentes (últimos 7 dias)  

---

## 4. FILTROS POR PARTICIPANTES

### 4.1. Solicitante
- **requestedByTaxId** – Usuário que solicitou  
- **requestedByTaxIds** – Múltiplos solicitantes  

### 4.2. Aprovador
- **approvedByTaxId** – Usuário que aprovou  
- **approvedByTaxIds** – Múltiplos aprovadores  
- **hasApprover** – Com aprovador definido  

### 4.3. Paciente (via Appointment)
- **patientTaxId** – Paciente da marcação  
- **patientAgeMin** – Idade mínima do paciente  
- **patientAgeMax** – Idade máxima do paciente  

### 4.4. Profissional (via Appointment)
- **professionalTaxId** – Profissional da marcação  
- **professionalTaxIds** – Múltiplos profissionais  

---

## 5. FILTROS DE ALTERAÇÕES

### 5.1. Alterações de Slot
- **oldSlotId** – Slot original específico  
- **newSlotId** – Novo slot específico  
- **slotChanged** – Com alteração de slot (`oldSlotId != newSlotId`)  
- **sameSlot** – Mesmo slot (apenas alteração de horário)  

### 5.2. Alterações de Horário
- **oldSelectedHourFrom** – Horário original a partir de  
- **oldSelectedHourTo** – Horário original até  
- **newSelectedHourFrom** – Novo horário a partir de  
- **newSelectedHourTo** – Novo horário até  
- **hourChanged** – Com alteração de horário  
- **timeChange** – Variação de tempo (em minutos)  

### 5.3. Tipo de Alteração
- **onlyTimeChange** – Apenas horário alterado (mesmo slot)  
- **onlySlotChange** – Apenas slot alterado (mesmo horário)  
- **bothChanged** – Slot e horário alterados  
- **noChanges** – Sem alterações (apenas registro)  

---

## 6. FILTROS DE CONTEÚDO

### 6.1. Motivo
- **reason** – Busca textual no motivo  
- **reasonContains** – Contém termo específico no motivo  
- **hasReason** – Com motivo definido (`true/false`)  
- **reasonLengthMin** – Tamanho mínimo do motivo  
- **reasonLengthMax** – Tamanho máximo do motivo  

### 6.2. Busca Textual
- **search** – Busca geral em motivo e campos relacionados  

---

## 7. FILTROS RELACIONADOS À MARCAÇÃO

### 7.1. Propriedades da Marcação
- **appointmentStatus** – Status da marcação relacionada  
- **appointmentTypeOfService** – Tipo de serviço da marcação  
- **appointmentSelectedHourFrom** – Data/hora original da marcação a partir de  
- **appointmentSelectedHourTo** – Data/hora original da marcação até  

### 7.2. Informações da Agenda
- **healthUnitTaxId** – Unidade de saúde da marcação  
- **specialityId** – Especialidade da marcação  
- **typeOfSchedule** – Tipo de agenda da marcação  

### 7.3. Slots Relacionados
- **slotSpecificDateFrom** – Data do slot a partir de  
- **slotSpecificDateTo** – Data do slot até  
- **slotWeekDay** – Dia da semana do slot  
- **slotIsClosed** – Slot fechado  

---

## 8. FILTROS DE FLUXO DE TRABALHO

### 8.1. Remarcações por Estágio
- **pendingApproval** – Aguardando aprovação  
- **recentlyApproved** – Aprovadas recentemente (últimas 24h)  
- **awaitingAction** – Requerem ação (pendentes + tempo)  
- **processed** – Processadas (com aprovação)  

### 8.2. Padrões de Remarcação
- **frequentReschedules** – Remarcações frequentes (por paciente/profissional)  
- **lastMinuteChanges** – Alterações de última hora (< 24h da consulta)  
- **advanceReschedules** – Remarcações com antecedência (> 7 dias)  

---

## 9. FILTROS ESTATÍSTICOS

### 9.1. Métricas de Tempo
- **responseTimeMin** – Tempo mínimo de resposta (em horas)  
- **responseTimeMax** – Tempo máximo de resposta (em horas)  
- **quickApprovals** – Aprovações rápidas (< 1 hora)  
- **delayedApprovals** – Aprovações demoradas (> 24 horas)  

### 9.2. Volume e Frequência
- **rescheduleCountMin** – Número mínimo de remarcações  
- **rescheduleCountMax** – Número máximo de remarcações  
- **hasMultipleReschedules** – Com múltiplas remarcações  

---

## 10. FILTROS DE PAGINAÇÃO E ORDENAÇÃO

### 10.1. Paginação
- **page** – Página atual (padrão: `1`)  
- **limit** – Itens por página (padrão: `20`, máximo: `100`)  
- **offset** – Alternativa à paginação  

### 10.2. Ordenação
- **sortBy** – Campo para ordenação:  
  - `requestedAt`  
  - `approvedAt`  
  - `createdAt`  
  - `updatedAt`  
  - `oldSelectedHour`  
  - `newSelectedHour`  

- **sortOrder** – `asc` ou `desc` (padrão: `desc` para datas)  

### 10.3. Seleção de Campos
- **fields** – Campos específicos a retornar  
- **include** – Relacionamentos a incluir:  
  - `appointment`  
  - `appointment.slot`  
  - `appointment.schedule`  
  - `appointment.patient` *(dados básicos)*  
  - `appointment.professional` *(dados básicos)*  
  - `oldSlot`  
  - `newSlot`  
  - `requestedBy` *(dados do usuário)*  
  - `approvedBy` *(dados do usuário)*  

---

## EXEMPLOS PRÁTICOS DE USO

### Exemplo 1: Remarcações pendentes de aprovação
```http
GET /reschedules?isPending=true&sortBy=requestedAt&sortOrder=asc&include=appointment.patient
```
### Exemplo 2: Remarcações de um paciente específico
```http
GET /reschedules?patientTaxId=BI12345&include=appointment,oldSlot,newSlot&sortBy=requestedAt&sortOrder=desc
```
### Exemplo 3: Remarcações aprovadas por um profissional
```http
GET /reschedules?approvedByTaxId=BI67890&approvedAtFrom=2024-12-01&include=appointment.patient,requestedBy
```
### Exemplo 4: Remarcações com alteração de slot
```http
GET /reschedules?slotChanged=true&requestedAtFrom=2024-12-01&include=appointment.schedule,oldSlot,newSlot
```
### Exemplo 5: Remarcações de última hora
```http
GET /reschedules?lastMinuteChanges=true&requestedAtFrom=2024-12-01&include=appointment.patient,professional
```
### Exemplo 6: Estatísticas de remarcação por unidade
```http
GET /reschedules?healthUnitTaxId=540102938&requestedAtFrom=2024-12-01&requestedAtTo=2024-12-31&fields=id,requestedAt,approvedAt,reason
```
### Exemplo 7: Remarcações por motivo específico
```http
GET /reschedules?reasonContains=emergência&isApproved=true&requestedAtFrom=2024-11-01
```
### Exemplo 8: Remarcações com aprovação rápida
```http
GET /reschedules?quickApprovals=true&approvedAtFrom=2024-12-01&include=appointment,approvedBy
```
### Exemplo 9: Histórico completo de remarcações
```http
GET /reschedules?patientTaxId=BI12345&include=appointment.schedule,oldSlot,newSlot,requestedBy,approvedBy&sortBy=requestedAt&sortOrder=desc&limit=50
```
### Exemplo 10: Remarcações por profissional
```http
GET /reschedules?professionalTaxId=BI67890&requestedAtFrom=2024-12-01&include=appointment.patient,oldSlot,newSlot&sortBy=requestedAt&sortOrder=asc
```
### Exemplo 11: Remarcações com tempo de resposta
```http
GET /reschedules?responseTimeMax=2&isApproved=true&requestedAtFrom=2024-12-01&include=appointment,approvedBy
```
### Exemplo 12: Dashboard de remarcações
```http
GET /reschedules?healthUnitTaxId=540102938&requestedAtFrom=2024-12-01&include=appointment.patient,appointment.professional,requestedBy&sortBy=requestedAt&sortOrder=desc
```
### Exemplo 13: Remarcações apenas de horário
```http
GET /reschedules?onlyTimeChange=true&requestedAtFrom=2024-12-01&include=appointment,oldSlot
```
### Exemplo 14: Remarcações frequentes
```http
GET /reschedules?frequentReschedules=true&patientTaxId=BI12345&include=appointment.schedule&sortBy=requestedAt&sortOrder=desc
```
### Exemplo 15: Busca complexa com múltiplos filtros
```http
GET /reschedules?healthUnitTaxId=540102938&specialityId=CARDIOLOGY&isPending=true&requestedAtFrom=2024-12-01&slotChanged=true&include=appointment.patient,appointment.professional,oldSlot,newSlot,requestedBy&sortBy=requestedAt&sortOrder=asc&limit=20
```
### Exemplo 16: Remarcações com alterações específicas de data
```http
GET /reschedules?oldSelectedHourFrom=2024-12-15&oldSelectedHourTo=2024-12-20&newSelectedHourFrom=2024-12-20&include=appointment,oldSlot,newSlot
```
### Exemplo 17: Remarcações sem aprovação
```http
GET /reschedules?hasApproval=false&requestedAtFrom=2024-12-01&include=appointment.patient,requestedBy&sortBy=requestedAt&sortOrder=desc
```
### Exemplo 18: Remarcações aprovadas recentemente
```http
GET /reschedules?recentlyApproved=true&include=appointment,approvedBy&sortBy=approvedAt&sortOrder=desc
```

---

## ESTRUTURA DE RESPOSTA (Exemplo)

```json
{
  "data": [
    {
      "id": "resched-123",
      "appointmentId": "appt-456",
      "oldSlotId": "slot-789",
      "oldSelectedHour": "2024-12-15T10:00:00Z",
      "newSlotId": "slot-012",
      "newSelectedHour": "2024-12-16T14:00:00Z",
      "reason": "Paciente com compromisso urgente",
      "requestedByTaxId": "BI12345",
      "requestedAt": "2024-12-14T15:30:00Z",
      "approvedByTaxId": "BI67890",
      "approvedAt": "2024-12-14T16:00:00Z",
      "createdAt": "2024-12-14T15:30:00Z",
      "updatedAt": "2024-12-14T16:00:00Z",
      "appointment": {
        "id": "appt-456",
        "patientTaxId": "BI99999",
        "patientAge": 35,
        "professionalTaxId": "BI12345",
        "typeOfService": "IN_PERSON",
        "status": "RESCHEDULED"
      },
      "oldSlot": {
        "id": "slot-789",
        "specificDate": "2024-12-15T00:00:00Z",
        "weekDay": "MONDAY"
      },
      "newSlot": {
        "id": "slot-012",
        "specificDate": "2024-12-16T00:00:00Z",
        "weekDay": "TUESDAY"
      },
      "requestedBy": {
        "name": "Dr. Silva",
        "taxId": "BI12345"
      },
      "approvedBy": {
        "name": "Coord. Santos",
        "taxId": "BI67890"
      }
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 65,
    "totalPages": 4,
    "hasNext": true,
    "hasPrev": false
  },
  "stats": {
    "totalReschedules": 65,
    "byStatus": {
      "pending": 15,
      "approved": 45,
      "rejected": 5
    },
    "byType": {
      "timeOnly": 20,
      "slotOnly": 25,
      "both": 20
    }
  }
}
```
