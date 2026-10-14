# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Autora** | Diana Arcanjo |
| **Status** | Pronto para implementação (pendente de revisão de segurança) |
| **Data** | 2026-10-14 |
| **Base** | [RFC](RFC.md) · [ADRs](adrs/) · [PRD](PRD.md) |

Este documento descreve **como construir** a feature. As decisões e seus porquês estão nos ADRs, e a proposta em nível de arquitetura está no RFC. Os itens têm IDs (`FDD-...`) rastreados no [TRACKER](TRACKER.md).

Quando um detalhe não foi definido na reunião e precisou ser escolhido aqui para tornar o documento acionável, ele aparece marcado como **[proposta do FDD]**.

---

## 1. Contexto e motivação técnica

A mudança de status de pedidos acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de um `prisma.$transaction`, em quatro passos:

1. valida a transição (`canTransition` em `src/modules/orders/order.status.ts`);
2. debita ou repõe estoque (`debitStock` / `replenishStock`);
3. atualiza `orders.status`;
4. grava `order_status_history`.

Não existe nenhum mecanismo de evento, fila ou notificação externa. Os clientes B2B descobrem mudanças fazendo polling em `GET /api/v1/orders`.

A feature adiciona um caminho assíncrono e confiável: o evento é registrado na mesma transação ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)) e entregue por um worker separado ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)). Por isso a transação de pedidos nunca fica esperando o cliente.

## 2. Objetivos técnicos

| ID | Objetivo |
|---|---|
| FDD-OBJ-01 | Registrar o evento de webhook atomicamente com a mudança de status: commit grava os dois, rollback descarta os dois. |
| FDD-OBJ-02 | Fazer a primeira tentativa de entrega em menos de 10 s após o commit, com polling de 2 s. |
| FDD-OBJ-03 | Entregar requisições assinadas com HMAC-SHA256 e secret por endpoint. |
| FDD-OBJ-04 | Garantir at-least-once com `event_id` estável em retentativas e replays. |
| FDD-OBJ-05 | Retentar com backoff de 1m/5m/30m/2h/12h e mover falhas permanentes para a DLQ. |
| FDD-OBJ-06 | Não alterar o comportamento nem o contrato dos endpoints existentes. |

## 3. Escopo e exclusões

**Dentro do escopo**
- Módulo `src/modules/webhooks/` com CRUD de endpoints, rotação de secret, histórico de entregas e replay da DLQ.
- Tabelas `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries` e `webhook_dead_letter`.
- Função `publishWebhookEvent` chamada por `changeStatus`.
- Entry point `src/worker.ts` e script `npm run worker`.

