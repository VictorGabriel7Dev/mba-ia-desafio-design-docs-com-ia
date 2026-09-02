# FDD - Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Autores** | Bruno (Eng. Pleno, Pedidos) e Diego (Eng. Sênior, Plataforma) |
| **Revisores** | Larissa (Tech Lead) · Sofia (Eng. Segurança) |
| **Status** | Pronto para implementação |
| **Relacionados** | [PRD](PRD.md) · [RFC](RFC.md) · [ADRs](adrs/) |

---

## 1. Contexto e motivação técnica

O [RFC](RFC.md) fixou a abordagem: **outbox no MySQL** com **worker em processo separado**. Este
documento desce ao "como construir".

A motivação técnica central é uma só: **a transação de mudança de status não pode depender de rede
externa**. O método `changeStatus` em `src/modules/orders/order.service.ts` hoje executa dentro de
`this.prisma.$transaction` a atualização do pedido, a inserção em `order_status_history` e a
movimentação de estoque. Colocar uma chamada HTTP ali dentro acopla o nosso fluxo à
disponibilidade de terceiro.

A solução grava o evento **na mesma transação** e delega a rede a outro processo.

## 2. Objetivos técnicos

| # | Objetivo |
| --- | --- |
| OT-1 | Gravar o evento atomicamente com a mudança de status, sem janela de inconsistência |
| OT-2 | Entregar em menos de 10s no caso normal, com piso de 2s pelo polling |
| OT-3 | Resistir a indisponibilidade do cliente por até ~15h antes de desistir |
| OT-4 | Não introduzir dependência de infraestrutura nova |
| OT-5 | Permitir ao cliente verificar origem e integridade de cada requisição |
| OT-6 | Reaproveitar integralmente os padrões do projeto, sem alterar o middleware de erro |

## 3. Escopo e exclusões

**Neste documento:** modelagem das tabelas, fluxos de outbox/worker/retry/DLQ, contratos HTTP,
matriz de erros, resiliência, observabilidade e a integração com o código existente.

