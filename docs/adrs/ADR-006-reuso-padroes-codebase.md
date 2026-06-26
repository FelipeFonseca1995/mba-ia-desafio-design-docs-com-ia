# ADR-006: Reuso de Padrões Existentes da Codebase

* **Status:** Aceito
* **Data:** 2026-06-26
* **Autor:** Felipe Fonseca

---

## Contexto e Problema

A aplicação Order Management System (OMS) possui uma arquitetura estabelecida com convenções claras de design de código. A introdução de um novo módulo de webhooks e de um processo worker paralelo não deve quebrar essa padronização ou redefinir utilitários fundamentais (como tratamento de erros, logging ou autorização).

Se codificarmos tratamentos de erro ad-hoc ou bibliotecas de logging diferentes para o módulo de webhooks e worker, teremos:
1. Fragmentação e dificuldade de manutenção do projeto.
2. Inconsistência nos logs (dificultando a agregação e análise centralizada).
3. Duplicação de código para validações comuns de segurança e autenticação.

Precisamos garantir que a feature de webhooks seja perfeitamente integrada às convenções da codebase.

---

## Decisão

Adotaremos o reuso estrito e sistemático de todos os padrões arquiteturais e utilitários da aplicação existente no desenvolvimento do novo módulo de webhooks e no worker.

### 1. Estrutura Modular
O módulo de webhooks será criado sob o diretório `src/modules/webhooks/`, contendo a divisão de responsabilidades padrão da aplicação:
* `webhook.controller.ts` para controle de rotas HTTP.
* `webhook.service.ts` para lógica de negócio e integração com a outbox/DLQ.
* `webhook.repository.ts` para acesso ao banco via Prisma.
* `webhook.routes.ts` para definição de rotas Express.
* `webhook.schemas.ts` para validações Zod.

### 2. Tratamento de Erros e Padrões de Código
* **Exceções de Domínio:** Utilizaremos a classe central `AppError` de `src/shared/errors/app-error.ts` para lançar todas as exceções de negócio do webhook.
* **Prefixo de Códigos de Erro:** Os erros de negócio do módulo receberão obrigatoriamente o prefixo `WEBHOOK_` (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_LIMIT_EXCEEDED`, `WEBHOOK_SECRET_REQUIRED`).
* **Middleware de Erro Centralizado:** O middleware global de Express `src/middlewares/error.middleware.ts` será o único responsável por capturar esses erros nas requisições HTTP e formatar a resposta JSON.

### 3. Observabilidade e Logging
* O worker e o serviço do webhook utilizarão a mesma instância do Logger Pino exportada em `src/shared/logger/index.ts`.
* Toda entrega bem-sucedida, falha temporária de rede (com o status retornado e número da tentativa) e movimentação para a DLQ serão logadas com o Pino contendo metadados (como `webhookId`, `customerId` e `orderId`).

### 4. Controle de Acesso
* As rotas de CRUD de configuração de webhook utilizarão o middleware de autenticação JWT padrão do sistema.
* O endpoint administrativo de replay de DLQ (`POST /admin/webhooks/dead-letter/:id/replay`) utilizará o middleware `requireRole(UserRole.ADMIN)` implementado em `src/middlewares/auth.middleware.ts` para garantir que apenas administradores autenticados executem a ação.

---

## Alternativas Consideradas

### 1. Criar Tratamento de Exceções e Logger Isolado para o Worker
* **Descrição:** Escrever uma classe de erro específica do worker e utilizar `console.log` para os logs de entrega.
* **Razão do Descarte:** Reduz a visibilidade operacional da aplicação. Sem o logger Pino estruturado, a ingestão de logs do worker em ferramentas de APM ou busca integrada (como Datadog ou Elasticsearch) ficaria inconsistente em relação aos logs da API principal.

---

## Consequências

### Positivas
* **Manutenibilidade:** Qualquer desenvolvedor familiarizado com os módulos existentes (como `orders` ou `customers`) conseguirá manter o módulo de webhooks sem curva de aprendizado extra.
* **Consistência de Observabilidade:** Logs unificados facilitam o rastreamento ponta a ponta e a criação de dashboards de monitoramento de saúde do sistema.
* **Robustez e Segurança:** Reaproveita middlewares de segurança já exaustivamente validados e testados no projeto principal.

### Negativas
* **Dependência do Prisma no Worker:** O worker herda o Prisma Client geral e precisa ser executado dentro do ecossistema do monorepo, impedindo que o worker seja extraído para um repositório isolado de microsserviço sem replicação do schema Prisma (consequência aceita dada a simplicidade necessária).
