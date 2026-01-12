# ArangoDB - Padrões Berry

> Para conceitos básicos de ArangoDB e AQL, consulte a [documentação oficial](https://docs.arangodb.com/).

---

## Service Pattern

Todos os services estendem `Service<T>`:

```typescript
export class DealService extends Service<Deal> {
  collection: CollectionName = 'deals'
  model = 'deal'
}

export const dealService = DealService.getInstance()
```

**Métodos herdados:**
- `get(key)` - busca com cache Redis
- `find(query)` / `findOne(query)` - queries AQL com cache
- `create(payload)` - auto-timestamps, auto-ID, event emission
- `update(key, payload)` - auto-timestamps, event emission, cache invalidation
- `archive(key, user)` - soft delete

**Auto-generated IDs:** `{collection}_{nanoId()}` (ex: `deals_abc123xyz`)

---

## Queries AQL

### Sempre usar `aql` tag

```typescript
import { aql, join } from 'arangojs/aql.js'

// ✅ CORRETO
const q = aql`FOR doc IN ${this.aql} FILTER doc._key == ${key} RETURN doc`

// ❌ NUNCA - SQL injection
const q = `FOR doc IN ${collection} FILTER doc._key == "${key}" RETURN doc`
```

### Filtros dinâmicos

```typescript
buildFilters(filter: Partial<DealFilter>): GeneratedAqlQuery[] {
  const filters: GeneratedAqlQuery[] = []
  
  if (filter.workspace) filters.push(aql`d.workspace == ${filter.workspace}`)
  if (filter.status?.length) filters.push(aql`d.status IN ${filter.status}`)
  if (filter.archived !== undefined) filters.push(aql`d.archived == ${filter.archived}`)
  
  return filters
}

// Uso
const q = aql`
  FOR d IN ${this.aql}
    ${filters.length > 0 ? aql`FILTER ${join(filters, ' FILTER ')}` : aql``}
  RETURN d
`
```

### Arrays

```typescript
// Buscar em array field
filters.push(aql`${user} IN d.assignees`)

// Índice para array
{ type: 'persistent', fields: ['assignees[*]'] }
```

### Agregações

```typescript
const q = aql`
  FOR d IN ${this.aql}
    COLLECT status = d.status
    AGGREGATE total = COUNT(d), amount = SUM(d.value)
  RETURN { status, count: total, amount }
`
```

### Full-Text Search (ArangoSearch)

```typescript
const q = aql`
  FOR d IN deals_search
    SEARCH ANALYZER(d.name IN TOKENS(${searchTerm}, "maia::pt_br_text_search"), "maia::pt_br_text_search")
    SORT BM25(d) DESC
  RETURN d
`
```

---

## Naming Conventions

| Tipo | Padrão | Exemplo |
|------|--------|---------|
| Collections | `snake_case` plural | `deals`, `whatsapp_messages` |
| Edge collections | `{entity1}_{entity2}_edge` | `user_tasks_edge` |
| Índices | `idx_{collection}_{fields}` | `idx_deals_workspace_status` |
| Campos | `camelCase` | `createdAt`, `workspaceKey` |
| IDs | `{collection}_{nanoId}` | `deals_abc123xyz` |

---

## Data Model Padrão

```typescript
export interface ArangoObject {
  _key: string
  _id?: string
  _rev?: string
  createdAt: string
  updatedAt: string
  createdBy?: string
  archived?: boolean
  archivedAt?: string
  archivedBy?: string
}

// Multi-tenant: sempre incluir workspace
export interface Deal extends ArangoObject {
  workspace: string // OBRIGATÓRIO
  name: string
  status: string
}
```

---

## Indexing

### Tipos de índice

| Tipo | Uso |
|------|-----|
| `persistent` | Filtros e ordenação (padrão) |
| `ttl` | Expiração automática |
| `fulltext` | Busca textual simples |

### Índices obrigatórios por collection

```typescript
// Exemplo: deals
{ type: 'persistent', fields: ['workspace'], name: 'idx_deals_workspace' }
{ type: 'persistent', fields: ['workspace', 'archived', 'status'], name: 'idx_deals_workspace_archived_status' }
{ type: 'persistent', fields: ['assignees[*]'], name: 'idx_deals_assignees' }
```

### Covering Index

Inclua campos retornados para evitar lookup:

```typescript
{ type: 'persistent', fields: ['workspace', 'status', 'name', 'value'] }
// Query usa apenas esses campos → não precisa ler documento
```

---

## Graph Operations

### Edge collections

```typescript
// Criar edge
await db.collection('user_tasks_edge').save({
  _from: `users/${userKey}`,
  _to: `tasks/${taskKey}`,
})

// Buscar relacionamentos
const q = aql`
  FOR e IN user_tasks_edge
    FILTER e._to == ${taskKey}
    RETURN PARSE_IDENTIFIER(e._from).key
`
```

### DOCUMENT() lookup

```typescript
const q = aql`
  FOR d IN deals
    LET contract = DOCUMENT("contracts", d.contractKey)
    FILTER contract != null
    RETURN { deal: d, contract }
`
```


---

## Anti-Patterns

| ❌ Evitar | ✅ Fazer |
|-----------|----------|
| String concatenation em queries | Usar `aql` tag |
| `FILTER` sem índice | Criar índice apropriado |
| `COLLECT` em collections grandes sem filtro | Filtrar antes de agregar |
| Múltiplos `DOCUMENT()` em loop | Usar `JOIN` ou batch |
| Queries sem `workspace` filter | Sempre filtrar por workspace |

---

## Referências

- [ArangoDB Docs](https://docs.arangodb.com/)
- [AQL Tutorial](https://docs.arangodb.com/stable/aql/tutorial/)
- [Indexing Guide](https://docs.arangodb.com/stable/index-and-search/indexing/)
- [arangojs](https://arangodb.github.io/arangojs/)
