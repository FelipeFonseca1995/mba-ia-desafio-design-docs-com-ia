# PRD — Product Requirements Document

## 1. Resumo e Contexto da Feature
Esta feature visa introduzir um **Sistema de Webhooks de Notificação de Pedidos** (Outbound Webhooks) no Order Management System (OMS) existente. O sistema enviará notificações HTTP `POST` assíncronas e em tempo real para os endpoints cadastrados pelos clientes B2B sempre que ocorrer uma alteração de status em seus respectivos pedidos.

---

## 2. Problema e Motivação
Clientes B2B da nossa plataforma (como Atlas Comercial, MaxDistribuição e Nova Cargo) necessitam integrar seus sistemas internos (ERP/WMS) para sincronizar o status dos pedidos. Atualmente, esses clientes realizam polling periódico repetitivo no endpoint `GET /orders` para buscar atualizações. 

Essa abordagem traz severas desvantagens:
1. **Ineficiência de Rede e Infraestrutura:** Requisições repetitivas sobrecarregam nossa API e consomem recursos computacionais desnecessariamente.
2. **Latência de Integração:** O polling causa atraso na percepção das mudanças de status pelos clientes.
3. **Risco de Churn:** Clientes importantes (como a Atlas Comercial) sinalizaram a possibilidade de migrar para concorrentes caso um sistema de notificações em tempo real não seja entregue até o fim do trimestre.

---

## 3. Público-Alvo e Cenários de Uso
* **Público-Alvo:** Desenvolvedores e engenheiros de integração dos clientes B2B que consomem nossos serviços.
* **Cenário de Uso:** O ERP do cliente Atlas Comercial precisa ser notificado assim que o status de um pedido mudar de `PROCESSING` para `SHIPPED`, a fim de coordenar a retirada de carga com a transportadora correspondente, sem ter que consultar nossa API de 1 em 1 minuto.

---

## 4. Objetivos e Métricas de Sucesso
* **Objetivo Geral:** Oferecer um canal de integração passivo seguro, confiável e assíncrono para os clientes receberem mudanças de status de pedidos em menos de 10 segundos.
* **Objetivo Quantitativo (Métrica de Sucesso):** 
  * **Latência de Entrega:** Pelo menos **95% dos webhooks gerados** devem ter sua primeira tentativa de entrega efetuada em **menos de 5 segundos** a partir do momento do commit da alteração de status do pedido (meta quantitativa).
  * **Redução de Polling Ineficiente:** Reduzir em até 80% o número de requisições `GET /orders` oriundas dos clientes que adotarem os webhooks.

---

## 5. Escopo

### Incluso no Escopo
* API RESTful de cadastro e gerenciamento de endpoints de webhook filtrados por tipos de status específicos.
* Gravação atômica transacional de eventos de alteração de status na tabela de Outbox.
* Worker assíncrono isolado para leitura periódica da Outbox e disparo das requisições HTTP POST.
* Política de retentativa automática (retry) resiliente com backoff exponencial/progressivo.
* Tabela de Letras Mortas (DLQ - Dead Letter Queue) para persistência de entregas falhas definitivas.
* Endpoint restrito para replay manual de eventos da DLQ.
* Criptografia de payloads de webhook via assinatura HMAC-SHA256.
* Rotação de chaves secretas dos endpoints com suporte a um período de carência (grace period) de 24 horas.

### Fora de Escopo (Explicitamente Descartado ou Adiado na Reunião)
1. **Envio de e-mails de alerta automático para os clientes em caso de falhas consecutivas de webhook:** Adiado para fases futuras após medição do impacto técnico.
2. **Painel Visual / Dashboard (Frontend) para gerenciamento e monitoramento de entregas de webhooks pelos clientes:** Escopo de responsabilidade do time de frontend em projeto separado; a entrega atual engloba estritamente as APIs de backend do monorepo.
3. **Mecanismo de controle de taxa de saída (Rate Limiting de saída) por cliente:** Adiado; o time de plataforma monitorará o tráfego de saída e planejará isso em caso de problemas futuros.

---

## 6. Requisitos Funcionais (RF)

