# Casos de Uso de Unidades

[ homepage Modulo HospitalManagement](../index.md) · [Visao Geral](./overview.md)

Grupo de rotas: `/api/v1/hospital-management/units` e `/api/v1/hospital-management/hospitals/{hospitalId}/units` · Requer autenticacao.

## 2.1 Criar unidade

- Caso de uso: `CreateUnitCommand`
- Endpoint: `POST /api/v1/hospital-management/hospitals/{hospitalId:guid}/units`
- Finalidade: criar uma unidade dentro de um hospital
- Modelo principal: `HospitalUnit`

### Body

```json
{
  "name": "Unidade Central",
  "unitType": "Main",
  "active": true,
  "address": "Av. 4 de Fevereiro, 100",
  "city": "Luanda",
  "province": "Luanda"
}
```

`active` assume `true` por omissao. `address`, `city`, `province` sao opcionais.

### Regras de negocio

- `name` e obrigatorio (`Unit.ValidationError`)
- nao pode existir outra unidade com o mesmo nome neste hospital (`Unit.AlreadyExists`)

### Sucesso esperado

- `201 Created`
- retorno: `UnitResponse`

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `400` | `Unit.ValidationError` | nome vazio |
| `400` | `Unit.AlreadyExists` | nome duplicado no hospital |

## 2.2 Listar unidades do hospital

- Caso de uso: `GetUnitsByHospitalQuery`
- Endpoint: `GET /api/v1/hospital-management/hospitals/{hospitalId:guid}/units`

### Sucesso esperado

- `200 OK`
- retorno: array de `UnitResponse`

## 2.3 Obter unidade por id

- Caso de uso: `GetUnitByIdQuery`
- Endpoint: `GET /api/v1/hospital-management/units/{id:guid}`

### Sucesso esperado

- `200 OK`
- retorno: `UnitResponse`

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `404` | `Unit.NotFound` | unidade inexistente |

## 2.4 Atualizar unidade

- Caso de uso: `UpdateUnitCommand`
- Endpoint: `PUT /api/v1/hospital-management/units/{id:guid}`

### Body

Mesmo shape do `POST /hospitals/{hospitalId}/units`.

### Sucesso esperado

- `200 OK`
- retorno: `UnitResponse` atualizada

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `400` | `Unit.ValidationError` | nome vazio |
| `400` | `Unit.Forbidden` | operacao nao permitida no estado atual |

## 2.5 Eliminar unidade

- Caso de uso: `DeleteUnitCommand`
- Endpoint: `DELETE /api/v1/hospital-management/units/{id:guid}`

### Sucesso esperado

- `204 No Content`

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `404` | `Unit.NotFound` | unidade inexistente |

## Contrato `UnitResponse`

```
Id, Name, UnitType, Active, Address, City, Province, HospitalId,
CreatedAt, UpdatedAt
```

---

## Navegacao

[ homepage Hospitais](./hospitals.md) · [Indice do modulo](../index.md) · [Departamentos](./departments.md)

**Neste modulo:** [Visao Geral](./overview.md) · [Modelos e Enumeracoes](./models-and-enums.md) · [Hospitais](./hospitals.md) · **Unidades** · [Departamentos](./departments.md) · [Estrutura Fisica](./physical-structure.md) · [Ofertas](./offers.md) · [Contas](./accounts.md) · [Seguranca](./security.md) · [Verificacoes](./checks.md) · [Casos Internos e Notas](./internal-and-events.md)
