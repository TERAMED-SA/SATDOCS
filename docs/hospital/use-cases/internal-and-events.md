# Casos Internos e Notas - HospitalManagement

[ homepage Modulo HospitalManagement](../index.md) · [Visao Geral](./overview.md)

## Casos de uso nao expostos por HTTP

### `IHospitalManagementApi` (porta ACL)

Definida em `Contracts/V1/HospitalManagement/IHospitalManagementApi.cs`:

```csharp
Task<HospitalDonationDecision> IsAllowedHospitalForDonationAsync(Guid hospitalId, CancellationToken cancellationToken);
```

`HospitalDonationDecision`: `Allowed`, `NotFound`, `Inactive`, `NoBloodBank`, `NotEligibleForDonation`.

Este e o ponto de integracao com o modulo BloodDonation. O BloodDonation consulta este metodo para validar se um hospital e elegivel para receber doacoes.

### Integracao com BloodDonation

O modulo BloodDonation consome `IHospitalManagementApi` via `IHospitalContracts` (porta ACL). Os pontos de integracao sao:

- `ConfirmDonationCommandHandler` — valida elegibilidade do hospital ao confirmar dacao
- `ValidateDonationEligibilityQueryHandler` — valida elegibilidade do hospital na verificacao de elegibilidade

O modulo BloodDonation apenas traduz a resposta para o seu vocabulario local. A regra de elegibilidade vive no HospitalManagement.

## Composition Root

`HospitalManagementModule` implementa `IModule` (SharedKernel) e e descoberto automaticamente por reflexao pelo `ModuleScanner`. Regista:

- Application: MediatR sobre o assembly marcado por `HospitalManagementApplicationMarker`
- Infrastructure: `DbContext` (Npgsql), UoW, repositorios
- Endpoints: `HospitalManagement`, `Unit`, `Department`, `PhysicalStructure`, `Offer`, `Account`, `Security`, `Check`

## Persistencia

- `DbContext`: `HospitalManagementDbContext`, schema PostgreSQL **`hospital_management`**
- `DbSet`s: `Hospitals`, `HospitalUnits`, `Departments`, `Buildings`, `Floors`, `Sectors`, `Rooms`, `Beds`, `HospitalAccounts`, `AccountRoles`, `AccountRolePermissions`, `AccountUnits`, `AccountDepartments`, `OfferedItems`, `OfferedSpecialties`
- Unit of Work: o proprio `DbContext` implementa `IHospitalManagementUnitOfWork`

### Aplicar migracoes

```bash
dotnet ef database update \
  --project modules/HospitalManagement/Infrastructure \
  --startup-project src/Sat.Api \
  --context HospitalManagementDbContext
```

## Soft Delete

Todas as entidades principais (Hospital, Unit, Department, Building, Floor, Sector, Room, Bed, Account, Role) suportam soft delete atraves do campo `DeletedAt`. O `DeletedAt` nao e exposto por HTTP.

## Notas de implementacao

### Hospitais inativos e suspenso

Os endpoints de leitura (`GET`) funcionam normalmente para hospitais `Inactive` ou `Suspended`. Apenas `PUT` e restrito (`Hospital.ReadOnly`).

### Eliminacao de departamentos

A eliminacao de departamentos e bloqueada se existirem contas associadas (`Dept.LinkedToAccounts`). E necessario remover as contas ou os seus scopes primeiro.

### Eliminacao de camas ocupadas

As camas com `Status = Occupied` nao podem ser eliminadas (`Structure.Forbidden`). E necessario alterar o estado da cama primeiro.

### FloorLevel

O nivel do piso (`FloorLevel`) aceita valores de -3 a 50. Valores fora deste rango sao rejeitados com `Structure.ValidationError`.

### AccountCode

O `AccountCode` tem de ser unico dentro de um hospital. A unicidade e verificada a nivel de handler, nao de base de dados (constraint de dominio).

### RoleName

O `RoleName` tem de ser unico dentro de uma conta. A unicidade e verificada a nivel de handler.

---

## Navegacao

[ homepage Verificacoes](./checks.md) · [Indice do modulo](../index.md)

**Neste modulo:** [Visao Geral](./overview.md) · [Modelos e Enumeracoes](./models-and-enums.md) · [Hospitais](./hospitals.md) · [Unidades](./units.md) · [Departamentos](./departments.md) · [Estrutura Fisica](./physical-structure.md) · [Ofertas](./offers.md) · [Contas](./accounts.md) · [Seguranca](./security.md) · [Verificacoes](./checks.md) · **Casos Internos e Notas**
