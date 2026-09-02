# PRD - Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Produto** | Order Management System (OMS) |
| **Feature** | Webhooks de Notificação de Pedidos (outbound) |
| **Autor** | Marcos (Product Manager) |
| **Status** | Aprovado em reunião técnica |
| **Fonte** | Reunião técnica de quinta-feira, 09:00 (`TRANSCRICAO.md`) |
| **Prazo alvo** | Fim de novembro (compromisso com a Atlas Comercial) |

---

## 1. Resumo e contexto

Clientes B2B integrados ao OMS precisam saber, em tempo quase real, quando o status de um pedido
muda. Hoje não existe mecanismo de notificação: o cliente descobre a mudança **perguntando**.

Esta feature entrega **webhooks outbound**: o OMS passa a chamar um endpoint HTTPS do cliente
sempre que um pedido dele muda de status, com payload assinado e garantia de entrega.

O fluxo é **unidirecional**, do OMS para o cliente. Não há webhook de entrada.

## 2. Problema e motivação

Três clientes B2B (**Atlas Comercial**, **MaxDistribuição** e **Nova Cargo**) abriram pedido
formal. O padrão atual de integração deles é **polling no `GET /orders`**, e isso produz três
problemas simultâneos:

| Problema | Efeito |
| --- | --- |
| Integração lenta | o cliente só descobre a mudança no próximo ciclo de polling |
| Integração cara | chamadas repetidas que na maioria das vezes não retornam novidade |
| Atrito operacional | o operador do cliente acaba atualizando manualmente para conferir |

**Risco comercial explícito:** a Atlas sinalizou que pode migrar para o concorrente se a entrega
não sair até o fim do trimestre.

## 3. Público-alvo e cenários de uso

**Público:** clientes B2B com integração sistêmica ao OMS, representados por usuários do próprio
sistema. O cadastro é feito pela API do OMS, autenticado com o JWT existente, e **não** por um
portal separado.

| # | Cenário |
| --- | --- |
| 1 | O cliente cadastra um endpoint HTTPS e escolhe **quais status** quer ouvir (ex.: só `SHIPPED` e `DELIVERED`) |
| 2 | Um pedido do cliente muda de status; o cliente recebe a notificação em poucos segundos, sem perguntar |
| 3 | O endpoint do cliente está fora do ar; o OMS **retenta** ao longo de ~15h antes de desistir |
| 4 | O cliente audita o que recebeu, consultando o **histórico de entregas** de um webhook |
| 5 | O cliente suspeita de vazamento e **rotaciona a secret**, com janela de convivência de 24h |
| 6 | Um evento esgotou as tentativas; um **admin** reprocessa manualmente a partir da DLQ |

## 4. Objetivos e métricas de sucesso

| Objetivo | Métrica |
| --- | --- |
| Notificar em tempo quase real | **latência p95 < 10s** entre a mudança de status e a entrega. O cliente definiu "tempo real" como qualquer coisa **abaixo de 10 segundos** |
| Não perder evento | **zero** casos de status alterado sem evento registrado (garantido por transação) |
| Eliminar o polling | redução mensurável de chamadas ao `GET /orders` vindas dos três clientes |
| Entregar apesar de instabilidade | evento entregue após indisponibilidade do cliente de até **~15h** |
| Não degradar o fluxo de pedidos | tempo de resposta do `changeStatus` **sem regressão** após a feature |

## 5. Escopo

### 5.1 Incluso

- Cadastro, edição, remoção e listagem de configurações de webhook por cliente
- Filtro de eventos por status na própria configuração
- Registro do evento na **outbox**, dentro da transação de mudança de status
- **Worker separado** que lê a outbox e entrega via HTTP
- **Retry** com backoff exponencial e **DLQ** persistida
- Assinatura **HMAC-SHA256** com secret por endpoint, e rotação com grace period
- Histórico de entregas consultável pelo cliente
- Endpoint administrativo de replay a partir da DLQ

### 5.2 Fora de escopo

Itens **descartados ou adiados explicitamente durante a reunião**:

| # | Item | Decisão | Origem |
| --- | --- | --- | --- |
| 1 | **Notificação por e-mail** quando o webhook do cliente falha repetidamente | Adiado. *"Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto."* | `[09:37] Larissa` |
| 2 | **Dashboard visual** para o cliente acompanhar seus webhooks | Fora. *"Só endpoints. Painel é projeto separado do time de frontend."* | `[09:40] Larissa` |
| 3 | **Rate limiting de saída** por cliente | Não implementar agora; observar e decidir depois | `[09:39] Diego` e `[09:39] Larissa` |
| 4 | **Arquivamento** das linhas entregues da outbox | Reconhecido como necessário (~30 dias), mas fora desta feature | `[09:08] Diego` |
| 5 | **Webhook inbound** (cliente enviando para o OMS) | Fora por definição de escopo | `[09:02] Marcos` |

## 6. Requisitos funcionais

| ID | Requisito |
| --- | --- |
| **RF-01** | Cadastrar webhook (`POST`) com `url`, `customer_id` e lista de status ouvidos. A **secret é gerada pelo sistema** e devolvida **apenas na criação** |
| **RF-02** | Editar webhook (`PATCH`): url, lista de status e estado ativo/inativo |
| **RF-03** | Remover webhook (`DELETE`) |
| **RF-04** | Listar webhooks de um cliente (`GET`) |
| **RF-05** | Filtrar eventos por status **na inserção** da outbox: se nenhum webhook do cliente ouve aquele status, o evento **não é gravado** |
| **RF-06** | Registrar o evento na outbox **dentro da mesma transação** que altera o pedido |
| **RF-07** | Entregar o evento por `POST` HTTPS ao endpoint do cliente, com payload assinado |
| **RF-08** | Retentar entregas falhas em **5 tentativas**, com backoff **1m / 5m / 30m / 2h / 12h** |
| **RF-09** | Mover para **DLQ** o evento que esgotou as tentativas, guardando payload, motivo e timestamp |
| **RF-10** | Consultar histórico de entregas de um webhook (`GET /webhooks/:id/deliveries`), com sucesso/falha, payload, resposta e tempo de resposta |
| **RF-11** | Rotacionar a secret de um webhook, mantendo a anterior válida por **24h** |
| **RF-12** | Reprocessar item da DLQ (`POST /admin/webhooks/dead-letter/:id/replay`), **restrito a role ADMIN** e com registro de quem executou |

## 7. Requisitos não funcionais

| ID | Requisito |
| --- | --- |
| **RNF-01** | Latência de entrega **< 10s** em condições normais; o polling de 2s do worker é o piso |
| **RNF-02** | **Timeout de 10s** na chamada HTTP ao cliente; acima disso é falha e entra em retry |
| **RNF-03** | Payload limitado a **64KB**; acima disso o evento **falha**, não é truncado |
| **RNF-04** | URL de webhook **obrigatoriamente HTTPS**; `http` é recusado na validação |
| **RNF-05** | Garantia **at-least-once**: o cliente pode receber o mesmo evento mais de uma vez e deve deduplicar pelo `X-Event-Id` |
| **RNF-06** | Ordenação garantida **apenas por `order_id`** e **apenas enquanto houver um único worker** |
| **RNF-07** | O worker roda em **processo separado** da API |
| **RNF-08** | Identificadores no padrão **UUID**, como no resto do projeto |
| **RNF-09** | Zero dependência de infraestrutura nova: sem Redis, sem broker |

## 8. Decisões e trade-offs principais

