# ADR-002: Worker em processo separado com polling de 2 segundos

- **Status:** Aceito
- **Data:** 2026-10-14
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos), Marcos (Produto)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-008](ADR-008-ordenacao-por-pedido-com-worker-unico.md)

## Contexto

Com a outbox ([ADR-001](ADR-001-outbox-no-mysql.md)), alguém precisa ler os eventos pendentes e entregá-los. O requisito de produto é notificar em menos de 10 segundos, que os clientes consideram "tempo real".

A API sobe por `src/server.ts`, que cria o app Express e usa o `PrismaClient` exportado por `src/config/database.ts`. Se a entrega rodar dentro desse processo, um restart da API interrompe o processamento.

## Decisão

1. **Processo separado.** Criar um novo entry point `src/worker.ts`, executado por um script `npm run worker`, no mesmo molde de `src/server.ts`. O worker usa o mesmo banco (`DATABASE_URL`) e a mesma stack, mas abre **o seu próprio `PrismaClient`**, porque o client é por processo.
2. **Polling em loop a cada 2 segundos.** A cada ciclo, o worker busca um lote pequeno dos eventos pendentes mais antigos (por `created_at`), processa e marca o resultado.
3. A latência mínima de 2 segundos no pior caso foi aceita explicitamente pelo produto.

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Trigger de banco / notificação reativa** | O MySQL não tem um listener nativo como o `LISTEN/NOTIFY` do Postgres. Triggers só executam SQL e não avisam processos externos. Avisar o worker exigiria improvisos (escrever em arquivo, chamar endpoint). |
| **Worker dentro do processo da API** | Um restart da API derrubaria o worker junto, e o ciclo de vida das entregas ficaria acoplado ao deploy da API. |

## Consequências

**Positivas**
- O polling de 2s atende com folga o requisito de menos de 10s.
- A implementação é simples, sem dependência nova.
- API e worker podem ser reiniciados e implantados de forma independente. O worker continua em instância única ([ADR-008](ADR-008-ordenacao-por-pedido-com-worker-unico.md)).

**Negativas**
- Há consultas periódicas ao banco mesmo sem eventos novos.
- Latência mínima de até 2s por evento.
- É mais um processo para operar e monitorar em produção.

**Trade-off:** aceitamos latência de até 2s e um custo constante de consultas em troca de não depender de mecanismos reativos que o MySQL não oferece.

## Referências

- Transcrição: [09:02] Marcos, [09:09] Diego e Bruno, [09:10] Marcos e Larissa, [09:11] Diego, Larissa e Bruno, [09:29] Diego, [09:30] Bruno
- Código: `src/server.ts`, `src/config/database.ts` (`createPrismaClient`), `package.json` (scripts)
