# Modelos e Enumeracoes - HospitalManagement

[ homepage Modulo HospitalManagement](../index.md) · [Visao Geral](./overview.md)

## Modelos de Dominio

Todas as entidades sao `AggregateRoot<Guid>` e vivem em `Domain/Entities`. Os setters sao privados: o estado so muda atraves de metodos de dominio que devolvem `Result`.

### Hospital

Entidade principal do modulo.

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `LegalName` | `string` | Obrigatorio |
| `CommercialName` | `string` | Obrigatorio |
| `HospitalType` | `HospitalType` | Public, Private, Mixed |
| `HospitalKind` | `HospitalKind` | General, Clinic, MedicalCenter, etc. |
| `Status` | `HospitalStatus` | Active por omissao |
| `Nif` | `string?` | NIF fiscal |
| `CommercialRegistry` | `string?` | Registo comercial |
| `HealthLicense` | `string?` | Licenca de saude |
| `Email` | `string?` | |
| `Phone` | `string?` | |
| `Website` | `string?` | |
| `Country` | `string` | Obrigatorio |
| `Province` | `string?` | |
| `City` | `string?` | |
| `Address` | `string?` | |
| `OpeningDate` | `DateTime?` | |
| `EmergencyEnabled` | `bool` | |
| `IntensiveCare` | `bool` | |
| `HasBloodBank` | `bool` | |
| `EligibleForDonation` | `bool` | |
| `CreatedAt` / `UpdatedAt` | `DateTime` | Auditoria temporal |
| `DeletedAt` | `DateTime?` | Soft delete |

**Metodos de dominio:** `Create`, `Update`, `SoftDelete`.

**Restricoes de transicao:**

- `Update` e rejeitado se o hospital estiver `Inactive` ou `Suspended` (`Hospital.ReadOnly`)
- `SoftDelete` define `Status = Inactive` e preenche `DeletedAt`

### HospitalUnit

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `HospitalId` | `Guid` | FK para Hospital |
| `Name` | `string` | Obrigatorio |
| `UnitType` | `UnitType` | Main, Branch, Clinic |
| `Active` | `bool` | `true` por omissao |
| `Address` | `string?` | |
| `City` | `string?` | |
| `Province` | `string?` | |
| `CreatedAt` / `UpdatedAt` | `DateTime` | |
| `DeletedAt` | `DateTime?` | Soft delete |

**Metodos de dominio:** `Create`, `Update`, `SoftDelete`.

### Department

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `HospitalId` | `Guid` | FK para Hospital |
| `Name` | `string` | Obrigatorio |
| `Description` | `string?` | |
| `Active` | `bool` | `true` por omissao |
| `CreatedAt` / `UpdatedAt` | `DateTime` | |
| `DeletedAt` | `DateTime?` | Soft delete |

**Metodos de dominio:** `Create`, `Update`, `SoftDelete`.

**Restricoes:** nao e possivel eliminar um departamento que tenha contas associadas (`Dept.LinkedToAccounts`).

### Building

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `UnitId` | `Guid?` | FK para HospitalUnit |
| `Name` | `string` | Obrigatorio |
| `Code` | `string` | Obrigatorio |
| `CreatedAt` / `UpdatedAt` | `DateTime` | |
| `DeletedAt` | `DateTime?` | Soft delete |

### Floor

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `BuildingId` | `Guid?` | FK para Building |
| `FloorLevel` | `int` | Entre -3 e 50 |
| `CreatedAt` / `UpdatedAt` | `DateTime` | |
| `DeletedAt` | `DateTime?` | Soft delete |

### Sector

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `FloorId` | `Guid?` | FK para Floor |
| `Name` | `string` | Obrigatorio |
| `SectorType` | `SectorType?` | Care, Support, Admin, Technical |
| `Description` | `string?` | |
| `CreatedAt` / `UpdatedAt` | `DateTime` | |
| `DeletedAt` | `DateTime?` | Soft delete |

### Room

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `SectorId` | `Guid?` | FK para Sector |
| `Name` | `string` | Obrigatorio |
| `Purpose` | `RoomPurpose` | Consultation, Surgery, Icu, etc. |
| `Capacity` | `int` | >= 1 |
| `Active` | `bool` | `true` por omissao |
| `Description` | `string?` | |
| `CreatedAt` / `UpdatedAt` | `DateTime` | |
| `DeletedAt` | `DateTime?` | Soft delete |