**Fora do escopo** (origem no [TRACKER](TRACKER.md))
- Webhooks de entrada (os clientes não enviam para nós).
- Notificação por e-mail ao cliente quando o webhook falha (próxima fase).
- Rate limiting de saída (observar e decidir depois).
- Dashboard visual para clientes (projeto do time de frontend).
- Arquivamento ou limpeza de linhas entregues da outbox (cerca de 30 dias, feature futura).
- Múltiplos workers em paralelo (particionamento por `order_id` ou lock pessimista).
- Evento na **criação** do pedido. A reunião tratou apenas de mudança de status; `OrderService.create` não publica eventos.
- Listagem da DLQ via API. Só o replay foi discutido. O administrador obtém o `deadLetterId` pelo log `webhook_moved_to_dead_letter` ou por consulta à tabela ([RFC-Q-09](RFC.md#5-questões-em-aberto)).

---

## 4. Modelo de dados

Adições em `prisma/schema.prisma`, seguindo o padrão do projeto: IDs UUID em `@db.Char(36)`, `@@map` em snake_case e `createdAt`/`updatedAt`.

```prisma
enum WebhookOutboxStatus {
  PENDING
  PROCESSING
  DELIVERED
  FAILED
}

model WebhookEndpoint {
  id                      String    @id @default(uuid()) @db.Char(36)
  customerId              String    @db.Char(36)
  url                     String    @db.VarChar(2048)
  secret                  String    @db.VarChar(128)
  previousSecret          String?   @db.VarChar(128)
  previousSecretExpiresAt DateTime?
  events                  Json
  active                  Boolean   @default(true)
  createdAt               DateTime  @default(now())
  updatedAt               DateTime  @updatedAt

  customer   Customer           @relation(fields: [customerId], references: [id], onDelete: Cascade)
  outbox     WebhookOutbox[]
  deliveries WebhookDelivery[]

  @@index([customerId, active])
  @@map("webhook_endpoints")
}

model WebhookOutbox {
  id            String              @id @default(uuid()) @db.Char(36)
  seq           Int                 @unique @default(autoincrement())
  eventId       String              @unique @db.Char(36)
  webhookId     String              @db.Char(36)
  orderId       String              @db.Char(36)
  eventType     String              @db.VarChar(64)
  payload       Json
  status        WebhookOutboxStatus @default(PENDING)
  attempts      Int                 @default(0)
  nextAttemptAt DateTime            @default(now())
  lastErrorCode String?             @db.VarChar(64)
  createdAt     DateTime            @default(now())
  updatedAt     DateTime            @updatedAt

  webhook    WebhookEndpoint   @relation(fields: [webhookId], references: [id], onDelete: Cascade)
  deliveries WebhookDelivery[]

  @@index([status])
  @@index([createdAt])
  @@index([status, nextAttemptAt])
  @@index([webhookId, orderId, status, seq])
  @@map("webhook_outbox")
}

model WebhookDelivery {
  id             String   @id @default(uuid()) @db.Char(36)
  outboxId       String   @db.Char(36)
  webhookId      String   @db.Char(36)
  eventId        String   @db.Char(36)
  attempt        Int
  success        Boolean
  responseStatus Int?
  responseBody   String?  @db.Text
  durationMs     Int
  errorCode      String?  @db.VarChar(64)
  createdAt      DateTime @default(now())

  outbox  WebhookOutbox   @relation(fields: [outboxId], references: [id], onDelete: Cascade)
  webhook WebhookEndpoint @relation(fields: [webhookId], references: [id], onDelete: Cascade)

  @@index([webhookId, createdAt])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {
  id            String    @id @default(uuid()) @db.Char(36)
  outboxId      String    @db.Char(36)
  webhookId     String    @db.Char(36)
  eventId       String    @db.Char(36)
  payload       Json
  failureReason String    @db.VarChar(500)
  errorCode     String    @db.VarChar(64)
  attempts      Int
  failedAt      DateTime  @default(now())
  replayedAt    DateTime?
  replayedById  String?   @db.Char(36)

  @@index([failedAt])
  @@map("webhook_dead_letter")
}
```

A relação nova `webhookEndpoints WebhookEndpoint[]` entra em `Customer`. **FDD-DADOS-11:** o `onDelete: Cascade` na FK para `Customer` é necessário porque `CustomerService.delete` (`src/modules/customers/customer.service.ts`) apaga o cliente diretamente. Sem cascata, a exclusão de um cliente com webhook passaria a falhar, mudando um comportamento existente. **[proposta do FDD]**

| ID | Detalhe de modelagem | Origem |
|---|---|---|
| FDD-DADOS-01 | `webhook_endpoints` guarda `url`, `secret`, `customer_id` e o estado ativo, mais o filtro `events` (array JSON de `OrderStatus`). | [09:21] Bruno, [09:33] Marcos |
| FDD-DADOS-02 | `previousSecret` e `previousSecretExpiresAt` implementam o grace period de 24h da rotação. **[proposta do FDD]** quanto à forma de armazenar. | [09:21] Sofia |
| FDD-DADOS-03 | `webhook_outbox` tem índice em `status` e em `created_at`. O índice composto `(status, nextAttemptAt)` atende a busca com backoff. **[proposta do FDD]** quanto ao composto. | [09:08] Diego |
| FDD-DADOS-04 | Uma linha de outbox **por endpoint** que assina o status. Cada linha tem seu próprio `eventId`, porque cada endpoint tem secret, retries e `X-Webhook-Id` próprios. **[proposta do FDD]** | [09:25] Diego, [09:44] Sofia |
| FDD-DADOS-05 | `payload` guarda o snapshot já renderizado. | [09:52] Larissa |
| FDD-DADOS-06 | `webhook_deliveries` registra cada tentativa (sucesso/falha, status HTTP, corpo da resposta e duração) para o histórico consultável. **[proposta do FDD]** quanto a ser tabela própria. | [09:34] Marcos |
| FDD-DADOS-07 | `webhook_dead_letter` guarda payload, motivo e timestamp, mais os dados de auditoria do replay (`replayedAt`, `replayedById`). | [09:18] Diego, [09:36] Sofia |
| FDD-DADOS-08 | Ao remover um endpoint, as linhas da outbox e o histórico dele são apagados em cascata. A DLQ é mantida como evidência, sem FK. **[proposta do FDD]** | [09:18] Diego |
| FDD-DADOS-09 | Chave primária `id` em UUID, como no resto do projeto. | [09:51] Larissa |
| FDD-DADOS-10 | Padrão de modelagem do projeto: `@db.Char(36)`, `@@map` em snake_case, `createdAt`/`updatedAt`. | `prisma/schema.prisma` |
| FDD-DADOS-12 | Coluna `seq` autoincremental, só para ordenação e desempate entre eventos criados no mesmo instante. O `id` continua UUID. Índice `(webhookId, orderId, status, seq)` atende a regra de ordem do worker. **[proposta do FDD]** | [09:12] Diego |

---

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox (`FDD-FLUXO-01`)

```
PATCH /api/v1/orders/:id/status
 └─ OrderController.changeStatus
     └─ OrderService.changeStatus  ── prisma.$transaction(tx) ─────────────────────┐
         1. findUnique(order) / canTransition / debitStock|replenishStock           │
         2. tx.order.update({ status: to })                                         │
         3. tx.orderStatusHistory.create(...)                                       │
         4. publishWebhookEvent(tx, order, from, to)   ◄── NOVO                     │
              a. endpoints = tx.webhookEndpoint.findMany({                          │
                   where: { customerId: order.customerId, active: true } })         │
              b. alvo = endpoints.filter(e => e.events.includes(to))                │
              c. se alvo vazio → return (nada é gravado)                            │
              d. para cada endpoint em alvo:                                        │
                   eventId = uuidv4()                                               │
                   payload = buildPayload(order, from, to, eventId, now)            │
                   tx.webhookOutbox.create({ status: PENDING, attempts: 0,          │
                                             nextAttemptAt: now, ... })             │
         5. findUnique(refreshed) e retorno                                         │
     commit ◄─────────────── qualquer erro em 1-5 → rollback de tudo ──────────────┘
```

Regras:
- `publishWebhookEvent` é uma **função pura do módulo**, sem estado. Ela recebe o `tx` e não exige injetar um repository no `OrderService`.
- Uma falha ao inserir na outbox propaga a exceção, e o `$transaction` faz rollback da mudança de status.
- O payload é montado a partir do `order` já carregado na transação (`id`, `orderNumber`, `customerId`, `totalCents`).

### 5.2 Processamento pelo worker (`FDD-FLUXO-02`)

`src/worker.ts` inicia o loop chamando `WebhookProcessor.runCycle()` a cada **2 segundos**. O ciclo **não espera as entregas terminarem**: ele só dispara os envios elegíveis e volta a dormir. Isso evita que um cliente lento atrase os eventos de outros clientes.

```
runCycle():
  1. vagas = WEBHOOK_MAX_IN_FLIGHT − envios em andamento          -- 10 [proposta do FDD]
     se vagas = 0 → encerra o ciclo
  2. eventos = SELECT o.* FROM webhook_outbox o
               WHERE o.status = 'PENDING' AND o.next_attempt_at <= NOW()
                 AND NOT EXISTS (                              -- ordem por pedido e endpoint
                   SELECT 1 FROM webhook_outbox anterior
                   WHERE anterior.webhook_id = o.webhook_id
                     AND anterior.order_id   = o.order_id
                     AND anterior.seq < o.seq
                     AND anterior.status IN ('PENDING', 'PROCESSING'))
               ORDER BY o.seq ASC
               LIMIT 100
     selecionados = candidatos, na ordem, respeitando:
       - no máximo WEBHOOK_MAX_IN_FLIGHT_PER_ENDPOINT envios em andamento por webhook_id  -- 2 [proposta do FDD]
       - no máximo `vagas` no total
  3. para cada selecionado: UPDATE status = 'PROCESSING' WHERE id = ? AND status = 'PENDING'
     (se 0 linhas afetadas, descarta) e dispara deliver(evento) sem await
  4. loga o resumo do ciclo (disparados, em andamento, backlog, idade do pendente mais antigo)

deliver(evento):
  a. endpoint = findUnique(webhookId)
     - endpoint inativo → DLQ com WEBHOOK_ENDPOINT_INACTIVE, sem retry        [proposta do FDD]
     - secret vazia     → DLQ com WEBHOOK_SECRET_REQUIRED, sem retry          [proposta do FDD]
  b. body = JSON.stringify(payload)
     - Buffer.byteLength(body) > 64 KB → DLQ com WEBHOOK_PAYLOAD_TOO_LARGE, sem retry
  c. headers = buildHeaders(evento, endpoint, body)        (seção 6.2)
  d. resposta = fetch(endpoint.url, { method: 'POST', headers, body,
                                      signal: AbortSignal.timeout(10_000) })
  e. grava webhook_deliveries (attempt, success, responseStatus,
     responseBody truncado em 2 KB [proposta do FDD], durationMs, errorCode)
  f. 2xx → status = DELIVERED, attempts += 1
     timeout, rede ou fora de 2xx → aplica a política de retry (5.3)
  g. try/finally em volta de a–f: qualquer exceção inesperada (ex.: erro de banco ao gravar
     a entrega) aplica a política de retry com WEBHOOK_PROCESSING_ERROR, e o finally
     sempre libera a vaga. Nenhuma linha fica presa em PROCESSING.
```

- **Ordem por pedido.** A comparação usa `seq` (FDD-DADOS-12), que desempata eventos criados no mesmo milissegundo. A cláusula `NOT EXISTS` só libera um evento quando todos os anteriores do mesmo pedido, para o mesmo endpoint, estão `DELIVERED` ou `FAILED`. Um `PAID` em retry segura o `SHIPPED` daquele pedido até ser entregue ou ir para a DLQ ([ADR-008](adrs/ADR-008-ordenacao-por-pedido-com-worker-unico.md)). Pedidos diferentes seguem em paralelo.
- **Isolamento entre clientes (`FDD-FLUXO-07`).** Um endpoint travado ocupa no máximo `WEBHOOK_MAX_IN_FLIGHT_PER_ENDPOINT` vagas, por até 10 s cada. As outras vagas continuam livres para os demais clientes, mesmo que o backlog do endpoint travado seja o mais antigo.
- O envio usa o `fetch` nativo do Node 20 (o `package.json` exige `node >= 20`). **Nenhuma dependência nova de HTTP.**
- O worker é um **processo único**. O paralelismo acontece só dentro dele, com envios HTTP assíncronos. O `UPDATE ... WHERE status = 'PENDING'` (passo 3) protege contra uma segunda instância subida por engano.

### 5.3 Retry (`FDD-FLUXO-03`)

A tentativa inicial é seguida de até **5 retentativas** ([ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md)):

| Tentativa | Quando ocorre | `attempts` após falhar |
|---|---|---|
| 1 (inicial) | No primeiro ciclo após o commit (≤ 2 s) | 1 |
| 2 | 1 min após a falha 1 | 2 |
| 3 | 5 min após a falha 2 | 3 |
| 4 | 30 min após a falha 3 | 4 |
| 5 | 2 h após a falha 4 | 5 |
| 6 | 12 h após a falha 5 | 6 → DLQ |

```ts
const RETRY_DELAYS_MS = [60_000, 300_000, 1_800_000, 7_200_000, 43_200_000];
const MAX_ATTEMPTS = RETRY_DELAYS_MS.length + 1;

if (attempts < MAX_ATTEMPTS) {
  status = 'PENDING';
  nextAttemptAt = now + RETRY_DELAYS_MS[attempts - 1];
  lastErrorCode = errorCode;
} else {
  moveToDeadLetter(evento, errorCode, reason);
}
```

Falhas que **contam para retry**: `WEBHOOK_DELIVERY_TIMEOUT`, `WEBHOOK_DELIVERY_NETWORK_ERROR` e `WEBHOOK_DELIVERY_HTTP_ERROR` (status fora de 2xx).
Falhas que **vão direto para a DLQ**: `WEBHOOK_PAYLOAD_TOO_LARGE`, `WEBHOOK_SECRET_REQUIRED` e `WEBHOOK_ENDPOINT_INACTIVE`.

### 5.4 DLQ e replay (`FDD-FLUXO-04`)

**Mover para a DLQ** (uma transação):
1. `webhook_outbox.status = FAILED` e `lastErrorCode = <código>`;
2. insere `webhook_dead_letter` com `outboxId`, `webhookId`, `eventId`, `payload`, `failureReason`, `errorCode` e `attempts`;
3. loga `webhook_moved_to_dead_letter` em nível `warn`.

**Replay** (`POST /api/v1/admin/webhooks/dead-letter/:id/replay`, role `ADMIN`):
1. busca a dead letter. Não encontrada → `WEBHOOK_DEAD_LETTER_NOT_FOUND` (404);
2. se `replayedAt` já estiver preenchido → `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` (409). Cada dead letter pode ser reprocessada uma vez; se o evento falhar de novo, ele gera uma nova dead letter **[proposta do FDD]**;
3. busca a linha da outbox e o endpoint. Endpoint removido → `WEBHOOK_NOT_FOUND` (404); endpoint inativo → `WEBHOOK_ENDPOINT_INACTIVE` (409);
4. em uma transação: `webhook_outbox` volta a `status = PENDING`, `attempts = 0` e `nextAttemptAt = now`, **mantendo o mesmo `eventId`**; a dead letter recebe `replayedAt = now` e `replayedById = req.user.id`;
5. loga `webhook_dead_letter_replayed` em nível `info`, com `userId`, `deadLetterId` e `eventId`, para auditoria.

No replay, o evento chega **depois** dos eventos seguintes do mesmo pedido, que foram liberados quando ele foi para a DLQ. O cliente reconcilia pelo `timestamp` e por `from_status`/`to_status` do payload ([ADR-008](adrs/ADR-008-ordenacao-por-pedido-com-worker-unico.md)).

### 5.5 Rotação de secret (`FDD-FLUXO-05`)

1. `POST /api/v1/webhooks/:id/rotate-secret` gera uma nova secret com `crypto.randomBytes(32).toString('hex')` **[proposta do FDD — algoritmo de geração a validar na revisão de segurança]**.
2. A secret atual vai para `previousSecret`, com `previousSecretExpiresAt = now + 24h`. A nova vira `secret`.
3. Uma nova rotação dentro das 24h descarta a `previousSecret` anterior, e só convivem duas secrets. **[proposta do FDD]** Isso encurta o prazo da secret mais antiga e está registrado para a revisão da Sofia ([RFC-Q-08](RFC.md#5-questões-em-aberto)).
4. Enquanto `previousSecretExpiresAt > now`, o worker envia **duas assinaturas** em `X-Signature` (seção 6.2), para o cliente validar com qualquer uma. **[proposta do FDD — validar na revisão de segurança]**
5. Depois do prazo, a `previousSecret` é ignorada. A limpeza do campo acontece na próxima rotação ou atualização.

### 5.6 Recuperação e shutdown do worker (`FDD-FLUXO-06`)

- **Na inicialização:** linhas em `PROCESSING` (worker morreu no meio do envio) voltam para `PENDING`. O cliente pode receber de novo, o que é coerente com at-least-once ([ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)).
- **No `SIGINT`/`SIGTERM`:** o worker para de agendar ciclos, aguarda os envios em andamento por até 10 s (o timeout de cada envio), chama `prisma.$disconnect()` e sai. Envios abandonados ficam em `PROCESSING` e voltam a `PENDING` no próximo start. É o mesmo padrão de `src/server.ts`.

---

## 6. Contratos públicos

Todas as rotas ficam sob `/api/v1`, como em `src/app.ts`. Todas exigem `Authorization: Bearer <jwt>` (`authenticate`). Os erros seguem o formato do `errorMiddleware`:

```json
{ "error": { "code": "WEBHOOK_NOT_FOUND", "message": "Webhook not found" } }
```

O `customerId` é enviado **no body** na criação e **no path** na listagem, as duas opções citadas na reunião. Ele não vem do JWT, porque o JWT é do usuário operador. **[proposta do FDD quanto a qual opção usar em cada rota]**

### 6.1 Endpoints da API

#### FDD-CONTRATO-01 — Criar endpoint

`POST /api/v1/webhooks`

```json
{
  "customerId": "6f1c2a9e-4b7d-4c1e-9a55-0d6b8e2f1a10",
  "url": "https://hooks.atlascomercial.com.br/oms",
  "events": ["SHIPPED", "DELIVERED"]
}
```

`201 Created`. A secret é devolvida **somente nesta resposta e na rotação**.

```json
{
  "id": "0b8f5d3e-2c41-4f7a-b1e9-7e6a4c2d9f01",
  "customerId": "6f1c2a9e-4b7d-4c1e-9a55-0d6b8e2f1a10",
  "url": "https://hooks.atlascomercial.com.br/oms",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "9c2f6a0e5b...e41d",
  "createdAt": "2026-10-14T13:00:00.000Z",
  "updatedAt": "2026-10-14T13:00:00.000Z"
}
```

| Status | Quando |
|---|---|
| 201 | Criado |
| 400 | `VALIDATION_ERROR` com detalhe `WEBHOOK_INVALID_URL` (URL não-https ou inválida) ou `WEBHOOK_INVALID_EVENTS` |
| 401 | `UNAUTHORIZED` |
| 404 | `WEBHOOK_CUSTOMER_NOT_FOUND` |

Schema Zod (`webhook.schemas.ts`):

```ts
const subscribableStatus = z.nativeEnum(OrderStatus).refine((s) => s !== OrderStatus.PENDING, {
  message: 'WEBHOOK_INVALID_EVENTS',
});

export const createWebhookSchema = z.object({
  customerId: z.string().uuid(),
  url: z.string().url().max(2048).refine((u) => u.startsWith('https://'), {
    message: 'WEBHOOK_INVALID_URL',
  }),
  events: z.array(subscribableStatus).min(1, 'WEBHOOK_INVALID_EVENTS'),
});
```

#### FDD-CONTRATO-02 — Listar endpoints de um cliente

`GET /api/v1/webhooks/customers/6f1c2a9e-4b7d-4c1e-9a55-0d6b8e2f1a10?page=1&pageSize=20`

`200 OK`, no formato paginado de `src/shared/http/response.ts`, **sem `secret`**:

```json
{
  "data": [
    {
      "id": "0b8f5d3e-2c41-4f7a-b1e9-7e6a4c2d9f01",
      "customerId": "6f1c2a9e-4b7d-4c1e-9a55-0d6b8e2f1a10",
      "url": "https://hooks.atlascomercial.com.br/oms",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-10-14T13:00:00.000Z",
      "updatedAt": "2026-10-14T13:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

| Status | Quando |
|---|---|
| 200 | Lista (pode ser vazia) |
| 400 | `VALIDATION_ERROR` (`customerId` do path não é UUID) |
| 401 | `UNAUTHORIZED` |

#### FDD-CONTRATO-03 — Editar endpoint

`PATCH /api/v1/webhooks/:id`

```json
{ "events": ["PAID", "SHIPPED", "DELIVERED"], "active": true }
```

`200 OK`: o endpoint atualizado, no mesmo formato da listagem (sem `secret`):

```json
{
  "id": "0b8f5d3e-2c41-4f7a-b1e9-7e6a4c2d9f01",
  "customerId": "6f1c2a9e-4b7d-4c1e-9a55-0d6b8e2f1a10",
  "url": "https://hooks.atlascomercial.com.br/oms",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true,
  "createdAt": "2026-10-14T13:00:00.000Z",
  "updatedAt": "2026-10-14T13:20:00.000Z"
}
```
 Os campos aceitos são `url`, `events` e `active`, todos opcionais, com as mesmas validações da criação. A mudança de `events` vale apenas para eventos futuros ([ADR-007](adrs/ADR-007-snapshot-e-filtro-na-insercao.md)).

| Status | Quando |
|---|---|
| 200 | Atualizado |
| 400 | `VALIDATION_ERROR` (`WEBHOOK_INVALID_URL`, `WEBHOOK_INVALID_EVENTS`) |
| 401 | `UNAUTHORIZED` |
| 404 | `WEBHOOK_NOT_FOUND` |

#### FDD-CONTRATO-04 — Remover endpoint

`DELETE /api/v1/webhooks/:id` → `204 No Content`, sem corpo. Remove o endpoint e, em cascata, as linhas da outbox e o histórico de entregas dele (FDD-DADOS-08).

| Status | Quando |
|---|---|
| 204 | Removido |
| 401 | `UNAUTHORIZED` |
| 404 | `WEBHOOK_NOT_FOUND` |

#### FDD-CONTRATO-05 — Rotacionar secret

`POST /api/v1/webhooks/:id/rotate-secret` (sem body)

`200 OK`:

```json
{
  "id": "0b8f5d3e-2c41-4f7a-b1e9-7e6a4c2d9f01",
  "secret": "4ad1e0b7c3...9f2a",
  "previousSecretExpiresAt": "2026-10-15T13:00:00.000Z"
}
```

| Status | Quando |
|---|---|
| 200 | Rotacionada |
| 401 | `UNAUTHORIZED` |
| 404 | `WEBHOOK_NOT_FOUND` |
| 409 | `WEBHOOK_ENDPOINT_INACTIVE` (rotação bloqueada em endpoint inativo **[proposta do FDD]**) |

#### FDD-CONTRATO-06 — Histórico de entregas

`GET /api/v1/webhooks/:id/deliveries?limit=100`. O `limit` vai de 1 a 100, com default 100. Retorna as tentativas mais recentes primeiro.

`200 OK`:

```json
{
  "data": [
    {
      "id": "c1e7a2b4-5d6f-4a8b-9c0d-1e2f3a4b5c6d",
      "eventId": "3f0e8b6a-1c2d-4e5f-8a9b-0c1d2e3f4a5b",
      "attempt": 2,
      "success": true,
      "responseStatus": 200,
      "responseBody": "{\"ok\":true}",
      "durationMs": 184,
      "errorCode": null,
      "payload": {
        "event_id": "3f0e8b6a-1c2d-4e5f-8a9b-0c1d2e3f4a5b",
        "event_type": "order.status_changed",
        "order_number": "ORD-000123",
        "to_status": "SHIPPED"
      },
      "createdAt": "2026-10-14T13:05:02.311Z"
    }
  ]
}
```

O `payload` vem da linha da outbox (join por `outboxId`). O exemplo está resumido. Status: `200`, `401 UNAUTHORIZED`, `404 WEBHOOK_NOT_FOUND`.

#### FDD-CONTRATO-07 — Replay de dead letter (admin)

`POST /api/v1/admin/webhooks/dead-letter/:id/replay`. Exige `authenticate` + `requireRole('ADMIN')`. Sem body.

`202 Accepted`. O evento volta para a fila e será entregue no próximo ciclo:

```json
{
  "deadLetterId": "8d2b6f1a-3c4e-4f5a-9b7c-2d1e0f3a4b5c",
  "outboxId": "5e4d3c2b-1a0f-4e9d-8c7b-6a5f4e3d2c1b",
  "eventId": "3f0e8b6a-1c2d-4e5f-8a9b-0c1d2e3f4a5b",
  "status": "PENDING",
  "replayedBy": "a7b6c5d4-e3f2-4a1b-9c8d-7e6f5a4b3c2d"
}
```

| Status | Quando |
|---|---|
| 202 | Reenfileirado |
| 401 | `UNAUTHORIZED` |
| 403 | `FORBIDDEN` (usuário não `ADMIN`; erro já existente do `requireRole`) |
| 404 | `WEBHOOK_DEAD_LETTER_NOT_FOUND` ou `WEBHOOK_NOT_FOUND` |
| 409 | `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` ou `WEBHOOK_ENDPOINT_INACTIVE` |

### 6.2 Requisição enviada ao cliente

#### FDD-CONTRATO-08 — Webhook outbound

`POST <url cadastrada>`

| Header | Valor |
|---|---|
| `Content-Type` | `application/json` |
| `X-Event-Id` | UUID do evento, estável em retentativas e replays |
| `X-Webhook-Id` | ID do endpoint cadastrado |
| `X-Timestamp` | Momento do envio em ISO 8601 (muda a cada tentativa) |
| `X-Signature` | HMAC-SHA256 do corpo bruto usando a secret do endpoint, no formato `sha256=<hex>`. Durante o grace period: `sha256=<hex_nova>,sha256=<hex_anterior>`. **[proposta do FDD quanto ao formato]** |

Corpo:

```json
{
  "event_id": "3f0e8b6a-1c2d-4e5f-8a9b-0c1d2e3f4a5b",
  "event_type": "order.status_changed",
  "timestamp": "2026-10-14T13:05:00.120Z",
  "order_id": "1d2c3b4a-5f6e-4d7c-8b9a-0f1e2d3c4b5a",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "6f1c2a9e-4b7d-4c1e-9a55-0d6b8e2f1a10",
  "total_cents": 158990
}
```

Semântica:
- `timestamp` é o momento da mudança de status (snapshot), e `X-Timestamp` é o momento do envio.
- Qualquer resposta **2xx** em até 10 s conta como entregue. O corpo da resposta é ignorado (só fica salvo no histórico).
- O cliente deve validar a assinatura com o corpo exatamente como recebido e deduplicar por `X-Event-Id`.

Validação do lado do cliente (para o portal do desenvolvedor):

```ts
const expected = createHmac('sha256', secret).update(rawBody).digest('hex');
const valid = header.split(',').some((part) =>
  timingSafeEqual(Buffer.from(part.replace('sha256=', '')), Buffer.from(expected)),
);
```

---

## 7. Matriz de erros

As classes ficam em `src/modules/webhooks/webhook.errors.ts`, estendendo as classes de `src/shared/errors/http-errors.ts`.

| ID | Código | HTTP | Onde ocorre | Classe base | Comportamento |
|---|---|---|---|---|---|
| FDD-ERR-01 | `WEBHOOK_NOT_FOUND` | 404 | CRUD, deliveries, rotação, replay | `AppError` (404) | Retorna o erro |
| FDD-ERR-02 | `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | Criação | `AppError` (404) | Retorna o erro |
| FDD-ERR-03 | `WEBHOOK_INVALID_URL` | 400 | Criação/edição | via Zod → `ValidationError` | Aparece em `details[].message` do `VALIDATION_ERROR`* |
| FDD-ERR-04 | `WEBHOOK_INVALID_EVENTS` | 400 | Criação/edição | via Zod → `ValidationError` | Igual ao anterior* |
| FDD-ERR-05 | `WEBHOOK_ENDPOINT_INACTIVE` | 409 | Rotação, replay; worker | `ConflictError` | API: retorna o erro. Worker: DLQ sem retry |
| FDD-ERR-06 | `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Replay | `AppError` (404) | Retorna o erro |
| FDD-ERR-07 | `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | Replay | `ConflictError` | Retorna o erro |
| FDD-ERR-08 | `WEBHOOK_SECRET_REQUIRED` | — | Worker (secret vazia; a coluna é NOT NULL, então é uma verificação defensiva) | `AppError` (500) | DLQ sem retry, log `error` |
| FDD-ERR-09 | `WEBHOOK_PAYLOAD_TOO_LARGE` | — | Worker (corpo > 64 KB) | `AppError` (500) | DLQ sem retry, log `error` |
| FDD-ERR-10 | `WEBHOOK_DELIVERY_TIMEOUT` | — | Worker (sem resposta em 10 s) | — (registrado em delivery) | Retry |
| FDD-ERR-11 | `WEBHOOK_DELIVERY_NETWORK_ERROR` | — | Worker (DNS, conexão recusada, TLS) | — | Retry |
| FDD-ERR-12 | `WEBHOOK_DELIVERY_HTTP_ERROR` | — | Worker (resposta fora de 2xx) | — | Retry |
| FDD-ERR-13 | `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | — | Worker (6ª tentativa falhou) | — | Vira `errorCode` da DLQ, com a última causa em `failureReason` |
| FDD-ERR-16 | `WEBHOOK_PROCESSING_ERROR` | — | Worker (exceção inesperada fora do envio HTTP) | — | Retry, log `error` |
| FDD-ERR-14 | `WEBHOOK_OUTBOX_INSERT_FAILED` | 500 | `publishWebhookEvent` | `AppError` (500) | Rollback da mudança de status. O `errorMiddleware` responde 500 |

\* O `validate` (`src/middlewares/validate.middleware.ts`) converte todo `ZodError` em `ValidationError` com código `VALIDATION_ERROR`. A Sofia pediu que a checagem de `https` fique no schema Zod ([09:23]), então o código específico aparece como mensagem do issue, sem alterar o middleware **[proposta do FDD]**. Se o time preferir `WEBHOOK_INVALID_URL` no campo `code`, a alternativa é repetir a checagem no `WebhookService` e lançar um `AppError` próprio. Resposta com a abordagem proposta:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [{ "path": "url", "message": "WEBHOOK_INVALID_URL" }]
  }
}
```

---

## 8. Estratégias de resiliência

| Mecanismo | Valor | Origem |
|---|---|---|
| Timeout por requisição | 10 s (`AbortSignal.timeout(10_000)`) | [09:42] Diego |
| Retentativas | 5 retentativas após a tentativa inicial (interpretação; confirmar em [RFC-Q-06](RFC.md#5-questões-em-aberto)) | [09:15] Diego, [09:17] Larissa |
| Backoff | 1m, 5m, 30m, 2h, 12h (≈ 14h36 no total) | [09:17] Diego |
| Fallback | DLQ em `webhook_dead_letter` + replay manual por ADMIN | [09:18] Diego |
| Isolamento da transação de pedidos | Outbox; a entrega nunca roda dentro de `changeStatus` | [09:04] Bruno, [09:06] Diego |
| Limite de payload | 64 KB, rejeição sem truncar | [09:23] Sofia, [09:24] Diego e Larissa |
| Recuperação de crash | `PROCESSING` → `PENDING` no start do worker | Derivado de [09:24] Diego (at-least-once) |
| Proteção contra worker duplicado | `UPDATE ... WHERE status='PENDING'` antes de enviar | Derivado de [09:12] Diego (single-worker) |

Um endpoint lento ou fora do ar atrasa só os próprios eventos. O envio é assíncrono dentro do worker (5.2) e o backoff tira o evento dos ciclos seguintes.

---

## 9. Observabilidade

### 9.1 Logs (Pino existente, `src/shared/logger/index.ts`)

| Evento | Nível | Campos |
|---|---|---|
| `webhook_event_enqueued` | info | `eventId`, `webhookId`, `orderId`, `toStatus` |
| `webhook_delivery_attempt` | info | `eventId`, `webhookId`, `orderId`, `attempt`, `responseStatus`, `durationMs`, `success` |
| `webhook_delivery_failed` | warn | os mesmos + `errorCode`, `nextAttemptAt` |
| `webhook_moved_to_dead_letter` | warn | `deadLetterId`, `eventId`, `webhookId`, `customerId`, `errorCode`, `attempts` |
| `webhook_dead_letter_replayed` | info | `deadLetterId`, `eventId`, `userId` (auditoria) |
| `webhook_worker_cycle` | debug | `dispatched`, `inFlight`, `pendingBacklog`, `oldestPendingAgeMs` |
| `webhook_worker_started` / `webhook_worker_stopped` | info | `pollIntervalMs`, `batchSize` |

Os `redactPaths` do logger ganham `'*.secret'`, `'*.previousSecret'` e `'*.signature'`. Secret e assinatura nunca vão para o log.

### 9.2 Métricas

O projeto não tem biblioteca de métricas, e a decisão foi não adicionar dependências ([ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)). Na primeira versão, as métricas são derivadas dos logs estruturados e de consultas às tabelas:

| Métrica | Fonte | Uso |
|---|---|---|
| `webhook_outbox_pending_backlog` | `pendingBacklog` do log de ciclo | Fila acumulando |
| `webhook_outbox_oldest_pending_age_ms` | `oldestPendingAgeMs` | Lag de entrega (meta < 10 s) |
| `webhook_delivery_success_rate` | `webhook_deliveries.success` | Saúde geral e por cliente |
| `webhook_delivery_duration_ms` (p50/p95) | `webhook_deliveries.durationMs` | Clientes lentos perto do timeout |
| `webhook_first_attempt_latency_ms` | 1º `webhook_deliveries.createdAt` − `webhook_outbox.createdAt` | Requisito de < 10 s |
| `webhook_dead_letter_total` | `COUNT(webhook_dead_letter)` | Falhas permanentes |
| `webhook_retries_total` | `webhook_deliveries` com `attempt > 1` | Instabilidade de clientes e insumo para a questão de rate limiting |

### 9.3 Tracing

Não há tracing distribuído no projeto. A correlação é feita por IDs presentes em todos os logs:
- `orderId` liga a mudança de status (log `http_request` do `requestLogger`) ao `webhook_event_enqueued`;
- `eventId` liga o enqueue, cada tentativa, a DLQ e o replay, e é o mesmo ID que o cliente recebe em `X-Event-Id`;
- `webhookId` (`X-Webhook-Id`) permite ao suporte cruzar uma reclamação do cliente com o histórico.

A instrumentação com OpenTelemetry fica como evolução, fora desta feature.

---

## 10. Integração com o sistema existente

| Arquivo existente | Como o módulo se integra |
|---|---|
| `src/modules/orders/order.service.ts` | Em `changeStatus`, logo após `tx.orderStatusHistory.create(...)` e antes do `findUnique` final, entra `await publishWebhookEvent(tx, order, from, to)`. A chamada fica **dentro** do `prisma.$transaction` já existente, e uma falha nela faz rollback ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)). O construtor de `OrderService` não muda, porque a função é importada do módulo e recebe o `tx`. `create` não é alterado. |
| `src/modules/orders/order.status.ts` | O filtro `events` aceita só os status que podem ser **destino** de uma transição no mapa `transitions` (`PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`). `PENDING` é rejeitado, porque só existe na criação. |
| `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` | `webhook.errors.ts` cria `WebhookNotFoundError`, `WebhookEndpointInactiveError`, `WebhookDeadLetterAlreadyReplayedError` etc., estendendo `AppError`/`ConflictError`, como `InvalidStatusTransitionError` e `InsufficientStockError`. A resposta HTTP mantém o formato `{ error: { code, message, details } }`. |
| `src/middlewares/error.middleware.ts` | **Sem alteração.** Os novos erros são `AppError` e já são tratados. Os erros do worker não passam por aqui e são logados pelo processor. |
| `src/middlewares/validate.middleware.ts` | Valida `webhook.schemas.ts` (body, query e params), como em `order.routes.ts`. |
| `src/middlewares/auth.middleware.ts` | `authenticate` em todas as rotas do módulo. `requireRole('ADMIN')` na rota de replay, do mesmo jeito que `src/modules/users/user.routes.ts`. |
| `src/routes/index.ts` | `Controllers` ganha `webhooks` e `adminWebhooks`. `buildApiRouter` monta `router.use('/webhooks', buildWebhookRouter(...))`, que inclui `GET /customers/:customerId` para a listagem, e `router.use('/admin/webhooks', buildAdminWebhookRouter(...))`. |
| `src/app.ts` | `buildControllers` instancia `WebhookRepository`, `WebhookService` e `WebhookController` com o mesmo `prisma`, como nos outros módulos. |
| `src/server.ts` | Serve de modelo para o novo `src/worker.ts`: `bootstrap()`, handlers de `SIGINT`/`SIGTERM` com `prisma.$disconnect()` e `logger.fatal` em falha de bootstrap. |
| `src/config/database.ts` | O worker chama `createPrismaClient()` para ter o **próprio** `PrismaClient`, com a mesma `DATABASE_URL`. |
| `src/config/env.ts` | O worker reutiliza o `env` validado. Nenhuma variável nova é obrigatória. Intervalo, lote e timeout ficam como constantes do módulo. |
| `src/shared/logger/index.ts` | Logger reaproveitado. `redactPaths` recebe `*.secret`, `*.previousSecret` e `*.signature`. |
| `src/shared/http/response.ts` | `paginated()` formata a listagem de endpoints. |
| `prisma/schema.prisma` | Quatro models novos, um enum e a relação `Customer.webhookEndpoints`. Migration nova em `prisma/migrations/`. |
| `package.json` | Scripts `"worker": "node --env-file=.env dist/worker.js"` e `"worker:dev": "tsx watch --env-file=.env src/worker.ts"`, no mesmo molde de `start` e `dev`. |
| `tests/setup.ts` | O `beforeEach` passa a limpar `webhookDelivery`, `webhookDeadLetter`, `webhookOutbox` e `webhookEndpoint` antes das tabelas de pedidos. |
| `tests/orders.test.ts` e `tests/helpers/factories.ts` | Novos casos para outbox atômica e uma factory de `WebhookEndpoint`. |

Arquivos novos:

```
src/worker.ts
src/modules/webhooks/
├── webhook.controller.ts       CRUD, rotação, deliveries
├── webhook.admin.controller.ts replay da DLQ
├── webhook.service.ts          regras de configuração e rotação
├── webhook.repository.ts       acesso a webhook_endpoints e webhook_deliveries
├── webhook.routes.ts           buildWebhookRouter e buildAdminWebhookRouter
├── webhook.schemas.ts          Zod
├── webhook.errors.ts           erros WEBHOOK_*
├── webhook.publisher.ts        publishWebhookEvent(tx, order, from, to) e buildPayload
├── webhook.processor.ts        loop do worker, retry e DLQ
└── webhook.signature.ts        sign(body, secrets) com crypto nativo
tests/webhooks.test.ts
```

---

## 11. Dependências e compatibilidade

- **Sem dependências novas.** HTTP com o `fetch` nativo (Node ≥ 20), HMAC e geração de secret com `node:crypto`, UUID com o pacote `uuid` já presente, validação com Zod, log com Pino e persistência com Prisma 5.22 + MySQL 8.
- **Banco:** migration aditiva (tabelas novas e uma relação). Nenhuma coluna existente muda.
- **API:** nenhum endpoint existente muda de contrato. `PATCH /orders/:id/status` continua com o mesmo request e response. A única diferença é uma escrita a mais na transação.
- **Deploy:** um processo novo (`npm run worker`), executado **em instância única**. A API sobe e funciona sem o worker (os eventos ficam acumulados na outbox até ele subir).
- **Clientes:** precisam expor um endpoint `https`, validar `X-Signature` e deduplicar por `X-Event-Id`.

## 12. Critérios de aceite técnicos

| ID | Critério |
|---|---|
| FDD-CA-01 | Uma mudança de status com endpoint assinante cria exatamente 1 linha `PENDING` na outbox por endpoint, na mesma transação. |
| FDD-CA-02 | Se `publishWebhookEvent` lançar erro, o status do pedido, o histórico e o estoque permanecem como antes (teste de rollback). |
| FDD-CA-03 | Uma mudança para um status não assinado não cria linha na outbox. |
| FDD-CA-04 | Um evento pendente, sem evento anterior retido do mesmo pedido e endpoint, recebe a primeira tentativa em até 10 s após o commit, com o worker rodando. |
| FDD-CA-05 | Requisições enviadas têm os 5 headers da seção 6.2, e `X-Signature` confere com o HMAC-SHA256 do corpo. |
| FDD-CA-06 | Um endpoint que responde 500 recebe as tentativas nos intervalos 1m/5m/30m/2h/12h e o evento termina na DLQ após a 6ª falha (teste com relógio simulado). |
| FDD-CA-07 | Uma resposta sem retorno em 10 s é registrada como `WEBHOOK_DELIVERY_TIMEOUT` e vai para retry. |
| FDD-CA-08 | O replay feito por `ADMIN` volta o evento a `PENDING` com o mesmo `eventId`. O replay feito por `OPERATOR` recebe 403. |
| FDD-CA-09 | Uma URL `http://` é recusada com 400. |
| FDD-CA-10 | Depois de uma rotação, as entregas nas 24h seguintes carregam as duas assinaturas, e depois disso só a nova. |
| FDD-CA-11 | Um payload acima de 64 KB vai para a DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`, sem nenhum envio. |
| FDD-CA-12 | O `GET /webhooks/:id/deliveries` devolve no máximo 100 registros, do mais recente para o mais antigo. |
| FDD-CA-13 | Secret e assinatura nunca aparecem em logs (teste dos `redactPaths`). |
| FDD-CA-14 | A suíte existente (`tests/orders.test.ts`, `tests/auth.test.ts`) continua passando sem alteração de asserts. |
| FDD-CA-15 | Um `SHIPPED` gravado depois de um `PAID` em retry, do mesmo pedido e endpoint, só é enviado depois que o `PAID` for entregue ou for para a DLQ. |
| FDD-CA-16 | Um endpoint que não responde (timeout de 10 s), mesmo com mais de 10 eventos pendentes, não atrasa a primeira tentativa de eventos de outros clientes além de um ciclo de polling. |

## 13. Riscos e mitigação

| ID | Risco | Mitigação |
|---|---|---|
| FDD-RISCO-01 | A consulta de endpoints dentro de `changeStatus` aumenta a duração da transação. | Índice `(customerId, active)`. A consulta é pequena (poucos endpoints por cliente). Medir a latência do `PATCH /orders/:id/status` antes e depois. |
| FDD-RISCO-02 | O backlog cresce se um cliente ficar lento perto do timeout. | O timeout de 10 s limita o pior caso. Métricas de backlog e lag (9.2). O rate limiting fica como questão em aberto. |
| FDD-RISCO-03 | Uma segunda instância de worker subida por engano quebra a ordem por pedido. | Proteção por `UPDATE ... WHERE status='PENDING'` e documentação de deploy com instância única ([ADR-008](adrs/ADR-008-ordenacao-por-pedido-com-worker-unico.md)). |
| FDD-RISCO-04 | Uma secret vaza por log ou por resposta. | `redactPaths`, secret só nas respostas de criação e rotação, revisão de segurança da Sofia. |
| FDD-RISCO-05 | O crescimento contínuo das tabelas de outbox e deliveries. | Índices por status e data. O arquivamento fica para a feature futura de cerca de 30 dias. |
| FDD-RISCO-06 | Para preservar a ordem, um evento em retry segura os seguintes do mesmo pedido para o mesmo endpoint, que podem atrasar até cerca de 15h. | O atraso fica restrito àquele pedido e àquele endpoint. Depois que o evento retido vai para a DLQ, os seguintes são liberados. A limitação está documentada no ADR-008. |
