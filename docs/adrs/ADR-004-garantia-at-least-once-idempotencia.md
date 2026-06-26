# ADR-004: Garantia At-Least-Once e Dedup com X-Event-Id

* **Status:** Aceito
* **Data:** 2026-06-26
* **Autor:** Felipe Fonseca

---

## Contexto e Problema

Em sistemas distribuídos que realizam comunicações de rede (HTTP), garantir que uma mensagem seja entregue **exatamente uma vez** (exactly-once) é extremamente complexo e, em protocolos baseados em HTTP padrão, teoricamente impossível devido ao problema das duas redes e quedas temporárias de conexão.

Podem ocorrer falhas de rede no exato momento após o cliente receber e processar a requisição com sucesso, mas antes que a resposta de sucesso (HTTP 200 OK) alcance nossa API. Nesses cenários:
1. Nosso worker interpretará o timeout ou erro de conexão como falha.
2. A política de retry será acionada.
3. O cliente receberá a mesma notificação de evento de pedido novamente.

Se o cliente reprocessar essa mensagem repetida de forma cega, poderá incorrer em inconsistências de dados ou disparos repetidos de ações internas em sua própria infraestrutura (ex: faturamento duplo de pedidos).

---

## Decisão

Adotaremos a garantia de entrega **At-Least-Once** (pelo menos uma vez).

Para mitigar os riscos de duplicidade do lado do cliente, a API do webhook enviará um identificador único universal para cada evento no cabeçalho **`X-Event-Id`** (contendo um UUID v4).

### 1. Garantia de Idempotência
* O valor de `X-Event-Id` é gerado no momento da gravação inicial do registro na tabela `webhook_events_outbox` (ainda na transação de alteração de status do pedido).
* Durante todas as retentativas (retries) do mesmo evento falho, o valor do cabeçalho `X-Event-Id` **permanece rigorosamente o mesmo**.

### 2. Responsabilidade do Cliente (Deduplicação)
* Fica sob responsabilidade do cliente registrar o `X-Event-Id` das notificações processadas com sucesso.
* Ao receber um webhook, o cliente deve checar se o `X-Event-Id` já foi processado anteriormente. Caso positivo, o cliente deve simplesmente ignorar o processamento do payload e retornar HTTP `200 OK` de imediato para indicar o recebimento bem-sucedido.

---

## Alternativas Consideradas

### 1. Garantia Exactly-Once (Exatamente Uma Vez)
* **Descrição:** Garantir matematicamente que o cliente receba a mensagem apenas uma única vez na história da plataforma.
* **Razão do Descarte:** Inviável sobre HTTP. Exigiria mecanismos complexos de consenso distribuído (como Two-Phase Commit), impactando severamente a performance e a simplicidade técnica da integração com sistemas terceiros legados dos clientes B2B.

### 2. Garantia At-Most-Once (No Máximo Uma Vez)
* **Descrição:** Disparar o webhook apenas uma vez. Se falhar, não tentar de novo.
* **Razão do Descarte:** Descartado porque violaria o requisito de confiabilidade de entrega. Se o cliente estiver temporariamente inacessível, ele perderia permanentemente a notificação da mudança de status do pedido, mantendo os sistemas inconsistentes.

---

## Consequências

### Positivas
* **Confiabilidade:** Nenhuma notificação de mudança de status de pedido bem-sucedida será perdida silenciosamente; a entrega é garantida mesmo sob instabilidade temporária.
* **Simplicidade do Worker:** O worker não precisa coordenar transações distribuídas com clientes externos; basta retransmitir em caso de dúvida de recebimento.
* **Alinhamento com Padrões de Mercado:** Segue as melhores práticas adotadas por APIs de grande porte como Stripe, GitHub e Twilio.

### Negativas
* **Esforço do Cliente:** Exige que os clientes implementem lógica de deduplicação/idempotência baseada no `X-Event-Id` em seus receptores de webhook.
