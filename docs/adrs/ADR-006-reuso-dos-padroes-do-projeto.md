# ADR-006: Reuso dos padrões existentes do projeto no módulo de webhooks

- **Status:** Aceito
- **Data:** 2026-10-14
- **Decisores:** Larissa (Tech Lead), Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-worker-separado-em-polling.md)

## Contexto

O OMS já tem convenções claras de organização, erros, validação e log. Cada domínio vive em `src/modules/<dominio>/` com cinco arquivos (`*.controller.ts`, `*.service.ts`, `*.repository.ts`, `*.routes.ts` e `*.schemas.ts`), e há infraestrutura compartilhada:

| Padrão | Onde está no código |
|---|---|
| Erro base com `statusCode`, `errorCode` e `details` | `src/shared/errors/app-error.ts` (`AppError`) |
| Erros específicos com código próprio | `src/shared/errors/http-errors.ts` (`InvalidStatusTransitionError` → `INVALID_STATUS_TRANSITION`, `InsufficientStockError` → `INSUFFICIENT_STOCK`) |
| Tratamento centralizado de `AppError`, `ZodError` e erros do Prisma | `src/middlewares/error.middleware.ts` |
| Validação de body, query e params com Zod | `src/middlewares/validate.middleware.ts` |
| Autenticação JWT e autorização por papel | `src/middlewares/auth.middleware.ts` (`authenticate`, `requireRole`) |
| Logger Pino com redaction | `src/shared/logger/index.ts` |
| Montagem de dependências e rotas | `src/app.ts` (`buildControllers`), `src/routes/index.ts` (`buildApiRouter`) |

A dúvida era se o módulo de webhooks deveria trazer abordagens próprias ou seguir essas convenções.

## Decisão

**Reuso máximo do que já existe.** O webhook é um módulo como os outros:

1. Pasta `src/modules/webhooks/` com controller, service, repository, routes e schemas. A lógica do worker fica em um arquivo do módulo (`webhook.processor.ts`), chamado pelo entry point `src/worker.ts`.
2. Erros estendem `AppError`, ou `ConflictError` e `UnprocessableEntityError`, que aceitam código próprio, com códigos de prefixo **`WEBHOOK_`**. Os 404 do módulo estendem `AppError` diretamente, porque `NotFoundError` fixa o código `NOT_FOUND`. Exemplos: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL` e `WEBHOOK_SECRET_REQUIRED`.
3. O `errorMiddleware` trata os novos erros **sem nenhuma alteração**.
4. Os logs usam o Pino existente. Nenhuma biblioteca de log nova.
5. Os schemas são Zod, aplicados pelo `validate`.
6. O replay da DLQ usa o `requireRole('ADMIN')` existente, do mesmo jeito que `src/modules/users/user.routes.ts` já faz.

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Estrutura e tratamento de erros próprios para o módulo** (ex.: um serviço de notificação fora de `src/modules`, com respostas de erro em formato próprio) | Criaria um segundo estilo na mesma codebase, exigiria mudanças no middleware de erro e aumentaria o custo de manutenção e de revisão para um time pequeno. |

## Consequências

**Positivas**
- Um dev familiarizado com `orders` entende o módulo de webhooks imediatamente.
- As respostas de erro seguem o mesmo formato (`{ error: { code, message, details } }`) que os clientes da API já conhecem.
- Menos código novo de infraestrutura para revisar, incluindo na revisão de segurança.

**Negativas**
- O módulo herda as limitações atuais. Por exemplo, `requireRole` só conhece os papéis `ADMIN` e `OPERATOR`, então não dá para restringir o CRUD ao "dono" do cliente sem evoluir a autenticação.
- O worker não passa pelo `errorMiddleware` (não é HTTP). Ele precisa logar e classificar os próprios erros com os mesmos códigos `WEBHOOK_*`.

**Trade-off:** priorizamos consistência e velocidade de entrega sobre liberdade de desenho do novo módulo.

## Referências

- Transcrição: [09:27] Bruno, [09:28] Bruno, [09:29] Larissa, Bruno e Diego, [09:30] Bruno e Larissa, [09:36] Larissa
- Código: `src/modules/orders/`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/middlewares/auth.middleware.ts`, `src/shared/logger/index.ts`, `src/app.ts`, `src/routes/index.ts`, `src/modules/users/user.routes.ts`
