# FDD — Sistema de Webhooks de Notificação de Pedidos

## 1. Contexto e motivação técnica

O OMS precisa emitir webhooks outbound quando o status de um pedido muda. A implementação deve preservar a transação existente de pedidos, desacoplar chamadas externas e seguir os padrões Node.js, TypeScript, Express, Prisma, Zod, AppError e Pino da aplicação.

Este documento detalha como construir a solução proposta no [RFC](RFC.md) e formalizada nos [ADRs](adrs/).

## 2. Objetivos técnicos

- Persistir atomicamente pedido, histórico, estoque e eventos aplicáveis.
- Entregar eventos em processo separado, por polling de 2 segundos.
- Autenticar o corpo com HMAC-SHA256 e secret por endpoint.
- Tolerar falhas com timeout, cinco retentativas e DLQ.
- Expor CRUD, rotação de secret, histórico e replay administrativo.
- Oferecer entrega at-least-once com identificador deduplicável.
- Integrar-se aos padrões atuais sem introduzir broker ou nova stack.

## 3. Escopo e exclusões

### Incluído

- Configuração de webhooks por cliente e filtro por status de pedido.
- Snapshot do evento na outbox.
- Worker único em processo separado.
- Registro das últimas entregas, incluindo resposta e duração.
- Rotação de secret com 24 horas de sobreposição.
- Replay manual da DLQ por ADMIN.
- Métricas, logs e propagação de contexto de tracing.

### Excluído

- Recepção de webhooks.
- Email como fallback.
- Dashboard.
- Rate limiting de saída.
- Múltiplos workers e garantia de ordenação nesse cenário.
- Retenção e arquivamento automáticos.
- Alteração de código como parte desta entrega documental.

## 4. Componentes

| Componente proposto | Responsabilidade |
| --- | --- |
| `WebhookController` | Traduzir requests e responses do CRUD, rotação, deliveries e replay |
| `WebhookService` | Aplicar regras de configuração e autorização |
| `WebhookRepository` | Persistir configurações, histórico, outbox e DLQ |
| `webhook.schemas.ts` | Validar UUIDs, HTTPS, filtros e payloads da API |
| `publishWebhookEvent` | Criar snapshots aplicáveis usando o `TransactionClient` corrente |
| `WebhookProcessor` | Buscar pendências, assinar, enviar, registrar resultado e reagendar |
| `src/worker.ts` | Inicializar Prisma, processor, loop, sinais e encerramento |

## 5. Modelo de dados proposto

Os nomes abaixo descrevem o desenho necessário; a implementação final deverá seguir as convenções Prisma de `prisma/schema.prisma`.

### `WebhookEndpoint`

| Campo | Finalidade |
| --- | --- |
| `id: UUID` | Identificador do cadastro e valor de `X-Webhook-Id` |
| `customerId: UUID` | Cliente proprietário |
| `url: string` | Endpoint HTTPS |
| `secretHash/encryptedSecret` | Secret protegida; a estratégia de armazenamento deve ser revisada por Segurança |
| `previousSecret*` | Secret anterior e expiração do grace period de 24h |
| `subscribedStatuses` | Status que geram eventos |
| `active: boolean` | Habilita ou desabilita a emissão |
| timestamps | Criação e atualização |

> A reunião exige secret por endpoint e rotação, mas não definiu criptografia em repouso. A estratégia exata é uma decisão de implementação sujeita à revisão de Segurança; a secret não pode ser recuperada por endpoints de leitura.

### `WebhookOutbox`

| Campo | Finalidade |
| --- | --- |
| `id: UUID` | Identificador operacional |
| `eventId: UUID` | Identidade pública e deduplicável do evento |
| `webhookId: UUID` | Destino configurado |
| `orderId: UUID` | Pedido que originou o evento |
| `payload: JSON` | Snapshot imutável |
| `status` | `PENDING`, `PROCESSING`, `DELIVERED` ou `FAILED` |
| `attemptCount` | Tentativas executadas |
| `nextAttemptAt` | Elegibilidade para polling |
| `createdAt/updatedAt` | Ordenação e operação |

Índices mínimos: `(status, nextAttemptAt, createdAt)`, `createdAt`, `orderId` e `eventId`.

