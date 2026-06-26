# ADR-003: Autenticação HMAC-SHA256 com Secret por Endpoint

* **Status:** Aceito
* **Data:** 2026-06-26
* **Autor:** Felipe Fonseca

---

## Contexto e Problema

Ao despachar notificações contendo dados de pedidos B2B sobre a internet pública, enfrentamos riscos de segurança significativos:
1. **Falsificação de Evento (Spoofing):** Um agente malicioso poderia simular nossa API e enviar payloads falsos para os servidores dos clientes.
2. **Adulteração de Payload (Tampering):** Um intermediário de rede (Man-in-the-Middle) poderia adulterar os dados da notificação no trajeto.
3. **Vazamento de Segredo Global:** Se usarmos uma única chave secreta global para todos os clientes, o vazamento dessa chave por parte de um único cliente comprometeria a segurança de todas as integrações da plataforma.

Precisamos de um método robusto que permita ao cliente verificar a **autenticidade** (garantir que a mensagem veio da nossa API) e a **integridade** (garantir que a mensagem não foi alterada) do payload recebido, mantendo o controle de acessos isolado por endpoint.

---

## Decisão

Adotaremos a assinatura de payload usando **HMAC com o algoritmo hash SHA-256 (HMAC-SHA256)**, com uma chave secreta (*secret*) gerada de forma única para cada endpoint de webhook cadastrado.

### 1. Geração de Chaves e Armazenamento
* Para cada webhook criado, a API gerará um segredo criptograficamente seguro com comprimento de 32 bytes (codificado em hexadecimal ou base64).
* Este segredo é armazenado no banco de dados. No cadastro, ele é exibido uma única vez ao cliente.

### 2. Fluxo de Assinatura
No momento do disparo, o worker de webhooks:
1. Extrai o corpo (body) do request serializado em string JSON.
2. Calcula o hash HMAC-SHA256 usando o segredo do endpoint correspondente.
3. Envia o valor resultante no cabeçalho HTTP **`X-Signature`** (em codificação hexadecimal).
4. Envia o timestamp do disparo no cabeçalho **`X-Timestamp`** para proteção contra replay attacks (o cliente pode recusar requisições antigas).

### 3. Rotação de Segredos com Período de Tolerância (*Grace Period*)
Para apoiar boas práticas de segurança e permitir a recuperação de vazamentos:
* A API fornecerá um endpoint para rotacionar a chave secreta: `POST /webhooks/:id/rotate-secret`.
* Quando rotacionada, a **secret antiga permanecerá ativa em paralelo por 24 horas**. O worker enviará duas assinaturas separadas por vírgula no cabeçalho `X-Signature` (uma gerada com a chave ativa e outra com a chave antiga) ou aceitará ambas durante o intervalo de tolerância de 24 horas, permitindo que o cliente migre sua chave sem sofrer indisponibilidade no recebimento de eventos.
* Após 24 horas, a chave antiga é permanentemente removida.

### 4. Uso Obrigatório de HTTPS
Apenas URLs com protocolo seguro HTTPS serão aceitas no cadastro de webhooks. Essa validação será aplicada diretamente no schema de validação do Zod durante a criação/edição do webhook, prevenindo o tráfego de payloads em texto puro na rede.

---

## Alternativas Consideradas

### 1. Autenticação Básica (Headers HTTP Fixos com Token de Autenticação)
* **Descrição:** O cliente cadastra um token estático que nossa API envia no header `Authorization`.
* **Razão do Descarte:** Embora simples, tokens fixos de cabeçalho não protegem contra ataques de adulteração (tampering) no payload e são facilmente expostos caso o cliente registre os cabeçalhos de entrada em logs desprotegidos.

### 2. HMAC com Chave Única Global da Plataforma
* **Descrição:** Usar a mesma chave de criptografia para assinar as requisições de todos os clientes.
* **Razão do Descarte:** Apresenta risco sistêmico inaceitável. O comprometimento ou vazamento da chave secreta por um único cliente comprometeria a validação de assinatura de todos os outros B2B, forçando uma rotação manual caótica e coordenada em massa.

---

## Consequências

### Positivas
* **Não-Repúdio e Autenticidade:** O cliente tem certeza absoluta de que a requisição partiu do nosso sistema e que o payload recebido está intacto.
* **Segurança Isolada:** Se um cliente expuser acidentalmente sua secret, apenas a integração dele estará vulnerável, podendo ser corrigida isoladamente via rotação de segredo.
* **Resiliência operacional:** A rotação com grace period de 24 horas elimina o downtime de integração durante processos de segurança preventiva.

### Negativas
* **Custo Computacional:** O cálculo de HMAC-SHA256 adiciona um leve overhead de processamento de CPU no worker antes de cada envio (desprezível para a volumetria esperada).
* **Esforço do Cliente:** O desenvolvedor do lado do cliente precisará codificar a validação do HMAC-SHA256 em sua aplicação, exigindo documentação técnica clara e exemplos no portal de desenvolvedores.
