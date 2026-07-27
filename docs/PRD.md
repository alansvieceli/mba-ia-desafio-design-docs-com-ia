# PRD — Sistema de Webhooks de Notificação de Pedidos

## 1. Resumo e contexto

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — precisam receber notificações quando seus pedidos mudarem de status. Hoje, eles consultam repetidamente `GET /orders`, o que torna a integração lenta e cara. A Atlas indicou risco de migração para um concorrente se a capacidade não for entregue até o fim do trimestre.

Esta feature oferece webhooks outbound configuráveis por cliente. Em condições normais, a notificação deve chegar em menos de 10 segundos, sem fazer a operação de pedidos depender da disponibilidade do sistema do cliente.

## 2. Problema e motivação

### Problema

- Clientes precisam consultar a API periodicamente para descobrir mudanças.
- Polling do cliente gera atraso, custo e tráfego desnecessário.
- O OMS não possui mecanismo de notificação externa.
- Uma integração síncrona tornaria a mudança de status vulnerável a endpoints lentos ou indisponíveis.

### Motivação

- Atender uma demanda formal de três clientes B2B.
- Reduzir o esforço e a latência das integrações.
- Mitigar o risco comercial relatado pela Atlas.
- Criar uma base confiável para notificações outbound sem introduzir infraestrutura desproporcional.

## 3. Público-alvo e cenários de uso

### Público-alvo

- Clientes B2B integrados à API do OMS.
- Operadores autenticados que configuram webhooks para clientes.
- Administradores que diagnosticam e reprocessam falhas permanentes.
- Times internos de Pedidos, Plataforma, Segurança e Suporte.

### Cenários

1. Um cliente cadastra uma URL HTTPS e escolhe os status de interesse.
2. Um pedido muda para `SHIPPED`; o cliente recebe o evento em menos de 10 segundos em condição normal.
3. O endpoint fica indisponível; o OMS retenta sem reverter a mudança do pedido.
4. Após esgotar as tentativas, um ADMIN diagnostica a DLQ e solicita replay.
5. Um cliente recebe uma duplicata e a ignora pelo Event ID.
6. Um cliente rotaciona sua secret sem interrupção abrupta.
7. Um usuário consulta as últimas 100 entregas para investigar sucesso, falha e duração.

## 4. Objetivos e métricas de sucesso

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Reduzir a espera por mudanças | Latência entre mudança confirmada e primeira tentativa, em condição normal | Menos de 10 segundos |
| Preservar consistência | Mudanças confirmadas e assinadas por webhook ativo sem evento persistido | 0 |
| Permitir recuperação | Retentativas antes da falha permanente | 5, nos intervalos acordados |
| Dar visibilidade operacional | Histórico por configuração | 100 entregas mais recentes |
| Permitir rotação segura | Sobreposição entre secret nova e anterior | 24 horas |

Não há meta de disponibilidade, volume ou taxa de sucesso definida na reunião. Esses indicadores deverão ser medidos após o lançamento antes de receberem metas formais.

## 5. Escopo

### Incluído

- Webhooks outbound para mudanças de status de pedidos.
- Cadastro, listagem, edição e remoção de configurações.
- Filtro dos status recebidos por endpoint.
- Secret única, assinatura HMAC-SHA256 e rotação.
- Entrega desacoplada, at-least-once e deduplicável.
- Timeout, retentativas e DLQ.
- Histórico das últimas 100 entregas.
- Replay manual de DLQ por ADMIN, com auditoria.
- Payload enxuto, sem itens do pedido.

### Fora de escopo

- **Email como fallback:** explicitamente adiado para uma fase futura.
- **Dashboard visual:** pertence a um projeto separado do frontend.
- Webhooks inbound.
- Rate limiting de saída; será avaliado após observação do uso.
- Broker ou Redis.
- Escala horizontal do worker e ordenação nesse cenário.
- Arquivamento automático de eventos entregues após 30 dias.
- Garantia exactly-once.

## 6. Requisitos funcionais

| ID | Requisito |
| --- | --- |
| PRD-FR-01 | Usuário autenticado deve poder cadastrar um webhook para um cliente, informando URL HTTPS e status de interesse. |
| PRD-FR-02 | O sistema deve gerar uma secret única por endpoint e mostrá-la na criação. |
| PRD-FR-03 | Usuário autenticado deve poder listar, editar e remover webhooks de um cliente. |
| PRD-FR-04 | O sistema deve criar evento somente quando o novo status fizer parte do filtro de um endpoint ativo. |
| PRD-FR-05 | O evento deve representar o estado do pedido no momento exato da mudança. |
| PRD-FR-06 | O sistema deve entregar um JSON assinado com Event ID, Webhook ID e timestamp. |
| PRD-FR-07 | O sistema deve retentar entregas malsucedidas segundo `1m / 5m / 30m / 2h / 12h`. |
| PRD-FR-08 | Após as cinco retentativas, o sistema deve preservar a falha em DLQ separada. |
| PRD-FR-09 | ADMIN deve poder solicitar replay manual de uma DLQ, com auditoria do ator. |
| PRD-FR-10 | Usuário autenticado deve poder consultar as 100 entregas mais recentes de um webhook, com resultado, payload, response e duração. |
| PRD-FR-11 | Usuário autenticado deve poder rotacionar a secret; a anterior permanece válida por 24 horas. |
| PRD-FR-12 | Retentativas do mesmo evento devem manter o Event ID para permitir deduplicação pelo cliente. |

## 7. Requisitos não funcionais