### `WebhookDelivery`

| Campo | Finalidade |
| --- | --- |
| `id: UUID` | Identificador da tentativa |
| `eventId/webhookId` | Correlação |
| `attemptNumber` | Número da tentativa |
| `requestPayload` | Evidência do corpo enviado |
| `responseStatus/responseBody` | Resultado limitado e seguro |
| `durationMs` | Tempo da chamada |
| `errorCode/errorMessage` | Falha normalizada |
| `createdAt` | Ordenação do histórico |

### `WebhookDeadLetter`

| Campo | Finalidade |
| --- | --- |
| `id: UUID` | Identificador da DLQ |
| `eventId/webhookId/orderId` | Correlação com o evento |
| `payload` | Snapshot que falhou |
| `failureReason` | Motivo final |
| `failedAt` | Momento da falha permanente |
| `replayedAt/replayedById` | Auditoria de replay, quando houver |

## 6. Fluxos detalhados

### FDD-FLOW-01 — Criação do evento na outbox

1. `OrderService.changeStatus` abre a transação Prisma já existente.
2. Carrega o pedido, valida a transição e executa débito/reposição de estoque quando aplicável.
3. Atualiza `orders` e insere `order_status_history`.
4. Chama `publishWebhookEvent(tx, orderSnapshot, fromStatus, toStatus)`.
5. A função busca endpoints ativos do `customerId` que assinam `toStatus`.
6. Se não houver assinantes, não cria evento.
7. Para cada endpoint aplicável, monta um snapshot com UUID e timestamp.
8. Serializa uma vez e rejeita o evento se exceder 64 KB.
9. Insere a linha da outbox usando o mesmo `tx`.
10. Qualquer falha propaga erro e provoca rollback de pedido, histórico, estoque e outbox.

O snapshot deve conter:

```json
{
  "event_id": "3d438f28-29c5-4c28-9f08-d21afc27de72",
  "event_type": "order.status_changed",
  "timestamp": "2026-07-27T15:30:00.000Z",
  "order_id": "7205034c-60b8-4221-b93b-d5ec8e020415",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "281dc70d-7077-4b93-a99b-4fcbfb6f80be",
  "total_cents": 25990
}
```

Itens do pedido não são incluídos. O consumidor usa `GET /orders/:id` quando precisar de detalhes.

### FDD-FLOW-02 — Polling e processamento

1. `src/worker.ts` cria seu próprio `PrismaClient`.
2. O loop aguarda 2 segundos entre ciclos.
3. Seleciona um lote pequeno de eventos `PENDING` ou elegíveis para retry, ordenado por `createdAt`.
4. Marca o item como `PROCESSING` antes da chamada para reduzir reprocessamento acidental.
5. Carrega a configuração e a secret válida.
6. Usa exatamente os bytes do snapshot serializado para calcular HMAC-SHA256.
7. Faz POST HTTPS com timeout de 10 segundos.
8. Registra uma linha de delivery com status, resposta limitada e duração.
9. Em sucesso, marca a outbox como `DELIVERED`.
10. Em falha, aplica o fluxo de retry.

Um response HTTP `2xx` representa sucesso. Timeout, erro de rede e response fora de `2xx` representam falha de entrega.

### FDD-FLOW-03 — Retry

Após cada falha, incrementar `attemptCount` e definir `nextAttemptAt`:

| Retentativa | Intervalo após a falha |
| --- | --- |
| 1 | 1 minuto |
| 2 | 5 minutos |
| 3 | 30 minutos |
| 4 | 2 horas |
| 5 | 12 horas |

A chamada inicial não integra a contagem de cinco retentativas. Portanto, um evento pode ter até seis chamadas: uma inicial e cinco retries.

Se ainda houver retentativa, voltar o evento para `PENDING` com `nextAttemptAt`. Não alterar `event_id`, payload ou timestamp do evento.

### FDD-FLOW-04 — DLQ

Depois da quinta retentativa malsucedida:

1. Inserir `WebhookDeadLetter` com payload, correlações, motivo e timestamp.
2. Marcar a outbox como `FAILED` ou removê-la apenas dentro da mesma transação da DLQ.
3. Emitir métrica e log de falha permanente.
4. Manter o item disponível para consulta e replay administrativo.

