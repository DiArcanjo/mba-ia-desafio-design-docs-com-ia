# ADR-004: Autenticação HMAC-SHA256 com secret por endpoint e rotação

- **Status:** Aceito
- **Data:** 2026-10-14
- **Decisores:** Sofia (Segurança), Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Relacionados:** [ADR-005](ADR-005-at-least-once-com-x-event-id.md)

## Contexto

Os webhooks enviam dados de pedidos para endpoints fora da nossa infraestrutura. O cliente precisa conseguir verificar que:

1. a requisição foi enviada pela nossa plataforma;
2. o payload não foi adulterado no caminho.

Já houve caso de cliente que vazou uma secret em log de aplicação. O tráfego é apenas de saída (outbound): os clientes recebem, não enviam.

## Decisão

1. **Assinatura HMAC-SHA256 sobre o corpo da requisição**, enviada no header `X-Signature`.
2. **Uma secret única por endpoint de webhook**, gerada pela plataforma e devolvida ao cliente somente na criação e na rotação. Ela nunca aparece em listagens. A configuração do endpoint armazena `url`, `secret`, `customer_id` e o estado ativo.
3. **Rotação via API.** O cliente pode pedir uma nova secret. A anterior continua válida por **24 horas** em paralelo e depois é descartada.
4. **TLS obrigatório.** A URL precisa usar `https`, o que é validado no schema Zod. Isso é tratado como validação, não como decisão arquitetural.

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Secret global da plataforma** | Se uma secret vazar, todos os clientes ficam comprometidos. Com secret por endpoint, o impacto de um vazamento fica isolado. |
| **Rotação sem período de convivência** | Obrigaria o cliente a trocar a secret em todos os sistemas no mesmo instante, com risco de rejeitar entregas legítimas. O grace period de 24h dá tempo de migrar. |

## Consequências

**Positivas**
- É o padrão de mercado: qualquer cliente tem biblioteca para validar HMAC-SHA256.
- Um vazamento atinge um único endpoint e pode ser resolvido com rotação.
- O cliente consegue rotacionar sozinho, sem acionar o suporte.

**Negativas**
- As secrets precisam ser armazenadas recuperáveis (não hash), porque precisamos delas para assinar. Isso exige cuidado de acesso e de logs.
- Durante o grace period, o worker precisa assinar com duas secrets (detalhe no [FDD](../FDD.md)).
- Sofia exige 2 dias úteis de revisão do código de HMAC e de geração de secret antes do deploy.

**Trade-off:** aceitamos guardar e gerenciar uma secret por endpoint, com rotação, em troca de isolamento de vazamentos e validação simples do lado do cliente.

## Referências

- Transcrição: [09:02] Sofia e Marcos, [09:03] Sofia, [09:19] Sofia, [09:20] Sofia e Bruno, [09:21] Sofia e Bruno, [09:22] Diego e Sofia, [09:23] Sofia, [09:31] Marcos, [09:46] Sofia
