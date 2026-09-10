# Modelos e Enumerações — ClinicalCatalog

[⌂ Módulo ClinicalCatalog](../index.md) · [Visão Geral](./overview.md)

## Modelos de Domínio

`MedicalSpecialty`, `ClinicalItem` e `ClinicalRule` são `AggregateRoot<Guid>`. `ClinicalRuleAudit` é uma entidade simples de auditoria, imutável após a criação.

### MedicalSpecialty

Especialidade médica, identificada por código SNOMED.

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `SnomedCode` | `string` | Obrigatório e único no catálogo |
| `Name` | `string` | Obrigatório |
| `IsActive` | `bool` | Por omissão `true` |
| `CreatedBy` / `UpdatedBy` | `string` | Id do utilizador autenticado (claim `sub`) |
| `CreatedAt` / `UpdatedAt` | `DateTime` | |

**Métodos de domínio:** `Create(snomedCode, name, isActive)`, `Update(name, isActive)`.

O `SnomedCode` é imutável após a criação — `Update` só permite alterar `Name` e `IsActive`.

### ClinicalItem

Item clínico agendável (consulta, exame, vacina ou procedimento), pertencente a uma especialidade.

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `SpecialtyId` | `Guid` | Especialidade a que pertence; tem de existir |
| `Type` | `ClinicalItemType` | |
| `Code` | `string` | Obrigatório e **único em todo o catálogo**, não apenas dentro da especialidade |
| `StandardCode` | `string?` | Código normalizado externo (ex.: SNOMED/LOINC) |
| `Name` | `string` | Obrigatório |
| `Description` | `string?` | |
| `AllowedScheduleType` | `AllowedScheduleType` | Por omissão `Both` |
| `RequiresPrescription` | `bool` | Por omissão `false` |
| `RequiresSafetyCheck` | `bool` | Marca o item como sujeito a avaliação de segurança clínica |
| `IsActive` | `bool` | Por omissão `true` |
| `CreatedBy` / `UpdatedBy` | `string` | Id do utilizador autenticado (claim `sub`) |
| `CreatedAt` / `UpdatedAt` | `DateTime` | |

**Métodos de domínio:** `Create(...)`, `Update(name, description, allowedScheduleType, requiresPrescription, requiresSafetyCheck, isActive)`.

`SpecialtyId`, `Type`, `Code` e `StandardCode` são imutáveis após a criação.

### ClinicalRule

Regra de segurança clínica associada a um item, com validade temporal.

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `ClinicalItemId` | `Guid` | Item a que se aplica; tem de existir |
| `MinHoursAfterBloodDonation` | `int` | Horas mínimas após dação de sangue |
| `MinDaysAfterVaccination` | `int` | Dias mínimos após vacinação |
| `MinDaysAfterSurgery` | `int` | Dias mínimos após cirurgia |
| `ContraindicatedAnticoagulants` | `bool` | Contraindicado sob anticoagulantes |
| `ValidFrom` | `DateTime` | Início da vigência (normalizado para UTC na criação) |
| `ValidTo` | `DateTime?` | Fim da vigência; `null` = vigência aberta |
| `CreatedBy` | `string` | Id do utilizador autenticado (claim `sub`) |
| `CreatedAt` / `UpdatedAt` | `DateTime` | |

**Métodos de domínio:** `Create(...)`, `Update(...)`.

Invariante: se `ValidTo` tiver valor, `ValidFrom` tem de ser estritamente anterior (`Rule.InvalidDateRange`). A não sobreposição entre regras do mesmo item é garantida nos handlers de criação e de atualização, não na entidade.

### ClinicalRuleAudit

Registo imutável de uma alteração a uma regra.

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `ClinicalRuleId` | `Guid` | Regra alterada |
| `ChangedBy` | `string` | Id do utilizador autenticado (claim `sub`) |
| `OldData` | `string` | JSON do `ClinicalRuleResponse` **antes** da operação; `{}` na criação |
| `NewData` | `string` | JSON do `ClinicalRuleResponse` **depois** da operação; `{}` na eliminação |
| `ChangedAt` | `DateTime` | |