**Fora deste documento:** notificação por e-mail, dashboard visual, rate limiting de saída e
arquivamento da outbox. Todos registrados como fora de escopo no [PRD](PRD.md#52-fora-de-escopo).

## 4. Modelo de dados

Três tabelas novas em `prisma/schema.prisma`, seguindo o padrão do arquivo: `String @id
@default(uuid()) @db.Char(36)`, como já usado em `User`, `Order` e `OrderStatusHistory`.

### 4.1 `webhook_endpoint`

Configuração cadastrada pelo cliente.

| Campo | Tipo | Nota |
| --- | --- | --- |
| `id` | `Char(36)` UUID | PK |
| `customerId` | `Char(36)` | FK para `Customer` |
| `url` | `VarChar(2048)` | **HTTPS obrigatório**, validado no schema |
| `secret` | `VarChar(255)` | secret ativa, gerada por nós |
| `previousSecret` | `VarChar(255)?` | válida durante a rotação |
| `previousSecretExpiresAt` | `DateTime?` | fim do grace period de 24h |
| `subscribedStatuses` | `Json` | lista de `OrderStatus` que este endpoint ouve |
| `active` | `Boolean` | permite desativar sem apagar |
| `createdAt` / `updatedAt` | `DateTime` | padrão do projeto |

### 4.2 `webhook_outbox`

Evento pendente de entrega, com **payload já renderizado** (ver [ADR-007](adrs/ADR-007-snapshot-do-payload-na-outbox.md)).

| Campo | Tipo | Nota |
| --- | --- | --- |
| `id` | `Char(36)` UUID | vira o `X-Event-Id` |
| `webhookEndpointId` | `Char(36)` | FK |
| `orderId` | `Char(36)` | usado para ordenação por pedido |
| `eventType` | `VarChar(64)` | ex.: `order.status_changed` |
| `payload` | `Json` | snapshot |
| `status` | `Enum` | `PENDING` · `PROCESSING` · `FAILED` · `DELIVERED` |
| `attempts` | `Int` | contador, teto 5 |
| `nextAttemptAt` | `DateTime` | quando o worker pode pegar de novo |
| `createdAt` | `DateTime` | **ordem de processamento** |

**Índices:** `(status, nextAttemptAt)` para a busca do worker e `(orderId, createdAt)` para
inspeção por pedido. Foi o ponto levantado em `[09:07]` Bruno e respondido em `[09:08]` Diego.

### 4.3 `webhook_delivery`

Histórico de tentativas, alimenta o `GET /webhooks/:id/deliveries`.

| Campo | Nota |
| --- | --- |
| `id`, `outboxId`, `webhookEndpointId` | identificação |
| `attempt` | número da tentativa |
| `requestPayload` | o que foi enviado |
| `responseStatus`, `responseBody` | o que voltou |
| `durationMs` | tempo de resposta |
| `error` | mensagem, quando houve |
| `createdAt` | quando ocorreu |

### 4.4 `webhook_dead_letter`

Evento que esgotou as tentativas ([ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md)). Guarda
`payload`, `reason`, `failedAt`, `webhookEndpointId`, `orderId` e `replayedAt` / `replayedById`
para auditoria do replay.

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox

```
changeStatus(id, input, userId)
└── prisma.$transaction(tx)
    ├── valida transição            (canTransition, já existe)
    ├── movimenta estoque           (debitStock / replenishStock, já existem)
    ├── tx.order.update             (já existe)
    ├── tx.orderStatusHistory.create(já existe)
    └── publishWebhookEvent(tx, order, from, to)      ◄── NOVO
        ├── busca endpoints ativos do customer que ouvem `to`
        ├── se nenhum: RETORNA sem gravar nada
        └── para cada endpoint: tx.webhookOutbox.create({ payload: snapshot })
```

Dois pontos decididos na reunião e refletidos acima:

- **O filtro acontece na inserção**, não no envio: *"Se nenhum webhook do customer quer aquele
  status, nem insere. Economiza linha na tabela."* (`[09:34]` Bruno)
- **A assinatura recebe o `tx`**, não um repositório: *"função pura recebendo o tx. Não precisa
  injetar repository inteiro."* (`[09:41]` Diego)

Se `publishWebhookEvent` lançar, a transação inteira sofre rollback e **o status não muda**. É a
garantia central da feature (`[09:40]` Bruno).

### 5.2 Processamento pelo worker

```
loop a cada 2s:
  candidatos = SELECT * FROM webhook_outbox
               WHERE status IN (PENDING, FAILED) AND nextAttemptAt <= now()
               ORDER BY createdAt ASC
               LIMIT <batch pequeno>

  para cada evento:
    marca PROCESSING
    resolve a secret ativa do endpoint
    assina  = HMAC-SHA256(payload, secret)
    POST url  (timeout 10s)  headers: X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id
    grava webhook_delivery (status, body, durationMs)
    2xx  -> marca DELIVERED
    erro -> attempts++ ; agenda nextAttemptAt pelo backoff ; marca FAILED
            se attempts == 5 -> move para webhook_dead_letter
```

O worker roda em `src/worker.ts` **(arquivo novo, a criar)**, iniciado por um script `worker` no
`package.json`, espelhando o
`dev`/`start` que já apontam para `src/server.ts`. Abre **`PrismaClient` próprio**, porque é outro
processo ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)).

### 5.3 Retry

| Tentativa | Espera | Acumulado |
| --- | --- | --- |
| 1 | 1 min | 1 min |
| 2 | 5 min | 6 min |
| 3 | 30 min | 36 min |
| 4 | 2 h | 2h36 |
| 5 | 12 h | **~14h36** |

`nextAttemptAt = now() + intervalo[attempts]`. O worker só considera eventos cujo `nextAttemptAt`
já passou, o que dispensa timer em memória e sobrevive a restart do processo.

### 5.4 DLQ e replay

Ao falhar a 5ª tentativa, o evento é copiado para `webhook_dead_letter` com o motivo, e sai da
outbox ativa. O replay é manual, por endpoint administrativo, e **recoloca o evento na outbox como
`PENDING` com `attempts` zerado**, registrando `replayedById` para auditoria (`[09:36]` Sofia).

