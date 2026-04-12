# **Guia Completo de Modelagem Clínica - Catálogo Clínico**

## **Introdução: Por que este sistema existe?**

Imagine que você precisa construir o "dicionário médico" de um hospital. Um lugar onde:

1. **Todos os procedimentos, exames e consultas** são definidos de forma padronizada
2. **As regras de segurança** são claras e automaticamente aplicadas
3. **Não há ambiguidade** sobre o que pode ser feito e quando

**Este é o Catálogo Clínico.** Ele serve como **fonte única da verdade** para:

* **Agendamento**: Saber o que pode ser marcado e quando
* **Prontuário**: Registrar procedimentos corretamente
* **Faturamento**: Cobrar pelo código correto
* **Segurança**: Prevenir procedimentos perigosos

**Usuários típicos deste sistema:**
- **Desenvolvedores**: Que integram com outros sistemas
- **Administradores**: Que cadastram novos serviços
- **Gestores hospitalares**: Que precisam de relatórios
- **Outros microserviços**: Agenda, Prontuário, Faturamento

---

## **Parte 1: Os 4 Níveis Fundamentais - A Anatomia do Catálogo**

### **Diagrama da Estrutura**

```
                    ┌─────────────────────────────────┐
                    │  1. ESPECIALIDADE               │
                    │  "Quem faz?"                    │
                    │  Ex: Cardiologia                │
                    └──────────┬──────────────────────┘
                               │
                    ┌──────────▼──────────────────────┐
                    │  2. SERVIÇO CLÍNICO             │
                    │  "Que tipo de atendimento?"     │
                    │  Ex: Consulta, Exame, Vacina    │
                    └──────────┬──────────────────────┘
                               │
                    ┌──────────▼──────────────────────┐
                    │  3. ITEM CLÍNICO                │
                    │  "O que exatamente?"            │
                    │  Ex: Hemograma, COVID-19 vaccine│
                    └──────────┬──────────────────────┘
                               │
                    ┌──────────▼──────────────────────┐
                    │  4. REGRA CLÍNICA               │
                    │  "Quando pode/não pode fazer?"  │
                    │  Ex: Esperar 30 dias após cirurgia
                    └─────────────────────────────────┘
```

---

## **Parte 2: Especialidades Médicas - "Os Departamentos do Hospital"**

### **O que é uma especialidade?**

É como um **departamento médico** dentro do hospital. Não é um procedimento, mas sim **quem** realiza os procedimentos.

### **Exemplos Reais com Códigos Padrão**

| Especialidade | Código | O que faz | Exemplo de Item |
|--------------|--------|-----------|-----------------|
| Cardiologia | CARD | Cuida do coração | Eletrocardiograma |
| Hematologia | HEMO | Cuida do sangue | Hemograma completo |
| Pediatria | PED | Cuida de crianças | Vacina BCG infantil |
| Infectologia | INF | Doenças infecciosas | Teste HIV |
| Neurologia | NEUR | Sistema nervoso | Tomografia cerebral |

### **API: Criando uma Especialidade**

```bash
# Endpoint
POST http://localhost:3000/specialties

# Cabeçalho (para desenvolvimento)
Content-Type: application/json

# Corpo da requisição
{
  "code": "CARDIOL",           // Único e em maiúsculas
  "name": "Cardiologia",       // Nome completo
  "description": "Especialidade médica responsável pelo diagnóstico e tratamento de doenças do coração e sistema circulatório."
}

# Resposta de sucesso (201 Created)
{
  "id": "123e4567-e89b-12d3-a456-426614174000",
  "code": "CARDIOL",
  "name": "Cardiologia",
  "description": "Especialidade médica responsável...",
  "isActive": true,
  "createdBy": "sistema",
  "updatedBy": "sistema",
  "createdAt": "2024-01-15T10:30:00.000Z",
  "updatedAt": "2024-01-15T10:30:00.000Z"
}
```

### **Regras Importantes para Especialidades**

1. **Não pode deletar** se houver serviços associados
2. **Código único** em todo o sistema
3. **Código em maiúsculas** (padrão internacional)
4. **isActive = false** desabilita todos os serviços e itens