### Bed

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `RoomId` | `Guid` | FK para Room |
| `BedCode` | `string` | Obrigatorio |
| `BedType` | `BedType` | Standard, Icu, Neonatal, Recovery |
| `Status` | `BedStatus` | Available por omissao |
| `Active` | `bool` | `true` por omissao |
| `CreatedAt` / `UpdatedAt` | `DateTime` | |
| `DeletedAt` | `DateTime?` | Soft delete |

**Restricoes:** nao e possivel eliminar uma cama em estado `Occupied` (`Structure.Forbidden`).

### HospitalAccount

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `HospitalId` | `Guid` | FK para Hospital |
| `UserId` | `Guid` | Utilizador associado |
| `AccountName` | `string` | Obrigatorio |
| `AccountCode` | `string` | Obrigatorio, unico por hospital |
| `Status` | `AccountStatus` | Active por omissao |
| `CreatedAt` / `UpdatedAt` | `DateTime` | |
| `DeletedAt` | `DateTime?` | Soft delete |

### AccountRoleModel

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `AccountId` | `Guid` | FK para HospitalAccount |
| `RoleName` | `string` | Obrigatorio, unico por conta |
| `BaseRole` | `AccountRole?` | Role base opcional |
| `Description` | `string?` | |
| `Status` | `RoleStatus` | Active por omissao |
| `CreatedAt` / `UpdatedAt` | `DateTime` | |
| `DeletedAt` | `DateTime?` | Soft delete |

### AccountRolePermission

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `RoleId` | `Guid` | FK para AccountRoleModel |
| `Resource` | `Resource` | Recurso alvo |
| `Action` | `PermissionAction` | Acao permitida |
| `CreatedAt` / `UpdatedAt` | `DateTime` | |
| `DeletedAt` | `DateTime?` | |

### HospitalOfferedItem

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `HospitalId` | `Guid` | FK para Hospital |
| `ClinicalItemId` | `string` | ID do item clinico externo |
| `IsActive` | `bool` | |
| `CreatedAt` / `UpdatedAt` | `DateTime` | |

### HospitalOfferedSpecialty

| Campo | Tipo | Notas |
|---|---|---|
| `Id` | `Guid` | |
| `HospitalId` | `Guid` | FK para Hospital |
| `SpecialtyId` | `string` | ID da especialidade externa |
| `IsActive` | `bool` | |
| `CreatedAt` / `UpdatedAt` | `DateTime` | |

## Enumeracoes de Negocio

Todas em `Domain/Enums`. Os valores sao serializados conforme a configuracao JSON global.

### HospitalType

- `Public`
- `Private`
- `Mixed`

### HospitalKind

- `General`
- `Clinic`
- `MedicalCenter`
- `Laboratory`
- `HealthPost`
- `Rehabilitation`

### HospitalStatus

- `Active` (0)
- `Inactive` (1)
- `Suspended` (2)

### UnitType

- `Main`
- `Branch`
- `Clinic`

### SectorType

- `Care`
- `Support`
- `Admin`
- `Technical`

### RoomPurpose

- `Consultation`
- `Surgery`
- `Icu`
- `Ward`
- `Lab`
- `Imaging`
- `Office`
- `Storage`
- `Recovery`

### BedType

- `Standard`
- `Icu`
- `Neonatal`
- `Recovery`

### BedStatus

- `Available`
- `Occupied`
- `Maintenance`
- `Blocked`

### AccountStatus

- `Active`
- `Suspended`
- `Closed`

### RoleStatus

- `Active`
- `Disabled`

### AccessLevel

- `View`
- `Edit`
- `Admin`

### AccountRole

- `SystemAdmin`
- `OrgAdmin`
- `Support`
- `ServiceAccount`
- `HospitalManager`
- `HospitalAdmin`
- `UnitAdmin`
- `UnitManager`
- `DepartmentHead`
- `Doctor`
- `Nurse`
- `Receptionist`
- `Auditor`
- `BillingOfficer`
- `Custom`

### Resource

