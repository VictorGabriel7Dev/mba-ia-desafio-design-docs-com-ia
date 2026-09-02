# RFC - Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Autor** | Larissa (Tech Lead) |
| **Status** | Aprovado (consenso ao fim da reunião técnica) |
| **Data** | Quinta-feira, reunião das 09:00 às 09:53 |
| **Revisores** | Diego (Eng. Sênior, Plataforma) · Bruno (Eng. Pleno, Pedidos) · Sofia (Eng. Segurança) · Marcos (Product Manager) |
| **Relacionados** | [PRD](PRD.md) · [FDD](FDD.md) · [ADRs](adrs/) |

---

## 1. TL;DR

Entregar notificação de mudança de status de pedido via **webhook outbound**, usando **padrão
outbox no MySQL que já temos**: a gravação do evento entra na **mesma transação** que altera o
pedido, e um **worker em processo separado** faz polling a cada 2 segundos para entregar.

Falhas são retentadas 5 vezes com backoff crescente até ~15h; o que sobra vai para uma **DLQ**
persistida. Cada payload é assinado com **HMAC-SHA256** usando secret **exclusiva daquele
endpoint**, rotacionável com 24h de convivência.

**Nenhuma infraestrutura nova.** Sem Redis, sem broker, sem serviço adicional para operar.

## 2. Contexto e problema

Três clientes B2B pediram formalmente notificação em tempo real. Hoje fazem **polling no
`GET /orders`**, o que é lento e caro para eles. A Atlas sinalizou risco de migrar para o
concorrente se não entregarmos até o fim do trimestre.

A restrição que molda a solução apareceu logo no início: a transação de mudança de status **já é
pesada** (atualiza o pedido, insere no histórico, movimenta estoque). Acrescentar uma chamada HTTP
no meio dela faz **um cliente lento travar a mudança de status de outros pedidos**. E se o cliente
estiver fora do ar, não há resposta boa: dar rollback numa mudança de status legítima porque um
terceiro caiu é inaceitável.

Isso elimina a abordagem síncrona antes de qualquer discussão de tecnologia.

## 3. Proposta técnica

### 3.1 Visão geral

```
changeStatus (transação)                      worker (processo separado)
┌───────────────────────────────┐            ┌──────────────────────────┐
│ UPDATE order                  │            │ a cada 2s:               │
│ INSERT order_status_history   │            │   SELECT pendentes       │
│ movimenta estoque             │   mesma    │   (batch pequeno,        │
│ INSERT webhook_outbox ◄───────┼───base────►│    ordem de created_at)  │
│   1 linha por webhook          │            │   POST HTTPS ao cliente  │
│   interessado no status       │            │   marca entregue/falhou  │
└───────────────────────────────┘            └─────────┬────────────────┘
       commit ou rollback juntos                       │ esgotou 5 tentativas
                                                       ▼
                                             webhook_dead_letter
```

**Quatro peças novas, todas dentro do padrão que o projeto já usa:**

| Peça | Papel |
| --- | --- |
| `webhook_outbox` | evento pendente de entrega, com payload **já renderizado** |
| `webhook_dead_letter` | evento que esgotou as tentativas, com motivo e payload |
| `src/modules/webhooks` *(a criar)* | CRUD de configuração, histórico de entregas, replay |
| `src/worker.ts` *(a criar)* | entry-point separado, iniciado por `npm run worker` |

### 3.2 O que garante a consistência

A gravação do evento acontece **dentro da transação existente**. Se o commit passa, o evento está
registrado; se dá rollback, o evento desaparece junto. Não existe estado intermediário em que o
status mudou e a notificação se perdeu.

Essa é a razão de escolher outbox em vez de publicar numa fila: **publicar numa fila externa não
participa da transação do banco**.

### 3.3 Filtro na origem

Cada configuração declara **quais status quer ouvir**. O filtro é aplicado **na inserção**: se
nenhum webhook daquele cliente ouve o status em questão, nenhuma linha é criada. Evita encher a
outbox com evento que ninguém consome.

### 3.4 Segurança

- **HMAC-SHA256** sobre o corpo do request, no header `X-Signature`
- Secret **por endpoint**, nunca global: *"se vaza uma, vaza tudo"*
- **Rotação** com a secret anterior aceita por 24h, para o cliente migrar sem downtime
- **HTTPS obrigatório** na URL cadastrada
- Teto de **64KB** por payload, com falha explícita acima disso

### 3.5 Semântica de entrega

Garantimos **at-least-once**. O cliente pode receber o mesmo evento mais de uma vez e deduplica
pelo `X-Event-Id`, um UUID gerado quando o evento entra na outbox.

## 4. Alternativas consideradas

### 4.1 Disparo síncrono dentro do serviço de pedidos