## 6. Contratos públicos

Todos os endpoints exigem autenticação, reusando o `authenticate` de
`src/middlewares/auth.middleware.ts`.

### 6.1 `POST /webhooks` - cadastrar

**Request**

```json
{
  "customerId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "url": "https://atlas-comercial.example.com/hooks/oms",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"]
}
```

**Response `201 Created`** - a `secret` aparece **apenas aqui** (`[09:31]` Marcos)

```json
{
  "id": "1f6c2b90-6a5d-4f5e-9a1d-3b0f9d7c2e11",
  "customerId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "url": "https://atlas-comercial.example.com/hooks/oms",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"],
  "secret": "whsec_2f8a9c1d4e6b7a03f5c8d2e1b4a7096c",
  "active": true,
  "createdAt": "2026-09-01T12:00:00.000Z"
}
```

| Status | Quando |
| --- | --- |
| `201` | criado |
| `400` | body inválido (`VALIDATION_ERROR`) |
| `422` | URL não HTTPS (`WEBHOOK_INVALID_URL`) |
| `404` | `customerId` inexistente |
| `401` | sem autenticação |

### 6.2 `PATCH /webhooks/:id` - editar

**Request** (campos opcionais)

```json
{ "subscribedStatuses": ["PAID", "SHIPPED", "DELIVERED"], "active": false }
```

**Response `200 OK`**

```json
{
  "id": "1f6c2b90-6a5d-4f5e-9a1d-3b0f9d7c2e11",
  "url": "https://atlas-comercial.example.com/hooks/oms",
  "subscribedStatuses": ["PAID", "SHIPPED", "DELIVERED"],
  "active": false,
  "updatedAt": "2026-09-01T12:30:00.000Z"
}
```

| Status | Quando |
| --- | --- |
| `200` | atualizado |
| `404` | `WEBHOOK_NOT_FOUND` |
| `422` | `WEBHOOK_INVALID_URL` |

### 6.3 `GET /webhooks?customerId=...` - listar

**Request** - sem corpo; o filtro vai na query string

```http
GET /webhooks?customerId=7c9e6679-7425-40de-944b-e07fc1f90ae7 HTTP/1.1
Authorization: Bearer <jwt>
```

**Response `200 OK`** - a `secret` **nunca** é devolvida aqui

```json
{
  "data": [
    {
      "id": "1f6c2b90-6a5d-4f5e-9a1d-3b0f9d7c2e11",
      "url": "https://atlas-comercial.example.com/hooks/oms",
      "subscribedStatuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-09-01T12:00:00.000Z"
    }
  ]
}
```

| Status | Quando |
| --- | --- |
| `200` | lista devolvida, vazia inclusive |
| `400` | `customerId` ausente ou não é uuid (`VALIDATION_ERROR`) |
| `401` | sem autenticação |

### 6.4 `GET /webhooks/:id/deliveries` - histórico de entregas

Pedido explícito do produto em `[09:34]` Marcos: *"esses são os últimos 100 webhooks que vocês
mandaram pra mim, sucesso/falha, payload, response, tempo de resposta"*.

**Request** - sem corpo; o webhook vai no path

```http
GET /webhooks/1f6c2b90-6a5d-4f5e-9a1d-3b0f9d7c2e11/deliveries HTTP/1.1
Authorization: Bearer <jwt>
```

**Response `200 OK`**

```json
{
  "data": [
    {
      "id": "b3a1...",
      "eventId": "9d2f...",
      "attempt": 1,
      "responseStatus": 200,
      "responseBody": "{\"ok\":true}",
      "durationMs": 143,
      "error": null,
      "createdAt": "2026-09-01T12:05:02.412Z"
    },
    {
      "id": "c4b2...",
      "eventId": "7e1a...",
      "attempt": 3,
      "responseStatus": null,
      "responseBody": null,
      "durationMs": 10000,
      "error": "ETIMEDOUT",
      "createdAt": "2026-09-01T12:41:10.008Z"
    }
  ]
}
```