### **Exemplo de Verificação Antes de Deletar**

```bash
# Antes de deletar, verifique se pode
GET http://localhost:3000/specialties/{id}/can-delete

# Resposta
{
  "canDelete": false,           // Atenção!
  "reason": "Specialty has 3 associated services",
  "serviceCount": 3             // Tem 3 serviços usando esta especialidade
}
```

---

## **Parte 3: Serviços Clínicos - "Os Tipos de Atendimento"**

### **O que é um Serviço Clínico?**

São as **categorias de atendimento** dentro de uma especialidade. Pense como "o que pode ser feito" em uma determinada área.

### **Os 4 Tipos de Serviço (ENUM fixo)**

| Tipo | Código | Exemplos Reais | Quando usar |
|------|--------|----------------|-------------|
| **CONSULTATION** | CONSULTA | Consulta cardiológica, Retorno pediátrico | Avaliação médica |
| **EXAM** | EXAME | Hemograma, Raio-X, Tomografia | Coleta/Análise de amostras |
| **VACCINE** | VACINA | COVID-19, Gripe, Febre amarela | Imunização |
| **PROCEDURE** | PROCEDIMENTO | Cirurgia, Sutura, Biópsia | Intervenção física |

### **Exemplo Prático: Estruturando a Cardiologia**

```
Cardiologia (Especialidade)
├── Consultas (Tipo: CONSULTATION)
│   ├── Primeira consulta
│   └── Retorno
├── Exames (Tipo: EXAM)
│   ├── Eletrocardiograma
│   └── Ecocardiograma
└── Procedimentos (Tipo: PROCEDURE)
    └── Cateterismo
```

### **API: Criando um Serviço Clínico**

```bash
POST http://localhost:3000/services

{
  "code": "CARD_CONSULT",              // Único no sistema
  "name": "Consulta Cardiológica",
  "specialtyId": "123e4567-e89b...",   // ID da especialidade Cardiologia
  "type": "CONSULTATION",              // Um dos 4 tipos fixos
  "description": "Avaliação inicial com cardiologista",
  "isActive": true                     // Pode ser desativado temporariamente
}

# Resposta
{
  "id": "456e7890-f12c-34d5-b678-567812345678",
  "code": "CARD_CONSULT",
  "name": "Consulta Cardiológica",
  "type": "CONSULTATION",
  "specialtyId": "123e4567-e89b...",
  "description": "Avaliação inicial...",
  "isActive": true,
  "createdAt": "2024-01-15T10:35:00.000Z",
  "updatedAt": "2024-01-15T10:35:00.000Z",
  "specialty": {
    "id": "123e4567-e89b...",
    "code": "CARDIOL",
    "name": "Cardiologia"
  }
}
```

### **Exemplo de Filtros e Buscas**

```bash
# Buscar todos os serviços de uma especialidade
GET http://localhost:3000/services?specialtyId=123e4567-e89b...

# Buscar apenas consultas ativas
GET http://localhost:3000/services?type=CONSULTATION&isActive=true

# Buscar por nome ou código
GET http://localhost:3000/services?search=cardio

# Paginação (50 por página)
GET http://localhost:3000/services?page=1&limit=50
```

### **Batch Request: Buscar Múltiplos Serviços**

```bash
# Quando a Agenda precisa de vários serviços de uma vez
GET http://localhost:3000/services/batch?ids=id1,id2,id3

# Ou buscar por especialidade em lote
GET http://localhost:3000/services/batch?specialtyId=123e4567...
```

---

## **Parte 4: Itens Clínicos - "Os Procedimentos Concretos"**

### **O que é um Item Clínico?**

É a **coisa real que será executada**. É o que aparece na agenda, no prontuário e na fatura.

### **Exemplos Reais por Tipo**

| Tipo | Item Clínico | Código | Para que serve |
|------|--------------|--------|----------------|
| **EXAM** | Hemograma Completo | HEMOGRAM | Contagem de células sanguíneas |
| **VACCINE** | Vacina COVID-19 | COVID19VAC | Imunização contra coronavírus |
| **PROCEDURE** | Cirurgia de Apendicite | APENDICEC | Remoção do apêndice |
| **CONSULTATION** | Consulta Pré-Operatória | PREOPCONS | Avaliação antes de cirurgia |

