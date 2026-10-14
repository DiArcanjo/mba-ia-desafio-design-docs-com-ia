# Tracker de Rastreabilidade

Cada item registrado no PRD, no RFC, no FDD e nos ADRs aponta para a sua origem: um momento da reunião (`TRANSCRICAO.md`, no formato `[hh:mm] Nome`) ou um arquivo do código base.

Quando um item foi decidido por mais de uma pessoa, a localização indica a fala principal. As demais falas relacionadas estão na seção "Referências" de cada ADR.

**Resumo:** 233 itens rastreados · 202 da transcrição (86%) · 31 do código.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox em tabela MySQL gravada na mesma transação da mudança de status | TRANSCRICAO | [09:06] Diego |
| ADR-001-DEC | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox em MySQL formalmente decidida pela tech lead | TRANSCRICAO | [09:08] Larissa |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa descartada | Disparo HTTP síncrono dentro de changeStatus | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa descartada | Redis Streams exigiria subir infraestrutura nova | TRANSCRICAO | [09:07] Larissa |
| ADR-001-TO | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Redis Cluster seria overengineering para time pequeno | TRANSCRICAO | [09:07] Diego |
| ADR-001-CTX | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Transação de status já atualiza orders, history e estoque | CODIGO | `src/modules/orders/order.service.ts` |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker lê a outbox por polling a cada 2 segundos | TRANSCRICAO | [09:09] Diego |
| ADR-002-DEC | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Latência mínima de 2 s aceita e registrada como decisão | TRANSCRICAO | [09:10] Larissa |
| ADR-002-PROC | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker em processo separado da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-ENTRY | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Entry point src/worker.ts e script npm run worker | TRANSCRICAO | [09:11] Larissa |
| ADR-002-PRISMA | docs/adrs/ADR-002-worker-separado-em-polling.md | Restrição | Worker usa PrismaClient próprio com a mesma DATABASE_URL | TRANSCRICAO | [09:30] Bruno |
| ADR-002-ALT-01 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa descartada | Trigger de banco / LISTEN-NOTIFY indisponível no MySQL | TRANSCRICAO | [09:09] Diego |
| ADR-002-CODE | docs/adrs/ADR-002-worker-separado-em-polling.md | Referência de código | src/server.ts como modelo de entry point e shutdown | CODIGO | `src/server.ts` |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | 5 retentativas com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| ADR-003-BACKOFF | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | Progressão 1m/5m/30m/2h/12h, quase 15 horas no total | TRANSCRICAO | [09:17] Diego |
| ADR-003-ALT-01 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa descartada | Retry indefinido deixa evento pendurado para sempre | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-02 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa descartada | 3 tentativas cobrem só ~30 min de indisponibilidade | TRANSCRICAO | [09:16] Diego |
| ADR-003-DLQ | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | DLQ em tabela separada webhook_dead_letter | TRANSCRICAO | [09:18] Diego |
| ADR-003-TO | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Trade-off | Indisponibilidade acima de ~15h é problema do cliente | TRANSCRICAO | [09:17] Marcos |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Assinatura HMAC-SHA256 no header X-Signature | TRANSCRICAO | [09:20] Sofia |
| ADR-004-SECRET | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Secret única por endpoint, não global | TRANSCRICAO | [09:21] Sofia |
| ADR-004-ROT | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Rotação de secret via API com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| ADR-004-CORPO | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC calculado sobre o corpo do request | TRANSCRICAO | [09:22] Sofia |
| ADR-004-TLS | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Restrição | URL obrigatoriamente https, validada no schema Zod | TRANSCRICAO | [09:23] Sofia |
| ADR-004-MOTIVO | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Contexto | Histórico de cliente que vazou secret em log | TRANSCRICAO | [09:22] Diego |
| ADR-005 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | Garantia de entrega at-least-once | TRANSCRICAO | [09:24] Diego |
| ADR-005-EVTID | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | X-Event-Id com UUID gerado na entrada da outbox | TRANSCRICAO | [09:25] Diego |
| ADR-005-ALT-01 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Alternativa descartada | Exactly-once exigiria coordenação dos dois lados | TRANSCRICAO | [09:25] Diego |
| ADR-005-TO | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Trade-off | Deduplicação passa a ser responsabilidade do cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Reuso máximo dos padrões existentes do projeto | TRANSCRICAO | [09:30] Larissa |
| ADR-006-MOD | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Módulo src/modules/webhooks no padrão controller/service/repository/routes/schemas | TRANSCRICAO | [09:27] Bruno |
| ADR-006-PROC | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Lógica do worker em arquivo do módulo (webhook.processor.ts) | TRANSCRICAO | [09:28] Bruno |
| ADR-006-ERR | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Erros estendem AppError com códigos WEBHOOK_* | TRANSCRICAO | [09:28] Bruno |
| ADR-006-LOG | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Restrição | Pino e error middleware reaproveitados, nada novo | TRANSCRICAO | [09:29] Bruno |
| ADR-006-CODE-01 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Referência de código | Classe base AppError (statusCode, errorCode, details) | CODIGO | `src/shared/errors/app-error.ts` |
| ADR-006-CODE-02 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Referência de código | authenticate e requireRole reaproveitados | CODIGO | `src/middlewares/auth.middleware.ts` |
| ADR-006-CODE-03 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Referência de código | requireRole('ADMIN') já usado em rotas de usuário | CODIGO | `src/modules/users/user.routes.ts` |
| ADR-007 | docs/adrs/ADR-007-snapshot-e-filtro-na-insercao.md | Decisão | Payload renderizado como snapshot na inserção | TRANSCRICAO | [09:52] Larissa |
| ADR-007-FILTRO | docs/adrs/ADR-007-snapshot-e-filtro-na-insercao.md | Decisão | Filtro de status aplicado na inserção da outbox | TRANSCRICAO | [09:34] Bruno |
| ADR-007-PAYLOAD | docs/adrs/ADR-007-snapshot-e-filtro-na-insercao.md | Decisão | Payload enxuto, sem itens do pedido | TRANSCRICAO | [09:43] Diego |
| ADR-007-64KB | docs/adrs/ADR-007-snapshot-e-filtro-na-insercao.md | Restrição | Limite de 64KB com erro, sem truncar | TRANSCRICAO | [09:24] Larissa |
| ADR-007-ALT-01 | docs/adrs/ADR-007-snapshot-e-filtro-na-insercao.md | Alternativa descartada | Truncar payload grande | TRANSCRICAO | [09:23] Sofia |
| ADR-008 | docs/adrs/ADR-008-ordenacao-por-pedido-com-worker-unico.md | Decisão | Worker único com ordem garantida só por order_id | TRANSCRICAO | [09:12] Diego |
| ADR-008-LIM | docs/adrs/ADR-008-ordenacao-por-pedido-com-worker-unico.md | Restrição | Ordering registrada como limitação conhecida | TRANSCRICAO | [09:13] Larissa |
| ADR-008-ALT-01 | docs/adrs/ADR-008-ordenacao-por-pedido-com-worker-unico.md | Alternativa adiada | Múltiplos workers com partição por order_id ou lock | TRANSCRICAO | [09:13] Diego |
| ADR-008-ALT-02 | docs/adrs/ADR-008-ordenacao-por-pedido-com-worker-unico.md | Alternativa descartada | Ordem global entre pedidos não foi pedida | TRANSCRICAO | [09:14] Marcos |
| ADR-008-CODE | docs/adrs/ADR-008-ordenacao-por-pedido-com-worker-unico.md | Referência de código | Transições permitidas da máquina de estados | CODIGO | `src/modules/orders/order.status.ts` |
| RFC-CTX-01 | docs/RFC.md | Contexto | Três clientes B2B pediram notificação de mudança de status | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-02 | docs/RFC.md | Restrição | Tempo real para o cliente é abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| RFC-CTX-03 | docs/RFC.md | Contexto | Clientes fazem polling em GET /orders | CODIGO | `src/modules/orders/order.routes.ts` |
| RFC-PROP-01 | docs/RFC.md | Decisão | Publicação via publishWebhookEvent(tx, order, fromStatus, toStatus) | TRANSCRICAO | [09:41] Bruno |
| RFC-PROP-02 | docs/RFC.md | Decisão | Headers X-Event-Id, X-Signature, X-Timestamp, Content-Type | TRANSCRICAO | [09:44] Diego |
| RFC-PROP-03 | docs/RFC.md | Decisão | Header X-Webhook-Id adicional | TRANSCRICAO | [09:44] Sofia |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | Disparo síncrono em changeStatus | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams / Redis Cluster | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger de banco para acordar o worker | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | Retry indefinido ou apenas 3 tentativas | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa descartada | Garantia exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa descartada | Secret global da plataforma | TRANSCRICAO | [09:21] Sofia |
| RFC-Q-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída por cliente | TRANSCRICAO | [09:39] Larissa |
| RFC-Q-02 | docs/RFC.md | Questão em aberto | Aviso por e-mail ao cliente com falhas repetidas | TRANSCRICAO | [09:37] Larissa |
| RFC-Q-03 | docs/RFC.md | Questão em aberto | Escalar para múltiplos workers | TRANSCRICAO | [09:13] Diego |
| RFC-Q-04 | docs/RFC.md | Questão em aberto | Endurecer autorização do CRUD de webhooks | TRANSCRICAO | [09:37] Sofia |
| RFC-Q-05 | docs/RFC.md | Questão em aberto | Arquivamento de linhas entregues da outbox (~30 dias) | TRANSCRICAO | [09:08] Diego |
| RFC-Q-06 | docs/RFC.md | Questão em aberto | Contagem de tentativas: 5 tentativas x quase 15 horas | TRANSCRICAO | [09:17] Diego |
| RFC-Q-07 | docs/RFC.md | Questão em aberto | customer_id no body ou no path | TRANSCRICAO | [09:32] Larissa |
| RFC-RISCO-01 | docs/RFC.md | Risco | Prazo de 3 sprints incluindo revisão de segurança | TRANSCRICAO | [09:46] Larissa |
| PRD-CTX-01 | docs/PRD.md | Contexto | Máquina de estados do pedido | CODIGO | `src/modules/orders/order.status.ts` |
| PRD-CTX-02 | docs/PRD.md | Restrição | Webhooks apenas de saída (outbound) | TRANSCRICAO | [09:02] Marcos |
| PRD-PROB-01 | docs/PRD.md | Problema | Polling em GET /orders é lento e caro para os clientes | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-02 | docs/PRD.md | Problema | Risco de churn da Atlas sem a feature até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-03 | docs/PRD.md | Restrição | Atlas pediu entrega até o fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-PROB-04 | docs/PRD.md | Contexto | OMS sem mecanismo de eventos/notificação (só módulos de domínio) | CODIGO | `src/app.ts` |
| PRD-PUB-01 | docs/PRD.md | Público | Usuários da plataforma que representam o cliente | TRANSCRICAO | [09:32] Marcos |
| PRD-PUB-02 | docs/PRD.md | Público | Administradores fazem o reenvio manual | TRANSCRICAO | [09:36] Sofia |
| PRD-CEN-02 | docs/PRD.md | Cenário | Cliente com indisponibilidade de 2h em manutenção | TRANSCRICAO | [09:16] Diego |
| PRD-OBJ-01 | docs/PRD.md | Objetivo/Métrica | p95 < 10 s até a primeira tentativa de entrega | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo/Métrica | 3 de 3 clientes solicitantes ativos até fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-03 | docs/PRD.md | Objetivo/Métrica | 100% das mudanças de status assinadas geram evento | TRANSCRICAO | [09:40] Bruno |
| PRD-OBJ-04 | docs/PRD.md | Objetivo/Métrica | Janela de ~15h de reenvios automáticos | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-05 | docs/PRD.md | Objetivo/Métrica | Redução do polling em GET /orders | TRANSCRICAO | [09:00] Marcos |
| PRD-OOS-01 | docs/PRD.md | Fora de escopo | Alerta por e-mail ao cliente (adiado) | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Fora de escopo | Rate limiting de envio (adiado, observar) | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-03 | docs/PRD.md | Fora de escopo | Dashboard visual (time de frontend) | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-04 | docs/PRD.md | Fora de escopo | Webhooks de entrada | TRANSCRICAO | [09:02] Marcos |
| PRD-OOS-05 | docs/PRD.md | Fora de escopo | Garantia exactly-once | TRANSCRICAO | [09:25] Diego |
| PRD-OOS-06 | docs/PRD.md | Fora de escopo | Garantia de ordem global | TRANSCRICAO | [09:14] Marcos |
| PRD-OOS-07 | docs/PRD.md | Fora de escopo | Arquivamento de notificações antigas | TRANSCRICAO | [09:08] Diego |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook com URL e status desejados | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Editar webhook (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Remover webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer (GET) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Filtro de status por webhook | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Mudança de status gera notificação aos assinantes | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Conteúdo do evento sem itens do pedido | TRANSCRICAO | [09:43] Diego |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Notificação assinada para validar origem e integridade | TRANSCRICAO | [09:19] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Rotação de secret com 24h de convivência | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Reenvio automático com intervalos crescentes | TRANSCRICAO | [09:15] Diego |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Fila de falhas com reenvio manual auditado | TRANSCRICAO | [09:18] Diego |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas por webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Identificador único do evento para deduplicação | TRANSCRICAO | [09:25] Diego |
| PRD-FR-15 | docs/PRD.md | Requisito Funcional | Identificação do webhook de origem | TRANSCRICAO | [09:44] Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Primeira tentativa < 10 s; latência mínima de 2 s aceita | TRANSCRICAO | [09:10] Larissa |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Sem status alterado sem evento, nem evento de rollback | TRANSCRICAO | [09:06] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Cliente lento não bloqueia mudanças de status | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Entrega at-least-once | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Somente URLs https | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | HMAC-SHA256 com secret única por webhook | TRANSCRICAO | [09:20] Sofia |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 s na resposta do cliente | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Limite de 64KB por notificação | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Ordem garantida só por pedido com processamento único | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Reenvio exige ADMIN; CRUD qualquer usuário autenticado | TRANSCRICAO | [09:36] Larissa |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Nenhuma infraestrutura nova | TRANSCRICAO | [09:07] Diego |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança de 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Documentação no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-03 | docs/PRD.md | Dependência | Confirmação de prazo com os clientes | TRANSCRICAO | [09:47] Marcos |
| PRD-RISCO-01 | docs/PRD.md | Risco | Prazo de 3 sprints contra pedido da Atlas | TRANSCRICAO | [09:46] Larissa |
| PRD-RISCO-02 | docs/PRD.md | Risco | Cliente não deduplica eventos | TRANSCRICAO | [09:25] Sofia |
| PRD-RISCO-03 | docs/PRD.md | Risco | Vazamento de secret | TRANSCRICAO | [09:22] Diego |
| PRD-RISCO-04 | docs/PRD.md | Risco | Rajada de notificações para um cliente | TRANSCRICAO | [09:38] Diego |
| PRD-RISCO-05 | docs/PRD.md | Risco | Indisponibilidade do cliente acima de 15h | TRANSCRICAO | [09:17] Marcos |
| PRD-RISCO-06 | docs/PRD.md | Risco | CRUD aberto a qualquer usuário autenticado | TRANSCRICAO | [09:37] Sofia |
| PRD-CA-01 | docs/PRD.md | Critério de aceitação | Notificação de SHIPPED em menos de 10 s | TRANSCRICAO | [09:02] Marcos |
| PRD-CA-02 | docs/PRD.md | Critério de aceitação | Status não assinado não gera envio | TRANSCRICAO | [09:34] Bruno |
| PRD-CA-03 | docs/PRD.md | Critério de aceitação | URL http recusada | TRANSCRICAO | [09:23] Sofia |
| PRD-CA-04 | docs/PRD.md | Critério de aceitação | Indisponibilidade de 2h recuperada sem ação manual | TRANSCRICAO | [09:16] Diego |
| PRD-CA-05 | docs/PRD.md | Critério de aceitação | Reenvio por ADMIN com autor registrado | TRANSCRICAO | [09:36] Sofia |
| PRD-CA-06 | docs/PRD.md | Critério de aceitação | Secret anterior válida por 24h após rotação | TRANSCRICAO | [09:21] Sofia |
| PRD-CA-07 | docs/PRD.md | Critério de aceitação | Histórico das últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-CA-08 | docs/PRD.md | Critério de aceitação | Mudança de status não depende do cliente | TRANSCRICAO | [09:04] Bruno |
| PRD-CA-09 | docs/PRD.md | Critério de aceitação | Notificação com IDs de evento e webhook e assinatura | TRANSCRICAO | [09:44] Diego |
| PRD-TESTE-01 | docs/PRD.md | Estratégia de teste | Suíte Vitest + Supertest existente | CODIGO | `tests/orders.test.ts` |
| FDD-OBJ-01 | docs/FDD.md | Objetivo técnico | Evento atômico com a mudança de status | TRANSCRICAO | [09:40] Bruno |
| FDD-OBJ-02 | docs/FDD.md | Objetivo técnico | Primeira tentativa < 10 s com polling de 2 s | TRANSCRICAO | [09:10] Larissa |
| FDD-OBJ-03 | docs/FDD.md | Objetivo técnico | Requisições assinadas HMAC-SHA256 por endpoint | TRANSCRICAO | [09:22] Sofia |
| FDD-OBJ-04 | docs/FDD.md | Objetivo técnico | event_id estável em retentativas e replays | TRANSCRICAO | [09:25] Diego |
| FDD-OBJ-05 | docs/FDD.md | Objetivo técnico | Backoff e DLQ | TRANSCRICAO | [09:17] Larissa |
| FDD-OBJ-06 | docs/FDD.md | Restrição | Contrato dos endpoints de pedidos inalterado | CODIGO | `src/modules/orders/order.routes.ts` |
| FDD-ESC-01 | docs/FDD.md | Exclusão | Criação de pedido não publica evento (só changeStatus) | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-DADOS-01 | docs/FDD.md | Modelo de dados | webhook_endpoints com url, secret, customer_id, ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-DADOS-02 | docs/FDD.md | Modelo de dados | previousSecret + expiração de 24h para rotação | TRANSCRICAO | [09:21] Sofia |
| FDD-DADOS-03 | docs/FDD.md | Modelo de dados | Índices da outbox em status e created_at | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-04 | docs/FDD.md | Decisão de implementação | Uma linha de outbox por endpoint, cada uma com X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| FDD-DADOS-05 | docs/FDD.md | Modelo de dados | Payload salvo como snapshot | TRANSCRICAO | [09:52] Larissa |
| FDD-DADOS-06 | docs/FDD.md | Modelo de dados | webhook_deliveries para o histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-DADOS-07 | docs/FDD.md | Modelo de dados | webhook_dead_letter com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-DADOS-08 | docs/FDD.md | Decisão de implementação | DLQ mantida como evidência mesmo após remoção do endpoint | TRANSCRICAO | [09:18] Diego |
| FDD-DADOS-09 | docs/FDD.md | Restrição | IDs UUID seguindo o padrão do projeto | TRANSCRICAO | [09:51] Larissa |
| FDD-DADOS-10 | docs/FDD.md | Referência de código | Padrão Char(36), @@map e timestamps dos models | CODIGO | `prisma/schema.prisma` |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | publishWebhookEvent dentro da transação de changeStatus | TRANSCRICAO | [09:41] Bruno |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Worker busca pendentes mais antigos em lote pequeno | TRANSCRICAO | [09:08] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Retry com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | DLQ e replay recolocando o evento como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-05 | docs/FDD.md | Fluxo | Rotação de secret com convivência de 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-FLUXO-06 | docs/FDD.md | Fluxo | Recuperação de PROCESSING coerente com at-least-once | TRANSCRICAO | [09:24] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /api/v1/webhooks (secret devolvida na criação) | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /api/v1/webhooks/customers/:customerId (customer no path) | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /api/v1/webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /api/v1/webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST /api/v1/webhooks/:id/rotate-secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | GET /api/v1/webhooks/:id/deliveries | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /api/v1/admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Headers do webhook outbound | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | Payload JSON order.status_changed | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-10 | docs/FDD.md | Decisão de implementação | customer_id no body (criação) ou no path (listagem), não no JWT | TRANSCRICAO | [09:32] Larissa |
| FDD-CONTRATO-11 | docs/FDD.md | Referência de código | Prefixo /api/v1 e formato de erro | CODIGO | `src/app.ts` |
| FDD-CONTRATO-12 | docs/FDD.md | Referência de código | Formato paginado da listagem | CODIGO | `src/shared/http/response.ts` |
| FDD-ERR-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Erro | Prefixo WEBHOOK_ para todos os códigos do módulo | TRANSCRICAO | [09:29] Larissa |
| FDD-ERR-03 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL para URL não-https | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-04 | docs/FDD.md | Erro | WEBHOOK_INVALID_EVENTS para filtro inválido | TRANSCRICAO | [09:33] Marcos |
| FDD-ERR-05 | docs/FDD.md | Erro | WEBHOOK_ENDPOINT_INACTIVE (estado ativo do endpoint) | TRANSCRICAO | [09:21] Bruno |
| FDD-ERR-06 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_NOT_FOUND / ALREADY_REPLAYED | TRANSCRICAO | [09:35] Diego |
| FDD-ERR-08 | docs/FDD.md | Erro | WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-09 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-10 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_TIMEOUT (10 s) | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-11 | docs/FDD.md | Erro | Falhas de rede/HTTP entram em retry | TRANSCRICAO | [09:15] Diego |
| FDD-ERR-14 | docs/FDD.md | Erro | Falha ao inserir na outbox causa rollback | TRANSCRICAO | [09:40] Bruno |
| FDD-ERR-15 | docs/FDD.md | Referência de código | Zod convertido em VALIDATION_ERROR pelo validate | CODIGO | `src/middlewares/validate.middleware.ts` |
| FDD-RES-01 | docs/FDD.md | Resiliência | Timeout de 10 s por requisição | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | docs/FDD.md | Resiliência | Cliente lento tratado como falha e retentado | TRANSCRICAO | [09:42] Diego |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs estruturados com o Pino existente | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Log de auditoria de quem fez o replay | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Redaction de secret e assinatura no logger | CODIGO | `src/shared/logger/index.ts` |
| FDD-OBS-04 | docs/FDD.md | Observabilidade | Correlação por eventId do X-Event-Id | TRANSCRICAO | [09:25] Diego |
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus chama publishWebhookEvent na transação | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-02 | docs/FDD.md | Integração | Filtro aceita status destino de transitions | CODIGO | `src/modules/orders/order.status.ts` |
| FDD-INT-03 | docs/FDD.md | Integração | Erros WEBHOOK_* estendem classes HTTP existentes | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-INT-04 | docs/FDD.md | Integração | errorMiddleware trata os novos erros sem mudança | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-INT-05 | docs/FDD.md | Integração | Montagem dos routers de webhooks e admin | CODIGO | `src/routes/index.ts` |
| FDD-INT-06 | docs/FDD.md | Integração | buildControllers instancia o módulo | CODIGO | `src/app.ts` |
| FDD-INT-07 | docs/FDD.md | Integração | Worker usa createPrismaClient() | CODIGO | `src/config/database.ts` |
| FDD-INT-08 | docs/FDD.md | Integração | Worker reutiliza o env validado | CODIGO | `src/config/env.ts` |
| FDD-INT-09 | docs/FDD.md | Integração | Scripts worker e worker:dev | CODIGO | `package.json` |
| FDD-INT-10 | docs/FDD.md | Integração | beforeEach limpa tabelas de webhook | CODIGO | `tests/setup.ts` |
| FDD-INT-11 | docs/FDD.md | Integração | Factory de WebhookEndpoint | CODIGO | `tests/helpers/factories.ts` |
| FDD-INT-12 | docs/FDD.md | Integração | Modelo do entry point src/worker.ts | TRANSCRICAO | [09:11] Larissa |
| FDD-DEP-01 | docs/FDD.md | Dependência | fetch nativo do Node >= 20, sem lib HTTP nova | CODIGO | `package.json` |
| FDD-DEP-02 | docs/FDD.md | Restrição | Worker executado em instância única | TRANSCRICAO | [09:12] Diego |
| FDD-CA-01 | docs/FDD.md | Critério técnico | 1 linha PENDING por endpoint assinante | TRANSCRICAO | [09:34] Bruno |
| FDD-CA-02 | docs/FDD.md | Critério técnico | Rollback total se a outbox falhar | TRANSCRICAO | [09:40] Bruno |
| FDD-CA-04 | docs/FDD.md | Critério técnico | Primeira tentativa em até 10 s | TRANSCRICAO | [09:02] Marcos |
| FDD-CA-05 | docs/FDD.md | Critério técnico | 5 headers e assinatura conferível | TRANSCRICAO | [09:44] Diego |
| FDD-CA-06 | docs/FDD.md | Critério técnico | Intervalos de retry e DLQ após a 6ª falha | TRANSCRICAO | [09:17] Diego |
| FDD-CA-08 | docs/FDD.md | Critério técnico | Replay por ADMIN; OPERATOR recebe 403 | TRANSCRICAO | [09:36] Sofia |
| FDD-CA-11 | docs/FDD.md | Critério técnico | Payload > 64KB vai para DLQ sem envio | TRANSCRICAO | [09:24] Larissa |
| FDD-CA-13 | docs/FDD.md | Critério técnico | Secret nunca aparece em logs | TRANSCRICAO | [09:22] Diego |
| FDD-CA-14 | docs/FDD.md | Critério técnico | Suíte existente continua passando | CODIGO | `tests/auth.test.ts` |
| FDD-RISCO-01 | docs/FDD.md | Risco | Transação de status mais longa | TRANSCRICAO | [09:04] Bruno |
| FDD-RISCO-02 | docs/FDD.md | Risco | Backlog com clientes lentos | TRANSCRICAO | [09:38] Diego |
| FDD-RISCO-03 | docs/FDD.md | Risco | Segunda instância de worker quebra a ordem | TRANSCRICAO | [09:12] Diego |
| FDD-RISCO-05 | docs/FDD.md | Risco | Crescimento das tabelas sem arquivamento | TRANSCRICAO | [09:08] Diego |
| FDD-ERR-07 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED evita replay duplicado | TRANSCRICAO | [09:18] Diego |
| FDD-ERR-12 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_HTTP_ERROR para respostas fora de 2xx | TRANSCRICAO | [09:15] Diego |
| FDD-ERR-13 | docs/FDD.md | Erro | WEBHOOK_MAX_ATTEMPTS_EXCEEDED ao esgotar o teto de tentativas | TRANSCRICAO | [09:15] Diego |
| FDD-CA-03 | docs/FDD.md | Critério técnico | Status não assinado não cria linha na outbox | TRANSCRICAO | [09:34] Bruno |
| FDD-CA-07 | docs/FDD.md | Critério técnico | Sem resposta em 10 s vira WEBHOOK_DELIVERY_TIMEOUT e retry | TRANSCRICAO | [09:42] Diego |
| FDD-CA-09 | docs/FDD.md | Critério técnico | URL http:// recusada com 400 | TRANSCRICAO | [09:23] Sofia |
| FDD-CA-10 | docs/FDD.md | Critério técnico | Duas assinaturas durante as 24h após a rotação | TRANSCRICAO | [09:21] Sofia |
| FDD-CA-12 | docs/FDD.md | Critério técnico | Deliveries limitado a 100 registros | TRANSCRICAO | [09:34] Marcos |
| FDD-RISCO-04 | docs/FDD.md | Risco | Vazamento de secret por log ou resposta | TRANSCRICAO | [09:22] Diego |
| FDD-RISCO-06 | docs/FDD.md | Risco | Evento em retry segura os seguintes do mesmo pedido | TRANSCRICAO | [09:13] Larissa |
| RFC-Q-08 | docs/RFC.md | Questão em aberto | X-Timestamp fora do HMAC, formato de duas assinaturas e segunda rotação | TRANSCRICAO | [09:44] Diego |
| RFC-Q-09 | docs/RFC.md | Questão em aberto | Listagem da DLQ via API não discutida | TRANSCRICAO | [09:18] Diego |
| RFC-META-01 | docs/RFC.md | Processo | Sessão de revisão do design com Bruno e Diego antes de codar | TRANSCRICAO | [09:50] Larissa |
| PRD-PLANO-01 | docs/PRD.md | Plano | 3 sprints com quebra por etapa e revisão de segurança no fim | TRANSCRICAO | [09:46] Larissa |
| FDD-CA-15 | docs/FDD.md | Critério técnico | Evento seguinte do mesmo pedido espera o anterior em retry | TRANSCRICAO | [09:13] Larissa |
| FDD-CA-16 | docs/FDD.md | Critério técnico | Endpoint travado não atrasa outros clientes | TRANSCRICAO | [09:04] Bruno |
| FDD-FLUXO-07 | docs/FDD.md | Fluxo | Envio assíncrono com limite de envios em voo no worker único | TRANSCRICAO | [09:12] Diego |
| FDD-DADOS-11 | docs/FDD.md | Restrição | FK para Customer com cascade para não quebrar a exclusão de clientes | CODIGO | `src/modules/customers/customer.service.ts` |
| FDD-DADOS-12 | docs/FDD.md | Decisão de implementação | Coluna seq para desempate de ordem por pedido; id continua UUID | TRANSCRICAO | [09:51] Larissa |
| FDD-ERR-16 | docs/FDD.md | Erro | WEBHOOK_PROCESSING_ERROR libera a linha e entra em retry | TRANSCRICAO | [09:24] Diego |
