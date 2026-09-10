# Visao Geral dos Casos de Uso - HospitalManagement

[ homepage Modulo HospitalManagement](../index.md)

## Objetivo

Esta documentacao descreve os casos de uso do modulo de gestao hospitalar a partir da implementacao real em `Application/Features` e da exposicao HTTP em `Module/Endpoints`.

O foco nao e apenas listar rotas. A ideia e explicar:

- que problema cada caso de uso resolve
- que modelos manipula
- como chamar cada endpoint
- que respostas esperar em sucesso
- que erros de negocio e de acesso podem ocorrer
- que casos de uso nao sao expostos por HTTP e correm internamente

## Visao Geral da API

- Prefixo global: `/api/v1` (aplicado em `ModuleLoader`)
- Prefixo do modulo: `/hospital-management`
- Stack HTTP: ASP.NET Core Minimal APIs + MediatR (`ISender`)
- Autenticacao: JWT Bearer, obrigatoria em todos os grupos de rotas do modulo
- Identidade do chamador: claim `ClaimTypes.NameIdentifier` (`sub`/`nameid`), convertida para `Guid`

## Indice rapido de endpoints por dominio

Todas as rotas abaixo consideram o prefixo `/api/v1`.

### Hospitais

| Metodo | Rota | Finalidade | Acesso |
|---|---|---|---|
| `POST` | `/hospital-management/hospitals` | Criar hospital | Autenticado |
| `GET` | `/hospital-management/hospitals` | Listar hospitais | Autenticado |
| `GET` | `/hospital-management/hospitals/{id}` | Obter hospital por id | Autenticado |
| `PUT` | `/hospital-management/hospitals/{id}` | Atualizar hospital | Autenticado |
| `DELETE` | `/hospital-management/hospitals/{id}` | Eliminar hospital | Autenticado |

Ver detalhes em [Hospitais](./hospitals.md).

### Unidades

| Metodo | Rota | Finalidade | Acesso |
|---|---|---|---|
| `POST` | `/hospital-management/hospitals/{hospitalId}/units` | Criar unidade | Autenticado |
| `GET` | `/hospital-management/hospitals/{hospitalId}/units` | Listar unidades do hospital | Autenticado |
| `GET` | `/hospital-management/units/{id}` | Obter unidade por id | Autenticado |
| `PUT` | `/hospital-management/units/{id}` | Atualizar unidade | Autenticado |
| `DELETE` | `/hospital-management/units/{id}` | Eliminar unidade | Autenticado |

Ver detalhes em [Unidades](./units.md).

### Departamentos

| Metodo | Rota | Finalidade | Acesso |
|---|---|---|---|
| `POST` | `/hospital-management/hospitals/{hospitalId}/departments` | Criar departamento | Autenticado |
| `GET` | `/hospital-management/hospitals/{hospitalId}/departments` | Listar departamentos do hospital | Autenticado |
| `GET` | `/hospital-management/departments/{id}` | Obter departamento por id | Autenticado |
| `PUT` | `/hospital-management/departments/{id}` | Atualizar departamento | Autenticado |
| `DELETE` | `/hospital-management/departments/{id}` | Eliminar departamento | Autenticado |

Ver detalhes em [Departamentos](./departments.md).

### Estrutura Fisica