- `Patient`, `Appointment`, `Bed`, `Room`, `Department`, `Unit`, `Hospital`, `User`, `Document`, `Billing`, `Report`, `Schedule`, `Inventory`, `Prescription`, `Slot`, `Forwarding`, `Reschedule`, `WorkingHours`, `ExcludeDay`, `ExcludeRange`, `Utility`

### PermissionAction

- `Create`
- `Read`
- `Update`
- `Delete`
- `Manage`
- `Approve`
- `Export`

## Catalogo de Erros

Definidos em `Domain/Errors/HospitalManagementErrors.cs`.

### Hospital

| Code | Categoria | Mensagem |
|---|---|---|
| `Hospital.NotFound` | NotFound | Hospital was not found. |
| `Hospital.AlreadyExists` | Conflict | A hospital with the same NIF or Legal Name already exists. |
| `Hospital.ValidationError` | Validation | Hospital data is invalid. |
| `Hospital.ReadOnly` | Conflict | Inactive or suspended hospitals are read-only. |
| `Hospital.LegalNameRequired` | Validation | Legal name is required. |
| `Hospital.CommercialNameRequired` | Validation | Commercial name is required. |
| `Hospital.CountryRequired` | Validation | Country is required. |

### Unit

| Code | Categoria | Mensagem |
|---|---|---|
| `Unit.NotFound` | NotFound | Unit was not found. |
| `Unit.AlreadyExists` | Conflict | A unit with the same name already exists in this hospital. |
| `Unit.ValidationError` | Validation | Unit data is invalid. |
| `Unit.Inactive` | Validation | Inactive units cannot accept new physical structures. |
| `Unit.Forbidden` | Validation | Unit operation is not permitted in current status. |

### Department

| Code | Categoria | Mensagem |
|---|---|---|
| `Dept.NotFound` | NotFound | Department was not found. |
| `Dept.AlreadyExists` | Conflict | A department with the same name already exists in this hospital. |
| `Dept.ValidationError` | Validation | Department data is invalid. |
| `Dept.LinkedToAccounts` | Conflict | Department cannot be deleted while accounts are linked. |

### PhysicalStructure

| Code | Categoria | Mensagem |
|---|---|---|
| `Structure.NotFound` | NotFound | Physical structure element was not found. |
| `Structure.ValidationError` | Validation | Physical structure data is invalid. |
| `Structure.Forbidden` | Validation | Physical structure operation is not permitted. |

### Account

| Code | Categoria | Mensagem |
|---|---|---|
| `Account.NotFound` | NotFound | Account was not found. |
| `Account.AlreadyExists` | Conflict | Account code must be unique in the hospital. |
| `Account.ValidationError` | Validation | Account data is invalid. |
| `Account.HospitalNotActive` | Validation | Account creation requires an ACTIVE hospital. |
| `Account.UserIdRequired` | Validation | User id is required. |
| `Account.AccountNameRequired` | Validation | Account name is required. |
| `Account.AccountCodeRequired` | Validation | Account code is required. |

### Security

| Code | Categoria | Mensagem |
|---|---|---|
| `Role.NotFound` | NotFound | Role was not found. |
| `Role.AlreadyExists` | Conflict | Role name must be unique by account. |
| `Permission.NotFound` | NotFound | Permission was not found. |
| `Permission.Duplicate` | Conflict | Permissions cannot be duplicated in the same role. |

### Scope

| Code | Categoria | Mensagem |
|---|---|---|
| `Scope.NotFound` | NotFound | Scope not found. |
| `Scope.Forbidden` | Validation | Scope operation is forbidden. |
| `Scope.Duplicate` | Conflict | Scope already exists. |

---

## Navegacao

[ homepage Visao Geral](./overview.md) · [Indice do modulo](../index.md) · [Hospitais](./hospitals.md)

**Neste modulo:** [Visao Geral](./overview.md) · **Modelos e Enumeracoes** · [Hospitais](./hospitals.md) · [Unidades](./units.md) · [Departamentos](./departments.md) · [Estrutura Fisica](./physical-structure.md) · [Ofertas](./offers.md) · [Contas](./accounts.md) · [Seguranca](./security.md) · [Verificacoes](./checks.md) · [Casos Internos e Notas](./internal-and-events.md)
