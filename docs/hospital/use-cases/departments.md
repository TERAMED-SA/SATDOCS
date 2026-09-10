# Casos de Uso de Departamentos

[ homepage Modulo HospitalManagement](../index.md) · [Visao Geral](./overview.md)

Grupo de rotas: `/api/v1/hospital-management/departments` e `/api/v1/hospital-management/hospitals/{hospitalId}/departments` · Requer autenticacao.

## 3.1 Criar departamento

- Caso de uso: `CreateDepartmentCommand`
- Endpoint: `POST /api/v1/hospital-management/hospitals/{hospitalId:guid}/departments`
- Finalidade: criar um departamento dentro de um hospital
- Modelo principal: `Department`

### Body

```json
{
  "name": "Cardiologia",
  "description": "Departamento de Cardiologia",
  "active": true
}
```

`description` e opcional. `active` assume `true` por omissao.

### Regras de negocio

- `name` e obrigatorio (`Dept.ValidationError`)
- nao pode existir outro departamento com o mesmo nome neste hospital (`Dept.AlreadyExists`)

### Sucesso esperado

- `201 Created`
- retorno: `DepartmentResponse`

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `400` | `Dept.ValidationError` | nome vazio |
| `400` | `Dept.AlreadyExists` | nome duplicado no hospital |

## 3.2 Listar departamentos do hospital

- Caso de uso: `GetDepartmentsByHospitalQuery`
- Endpoint: `GET /api/v1/hospital-management/hospitals/{hospitalId:guid}/departments`

### Sucesso esperado

- `200 OK`
- retorno: array de `DepartmentResponse`

## 3.3 Obter departamento por id

- Caso de uso: `GetDepartmentByIdQuery`
- Endpoint: `GET /api/v1/hospital-management/departments/{id:guid}`

### Sucesso esperado

- `200 OK`
- retorno: `DepartmentResponse`

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `404` | `Dept.NotFound` | departamento inexistente |

## 3.4 Atualizar departamento

- Caso de uso: `UpdateDepartmentCommand`
- Endpoint: `PUT /api/v1/hospital-management/departments/{id:guid}`

### Body

Mesmo shape do `POST /hospitals/{hospitalId}/departments`.

### Sucesso esperado

- `200 OK`
- retorno: `DepartmentResponse` atualizado

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `400` | `Dept.ValidationError` | nome vazio |

## 3.5 Eliminar departamento

- Caso de uso: `DeleteDepartmentCommand`
- Endpoint: `DELETE /api/v1/hospital-management/departments/{id:guid}`

### Regras de negocio

- nao e possivel eliminar um departamento que tenha contas associadas (`Dept.LinkedToAccounts`)

### Sucesso esperado

- `204 No Content`

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `409` | `Dept.LinkedToAccounts` | departamento tem contas associadas |

## Contrato `DepartmentResponse`

```
Id, Name, Description, Active, HospitalId,
CreatedAt, UpdatedAt
```

---

## Navegacao

[ homepage Unidades](./units.md) · [Indice do modulo](../index.md) · [Estrutura Fisica](./physical-structure.md)

**Neste modulo:** [Visao Geral](./overview.md) · [Modelos e Enumeracoes](./models-and-enums.md) · [Hospitais](./hospitals.md) · [Unidades](./units.md) · **Departamentos** · [Estrutura Fisica](./physical-structure.md) · [Ofertas](./offers.md) · [Contas](./accounts.md) · [Seguranca](./security.md) · [Verificacoes](./checks.md) · [Casos Internos e Notas](./internal-and-events.md)
