# Guia de Testes Automatizados - Berry

## 1. Introdução

### 1.1 Por que Testes Automatizados?

Testes automatizados são essenciais para o desenvolvimento moderno de software na Berry. Eles fornecem:

- **Confiança:** Deploy com segurança sabendo que funcionalidades críticas estão protegidas contra regressões
- **Velocidade:** Detectar bugs imediatamente durante o desenvolvimento, não em produção
- **Documentação:** Testes servem como documentação viva do comportamento esperado do sistema
- **Refatoração:** Permite melhorias e otimizações no código sem medo de quebrar funcionalidades existentes
- **Qualidade:** Código testado é código mais robusto, com menos edge cases não tratados

### 1.2 Tipos de Testes na Berry

**Unit Tests (Testes Unitários):**
- Testam funções, classes ou métodos isoladamente
- Rápidos (< 1s por teste)
- Usam mocks para dependências externas
- Exemplo: Testar cálculo de CAC, validação de input, funções utilitárias

**Integration Tests (Testes de Integração):**
- Testam interação entre múltiplos componentes ou módulos
- Mais lentos (1-5s por teste)
- Podem usar dependências reais (Elasticsearch, Redis) ou mocks complexos
- Exemplo: Testar fluxo de Stripe webhook → Deal update, GraphQL resolver com múltiplas dependências

### 1.3 Quando Escrever Cada Tipo

**Unit Test quando:**
- Função é pura (sem side effects)
- Lógica de negócio complexa que pode ser isolada
- Cálculos matemáticos ou financeiros
- Validações e transformações de dados
- Utilities e helpers

**Integration Test quando:**
- Fluxo entre múltiplos módulos (Deal → Contract → Payment)
- Integrações externas (Stripe, Google, WhatsApp)
- Event listeners que orquestram múltiplos serviços
- GraphQL resolvers com múltiplas dependências
- Casos onde o comportamento emerge da interação entre componentes

### 1.4 Relação com Fluxo de Desenvolvimento

Testes são parte essencial do **Definition of Done**:

1. Feature implementada com código funcional
2. **Testes escritos** (unit + integration conforme necessário)
3. **Coverage >= 90%** mantido (backend)
4. Testes passando localmente (`npm run test`)
5. Code review (reviewer verifica qualidade dos testes)
6. CI/CD passa (quando configurado)
7. Merge aprovado

**Importante:** Code review deve incluir verificação de testes. Reviewers devem validar que os testes cobrem casos de sucesso, edge cases e cenários de erro.

---

## 2. Configuração e Stack de Testes

### 2.1 Backend (Maia API)

**Framework:** Vitest 2.1.9
**Runtime:** Node.js 22+
**Linguagem:** TypeScript (strict mode)

**Arquivo de configuração:** `packages/api/vite.config.ts`

```typescript
import tsconfigPaths from 'vite-tsconfig-paths'
import { defineConfig } from 'vitest/config'

export default defineConfig({
  plugins: [tsconfigPaths()],
  resolve: {
    alias: {
      '@app': '/src',
    },
  },
  test: {
    globals: true,                    // Vitest globals sem import
    include: ['src/**/*.test.ts'],    // Pattern matching
    coverage: {
      thresholds: {
        lines: 90,                    // 90% obrigatório
        functions: 90,
        branches: 90,
        statements: 90,
      },
    },
  },
})
```

**Scripts no `package.json`:**

```json
{
  "scripts": {
    "test": "vitest run --reporter=dot",
    "test:ui": "vitest --reporter=verbose --ui",
    "test:coverage": "vitest run --coverage --reporter=verbose"
  }
}
```

**Como rodar:**

```bash
# Rodar todos os testes (rápido)
npm run test

# Rodar com UI interativa
npm run test:ui

# Gerar relatório de cobertura
npm run test:coverage

# Rodar teste específico
npx vitest src/deals/cases/lead-score.test.ts

# Watch mode (re-roda ao salvar)
npx vitest --watch
```

### 2.2 Frontend (Maia Vite)

**Framework:** Vitest 2.1.9
**Runtime:** React 19.1.0
**Linguagem:** TypeScript

**Scripts no `package.json`:**

```json
{
  "scripts": {
    "test": "vitest"
  }
}
```

**Status atual:** Sem configuração explícita de `vitest.config.ts`, usa defaults do Vitest.

**Roadmap:**
- **Fase 1 (Atual):** Testar utilities, services, cálculos (funções puras)
- **Fase 2 (Futuro):** Adicionar React Testing Library e testar componentes
- **Fase 3 (Futuro):** E2E tests com Playwright

---

## 3. Testes Backend - Visão Geral

### 3.1 Estatísticas Atuais

- **Total de Test Files:** 125 arquivos `.test.ts`
- **Linhas de Teste:** ~49,563 linhas
- **Coverage Threshold:** 90% (lines, functions, branches, statements)
- **Framework:** Vitest 2.1.9 com Node.js 22+
- **Setup com `beforeEach`:** 313 usages

### 3.2 Estrutura de Arquivos

Testes ficam **ao lado do código** que testam (co-location pattern):

```
packages/api/src/
├── deals/
│   ├── cases/
│   │   ├── lead-score.ts
│   │   └── lead-score.test.ts       # ← Teste ao lado
│   ├── service.ts
│   └── service.test.ts               # ← Teste ao lado
├── infra/
│   ├── test/                         # ← Mocks compartilhados
│   │   ├── infra-mocks.ts
│   │   ├── service-mocks.ts
│   │   └── util-mocks.ts
│   ├── crypto.ts
│   └── crypto.test.ts                # ← Teste ao lado
```

**Vantagens da co-location:**
- Fácil encontrar testes relacionados ao código
- Refatoração move testes junto com o código
- Importação de tipos fica simples

### 3.3 Padrões Identificados

**AAA Pattern (Arrange-Act-Assert):**

```typescript
it('should calculate CAC correctly', () => {
  // Arrange: Setup dados de teste
  const bidAmount = 5000
  const applicationFee = 0.24

  // Act: Executar função sendo testada
  const cac = calculateCAC(bidAmount, applicationFee)

  // Assert: Verificar resultado esperado
  expect(cac).toBe(6200) // 5000 + (5000 * 0.24)
})
```

**Factory Functions para Dados Complexos:**