### **Exemplo Detalhado: O Hemograma**

```json
{
  "item": {
    "id": "789a0123-b45c-67d8-e901-234567890123",
    "code": "HEMOGRAM_COMPLETE",
    "name": "Hemograma Completo",
    "type": "EXAM",
    "description": "Exame de sangue que avalia glóbulos vermelhos, brancos e plaquetas",
    "clinicalService": {
      "id": "456e7890-f12c-34d5-b678-567812345678",
      "name": "Exames Hematológicos",
      "specialty": {
        "name": "Hematologia",
        "code": "HEMO"
      }
    },
    "rules": [  // Regras de segurança aplicáveis
      {
        "minHoursAfterBloodDonation": 72,
        "minDaysAfterVaccination": 7,
        "contraindicatedAnticoagulants": true
      }
    ]
  }
}
```

### **API: Criando um Item Clínico**

```bash
POST http://localhost:3000/items

{
  "code": "COVID19_VACCINE",
  "name": "Vacina COVID-19",
  "clinicalServiceId": "456e7890-f12c...",  // ID do serviço de vacinação
  "description": "Vacina mRNA contra SARS-CoV-2, dose única ou reforço"
}

# Resposta
{
  "id": "890b1234-c56d-78e9-f012-345678901234",
  "code": "COVID19_VACCINE",
  "name": "Vacina COVID-19",
  "clinicalServiceId": "456e7890-f12c...",
  "isActive": true,
  "createdAt": "2024-01-15T10:40:00.000Z",
  "clinicalService": {
    "id": "456e7890-f12c...",
    "code": "VACCINATION_SRV",
    "name": "Serviço de Vacinação",
    "type": "VACCINE",
    "specialty": {
      "name": "Infectologia",
      "code": "INF"
    }
  }
}
```

### **Cenário Completo: Sistema de Doação de Sangue**

```yaml
# Estrutura do exemplo
Especialidade: Hematologia (HEMO)
├── Serviço: Doação de Sangue (Tipo: PROCEDURE)
│   └── Item: Doação de Sangue Completa
│       └── Regra: "Bloqueia exames por 30 dias"
│
└── Serviço: Exames Laboratoriais (Tipo: EXAM)
    ├── Item: Hemograma
    ├── Item: Glicemia
    └── Item: Colesterol
```

**Na prática:** Quando alguém doa sangue, automaticamente não pode fazer exames por 30 dias.

---

## **Parte 5: Regras Clínicas - "O Sistema de Segurança"**

### **O que são Regras Clínicas?**

São as **restrições médicas** que garantem que procedimentos não coloquem pacientes em risco. Elas são **versionadas** no tempo.

### **Os 4 Parâmetros de Segurança**

| Parâmetro | Unidade | Exemplo Médico | Por que existe? |
|-----------|---------|----------------|-----------------|
| **minHoursAfterBloodDonation** | Horas | 72 horas | Para evitar anemia após doação |
| **minDaysAfterVaccination** | Dias | 7 dias | Evitar reações cruzadas |
| **minDaysAfterSurgery** | Dias | 30 dias | Tempo de cicatrização |
| **contraindicatedAnticoagulants** | Booleano | true/false | Risco de sangramento |

### **Exemplo Real: Regra para Cirurgia**

```json
{
  "rule": {
    "id": "901c2345-d67e-89f0-g123-456789012345",
    "clinicalItemId": "APENDICEC",  // Cirurgia de apêndice
    "minHoursAfterBloodDonation": 168,      // 7 dias
    "minDaysAfterVaccination": 14,          // 2 semanas
    "minDaysAfterSurgery": 90,              // 3 meses
    "contraindicatedAnticoagulants": true,  // Perigoso
    "validFrom": "2024-01-01T00:00:00.000Z",
    "validTo": null,                        // Válida indefinidamente
    "createdBy": "dr.silva@hospital.com",
    "explanation": "Pacientes em anticoagulantes têm risco aumentado de sangramento durante cirurgias"
  }
}
```

### **API: Criando uma Regra Clínica**