| Status | Quando |
| --- | --- |
| `200` | histórico devolvido, limitado aos 100 mais recentes |
| `404` | `WEBHOOK_NOT_FOUND` |
| `401` | sem autenticação |

### 6.5 `POST /webhooks/:id/rotate-secret` - rotacionar secret

**Request** - sem corpo; a operação é identificada pelo path

```http
POST /webhooks/1f6c2b90-6a5d-4f5e-9a1d-3b0f9d7c2e11/rotate-secret HTTP/1.1
Authorization: Bearer <jwt>
```

**Response `200 OK`**

```json
{
  "secret": "whsec_5b1d8e0a7c3f4926ad1e8b0c7f2a3d64",
  "previousSecretExpiresAt": "2026-09-02T12:00:00.000Z"
}
```

| Status | Quando |
| --- | --- |
| `200` | secret rotacionada; a nova aparece **apenas** nesta resposta |
| `404` | `WEBHOOK_NOT_FOUND` |
| `401` | sem autenticação |

A anterior continua aceita até `previousSecretExpiresAt` (`[09:21]` Sofia).

### 6.6 `POST /admin/webhooks/dead-letter/:id/replay` - replay (ADMIN)

Protegido por `requireRole('ADMIN')`, já exportado de `src/middlewares/auth.middleware.ts` e usado
em `src/modules/users/user.routes.ts`.

**Request** - sem corpo; o item da DLQ vai no path

```http
POST /admin/webhooks/dead-letter/e8f0a1b2-c3d4-4e5f-9a0b-1c2d3e4f5a6b/replay HTTP/1.1
Authorization: Bearer <jwt de um usuário ADMIN>
```

**Response `202 Accepted`**

```json
{ "outboxId": "e8f0...", "status": "PENDING", "replayedBy": "3a7c...", "replayedAt": "2026-09-01T13:00:00.000Z" }
```

| Status | Quando |
| --- | --- |
| `202` | recolocado na outbox |
| `403` | role diferente de ADMIN (`FORBIDDEN`) |
| `404` | `WEBHOOK_DEAD_LETTER_NOT_FOUND` |

### 6.7 Requisição enviada ao cliente

```http
POST /hooks/oms HTTP/1.1
Host: atlas-comercial.example.com
Content-Type: application/json
X-Event-Id: 9d2f1c34-7b8a-4e5d-9f01-2c3b4a5d6e7f
X-Webhook-Id: 1f6c2b90-6a5d-4f5e-9a1d-3b0f9d7c2e11
X-Signature: sha256=7f3a...c91d
X-Timestamp: 2026-09-01T12:05:02.269Z
```

```json
{
  "event_id": "9d2f1c34-7b8a-4e5d-9f01-2c3b4a5d6e7f",
  "event_type": "order.status_changed",
  "timestamp": "2026-09-01T12:05:02.269Z",
  "order_id": "5c1e...",
  "order_number": "ORD-2026-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "total_cents": 148900
}
```

Os itens do pedido **não** vão no corpo; o cliente que precisar do detalhe usa `GET /orders/:id`
(`[09:43]` Diego).

## 7. Matriz de erros

Todos herdam de `AppError`, definido em `src/shared/errors/app-error.ts`, e são tratados sem
alteração pelo `src/middlewares/error.middleware.ts` (`[09:29]` Bruno).

| Código | HTTP | Quando |
| --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | endpoint inexistente |
| `WEBHOOK_INVALID_URL` | 422 | URL não é HTTPS |
| `WEBHOOK_SECRET_REQUIRED` | 422 | operação exige secret ausente |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | payload acima de 64KB |
| `WEBHOOK_INVALID_STATUS_FILTER` | 422 | status fora do enum `OrderStatus` |
| `WEBHOOK_ENDPOINT_INACTIVE` | 409 | tentativa de entrega em endpoint desativado |
| `WEBHOOK_DELIVERY_TIMEOUT` | interno | timeout de 10s; vira falha e retry |
| `WEBHOOK_MAX_ATTEMPTS_REACHED` | interno | 5ª falha; move para DLQ |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | replay de item inexistente |
| `WEBHOOK_REPLAY_FORBIDDEN` | 403 | replay sem role ADMIN |

