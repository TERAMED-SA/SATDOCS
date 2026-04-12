# Controle de concorrência com ETag e If-Match

## Visão geral

O uso dos cabeçalhos **`ETag`** e **`If-Match`** permite que a API implemente **controle de concorrência otimista** (_optimistic concurrency control_), evitando o problema conhecido como **"lost update"** — quando dois clientes alteram o mesmo recurso quase ao mesmo tempo e uma atualização sobrescreve a outra.

---

## O que é `ETag`

O **ETag** (_Entity Tag_) é um identificador único que representa a **versão atual** de um recurso.

- Cada vez que o recurso é modificado, o ETag também muda.  
- Ele é retornado no cabeçalho HTTP de resposta.  
- Pode ser baseado em um **hash do conteúdo**, em um **timestamp (`updatedAt`)**, ou em qualquer informação que identifique a versão do recurso.

### Exemplo de resposta com ETag

```http
HTTP/1.1 200 OK
ETag: "9f620c9f5c7a3b2d8f8e5a3e9b8a1c23"
Content-Type: application/json

{
  "id": "1",
  "name": "Dr. Itamar",
  "email": "itamar@example.com",
  "updatedAt": "2025-10-17T08:30:00Z"
}
```

---

## O que é `If-Match`

O cabeçalho **`If-Match`** é usado em requisições de atualização (`PUT`, `PATCH` ou `DELETE`) para garantir que a operação **só será realizada se o recurso não tiver sido alterado** desde a última leitura.

- O cliente deve enviar o mesmo valor de ETag que recebeu no `GET`.  
- Se o ETag atual do recurso no servidor **for diferente**, significa que outro cliente já alterou o recurso.  
- Nesse caso, a API deve retornar **`412 Precondition Failed`**.

---

## Fluxo de uso

| Etapa | Ação | Descrição |
|------:|------|------------|
| 1️⃣ | **GET** | O cliente obtém o recurso e o cabeçalho `ETag`. |
| 2️⃣ | **PUT / PATCH** | O cliente envia o recurso atualizado com `If-Match` igual ao ETag anterior. |
| 3️⃣ | **Servidor compara** | Se o ETag enviado for diferente → rejeita com `412 Precondition Failed`. |
| 4️⃣ | **Atualização segura** | Se iguais → aplica atualização e gera novo `ETag`. |

---

## Exemplo completo

### 1️⃣ Cliente lê o recurso

```bash
GET /user/1
```

**Resposta:**

```http
HTTP/1.1 200 OK
ETag: "abc123"
Content-Type: application/json

{ "id": 1, "name": "Itamar", "email": "itamar@example.com" }
```

---

### 2️⃣ Cliente tenta atualizar o recurso

```bash
PUT /user/1
If-Match: "abc123"
Content-Type: application/json

{ "email": "itamar@novoemail.com" }
```

**Resposta (sucesso):**

```http
HTTP/1.1 200 OK
ETag: "def456"
Content-Type: application/json

{ "id": 1, "name": "Itamar", "email": "itamar@novoemail.com" }
```

---

### 3️⃣ Outro cliente alterou antes (ETag diferente)

```bash
PUT /user/1
If-Match: "abc123"
```

**Resposta:**

```http
HTTP/1.1 412 Precondition Failed
Content-Type: application/json

{ "error": "Recurso foi modificado por outro cliente" }
```

---

## Exemplo prático com Node.js (Express)

A seguir, um exemplo funcional de como implementar **ETag** e **If-Match** em uma API Node.js com **Express**:

```prisma
model User {
  id        String   @id @default(uuid())
  name      String
  email     String   @unique
  updatedAt DateTime @updatedAt
  createdAt DateTime @default(now())
}
```

```js
import express from "express";
import crypto from "crypto";
import { PrismaClient } from "@prisma/client";

const app = express();
const prisma = new PrismaClient();
app.use(express.json());

// Função para gerar ETag baseado no conteúdo do registro
function generateETag(data: any) {
  return crypto.createHash("md5").update(JSON.stringify(data)).digest("hex");
}

// ✅ GET /users/:id — Retorna o usuário e o ETag
app.get("/users/:id", async (req, res) => {
  const user = await prisma.user.findUnique({
    where: { id: req.params.id },
  });

  if (!user) {
    return res.status(404).json({ error: "Usuário não encontrado" });
  }

  const etag = generateETag(user);
  res.set("ETag", etag);
  return res.json(user);
});

// ✅ PUT /users/:id — Atualiza com controle de concorrência
app.put("/users/:id", async (req, res) => {
  const ifMatch = req.headers["if-match"];
  if (!ifMatch) {
    return res
      .status(428)
      .json({ error: "Cabeçalho 'If-Match' é obrigatório." });
  }

  const user = await prisma.user.findUnique({
    where: { id: req.params.id },
  });

  if (!user) {
    return res.status(404).json({ error: "Usuário não encontrado" });
  }

  const currentETag = generateETag(user);
  if (ifMatch !== currentETag) {
    return res
      .status(412)
      .json({ error: "Recurso foi modificado por outro cliente." });
  }

  // Atualiza o usuário
  const updatedUser = await prisma.user.update({
    where: { id: req.params.id },
    data: {
      ...req.body,
      updatedAt: new Date(),
    },
  });

  // Gera novo ETag
  const newETag = generateETag(updatedUser);
  res.set("ETag", newETag);
  return res.json(updatedUser);
});

app.listen(3000, () =>
  console.log("Servidor rodando em http://localhost:3000")
);
```

### Como testar

1️⃣ **GET** → `http://localhost:3000/user/1`  
   - Verifique o cabeçalho `ETag` retornado.  
2️⃣ **PUT** → envie `If-Match` com o valor do ETag anterior.  
3️⃣ **PUT** novamente com o mesmo `If-Match` (sem atualizar o GET antes) → deve retornar `412 Precondition Failed`.

---

## Boas práticas

- Sempre exponha `ETag` nas respostas `GET` e `HEAD`.  
- Exija `If-Match` em requisições `PUT` e `PATCH`.  
- Gere o `ETag` a partir de:
  - Hash do conteúdo (`MD5`, `SHA1`, etc.); ou  
  - Campo `updatedAt` (mais simples e comum).  
- Retorne **`412 Precondition Failed`** quando o ETag divergir.  
- Opcionalmente, use também `If-None-Match` para **cache e validação condicional** em `GET`.

---

## Benefícios

- Evita **atualizações perdidas (lost updates)**.  
- Mantém **integridade dos dados** em APIs concorrentes.  
- Facilita **cache inteligente** e sincronização de estado entre clientes.  
- É **simples e eficiente** para implementar em qualquer stack (Node.js, Java, .NET, etc.).

---