### FDD-FLOW-05 — Replay manual

1. Autenticar o JWT.
2. Exigir `requireRole('ADMIN')`.
3. Localizar a entrada da DLQ.
4. Impedir replay concorrente ou repetido sem confirmação do estado.
5. Criar nova entrada pendente na outbox e registrar `replayedById/replayedAt` atomicamente.
6. Registrar log de auditoria com usuário, DLQ e resultado.

**Questão bloqueante do contrato:** antes da implementação, decidir se a nova entrada preserva o `event_id` original ou recebe outro. A transcrição não fechou essa semântica.

### FDD-FLOW-06 — Rotação de secret

1. Autenticar o usuário.
2. Localizar a configuração.
3. Gerar uma nova secret forte.
4. Tornar a secret atual válida como anterior até `now + 24h`.
5. Persistir a nova secret e retornar seu valor uma única vez.
6. Após o grace period, não aceitar/usar a anterior e removê-la conforme a estratégia de armazenamento.

## 7. Contratos públicos

Todos os endpoints ficam sob `/api/v1`, exigem `Authorization: Bearer <JWT>` e usam o envelope de erro já adotado pela aplicação.

A reunião deixou `customer_id` “no body ou no path”. Para manter coerência com `createOrderSchema` e `listOrdersQuerySchema`, este FDD adota `customerId` no body de criação e na query de listagem.

### FDD-API-01 — Criar webhook

`POST /api/v1/webhooks`

Request:

```json
{
  "customerId": "281dc70d-7077-4b93-a99b-4fcbfb6f80be",
  "url": "https://integracao.atlas.example/webhooks/orders",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"]
}
```

Response `201 Created`:

```json
{
  "id": "92c277ad-c4b4-487b-a7f6-0cf1a9f2bf03",
  "customerId": "281dc70d-7077-4b93-a99b-4fcbfb6f80be",
  "url": "https://integracao.atlas.example/webhooks/orders",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_<valor-mostrado-uma-vez>",
  "createdAt": "2026-07-27T15:00:00.000Z"
}
```

Erros: `400 WEBHOOK_INVALID_URL`, `400 WEBHOOK_INVALID_STATUS_FILTER`, `404 WEBHOOK_CUSTOMER_NOT_FOUND`.

### FDD-API-02 — Listar webhooks

`GET /api/v1/webhooks?customerId=<uuid>`

Response `200 OK`:

```json
{
  "data": [
    {
      "id": "92c277ad-c4b4-487b-a7f6-0cf1a9f2bf03",
      "customerId": "281dc70d-7077-4b93-a99b-4fcbfb6f80be",
      "url": "https://integracao.atlas.example/webhooks/orders",
      "subscribedStatuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-07-27T15:00:00.000Z",
      "updatedAt": "2026-07-27T15:00:00.000Z"
    }
  ],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "total": 1,
    "totalPages": 1
  }
}
```

Secrets nunca aparecem em listagens.

### FDD-API-03 — Atualizar webhook

`PATCH /api/v1/webhooks/:id`

Request:

```json
{
  "url": "https://integracao.atlas.example/v2/orders",
  "subscribedStatuses": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true
}
```

Response `200 OK`:

```json
{
  "id": "92c277ad-c4b4-487b-a7f6-0cf1a9f2bf03",
  "customerId": "281dc70d-7077-4b93-a99b-4fcbfb6f80be",
  "url": "https://integracao.atlas.example/v2/orders",
  "subscribedStatuses": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true,
  "updatedAt": "2026-07-27T16:00:00.000Z"
}
```

Erros: `404 WEBHOOK_NOT_FOUND`, `400 WEBHOOK_INVALID_URL`, `400 WEBHOOK_INVALID_STATUS_FILTER`.

### FDD-API-04 — Remover webhook

`DELETE /api/v1/webhooks/:id`

Response `204 No Content`, sem corpo.

Erro: `404 WEBHOOK_NOT_FOUND`.

### FDD-API-05 — Rotacionar secret

