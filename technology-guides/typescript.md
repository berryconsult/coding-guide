# TypeScript - Padrões Berry

> Para documentação completa, consulte [typescriptlang.org](https://www.typescriptlang.org/docs/).

---

## Configuração

- **Node.js:** 24+
- **Strict mode:** Sempre ativo
- **ES Modules:** `"module": "esnext"`
- **Path aliases:** `@app/*` → `src/*`

---

## Regras Berry

### 1. Use `undefined` (não `null`)

```typescript
// ✅ CORRETO
interface User {
  email?: string // string | undefined
}

// ❌ INCORRETO
interface User {
  email: string | null
}
```

### 2. Use `??` (não `||`)

```typescript
// ✅ CORRETO
const name = user.name ?? 'Anonymous'

// ❌ INCORRETO - "" também retorna 'Anonymous'
const name = user.name || 'Anonymous'
```

### 3. Use utility functions

```typescript
import { isNotEmptyValue, isEmptyValue } from '@app/utils/validator'

// ✅ CORRETO
if (isNotEmptyValue(value)) { ... }

// ❌ INCORRETO
if (value !== undefined && value !== null) { ... }
```

### 4. Tipos de retorno explícitos

```typescript
// ✅ CORRETO
function getUser(id: string): Promise<User> {
  return userService().get(id)
}

// ❌ INCORRETO
function getUser(id: string) {
  return userService().get(id)
}
```

### 5. Proibido default exports

```typescript
// ✅ CORRETO
export class UserService extends Service<User> { }
export const userService = new UserService()

// ❌ INCORRETO
export default class UserService { }
```

### 6. Proibido `any`

```typescript
// ✅ CORRETO
function process(data: unknown): void {
  if (isUser(data)) {
    console.log(data.name)
  }
}

// ❌ INCORRETO
function process(data: any): void {
  console.log(data.name)
}
```

---

## Service Pattern

```typescript
// entities.ts
export interface Deal extends ArangoObject {
  _key: string
  workspace: string
  title: string
  status: DealStatus
}

export type DealStatus = 'open' | 'won' | 'lost'

// service.ts
export class DealService extends Service<Deal> {
  collection: CollectionName = 'deals'
  model = 'Deal'

  async findByStatus(status: DealStatus): Promise<Deal[]> {
    return this.find(aql`
      FOR d IN ${this.aql}
      FILTER d.status == ${status}
      RETURN d
    `)
  }
}

export const dealService = new DealService()
```

---

## Utility Types

```typescript
// Partial - todos opcionais
type UserUpdate = Partial<User>

// Pick - seleciona campos
type UserSummary = Pick<User, '_key' | 'name'>

// Omit - remove campos
type UserInput = Omit<User, '_key' | 'createdAt'>

// Record - mapeia chaves
type Permissions = Record<string, boolean>
```

---

## Type Guards

```typescript
function isUser(data: unknown): data is User {
  return (
    typeof data === 'object' &&
    data !== null &&
    '_key' in data &&
    'email' in data
  )
}

// Uso
if (isUser(data)) {
  console.log(data.email) // type-safe
}
```

---

## Discriminated Unions

```typescript
interface SuccessResult {
  success: true
  data: User
}

interface ErrorResult {
  success: false
  error: string
}

type Result = SuccessResult | ErrorResult

function handle(result: Result) {
  if (result.success) {
    console.log(result.data.name) // type narrowing
  } else {
    console.error(result.error)
  }
}
```

---

## Validação Runtime (Zod)

```typescript
import { z } from 'zod'

const CreateUserSchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
  role: z.enum(['admin', 'user']),
})

type CreateUserInput = z.infer<typeof CreateUserSchema>

// Controller
const input = CreateUserSchema.parse(args.input) // valida em runtime
```

---

## Checklist

- [ ] Strict mode ativo
- [ ] Tipos de retorno explícitos
- [ ] `undefined` ao invés de `null`
- [ ] `??` ao invés de `||`
- [ ] Sem default exports
- [ ] Sem `any` (usar `unknown`)
- [ ] Path aliases (`@app/*`)
