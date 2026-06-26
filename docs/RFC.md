# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados
* **Autor:** Felipe Fonseca
* **Status:** Em Revisão
* **Data:** 2026-06-26
* **Revisores:**
  * Larissa (Tech Lead)
  * Marcos (Product Manager)
  * Bruno (Engenheiro de Software, time de Pedidos)
  * Diego (Engenheiro de Software Sênior, time de Plataforma)
  * Sofia (Engenheira de Segurança)

---

## 1. Resumo Executivo (TL;DR)
Esta RFC propõe a implementação de um sistema assíncrono de notificações de alteração de status de pedidos por meio de **outbound webhooks**. A solução baseia-se no padrão **Transactional Outbox** gravado no banco de dados MySQL existente e consumido por um processo **worker isolado** em loops de polling de 2 segundos. A comunicação será protegida por criptografia de payload via assinaturas **HMAC-SHA256** exclusivas por endpoint, oferecendo garantia de entrega **At-Least-Once** com mecanismos de retentativa e uma fila de eventos falhos (**Dead Letter Queue - DLQ**).

---

## 2. Contexto e Problema
Grandes clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) exigem atualizações de status de pedidos em tempo real (latência inferior a 10 segundos) para integrar seus ERPs. O método atual de integração é baseado em polling repetitivo nos endpoints de consulta de pedidos (`GET /orders`), o que gera gargalos de desempenho e desperdício de recursos. 

A falta de um canal de notificações ativas (webhooks) representa um risco imediato de churn da Atlas Comercial para soluções concorrentes. A implementação da feature deve ser robusta, não afetar o desempenho da API Express existente e reutilizar os recursos e padrões da nossa codebase.

---

## 3. Proposta Técnica
A solução de webhooks arquitetada divide-se em três camadas principais:

```
+-------------------------------------------------------------------+
|                           PROCESSO API                            |
|                                                                   |
|  [Cliente B2B] --(JWT Autenticado)--> [CRUD / Config Webhooks]    |
|                                                                   |
|  [Operador] ------(Mudar Status)------> [OrderService]            |
|                                              |                    |
|                                     (Transação Prisma)            |
|                                              |                    |
|                                              v                    |
|                                  [MySQL: orders & outbox]         |
+-------------------------------------------------------------------+
                                               |
                                     (Polling a cada 2s)
                                               |
                                               v
+-------------------------------------------------------------------+
|                          PROCESSO WORKER                          |
|                                                                   |
|   [Outbox Event] --> [Calcular HMAC-SHA256] --> [HTTP POST Client]|
|                                                          |        |
|                                                       (Falha)     |
|                                                          |        |
|                                                      (5x Retries) |
|                                                          v        |
|                                                 [MySQL: DLQ Table]|
+-------------------------------------------------------------------+
```

### 3.1. Persistência Consistente (Outbox)
Sempre que ocorrer a alteração de status de um pedido no `OrderService.changeStatus` (localizado em `src/modules/orders/order.service.ts`), a inserção do evento na tabela `webhook_events_outbox` ocorrerá dentro do mesmo bloco transacional SQL. Se o status do pedido não mudar ou a transação falhar, nenhuma notificação fantasma será gerada. O evento guardará o snapshot do payload renderizado em JSON no momento do commit.

### 3.2. Desacoplamento de Entrega (Worker)
Um script de entrada dedicado `src/worker.ts` será inicializado em contêiner ou processo isolado. Este worker consultará periodicamente (a cada 2 segundos) os registros pendentes do banco e realizará as requisições HTTP para os clientes. Caso a entrega falhe, o worker executará retentativas de envio espaçadas usando um algoritmo de backoff com intervalos fixos (1m, 5m, 30m, 2h, 12h) e, em caso de falha definitiva, moverá o evento para a tabela de DLQ.

### 3.3. Segurança e Assinatura
Toda notificação enviada incluirá no cabeçalho `X-Signature` o hash digest HMAC-SHA256 gerado a partir do corpo JSON da requisição e da secret única associada àquele endpoint do cliente, garantindo que o receptor possa validar a autenticidade e a integridade da mensagem.

---

## 4. Alternativas Consideradas

### Alternativa A: Requisição HTTP Síncrona direta no OrderService
* **Abordagem:** Chamar o endpoint do cliente via `axios` ou `fetch` de forma síncrona dentro do método `changeStatus`.
* **Trade-off e Descarte:** Descartada devido ao alto acoplamento e latência de rede introduzida no processo transacional de pedidos, além de não fornecer uma forma limpa de retentar envios em falhas temporárias dos servidores clientes.

