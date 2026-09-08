# Casos de Uso de Administração

[⌂ Módulo BloodDonation](../index.md) · [Visão Geral](./overview.md)

Grupo de rotas: `/api/v1/blood-donation/admin` · Todas exigem a role **`SystemAdmin`**.

```csharp
app.MapGroup("/blood-donation/admin")
   .RequireAuthorization(p => p.RequireRole("SystemAdmin"));
```

Sem a role, o pipeline devolve `403 Forbidden` antes de chegar ao handler.

## Filtro comum `StatsFilter`

Vários endpoints partilham os mesmos query params, materializados em `StatsFilter`:

| Parâmetro | Tipo | Omissão | Descrição |
|---|---|---|---|
| `startDate` | datetime | — | início da janela |
| `endDate` | datetime | — | fim da janela |
| `bloodType` | string | — | filtra por grupo sanguíneo; valor inválido é ignorado |
| `lastDays` | int | `30` | janela relativa em dias; valores nulos ou negativos caem no default |

Resolução da janela no handler:

- início efetivo = `startDate` ?? `agora - lastDays`
- fim efetivo = `endDate` ?? sem limite superior

## 7.1 Estatísticas do módulo

- Caso de uso: `GetSystemStatsQuery`
- Endpoint: `GET /api/v1/blood-donation/admin/stats`

### Regras de negócio

- as contagens são feitas pela base de dados; a resposta não depende do volume de registos
- os dadores são contados pelo **estado efetivo** — quem tem um adiamento ou bloqueio ativo conta como `TemporarilyIneligible` ou `Blocked`, ainda que o estado guardado seja outro
- o filtro de data aplica-se a pedidos, matches e dações (por `CreatedAt`); os dadores só são filtrados por tipo sanguíneo
- `NewLast30Days` é sempre calculado sobre os últimos 30 dias, independentemente do `lastDays` pedido

### Exemplo de chamada

```bash
curl 'http://localhost:5000/api/v1/blood-donation/admin/stats?lastDays=7&bloodType=O_Negative' \
  -H 'Authorization: Bearer <jwt-admin>'
```

### Sucesso esperado

- `200 OK`
- retorno: `SystemStatsResponse`

```json
{
  "donors": {
    "total": 120,
    "eligible": 84,
    "pendingAnalysis": 21,
    "blocked": 3,
    "temporarilyIneligible": 10,
    "permanentlyIneligible": 2,
    "newLast30Days": 15
  },
  "requests": {
    "total": 40,
    "open": 12,
    "matching": 8,
    "fulfilled": 5,
    "completed": 5,
    "cancelled": 5,
    "expired": 15
  },
  "matches": {
    "total": 63,
    "pending": 9,
    "accepted": 21,
    "rejected": 33
  },
  "donations": {
    "total": 19,
    "totalUnits": 24,
    "last30Days": 6
  }
}
```

`requests.fulfilled` conta os pedidos em `RequestStatus.Fulfilled` — o estado a que um pedido chega quando a dação é confirmada. `requests.completed` mantém-se no payload, **obsoleto**, com o mesmo valor: antes contava um estado que nenhum caminho de código atribuía e devolvia sempre `0`.

## 7.2 Listar dadores

- Caso de uso: `GetDonorsQuery`
- Endpoint: `GET /api/v1/blood-donation/admin/donors`
- Query params: os de `StatsFilter`, mais `page` (por omissão `1`) e `limit` (por omissão `20`, máximo `200`)

### Regras de negócio

- o filtro de data/tipo sanguíneo só é construído se pelo menos um dos parâmetros vier preenchido; caso contrário nenhum filtro é aplicado — mas a listagem continua paginada
- filtragem e paginação são feitas na base de dados; `total` conta todos os dadores que passam o filtro
- ordenação por `createdAt` descendente

### Sucesso esperado

