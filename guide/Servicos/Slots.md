# API Hospitalar — Endpoints (Slots)

> Todas as datas em **UTC** (ISO 8601). Padrões: `page=1`, `limit=20`, `sortOrder=desc`.
> Autenticação: Bearer Token. Autorização: RBAC/ABAC conforme perfis.

---

## 1. SLOTS

### 3.1. Geração e Gestão
- **POST** `/schedules/{id}/generate-slots` — Gerar slots automaticamente
- **POST** `/schedules/{id}/generate-slots/async` — Gerar slots assincronamente
- **GET** `/operations/{operationId}` — Consultar status de operação assíncrona
- **POST** `/slots/manual` — Criar slot manualmente
- **PUT** `/slots/{id}` — Atualizar slot
- **PATCH** `/slots/{id}` — Atualização parcial do slot

### 3.2. Consultas
- **GET** `/slots/{id}` — Obter slot por ID
- **GET** `/slots` — Listar slots com filtros
- **GET** `/schedules/{scheduleId}/slots` — Listar slots da agenda
- **GET** `/slots/available` — Listar slots disponíveis
- **GET** `/slots/date/{date}` — Listar slots por data
- **GET** `/slots/professional/{professionalTaxId}` — Listar slots por profissional

### 3.3. Estados e Capacidade
- **PATCH** `/slots/{id}/close` — Fechar slot
- **PATCH** `/slots/{id}/open` — Reabrir slot
- **PATCH** `/slots/{id}/capacity` — Atualizar capacidade do slot
- **PATCH** `/slots/{id}/professionals` — Atualizar profissionais do slot

---

# Filtros para Listagem de Slots (`GET /slots`)

## FILTROS BÁSICOS E IDENTIFICAÇÃO

### 1.1. Identificação Direta
- **id** — ID específico do slot  
- **scheduleId** — Agenda específica  
- **scheduleIds** — Múltiplas agendas (ex.: `scheduleIds=123,456,789`)

### 1.2. Período e Data
- **specificDate** — Data específica  
- **specificDateFrom** — Data a partir de  
- **specificDateTo** — Data até  
- **dateRange** — Período específico (ex.: `2024-12-01,2024-12-31`)  
- **weekDay** — Dia da semana (`SUNDAY`…`SATURDAY`)

---

## FILTROS DE CONFIGURAÇÃO E ESTADO

### 2.1. Estado do Slot
- **isClosed** — Slots fechados/abertos (`true/false`)  
- **inheritAllProfessionals** — Herda profissionais da agenda (`true/false`)  
- **markingLimit** — Capacidade específica  
- **markingLimitMin** — Capacidade mínima  
- **markingLimitMax** — Capacidade máxima

### 2.2. Modo de Operação
- **mode** — `"TIMED"` (com horas) ou `"CAPACITY"` (apenas capacidade)  
- **hasHours** — Slots com horas definidas (`true/false`)  
- **hoursCountMin** — Mínimo de horas no array  
- **hoursCountMax** — Máximo de horas no array

> **Nota:** Em conformidade com o schema, `hours` é um array JSON com objetos que representam caixas horárias (`hour`, `status`, `appointmentId`, etc.).

---

## FILTROS POR PROFISSIONAIS

### 3.1. Profissionais Disponíveis
- **availableProfessionalTaxIds** — Contém profissional específico  
- **professionalTaxId** — Slot disponível para profissional  
- **hasProfessionals** — Com profissionais definidos (`true/false`)  
- **professionalsCountMin** — Mínimo de profissionais  
- **professionalsCountMax** — Máximo de profissionais

---

## FILTROS DE DISPONIBILIDADE E CAPACIDADE

