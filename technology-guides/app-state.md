# Legend App State - Padrões Berry

> Para documentação completa, consulte [legendapp.com/open-source/state](https://legendapp.com/open-source/state/).

Nunca use o gerenciamento de estado nativo do React.
Por motivos de perfomance foi determinado que nesse projeto, deve-se somente usar o gerenciamento de estado com o LegendApp State.

---

## ⚠️ REGRA FUNDAMENTAL: useState é PROIBIDO

```typescript
// ❌ PROIBIDO
const [name, setName] = useState('')

// ✅ CORRETO
const state$ = useObservable({ name: '' })
```

**Por quê?** Performance (fine-grained reactivity), consistência, sincronização automática.

> **Status:** 130+ arquivos legacy ainda usam `useState`. Novos componentes devem usar `useObservable`.
Se você encontrar `useState` Substitua por `useObservable` no seu contexto de trabalho da tarefa.
---

## Setup

```typescript
// Core
import { observable, beginBatch, endBatch, mergeIntoObservable } from '@legendapp/state'

// React
import { observer, useObservable, useComputed } from '@legendapp/state/react'
```

---

## Regras Essenciais

### 1. Sempre usar `observer()`

```typescript
// ✅ CORRETO
export const MyComponent = observer(() => {
  const state$ = useObservable({ count: 0 })
  return <div>{state$.count.get()}</div>
})

// ❌ INCORRETO - não re-renderiza
export const MyComponent = () => {
  const state$ = useObservable({ count: 0 })
  return <div>{state$.count.get()}</div>
}
```

### 2. Sufixo `$` em observables

```typescript
// ✅ CORRETO
const state$ = useObservable({ count: 0 })
const form$ = useObservable({ name: '' })

// ❌ INCORRETO
const state = useObservable({ count: 0 })
```

### 3. `.get()` vs `.peek()`

```typescript
// .get() → reativo (dentro de render)
return <div>{state$.count.get()}</div>

// .peek() → não-reativo (dentro de callbacks)
const handleSubmit = () => {
  const name = form$.name.peek() // ✅
  const name = form$.name.get()  // ❌ re-render desnecessário
}
```

---

## Estado Global

**Arquivo:** `packages/app/src/modules/state.ts`

```typescript
export const state$ = observable<AppState>({
  me: {} as User,
  isGod: false,
  layout: {
    darkMode: false,
    sidebarWidth: 72,
  },
  // ...
})

// Uso
import { state$ } from '@app/modules/state'

const currentUser = state$.me.get()
state$.layout.darkMode.set(true)
```

---

## Estado Local

```typescript
export const UserForm = observer(() => {
  const form$ = useObservable({
    name: '',
    email: '',
    loading: false,
  })

  const handleSubmit = async () => {
    form$.loading.set(true)
    await userService.create({
      name: form$.name.peek(),
      email: form$.email.peek(),
    })
    form$.loading.set(false)
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={form$.name.get()}
        onChange={e => form$.name.set(e.target.value)}
      />
      <button disabled={form$.loading.get()}>Salvar</button>
    </form>
  )
})
```

---

## Estados Modulares por Feature

```typescript
// packages/app/src/modules/whats/pages/chats.state.ts
export const whatsappState$ = observable({
  conversations: [] as WhatsConversation[],
  messages: [] as WappMessage[],
  activeChat: null as WhatsConversation | null,
  
  insertMessage: (message: WappMessage) => {
    whatsappState$.messages.push(message)
  },
})
```

---

## Batch Updates

```typescript
import { beginBatch, endBatch } from '@legendapp/state'

const handleUpdate = () => {
  beginBatch()
  state$.user.name.set('John')
  state$.user.email.set('john@example.com')
  state$.user.age.set(30)
  endBatch()
  // → 1 re-render ao invés de 3
}
```

---

## Merge Parcial

```typescript
import { mergeIntoObservable } from '@legendapp/state'

// Atualiza apenas campos especificados
mergeIntoObservable(user$, { name: 'Jane', age: 31 })
// email permanece inalterado
```

---

## Computed Values

```typescript
import { useComputed } from '@legendapp/state/react'

const completedCount = useComputed(() => {
  return tasks$.get().filter(t => t.status === 'completed').length
})
// Recalcula apenas quando tasks$ muda
```

---

## Sincronização com localStorage

```typescript
import { syncToLocalStorage } from '@app/modules/state'

syncToLocalStorage(
  state$.layout,
  ['darkMode', 'sidebarWidth'],
  'layout-prefs'
)
```

---

## TanStack Query + Legend State

```typescript
export const UsersPage = observer(() => {
  // UI state → Legend State
  const filter$ = useObservable({ search: '' })

  // Server state → TanStack Query
  const { data: users, isLoading } = useQuery({
    queryKey: ['users', filter$.search.get()],
    queryFn: () => userService.getUsers({ search: filter$.search.get() }),
  })

  return (
    <div>
      <input
        value={filter$.search.get()}
        onChange={e => filter$.search.set(e.target.value)}
      />
      {isLoading ? <Spinner /> : users?.map(u => <div key={u._key}>{u.name}</div>)}
    </div>
  )
})
```

---

## Anti-Patterns

| ❌ Evitar | ✅ Fazer |
|-----------|----------|
| `useState` | `useObservable` |
| Observable sem `$` | Sempre sufixo `$` |
| Componente sem `observer()` | Sempre `observer()` |
| `.get()` em callbacks | `.peek()` em callbacks |
| Múltiplos `.get()` em loop | Um `.get()` antes do loop |
| Tudo no `state$` global | Estados modulares por feature |

---

## Checklist

- [ ] Componente usa `observer()`
- [ ] Observables têm sufixo `$`
- [ ] `.peek()` em callbacks, `.get()` em render
- [ ] Estado local com `useObservable`
- [ ] Estado compartilhado em módulo separado
- [ ] Batch updates para múltiplas mudanças