```typescript
const createMockDeal = (overrides?: Partial<Deal>): Deal => ({
  _key: 'deal-123',
  title: 'Test Deal',
  status: 'open',
  type: 'inbound',
  workspaceKey: 'workspace-1',
  leadScore: 3,
  ...overrides, // Permite customização por teste
})

// Uso
const deal = createMockDeal({ status: 'won', leadScore: 5 })
```

**Mocks Centralizados:**

```typescript
// Importar de infra/test
import { databaseMock, cacheMock, redisMock } from '@app/infra/test/infra-mocks'
```

### 3.4 Coverage Atual

**Módulos com Boa Cobertura:**
- ✅ Deals: 20+ test files (~9,892 linhas)
- ✅ Stripe/Payment: 18+ test files
- ✅ Authentication: Token, User, Code cases
- ✅ Projects: Create, Transfer, Archive, Export
- ✅ Auctions: Create, End, Bid

**Gaps Identificados:**
- GraphQL resolvers (controllers) - poucos testes diretos
- REST endpoints (Fastify routes) - não testados diretamente
- Performance/concurrency tests - não implementados

---

## 4. Testes Backend - Padrões de Mocking

### 4.1 Estrutura de Mocks Compartilhados

**Local:** `packages/api/src/infra/test/`

**Três categorias principais:**

1. **`infra-mocks.ts`** - Infraestrutura (Redis, Database, Logger, WebSocket)
2. **`service-mocks.ts`** - Serviços de negócio (ProjectService, TaskService, etc.)
3. **`util-mocks.ts`** - Utilities (isNotEmptyValue, isEmptyValue)

### 4.2 Exemplos de Mocks de Infraestrutura

**Redis Mock:**

```typescript
export const redisMock = (): unknown => ({
  get: vi.fn(),
  set: vi.fn(),
  del: vi.fn(),
  hget: vi.fn(),
  hset: vi.fn(),
  hdel: vi.fn(),
  on: vi.fn(),
  subscribe: vi.fn(),
  sadd: vi.fn(),
  srem: vi.fn(),
  keys: vi.fn().mockResolvedValue([]),
  publish: vi.fn(),
})
```

**Database Mock:**

```typescript
export const databaseMock = (): unknown => ({
  connection: {},
  get: vi.fn().mockResolvedValue({}),
  query: vi.fn().mockResolvedValue([]),
  queryOne: vi.fn().mockResolvedValue({}),
  insertOne: vi.fn().mockResolvedValue({
    _key: 'test-key',
    _id: 'test-collection/test-key',
    _rev: 'test-rev',
    new: {},
  }),
  updateOne: vi.fn().mockResolvedValue({
    _key: 'test-key',
    _id: 'test-collection/test-key',
    _rev: 'test-rev',
    new: {},
    old: {},
  }),
  removeOne: vi.fn().mockResolvedValue({
    _key: 'test-key',
    _id: 'test-collection/test-key',
    _rev: 'test-rev',
  }),
})
```

**Logger Mock:**

```typescript
export const loggerMock = (): {
  warn: ReturnType<typeof vi.fn>
  error: ReturnType<typeof vi.fn>
} => ({
  warn: vi.fn(),
  error: vi.fn(),
})
```

### 4.3 Como Usar Mocks

**Padrão 1: Mock com `vi.mock()` ANTES de imports**

```typescript
import { beforeEach, describe, expect, it, vi } from 'vitest'

// IMPORTANTE: Mock ANTES de importar o código
vi.mock('@app/infra/database')
vi.mock('@app/infra/logger')

// Importar APÓS mocks
import { dealService } from '@app/deals/service'
import { Logger } from '@app/infra/logger'

describe('DealService', () => {
  beforeEach(() => {
    vi.clearAllMocks() // Limpar histórico de chamadas
  })

  it('should log errors', async () => {
    vi.mocked(Logger.error).mockImplementation(() => {})

    // Seu teste aqui
  })
})
```

**Padrão 2: Mock com implementação customizada**

```typescript
vi.mocked(isNotEmptyValue).mockReturnValue(true)
vi.mocked(maiaGenerate).mockResolvedValue(createMockGenerateResult('85.5'))
vi.mocked(getPromptByName).mockReturnValue(mockTemplate)
```

**Padrão 3: Mock de Environment Variables**

```typescript
beforeEach(() => {
  vi.stubEnv('STRIPE_SECRET_KEY', 'sk_test_123')
  vi.stubEnv('NODE_ENV', 'test')
})

afterEach(() => {
  vi.unstubAllEnvs()
})
```

### 4.4 Como Criar Novos Mocks

Quando adicionar um novo mock compartilhado, siga este padrão:

**Para novo Service:**

```typescript
// packages/api/src/infra/test/service-mocks.ts

export const mockNewServiceImport = (): void => {
  vi.mock('@app/new/service', () => ({
    newService: vi.fn(),
  }))
}

export const newServiceMock = (): { method: Mock } => ({
  method: vi.fn().mockResolvedValue({ success: true }),
})
```

**Para nova Infraestrutura:**

```typescript
// packages/api/src/infra/test/infra-mocks.ts

export const newInfraMock = (): unknown => ({
  connect: vi.fn().mockResolvedValue(true),
  disconnect: vi.fn().mockResolvedValue(void 0),
  query: vi.fn().mockResolvedValue([]),
})
```

---

## 5. Testes Backend - Estrutura de Testes

### 5.1 Template Básico

```typescript
import { beforeEach, describe, expect, it, vi, type Mock } from 'vitest'

// Mock dependencies FIRST
vi.mock('@app/module/service')
vi.mock('@app/infra/logger')

// Import AFTER mocking
import { someService } from '@app/module/service'
import { Logger } from '@app/infra/logger'
import { functionToTest } from './feature'

describe('Feature Name', () => {
  // Factory para dados de teste
  const createMockData = (overrides?: Partial<Data>): Data => ({
    _key: 'test-123',
    name: 'Test Data',
    value: 100,
    ...overrides,
  })

  beforeEach(() => {
    vi.clearAllMocks() // SEMPRE limpar mocks
  })

  describe('successful scenarios', () => {
    it('should work correctly with valid input', async () => {
      // Arrange
      const mockData = createMockData({ value: 500 })
      vi.mocked(someService).mockResolvedValue({ success: true })

      // Act
      const result = await functionToTest(mockData)

      // Assert
      expect(result).toBeDefined()
      expect(someService).toHaveBeenCalledWith(mockData)
    })
  })

  describe('edge cases', () => {
    it('should handle empty values', async () => {
      const result = await functionToTest(createMockData({ value: 0 }))
      expect(result).toBeDefined()
    })

    it('should handle null values', async () => {
      const result = await functionToTest(createMockData({ name: undefined }))
      expect(result).toBeDefined()
    })
  })

  describe('error handling', () => {
    it('should handle service errors gracefully', async () => {
      vi.mocked(someService).mockRejectedValue(new Error('Service failed'))

      await expect(functionToTest(createMockData())).rejects.toThrow()
      expect(Logger.error).toHaveBeenCalled()
    })
  })
})
```

