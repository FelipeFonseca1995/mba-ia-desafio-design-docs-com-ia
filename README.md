# Design Docs Gerados por IA — Sistema de Webhooks de Notificação de Pedidos

---

## 1. Sobre o Desafio
Este desafio teve como objetivo simular um cenário real de engenharia de software onde decisões técnicas tomadas em reuniões de alinhamento precisam ser transformadas em especificações técnicas precisas e acionáveis, prontas para serem desenvolvidas. A partir da transcrição literal de uma reunião de 55 minutos (`TRANSCRICAO.md`) e da análise do código-fonte de um Order Management System (OMS) existente em Node.js e TypeScript, estruturamos um pacote completo de documentação composto por PRD, RFC, FDD, 6 ADRs e um Tracker de rastreabilidade transversal.

A documentação gerada garante que o novo **Sistema de Webhooks de Notificação de Pedidos** (outbound webhook) seja integrado com sucesso à arquitetura modular existente, preservando as políticas de consistência atômica, segurança criptográfica por assinaturas HMAC-SHA256, resiliência no envio com backoff exponencial e rastreabilidade total de dados.

---

## 2. Ferramentas de IA Utilizadas
Durante a elaboração deste projeto, foram utilizadas as seguintes ferramentas de Inteligência Artificial:
* **Antigravity (Agente de Coding do Google DeepMind):** Atuou como par de programação e engenharia principal, responsável por escanear a estrutura de pastas do projeto, interpretar os ganchos com arquivos reais de código (como `order.service.ts` e `app-error.ts`) e redigir o esqueleto técnico dos documentos.
* **Claude 3.5 Sonnet:** Utilizado para refinar o tom técnico da documentação, garantindo a separação correta de níveis de abstração (estratégico no PRD, arquitetural no RFC, de decisão pontual nos ADRs e de implementação no FDD).
* **Gemini Pro (Advanced):** Utilizado para realizar a auditoria cruzada semântica no preenchimento final do `TRACKER.md` a fim de evitar qualquer alucinação de requisitos que não constassem na transcrição original da call.

---

## 3. Workflow Adotado
Organizamos o processo de engenharia de documentação nas seguintes etapas sequenciais:

1. **Leitura e Contextualização:** Analisamos a transcrição para catalogar participantes, restrições e conflitos. Em paralelo, exploramos o codebase (módulos, tratamento de erros `AppError`, logger Pino e middlewares de auth/erro) para mapear os pontos de extensão e integração.
2. **Definição de Decisões (ADRs):** Produzimos primeiramente os registros de decisões de arquitetura (ADRs) sob `docs/adrs/`. Definir o esqueleto de infraestrutura (Outbox, Retry/DLQ, HMAC, At-least-once, Polling Worker) primeiro garantiu que os demais documentos herdassem definições coesas.
3. **Elaboração da RFC (Proposta Arquitetural):** Escrevemos a RFC ligando as decisões tomadas aos debates da reunião, detalhando as 3 alternativas técnicas consideradas/descartadas e as 2 questões em aberto que permaneceram sob observação da equipe.
4. **Construção do FDD (Especificação de Construção):** Detalhamos o "como construir" no FDD, adicionando fluxos Mermaid de sequência/atividades, payloads exatos de requests e responses de 4 endpoints e a matriz de erros sob o prefixo `WEBHOOK_*`.
5. **Consolidação do PRD (Nível de Produto):** Elaboramos o PRD para amarrar os requisitos de negócio e alinhar as metas quantitativas de latência de entrega e métricas de sucesso com o impacto estratégico do projeto.
6. **Mapeamento do Tracker de Rastreabilidade:** Consolidamos o `TRACKER.md` indexando cada item técnico à sua exata coordenada temporal na transcrição ou arquivo físico correspondente no código-fonte.

---

## 4. Prompts Customizados

Abaixo encontram-se dois prompts customizados criados e utilizados para conduzir a geração dos documentos com as IAs de forma eficiente e contextualizada:

