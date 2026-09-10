# Casos Internos e Notas - ClinicalProntuary

[ homepage Modulo ClinicalProntuary](../index.md) · [Visao Geral](./overview.md)

## Entidade sem endpoint

### MedicalNote

A entidade `MedicalNote` existe no Dominio e esta configurada no EF Core, mas nao tem endpoint, comando ou request associado. đựe ser um recurso incompleto ou reservado para uso interno/futuro.

## Composition Root

`ClinicalProntuaryModule` implementa `IModule` (SharedKernel) e e descoberto automaticamente por reflexao pelo `ModuleScanner`. Regista:

- Application: MediatR sobre o assembly marcado por `ClinicalProntuaryApplicationMarker`
- Infrastructure: `DbContext` (Npgsql), repositorios
- Endpoints: `Prontuaries`, `Visits`, `ClinicalRecords`

## Persistencia

- `DbContext`: `ClinicalProntuaryDbContext` estende `SatDbContext` (building block que combina `DbContext` + `IUnitOfWork`), schema PostgreSQL **`clinical_prontuary`**
- `DbSet`s: `Prontuaries`, `Visits`, `Diagnoses`, `Prescriptions`, `LabResults`, `AllergyRecords`, `ImmunizationRecords`, `MedicalNotes`
- Unit of Work: `EfUnitOfWork` implementa `IClinicalProntuaryUnitOfWork` (extensao de `Sat.Persistence.UnitOfWork.IUnitOfWork`)
- Repositorios: `IProntuaryRepository : IRepository<Prontuary, Guid>` (SharedKernel) e `ProntuaryRepository : BaseRepository<Prontuary, Guid, ClinicalProntuaryDbContext>` (Persistence) — herda `GetByIdAsync`, `AddAsync` e `Remove` do building block

### Aplicar migracoes

```bash
dotnet ef database update \
  --project modules/ClinicalProntuary/Infrastructure \
  --startup-project src/Sat.Api \
  --context ClinicalProntuaryDbContext
```

## Notas de implementacao

### Prontuario idempotente

O `CreateProntuaryCommand` e idempotente: se ja existir prontuario para o `PatientId`, retorna o existente em vez de falhar.

### Medicacoes ativas

`GetActiveMedications()` filtra prescricoes por data: inclui prescricoes onde `StartDate <= agora` e `EndDate` e null ou `>= agora`.

### Resultados laboratoriais

`LabResult.ResultData` e armazenado como JSON PostgreSQL (`jsonb`). O formato e livre — a aplicacao deve garantir consistencia.

### FHIR

O endpoint `/fhir` e um wrapper simplificado que combina `FindProntuaryByPatientIdQuery` e `GetPatientClinicalSummaryQuery` num formato inspirado em FHIR Bundle. Nao e uma implementacao FHIR completa.

---

## Navegacao

[ homepage Registos Clinicos](./clinical-records.md) · [Indice do modulo](../index.md)

**Neste modulo:** [Visao Geral](./overview.md) · [Modelos e Enumeracoes](./models-and-enums.md) · [Prontuarios](./prontuaries.md) · [Consultas](./visits.md) · [Registos Clinicos](./clinical-records.md) · **Casos Internos e Notas**
