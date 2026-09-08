# Casos Internos, Eventos e Notas — ClinicalCatalog

[⌂ Módulo ClinicalCatalog](../index.md) · [Visão Geral](./overview.md)

## Casos de uso não expostos por HTTP

**Nenhum.** Ao contrário do módulo BloodDonation, o ClinicalCatalog não regista background services nem jobs. Todos os casos de uso implementados têm rota HTTP correspondente.

## Eventos de Domínio

Definidos em `Domain/Events`, herdando de `DomainEvent` (SharedKernel):

| Evento | Payload | Intenção |
|---|---|---|
| `SpecialtyCreatedV1` | `SpecialtyId`, `SnomedCode`, `Name` | nova especialidade criada |
| `SpecialtyDeactivatedV1` | `SpecialtyId` | especialidade desativada |
| `ClinicalItemCreatedV1` | `ItemId`, `SpecialtyId`, `Code`, `Type` | novo item clínico criado |
| `ClinicalItemDeactivatedV1` | `ItemId` | item desativado |
| `ClinicalRuleCreatedV1` | `RuleId`, `ClinicalItemId`, `ValidFrom`, `ValidTo` | nova regra publicada |

Estes tipos definem o formato dos eventos para uso futuro. Não são publicados nem consumidos pela versão atual, pelo que não devem ser usados como mecanismo de integração.

## Comunicação entre módulos

O projeto `Sat.ClinicalCatalog.Contracts` **está vazio**: não existe interface pública nem tipo partilhado.

Consequência prática: outros módulos só conseguem consumir o catálogo **por HTTP**, através das listagens públicas de `/items` e `/rules` — não existe nenhum endpoint de leitura em lote, nem chamada em processo tipada, como acontece entre BloodDonation e HospitalManagement (`IHospitalContracts`).

Se se pretender expor o catálogo a outros módulos do monólito, o caminho consistente com o resto da solução é definir uma interface pública em `Contracts/V1` e um adaptador ACL do lado do consumidor.

## Persistência

- `DbContext`: `ClinicalCatalogDbContext`, schema PostgreSQL **`clinical_catalog`**
- `DbSet`s: `MedicalSpecialties`, `ClinicalItems`, `ClinicalRules`, `ClinicalRuleAudits`
- Unit of Work: o próprio `DbContext` implementa `IClinicalCatalogUnitOfWork`
- Repositórios *scoped*: `IMedicalSpecialtyRepository`, `IClinicalItemRepository`, `IClinicalRuleRepository`

### Tabelas e índices

| Tabela | Índices |
|---|---|
| `medical_specialties` | `SnomedCode` **único** |
| `clinical_items` | `Code` **único**; `SpecialtyId` |
| `clinical_rules` | `ClinicalItemId`; restrição de exclusão `ck_clinical_rules_no_overlap` |
| `clinical_rule_audits` | `ClinicalRuleId` |

As enumerações `ClinicalItemType` e `AllowedScheduleType` são persistidas **como texto** (`HasConversion<string>`), o que torna os dados legíveis em SQL e imunes a reordenação da enumeração em C#.

As relações entre especialidade → item → regra → auditoria são colunas `uuid` indexadas, sem chaves estrangeiras: a integridade referencial é assegurada pelos handlers, que impedem apagar uma especialidade com itens ou um item com regras.

A não-sobreposição de vigências, essa, é garantida pela base de dados através de `ck_clinical_rules_no_overlap`, uma restrição de exclusão sobre o par (item, intervalo de vigência) com ambos os extremos inclusivos e `validTo` nulo tratado como sem fim.

### Migrações

| Migração | Efeito |
|---|---|
| `20260621120000_InitialClinicalCatalog` | criação do schema e das quatro tabelas |
| `20260623091538_RemoveClinicalSafetyAudit` | remoção da tabela `clinical_safety_audits` |
| `20260818185422_PreventOverlappingClinicalRules` | restrição de exclusão que impede vigências sobrepostas no mesmo item |

A segunda migração é relevante para perceber a fronteira entre módulos: a tabela `clinical_safety_audits` (com `PatientId`, `ClinicalItemId`, `ClinicalRuleId`, `DesiredDate`, `EvaluatedAt`, `WasAllowed`, `Reason`) registava avaliações de segurança clínica dentro do catálogo. Foi removida em junho de 2026 quando essa responsabilidade foi separada para o módulo `ClinicalSafety` — que continua por implementar. O catálogo mantém apenas a **definição** das regras; a **avaliação** delas deixou de lhe pertencer.

### Aplicar migrações

```bash
make migrate-clinical-catalog
```

Equivalente a:

```bash
dotnet ef database update \
  --project modules/ClinicalCatalog/Infrastructure \
  --startup-project src/Sat.Api \
  --context ClinicalCatalogDbContext
```

## Composition root

`ClinicalCatalogModule` implementa `IModule` e é descoberto por reflexão pelo `ModuleScanner`. Regista:

- Application: MediatR sobre o assembly de `ClinicalCatalogApplicationMarker`
- Infrastructure: `DbContext` (Npgsql), UoW, três repositórios
- Endpoints: `Specialties`, `Items`, `Rules`

## Cobertura de testes

Em `Tests/`, 73 testes:

- **Domínio:** `MedicalSpecialtyTests`, `ClinicalItemTests`, `ClinicalRuleTests`
- **Aplicação:** criação de especialidades, itens e regras; atualização de itens, especialidades e regras; eliminações; consultas em lote; histórico de auditoria
- **Infraestrutura:** `LikePatternTests`

Cobertos em particular: a validação de sobreposição na atualização (incluindo a exclusão da própria regra), a normalização UTC das datas, a auditoria de criação/atualização/eliminação de regras, o histórico legível após eliminação, o limite dos pedidos em lote, o escape do termo de pesquisa, o bloqueio de itens com regras associadas e o registo da autoria a partir do id do chamador.

---

## Navegação

← [Regras Clínicas](./rules.md) · [Índice do módulo](../index.md)

**Neste módulo:** [Visão Geral](./overview.md) · [Modelos e Enumerações](./models-and-enums.md) · [Especialidades](./specialties.md) · [Itens Clínicos](./items.md) · [Regras Clínicas](./rules.md) · **Casos Internos, Eventos e Notas**
