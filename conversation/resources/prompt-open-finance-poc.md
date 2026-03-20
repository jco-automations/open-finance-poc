# POC: Integração Open Finance Brasil - Solução Caseira

Crie um **POC em Python** que se conecte **direto ao Open Finance Brasil** usando **OAuth 2.0 + FAPI**, sem intermediárias (sem Pluggy, sem Belvo). Arquitetura idêntica ao Visor Finance.

---

## ARQUITETURA GERAL

**Stack:**
- Python 3.10+
- FastAPI (para receber callback do OAuth)
- `requests` + `requests_oauthlib` para fluxo OAuth
- Armazenamento local: JSON (sem BD)
- mTLS opcional (pode ser adicionado depois)

**Fluxo:**
1. Usuário insere CPF em CLI
2. App gera "consentimento" e URL de jornada
3. Usuário é redirecionado pro banco (Nubank, Inter, BB, Contabilizei)
4. Autentica lá (biometria/senha/token)
5. Autoriza o compartilhamento de dados
6. Banco redireciona de volta com `authorization code`
7. App troca `code` por `access_token`
8. App puxa dados das APIs de contas/cartões/transações
9. Consolida tudo em JSON local

---

## ENDPOINTS REAIS DO OPEN FINANCE

**Base URL por banco:**
- Nubank: `https://api.nubank.com.br/open-banking/`
- Banco do Brasil: `https://openbanking.bb.com.br/open-banking/`
- Inter: `https://api.inter.co/open-banking/`
- Contabilizei Bank: `https://api.contabilizei.com.br/open-banking/`

(Disclaimer: URLs são exemplo; cada banco fornece seu próprio endpoint no diretório de participantes)

### 1. CONSENTIMENTO (Create Consent)

**Endpoint:** `POST /consents/v2/consents`

**Request:**
```json
{
  "data": {
    "permissions": [
      "ACCOUNTS_READ",
      "ACCOUNTS_BALANCES_READ",
      "ACCOUNTS_TRANSACTIONS_READ",
      "CREDIT_CARDS_ACCOUNTS_READ",
      "CREDIT_CARDS_ACCOUNTS_BILLS_READ",
      "CREDIT_CARDS_ACCOUNTS_TRANSACTIONS_READ",
      "CUSTOMERS_PERSONAL_IDENTIFICATIONS_READ"
    ],
    "expirationDateTime": "2025-12-31T23:59:59Z",
    "loggedUser": {
      "document": {
        "identification": "12345678900",
        "rel": "CPF"
      }
    }
  },
  "meta": {}
}
```

**Response:**
```json
{
  "data": {
    "consentId": "urn:banco:consent:12345678-1234-1234-1234",
    "creationDateTime": "2025-03-09T10:00:00Z",
    "expirationDateTime": "2025-12-31T23:59:59Z",
    "permissions": [/* ... */],
    "status": "AWAITING_AUTHORISATION"
  },
  "links": {
    "self": "https://banco.com.br/open-banking/consents/v2/consents/urn:banco:consent:12345678"
  }
}
```

**Usa:** `consentId` para construir URL de autorização

### 2. AUTORIZAÇÃO (Redirect para banco)

**URL de redirecionamento:**

```
https://[banco-authorization-server]/authorize?
  client_id=[seu-client-id]&
  response_type=code&
  scope=[scopes-separados-por-espaço]&
  redirect_uri=http://localhost:8000/callback&
  state=[random-string]&
  consent_id=urn:banco:consent:12345678
```

**Bancos usam:** FAPI + OpenID Connect
- **Auth Server Nubank:** `https://auth.nubank.com.br`
- **Auth Server BB:** `https://auth.bb.com.br`
- **Auth Server Inter:** `https://auth.inter.co`

### 3. CALLBACK (Seu servidor recebe o code)

**Endpoint seu:** `GET http://localhost:8000/callback?code=...&state=...`

Valida `state`, extrai `code`, prepara para trocar por token.

### 4. TOKEN (Trade code por access_token)

**Endpoint:** `POST /consents/v2/consents/{consentId}/authorise`

Ou direto no **token endpoint do banco** (depende da implementação):

