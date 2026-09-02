# ADR-006: Reuso dos padrões existentes do projeto

## Status

Aceito. Reunião técnica, `[09:27]` a `[09:30]`.

## Contexto

O módulo de webhooks é a primeira funcionalidade da feature a ser escrita do zero. Havia a escolha
entre seguir os padrões já estabelecidos na codebase ou introduzir estruturas próprias.

Este é o ADR que **referencia diretamente o código existente**.

## Decisão

Reuso máximo. Nada novo onde já existe padrão.

### Estrutura de módulo

O módulo vive em **`src/modules/webhooks/`** (a criar), com a mesma anatomia dos demais. O módulo de pedidos
serve de referência direta, com `order.controller.ts`, `order.service.ts`, `order.repository.ts`,
`order.routes.ts` e `order.schemas.ts`.

### Erros

Reuso de **`src/shared/errors/`**, que já define `AppError` em `app-error.ts` e os erros de HTTP em
`http-errors.ts`. O padrão de código em maiúsculas já existe no projeto, com casos como
`INSUFFICIENT_STOCK` e `INVALID_STATUS_TRANSITION`.

Os erros do módulo seguem o **prefixo `WEBHOOK_`**: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`,
`WEBHOOK_SECRET_REQUIRED`, entre outros.

> *"O middleware de erro centralizado já trata AppError, Zod e Prisma. Vai pegar nossos erros sem
> precisar mudar nada."* - Bruno, `[09:29]`

O middleware em questão é **`src/middlewares/error.middleware.ts`**.

### Autorização

O endpoint de replay usa **`requireRole('ADMIN')`**, exportado de
**`src/middlewares/auth.middleware.ts`** e já em uso em `src/modules/users/user.routes.ts`.

### Logging

**Pino**, já presente no projeto inteiro, inclusive com `pino-http` no
`src/middlewares/request-logger.middleware.ts`. Nada novo é introduzido.

### Validação

**Zod**, no padrão dos `*.schemas.ts` de cada módulo. A exigência de HTTPS e o teto de 64KB são
validações de schema, não decisões arquiteturais (`[09:23]` e `[09:24]`).

### Identificadores

**UUID**, seguindo o resto do projeto.

> *"UUID, segue o padrão do resto do projeto. Tudo é uuid."* - Larissa, `[09:51]`

### Acesso a dados

**Prisma**, com o schema em `prisma/schema.prisma`, onde já vivem `Order`, `OrderItem`,
`OrderStatusHistory` e o enum `OrderStatus`. O worker abre **um `PrismaClient` próprio**, porque
`PrismaClient` é por processo (`[09:30]` Bruno).

## Alternativas consideradas

### Estrutura própria para o módulo de webhooks

Descartada. Um módulo com anatomia diferente aumentaria o custo de leitura para quem já conhece o
projeto e exigiria tratamento especial no middleware de erro.

### Códigos de erro sem prefixo de módulo

Descartada em favor do prefixo `WEBHOOK_`, decidido por Larissa em `[09:29]`. Sem prefixo, o
código de erro não diz de onde veio.

## Consequências

### Positivas

- Curva de leitura zero para quem já conhece o projeto
- O middleware de erro **não precisa de alteração nenhuma**
- Revisão de código mais rápida, inclusive a revisão de segurança
- Menos superfície nova para testar

### Negativas

- Nenhuma liberdade para melhorar padrões que talvez merecessem revisão
- Se o padrão atual tiver limitação, ela é herdada pelo módulo novo
- Acopla o módulo à estrutura vigente: uma refatoração futura do padrão de módulos alcança também
  os webhooks