O prefixo `WEBHOOK_` foi decidido em `[09:29]` Larissa e segue o padrão de `INSUFFICIENT_STOCK` e
`INVALID_STATUS_TRANSITION`, que já existem em `src/shared/errors/http-errors.ts`.

## 8. Estratégias de resiliência

| Mecanismo | Valor | Origem |
| --- | --- | --- |
| **Timeout** da chamada ao cliente | 10 s | `[09:42]` Diego |
| **Retry** | 5 tentativas, 1m/5m/30m/2h/12h | `[09:17]` Diego |
| **Backoff persistido** | em `nextAttemptAt`, não em memória | sobrevive a restart do worker |
| **DLQ** | tabela separada | `[09:18]` Diego |
| **Fallback** | replay manual por ADMIN | `[09:18]` Diego |
| **Batch pequeno** | evita monopolizar o worker | `[09:08]` Diego |
| **Teto de payload** | 64 KB, com falha explícita | `[09:24]` Diego |
| **Atomicidade** | inserção dentro da transação | `[09:40]` Bruno |

Não há circuit breaker por endpoint nesta fase. O backoff crescente já espaça naturalmente as
chamadas a um cliente instável.

## 9. Observabilidade

### Logs

**Pino**, já configurado em `src/shared/logger` e usado como `pino-http` em
`src/middlewares/request-logger.middleware.ts`. Nada novo entra (`[09:29]` Bruno).

O worker loga por evento, com campos estruturados: `event_id`, `webhook_endpoint_id`, `order_id`,
`attempt`, `response_status`, `duration_ms`, `outcome`. **A secret nunca é logada**, nem inteira
nem parcial.

### Métricas

| Métrica | Tipo | Para quê |
| --- | --- | --- |
| `webhook_outbox_pending` | gauge | tamanho da fila; se cresce, o worker não vence |
| `webhook_delivery_latency_seconds` | histograma | valida a meta de **p95 < 10s** do PRD |
| `webhook_delivery_total{outcome}` | contador | sucesso, falha e timeout |
| `webhook_attempts_total{attempt}` | contador | distribuição das tentativas |
| `webhook_dead_letter_total` | contador | quantos desistimos de entregar |
| `webhook_worker_loop_duration_seconds` | histograma | saúde do ciclo de 2s |

### Tracing

Propagar o `X-Event-Id` como identificador de correlação entre a transação que originou o evento,
o ciclo do worker e a chamada HTTP ao cliente. Assim uma única busca por `event_id` reconstrói o
caminho completo, de `changeStatus` até a resposta do cliente, atravessando os dois processos.

## 10. Dependências e compatibilidade

### 10.1 Dependências de runtime

A feature **não acrescenta nenhuma dependência nova**. Tudo o que ela precisa já está no
`package.json` do projeto, o que é consequência direta da decisão de ficar dentro dos padrões
existentes ([ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)).

| O que a feature usa | De onde vem | Já no projeto |
| --- | --- | --- |
| Persistência e transação da outbox | `@prisma/client` 5.22.0 | sim |
| HMAC-SHA256 da assinatura | `node:crypto` (biblioteca padrão) | sim, não é dependência |
| `X-Event-Id` por evento | `uuid` 11.0.3 | sim |
| Validação dos payloads do CRUD | `zod` 3.23.8 | sim |
| Rotas do módulo de webhooks | `express` 4.21.1 | sim |
| Logs estruturados do worker | `pino` 9.5.0 | sim |
| Testes do módulo e do worker | `vitest`, `supertest` | sim |

A entrega HTTP ao cliente usa o **`fetch` global do Node**, disponível a partir do Node 18, e não
um cliente HTTP novo. O `package.json` declara `"node": ">=20"`, então o piso já está garantido.

Nenhuma infraestrutura nova: sem Redis, sem broker, sem serviço adicional para operar. Essa é a
propriedade que a proposta compra ao usar outbox no MySQL existente
([ADR-001](adrs/ADR-001-outbox-no-mysql.md)).

### 10.2 Compatibilidade com o banco