| ID | Requisito Funcional | Descrição | Origem (Transcrição) |
|---|---|---|---|
| **RF-01** | Cadastro de Webhooks | Os clientes B2B podem registrar um endpoint HTTP de webhook fornecendo a URL, um filtro contendo a lista de status de pedidos que desejam escutar, e receber uma secret gerada automaticamente pelo sistema. | `[09:31] Marcos` |
| **RF-02** | Gerenciamento de Webhooks | Permite que o usuário cliente execute operações de consulta, edição parcial (PATCH) e exclusão (DELETE) das configurações de webhook vinculadas ao seu `customer_id`. | `[09:32] Bruno` |
| **RF-03** | Filtragem no Enfileiramento | Os eventos de webhook só devem ser inseridos na outbox caso o status alterado esteja contido na lista de filtros configurada pelo webhook do cliente. | `[09:33] Marcos` / `[09:34] Bruno` |
| **RF-04** | Histórico de Entregas | Disponibilizar um endpoint (`GET /webhooks/:id/deliveries`) para o cliente consultar os logs das últimas 100 tentativas de envio do webhook (incluindo status, payload enviado, resposta recebida e tempo de resposta). | `[09:34] Marcos` |
| **RF-05** | Persistência em DLQ | Eventos que esgotarem as 5 tentativas de envio sem sucesso devem ser removidos da outbox e gravados na tabela de Letras Mortas (`webhook_dead_letter`), registrando a última mensagem de erro. | `[09:17] Diego` / `[09:18] Diego` |
| **RF-06** | Replay Manual de DLQ | Disponibilizar endpoint administrativo (`POST /admin/webhooks/dead-letter/:id/replay`) restrito a usuários com permissão de `ADMIN` para reenviar eventos falhos da DLQ. | `[09:18] Diego` / `[09:35] Larissa` |
| **RF-07** | Rotação de Segredo com Carência | O cliente pode solicitar a rotação do segredo do endpoint. O segredo antigo deve continuar aceito e ativo em paralelo por até 24 horas para evitar downtime. | `[09:21] Sofia` |
| **RF-08** | Assinatura HMAC-SHA256 | Todas as requisições de webhook enviadas devem conter a assinatura do payload gerada usando algoritmo HMAC-SHA256, enviada no cabeçalho `X-Signature`. | `[09:20] Sofia` |
| **RF-09** | Envio de Cabeçalhos de Controle | As requisições HTTP do webhook devem incluir os cabeçalhos `X-Event-Id` (UUID único de controle do evento), `X-Webhook-Id` (UUID do cadastro), `X-Timestamp` (timestamp de envio) e `Content-Type: application/json`. | `[09:25] Diego` / `[09:44] Diego` / `[09:44] Sofia` |

---

## 7. Requisitos Não Funcionais (RNF)

| ID | Requisito Não Funcional | Descrição |
|---|---|---|
| **RNF-01** | Consistência Transacional (Outbox) | A gravação do evento na tabela `webhook_events_outbox` deve ser realizada dentro da mesma transação de banco de dados (`Prisma transaction`) que atualiza o status do pedido em `OrderService.changeStatus`. |
| **RNF-02** | Latência Máxima de Polling | O worker de processamento deve consultar a tabela outbox a cada 2 segundos, garantindo latência máxima de entrega inferior a 10 segundos sob condições normais de rede. |
| **RNF-03** | Isolamento de Processos | O worker deve ser executado em uma instância do Node.js isolada e dedicada (`npm run worker`), sem compartilhar recursos do event-loop da API REST principal. |
| **RNF-04** | Protocolo HTTPS Obrigatório | O sistema deve validar e aceitar exclusivamente URLs que utilizem o protocolo criptográfico HTTPS (`https://...`), recusando URLs `http://`. |
| **RNF-05** | Limite de Tamanho do Payload | O tamanho do corpo da notificação de webhook está limitado a no máximo 64 KB. Caso exceda, o envio deve falhar de imediato e ser reportado como erro de limite. |
| **RNF-06** | Garantia At-Least-Once | O sistema deve assegurar que cada alteração de status elegível seja entregue pelo menos uma vez ao destino. Casos de retransmissão de eventos duplicados devem ser identificáveis pelo ID de evento estável. |

