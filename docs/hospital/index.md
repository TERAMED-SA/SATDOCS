# Modulo HospitalManagement

## Visao Geral

O modulo `HospitalManagement` e responsavel pela gestao de hospitais, unidades, departamentos, estrutura fisica (edificios, pisos, setores, salas, camas), contas de hospital, seguranca (roles e permissoes), ofertas de itens clinicos e especialidades, e verificacoes de hospital.

E um modulo do monolito modular SAT, implementado em .NET 8 com Clean Architecture, CQRS via MediatR e persistencia EF Core sobre PostgreSQL (schema `hospital_management`).

## Dominios do sistema

### 1. Dominio Organizacional

Representa a estrutura administrativa e funcional da instituicao.

```
HOSPITAL
├── Entidade legal
├── Identificadores unicos (NIF, etc.)
├── Contas e permissoes
│
├── UNIDADES
│   ├── MAIN    (Sede principal)
│   ├── BRANCH  (Filiais)
│   └── CLINIC  (Clinicas especializadas)
│
└── DEPARTAMENTOS
    ├── Pediatria
    ├── Cardiologia
    ├── Administracao
    └── Suporte Tecnico
```

### 2. Dominio Estrutural (Fisico)

Representa a infraestrutura fisica da instituicao.

```
HOSPITAL
└── UNIDADE
    └── EDIFICIO
        ├── Bloco Clinico
        ├── Bloco Administrativo
        └── Centro Cirurgico
            └── PISO
                ├── Terreo
                ├── Primeiro Andar
                └── Segundo Andar
                    └── SETOR
                        ├── CARE        (Atendimento)
                        ├── SUPPORT     (Apoio)
                        ├── ADMIN       (Administrativo)
                        └── TECHNICAL   (Tecnico)
                            └── SALA
                                ├── Consulta
                                ├── Internamento
                                ├── Cirurgia
                                └── Observacao
                                    └── CAMA
                                        ├── STANDARD
                                        ├── ICU
                                        ├── NEONATAL
                                        └── RECOVERY
```

## Estrutura de projetos

| Projeto | Responsabilidade |
|---|---|
| `Sat.HospitalManagement.Domain` | Entidades, enumeracoes, erros e interfaces de repositorio |
| `Sat.HospitalManagement.Application` | Commands, Queries, Handlers (MediatR), DTOs de request/response |
| `Sat.HospitalManagement.Contracts` | Porta de anti-corrupcao para outros modulos (`IHospitalManagementApi`) |
| `Sat.HospitalManagement.Infrastructure` | `DbContext`, configuracoes EF, repositorios |
| `Sat.HospitalManagement.Module` | Composition root: registo de servicos e mapeamento de endpoints |

## Navegacao

- [Visao Geral dos Casos de Uso](./use-cases/overview.md)
- [Modelos e Enumeracoes](./use-cases/models-and-enums.md)
- [Hospitais](./use-cases/hospitals.md)
- [Unidades](./use-cases/units.md)
- [Departamentos](./use-cases/departments.md)
- [Estrutura Fisica](./use-cases/physical-structure.md)
- [Ofertas](./use-cases/offers.md)
- [Contas](./use-cases/accounts.md)
- [Seguranca](./use-cases/security.md)
- [Verificacoes](./use-cases/checks.md)
- [Casos Internos e Notas](./use-cases/internal-and-events.md)

## Estrutura sugerida de leitura

1. Comece por [Visao Geral](./use-cases/overview.md)
2. Consulte [Modelos e Enumeracoes](./use-cases/models-and-enums.md)
3. Entre no dominio funcional desejado
4. Feche em [Casos Internos e Notas](./use-cases/internal-and-events.md)