As quatro tabelas da feature são **aditivas**: nenhuma coluna de tabela existente é alterada ou
removida, e nenhuma constraint atual é tocada. A migration é, portanto, compatível com a versão
anterior da aplicação: um deploy que rode a migration antes de subir o código novo não quebra o
código antigo, porque o código antigo simplesmente não conhece as tabelas.

O provider é `mysql`, conforme `prisma/schema.prisma`. A escolha de polling em vez de notificação
do banco ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)) é consequência direta disso: o
MySQL não tem `LISTEN`/`NOTIFY`.

### 10.3 Compatibilidade da API

Todos os endpoints do módulo são **novos**. Nenhuma rota existente muda de contrato, de status code
ou de formato de erro, então nenhum consumidor atual precisa ser alterado.

O único ponto de contato com código existente é `changeStatus`, e ele muda de forma aditiva: uma
chamada a mais dentro da transação que já existe. A assinatura pública do método não muda.

O envelope de erro dos endpoints novos é exatamente o que o
`src/middlewares/error.middleware.ts` já produz para qualquer `AppError`, sem nenhuma adaptação:

```json
{ "error": { "code": "WEBHOOK_NOT_FOUND", "message": "Webhook not found" } }
```

O campo opcional `details` continua sendo preenchido só quando a exceção o traz, como acontece hoje
com os erros de validação do Zod. Os códigos `WEBHOOK_*` convivem com os atuais sem colisão de
prefixo.

### 10.4 Ordem de deploy

O worker é um processo separado ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)), o que
introduz a única restrição de ordem da feature:

1. **migration** das quatro tabelas;
2. **API** com a gravação na outbox ativa;
3. **worker**.

Subir o worker antes da migration faz ele falhar contra tabela inexistente. Subir a API antes do
worker é seguro por construção: os eventos se acumulam na outbox e são entregues quando o worker
aparecer, com o atraso correspondente. Essa é a mesma propriedade que sustenta o *at-least-once*
([ADR-005](adrs/ADR-005-at-least-once-com-event-id.md)).

Para o rollback vale a ordem inversa, e as tabelas podem ficar: são aditivas e inertes sem o
worker.

### 10.5 Compatibilidade do lado do cliente

O contrato com o cliente é a requisição descrita em 6.7. Ele é versionado pelo campo `event` do
payload, e o teto de **64KB** por corpo é uma restrição declarada, não um limite acidental: um
pedido com muitos itens que ultrapasse o teto falha de forma explícita em vez de ser truncado.

A rotação de secret mantém a secret anterior válida por 24h
([ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)), o que é justamente o que permite ao
cliente trocar a chave sem janela de indisponibilidade.

## 11. Integração com o sistema existente

> Seção obrigatória.
>
> **Legenda:** os caminhos desta seção **existem hoje** no repositório e foram verificados em
> disco. Onde a feature acrescenta arquivo novo, ele vem marcado como **(a criar)** - é o caso de
> `src/worker.ts` e de todo o diretório `src/modules/webhooks/`, que ainda não existem.

### 11.1 `src/modules/orders/order.service.ts`

**O ponto crítico.** O método `changeStatus` já roda tudo dentro de `this.prisma.$transaction(async (tx) => {...})`:
valida a transição com `canTransition`, chama `debitStock` / `replenishStock`, executa
`tx.order.update` e `tx.orderStatusHistory.create`.

A alteração é **uma chamada nova ao fim desse bloco**:

```ts
await publishWebhookEvent(tx, order, from, to);
```

A função recebe o `tx` da transação corrente, e não um repositório injetado (`[09:41]` Diego).
Nenhuma outra parte do método muda.

### 11.2 `src/shared/errors/`

`app-error.ts` define `AppError` e `http-errors.ts` traz os erros de HTTP com códigos como
`INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`, `VALIDATION_ERROR` e `FORBIDDEN`. Os erros de
webhook estendem as mesmas classes e adotam o prefixo `WEBHOOK_`, sem inventar hierarquia nova.

### 11.3 `src/middlewares/error.middleware.ts`