**Request:**
```json
POST https://[banco-auth-server]/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code=[authorization-code]&
client_id=[seu-client-id]&
client_secret=[seu-client-secret]&
redirect_uri=http://localhost:8000/callback
```

**Response:**
```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "refresh_token_value",
  "scope": "ACCOUNTS_READ ACCOUNTS_TRANSACTIONS_READ ..."
}
```

---

## APIS DE DADOS (Com access_token)

Todos os calls incluem header:
```
Authorization: Bearer {access_token}
x-fapi-interaction-id: {uuid-único}
```

### 5. LISTAR CONTAS

**Endpoint:** `GET /accounts/v2/accounts`

**Response:**
```json
{
  "data": [
    {
      "accountId": "account-id-12345",
      "name": "Conta Corrente",
      "type": "CHECKING",
      "subtype": "INDIVIDUAL",
      "currency": "BRL",
      "accountNumber": "123456",
      "branchNumber": "0001"
    }
  ],
  "links": {
    "self": "https://banco.com.br/open-banking/accounts/v2/accounts"
  }
}
```

### 6. SALDO DE CONTA

**Endpoint:** `GET /accounts/v2/accounts/{accountId}/balances`

**Response:**
```json
{
  "data": {
    "balances": [
      {
        "amount": "1234.56",
        "currency": "BRL",
        "type": "AVAILABLE"
      },
      {
        "amount": "100.00",
        "currency": "BRL",
        "type": "BLOCKED"
      }
    ]
  }
}
```

### 7. TRANSAÇÕES DE CONTA

**Endpoint:** `GET /accounts/v2/accounts/{accountId}/transactions?fromDate=2025-01-01&toDate=2025-03-09&pageSize=50`

**Response:**
```json
{
  "data": [
    {
      "transactionId": "txn-12345",
      "booked": true,
      "type": "DEBIT",
      "transactionDate": "2025-03-08",
      "amount": "-50.00",
      "currency": "BRL",
      "description": "Compra no supermercado",
      "counterParty": {
        "name": "SUPERMERCADO XYZ"
      }
    }
  ],
  "links": {
    "self": "...",
    "first": "...",
    "next": "..."
  }
}
```

### 8. CARTÕES DE CRÉDITO

**Endpoint:** `GET /credit-cards-accounts/v2/accounts`

**Response:**
```json
{
  "data": [
    {
      "creditCardAccountId": "cc-account-12345",
      "name": "Meu Cartão Nubank",
      "issuer": "NUBANK",
      "network": "MASTERCARD"
    }
  ]
}
```

### 9. LIMITE E FATURAS DO CARTÃO

**Endpoint:** `GET /credit-cards-accounts/v2/accounts/{creditCardAccountId}/limits`

**Response:**
```json
{
  "data": [
    {
      "limitType": "CREDIT_LINE",
      "amount": "5000.00",
      "currency": "BRL",
      "isLimited": true,
      "usedAmount": "2000.00",
      "availableAmount": "3000.00"
    }
  ]
}
```

**Endpoint faturas:** `GET /credit-cards-accounts/v2/accounts/{creditCardAccountId}/bills`

**Transações da fatura:**
`GET /credit-cards-accounts/v2/accounts/{creditCardAccountId}/bills/{billId}/transactions`

---

## SETUP / CONFIGURAÇÃO

### 1. Registro no Diretório Open Finance

Cada banco mantém um **diretório de participantes** onde você registra sua aplicação:
- **Nome da app**
- **Redirect URI** (ex: `http://localhost:8000/callback`)
- **Client ID + Client Secret** (gerados automaticamente)

Banco fornece:
- `client_id`
- `client_secret` (guarde com segurança)
- **Authorization Server URL**
- **Token Endpoint URL**
- **Resource Server URLs** (para puxar dados)

### 2. Scopes Necessários

```
ACCOUNTS_READ
ACCOUNTS_BALANCES_READ
ACCOUNTS_TRANSACTIONS_READ
CREDIT_CARDS_ACCOUNTS_READ
CREDIT_CARDS_ACCOUNTS_BILLS_READ
CREDIT_CARDS_ACCOUNTS_TRANSACTIONS_READ
CUSTOMERS_PERSONAL_IDENTIFICATIONS_READ
```

---

## ESTRUTURA DE CÓDIGO ESPERADA

