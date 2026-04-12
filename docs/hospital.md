# API de Gestão Hospitalar - Organizacional e Estrutural

## Propósito do Microsserviço
> Este microsserviço é responsável por definir, validar e manter a **Estrutura organizacional e física das instituições de saúde**, atuando como fonte única da verdade para todos os outros serviços do ecossistema hospitalar.

## Serviços Consumidores
- **Gestão de Usuários** – Definição de escopos de acesso  
- **Marcação de Consultas** – Utilização da estrutura física  
- **Agenda Médica** – Vínculo de profissionais a departamentos  
- **Internamento** – Gestão de leitos e salas  
- **Prontuário Clínico** – Associação de pacientes à estrutura  
- **Faturação** – Vínculo de serviços a departamentos  
- **Inventário** – Controle de estoques por setor 

## DOMÍNIOS DO SISTEMA

### 1. Domínio Organizacional
> Representa a estrutura administrativa e funcional da instituição.

```
HOSPITAL
├── Entidade legal
├── Identificadores únicos (NIF, etc.)
├── Contas e permissões
│
├── UNIDADES
│   ├── MAIN    (Sede principal)
│   ├── BRANCH  (Filiais)
│   └── CLINIC  (Clínicas especializadas)
│
└── DEPARTAMENTOS
    ├── Pediatria
    ├── Cardiologia
    ├── Administração
    └── Suporte Técnico
```

---

### 2. Domínio Estrutural (Físico)
Representa a infraestrutura física da instituição.

```
HOSPITAL
└── UNIDADE
    └── EDIFÍCIO
        ├── Bloco Clínico
        ├── Bloco Administrativo
        └── Centro Cirúrgico
            └── PISO
                ├── Térreo
                ├── Primeiro Andar
                └── Segundo Andar
                    └── SETOR
                        ├── CARE        (Atendimento)
                        ├── SUPPORT     (Apoio)
                        ├── ADMIN       (Administrativo)
                        └── TECHNICAL   (Técnico)
                            └── SALA
                                ├── Consulta
                                ├── Internamento
                                ├── Cirurgia
                                └── Observação
                                    └── CAMA
                                        ├── STANDARD
                                        ├── ICU
                                        ├── NEONATAL
                                        └── RECOVERY
```

---

## HOSPITAL – REGRAS DETALHADAS

### Definição e Identidade Legal
Um hospital é uma instituição de saúde legalmente reconhecida, com personalidade jurídica própria.

### Estados Operacionais
- **PENDING** – Cadastro inicial
- **ACTIVE** – Operacional e validado
- **SUSPENDED** – Operações bloqueadas por infração
- **INACTIVE** – Encerramento temporário
- **ARCHIVED** – Exclusão permanente


### Identificadores Únicos

| Identificador        | Obrigatório Para     | Unicidade       | Validação |
|----------------------|----------------------|-----------------|-----------|
| NIF                  | Hospitais privados   | Global          | Formato Angola |
| Email                | Opcional             | Por hospital    | Email válido |
| Registro Comercial   | Opcional             | Global          | Alfanumérico |
| Licença Sanitária    | Opcional             | Global          | Por país |

---

## UNIDADES HOSPITALARES – REGRAS DETALHADAS

### Hierarquia Organizacional – Exemplo

```
HOSPITAL GERAL DE LUANDA
├── UNIDADE PRINCIPAL (MAIN)
│   ├── Edifício A – Bloco Clínico
│   ├── Edifício B – Administração
│   └── Edifício C – Emergência
├── CLÍNICA DO CAZENGA (BRANCH)
│   ├── Ambulatório Geral
│   ├── Pediatria
│   └── Laboratório
└── CENTRO DE DIAGNÓSTICO (CLINIC)
    ├── Radiologia
    ├── Tomografia
    └── Ultrassonografia
```

### Tipos de Unidade

| Tipo   | Descrição           | Exemplo |
|-------|---------------------|---------|
| MAIN  | Sede principal       | HGL Matriz |
| BRANCH| Filial completa      | HGL Cazenga |
| CLINIC| Unidade especializada| Centro Diagnóstico HGL |



## 3 Desativação de Unidades com Validação de Dependências

Antes de desativar uma unidade hospitalar, o sistema deve validar **todas as dependências operacionais**.

### Regras de Validação
1. Não podem existir camas ocupadas  
2. Não podem existir contas ativas vinculadas  
3. Todas as dependências estruturais devem estar livres  

---

## 4. ESTRUTURA FÍSICA – REGRAS DETALHADAS

### 4.1 Hierarquia Estrita – Exemplo Completo

