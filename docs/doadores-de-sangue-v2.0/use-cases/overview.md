# Visão Geral dos Casos de Uso — BloodDonation

[⌂ Módulo BloodDonation](../index.md)

## Objetivo

Esta documentação descreve os casos de uso do módulo de dação de sangue a partir da implementação real em `Application/Features` e da exposição HTTP em `Module/Endpoints`.

O foco não é apenas listar rotas. A ideia é explicar:

- que problema cada caso de uso resolve
- que modelos manipula
- como chamar cada endpoint
- que respostas esperar em sucesso
- que erros de negócio e de acesso podem ocorrer
- que casos de uso não são expostos por HTTP e correm internamente

## Visão Geral da API

- Prefixo global: `/api/v1` (aplicado em `ModuleLoader`, `src/Sat.Bootstrap/Modules/ModuleLoader.cs`)
- Prefixo do módulo: `/blood-donation`
- Stack HTTP: ASP.NET Core Minimal APIs + MediatR (`ISender`)
- Autenticação: JWT Bearer (`AddJwtBearer`), obrigatória em todos os grupos de rotas do módulo
- Identidade do chamador: claim `ClaimTypes.NameIdentifier` (`sub`/`nameid`), convertida para `Guid`
- Rotas administrativas: exigem adicionalmente a role `SystemAdmin`

## Índice rápido de endpoints por domínio

Todas as rotas abaixo consideram o prefixo `/api/v1`.

### Dadores

| Método | Rota | Finalidade | Acesso |
|---|---|---|---|
| `POST` | `/blood-donation/donors` | Registar-se como dador | Autenticado |
| `GET` | `/blood-donation/donors/{id}` | Obter perfil de dador por id | Autenticado |
| `GET` | `/blood-donation/donors/me` | Obter o meu perfil de dador | Autenticado |
| `PATCH` | `/blood-donation/donors/me` | Atualizar o meu perfil | Autenticado |
| `PATCH` | `/blood-donation/donors/me/availability` | Atualizar disponibilidade | Autenticado |
| `PATCH` | `/blood-donation/donors/me/blood-type` | Corrigir o tipo sanguíneo | Autenticado |
| `POST` | `/blood-donation/donors/me/reanalysis` | Pedir nova análise clínica | Autenticado |

Ver detalhes em [Dadores](./donors.md).

### Análises Clínicas

| Método | Rota | Finalidade | Acesso |
|---|---|---|---|
| `GET` | `/blood-donation/analyses/pending` | Listar análises pendentes | Autenticado |
| `GET` | `/blood-donation/analyses/donor/{donorId}` | Listar análises de um dador | Autenticado |
| `POST` | `/blood-donation/analyses/{id}/approve` | Aprovar análise | Autenticado |
| `POST` | `/blood-donation/analyses/{id}/reject` | Rejeitar análise | Autenticado |

Ver detalhes em [Análises Clínicas](./analysis.md).

### Pedidos de Sangue

| Método | Rota | Finalidade | Acesso |
|---|---|---|---|
| `POST` | `/blood-donation/requests` | Criar pedido de sangue | Autenticado |
| `GET` | `/blood-donation/requests/my` | Listar os meus pedidos | Autenticado |
| `GET` | `/blood-donation/requests/public/search` | Procurar pedidos públicos | Autenticado |
| `GET` | `/blood-donation/requests/{id}` | Obter detalhes de um pedido | Autenticado |
| `PATCH` | `/blood-donation/requests/{id}/cancel` | Cancelar pedido | Autenticado (dono) |

Ver detalhes em [Pedidos de Sangue](./requests.md).

### Matching

| Método | Rota | Finalidade | Acesso |
|---|---|---|---|
| `POST` | `/blood-donation/requests/{id}/match` | Dador propõe match a um pedido | Autenticado (dador) |
| `POST` | `/blood-donation/matches/{id}/accept` | Aceitar match | Autenticado (requerente) |
| `POST` | `/blood-donation/matches/{id}/reject` | Rejeitar match | Autenticado (requerente ou dador) |
| `POST` | `/blood-donation/matches/{id}/withdraw` | Desistir do match | Autenticado (dador) |
| `GET` | `/blood-donation/matches/my` | Listar os meus matches | Autenticado |
| `GET` | `/blood-donation/requests/{id}/matches` | Listar matches do pedido | Autenticado (dono do pedido) |

Ver detalhes em [Matching](./matching.md).

### Dações

| Método | Rota | Finalidade | Acesso |
|---|---|---|---|
| `POST` | `/blood-donation/donations/confirm` | Confirmar dação realizada | Autenticado (requerente) |
| `POST` | `/blood-donation/donations/{id}/hospital-confirmation` | Confirmação hospitalar | Autenticado |
| `GET` | `/blood-donation/donations/my-history` | Histórico das minhas dações | Autenticado |
| `GET` | `/blood-donation/donations/eligibility/check` | Verificar elegibilidade | Autenticado |

Ver detalhes em [Dações](./donations.md).

### Moderação

