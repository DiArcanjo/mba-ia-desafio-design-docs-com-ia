# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Autora** | Diana Arcanjo |
| **Product Manager** | Marcos |
| **Status** | Aprovado na reunião técnica, em detalhamento |
| **Data** | 2026-10-14 |
| **Documentos relacionados** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

---

## 1. Resumo e contexto

O Order Management System (OMS) controla o ciclo de vida dos pedidos dos nossos clientes B2B, com a seguinte máquina de estados:

```
PENDING → PAID → PROCESSING → SHIPPED → DELIVERED
              ↘ CANCELLED (a partir de PENDING, PAID ou PROCESSING)
```

Hoje, a única forma de um cliente saber que um pedido mudou de status é consultar a nossa API repetidamente.

Esta feature cria **webhooks de notificação**. O cliente cadastra uma URL e escolhe quais mudanças de status quer acompanhar, e o OMS passa a **avisá-lo automaticamente, em poucos segundos**, sempre que um pedido dele muda para um desses status. A comunicação é apenas de saída: o OMS envia, o cliente recebe.

## 2. Problema e motivação

- **Polling caro e lento.** Atlas Comercial, MaxDistribuição e Nova Cargo consultam `GET /orders` periodicamente para descobrir mudanças. A integração fica lenta e cara para eles.
- **Pedido formal de três clientes.** Os três pediram formalmente notificação em tempo real na semana anterior à reunião.
- **Risco comercial.** A Atlas sinalizou que pode migrar para um concorrente se não houver solução até o fim do trimestre, e pediu entrega até o **fim de novembro**.
- **Lacuna na plataforma.** O OMS não tem nenhum mecanismo de notificação externa, evento ou fila.

## 3. Público-alvo e cenários de uso

| Público | Necessidade |
|---|---|
| **Clientes B2B integrados por API** (inicialmente Atlas Comercial, MaxDistribuição e Nova Cargo) | Saber em tempo real quando seus pedidos mudam de status, sem polling. |
| **Usuários da plataforma que representam o cliente** | Cadastrar e manter os webhooks pela nossa API, autenticados com JWT do OMS. |
| **Administradores do OMS** | Reprocessar notificações que falharam definitivamente. |
| **Suporte e engenharia** | Investigar por que um cliente diz que não recebeu uma notificação. |

**Cenários**
1. **Acompanhar só o que importa.** A Nova Cargo cadastra um webhook para receber apenas `SHIPPED` e `DELIVERED`. Quando um pedido dela é despachado, o sistema logístico dela é avisado em segundos.
2. **Cliente em manutenção.** O endpoint da MaxDistribuição fica fora do ar por 2 horas. As notificações do período são reenviadas automaticamente quando ele volta, sem nenhuma ação manual.
3. **Secret vazada.** A Atlas percebe que vazou a secret do webhook em um log. Ela pede uma nova pela API e tem 24 horas para atualizar os sistemas, sem perder notificações nesse meio-tempo.
4. **Investigação.** Um cliente diz que não recebeu um aviso. Ele mesmo, ou o suporte, consulta o histórico das últimas entregas daquele webhook, com resultado, resposta e tempo.
5. **Falha definitiva.** Um evento esgota as tentativas. Depois que o cliente corrige o endpoint, um administrador reenvia o evento.

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta |
|---|---|---|---|
| PRD-OBJ-01 | Notificar em "tempo real" na percepção do cliente | Tempo entre a mudança de status e a primeira tentativa de entrega, com o processo de entrega em operação | **< 10 segundos** |
| PRD-OBJ-02 | Atender os clientes que pediram a feature | Clientes solicitantes com ao menos um webhook ativo recebendo eventos | **3 de 3** (Atlas, MaxDistribuição, Nova Cargo). Prazo proposto: fim de novembro, a data pedida pela Atlas |
| PRD-OBJ-03 | Não perder notificações | Mudanças de status, com webhook assinante, que geraram evento registrado | **100%** |
| PRD-OBJ-04 | Recuperar automaticamente de indisponibilidades do cliente | Janela coberta por reenvios automáticos antes de exigir ação manual | **≈ 15 horas** |
| PRD-OBJ-05 | Reduzir o polling dos clientes integrados | Volume de chamadas a `GET /orders` dos três clientes antes e depois da adoção | Tendência de queda (sem meta numérica definida na reunião) |

