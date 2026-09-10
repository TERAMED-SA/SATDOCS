# Casos de Uso de Estrutura Fisica

[ homepage Modulo HospitalManagement](../index.md) · [Visao Geral](./overview.md)

Grupo de rotas: `/api/v1/hospital-management` (rotas de buildings, floors, sectors, rooms, beds) · Requer autenticacao.

A estrutura fisica segue a hierarquia:
**Unidade > Edificio > Piso > Setor > Sala > Cama**

## Edificios

### 4.1 Criar edificio

- Caso de uso: `CreateBuildingCommand`
- Endpoint: `POST /api/v1/hospital-management/units/{unitId:guid}/buildings`
- Modelo principal: `Building`

### Body

```json
{
  "name": "Edificio Principal",
  "code": "ED-01"
}
```

### Regras de negocio

- `name` e `code` sao obrigatorios (`Structure.ValidationError`)

### Sucesso esperado

- `201 Created`
- retorno: `BuildingResponse`

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `400` | `Structure.ValidationError` | name ou code vazio |

### 4.2 Listar edificios da unidade

- Caso de uso: `GetBuildingsByUnitQuery`
- Endpoint: `GET /api/v1/hospital-management/units/{unitId:guid}/buildings`
- Sucesso: `200 OK` → array de `BuildingResponse`

### 4.3 Obter edificio por id

- Caso de uso: `GetBuildingByIdQuery`
- Endpoint: `GET /api/v1/hospital-management/buildings/{id:guid}`
- Sucesso: `200 OK` → `BuildingResponse`
- Erros: `404 Structure.NotFound`

### 4.4 Atualizar edificio

- Caso de uso: `UpdateBuildingCommand`
- Endpoint: `PUT /api/v1/hospital-management/buildings/{id:guid}`
- Body: mesmo shape do POST
- Sucesso: `200 OK` → `BuildingResponse`
- Erros: `400 Structure.ValidationError`

### 4.5 Eliminar edificio

- Caso de uso: `DeleteBuildingCommand`
- Endpoint: `DELETE /api/v1/hospital-management/buildings/{id:guid}`
- Sucesso: `204 No Content`
- Erros: `409 Structure.ValidationError`

---

## Pisos

### 4.6 Criar piso

- Caso de uso: `CreateFloorCommand`
- Endpoint: `POST /api/v1/hospital-management/buildings/{buildingId:guid}/floors`
- Modelo principal: `Floor`

### Body

```json
{
  "floorLevel": 1
}
```

`floorLevel` tem de estar entre -3 e 50.

### Sucesso esperado

- `201 Created`
- retorno: `FloorResponse`

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `400` | `Structure.ValidationError` | floorLevel fora de rango |

### 4.7 Listar pisos do edificio

- Caso de uso: `GetFloorsByBuildingQuery`
- Endpoint: `GET /api/v1/hospital-management/buildings/{buildingId:guid}/floors`
- Sucesso: `200 OK` → array de `FloorResponse`

### 4.8 Obter piso por id

- Caso de uso: `GetFloorByIdQuery`
- Endpoint: `GET /api/v1/hospital-management/floors/{id:guid}`
- Sucesso: `200 OK` → `FloorResponse`
- Erros: `404 Structure.NotFound`

### 4.9 Atualizar piso

- Caso de uso: `UpdateFloorCommand`
- Endpoint: `PUT /api/v1/hospital-management/floors/{id:guid}`
- Body: mesmo shape do POST
- Sucesso: `200 OK` → `FloorResponse`
- Erros: `400 Structure.ValidationError`

### 4.10 Eliminar piso

- Caso de uso: `DeleteFloorCommand`
- Endpoint: `DELETE /api/v1/hospital-management/floors/{id:guid}`
- Sucesso: `204 No Content`
- Erros: `409 Structure.ValidationError`

---

## Setores

### 4.11 Criar setor

- Caso de uso: `CreateSectorCommand`
- Endpoint: `POST /api/v1/hospital-management/floors/{floorId:guid}/sectors`
- Modelo principal: `Sector`

### Body

```json
{
  "name": "Enfermaria Geral",
  "sectorType": "Care",
  "description": "Setor de enfermaria geral"
}
```

`sectorType` e `description` sao opcionais.

### Sucesso esperado

- `201 Created`
- retorno: `SectorResponse`

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `400` | `Structure.ValidationError` | name vazio |

### 4.12 Listar setores do piso

- Caso de uso: `GetSectorsByFloorQuery`
- Endpoint: `GET /api/v1/hospital-management/floors/{floorId:guid}/sectors`
- Sucesso: `200 OK` → array de `SectorResponse`

### 4.13 Obter setor por id

- Caso de uso: `GetSectorByIdQuery`
- Endpoint: `GET /api/v1/hospital-management/sectors/{id:guid}`
- Sucesso: `200 OK` → `SectorResponse`
- Erros: `404 Structure.NotFound`

### 4.14 Atualizar setor

- Caso de uso: `UpdateSectorCommand`
- Endpoint: `PUT /api/v1/hospital-management/sectors/{id:guid}`
- Body: mesmo shape do POST
- Sucesso: `200 OK` → `SectorResponse`
- Erros: `400 Structure.ValidationError`