`POST /api/v1/webhooks/:id/rotate-secret`

Request:

```json
{}
```

Response `200 OK`:

```json
{
  "webhookId": "92c277ad-c4b4-487b-a7f6-0cf1a9f2bf03",
  "secret": "whsec_<novo-valor-mostrado-uma-vez>",
  "previousSecretValidUntil": "2026-07-28T16:00:00.000Z"
}
```

Erro: `404 WEBHOOK_NOT_FOUND`.

### FDD-API-06 — Consultar entregas

`GET /api/v1/webhooks/:id/deliveries`

Response `200 OK` com no máximo as 100 entregas mais recentes:

```json
{
  "data": [
    {
      "id": "a965a70b-e342-4cf8-b9eb-4152846646ba",
      "eventId": "3d438f28-29c5-4c28-9f08-d21afc27de72",
      "attemptNumber": 1,
      "status": "DELIVERED",
      "requestPayload": {
        "event_type": "order.status_changed",
        "order_id": "7205034c-60b8-4221-b93b-d5ec8e020415"
      },
      "responseStatus": 200,
      "responseBody": "accepted",
      "durationMs": 184,
      "createdAt": "2026-07-27T15:30:00.184Z"
    }
  ]
}
```

Erro: `404 WEBHOOK_NOT_FOUND`.

### FDD-API-07 — Replay de DLQ

`POST /api/v1/admin/webhooks/dead-letter/:id/replay`

Request:

```json
{}
```

Response `202 Accepted`:

```json
{
  "deadLetterId": "54444953-94da-4b07-8d70-7bcba9d686f5",
  "status": "REPLAY_QUEUED"
}
```

Erros: `403 WEBHOOK_REPLAY_FORBIDDEN`, `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`, `409 WEBHOOK_REPLAY_CONFLICT`.

## 8. Contrato outbound

Request do worker:

```http
POST /webhooks/orders HTTP/1.1
Host: integracao.atlas.example
Content-Type: application/json
X-Event-Id: 3d438f28-29c5-4c28-9f08-d21afc27de72
X-Webhook-Id: 92c277ad-c4b4-487b-a7f6-0cf1a9f2bf03
X-Timestamp: 2026-07-27T15:30:00.000Z
X-Signature: <hmac-sha256-do-corpo>
```

O corpo é o snapshot de `FDD-FLOW-01`. O HMAC deve usar os mesmos bytes transmitidos, sem nova serialização após a assinatura.

## 9. Matriz de erros

| Código | HTTP | Condição |
| --- | ---: | --- |
| `WEBHOOK_INVALID_URL` | 400 | URL ausente, inválida ou sem HTTPS |
| `WEBHOOK_INVALID_STATUS_FILTER` | 400 | Lista vazia ou status inexistente |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Snapshot excede 64 KB |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | Cliente informado não existe |
| `WEBHOOK_NOT_FOUND` | 404 | Configuração não encontrada |
| `WEBHOOK_SECRET_GENERATION_FAILED` | 500 | Não foi possível gerar secret |
| `WEBHOOK_DELIVERY_TIMEOUT` | interno | Endpoint não respondeu em 10s |
| `WEBHOOK_DELIVERY_FAILED` | interno | Rede ou response fora de `2xx` |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Entrada de DLQ não encontrada |
| `WEBHOOK_REPLAY_FORBIDDEN` | 403 | Usuário não é ADMIN |
| `WEBHOOK_REPLAY_CONFLICT` | 409 | Replay incompatível com o estado atual |
| `WEBHOOK_OUTBOX_WRITE_FAILED` | interno | Falha que provoca rollback da mudança |

O middleware atual usa códigos genéricos para autenticação e validação Zod. O módulo deverá mapear falhas próprias para subclasses de `AppError` com prefixo `WEBHOOK_`, sem alterar o comportamento global não relacionado.

## 10. Estratégias de resiliência

