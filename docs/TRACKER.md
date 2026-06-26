# Tracker de Rastreabilidade

Este documento estabelece a matriz de rastreabilidade transversal, mapeando cada requisito, decisão arquitetural, restrição técnica e trade-off contido nos design docs à sua respectiva fonte original, seja ela a transcrição da reunião técnica (`TRANSCRICAO.md`) ou o código-fonte da aplicação existente (`src/**`).

---

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| **PRD-FR-01** | docs/PRD.md | Requisito Funcional | Cadastro de webhook com URL, lista de status e secret gerada. | TRANSCRICAO | `[09:31] Marcos` |
| **PRD-FR-02** | docs/PRD.md | Requisito Funcional | Edição, consulta e deleção de webhook vinculado a customer. | TRANSCRICAO | `[09:32] Bruno` |
| **PRD-FR-03** | docs/PRD.md | Requisito Funcional | Filtro de eventos de status de pedido na inserção da outbox. | TRANSCRICAO | `[09:33] Marcos` |
| **PRD-FR-04** | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas de webhook para consulta. | TRANSCRICAO | `[09:34] Marcos` |
| **PRD-FR-05** | docs/PRD.md | Requisito Funcional | Movimentação automática para DLQ após 5 falhas no envio. | TRANSCRICAO | `[09:17] Diego` |
| **PRD-FR-06** | docs/PRD.md | Requisito Funcional | Replay manual de DLQ via endpoint de administração. | TRANSCRICAO | `[09:18] Diego` |
| **PRD-FR-07** | docs/PRD.md | Requisito Funcional | Rotação de chaves com período de carência (grace period) de 24 horas. | TRANSCRICAO | `[09:21] Sofia` |
| **PRD-FR-08** | docs/PRD.md | Requisito Funcional | Assinatura digital do corpo via HMAC-SHA256 em `X-Signature`. | TRANSCRICAO | `[09:20] Sofia` |
| **PRD-FR-09** | docs/PRD.md | Requisito Funcional | Cabeçalhos HTTP de controle: `X-Event-Id`, `X-Webhook-Id`, `X-Timestamp`. | TRANSCRICAO | `[09:25] Diego` |
| **PRD-RNF-01** | docs/PRD.md | Requisito Não Funcional | Gravação da outbox na mesma transação SQL que altera o pedido. | TRANSCRICAO | `[09:06] Diego` |
| **PRD-RNF-02** | docs/PRD.md | Requisito Não Funcional | Polling periódico com intervalo máximo de 2 segundos. | TRANSCRICAO | `[09:09] Diego` |
| **PRD-RNF-03** | docs/PRD.md | Requisito Não Funcional | Worker isolado executado em processo Node.js separado. | TRANSCRICAO | `[09:11] Diego` |
| **PRD-RNF-04** | docs/PRD.md | Requisito Não Funcional | Validação de protocolo obrigatório HTTPS para URL cadastrada. | TRANSCRICAO | `[09:23] Sofia` |
| **PRD-RNF-05** | docs/PRD.md | Requisito Não Funcional | Limite físico máximo de tamanho de payload fixado em 64 KB. | TRANSCRICAO | `[09:24] Diego` |
| **PRD-RNF-06** | docs/PRD.md | Requisito Não Funcional | Garantia at-least-once com cabeçalho de controle para deduplicação. | TRANSCRICAO | `[09:25] Diego` |
| **RFC-ALT-01** | docs/RFC.md | Decisão/Trade-off | Descarte de envio HTTP síncrono no fluxo de pedidos. | TRANSCRICAO | `[09:04] Bruno` |
| **RFC-ALT-02** | docs/RFC.md | Decisão/Trade-off | Descarte de mensageria externa (Redis Streams/RabbitMQ). | TRANSCRICAO | `[09:07] Diego` |
| **RFC-ALT-03** | docs/RFC.md | Decisão/Trade-off | Descarte de listeners por triggers nativas do MySQL. | TRANSCRICAO | `[09:09] Diego` |
| **RFC-QA-01** | docs/RFC.md | Questão em Aberto | Rate Limiting de envio na saída do webhook por cliente. | TRANSCRICAO | `[09:38] Diego` |
| **RFC-QA-02** | docs/RFC.md | Questão em Aberto | Notificação automática (email) do cliente em caso de queda do webhook. | TRANSCRICAO | `[09:37] Marcos` |
| **ADR-001-DEC** | docs/adrs/ADR-001-padrao-outbox-no-mysql.md | Decisão | Uso de tabela outbox no banco MySQL existente do projeto. | TRANSCRICAO | `[09:06] Diego` |
| **ADR-002-DEC** | docs/adrs/ADR-002-politica-retry-backoff-dlq.md | Decisão | Escala de backoff com intervalos fixos (1m/5m/30m/2h/12h). | TRANSCRICAO | `[09:17] Diego` |
| **ADR-003-DEC** | docs/adrs/ADR-003-autenticacao-hmac-sha256.md | Decisão | Assinatura HMAC-SHA256 baseada em segredo individual por webhook. | TRANSCRICAO | `[09:20] Sofia` |
| **ADR-004-DEC** | docs/adrs/ADR-004-garantia-at-least-once-idempotencia.md | Decisão | Dedup do lado do cliente via ID único enviado em `X-Event-Id`. | TRANSCRICAO | `[09:25] Diego` |
| **ADR-005-DEC** | docs/adrs/ADR-005-worker-polling-processo-separado.md | Decisão | Instância isolada do worker com loop ativo e limite em batch. | TRANSCRICAO | `[09:11] Diego` |
| **ADR-006-DEC** | docs/adrs/ADR-006-reuso-padroes-codebase.md | Decisão | Reuso de AppError, requireRole, error middleware e Pino logger. | TRANSCRICAO | `[09:30] Larissa` |
| **FDD-FLX-01** | docs/FDD.md | Requisito Técnico | Inserção transacional com snapshot do payload na criação. | TRANSCRICAO | `[09:52] Larissa` |
| **FDD-FLX-02** | docs/FDD.md | Requisito Técnico | Timeout das requisições HTTP do worker limitado a 10s. | TRANSCRICAO | `[09:42] Diego` |
| **FDD-FLX-03** | docs/FDD.md | Requisito Técnico | IDs das tabelas gerados no padrão UUID v4 global da aplicação. | TRANSCRICAO | `[09:51] Larissa` |
| **COD-INT-01** | docs/FDD.md | Integração de Código | Inserção transacional no service de pedidos. | CODIGO | `src/modules/orders/order.service.ts` |
| **COD-INT-02** | docs/FDD.md | Integração de Código | Modelagem de erros com a classe AppError. | CODIGO | `src/shared/errors/app-error.ts` |
| **COD-INT-03** | docs/FDD.md | Integração de Código | Controle de privilégios de administrador no replay de DLQ. | CODIGO | `src/middlewares/auth.middleware.ts` |
| **COD-INT-04** | docs/FDD.md | Integração de Código | Filtro centralizado de formatação de payloads de exceção. | CODIGO | `src/middlewares/error.middleware.ts` |
| **COD-INT-05** | docs/FDD.md | Integração de Código | Ingestão e centralização de logs do worker de disparo. | CODIGO | `src/shared/logger/index.ts` |
| **COD-INT-06** | docs/FDD.md | Integração de Código | Validação das transições de status da máquina de estado de pedidos. | CODIGO | `src/modules/orders/order.status.ts` |
