# Visão Geral dos Casos de Uso — ClinicalCatalog

[⌂ Módulo ClinicalCatalog](../index.md)

## Objetivo

Esta documentação descreve os casos de uso do catálogo clínico a partir da implementação real em `Application/Features` e da exposição HTTP em `Module/Endpoints`.

Para cada caso de uso explica-se:

- que problema resolve
- que modelos manipula
- como chamar o endpoint
- que resposta esperar em sucesso
- que erros de negócio podem ocorrer

## Visão Geral da API

- Prefixo global: `/api/v1` (`ModuleLoader`)
- Prefixo do módulo: `/clinical-catalog`
- Stack HTTP: ASP.NET Core Minimal APIs + MediatR (`ISender`)
- Autenticação: JWT Bearer, obrigatória nos três grupos de rotas
- **Autorização: nenhuma role é exigida.** Qualquer utilizador autenticado pode criar, alterar e apagar especialidades, itens e regras clínicas

## Índice rápido de endpoints por domínio

Todas as rotas consideram o prefixo `/api/v1`.

### Especialidades

| Método | Rota | Finalidade |
|---|---|---|
| `POST` | `/clinical-catalog/specialties` | Criar especialidade |
| `GET` | `/clinical-catalog/specialties` | Listar especialidades |
| `GET` | `/clinical-catalog/specialties/{id}` | Obter especialidade por id |
| `GET` | `/clinical-catalog/specialties/{id}/can-delete` | Verificar se pode ser apagada |
| `PATCH` | `/clinical-catalog/specialties/{id}` | Atualizar especialidade |
| `DELETE` | `/clinical-catalog/specialties/{id}` | Apagar especialidade |

Ver detalhes em [Especialidades](./specialties.md).

### Itens Clínicos

| Método | Rota | Finalidade |
|---|---|---|
| `POST` | `/clinical-catalog/items` | Criar item clínico |
| `GET` | `/clinical-catalog/items` | Listar itens |
| `GET` | `/clinical-catalog/items/{id}` | Obter item por id |
| `PATCH` | `/clinical-catalog/items/{id}` | Atualizar item |
| `DELETE` | `/clinical-catalog/items/{id}` | Apagar item |

Ver detalhes em [Itens Clínicos](./items.md).

### Regras Clínicas

| Método | Rota | Finalidade |
|---|---|---|
| `POST` | `/clinical-catalog/rules` | Criar regra (gera auditoria) |
| `GET` | `/clinical-catalog/rules` | Listar regras |
| `GET` | `/clinical-catalog/rules/{id}` | Obter regra por id |
| `PATCH` | `/clinical-catalog/rules/{id}` | Atualizar regra (gera auditoria) |
| `DELETE` | `/clinical-catalog/rules/{id}` | Apagar regra (gera auditoria) |
| `GET` | `/clinical-catalog/rules/{id}/audits` | Histórico de alterações da regra |

Ver detalhes em [Regras Clínicas](./rules.md).

## Headers

```http
Authorization: Bearer <jwt>
Content-Type: application/json
```

### Autoria dos registos

Todas as escritas do catálogo registam o autor a partir da claim `sub` do token (mapeada para `ClaimTypes.NameIdentifier`), tal como no módulo BloodDonation:

- especialidades e itens: `CreatedBy`/`UpdatedBy`
- regras: `CreatedBy` e o `ChangedBy` de cada registo de auditoria

O valor gravado é o **id do utilizador**, não o nome. Nenhum destes campos é aceite no body — enviá-los não tem efeito. Um token sem `sub` legível devolve `401`, mesmo que a autenticação tenha passado.

## Padrão de Resposta e Erro

### Sucesso

- **`201 Created` nas criações** (`POST /items`, `POST /specialties`, `POST /rules`), com `Location` a apontar para o `GET` do recurso criado
- `200 OK` em leituras e atualizações; `204 No Content` nas eliminações
- `204 No Content` nos três endpoints `DELETE`

### Envelope de erro

Em falha, o corpo é o record `Error` do SharedKernel:

```json
{
  "code": "Item.AlreadyExists",
  "message": "A clinical item with this code already exists."
}
```

O código HTTP deriva da categoria semântica do erro, atribuída no domínio pela fábrica que o cria (`Error.Conflict`, `Error.NotFound`, ...) e traduzida no endpoint por `result.ToProblem()`:

| Categoria (`ErrorType`) | Status | Exemplo |
|---|---|---|
| `Validation` | `400 Bad Request` | `Item.InvalidCode` |
| `NotFound` | `404 Not Found` | `Specialty.NotFound` |
| `Conflict` | `409 Conflict` | `Specialty.HasAssociatedItems` |

O status é sempre coerente com o `code`, em qualquer endpoint. Apagar uma especialidade com itens associados devolve `409` com `code = "Specialty.HasAssociatedItems"`; ler um id inexistente devolve `404`. O cliente pode usar o status para o tratamento genérico e o `code` para a mensagem específica.

`401 Unauthorized` continua a ser devolvido, com corpo vazio, quando não há token válido — a rejeição acontece antes de existir um `Result`.

### Eliminação em cascata

Nenhuma eliminação é em cascata, e cada nível protege o de baixo: uma especialidade com itens devolve `Specialty.HasAssociatedItems`, um item com regras devolve `Item.HasAssociatedRules`. Para desmontar uma árvore do catálogo, apague de baixo para cima — regras, depois itens, depois a especialidade.

A exceção deliberada é a auditoria de regras: os registos de `clinical_rule_audits` **não** são apagados com a regra, porque são o único vestígio de que ela existiu.

### Envelope de paginação

As listagens devolvem `PagedResult<T>`:

```json
{
  "data": [],
  "total": 0,
  "page": 1,
  "limit": 20,
  "totalPages": 0,
  "hasNext": false,
  "hasPrev": false
}
```

Todas as listagens paginam de facto: `page` e `limit` são aplicados como `Skip`/`Take` no repositório, e `data` traz no máximo `limit` elementos. `total` conta o conjunto completo de resultados que passam os mesmos filtros.

Os valores recebidos são normalizados antes de serem usados — e os metadados devolvidos refletem sempre os valores normalizados, não os enviados:

| Enviado | Usado |
|---|---|
| `page < 1` | `1` |
| `limit < 1` | `20` (default) |
| `limit > 200` | `200` (máximo) |

---

## Navegação

[Índice do módulo](../index.md) · [Modelos e Enumerações](./models-and-enums.md) →

**Neste módulo:** **Visão Geral** · [Modelos e Enumerações](./models-and-enums.md) · [Especialidades](./specialties.md) · [Itens Clínicos](./items.md) · [Regras Clínicas](./rules.md) · [Casos Internos, Eventos e Notas](./internal-and-events.md)
