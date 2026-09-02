# ADR-002: Worker em processo separado, com polling de 2 segundos

## Status

Aceito. Reunião técnica, `[09:09]` a `[09:11]`.

## Contexto

Definido o outbox (ADR-001), restava decidir **como** o worker lê a tabela e **onde** ele roda.

O requisito de produto é que o cliente seja notificado em **menos de 10 segundos**: foi assim que
os clientes definiram "tempo real" (`[09:02]` Marcos).

## Decisão

**Polling em loop, a cada 2 segundos**, buscando os eventos pendentes mais antigos, processando e
marcando o resultado.

O worker roda como **processo separado da API**, com entry-point próprio em `src/worker.ts` **(a criar)**
e script `npm run worker`, espelhando o `src/server.ts` que já existe.

Cada processo abre **seu próprio `PrismaClient`**, apontando para o mesmo banco e a mesma
`DATABASE_URL`. `PrismaClient` é por processo.

Enquanto houver **um único worker**, os eventos são processados em ordem de `created_at`, o que dá
ordenação implícita **por `order_id`**.

## Alternativas consideradas

### Trigger de banco notificando o worker

Descartada por limitação do MySQL. Não existe `NOTIFY`/`LISTEN` como no Postgres. Um trigger
executa SQL, mas **não notifica processo externo**.

> *"Pra avisar o worker, a gente teria que improvisar algo tipo escrever em arquivo ou bater num
> endpoint, fica esquisito."* - Diego, `[09:09]`

### Worker dentro do processo da API

Descartada por fragilidade operacional.

> *"O worker tem que rodar como processo separado, não dentro da mesma instância da API. Senão se a
> API reinicia, perde o worker."* - Diego, `[09:11]`

### Múltiplos workers em paralelo

Não adotada agora. Ganharia vazão, mas **perde a garantia de ordenação**. As saídas conhecidas
(particionar por `order_id`, lock pessimista) foram registradas como trabalho futuro.

## Consequências

### Positivas

- Implementação simples, sem componente novo
- Reiniciar a API **não derruba** a entrega, e vice-versa
- Ordenação por `order_id` sai de graça com um único worker
- 2s de polling atende o requisito de 10s com folga

### Negativas

- **Latência mínima de 2 segundos**, aceita explicitamente na reunião
- Polling consulta o banco mesmo quando não há evento
- **Ponto único de processamento**: um worker parado interrompe todas as entregas
- Escalar horizontalmente exige resolver ordenação antes
