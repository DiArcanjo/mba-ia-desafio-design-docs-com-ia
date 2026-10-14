# ADR-003: Retry com backoff exponencial e DLQ em tabela separada

- **Status:** Aceito (a contagem exata de tentativas está pendente de confirmação, ver RFC-Q-06)
- **Data:** 2026-10-14
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos), Marcos (Produto), Sofia (Segurança)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-005](ADR-005-at-least-once-com-x-event-id.md)

## Contexto

O endpoint do cliente pode estar fora do ar, lento ou devolvendo erro. Já houve cliente com indisponibilidade de duas horas em manutenção planejada. Precisamos de uma política que:

- cubra janelas longas de indisponibilidade;
- não mantenha eventos pendurados para sempre quando o cliente sumiu;
- preserve o evento para investigação e reprocessamento.

## Decisão

1. **Backoff exponencial com 5 retentativas**, nos intervalos **1 min, 5 min, 30 min, 2 h e 12 h**. Isso dá cerca de 15 horas entre a primeira falha e a última tentativa.
2. Considera-se falha: resposta fora da faixa 2xx, erro de rede ou ausência de resposta em 10 segundos (timeout).
3. Esgotadas as retentativas, o evento vira **falha permanente**. Ele é marcado `FAILED` na outbox e copiado para a tabela **`webhook_dead_letter`**, com o payload, o motivo da falha e o timestamp.
4. O reprocessamento é **manual**, via `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente. O endpoint exige role `ADMIN` e registra quem executou o replay.

> **Nota de interpretação:** a reunião fala em "5 tentativas" e também em "quase 15 horas entre primeira falha e última tentativa". As duas afirmações só são compatíveis se forem 5 retentativas após a tentativa inicial (1 + 5 + 30 + 120 + 720 = 876 min ≈ 14h36). Essa leitura está registrada como questão a confirmar no [RFC](../RFC.md#5-questões-em-aberto).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Retry indefinido com backoff** | O evento fica pendurado para sempre se o cliente desapareceu, ocupando a outbox sem perspectiva de entrega. |
| **3 tentativas** | É pouco. Uma indisponibilidade pela manhã esgotaria as tentativas em cerca de 30 minutos, e já houve indisponibilidade de 2 horas. |
| **Marcar como `failed` na própria outbox, sem tabela de DLQ** | Mistura falhas permanentes com o fluxo operacional da outbox. A tabela separada deixa a leitura da outbox mais limpa e funciona como evidência para debug e replay. |

## Consequências

**Positivas**
- Cobre indisponibilidades de até cerca de 15 horas sem intervenção humana.
- Os eventos com falha não se perdem: ficam auditáveis e reprocessáveis.
- Limite claro de tentativas, sem eventos eternos.

**Negativas**
- Uma entrega que só dá certo na última retentativa chega com cerca de 15 horas de atraso.
- O replay depende de ação manual de um administrador.
- A DLQ duplica o payload em uma segunda tabela.

**Trade-off:** escolhemos uma janela de retentativas longa e finita em vez de insistência infinita ou desistência rápida. Uma indisponibilidade acima de 15 horas foi considerada problema do cliente.

## Referências

- Transcrição: [09:15] Diego, [09:16] Bruno, Diego e Larissa, [09:17] Diego, Marcos e Larissa, [09:18] Diego e Bruno, [09:35] Diego, [09:36] Sofia e Larissa, [09:42] Diego
