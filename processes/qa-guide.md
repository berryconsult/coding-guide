# QA Guide - Berry

Guia unificado: testes manuais → testes E2E (Playwright).

> Para testes unitários (Vitest), veja [automated-testing.md](./automated-testing.md).

---

## 1. Testes Manuais

### Quando Testar

| Momento | Escopo |
|---------|--------|
| Antes de Merge | Funcionalidades modificadas |
| Antes de Deploy | Funcionalidades combinadas antes de um release |

### Fluxos E2E Manuais

**Lead → Projeto**
```
Formulário /contact → Lead criado → MQL (BDR atribuído) → SQL → Won → Contrato → Assinatura → Pagamento → Projeto ativo
```

**Leilão**
```
Deal > R$10k Won → Leilão criado → Lances → Encerramento → Cobrança CAC → Transferência
```

### Critérios de Aprovação

- **APROVAR:** passaram, sem erros críticos, performance OK
- **REJEITAR:** falhou, bug crítico, regressão, performance > 10s

---

## 2. Testes E2E (Playwright)

### Setup

```
packages/e2e/
├── fixtures/       # Helpers (auth, graphql, webhooks)
├── pages/          # Page Object Models
├── support/        # Dados de teste, builders
├── tests/          # Specs (.spec.ts)
└── playwright.config.ts
```

```bash
cd packages/e2e
pnpm add -D @playwright/test graphql-request
pnpm exec playwright install chromium
```

### Variáveis de Ambiente

```bash
BASE_URL=http://localhost:3000
API_URL=http://localhost:8000/graphql
TEST_ADMIN_EMAIL=qa@berry.com.br
TEST_BDR_EMAIL=bdr@berry.com.br
```

### Pré-requisitos

```bash
# Terminal 1: Backend
cd packages/api && pnpm dev

# Terminal 2: Frontend
cd packages/app && pnpm dev

# Terminal 3: Infra
docker-compose up -d
```

### Estrutura de Teste

```typescript
import { test, expect } from '@fixtures/test-base'
import { CRMPage } from '@pages/crm.page'

test.describe('Fluxo de Deal Won', () => {
  test('deve criar deal e marcar como won', async ({ page, auth, graphql }) => {
    // Arrange
    await auth.loginAsBDR()
    const crmPage = new CRMPage(page)
    const dealKey = await graphql.createTestDeal(TEST_DEAL)

    // Act
    await crmPage.openDeal(dealKey)
    await crmPage.markAsWon()

    // Assert
    await expect(page.getByText('Deal fechado')).toBeVisible()
  })
})
```

### Page Objects

```typescript
export class CRMPage extends BasePage {
  async openDeal(dealKey: string) {
    await this.goto(`/crm/${dealKey}`)
  }

  async markAsWon() {
    await this.clickButton('Marcar como Won')
    await this.waitForToast('Deal fechado com sucesso!')
  }
}
```

### Builders

```typescript
const dealData = new DealBuilder()
  .withOrganization('Empresa LTDA', '00.000.000/0001-00')
  .withAmount(10000)
  .build()
```

### Mocking Webhooks

```typescript
await webhooks.mockStripePaymentSuccess(invoiceId)
await webhooks.mockZapSignSigned(contractKey)
await webhooks.mockWhatsAppIncoming({
  from: '+5511999887766',
  message: { type: 'text', text: { body: 'Olá!' } },
})
```

### Comandos

```bash
pnpm --filter @berry/e2e test                   # Todos
pnpm --filter @berry/e2e test --grep @critical  # Apenas críticos
pnpm --filter @berry/e2e test:ui                # UI interativa
pnpm --filter @berry/e2e report                 # Ver relatório
```

## Best Practices

| ✅ Fazer | ❌ Evitar |
|----------|-----------|
| Fixtures para setup | Login manual em cada teste |
| Page Objects | Seletores inline |
| Builders para dados | Objetos hardcoded |
| Validar UI + API | Apenas UI |
| Dados únicos por teste | Dados compartilhados |
