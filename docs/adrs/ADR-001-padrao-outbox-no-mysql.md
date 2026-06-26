# ADR-001: Padrão Transactional Outbox no MySQL

* **Status:** Aceito
* **Data:** 2026-06-26
* **Autor:** Felipe Fonseca

---

## Contexto e Problema

O módulo de pedidos (*orders*) precisa notificar sistemas externos de clientes B2B (como Atlas Comercial, MaxDistribuição e Nova Cargo) sempre que o status de um pedido for alterado. As mudanças de status ocorrem no método `changeStatus` de `src/modules/orders/order.service.ts` e envolvem atualizações na tabela `orders`, escrita no histórico de status (`order_status_history`) e débito/crédito de estoque de produtos em transações de banco de dados.

Disparar requisições HTTP síncronas diretamente dentro deste fluxo traria sérios problemas:
1. **Latência e Bloqueio:** O tempo de resposta de endpoints externos (especialmente lentos ou instáveis) bloquearia a conexão do banco de dados e atrasaria a resposta para os operadores do sistema.
2. **Consistência:** Se a transação HTTP falhar, não podemos dar rollback em uma mudança de status de pedido real que já foi consolidada. Caso a transação do banco falhe após a chamada HTTP, o cliente receberia uma notificação falsa.
3. **Resiliência:** Ausência de um mecanismo robusto e simples para retentar requisições em caso de falha temporária do cliente.

Precisamos de uma solução que garanta a entrega eventual das notificações de forma assíncrona, desacoplada e atomicamente consistente com as alterações de dados no banco de dados.

---

## Decisão

Adotaremos o padrão **Transactional Outbox** utilizando o banco de dados MySQL existente (gerenciado via Prisma em `prisma/schema.prisma`).

Sempre que ocorrer uma alteração de status de pedido, a gravação do evento de notificação na nova tabela `webhook_events_outbox` será executada **dentro da mesma transação SQL** que atualiza o pedido e o estoque.

Um processo worker separado lerá periodicamente os registros pendentes desta tabela e efetuará os disparos HTTP correspondentes.

### Estrutura do Modelo Outbox (a ser adicionada ao `prisma/schema.prisma`):

```prisma
model WebhookEventOutbox {
  id           String      @id @default(uuid()) @db.Char(36)
  webhookId    String      @db.Char(36)
  customerId   String      @db.Char(36)
  eventType    String      @db.VarChar(64)
  payload      Json
  status       String      @default("PENDING") @db.VarChar(20) // PENDING, PROCESSING, COMPLETED, FAILED
  attempts     Int         @default(0)
  nextAttemptAt DateTime    @default(now())
  createdAt    DateTime    @default(now())
  updatedAt    DateTime    @updatedAt

  @@index([status, nextAttemptAt])
  @@map("webhook_events_outbox")
}
```

---

## Alternativas Consideradas

### 1. Chamada HTTP Síncrona no OrderService
* **Descrição:** Executar o disparo HTTP direto no fluxo do método `changeStatus`.
* **Razão do Descarte:** Descartado devido ao impacto em performance (bloqueio de threads de execução e conexões do pool de banco de dados) e falta de consistência transacional segura em cenários de falhas de rede.

### 2. Mensageria Dedicada (Redis Streams ou Message Broker externo)
* **Descrição:** Publicar os eventos em um broker como Redis ou RabbitMQ imediatamente após a mudança de status.
* **Razão do Descarte:** Embora escalável, introduziria novas dependências de infraestrutura complexas de operar e manter para um time de engenharia pequeno. Além disso, não resolve nativamente a consistência atômica da escrita em banco de dados e publicação da mensagem sem padrões mais complexos como Two-Phase Commit ou Dual Write (que também sofrem de falhas de consistência parcial).

---

## Consequências

### Positivas
* **Garantia de Registro:** O evento de webhook só será registrado se e somente se a transação do banco de dados que altera o status do pedido for bem-sucedida. Caso haja rollback no pedido, o evento também sofre rollback.
* **Simplicidade de Infraestrutura:** Reutiliza o banco MySQL e o Prisma Client existentes no projeto, sem necessidade de implantar servidores adicionais de mensageria neste momento.
* **Desacoplamento Completo:** A API do sistema de pedidos não depende da saúde, tempo de resposta ou disponibilidade dos endpoints dos clientes.

### Negativas
* **Escrita Adicional no Banco de Dados:** Adiciona mais uma query de escrita (`INSERT`) em cada transação de alteração de status do pedido.
* **Estratégia de Cleanup necessária:** A tabela `webhook_events_outbox` crescerá continuamente. Será necessária uma rotina posterior (fora do escopo desta feature) para expurgar ou arquivar registros com mais de 30 dias de processamento concluído.