- `200 OK`
- retorno: `PagedResult<DonorAdminResponse>` — ver [Envelope de paginação](./overview.md#envelope-de-paginacao)

```json
{
  "data": [
    {
      "id": "f1ac3c5e-6fd2-4a7a-b64e-72f40be690cf",
      "userId": "9a7b6c5d-4e3f-2a1b-0c9d-8e7f6a5b4c3d",
      "bloodType": "O_Negative",
      "status": "Eligible",
      "totalDonations": 5,
      "lastDonationDate": "2026-06-19T09:30:00Z",
      "createdAt": "2026-01-10T08:00:00Z"
    }
  ],
  "total": 1,
  "page": 1,
  "limit": 20,
  "totalPages": 1,
  "hasNext": false,
  "hasPrev": false
}
```

## 7.3 Listar pedidos ativos

- Caso de uso: `GetActiveRequestsQuery`
- Endpoint: `GET /api/v1/blood-donation/admin/active-requests`
- Query params: `page` (por omissão `1`), `limit` (por omissão `20`, máximo `200`)

### Regras de negócio

- `matchCount` é agregado na base de dados e apenas para os pedidos da página pedida
- ordenação por `createdAt` descendente

### Sucesso esperado

- `200 OK`
- retorno: `PagedResult<ActiveRequestAdminResponse>` — ver [Envelope de paginação](./overview.md#envelope-de-paginacao)

```json
{
  "data": [
    {
      "id": "8d5e1b2a-3c4f-4a5b-9c6d-7e8f9a0b1c2d",
      "requesterUserId": "9a7b6c5d-4e3f-2a1b-0c9d-8e7f6a5b4c3d",
      "bloodType": "O_Negative",
      "status": "Matching",
      "urgency": "Critical",
      "expiresAt": "2026-08-15T10:00:00Z",
      "createdAt": "2026-08-14T10:00:00Z",
      "matchCount": 3
    }
  ],
  "total": 1,
  "page": 1,
  "limit": 20,
  "totalPages": 1,
  "hasNext": false,
  "hasPrev": false
}
```

## 7.4 Atividade recente

- Caso de uso: `GetRecentActivityQuery`
- Endpoint: `GET /api/v1/blood-donation/admin/recent-activity`
- Query params: `limit` (por omissão `20`, máximo `200`)

### Regras de negócio

- **não usa o envelope de paginação**: não é uma listagem, são quatro feeds independentes no mesmo payload, cada um limitado a `limit` elementos
- o `limit` passa pela mesma normalização das listagens paginadas

### Sucesso esperado

- `200 OK`
- retorno: `RecentActivityResponse`, com listas dos pedidos, matches, dações e denúncias mais recentes

```json
{
  "recentRequests": [],
  "recentMatches": [],
  "recentDonations": [],
  "recentReports": [],
  "timestamp": "2026-08-14T12:00:00Z"
}
```

## 7.5 Exportar estatísticas

- Caso de uso: `GetSystemStatsQuery` (reutilizado)
- Endpoint: `GET /api/v1/blood-donation/admin/export/stats`
- Query params: os de `StatsFilter`

### Regras de negócio

- reutiliza exatamente a mesma query de 7.1 e envolve o resultado num objeto anónimo
- **o formato é JSON**, não CSV nem ficheiro para download

### Sucesso esperado

- `200 OK`

```json
{
  "exportDate": "2026-08-14T12:00:00Z",
  "filter": {
    "startDate": null,
    "endDate": null,
    "bloodType": null,
    "lastDays": 30
  },
  "stats": { "donors": {}, "requests": {}, "matches": {}, "donations": {} }
}
```

---

## Navegação

← [Moderação](./moderation.md) · [Índice do módulo](../index.md) · [Casos Internos, Eventos e Notas](./internal-and-events.md) →

**Neste módulo:** [Visão Geral](./overview.md) · [Modelos e Enumerações](./models-and-enums.md) · [Dadores](./donors.md) · [Análises Clínicas](./analysis.md) · [Pedidos de Sangue](./requests.md) · [Matching](./matching.md) · [Dações](./donations.md) · [Moderação](./moderation.md) · **Administração** · [Casos Internos, Eventos e Notas](./internal-and-events.md)
