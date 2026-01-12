# Testes Automatizados - Berry

Para documentação do Vitest, consulte [vitest.dev](https://vitest.dev/).

---

## Stack

- **Framework:** Vitest 2.1.9
- **Runtime:** Node.js 24+
- **Coverage threshold:** 90% (backend)

```bash
# Comandos
pnpm test              # Rodar todos
pnpm test:ui           # UI interativa
pnpm test:coverage     # Com cobertura
npx vitest src/deals/  # Específico
```

---

## Estrutura

Testes ficam **ao lado do código** (co-location):

```
src/deals/
├── service.ts
├── service.test.ts      # ← teste ao lado
├── cases/
│   ├── lead-score.ts
│   └── lead-score.test.ts
```

---

## Padrão AAA

```typescript
it('should calculate CAC correctly', () => {
  // Arrange
  const bidAmount = 5000
  const fee = 0.24

  // Act
  const cac = calculateCAC(bidAmount, fee)

  // Assert
  expect(cac).toBe(6200)
})
```

---

## Mocking

### Setup de mocks

```typescript
import { beforeEach, describe, expect, it, vi } from 'vitest'

// Mock ANTES de importar
vi.mock('@app/infra/database')
vi.mock('@app/infra/logger')

// Importar DEPOIS
import { dealService } from './service'
import { db } from '@app/infra/database'

describe('DealService', () => {
  beforeEach(() => {
    vi.clearAllMocks() // SEMPRE limpar
  })
})
```

### Mocks compartilhados

**Arquivo:** `packages/api/src/infra/test/`

```typescript
// infra-mocks.ts
export const databaseMock = () => ({
  get: vi.fn().mockResolvedValue({}),
  query: vi.fn().mockResolvedValue([]),
  insertOne: vi.fn().mockResolvedValue({ _key: 'test-key', new: {} }),
  updateOne: vi.fn().mockResolvedValue({ _key: 'test-key', new: {}, old: {} }),
})

export const redisMock = () => ({
  get: vi.fn(),
  set: vi.fn(),
  del: vi.fn(),
})

export const loggerMock = () => ({
  warn: vi.fn(),
  error: vi.fn(),
})
```

### Mock de environment

```typescript
beforeEach(() => {
  vi.stubEnv('STRIPE_SECRET_KEY', 'sk_test_123')
})

afterEach(() => {
  vi.unstubAllEnvs()
})
```

---

## Factory Functions

```typescript
const createMockDeal = (overrides?: Partial<Deal>): Deal => ({
  _key: 'deal-123',
  title: 'Test Deal',
  status: 'open',
  workspace: 'workspace-1',
  ...overrides,
})

// Uso
const deal = createMockDeal({ status: 'won', leadScore: 5 })
```

---

## Template de Teste

```typescript
import { beforeEach, describe, expect, it, vi } from 'vitest'

vi.mock('@app/module/service')

import { functionToTest } from './feature'
import { someService } from '@app/module/service'

describe('Feature Name', () => {
  const createMockData = (overrides?: Partial<Data>): Data => ({
    _key: 'test-123',
    name: 'Test',
    ...overrides,
  })

  beforeEach(() => {
    vi.clearAllMocks()
  })

  describe('successful scenarios', () => {
    it('should work with valid input', async () => {
      vi.mocked(someService).mockResolvedValue({ success: true })
      const result = await functionToTest(createMockData())
      expect(result).toBeDefined()
    })
  })

  describe('edge cases', () => {
    it('should handle empty values', async () => {
      const result = await functionToTest(createMockData({ name: '' }))
      expect(result).toBeDefined()
    })
  })

  describe('error handling', () => {
    it('should handle service errors', async () => {
      vi.mocked(someService).mockRejectedValue(new Error('Failed'))
      await expect(functionToTest(createMockData())).rejects.toThrow()
    })
  })
})
```

---

## Testando Controllers GraphQL

```typescript
describe('getDealController', () => {
  const mockContext: GraphqlContext = {
    user: 'user-1',
    workspace: 'workspace-1',
    locale: 'pt-BR',
  }

  it('should return deal when found', async () => {
    vi.mocked(dealService.get).mockResolvedValue(mockDeal)
    const result = await getDealController(null, { key: 'deal-123' }, mockContext)
    expect(result).toEqual(mockDeal)
  })

  it('should enforce workspace isolation', async () => {
    vi.mocked(dealService.get).mockResolvedValue({ ...mockDeal, workspace: 'other' })
    await expect(getDealController(null, { key: 'deal-123' }, mockContext))
      .rejects.toThrow('Access denied')
  })
})
```

---

## Testando Event Listeners

```typescript
describe('CrmDealIsMQL', () => {
  let listener: CrmDealIsMQL

  beforeEach(() => {
    vi.clearAllMocks()
    listener = new CrmDealIsMQL()
  })

  it('should have correct priority', () => {
    expect(listener.priority).toBe(10)
  })

  describe('shouldRun', () => {
    it('should return true when deal becomes MQL', async () => {
      const event = {
        data: { status: 'mql' },
        previousData: { status: 'analysis' },
      }
      expect(await listener.shouldRun(event)).toBe(true)
    })

    it('should return false when already MQL', async () => {
      const event = {
        data: { status: 'mql' },
        previousData: { status: 'mql' },
      }
      expect(await listener.shouldRun(event)).toBe(false)
    })
  })
})
```

---

## Checklist

- [ ] `beforeEach` com `vi.clearAllMocks()`
- [ ] Mocks declarados ANTES dos imports
- [ ] Factory functions para dados de teste
- [ ] Testes de sucesso, edge cases e erros
- [ ] Verificar chamadas com `toHaveBeenCalledWith`
- [ ] Coverage ≥ 90%

---

## Anti-Patterns

| ❌ Evitar | ✅ Fazer |
|-----------|----------|
| Dados hardcoded repetidos | Factory functions |
| Mocks após imports | Mocks antes de importar |
| Esquecer `clearAllMocks` | Sempre em `beforeEach` |
| Testar implementação | Testar comportamento |
| Testes dependentes entre si | Testes isolados |