Escrito nas três operações de escrita sobre regras — `POST`, `PATCH` e `DELETE` — sempre na mesma transação da operação que o origina. Os registos não são apagados com a regra.

**Métodos de domínio:** `ForCreation(...)`, `ForUpdate(...)`, `ForDeletion(...)`.

## Enumerações

### ClinicalItemType

- `Consultation`
- `Exam`
- `Vaccine`
- `Procedure`

### AllowedScheduleType

- `Public` — agendável apenas por canal público
- `Internal` — agendável apenas internamente
- `Both` — ambos (valor por omissão)

## Contratos de Resposta

### `SpecialtyResponse`

```
Id, SnomedCode, Name, IsActive, CreatedBy, UpdatedBy, CreatedAt, UpdatedAt, ItemCount?
```

`ItemCount` é preenchido em `GET /specialties` e `GET /specialties/{id}`; vem `null` nas respostas de `POST` e `PATCH`.

### `ClinicalItemResponse`

```
Id, SpecialtyId, Type, Code, StandardCode, Name, Description,
AllowedScheduleType, RequiresPrescription, RequiresSafetyCheck, IsActive,
CreatedBy, UpdatedBy, CreatedAt, UpdatedAt, Specialty?, RuleCount?
```

`Specialty` (objeto `SpecialtyResponse` aninhado) e `RuleCount` só são preenchidos em `GET /items` e `GET /items/{id}`. Em `POST` e `PATCH` vêm `null`.

### `ClinicalRuleResponse`

```
Id, ClinicalItemId, MinHoursAfterBloodDonation, MinDaysAfterVaccination,
MinDaysAfterSurgery, ContraindicatedAnticoagulants, ValidFrom, ValidTo,
CreatedBy, CreatedAt, UpdatedAt
```

### `RuleAuditResponse`

```
Id, ClinicalRuleId, ChangedBy, OldData, NewData, ChangedAt
```

### `CanDeleteSpecialtyResponse`

```
CanDelete, Reason
```

### `PagedResult<T>`

```
Data, Total, Page, Limit, TotalPages, HasNext, HasPrev
```

`TotalPages`, `HasNext` e `HasPrev` são calculados a partir de `Total`, `Page` e `Limit`.

## Catálogo de Erros

Definidos em `Domain/Errors/ClinicalCatalogErrors.cs`.

### Specialty

| Code | Categoria | Mensagem |
|---|---|---|
| `Specialty.NotFound` | NotFound | Medical specialty was not found. |
| `Specialty.AlreadyExists` | Conflict | A specialty with this SNOMED code already exists. |
| `Specialty.InvalidSnomedCode` | Validation | SNOMED code is required. |
| `Specialty.InvalidName` | Validation | Specialty name is required. |
| `Specialty.HasAssociatedItems` | Conflict | Specialty has associated clinical items and cannot be deleted. |

### Item

| Code | Categoria | Mensagem |
|---|---|---|
| `Item.NotFound` | NotFound | Clinical item was not found. |
| `Item.AlreadyExists` | Conflict | A clinical item with this code already exists. |
| `Item.InvalidCode` | Validation | Item code is required. |
| `Item.InvalidName` | Validation | Item name is required. |
| `Item.SpecialtyNotFound` | Validation | The referenced medical specialty does not exist. |

### Rule

| Code | Categoria | Mensagem |
|---|---|---|
| `Rule.NotFound` | NotFound | Clinical rule was not found. |
| `Rule.ItemNotFound` | Validation | The referenced clinical item does not exist. |
| `Rule.InvalidDateRange` | Validation | validFrom must be before validTo. |
| `Rule.OverlappingDates` | Conflict | Rule dates overlap with an existing rule for this item. |

---

## Navegação

← [Visão Geral](./overview.md) · [Índice do módulo](../index.md) · [Especialidades](./specialties.md) →

**Neste módulo:** [Visão Geral](./overview.md) · **Modelos e Enumerações** · [Especialidades](./specialties.md) · [Itens Clínicos](./items.md) · [Regras Clínicas](./rules.md) · [Casos Internos, Eventos e Notas](./internal-and-events.md)
