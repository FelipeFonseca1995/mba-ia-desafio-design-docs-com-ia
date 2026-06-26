# ADR-002: Política de Retry com Backoff Progressivo e DLQ

* **Status:** Aceito
* **Data:** 2026-06-26
* **Autor:** Felipe Fonseca

---

## Contexto e Problema

Endpoints de clientes externos de webhooks podem falhar temporariamente por instabilidade de rede, indisponibilidade programada, reinicializações ou sobrecarga. Se um disparo falhar, a notificação não pode ser descartada de imediato. É crucial tentar novamente.

No entanto, retentar de forma ingênua e imediata pode:
1. Piorar o estado de um servidor cliente que já está sobrecarregado (efeito "manada").
2. Travar o worker de disparo em loops de espera ineficientes.
3. Impedir que outros eventos saudáveis sejam entregues.

Além disso, após esgotadas as tentativas razoáveis, o evento não deve sumir sem deixar rastros. Ele precisa ser armazenado para que o cliente ou suporte técnico possa investigar e, se necessário, disparar um reenvio manual após a correção do problema.

---

## Decisão

Adotaremos uma política de **5 tentativas de reenvio**, com um intervalo de **backoff progressivo** fixo (pré-calculado) entre as tentativas e uma **Fila de Letras Mortas (Dead Letter Queue - DLQ)** em tabela separada para falhas definitivas.

### 1. Intervalos de Retry (Progressivos)
Os intervalos entre as tentativas seguirão a escala definida pela equipe técnica:
* **Tentativa 1 (Falha inicial):** Reenviar após **1 minuto**.
* **Tentativa 2:** Reenviar após **5 minutos**.
* **Tentativa 3:** Reenviar após **30 minutos**.
* **Tentativa 4:** Reenviar após **2 horas**.
* **Tentativa 5 (Última):** Reenviar após **12 horas**.

A janela de tempo acumulada desde a primeira falha até a desistência final é de aproximadamente **14 horas e 38 minutos**.

### 2. Dead Letter Queue (DLQ)
Caso a 5ª tentativa falhe, o evento será excluído (ou marcado como arquivado) na outbox e movido para a tabela `webhook_dead_letter`. Esta tabela conterá:
* O payload original do evento.
* O histórico condensado ou motivo da última falha (HTTP Status Code, mensagem de erro de rede).
* Timestamps de criação e falha definitiva.

#### Estrutura do Modelo DLQ (`prisma/schema.prisma`):

```prisma
model WebhookDeadLetter {
  id           String   @id @default(uuid()) @db.Char(36)
  webhookId    String   @db.Char(36)
  customerId   String   @db.Char(36)
  eventType    String   @db.VarChar(64)
  payload      Json
  errorMessage String   @db.Text
  createdAt    DateTime @default(now())
  failedAt     DateTime @default(now())

  @@map("webhook_dead_letter")
}
```

### 3. Replay Manual (Restrito a Admins)
Para reprocessar um evento que caiu na DLQ, a API disponibilizará um endpoint administrativo:
`POST /admin/webhooks/dead-letter/:id/replay`

* **Restrição:** Role `ADMIN` obrigatória (verificada por meio do middleware `requireRole` existente em `src/middlewares/auth.middleware.ts`).
* **Lógica:** O evento correspondente é removido da DLQ, suas tentativas são zeradas e ele é reinserido como `PENDING` na `webhook_events_outbox`. O log gerado pelo Logger Pino registrará o ID do usuário administrador que solicitou o replay para fins de auditoria.

---

## Alternativas Consideradas

### 1. Retries Indefinidos com Backoff Exponencial Padrão
* **Descrição:** Tentar enviar para sempre até que o cliente responda.
* **Razão do Descarte:** Clientes desativados ou URLs abandonadas acumulariam eventos indefinidamente na outbox ativa, degradando a performance do polling do worker.

### 2. Manter eventos falhos na tabela de Outbox principal
* **Descrição:** Marcar o status como `FAILED` na própria tabela `webhook_events_outbox`.
* **Razão do Descarte:** A presença de registros falhos definitivos na mesma tabela de trabalho ativo do worker aumentaria o tamanho da tabela desnecessariamente, comprometendo a performance de leitura. A separação física em uma tabela DLQ mantém a outbox limpa e otimizada para processamento rápido.

---

## Consequências

### Positivas
* **Resiliência:** Tolerância a falhas temporárias do cliente com tempo de cobertura de até ~15 horas.
* **Proteção ao Cliente:** O backoff espaçado evita que nossa infraestrutura atue como um ataque DDoS contra um cliente instável.
* **Auditoria e Replay:** Eventos permanentemente falhos são salvos com o erro explicativo e podem ser reenviados manualmente após auditoria.

### Negativas
* **complexidade no Worker:** O worker precisa atualizar os campos `attempts` e `nextAttemptAt` no banco de dados e calcular os tempos adicionais baseados no número da tentativa atual.
