# Casos de Uso de Seguranca

[ homepage Modulo HospitalManagement](../index.md) · [Visao Geral](./overview.md)

Grupo de rotas: `/api/v1/hospital-management/roles` e `/api/v1/hospital-management/permissions` · Requer autenticacao.

## Roles

### 7.1 Criar role

- Caso de uso: `CreateRoleCommand`
- Endpoint: `POST /api/v1/hospital-management/accounts/{accountId:guid}/roles`
- Modelo principal: `AccountRoleModel`

### Body

```json
{
  "roleName": "Medico Chefe",
  "baseRole": "Doctor",
  "description": "Chefe de departamento medico",
  "status": "Active"
}
```

`baseRole` e opcional (um dos valores de `AccountRole`). `description` e opcional. `status` assume `Active` por omissao.

### Regras de negocio

- `roleName` e obrigatorio e tem de ser unico por conta (`Role.AlreadyExists`)

### Sucesso esperado

- `201 Created`
- retorno: `RoleResponse`

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `400` | `Role.AlreadyExists` | nome duplicado na conta |

### 7.2 Listar roles da conta

- Caso de uso: `GetRolesByAccountQuery`
- Endpoint: `GET /api/v1/hospital-management/accounts/{accountId:guid}/roles`
- Sucesso: `200 OK` → array de `RoleResponse`

### 7.3 Obter role por id

- Caso de uso: `GetRoleByIdQuery`
- Endpoint: `GET /api/v1/hospital-management/roles/{id:guid}`
- Sucesso: `200 OK` → `RoleResponse`
- Erros: `404 Role.NotFound`

### 7.4 Atualizar role

- Caso de uso: `UpdateRoleCommand`
- Endpoint: `PUT /api/v1/hospital-management/roles/{id:guid}`
- Body: mesmo shape do POST
- Sucesso: `200 OK` → `RoleResponse`
- Erros: `400 Role.AlreadyExists`

### 7.5 Eliminar role

- Caso de uso: `DeleteRoleCommand`
- Endpoint: `DELETE /api/v1/hospital-management/roles/{id:guid}`
- Sucesso: `204 No Content`
- Erros: `400 Role.NotFound`

---

## Permissoes

### 7.6 Adicionar permissao a role

- Caso de uso: `AddPermissionCommand`
- Endpoint: `POST /api/v1/hospital-management/roles/{roleId:guid}/permissions`
- Modelo principal: `AccountRolePermission`

### Body

```json
{
  "resource": "Patient",
  "action": "Read"
}
```

### Regras de negocio

- nao e possivel duplicar uma permissao na mesma role (`Permission.Duplicate`)

### Sucesso esperado

- `201 Created`
- retorno: `PermissionResponse`

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `400` | `Permission.Duplicate` | permissao ja existe na role |

### 7.7 Listar permissoes da role

- Caso de uso: `GetPermissionsByRoleQuery`
- Endpoint: `GET /api/v1/hospital-management/roles/{roleId:guid}/permissions`
- Sucesso: `200 OK` → array de `PermissionResponse`

### 7.8 Remover permissao

- Caso de uso: `RemovePermissionCommand`
- Endpoint: `DELETE /api/v1/hospital-management/permissions/{id:guid}`
- Sucesso: `204 No Content`
- Erros: `400 Permission.NotFound`

## Contratos

### `RoleResponse`

```
Id, RoleName, Description, Status, AccountId
```

### `PermissionResponse`

```
Id, Resource, Action
```

---

## Navegacao

[ homepage Contas](./accounts.md) · [Indice do modulo](../index.md) · [Verificacoes](./checks.md)

**Neste modulo:** [Visao Geral](./overview.md) · [Modelos e Enumeracoes](./models-and-enums.md) · [Hospitais](./hospitals.md) · [Unidades](./units.md) · [Departamentos](./departments.md) · [Estrutura Fisica](./physical-structure.md) · [Ofertas](./offers.md) · [Contas](./accounts.md) · **Seguranca** · [Verificacoes](./checks.md) · [Casos Internos e Notas](./internal-and-events.md)