### 5.2 Organização de Testes

**Ordem recomendada:**
1. **successful scenarios** - Casos de sucesso (happy path)
2. **edge cases** - Casos extremos (valores vazios, limites, null/undefined)
3. **error handling** - Cenários de erro (falhas de rede, validações, exceptions)

**Nomenclatura de testes:**
- ✅ `should calculate total correctly`
- ✅ `should throw error when input is invalid`
- ✅ `should handle empty array`
- ❌ `test 1`, `works`, `correct`

---

## 6. Testes Backend - Exemplos Práticos

### 6.1 Exemplo 1: Unit Test Simples (Crypto)

**Cenário:** Testar funções de criptografia e hash.

**Arquivo:** `packages/api/src/infra/crypto.test.ts`

**Código completo:**

```typescript
import { afterAll, beforeAll, describe, expect, it } from 'vitest'
import {
  createHashValue,
  decrypt,
  encrypt,
  hashValue,
  validateHashedValue,
} from './crypto'

describe('crypto', () => {
  const originalEnv = process.env

  beforeAll(() => {
    // Setup environment variables para testes
    process.env.CRYPTO_KEY = '0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef'
    process.env.CRYPTO_CIPHER = '0123456789abcdef0123456789abcdef'
  })

  afterAll(() => {
    process.env = originalEnv // Restore original env
  })

  describe('hashValue', () => {
    it('should generate a hash with salt for a given value', () => {
      const value = 'test-password'
      const result = hashValue(value)

      expect(result).toHaveProperty('salt')
      expect(result).toHaveProperty('hash')
      expect(typeof result.salt).toBe('string')
      expect(typeof result.hash).toBe('string')
      expect(result.salt.length).toBe(32) // 16 bytes * 2 (hex)
      expect(result.hash.length).toBe(128) // 64 bytes * 2 (hex)
    })

    it('should generate different salt and hash for each call', () => {
      const value = 'same-password'
      const result1 = hashValue(value)
      const result2 = hashValue(value)

      expect(result1.salt).not.toBe(result2.salt)
      expect(result1.hash).not.toBe(result2.hash)
    })

    it('should handle empty string', () => {
      const result = hashValue('')

      expect(result).toHaveProperty('salt')
      expect(result).toHaveProperty('hash')
    })
  })

  describe('validateHashedValue', () => {
    it('should validate correct password', () => {
      const value = 'correct-password'
      const { salt, hash } = hashValue(value)

      const isValid = validateHashedValue({ text: value, salt, hash })

      expect(isValid).toBe(true)
    })

    it('should reject incorrect password', () => {
      const value = 'correct-password'
      const { salt, hash } = hashValue(value)

      const isValid = validateHashedValue({
        text: 'wrong-password',
        salt,
        hash,
      })

      expect(isValid).toBe(false)
    })
  })

  describe('encrypt/decrypt integration', () => {
    it('should successfully encrypt and decrypt various data', () => {
      const testCases = [
        'simple text',
        '{"json": "object"}',
        '12345',
        'multi\nline\ntext',
        '',
      ]

      testCases.forEach(testCase => {
        const encrypted = encrypt(testCase)
        const decrypted = decrypt(encrypted)
        expect(decrypted).toBe(testCase)
      })
    })
  })
})
```

**Pontos-chave:**
- ✅ Usa `beforeAll`/`afterAll` para setup de environment variables
- ✅ Testa múltiplos cenários (sucesso, edge cases)
- ✅ Verifica propriedades específicas (length, type)
- ✅ Testes de integração no final (encrypt + decrypt)

**Quando usar este padrão:**
- Funções utilitárias sem dependências externas
- Cálculos ou transformações de dados
- Validações e formatações

### 6.2 Exemplo 2: Service Test com Mocks

**Cenário:** Testar método de service que usa Database.

**Arquivo:** `packages/api/src/deals/service.test.ts` (simplificado)

**Código:**

```typescript
import { beforeEach, describe, expect, it, vi } from 'vitest'

// Mock database ANTES de importar service
vi.mock('@app/infra/database', () => ({
  db: {
    query: vi.fn(),
    collection: vi.fn(() => ({
      save: vi.fn(),
      update: vi.fn(),
    })),
  },
}))

import { dealService } from './service'
import { db } from '@app/infra/database'

describe('DealService', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  describe('get', () => {
    it('should fetch deal by key', async () => {
      const mockDeal = {
        _key: 'deal-123',
        title: 'Test Deal',
        status: 'open',
      }

      vi.mocked(db.query).mockResolvedValue({
        next: async () => mockDeal,
      } as any)

      const result = await dealService().get('deal-123')

      expect(result).toEqual(mockDeal)
      expect(db.query).toHaveBeenCalledWith(expect.any(Object))
    })

    it('should handle deal not found', async () => {
      vi.mocked(db.query).mockResolvedValue({
        next: async () => null,
      } as any)

      const result = await dealService().get('non-existent')

      expect(result).toBeNull()
    })
  })

  describe('create', () => {
    it('should create new deal and emit event', async () => {
      const newDeal = {
        title: 'New Deal',
        status: 'open',
      }

      const mockResult = {
        _key: 'deal-456',
        ...newDeal,
      }

      vi.mocked(db.collection).mockReturnValue({
        save: vi.fn().mockResolvedValue(mockResult),
      } as any)

      const result = await dealService().create(newDeal)

      expect(result).toEqual(mockResult)
      expect(db.collection).toHaveBeenCalledWith('deals')
    })
  })
})
```

**Pontos-chave:**
- ✅ Mock de database com `vi.mock()`
- ✅ `clearAllMocks()` em `beforeEach`
- ✅ Verificação de chamadas com `toHaveBeenCalledWith`
- ✅ Testa casos de sucesso e erro

**Quando usar este padrão:**
- Services que dependem de database ou cache
- Operações CRUD
- Métodos que emitem eventos

