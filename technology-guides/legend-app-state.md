# Guia de Boas Práticas Legend App State - Berry/Maia Frontend

**Versão:** 1.0.0
**Última atualização:** 01 de Dezembro de 2025
**Objetivo:** Documentar e padronizar a forma como a Berry usa Legend App State no frontend

---

## Índice

1. [Introdução](#introdução)
2. [⚠️ REGRA FUNDAMENTAL: useState é PROIBIDO](#-regra-fundamental-usestate-é-proibido)
3. [Configuração e Setup](#configuração-e-setup)
4. [Padrões de Uso](#padrões-de-uso)
5. [Operações com Observables](#operações-com-observables)
6. [Sincronização e Persistência](#sincronização-e-persistência)
7. [Performance e Otimização](#performance-e-otimização)
8. [Padrões Avançados](#padrões-avançados)
9. [Best Practices - Checklist](#best-practices---checklist)
10. [Anti-Patterns - O Que Evitar](#anti-patterns---o-que-evitar)
11. [Exemplos Práticos Completos](#exemplos-práticos-completos)
12. [Troubleshooting](#troubleshooting)
13. [Referências](#referências)

---

## Introdução

Este documento **não** ensina Legend App State do zero. Ele documenta e padroniza **como a Berry/Maia Frontend já utiliza Legend App State** na prática, servindo como guia de referência para manter consistência no desenvolvimento.

### Visão Geral do Legend App State na Berry

O **Legend App State** (`@legendapp/state`) é a biblioteca de gerenciamento de estado padrão da Berry/Maia Frontend, utilizada para **todo** gerenciamento de estado reativo na aplicação. A escolha do Legend App State se deve à sua natureza **fine-grained reactive**, combinando performance, simplicidade e sincronização em um único sistema:

- **Fine-Grained Reactivity**: Re-renderiza apenas os componentes que realmente precisam atualizar
- **Performance Superior**: Evita re-renders desnecessários, resultando em aplicações mais rápidas
- **Sincronização**: Suporte nativo para sincronização com localStorage, backend e WebSocket
- **Simplicidade**: API simples com `get()`, `set()`, sem boilerplate complexo

### Por Que Legend App State?

A Berry precisa de:

1. **Performance**: Aplicação B2B complexa com muitos componentes, precisa de reatividade granular
2. **Sincronização Real-Time**: WebSocket para atualizações em tempo real (chat, mensagens, tarefas)
3. **Persistência**: Sincronização automática com localStorage para preferências do usuário
4. **Simplicidade**: Menos boilerplate que Redux ou Zustand, mais intuitivo que MobX

### Stack Tecnológico

- **Legend App State**: `@legendapp/state@3.0.0-beta.30` (state management)
- **React**: 19.1.0 com TypeScript
- **TanStack Query**: Data fetching (complementa Legend State)
- **Vite**: Build tool
- **TypeScript**: Type safety

### Documentação Oficial

Para conceitos básicos de Legend App State, consulte:

- **Documentação Oficial**: [https://legendapp.com/open-source/state/](https://legendapp.com/open-source/state/)
- **Getting Started**: [https://legendapp.com/open-source/state/v3/intro/introduction/](https://legendapp.com/open-source/state/v3/intro/introduction/)
- **React Integration**: [https://legendapp.com/open-source/state/react/](https://legendapp.com/open-source/state/react/)
- **Persistence**: [https://legendapp.com/open-source/state/persistence/](https://legendapp.com/open-source/state/persistence/)

---

## ⚠️ REGRA FUNDAMENTAL: useState é PROIBIDO

### 🚫 PROIBIÇÃO EXPLÍCITA

**`useState` do React está PROIBIDO no projeto Berry/Maia Frontend.**

Todos os novos componentes **DEVEM** usar `useObservable` do Legend App State. Componentes existentes que usam `useState` são considerados **legacy code** e devem ser migrados gradualmente.

### Por Que useState é Proibido?

1. **Performance**: `useState` causa re-render de todo o componente, mesmo quando apenas uma parte do estado muda
2. **Consistência**: Legend App State é o padrão da Berry, misturar com `useState` cria inconsistência
3. **Sincronização**: Legend State oferece sincronização automática (localStorage, WebSocket) que `useState` não oferece
4. **Fine-Grained Reactivity**: Legend State re-renderiza apenas o que mudou, `useState` re-renderiza tudo

### ❌ Código INCORRETO (useState)

```typescript
// ❌ PROIBIDO - NUNCA FAÇA ISSO
import { useState } from 'react'

export const MyComponent = () => {
  const [name, setName] = useState('')
  const [loading, setLoading] = useState(false)
  const [data, setData] = useState<User[]>([])

  const handleSubmit = async () => {
    setLoading(true)
    const result = await fetchData()
    setData(result)
    setLoading(false)
  }

  return (
    <div>
      <input value={name} onChange={e => setName(e.target.value)} />
      {loading && <Spinner />}
      {data.map(user => <div key={user._key}>{user.name}</div>)}
    </div>
  )
}
```

### ✅ Código CORRETO (useObservable)

```typescript
// ✅ CORRETO - Use sempre useObservable
import { observer, useObservable } from '@legendapp/state/react'

export const MyComponent = observer(() => {
  const state$ = useObservable({
    name: '',
    loading: false,
    data: [] as User[],
  })

  const handleSubmit = async () => {
    state$.loading.set(true)
    const result = await fetchData()
    state$.data.set(result)
    state$.loading.set(false)
  }

  return (
    <div>
      <input
        value={state$.name.get()}
        onChange={e => state$.name.set(e.target.value)}
      />
      {state$.loading.get() && <Spinner />}
      {state$.data.get().map(user => (
        <div key={user._key}>{user.name}</div>
      ))}
    </div>
  )
})
```

### Status da Migração

- **130+ arquivos** ainda usam `useState` (legacy code)
- **Padrão atual**: Todos os novos componentes devem usar `useObservable`
- **Migração**: Gradual, priorizar componentes críticos e novos desenvolvimentos

---

## Configuração e Setup

### Instalação

Legend App State já está instalado no projeto:

```json
{
  "dependencies": {
    "@legendapp/state": "3.0.0-beta.30",
    "@legendapp/state-react": "3.0.0-beta.30"
  }
}
```

### Importações Necessárias

```typescript
// Core functions
import {
  observable,
  Observable,
  beginBatch,
  endBatch,
  mergeIntoObservable,
} from '@legendapp/state'

// React hooks
import {
  observer,
  useObservable,
  useComputed,
  useObserveEffect,
  Show,
  For,
} from '@legendapp/state/react'
```

### Estado Global - `state$`

**Arquivo:** `packages/app/src/modules/state.ts`

A Berry mantém um estado global centralizado para dados compartilhados entre componentes:

```typescript
export const state$ = observable<AppState>({
  me: {} as User,
  isGod: false,
  permissions: new Set(),
  favorites: [],
  favoriteModelKeys: new Set(),
  floatDialogDeals: false,
  floatDialogDealsPinned: false,
  floatDialogDealsSize: { width: 640, height: 360 },
  floatDialogDealsOffset: { x: 20, y: 125 },
  layout: {
    isMobile: false,
    darkMode: false,
    sidebarWidth: 72,
    themePreference: 'system',
  },
  search: {
    dialogOpen: false,
  },
  task: {
    dialogOpen: false,
  },
  // ... outros campos
})
```

**Interface:**

```typescript
export interface AppState {
  me: User
  isGod: boolean
  permissions: Set<Permission>
  favorites: FavoriteMessage[]
  favoriteModelKeys: Set<string>
  floatDialogDeals: boolean
  floatDialogDealsPinned: boolean
  floatDialogDealsSize: { width: number; height: number }
  floatDialogDealsOffset: { x: number; y: number }
  layout: {
    isMobile: boolean
    darkMode: boolean
    sidebarWidth: number
    themePreference: 'system' | 'light' | 'dark'
  }
  search: {
    dialogOpen: boolean
  }
  task: {
    dialogOpen: false
  }
  // ... outros campos
}
```

**Uso:**

```typescript
import { state$ } from '@app/modules/state'

// Ler valor
const currentUser = state$.me.get()
const isDarkMode = state$.layout.darkMode.get()

// Atualizar valor
state$.layout.darkMode.set(true)
state$.search.dialogOpen.set(true)
```

### Estado Local - `useObservable`

Para estado específico de um componente, use `useObservable`:

```typescript
import { observer, useObservable } from '@legendapp/state/react'

export const MyComponent = observer(() => {
  const localState$ = useObservable({
    loading: false,
    data: null as User | null,
    filter: { search: '', active: true },
  })

  // Usar estado
  return (
    <div>
      {localState$.loading.get() && <Spinner />}
      <input
        value={localState$.filter.search.get()}
        onChange={e => localState$.filter.search.set(e.target.value)}
      />
    </div>
  )
})
```

---

## Padrões de Uso

### Componentes com `observer`

**TODOS os componentes que usam observables DEVEM ser envolvidos com `observer()`:**

```typescript
// ✅ CORRETO
import { observer, useObservable } from '@legendapp/state/react'

export const MyComponent = observer(() => {
  const state$ = useObservable({ count: 0 })
  return <div>{state$.count.get()}</div>
})
```

**Por que `observer()`?**

- `observer()` faz o componente re-renderizar automaticamente quando observables mudam
- Sem `observer()`, mudanças em observables não causam re-render
- É equivalente ao `observer` do MobX, mas para Legend State

### Estado Local com `useObservable`

**Padrão:** Use `useObservable` para estado local do componente:

```typescript
export const UserForm = observer(() => {
  const form$ = useObservable({
    name: '',
    email: '',
    loading: false,
    errors: {} as Record<string, string>,
  })

  const handleSubmit = async () => {
    form$.loading.set(true)
    try {
      await userService.create({
        name: form$.name.get(),
        email: form$.email.get(),
      })
    } catch (error) {
      form$.errors.set({ submit: 'Erro ao criar usuário' })
    } finally {
      form$.loading.set(false)
    }
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

### Estado Global com `state$`

**Padrão:** Use `state$` para dados compartilhados entre múltiplos componentes:

```typescript
import { state$ } from '@app/modules/state'

export const UserProfile = observer(() => {
  const currentUser = state$.me.get()

  return (
    <div>
      <h1>{currentUser.name}</h1>
      <p>{currentUser.email}</p>
    </div>
  )
})

export const UserSettings = observer(() => {
  const handleToggleDarkMode = () => {
    state$.layout.darkMode.set(!state$.layout.darkMode.get())
  }

  return (
    <button onClick={handleToggleDarkMode}>
      {state$.layout.darkMode.get() ? 'Light' : 'Dark'} Mode
    </button>
  )
})
```

### Naming Conventions - Sufixo `$`

**Regra:** Todos os observables devem ter sufixo `$`:

```typescript
// ✅ CORRETO
const state$ = useObservable({ count: 0 })
const form$ = useObservable({ name: '' })
const user$ = observable({ name: 'John' })

// ❌ INCORRETO
const state = useObservable({ count: 0 }) // Sem sufixo $
const form = useObservable({ name: '' }) // Sem sufixo $
```

**Por que sufixo `$`?**

- Convenção do Legend App State e MobX
- Facilita identificação de observables no código
- Diferencia observables de valores normais
- Melhora legibilidade

---

## Operações com Observables

### Leitura: `.get()`, `.peek()`, `use()`

#### `.get()` - Leitura Reativa

**Use `.get()` quando precisar que o componente re-renderize quando o valor mudar:**

```typescript
export const MyComponent = observer(() => {
  const count$ = useObservable(0)

  // ✅ Re-renderiza quando count$ muda
  return <div>{count$.get()}</div>
})
```

**Quando usar:**
- Dentro de `observer()` components
- Quando precisa de reatividade
- Para valores que mudam e devem atualizar a UI

#### `.peek()` - Leitura Não-Reativa

**Use `.peek()` quando NÃO precisa de reatividade (dentro de funções, callbacks):**

```typescript
export const MyComponent = observer(() => {
  const form$ = useObservable({ name: '', email: '' })

  const handleSubmit = async () => {
    // ✅ peek() não causa re-render, apenas lê o valor atual
    const name = form$.name.peek()
    const email = form$.email.peek()

    await api.submit({ name, email })
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={form$.name.get()}
        onChange={e => form$.name.set(e.target.value)}
      />
    </form>
  )
})
```

**Quando usar:**
- Dentro de callbacks (onClick, onSubmit, etc.)
- Em funções assíncronas
- Quando precisa ler valor sem causar re-render
- Em loops ou operações que não devem ser reativas

#### `use()` - Hook Reativo (Alternativa a `.get()`)

**`use()` é um hook que funciona como `.get()` mas é mais idiomático em alguns casos:**

```typescript
import { use } from '@legendapp/state/react'

export const MyComponent = observer(() => {
  const count$ = useObservable(0)

  // ✅ use() é equivalente a count$.get()
  const count = use(count$)

  return <div>{count}</div>
})
```

**Quando usar:**
- Preferência pessoal (equivalente a `.get()`)
- Quando quer código mais limpo sem `.get()` em todo lugar

### Escrita: `.set()`, `.push()`, `.delete()`

#### `.set()` - Atualizar Valor

```typescript
const state$ = useObservable({ count: 0, name: '' })

// Atualizar valor simples
state$.count.set(10)

// Atualizar objeto
state$.set({ count: 10, name: 'John' })

// Atualizar propriedade aninhada
state$.user.name.set('John')
```

#### `.push()` - Adicionar a Array

```typescript
const tasks$ = useObservable<Task[]>([])

// Adicionar item
tasks$.push(newTask)

// Adicionar múltiplos itens
tasks$.push(...newTasks)
```

**Exemplo real do codebase:**

```typescript
// packages/app/src/modules/work/tasks.tsx
if (append) {
  tasks$.push(...(tasks as Task[]))
} else {
  tasks$.set(tasks as Task[])
}
```

#### `.delete()` - Remover de Array ou Objeto

```typescript
const items$ = useObservable(['a', 'b', 'c'])

// Remover por índice
items$.delete(0) // Remove 'a'

// Remover propriedade de objeto
const user$ = useObservable({ name: 'John', email: 'john@example.com' })
user$.email.delete() // Remove email
```

### Batch Updates - `beginBatch()` / `endBatch()`

**Use batch updates quando precisar fazer múltiplas mudanças sem causar múltiplos re-renders:**

```typescript
import { beginBatch, endBatch } from '@legendapp/state'

const handleUpdate = () => {
  beginBatch()
  state$.user.name.set('John')
  state$.user.email.set('john@example.com')
  state$.user.age.set(30)
  endBatch()
  // ✅ Apenas 1 re-render ao invés de 3
}
```

**Exemplo real do codebase:**

```typescript
// packages/app/src/modules/state.ts
export function maiaMergeIntoObservable<T>(target: T, ...sources: unknown[]): T {
  beginBatch()
  for (let i = 0; i < sources.length; i++) {
    target = _mergeIntoObservable(target, sources[i], i < sources.length - 1)
  }
  endBatch()
  return target
}
```

**Quando usar:**
- Múltiplas atualizações relacionadas
- Operações complexas que atualizam vários campos
- Quando quer evitar re-renders intermediários

### Merge - `mergeIntoObservable()`

**Use `mergeIntoObservable()` para fazer updates parciais em objetos:**

```typescript
import { mergeIntoObservable } from '@legendapp/state'

const user$ = useObservable({
  name: 'John',
  email: 'john@example.com',
  age: 30,
})

// ✅ Merge parcial (mantém campos não especificados)
mergeIntoObservable(user$, {
  name: 'Jane',
  age: 31,
})
// Resultado: { name: 'Jane', email: 'john@example.com', age: 31 }
```

**Exemplo real do codebase:**

```typescript
// packages/app/src/modules/crm/inbound/pages/state.tsx
fetchDeal: async (key: string) => {
  const deal = await dealService.deal(key)
  mergeIntoObservable(crmPageState$.deal, deal)
},
```

**Quando usar:**
- Updates parciais de objetos
- Quando recebe dados do backend e quer mesclar
- Evitar sobrescrever campos não atualizados

---

## Sincronização e Persistência

### `syncToLocalStorage()` Helper

A Berry fornece um helper customizado para sincronizar observables com localStorage:

**Arquivo:** `packages/app/src/modules/state.ts`

```typescript
export function syncToLocalStorage(
  observable$: Observable,
  properties: string[],
  key: string,
) {
  const value = localStorage.getItem(key)
  if (value) {
    const parsedValue = JSON.parse(value)
    properties.forEach(property => {
      if (Object.prototype.hasOwnProperty.call(parsedValue, property)) {
        observable$[property].set(parsedValue[property])
      }
    })
  }

  observable$.onChange(() => {
    const value = properties.reduce(
      (acc, property) => {
        ;(acc as Record<string, unknown>)[property] =
          observable$[property].get()
        return acc
      },
      {} as Record<string, unknown>,
    )
    localStorage.setItem(key, JSON.stringify(value))
  })
}
```

**Uso:**

```typescript
import { state$ } from '@app/modules/state'
import { syncToLocalStorage } from '@app/modules/state'

// Sincronizar preferências de layout
syncToLocalStorage(
  state$.layout,
  ['darkMode', 'sidebarWidth', 'themePreference'],
  'layout-prefs',
)
```

**Como funciona:**
1. Carrega valores do localStorage na inicialização
2. Salva automaticamente quando valores mudam
3. Sincroniza apenas propriedades especificadas

### Sincronização com Backend

**Padrão:** Use TanStack Query para data fetching, Legend State para UI state:

```typescript
import { useQuery } from '@tanstack/react-query'
import { observer, useObservable } from '@legendapp/state/react'

export const UsersList = observer(() => {
  const filter$ = useObservable({ search: '', active: true })

  // TanStack Query para dados do servidor
  const { data: users, isLoading } = useQuery({
    queryKey: ['users', filter$.get()],
    queryFn: () => userService.getUsers(filter$.get()),
  })

  // Legend State para estado da UI
  const uiState$ = useObservable({
    selectedUser: null as User | null,
    dialogOpen: false,
  })

  return (
    <div>
      <input
        value={filter$.search.get()}
        onChange={e => filter$.search.set(e.target.value)}
      />
      {isLoading && <Spinner />}
      {users?.map(user => (
        <div key={user._key}>{user.name}</div>
      ))}
    </div>
  )
})
```

### WebSocket Integration

**Exemplo real:** Integração WebSocket para atualizações em tempo real:

**Arquivo:** `packages/app/src/modules/whats/pages/chats.state.ts`

```typescript
export const whatsappState$ = observable({
  conversations: [] as WhatsConversation[],
  messages: [] as WappMessage[],
  activeChat: null as WhatsConversation | null,
  insertMessage: (message: WappMessage) => {
    whatsappState$.messages.push(message)
  },
  updateWhatsMessage: (message: WappMessage) => {
    const messageIndex = whatsappState$.messages.findIndex(
      m => m._key.peek() === message._key,
    )
    if (messageIndex > -1) {
      mergeIntoObservable(whatsappState$.messages[messageIndex], message)
    }
  },
})

export function setupWhatsappWebsocket() {
  const ws = MaiaWebSocket.getInstance()
  whatsappState$.websocket.set(ObservableHint.opaque(ws))

  ws.on('message', rawData => {
    const data = rawData as unknown
    if (!MessageParser.isValidServerMessage(data)) return

    if (data.type === 'insert') {
      whatsappState$.insertMessage(payload as WappMessage)
    } else if (data.type === 'patch') {
      whatsappState$.updateWhatsMessage(payload as WappMessage)
    }
  })

  ws.connect()
}
```

**Padrão:**
1. Estado modular para feature específica (`whatsappState$`)
2. Métodos helper no observable (`insertMessage`, `updateWhatsMessage`)
3. WebSocket atualiza estado diretamente
4. Componentes re-renderizam automaticamente via `observer()`

---

## Performance e Otimização

### Fine-Grained Reactivity

**Legend State re-renderiza apenas o que mudou:**

```typescript
export const UserProfile = observer(() => {
  // ✅ Apenas re-renderiza quando state$.me.name muda
  return <div>{state$.me.name.get()}</div>
})

export const UserEmail = observer(() => {
  // ✅ Apenas re-renderiza quando state$.me.email muda
  return <div>{state$.me.email.get()}</div>
})
```

**Benefício:** Se `state$.me.name` muda, apenas `UserProfile` re-renderiza, `UserEmail` não.

### Evitar `.get()` Desnecessários

**❌ INCORRETO - Múltiplos `.get()` em loops:**

```typescript
export const TaskList = observer(() => {
  const tasks$ = useObservable<Task[]>([])

  // ❌ Cada .get() causa re-render
  return (
    <div>
      {tasks$.get().map(task => (
        <div key={task._key}>
          {task.name} - {task.status}
        </div>
      ))}
    </div>
  )
})
```

**✅ CORRETO - Um único `.get()`:**

```typescript
export const TaskList = observer(() => {
  const tasks$ = useObservable<Task[]>([])

  // ✅ Um único .get(), depois usa array normal
  const tasks = tasks$.get()

  return (
    <div>
      {tasks.map(task => (
        <div key={task._key}>
          {task.name} - {task.status}
        </div>
      ))}
    </div>
  )
})
```

### Computed Values

**Use `useComputed()` para valores derivados:**

```typescript
import { useComputed } from '@legendapp/state/react'

export const TaskStats = observer(() => {
  const tasks$ = useObservable<Task[]>([])

  // ✅ Computed re-renderiza apenas quando tasks$ muda
  const completedCount = useComputed(() => {
    return tasks$.get().filter(t => t.status === 'completed').length
  })

  return <div>Completed: {completedCount}</div>
})
```

**Benefício:** `completedCount` só recalcula quando `tasks$` muda, não em cada re-render.

### Memoização

**Para valores computados complexos, use `useMemo` do React:**

```typescript
import { useMemo } from 'react'

export const ExpensiveComponent = observer(() => {
  const data$ = useObservable<Data[]>([])

  // ✅ Memoiza resultado de cálculo pesado
  const processedData = useMemo(() => {
    return data$.get().map(expensiveTransformation)
  }, [data$.get()])

  return <div>{/* usar processedData */}</div>
})
```

---

## Padrões Avançados

### Estados Modulares

**Padrão:** Criar estados modulares por feature ao invés de tudo no `state$` global:

**Exemplo:** `whatsappState$`, `taskState$`, `crmPageState$`

**Arquivo:** `packages/app/src/modules/whats/pages/chats.state.ts`

```typescript
export const whatsappState$ = observable({
  conversations: [] as WhatsConversation[],
  chatsSearchTerm: '',
  attachments: [] as Asset[],
  activeChat: null as WhatsConversation | null,
  accounts: [] as WhatsappAccount[],
  activeAccount: null as WhatsappAccount | null,
  messages: [] as WappMessage[],
  message: { content: '', attachments: [] } as Partial<ChatMessage>,
  isRecording: false,
  emojiPickerOpen: false,
  addConversationDialogOpen: false,
  websocket: {} as MaiaWebSocket,
  // Métodos helper
  insertMessage: (message: WappMessage) => {
    whatsappState$.messages.push(message)
  },
  updateWhatsMessage: (message: WappMessage) => {
    const messageIndex = whatsappState$.messages.findIndex(
      m => m._key.peek() === message._key,
    )
    if (messageIndex > -1) {
      mergeIntoObservable(whatsappState$.messages[messageIndex], message)
    }
  },
})
```

**Benefícios:**
- Organização por feature
- Estado isolado e reutilizável
- Fácil de testar e manter

### Helpers Customizados

**A Berry criou helpers customizados para operações comuns:**

**Arquivo:** `packages/app/src/modules/state.ts`

```typescript
export function maiaMergeIntoObservable<
  T extends ObservableParam<Record<string, unknown>> | object,
>(target: T, ...sources: unknown[]): T {
  beginBatch()
  for (let i = 0; i < sources.length; i++) {
    target = _mergeIntoObservable(target, sources[i], i < sources.length - 1)
  }
  endBatch()
  return target
}
```

**Uso:**

```typescript
import { maiaMergeIntoObservable } from '@app/modules/state'

// Merge múltiplos objetos com batch update
maiaMergeIntoObservable(user$, update1, update2, update3)
```

### Integração TanStack Query + Legend State

**Padrão:** TanStack Query para server state, Legend State para UI state:

```typescript
import { useQuery } from '@tanstack/react-query'
import { observer, useObservable } from '@legendapp/state/react'

export const UsersPage = observer(() => {
  // UI state com Legend State
  const uiState$ = useObservable({
    search: '',
    selectedUser: null as User | null,
    dialogOpen: false,
  })

  // Server state com TanStack Query
  const { data: users, isLoading } = useQuery({
    queryKey: ['users', uiState$.search.get()],
    queryFn: () => userService.getUsers({ search: uiState$.search.get() }),
  })

  return (
    <div>
      <input
        value={uiState$.search.get()}
        onChange={e => uiState$.search.set(e.target.value)}
      />
      {isLoading && <Spinner />}
      {users?.map(user => (
        <div key={user._key}>{user.name}</div>
      ))}
    </div>
  )
})
```

### WebSocket Real-Time Updates

**Padrão completo:** WebSocket atualiza estado modular, componentes re-renderizam automaticamente:

```typescript
// 1. Estado modular
export const chatState$ = observable({
  messages: [] as Message[],
  insertMessage: (message: Message) => {
    chatState$.messages.push(message)
  },
})

// 2. Setup WebSocket
export function setupChatWebsocket() {
  const ws = WebSocket.getInstance()
  ws.on('message', data => {
    if (data.type === 'new_message') {
      chatState$.insertMessage(data.message)
    }
  })
  ws.connect()
}

// 3. Componente reativo
export const ChatMessages = observer(() => {
  // ✅ Re-renderiza automaticamente quando messages mudam
  return (
    <div>
      {chatState$.messages.get().map(msg => (
        <div key={msg._key}>{msg.content}</div>
      ))}
    </div>
  )
})
```

---

## Best Practices - Checklist

### ✅ Queries

- [ ] Sempre usar `observer()` em componentes que usam observables
- [ ] Sempre usar sufixo `$` em observables
- [ ] Usar `useObservable` para estado local
- [ ] Usar `state$` para estado global compartilhado
- [ ] Usar `.get()` para leitura reativa (dentro de `observer()`)
- [ ] Usar `.peek()` para leitura não-reativa (callbacks, async)
- [ ] Usar `beginBatch()` / `endBatch()` para múltiplas atualizações
- [ ] Usar `mergeIntoObservable()` para updates parciais

### ✅ Estado

- [ ] Criar estados modulares por feature (ex: `whatsappState$`, `taskState$`)
- [ ] Usar `syncToLocalStorage()` para persistência de preferências
- [ ] Separar server state (TanStack Query) de UI state (Legend State)
- [ ] Usar `useComputed()` para valores derivados

### ✅ Performance

- [ ] Evitar múltiplos `.get()` em loops
- [ ] Usar batch updates para múltiplas mudanças
- [ ] Usar `useComputed()` ao invés de calcular em cada render
- [ ] Memoizar cálculos pesados com `useMemo`

### ✅ Naming

- [ ] Sempre usar sufixo `$` em observables
- [ ] Nomes descritivos: `form$`, `userState$`, `taskList$`
- [ ] Estados modulares: `{feature}State$` (ex: `whatsappState$`)

### ✅ Componentes

- [ ] Todos os componentes com observables devem usar `observer()`
- [ ] Usar `Show` e `For` do Legend State quando apropriado
- [ ] Evitar prop drilling, usar estado global ou modular

---

## Anti-Patterns - O Que Evitar

### ❌ useState

- **NUNCA use `useState` do React:**
  ```typescript
  // ❌ INCORRETO
  const [count, setCount] = useState(0)

  // ✅ CORRETO
  const count$ = useObservable(0)
  ```

### ❌ Observable sem sufixo `$`

- **NUNCA crie observables sem sufixo `$`:**
  ```typescript
  // ❌ INCORRETO
  const state = useObservable({ count: 0 })

  // ✅ CORRETO
  const state$ = useObservable({ count: 0 })
  ```

### ❌ Componente sem `observer()`

- **NUNCA use observables sem `observer()`:**
  ```typescript
  // ❌ INCORRETO - Não re-renderiza quando observable muda
  export const MyComponent = () => {
    const count$ = useObservable(0)
    return <div>{count$.get()}</div>
  }

  // ✅ CORRETO
  export const MyComponent = observer(() => {
    const count$ = useObservable(0)
    return <div>{count$.get()}</div>
  })
  ```

### ❌ `.get()` em loops

- **Evite múltiplos `.get()` em loops:**
  ```typescript
  // ❌ INCORRETO
  {items$.get().map(item => (
    <div>{item.name.get()} - {item.status.get()}</div>
  ))}

  // ✅ CORRETO
  {const items = items$.get()
  items.map(item => (
    <div>{item.name} - {item.status}</div>
  ))}
  ```

### ❌ `.get()` em callbacks

- **Use `.peek()` em callbacks ao invés de `.get()`:**
  ```typescript
  // ❌ INCORRETO
  const handleSubmit = async () => {
    const name = form$.name.get() // Causa re-render desnecessário
    await api.submit({ name })
  }

  // ✅ CORRETO
  const handleSubmit = async () => {
    const name = form$.name.peek() // Não causa re-render
    await api.submit({ name })
  }
  ```

### ❌ Estado global para tudo

- **Não coloque tudo no `state$` global:**
  ```typescript
  // ❌ INCORRETO - Estado específico de feature no global
  state$.whatsappMessages.set([])

  // ✅ CORRETO - Estado modular
  whatsappState$.messages.set([])
  ```

---

## Exemplos Práticos Completos

### Exemplo 1: Componente Simples com Estado Local

**Formulário de criação de usuário:**

```typescript
import { observer, useObservable } from '@legendapp/state/react'
import { userService } from '@app/modules/user/service'
import { toast } from 'sonner'

export const CreateUserForm = observer(() => {
  const form$ = useObservable({
    name: '',
    email: '',
    loading: false,
    errors: {} as Record<string, string>,
  })

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()

    // Validação
    if (!form$.name.peek()) {
      form$.errors.set({ name: 'Nome é obrigatório' })
      return
    }

    if (!form$.email.peek()) {
      form$.errors.set({ email: 'Email é obrigatório' })
      return
    }

    form$.loading.set(true)
    form$.errors.set({})

    try {
      await userService.create({
        name: form$.name.peek(),
        email: form$.email.peek(),
      })
      toast.success('Usuário criado com sucesso')
      // Reset form
      form$.name.set('')
      form$.email.set('')
    } catch (error) {
      form$.errors.set({ submit: 'Erro ao criar usuário' })
      toast.error('Erro ao criar usuário')
    } finally {
      form$.loading.set(false)
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <div>
        <input
          value={form$.name.get()}
          onChange={e => {
            form$.name.set(e.target.value)
            form$.errors.name?.delete()
          }}
          placeholder="Nome"
        />
        {form$.errors.name?.get() && (
          <span className="text-red-500">{form$.errors.name.get()}</span>
        )}
      </div>

      <div>
        <input
          type="email"
          value={form$.email.get()}
          onChange={e => {
            form$.email.set(e.target.value)
            form$.errors.email?.delete()
          }}
          placeholder="Email"
        />
        {form$.errors.email?.get() && (
          <span className="text-red-500">{form$.errors.email.get()}</span>
        )}
      </div>

      {form$.errors.submit?.get() && (
        <div className="text-red-500">{form$.errors.submit.get()}</div>
      )}

      <button type="submit" disabled={form$.loading.get()}>
        {form$.loading.get() ? 'Salvando...' : 'Criar Usuário'}
      </button>
    </form>
  )
})
```

### Exemplo 2: Componente com Estado Global

**Componente que usa estado global:**

```typescript
import { observer } from '@legendapp/state/react'
import { state$ } from '@app/modules/state'

export const UserProfile = observer(() => {
  const currentUser = state$.me.get()

  if (!currentUser._key) {
    return <div>Carregando...</div>
  }

  return (
    <div>
      <h1>{currentUser.name}</h1>
      <p>{currentUser.email}</p>
      <p>Workspace: {currentUser.workspace?.name}</p>
    </div>
  )
})

export const DarkModeToggle = observer(() => {
  const isDarkMode = state$.layout.darkMode.get()

  const handleToggle = () => {
    state$.layout.darkMode.set(!isDarkMode)
  }

  return (
    <button onClick={handleToggle}>
      {isDarkMode ? '☀️ Light Mode' : '🌙 Dark Mode'}
    </button>
  )
})
```

### Exemplo 3: Estado Modular Completo (WhatsApp)

**Arquivo:** `packages/app/src/modules/whats/pages/chats.state.ts`

```typescript
import { observable, mergeIntoObservable } from '@legendapp/state'
import { MaiaWebSocket, MessageParser } from '@app/modules/infra/websocket'
import { WappMessage, WhatsConversation } from '@app/modules/whats/entity.whats'
import { ChatMessage } from '@app/modules/message/entity.message'
import { Asset } from '@app/modules/asset/entities'
import { WhatsappAccount } from '@app/modules/model/entity.model'

export const whatsappState$ = observable({
  conversations: [] as WhatsConversation[],
  chatsSearchTerm: '',
  attachments: [] as Asset[],
  activeChat: null as WhatsConversation | null,
  accounts: [] as WhatsappAccount[],
  activeAccount: null as WhatsappAccount | null,
  messages: [] as WappMessage[],
  message: { content: '', attachments: [] } as Partial<ChatMessage>,
  isRecording: false,
  emojiPickerOpen: false,
  addConversationDialogOpen: false,
  websocket: {} as MaiaWebSocket,

  // Métodos helper
  insertMessage: (message: WappMessage) => {
    whatsappState$.messages.push(message)
  },

  updateWhatsMessage: (message: WappMessage) => {
    const messageIndex = whatsappState$.messages.findIndex(
      m => m._key.peek() === message._key,
    )
    if (messageIndex > -1) {
      mergeIntoObservable(whatsappState$.messages[messageIndex], message)
    }
  },

  updateChat: (chat: ConversationMessage) => {
    const chatIndex = whatsappState$.conversations.findIndex(
      c => c._key.peek() === chat._key,
    )
    if (chatIndex > -1) {
      mergeIntoObservable(whatsappState$.conversations[chatIndex], chat)
    }
  },
})

export function setupWhatsappWebsocket() {
  const ws = MaiaWebSocket.getInstance()
  whatsappState$.websocket.set(ObservableHint.opaque(ws))

  ws.on('message', rawData => {
    const data = rawData as unknown
    if (!MessageParser.isValidServerMessage(data)) return

    if (typeof data.channel === 'string' && data.channel.includes('chats')) {
      const payload = data.data as Partial<ConversationMessage>
      if (payload && typeof payload === 'object' && '_key' in payload) {
        whatsappState$.updateChat(payload as ConversationMessage)
      }
      return
    }

    if (
      typeof data.channel === 'string' &&
      data.channel.includes('messages') &&
      typeof data.type === 'string'
    ) {
      const payload = data.data as Partial<WappMessage>
      if (!payload || typeof payload !== 'object' || !('_key' in payload))
        return

      if (data.type === 'insert') {
        whatsappState$.insertMessage(payload as WappMessage)
      } else if (data.type === 'patch') {
        whatsappState$.updateWhatsMessage(payload as WappMessage)
      }
    }
  })

  ws.connect()
}
```

**Uso em componente:**

```typescript
import { observer } from '@legendapp/state/react'
import { whatsappState$ } from '@app/modules/whats/pages/chats.state'

export const ChatMessages = observer(() => {
  const messages = whatsappState$.messages.get()
  const activeChat = whatsappState$.activeChat.get()

  if (!activeChat) {
    return <div>Selecione uma conversa</div>
  }

  return (
    <div>
      {messages.map(message => (
        <div key={message._key}>{message.content}</div>
      ))}
    </div>
  )
})
```

### Exemplo 4: Integração com TanStack Query

**Componente que combina TanStack Query (server state) e Legend State (UI state):**

```typescript
import { observer, useObservable } from '@legendapp/state/react'
import { useQuery } from '@tanstack/react-query'
import { userService } from '@app/modules/user/service'
import { state$ } from '@app/modules/state'

export const UsersList = observer(() => {
  // UI state com Legend State
  const uiState$ = useObservable({
    search: '',
    selectedUser: null as User | null,
    dialogOpen: false,
    page: 0,
  })

  // Server state com TanStack Query
  const { data: users, isLoading, refetch } = useQuery({
    queryKey: ['users', uiState$.search.get(), uiState$.page.get()],
    queryFn: () =>
      userService.getUsers({
        search: uiState$.search.get(),
        workspace: state$.me.workspace._key.get(),
        limit: 20,
        offset: uiState$.page.get() * 20,
      }),
  })

  const handleSelectUser = (user: User) => {
    uiState$.selectedUser.set(user)
    uiState$.dialogOpen.set(true)
  }

  return (
    <div>
      <input
        value={uiState$.search.get()}
        onChange={e => {
          uiState$.search.set(e.target.value)
          uiState$.page.set(0) // Reset page on search
        }}
        placeholder="Buscar usuários..."
      />

      {isLoading && <Spinner />}

      {users?.map(user => (
        <div
          key={user._key}
          onClick={() => handleSelectUser(user)}
        >
          {user.name}
        </div>
      ))}

      {uiState$.dialogOpen.get() && uiState$.selectedUser.get() && (
        <UserDialog
          user={uiState$.selectedUser.get()}
          onClose={() => uiState$.dialogOpen.set(false)}
        />
      )}
    </div>
  )
})
```

---

## Troubleshooting

### Componente Não Re-Renderiza

**Problema:** Componente não atualiza quando observable muda.

**Solução:** Verificar se componente está envolvido com `observer()`:

```typescript
// ❌ INCORRETO - Não re-renderiza
export const MyComponent = () => {
  const count$ = useObservable(0)
  return <div>{count$.get()}</div>
}

// ✅ CORRETO - Re-renderiza
export const MyComponent = observer(() => {
  const count$ = useObservable(0)
  return <div>{count$.get()}</div>
})
```

### Estado Não Atualiza

**Problema:** Mudanças em observable não refletem na UI.

**Solução:** Verificar se está usando `.set()` corretamente:

```typescript
// ❌ INCORRETO - Mutação direta não funciona
state$.user.name = 'John'

// ✅ CORRETO - Use .set()
state$.user.name.set('John')
```

### Performance Issues

**Problema:** Componente re-renderiza muito frequentemente.

**Soluções:**

1. **Evitar múltiplos `.get()` em loops:**
   ```typescript
   // ❌ INCORRETO
   {items$.get().map(item => item.name.get())}

   // ✅ CORRETO
   {const items = items$.get()
   items.map(item => item.name)}
   ```

2. **Usar batch updates:**
   ```typescript
   beginBatch()
   state$.user.name.set('John')
   state$.user.email.set('john@example.com')
   endBatch()
   ```

3. **Usar `useComputed()` para valores derivados:**
   ```typescript
   const filteredItems = useComputed(() => {
     return items$.get().filter(item => item.active)
   })
   ```

### Debug Tips

**Acessar estado global no console (dev mode):**

```typescript
// packages/app/src/modules/state.ts
const isDev = import.meta.env.MODE === 'development'
if (isDev) {
  global.state$ = state$
}
```

**No console do browser:**
```javascript
// Acessar estado global
global.state$.me.get()

// Observar mudanças
global.state$.me.onChange((value) => {
  console.log('User changed:', value)
})
```

---

## Referências

### Documentação Oficial

- **Documentação Oficial**: [https://legendapp.com/open-source/state/](https://legendapp.com/open-source/state/)
- **Getting Started**: [https://legendapp.com/open-source/state/v3/intro/introduction/](https://legendapp.com/open-source/state/v3/intro/introduction/)
- **React Integration**: [https://legendapp.com/open-source/state/react/](https://legendapp.com/open-source/state/react/)
- **Persistence**: [https://legendapp.com/open-source/state/persistence/](https://legendapp.com/open-source/state/persistence/)
- **Performance**: [https://legendapp.com/open-source/state/v3/intro/why/](https://legendapp.com/open-source/state/v3/intro/why/)

### Documentos Relacionados

- [CLAUDE.md](../../CLAUDE.md) - Handbook completo da Maia Frontend
- [TypeScript Guide](./typescript.md) - Guia de TypeScript da Berry
- [ArangoDB Guide](./arangodb.md) - Guia de ArangoDB (backend)

### Arquivos de Referência no Codebase

#### Estado Global
- `packages/app/src/modules/state.ts` - Estado global `state$` e helpers

#### Estados Modulares
- `packages/app/src/modules/whats/pages/chats.state.ts` - Estado WhatsApp
- `packages/app/src/modules/task/state.ts` - Estado de tarefas
- `packages/app/src/modules/crm/inbound/pages/state.tsx` - Estado CRM
- `packages/app/src/modules/work/tasks.tsx` - Exemplo de componente com estado

#### Helpers Customizados
- `packages/app/src/modules/state.ts` - `syncToLocalStorage()`, `maiaMergeIntoObservable()`

#### Exemplos Práticos
- `packages/app/src/modules/work/tasks.tsx` - Componente completo com paginação
- `packages/app/src/modules/whats/pages/chats.tsx` - Integração WebSocket
- `packages/app/src/modules/crm/inbound/pages/inbound.tsx` - TanStack Query + Legend State

---

**Fim do Documento**

Este guia documenta e padroniza a forma como a Berry/Maia Frontend utiliza Legend App State. Para dúvidas ou sugestões, consulte o Tech Lead ou abra uma issue no repositório.

**Lembre-se: `useState` é PROIBIDO. Use sempre `useObservable`!**
