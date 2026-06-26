# ADR-005: Worker em Processo Separado com Polling de 2s

* **Status:** Aceito
* **Data:** 2026-06-26
* **Autor:** Felipe Fonseca

---

## Contexto e Problema

Precisamos definir como o processador assíncrono (worker) irá consumir os eventos registrados na tabela `webhook_events_outbox` e efetuar os disparos de rede HTTP para os clientes. Duas questões fundamentais precisam ser respondidas:
1. **Acoplamento de Processo:** O worker deve rodar dentro do mesmo processo Node.js da API Express principal ou em um processo separado?
2. **Mecanismo de Despertar:** Como o worker sabe que há novos eventos a serem processados? Ele deve ser notificado por um evento do banco de dados (reacionário) ou buscar ativamente em intervalos de tempo (polling)?

---

## Decisão

1. **Processo Separado:** O worker rodará como um processo Node.js totalmente independente do processo da API. Terá seu próprio ponto de entrada em `src/worker.ts` e um script de execução específico no `package.json` (`npm run worker`). Ele abrirá sua própria conexão PrismaClient independente com o banco de dados.
2. **Polling de 2 segundos:** O worker buscará novos eventos pendentes (`status: "PENDING"` ou prontos para retry) por meio de consultas SQL periódicas executadas a cada 2 segundos.

### Lógica de Polling Simplificada:
A cada ciclo de 2 segundos, o worker fará:
```sql
SELECT * FROM webhook_events_outbox 
WHERE status = 'PENDING' AND nextAttemptAt <= NOW()
ORDER BY createdAt ASC 
LIMIT 20;
```
Para cada evento retornado:
1. Marca o status como `PROCESSING` ou utiliza um bloqueio pessimista para evitar concorrência se houver mais de um worker (ex.: `SELECT ... FOR UPDATE` ou atualização de controle por token).
2. Tenta disparar a requisição HTTP.
3. Se sucesso, atualiza para `COMPLETED`.
4. Se falha, recalcula `nextAttemptAt` baseado no backoff ou move para a DLQ se esgotadas as tentativas.

---

## Alternativas Consideradas

### 1. Executar Worker no mesmo Processo da API Express
* **Descrição:** Inicializar uma rotina de loop assíncrona ou timer dentro do próprio arquivo `src/server.ts` da API Express.
* **Razão do Descarte:** Causa acoplamento de recursos. Se a API Express sofrer oscilações ou for reiniciada devido a uma sobrecarga de requisições web, as tentativas de entrega de webhooks seriam interrompidas. Além disso, o processamento de rede e I/O de webhooks competiria pelo mesmo event loop da API pública, prejudicando o tempo de resposta dos usuários da API.

### 2. Notificação por Trigger do Banco de Dados (LISTEN/NOTIFY)
* **Descrição:** Usar triggers ou listeners do MySQL para avisar o worker em tempo real quando um registro entra na tabela outbox.
* **Razão do Descarte:** O MySQL não dispõe de um sistema nativo e eficiente de LISTEN/NOTIFY para processos externos (como o Postgres possui). Criar mecanismos simulados de listener no MySQL geraria soluções complexas de manter. O polling simples de 2 segundos atende com folga o SLA exigido pelos clientes B2B (latência inferior a 10 segundos).

---

## Consequências

### Positivas
* **Resiliência e Isolamento:** Se a API cair ou for escalada horizontalmente para lidar com tráfego web, o worker de webhooks continua rodando e enviando requisições em seu próprio ritmo, sem interferência.
* **Simplicidade de Implementação:** O loop de polling em TypeScript com `setInterval` ou loops de controle com atrasos de 2 segundos é simples de escrever, testar e debugar.
* **Consumo Previsível:** O polling de batch limitados (ex.: `LIMIT 20`) impede que o worker consuma memória ou conexões excessivas se houver uma explosão de pedidos.

### Negativas
* **Desperdício de Recursos Ociosos:** O worker fará queries a cada 2 segundos mesmo se o sistema estiver inativo (ex.: de madrugada), gerando atividade ociosa mínima no banco de dados (que pode ser minimizada com índices eficientes e consultas rápidas).
* **Latência Mínima Inerente:** Introduce uma latência de até 2 segundos para o início do processamento de qualquer evento, o que é plenamente aceitável dentro do requisito do negócio (< 10 segundos).