```bash
POST http://localhost:3000/rules

{
  "clinicalItemId": "HEMOGRAM",  // Item: Hemograma
  "minHoursAfterBloodDonation": 72,          // 3 dias
  "minDaysAfterVaccination": 7,              // 1 semana
  "minDaysAfterSurgery": 0,                  // Pode fazer imediatamente
  "contraindicatedAnticoagulants": true,     // CUIDADO!
  "validFrom": "2024-01-01T00:00:00.000Z",   // Data de início
  "validTo": "2024-12-31T23:59:59.999Z",     // Data de fim (opcional)
  "createdBy": "lab.manager@hospital.com"    // Quem criou
}

# Resposta
{
  "id": "012d3456-e78f-90a1-h234-567890123456",
  "clinicalItemId": "HEMOGRAM",
  "minHoursAfterBloodDonation": 72,
  "minDaysAfterVaccination": 7,
  "minDaysAfterSurgery": 0,
  "contraindicatedAnticoagulants": true,
  "validFrom": "2024-01-01T00:00:00.000Z",
  "validTo": "2024-12-31T23:59:59.999Z",
  "isCurrent": true,  // Regra atualmente válida
  "createdBy": "lab.manager@hospital.com",
  "createdAt": "2024-01-15T10:45:00.000Z",
  "clinicalItem": {
    "id": "789a0123-b45c...",
    "code": "HEMOGRAM",
    "name": "Hemograma Completo"
  }
}
```

### **Versionamento de Regras**

As regras podem mudar com o tempo. O sistema mantém todas as versões:

```bash
# Buscar regras válidas em uma data específica
GET http://localhost:3000/rules?itemId=HEMOGRAM&validAt=2024-06-15

# Buscar apenas regra atual
GET http://localhost:3000/rules?itemId=HEMOGRAM&currentOnly=true

# Ver histórico de mudanças
GET http://localhost:3000/rules/{ruleId}/audit
```

### **Exemplo de Auditoria de Regra**

```json
{
  "auditTrail": [
    {
      "changedAt": "2024-06-01T14:30:00.000Z",
      "changedBy": "dr.costa@hospital.com",
      "change": "Aumentou minDaysAfterVaccination de 3 para 7",
      "oldValue": 3,
      "newValue": 7,
      "reason": "Novos estudos mostram maior risco de reação"
    }
  ]
}
```

---

## **Parte 6: Auditoria de Segurança - "O Diário de Bordo Médico"**

### **O que é Auditoria de Segurança?**

Toda vez que o sistema verifica se um procedimento pode ser feito, ele **registra a decisão**. Isso cria um histórico completo para:

1. **Auditoria regulatória** (ANVISA, CFM)
2. **Análise de incidentes**
3. **Melhoria de regras**

### **Exemplo de Fluxo Completo**

```
Paciente João (ID: PAT123) quer fazer Hemograma hoje (2024-01-15)

1. Sistema verifica histórico do paciente:
   - Doou sangue em: 2024-01-10 (5 dias atrás)
   - Tomou vacina em: 2024-01-01 (14 dias atrás)
   - Está usando: Varfarina (anticoagulante)

2. Sistema aplica regras do Hemograma:
   - minHoursAfterBloodDonation: 72h (3 dias) FALHA
   - minDaysAfterVaccination: 7d PASSOU
   - anticoagulantes: contraindicated FALHA

3. Resultado: NÃO PERMITIDO
   Motivo: "Doação de sangue recente + uso de anticoagulante"

4. Sistema registra auditoria:
```

```json
{
  "safetyAudit": {
    "id": "345f6789-g01h-23i4-j567-890123456789",
    "patientId": "PAT123",
    "clinicalItemId": "HEMOGRAM",
    "desiredDate": "2024-01-15T09:00:00.000Z",
    "wasAllowed": false,  // Negado!
    "reason": "Doação de sangue em 2024-01-10 (necessário aguardar 72h) + paciente em anticoagulante Varfarina",
    "clinicalRuleId": "012d3456-e78f...",
    "evaluatedAt": "2024-01-15T08:30:00.000Z"
  }
}
```

### **API: Registrar Auditoria**

