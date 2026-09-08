# Módulo BloodDonation

## Visão Geral

O módulo `BloodDonation` é responsável pela gestão de dadores de sangue, pedidos de sangue, emparelhamento (matching) entre dador e pedido, confirmação de dações, moderação e estatísticas administrativas.

É um módulo do monólito modular SAT, implementado em .NET 8 com Clean Architecture, CQRS via MediatR e persistência EF Core sobre PostgreSQL (schema `blood_donation`).

## Estrutura de projetos

| Projeto | Responsabilidade |
|---|---|
| `Sat.BloodDonation.Domain` | Entidades, enumerações, eventos de domínio, erros e interfaces de repositório |
| `Sat.BloodDonation.Application` | Commands, Queries, Handlers (MediatR), DTOs de request/response |
| `Sat.BloodDonation.Contracts` | Porta de anti-corrupção para outros módulos (`IHospitalContracts`) |
| `Sat.BloodDonation.Infrastructure` | `DbContext`, configurações EF, repositórios, adaptadores ACL, background services |
| `Sat.BloodDonation.Module` | Composition root: registo de serviços e mapeamento de endpoints |
| `Sat.BloodDonation.Tests` | Testes de domínio e de handlers |

## Navegação

- [Visão Geral dos Casos de Uso](./use-cases/overview.md)
- [Modelos e Enumerações](./use-cases/models-and-enums.md)
- [Dadores](./use-cases/donors.md)
- [Análises Clínicas](./use-cases/analysis.md)
- [Pedidos de Sangue](./use-cases/requests.md)
- [Matching](./use-cases/matching.md)
- [Dações](./use-cases/donations.md)
- [Moderação](./use-cases/moderation.md)
- [Administração](./use-cases/admin.md)
- [Casos Internos, Eventos e Notas](./use-cases/internal-and-events.md)

## Estrutura sugerida de leitura

1. Comece por [Visão Geral](./use-cases/overview.md)
2. Consulte [Modelos e Enumerações](./use-cases/models-and-enums.md)
3. Entre no domínio funcional desejado
4. Feche em [Casos Internos, Eventos e Notas](./use-cases/internal-and-events.md) — trabalhos em background, eventos de domínio, persistência e cobertura de testes

