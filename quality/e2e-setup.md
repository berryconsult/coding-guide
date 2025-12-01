# Testes End-to-End (E2E) - Setup e Configuração

**Versão:** 1.0.0
**Última atualização:** 28 de Novembro de 2025
**Responsável:** Equipe de QA e Desenvolvimento Berry

---

## Índice

- [Introdução](#introdução)
- [Instalação e Configuração](#instalação-e-configuração)
- [Ambiente Local](#ambiente-local)
- [Fixtures e Helpers](#fixtures-e-helpers)
- [Dados de Teste](#dados-de-teste)
- [Integrações Externas](#integrações-externas)
- [CI/CD (Futuro)](#cicd-futuro)
- [Troubleshooting](#troubleshooting)
- [Referências](#referências)

---

## Introdução

### O que são Testes E2E?

Testes End-to-End (E2E) validam fluxos completos do sistema, simulando o comportamento real do usuário desde o início até o fim de uma jornada. No Berry, testes E2E cobrem:

- **Interação Frontend + Backend**: Clicks, navegação, formulários, API calls
- **Eventos Assíncronos**: Listeners, webhooks, jobs em fila (BullMQ)
- **Integrações Externas**: Stripe, ZapSign, WhatsApp, Google Calendar
- **Fluxos Críticos**: Lead → Projeto, Leilão, Pagamento → Ativação

### Por que Playwright?

- **Instalado no projeto**: `packages/api` já possui Playwright 1.40.0
- **Multi-browser**: Chromium, Firefox, WebKit
- **Auto-wait**: Aguarda automaticamente elementos aparecerem
- **Paralelo**: Executa testes simultaneamente
- **Screenshots/Vídeos**: Captura automática em falhas

### Escopo Atual vs. Futuro

**Fase 1 (Atual):**
- 3 fluxos críticos documentados
- Testes locais obrigatórios antes de merge
- Cobertura manual de integrações complexas

**Fase 2 (Futuro):**
- 10+ fluxos E2E completos
- CI/CD com execução automática em PRs
- Cobertura completa de edge cases

---

## Instalação e Configuração

### Estrutura do Projeto

```
berrymax/
├── packages/
│   ├── api/          # Backend (já tem Playwright)
│   ├── app/          # Frontend React
│   └── e2e/          # 🆕 Testes E2E (novo package)
│       ├── fixtures/       # Helpers reutilizáveis
│       ├── pages/          # Page Object Models
│       ├── support/        # Dados de teste, builders
│       ├── tests/          # Specs (.spec.ts)
│       ├── playwright.config.ts
│       └── package.json
```

### Instalação

**1. Criar package E2E:**

```bash
cd packages/
mkdir e2e
cd e2e
pnpm init
```

**2. Instalar dependências:**

```bash
pnpm add -D @playwright/test@^1.40.0
pnpm add -D graphql-request@^6.1.0
pnpm add -D @types/node
```

**3. Instalar browsers:**

```bash
pnpm exec playwright install chromium
```

### Configuração Básica

**`packages/e2e/playwright.config.ts`:**

```typescript
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,

  reporter: [
    ['html'],
    ['junit', { outputFile: 'results.xml' }],
  ],

  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },

  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
  ],

  globalSetup: require.resolve('./global-setup.ts'),
})
```

**`packages/e2e/package.json`:**

```json
{
  "name": "@berry/e2e",
  "version": "1.0.0",
  "scripts": {
    "test": "playwright test",
    "test:ui": "playwright test --ui",
    "test:debug": "playwright test --debug",
    "test:headed": "playwright test --headed",
    "report": "playwright show-report"
  },
  "dependencies": {
    "@playwright/test": "^1.40.0",
    "graphql-request": "^6.1.0"
  }
}
```

**`packages/e2e/tsconfig.json`:**

```json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@fixtures/*": ["fixtures/*"],
      "@pages/*": ["pages/*"],
      "@support/*": ["support/*"]
    }
  },
  "include": ["**/*.ts"]
}
```

---

## Ambiente Local

### Pré-requisitos

**Serviços rodando:**

```bash
# Backend API (porta 8000)
cd packages/api
pnpm dev

# Frontend (porta 3000)
cd packages/app
pnpm dev

# Infraestrutura (Docker)
docker-compose up -d  # ArangoDB, Redis, Elasticsearch
```

### Variáveis de Ambiente

**`packages/e2e/.env.example`:**

```bash
# URLs
BASE_URL=http://localhost:3000
API_URL=http://localhost:8000/graphql

# Credenciais de Teste
TEST_ADMIN_EMAIL=qa@berry.com.br
TEST_BDR_EMAIL=bdr@berry.com.br
TEST_FRANCHISEE_EMAIL=franchisee@berry.com.br

# Integrações Sandbox
STRIPE_TEST_KEY=sk_test_xxxxx
ZAPSIGN_SANDBOX_TOKEN=sandbox_xxxxx

# Mocks (opcional)
MOCK_WHATSAPP=true
MOCK_GOOGLE_CALENDAR=true
```

**Copiar arquivo:**

```bash
cp .env.example .env
```

### Global Setup

**`packages/e2e/global-setup.ts`:**

```typescript
import { chromium, FullConfig } from '@playwright/test'

async function globalSetup(config: FullConfig) {
  console.log('🚀 Starting E2E test suite...')

  const browser = await chromium.launch()
  const page = await browser.newPage()

  // Verificar frontend
  try {
    await page.goto(config.use?.baseURL || 'http://localhost:3000')
    console.log('✅ Frontend is running')
  } catch (error) {
    console.error('❌ Frontend is not running. Start with: pnpm dev')
    process.exit(1)
  }

  // Verificar backend
  try {
    const apiUrl = process.env.API_URL || 'http://localhost:8000/graphql'
    const response = await page.context().request.post(apiUrl, {
      data: {
        query: '{ __typename }',
      },
    })
    if (response.ok()) {
      console.log('✅ Backend API is running')
    } else {
      throw new Error('API not responding')
    }
  } catch (error) {
    console.error('❌ Backend API is not running. Start with: cd packages/api && pnpm dev')
    process.exit(1)
  }

  await browser.close()
}

export default globalSetup
```

### Executando Testes Locais

```bash
cd packages/e2e

# Executar todos os testes
pnpm test

# Executar com UI interativa
pnpm test:ui

# Executar em modo debug
pnpm test:debug

# Executar teste específico
pnpm test tests/deals.spec.ts

# Ver relatório HTML
pnpm report
```

---

## Fixtures e Helpers

### Sistema de Fixtures

Fixtures são helpers reutilizáveis que eliminam código duplicado nos testes.

**`packages/e2e/fixtures/test-base.ts`:**

```typescript
import { test as base } from '@playwright/test'
import { GraphQLClient } from './api'
import { AuthFixture } from './auth'
import { WebhookMock } from './webhooks'

type Fixtures = {
  graphql: GraphQLClient
  auth: AuthFixture
  webhooks: WebhookMock
}

export const test = base.extend<Fixtures>({
  graphql: async ({}, use) => {
    const client = new GraphQLClient(process.env.API_URL!)
    await use(client)
  },

  auth: async ({ page, graphql }, use) => {
    const auth = new AuthFixture(page, graphql)
    await use(auth)
  },

  webhooks: async ({ page }, use) => {
    const webhooks = new WebhookMock(page)
    await use(webhooks)
  },
})

export { expect } from '@playwright/test'
```

### Fixture de Autenticação

**`packages/e2e/fixtures/auth.ts`:**

```typescript
import { Page } from '@playwright/test'
import { GraphQLClient } from './api'

export class AuthFixture {
  constructor(
    private page: Page,
    private graphql: GraphQLClient,
  ) {}

  async loginAsAdmin() {
    await this.loginAs('qa@berry.com.br')
  }

  async loginAsBDR() {
    await this.loginAs('bdr@berry.com.br')
  }

  async loginAsFranchisee() {
    await this.loginAs('franchisee@berry.com.br')
  }

  private async loginAs(email: string) {
    const token = await this.graphql.getTestToken(email)

    await this.page.goto('/')
    await this.page.evaluate((t) => {
      localStorage.setItem('BERRY_TOKEN', t)
    }, token)

    await this.page.goto('/')
    await this.page.waitForURL('/')
  }
}
```

### Fixture de GraphQL

**`packages/e2e/fixtures/api.ts`:**

```typescript
import { GraphQLClient as GQLClient } from 'graphql-request'

export class GraphQLClient {
  private client: GQLClient

  constructor(endpoint: string) {
    this.client = new GQLClient(endpoint)
  }

  async getTestToken(email: string): Promise<string> {
    const query = `
      mutation GetTestToken($email: String!) {
        getTestToken(email: $email)
      }
    `
    const result = await this.client.request<{ getTestToken: string }>(query, { email })
    return result.getTestToken
  }

  async createTestDeal(data: any): Promise<string> {
    const mutation = `
      mutation CreateDeal($input: DealInput!) {
        createDeal(input: $input) {
          _key
        }
      }
    `
    const result = await this.client.request<{ createDeal: { _key: string } }>(mutation, { input: data })
    return result.createDeal._key
  }
}
```

### Fixture de Webhooks

**`packages/e2e/fixtures/webhooks.ts`:**

```typescript
import { Page } from '@playwright/test'

export class WebhookMock {
  constructor(private page: Page) {}

  async mockStripePaymentSuccess(invoiceId: string) {
    await this.sendWebhook('/api/webhooks/stripe', {
      type: 'invoice.payment_succeeded',
      data: {
        object: {
          id: invoiceId,
          status: 'paid',
        },
      },
    })
  }

  async mockZapSignSigned(contractId: string) {
    await this.sendWebhook('/api/webhooks/zapsign', {
      event: 'document.signed',
      contractId,
      signedAt: new Date().toISOString(),
    })
  }

  private async sendWebhook(url: string, data: any) {
    const baseURL = process.env.API_URL?.replace('/graphql', '')
    await this.page.context().request.post(`${baseURL}${url}`, {
      data,
    })
  }
}
```

---

## Dados de Teste

### Usuários de Teste

**`packages/e2e/support/test-data.ts`:**

```typescript
export const TEST_USERS = {
  admin: {
    email: 'qa@berry.com.br',
    workspace: 'test-workspace',
    role: 'admin',
  },
  bdr: {
    email: 'bdr@berry.com.br',
    workspace: 'test-workspace',
    role: 'bdr',
  },
  sdr: {
    email: 'sdr@berry.com.br',
    workspace: 'test-workspace',
    role: 'sdr',
  },
  closer: {
    email: 'closer@berry.com.br',
    workspace: 'test-workspace',
    role: 'closer',
  },
  franchisee: {
    email: 'franchisee@berry.com.br',
    workspace: 'franchisee-workspace',
    role: 'admin',
  },
}

export const TEST_DEAL = {
  organization: {
    name: 'Test Company LTDA',
    cnpj: '00.000.000/0000-00',
    segment: 'Tecnologia',
  },
  contact: {
    name: 'João Silva',
    email: 'joao@testcompany.com',
    phone: '+5511999990000',
  },
  amount: 5000,
  services: ['CRM', 'Marketing'],
}

export const STRIPE_TEST_CARDS = {
  success: '4242 4242 4242 4242',
  declined: '4000 0000 0000 0002',
  requiresAuth: '4000 0025 0000 3155',
}
```

### Builders

**`packages/e2e/support/builders.ts`:**

```typescript
export class DealBuilder {
  private data: any = {
    organization: { name: 'Test Company' },
    contact: { name: 'Test Contact', email: 'test@test.com' },
    amount: 1000,
  }

  withOrganization(name: string, cnpj: string) {
    this.data.organization = { name, cnpj }
    return this
  }

  withAmount(amount: number) {
    this.data.amount = amount
    return this
  }

  build() {
    return this.data
  }
}

export class ContractBuilder {
  private data: any = {
    monthlyAmount: 500,
    duration: 12,
    paymentMethod: 'credit_card',
  }

  withPaymentMethod(method: string) {
    this.data.paymentMethod = method
    return this
  }

  build() {
    return this.data
  }
}
```

---

## Integrações Externas

### Estratégia de Mocking vs. Sandbox

| Integração | Estratégia | Razão |
|------------|-----------|-------|
| **Stripe** | Sandbox (test mode) | Crítico para pagamento, possui ambiente de teste robusto |
| **ZapSign** | Sandbox | Crítico para assinaturas, possui ambiente sandbox |
| **WhatsApp** | Mock | Sem sandbox oficial, mock via webhooks |
| **Google Calendar** | Mock | Testes não precisam criar eventos reais |
| **Brevo (Email)** | Mock | Evitar envio de emails de teste |

### Configuração Stripe Test Mode

1. **Obter chaves de teste:**
   - Acessar [Stripe Dashboard](https://dashboard.stripe.com/test/apikeys)
   - Copiar `Publishable key` e `Secret key` (começam com `pk_test_` e `sk_test_`)

2. **Configurar no backend:**
   ```bash
   # packages/api/.env.local
   STRIPE_SECRET_KEY=sk_test_xxxxx
   ```

3. **Usar cartões de teste:**
   ```typescript
   // packages/e2e/support/test-data.ts
   export const STRIPE_TEST_CARDS = {
     success: '4242 4242 4242 4242',  // Sempre aprova
     declined: '4000 0000 0000 0002', // Sempre recusa
   }
   ```

### Configuração ZapSign Sandbox

1. **Criar conta sandbox:**
   - Contatar suporte ZapSign para ativar sandbox
   - Obter token de sandbox

2. **Configurar no backend:**
   ```bash
   # packages/api/.env.local
   ZAPSIGN_API_TOKEN=sandbox_xxxxx
   ZAPSIGN_SANDBOX=true
   ```

---

## CI/CD (Futuro)

### Planejamento CI/CD

**Quando implementar:**
- Após reestruturação do processo de entregas
- Integração com GitHub Actions ou Jenkins

**Configuração planejada:**

```yaml
# .github/workflows/e2e.yml (futuro)
name: E2E Tests

on:
  pull_request:
    branches: [main, develop]

jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: pnpm/action-setup@v2

      - name: Install dependencies
        run: pnpm install

      - name: Start services
        run: docker-compose up -d

      - name: Run E2E tests
        run: pnpm --filter @berry/e2e test

      - name: Upload test results
        if: failure()
        uses: actions/upload-artifact@v3
        with:
          name: playwright-report
          path: packages/e2e/playwright-report/
```

---

## Troubleshooting

### Problemas Comuns

**1. Frontend/Backend não está rodando**

```bash
❌ Frontend is not running. Start with: pnpm dev
```

**Solução:**
```bash
# Terminal 1: Backend
cd packages/api
pnpm dev

# Terminal 2: Frontend
cd packages/app
pnpm dev
```

**2. Timeout em testes**

```
Error: Timeout 30000ms exceeded
```

**Solução:**
- Aumentar timeout no teste:
  ```typescript
  test('fluxo completo', async ({ page }) => {
    test.setTimeout(60000) // 60 segundos
  })
  ```

**3. Elemento não encontrado**

```
Error: locator.click: Target closed
```

**Solução:**
- Adicionar espera explícita:
  ```typescript
  await page.waitForSelector('button[type="submit"]')
  await page.click('button[type="submit"]')
  ```

**4. Testes falhando intermitentemente**

**Solução:**
- Desabilitar paralelismo:
  ```typescript
  // playwright.config.ts
  fullyParallel: false,
  workers: 1,
  ```

**5. Banco de dados com dados residuais**

**Solução:**
- Criar script de limpeza:
  ```bash
  # packages/e2e/scripts/clean-test-data.sh
  curl -X POST http://localhost:8000/test/cleanup
  ```

---

## Referências

### Documentação Oficial

- [Playwright Documentation](https://playwright.dev/)
- [Playwright Best Practices](https://playwright.dev/docs/best-practices)
- [GraphQL Request](https://github.com/jasonkuhrt/graphql-request)

### Documentação Berry

- [Testes Automatizados](./automated-testing.md)
- [Testes Manuais](./manual-testing.md)
- [E2E - Guia Prático](./e2e-guide.md) *(próximo documento)*

### Scripts Úteis

```bash
# Ver todos os testes
pnpm test --list

# Executar apenas testes com tag
pnpm test --grep @critical

# Executar com mais workers (paralelismo)
pnpm test --workers=4

# Gerar trace para debug
pnpm test --trace on
```

---

**Próximos Passos:**

1. ✅ Setup completo (este documento)
2. 📝 Criar guia prático com exemplos de código ([e2e-guide.md](./e2e-guide.md))
3. 🔧 Implementar estrutura `packages/e2e/`
4. 🧪 Escrever testes para 3 fluxos críticos
5. 📊 Integrar com processo de Code Review

**Dúvidas?** Consulte o Tech Lead ou abra issue no repositório.
