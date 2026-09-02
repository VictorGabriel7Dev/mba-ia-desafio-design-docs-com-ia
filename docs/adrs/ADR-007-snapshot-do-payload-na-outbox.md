# ADR-007: Snapshot do payload no momento da inserção na outbox

## Status

Aceito. Conversa final da reunião, `[09:51]` a `[09:52]`.

## Contexto

Restava definir o que a linha da outbox guarda: o **payload já renderizado** ou apenas o
`order_id`, deixando a renderização para o momento do envio.

A diferença aparece quando o pedido muda de novo entre a inserção do evento e a entrega, o que é
plausível com retry de até 15 horas (ADR-003).

## Decisão

Guardar o **payload renderizado no momento da inserção**. O evento é um **snapshot** do estado em
que o pedido estava quando o status mudou.

> *"Se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou. Senão tem
> caso esquisito."* - Larissa, `[09:52]`

### Conteúdo do payload

Decidido em `[09:43]`: `event_id`, `event_type` (ex.: `order.status_changed`), `timestamp` em
ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos do
pedido, como `total_cents`.

**Os itens do pedido não vão no payload**, para não inflar. Se o cliente quiser o detalhe, consulta
`GET /orders/:id`.

## Alternativas consideradas

### Guardar apenas o `order_id` e renderizar no envio

Descartada. Com retry longo, o pedido pode ter mudado várias vezes até a entrega, e o cliente
receberia um evento rotulado como "mudou para SHIPPED" contendo o estado atual, que já pode ser
outro. O evento deixaria de descrever o fato que o originou.

### Guardar o payload completo, com itens

Descartada por tamanho. Infla a linha da outbox e o corpo do request sem necessidade, e esbarra no
teto de 64KB definido em `[09:24]`. O caminho para o detalhe é o `GET /orders/:id`.

## Consequências

### Positivas

- O evento **descreve o fato**, não o estado atual: fica correto mesmo entregue horas depois
- A entrega não depende de consultar o pedido de novo, o que simplifica o worker
- Reprocessar a partir da DLQ reenvia exatamente o que deveria ter sido enviado
- Payload enxuto respeita o teto de 64KB com folga

### Negativas

- **Duplicação de dado**: o mesmo conteúdo existe no pedido e na linha da outbox
- A linha da outbox fica maior, o que pesa no crescimento da tabela
- Se houver bug na renderização, ele fica **congelado** no evento; corrigir exige regravar as
  linhas pendentes