| Decisão | Trade-off aceito | ADR |
| --- | --- | --- |
| **Outbox no MySQL** em vez de fila dedicada | perde-se push nativo; ganha-se atomicidade sem subir infra | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| **Worker em polling de 2s**, processo separado | latência mínima de 2s; em troca, simplicidade e resiliência a restart da API | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| **5 tentativas**, backoff até 12h | evento pode levar ~15h para morrer; em troca cobre indisponibilidade real de cliente | [ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md) |
| **HMAC-SHA256**, secret por endpoint | gestão de N secrets; em troca, vazamento fica contido a um cliente | [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) |
| **At-least-once** com `X-Event-Id` | joga a deduplicação para o cliente; em troca evita coordenação distribuída | [ADR-005](adrs/ADR-005-at-least-once-com-event-id.md) |
| **Reuso dos padrões do projeto** | nenhuma liberdade de reinventar; em troca, custo de revisão e manutenção baixo | [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |
| **Snapshot do payload** na inserção | duplica dado; em troca o evento reflete o estado do momento da mudança | [ADR-007](adrs/ADR-007-snapshot-do-payload-na-outbox.md) |

## 9. Dependências

| Dependência | Natureza |
| --- | --- |
| Transação de `changeStatus` no serviço de pedidos | **bloqueante**: a inserção na outbox precisa entrar nela |
| MySQL existente | armazena outbox, DLQ, configurações e entregas |
| Prisma | acesso a dados; o worker abre **client próprio** por ser outro processo |
| `AppError` e middleware de erro | reuso obrigatório |
| Pino | logging, já presente |
| `requireRole` | autorização do endpoint de replay |
| Endpoint HTTPS do cliente | **externo**, fora do nosso controle |
| Revisão de segurança da Sofia | **2 dias úteis** reservados antes do deploy |

## 10. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Cliente lento degrada a fila de entrega | **Alta** - três clientes distintos, nenhum sob nosso controle | Médio: atrasa eventos de todos os clientes, pois o worker é único | Timeout de 10s por chamada, decidido em `[09:42]` |
| Acúmulo de eventos na outbox degrada a leitura | **Média** - cresce com o volume de pedidos e sem arquivamento no escopo | Médio: latência sobe e pode furar os 10s | Índices em status e `created_at`, leitura em batch pequeno, arquivamento previsto para depois |
| Vazamento de secret pelo cliente | **Média** - já aconteceu antes, cliente vazou secret em log de aplicação (`[09:22]` Diego) | Alto: permite forjar eventos naquele endpoint | Secret **por endpoint**, nunca global, e rotação com grace period de 24h |
| Perda de ordem ao escalar para vários workers | **Baixa** - depende de uma decisão futura nossa, não de terceiro | Baixo: os clientes nunca pediram ordering global (`[09:14]` Marcos) | Manter single-worker; limitação documentada como conhecida |
| Falha ao inserir na outbox deixa status alterado sem evento | **Muito baixa** - só por falha de banco | **Crítico**: quebra a promessa central da feature | A inserção está **na mesma transação**: se falha, dá rollback no status junto |
| Evento preso indefinidamente na outbox | **Baixa** - exige cliente que sumiu de vez | Médio: tabela cresce sem limite | Teto de 5 tentativas e transferência para a DLQ |
| Não entregar até o fim do trimestre | **Média** - 3 sprints estimados contra prazo de fim de novembro | **Alto**: a Atlas sinalizou migração para o concorrente | Estimativa já inclui os 2 dias de revisão de segurança; escopo enxuto, com e-mail e dashboard fora |

## 11. Critérios de aceitação

| # | Critério |
| --- | --- |
| **CA-01** | Mudar o status de um pedido cria exatamente uma linha na outbox por webhook interessado, **dentro da mesma transação** |
| **CA-02** | Se a inserção na outbox falhar, a mudança de status **não persiste** |
| **CA-03** | Nenhuma linha é criada quando nenhum webhook do cliente ouve aquele status |
| **CA-04** | O cliente recebe `POST` com os headers `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id` |
| **CA-05** | A assinatura confere com HMAC-SHA256 do corpo usando a secret daquele endpoint |
| **CA-06** | Endpoint offline gera 5 tentativas nos intervalos definidos e depois uma entrada na DLQ |
| **CA-07** | Secret rotacionada mantém a anterior aceita por 24h |
| **CA-08** | Cadastro com URL `http://` é recusado com erro de validação |
| **CA-09** | Payload acima de 64KB resulta em falha explícita, não em truncamento |
| **CA-10** | Replay de DLQ exige role ADMIN e registra o autor |
| **CA-11** | O histórico de entregas devolve status, payload, resposta e tempo de resposta |
| **CA-12** | Latência p95 medida abaixo de 10s |

## 12. Estratégia de testes e validação

| Nível | Cobertura |
| --- | --- |
| **Unidade** | cálculo do backoff; geração e verificação do HMAC; validação de schema (HTTPS, tamanho); decisão de filtro por status |
| **Integração** | transação de `changeStatus` gravando a outbox; **rollback** desfazendo o evento; leitura em batch pelo worker; transição para DLQ no esgotamento |
| **Ponta a ponta** | servidor HTTP de teste como endpoint do cliente: recebe, valida assinatura, simula timeout e erro 5xx, confirma retry e DLQ |
| **Segurança** | revisão dedicada da Sofia sobre HMAC e geração de secret, **antes do deploy**, com 2 dias úteis reservados |
| **Carga** | rajada de mudanças de status para observar acúmulo na outbox e o comportamento do batch |

> A estimativa de **3 sprints** apresentada na reunião já contempla modelagem, worker, CRUD,
> integração no serviço de pedidos, testes ponta a ponta e a revisão de segurança.