### 4.1. Capacidade e Ocupação
- **availableCapacityMin** — Capacidade disponível mínima  
- **availableCapacityMax** — Capacidade disponível máxima  
- **occupancyRateMin** — Taxa de ocupação mínima (%)  
- **occupancyRateMax** — Taxa de ocupação máxima (%)  
- **isFull** — Slots lotados (`true/false`)  
- **hasAvailableCapacity** — Com capacidade disponível (`true/false`)

### 4.2. Horas Específicas
- **hourAvailable** — Horário específico disponível (ex.: `08:00`)  
- **timeRangeAvailable** — Período disponível (ex.: `08:00-12:00`)  
- **hasAvailableHours** — Com horas disponíveis (`true/false`)

---

## FILTROS RELACIONADOS A AGENDA

### 5.1. Propriedades da Agenda
- **healthUnitTaxId** — Unidade de saúde  
- **specialityId** — Especialidade  
- **typeOfService** — Tipo de serviço (`IN_PERSON`, `TELEMEDICINE`, `HOME`)  
- **typeOfSchedule** — Tipo de agenda (`CONSULTATION`, `VACCINE`, `EXAM`, `RETURN`)

### 5.2. Configurações da Agenda
- **scheduleTypeOfRecurrence** — Recorrência da agenda  
- **automaticallyGenerated** — Slots gerados automaticamente (`true/false`)

---

## FILTROS TEMPORAIS AVANÇADOS

### 6.1. Criação e Atualização
- **createdAtFrom** — Criados a partir de  
- **createdAtTo** — Criados até  
- **updatedAtFrom** — Atualizados a partir de  
- **updatedAtTo** — Atualizados até  
- **updatedRecently** — Atualizados nas últimas X horas

### 6.2. Períodos Especiais
- **today** — Slots de hoje (`true/false`)  
- **tomorrow** — Slots de amanhã (`true/false`)  
- **thisWeek** — Slots desta semana (`true/false`)  
- **nextWeek** — Slots da próxima semana (`true/false`)  
- **weekend** — Slots de fim de semana (`true/false`)

---

## FILTROS DE MARCAÇÕES

### 7.1. Relacionados a Appointments
- **hasAppointments** — Com marcações (`true/false`)  
- **appointmentsCountMin** — Mínimo de marcações  
- **appointmentsCountMax** — Máximo de marcações  
- **appointmentStatus** — Status das marcações no slot

---

## FILTROS DE STATUS E VISIBILIDADE

### 8.1. Status do Slot
- **isActive** — Slots ativos (baseado em `deletedAt`)  
- **includeDeleted** — Incluir slots deletados (padrão: `false`)  
- **onlyDeleted** — Apenas slots deletados

### 8.2. Filtros de JSON Hours
- **hoursStatus** — Status específico nas horas (`AVAILABLE`, `BOOKED`, etc.)  
- **hasBookedHours** — Com horas marcadas (`true/false`)  
- **hasAvailableHours** — Com horas disponíveis (`true/false`)  
- **hasBlockedHours** — Com horas bloqueadas (`true/false`)

---

## FILTROS DE PAGINAÇÃO E ORDENAÇÃO

### 9.1. Paginação
- **page** — Página atual (padrão: 1)  
- **limit** — Itens por página (padrão: 20, máximo: 100)  
- **offset** — Alternativa à paginação

### 9.2. Ordenação
- **sortBy** — Campo para ordenação (`specificDate`, `createdAt`, `updatedAt`, `markingLimit`, `weekDay`, `availableCapacity`)  
- **sortOrder** — `asc` ou `desc` (padrão: `asc` para datas)

### 9.3. Seleção de Campos
- **fields** — Campos específicos a retornar  
- **include** — Relacionamentos a incluir (`schedule`, `appointments`, `schedule.excludeRanges`, `schedule.excludeDays`)

---

# EXEMPLOS PRÁTICOS DE USO — SLOTS (`GET /slots`)

### Exemplo 1: Slots disponíveis para um profissional
```http
GET /slots?professionalTaxId=BI12345&isClosed=false&specificDateFrom=2024-12-01&hasAvailableHours=true
```