```
poc-open-finance/
├── main.py                    # Entrada, CLI
├── oauth_flow.py              # Fluxo OAuth completo
├── api_client.py              # Chamadas aos bancos
├── models.py                  # Dataclasses (Account, Transaction, etc)
├── config.py                  # Credenciais (banco-específicas)
├── storage.py                 # JSON local
├── server.py                  # FastAPI para callback
└── data/
    ├── consolidated.json      # Dados consolidados
    ├── tokens.json            # Access tokens + refresh tokens
    └── consents.json          # Histórico de consents
```

---

## FLUXO DE EXECUÇÃO ESPERADO

```
$ python main.py

1. Insira seu CPF: 12345678900
2. Qual banco você quer conectar? (1=Nubank, 2=Inter, 3=BB, 4=Contabilizei): 1
3. Abrindo jornada... Abra o navegador:
   → https://auth.nubank.com.br/authorize?client_id=...&consent_id=...
4. [Usuário faz login no Nubank]
5. [Usuário autoriza compartilhamento de dados]
6. Pronto! Seu código foi capturado. Vou buscar os dados...
7. ✓ Saldos das contas: R$ 1.234,56
8. ✓ Últimas 50 transações
9. ✓ Cartão de Crédito: Limite R$ 5.000 | Disponível R$ 3.000
10. Dados salvos em: consolidated.json
```

---

## DETALHES TÉCNICOS IMPORTANTES

### Segurança

1. **State Parameter:** Gere UUID aleatório, armazene em sessão, valide no callback
2. **PKCE (Proof Key for Code Exchange):** Recomendado para SPAs, opcional para backend
3. **Tokens em disco:** Criptografe com `cryptography` se possível (ou aviso em console)
4. **Refresh Token:** Implemente rotação de token antes de expirar
5. **mTLS:** Suporte "nice-to-have" (certificados do banco)

### Paginação

Alguns bancos retornam apenas 50 registros por página. Implemente:
```python
while "next" in response.get("links", {}):
    next_url = response["links"]["next"]
    response = requests.get(next_url, headers=headers)
    data.extend(response["data"])
```

### Tratamento de Erros

- Token expirado → Use refresh_token
- Consentimento revogado → Peça novo consentimento
- Rate limit (429) → Exponential backoff
- Timeout → Retry com jitter

### Data Freshness

Segundo Resolução 86 do Banco Central:
- **Saldos + Transações:** até 5 minutos de defasagem
- **Outros dados:** até 1 hora

### Headers Obrigatórios

```python
headers = {
    "Authorization": f"Bearer {access_token}",
    "Content-Type": "application/json",
    "x-fapi-interaction-id": str(uuid.uuid4()),  # Obrigatório
    "Accept": "application/json"
}
```

---

## DADOS A CONSOLIDAR EM JSON

**Output esperado em `consolidated.json`:**

```json
{
  "metadata": {
    "cpf": "12345678900",
    "banks": ["Nubank", "Banco do Brasil"],
    "last_updated": "2025-03-09T14:30:00Z"
  },
  "accounts": [
    {
      "bank": "Nubank",
      "account_id": "...",
      "name": "Conta Corrente",
      "type": "CHECKING",
      "balance": {
        "available": "1234.56",
        "blocked": "100.00",
        "currency": "BRL"
      },
      "transactions": [
        {
          "date": "2025-03-08",
          "description": "Compra online",
          "amount": "-50.00",
          "type": "DEBIT"
        }
      ]
    }
  ],
  "credit_cards": [
    {
      "bank": "Nubank",
      "card_id": "...",
      "name": "Meu Cartão",
      "limit": "5000.00",
      "available": "3000.00",
      "current_bill": "2000.00",
      "bills": [
        {
          "bill_id": "...",
          "due_date": "2025-04-15",
          "total": "2000.00",
          "minimum": "100.00",
          "transactions": [...]
        }
      ]
    }
  ]
}
```

---

## PRÓXIMOS PASSOS (Não incluir na v1)

1. Suporte a mTLS (certificados)
2. Webhooks para atualização automática de dados
3. UI web (React/Vue) ao invés de CLI
4. Sincronização com Obsidian/Neo4j
5. Detecção de padrões de gasto (IA)
6. Integração com Pix para análise de fluxo


