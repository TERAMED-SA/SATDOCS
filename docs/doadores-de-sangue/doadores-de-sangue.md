# 🩸 Rede de Doadores de Sangue

## Visão Geral

A **Rede de Doadores de Sangue** é um microserviço do ecossistema **Baika + Saúde** que conecta pessoas e instituições que precisam de sangue a doadores disponíveis, de forma segura, auditável e ética.

O sistema funciona como uma **bolsa de sangue digital**, permitindo que hospitais, clínicas e famílias encontrem rapidamente doadores compatíveis.

---

## O problema que o sistema resolve

Quando um hospital precisa de sangue, o processo tradicional envolve:
- Telefonar para familiares
- Procurar doadores manualmente
- Usar grupos informais (WhatsApp, chamadas)

Isso consome tempo crítico.

Com a Rede de Doadores:

> “Preciso de 2 unidades de O− em Luanda” → o sistema encontra doadores compatíveis automaticamente.

---

## Conceitos principais

| Conceito | Descrição |
|--------|---------|
| **Doador** | Pessoa que pode doar sangue |
| **Pedido** | Solicitação de sangue feita por um hospital ou paciente |
| **Match** | Tentativa de ligar um doador a um pedido |
| **Doação** | Registro de que o sangue foi efetivamente doado |

---

## Doador

Um doador é um usuário que disponibiliza seu sangue para ajudar outras pessoas.

Cada doador informa:
- Tipo sanguíneo (A+, O−, etc.)
- Se é voluntário ou remunerado
- Localização (bairro / município)
- Frequência disponível

O sistema controla:
- Data da última doação
- Disponibilidade
- Histórico de doações
- Bloqueios ou restrições

> O sistema nunca armazena dados clínicos além do necessário para garantir segurança.

---

## 🏥 Pedido de sangue

Um pedido representa uma necessidade real de sangue.

Pode ser criado por:
- Um hospital
- Uma clínica
- Um familiar de um paciente

Cada pedido inclui:
- Tipo sanguíneo necessário
- Quantidade de bolsas
- Local da coleta
- Grau de urgência
- Observações (ex: nome do hospital)

### Exemplo real

> O Hospital Central de Luanda precisa de **3 bolsas de A−** para uma cirurgia de emergência.

Esse pedido entra no sistema como **URGENTE**.

---

## Match (ligação entre doador e pedido)

Quando existe um pedido, o sistema procura doadores compatíveis.

Um **match** é criado quando:
- O tipo sanguíneo é compatível
- O doador está ativo
- O intervalo mínimo desde a última doação foi respeitado

O doador pode:
- Aceitar
- Recusar
- Negociar valor (se for remunerado)

Um doador só pode ter **um match ativo** de cada vez.

---

## 🩸 Doação

Uma doação é registrada quando:
- Um match foi aceito
- O doador compareceu
- O sangue foi coletado

A doação pode ser confirmada:
- Pelo próprio doador
- Pelo hospital

Após a doação:
- O doador entra em período de descanso
- O pedido é atualizado
- O histórico é registrado

---

## Regras médicas

| Regra | Valor |
|------|------|
Intervalo mínimo entre doações | 56 dias  
Máximo de bolsas por doação | 5  
Máximo de matches ativos por doador | 1  

Pedidos expiram conforme a urgência:

| Urgência | Expira em |
|--------|---------|
URGENTE | 24 horas  
ALTA | 3 dias  
MÉDIA | 7 dias  
BAIXA | 14 dias  

---

## Segurança e ética

O sistema implementa:

- Verificação de identidade
- Histórico de comportamento
- Sistema de denúncias
- Bloqueio automático em casos de abuso
- Auditoria completa

Nenhuma negociação ocorre fora do sistema.

---

## Painel administrativo

A equipa do Baika + Saúde pode visualizar:
- Doadores ativos por região
- Pedidos em aberto
- Estatísticas de doações
- Histórico de abuso e bloqueios

---

## Compatibilidade sanguínea

O sistema respeita as regras médicas de compatibilidade:

Exemplo:
- O− pode doar para todos
- AB+ só recebe de AB+

A correspondência é sempre segura.

---

## Impacto social

A Rede de Doadores:
- Salva vidas
- Reduz tempo de resposta em emergências
- Promove solidariedade
- Cria confiança entre hospitais e cidadãos

---

## Princípios do sistema

1. **Vidas primeiro**
2. **Dados mínimos**
3. **Transparência**
4. **Segurança**
5. **Ética médica**

> Este sistema não é um marketplace. É uma infraestrutura de saúde pública.