| Metodo | Rota | Finalidade | Acesso |
|---|---|---|---|
| `POST` | `/hospital-management/units/{unitId}/buildings` | Criar edificio | Autenticado |
| `GET` | `/hospital-management/units/{unitId}/buildings` | Listar edificios da unidade | Autenticado |
| `GET` | `/hospital-management/buildings/{id}` | Obter edificio por id | Autenticado |
| `PUT` | `/hospital-management/buildings/{id}` | Atualizar edificio | Autenticado |
| `DELETE` | `/hospital-management/buildings/{id}` | Eliminar edificio | Autenticado |
| `POST` | `/hospital-management/buildings/{buildingId}/floors` | Criar piso | Autenticado |
| `GET` | `/hospital-management/buildings/{buildingId}/floors` | Listar pisos do edificio | Autenticado |
| `GET` | `/hospital-management/floors/{id}` | Obter piso por id | Autenticado |
| `PUT` | `/hospital-management/floors/{id}` | Atualizar piso | Autenticado |
| `DELETE` | `/hospital-management/floors/{id}` | Eliminar piso | Autenticado |
| `POST` | `/hospital-management/floors/{floorId}/sectors` | Criar setor | Autenticado |
| `GET` | `/hospital-management/floors/{floorId}/sectors` | Listar setores do piso | Autenticado |
| `GET` | `/hospital-management/sectors/{id}` | Obter setor por id | Autenticado |
| `PUT` | `/hospital-management/sectors/{id}` | Atualizar setor | Autenticado |
| `DELETE` | `/hospital-management/sectors/{id}` | Eliminar setor | Autenticado |
| `POST` | `/hospital-management/sectors/{sectorId}/rooms` | Criar sala | Autenticado |
| `GET` | `/hospital-management/sectors/{sectorId}/rooms` | Listar salas do setor | Autenticado |
| `GET` | `/hospital-management/rooms/{id}` | Obter sala por id | Autenticado |
| `PUT` | `/hospital-management/rooms/{id}` | Atualizar sala | Autenticado |
| `DELETE` | `/hospital-management/rooms/{id}` | Eliminar sala | Autenticado |
| `POST` | `/hospital-management/rooms/{roomId}/beds` | Criar cama | Autenticado |
| `GET` | `/hospital-management/rooms/{roomId}/beds` | Listar camas da sala | Autenticado |
| `GET` | `/hospital-management/beds/{id}` | Obter cama por id | Autenticado |
| `PUT` | `/hospital-management/beds/{id}` | Atualizar cama | Autenticado |
| `DELETE` | `/hospital-management/beds/{id}` | Eliminar cama | Autenticado |

Ver detalhes em [Estrutura Fisica](./physical-structure.md).

### Ofertas

| Metodo | Rota | Finalidade | Acesso |
|---|---|---|---|
| `POST` | `/hospital-management/hospitals/{hospitalId}/offered-items` | Associar item clinico | Autenticado |
| `GET` | `/hospital-management/hospitals/{hospitalId}/offered-items` | Listar itens clinicos | Autenticado |
| `DELETE` | `/hospital-management/hospitals/{hospitalId}/offered-items/{clinicalItemId}` | Remover item clinico | Autenticado |
| `POST` | `/hospital-management/hospitals/{hospitalId}/offered-specialties` | Associar especialidade | Autenticado |
| `GET` | `/hospital-management/hospitals/{hospitalId}/offered-specialties` | Listar especialidades | Autenticado |
| `DELETE` | `/hospital-management/hospitals/{hospitalId}/offered-specialties/{specialtyId}` | Remover especialidade | Autenticado |

Ver detalhes em [Ofertas](./offers.md).

### Contas

| Metodo | Rota | Finalidade | Acesso |
|---|---|---|---|
| `POST` | `/hospital-management/hospitals/{hospitalId}/accounts` | Criar conta | Autenticado |
| `GET` | `/hospital-management/hospitals/{hospitalId}/accounts` | Listar contas do hospital | Autenticado |
| `GET` | `/hospital-management/accounts/{id}` | Obter conta por id | Autenticado |
| `PUT` | `/hospital-management/accounts/{id}` | Atualizar conta | Autenticado |
| `DELETE` | `/hospital-management/accounts/{id}` | Eliminar conta | Autenticado |
| `POST` | `/hospital-management/accounts/{accountId}/units` | Atribuir scope de unidade | Autenticado |
| `DELETE` | `/hospital-management/accounts/{accountId}/units/{unitId}` | Remover scope de unidade | Autenticado |
| `POST` | `/hospital-management/accounts/{accountId}/departments` | Atribuir scope de departamento | Autenticado |
| `DELETE` | `/hospital-management/accounts/{accountId}/departments/{departmentId}` | Remover scope de departamento | Autenticado |