## 5. Escopo

### Incluso
- Cadastro, edição, listagem e remoção de webhooks por cliente, via API.
- Escolha, por webhook, de quais status de pedido disparam notificação.
- Notificação automática a cada mudança de status de pedido assinada.
- Assinatura de segurança em cada notificação, com secret própria por webhook e rotação pela API.
- Reenvio automático em caso de falha e fila de falhas definitivas com reenvio manual por administrador.
- Histórico das últimas entregas por webhook.
- Documentação de integração no portal do desenvolvedor (responsabilidade de Produto).

### Fora de escopo

| Item | Situação | Origem |
|---|---|---|
| Alerta por **e-mail** ao cliente quando o webhook dele falha repetidamente | **Adiado** para a próxima fase, depois de medir o impacto | [09:37] Marcos e Larissa |
| **Rate limiting** de envio por cliente | **Adiado**: observar em produção e decidir depois | [09:38] Diego, [09:39] Larissa |
| **Dashboard visual** para o cliente ver os webhooks | **Descartado** nesta feature; é projeto do time de frontend | [09:39] Marcos, [09:40] Larissa |
| Webhooks **de entrada** (o cliente enviando para nós) | **Descartado**: o pedido é só de saída | [09:02] Marcos e Sofia |
| Garantia de entrega **exatamente uma vez** | **Descartado**: a garantia é "pelo menos uma vez" | [09:25] Diego |
| Garantia de **ordem global** entre pedidos | **Descartado**: os clientes nunca pediram | [09:13] Larissa, [09:14] Marcos |
| Arquivamento de notificações antigas (cerca de 30 dias) | **Adiado** para uma feature futura | [09:08] Diego |

## 6. Requisitos funcionais

| ID | Requisito |
|---|---|
| PRD-FR-01 | O cliente pode **cadastrar** um webhook informando a URL e a lista de status que deseja receber. |
| PRD-FR-02 | A plataforma **gera a secret** do webhook e a devolve ao cliente na criação. |
| PRD-FR-03 | O cliente pode **editar** um webhook (URL, status assinados, ativo/inativo). |
| PRD-FR-04 | O cliente pode **remover** um webhook. |
| PRD-FR-05 | O cliente pode **listar** os webhooks cadastrados de um customer. |
| PRD-FR-06 | O cliente escolhe, por webhook, **quais status** geram notificação (ex.: só `SHIPPED` e `DELIVERED`). Status não escolhidos não geram envio. |
| PRD-FR-07 | Toda **mudança de status** de um pedido gera notificação para os webhooks ativos do cliente que assinam o novo status. |
| PRD-FR-08 | A notificação traz os dados essenciais da mudança (qual pedido, de qual status para qual, e quando), **sem os itens do pedido**. O conteúdo completo está no FDD. |
| PRD-FR-09 | Cada notificação é **assinada**, para o cliente comprovar a origem e a integridade. |
| PRD-FR-10 | O cliente pode **rotacionar a secret** pela API. A secret anterior continua válida por **24 horas**. |
| PRD-FR-11 | Notificações que falham são **reenviadas automaticamente** com intervalos crescentes, por até cerca de 15 horas. |
| PRD-FR-12 | Notificações que esgotam os reenvios ficam em uma **fila de falhas** e podem ser **reenviadas manualmente** por um administrador, com registro de quem fez. |
| PRD-FR-13 | O cliente pode consultar o **histórico das últimas 100 entregas** de um webhook: sucesso ou falha, conteúdo enviado, resposta recebida e tempo de resposta. |
| PRD-FR-14 | Cada notificação carrega um **identificador único do evento**, para o cliente descartar duplicatas. |
| PRD-FR-15 | Cada notificação informa **qual webhook** a originou, para clientes com vários cadastros. |

## 7. Requisitos não funcionais