### Exemplo 2: Slots de uma agenda específica
```http
GET /slots?scheduleId=123&specificDateFrom=2024-12-01&specificDateTo=2024-12-31&isClosed=false
```

### Exemplo 3: Slots com capacidade disponível
```http
GET /slots?availableCapacityMin=1&isClosed=false&specificDateFrom=2024-12-01&mode=CAPACITY
```

### Exemplo 4: Slots por tipo de serviço
```http
GET /slots?typeOfService=TELEMEDICINE&healthUnitTaxId=540102938&isClosed=false&today=true
```

### Exemplo 5: Slots com horários específicos
```http
GET /slots?hourAvailable=08:00&timeRangeAvailable=08:00-12:00&hasAvailableHours=true
```

### Exemplo 6: Slots para agendamento urgente
```http
GET /slots?today=true&hasAvailableCapacity=true&occupancyRateMax=50&sortBy=specificDate&sortOrder=asc
```

### Exemplo 7: Slots com paginação avançada
```http
GET /slots?scheduleId=123&page=2&limit=10&sortBy=markingLimit&sortOrder=desc&fields=id,specificDate,markingLimit,isClosed
```

### Exemplo 8: Slots de fim de semana
```http
GET /slots?weekend=true&weekDay=SATURDAY,SUNDAY&isClosed=false&hasAvailableHours=true
```

### Exemplo 9: Slots com estatísticas de ocupação
```http
GET /slots?occupancyRateMin=0&occupancyRateMax=80&appointmentsCountMax=5&isClosed=false
```

### Exemplo 10: Slots para dashboard administrativo
```http
GET /slots?healthUnitTaxId=540102938&specificDateFrom=2024-12-01&includeDeleted=false&hasAppointments=true&include=schedule,appointments
```

### Exemplo 11: Slots com filtro por hora específica
```http
GET /slots?hoursStatus=AVAILABLE&hourAvailable=14:00&specificDate=2024-12-15&isClosed=false
```

### Exemplo 12: Slots recém-atualizados
```http
GET /slots?updatedRecently=24&isClosed=false&hasAvailableCapacity=true
```

---

## ESTRUTURA DE RESPOSTA
### Resposta Paginada com Estatísticas
```json
{
  "data": [
    {
      "id": "slot-123",
      "specificDate": "2024-12-15T00:00:00Z",
      "weekDay": "MONDAY",
      "markingLimit": 10,
      "isClosed": false,
      "availableCapacity": 3,
      "occupancyRate": 70,
      "inheritAllProfessionals": true,
      "availableProfessionalTaxIds": ["BI12345", "BI67890"],
      "hours": [
        {
          "hour": "2024-12-15T10:00:00Z",
          "status": "AVAILABLE",
          "appointmentId": null,
          "professionalTaxId": null,
          "updatedAt": "2024-12-10T12:00:00Z"
        }
      ],
      "schedule": {
        "id": "sched-456",
        "healthUnitTaxId": "540102938",
        "specialityId": "CARDIOLOGY",
        "typeOfService": "TELEMEDICINE"
      }
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "totalPages": 8,
    "hasNext": true,
    "hasPrev": false
  },
  "stats": {
    "totalSlots": 150,
    "availableSlots": 45,
    "closedSlots": 25,
    "averageOccupancy": 65.5
  }
}
```

---

## VALIDAÇÕES IMPORTANTES
- **Datas**: Formato ISO 8601 (YYYY-MM-DD)  
- **Horas**: Formato HH:MM (24 horas)  
- **Enums**: Valores devem corresponder aos definidos no schema  
- **Capacidade**: Valores numéricos positivos  
- **Permissões**: Usuário deve ter acesso às agendas relacionadas  
- **Performance**: Filtros complexos podem ter timeout aumentado  
- **Limites**: Máximo de 100 itens por página para performance