### 6.3 Exemplo 3: Event Listener Test

**Cenário:** Testar listener que executa quando Deal vira MQL.

**Arquivo:** `packages/api/src/crm/cases/deal-is-mql.test.ts` (simplificado)

**Código:**

```typescript
import { beforeEach, describe, expect, it, vi } from 'vitest'

vi.mock('@app/tasks/service')
vi.mock('@app/deals/service')

import { CrmDealIsMQL } from './deal-is-mql'
import { taskService } from '@app/tasks/service'
import { DocumentUpdatedEvent } from '@app/event/types'

describe('CrmDealIsMQL', () => {
  let listener: CrmDealIsMQL

  beforeEach(() => {
    vi.clearAllMocks()
    listener = new CrmDealIsMQL()
  })

  describe('class properties', () => {
    it('should have correct priority and model', () => {
      expect(listener.priority).toBe(1)
      expect(listener.model).toBe('Deal')
    })
  })

  describe('shouldRun', () => {
    it('should return true when deal becomes MQL', async () => {
      const event: DocumentUpdatedEvent = {
        type: 'document_updated',
        collection: 'deals',
        key: 'deal-123',
        operation: 'update',
        data: { status: 'mql' },
        previousData: { status: 'analysis' },
      }

      const result = await listener.shouldRun(event)

      expect(result).toBe(true)
    })

    it('should return false when deal is already MQL', async () => {
      const event: DocumentUpdatedEvent = {
        type: 'document_updated',
        collection: 'deals',
        key: 'deal-123',
        operation: 'update',
        data: { status: 'mql' },
        previousData: { status: 'mql' },
      }

      const result = await listener.shouldRun(event)

      expect(result).toBe(false)
    })
  })

  describe('run', () => {
    it('should create BDR assignment task', async () => {
      const event: DocumentUpdatedEvent = {
        type: 'document_updated',
        collection: 'deals',
        key: 'deal-123',
        operation: 'update',
        data: {
          _key: 'deal-123',
          title: 'Test Deal',
          status: 'mql',
          workspaceKey: 'workspace-1',
        },
        previousData: { status: 'analysis' },
      }

      vi.mocked(taskService).mockReturnValue({
        create: vi.fn().mockResolvedValue({ _key: 'task-456' }),
      } as any)

      await listener.run(event)

      expect(taskService().create).toHaveBeenCalledWith(
        expect.objectContaining({
          title: expect.stringContaining('Qualificar lead'),
          dealKey: 'deal-123',
          status: 'to_do',
          priority: 'high',
        }),
      )
    })
  })
})
```

**Pontos-chave:**
- ✅ Testa `priority` e `model` da classe
- ✅ Testa lógica de `shouldRun` com diferentes cenários
- ✅ Verifica que `run` executa ações corretas
- ✅ Mock de services usados pelo listener

**Quando usar este padrão:**
- Event listeners (classes que estendem `EventUseCase`)
- Use cases complexos com múltiplas dependências
- Fluxos de negócio acionados por eventos

---

## 7. Testes Backend - Casos Específicos

### 7.1 GraphQL Controllers

GraphQL controllers são funções que recebem `parent`, `args`, `context` e retornam dados.

**Exemplo:**

```typescript
import { beforeEach, describe, expect, it, vi } from 'vitest'

vi.mock('@app/deals/service')

import { getDealController } from './controllers/get-deal'
import { dealService } from '@app/deals/service'
import { GraphqlContext } from '@app/infra/context'

describe('getDealController', () => {
  let mockContext: GraphqlContext

  beforeEach(() => {
    vi.clearAllMocks()
    mockContext = {
      user: 'user-1',
      workspace: 'workspace-1',
      locale: 'pt-BR',
    }
  })

  it('should return deal when found', async () => {
    const mockDeal = {
      _key: 'deal-123',
      title: 'Test Deal',
      workspaceKey: 'workspace-1',
    }

    vi.mocked(dealService).mockReturnValue({
      get: vi.fn().mockResolvedValue(mockDeal),
    } as any)

    const result = await getDealController(
      null,
      { key: 'deal-123' },
      mockContext,
    )

    expect(result).toEqual(mockDeal)
  })

  it('should throw error when key is missing', async () => {
    await expect(
      getDealController(null, { key: '' }, mockContext),
    ).rejects.toThrow('Deal key is required')
  })

  it('should throw error when deal not found', async () => {
    vi.mocked(dealService).mockReturnValue({
      get: vi.fn().mockResolvedValue(null),
    } as any)

    await expect(
      getDealController(null, { key: 'deal-123' }, mockContext),
    ).rejects.toThrow('Deal not found')
  })

  it('should enforce workspace isolation', async () => {
    const mockDeal = {
      _key: 'deal-123',
      title: 'Test Deal',
      workspaceKey: 'workspace-2', // Different workspace!
    }

    vi.mocked(dealService).mockReturnValue({
      get: vi.fn().mockResolvedValue(mockDeal),
    } as any)

    await expect(
      getDealController(null, { key: 'deal-123' }, mockContext),
    ).rejects.toThrow('Access denied')
  })
})
```

**Pontos-chave:**
- Criar mock de `GraphqlContext`
- Testar workspace isolation
- Testar validação de input
- Testar casos de erro (not found, access denied)

### 7.2 Async/Await Patterns

**Exemplo de teste de função assíncrona:**

```typescript
describe('async function', () => {
  it('should resolve with data', async () => {
    const result = await asyncFunction()
    expect(result).toBeDefined()
  })

  it('should reject with error', async () => {
    await expect(asyncFunction()).rejects.toThrow('Error message')
  })

  it('should handle promise resolution', () => {
    return asyncFunction().then(result => {
      expect(result).toBeDefined()
    })
  })
})
```

### 7.3 Environment Variables

**Exemplo com `vi.stubEnv()`:**

```typescript
describe('feature with env vars', () => {
  const originalEnv = process.env

  beforeEach(() => {
    vi.stubEnv('STRIPE_SECRET_KEY', 'sk_test_123')
    vi.stubEnv('NODE_ENV', 'test')
  })

  afterEach(() => {
    vi.unstubAllEnvs()
    process.env = originalEnv
  })

  it('should use env var', () => {
    expect(process.env.STRIPE_SECRET_KEY).toBe('sk_test_123')
  })
})
```

---

## 8. Testes Frontend - Visão Geral

### 8.1 Status Atual