| Mecanismo | Regra |
| --- | --- |
| Atomicidade | Outbox no mesmo `Prisma.TransactionClient` de `changeStatus` |
| Timeout | 10 segundos por chamada HTTP |
| Retry | Cinco retentativas em 1m/5m/30m/2h/12h |
| DLQ | Persistência separada após esgotar retries |
| Idempotência | Mesmo `event_id` em todas as tentativas do evento |
| Fallback | DLQ e replay manual; email não faz parte da fase |
| Ordenação | Por criação, por pedido, somente com worker único |
| Limite | Snapshot de no máximo 64 KB; falhar sem truncar |
| Credenciais | Secret por endpoint e rotação com 24h de sobreposição |

O worker deve encerrar graciosamente em `SIGINT` e `SIGTERM`, parando novos polls, concluindo ou liberando de forma segura o item corrente e desconectando o Prisma.

## 11. Observabilidade

### Métricas

- `webhook_outbox_pending_total` — backlog elegível.
- `webhook_oldest_pending_age_seconds` — idade do evento pendente mais antigo.
- `webhook_delivery_total{result,status_code}` — entregas por resultado.
- `webhook_delivery_duration_seconds` — histograma de duração.
- `webhook_retry_scheduled_total{attempt}` — retries agendados.
- `webhook_dead_letter_total` — entradas na DLQ.
- `webhook_replay_total{result}` — replays administrativos.
- `webhook_payload_bytes` — distribuição do tamanho dos snapshots.

### Logs

Usar Pino com eventos estruturados como `webhook_delivery_started`, `webhook_delivery_succeeded`, `webhook_retry_scheduled`, `webhook_moved_to_dlq` e `webhook_replay_requested`.

Campos de correlação: `eventId`, `webhookId`, `orderId`, `attempt`, `durationMs`, `responseStatus`, `requestId` e, no replay, `userId`. Nunca registrar secret, `X-Signature`, Authorization ou corpo de resposta sem limitação/redação.

### Tracing

Propagar o contexto disponível da alteração de status para o registro da outbox e criar um span por tentativa no worker. Como a execução é assíncrona, relacionar os spans por `eventId`/trace context persistido, cobrindo:

```text
orders.change_status
  └── webhooks.outbox_insert

webhooks.poll
  └── webhooks.delivery_attempt
```

A biblioteca de tracing não foi definida na reunião; a implementação deve integrar-se à solução adotada pelo projeto quando disponível, sem bloquear métricas e logs.

## 12. Integração com o sistema existente

| Caminho real | Integração necessária |
| --- | --- |
| `src/modules/orders/order.service.ts` | Dentro de `changeStatus`, após validar/aplicar a transição e ainda no `$transaction`, chamar `publishWebhookEvent(tx, ...)`. Falha da outbox deve abortar toda a transação. |
| `src/modules/orders/order.status.ts` | Reutilizar `OrderStatus` e as transições existentes para validar filtros e preencher `from_status`/`to_status`. |
| `prisma/schema.prisma` | Adicionar modelos de endpoint, outbox, delivery e DLQ com UUIDs, relações e índices; manter MySQL e convenções de nomes. |
| `src/app.ts` | Compor repository, service e controller de webhooks junto aos controllers atuais. |
| `src/routes/index.ts` | Registrar rotas `/webhooks` e `/admin/webhooks` sob `/api/v1`. |
| `src/middlewares/auth.middleware.ts` | Aplicar `authenticate` a todos os endpoints e `requireRole('ADMIN')` ao replay. |
| `src/middlewares/validate.middleware.ts` | Validar params, query e body com schemas Zod do módulo. |
| `src/middlewares/error.middleware.ts` | Reutilizar serialização de `AppError`, Zod e Prisma. |
| `src/shared/errors/app-error.ts` | Criar erros específicos do módulo com códigos `WEBHOOK_*`. |
| `src/shared/logger/index.ts` | Reutilizar Pino e ampliar redaction para secrets/assinaturas do módulo. |
| `src/config/database.ts` | API mantém o client atual; o processo worker chama `createPrismaClient()` e possui lifecycle separado. |
| `src/server.ts` | Espelhar bootstrap, captura de sinais e encerramento na nova entry point `src/worker.ts`. |
| `tests/orders.test.ts` | Estender testes de mudança de status para provar criação/rollback da outbox e manter regressão de estoque/histórico. |

## 13. Dependências e compatibilidade