```bash
POST http://localhost:3000/rules/safety-audits

{
  "patientId": "PAT123",
  "clinicalItemId": "HEMOGRAM",
  "desiredDate": "2024-01-15T09:00:00.000Z",
  "wasAllowed": false,
  "reason": "Doação de sangue recente + anticoagulante",
  "clinicalRuleId": "012d3456-e78f..."  // Opcional: qual regra foi aplicada
}
```

---

## **Parte 7: Integração com Outros Sistemas (gRPC)**

### **Por que gRPC?**

Para **alta performance** quando outros sistemas precisam consultar o catálogo frequentemente:

1. **Agendamento**: Verifica disponibilidade 1000x por minuto
2. **Prontuário**: Busca informações de procedimentos
3. **Faturamento**: Obtém códigos TUSS/AMB

### **Exemplo: A Agenda Consultando Regras**

```protobuf
// A agenda pergunta: "João pode fazer Hemograma amanhã?"
rpc BatchGetCurrentRules(BatchItemRequest) returns (RuleListResponse);

message BatchItemRequest {
  repeated string ids = 1;  // IDs dos itens a verificar
}

message RuleListResponse {
  repeated RuleResponse rules = 1;
}
```

### **Chamada gRPC em Ação**

```typescript
// Código do microserviço de Agenda
const rules = await grpcClient.batchGetCurrentRules({
  ids: ['HEMOGRAM', 'COVID19_VACCINE', 'ECOCARDIO']
});

// Resultado
rules.forEach(rule => {
  console.log(`${rule.item_name}:`);
  console.log(`  Aguardar ${rule.minHoursAfterBloodDonation}h após doação`);
  console.log(`  Aguardar ${rule.minDaysAfterVaccination}d após vacina`);
  console.log(`  Anticoagulantes: ${rule.contraindicatedAnticoagulants ? 'PROIBIDO' : 'PERMITIDO'}`);
});
```

### **Endpoints gRPC Disponíveis**

| Método | Quando usar | Exemplo |
|--------|-------------|---------|
| `ListSpecialties` | Configurar agenda por especialidade | Filtrar cardiologistas |
| `ListServicesBySpecialty` | Mostrar serviços de uma área | Cardio → Consultas, Exames |
| `BatchGetItems` | Verificar múltiplos itens | Hemo+Glicemia+Colesterol |
| `GetCurrentRuleByItem` | Ver regra ativa de um item | Regra do Hemograma hoje |
| `BatchGetCurrentRules` | Verificar várias regras de uma vez | Otimização de performance |

---

## **Parte 8: Casos Reais Completos**

### **Caso 1: Paciente Diabético em Cirurgia**

**Cenário:** Maria, 65 anos, diabética, vai fazer cirurgia de catarata.

```yaml
Paciente: Maria
Condições: Diabetes tipo 2, usa insulina
Procedimento desejado: Cirurgia de Catarata (CATARACT_SURGERY)

Verificações do sistema:
1. Regra da cirurgia: anticoagulantes contraindicated
   - Maria usa AAS (anticoagulante) PERMITIDO (AAS é permitido)
   
2. Regra do diabetes: precisa glicemia < 200mg/dL
   - Glicemia de Maria: 180mg/dL PASSOU
   
3. Resultado: PERMITIDO com observação
   - "Monitorar glicemia durante procedimento"
```

### **Caso 2: Vacinação Infantil**

**Cenário:** João, 2 anos, vai tomar múltiplas vacinas.

```yaml
Paciente: João (2 anos)
Vacinas agendadas:
- Pentavalente (PENTA_VAC)
- Pneumocócica (PNEUMO_VAC)
- Influenza (FLU_VAC)

Regras aplicadas:
1. Intervalo mínimo entre vacinas: 30 dias
2. João tomou Pentavalente há 20 dias
3. Sistema sugere: "Adiar Pneumocócica por 10 dias"
```

### **Caso 3: Transfusão de Sangue Urgente**

**Cenário:** Emergência, paciente perdeu muito sangue.

