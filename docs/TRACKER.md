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
| FDD-FLOW-01 | `docs/FDD.md` | Fluxo | Criar snapshot na outbox dentro da transação de mudança de status | TRANSCRICAO | `[09:40] Bruno`; `[09:41] Diego`; `[09:52] Larissa` |
| FDD-FLOW-01-FILTER | `docs/FDD.md` | Regra | Criar outbox somente para endpoints que assinam o novo status | TRANSCRICAO | `[09:33] Marcos`; `[09:34] Bruno` |
| FDD-FLOW-01-SIZE | `docs/FDD.md` | Restrição | Rejeitar payload acima de 64 KB sem truncar | TRANSCRICAO | `[09:23] Sofia`; `[09:24] Larissa` |
| FDD-FLOW-02 | `docs/FDD.md` | Fluxo | Worker busca pendentes em lote pequeno e ordem de criação | TRANSCRICAO | `[09:08] Diego`; `[09:09] Diego` |
| FDD-FLOW-02-PROC | `docs/FDD.md` | Integração | Worker abre um PrismaClient próprio por ser outro processo | TRANSCRICAO | `[09:29] Diego`; `[09:30] Bruno` |
| FDD-FLOW-03 | `docs/FDD.md` | Resiliência | Aplicar backoff 1m/5m/30m/2h/12h | TRANSCRICAO | `[09:17] Diego`; `[09:17] Larissa` |
| FDD-FLOW-04 | `docs/FDD.md` | Fluxo | Persistir falha permanente em DLQ separada | TRANSCRICAO | `[09:17] Larissa`; `[09:18] Diego` |
| FDD-FLOW-05 | `docs/FDD.md` | Fluxo | Replay recoloca evento na outbox, exige ADMIN e auditoria | TRANSCRICAO | `[09:18] Diego`; `[09:36] Sofia` |
| FDD-FLOW-06 | `docs/FDD.md` | Segurança | Rotacionar secret com sobreposição de 24 horas | TRANSCRICAO | `[09:21] Sofia`; `[09:22] Sofia` |
| FDD-PAYLOAD-01 | `docs/FDD.md` | Contrato outbound | Payload contém IDs, tipo, timestamp, status e dados básicos sem items | TRANSCRICAO | `[09:43] Diego`; `[09:44] Bruno` |
| FDD-HEADER-01 | `docs/FDD.md` | Contrato outbound | Enviar Event ID, Signature, Timestamp, Webhook ID e Content-Type | TRANSCRICAO | `[09:44] Diego`; `[09:44] Sofia` |
| FDD-API-01 | `docs/FDD.md` | Contrato HTTP | Criar configuração com URL, cliente, secret gerada e filtros | TRANSCRICAO | `[09:31] Marcos`; `[09:32] Larissa` |
| FDD-API-02 | `docs/FDD.md` | Contrato HTTP | Listar webhooks de um cliente autenticado | TRANSCRICAO | `[09:32] Larissa`; `[09:33] Bruno` |
| FDD-API-03 | `docs/FDD.md` | Contrato HTTP | Editar URL, filtros e estado da configuração | TRANSCRICAO | `[09:21] Bruno`; `[09:33] Bruno` |
| FDD-API-04 | `docs/FDD.md` | Contrato HTTP | Remover configuração de webhook | TRANSCRICAO | `[09:33] Bruno` |
| FDD-API-05 | `docs/FDD.md` | Contrato HTTP | Rotacionar secret pela API | TRANSCRICAO | `[09:21] Sofia`; `[09:22] Sofia` |
| FDD-API-06 | `docs/FDD.md` | Contrato HTTP | Consultar as últimas 100 entregas com resultado e duração | TRANSCRICAO | `[09:34] Marcos` |
| FDD-API-07 | `docs/FDD.md` | Contrato HTTP | Reprocessar DLQ por endpoint administrativo | TRANSCRICAO | `[09:18] Diego`; `[09:35] Diego` |
| FDD-ERR-01 | `docs/FDD.md` | Padrão de erro | Usar códigos específicos com prefixo `WEBHOOK_` | TRANSCRICAO | `[09:28] Bruno`; `[09:29] Larissa` |
| FDD-RES-01 | `docs/FDD.md` | Resiliência | Timeout HTTP de 10 segundos gera falha e retry | TRANSCRICAO | `[09:42] Sofia`; `[09:42] Diego` |
| FDD-RES-02 | `docs/FDD.md` | Resiliência | Manter Event ID estável durante as tentativas do evento | TRANSCRICAO | `[09:24] Diego`; `[09:25] Diego` |
| FDD-OBS-01 | `docs/FDD.md` | Observabilidade | Reutilizar Pino para logs estruturados e correlacionados | TRANSCRICAO | `[09:29] Bruno`; `[09:30] Larissa` |
| FDD-OBS-02 | `docs/FDD.md` | Observabilidade | Monitorar backlog, latência, retries e DLQ | TRANSCRICAO | `[09:07] Bruno`; `[09:08] Diego`; `[09:38] Diego` |
| FDD-INT-01 | `docs/FDD.md` | Integração | Estender a transação de `changeStatus` | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-02 | `docs/FDD.md` | Integração | Reutilizar estados e regras de transição do pedido | CODIGO | `src/modules/orders/order.status.ts` |
| FDD-INT-03 | `docs/FDD.md` | Integração | Adicionar modelos e índices seguindo as convenções Prisma | CODIGO | `prisma/schema.prisma` |
| FDD-INT-04 | `docs/FDD.md` | Integração | Compor controller e registrar as rotas do módulo | CODIGO | `src/app.ts`; `src/routes/index.ts` |
| FDD-INT-05 | `docs/FDD.md` | Integração | Reutilizar autenticação e autorização ADMIN | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-INT-06 | `docs/FDD.md` | Integração | Reutilizar validação, AppError e middleware central | CODIGO | `src/middlewares/validate.middleware.ts`; `src/shared/errors/app-error.ts`; `src/middlewares/error.middleware.ts` |
| FDD-INT-07 | `docs/FDD.md` | Integração | Reutilizar logger e ampliar redaction de credenciais | CODIGO | `src/shared/logger/index.ts` |
| FDD-INT-08 | `docs/FDD.md` | Integração | Espelhar bootstrap e shutdown no worker | CODIGO | `src/server.ts`; `src/config/database.ts` |
| FDD-TEST-01 | `docs/FDD.md` | Teste | Estender testes de pedidos para atomicidade e regressão | CODIGO | `tests/orders.test.ts` |
| PRD-OBJ-01 | `docs/PRD.md` | Objetivo | Substituir polling dos três clientes por notificação de mudança de status | TRANSCRICAO | `[09:00] Marcos` |
| PRD-METRIC-01 | `docs/PRD.md` | Métrica | Fazer primeira tentativa em menos de 10 segundos em condição normal | TRANSCRICAO | `[09:02] Marcos`; `[09:09] Diego` |
| PRD-METRIC-02 | `docs/PRD.md` | Métrica | Manter zero mudanças assinadas sem evento persistido | TRANSCRICAO | `[09:06] Diego`; `[09:40] Bruno` |
| PRD-FR-01 | `docs/PRD.md` | Requisito Funcional | Cadastrar webhook com cliente, URL e status de interesse | TRANSCRICAO | `[09:31] Marcos`; `[09:32] Larissa` |
| PRD-FR-02 | `docs/PRD.md` | Requisito Funcional | Gerar e devolver secret única na criação | TRANSCRICAO | `[09:21] Sofia`; `[09:31] Marcos` |
| PRD-FR-03 | `docs/PRD.md` | Requisito Funcional | Listar, editar e remover configurações | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-04 | `docs/PRD.md` | Requisito Funcional | Filtrar eventos pelo novo status na inserção | TRANSCRICAO | `[09:33] Marcos`; `[09:34] Bruno` |
| PRD-FR-05 | `docs/PRD.md` | Requisito Funcional | Preservar snapshot do pedido no instante da mudança | TRANSCRICAO | `[09:51] Bruno`; `[09:52] Larissa` |
| PRD-FR-06 | `docs/PRD.md` | Requisito Funcional | Entregar JSON assinado com IDs e timestamp | TRANSCRICAO | `[09:43] Diego`; `[09:44] Diego`; `[09:44] Sofia` |
| PRD-FR-07 | `docs/PRD.md` | Requisito Funcional | Retentar em 1m/5m/30m/2h/12h | TRANSCRICAO | `[09:17] Diego`; `[09:17] Larissa` |
| PRD-FR-08 | `docs/PRD.md` | Requisito Funcional | Mover falha permanente para DLQ separada | TRANSCRICAO | `[09:17] Larissa`; `[09:18] Diego` |
| PRD-FR-09 | `docs/PRD.md` | Requisito Funcional | Permitir replay manual por ADMIN com auditoria | TRANSCRICAO | `[09:18] Diego`; `[09:36] Sofia` |
| PRD-FR-10 | `docs/PRD.md` | Requisito Funcional | Consultar as últimas 100 entregas e seus resultados | TRANSCRICAO | `[09:34] Marcos` |
| PRD-FR-11 | `docs/PRD.md` | Requisito Funcional | Rotacionar secret com 24h de sobreposição | TRANSCRICAO | `[09:21] Sofia`; `[09:22] Sofia` |
| PRD-FR-12 | `docs/PRD.md` | Requisito Funcional | Manter Event ID nas retentativas para deduplicação | TRANSCRICAO | `[09:24] Diego`; `[09:25] Diego` |
| PRD-NFR-01 | `docs/PRD.md` | Requisito Não Funcional | Entregar em menos de 10 segundos em condição normal | TRANSCRICAO | `[09:02] Marcos`; `[09:10] Marcos` |
| PRD-NFR-02 | `docs/PRD.md` | Requisito Não Funcional | Tornar mudança de status e outbox atômicas | TRANSCRICAO | `[09:06] Diego`; `[09:41] Diego` |
| PRD-NFR-03 | `docs/PRD.md` | Requisito Não Funcional | Oferecer garantia at-least-once | TRANSCRICAO | `[09:24] Diego`; `[09:26] Larissa` |
| PRD-NFR-04 | `docs/PRD.md` | Requisito Não Funcional | Aceitar somente endpoint HTTPS | TRANSCRICAO | `[09:23] Sofia` |
| PRD-NFR-05 | `docs/PRD.md` | Requisito Não Funcional | Assinar corpo com HMAC-SHA256 e secret isolada | TRANSCRICAO | `[09:20] Sofia`; `[09:21] Sofia` |
| PRD-NFR-06 | `docs/PRD.md` | Requisito Não Funcional | Limitar payload a 64 KB sem truncamento | TRANSCRICAO | `[09:23] Sofia`; `[09:24] Larissa` |
| PRD-NFR-07 | `docs/PRD.md` | Requisito Não Funcional | Aplicar timeout HTTP de 10 segundos | TRANSCRICAO | `[09:42] Diego` |
| PRD-NFR-08 | `docs/PRD.md` | Requisito Não Funcional | Não expor credenciais em logs | TRANSCRICAO | `[09:22] Diego`; `[09:29] Bruno` |
| PRD-NFR-09 | `docs/PRD.md` | Limitação | Ordem por pedido depende de worker único | TRANSCRICAO | `[09:12] Diego`; `[09:13] Larissa` |
| PRD-NFR-10 | `docs/PRD.md` | Requisito Não Funcional | Reutilizar stack e padrões atuais | TRANSCRICAO | `[09:27] Bruno`; `[09:30] Larissa` |
| PRD-SCOPE-01 | `docs/PRD.md` | Fora de escopo | Email como fallback fica para outra fase | TRANSCRICAO | `[09:37] Larissa` |
| PRD-SCOPE-02 | `docs/PRD.md` | Fora de escopo | Dashboard visual fica em projeto separado | TRANSCRICAO | `[09:39] Marcos`; `[09:40] Larissa` |
| PRD-SCOPE-03 | `docs/PRD.md` | Fora de escopo | Rate limiting será observado antes de decidido | TRANSCRICAO | `[09:38] Diego`; `[09:39] Larissa` |
| PRD-RISK-01 | `docs/PRD.md` | Risco | Endpoint lento pode acumular backlog | TRANSCRICAO | `[09:07] Bruno`; `[09:42] Diego` |
| PRD-RISK-02 | `docs/PRD.md` | Risco | Cliente pode processar duplicata | TRANSCRICAO | `[09:24] Diego`; `[09:25] Sofia` |
| PRD-RISK-03 | `docs/PRD.md` | Risco | Vazamento de secret compromete um endpoint | TRANSCRICAO | `[09:21] Sofia`; `[09:22] Diego` |
| PRD-RISK-04 | `docs/PRD.md` | Risco | Crescimento da outbox pode degradar o banco | TRANSCRICAO | `[09:07] Bruno`; `[09:08] Diego` |
| PRD-DEP-01 | `docs/PRD.md` | Dependência | Depender do fluxo transacional e dos estados atuais de pedido | CODIGO | `src/modules/orders/order.service.ts`; `src/modules/orders/order.status.ts` |
| PRD-DEP-02 | `docs/PRD.md` | Dependência | Depender de JWT e papel ADMIN existentes | CODIGO | `src/middlewares/auth.middleware.ts` |
| PRD-TEST-01 | `docs/PRD.md` | Validação | Preservar regressão de transições, estoque e histórico | CODIGO | `tests/orders.test.ts` |