| ID | Requisito |
|---|---|
| PRD-NFR-01 | **Latência:** a primeira tentativa de entrega ocorre em menos de 10 s após a mudança de status. A latência mínima de até 2 s é aceita. Exceção: se uma notificação anterior do mesmo pedido estiver em reenvio, a seguinte espera por ela, para preservar a ordem (PRD-NFR-09). |
| PRD-NFR-02 | **Consistência:** não pode haver mudança de status sem o evento correspondente registrado, nem evento de mudança desfeita. |
| PRD-NFR-03 | **Isolamento:** a lentidão ou indisponibilidade de um cliente não pode atrasar nem bloquear mudanças de status de pedidos. |
| PRD-NFR-04 | **Garantia de entrega:** pelo menos uma vez (at-least-once). Duplicatas são possíveis e identificáveis. |
| PRD-NFR-05 | **Segurança de transporte:** apenas URLs `https`. Cadastros `http` são recusados. |
| PRD-NFR-06 | **Segurança da assinatura:** HMAC-SHA256 com secret única por webhook, nunca uma secret global. |
| PRD-NFR-07 | **Tempo de resposta do cliente:** sem resposta em 10 s, a tentativa conta como falha. |
| PRD-NFR-08 | **Tamanho:** notificações acima de 64 KB não são enviadas e são tratadas como erro. |
| PRD-NFR-09 | **Ordem:** garantida apenas entre eventos do mesmo pedido, na configuração inicial de processamento único. |
| PRD-NFR-10 | **Autorização:** reenvio manual exige perfil administrador. A gestão de webhooks exige apenas usuário autenticado (a ser endurecida no futuro). |
| PRD-NFR-11 | **Infraestrutura:** nenhuma infraestrutura nova. A solução usa o banco MySQL e a stack atuais. |

## 8. Decisões e trade-offs principais