```yaml
Paciente: Emergência traumática
Necessidade: Transfusão urgente (BLOOD_TRANSFUSION)

Regras normais: Verificar tipo sanguíneo, testes
Regra de emergência: "Ignorar algumas verificações"

Sistema registra:
- wasAllowed: true
- reason: "EMERGÊNCIA - Protocolo de exceção ativado"
- bypassedRules: ["tipagem_completa", "teste_cruzado"]
- authorizedBy: "dr.emergencia@hospital.com"
```

---

## **Parte 9: Boas Práticas e Armadilhas Comuns**

### **O que FAZER**

1. **Use códigos padronizados**
   ```json
   CERTO: {"code": "COVID19_VAC", "name": "Vacina COVID-19"}
   ERRADO: {"code": "vacina_covid", "name": "vacina"}
   ```

2. **Mantenha histórico de regras**
   ```bash
   # Sempre crie nova regra em vez de editar
   POST /rules (nova regra com validFrom=amanhã)
   # Em vez de
   PATCH /rules/{id} (editar regra existente)
   ```

3. **Documente mudanças**
   ```json
   {
     "createdBy": "dr.silva@hospital.com",
     "changeReason": "Baseado em estudo JAMA 2024..."
   }
   ```

### **O que NÃO FAZER**

1. **Não apague, desative**
   ```bash
   # Em vez de deletar
   DELETE /items/{id}
   
   # Faça
   PATCH /items/{id}
   {
     "isActive": false,
     "deactivationReason": "Substituído por novo protocolo"
   }
   ```

2. **Não misture tipos**
   ```json
   ERRADO: "type": "EXAME"  // Português
   CERTO: "type": "EXAM"    // Inglês (padrão ENUM)
   ```

3. **Não ignore auditoria**
   ```json
   // Sempre registre decisões
   {
     "wasAllowed": true,
     "reason": "Todas as regras atendidas",
     "clinicalRuleId": "..."  // Qual regra foi usada
   }
   ```

---

## **Parte 10: Fluxo de Trabalho Típico**

### **Passo a Passo: Adicionar Novo Exame**

1. **Verifique se a especialidade existe**
   ```bash
   GET /specialties?search=Hematologia
   ```

2. **Crie ou use serviço existente**
   ```bash
   POST /services
   {
     "specialtyId": "...",
     "name": "Exames Hematológicos Especiais",
     "type": "EXAM"
   }
   ```

3. **Crie o item clínico**
   ```bash
   POST /items
   {
     "clinicalServiceId": "...",
     "code": "FERRO_SERICO",
     "name": "Dosagem de Ferro Sérico",
     "description": "Mede quantidade de ferro no sangue"
   }
   ```

4. **Crie regras de segurança**
   ```bash
   POST /rules
   {
     "clinicalItemId": "...",
     "minHoursAfterBloodDonation": 72,
     "minDaysAfterVaccination": 7,
     "contraindicatedAnticoagulants": false,
     "validFrom": "2024-01-01",
     "createdBy": "lab.manager@hospital.com"
   }
   ```

5. **Teste integração**
   ```bash
   # Verifique se a agenda consegue acessar
   GET /rules?itemId=FERRO_SERICO&currentOnly=true
   ```

---

## **Parte 11: Monitoramento e Métricas**

### **Métricas Importantes**

1. **Utilização de itens**
   ```sql
   -- Itens mais usados
   SELECT item_code, COUNT(*) as usage_count
   FROM safety_audits
   GROUP BY item_code
   ORDER BY usage_count DESC
   ```

2. **Taxa de bloqueios**
   ```sql
   -- Quantos procedimentos foram bloqueados
   SELECT 
     SUM(CASE WHEN was_allowed = false THEN 1 ELSE 0 END) as blocked,
     COUNT(*) as total,
     (blocked * 100.0 / total) as block_percentage
   FROM safety_audits
   ```

3. **Regras mais acionadas**
   ```sql
   -- Quais regras mais impedem procedimentos
   SELECT rule_id, COUNT(*) as trigger_count
   FROM safety_audits
   WHERE was_allowed = false
   GROUP BY rule_id
   ```

### **Dashboard Recomendado**