**Descartada.** A chamada HTTP entraria na transação, que já é pesada. Um cliente lento passaria a
travar a mudança de status de outros pedidos, e a indisponibilidade de um terceiro forçaria a
pergunta sem resposta boa: *"dá rollback na mudança de status?"*

> **Trade-off recusado:** simplicidade de implementação ao custo de acoplar a disponibilidade do
> nosso fluxo de pedidos à disponibilidade do cliente.

### 4.2 Redis Streams (ou broker dedicado)

**Descartada.** Resolveria o desacoplamento, mas exige **subir e operar infraestrutura nova** para
um time pequeno. *"Subir Redis Cluster pra isso é overengineering. Outbox no MySQL existente
resolve."*

> **Trade-off recusado:** escala e push nativo ao custo de mais um componente para operar,
> monitorar e manter disponível.

### 4.3 Trigger de banco para acordar o worker

**Descartada por limitação técnica.** MySQL não tem `NOTIFY`/`LISTEN` como o Postgres. Trigger
executa SQL, mas **não notifica processo externo**. Avisar o worker exigiria improviso (escrever
arquivo, bater num endpoint), e o polling de 2s atende "abaixo de 10 segundos" com folga.

### 4.4 Retry com 3 tentativas

**Descartada por agressiva.** Esgotaria em ~30 minutos. Já houve cliente com **duas horas** de
indisponibilidade em manutenção planejada; o evento morreria antes de ele voltar.

### 4.5 Retry indefinido com backoff

**Descartada pelo extremo oposto.** Sem teto, cliente que sumiu deixa evento pendurado para sempre
e a outbox cresce sem limite.

### 4.6 DLQ como flag na própria outbox

**Descartada.** Marcar `failed` na outbox principal polui a leitura do worker. Tabela separada
mantém a outbox enxuta e serve de evidência para debug e reprocessamento.

### 4.7 Guardar só o `order_id` e renderizar no envio

**Descartada.** Se o pedido mudar entre a inserção e o envio, o cliente receberia evento que não
corresponde ao momento da mudança. Escolhido **snapshot na inserção**.

## 5. Questões em aberto

| # | Questão | Situação |
| --- | --- | --- |
| **Q1** | **Rate limiting de saída.** Cliente com 50 pedidos mudando em um minuto recebe 50 chamadas? | **Em aberto por decisão.** *"A gente observa e implementa se virar problema."* Registrado para acompanhar |
| **Q2** | **Ordenação com múltiplos workers.** Garantia existe só por `order_id` e só com um worker. | **Adiado.** Saídas conhecidas: particionar por `order_id` ou lock pessimista. *"Problema do futuro."* Os clientes nunca pediram ordering global |
| **Q3** | **Avisar o cliente quando o webhook dele falha** (e-mail após N falhas). | **Adiado para a próxima fase**, depois de medir o impacto |
| **Q4** | **Arquivamento da outbox** (~30 dias). | **Reconhecido, fora do escopo desta feature** |
| **Q5** | **Endurecer autorização do CRUD.** Hoje qualquer role autenticada cadastra; só o replay exige ADMIN. | *"Por enquanto sim. Mais pra frente a gente pode endurecer."* |

## 6. Impacto e riscos

### 6.1 Impacto no sistema existente

A alteração crítica é **um ponto só**: `changeStatus` passa a gravar na outbox dentro da transação
que já existe. A proposta é uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que
recebe o client da transação corrente, em vez de injetar um repositório inteiro no serviço de
pedidos.

Todo o resto é aditivo: módulo novo, tabelas novas, entry-point novo.

### 6.2 Riscos

| Risco | Mitigação |
| --- | --- |
| Cliente lento degrada a fila | timeout de 10s por chamada |
| Outbox cresce e a leitura fica lenta | índices em status e `created_at`, batch pequeno, arquivamento futuro |
| Perda de ordem ao escalar | manter single-worker; limitação documentada |
| Vazamento de secret | secret por endpoint e rotação com grace period |
| Prazo | 3 sprints estimados, já incluindo 2 dias de revisão de segurança |

## 7. Decisões relacionadas

| ADR | Decisão |
| --- | --- |
| [ADR-001](adrs/ADR-001-outbox-no-mysql.md) | Padrão outbox no MySQL existente |
| [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) | Worker em processo separado, polling de 2s |
| [ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md) | 5 tentativas, backoff 1m/5m/30m/2h/12h, DLQ em tabela própria |
| [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint e rotação |
| [ADR-005](adrs/ADR-005-at-least-once-com-event-id.md) | At-least-once com `X-Event-Id` |
| [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) | Reuso dos padrões existentes do projeto |
| [ADR-007](adrs/ADR-007-snapshot-do-payload-na-outbox.md) | Snapshot do payload na inserção |

---

> **Próximo passo acordado:** abrir o FDD e marcar sessão de revisão com Bruno e Diego antes de
> começar a implementação.