- MySQL e Prisma atuais.
- Node.js/TypeScript e processo adicional para o worker.
- Express, Zod, JWT, AppError e Pino existentes.
- Cliente HTTP com suporte a timeout e acesso aos bytes do corpo; a biblioteca não foi decidida.
- Compatibilidade com todos os valores de `OrderStatus` atuais.
- Consumidores precisam aceitar JSON, validar HMAC-SHA256 e deduplicar UUID.

## 14. Estratégia de testes

### Unidade

- Serialização determinística e HMAC sobre bytes exatos.
- Geração de secret e janela de rotação.
- Cálculo de backoff.
- Validação de HTTPS, status e 64 KB.
- Classificação de responses `2xx`, não `2xx`, timeout e rede.

### Integração

- CRUD autenticado e respostas `WEBHOOK_*`.
- Replay negado a OPERATOR e permitido a ADMIN.
- Filtro evita outbox para status não assinado.
- Alteração de status e outbox fazem commit juntas.
- Falha simulada de outbox reverte status, histórico e estoque.
- Worker registra delivery, retry e DLQ.

### Ponta a ponta

- Mudar pedido e receber payload/headers válidos em servidor HTTP de teste.
- Simular timeout e verificar os cinco agendamentos.
- Simular confirmação perdida e provar que o `X-Event-Id` não muda.
- Rotacionar secret e verificar a sobreposição de 24 horas.
- Fazer replay da DLQ e confirmar auditoria.

### Segurança

- Rejeitar HTTP, assinatura adulterada e filtros inválidos.
- Confirmar ausência de secrets nos logs e endpoints de leitura.
- Revisar HMAC e geração/armazenamento de secret por pelo menos dois dias úteis antes do deploy.

## 15. Critérios de aceite técnicos

- [ ] Mudança assinada e outbox confirmam ou revertem juntas.
- [ ] Polling ocorre a cada 2 segundos em processo separado.
- [ ] Entrega normal permanece abaixo de 10 segundos em teste controlado.
- [ ] Payload possui os campos acordados, não contém items e respeita 64 KB.
- [ ] Headers incluem Event ID, Webhook ID, timestamp, assinatura e content type.
- [ ] Timeout de 10 segundos gera retry.
- [ ] Cinco retries seguem exatamente a progressão definida e terminam na DLQ.
- [ ] Retentativas do evento mantêm o mesmo Event ID.
- [ ] CRUD, rotação, deliveries e replay seguem os contratos.
- [ ] Replay exige ADMIN e produz auditoria.
- [ ] Secrets não aparecem em listagens nem logs.
- [ ] Métricas, logs e correlação por evento estão disponíveis.
- [ ] Testes atuais de pedidos continuam passando.

## 16. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Outbox crescer e degradar polling | Média | Alto | Índices, lotes pequenos, métricas de backlog e política futura de retenção |
| Endpoint lento ocupar o worker único | Alta | Alto | Timeout de 10s, backoff e alertas de latência/backlog |
| Duplicata processada pelo cliente | Média | Médio | Event ID estável e documentação de deduplicação |
| Secret exposta | Baixa | Alto | HTTPS, redaction, retorno único, rotação e revisão de Segurança |
| Corrida ao recuperar itens `PROCESSING` após crash | Média | Alto | Claim transacional e política explícita de recuperação na implementação |
| Ambiguidade do Event ID no replay | Média | Médio | Fechar RFC-OPEN-04 antes de implementar o endpoint |

## 17. Questões pendentes antes da implementação

1. Definir se replay preserva ou renova o `event_id`.
2. Definir estratégia de proteção da secret em repouso.
3. Definir claim/lease e recuperação de item `PROCESSING` após crash.
4. Escolher cliente HTTP e mecanismo de tracing compatíveis com a stack.
5. Definir limite/redação do corpo de resposta salvo no histórico.

## 18. ADRs relacionados

- [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md)
- [ADR-002](adrs/ADR-002-worker-separado-com-polling.md)
- [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md)
- [ADR-004](adrs/ADR-004-autenticacao-hmac-e-rotacao-de-secret.md)
- [ADR-005](adrs/ADR-005-entrega-at-least-once-com-event-id.md)
- [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)