```yaml
Dashboard do Catálogo Clínico:
- Total de especialidades: 25
- Total de serviços: 150
- Total de itens: 500
- Total de regras: 750
- Auditorias hoje: 1,234
- Bloqueios hoje: 45 (3.6%)
- Item mais bloqueado: Hemograma (12x)
- Regra mais acionada: anticoagulantes (28x)
```

---

## **Parte 12: Solução de Problemas Comuns**

### **Problema: "Item não aparece na agenda"**

**Solução passo a passo:**

1. Verifique se está ativo:
   ```bash
   GET /items/{id}
   # Verifique: "isActive": true
   ```

2. Verifique o serviço pai:
   ```bash
   GET /services/{serviceId}
   # Verifique: "isActive": true
   ```

3. Verifique a especialidade:
   ```bash
   GET /specialties/{specialtyId}
   # Verifique: "isActive": true
   ```

### **Problema: "Regra não está sendo aplicada"**

1. Verifique validade:
   ```bash
   GET /rules?itemId=ITEM_ID&validAt=2024-01-15
   ```

2. Verifique se há regra atual:
   ```bash
   GET /rules?itemId=ITEM_ID&currentOnly=true
   ```

3. Verifique auditoria:
   ```bash
   GET /rules/safety-audits?itemId=ITEM_ID&startDate=2024-01-01
   ```

### **Problema: "Código já existe"**

```bash
# Busque o código conflitante
GET /specialties?search=CODE
GET /services?search=CODE  
GET /items?search=CODE

# Use prefixos únicos
{
  "code": "CARDIOL_CONSULT_2024"  // Adicione ano/versão
}
```

---

## **Parte 13: Glossário Técnico-Clínico**

| Termo | Significado Técnico | Exemplo |
|-------|-------------------|---------|
| **Especialidade** | Departamento médico | Cardiologia |
| **Serviço Clínico** | Categoria de atendimento | Consultas, Exames |
| **Item Clínico** | Procedimento concreto | Hemograma, Vacina COVID |
| **Regra Clínica** | Restrição de segurança | Esperar 30d após cirurgia |
| **ValidFrom/ValidTo** | Vigência da regra | 2024-01-01 a 2024-12-31 |
| **Safety Audit** | Registro de decisão | "Permitido porque..." |
| **Contraindicação** | Razão para não fazer | Anticoagulantes + cirurgia |
| **Batch Request** | Múltiplos itens de uma vez | [Hemo, Glicemia, Colesterol] |

---

## **Suporte e Contato**

### **Canais de Suporte**

1. **Documentação online**: `https://docs.hospital.com/catalogo`
2. **API Playground**: `https://api.hospital.com/swagger`
3. **Suporte técnico**: `suporte-catalogo@hospital.com`
4. **Urgências clínicas**: `emergencia-regras@hospital.com`

### **Escalação de Problemas**

```
Nível 1: Desenvolvedor → Verificar logs e documentação
Nível 2: Administrador → Verificar configurações
Nível 3: Clínico → Verificar regras médicas
Nível 4: Comitê de Segurança → Revisar protocolos
```

---

## **Conclusão**

Este catálogo clínico é o **sistema nervoso central** da segurança médica do hospital. Ele:

**Padroniza** todos os procedimentos
**Protege** pacientes com regras automáticas
**Audita** todas as decisões
**Integra-se** com todos os outros sistemas

**Lembre-se:** Um catálogo bem mantido previne erros médicos, facilita auditorias e salva vidas.

---

*Documento atualizado em: 15 de janeiro de 2024*  
*Versão: 2.1.0*  
*Próxima revisão: 15 de julho de 2024*  

---

## **Checklist de Implementação**

- [ ] Todas especialidades cadastradas com códigos únicos
- [ ] Serviços organizados por tipo (CONSULTATION, EXAM, etc.)
- [ ] Itens clínicos com descrições claras
- [ ] Regras com todas as 4 medidas de segurança
- [ ] Auditoria ativada para todas as decisões
- [ ] Integração gRPC testada com Agenda
- [ ] Backup diário do catálogo
- [ ] Equipe treinada para manutenção

---

**Próximo capítulo:** [Como a Agenda usa o Catálogo via gRPC para decisões em tempo real](https://docs.hospital.com/agenda-integracao)