### Prompt 1: Extração de Alternativas de Arquitetura e Trade-offs para RFC
```markdown
Contexto: Você é a Tech Lead Larissa revisando a transcrição da reunião (TRANSCRICAO.md).
Tarefa: Identifique e extraia exatamente as discussões sobre alternativas técnicas que foram levantadas durante a call para solucionar o envio de notificações. 
Para cada alternativa identifique:
1. Qual era a proposta original e quem a sugeriu.
2. Qual foi o argumento técnico contra a proposta e quem o levantou.
3. Qual foi o trade-off aceito pelo time ao descartá-la.
Mantenha os fatos estritamente rastreáveis à transcrição. Não invente alternativas adicionais ou discussões que não ocorreram na gravação.
```

### Prompt 2: Alinhamento de Padrões e Códigos de Erro para FDD
```markdown
Contexto: O sistema existente em NodeJS possui classes de exceção estendendo AppError (localizado em src/errors/app-error.ts), logger Pino centralizado em src/shared/logger/index.ts e middleware centralizado de erros em src/middlewares/error.middleware.ts.
Tarefa: Modele uma matriz de tratamento de erros para a feature de Webhook baseando-se estritamente nestes arquivos de código. A matriz deve listar:
- Nome do erro no padrão WEBHOOK_*
- Status Code HTTP correspondente
- Descrição da mensagem amigável de erro
- O fluxo de disparo desse erro no código da integração (ex: na chamada de validação HTTPS Zod ou falha de timeout do Axios).
Garante que os arquivos nomeados existam e descreva de que maneira o middleware existente vai capturar essas exceções de forma transparente.
```

---

## 5. Iterações e Ajustes
A interação contínua com a IA exigiu revisões críticas e ajustes pontuais. Dois momentos principais de correção foram:

* **Ajuste 1 (Identificação de Escopo):** Na primeira geração do PRD e FDD, a IA incluiu um "Módulo de Envio de E-mails de Alerta em caso de falhas consecutivas de webhook" como requisito funcional concluído. Realizei a correção manual instruindo a IA a reanalisar o trecho `[09:37] Larissa` da transcrição, no qual ela explicitamente descarta essa funcionalidade, classificando-a como fora de escopo para esta fase. O item foi realocado corretamente para a seção de "Fora de Escopo / Próximas Fases".
* **Ajuste 2 (Decisão de Infraestrutura / Redis):** Durante o rascunho do ADR-001, a IA sugeriu utilizar o Redis Streams como o broker adotado para a fila do webhook. Intervi com base na transcrição (`[09:07] Diego` e `[09:07] Larissa`), lembrando que o time descartou a implantação de um cluster Redis devido a restrições de equipe de operações, optando por usar tabelas na base MySQL e polling periódico de 2 segundos. O ADR foi reescrito para refletir com exatidão a decisão do banco MySQL.

---

## 6. Como Navegar a Entrega
Para analisar o pacote de design docs entregue, sugerimos seguir a ordem de leitura abaixo:

1. **[docs/PRD.md](file:///c:/Users/Felipe%20Fonseca/Documents/01.%20primeiro_projeto/Projetos%20Full%20Cycle/Atividade%205/docs/PRD.md):** Fornece a visão estratégica, requisitos de negócio (funcionais e não funcionais) e limites de escopo.
2. **[docs/RFC.md](file:///c:/Users/Felipe%20Fonseca/Documents/01.%20primeiro_projeto/Projetos%20Full%20Cycle/Atividade%205/docs/RFC.md):** Apresenta o desenho de arquitetura macro proposto, as decisões tomadas e os trade-offs das opções descartadas.
3. **[docs/adrs/](file:///c:/Users/Felipe%20Fonseca/Documents/01.%20primeiro_projeto/Projetos%20Full%20Cycle/Atividade%205/docs/adrs/):** Diretório que contém os 6 registros de decisão detalhando cada peça técnica da infraestrutura.
4. **[docs/FDD.md](file:///c:/Users/Felipe%20Fonseca/Documents/01.%20primeiro_projeto/Projetos%20Full%20Cycle/Atividade%205/docs/FDD.md):** Contém os diagramas de sequência, contratos exatos de payload/endpoints e a lógica de integração no código.
5. **[docs/TRACKER.md](file:///c:/Users/Felipe%20Fonseca/Documents/01.%20primeiro_projeto/Projetos%20Full%20Cycle/Atividade%205/docs/TRACKER.md):** A tabela de auditoria cruzada que certifica a rastreabilidade integral da documentação.