| ID | Requisito |
| --- | --- |
| PRD-NFR-01 | A primeira tentativa deve ocorrer em menos de 10 segundos em condições normais. |
| PRD-NFR-02 | A mudança do pedido e o registro dos eventos aplicáveis devem ser atômicos. |
| PRD-NFR-03 | A entrega deve ser at-least-once; duplicatas são permitidas e identificáveis. |
| PRD-NFR-04 | Somente URLs HTTPS devem ser aceitas. |
| PRD-NFR-05 | O corpo deve ser assinado com HMAC-SHA256 e secret isolada por endpoint. |
| PRD-NFR-06 | O payload deve ter no máximo 64 KB; exceder o limite causa erro, sem truncamento. |
| PRD-NFR-07 | Cada chamada ao cliente deve expirar após 10 segundos. |
| PRD-NFR-08 | Secrets e assinaturas não devem aparecer em logs ou endpoints de leitura. |
| PRD-NFR-09 | A ordem por pedido é preservada somente enquanto existir um único worker. |
| PRD-NFR-10 | A solução deve reutilizar banco, autenticação, erros, validação e logs existentes. |

## 8. Decisões e trade-offs principais

| Decisão | Benefício | Trade-off |
| --- | --- | --- |
| Outbox no MySQL | Atomicidade com a mudança do pedido e ausência de nova infraestrutura | Crescimento da tabela exige índices, monitoramento e retenção futura |
| Worker único em polling | Atende à latência e simplifica ordem por pedido | Limita vazão e escala horizontal |
| Retry finito e DLQ | Recupera falhas temporárias sem eventos eternos | Pode atrasar entrega por horas e exige operação da DLQ |
| At-least-once | Evita perda em falhas ambíguas | Cliente precisa deduplicar |
| HMAC por endpoint | Isola vazamento e comprova integridade | Cliente gerencia secret e rotação |
| Payload sem items | Mantém evento enxuto | Cliente consulta o pedido para obter detalhes |

## 9. Dependências

- Fluxo transacional de mudança de status do módulo de pedidos.
- MySQL e Prisma existentes.
- Autenticação JWT e papel ADMIN.
- Cadastro de clientes e estados de pedido atuais.
- Processo worker operável separadamente da API.
- Revisão de Segurança sobre HMAC, geração e proteção de secrets.
- Consumidores capazes de receber HTTPS, validar HMAC e deduplicar UUID.

## 10. Riscos e mitigação

| ID | Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| PRD-RISK-01 | Endpoint lento ou indisponível acumular backlog | Alta | Alto | Timeout de 10s, backoff, DLQ e métricas de backlog |
| PRD-RISK-02 | Cliente processar duplicata | Média | Médio | Event ID estável e documentação explícita de deduplicação |
| PRD-RISK-03 | Vazamento de secret permitir falsificação | Baixa | Alto | Secret por endpoint, HTTPS, rotação e redação de logs |
| PRD-RISK-04 | Outbox crescer e degradar o banco | Média | Alto | Índices, lotes pequenos, monitoramento e política futura de retenção |
| PRD-RISK-05 | Worker único interromper entregas | Média | Alto | Processo independente, health check, reinício e alerta por idade do backlog |
| PRD-RISK-06 | Escala futura quebrar ordem por pedido | Baixa nesta fase | Médio | Documentar limitação e decidir particionamento ou locking antes de escalar |

## 11. Critérios de aceitação

- [ ] É possível criar, listar, editar e remover configurações autenticadas.
- [ ] A secret é gerada por endpoint e não aparece em consultas posteriores.
- [ ] Um status assinado gera snapshot; um status não assinado não gera evento.
- [ ] A mudança do pedido e a outbox confirmam ou revertem juntas.
- [ ] A primeira tentativa ocorre em menos de 10 segundos em condição normal.
- [ ] O cliente recebe JSON com os campos e headers acordados.
- [ ] HTTPS, HMAC-SHA256, 64 KB e timeout de 10 segundos são aplicados.
- [ ] Falhas seguem os cinco intervalos e terminam na DLQ.
- [ ] Retentativas mantêm o Event ID.
- [ ] OPERATOR não pode fazer replay; ADMIN pode e fica auditado.
- [ ] A consulta de histórico retorna as 100 entregas mais recentes.
- [ ] A rotação mantém a secret anterior válida por 24 horas.
- [ ] Email, dashboard e rate limiting não aparecem como capacidades desta fase.

## 12. Estratégia de testes e validação

### Funcional

- Exercitar o CRUD, filtros, rotação, histórico e replay.
- Validar cada status do ciclo de pedidos.
- Confirmar que não há evento quando nenhum endpoint assina o status.

### Confiabilidade

- Simular sucesso, response não `2xx`, timeout e falha de rede.
- Verificar os cinco intervalos e a passagem para DLQ.
- Simular confirmação perdida e conferir deduplicação.
- Forçar falha de outbox e provar rollback integral.

### Segurança

- Validar assinatura correta e detecção de corpo alterado.
- Rejeitar HTTP e acesso não ADMIN ao replay.
- Inspecionar logs e respostas para ausência de secrets.
- Reservar dois dias úteis para revisão de Segurança antes do deploy.

### Observabilidade e operação

- Confirmar métricas de backlog, latência, sucesso, retry e DLQ.
- Confirmar correlação por Event ID, Webhook ID e pedido.
- Testar reinício do worker com pendências.

### Regressão

- Executar os testes existentes de pedidos, incluindo transições, estoque e histórico.

## 13. Questões abertas

- Rate limiting por cliente ou endpoint, após observar volume real.
- Retenção e arquivamento da outbox.
- Ordenação por pedido quando houver múltiplos workers.
- Semântica do Event ID em replay manual da DLQ.
- Proteção exata da secret em repouso, sujeita à revisão de Segurança.

## 14. Documentos relacionados

- [RFC](RFC.md)
- [FDD](FDD.md)
- [ADRs](adrs/)
- [Tracker de rastreabilidade](TRACKER.md)