### 4.15 Eliminar setor

- Caso de uso: `DeleteSectorCommand`
- Endpoint: `DELETE /api/v1/hospital-management/sectors/{id:guid}`
- Sucesso: `204 No Content`
- Erros: `409 Structure.ValidationError`

---

## Salas

### 4.16 Criar sala

- Caso de uso: `CreateRoomCommand`
- Endpoint: `POST /api/v1/hospital-management/sectors/{sectorId:guid}/rooms`
- Modelo principal: `Room`

### Body

```json
{
  "name": "Sala 101",
  "purpose": "Consultation",
  "capacity": 1,
  "active": true,
  "description": "Sala de consulta geral"
}
```

`active` assume `true` por omissao. `description` e opcional. `capacity` tem de ser >= 1.

### Sucesso esperado

- `201 Created`
- retorno: `RoomResponse`

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `400` | `Structure.ValidationError` | name vazio ou capacity < 1 |

### 4.17 Listar salas do setor

- Caso de uso: `GetRoomsBySectorQuery`
- Endpoint: `GET /api/v1/hospital-management/sectors/{sectorId:guid}/rooms`
- Sucesso: `200 OK` → array de `RoomResponse`

### 4.18 Obter sala por id

- Caso de uso: `GetRoomByIdQuery`
- Endpoint: `GET /api/v1/hospital-management/rooms/{id:guid}`
- Sucesso: `200 OK` → `RoomResponse`
- Erros: `404 Structure.NotFound`

### 4.19 Atualizar sala

- Caso de uso: `UpdateRoomCommand`
- Endpoint: `PUT /api/v1/hospital-management/rooms/{id:guid}`
- Body: mesmo shape do POST
- Sucesso: `200 OK` → `RoomResponse`
- Erros: `400 Structure.ValidationError`

### 4.20 Eliminar sala

- Caso de uso: `DeleteRoomCommand`
- Endpoint: `DELETE /api/v1/hospital-management/rooms/{id:guid}`
- Sucesso: `204 No Content`
- Erros: `409 Structure.ValidationError`

---

## Camas

### 4.21 Criar cama

- Caso de uso: `CreateBedCommand`
- Endpoint: `POST /api/v1/hospital-management/rooms/{roomId:guid}/beds`
- Modelo principal: `Bed`

### Body

```json
{
  "bedCode": "ED-01-101-A",
  "bedType": "Standard",
  "status": "Available",
  "active": true
}
```

`status` assume `Available` por omissao. `active` assume `true` por omissao.

### Sucesso esperado

- `201 Created`
- retorno: `BedResponse`

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `400` | `Structure.ValidationError` | bedCode vazio |

### 4.22 Listar camas da sala

- Caso de uso: `GetBedsByRoomQuery`
- Endpoint: `GET /api/v1/hospital-management/rooms/{roomId:guid}/beds`
- Sucesso: `200 OK` → array de `BedResponse`

### 4.23 Obter cama por id

- Caso de uso: `GetBedByIdQuery`
- Endpoint: `GET /api/v1/hospital-management/beds/{id:guid}`
- Sucesso: `200 OK` → `BedResponse`
- Erros: `404 Structure.NotFound`

### 4.24 Atualizar cama

- Caso de uso: `UpdateBedCommand`
- Endpoint: `PUT /api/v1/hospital-management/beds/{id:guid}`
- Body: mesmo shape do POST
- Sucesso: `200 OK` → `BedResponse`
- Erros: `400 Structure.ValidationError`

### 4.25 Eliminar cama

- Caso de uso: `DeleteBedCommand`
- Endpoint: `DELETE /api/v1/hospital-management/beds/{id:guid}`

### Regras de negocio

- nao e possivel eliminar uma cama em estado `Occupied` (`Structure.Forbidden`)

### Sucesso esperado

- `204 No Content`

### Erros esperados

| HTTP | Code | Situacao |
|---|---|---|
| `400` | `Structure.Forbidden` | cama ocupada |
| `409` | `Structure.ValidationError` | erro de validacao |

## Contratos

### `BuildingResponse`

```
Id, Name, Code, UnitId
```

### `FloorResponse`

```
Id, FloorLevel, BuildingId
```

### `SectorResponse`

```
Id, Name, Description, SectorType, FloorId
```

### `RoomResponse`

```
Id, Name, Purpose, Capacity, Active, Description, SectorId
```

### `BedResponse`

```
Id, BedCode, BedType, Status, Active, RoomId
```

---

## Navegacao

[ homepage Departamentos](./departments.md) · [Indice do modulo](../index.md) · [Ofertas](./offers.md)

**Neste modulo:** [Visao Geral](./overview.md) · [Modelos e Enumeracoes](./models-and-enums.md) · [Hospitais](./hospitals.md) · [Unidades](./units.md) · [Departamentos](./departments.md) · **Estrutura Fisica** · [Ofertas](./offers.md) · [Contas](./accounts.md) · [Seguranca](./security.md) · [Verificacoes](./checks.md) · [Casos Internos e Notas](./internal-and-events.md)
