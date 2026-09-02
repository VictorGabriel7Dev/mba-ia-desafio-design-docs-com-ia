# ADR-005: Garantia at-least-once com X-Event-Id para deduplicação no cliente

## Status

Aceito. Reunião técnica, `[09:24]` a `[09:26]`.

## Contexto

Com retry (ADR-003), o mesmo evento pode ser entregue mais de uma vez: basta a chamada chegar ao
cliente e a resposta se perder no caminho. Do nosso lado aquilo conta como falha e vira
retentativa; do lado dele, o evento chegou duas vezes.

Era preciso decidir qual garantia oferecer e quem paga o custo da deduplicação.

## Decisão

Garantia **at-least-once**, assumida explicitamente e documentada para o cliente.

Cada evento carrega um **UUID gerado no momento em que entra na outbox**, enviado no header
**`X-Event-Id`**. O cliente deduplica por esse identificador do lado dele.

> *"Garantir exactly-once exigiria coordenação dos dois lados e fica muito mais complexo.
> At-least-once com event_id resolve 99% dos casos."* - Diego, `[09:25]`

O precedente de mercado foi o argumento decisivo: **Stripe e GitHub fazem assim**.

Marcos assumiu documentar isso **em destaque** no portal do desenvolvedor (`[09:26]`), já que a
responsabilidade recai sobre quem integra.

Junto com o `X-Event-Id`, o request leva `X-Signature`, `X-Timestamp` (permite ao cliente detectar
replay attack, se quiser) e `X-Webhook-Id`, este último sugerido pela Sofia para que um cliente com
vários cadastros saiba **qual deles** originou o envio (`[09:44]`).

## Alternativas consideradas

### Exactly-once

Descartada por complexidade. Exigiria coordenação entre os dois lados, com confirmação e controle
de estado distribuído. O custo não se justifica para o problema.

### At-most-once (sem retry)

Não colocada na mesa, mas é a alternativa lógica oposta: eliminaria a duplicata ao custo de
**perder eventos** sempre que o cliente estivesse instável, que é justamente o cenário que a
feature precisa cobrir.

### Deduplicação do nosso lado

Descartada implicitamente. Não temos como saber se o cliente processou: sem resposta, não há
diferença observável entre "não chegou" e "chegou e a resposta se perdeu".

## Consequências

### Positivas

- Nenhuma coordenação distribuída, implementação simples
- Alinhado com o que integradores B2B já esperam de Stripe e GitHub
- O `X-Event-Id` também serve de chave de correlação em suporte e debug
- Combinado com `X-Timestamp`, o cliente pode se defender de replay

### Negativas

- **A deduplicação é responsabilidade do cliente**, e Sofia registrou a ressalva: *"isso joga
  responsabilidade pro cliente"* (`[09:25]`)
- Cliente que não deduplicar pode processar o mesmo evento duas vezes, com efeito colateral no
  sistema dele
- Cria dependência de documentação clara no portal: se a comunicação falhar, o problema aparece
  como bug nosso