- **Total de Test Files:** 2 arquivos
  - `lib/contract-utils.test.ts` (88 linhas)
  - `modules/commission-management/salary-system/calculator.test.ts` (441 linhas)
- **Coverage:** < 0.2% do codebase
- **Framework:** Vitest 2.1.9 (sem React Testing Library ainda)
- **Total de componentes React:** ~1.690 arquivos `.tsx`

### 8.2 Por Que Começar Sem React Testing Library?

**Abordagem Incremental:**

1. **Fase 1 (Atual):** Testar utilities, services, cálculos (funções puras)
2. **Fase 2 (Futuro):** Adicionar React Testing Library e testar componentes
3. **Fase 3 (Futuro):** E2E tests com Playwright

**Vantagens:**
- Começar simples sem setup complexo de DOM
- Foco em lógica de negócio (não UI visual)
- Build confidence gradualmente na equipe
- Utilizar os 2 testes existentes como modelo
- Baixa curva de aprendizado para equipe nova em testes

### 8.3 Estrutura de Arquivos

Similar ao backend, testes ao lado do código:

```
packages/app/src/
├── lib/
│   ├── contract-utils.ts
│   └── contract-utils.test.ts        # ← Existe!
├── modules/
│   ├── commission-management/
│   │   └── salary-system/
│   │       ├── calculator.ts
│   │       └── calculator.test.ts    # ← Existe!
│   ├── auth/
│   │   ├── service.auth.ts
│   │   └── service.auth.test.ts      # ← Criar
│   ├── infra/
│   │   ├── http.ts
│   │   └── http.test.ts              # ← Criar
```

### 8.4 O Que Já Funciona Bem

**Teste modelo 1: `contract-utils.test.ts`**
- ✅ Factories para dados complexos (`makeProduct`, `makeItem`, `makeCoupon`)
- ✅ Múltiplos cenários (desconto fixo, percentual, combinado)
- ✅ Edge cases (arrays vazios, valores inválidos, mínimo de 0)
- ✅ Descrições em português ("deve calcular o total corretamente")

**Teste modelo 2: `calculator.test.ts`**
- ✅ Validação de input (catch errors)
- ✅ Testes de casos extremos (0%, 100%, 200%)
- ✅ Assertions específicas (`toBeCloseTo`, `toBeGreaterThan`)
- ✅ Testes isolados e independentes
- ✅ 441 linhas cobrindo múltiplos cenários

---

## 9. Testes Frontend - O Que Testar Primeiro

### 9.1 Priorização

**Prioridade 1: Utilities**
- `lib/utils.ts` (373 linhas) - Guards, string manipulation, mobile detect
- `lib/contract-utils.ts` (88 linhas) - ✅ JÁ TEM TESTE
- `lib/social-utils.ts` - Social media utilities
- `lib/lead-time.utils.ts` - Time calculations

**Prioridade 2: Services**
- `modules/infra/http.ts` (80+ linhas) - HttpService, GraphQL, SSE
- `modules/auth/service.auth.ts` (46 linhas) - OAuth, token exchange
- Services de módulos (CRM, Task, Project)

**Prioridade 3: Data Hooks (TanStack Query)**
- `modules/*/data.ts` - Hooks de `useQuery` e `useMutation`
- Testar lógica de query keys
- Testar lógica de transformação de dados

**Prioridade 4: Custom Hooks**
- `modules/infra/hooks.ts` (64 linhas) - useKeypress, useMutationObserver
- Hooks de módulos específicos

**Prioridade 5: Components (Fase 2 - Futuro)**
- Começar por componentes críticos (Login, Forms)
- Requer React Testing Library

### 9.2 Coverage Goals

**Backend:** 90% obrigatório
**Frontend:**
- **Sem threshold definido ainda** (< 0.2% atual)
- **Meta inicial:** 20% nos próximos 3 meses
- **Focar em:** utils, services, data hooks
- **Meta longo prazo:** 60-70% após implementar testes de componentes

---

## 10. Testes Frontend - Padrões Sem React Testing Library

### 10.1 Quando é Possível Testar Sem RTL

**Testável sem RTL:**
- ✅ Funções puras (utilities)
- ✅ Cálculos e validações
- ✅ Classes de service (AuthService, HttpService)
- ✅ Formatação e transformação de dados
- ✅ Lógica de negócio isolada

**Não testável sem RTL:**
- ❌ Componentes React (JSX rendering)
- ❌ Hooks que dependem de contexto React
- ❌ Interações de usuário (clicks, inputs)
- ❌ Ciclo de vida de componentes

### 10.2 Como Testar Services

**Exemplo: Mock de HttpService**

```typescript
import { describe, expect, it, vi, beforeEach } from 'vitest'

vi.mock('@app/modules/infra/http', () => ({
  http: {
    graphql: vi.fn(),
    get: vi.fn(),
    post: vi.fn(),
  },
}))

import { authService } from './service.auth'
import { http } from '@app/modules/infra/http'

describe('AuthService', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  describe('oAuth', () => {
    it('should call graphql with correct query', async () => {
      const mockResponse = {
        token: 'jwt-token-123',
        user: { _key: 'user-1', name: 'Test User' },
      }

      vi.mocked(http.graphql).mockResolvedValue(mockResponse)

      const result = await authService.oAuth({
        code: 'auth-code-123',
        redirect: 'http://localhost:3000/callback',
      })

      expect(result).toEqual(mockResponse)
      expect(http.graphql).toHaveBeenCalledWith(
        expect.any(String), // query
        expect.objectContaining({
          input: {
            code: 'auth-code-123',
            redirect: 'http://localhost:3000/callback',
          },
        }),
      )
    })

    it('should handle auth errors', async () => {
      vi.mocked(http.graphql).mockRejectedValue(
        new Error('Invalid auth code'),
      )

      await expect(
        authService.oAuth({
          code: 'invalid-code',
          redirect: 'http://localhost:3000/callback',
        }),
      ).rejects.toThrow('Invalid auth code')
    })
  })
})
```

### 10.3 Como Mock TanStack Query (quando necessário)

```typescript
vi.mock('@tanstack/react-query', () => ({
  useQuery: vi.fn(() => ({
    data: mockData,
    isLoading: false,
    error: null,
    refetch: vi.fn(),
  })),
  useMutation: vi.fn(() => ({
    mutate: vi.fn(),
    mutateAsync: vi.fn(),
    isLoading: false,
  })),
}))
```

