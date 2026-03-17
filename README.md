# 📦 Pagar.me API v5 — Guia de Implementação para IA

> Documentação consolidada e estruturada da [API Pagar.me v5](https://docs.pagar.me/reference/) para uso como **contexto de sistema em agentes de IA** (Claude Code, Cursor, GitHub Copilot, GPT-4 etc.), acelerando a implementação de checkout e verificação de pagamentos no back-end.

---

## 🎯 O Problema

A documentação oficial do Pagar.me é excelente, mas está fragmentada em dezenas de páginas individuais. Ao pedir para uma IA implementar uma integração de pagamentos, ela frequentemente:

- Alucina endpoints ou parâmetros que não existem
- Confunde campos obrigatórios entre PSP e Gateway
- Esquece regras críticas (como `billing_address` obrigatório em cartão)
- Não sabe a formatação correta do `line_1` de endereço
- Desconhece particularidades como o e-mail único de customer ou o comportamento do `PUT` que sobrescreve tudo

Este repositório resolve isso com **um único arquivo `.md` autocontido** que uma IA pode ler como contexto antes de começar a implementar.

---

## 📄 O que está no guia

### [`pagarme-v5-checkout-implementation-guide.md`](./pagarme-v5-checkout-implementation-guide.md)

Guia completo com **16 seções**, cobrindo tudo que é necessário para implementar um ecossistema de checkout do zero:

| # | Seção | Destaques |
|---|-------|-----------|
| 1 | **Autenticação** | SK vs PK, Basic Auth, sandox vs produção, Node.js exemplo |
| 2 | **Erros e Códigos HTTP** | Tabela completa de status, erros comuns com causa, PSP vs Gateway |
| 3 | **Objeto `address`** | Formatação obrigatória do `line_1`, campos descontinuados, endereços internacionais |
| 4 | **Objeto `phones`** | Estrutura completa com `country_code`, `area_code`, `number` |
| 5 | **Clientes** | CRUD completo, e-mail único, `PUT` sobrescreve tudo, obrigatoriedade PSP |
| 6 | **Cartões e Wallet** | CRUD completo, private label, voucher, renovação via Card Updater |
| 7 | **Token de Cartão** | Endpoint `/tokens` com PK, single-use, sem `Authorization` header |
| 8 | **Endereços** | CRUD completo, `PUT` só edita `line_2` e `metadata` |
| 9 | **Pedido — Cartão de Crédito** | Todos os atributos de `credit_card`, 3DS, `auth_only`, `pre_auth`, exemplos |
| 10 | **Pedido — Pix** | QR Code, `expires_in`, split, estorno |
| 11 | **Obter Pedido** | Estratégia de polling com intervalos recomendados |
| 12 | **Webhooks** | Estrutura completa, exemplo real `order.paid`, todos os ~40 eventos, retry |
| 13 | **Tabela de Status** | Prefixos de IDs, status de pedido/cobrança/transação, valores em centavos |
| 14 | **Fluxo de Implementação** | Pseudocódigo TypeScript para cartão, Pix, Wallet e handler de webhook |
| 15 | **Regras de Negócio** | Tabela consolidada com 16 regras críticas |
| 16 | **Referência de Endpoints** | Todos os endpoints cobertos em tabela única |

---

## 🚀 Como usar

### Com Claude Code

```bash
# Na raiz do seu projeto
claude --context pagarme-v5-checkout-implementation-guide.md

# Ou referencie no CLAUDE.md do projeto
echo "Leia o arquivo pagarme-v5-checkout-implementation-guide.md antes de implementar qualquer integração com Pagar.me." >> CLAUDE.md
```

### Com Cursor

Adicione o arquivo ao seu projeto e referencie no `.cursorrules`:

```
Antes de implementar integrações com Pagar.me, leia e siga estritamente
o arquivo pagarme-v5-checkout-implementation-guide.md na raiz do projeto.
```

### Com qualquer LLM via API

```python
with open("pagarme-v5-checkout-implementation-guide.md") as f:
    doc = f.read()

messages = [
    {
        "role": "system",
        "content": f"Você é um especialista em integração Pagar.me. Use esta documentação como referência:\n\n{doc}"
    },
    {
        "role": "user",
        "content": "Implemente um endpoint Node.js para criar pedido com Pix"
    }
]
```

### Como MCP (Model Context Protocol)

```json
// mcp-config.json
{
  "mcpServers": {
    "pagarme-docs": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/caminho/para/este/repo"]
    }
  }
}
```

---

## 📊 Cobertura da Documentação Oficial

### ✅ Coberto neste guia

Baseado em [`docs.pagar.me/reference/`](https://docs.pagar.me/reference/):

| Área | Endpoints / Seções |
|------|--------------------|
| **Autenticação** | Tipos de chave, Basic Auth, ambientes |
| **Erros** | Todos os HTTP status codes, erros comuns |
| **Telefones** | Objeto `phones` completo |
| **Clientes** | Criar, Obter, Editar, Listar |
| **Cartões** | Criar, Obter, Listar, Editar, Excluir, Renovar |
| **Token de Cartão** | Criar Token (endpoint `/tokens`) |
| **Endereços** | Criar, Obter, Editar, Listar, Excluir |
| **Pagamentos — Cartão de Crédito** | Objeto completo, 3DS, network token, status |
| **Pagamentos — Pix** | Objeto completo, split, estorno |
| **Pedidos** | Criar (crédito e Pix), Obter |
| **Webhooks** | Estrutura, todos os eventos, Obter, Reenviar |

---

### ❌ Ainda não coberto (PRs são bem-vindos!)

#### Fundamentos
| Seção | URL da doc |
|-------|-----------|
| Paginação | `/reference/paginação` |
| Metadata | `/reference/metadata` |
| Entregas | `/reference/entregas` |
| Facilitadores de pagamento (SubMerchant) | `/reference/facilitadores-de-pagamento` |
| Segurança — Rate Limit | `/reference/rate-limit` |
| Segurança — IP Allowlist | `/reference/ip-allowlist` |
| BIN | `/reference/obter-informações-do-bin` |

#### Métodos de pagamento
| Método | URL da doc |
|--------|-----------|
| Boleto | `/reference/boleto` |
| Cartão de débito | `/reference/cartão-de-débito` |
| Cartão private label | `/reference/cartão-private-label` |
| Voucher | `/reference/voucher` |
| Google Pay | `/reference/google-pay` |
| Cash | `/reference/cash` |
| SafetyPay | `/reference/safetypay` |

#### Pedidos — endpoints restantes
| Endpoint | Método | URL da doc |
|----------|--------|-----------|
| Criar pedido multimeios | `POST` | `/reference/criar-pedido-multimeios` |
| Criar pedido multicompradores | `POST` | `/reference/criar-pedido-multicompradores` |
| Fechar pedido | `PATCH` | `/reference/fechar-um-pedido` |
| Listar pedidos | `GET` | `/reference/listar-pedidos` |
| Incluir cobrança no pedido | `POST` | `/reference/incluir-cobrança-no-pedido` |

#### Item do pedido
| Endpoint | Método |
|----------|--------|
| Incluir item | `POST` |
| Editar item | `PUT` |
| Deletar item | `DELETE` |
| Remover todos os itens | `DELETE` |
| Obter item | `GET` |

#### Cobranças
| Endpoint | Método | URL da doc |
|----------|--------|-----------|
| Capturar cobrança | `POST` | `/reference/capturar-cobrança` |
| Obter cobrança | `GET` | `/reference/obter-cobrança` |
| Editar cartão de cobrança | `PATCH` | `/reference/editar-cartão-de-cobrança` |
| Editar data de vencimento | `PATCH` | `/reference/editar-data-de-vencimento-da-cobrança` |
| Editar método de pagamento | `PATCH` | `/reference/editar-método-de-pagamento` |
| Cancelar cobrança | `DELETE` | `/reference/cancelar-cobrança` |
| Listar cobranças | `GET` | `/reference/listar-cobranças` |
| Retentar cobrança | `POST` | `/reference/retentar-uma-cobrança-manualmente` |
| Confirmar cobrança (cash) | `POST` | `/reference/confirmar-cobrança-cash` |

#### Antifraude
| Seção | URL da doc |
|-------|-----------|
| Visão Geral | `/reference/visão-geral-sobre-antifraude` |
| Criar pedido com Antifraude | `/reference/criar-pedido-antifraude` |

#### Link de Pagamento e Checkout
| Seção | URL da doc |
|-------|-----------|
| Link de Pagamento | `/reference/link-de-pagamento` |
| Checkout Pagar.me | `/reference/checkout-pagarme` |
| Criar / Obter / Listar / Ativar / Cancelar link | Vários |

#### Recorrência (Assinaturas)
| Área | Endpoints |
|------|-----------|
| Planos | Criar, Obter, Editar, Listar, Excluir |
| Assinaturas | Criar avulsa, Criar de plano, Obter, Listar, Cancelar, Editar |
| Item da assinatura | Incluir, Editar, Remover, Listar |
| Uso de item | Incluir, Remover, Listar |
| Item do plano | Incluir, Editar, Remover |
| Desconto | Incluir, Obter, Listar, Remover |
| Incremento | Incluir, Obter, Listar, Remover |
| Faturas | Criar, Obter, Listar, Editar, Cancelar |
| Ciclos | Renovar, Obter, Listar |

#### Split e Marketplace
| Área | URL da doc |
|------|-----------|
| Editar regras do Split | `/reference/editar-regras-do-split` |
| Criar pedido com split | `/reference/criar-pedido-com-split` |
| Capturar cobrança com split | `/reference/capturar-cobrança-com-split` |
| Cancelar cobrança com split | `/reference/cancelar-cobrança-com-split` |
| Visão Geral do Marketplace | `/reference/visão-geral-do-marketplace` |
| Res.264/349 Interface Eletrônica | Vários endpoints |

#### Recebedores (Recipients)
| Área | Endpoints |
|------|-----------|
| Recebedores | Criar, Obter, Editar, Listar, Editar code |
| KYC (Prova de Vida) | Criar link |
| Conta bancária | Atualizar |
| Saldo | Obter |
| Transferências | Criar, Obter, Listar, Cancelar, Comprovante |
| Antecipações | Criar, Simular, Obter limites, Cancelar, Listar |
| Liquidações | Obter, Listar |
| Operações de Saldo | Obter histórico |
| Configurações | Transferência automática, Antecipação automática |

#### Tokenização
| Área | URL da doc |
|------|-----------|
| Tokenizecard JS | `/reference/tokenizecard-js` |
| Alternativas ao Tokenizecard JS | `/reference/alternativas-ao-tokenizecard-js` |

#### Webhooks — restante
| Endpoint | Método |
|----------|--------|
| Listar webhooks | `GET /hooks` |

---

## 🤝 Como contribuir

1. Faça um fork do repositório
2. Escolha uma seção da lista de **não coberto** acima
3. Acesse a [documentação oficial](https://docs.pagar.me/reference/) e salve a página como HTML
4. Extraia o conteúdo relevante e adicione ao guia seguindo o padrão existente
5. Atualize a tabela de cobertura no README
6. Abra um Pull Request

### Padrão de documentação de endpoint

Cada endpoint deve incluir:
- Método HTTP + URL completa
- Avisos importantes (⚠️ ❗️ 🚧 📘)
- Tabela de path params (se houver)
- Tabela de body/query params com tipos, obrigatoriedade e descrição
- Exemplo de request (curl ou JSON)
- Exemplo de response quando relevante
- Regras de negócio específicas

---

## 📁 Estrutura do repositório

```
pagarme-v5-ai-guide/
├── README.md                                      # Este arquivo
└── pagarme-v5-checkout-implementation-guide.md   # O guia completo
```

---

## ⚡ Exemplo de resultado

Com este guia como contexto, uma IA pode gerar diretamente:

```typescript
// Gerado pela IA com o guia como contexto — sem alucinações
import axios from 'axios'

const PAGARME_SK = process.env.PAGARME_SECRET_KEY!
const auth = Buffer.from(`${PAGARME_SK}:`).toString('base64')
const api = axios.create({
  baseURL: 'https://api.pagar.me/core/v5',
  headers: { Authorization: `Basic ${auth}`, 'Content-Type': 'application/json' }
})

// Criar pedido Pix
export async function createPixOrder(params: {
  customerId: string
  amount: number
  description: string
  expiresIn?: number
}) {
  const { data } = await api.post('/orders', {
    customer_id: params.customerId,
    items: [{ amount: params.amount, description: params.description, quantity: 1 }],
    payments: [{
      payment_method: 'pix',
      pix: { expires_in: params.expiresIn ?? 3600 }
    }],
    closed: true
  })

  const charge = data.charges[0]
  const tx = charge.last_transaction

  return {
    orderId: data.id,
    chargeId: charge.id,
    qrCode: tx.qr_code,           // copia-e-cola
    qrCodeUrl: tx.qr_code_url,    // imagem
    expiresAt: tx.expires_at,
    status: data.status
  }
}

// Verificar status do pedido (polling)
export async function getOrderStatus(orderId: string) {
  const { data } = await api.get(`/orders/${orderId}`)
  return {
    status: data.status,
    chargeStatus: data.charges?.[0]?.status,
    paidAt: data.charges?.[0]?.paid_at
  }
}
```

---

## 📝 Licença

MIT — use, modifique e distribua livremente.

---

## 🔗 Links úteis

- [Documentação oficial Pagar.me API v5](https://docs.pagar.me/reference/)
- [Dashboard Pagar.me](https://dashboard.pagar.me/)
- [Status da API](https://status.pagar.me/)
- [Suporte Pagar.me](https://suporte.pagar.me/)

---

*Mantido pela comunidade. Não é um produto oficial do Pagar.me.*