Trata `AppError`, `ZodError` e erros do Prisma de forma centralizada. Como os erros novos herdam de
`AppError`, **este arquivo não precisa de nenhuma alteração** para responder corretamente aos
endpoints de webhook.

### 11.4 `src/middlewares/auth.middleware.ts`

Exporta `authenticate` e `requireRole`. O CRUD de webhook usa `authenticate`; o endpoint de replay
usa `requireRole('ADMIN')`, exatamente como `src/modules/users/user.routes.ts` já faz na linha em
que monta a rota protegida.

### 11.5 `prisma/schema.prisma`

Já contém `User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory` e o enum
`OrderStatus` com `PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED` e `CANCELLED`. As quatro
tabelas novas entram no mesmo arquivo, reusando esse enum no filtro de status e seguindo o padrão
de `@id @default(uuid()) @db.Char(36)`.

### 11.6 `src/middlewares/validate.middleware.ts`

Recebe schemas Zod para `body`, `query` e `params` e converte `ZodError` em `ValidationError`. Os
schemas do módulo de webhook usam esse mesmo middleware, e é nele que a exigência de HTTPS e o teto
de 64KB são aplicados, por serem validação e não regra de arquitetura (`[09:23]` Sofia).

### 11.7 `src/server.ts` e `package.json`

`src/server.ts` é o entry-point da API, iniciado pelos scripts `dev` e `start`. O worker entra como
**entry-point irmão** em `src/worker.ts` **(a criar)**, com um script `worker` no mesmo padrão. A intenção foi
declarada em `[09:11]` Larissa: *"Tipo o que a gente já tem em src/server.ts, criar um
src/worker.ts e um script npm run worker"*.

### 11.8 Estrutura do módulo

`src/modules/webhooks/` segue a anatomia dos módulos existentes, visível em
`src/modules/orders/`: `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`,
`webhook.routes.ts` e `webhook.schemas.ts`, mais `webhook.processor.ts` com a lógica que o worker
executa (`[09:28]` Bruno).

## 12. Critérios de aceite técnicos

| # | Critério |
| --- | --- |
| **CAT-01** | Alterar status grava N linhas na outbox, uma por endpoint ativo que ouve o status |
| **CAT-02** | Erro forçado em `publishWebhookEvent` provoca rollback, e o pedido mantém o status anterior |
| **CAT-03** | Nenhum endpoint interessado resulta em **zero** linhas na outbox |
| **CAT-04** | O worker processa em ordem de `createdAt` e respeita `nextAttemptAt` |
| **CAT-05** | Assinatura conferida pelo receptor bate com HMAC-SHA256 do corpo |
| **CAT-06** | Endpoint que responde 500 nas 5 tentativas termina em `webhook_dead_letter` |
| **CAT-07** | Endpoint que demora mais de 10s conta como falha, com `durationMs` ≈ 10000 |
| **CAT-08** | Após rotação, requisições assinadas com a secret antiga são aceitas dentro das 24h |
| **CAT-09** | `POST /webhooks` com `http://` responde 422 com `WEBHOOK_INVALID_URL` |
| **CAT-10** | Replay sem role ADMIN responde 403 |
| **CAT-11** | Reiniciar o worker no meio do backoff **não** perde nem antecipa tentativas |
| **CAT-12** | Nenhum log contém a secret |

## 13. Riscos técnicos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Evento preso em `PROCESSING` após crash do worker | Média | Entrega nunca acontece | Considerar `PROCESSING` mais velho que N minutos como elegível de novo |
| Batch grande monopoliza o ciclo | Baixa | Latência sobe para os demais | Batch pequeno e limite explícito na query |
| Índice ausente degrada a busca | Baixa | Worker fica lento com a tabela cheia | Índice `(status, nextAttemptAt)` desde a primeira migration |
| Renderização do payload com bug fica congelada | Baixa | Evento errado é reenviado no replay | Testes de snapshot do payload; correção exige regravar pendentes |
| Duas secrets válidas na janela de rotação | Média | Confusão na verificação | Ordem explícita: tenta a ativa, depois a anterior se ainda vigente |