```
HOSPITAL: Clínica Multiperfil
|
└── UNIDADE: Sede Central
    |
    └── EDIFÍCIO: Torre Médica (TORRE-A)
        |
        ├── PISO: Térreo
        |   |
        │   └── SETOR: Recepção (ADMIN)
        │       └── SALA: Recepção 01
        |
        ├── PISO: 1º Andar
        |   |
        │   └── SETOR: Consultórios (CARE)
        │       ├── SALA: Consultório 101
        │       └── SALA: Consultório 102
        |
        └── PISO: 2º Andar
            |
            └── SETOR: Enfermaria (CARE)
                ├── SALA: Enfermaria 201
                │   ├── CAMA: 201-BED-01
                │   ├── CAMA: 201-BED-02
                │   └── CAMA: 201-BED-03
                |
                └── SALA: Isolamento 202
                    └── CAMA: 202-BED-01
```


## 5. MODELO DE CONTAS HOSPITALARES

---

## O que é uma Conta

Uma conta (**HospitalAccount**) representa **como um usuário atua dentro de um hospital específico**.

Um mesmo usuário pode possuir **várias contas**, desde que cada conta esteja vinculada a **hospitais diferentes**.  
As contas são completamente isoladas entre si em termos de permissões, responsabilidades e auditoria.

### Exemplo Prático


Usuário: João Silva (userId = 42)

### Hospital Geral
```
- Conta: ADMIN-001
- Função: Administrador
```

### Clínica Privada
```
- Conta: DOC-014
- Função: Médico
```

Apesar de ser a mesma pessoa, **os acessos, poderes e responsabilidades não se misturam**.

---

## 5.2 Estado da Conta

Cada conta possui um estado operacional que define se ela pode executar ações no sistema.

| Estado     | Descrição |
|-----------|-----------|
| ACTIVE    | Conta ativa, pode executar ações normalmente |
| SUSPENDED | Conta existente, mas com operações bloqueadas |
| CLOSED    | Conta encerrada definitivamente |

Contas nos estados **SUSPENDED** ou **CLOSED**:
- Não podem criar, alterar ou aprovar dados
- Mantêm todo o histórico para fins de auditoria

---

## 5.3 Papéis da Conta

Os papéis definem a **função organizacional** exercida pela conta dentro do hospital.

Eles representam cargos reais, tais como:
- Médico
- Enfermeiro
- Gestor de Unidade
- Administrador Hospitalar

Uma conta pode possuir **um ou vários papéis simultaneamente**.

### Exemplo

```
Conta: Maria Fernandes

Papéis:
- Médica
- Chefe do Departamento de Cardiologia
```

---

## 5.4 Permissões

As permissões representam **ações técnicas** que podem ser executadas no sistema.

Cada permissão é composta por:
- Um recurso (ex: PATIENT, APPOINTMENT, BED)
- Uma ação (ex: READ, CREATE, UPDATE, DELETE)

### Princípios Importantes
- Permissões **não são atribuídas diretamente às contas**
- Permissões são sempre atribuídas aos **papéis**
- As contas herdam permissões através dos papéis

### Exemplo

```
Papel: Médico

Permissões:
- PATIENT: READ
- APPOINTMENT: READ
- APPOINTMENT: CREATE
- PRESCRIPTION: CREATE
```

---

## 5.5 Acesso por Unidade

O acesso por unidade define **em quais unidades hospitalares a conta pode atuar**.

Uma unidade pode ser:
- Unidade principal
- Filial
- Clínica associada

Cada vínculo de acesso possui um **nível de autorização**:

| Nível | Descrição |
|-----|----------|
| VIEW | Apenas visualização |
| EDIT | Criação e edição |
| ADMIN | Gestão completa da unidade |

### Exemplo

```
Conta: Gestor Operacional

Unidade Central: ADMIN
Unidade Filial: VIEW
```

Mesmo com permissões elevadas, a conta **só pode atuar nas unidades explicitamente autorizadas**.

---

## 5.6 Acesso por Departamento

Além do acesso por unidade, a conta pode ter **restrições adicionais por departamento**.

Departamentos representam áreas funcionais, como:
- Emergência
- Internamento
- Laboratório
- Administração

O acesso por departamento atua como um **filtro adicional de segurança**.

### Exemplo

```
Conta: Enfermeira

Unidade: Hospital Central
Departamentos:
- Emergência (EDIT)
- Internamento (VIEW)
```

Mesmo dentro da mesma unidade, a conta **só pode atuar nos departamentos autorizados**.

---

## 5.7 Como o Sistema Decide se uma Ação é Permitida

Uma ação só é permitida se **todas as condições abaixo forem verdadeiras**:

1. A conta está no estado ACTIVE
2. A conta possui pelo menos um papel válido
3. O papel possui a permissão necessária
4. A conta tem acesso à unidade envolvida
5. Se aplicável, a conta tem acesso ao departamento envolvido

Se qualquer uma dessas condições falhar, a ação é automaticamente bloqueada.

---

## Exemplo Completo de Avaliação

Um médico tenta criar uma prescrição:

- Conta: ACTIVE
- Papel: Médico
- Permissão: PRESCRIPTION: CREATE
- Unidade: Autorizada
- Departamento: Autorizado

Resultado: **Operação permitida**

Se o mesmo médico tentar criar uma prescrição em um departamento não autorizado:

Resultado: **Operação bloqueada**


## 6. INTERNAMENTO DE PACIENTES – REGRAS DE NEGÓCIO

### 6.1 Conceito de Internamento

O internamento representa o processo pelo qual um paciente é admitido formalmente para permanência contínua dentro da instituição hospitalar, ocupando um leito físico específico (cama) por um período determinado ou indeterminado.

O internamento não pertence a este microsserviço, mas depende integralmente dele, pois utiliza:

- Estrutura física (unidade, edifício, piso, setor, sala, cama)
- Regras de disponibilidade de leitos
- Restrições organizacionais e operacionais

Este microsserviço atua como fonte única da verdade sobre:

- Existência da cama
- Estado de ocupação
- Localização física exata
- Capacidade estrutural disponível

---

### 6.2 Pré-requisitos Obrigatórios para Internamento

Antes de um internamento ser autorizado, todas as condições abaixo devem ser satisfeitas:

1. O hospital deve estar no estado **ACTIVE**
2. A unidade hospitalar deve estar **ativa**
3. A estrutura física deve existir e estar íntegra:
   - Edifício ativo
   - Piso válido
   - Setor válido
   - Sala válida
4. A sala deve ser adequada ao tipo de internamento
5. A cama deve:
   - Existir
   - Estar ativa
   - Estar disponível (não ocupada, não bloqueada)
6. A conta que executa a ação deve possuir:
   - Acesso à unidade
   - Acesso ao departamento de internamento
   - Permissão para gerir leitos ou internamentos

Se qualquer uma dessas validações falhar, o internamento é automaticamente rejeitado.

---

### 6.3 Localização Física do Internamento

Todo internamento está sempre associado a uma cama específica, e a cama está rigidamente localizada dentro da hierarquia física.

Exemplo real:

```
HOSPITAL: Hospital Geral de Luanda
└── UNIDADE: Unidade Principal
    └── EDIFÍCIO: Bloco Clínico
        └── PISO: 2º Andar
            └── SETOR: Enfermaria Geral
                └── SALA: Enfermaria 201
                    └── CAMA: 201-BED-02
```

Essa hierarquia não pode ser quebrada.  
Uma cama não existe fora de uma sala, e uma sala não existe fora de um setor válido.

---

### 6.4 Estados de uma Cama no Contexto de Internamento

Embora o modelo físico registre apenas se a cama está ativa, o estado operacional da cama é interpretado no contexto do internamento.

Estados lógicos possíveis:

| Estado Lógico | Descrição |
|-------------|----------|
| AVAILABLE   | Cama ativa e livre |
| OCCUPIED    | Cama ocupada por um paciente |
| MAINTENANCE | Temporariamente indisponível |
| BLOCKED     | Bloqueada por decisão administrativa |

O estado **OCCUPIED** é determinado pelo serviço de internamento, mas validado contra esta estrutura.

---

### 6.5 Processo de Internamento – Fluxo Completo

#### Passo a Passo Conceitual

1. O serviço de internamento solicita a lista de camas disponíveis
2. O sistema filtra camas por:
   - Unidade
   - Departamento
   - Tipo de cama (STANDARD, ICU, etc.)
3. Uma cama é selecionada
4. O sistema valida novamente sua disponibilidade
5. O internamento é criado no serviço clínico
6. A cama passa ao estado ocupada
7. A estrutura física permanece bloqueada até a alta

---

### 6.6 Troca de Cama Durante Internamento

Um paciente pode ser transferido para outra cama durante o internamento.

Regras obrigatórias:

- A nova cama deve estar disponível
- A cama atual deve ser liberada
- A troca deve ocorrer de forma atômica
- Não pode existir intervalo onde o paciente fique sem cama atribuída

Exemplo:

```
Paciente: João Silva

ANTES
- Sala 201 → Cama 02

DEPOIS
- Sala 202 → Cama 01
```

Ambas as operações devem ser validadas estruturalmente antes da confirmação.

---

### 6.7 Alta Hospitalar

A alta hospitalar encerra o internamento e libera automaticamente a cama.

Regras:

- A cama retorna ao estado **AVAILABLE**
- O histórico de ocupação é preservado
- Nenhuma estrutura física é removida
- O internamento não pode ser reaberto

---

### 6.8 Bloqueio de Estrutura com Internamentos Ativos

Não é permitido:

- Desativar unidades
- Desativar setores
- Desativar salas
- Desativar camas

Enquanto existir qualquer internamento ativo associado.

Essa regra garante:

- Segurança do paciente
- Integridade estrutural
- Consistência operacional

---

### 6.9 Responsabilidade entre Microsserviços

| Responsabilidade | Serviço |
|------------------|---------|
| Criação do internamento | Serviço de Internamento |
| Dados clínicos | Prontuário Clínico |
| Ocupação da cama | Internamento |
| Validação estrutural | Gestão Hospitalar |
| Regras de acesso | Gestão Hospitalar |
| Auditoria | Ambos |