**Nota:** Na Fase 1, preferimos testar a lógica dentro dos hooks diretamente, não o hook React em si.

### 10.4 Limitações Sem RTL

Sem React Testing Library, **não é possível testar:**
- Renderização de componentes
- Estado de componentes (useState, useEffect)
- Contextos React
- Interações visuais
- Ciclo de vida completo de componentes

**Solução:** Focar em **lógica de negócio** que pode ser extraída e testada isoladamente.

---

## 11. Testes Frontend - Estrutura de Testes

### 11.1 Template Básico (Similar ao Backend)

```typescript
import { describe, expect, it } from 'vitest'
import { functionToTest } from './feature'

// Factory para dados de teste
function makeMockData(overrides?: Partial<Data>): Data {
  return {
    _key: 'test-123',
    name: 'Test',
    value: 100,
    ...overrides,
  }
}

describe('Feature Name', () => {
  describe('functionToTest', () => {
    it('should work correctly', () => {
      const data = makeMockData({ value: 200 })
      const result = functionToTest(data)

      expect(result).toBeDefined()
      expect(result.value).toBe(200)
    })

    it('should handle edge cases', () => {
      const data = makeMockData({ value: 0 })
      const result = functionToTest(data)

      expect(result).toBeDefined()
    })
  })
})
```

### 11.2 Descrições em Português

**Padrão adotado na Berry (frontend):**

```typescript
describe('contract-utils', () => {
  describe('calculateTotalAmount', () => {
    it('deve calcular o total corretamente', () => {
      // teste
    })

    it('deve considerar quantidade', () => {
      // teste
    })

    it('deve retornar 0 para array vazio', () => {
      // teste
    })
  })
})
```

**Por que português?**
- Alinhado com o negócio (termos em português)
- Facilita comunicação com stakeholders
- Descrições mais naturais para equipe brasileira

---

## 12. Testes Frontend - Exemplos Práticos

### 12.1 Exemplo 1: Utility Function (contract-utils)

**Cenário:** Testar cálculo de valores de contrato com descontos.

**Arquivo:** `packages/app/src/lib/contract-utils.test.ts`

**Código completo:**

```typescript
import { Contract, ContractItem, Coupon } from '@app/modules/contract/entities'
import { Product } from '@app/modules/product/entities'
import { describe, expect, it } from 'vitest'
import {
  calculateDiscountedAmount,
  calculateTotalAmount,
  formatCurrency,
  getAllCoupons,
} from './contract-utils'

// Helper: criar Product mínimo
function makeProduct(): Product {
  return {
    _key: 'prod',
    name: 'Produto',
    type: 'product',
  } as Product
}

// Helper: criar ContractItem
function makeItem(amount: number, quantity: number = 1): ContractItem {
  return {
    amount,
    quantity,
    product: makeProduct(),
  }
}

// Helper: criar Coupon
function makeCoupon(props: Partial<Coupon>): Coupon {
  return {
    id: 'coupon-id',
    amountOff: 0,
    percentOff: 0,
    created: 0,
    currency: 'BRL',
    duration: 'once',
    timesRedeemed: 0,
    valid: true,
    ...props,
  } as Coupon
}

describe('contract-utils', () => {
  describe('calculateTotalAmount', () => {
    it('deve calcular o total corretamente', () => {
      const items: ContractItem[] = [
        makeItem(4000, 1),
        makeItem(800, 1),
        makeItem(200, 1),
      ]
      expect(calculateTotalAmount(items)).toBe(5000)
    })

    it('deve considerar quantidade', () => {
      const items: ContractItem[] = [makeItem(100, 3)]
      expect(calculateTotalAmount(items)).toBe(300)
    })

    it('deve retornar 0 para array vazio', () => {
      expect(calculateTotalAmount([])).toBe(0)
    })
  })

  describe('calculateDiscountedAmount', () => {
    it('deve aplicar desconto percentual', () => {
      const items: ContractItem[] = [makeItem(100, 1)]
      const coupons: Coupon[] = [makeCoupon({ percentOff: 10 })]
      // 100 * (100 - 10) / 100 = 90
      expect(calculateDiscountedAmount(items, coupons)).toBe(90)
    })

    it('deve aplicar desconto fixo', () => {
      const items: ContractItem[] = [makeItem(100, 1)]
      const coupons: Coupon[] = [makeCoupon({ amountOff: 20 })]
      // 100 - 20 = 80
      expect(calculateDiscountedAmount(items, coupons)).toBe(80)
    })

    it('deve retornar no mínimo 0', () => {
      const items: ContractItem[] = [makeItem(100, 1)]
      const coupons: Coupon[] = [makeCoupon({ amountOff: 200 })]
      expect(calculateDiscountedAmount(items, coupons)).toBe(0)
    })
  })

  describe('formatCurrency', () => {
    it('deve formatar em R$', () => {
      expect(formatCurrency(1234.56)).toBe('R$ 1.234,56')
    })
  })
})
```

**Pontos-chave:**
- ✅ Factories para criar objetos complexos (`makeProduct`, `makeItem`, `makeCoupon`)
- ✅ Testes de múltiplos cenários (desconto fixo, percentual, combinado)
- ✅ Edge cases (array vazio, valor negativo limita a 0)
- ✅ Descrições em português alinhadas com negócio

**Quando usar este padrão:**
- Utilities de cálculo financeiro
- Formatação e transformação de dados
- Validações de business rules
- Funções puras sem side effects

### 12.2 Exemplo 2: Calculator (Validação + Cálculos)

**Cenário:** Testar sistema de cálculo de salário de consultores.

**Arquivo:** `packages/app/src/modules/commission-management/salary-system/calculator.test.ts`

**Código (selecionado):**

