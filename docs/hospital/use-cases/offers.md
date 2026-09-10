# Casos de Uso de Ofertas

[ homepage Modulo HospitalManagement](../index.md) · [Visao Geral](./overview.md)

Grupo de rotas: `/api/v1/hospital-management/hospitals/{hospitalId}/offered-items` e `/api/v1/hospital-management/hospitals/{hospitalId}/offered-specialties` · Requer autenticacao.

## Itens Clinicos

### 5.1 Associar item clinico a hospital

- Caso de uso: `CreateOfferedItemCommand`
- Endpoint: `POST /api/v1/hospital-management/hospitals/{hospitalId:guid}/offered-items`
- Modelo principal: `HospitalOfferedItem`

### Body

```json
{
  "clinicalItemId": "uuid-string"
}
```

### Sucesso esperado

- `201 Created`
- retorno: `OfferedItemResponse`

### 5.2 Listar itens clinicos do hospital

- Caso de uso: `GetOfferedItemsQuery`
- Endpoint: `GET /api/v1/hospital-management/hospitals/{hospitalId:guid}/offered-items`
- Sucesso: `200 OK` → array de `OfferedItemResponse`

### 5.3 Remover item clinico

- Caso de uso: `DeleteOfferedItemCommand`
- Endpoint: `DELETE /api/v1/hospital-management/hospitals/{hospitalId:guid}/offered-items/{clinicalItemId}`
- Sucesso: `204 No Content`
- Erros: `404 Structure.NotFound`

---

## Especialidades

### 5.4 Associar especialidade a hospital

- Caso de uso: `CreateOfferedSpecialtyCommand`
- Endpoint: `POST /api/v1/hospital-management/hospitals/{hospitalId:guid}/offered-specialties`
- Modelo principal: `HospitalOfferedSpecialty`

### Body

```json
{
  "specialtyId": "uuid-string"
}
```

### Sucesso esperado

- `201 Created`
- retorno: `OfferedSpecialtyResponse`

### 5.5 Listar especialidades do hospital

- Caso de uso: `GetOfferedSpecialtiesQuery`
- Endpoint: `GET /api/v1/hospital-management/hospitals/{hospitalId:guid}/offered-specialties`
- Sucesso: `200 OK` → array de `OfferedSpecialtyResponse`

### 5.6 Remover especialidade

- Caso de uso: `DeleteOfferedSpecialtyCommand`
- Endpoint: `DELETE /api/v1/hospital-management/hospitals/{hospitalId:guid}/offered-specialties/{specialtyId}`
- Sucesso: `204 No Content`
- Erros: `404 Structure.NotFound`

## Contratos

### `OfferedItemResponse`

```
Id, HospitalId, ClinicalItemId, IsActive
```

### `OfferedSpecialtyResponse`

```
Id, HospitalId, SpecialtyId, IsActive
```

---

## Navegacao

[ homepage Estrutura Fisica](./physical-structure.md) · [Indice do modulo](../index.md) · [Contas](./accounts.md)

**Neste modulo:** [Visao Geral](./overview.md) · [Modelos e Enumeracoes](./models-and-enums.md) · [Hospitais](./hospitals.md) · [Unidades](./units.md) · [Departamentos](./departments.md) · [Estrutura Fisica](./physical-structure.md) · **Ofertas** · [Contas](./accounts.md) · [Seguranca](./security.md) · [Verificacoes](./checks.md) · [Casos Internos e Notas](./internal-and-events.md)
