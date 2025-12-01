# Testes End-to-End (E2E) - Guia Prático

**Versão:** 1.0.0
**Última atualização:** 28 de Novembro de 2025
**Responsável:** Equipe de QA e Desenvolvimento Berry

---

## Índice

- [Anatomia de um Teste E2E](#anatomia-de-um-teste-e2e)
- [Fluxos Críticos Implementados](#fluxos-críticos-implementados)
  - [Fluxo 1: Lead → Projeto Completo](#fluxo-1-lead--projeto-completo)
  - [Fluxo 2: Leilão de Deal de Alto Valor](#fluxo-2-leilão-de-deal-de-alto-valor)
  - [Fluxo 3: WhatsApp Lead → CRM → Atribuição](#fluxo-3-whatsapp-lead--crm--atribuição)
- [Page Object Models](#page-object-models)
- [Padrões de Mocking](#padrões-de-mocking)
- [Assertions e Validações](#assertions-e-validações)
- [Best Practices](#best-practices)
- [Exemplos Adicionais](#exemplos-adicionais)

---

## Anatomia de um Teste E2E

### Estrutura Básica

Um teste E2E completo no Berry segue esta estrutura:

```typescript
import { test, expect } from '@fixtures/test-base'
import { CRMPage } from '@pages/crm.page'
import { TEST_USERS, TEST_DEAL } from '@support/test-data'

test.describe('Fluxo de Deal Won', () => {
  test('deve criar deal, marcar como won e ativar projeto', async ({ page, auth, graphql, webhooks }) => {
    // 1. ARRANGE: Setup inicial
    await auth.loginAsBDR()
    const crmPage = new CRMPage(page)

    // 2. ACT: Executar ações do usuário
    const dealKey = await graphql.createTestDeal(TEST_DEAL)
    await crmPage.openDeal(dealKey)
    await crmPage.markAsMQL()
    await crmPage.markAsSQL()
    await crmPage.markAsWon()

    // 3. ASSERT: Validar resultado esperado
    await expect(page.getByText('Deal marcado como Won')).toBeVisible()
    await expect(page).toHaveURL(/\/projects\//)
  })
})
```

### Componentes do Teste

**1. Imports:**
- `test, expect`: Fixtures customizadas do Berry
- Page Objects: Classes que encapsulam interações com páginas
- Test Data: Dados reutilizáveis para testes

**2. Fixtures:**
- `page`: Instância do Playwright Page
- `auth`: Helper de autenticação
- `graphql`: Cliente GraphQL para setup via API
- `webhooks`: Mock de webhooks externos

**3. Pattern AAA (Arrange-Act-Assert):**
- **Arrange**: Login, criar dados de teste, navegar para página
- **Act**: Executar ações do usuário (clicks, preenchimento de formulários)
- **Assert**: Validar UI, URLs, dados no banco

---

## Fluxos Críticos Implementados

### Fluxo 1: Lead → Projeto Completo

**Descrição:** Usuário BDR qualifica lead, SDR converte em SQL, Closer marca como Won, contrato é assinado, pagamento processado e projeto ativado.

**Duração estimada:** ~45 segundos

**Listeners envolvidos:**
1. `CrmLeadAnalysisUseCase` (Priority 10)
2. `CrmDealIsMQL` (Priority 20)
3. `CrmDealIsSQL` (Priority 30)
4. `CrmDealClose` (Priority 40)
5. `GenerateContractSigningFromContract` (Priority 50)
6. `CreateProjectFromDealUseCase` (Priority 60)

**Código completo:**

```typescript
import { test, expect } from '@fixtures/test-base'
import { CRMPage } from '@pages/crm.page'
import { ContractPage } from '@pages/contract.page'
import { ProjectPage } from '@pages/project.page'
import { TEST_USERS, TEST_DEAL } from '@support/test-data'
import { DealBuilder, ContractBuilder } from '@support/builders'

test.describe('Fluxo Completo: Lead → Projeto', () => {
  test('deve qualificar lead, fechar deal, assinar contrato e ativar projeto', async ({
    page,
    auth,
    graphql,
    webhooks,
  }) => {
    // ===== ETAPA 1: Criar Lead (BDR) =====
    await auth.loginAsBDR()
    const crmPage = new CRMPage(page)

    const dealData = new DealBuilder()
      .withOrganization('Empresa Teste LTDA', '00.000.000/0001-00')
      .withAmount(10000)
      .build()

    const dealKey = await graphql.createTestDeal(dealData)
    console.log('✅ Deal criado:', dealKey)

    // ===== ETAPA 2: Marcar como MQL =====
    await crmPage.openDeal(dealKey)
    await expect(page.getByText('Empresa Teste LTDA')).toBeVisible()

    await crmPage.markAsMQL()
    await expect(page.getByText('Status: MQL')).toBeVisible()

    // Aguardar listener CrmDealIsMQL criar tarefa para BDR
    await page.waitForTimeout(2000)
    await expect(page.getByText('Tarefa: Entrar em contato via WhatsApp')).toBeVisible()

    // ===== ETAPA 3: Qualificar como SQL (SDR) =====
    await auth.loginAs('sdr@berry.com.br')
    await crmPage.openDeal(dealKey)

    await crmPage.markAsSQL()
    await expect(page.getByText('Status: SQL')).toBeVisible()

    // Listener CrmDealIsSQL atribui Closer
    await page.waitForTimeout(2000)
    await expect(page.getByText('Atribuído a: closer@berry.com.br')).toBeVisible()

    // ===== ETAPA 4: Marcar como Won (Closer) =====
    await auth.loginAs('closer@berry.com.br')
    await crmPage.openDeal(dealKey)

    await crmPage.fillMeetingOutcome({
      outcome: 'success',
      notes: 'Cliente aprovou proposta de R$ 10.000',
    })

    await crmPage.markAsWon()
    await expect(page.getByText('Deal fechado com sucesso!')).toBeVisible()

    // Listener CrmDealClose cria contrato
    await page.waitForTimeout(3000)
    await expect(page.getByText('Contrato gerado')).toBeVisible()

    // ===== ETAPA 5: Assinar Contrato =====
    const contractPage = new ContractPage(page)
    await contractPage.openFromDeal(dealKey)

    const contractData = new ContractBuilder()
      .withPaymentMethod('credit_card')
      .build()

    await contractPage.fillContractDetails(contractData)
    await contractPage.submitForSigning()

    // Aguardar listener GenerateContractSigningFromContract enviar para ZapSign
    await page.waitForTimeout(2000)
    await expect(page.getByText('Enviado para assinatura')).toBeVisible()

    const contractKey = await page.getAttribute('[data-contract-key]', 'data-contract-key')

    // Mock webhook ZapSign (documento assinado)
    await webhooks.mockZapSignSigned(contractKey!)
    await page.waitForTimeout(2000)

    // ===== ETAPA 6: Processar Pagamento (Stripe) =====
    // Listener cria subscription no Stripe
    await expect(page.getByText('Aguardando pagamento')).toBeVisible()

    // Simular pagamento aprovado
    const invoiceId = `inv_test_${Date.now()}`
    await webhooks.mockStripePaymentSuccess(invoiceId)
    await page.waitForTimeout(3000)

    // ===== ETAPA 7: Validar Projeto Ativado =====
    await expect(page).toHaveURL(/\/projects\//)

    const projectPage = new ProjectPage(page)
    await projectPage.waitForProjectActive()

    await expect(page.getByText('Projeto Ativo')).toBeVisible()
    await expect(page.getByText('Empresa Teste LTDA')).toBeVisible()
    await expect(page.getByText('R$ 10.000,00')).toBeVisible()

    // Validar via GraphQL
    const project = await graphql.getProjectByDeal(dealKey)
    expect(project.status).toBe('active')
    expect(project.organization.name).toBe('Empresa Teste LTDA')

    console.log('✅ Fluxo completo validado com sucesso!')
  })
})
```

**Pontos de Atenção:**

- **Waits**: `page.waitForTimeout(2000)` aguarda listeners processarem (BullMQ)
- **Logins múltiplos**: Simula handoff entre BDR → SDR → Closer
- **Webhooks**: Mocks de ZapSign e Stripe simulam integrações externas
- **Validação dupla**: UI + GraphQL para garantir consistência

---

### Fluxo 2: Leilão de Deal de Alto Valor

**Descrição:** Deal de alto valor entra em leilão, franchisees fazem lances, vencedor paga CAC e recebe o deal.

**Duração estimada:** ~3 horas reais (teste usa mock de tempo)

**Listeners envolvidos:**
1. `AuctionCreatedUseCase` (Priority 10)
2. `AuctionEndedWithWinnerUseCase` (Priority 20)
3. `ChargeWorkspaceForCAC` (Priority 30)

**Código completo:**

```typescript
import { test, expect } from '@fixtures/test-base'
import { AuctionPage } from '@pages/auction.page'
import { CRMPage } from '@pages/crm.page'
import { TEST_USERS } from '@support/test-data'
import { DealBuilder } from '@support/builders'

test.describe('Fluxo de Leilão', () => {
  test('deve criar leilão, receber lances e transferir deal para vencedor', async ({
    page,
    auth,
    graphql,
    webhooks,
  }) => {
    // ===== ETAPA 1: Criar Deal de Alto Valor =====
    await auth.loginAsAdmin()

    const dealData = new DealBuilder()
      .withOrganization('Grande Empresa S.A.', '11.111.111/0001-11')
      .withAmount(50000) // Deal de alto valor
      .build()

    const dealKey = await graphql.createTestDeal(dealData)
    console.log('✅ Deal de alto valor criado:', dealKey)

    // ===== ETAPA 2: Marcar como Won (trigger auction) =====
    const crmPage = new CRMPage(page)
    await crmPage.openDeal(dealKey)
    await crmPage.markAsWon()

    // Listener detecta alto valor e cria leilão
    await page.waitForTimeout(2000)
    await expect(page.getByText('Leilão criado')).toBeVisible()

    const auctionKey = await page.getAttribute('[data-auction-key]', 'data-auction-key')

    // ===== ETAPA 3: Franchisees fazem lances =====
    const auctionPage = new AuctionPage(page)

    // Franchisee 1: Lance inicial
    await auth.loginAs('franchisee1@berry.com.br')
    await auctionPage.openAuction(auctionKey!)

    await expect(page.getByText('Lance Inicial: R$ 5.000,00')).toBeVisible()
    await auctionPage.placeBid(5500)
    await expect(page.getByText('Lance registrado!')).toBeVisible()

    // Franchisee 2: Lance maior
    await auth.loginAs('franchisee2@berry.com.br')
    await auctionPage.openAuction(auctionKey!)

    await expect(page.getByText('Lance Atual: R$ 5.500,00')).toBeVisible()
    await auctionPage.placeBid(6000)
    await expect(page.getByText('Você é o maior lance!')).toBeVisible()

    // Franchisee 1: Contra-lance
    await auth.loginAs('franchisee1@berry.com.br')
    await auctionPage.openAuction(auctionKey!)

    await auctionPage.placeBid(6200)
    await expect(page.getByText('Você é o maior lance!')).toBeVisible()

    // ===== ETAPA 4: Finalizar Leilão (mock de tempo) =====
    await auth.loginAsAdmin()

    // Simular fim de leilão via GraphQL (pular 3 horas)
    await graphql.endAuction(auctionKey!)

    // Listener AuctionEndedWithWinnerUseCase processa vencedor
    await page.waitForTimeout(3000)

    await auctionPage.openAuction(auctionKey!)
    await expect(page.getByText('Leilão Encerrado')).toBeVisible()
    await expect(page.getByText('Vencedor: franchisee1@berry.com.br')).toBeVisible()
    await expect(page.getByText('Valor do Lance: R$ 6.200,00')).toBeVisible()

    // ===== ETAPA 5: Cobrar CAC do Vencedor =====
    // Listener ChargeWorkspaceForCAC cria invoice no Stripe
    await page.waitForTimeout(2000)

    const invoiceId = `inv_cac_${Date.now()}`
    await webhooks.mockStripePaymentSuccess(invoiceId)
    await page.waitForTimeout(2000)

    // ===== ETAPA 6: Validar Transferência do Deal =====
    await auth.loginAs('franchisee1@berry.com.br')
    await crmPage.openDeal(dealKey)

    await expect(page.getByText('Grande Empresa S.A.')).toBeVisible()
    await expect(page.getByText('R$ 50.000,00')).toBeVisible()
    await expect(page.getByText('CAC Pago: R$ 6.200,00')).toBeVisible()

    // Validar via GraphQL
    const deal = await graphql.getDeal(dealKey)
    expect(deal.workspace).toBe('franchisee1-workspace')
    expect(deal.cacAmount).toBe(6200)

    console.log('✅ Fluxo de leilão validado com sucesso!')
  })
})
```

**Pontos de Atenção:**

- **Múltiplos logins**: Simula 3 franchisees competindo
- **Mock de tempo**: `graphql.endAuction()` pula espera de 3 horas
- **Validação de CAC**: Confirma cobrança correta do vencedor
- **Transfer de workspace**: Deal muda de Berry para franchisee

---

### Fluxo 3: WhatsApp Lead → CRM → Atribuição

**Descrição:** Lead chega via WhatsApp, Maia qualifica, cria deal no CRM e atribui a BDR.

**Duração estimada:** ~15 segundos

**Listeners envolvidos:**
1. `WhatsappIncomingUseCase` (Priority 5)
2. `MaiaRespondUseCase` (Priority 10)
3. `CrmLeadAnalysisUseCase` (Priority 15)
4. `CrmDealIsMQL` (Priority 20)

**Código completo:**

```typescript
import { test, expect } from '@fixtures/test-base'
import { CRMPage } from '@pages/crm.page'
import { ChatPage } from '@pages/chat.page'
import { TEST_USERS } from '@support/test-data'

test.describe('Fluxo WhatsApp → CRM', () => {
  test('deve receber lead via WhatsApp, Maia qualificar e criar deal', async ({
    page,
    auth,
    graphql,
    webhooks,
  }) => {
    // ===== ETAPA 1: Simular Mensagem WhatsApp =====
    const whatsappPayload = {
      from: '+5511999887766',
      message: {
        type: 'text',
        text: {
          body: 'Olá! Preciso de consultoria em marketing digital. Minha empresa fatura R$ 500k/mês.',
        },
      },
      timestamp: Date.now(),
    }

    await webhooks.mockWhatsAppIncoming(whatsappPayload)

    // Listener WhatsappIncomingUseCase processa mensagem
    await page.waitForTimeout(2000)

    // ===== ETAPA 2: Validar Conversa Criada =====
    await auth.loginAsBDR()
    const chatPage = new ChatPage(page)

    await chatPage.openWhatsAppConversations()
    await expect(page.getByText('+5511999887766')).toBeVisible()

    await chatPage.openConversation('+5511999887766')
    await expect(page.getByText('Preciso de consultoria em marketing digital')).toBeVisible()

    // ===== ETAPA 3: Maia Responde e Qualifica =====
    // Listener MaiaRespondUseCase analisa e responde
    await page.waitForTimeout(3000)

    await expect(page.getByText('Olá! Sou a Maia, assistente virtual da Berry.')).toBeVisible()
    await expect(page.getByText('Lead Score: 4/5')).toBeVisible()

    // ===== ETAPA 4: Validar Deal Criado no CRM =====
    // Listener CrmLeadAnalysisUseCase cria deal
    await page.waitForTimeout(2000)

    const crmPage = new CRMPage(page)
    await crmPage.goToDeals()

    await expect(page.getByText('Lead WhatsApp: +5511999887766')).toBeVisible()

    const dealRow = page.locator('tr', { hasText: '+5511999887766' })
    await dealRow.click()

    // ===== ETAPA 5: Validar Dados do Deal =====
    await expect(page.getByText('Status: MQL')).toBeVisible()
    await expect(page.getByText('Lead Score: 4')).toBeVisible()
    await expect(page.getByText('Fonte: WhatsApp')).toBeVisible()
    await expect(page.getByText('Receita Estimada: R$ 500.000,00')).toBeVisible()

    // Listener CrmDealIsMQL atribui BDR
    await expect(page.getByText('Atribuído a: bdr@berry.com.br')).toBeVisible()

    // ===== ETAPA 6: Validar Tarefa Criada =====
    await page.click('text=Tarefas')
    await expect(page.getByText('Entrar em contato via WhatsApp')).toBeVisible()
    await expect(page.getByText('Prioridade: Alta')).toBeVisible()

    // Validar via GraphQL
    const deals = await graphql.getDeals({ source: 'whatsapp' })
    expect(deals.length).toBeGreaterThan(0)

    const deal = deals[0]
    expect(deal.status).toBe('mql')
    expect(deal.leadScore).toBe(4)
    expect(deal.assignedTo).toBe('bdr@berry.com.br')

    console.log('✅ Fluxo WhatsApp → CRM validado com sucesso!')
  })
})
```

**Pontos de Atenção:**

- **Mock WhatsApp**: `webhooks.mockWhatsAppIncoming()` simula webhook real
- **Processamento Maia**: Aguarda 3 segundos para análise de IA
- **Validação multi-camada**: Chat, Deal, Tarefa
- **Lead Score**: Confirma que Maia calculou corretamente (4/5)

---

## Page Object Models

### BasePage

**`packages/e2e/pages/base.page.ts`:**

```typescript
import { Page, Locator } from '@playwright/test'

export class BasePage {
  constructor(protected page: Page) {}

  async goto(path: string) {
    await this.page.goto(path)
  }

  async waitForUrl(urlPattern: string | RegExp) {
    await this.page.waitForURL(urlPattern)
  }

  async clickButton(text: string) {
    await this.page.getByRole('button', { name: text }).click()
  }

  async fillInput(label: string, value: string) {
    await this.page.getByLabel(label).fill(value)
  }

  async selectOption(label: string, value: string) {
    await this.page.getByLabel(label).selectOption(value)
  }

  async waitForToast(message: string) {
    await this.page.waitForSelector(`text=${message}`, { timeout: 5000 })
  }

  async screenshot(name: string) {
    await this.page.screenshot({ path: `screenshots/${name}.png` })
  }
}
```

### CRMPage

**`packages/e2e/pages/crm.page.ts`:**

```typescript
import { BasePage } from './base.page'

export class CRMPage extends BasePage {
  async goToDeals() {
    await this.goto('/crm')
  }

  async openDeal(dealKey: string) {
    await this.goto(`/crm/${dealKey}`)
  }

  async markAsMQL() {
    await this.clickButton('Marcar como MQL')
    await this.waitForToast('Deal marcado como MQL')
  }

  async markAsSQL() {
    await this.clickButton('Qualificar como SQL')
    await this.waitForToast('Deal qualificado como SQL')
  }

  async markAsWon() {
    await this.clickButton('Marcar como Won')
    await this.waitForToast('Deal fechado com sucesso!')
  }

  async fillMeetingOutcome(data: { outcome: string; notes: string }) {
    await this.selectOption('Resultado', data.outcome)
    await this.fillInput('Observações', data.notes)
    await this.clickButton('Salvar Reunião')
  }
}
```

### ContractPage

**`packages/e2e/pages/contract.page.ts`:**

```typescript
import { BasePage } from './base.page'

export class ContractPage extends BasePage {
  async openFromDeal(dealKey: string) {
    await this.goto(`/contracts?deal=${dealKey}`)
  }

  async fillContractDetails(data: {
    monthlyAmount?: number
    duration?: number
    paymentMethod?: string
  }) {
    if (data.monthlyAmount) {
      await this.fillInput('Valor Mensal', data.monthlyAmount.toString())
    }
    if (data.duration) {
      await this.fillInput('Duração (meses)', data.duration.toString())
    }
    if (data.paymentMethod) {
      await this.selectOption('Forma de Pagamento', data.paymentMethod)
    }
  }

  async submitForSigning() {
    await this.clickButton('Enviar para Assinatura')
    await this.waitForToast('Contrato enviado para assinatura')
  }
}
```

---

## Padrões de Mocking

### Mock de WhatsApp

```typescript
export class WebhookMock {
  async mockWhatsAppIncoming(payload: {
    from: string
    message: { type: string; text: { body: string } }
  }) {
    await this.sendWebhook('/api/webhooks/whatsapp', {
      type: 'message',
      from: payload.from,
      message: payload.message,
      timestamp: Date.now(),
    })
  }

  async mockWhatsAppStatusUpdate(messageId: string, status: string) {
    await this.sendWebhook('/api/webhooks/whatsapp', {
      type: 'status',
      messageId,
      status, // 'sent', 'delivered', 'read'
    })
  }
}
```

### Mock de Google Calendar

```typescript
// packages/e2e/fixtures/google-mock.ts
export class GoogleMock {
  constructor(private page: Page) {}

  async mockCalendarEventCreated(eventId: string) {
    await this.page.route('**/googleapis.com/calendar/v3/calendars/**', async (route) => {
      if (route.request().method() === 'POST') {
        await route.fulfill({
          status: 200,
          body: JSON.stringify({
            id: eventId,
            status: 'confirmed',
            htmlLink: `https://calendar.google.com/event/${eventId}`,
          }),
        })
      }
    })
  }
}
```

---

## Assertions e Validações

### Validação de UI

```typescript
// Texto visível
await expect(page.getByText('Deal fechado')).toBeVisible()

// URL correta
await expect(page).toHaveURL(/\/projects\/proj_\w+/)

// Elemento existe
await expect(page.locator('[data-status="active"]')).toHaveCount(1)

// Valor de input
await expect(page.getByLabel('Valor')).toHaveValue('10000')

// Estado de botão
await expect(page.getByRole('button', { name: 'Salvar' })).toBeDisabled()
```

### Validação via GraphQL

```typescript
// packages/e2e/support/assertions.ts
export async function assertDealStatus(
  graphql: GraphQLClient,
  dealKey: string,
  expectedStatus: string,
) {
  const deal = await graphql.getDeal(dealKey)
  expect(deal.status).toBe(expectedStatus)
}

export async function assertProjectActive(
  graphql: GraphQLClient,
  projectKey: string,
) {
  const project = await graphql.getProject(projectKey)
  expect(project.status).toBe('active')
  expect(project.startDate).toBeTruthy()
}
```

### Validação de Eventos

```typescript
// Validar que listener executou
export async function assertListenerExecuted(
  graphql: GraphQLClient,
  eventType: string,
  documentKey: string,
) {
  const events = await graphql.getEvents({
    type: eventType,
    documentKey,
  })
  expect(events.length).toBeGreaterThan(0)
  expect(events[0].status).toBe('completed')
}
```

---

## Best Practices

### 1. Use Fixtures para Setup

**❌ Ruim:**

```typescript
test('teste', async ({ page }) => {
  await page.goto('/login')
  await page.fill('[name="email"]', 'test@berry.com.br')
  await page.fill('[name="password"]', 'senha123')
  await page.click('button[type="submit"]')
  // ... resto do teste
})
```

**✅ Bom:**

```typescript
test('teste', async ({ page, auth }) => {
  await auth.loginAsBDR()
  // ... resto do teste
})
```

### 2. Use Page Objects

**❌ Ruim:**

```typescript
test('marcar deal como won', async ({ page }) => {
  await page.click('button:has-text("Marcar como Won")')
  await page.waitForSelector('text=Deal fechado')
})
```

**✅ Bom:**

```typescript
test('marcar deal como won', async ({ page }) => {
  const crmPage = new CRMPage(page)
  await crmPage.markAsWon()
})
```

### 3. Use Builders para Dados de Teste

**❌ Ruim:**

```typescript
const dealData = {
  organization: { name: 'Test', cnpj: '00.000.000/0000-00' },
  contact: { name: 'Test', email: 'test@test.com', phone: '+5511999999999' },
  amount: 5000,
  services: ['CRM'],
}
```

**✅ Bom:**

```typescript
const dealData = new DealBuilder()
  .withOrganization('Test Company', '00.000.000/0000-00')
  .withAmount(5000)
  .build()
```

### 4. Aguarde Listeners com Waits Explícitos

**❌ Ruim:**

```typescript
await crmPage.markAsMQL()
await expect(page.getByText('Tarefa criada')).toBeVisible() // Pode falhar se listener for lento
```

**✅ Bom:**

```typescript
await crmPage.markAsMQL()
await page.waitForTimeout(2000) // Aguarda listener processar
await expect(page.getByText('Tarefa criada')).toBeVisible()
```

### 5. Valide UI + API

**❌ Ruim:**

```typescript
await expect(page.getByText('Projeto Ativo')).toBeVisible()
// Só valida UI, não garante que banco está correto
```

**✅ Bom:**

```typescript
await expect(page.getByText('Projeto Ativo')).toBeVisible()

const project = await graphql.getProject(projectKey)
expect(project.status).toBe('active')
```

### 6. Isole Testes com Dados Únicos

**❌ Ruim:**

```typescript
const dealData = { organization: { name: 'Test Company' } }
// Todos os testes usam mesmo nome, podem conflitar
```

**✅ Bom:**

```typescript
const dealData = {
  organization: { name: `Test Company ${Date.now()}` },
}
// Nome único por execução
```

### 7. Use Tags para Priorização

```typescript
test.describe('Fluxos Críticos', () => {
  test('lead → projeto @critical @smoke', async ({ page, auth }) => {
    // Teste crítico, deve rodar em smoke tests
  })

  test('leilão completo @critical', async ({ page, auth }) => {
    // Teste crítico, mas mais lento
  })
})
```

Execute apenas testes críticos:

```bash
pnpm test --grep @critical
```

---

## Exemplos Adicionais

### Exemplo 1: Validar Emails Enviados

```typescript
test('deve enviar email de boas-vindas ao fechar deal', async ({ page, auth, webhooks }) => {
  await auth.loginAsCloser()

  const dealKey = await graphql.createTestDeal(TEST_DEAL)
  const crmPage = new CRMPage(page)

  await crmPage.openDeal(dealKey)
  await crmPage.markAsWon()

  // Mock Brevo (email service)
  const emailSent = await webhooks.waitForBrevoEmail({
    to: TEST_DEAL.contact.email,
    template: 'welcome',
  })

  expect(emailSent).toBeTruthy()
  expect(emailSent.subject).toContain('Bem-vindo à Berry')
})
```

### Exemplo 2: Validar Falha de Pagamento

```typescript
test('deve notificar quando pagamento falhar', async ({ page, auth, webhooks }) => {
  await auth.loginAsAdmin()

  const dealKey = await graphql.createTestDeal(TEST_DEAL)
  const crmPage = new CRMPage(page)

  await crmPage.openDeal(dealKey)
  await crmPage.markAsWon()

  // Simular falha de pagamento Stripe
  const invoiceId = `inv_test_${Date.now()}`
  await webhooks.mockStripePaymentFailed(invoiceId, 'card_declined')

  await page.waitForTimeout(2000)

  // Validar notificação
  await expect(page.getByText('Pagamento recusado')).toBeVisible()
  await expect(page.getByText('Método de pagamento inválido')).toBeVisible()

  // Validar que projeto NÃO foi ativado
  const project = await graphql.getProjectByDeal(dealKey)
  expect(project.status).toBe('awaiting_payment')
})
```

### Exemplo 3: Validar Permissões

```typescript
test('franchisee não deve ver deals de outro workspace', async ({ page, auth }) => {
  await auth.loginAs('franchisee1@berry.com.br')

  const dealFromAnotherWorkspace = await graphql.createTestDeal({
    ...TEST_DEAL,
    workspace: 'franchisee2-workspace',
  })

  const crmPage = new CRMPage(page)
  await crmPage.goToDeals()

  // Não deve aparecer na lista
  await expect(page.getByText(dealFromAnotherWorkspace)).not.toBeVisible()

  // Tentar acessar diretamente deve dar erro
  await crmPage.openDeal(dealFromAnotherWorkspace)
  await expect(page.getByText('Acesso negado')).toBeVisible()
})
```

---

## Resumo de Comandos

```bash
# Executar todos os testes E2E
pnpm --filter @berry/e2e test

# Executar apenas fluxos críticos
pnpm --filter @berry/e2e test --grep @critical

# Executar em modo UI interativo
pnpm --filter @berry/e2e test:ui

# Executar teste específico
pnpm --filter @berry/e2e test tests/deals.spec.ts

# Executar com debug
pnpm --filter @berry/e2e test:debug

# Ver relatório de resultados
pnpm --filter @berry/e2e report
```

---

## Próximos Passos

1. ✅ Setup técnico ([e2e-setup.md](./e2e-setup.md))
2. ✅ Guia prático (este documento)
3. 🔧 Implementar estrutura `packages/e2e/`
4. 🧪 Escrever testes para os 3 fluxos críticos
5. 📊 Integrar com processo de Code Review
6. 🚀 Configurar CI/CD (quando processo de entregas estiver pronto)

**Dúvidas?** Consulte o Tech Lead ou abra issue no repositório.