```typescript
import { describe, expect, it } from 'vitest'
import {
  calculateConsultantSalary,
  calculatePerformancePercentage,
  validateSalaryInput,
} from './calculator'
import { ConsultantLevel, SalaryCalculationInput } from './types'

describe('Salary Calculation System', () => {
  describe('validateSalaryInput', () => {
    it('should validate correct input', () => {
      const validInput: SalaryCalculationInput = {
        consultantId: 'test-consultant',
        level: 'pleno',
        projects: [
          {
            _key: 'project-1',
            name: 'Projeto Teste 1',
            value: 10000,
            status: 'ativo',
            startDate: '2024-01-01',
            endDate: '2024-01-31',
          },
        ],
        periodMetrics: {
          npsReal: 9.0,
          processReal: 95.0,
        },
      }

      const errors = validateSalaryInput(validInput)
      expect(errors).toHaveLength(0)
    })

    it('should catch invalid consultant ID', () => {
      const invalidInput: SalaryCalculationInput = {
        consultantId: '',
        level: 'pleno',
        projects: [],
        periodMetrics: { npsReal: 9.0, processReal: 95.0 },
      }

      const errors = validateSalaryInput(invalidInput)
      expect(errors).toHaveLength(1)
      expect(errors[0].field).toBe('consultantId')
    })

    it('should catch invalid NPS values', () => {
      const invalidInput: SalaryCalculationInput = {
        consultantId: 'test',
        level: 'pleno',
        projects: [],
        periodMetrics: { npsReal: 15.0, processReal: 95.0 },
      }

      const errors = validateSalaryInput(invalidInput)
      expect(errors).toHaveLength(1)
      expect(errors[0].field).toBe('periodMetrics.npsReal')
    })
  })

  describe('calculatePerformancePercentage', () => {
    it('should handle extreme revenue (200% of target)', () => {
      const realRevenue = 200000 // 200% do target
      const targetRevenue = 100000
      const performance = calculatePerformancePercentage(
        realRevenue,
        targetRevenue,
      )

      expect(performance).toBeGreaterThan(100)
      expect(performance).toBeLessThanOrEqual(200)
    })

    it('should handle 0% performance', () => {
      const realRevenue = 0
      const targetRevenue = 100000
      const performance = calculatePerformancePercentage(
        realRevenue,
        targetRevenue,
      )

      expect(performance).toBe(0)
    })

    it('should handle 100% performance', () => {
      const realRevenue = 100000
      const targetRevenue = 100000
      const performance = calculatePerformancePercentage(
        realRevenue,
        targetRevenue,
      )

      expect(performance).toBeCloseTo(100, 1)
    })
  })
})
```

**Pontos-chave:**
- ✅ Validação de input (field-level errors)
- ✅ Edge cases (0%, 100%, 200%)
- ✅ Assertions específicas (`toBeCloseTo`, `toBeGreaterThan`)
- ✅ Testes de boundary values

**Quando usar este padrão:**
- Cálculos complexos (financeiros, métricas)
- Validação de formulários
- Business rules com múltiplos edge cases
- Funções com muitos branches (if/else, switch)

---

## 13. Boas Práticas e Checklist

### 13.1 Checklist para Novo Test File

**Setup:**
- [ ] Import vitest: `beforeEach, describe, expect, it, vi`
- [ ] Mock dependencies com `vi.mock()` ANTES de imports
- [ ] Import código após mocks
- [ ] `beforeEach(() => { vi.clearAllMocks() })`

**Estrutura:**
- [ ] Organizar em `describe` suites lógicas
- [ ] Seguir padrão: successful → edge cases → errors
- [ ] Usar factories para dados complexos
- [ ] Implementar AAA (Arrange-Act-Assert)

**Mocking (Backend):**
- [ ] Usar mocks de `infra/test/` quando disponíveis
- [ ] Adicionar novos mocks em `infra/test/` para reutilização
- [ ] Mock environment variables com `vi.stubEnv()`
- [ ] Verificar chamadas: `expect(...).toHaveBeenCalledWith()`

**Coverage (Backend):**
- [ ] Manter 90% threshold: lines, functions, branches, statements
- [ ] Testar casos de sucesso
- [ ] Testar edge cases (null, undefined, empty, extremos)
- [ ] Testar error scenarios com try-catch
- [ ] Testar integração com dependências mockadas

**Frontend:**
- [ ] Começar com utilities e services (funções puras)
- [ ] Usar exemplos existentes como modelo
- [ ] Edge cases e validações
- [ ] Descrições em português

### 13.2 Padrões de Nomenclatura

**Arquivos:**
- `feature.test.ts` - Unit tests
- `feature.integration.test.ts` - Integration tests

**Describes:**

```typescript
describe('ModuleName ou FunctionName', () => {
  describe('methodName ou scenario', () => {
    it('should do something specific', () => {})
  })
})
```

**Test descriptions:**
- ✅ "should calculate total correctly"
- ✅ "should throw error when input is invalid"
- ✅ "deve retornar vazio quando array está vazio"
- ❌ "test 1", "works", "correct"

### 13.3 Coverage Mínimo Esperado

**Backend:**
- **90% obrigatório** (lines, functions, branches, statements)
- CI/CD irá falhar se < 90% (quando configurado)

**Frontend:**
- **Sem threshold definido ainda** (< 0.2% atual)
- Meta inicial: **20% nos próximos 3 meses**
- Focar em: utils, services, data hooks
- Meta longo prazo: **60-70%** (após React Testing Library)

### 13.4 Code Review - Checklist de Testes

Reviewer deve verificar:
- [ ] Testes escritos para nova feature ou bug fix
- [ ] Coverage mantém ou aumenta threshold
- [ ] Testes passam localmente (`npm run test`)
- [ ] Factories usados para dados complexos
- [ ] Edge cases cobertos
- [ ] Error scenarios testados
- [ ] Mocks apropriados (não testar implementação de dependências)
- [ ] Descrições claras e descritivas

### 13.5 Performance de Testes

**Boas práticas:**
- Unit tests devem ser rápidos (< 1s cada)
- Integration tests podem ser lentos (1-5s)
- Use `it.only` para rodar teste específico durante desenvolvimento
- Use `it.skip` para desabilitar temporariamente
- Evite `setTimeout` - use `vi.useFakeTimers()`
- Minimize operações de I/O em unit tests

**Exemplo de fake timers:**

```typescript
import { beforeEach, afterEach, describe, it, expect, vi } from 'vitest'

describe('function with setTimeout', () => {
  beforeEach(() => {
    vi.useFakeTimers()
  })

  afterEach(() => {
    vi.useRealTimers()
  })

  it('should execute after delay', () => {
    const callback = vi.fn()
    functionWithTimeout(callback, 1000)

    // Avançar tempo
    vi.advanceTimersByTime(1000)

    expect(callback).toHaveBeenCalled()
  })
})
```

---

## 14. Troubleshooting e Erros Comuns

### 14.1 Import Errors com Path Aliases

**Erro:**

```
Cannot find module '@app/...'
```

**Solução:**

