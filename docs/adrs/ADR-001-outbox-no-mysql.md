# ADR-001: Padrão Outbox no MySQL existente

## Status

Aceito. Reunião técnica de quinta-feira, `[09:06]` a `[09:08]`.

## Contexto

A entrega de webhook precisa acontecer quando o status de um pedido muda. A transação de
`changeStatus` já é pesada: atualiza `order`, insere em `order_status_history` e movimenta o
estoque dos itens.

Duas restrições apareceram na discussão:

1. **Um cliente lento não pode travar a mudança de status de outros pedidos.** Uma chamada HTTP
   dentro da transação faz exatamente isso.
2. **Não pode existir status alterado sem evento registrado.** Se a notificação sair de fora da
   transação, um crash entre o commit e a publicação perde o evento em silêncio.

A segunda restrição é a que decide: qualquer mecanismo que publique **fora** da transação do banco
não consegue garantir atomicidade entre "o status mudou" e "o evento existe".

## Decisão

Adotar o **padrão outbox** na base MySQL que já usamos.

Na mesma transação SQL que atualiza `order` e `order_status_history`, gravamos uma linha em
`webhook_outbox` com o evento. Um worker separado lê essa tabela e dispara as chamadas HTTP.

> *"Garante que se a transação principal commitou, o evento foi registrado, e se ela deu rollback,
> o evento some junto. Não tem inconsistência possível."* - Diego, `[09:06]`

A tabela leva índice em **status** (pendente, processando, falhou, entregue) e em **`created_at`**.
O worker lê apenas os pendentes, em **batch pequeno**, e marca o resultado.

## Alternativas consideradas

### Disparo síncrono dentro do serviço de pedidos

Descartada. Acopla a disponibilidade do nosso fluxo de pedidos à disponibilidade do cliente. E não
existe resposta boa para o cliente offline: dar rollback numa mudança de status legítima porque um
terceiro caiu é inaceitável.

### Redis Streams ou broker dedicado

Descartada por custo operacional. Resolveria o desacoplamento, mas exige subir e manter
infraestrutura nova para um time pequeno.

> *"Subir Redis Cluster pra isso é overengineering. Outbox no MySQL existente resolve."*
> - Diego, `[09:07]`

Há também o ponto técnico de fundo: publicar num broker **não participa da transação do banco**, o
que reintroduz a janela de inconsistência que estamos justamente fechando.

## Consequências

### Positivas

- **Atomicidade real** entre mudança de status e registro do evento
- **Zero infraestrutura nova**: nada a subir, monitorar ou manter disponível
- A outbox é uma tabela comum, inspecionável com SQL, o que facilita debug
- O rollback da transação limpa o evento sem código de compensação

### Negativas

- **Não há push**: a entrega depende de polling, com latência mínima de 2s (ver ADR-002)
- A tabela **cresce** e precisa de arquivamento. Reconhecido na reunião (~30 dias), mas deixado
  fora do escopo desta feature
- Acúmulo de eventos pode degradar a leitura. Mitigado por índice e batch pequeno
- A carga de entrega passa a competir pelo mesmo banco da aplicação