### Alternativa B: Mensageria Assíncrona com Redis Streams / RabbitMQ
* **Abordagem:** Publicar eventos de alteração de status em uma fila Redis ou servidor de mensageria dedicado.
* **Trade-off e Descarte:** Embora escalável, introduziria complexidade operacional excessiva e custos adicionais de infraestrutura desproporcionais para a nossa volumetria atual. A arquitetura de Outbox no banco relacional MySQL reaproveita todo o ecossistema existente com segurança transacional e impacto de custos nulo.

### Alternativa C: Listener de Banco via Triggers do MySQL
* **Abordagem:** Utilizar triggers atreladas à tabela `orders` para notificar um processo de worker ativo.
* **Trade-off e Descarte:** O MySQL não dispõe de listeners ativos em tempo real para avisar agentes externos, o que obrigaria a criação de hacks ou rotinas secundárias complexas de monitoramento de logs binários. O loop de polling simples no worker resolve a latência requerida de forma robusta e limpa.

---

## 5. Questões em Aberto
1. **Controle de Taxa de Disparo (Rate Limiting de Saída):** O que deve ser feito se um cliente sofrer picos de eventos e sua API for sobrecarregada pelos nossos envios? O sistema de webhooks deve throttling ou enfileirar de forma inteligente para proteger o cliente?
2. **Email de Alerta de Falha Concorrente:** Devemos implantar um alerta por e-mail caso os webhooks de um cliente apresentem erros contínuos por mais de 3 horas? (Foi classificado como futuro/fora de escopo nesta fase).

---

## 6. Impacto e Riscos
* **Crescimento de Tabelas:** O volume de escritas nas tabelas outbox e histórico de entregas aumentará gradativamente. Será obrigatória uma rotina de expurgo para excluir logs de entregas concluídas com mais de 30 dias de vida.
* **Complexidade do Cliente B2B:** Os desenvolvedores dos clientes deverão programar validadores de HMAC-SHA256 e lógicas de idempotência. Isso exige que nosso time escreva uma documentação detalhada da API e forneça trechos de código de exemplo para acelerar o onboarding.

---

## 7. Decisões Relacionadas (ADRs)
As principais decisões arquiteturais que dão sustentação a esta RFC estão formalizadas nos seguintes documentos:
* **[ADR-001: Padrão Transactional Outbox no MySQL](file:///c:/Users/Felipe%20Fonseca/Documents/01.%20primeiro_projeto/Projetos%20Full%20Cycle/Atividade%205/docs/adrs/ADR-001-padrao-outbox-no-mysql.md)**: Justifica a consistência atômica via transação Prisma.
* **[ADR-002: Política de Retry com Backoff Progressivo e DLQ](file:///c:/Users/Felipe%20Fonseca/Documents/01.%20primeiro_projeto/Projetos%20Full%20Cycle/Atividade%205/docs/adrs/ADR-002-politica-retry-backoff-dlq.md)**: Detalha as 5 tentativas de envio e a estrutura da tabela de Letras Mortas para auditoria.
* **[ADR-003: Autenticação HMAC-SHA256 com Secret por Endpoint](file:///c:/Users/Felipe%20Fonseca/Documents/01.%20primeiro_projeto/Projetos%20Full%20Cycle/Atividade%205/docs/adrs/ADR-003-autenticacao-hmac-sha256.md)**: Descreve o modelo criptográfico das assinaturas de webhook e o grace period de 24h para rotação de segredo.
* **[ADR-004: Garantia At-Least-Once e Dedup com X-Event-Id](file:///c:/Users/Felipe%20Fonseca/Documents/01.%20primeiro_projeto/Projetos%20Full%20Cycle/Atividade%205/docs/adrs/ADR-004-garantia-at-least-once-idempotencia.md)**: Aborda a entrega de pelo menos uma vez e cabeçalhos de controle.
* **[ADR-005: Worker em Processo Separado com Polling de 2s](file:///c:/Users/Felipe%20Fonseca/Documents/01.%20primeiro_projeto/Projetos%20Full%20Cycle/Atividade%205/docs/adrs/ADR-005-worker-polling-processo-separado.md)**: Estabelece o desacoplamento de processos.
* **[ADR-006: Reuso de Padrões Existentes da Codebase](file:///c:/Users/Felipe%20Fonseca/Documents/01.%20primeiro_projeto/Projetos%20Full%20Cycle/Atividade%205/docs/adrs/ADR-006-reuso-padroes-codebase.md)**: Define a reutilização de `AppError`, Pino logger e controle de acesso com JWT/requireRole.