Verifique `tsconfig.json` e `vite.config.ts`:

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@app/*": ["src/*"]
    }
  }
}
```

```typescript
// vite.config.ts
import tsconfigPaths from 'vite-tsconfig-paths'

export default defineConfig({
  plugins: [tsconfigPaths()],
})
```

### 14.2 Mock Não Funcionando

**Problema:** Mock não está sendo aplicado.

**Causas comuns:**
1. Mock declarado APÓS import
2. Caminho do mock incorreto
3. `vi.clearAllMocks()` não chamado em `beforeEach`

**Solução:**

```typescript
// ✅ CORRETO: Mock ANTES de import
vi.mock('@app/module/service')

import { service } from '@app/module/service'

describe('test', () => {
  beforeEach(() => {
    vi.clearAllMocks() // Sempre limpar
  })
})
```

### 14.3 Async/Await Issues

**Erro:**

```
Test timeout (no async/await)
```

**Solução:**

```typescript
// ❌ INCORRETO
it('should work', () => {
  asyncFunction() // Sem await
  expect(result).toBeDefined()
})

// ✅ CORRETO
it('should work', async () => {
  await asyncFunction()
  expect(result).toBeDefined()
})
```

### 14.4 Coverage Não Atingindo Threshold

**Problema:** Coverage < 90% no backend.

**Solução:**
1. Rodar `npm run test:coverage` para ver quais linhas não estão cobertas
2. Adicionar testes para branches não testados (if/else)
3. Testar error scenarios (try/catch)
4. Verificar edge cases

**Exemplo de branch não coberto:**

```typescript
function divide(a: number, b: number): number {
  if (b === 0) {
    throw new Error('Cannot divide by zero') // ← Não testado!
  }
  return a / b
}

// Adicionar teste:
it('should throw error when dividing by zero', () => {
  expect(() => divide(10, 0)).toThrow('Cannot divide by zero')
})
```

### 14.5 Testes Flaky (Instáveis)

**Problema:** Testes passam às vezes, falham outras vezes.

**Causas comuns:**
1. Dependência de timers (setTimeout, setInterval)
2. Dependência de ordem de execução
3. Estado compartilhado entre testes
4. Operações assíncronas não aguardadas

**Solução:**
1. Use `vi.useFakeTimers()`
2. Garanta que `beforeEach` limpa todo estado
3. Sempre use `await` em operações assíncronas
4. Não compartilhe variáveis entre testes

---

## 15. Referências e Recursos

### 15.1 Arquivos de Teste Modelo (Backend)

**Unit Tests Simples:**
- `packages/api/src/infra/crypto.test.ts` - Criptografia e hash
- `packages/api/src/infra/shield.test.ts` - Autorização
- `packages/api/src/infra/websocket.test.ts` - WebSocket

**Service Tests:**
- `packages/api/src/deals/service.test.ts` - DealService
- `packages/api/src/projects/service.test.ts` - ProjectService

**Event Listener Tests:**
- `packages/api/src/crm/cases/deal-is-mql.test.ts` - Listener de MQL
- `packages/api/src/deals/cases/won.test.ts` - Listener de Deal Won

**Integration Tests:**
- `packages/api/src/deals/search/service.elastic.integration.test.ts` - Elasticsearch
- `packages/api/src/projects/search/service.elastic.integration.test.ts` - Elasticsearch

**Complex Tests:**
- `packages/api/src/deals/cases/lead-score.test.ts` (440 linhas) - AI Integration
- `packages/api/src/infra/stripe/cases/invoice-paid.test.ts` (290 linhas) - Webhook
- `packages/api/src/auth/cases/token.test.ts` (274 linhas) - OAuth

### 15.2 Arquivos de Teste Modelo (Frontend)

**Unit Tests:**
- `packages/app/src/lib/contract-utils.test.ts` (88 linhas) - Utilities
- `packages/app/src/modules/commission-management/salary-system/calculator.test.ts` (441 linhas) - Cálculos complexos

### 15.3 Infraestrutura de Testes

**Mocks:**
- `packages/api/src/infra/test/infra-mocks.ts` - Infrastructure mocks
- `packages/api/src/infra/test/service-mocks.ts` - Service mocks
- `packages/api/src/infra/test/util-mocks.ts` - Utility mocks

**Configuração:**
- `packages/api/vite.config.ts` - Vitest config backend
- `packages/api/package.json` - Scripts de teste backend
- `packages/app/package.json` - Scripts de teste frontend

### 15.4 Documentação Externa

**Vitest:**
- [Vitest Documentation](https://vitest.dev/) - Docs oficiais
- [Vitest API Reference](https://vitest.dev/api/) - API completa
- [Vitest Mocking Guide](https://vitest.dev/guide/mocking.html) - Guia de mocks

**TypeScript:**
- [TypeScript Handbook](https://www.typescriptlang.org/docs/) - Docs oficiais
- [TypeScript Testing Guide](https://github.com/goldbergyoni/javascript-testing-best-practices) - Best practices

**Recursos Internos:**
- [CLAUDE.md](../CLAUDE.md) - Guia completo de desenvolvimento Berry
- [Git Workflow](../processes/git-workflow.md) - Fluxo de commits e branches
- [Code Review](../processes/code-review.md) - Processo de code review
- [TypeScript Guide](../technology-guides/typescript.md) - Padrões TypeScript na Berry

---

## Conclusão

Testes automatizados são essenciais para manter a qualidade e confiabilidade do código na Berry. Este guia fornece:

**Backend:**
- ✅ 125 test files, 90% coverage estabelecido
- ✅ Padrões maduros e bem documentados
- ✅ Mocks centralizados e reutilizáveis
- ✅ Exemplos reais do codebase

**Frontend:**
- ⚠️ < 0.2% coverage atual
- ✅ 2 arquivos modelo para começar
- ✅ Abordagem incremental (Fase 1: utils/services)
- 🚧 Fase 2 futura: React Testing Library

**Próximos passos:**
1. Manter 90% coverage no backend
2. Aumentar coverage frontend para 20% (foco em utils/services)
3. Adicionar React Testing Library (Fase 2)
4. Implementar E2E tests com Playwright (Fase 3)

**Lembre-se:**
- Testes são parte do Definition of Done
- Code review deve incluir verificação de testes
- Começar simples e incrementar gradualmente
- Usar os exemplos deste guia como referência

Com testes bem escritos, garantimos que o código da Berry continua funcionando conforme esperado, mesmo com constantes evoluções e refatorações.
