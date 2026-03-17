# Pagar.me API v5 — Guia Completo de Implementação

> **Propósito:** Documento autocontido para que uma IA (Claude Code, Cursor, GPT etc.) implemente do zero um ecossistema completo de checkout e verificação de pagamentos com a API Pagar.me v5 — sem consultar nenhuma outra fonte.

**Base URL:** `https://api.pagar.me/core/v5`  
**Versão:** v5 (2021-09-01)  
**Modelos suportados:** Gateway e PSP Pagar.me

---

## Índice

1. [Autenticação](#1-autenticação)
2. [Erros e Códigos HTTP](#2-erros-e-códigos-http)
3. [Objeto address](#3-objeto-address)
4. [Objeto phones](#4-objeto-phones)
5. [Clientes (Customers)](#5-clientes-customers)
6. [Cartões (Cards) e Wallet](#6-cartões-cards-e-wallet)
7. [Token de Cartão](#7-token-de-cartão)
8. [Endereços (Addresses)](#8-endereços-addresses)
9. [Criar Pedido — Cartão de Crédito](#9-criar-pedido--cartão-de-crédito)
10. [Criar Pedido — Pix](#10-criar-pedido--pix)
11. [Obter Pedido](#11-obter-pedido)
12. [Sistema de Webhooks](#12-sistema-de-webhooks)
13. [Tabela de Status](#13-tabela-de-status)
14. [Fluxo Completo de Implementação](#14-fluxo-completo-de-implementação)
15. [Regras de Negócio Críticas](#15-regras-de-negócio-críticas)
16. [Referência de Endpoints](#16-referência-de-endpoints)

---

## 1. Autenticação

### Como funciona

A API usa **HTTP Basic Auth**. A `secret_key` é enviada como usuário e a senha é sempre **vazia**. O dois-pontos final é obrigatório.

```
Authorization: Basic {base64("sk_live_XXXXXXXX:")}
```

**Node.js — montando o header:**
```javascript
const auth = Buffer.from("sk_test_XXXXXXXX:").toString("base64");
// Header: `Authorization: Basic ${auth}`
```

**Cabeçalhos obrigatórios em toda requisição:**
```http
Authorization: Basic {BASE64_SECRET_KEY}
Content-Type: application/json
Accept: application/json
```

### Tipos de chave

| Chave | Prefixo Sandbox | Prefixo Produção | Uso |
|-------|-----------------|------------------|-----|
| Secreta (SK) | `sk_test_*` | `sk_*` | Chamadas server-side. Nunca expor no front-end. |
| Pública (PK) | `pk_test_*` | `pk_*` | Tokenização de cartão no front-end via `/tokens`. |

> ⚠️ **NUNCA compartilhe a Secret Key.** Ela não deve aparecer no código do front-end, em logs ou ser enviada para o cliente.

Sandbox e Produção usam o **mesmo endpoint** `https://api.pagar.me/core/v5`. O que diferencia é o tipo da chave enviada.

---

## 2. Erros e Códigos HTTP

### Tabela de HTTP Status Code

| Código | Status | Definição |
|--------|--------|-----------|
| `200` | OK | Sucesso |
| `400` | Bad Request | Requisição inválida |
| `401` | Unauthorized | Chave de API inválida |
| `403` | Forbidden | Bloqueio por IP/Domínio |
| `404` | Not Found | Recurso não existe |
| `412` | Precondition Failed | Parâmetros válidos mas a requisição falhou |
| `422` | Unprocessable Entity | Parâmetros inválidos ou regra de negócio violada |
| `429` | Too Many Requests | Rate limit atingido |
| `500` | Internal Server Error | Erro interno Pagar.me |

### Estrutura de resposta de erro

```json
{
  "message": "The request is invalid.",
  "errors": {
    "order.customer.name": ["The name field is required."],
    "order.payments[0].credit_card.card": ["The number field is not a valid card number"]
  }
}
```

### Erros comuns

| Código | Mensagem | Causa |
|--------|----------|-------|
| `404` | `Customer not found.` | `customer_id` inválido ou `customer` ausente |
| `422` | `The name field is required.` | Campo `name` ausente no `customer` |
| `422` | `The number field is not a valid card number` | Número de cartão inválido |
| `422` | `The field number must be a string with a minimum length of 13...` | Número de cartão com tamanho incorreto |
| `422` | `The items field is required` | Array `items` ausente no pedido |
| `422` | `At least one customer phone is required.` | `phones` ausente no `customer` (obrigatório para PSP) |
| `412` | `Could not create credit card. The card verification failed.` | Verificação de cartão ativa e cartão inválido |

> **PSP vs Gateway:** A integração PSP exige mais campos obrigatórios (telefone, endereço completo). Sempre verifique qual modelo o cliente usa.

---

## 3. Objeto `address`

Usado em: `customer.address`, `billing_address` no cartão, `shipping.address`.

### Atributos

| Atributo | Tipo | Obrigatório | Descrição |
|----------|------|:-----------:|-----------|
| `id` | string | — | Formato: `addr_XXXXXXXXXXXXXXXX` (gerado pela API) |
| `line_1` | string | Sim | **Número, Rua, Bairro** — nesta ordem, separados por vírgula |
| `line_2` | string | Não | Complemento: andar, apto, sala etc. |
| `zip_code` | string | Sim | CEP — apenas numérico. Máx: 16 chars |
| `city` | string | Sim | Cidade. Máx: 64 chars |
| `state` | string | Sim | Código ISO 3166-2 (ex: `SP`, `RJ`) |
| `country` | string | Sim | Código ISO 3166-1 alpha-2 (ex: `BR`, `US`) |
| `status` | enum | — | `active` ou `deleted` |
| `metadata` | object | Não | Chave/valor livre |

> ❗️ **Formatação obrigatória do `line_1`:** deve seguir exatamente o formato `Número, Rua, Bairro`. Necessário para antifraude e boleto com registro.

```json
{
  "line_1": "375, Av. General Justo, Centro",
  "line_2": "7º andar, sala 01",
  "zip_code": "20021130",
  "city": "Rio de Janeiro",
  "state": "RJ",
  "country": "BR"
}
```

> ⚠️ Os atributos `street`, `number`, `complement` e `neighborhood` estão **descontinuados**. Não usar.

> 🌍 Clientes com **passaporte** podem usar endereços internacionais, substituindo o CEP pelo ZIP Code do país. Não é possível misturar passaporte com endereço nacional.

---

## 4. Objeto `phones`

Obrigatório para clientes PSP. Usado dentro do objeto `customer`.

```json
"phones": {
  "home_phone": {
    "country_code": "55",
    "area_code": "11",
    "number": "000000000"
  },
  "mobile_phone": {
    "country_code": "55",
    "area_code": "11",
    "number": "000000000"
  }
}
```

| Atributo | Tipo | Descrição |
|----------|------|-----------|
| `home_phone` | object | Telefone residencial |
| `mobile_phone` | object | Telefone celular |
| `country_code` | string | Código do país — apenas numérico (ex: `55`) |
| `area_code` | string | DDD — apenas numérico (ex: `11`) |
| `number` | string | Número — apenas numérico |

---

## 5. Clientes (Customers)

### 5.1 Objeto `customer`

| Atributo | Tipo | Descrição |
|----------|------|-----------|
| `id` | string | Formato: `cus_XXXXXXXXXXXXXXXX` |
| `name` | string | Nome completo. Máx: 64 chars |
| `email` | string | E-mail único por conta. Máx: 64 chars |
| `document` | string | CPF, CNPJ ou PASSPORT. Máx: 16 (CPF/CNPJ) ou 50 (PASSPORT) |
| `document_type` | string | `CPF`, `CNPJ` ou `PASSPORT` |
| `type` | enum | `individual` (pessoa física) ou `company` (jurídica). Obrigatório se `document` enviado |
| `gender` | string | `male` ou `female` |
| `phones` | object | Ver seção 4. **Obrigatório para PSP** |
| `address` | object | Ver seção 3 |
| `birthdate` | datetime | Formato: `mm/dd/aa` |
| `code` | string | Referência interna da loja. Máx: 52 chars |
| `delinquent` | boolean | Cliente inadimplente |
| `metadata` | object | Chave/valor livre |
| `created_at` | datetime | UTC |
| `updated_at` | datetime | UTC |

### 5.2 Criar Cliente — `POST /customers`

> ⚠️ O **e-mail é único**: criar um cliente com e-mail já cadastrado **atualiza** o customer existente.

> 🚧 Clientes com **Passaporte** na integração PSP só transacionam com endereços internacionais.

**Body Params:**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|:-----------:|-----------|
| `name` | string | PSP: Sim | Nome. Máx: 64 chars |
| `email` | string | PSP: Sim | E-mail único. Máx: 64 chars |
| `document` | string | PSP: Sim | CPF, CNPJ ou PASSPORT |
| `document_type` | string | Condicional | Obrigatório se `document` enviado |
| `type` | string | Condicional | Obrigatório se `document` enviado |
| `code` | string | Não | Referência da loja. Máx: 52 chars |
| `gender` | string | Não | `male` ou `female` |
| `birthdate` | date | Não | Formato: `mm/dd/aaa` |
| `phones` | object | **PSP: Sim** | Ver seção 4 |
| `address` | object | Não | Ver seção 3 |
| `metadata` | string | Não | Chave/valor |

```bash
curl --request POST \
  --url https://api.pagar.me/core/v5/customers \
  --header 'authorization: Basic {BASE64_SK}' \
  --header 'content-type: application/json' \
  --data '{
    "name": "Tony Stark",
    "email": "tony@avengers.com",
    "document": "01234567890",
    "document_type": "CPF",
    "type": "individual",
    "phones": {
      "mobile_phone": {
        "country_code": "55",
        "area_code": "11",
        "number": "999999999"
      }
    }
  }'
```

### 5.3 Obter Cliente — `GET /customers/{customer_id}`

```bash
curl --request GET \
  --url https://api.pagar.me/core/v5/customers/{customer_id} \
  --header 'authorization: Basic {BASE64_SK}'
```

### 5.4 Editar Cliente — `PUT /customers/{customer_id}`

> ❗️ **Substitui todos os dados.** Campos não enviados viram `null`. Se o cliente tem telefone e você não enviar `phones`, o telefone será apagado. Sempre envie o objeto completo.

Body Params: mesmos campos do Criar Cliente.

### 5.5 Listar Clientes — `GET /customers`

**Query Params:**

| Parâmetro | Tipo | Padrão | Descrição |
|-----------|------|--------|-----------|
| `name` | string | — | Filtro por nome |
| `document` | string | — | CPF ou CNPJ |
| `email` | string | — | Filtro por e-mail |
| `gender` | string | — | `male` ou `female` |
| `code` | string | — | Código da loja |
| `page` | int32 | `1` | Página atual |
| `size` | int32 | `10` | Itens por página |

---

## 6. Cartões (Cards) e Wallet

O conjunto de cartões de um cliente forma sua **Wallet**.

### 6.1 Objeto `card`

| Atributo | Tipo | Descrição |
|----------|------|-----------|
| `id` | string | Formato: `card_XXXXXXXXXXXXXXXX` |
| `last_four_digits` | string | Últimos 4 dígitos |
| `first_six_digits` | string | Primeiros 6 dígitos (BIN) |
| `brand` | string | Crédito: `Elo`, `Mastercard`, `Visa`, `Amex`, `JCB`, `Aura`, `Hipercard`, `Diners`, `Discover`. Voucher: `VR`, `Pluxee`, `Ticket`, `Alelo` |
| `holder_name` | string | Nome impresso no cartão |
| `holder_document` | string | CPF ou CNPJ do portador |
| `exp_month` | integer | Mês de vencimento |
| `exp_year` | integer | Ano de vencimento (`yy` ou `yyyy`) |
| `status` | enum | `active`, `deleted` ou `expired` |
| `billing_address` | object | Endereço de cobrança |
| `customer` | object | Cliente associado |
| `private_label` | boolean | Se é cartão private label |
| `type` | string | `credit` ou `voucher` |
| `metadata` | object | Chave/valor livre |

### 6.2 Criar Cartão — `POST /customers/{customer_id}/cards`

> 🚧 Mesmo cartão cadastrado novamente retorna o `card_id` já existente (sem duplicar).
> ❗️ `brand` é **obrigatório** para cartões Private Label (`private_label: true`).

**Body Params:**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|:-----------:|-----------|
| `number` | string | Sim | 13 a 19 dígitos |
| `holder_name` | string | Sim | Nome como impresso. Máx 64 chars. Sem números/especiais |
| `holder_document` | string | Voucher: Sim | CPF/CNPJ. Obrigatório para bandeiras VR e Pluxee |
| `exp_month` | int32 | Sim | 1–12 |
| `exp_year` | int32 | Sim | Formato `yy` ou `yyyy` |
| `cvv` | string | Sim | 3 ou 4 chars |
| `brand` | string | Private Label: Sim | `elo`, `mastercard`, `visa`, `amex`, `jcb`, `aura`, `hipercard`, `diners`, `unionpay`, `discover` |
| `billing_address_id` | string | Não | Alternativo ao `billing_address` |
| `billing_address` | object | Não | Ver seção 3 |
| `token` | string | Não | Token gerado via `/tokens` — substitui os dados do cartão |
| `metadata` | string | Não | Chave/valor |

### 6.3 Obter Cartão — `GET /customers/{customer_id}/cards/{card_id}`

### 6.4 Listar Cartões (Wallet) — `GET /customers/{customer_id}/cards`

### 6.5 Editar Cartão — `PUT /customers/{customer_id}/cards/{card_id}`

**Body Params:**

| Campo | Tipo | Obrigatório |
|-------|------|:-----------:|
| `exp_month` | int32 | Sim |
| `exp_year` | int32 | Sim |
| `holder_name` | string | Não |
| `holder_document` | string | Voucher: Sim |
| `billing_address_id` | string | Não |

### 6.6 Excluir Cartão — `DELETE /customers/{customer_id}/cards/{card_id}`

Remove o cartão da Wallet do cliente.

### 6.7 Renovar Cartão — `POST /customers/{customer_id}/cards/{card_id}/renew`

Renova o cartão manualmente via Card Updater junto à adquirente.

---

## 7. Token de Cartão

**`POST /tokens?appId={public_key}`**

Tokeniza dados de cartão **diretamente do front-end** usando a **Chave Pública (PK)**.

> ❗️ Use a **PK**, nunca a SK. A SK não deve aparecer no front-end.
> ❗️ **Não inclua o cabeçalho `Authorization`** neste endpoint. Apenas `Content-Type` é permitido.
> ❗️ O domínio deve estar cadastrado no painel Pagar.me.
> 🚧 O `billing_address` **não é tokenizado** — enviar separadamente ao criar o pedido.
> O token é de **uso único** (single-use).

**Query Params:**

| Param | Tipo | Obrigatório |
|-------|------|:-----------:|
| `appId` | string | Sim — Chave pública (`pk_test_*` ou `pk_*`) |

**Body Params:**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|:-----------:|-----------|
| `type` | string | Sim | Tipo do cartão (ex: `card`) |
| `card.number` | string | Sim | Número do cartão |
| `card.holder_name` | string | Sim | Nome no cartão |
| `card.exp_month` | integer | Sim | Mês de validade |
| `card.exp_year` | integer | Sim | Ano de validade |
| `card.cvv` | string | Sim | CVV |

```javascript
// Front-end — PK, SEM Authorization header
const response = await fetch(
  `https://api.pagar.me/core/v5/tokens?appId=${PUBLIC_KEY}`,
  {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      type: "card",
      card: {
        number: "4000000000000010",
        holder_name: "Tony Stark",
        exp_month: 1,
        exp_year: 30,
        cvv: "3531"
      }
    })
  }
);
const { id: cardToken } = await response.json();
// Enviar apenas cardToken ao back-end
```

**Resposta:**
```json
{
  "id": "token_XXXXXXXXXXXXXXXX",
  "type": "card",
  "created_at": "2023-01-01T00:00:00Z",
  "expires_at": "2023-01-01T00:30:00Z",
  "card": {
    "first_six_digits": "400000",
    "last_four_digits": "0010",
    "holder_name": "Tony Stark",
    "exp_month": 1,
    "exp_year": 30,
    "brand": "Visa"
  }
}
```

---

## 8. Endereços (Addresses)

Endereços associados a customers, gerenciados de forma independente.

### 8.1 Criar Endereço — `POST /customers/{customer_id}/addresses`

> 🚧 Clientes com passaporte: usar ZIP Code internacional. Não misturar passaporte com endereço nacional.

**Body Params:**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|:-----------:|-----------|
| `line_1` | string | Sim | Número, Rua, Bairro (nesta ordem, separados por vírgula). Máx: 256 chars |
| `line_2` | string | Não | Complemento. Máx: 128 chars |
| `zip_code` | string | Sim | CEP numérico. Máx: 16 chars |
| `city` | string | Sim | Cidade. Máx: 64 chars |
| `state` | string | Sim | ISO 3166-2 (ex: `SP`) |
| `country` | string | Sim | ISO 3166-1 alpha-2 (ex: `BR`) |
| `metadata` | string | Não | Chave/valor |

### 8.2 Obter Endereço — `GET /customers/{customer_id}/addresses/{address_id}`

Formato do `address_id`: `addr_XXXXXXXXXXXXXXXX`

```bash
curl --request GET \
  --url https://api.pagar.me/core/v5/customers/{customer_id}/addresses/{address_id} \
  --header 'authorization: Basic {BASE64_SK}'
```

### 8.3 Editar Endereço — `PUT /customers/{customer_id}/addresses/{address_id}`

> ⚠️ **Atenção:** O endpoint de edição permite alterar **apenas** `line_2` e `metadata`. Os campos principais (`line_1`, `zip_code`, `city`, `state`, `country`) **não podem ser alterados após a criação**. Para mudar esses dados, exclua e crie um novo endereço.

**Body Params:**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|:-----------:|-----------|
| `line_2` | string | Não | Complemento (andar, sala, apto). Máx: 128 chars |
| `metadata` | string | Não | Chave/valor |

```bash
curl --request PUT \
  --url https://api.pagar.me/core/v5/customers/{customer_id}/addresses/{address_id} \
  --header 'authorization: Basic {BASE64_SK}' \
  --header 'content-type: application/json' \
  --data '{ "line_2": "8º andar, sala 02" }'
```

### 8.4 Listar Endereços — `GET /customers/{customer_id}/addresses`

Retorna todos os endereços de um cliente. Suporta paginação.

**Query Params:**

| Parâmetro | Tipo | Padrão | Descrição |
|-----------|------|--------|-----------|
| `page` | int32 | `1` | Página atual |
| `size` | int32 | `10` | Itens por página |

```bash
curl --request GET \
  --url 'https://api.pagar.me/core/v5/customers/{customer_id}/addresses?page=1&size=10' \
  --header 'authorization: Basic {BASE64_SK}'
```

### 8.5 Excluir Endereço — `DELETE /customers/{customer_id}/addresses/{address_id}`

Remove o endereço do cliente.

```bash
curl --request DELETE \
  --url https://api.pagar.me/core/v5/customers/{customer_id}/addresses/{address_id} \
  --header 'authorization: Basic {BASE64_SK}'
```

---

## 9. Criar Pedido — Cartão de Crédito

**`POST /orders`**

> ❗️ **Não envie dados abertos do cartão** sem PCI Compliance. Use `card_id` (PSP) ou `card_token` (Gateway).
> ❗️ **`customer` ou `customer_id` são obrigatórios.** PSP: todos os campos do customer são obrigatórios.
> 🚧 **CVV em recorrência:** somente na primeira transação (`recurrence_cycle = "first"`).

### 9.1 Body Params do pedido

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|:-----------:|-----------|
| `code` | string | Não | Referência da loja. Máx: 52 chars |
| `customer` | object | Condicional | Obrigatório se `customer_id` ausente |
| `customer_id` | string | Condicional | Obrigatório se `customer` ausente |
| `items` | array | Sim | Itens do pedido |
| `payments` | array | Sim | Dados de pagamento |
| `shipping` | object | Não | Dados de entrega |
| `closed` | boolean | Não | Padrão: `true` |
| `ip` | string | Não | IP do dispositivo (IPv4 ou IPv6) |
| `session_id` | string | Não | ID de sessão |
| `antifraud_enabled` | boolean | Não | Se ausente, usa configuração da conta |
| `SubMerchant` | object | Não | Facilitador de pagamento |

### 9.2 Propriedades do objeto `credit_card`

| Atributo | Tipo | Obrigatório | Descrição |
|----------|------|:-----------:|-----------|
| `installments` | integer | Não | Parcelas. Padrão: `1` |
| `statement_descriptor` | string | Não | Texto na fatura. Máx 13 (PSP) / 22 (Gateway) |
| `operation_type` | string | Não | `auth_and_capture` (padrão), `auth_only`, `pre_auth` |
| `card_id` | string | Condicional | ID de cartão salvo — recomendado para PSP |
| `card_token` | string | Condicional | Token single-use — apenas Gateway |
| `card` | object | Condicional | Dados abertos — exige PCI |
| `network_token` | object | Condicional | Token de bandeira — apenas Gateway |
| `billing_address` | object | **Sim** | Endereço de cobrança — obrigatório sempre |
| `recurrence_cycle` | string | Não | `first` ou `subsequent` |
| `metadata` | object | Não | Chave/valor livre |
| `authentication` | object | Condicional | Para transações 3DS |
| `auto_recovery` | boolean | Não | Desabilita retentativa offline |
| `funding_source` | string | Não | `credit`, `debit` ou `prepaid` |
| `initiated_type` | string | Não | `partial_shipment`, `related_or_delayed_charge`, `no_show` ou `retry` |
| `recurrence_model` | string | Não | `standing_order`, `instalment` ou `subscription` |
| `channel` | string | Condicional | `payment_link` — obrigatório para Elo a partir de 17/10/2025 |

### 9.3 Status da transação (`charge.last_transaction.status`)

| Status | Descrição |
|--------|-----------|
| `authorized_pending_capture` | Autorizada, pendente de captura |
| `not_authorized` | Não autorizada |
| `captured` | **Capturada** ✅ pagamento confirmado |
| `partial_capture` | Captura parcial |
| `waiting_capture` | Aguardando captura |
| `refunded` | Estornada |
| `voided` | Cancelada |
| `partial_refunded` | Estornada parcialmente |
| `partial_void` | Cancelada parcialmente |
| `error_on_voiding` | Erro no cancelamento |
| `error_on_refunding` | Erro no estorno |
| `waiting_cancellation` | Aguardando cancelamento |
| `with_error` | Com erro |
| `failed` | Falha |

### 9.4 Autenticação 3DS

Disponível apenas para Gateway. Usar versões 2.1.0 ou 2.2.0 (v1.0 descontinuada).

```json
"authentication": {
  "type": "threed_secure",
  "threed_secure": {
    "mpi": "third_party",
    "eci": "05",
    "cavv": "AAABBBCCC...",
    "transaction_id": "abc123",
    "ds_transaction_id": "xyz456",
    "version": "2.2.0"
  }
}
```

Campos obrigatórios quando `mpi = "third_party"`: `eci`, `cavv`, `transaction_id`.

### 9.5 Exemplo de Request — Crédito com `card_id` (PSP)

```json
{
  "customer_id": "cus_xxxxxxxxxxxxxxxx",
  "items": [
    { "amount": 2990, "description": "Chaveiro do Tesseract", "quantity": 1, "code": "SKU-001" }
  ],
  "payments": [
    {
      "payment_method": "credit_card",
      "credit_card": {
        "installments": 1,
        "statement_descriptor": "MINHA LOJA",
        "card_id": "card_xxxxxxxxxxxxxxxx",
        "billing_address": {
          "line_1": "375, Av. Paulista, Bela Vista",
          "zip_code": "01310100",
          "city": "São Paulo",
          "state": "SP",
          "country": "BR"
        }
      }
    }
  ],
  "closed": true
}
```

### 9.6 Exemplo de Request — Crédito com dados abertos (requer PCI)

```json
{
  "customer": { "name": "Tony Stark", "email": "tony@avengers.com" },
  "items": [{ "amount": 2990, "description": "Produto", "quantity": 1 }],
  "payments": [
    {
      "payment_method": "credit_card",
      "credit_card": {
        "installments": 1,
        "statement_descriptor": "AVENGERS",
        "card": {
          "number": "4000000000000010",
          "holder_name": "Tony Stark",
          "exp_month": 1,
          "exp_year": 30,
          "cvv": "3531",
          "billing_address": {
            "line_1": "10880, Malibu Point, Malibu Central",
            "zip_code": "90265",
            "city": "Malibu",
            "state": "CA",
            "country": "US"
          }
        }
      }
    }
  ]
}
```

### 9.7 Exemplo de Response — Crédito

```json
{
  "id": "or_DNobwn2CpGuvw0zp",
  "code": "GP8KUL0B2D",
  "amount": 2990,
  "currency": "BRL",
  "closed": true,
  "status": "pending",
  "created_at": "2023-03-03T19:49:14Z",
  "charges": [
    {
      "id": "ch_p4lnAGyU0GT1E9MZ",
      "amount": 2990,
      "status": "pending",
      "payment_method": "credit_card",
      "last_transaction": {
        "id": "tran_xxxxxxxxxxxxxxxx",
        "transaction_type": "credit_card",
        "status": "authorized_pending_capture",
        "amount": 2990,
        "installments": 1,
        "acquirer_name": "redecard",
        "acquirer_auth_code": "236689"
      }
    }
  ]
}
```

---

## 10. Criar Pedido — Pix

**`POST /orders`**

> 📘 Disponível apenas para contas com gateway **Pagar.me**.
> 📘 Para estornar Pix: `DELETE /charges/{charge_id}` (não o pedido).

### 10.1 Propriedades do objeto `pix`

| Atributo | Tipo | Obrigatório | Descrição |
|----------|------|:-----------:|-----------|
| `expires_in` | integer | Condicional | Em segundos. Obrigatório se `expires_at` ausente |
| `expires_at` | datetime | Condicional | Formato: `YYYY-MM-DDThh:mm:ss`. Obrigatório se `expires_in` ausente. Máximo: 10 anos |
| `additional_information` | array | Não | Cada item: `{ "name": "string", "value": "string" }` |

### 10.2 Exemplo de Request — Pix

```json
{
  "items": [{ "amount": 2990, "description": "Chaveiro do Tesseract", "quantity": 1 }],
  "customer": {
    "name": "Tony Stark",
    "email": "tony@avengers.com",
    "type": "individual",
    "document": "01234567890",
    "phones": {
      "home_phone": { "country_code": "55", "area_code": "21", "number": "22180513" }
    }
  },
  "payments": [
    {
      "payment_method": "pix",
      "pix": {
        "expires_in": 3600,
        "additional_information": [{ "name": "Pedido", "value": "001" }]
      }
    }
  ]
}
```

### 10.3 Campos críticos da Response — Pix

```json
{
  "id": "or_56GXnk6T0eU88qMm",
  "status": "pending",
  "charges": [
    {
      "id": "ch_K2rJ5nlHwTE4qRDP",
      "status": "pending",
      "payment_method": "pix",
      "last_transaction": {
        "status": "waiting_payment",
        "qr_code": "00020101021226...",
        "qr_code_url": "https://api.pagar.me/core/v5/transactions/tran_xxx/qrcode",
        "expires_at": "2023-01-01T01:00:00Z",
        "pix_provider_tid": "E1234..."
      }
    }
  ]
}
```

- `qr_code` → string copia-e-cola (exibir ao usuário)
- `qr_code_url` → URL da imagem do QR Code
- `expires_at` → quando o QR Code expira
- `status: "waiting_payment"` → aguardando pagamento

### 10.4 Pix com Split

```json
"payments": [
  {
    "payment_method": "pix",
    "pix": { "expires_in": 3600 },
    "split": [
      {
        "amount": 50, "recipient_id": "rp_xxxxxxxxxxxxxxxx", "type": "percentage",
        "options": { "charge_processing_fee": true, "charge_remainder_fee": true, "liable": true }
      },
      {
        "amount": 50, "recipient_id": "rp_yyyyyyyyyyyyyyyy", "type": "percentage",
        "options": { "charge_processing_fee": false, "charge_remainder_fee": false, "liable": false }
      }
    ]
  }
]
```

---

## 11. Obter Pedido

**`GET /orders/{order_id}`**

| Path Param | Tipo | Obrigatório | Descrição |
|------------|------|:-----------:|-----------|
| `order_id` | string | Sim | Prefixo `or_`. Máx: 52 chars |

```bash
curl --request GET \
  --url https://api.pagar.me/core/v5/orders/{order_id} \
  --header 'authorization: Basic {BASE64_SK}'
```

### Estratégia de polling para Pix

- Primeiros 2 minutos: a cada **5 segundos**
- Após 2 minutos até `expires_at`: a cada **15 segundos**
- Parar ao detectar `status = "paid"` ou após expiração
- **Prefira webhooks ao polling** — polling é apenas fallback

---

## 12. Sistema de Webhooks

O Pagar.me envia `POST` automáticos para uma URL configurada no painel.

### 12.1 Estrutura raiz do webhook

```json
{
  "id": "hook_RyEKQO789TRpZjv5",
  "account": { "id": "acc_jZkdN857et650oNv", "name": "Lojinha" },
  "type": "order.paid",
  "created_at": "2022-06-29T20:23:47",
  "data": { }
}
```

| Campo | Descrição |
|-------|-----------|
| `id` | ID único do webhook — use para idempotência |
| `account.id` | ID da conta — valide para autenticidade |
| `type` | Tipo do evento |
| `data` | Objeto completo do recurso afetado |

### 12.2 Exemplo completo — Webhook `order.paid`

```json
{
  "id": "hook_RyEKQO789TRpZjv5",
  "account": { "id": "acc_jZkdN857et650oNv", "name": "Lojinha" },
  "type": "order.paid",
  "created_at": "2022-06-29T20:23:47",
  "data": {
    "id": "or_ZdnB5BBCmYhk534R",
    "code": "1303724",
    "amount": 12356,
    "currency": "BRL",
    "closed": true,
    "status": "paid",
    "created_at": "2022-06-29T20:23:42",
    "updated_at": "2022-06-29T20:23:47",
    "closed_at": "2022-06-29T20:23:44",
    "items": [
      { "id": "oi_EqnMMrbFgBf0MaN1", "description": "Produto", "amount": 10166, "quantity": 1, "status": "active" }
    ],
    "customer": {
      "id": "cus_oy23JRQCM1cvzlmD", "name": "Fabio", "email": "fabio@email.com",
      "document": "09006068709", "type": "individual", "delinquent": false
    },
    "shipping": {
      "amount": 2190, "description": "Economico",
      "address": { "zip_code": "90265", "city": "Malibu", "state": "CA", "country": "US", "line_1": "10880, Malibu Point, Malibu Central" }
    },
    "charges": [
      {
        "id": "ch_d22356Jf4WuGr8no",
        "amount": 12356, "status": "paid", "payment_method": "credit_card", "paid_at": "2022-06-29T20:23:47",
        "last_transaction": {
          "id": "tran_opAqDj2390S1lKQO", "transaction_type": "credit_card",
          "status": "captured", "success": true, "amount": 12356, "installments": 2,
          "acquirer_name": "redecard", "acquirer_auth_code": "236689", "operation_type": "capture",
          "card": {
            "id": "card_BjKOmahgAf0D23lw", "last_four_digits": "4485", "brand": "Visa",
            "holder_name": "FABIO", "exp_month": 6, "exp_year": 2025, "status": "active", "type": "credit"
          },
          "gateway_response": { "code": "200" }
        }
      }
    ]
  }
}
```

### 12.3 Todos os Eventos de Webhook

#### Pedidos

| Evento | Descrição |
|--------|-----------|
| `order.created` | Pedido criado |
| `order.paid` | Pedido pago ✅ **Principal evento para liberar produto** |
| `order.payment_failed` | Falha no pagamento |
| `order.canceled` | Pedido cancelado |
| `order.closed` | Pedido fechado |
| `order.updated` | Pedido atualizado |
| `order_item.created` | Item criado |
| `order_item.updated` | Item atualizado |
| `order_item.deleted` | Item excluído |

#### Cobranças

| Evento | Descrição |
|--------|-----------|
| `charge.created` | Cobrança criada |
| `charge.paid` | Cobrança paga ✅ **Use para Pix e boleto** |
| `charge.payment_failed` | Falha na cobrança |
| `charge.refunded` | Estornada |
| `charge.partial_canceled` | Parcialmente cancelada |
| `charge.chargedback` | Chargeback recebido |
| `charge.pending` | Cobrança pendente |
| `charge.processing` | Em processamento |
| `charge.underpaid` | Pago a menos |
| `charge.overpaid` | Pago a mais |
| `charge.updated` | Cobrança atualizada |

#### Faturas, Assinaturas e Planos

| Evento | Descrição |
|--------|-----------|
| `invoice.created / updated / paid / payment_failed / canceled` | Faturas |
| `subscription.created / canceled` | Assinaturas |
| `subscription_item.created / updated / deleted` | Itens de assinatura |
| `plan.created / updated / deleted` | Planos |
| `plan_item.created / updated / deleted` | Itens de plano |

#### Clientes, Cartões e Endereços

| Evento | Descrição |
|--------|-----------|
| `customer.created / updated` | Cliente |
| `card.created / updated / deleted / expired` | Cartão |
| `address.created / updated / deleted` | Endereço |

#### Recebedores e Financeiro

| Evento | Descrição |
|--------|-----------|
| `recipient.created / updated / deleted` | Recebedor |
| `bank_account.created` | Conta bancária |
| `usage.created / deleted` | Uso de item de assinatura |
| `discount.created / deleted` | Desconto |
| `increment.created / deleted` | Incremento |

### 12.4 Reenviar Webhook — `POST /hooks/{hook_id}/retry`

```bash
curl --request POST \
  --url https://api.pagar.me/core/v5/hooks/{hook_id}/retry \
  --header 'authorization: Basic {BASE64_SK}'
```

### 12.5 Obter Webhook — `GET /hooks/{hook_id}`

```bash
curl --request GET \
  --url https://api.pagar.me/core/v5/hooks/{hook_id} \
  --header 'authorization: Basic {BASE64_SK}'
```

---

## 13. Tabela de Status

### Prefixos de IDs

| Prefixo | Recurso |
|---------|---------|
| `or_` | Order (pedido) |
| `ch_` | Charge (cobrança) |
| `tran_` | Transaction |
| `cus_` | Customer |
| `card_` | Card |
| `addr_` | Address |
| `hook_` | Webhook |
| `oi_` | Order item |
| `rp_` | Recipient |
| `acc_` | Account |
| `token_` | Token de cartão |

### Status do Pedido (`order.status`)

| Status | Descrição |
|--------|-----------|
| `pending` | Aguardando pagamento |
| `paid` | Pago ✅ |
| `canceled` | Cancelado |
| `failed` | Falhou |

### Status da Cobrança (`charge.status`)

| Status | Descrição |
|--------|-----------|
| `pending` | Aguardando |
| `paid` | Pago ✅ |
| `canceled` | Cancelado |
| `failed` | Falhou |
| `refunded` | Estornado |
| `partial_refunded` | Estorno parcial |
| `overpaid` | Pago a mais |
| `underpaid` | Pago a menos |
| `chargedback` | Chargeback |
| `processing` | Em processamento |

### Mapeamento: evento confirmatório por método

| Método | Evento confirmador | Campo a verificar |
|--------|--------------------|-------------------|
| Cartão | `order.paid` | `order.status == "paid"` |
| Pix | `charge.paid` ou `order.paid` | `charge.status == "paid"` |
| Boleto | `charge.paid` | `charge.status == "paid"` |

### Valores monetários

**TODOS os valores são em centavos (integer).**
- R$ 29,90 → `2990`
- R$ 100,00 → `10000`
- R$ 1.234,56 → `123456`

---

## 14. Fluxo Completo de Implementação

### 14.1 Fluxo — Checkout com Cartão de Crédito

```
1. Front-end: tokenizar cartão via POST /tokens?appId={PK}
   → recebe card_token (single-use)

2. Front-end: enviar card_token + itens ao back-end

3. Back-end: POST /orders com payment_method: "credit_card"
   → usar card_token (Gateway) ou card_id (PSP)
   → incluir billing_address (SEMPRE obrigatório)

4. Verificar resposta imediata:
   - last_transaction.status == "captured" → aprovar
   - "authorized_pending_capture" → aguardar captura
   - "not_authorized" → informar usuário, não prosseguir

5. Aguardar webhook order.paid para confirmação definitiva
   → liberar produto/acesso
```

### 14.2 Fluxo — Checkout com Pix

```
1. Front-end: enviar dados ao back-end

2. Back-end: POST /orders com payment_method: "pix" e expires_in
   → resposta inclui qr_code e qr_code_url

3. Back-end: retornar ao front-end qr_code, qr_code_url, expires_at

4. Front-end: exibir QR Code ao usuário

5. Back-end: polling GET /orders/{id} (ou aguardar webhook)
   - Cada 5s nos primeiros 2min
   - Cada 15s até expires_at
   - Parar ao detectar status "paid" ou expiração

6. Webhook charge.paid confirma definitivamente
   → liberar produto/acesso, parar polling
```

### 14.3 Fluxo — Gerenciamento de Wallet (Cartões Salvos)

```
1. Primeira compra:
   - POST /customers → salvar customer_id
   - POST /customers/{id}/cards → salvar card_id
   - Associar customer_id + card_id ao usuário no banco

2. Compras futuras:
   - Buscar card_id no banco
   - POST /orders com card_id
   - Sem necessidade de novo token

3. Manutenção da Wallet:
   - card.expired webhook → notificar usuário para atualizar
   - PUT /customers/{id}/cards/{card_id} → atualizar validade
   - POST /customers/{id}/cards/{card_id}/renew → Card Updater
   - DELETE /customers/{id}/cards/{card_id} → remover cartão
```

### 14.4 Handler de Webhook (TypeScript)

```typescript
// POST /webhooks/pagarme
async function handlePagarmeWebhook(req: Request, res: Response) {
  // 1. Responder 200 IMEDIATAMENTE
  res.status(200).json({ received: true });

  const { id: hookId, type, data, account } = req.body;

  // 2. Verificar autenticidade
  if (account.id !== process.env.PAGARME_ACCOUNT_ID) return;

  // 3. Idempotência
  const alreadyProcessed = await db.webhooks.findOne({ hookId });
  if (alreadyProcessed) return;
  await db.webhooks.insert({ hookId, type, processedAt: new Date() });

  // 4. Processar assincronamente (fila/worker)
  await queue.push({ hookId, type, data });
}

async function processWebhook({ type, data }: WebhookJob) {
  switch (type) {
    case "order.paid":
      await db.orders.update({ pagarmeId: data.id }, { status: "paid", paidAt: new Date() });
      await email.sendConfirmation(data.customer.email, data);
      await fulfillment.releaseProduct(data.code);
      break;

    case "charge.paid":
      // Crítico para Pix e boleto
      await db.charges.update({ pagarmeId: data.id }, { status: "paid", paidAt: data.paid_at });
      break;

    case "order.payment_failed":
      await db.orders.update({ pagarmeId: data.id }, { status: "failed" });
      await email.sendPaymentFailed(data.customer.email);
      await inventory.releaseReservation(data.code);
      break;

    case "charge.refunded":
      await db.orders.update({ pagarmeId: data.order_id }, { status: "refunded" });
      await fulfillment.revokeAccess(data.order_id);
      break;

    case "charge.chargedback":
      await disputes.openCase(data);
      await alerts.notifyFinanceTeam(data);
      break;

    case "card.expired":
      await email.sendCardExpiredNotification(data.customer?.email);
      break;
  }
}
```

---

## 15. Regras de Negócio Críticas

| Regra | Detalhe |
|-------|---------|
| **Segurança de cartão** | Nunca enviar dados abertos sem PCI. Use `card_id` (PSP) ou `card_token` (Gateway) |
| **`card_token` é single-use** | Expira após uso ou timeout. Gerar novo a cada transação |
| **`billing_address` obrigatório** | Sempre obrigatório em transações de cartão, inclusive com token |
| **E-mail único de customer** | Criar com e-mail existente ATUALIZA o customer existente |
| **`PUT /customers` substitui tudo** | Sempre enviar o objeto completo; campos omitidos viram `null` |
| **CVV em recorrência** | Enviar CVV apenas com `recurrence_cycle = "first"`. Omitir em `subsequent` |
| **Idempotência de webhook** | Verificar `hook.id` antes de processar; responder 200 imediatamente |
| **PSP: dados obrigatórios** | `name`, `email`, `document`, `phones` e endereço são obrigatórios para clientes PSP |
| **Formatação de `line_1`** | Obrigatório: `Número, Rua, Bairro` nesta ordem, separados por vírgula |
| **Campos descontinuados** | Não usar `street`, `number`, `complement`, `neighborhood` no endereço |
| **Pré-autorização** | Requer liberação na adquirente. Capturar depois com `POST /charges/{id}/capture` |
| **Elo + Payment Link** | A partir de 17/10/2025: enviar `"channel": "payment_link"` para Elo |
| **Private Label** | `brand` obrigatório; `holder_document` obrigatório para voucher VR/Pluxee |
| **Rate limit** | `429`: backoff exponencial com jitter. `5xx`: também retentar. `4xx`: corrigir dados |
| **Pix: estorno** | Cancelar a cobrança (`DELETE /charges/{id}`), não o pedido |
| **Token de cartão: sem SK** | Endpoint `/tokens` usa PK no query param. Proibido incluir cabeçalho `Authorization` |

---

## 16. Referência de Endpoints

### Autenticação / Tokens

| Operação | Método | Endpoint |
|----------|--------|----------|
| Criar token de cartão (front-end, PK) | `POST` | `/tokens?appId={pk}` |

### Clientes

| Operação | Método | Endpoint |
|----------|--------|----------|
| Criar cliente | `POST` | `/customers` |
| Obter cliente | `GET` | `/customers/{customer_id}` |
| Editar cliente | `PUT` | `/customers/{customer_id}` |
| Listar clientes | `GET` | `/customers` |

### Cartões

| Operação | Método | Endpoint |
|----------|--------|----------|
| Criar cartão | `POST` | `/customers/{customer_id}/cards` |
| Obter cartão | `GET` | `/customers/{customer_id}/cards/{card_id}` |
| Listar cartões (wallet) | `GET` | `/customers/{customer_id}/cards` |
| Editar cartão | `PUT` | `/customers/{customer_id}/cards/{card_id}` |
| Excluir cartão | `DELETE` | `/customers/{customer_id}/cards/{card_id}` |
| Renovar cartão | `POST` | `/customers/{customer_id}/cards/{card_id}/renew` |

### Endereços

| Operação | Método | Endpoint |
|----------|--------|----------|
| Criar endereço | `POST` | `/customers/{customer_id}/addresses` |
| Obter endereço | `GET` | `/customers/{customer_id}/addresses/{address_id}` |
| Editar endereço | `PUT` | `/customers/{customer_id}/addresses/{address_id}` |
| Listar endereços | `GET` | `/customers/{customer_id}/addresses` |
| Excluir endereço | `DELETE` | `/customers/{customer_id}/addresses/{address_id}` |

### Pedidos

| Operação | Método | Endpoint |
|----------|--------|----------|
| Criar pedido | `POST` | `/orders` |
| Obter pedido | `GET` | `/orders/{order_id}` |
| Listar pedidos | `GET` | `/orders` |
| Fechar pedido | `PATCH` | `/orders/{order_id}/close` |

### Cobranças

| Operação | Método | Endpoint |
|----------|--------|----------|
| Capturar cobrança | `POST` | `/charges/{charge_id}/capture` |
| Obter cobrança | `GET` | `/charges/{charge_id}` |
| Cancelar / estornar cobrança | `DELETE` | `/charges/{charge_id}` |
| Listar cobranças | `GET` | `/charges` |
| Editar cartão da cobrança | `PATCH` | `/charges/{charge_id}/credit-card` |
| Editar método de pagamento | `PATCH` | `/charges/{charge_id}/payment-method` |
| Retentar cobrança | `POST` | `/charges/{charge_id}/retry` |

### Webhooks

| Operação | Método | Endpoint |
|----------|--------|----------|
| Obter webhook | `GET` | `/hooks/{hook_id}` |
| Reenviar webhook | `POST` | `/hooks/{hook_id}/retry` |
| Listar webhooks | `GET` | `/hooks` |

---

*Fonte: Documentação oficial Pagar.me API v5 — [docs.pagar.me](https://docs.pagar.me)*  
*Compilado em: Março/2026*
