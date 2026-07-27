# ADR-004 — Autenticação HMAC e rotação de secret

## Status

Aceito

## Contexto

Os webhooks transportam dados de pedidos para endpoints fora da infraestrutura do OMS. O consumidor precisa verificar a origem e a integridade do corpo recebido. Uma credencial global ampliaria o impacto de qualquer vazamento.

## Decisão

Assinar os bytes do corpo JSON com HMAC-SHA256 usando uma secret única por endpoint. Enviar a assinatura no header `X-Signature`.

O OMS gerará e retornará a secret na criação do endpoint. Uma operação autenticada permitirá rotacioná-la. Após a rotação, a secret anterior continuará válida por 24 horas e então será invalidada.

Somente URLs HTTPS serão aceitas. Secrets e assinaturas deverão ser redigidas de logs. O payload máximo será 64 KB; eventos acima do limite serão tratados como erro, sem truncamento.

## Alternativas consideradas

### Secret global

Descartada porque o vazamento de uma credencial comprometeria todos os clientes.

### Payload sem assinatura

Descartado porque o cliente não conseguiria validar origem e integridade.

### Invalidação imediata na rotação

Descartada porque não daria ao cliente tempo para atualizar seus consumidores sem interrupção.

## Consequências

### Positivas

- Cada consumidor pode autenticar e verificar o payload.
- O impacto de vazamento fica restrito a um endpoint.
- O período de 24 horas permite rotação sem interrupção abrupta.
- HTTPS protege os dados em trânsito.

### Negativas

- O cliente precisa armazenar e validar secrets com segurança.
- Durante 24 horas existem duas secrets válidas.
- A assinatura depende dos bytes exatos do corpo; qualquer nova serialização no consumidor invalida a verificação.
- A implementação precisa impedir exposição de secrets em logs e respostas posteriores.

## Referências

- `TRANSCRICAO.md`: `[09:19] Sofia` a `[09:24] Larissa`.
- Código: `src/shared/logger/index.ts` e `src/modules/orders/order.schemas.ts`.