| Decisão | Trade-off aceito | Registro |
|---|---|---|
| Registrar o evento junto com a mudança de status e entregar depois, de forma assíncrona | Latência de até 2 s, em troca de nunca perder eventos nem travar pedidos | [ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| 5 reenvios ao longo de cerca de 15 horas, depois fila de falhas | Indisponibilidades acima de 15h exigem ação manual | [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| Assinatura com secret por webhook e rotação | Mais gestão de secrets, em troca de isolar vazamentos | [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) |
| "Pelo menos uma vez" com identificador do evento | O cliente precisa descartar duplicatas | [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| Construir com os padrões já existentes no OMS | Menos liberdade de desenho, mais consistência e velocidade | [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |
| Conteúdo enxuto e congelado no momento da mudança | Quem quiser os itens precisa consultar a API | [ADR-007](adrs/ADR-007-snapshot-e-filtro-na-insercao.md) |
| Ordem garantida só por pedido | Sem ordem global, vazão inicial limitada | [ADR-008](adrs/ADR-008-ordenacao-por-pedido-com-worker-unico.md) |

## 9. Dependências

| Dependência | Responsável | Observação |
|---|---|---|
| Revisão de segurança do código de assinatura e de geração de secret | Sofia | Mínimo de 2 dias úteis antes do deploy |
| Documentação no portal do desenvolvedor (integração, validação da assinatura e duplicatas) | Marcos | Deve destacar o comportamento "pelo menos uma vez" |
| Comunicação de prazo aos clientes | Marcos | Confirmar o prazo com a Atlas |
| Novo processo de entrega em produção | Engenharia / Plataforma | Executado em instância única |
| Clientes com endpoint `https` e validação de assinatura | Clientes | Pré-requisito para receber |

### Plano de entrega

Estimativa da tech lead: **3 sprints**, com a revisão de segurança incluída no fim.

| Etapa | Esforço estimado |
|---|---|
| Modelagem da outbox e da fila de falhas | 1 sprint |
| Processo de entrega e reenvios | 1 sprint |
| Cadastro de webhooks e histórico de entregas | ½ sprint |
| Integração com a mudança de status e testes ponta a ponta | ½ sprint |
| Assinatura, validações e revisão de segurança (mínimo de 2 dias úteis) | restante da 3ª sprint |

## 10. Riscos e mitigação

| ID | Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|---|
| PRD-RISCO-01 | **Prazo:** a estimativa é de 3 sprints, contra o pedido da Atlas para o fim de novembro, com risco de churn | Média | Alto | Plano de 3 sprints com a revisão de segurança já incluída. Marcos confirma o prazo com a Atlas no mesmo dia. |
| PRD-RISCO-02 | **Duplicatas mal tratadas pelo cliente** geram ação repetida no sistema dele | Média | Médio | Identificador único por evento e documentação destacada no portal do desenvolvedor. |
| PRD-RISCO-03 | **Vazamento de secret** de um cliente (já aconteceu antes) | Média | Alto | Secret por webhook, rotação com 24h de convivência e revisão de segurança. |
| PRD-RISCO-04 | **Cliente sobrecarregado** por rajadas de notificações (ex.: 50 pedidos em 1 minuto) | Baixa | Médio | Monitorar o volume por cliente. O rate limiting fica como questão em aberto ([RFC-Q-01](RFC.md#5-questões-em-aberto)). |
| PRD-RISCO-05 | **Indisponibilidade do cliente acima de 15h** leva notificações para a fila de falhas | Baixa | Médio | Histórico de entregas para o cliente e reenvio manual por administrador. O alerta por e-mail fica para a próxima fase. |
| PRD-RISCO-06 | **Gestão de webhooks aberta a qualquer usuário autenticado**, inclusive para outros clientes | Média | Médio | Aceito por ora pela Segurança. O endurecimento da autorização fica registrado como evolução ([RFC-Q-04](RFC.md#5-questões-em-aberto)). |

## 11. Critérios de aceitação

| ID | Critério |
|---|---|
| PRD-CA-01 | Dado um webhook ativo assinando `SHIPPED`, quando um pedido do cliente muda para `SHIPPED`, então o endpoint recebe a notificação em menos de 10 s. |
| PRD-CA-02 | Dado um webhook que assina só `SHIPPED`, quando o pedido muda para `PAID`, então nenhuma notificação é enviada a ele. |
| PRD-CA-03 | Dado um cadastro com URL `http://`, então a plataforma recusa com erro de validação. |
| PRD-CA-04 | Dado um endpoint fora do ar por 2 horas, quando ele volta, então as notificações do período chegam sem ação manual. |
| PRD-CA-05 | Dado um evento que esgotou os reenvios, quando um administrador pede o reenvio, então o cliente recebe o mesmo evento (mesmo identificador) e o reenvio fica registrado com o autor. Um usuário não administrador é recusado. |
| PRD-CA-06 | Dada uma rotação de secret, então notificações assinadas continuam validáveis pela secret anterior por 24 horas. |
| PRD-CA-07 | O cliente consegue ver as últimas 100 entregas de um webhook, com resultado, resposta e tempo. |
| PRD-CA-08 | Nenhuma mudança de status de pedido fica mais lenta por causa da disponibilidade de um cliente. |
| PRD-CA-09 | Toda notificação traz o identificador do evento e do webhook, e uma assinatura válida. |

## 12. Estratégia de testes e validação

- **Testes automatizados** na suíte existente (Vitest + Supertest, `tests/`), cobrindo os critérios de aceitação acima. Os critérios técnicos estão detalhados no [FDD](FDD.md#12-critérios-de-aceite-técnicos).
- **Teste ponta a ponta** com um endpoint receptor de teste: mudança de status → recebimento → validação da assinatura → deduplicação pelo identificador.
- **Revisão de segurança** da Sofia (mínimo de 2 dias úteis) antes do deploy, focada em assinatura e geração de secret.
- **Validação com clientes:** acompanhar o primeiro cliente integrado (Atlas) usando o histórico de entregas e as métricas de latência e sucesso.
- **Acompanhamento pós-lançamento:** latência p95, taxa de sucesso, volume da fila de falhas e volume de reenvios. Esses números alimentam as decisões adiadas (rate limiting e alerta por e-mail).
