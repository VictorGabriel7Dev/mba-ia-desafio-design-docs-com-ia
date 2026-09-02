# Tracker de Rastreabilidade

Cada linha liga um item dos documentos à sua origem: um trecho da transcrição ou um arquivo do
código. **Item sem origem identificável foi removido dos documentos**, não relaxado aqui. É a
função do tracker: se a coluna `Localização` não fecha, aquilo provavelmente foi inventado.

**Cobertura:** 89 itens · **75 com origem em `TRANSCRICAO`** (84%) · **14 com origem em `CODIGO`**.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Contexto | Três clientes B2B (Atlas, MaxDistribuição, Nova Cargo) pediram notificação formalmente | TRANSCRICAO | `[09:00] Marcos` |
| PRD-CTX-02 | docs/PRD.md | Contexto | Hoje eles fazem polling no `GET /orders`, o que é lento e caro | TRANSCRICAO | `[09:00] Marcos` |
| PRD-CTX-03 | docs/PRD.md | Restrição | Risco comercial: Atlas pode migrar para o concorrente se não sair até o fim do trimestre | TRANSCRICAO | `[09:00] Marcos` |
| PRD-OBJ-01 | docs/PRD.md | Requisito Não Funcional | "Tempo real" definido pelo cliente como qualquer coisa abaixo de 10 segundos | TRANSCRICAO | `[09:02] Marcos` |
| PRD-ESC-01 | docs/PRD.md | Restrição | Escopo é outbound apenas; o cliente recebe, não envia | TRANSCRICAO | `[09:02] Marcos` e `[09:03] Sofia` |
| PRD-FE-01 | docs/PRD.md | Fora de escopo | Notificação por e-mail ao cliente após falhas fica para a próxima fase | TRANSCRICAO | `[09:37] Larissa` |
| PRD-FE-02 | docs/PRD.md | Fora de escopo | Dashboard visual é projeto separado do time de frontend | TRANSCRICAO | `[09:40] Larissa` |
| PRD-FE-03 | docs/PRD.md | Fora de escopo | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | `[09:39] Diego` e `[09:39] Larissa` |
| PRD-FE-04 | docs/PRD.md | Fora de escopo | Arquivamento das linhas entregues (~30 dias) fica fora desta feature | TRANSCRICAO | `[09:08] Diego` |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook com url, lista de status e secret gerada por nós | TRANSCRICAO | `[09:31] Marcos` |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | PATCH para editar o webhook | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | DELETE para remover o webhook | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | GET para listar os webhooks de um customer | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de eventos por status, aplicado na inserção da outbox | TRANSCRICAO | `[09:33] Marcos` e `[09:34] Bruno` |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Gravar o evento na outbox dentro da mesma transação da mudança de status | TRANSCRICAO | `[09:06] Diego` |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Entregar por POST HTTPS com payload assinado | TRANSCRICAO | `[09:20] Sofia` |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Retry em 5 tentativas com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | `[09:17] Diego` |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | DLQ em tabela separada com payload, motivo e timestamp | TRANSCRICAO | `[09:18] Diego` |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Histórico de entregas: sucesso/falha, payload, response, tempo de resposta | TRANSCRICAO | `[09:34] Marcos` |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace period de 24h | TRANSCRICAO | `[09:21] Sofia` |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Replay de DLQ por endpoint admin | TRANSCRICAO | `[09:18] Diego` e `[09:35] Diego` |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência abaixo de 10s; piso de 2s pelo polling | TRANSCRICAO | `[09:09] Diego` e `[09:10] Larissa` |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s na chamada ao cliente | TRANSCRICAO | `[09:42] Diego` |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Teto de 64KB no payload, com erro e não truncamento | TRANSCRICAO | `[09:23] Sofia` e `[09:24] Diego` |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | URL obrigatoriamente HTTPS | TRANSCRICAO | `[09:23] Sofia` |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Garantia at-least-once | TRANSCRICAO | `[09:24] Diego` |
| PRD-NFR-06 | docs/PRD.md | Restrição | Ordenação só por order_id e só enquanto single-worker | TRANSCRICAO | `[09:12] Diego` e `[09:13] Larissa` |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado da API | TRANSCRICAO | `[09:11] Diego` |
| PRD-NFR-08 | docs/PRD.md | Restrição | Identificadores em UUID, seguindo o projeto | TRANSCRICAO | `[09:51] Larissa` |
| PRD-NFR-09 | docs/PRD.md | Restrição | Sem infraestrutura nova (nem Redis, nem broker) | TRANSCRICAO | `[09:07] Diego` |
| PRD-RSK-01 | docs/PRD.md | Risco | Vazamento de secret já aconteceu com cliente, em log de aplicação | TRANSCRICAO | `[09:22] Diego` |
| PRD-RSK-02 | docs/PRD.md | Risco | Acúmulo de eventos na outbox pode deixar o worker lento | TRANSCRICAO | `[09:07] Bruno` |
| PRD-RSK-03 | docs/PRD.md | Risco | Prazo de fim de novembro contra estimativa de 3 sprints | TRANSCRICAO | `[09:45] Marcos` e `[09:46] Larissa` |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança da Sofia, 2 dias úteis antes do deploy | TRANSCRICAO | `[09:46] Sofia` |
| PRD-TST-01 | docs/PRD.md | Decisão | Estimativa de 3 sprints cobre modelagem, worker, CRUD, integração e testes | TRANSCRICAO | `[09:46] Larissa` |
| RFC-CTX-01 | docs/RFC.md | Restrição | A transação de mudança de status já é pesada: order, history e estoque | TRANSCRICAO | `[09:04] Bruno` |
| RFC-CTX-02 | docs/RFC.md | Restrição | Não dá para dar rollback na mudança de status porque o cliente caiu | TRANSCRICAO | `[09:04] Bruno` |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Síncrono descartado: cliente lento travaria mudança de status de outros pedidos | TRANSCRICAO | `[09:04] Bruno` e `[09:06] Diego` |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Redis Streams descartado: subir infra nova é overengineering para time pequeno | TRANSCRICAO | `[09:07] Diego` |
| RFC-ALT-03 | docs/RFC.md | Trade-off | Trigger de banco descartado: MySQL não tem NOTIFY/LISTEN e não avisa processo externo | TRANSCRICAO | `[09:09] Diego` |
| RFC-ALT-04 | docs/RFC.md | Trade-off | 3 tentativas descartado: esgotaria em 30 min, e já houve cliente com 2h fora | TRANSCRICAO | `[09:16] Diego` |
| RFC-ALT-05 | docs/RFC.md | Trade-off | Retry indefinido descartado: evento ficaria pendurado para sempre | TRANSCRICAO | `[09:15] Diego` |
| RFC-ALT-06 | docs/RFC.md | Trade-off | DLQ como flag na outbox descartado: poluiria a leitura da tabela principal | TRANSCRICAO | `[09:18] Diego` |
| RFC-ALT-07 | docs/RFC.md | Trade-off | Renderizar payload no envio descartado: evento deixaria de refletir o momento da mudança | TRANSCRICAO | `[09:52] Larissa` |
| RFC-Q-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída: 50 pedidos num minuto viram 50 chamadas? | TRANSCRICAO | `[09:38] Diego` |
| RFC-Q-02 | docs/RFC.md | Questão em aberto | Ordenação com múltiplos workers: particionar por order_id ou lock pessimista | TRANSCRICAO | `[09:13] Diego` |
| RFC-Q-03 | docs/RFC.md | Questão em aberto | E-mail ao cliente após falhas: adiado para a próxima fase | TRANSCRICAO | `[09:37] Marcos` e `[09:37] Larissa` |
| RFC-Q-04 | docs/RFC.md | Questão em aberto | Arquivamento da outbox, reconhecido e fora de escopo | TRANSCRICAO | `[09:08] Diego` |
| RFC-Q-05 | docs/RFC.md | Questão em aberto | Endurecer autorização do CRUD mais para frente | TRANSCRICAO | `[09:37] Sofia` |
| RFC-IMP-01 | docs/RFC.md | Decisão | Integração via função `publishWebhookEvent(tx, ...)`, sem injetar repositório | TRANSCRICAO | `[09:41] Bruno` e `[09:41] Diego` |
| FDD-MOD-01 | docs/FDD.md | Decisão | Outbox com índice em status e em created_at, leitura em batch pequeno | TRANSCRICAO | `[09:08] Diego` |
| FDD-MOD-02 | docs/FDD.md | Decisão | Configuração guarda url, secret, customer_id e estado ativo | TRANSCRICAO | `[09:21] Bruno` e `[09:21] Sofia` |
| FDD-MOD-03 | docs/FDD.md | Decisão | Payload gravado como snapshot no momento da inserção | TRANSCRICAO | `[09:52] Larissa` e `[09:52] Diego` |
| FDD-FLX-01 | docs/FDD.md | Decisão | Worker em polling de 2s, lendo os pendentes mais antigos | TRANSCRICAO | `[09:09] Diego` |
| FDD-FLX-02 | docs/FDD.md | Decisão | Filtro por status aplicado na inserção, não no envio | TRANSCRICAO | `[09:34] Bruno` |
| FDD-CONTRATO-01 | docs/FDD.md | Requisito Funcional | POST /webhooks devolve a secret apenas na criação | TRANSCRICAO | `[09:31] Marcos` |
| FDD-CONTRATO-02 | docs/FDD.md | Requisito Funcional | GET /webhooks/:id/deliveries com sucesso, payload, response e duração | TRANSCRICAO | `[09:34] Marcos` |
| FDD-CONTRATO-03 | docs/FDD.md | Requisito Funcional | POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | `[09:35] Diego` |
| FDD-CONTRATO-04 | docs/FDD.md | Restrição | Replay exige role ADMIN e registra quem executou, para auditoria | TRANSCRICAO | `[09:36] Sofia` |
| FDD-CONTRATO-05 | docs/FDD.md | Decisão | customer_id vem do body ou do path, não do JWT | TRANSCRICAO | `[09:32] Larissa` |
| FDD-CONTRATO-06 | docs/FDD.md | Decisão | CRUD de configuração aberto a qualquer role autenticada, por enquanto | TRANSCRICAO | `[09:37] Sofia` |
| FDD-PAYLOAD-01 | docs/FDD.md | Decisão | Campos do payload: event_id, event_type, timestamp, order_id, order_number, from/to_status, customer_id, total_cents | TRANSCRICAO | `[09:43] Diego` |
| FDD-PAYLOAD-02 | docs/FDD.md | Decisão | Itens do pedido ficam fora do payload; detalhe via GET /orders/:id | TRANSCRICAO | `[09:43] Diego` |
| FDD-HDR-01 | docs/FDD.md | Decisão | Headers X-Event-Id, X-Signature, X-Timestamp e Content-Type | TRANSCRICAO | `[09:44] Diego` |
| FDD-HDR-02 | docs/FDD.md | Decisão | X-Webhook-Id acrescentado para o cliente saber qual cadastro originou o envio | TRANSCRICAO | `[09:44] Sofia` |
| FDD-ERR-01 | docs/FDD.md | Restrição | Códigos de erro com prefixo WEBHOOK_ | TRANSCRICAO | `[09:29] Larissa` |
| FDD-RES-01 | docs/FDD.md | Decisão | Timeout de 10s por chamada ao cliente | TRANSCRICAO | `[09:42] Diego` |
| FDD-OBS-01 | docs/FDD.md | Decisão | Logging com Pino, sem introduzir biblioteca nova | TRANSCRICAO | `[09:29] Bruno` |
| FDD-INT-01 | docs/FDD.md | Decisão | `changeStatus` roda em `prisma.$transaction` com update, history e estoque | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-02 | docs/FDD.md | Decisão | Reuso de `AppError` e dos erros de HTTP existentes | CODIGO | `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` |
| FDD-INT-03 | docs/FDD.md | Decisão | Middleware de erro central não precisa de alteração | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-INT-04 | docs/FDD.md | Decisão | `requireRole('ADMIN')` reaproveitado no endpoint de replay | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-INT-05 | docs/FDD.md | Decisão | Precedente de uso de `requireRole` já existente no projeto | CODIGO | `src/modules/users/user.routes.ts` |
| FDD-INT-06 | docs/FDD.md | Restrição | Enum `OrderStatus` e padrão `@id @default(uuid()) @db.Char(36)` | CODIGO | `prisma/schema.prisma` |
| FDD-INT-07 | docs/FDD.md | Decisão | Validação Zod pelo middleware existente, que converte ZodError em ValidationError | CODIGO | `src/middlewares/validate.middleware.ts` |
| FDD-INT-08 | docs/FDD.md | Decisão | Worker como entry-point irmão do servidor | CODIGO | `src/server.ts` |
| FDD-INT-09 | docs/FDD.md | Decisão | Anatomia do módulo espelha o módulo de pedidos | CODIGO | `src/modules/orders/order.routes.ts` e `src/modules/orders/order.schemas.ts` |
| FDD-INT-10 | docs/FDD.md | Decisão | Pino já instalado como `pino-http` no logger de requisição | CODIGO | `src/middlewares/request-logger.middleware.ts` |
| FDD-DEP-01 | docs/FDD.md | Restrição | A feature não acrescenta nenhuma dependência nova; tudo já está no manifesto | CODIGO | `package.json` |
| FDD-DEP-02 | docs/FDD.md | Restrição | Provider do banco é `mysql`, o que exclui `LISTEN`/`NOTIFY` e força o polling | CODIGO | `prisma/schema.prisma` |
| FDD-DEP-03 | docs/FDD.md | Decisão | Entrega HTTP usa o `fetch` global do Node; o manifesto exige `node >=20` | CODIGO | `package.json` |
| FDD-DEP-04 | docs/FDD.md | Restrição | Envelope de erro dos endpoints novos é o que o middleware já produz, sem adaptação | CODIGO | `src/middlewares/error.middleware.ts` |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão outbox no MySQL existente, na mesma transação | TRANSCRICAO | `[09:06] Diego` |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker em processo separado, polling de 2s | TRANSCRICAO | `[09:09] Diego` e `[09:11] Diego` |
| ADR-003 | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | 5 tentativas, backoff 1m/5m/30m/2h/12h, DLQ em tabela separada | TRANSCRICAO | `[09:17] Diego` e `[09:18] Diego` |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256, secret por endpoint, rotação com grace period de 24h | TRANSCRICAO | `[09:20] Sofia` e `[09:21] Sofia` |
| ADR-005 | docs/adrs/ADR-005-at-least-once-com-event-id.md | Decisão | At-least-once com X-Event-Id para deduplicação no cliente | TRANSCRICAO | `[09:25] Diego` |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Reuso de AppError, Pino, middleware de erro, módulos e schemas Zod | TRANSCRICAO | `[09:30] Larissa` |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-na-outbox.md | Decisão | Snapshot do payload na inserção | TRANSCRICAO | `[09:52] Larissa` |

---

## Itens verificados e descartados

Registro do que **não** entrou, para deixar explícito que a ausência foi decisão e não esquecimento.

| Item | Por que ficou de fora |
| --- | --- |
| Notificação por e-mail ao cliente | Adiado na reunião (`[09:37] Larissa`). Aparece apenas como "fora de escopo", nunca como requisito |
| Dashboard visual | Fora de escopo (`[09:40] Larissa`), citado apenas na seção de exclusões |
| Rate limiting de saída | Questão em aberto (`[09:39]`), registrada no RFC e **não** convertida em requisito |
| Circuit breaker por endpoint | **Não discutido na reunião.** Mencionado no FDD apenas para declarar sua ausência, sem virar decisão |
| Ordem de deploy (migration, API, worker) | **Não discutida na reunião.** É consequência direta do worker ser processo separado (ADR-002); aparece no FDD 10.4 como inferência declarada, sem linha de tracker, porque não tem origem na transcrição nem no código |
| mTLS | **Não discutido.** Aparece no ADR-004 explicitamente rotulado como alternativa plausível não levantada, para não passar por decisão da equipe |
| Arquivamento da outbox | Reconhecido (`[09:08] Diego`) e mantido fora do escopo desta feature |
