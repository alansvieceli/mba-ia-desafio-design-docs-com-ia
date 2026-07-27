# Tracker de Rastreabilidade

Este documento liga itens normativos e decisões às suas fontes. Ele será consolidado à medida que PRD, RFC e FDD forem produzidos.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001 | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Decisão | Persistir snapshot do evento na outbox MySQL na mesma transação da mudança de status | TRANSCRICAO | `[09:06] Diego`; `[09:40] Bruno`; `[09:52] Larissa` |
| ADR-001-INT | `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` | Integração | `changeStatus` já usa transação para pedido, histórico e estoque | CODIGO | `src/modules/orders/order.service.ts` |
| ADR-002 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Decisão | Usar processo worker separado e polling a cada 2 segundos | TRANSCRICAO | `[09:09] Diego`; `[09:11] Diego` |
| ADR-002-LIM | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Limitação | Ordenação por pedido depende de um único worker | TRANSCRICAO | `[09:12] Diego`; `[09:13] Larissa` |
| ADR-003 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Decisão | Fazer 5 retentativas em 1m/5m/30m/2h/12h e depois mover para DLQ | TRANSCRICAO | `[09:17] Larissa`; `[09:18] Diego` |
| ADR-003-REPLAY | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Decisão | Replay de DLQ é manual, restrito a ADMIN e auditado | TRANSCRICAO | `[09:18] Diego`; `[09:36] Sofia` |
| ADR-004 | `docs/adrs/ADR-004-autenticacao-hmac-e-rotacao-de-secret.md` | Decisão | Assinar corpo com HMAC-SHA256 e secret única por endpoint | TRANSCRICAO | `[09:20] Sofia`; `[09:21] Sofia` |
| ADR-004-ROT | `docs/adrs/ADR-004-autenticacao-hmac-e-rotacao-de-secret.md` | Decisão | Manter a secret anterior válida por 24h após rotação | TRANSCRICAO | `[09:21] Sofia`; `[09:22] Sofia` |
| ADR-004-TLS | `docs/adrs/ADR-004-autenticacao-hmac-e-rotacao-de-secret.md` | Restrição | Aceitar somente endpoint HTTPS e payload de até 64 KB | TRANSCRICAO | `[09:23] Sofia`; `[09:24] Larissa` |
| ADR-005 | `docs/adrs/ADR-005-entrega-at-least-once-com-event-id.md` | Decisão | Garantir at-least-once e deduplicação por `X-Event-Id` | TRANSCRICAO | `[09:24] Diego`; `[09:26] Larissa` |
| ADR-005-ID | `docs/adrs/ADR-005-entrega-at-least-once-com-event-id.md` | Decisão | Usar UUID para o evento | TRANSCRICAO | `[09:51] Larissa` |
| ADR-006 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Decisão | Reutilizar módulo, Zod, AppError, Pino, middleware e Prisma | TRANSCRICAO | `[09:27] Bruno`; `[09:30] Larissa` |
| ADR-006-AUTH | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Integração | Reutilizar autenticação JWT e autorização por papel | CODIGO | `src/middlewares/auth.middleware.ts` |
| ADR-006-ERR | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Integração | Reutilizar erro estruturado e middleware central | CODIGO | `src/shared/errors/app-error.ts`; `src/middlewares/error.middleware.ts` |
| ADR-006-LOG | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Integração | Reutilizar logger Pino existente | CODIGO | `src/shared/logger/index.ts` |
| RFC-OBJ-01 | `docs/RFC.md` | Objetivo | Notificar três clientes B2B sobre mudanças de status em menos de 10 segundos | TRANSCRICAO | `[09:00] Marcos`; `[09:02] Marcos` |
| RFC-DEC-01 | `docs/RFC.md` | Decisão | Persistir evento na outbox dentro da transação do pedido | TRANSCRICAO | `[09:06] Diego`; `[09:40] Bruno` |
| RFC-DEC-02 | `docs/RFC.md` | Decisão | Consumir outbox com worker separado em polling de 2 segundos | TRANSCRICAO | `[09:09] Diego`; `[09:11] Diego` |
| RFC-DEC-03 | `docs/RFC.md` | Decisão | Usar cinco retries com backoff definido e DLQ separada | TRANSCRICAO | `[09:17] Larissa`; `[09:18] Diego` |
| RFC-DEC-04 | `docs/RFC.md` | Decisão | Assinar com HMAC-SHA256 e secret por endpoint | TRANSCRICAO | `[09:20] Sofia`; `[09:21] Sofia` |
| RFC-DEC-05 | `docs/RFC.md` | Decisão | Entregar at-least-once com Event ID deduplicável | TRANSCRICAO | `[09:24] Diego`; `[09:26] Larissa` |
| RFC-DEC-06 | `docs/RFC.md` | Decisão | Limitar payload a 64 KB, exigir HTTPS e timeout de 10s | TRANSCRICAO | `[09:23] Sofia`; `[09:24] Larissa`; `[09:42] Diego` |
| RFC-DEC-07 | `docs/RFC.md` | Decisão | Filtrar eventos na inserção da outbox | TRANSCRICAO | `[09:33] Marcos`; `[09:34] Bruno` |
| RFC-DEC-08 | `docs/RFC.md` | Decisão | Restringir replay a ADMIN e auditar o ator | TRANSCRICAO | `[09:35] Larissa`; `[09:36] Sofia` |
| RFC-ALT-01 | `docs/RFC.md` | Alternativa descartada | Não enviar webhook sincronamente em `changeStatus` | TRANSCRICAO | `[09:04] Bruno`; `[09:06] Diego` |
| RFC-ALT-02 | `docs/RFC.md` | Alternativa descartada | Não adotar Redis ou broker nesta fase | TRANSCRICAO | `[09:07] Larissa`; `[09:07] Diego` |
| RFC-ALT-03 | `docs/RFC.md` | Alternativa descartada | Não usar trigger MySQL para acordar processo externo | TRANSCRICAO | `[09:09] Bruno`; `[09:09] Diego` |
| RFC-ALT-04 | `docs/RFC.md` | Alternativa descartada | Não oferecer exactly-once | TRANSCRICAO | `[09:25] Diego` |
| RFC-OPEN-01 | `docs/RFC.md` | Questão em aberto | Avaliar rate limiting após observar o uso | TRANSCRICAO | `[09:38] Diego`; `[09:39] Larissa` |
| RFC-OPEN-02 | `docs/RFC.md` | Questão em aberto | Definir ordenação quando houver múltiplos workers | TRANSCRICAO | `[09:12] Diego`; `[09:13] Diego` |
| RFC-OPEN-03 | `docs/RFC.md` | Questão em aberto | Definir retenção e arquivamento da outbox em fase futura | TRANSCRICAO | `[09:08] Diego` |
| RFC-OPEN-04 | `docs/RFC.md` | Questão em aberto | Definir identidade do evento após replay de DLQ | TRANSCRICAO | `[09:18] Diego`; `[09:25] Diego` |
| RFC-SCOPE-01 | `docs/RFC.md` | Fora de escopo | Email como fallback fica para fase futura | TRANSCRICAO | `[09:37] Larissa` |
| RFC-SCOPE-02 | `docs/RFC.md` | Fora de escopo | Dashboard visual pertence a projeto separado | TRANSCRICAO | `[09:39] Marcos`; `[09:40] Larissa` |
| RFC-INT-01 | `docs/RFC.md` | Integração | Fluxo de mudança de status já é transacional | CODIGO | `src/modules/orders/order.service.ts` |
| RFC-PLAN-01 | `docs/RFC.md` | Planejamento | Estimar três sprints e dois dias úteis de revisão de segurança | TRANSCRICAO | `[09:46] Larissa`; `[09:46] Sofia`; `[09:47] Larissa` |
