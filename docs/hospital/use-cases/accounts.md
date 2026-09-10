# Casos de Uso de Contas

[ homepage Modulo HospitalManagement](../index.md) · [Visao Geral](./overview.md)

Grupo de rotas: `/api/v1/hospital-management/accounts` e `/api/v1/hospital-management/hospitals/{hospitalId}/accounts` · Requer autenticacao.

## 6.1 Criar conta de hospital

- Caso de uso: `CreateAccountCommand`
- Endpoint: `POST /api/v1/hospital-management/hospitals/{hospitalId:guid}/accounts`
- Finalidade: associar um utilizador a um hospital
- Modelo principal: `HospitalAccount`

### Body

```json
{
  "userId": "uuid-string",
  "accountName": "Dr. Joao Silva",
  "accountCode": "DOC-001",
  "status": "Active"
}
```

`status` assume `Active` por omissao.

### Regras de negocio

- `userId` e obrigatorio (`Account.UserIdRequired`)
- `accountName` e obrigatorio (`Account.AccountNameRequired`)
- `accountCode` e obrigatorio (`Account.AccountCodeRequired`) e tem de ser unico por hospital (`Account.AlreadyExists`)
- o hospital tem de estar ativo (`Account.HospitalNotActive`)

### Sucesso esperado

- `201 Created`
- retorno: `HospitalAccountResponse`

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `400` | `Account.UserIdRequired` | userId vazio |
| `400` | `Account.AccountNameRequired` | accountName vazio |
| `400` | `Account.AccountCodeRequired` | accountCode vazio |
| `400` | `Account.AlreadyExists` | accountCode duplicado no hospital |
| `400` | `Account.HospitalNotActive` | hospital nao esta ativo |

## 6.2 Listar contas do hospital

- Caso de uso: `GetAccountsByHospitalQuery`
- Endpoint: `GET /api/v1/hospital-management/hospitals/{hospitalId:guid}/accounts`
- Sucesso: `200 OK` → array de `HospitalAccountResponse`

## 6.3 Obter conta por id

- Caso de uso: `GetAccountByIdQuery`
- Endpoint: `GET /api/v1/hospital-management/accounts/{id:guid}`
- Sucesso: `200 OK` → `HospitalAccountResponse`
- Erros: `404 Account.NotFound`

## 6.4 Atualizar conta

- Caso de uso: `UpdateAccountCommand`
- Endpoint: `PUT /api/v1/hospital-management/accounts/{id:guid}`
- Body: mesmo shape do POST
- Sucesso: `200 OK` → `HospitalAccountResponse`
- Erros: `400 Account.ValidationError`

## 6.5 Eliminar conta

- Caso de uso: `DeleteAccountCommand`
- Endpoint: `DELETE /api/v1/hospital-management/accounts/{id:guid}`
- Sucesso: `204 No Content`
- Erros: `404 Account.NotFound`

---

## Scopes de Conta

### 6.6 Atribuir scope de unidade

- Caso de uso: `AddUnitScopeCommand`
- Endpoint: `POST /api/v1/hospital-management/accounts/{accountId:guid}/units`

### Body

```json
{
  "unitId": "uuid",
  "accessLevel": "Edit"
}
```

### Sucesso esperado

- `201 Created`
- retorno: `IdResponse`

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `400` | `Scope.Duplicate` | scope ja existe |
| `400` | `Scope.Forbidden` | operacao nao permitida |

### 6.7 Remover scope de unidade

- Caso de uso: `RemoveUnitScopeCommand`
- Endpoint: `DELETE /api/v1/hospital-management/accounts/{accountId:guid}/units/{unitId:guid}`
- Sucesso: `204 No Content`
- Erros: `404 Scope.NotFound`

### 6.8 Atribuir scope de departamento

- Caso de uso: `AddDepartmentScopeCommand`
- Endpoint: `POST /api/v1/hospital-management/accounts/{accountId:guid}/departments`

### Body

```json
{
  "departmentId": "uuid",
  "accessLevel": "View"
}
```

### Sucesso esperado

- `201 Created`
- retorno: `IdResponse`

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `400` | `Scope.Duplicate` | scope ja existe |
| `400` | `Scope.Forbidden` | operacao nao permitida |

### 6.9 Remover scope de departamento

- Caso de uso: `RemoveDepartmentScopeCommand`
- Endpoint: `DELETE /api/v1/hospital-management/accounts/{accountId:guid}/departments/{departmentId:guid}`
- Sucesso: `204 No Content`
- Erros: `404 Scope.NotFound`

## Contrato `HospitalAccountResponse`

```
Id, AccountName, AccountCode, Status, HospitalId
```

---

## Navegacao

[ homepage Ofertas](./offers.md) · [Indice do modulo](../index.md) · [Seguranca](./security.md)

**Neste modulo:** [Visao Geral](./overview.md) · [Modelos e Enumeracoes](./models-and-enums.md) · [Hospitais](./hospitals.md) · [Unidades](./units.md) · [Departamentos](./departments.md) · [Estrutura Fisica](./physical-structure.md) · [Ofertas](./offers.md) · **Contas** · [Seguranca](./security.md) · [Verificacoes](./checks.md) · [Casos Internos e Notas](./internal-and-events.md)
