# Modulo ClinicalProntuary

O modulo **ClinicalProntuary** gere o prontuario clinico dos pacientes: criacao de prontuarios, registo de consultas, diagnosticos, prescricoes, exames laboratoriais, alergias e imunizacoes.

## Projectos

| Camada | Assembly | Responsabilidade |
|---|---|---|
| **Domain** | `Sat.ClinicalProntuary.Domain` | Entidades, enumeracoes, regras de negocio |
| **Application** | `Sat.ClinicalProntuary.Application` | Ports (interfaces), features CQRS |
| **Infrastructure** | `Sat.ClinicalProntuary.Infrastructure` | EF Core DbContext, repositorios |
| **Contracts** | `Sat.ClinicalProntuary.Contracts` | Contratos publicos para integracao cross-module |
| **Module** | `Sat.ClinicalProntuary.Module` | Composition root, endpoints, DI |

## Leitura

Comece pela [Visao Geral](./use-cases/overview.md) para ver a superficie da API, depois consulte o dominio que pretende:

1. [Visao Geral](./use-cases/overview.md)
2. [Modelos e Enumeracoes](./use-cases/models-and-enums.md)
3. [Prontuarios](./use-cases/prontuaries.md)
4. [Consultas](./use-cases/visits.md)
5. [Registos Clinicos](./use-cases/clinical-records.md)
6. [Casos Internos e Notas](./use-cases/internal-and-events.md)

## Navegacao

[Indice do modulo](./index.md) · **Visao Geral** · [Modelos e Enumeracoes](./use-cases/models-and-enums.md) · [Prontuarios](./use-cases/prontuaries.md) · [Consultas](./use-cases/visits.md) · [Registos Clinicos](./use-cases/clinical-records.md) · [Casos Internos e Notas](./use-cases/internal-and-events.md)
