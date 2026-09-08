# Módulo ClinicalCatalog

## Visão Geral

O módulo `ClinicalCatalog` é o catálogo mestre da plataforma clínica: define **especialidades médicas**, os **itens clínicos** que cada especialidade oferece (consultas, exames, vacinas, procedimentos) e as **regras clínicas** com validade temporal que condicionam o agendamento desses itens.

É a fonte de verdade de referência para outros módulos — nomeadamente Scheduling e o futuro `ClinicalSafety`, que precisa das regras para decidir se um paciente pode realizar um item numa data específica.

## Estrutura de projetos

| Projeto | Responsabilidade |
|---|---|
| `Sat.ClinicalCatalog.Domain` | `MedicalSpecialty`, `ClinicalItem`, `ClinicalRule`, `ClinicalRuleAudit`, enumerações, erros, eventos, interfaces de repositório |
| `Sat.ClinicalCatalog.Application` | Commands, Queries, Handlers, DTOs, `PagedResult<T>` |
| `Sat.ClinicalCatalog.Contracts` | Projeto de contratos públicos — **atualmente vazio** |
| `Sat.ClinicalCatalog.Infrastructure` | `DbContext`, configurações EF, repositórios, migrações |
| `Sat.ClinicalCatalog.Module` | Composition root: registo de serviços e mapeamento de endpoints |
| `Sat.ClinicalCatalog.Tests` | Testes de domínio e de handlers de criação |

## Modelo mental

```
MedicalSpecialty  1 ──── N  ClinicalItem  1 ──── N  ClinicalRule
   (SNOMED)                  (código)              (janela ValidFrom→ValidTo)
                                                          │
                                                          └── N  ClinicalRuleAudit
```

- uma especialidade agrupa itens clínicos e não pode ser apagada enquanto tiver itens
- um item clínico tem código único e pode ter várias regras ao longo do tempo, **sem sobreposição de datas**
- cada alteração a uma regra gera um registo de auditoria com o estado anterior e o novo

## Navegação

- [Visão Geral dos Casos de Uso](./use-cases/overview.md)
- [Modelos e Enumerações](./use-cases/models-and-enums.md)
- [Especialidades](./use-cases/specialties.md)
- [Itens Clínicos](./use-cases/items.md)
- [Regras Clínicas](./use-cases/rules.md)
- [Casos Internos, Eventos e Notas](./use-cases/internal-and-events.md)

## Estrutura sugerida de leitura

1. Comece por [Visão Geral](./use-cases/overview.md)
2. Consulte [Modelos e Enumerações](./use-cases/models-and-enums.md)
3. Siga a ordem natural de dependência: [Especialidades](./use-cases/specialties.md) → [Itens](./use-cases/items.md) → [Regras](./use-cases/rules.md)
4. Feche em [Casos Internos, Eventos e Notas](./use-cases/internal-and-events.md) — eventos de domínio, persistência e cobertura de testes