---

## 8. Decisões e Trade-offs Principais
* **Consistência eventual via Banco vs. Brokers de Evento:** Optou-se por utilizar tabelas outbox no banco MySQL existente ao invés de implantar uma infraestrutura de mensageria como Redis Cluster ou RabbitMQ. Isso reduz o custo operacional e a complexidade técnica para o time atual (que é pequeno), aceitando o trade-off de um overhead sutil de escrita transacional no banco de dados relacional.
* **Garantia At-Least-Once vs. Exactly-Once:** A plataforma assume que webhooks podem ser retransmitidos em caso de incerteza de recepção (ex.: timeouts). Isso simplifica o worker e delega ao cliente B2B a responsabilidade de implementar deduplicação local com base no `X-Event-Id` enviado no cabeçalho.

---

## 9. Dependências
* **Prisma Client & MySQL:** Necessários para a criação das tabelas, índices e manipulação transacional dos eventos.
* **Express & Node.js:** Base de execução da API e do worker.
* **Zod:** Utilizado para a validação rigorosa dos payloads de criação e URLs HTTPS seguros.
* **Pino Logger:** Biblioteca de logging do projeto existente utilizada para o rastreamento operacional do worker.
* **requireRole middleware:** Necessário para restringir a rota de replay de DLQ apenas para `ADMIN` (em `src/middlewares/auth.middleware.ts`).

---

## 10. Riscos e Mitigação

### Risco 1: Vazamento ou exposição acidental de chave secreta (*secret*) de assinatura pelo cliente
* **Probabilidade:** Média
* **Impacto:** Alto (Pode permitir que terceiros enviem requisições forjadas que pareçam válidas).
* **Mitigação:** Cada endpoint possui um segredo gerado isoladamente. Disponibilizaremos o endpoint de rotação rápida de chaves com 24 horas de grace period, permitindo a substituição imediata sem derrubar o recebimento de webhooks ativos.

### Risco 2: Sobrecarga de leitura no banco MySQL devido ao loop de polling de 2s do worker
* **Probabilidade:** Baixa
* **Impacto:** Médio (Pode degradar consultas concorrentes de usuários se a tabela outbox crescer descontroladamente).
* **Mitigação:** Criação de índices específicos compostos na tabela outbox nos campos `[status, nextAttemptAt]`, consultas limitadas em batches curtos (ex: `LIMIT 20`), e estabelecimento de uma política de expurgo periódico de dados antigos processados (histórico superior a 30 dias).

---

## 11. Critérios de Aceitação
1. A alteração do status do pedido no banco de dados só pode ser confirmada (commit) se a gravação do registro correspondente na tabela de outbox também for concluída com sucesso.
2. Nenhuma URL com protocolo `http://` deve ser aceita na criação ou atualização de webhooks.
3. Se um endpoint cliente retornar falha (status fora da faixa 200-299) ou timeout de 10 segundos, o worker deve agendar retentativas estritamente respeitando a progressão: 1m, 5m, 30m, 2h, 12h.
4. O replay manual de eventos na DLQ deve validar e exigir que o token JWT do operador possua a role `UserRole.ADMIN`. Operadores normais (`OPERATOR`) devem receber erro HTTP `403 Forbidden`.

---

## 12. Estratégia de Testes e Validação
* **Testes Unitários:** Validação das regras de geração de segredo HMAC, validações de schema Zod (rejeitando http e aceitando https) e cálculo correto dos intervalos de backoff progressivo.
* **Testes de Integração:** Validar que ao chamar `OrderService.changeStatus` dentro de uma transação Prisma, o registro outbox correspondente é escrito perfeitamente.
* **Testes End-to-End (E2E):** Subir um mock server HTTP local, disparar mudanças de status de pedidos via API, e verificar que o worker rodando em separado consome a outbox e entrega as requisições HTTP contendo os headers `X-Signature` e `X-Event-Id` corretos.
* **Testes de Segurança:** Verificar que o endpoint de replay rejeita operadores sem a role `ADMIN` e logs do Pino registram o ID de auditoria do administrador solicitante.
