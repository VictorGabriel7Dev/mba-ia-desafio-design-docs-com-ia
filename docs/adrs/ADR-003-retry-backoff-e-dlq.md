# ADR-003: Retry com backoff exponencial e DLQ em tabela separada

## Status

Aceito. Reunião técnica, `[09:14]` a `[09:19]`.

## Contexto

O endpoint do cliente é externo e pode estar fora do ar. Era preciso definir quantas vezes
retentar, com que espaçamento, e o que fazer quando as tentativas acabam.

O histórico pesou na decisão: já houve cliente com **duas horas** de indisponibilidade em
manutenção planejada.

## Decisão

**Cinco tentativas**, com backoff exponencial:

| Tentativa | Intervalo desde a falha anterior |
| --- | --- |
| 1ª retentativa | 1 minuto |
| 2ª | 5 minutos |
| 3ª | 30 minutos |
| 4ª | 2 horas |
| 5ª | 12 horas |

Total de aproximadamente **15 horas** entre a primeira falha e a última tentativa.

Esgotadas as tentativas, o evento vai para **`webhook_dead_letter`**, uma **tabela separada**, com
payload, motivo da falha e timestamp.

O reprocessamento é **manual**, por endpoint administrativo
`POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente.

O timeout de cada chamada HTTP é de **10 segundos**; acima disso conta como falha (`[09:42]`).

## Alternativas consideradas

### Três tentativas

Descartada por agressiva. Esgotaria em cerca de 30 minutos.

> *"Se o cliente teve indisponibilidade de manhã, a gente retentaria três vezes em 30 minutos e
> mataria. Já tinha cliente nosso com indisponibilidade de duas horas em manutenção planejada."*
> - Diego, `[09:16]`

### Retry indefinido com backoff

Descartada pelo extremo oposto: sem teto, o evento fica pendurado para sempre se o cliente sumiu, e
a outbox cresce sem limite.

### DLQ como flag `failed` na própria outbox

Descartada. Poluiria a leitura do worker na tabela principal.

> *"Mais limpa a leitura da outbox principal, e fica como evidence pra debug e reprocessamento."*
> - Diego, `[09:18]`

## Consequências

### Positivas

- Cobre indisponibilidade real de cliente, inclusive manutenção longa
- Teto explícito evita evento eterno e crescimento indefinido da outbox
- Tabela separada mantém a outbox enxuta e serve de evidência para debug
- Replay manual dá caminho de recuperação sem intervenção no banco

### Negativas

- Um evento pode levar **~15 horas** até ser considerado morto
- Reprocessamento é **manual**: alguém precisa perceber e agir
- Mais uma tabela para modelar, manter e arquivar
- Retentativas espaçadas significam que o cliente pode receber eventos **fora de ordem** em
  relação a eventos novos que passaram de primeira