| Método | Rota | Finalidade | Acesso |
|---|---|---|---|
| `POST` | `/blood-donation/moderation/reports` | Criar denúncia | Autenticado |
| `GET` | `/blood-donation/moderation/reports/my` | Listar as minhas denúncias | Autenticado |
| `GET` | `/blood-donation/moderation/reports/pending` | Listar denúncias pendentes | `SystemAdmin` |
| `GET` | `/blood-donation/moderation/reports/{id}` | Obter denúncia por id | `SystemAdmin` |
| `PATCH` | `/blood-donation/moderation/reports/{id}` | Atualizar denúncia | `SystemAdmin` |
| `POST` | `/blood-donation/moderation/block-donor/{id}` | Bloquear dador | `SystemAdmin` |
| `POST` | `/blood-donation/moderation/unblock-donor/{id}` | Desbloquear dador | `SystemAdmin` |
| `GET` | `/blood-donation/moderation/stats` | Estatísticas de moderação | `SystemAdmin` |
| `GET` | `/blood-donation/moderation/risk-assessment/{donorId}` | Avaliação de risco do dador | `SystemAdmin` |
| `GET` | `/blood-donation/moderation/target-reports/{targetType}/{targetId}` | Denúncias por alvo | `SystemAdmin` |

Ver detalhes em [Moderação](./moderation.md).

### Administração

| Método | Rota | Finalidade | Acesso |
|---|---|---|---|
| `GET` | `/blood-donation/admin/stats` | Estatísticas do módulo | `SystemAdmin` |
| `GET` | `/blood-donation/admin/donors` | Listar dadores | `SystemAdmin` |
| `GET` | `/blood-donation/admin/active-requests` | Listar pedidos ativos | `SystemAdmin` |
| `GET` | `/blood-donation/admin/recent-activity` | Atividade recente | `SystemAdmin` |
| `GET` | `/blood-donation/admin/export/stats` | Exportar estatísticas | `SystemAdmin` |

Ver detalhes em [Administração](./admin.md).

## Headers

### Rotas autenticadas

```http
Authorization: Bearer <jwt>
Content-Type: application/json
```

O token tem de conter a claim `NameIdentifier` com um `Guid` válido. Se a claim estiver ausente ou não for um `Guid`, o endpoint devolve `401 Unauthorized` — mesmo que o JWT em si seja válido.

### Rotas administrativas

O mesmo token, mas com a role `SystemAdmin`:

```http
Authorization: Bearer <jwt-com-role-SystemAdmin>
```

Sem a role, o pipeline de autorização devolve `403 Forbidden`.

## Padrão de Resposta e Erro

### Sucesso

Todos os handlers devolvem `Result<T>` (SharedKernel). Os endpoints traduzem sucesso em `Results.Ok(result.Value)`, ou seja:

- **`201 Created` nas criações.** `POST /donors`, `POST /requests` e `POST /moderation/reports` devolvem `Location` a apontar para o `GET` do recurso criado; `POST /requests/{id}/match`, `POST /donations/confirm` e `POST /donors/me/reanalysis` devolvem `201` sem `Location`, por não existir rota de leitura individual para esses recursos
- **`200 OK`** nas restantes operações bem-sucedidas, incluindo os `POST` que atuam sobre um recurso existente (aprovar, rejeitar, aceitar, retirar, bloquear, desbloquear, confirmação hospitalar)
- `204 No Content` não é usado neste módulo

### Envelope de erro

Em falha, o endpoint devolve o objeto `Error` do SharedKernel diretamente no corpo:

```json
{
  "code": "Donor.NotFound",
  "message": "Donor was not found."
}
```

O código HTTP deriva da categoria semântica do erro, atribuída no domínio pela fábrica que o cria (`Error.Conflict`, `Error.NotFound`, ...) e traduzida no endpoint por `result.ToProblem()`:

| Categoria (`ErrorType`) | Status | Exemplo |
|---|---|---|
| `Validation` | `400 Bad Request` | `BloodRequest.InvalidQuantity` |
| `NotFound` | `404 Not Found` | `Donor.NotFound` |
| `Conflict` | `409 Conflict` | `Donor.AlreadyExists` |
| `Unauthorized` | `401 Unauthorized` | — |
| `Forbidden` | `403 Forbidden` | `Auth.Forbidden` |

O status é sempre coerente com o `code`, em qualquer endpoint: `Donor.NotFound` devolve `404` tanto numa leitura por id como no meio de um comando. O cliente pode usar o status para o tratamento genérico (repetir, re-autenticar, mostrar conflito) e o `code` para a mensagem específica.

Independentemente disto, `401` continua a ser devolvido quando a claim `NameIdentifier` está ausente ou inválida, e `403` quando falta a role `SystemAdmin` numa rota administrativa — nesses casos o corpo é vazio, porque a rejeição acontece antes de existir um `Result`.

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

Todas as listagens paginam de facto: `page` e `limit` são aplicados como `Skip`/`Take` no repositório, e `data` traz no máximo `limit` elementos. `total` conta o conjunto completo de resultados que passam os mesmos filtros — não apenas os da página.

Os valores recebidos são normalizados antes de serem usados — e os metadados devolvidos refletem sempre os valores normalizados, não os enviados:

| Enviado | Usado |
|---|---|
| `page < 1` | `1` |
| `limit < 1` | `20` (default) |
| `limit > 200` | `200` (máximo) |

A única leitura que não usa este envelope é `GET /blood-donation/admin/recent-activity`: não é uma listagem, é um agregado de quatro feeds independentes num só payload. Ainda assim o seu `limit` passa pela mesma normalização, pelo que também está limitado a `200` por feed.

---

## Navegação

[Índice do módulo](../index.md) · [Modelos e Enumerações](./models-and-enums.md) →

**Neste módulo:** **Visão Geral** · [Modelos e Enumerações](./models-and-enums.md) · [Dadores](./donors.md) · [Análises Clínicas](./analysis.md) · [Pedidos de Sangue](./requests.md) · [Matching](./matching.md) · [Dações](./donations.md) · [Moderação](./moderation.md) · [Administração](./admin.md) · [Casos Internos, Eventos e Notas](./internal-and-events.md)