Ver detalhes em [Contas](./accounts.md).

### Seguranca

| Metodo | Rota | Finalidade | Acesso |
|---|---|---|---|
| `POST` | `/hospital-management/accounts/{accountId}/roles` | Criar role | Autenticado |
| `GET` | `/hospital-management/accounts/{accountId}/roles` | Listar roles da conta | Autenticado |
| `GET` | `/hospital-management/roles/{id}` | Obter role por id | Autenticado |
| `PUT` | `/hospital-management/roles/{id}` | Atualizar role | Autenticado |
| `DELETE` | `/hospital-management/roles/{id}` | Eliminar role | Autenticado |
| `POST` | `/hospital-management/roles/{roleId}/permissions` | Adicionar permissao | Autenticado |
| `GET` | `/hospital-management/roles/{roleId}/permissions` | Listar permissoes da role | Autenticado |
| `DELETE` | `/hospital-management/permissions/{id}` | Remover permissao | Autenticado |

Ver detalhes em [Seguranca](./security.md).

### Verificacoes

| Metodo | Rota | Finalidade | Acesso |
|---|---|---|---|
| `GET` | `/hospital-management/hospitals/{hospitalId}/checks/blood-bank` | Verificar banco de sangue | Autenticado |
| `GET` | `/hospital-management/hospitals/{hospitalId}/checks/donation-eligibility` | Verificar elegibilidade para dacao | Autenticado |
| `GET` | `/hospital-management/hospitals/{hospitalId}/checks/location-compatibility` | Verificar compatibilidade de localizacao | Autenticado |
| `GET` | `/hospital-management/hospitals/{hospitalId}/checks/has-specialty/{specialtyId}` | Verificar especialidade | Autenticado |
| `GET` | `/hospital-management/hospitals/{hospitalId}/checks/offers-clinical-item/{clinicalItemId}` | Verificar item clinico | Autenticado |

Ver detalhes em [Verificacoes](./checks.md).

## Headers

### Rotas autenticadas

```http
Authorization: Bearer <jwt>
Content-Type: application/json
```

O token tem de conter a claim `NameIdentifier` com um `Guid` valido. Se a claim estiver ausente ou nao for um `Guid`, o endpoint devolve `401 Unauthorized` - mesmo que o JWT em si seja valido.

## Padrao de Resposta e Erro

### Sucesso

Todos os handlers devolvem `Result<T>` (SharedKernel). Os endpoints traduzem sucesso em `Results.Ok(result.Value)` ou `Results.Created(...)`:

- **`200 OK`** em operacoes de leitura e atualizacao
- **`201 Created`** em criacoes (`POST`)
- **`204 No Content`** em eliminacoes

### Envelope de erro

Em falha, o endpoint devolve o objeto `Error` do SharedKernel diretamente no corpo:

```json
{
  "code": "Hospital.NotFound",
  "message": "Hospital was not found."
}
```

O codigo HTTP e escolhido **no endpoint**:

- `400 Bad Request`: resultado de falha na maioria dos endpoints
- `404 Not Found`: apenas nos endpoints de leitura por id
- `409 Conflict`: conflitos de duplicacao ou restricoes de integridade

### Paginacao

Este modulo nao tem envelope de paginacao. As listagens devolvem arrays simples.

---

## Navegacao

[Indice do modulo](../index.md) · [Modelos e Enumeracoes](./models-and-enums.md)

**Neste modulo:** **Visao Geral** · [Modelos e Enumeracoes](./models-and-enums.md) · [Hospitais](./hospitals.md) · [Unidades](./units.md) · [Departamentos](./departments.md) · [Estrutura Fisica](./physical-structure.md) · [Ofertas](./offers.md) · [Contas](./accounts.md) · [Seguranca](./security.md) · [Verificacoes](./checks.md) · [Casos Internos e Notas](./internal-and-events.md)
