# Guia de Boas Práticas ArangoDB - Berry/Maia API

**Versão:** 1.0.0
**Última atualização:** 01 de Dezembro de 2025
**Objetivo:** Documentar e padronizar a forma como a Berry usa ArangoDB

---

## Índice

1. [Introdução](#introdução)
2. [Configuração e Conexão](#configuração-e-conexão)
3. [Service Architecture Pattern](#service-architecture-pattern)
4. [Queries AQL - Padrões e Boas Práticas](#queries-aql---padrões-e-boas-práticas)
5. [Naming Conventions](#naming-conventions)
6. [Data Model Padrão](#data-model-padrão)
7. [Indexing - Estratégias e Boas Práticas](#indexing---estratégias-e-boas-práticas)
8. [ArangoSearch Views & Analyzers](#arangosearch-views--analyzers)
9. [Graph Operations](#graph-operations)
10. [Transactions](#transactions)
11. [Performance e Otimização](#performance-e-otimização)
12. [Error Handling](#error-handling)
13. [Collection Initialization](#collection-initialization)
14. [Best Practices - Checklist](#best-practices---checklist)
15. [Anti-Patterns - O Que Evitar](#anti-patterns---o-que-evitar)
16. [Exemplos Práticos Completos](#exemplos-práticos-completos)
17. [Troubleshooting](#troubleshooting)
18. [Referências](#referências)

---

## Introdução

Este documento **não** ensina ArangoDB do zero. Ele documenta e padroniza **como a Berry/Maia API já utiliza ArangoDB** na prática, servindo como guia de referência para manter consistência no desenvolvimento.

### Visão Geral do ArangoDB na Berry

O ArangoDB é o banco de dados principal da Berry/Maia API, utilizado para armazenar todos os dados da plataforma. A escolha do ArangoDB se deve à sua natureza **multi-model**, combinando três modelos de dados em um único sistema:

- **Document Store**: Armazenamento de documentos JSON flexíveis (deals, users, projects, etc.)
- **Graph Database**: Relacionamentos complexos via edge collections (user_tasks_edge, etc.)
- **Key-Value Store**: Acesso rápido por chave primária (`_key`)

### Por Que Multi-Model?

A Berry precisa de:

1. **Flexibilidade de Schema**: Documentos JSON permitem evolução rápida do modelo de dados
2. **Relacionamentos Complexos**: Graph queries para traversals de usuários, tarefas, projetos
3. **Performance**: Queries AQL otimizadas com índices persistentes e covering indexes
4. **Full-Text Search**: ArangoSearch com analyzers customizados para português brasileiro

### Stack Tecnológico

- **ArangoDB**: 3.11+ (multi-model database)
- **arangojs**: Driver oficial Node.js para ArangoDB
- **Node.js**: 24+ com TypeScript e ES Modules
- **Redis**: Cache layer para queries frequentes
- **BullMQ**: Event-driven architecture com listeners

### Documentação Oficial

Para conceitos básicos de ArangoDB, consulte:

- **Documentação Oficial**: [https://docs.arangodb.com/](https://docs.arangodb.com/)
- **AQL Tutorial**: [https://docs.arangodb.com/stable/aql/tutorial/](https://docs.arangodb.com/stable/aql/tutorial/)
- **Indexing**: [https://docs.arangodb.com/stable/index-and-search/indexing/](https://docs.arangodb.com/stable/index-and-search/indexing/)
- **ArangoSearch**: [https://docs.arangodb.com/stable/index-and-search/arangosearch/](https://docs.arangodb.com/stable/index-and-search/arangosearch/)
- **Graph Queries**: [https://docs.arangodb.com/stable/aql/graphs/](https://docs.arangodb.com/stable/aql/graphs/)
- **arangojs Documentation**: [https://arangodb.github.io/arangojs/](https://arangodb.github.io/arangojs/)

---

## Configuração e Conexão

### Singleton Pattern

A Berry utiliza um **singleton** para reutilizar a conexão com ArangoDB em toda a aplicação. A conexão é criada uma única vez e compartilhada entre todos os services.

**Arquivo:** `packages/api/src/infra/database.ts`

```typescript
import { Database as Arangodb } from 'arangojs'

let connection: Arangodb

export class Database {
  connection: Arangodb

  constructor() {
    const host = process.env.DB_HOST
    const port = process.env.DB_PORT
    const databaseName = process.env.DB_NAME ?? '_system'
    const username = process.env.DB_USERNAME ?? 'root'
    const password = process.env.DB_PASSWORD ?? ''

    Logger.debug(`Connecting to database ${host}:${port}`)

    if (isEmptyValue(connection)) {
      connection = new Arangodb({
        url: `http://${host}:${port}`,
        databaseName,
        auth: { username, password },
        agentOptions: {
          keepAlive: true,
          timeout: 120_000, // 2 minutos
          maxSockets: 50, // Limite de conexões simultâneas
          maxFreeSockets: 10, // Manter algumas conexões livres
        },
      })
    }

    this.connection = connection
  }
}
```

### Connection Pool - Configuração Otimizada

O `agentOptions` configura o pool de conexões HTTP do Node.js:

- **`keepAlive: true`**: Reutiliza conexões TCP ao invés de criar novas a cada request
- **`timeout: 120_000`**: Timeout de 2 minutos para operações longas (queries complexas, aggregations)
- **`maxSockets: 50`**: Permite até 50 conexões simultâneas com o ArangoDB
- **`maxFreeSockets: 10`**: Mantém 10 conexões livres no pool para reutilização imediata

**Por que esses valores?**

- **keepAlive**: Reduz latência ao evitar handshake TCP a cada query
- **timeout alto**: Queries complexas (aggregations, graph traversals) podem demorar
- **maxSockets alto**: API com múltiplos usuários simultâneos precisa de throughput
- **maxFreeSockets**: Balance entre performance e uso de recursos

### Variáveis de Ambiente Obrigatórias

```bash
DB_HOST=localhost
DB_PORT=8529
DB_NAME=maia
DB_USERNAME=root
DB_PASSWORD=your_secure_password
```

### Retry Logic - Tratamento de Lock Errors e Connection Errors

A Berry implementa **retry automático** para erros temporários de lock e conexão:

```typescript
public query = async <T extends object>(
  query: GeneratedAqlQuery,
  retryCount = 0,
): Promise<T[]> => {
  try {
    const results = await connection.query(query)
    return results.all()
  } catch (error: unknown) {
    const error_ = error as ArangoError

    Logger.error(error)
    Logger.error('Query Error: ', query)

    const lockErroNums = [18, 28, 29]
    const connectionErroNums = [1200, 1204, 1205] // Erros de conexão

    // Se for erro de lock ou conexão, tentar novamente
    if (
      error_.errorNum &&
      (lockErroNums.includes(error_.errorNum) ||
        connectionErroNums.includes(error_.errorNum)) &&
      retryCount < 3
    ) {
      Logger.warn(`Retrying query (attempt ${retryCount + 1}/3)`)
      await new Promise(resolve =>
        setTimeout(resolve, 1000 * (retryCount + 1)),
      ) // Delay progressivo
      return this.query(query, retryCount + 1)
    }

    // Se for erro de socket hang up, tentar reconectar
    if (
      (error_.message.includes('socket hang up') ||
        error_.message.includes('ECONNRESET')) &&
      retryCount < 2
    ) {
      Logger.warn(
        `Socket hang up detected, retrying query (attempt ${retryCount + 1}/2)`,
      )
      await new Promise(resolve =>
        setTimeout(resolve, 2000 * (retryCount + 1)),
      )
      return this.query(query, retryCount + 1)
    }

    throw error_
  }
}
```

**Erros Tratados:**

- **Lock Errors (18, 28, 29)**: Conflitos de write em documents (2 writes simultâneos)
- **Connection Errors (1200, 1204, 1205)**: Perda de conexão temporária com ArangoDB
- **Socket Errors**: `ECONNRESET`, `socket hang up` (network issues)

**Estratégia de Retry:**

- **Máximo de 3 tentativas** para lock/connection errors
- **Máximo de 2 tentativas** para socket errors
- **Backoff progressivo**: 1s, 2s, 3s (evita sobrecarga no server)
- **Re-throw** do erro após tentativas esgotadas

### Código Completo do database.ts

O arquivo completo inclui também:

```typescript
// GET de documento único com retry
public get = async (collection: string, key: string): Promise<JSObject> => {
  try {
    return await connection.collection(collection).document(key, {
      allowDirtyRead: false,
    })
  } catch (error: unknown) {
    const error_ = error as ArangoError

    if (error_.errorNum === 1202) {
      return undefined as unknown as JSObject
    }

    Logger.error(
      `ArangoDB GET error at collection: ${collection}. Key: ${key}`,
    )

    const lockErroNums = [18, 28, 29]
    if (error_.errorNum && lockErroNums.includes(error_.errorNum)) {
      return this.get(collection, key)
    }

    throw error_
  }
}

// Ensure Index com retry
public ensureIndex = async (
  collection: string,
  options: EnsurePersistentIndexOptions,
): Promise<Index> => {
  try {
    return await connection.collection(collection).ensureIndex(options)
  } catch (error: unknown) {
    const err = error as ArangoError
    const lockErrorNums = [18, 28, 29, 1200, 1204]

    if (err.errorNum && lockErrorNums.includes(err.errorNum)) {
      return this.ensureIndex(collection, options) // retry
    }

    Logger.error(
      `ARANGODB ENSURE INDEX error on ${collection}: ${err.message}`,
      options,
    )
    throw err
  }
}
```

---

## Service Architecture Pattern

### Base Service Class - Fundação de Todos os Services

**Todos os domain services na Berry estendem a classe base `Service<T>`**, que fornece funcionalidades comuns de CRUD, caching, DataLoader, e event emission.

**Arquivo:** `packages/api/src/infra/service.ts`

```typescript
export class Service<T extends ArangoObject> {
  elastic: ElasticService = elasticService
  database: Database = database
  dataLoader = new DataLoader<string, T>(this.findByKeys.bind(this))
  collection: CollectionName = 'none'
  model = ''

  get aql(): DocumentCollection {
    return this.database.connection.collection(this.collection)
  }
}
```

**Propriedades:**

- **`collection`**: Nome da collection (ex: `'deals'`, `'users'`, `'projects'`)
- **`model`**: Nome do modelo para eventos (ex: `'deal'`, `'user'`, `'project'`)
- **`dataLoader`**: DataLoader para batch loading (evita N+1 queries)
- **`aql`**: Referência à collection do ArangoDB (usado em queries: `FOR doc IN ${this.aql}`)

### Métodos CRUD Padronizados

#### `get(key: string): Promise<T>`

Busca documento único por `_key` com cache Redis:

```typescript
async get<R extends T>(key: string | undefined): Promise<R | undefined> {
  if (isEmptyValue(key)) {
    throw new Error(`invalid_collection_key: ${key}`)
  }

  const cachedResponse = await this.getCached(key)

  if (isNotEmptyValue(cachedResponse)) {
    return JSON.parse(cachedResponse) as R
  }

  const data = await this.database.get(this.collection, key)

  if (isNotEmptyValue(data)) {
    await this.setCache(key, JSON.stringify(data))
  }

  return data as R
}
```

#### `find(query: GeneratedAqlQuery): Promise<T[]>`

Executa query AQL customizada com cache:

```typescript
async find(query: GeneratedAqlQuery): Promise<T[] | undefined> {
  const stringQuery = `find:${JSON.stringify(query)}`

  const cachedResponse = await this.getCached(stringQuery)

  if (isNotEmptyValue(cachedResponse)) {
    return JSON.parse(cachedResponse) as T[]
  }

  const data = await this.database.query(query)

  if (isNotEmptyValue(data)) {
    await this.setCache(stringQuery, JSON.stringify(data))
  }

  return data as T[]
}
```

#### `findOne(query: GeneratedAqlQuery): Promise<T>`

Executa query AQL e retorna primeiro resultado:

```typescript
async findOne<R>(query: GeneratedAqlQuery): Promise<T | R | undefined> {
  const stringQuery = `findOne:${JSON.stringify(query)}`

  const cachedResponse = await this.getCached(stringQuery)

  if (isNotEmptyValue(cachedResponse)) {
    return JSON.parse(cachedResponse) as T
  }

  const data = await this.database.queryOne(query)

  if (data) {
    await this.setCache(stringQuery, JSON.stringify(data))
  }

  return data as T
}
```

#### `create(payload: Partial<T>): Promise<CreateUpdateResponse<T>>`

Cria novo documento com auto-timestamps, auto-generated IDs, e event emission:

```typescript
async create<T extends object>(
  payload: Partial<T> & Partial<ArangoObject>,
  options?: Partial<QueryOptions>,
): Promise<CreateUpdateResponse<T>> {
  const requestUser = options?.user ?? getMaiaUserKey()
  if (isEmptyValue(payload._key)) {
    const id = `${this.collection}_${nanoId()}`
    payload._key = id
  }
  if (isEmptyValue(payload.createdAt)) {
    payload.createdAt = dayjs().utc().toISOString()
  }
  if (isEmptyValue(payload.updatedAt)) {
    payload.updatedAt = dayjs().utc().toISOString()
  }
  if (isEmptyValue(payload.createdBy)) {
    payload.createdBy = requestUser
  }

  const response = await this.database.insertOne<T>(this.collection, payload)

  if (options?.dontEmit !== true && response.new) {
    try {
      const { handleObjectUpdate } = await import('@app/event/service')
      await handleObjectUpdate({
        collection: this.collection,
        current: response.new,
        previous: undefined,
        author: requestUser,
        changes: undefined,
      })
    } catch (error) {
      // Event handling is optional, don't fail the create operation
      Logger.warn('Failed to emit create event:', error)
    }
  }

  this.dataLoader.clearAll()
  await this.clearCache()
  return response
}
```

#### `update(key: string, payload: Partial<T>): Promise<CreateUpdateResponse<T>>`

Atualiza documento existente, emite eventos, invalida cache.

#### `delete(key: string): Promise<void>`

**Evitado na Berry** - Use `archive()` ao invés de delete físico.

#### `archive(key: string, user: string): Promise<void>`

Soft delete - marca documento como arquivado:

```typescript
async archive(key: string, user: string): Promise<void> {
  await this.update(key, {
    archived: true,
    archivedAt: dayjs().utc().toISOString(),
    archivedBy: user,
  })
}
```

#### `unarchive(key: string): Promise<void>`

Remove flag de arquivo:

```typescript
async unarchive(key: string): Promise<void> {
  await this.update(key, {
    archived: false,
    archivedAt: null,
    archivedBy: null,
  })
}
```

### Features Automáticas da Service Base

#### Auto-Timestamps

- **`createdAt`**: ISO 8601 UTC no create
- **`updatedAt`**: ISO 8601 UTC no create e update

#### Auto-Generated IDs

- **Formato:** `{collection}_{nanoId()}`
- **Exemplo:** `deals_abc123xyz`, `users_def456uvw`

#### Event Emission

Todos os creates e updates emitem evento `document_updated` que dispara listeners:

```typescript
await handleObjectUpdate({
  collection: this.collection,
  current: response.new,
  previous: undefined,
  author: requestUser,
  changes: undefined,
})
```

#### Redis Caching

- Cache automático em `get()`, `find()`, `findOne()`
- Invalidação automática em `create()`, `update()`, `delete()`
- TTL configurável (padrão: 5 minutos)

#### DataLoader para Batch Loading

Evita N+1 queries ao buscar múltiplos documentos:

```typescript
dataLoader = new DataLoader<string, T>(this.findByKeys.bind(this))

async findByKeys<T>(keys: readonly string[]): Promise<T[]> {
  const query = aql`
    FOR doc IN ${this.aql}
      FILTER doc._key IN ${keys}
    RETURN doc
  `
  const data = (await this.query(query)) as T[]
  const resultMap = new Map<string, T>(
    data.map((document: T) => [
      (document as ArangoObject)._key,
      document,
    ]) as [string, T][],
  )
  return keys.map(key => resultMap.get(key) ?? ({} as T)) as T[]
}
```

### Exemplo Completo: DealService Estendendo Service Base

**Arquivo:** `packages/api/src/deals/service.ts`

```typescript
export class DealService extends Service<Deal> {
  private static instance: DealService | undefined

  public constructor() {
    super()
    this.collection = 'deals'
    this.model = 'deal'
    this.dataLoader = new DataLoader<string, Deal>(this.findByKeys.bind(this))
  }

  public static getInstance(): DealService {
    DealService.instance ??= new DealService()
    return DealService.instance
  }

  // Métodos customizados além dos herdados do Service<T>
  async search(
    filter: Partial<DealFilter>,
    user: string | undefined,
  ): Promise<Deal[]> {
    // Lógica customizada de search
  }

  searchByName(filter: Partial<DealFilter>, user: string): Promise<Deal[]> {
    // Lógica de full-text search
  }

  buildDealFilters(
    filter: Partial<DealFilter>,
    user: string,
  ): GeneratedAqlQuery[] {
    // Construção de filtros dinâmicos
  }
}

export const dealService = DealService.getInstance()
```

**Uso:**

```typescript
// GET
const deal = await dealService.get('deals_abc123')

// CREATE
const newDeal = await dealService.create({
  name: 'Novo Deal',
  workspace: 'workspace_xyz',
  status: 'MQL',
})

// UPDATE
await dealService.update('deals_abc123', { status: 'SQL' })

// ARCHIVE
await dealService.archive('deals_abc123', 'user_xyz')

// CUSTOM QUERY
const deals = await dealService.search({ status: 'MQL' }, 'user_xyz')
```

---

## Queries AQL - Padrões e Boas Práticas

### 4.1 Template Literals com `aql`

**NUNCA use string concatenation para queries AQL.** Sempre use o `aql` tag template para type-safety e auto-escape.

#### Importações Necessárias

```typescript
import { aql, join, literal, GeneratedAqlQuery } from 'arangojs/aql.js'
```

- **`aql`**: Tag template para queries (type-safe, auto-escape de SQL injection)
- **`join()`**: Combina múltiplos fragments AQL com separador
- **`literal()`**: Insere string literal sem escape (use com cuidado!)
- **`GeneratedAqlQuery`**: Tipo do retorno de `aql`

#### Exemplo Básico

```typescript
const q = aql`
  FOR doc IN ${this.aql}
    FILTER doc._key == ${key}
  RETURN doc
`
```

**Por que usar `aql` tag?**

1. **Type-Safety**: TypeScript valida os tipos
2. **Auto-Escape**: Previne SQL injection automaticamente
3. **Composição**: Facilita construção incremental de queries

**❌ NUNCA FAÇA ISSO:**

```typescript
// ERRADO - String concatenation
const query = `FOR doc IN ${collectionName} FILTER doc._key == "${key}" RETURN doc`
```

### 4.2 Filtros Dinâmicos

A Berry usa **construção incremental de filtros** com `join()` para queries complexas.

#### Padrão de Construção

```typescript
buildDealFilters(
  filter: Partial<DealFilter>,
  user: string,
): GeneratedAqlQuery[] {
  const filters: GeneratedAqlQuery[] = []

  // Adiciona filtros condicionais
  if (filter.workspace !== undefined) {
    filters.push(aql`d.workspace == ${filter.workspace}`)
  }

  if (filter.status !== undefined && filter.status.length > 0) {
    filters.push(aql`d.status IN ${filter.status}`)
  }

  if (filter.archived !== undefined) {
    filters.push(aql`d.archived == ${filter.archived}`)
  }

  return filters
}
```

#### Uso em Query

```typescript
const filters = this.buildDealFilters(filter, user)
const limit = this.buildLimit(filter.limit, filter.offset)

const q = aql`
  FOR d IN ${this.aql}
    ${filters.length > 0 ? aql`FILTER ${join(filters, ' FILTER ')}` : aql``}
    ${limit}
  RETURN d
`
```

**Como funciona:**

- **`join(filters, ' FILTER ')`**: Combina `[filter1, filter2, filter3]` em `filter1 FILTER filter2 FILTER filter3`
- **`aql`` `**: Fragment vazio quando não há filtros
- **Condicional**: `${filters.length > 0 ? ... : aql``}` adiciona FILTER só se houver filtros

### 4.3 Queries com Arrays

#### IN para Arrays

```typescript
if (filter.status !== undefined && filter.status.length > 0) {
  filters.push(aql`d.status IN ${filter.status}`)
}
```

#### Array Indexes `field[*]`

Para buscar dentro de arrays, use `field[*]`:

```typescript
// Buscar deals com assignee específico
filters.push(aql`${user} IN d.assignees`)

// Índice para array field
{
  type: 'persistent',
  name: 'idx_deals_assignees',
  fields: ['assignees[*]'],
  sparse: false,
  cacheEnabled: true,
}
```

#### Substituição de Variáveis Mágicas

A Berry usa `$me` como placeholder para o usuário atual:

```typescript
private addAssigneeFilters(
  filters: GeneratedAqlQuery[],
  filter: Partial<DealFilter>,
  user: string,
): void {
  if (filter.assignees !== undefined && filter.assignees.length > 0) {
    if (filter.assignees.includes('$me') && user) {
      filter.assignees = filter.assignees.map(assignee =>
        assignee === '$me' ? user : assignee,
      )
    }
    const arrAqls: GeneratedAqlQuery[] = []
    for (const [index, assignee] of filter.assignees.entries()) {
      if (index === 0) {
        arrAqls.push(aql`${assignee} IN d.assignees`)
      } else {
        arrAqls.push(aql`OR ${assignee} IN d.assignees`)
      }
    }
    filters.push(aql`${join(arrAqls, ' OR ')}`)
  }
}
```

### 4.4 Agregações e Cálculos

#### COLLECT para Agrupar

```typescript
const q = aql`
  FOR d IN ${this.aql}
    ${filters.length > 0 ? aql`FILTER ${join(filters, ' FILTER ')}` : aql``}
    COLLECT status = d.status
    AGGREGATE total = COUNT(d), amount = SUM(d.value)
  RETURN { status, count: total, amount }
`
```

#### LET para Variáveis Intermediárias

```typescript
const q = aql`
  FOR d IN ${this.aql}
    ${filters.length > 0 ? aql`FILTER ${join(filters, ' FILTER ')}` : aql``}
    LET contract = DOCUMENT("contracts", d.contract)
    FILTER contract != null

    LET contractValue = SUM(
      FOR it IN contract.items || []
        RETURN it.amount * it.quantity
    )

    FILTER contractValue >= ${minAmount}
  RETURN d
`
```

#### Subqueries com FOR

```typescript
LET contractValue = SUM(
  FOR it IN contract.items || []
    RETURN it.amount * it.quantity
)

LET couponPercent = SUM(
  FOR c IN contract.coupons || []
    RETURN c.percentOff
)

LET couponAmount = SUM(
  FOR c IN contract.coupons || []
    RETURN c.amountOff
)

LET valorContrato = contractValue * (100 - couponPercent) / 100 - couponAmount
```

#### Exemplo Completo: Cálculo de Valor de Contrato com Cupons

**Arquivo:** `packages/api/src/projects/service.ts`

```typescript
buildContractFilters(filter: Partial<ProjectFilter>): GeneratedAqlQuery {
  const minAmount = filter.minAmount ?? null
  const maxAmount = filter.maxAmount ?? null

  if (minAmount !== null || maxAmount !== null) {
    return aql`
      LET contract = DOCUMENT("contracts", d.contract)
      FILTER contract != null

      LET contractValue = SUM(
        FOR it IN contract.items || []
          RETURN it.amount * it.quantity
      )

      LET pp = contract.paymentPlan || null
      LET couponSource = pp != null && pp.subscription != null
        ? pp.subscription.coupons
        : contract.coupons

      LET couponPercent = SUM(FOR c IN couponSource || [] RETURN c.percentOff)
      LET couponAmount = SUM(FOR c IN couponSource || [] RETURN c.amountOff)

      LET valorContrato = contractValue * (100 - couponPercent) / 100 - couponAmount

      ${minAmount === null ? aql`` : aql`FILTER valorContrato >= ${minAmount}`}
      ${maxAmount === null ? aql`` : aql`FILTER valorContrato <= ${maxAmount}`}
    `
  }

  return aql``
}
```

### 4.5 Full-Text Search

#### LIKE com Wildcards (Busca Simples)

```typescript
const q = aql`
  FOR d IN ${this.aql}
    FILTER LIKE(d.name, ${'%' + searchTerm + '%'}, true)
  RETURN d
`
```

#### ArangoSearch com Analyzers Customizados

Para busca avançada, use **ArangoSearch views** com analyzers customizados:

```typescript
searchByName(filter: Partial<DealFilter>, user: string): Promise<Deal[]> {
  const filters = this.buildDealFilters(filter, user)
  const limit = this.buildLimit(filter.limit, filter.offset)

  const q = aql`
    FOR d IN deals_search
      SEARCH (
        BOOST(ANALYZER(d.name IN TOKENS(${filter.name}, "maia::pt_br_text_search"), "maia::pt_br_text_search"), 2.0) OR
        ANALYZER(d.organizationChallenges IN TOKENS(${filter.name}, "maia::pt_br_text_search"), "maia::pt_br_text_search") OR
        ANALYZER(d.organizationTradeName IN TOKENS(${filter.name}, "maia::pt_br_text_search"), "maia::pt_br_text_search") OR
        ANALYZER(d.contactNames IN TOKENS(${filter.name}, "maia::pt_br_text_search"), "maia::pt_br_text_search")
      )
      ${filters.length > 0 ? aql`FILTER ${join(filters, ' FILTER ')}` : aql``}
      SORT BM25(d) DESC
      ${limit}
    RETURN d
  `
  return this.query<Deal>(q)
}
```

**Componentes:**

- **`SEARCH`**: Usa índice ArangoSearch (não `FILTER`)
- **`ANALYZER()`**: Aplica analyzer customizado ao campo
- **`TOKENS()`**: Tokeniza o termo de busca
- **`BOOST()`**: Pondera campo (2.0 = dobra a relevância)
- **`BM25()`**: Ordena por relevância (scoring)

### 4.6 Graph Queries

#### DOCUMENT() Lookup de Documentos Relacionados

```typescript
LET contract = DOCUMENT("contracts", d.contract)
FILTER contract != null
```

#### Edge Collections

Edge collections armazenam relacionamentos com `_from` e `_to`:

```typescript
const currentAssignees: string[] = await database.connection
  .query(
    aql`
      FOR e IN user_tasks_edge
        FILTER e._to == ${edgeTaskKey}
        RETURN PARSE_IDENTIFIER(e._from).key
    `,
  )
  .then(c => c.all())
```

#### PARSE_IDENTIFIER() para Extrair Keys

```typescript
// _from = "users/user_abc123"
// PARSE_IDENTIFIER(_from).key = "user_abc123"

FOR e IN user_tasks_edge
  FILTER e._to == ${taskKey}
  RETURN PARSE_IDENTIFIER(e._from).key
```

#### Exemplo: Busca de Tarefas por Usuário

```typescript
const q = aql`
  FOR e IN user_tasks_edge
    FILTER e._from == ${userKey}
    LET task = DOCUMENT(e._to)
    FILTER task != null
    FILTER task.archived != true
  RETURN task
`
```

---

## Naming Conventions

### Collections: `lowercase_snake_case` (Plural)

**Regra:** Collections devem ser nomeadas em **snake_case** no **plural**.

**Exemplos:**

```typescript
✅ Correto:
- users
- deals
- projects
- tasks
- whatsapp_messages
- chat_messages
- blog_posts
- call_records

❌ Incorreto:
- Users (PascalCase)
- user (singular)
- whatsappMessages (camelCase)
- USERS (UPPERCASE)
```

#### Edge Collections

Edge collections seguem o padrão `{entity1}_{entity2}_edge`:

```typescript
✅ Correto:
- user_tasks_edge
- deal_contacts_edge
- project_users_edge

❌ Incorreto:
- userTasksEdge
- user_task (faltando _edge)
- tasks_users_edge (ordem errada)
```

### Índices: `idx_{collection}_{field1}_{field2}`

**Regra:** Índices devem ser nomeados com prefixo `idx_`, seguido do nome da collection e campos.

**Exemplos:**

```typescript
✅ Correto:
- idx_deals_workspace
- idx_deals_workspace_archived_wonAt
- idx_deals_assignees
- idx_tasks_assignees

❌ Incorreto:
- deals_workspace_index
- workspace_deals_idx
- dealsWorkspaceIndex
```

### Campos: `camelCase`

**Regra:** Campos de documentos devem ser nomeados em **camelCase**.

**Exemplos:**

```typescript
✅ Correto:
{
  _key: "deals_abc123",
  createdAt: "2025-12-01T10:30:00.000Z",
  updatedAt: "2025-12-01T11:00:00.000Z",
  organizationTradeName: "Berry Consultoria",
  expectedMeetDate: "2025-12-15T14:00:00.000Z",
  readyForProject: true,
}

❌ Incorreto:
{
  _key: "deals_abc123",
  created_at: "...", // snake_case
  OrganizationTradeName: "...", // PascalCase
  expected_meet_date: "...", // snake_case
}
```

### IDs: `{collection}_{nanoId()}`

**Regra:** IDs auto-gerados seguem o formato `{collection}_{nanoId()}`.

**Exemplos:**

```typescript
✅ Correto:
- deals_abc123xyz
- users_def456uvw
- projects_ghi789rst
- tasks_jkl012mno

❌ Incorreto:
- abc123xyz (sem prefixo de collection)
- deal_abc123xyz (singular)
- deals-abc123xyz (hífen ao invés de underscore)
```

### Tabela de Referência: 80+ Collections

**Arquivo:** `packages/api/src/infra/collections.ts`

```typescript
export const collections = [
  'accounts_berrymars',
  'agents',
  'applications',
  'assets',
  'auction_bids',
  'auctions',
  'blog_posts',
  'calendar',
  'call_records',
  'cases',
  'chat_messages',
  'chats',
  'comments',
  'contracts',
  'courses_progress',
  'courses',
  'creatives',
  'crm_companies_berrymars',
  'deals',
  'drive_files',
  'events',
  'favorites',
  'file_parsings',
  'forms',
  'goals',
  'handbooks',
  'healths',
  'hires',
  'inbox_conversations',
  'inbox_messages',
  'jobs',
  'kpi_updates',
  'kpis',
  'learning',
  'lessons_contents_progress',
  'lessons_contents',
  'lessons_progress',
  'lessons',
  'meets',
  'messages',
  'models',
  'newsletters',
  'notes',
  'notifications',
  'organizations',
  'products',
  'projects',
  'reactions',
  'resources',
  'roleplays',
  'scenarios',
  'squads',
  'status_history',
  'statuses',
  'subs',
  'tasks',
  'teams',
  'tickets',
  'timeline',
  'tools',
  'tracks_progress',
  'tracks',
  'twilio',
  'users',
  'whatsapp_accounts',
  'whatsapp_conversations',
  'whatsapp_messages',
  'whatsapp',
  'workspaces',
  'analytics',
  'change_logs',
  'iugu_subs',
  'targets',
  'churn_signals',
  'partner_companies',
  'seniorities',
  'multiplier_tables',
] as const

export type CollectionName = (typeof collections)[number]
```

---

## Data Model Padrão

### Interface `ArangoObject` - Campos Obrigatórios

Todos os documentos na Berry implementam a interface `ArangoObject`:

**Arquivo:** `packages/api/src/infra/interfaces.ts`

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
```

**Campos:**

- **`_key`**: Identificador único do documento (ex: `deals_abc123`)
- **`_id`**: ID completo com collection (ex: `deals/deals_abc123`) - auto-gerado pelo ArangoDB
- **`_rev`**: Revision token - auto-gerado pelo ArangoDB
- **`createdAt`**: Timestamp ISO 8601 UTC de criação
- **`updatedAt`**: Timestamp ISO 8601 UTC de última atualização
- **`createdBy`**: Key do usuário que criou o documento
- **`archived`**: Flag de soft delete (true = arquivado)
- **`archivedAt`**: Timestamp de arquivamento
- **`archivedBy`**: Key do usuário que arquivou

### Workspace Isolation

**Todos os documentos multi-tenant incluem campo `workspace`:**

```typescript
export interface Deal extends ArangoObject {
  _key: string
  workspace: string // OBRIGATÓRIO para multi-tenancy
  name: string
  status: string
  // ... outros campos
}
```

**Filtro automático por workspace:**

```typescript
buildDealFilters(filter: Partial<DealFilter>, user: string): GeneratedAqlQuery[] {
  const filters: GeneratedAqlQuery[] = []

  if (filter.workspace !== undefined) {
    filters.push(aql`d.workspace == ${filter.workspace}`)
  }

  // ... outros filtros
}
```

### Soft Delete - Archive Pattern

**NUNCA use delete físico**. Sempre use o padrão de archive:

```typescript
// ❌ EVITAR
await dealService.delete('deals_abc123')

// ✅ USAR
await dealService.archive('deals_abc123', 'user_xyz')
```

**Implementação:**

```typescript
async archive(key: string, user: string): Promise<void> {
  await this.update(key, {
    archived: true,
    archivedAt: dayjs().utc().toISOString(),
    archivedBy: user,
  })
}

async unarchive(key: string): Promise<void> {
  await this.update(key, {
    archived: false,
    archivedAt: null,
    archivedBy: null,
  })
}
```

**Filtro para excluir arquivados:**

```typescript
const q = aql`
  FOR d IN ${this.aql}
    FILTER d.archived != true
  RETURN d
`
```

### Timestamps - ISO 8601 UTC

**Sempre use ISO 8601 UTC** para timestamps:

```typescript
createdAt: "2025-12-01T10:30:00.000Z"
updatedAt: "2025-12-01T11:00:00.000Z"
archivedAt: "2025-12-01T12:00:00.000Z"
```

**Geração:**

```typescript
import dayjs from 'dayjs'

payload.createdAt = dayjs().utc().toISOString()
payload.updatedAt = dayjs().utc().toISOString()
```

**❌ EVITAR:**

```typescript
// Timestamp Unix
createdAt: 1701427800

// Date local sem UTC
createdAt: "2025-12-01T10:30:00-03:00"

// String formatada
createdAt: "01/12/2025 10:30"
```

### Auditoria - createdBy, archivedBy

**Campos de auditoria rastreiam ações:**

```typescript
{
  createdBy: "user_abc123", // Quem criou
  archivedBy: "user_xyz789", // Quem arquivou
}
```

**Uso em queries:**

```typescript
// Filtrar documentos criados por usuário específico
filters.push(aql`d.createdBy == ${user}`)

// Filtrar documentos arquivados por usuário específico
filters.push(aql`d.archivedBy == ${user}`)
```

---

## Indexing - Estratégias e Boas Práticas

### 7.1 Tipos de Índices

#### Persistent - Uso Geral (Default)

**Índice padrão na Berry**. Armazenado em disco com cache em memória.

```typescript
{
  type: 'persistent',
  name: 'idx_deals_workspace',
  fields: ['workspace'],
  sparse: false,
  cacheEnabled: true,
}
```

#### Unique - Garantir Unicidade

Impede duplicatas em campo ou combinação de campos:

```typescript
{
  type: 'persistent',
  name: 'idx_users_email',
  fields: ['email'],
  unique: true,
  sparse: false,
  cacheEnabled: true,
}
```

#### Sparse - Apenas Documentos com o Campo

Indexa apenas documentos que possuem o campo (útil para campos opcionais):

```typescript
{
  type: 'persistent',
  name: 'idx_deals_wonAt',
  fields: ['wonAt'],
  sparse: true, // Apenas deals com wonAt
  cacheEnabled: true,
}
```

#### Array Indexes - `field[*]`

Indexa valores dentro de arrays:

```typescript
{
  type: 'persistent',
  name: 'idx_deals_assignees',
  fields: ['assignees[*]'],
  sparse: false,
  cacheEnabled: true,
}
```

**Uso:**

```typescript
// Buscar deals com assignee específico
const q = aql`
  FOR d IN ${this.aql}
    FILTER ${user} IN d.assignees
  RETURN d
`
```

### 7.2 Ordem de Campos (CRÍTICO)

**Regra de Ouro: Igualdades → Ranges → Sort**

A **ordem dos campos no índice** é crítica para performance. ArangoDB usa índices de **esquerda para direita**.

#### ✅ Ordem Correta

```typescript
{
  type: 'persistent',
  name: 'idx_deals_ws_archived_wonAt',
  fields: ['workspace', 'archived', 'wonAt'], // ✅ Correto
  sparse: false,
  cacheEnabled: true,
}
```

**Query correspondente:**

```typescript
const q = aql`
  FOR d IN ${this.aql}
    FILTER d.workspace == ${workspace}  // Igualdade (1º campo)
    FILTER d.archived == false          // Igualdade (2º campo)
    FILTER d.wonAt >= ${startDate}      // Range (3º campo)
    SORT d.wonAt DESC
  RETURN d
`
```

**Por que funciona?**

1. **`workspace == ...`**: Equality filter no 1º campo do índice
2. **`archived == ...`**: Equality filter no 2º campo do índice
3. **`wonAt >= ...`**: Range filter no 3º campo do índice
4. **`SORT wonAt DESC`**: Usa o 3º campo do índice

#### ❌ Ordem Incorreta

```typescript
{
  type: 'persistent',
  name: 'idx_deals_wonAt_ws_archived',
  fields: ['wonAt', 'workspace', 'archived'], // ❌ Incorreto
  sparse: false,
  cacheEnabled: true,
}
```

**Query correspondente:**

```typescript
const q = aql`
  FOR d IN ${this.aql}
    FILTER d.workspace == ${workspace}  // ❌ Não usa índice
    FILTER d.archived == false          // ❌ Não usa índice
    FILTER d.wonAt >= ${startDate}      // ✅ Usa índice
  RETURN d
`
```

**Por que NÃO funciona?**

- **`wonAt`** é o 1º campo do índice, mas a query filtra por **`workspace`** primeiro
- ArangoDB não consegue usar o índice eficientemente
- Resultado: **full collection scan** (lento)

#### Prioridade de Campos

1. **Igualdades (`==`)**: Campos com equality filters devem vir primeiro
2. **Ranges (`>`, `<`, `>=`, `<=`, `IN`)**: Depois das igualdades
3. **Sort**: Último campo do índice

**Exemplo complexo:**

```typescript
// Query
FILTER d.workspace == ${workspace}    // Equality
FILTER d.archived == false            // Equality
FILTER d.status IN ${statuses}        // Range (IN)
FILTER d.wonAt >= ${startDate}        // Range
SORT d.wonAt DESC

// Índice correspondente
fields: ['workspace', 'archived', 'status', 'wonAt']
```

### 7.3 Covering Indexes

**Covering index** inclui todos os campos necessários na query, evitando materialização do documento completo.

#### O Que São?

Índice que contém **todos os campos** usados na query (FILTER, SORT, RETURN).

#### Quando Usar?

- Queries **frequentes** com poucos campos no RETURN
- Performance crítica (dashboards, analytics)
- Queries com agregações simples

#### Exemplo

```typescript
{
  type: 'persistent',
  name: 'idx_deals_ws_archived_won_wonAt',
  fields: ['workspace', 'archived', 'won', 'wonAt'],
  sparse: false,
  cacheEnabled: true,
  storedValues: ['contract'], // ✅ Covering index
}
```

**Query correspondente:**

```typescript
const q = aql`
  FOR d IN ${this.aql}
    FILTER d.workspace == ${workspace}
    FILTER d.archived == false
    FILTER d.won == true
  RETURN { _key: d._key, contract: d.contract, wonAt: d.wonAt }
`
```

**Por que funciona?**

- Todos os campos do RETURN estão no índice: `_key`, `wonAt` (no `fields`) e `contract` (no `storedValues`)
- ArangoDB **não precisa** materializar o documento completo
- **Performance gain significativo** (2-5x mais rápido)

#### ❌ Quando NÃO Usar

- Queries que retornam muitos campos
- Índices ficam muito grandes (aumenta uso de memória)
- Queries infrequentes

### 7.4 Cache Enabled

**Sempre use `cacheEnabled: true` para queries frequentes.**

```typescript
{
  type: 'persistent',
  name: 'idx_deals_workspace',
  fields: ['workspace'],
  cacheEnabled: true, // ✅ Performance gain
}
```

**O que faz?**

- ArangoDB cacheia resultados de index lookups em memória
- Reduz I/O de disco
- **Performance gain significativo** para queries frequentes

**Quando desabilitar?**

- Índices raramente usados (economiza memória)
- Queries com alta cardinalidade (muitos valores únicos)

### 7.5 Criação de Índices

#### Script de Inicialização

**Arquivo:** `packages/api/src/scripts/indexes/add-new-indexes.ts`

```typescript
export async function createDealIndexes(): Promise<void> {
  const db = new Database().database('maia')
  const col = db.collection('deals')

  // Verificar índices existentes antes de criar
  const existingIndexes = await col.indexes()
  const existingIndexNames = new Set(existingIndexes.map(idx => idx.name))

  const indexes: EnsurePersistentIndexOptions[] = [
    {
      type: 'persistent',
      name: 'idx_deals_workspace',
      fields: ['workspace'],
      sparse: false,
      cacheEnabled: true,
    },
    {
      type: 'persistent',
      name: 'idx_deals_workspace_archived',
      fields: ['workspace', 'archived'],
      sparse: false,
      cacheEnabled: true,
    },
    {
      type: 'persistent',
      name: 'idx_deals_ws_archived_wonAt',
      fields: ['workspace', 'archived', 'wonAt'],
      sparse: false,
      cacheEnabled: true,
      storedValues: ['contract'], // Covering index
    },
    {
      type: 'persistent',
      name: 'idx_deals_assignees',
      fields: ['assignees[*]'],
      sparse: false,
      cacheEnabled: true,
    },
  ]

  for (const indexOptions of indexes) {
    if (!existingIndexNames.has(indexOptions.name)) {
      await col.ensureIndex(indexOptions)
      Logger.info(`Index ${indexOptions.name} created`)
    } else {
      Logger.debug(`Index ${indexOptions.name} already exists, skipping`)
    }
  }
}
```

#### Verificar Índices Existentes

**Sempre verifique se o índice já existe antes de criar:**

```typescript
const existingIndexes = await col.indexes()
const existingIndexNames = new Set(existingIndexes.map(idx => idx.name))

if (!existingIndexNames.has('idx_deals_workspace')) {
  await col.ensureIndex({
    type: 'persistent',
    name: 'idx_deals_workspace',
    fields: ['workspace'],
  })
}
```

#### Logs de Criação

**Use logging estruturado:**

```typescript
Logger.info(`Creating index: ${indexOptions.name}`)
Logger.info(`Index ${indexOptions.name} created successfully`)
Logger.debug(`Index ${indexOptions.name} already exists, skipping`)
```

---

## ArangoSearch Views & Analyzers

### 8.1 Analyzers Customizados

A Berry usa **analyzers customizados** para full-text search em português brasileiro.

**Arquivo:** `packages/api/src/infra/arangodb/analyzers.ts`

#### `maia::pt_br_text_search` - Texto em Português

```typescript
await db.connection.createAnalyzer('maia::pt_br_text_search', {
  type: 'text',
  properties: {
    locale: 'pt_BR',
    case: 'lower',
    accent: false,
    stemming: true,
    edgeNgram: {
      min: 1,
      max: 15,
      preserveOriginal: true,
    },
  },
  features: ['frequency', 'position', 'norm'],
})
```

**Características:**

- **`locale: 'pt_BR'`**: Stemming específico para português brasileiro
- **`case: 'lower'`**: Normaliza para minúsculas
- **`accent: false`**: Remove acentos (café = cafe)
- **`stemming: true`**: Reduz palavras à raiz (consultoria = consult)
- **`edgeNgram`**: Prefixos de 1-15 caracteres para busca por prefixo
- **`preserveOriginal: true`**: Mantém token original além dos n-grams

**Uso:**

```typescript
SEARCH ANALYZER(d.name IN TOKENS(${searchTerm}, "maia::pt_br_text_search"), "maia::pt_br_text_search")
```

#### `maia::pt_br_ngram` - N-grams

```typescript
await db.connection.createAnalyzer('maia::pt_br_ngram', {
  type: 'ngram',
  properties: {
    max: 5,
    min: 2,
    preserveOriginal: true,
  },
  features: ['position', 'norm', 'frequency'],
})
```

**Características:**

- **`min: 2, max: 5`**: N-grams de 2 a 5 caracteres
- **`preserveOriginal: true`**: Mantém token original

**Uso:** Busca fuzzy, typos, partial matches.

#### Código de Criação Completo

```typescript
export async function createPtBrAnalyzers(): Promise<void> {
  const db = database

  // Drop analyzers existentes
  const analyzers = await db.connection.listAnalyzers()
  const maiaAnalyzers = analyzers.filter(a => a.name.startsWith('maia::'))
  for (const analyzer of maiaAnalyzers) {
    try {
      await db.connection.analyzer(analyzer.name).drop()
    } catch {
      console.error(`Analyzer ${analyzer.name} not found`)
    }
  }

  // Criar analyzers
  await db.connection.createAnalyzer('maia::pt_br_ngram', {
    type: 'ngram',
    properties: {
      max: 5,
      min: 2,
      preserveOriginal: true,
    },
    features: ['position', 'norm', 'frequency'],
  })

  await db.connection.createAnalyzer('maia::pt_br_text_search', {
    type: 'text',
    properties: {
      locale: 'pt_BR',
      case: 'lower',
      accent: false,
      stemming: true,
      edgeNgram: {
        min: 1,
        max: 15,
        preserveOriginal: true,
      },
    },
    features: ['frequency', 'position', 'norm'],
  })
}
```

### 8.2 Views de Full-Text Search

**Arquivo:** `packages/api/src/infra/arangodb/views.ts`

#### Estrutura de uma View

```typescript
await db.createView('deals_search', {
  type: 'arangosearch',
  primarySort: [
    {
      field: 'createdAt',
      direction: 'desc',
    },
  ],
  links: {
    deals: {
      fields: {
        name: { analyzers: ['maia::pt_br_text_search'] },
        organizationChallenges: { analyzers: ['maia::pt_br_text_search'] },
        organizationTradeName: { analyzers: ['maia::pt_br_text_search'] },
        contactNames: { analyzers: ['maia::pt_br_text_search'] },
        contactPhones: { analyzers: ['maia::pt_br_text_search'] },
        contactEmails: { analyzers: ['maia::pt_br_text_search'] },
      },
      includeAllFields: false,
      storeValues: 'none',
      trackListPositions: false,
    },
  },
})
```

**Componentes:**

- **`type: 'arangosearch'`**: Tipo da view
- **`primarySort`**: Ordenação padrão (otimiza queries com SORT)
- **`links`**: Mapeia collections e campos indexados
- **`fields`**: Campos indexados com analyzers
- **`includeAllFields: false`**: Indexa apenas campos especificados
- **`storeValues: 'none'`**: Não armazena valores originais (economiza espaço)
- **`trackListPositions: false`**: Não rastreia posições em arrays (economiza espaço)

#### primarySort para Otimização

**`primarySort` otimiza queries que ordenam pelo mesmo campo:**

```typescript
primarySort: [
  {
    field: 'createdAt',
    direction: 'desc',
  },
]
```

**Query otimizada:**

```typescript
const q = aql`
  FOR d IN deals_search
    SEARCH ANALYZER(d.name IN TOKENS(${searchTerm}, "maia::pt_br_text_search"), "maia::pt_br_text_search")
    SORT d.createdAt DESC  // ✅ Usa primarySort
  RETURN d
`
```

#### Links e Campos Indexados

**Apenas campos especificados em `links.fields` são indexados:**

```typescript
links: {
  deals: {
    fields: {
      name: { analyzers: ['maia::pt_br_text_search'] },
      organizationChallenges: { analyzers: ['maia::pt_br_text_search'] },
      contactNames: { analyzers: ['maia::pt_br_text_search'] },
    },
    includeAllFields: false, // ✅ Indexa apenas campos especificados
  },
}
```

**❌ EVITAR:**

```typescript
includeAllFields: true // Indexa TODOS os campos (lento, alto uso de memória)
```

#### Exemplo: `deals_search` View

```typescript
await db.createView('deals_search', {
  type: 'arangosearch',
  primarySort: [
    {
      field: 'createdAt',
      direction: 'desc',
    },
  ],
  links: {
    deals: {
      fields: {
        name: { analyzers: ['maia::pt_br_text_search'] },
        organizationChallenges: { analyzers: ['maia::pt_br_text_search'] },
        organizationTradeName: { analyzers: ['maia::pt_br_text_search'] },
        organizationLegalName: { analyzers: ['maia::pt_br_text_search'] },
        contactNames: { analyzers: ['maia::pt_br_text_search'] },
        contactPhones: { analyzers: ['maia::pt_br_text_search'] },
        contactEmails: { analyzers: ['maia::pt_br_text_search'] },
      },
      includeAllFields: false,
      storeValues: 'none',
      trackListPositions: false,
    },
  },
})
```

### 8.3 Usando Views em Queries

#### Sintaxe `SEARCH` vs `FILTER`

**SEARCH**: Usa índice ArangoSearch (fast)
**FILTER**: Usa índice persistente ou full scan (slower para text search)

```typescript
// ✅ USAR SEARCH para full-text
FOR d IN deals_search
  SEARCH ANALYZER(d.name IN TOKENS(${searchTerm}, "maia::pt_br_text_search"), "maia::pt_br_text_search")
  RETURN d

// ❌ EVITAR FILTER para full-text
FOR d IN deals
  FILTER LIKE(d.name, ${'%' + searchTerm + '%'})
  RETURN d
```

#### BM25() para Ordenação por Relevância

**BM25** é um algoritmo de scoring que ordena resultados por relevância:

```typescript
FOR d IN deals_search
  SEARCH ANALYZER(d.name IN TOKENS(${searchTerm}, "maia::pt_br_text_search"), "maia::pt_br_text_search")
  SORT BM25(d) DESC  // ✅ Ordena por relevância
  LIMIT 10
RETURN d
```

#### BOOST() para Ponderar Campos

**BOOST** aumenta a relevância de campos específicos:

```typescript
FOR d IN deals_search
  SEARCH (
    BOOST(ANALYZER(d.name IN TOKENS(${searchTerm}, "maia::pt_br_text_search"), "maia::pt_br_text_search"), 2.0) OR
    ANALYZER(d.organizationChallenges IN TOKENS(${searchTerm}, "maia::pt_br_text_search"), "maia::pt_br_text_search")
  )
  SORT BM25(d) DESC
RETURN d
```

**Como funciona:**

- **`BOOST(..., 2.0)`**: Dobra a relevância do campo `name`
- Matches em `name` aparecem antes de matches em `organizationChallenges`

#### Exemplo Completo de Query

**Arquivo:** `packages/api/src/deals/service.ts`

```typescript
searchByName(filter: Partial<DealFilter>, user: string): Promise<Deal[]> {
  const filters = this.buildDealFilters(filter, user)
  const limit = this.buildLimit(filter.limit, filter.offset)

  const q = aql`
    FOR d IN deals_search
      SEARCH (
        BOOST(ANALYZER(d.name IN TOKENS(${filter.name}, "maia::pt_br_text_search"), "maia::pt_br_text_search"), 2.0) OR
        ANALYZER(d.organizationChallenges IN TOKENS(${filter.name}, "maia::pt_br_text_search"), "maia::pt_br_text_search") OR
        ANALYZER(d.organizationTradeName IN TOKENS(${filter.name}, "maia::pt_br_text_search"), "maia::pt_br_text_search") OR
        ANALYZER(d.organizationLegalName IN TOKENS(${filter.name}, "maia::pt_br_text_search"), "maia::pt_br_text_search") OR
        ANALYZER(d.contactNames IN TOKENS(${filter.name}, "maia::pt_br_text_search"), "maia::pt_br_text_search") OR
        ANALYZER(d.contactPhones IN TOKENS(${filter.name}, "maia::pt_br_text_search"), "maia::pt_br_text_search") OR
        ANALYZER(d.contactEmails IN TOKENS(${filter.name}, "maia::pt_br_text_search"), "maia::pt_br_text_search")
      )
      ${filters.length > 0 ? aql`FILTER ${join(filters, ' FILTER ')}` : aql``}
      SORT BM25(d) DESC
      ${limit}
    RETURN d
  `
  return this.query<Deal>(q)
}
```

---

## Graph Operations

### 9.1 Edge Collections

#### Estrutura

Edge collections armazenam relacionamentos com campos especiais `_from` e `_to`:

```typescript
{
  _key: "user_tasks_edge_abc123",
  _from: "users/user_xyz789",  // Document ID do usuário
  _to: "tasks/task_abc123",    // Document ID da tarefa
}
```

**Campos obrigatórios:**

- **`_from`**: ID completo do documento de origem (formato: `{collection}/{_key}`)
- **`_to`**: ID completo do documento de destino (formato: `{collection}/{_key}`)

#### Naming

**Padrão:** `{entity1}_{entity2}_edge`

**Exemplos:**

```typescript
✅ Correto:
- user_tasks_edge
- deal_contacts_edge
- project_users_edge

❌ Incorreto:
- userTasksEdge (camelCase)
- user_task (faltando _edge)
- tasks_users_edge (ordem errada)
```

#### Exemplo: `user_tasks_edge`

```typescript
// Criar edge collection
await db.connection.createCollection('user_tasks_edge', {
  type: 3, // Edge collection
})

// Criar índice para _from
await db.connection.collection('user_tasks_edge').ensureIndex({
  type: 'persistent',
  fields: ['_from'],
  unique: false,
})

// Criar índice para _to
await db.connection.collection('user_tasks_edge').ensureIndex({
  type: 'persistent',
  fields: ['_to'],
  unique: false,
})
```

### 9.2 Criando e Atualizando Edges

#### Diff Pattern - Comparar Estado Atual vs Desejado

A Berry usa o **diff pattern** para sincronizar edges:

**Arquivo:** `packages/api/src/tasks/cases/assignee-edge.ts`

```typescript
export class UpdateUserTaskEdgeCollection extends EventListener<Task> {
  override async execute(payload: ChangePayload<Model>): Promise<void> {
    const { current } = payload
    const edgeTaskKey = 'tasks/' + current._key

    // 1. Buscar assignees atuais nas edges
    const currentAssignees: string[] = await database.connection
      .query(
        aql`
          FOR e IN user_tasks_edge
            FILTER e._to == ${edgeTaskKey}
            RETURN PARSE_IDENTIFIER(e._from).key
        `,
      )
      .then(c => c.all())

    // 2. Diff: comparar estado atual vs desejado
    const toAdd =
      current.assignees?.filter(u => !currentAssignees.includes(u)) ?? []
    const toRemove = currentAssignees.filter(
      u => (current.assignees && !current.assignees.includes(u)) ?? false,
    )

    // 3. Transaction para atomicidade
    const trx: Transaction = await database.connection.beginTransaction({
      write: ['user_tasks_edge'],
    })

    // Adicionar edges faltantes
    for (const userKey of toAdd) {
      await trx.step(async () => {
        const fromVertex = `users/${userKey}`
        const toVertex = edgeTaskKey

        // Verificar se edge já existe
        const existingEdge = await database.connection
          .query(
            aql`
              FOR e IN user_tasks_edge
                FILTER e._from == ${fromVertex} AND e._to == ${toVertex}
                LIMIT 1
              RETURN e
            `,
          )
          .then(cursor => cursor.next())

        if (isNotEmptyValue(existingEdge)) {
          return
        }

        // Criar edge
        return database.connection
          .collection('user_tasks_edge')
          .save({ _from: fromVertex, _to: toVertex })
      })
    }

    // Remover edges obsoletas
    if (toRemove.length > 0) {
      await trx.step(() =>
        database.connection.query(aql`
          FOR e IN user_tasks_edge
            FILTER e._to == ${edgeTaskKey}
              AND PARSE_IDENTIFIER(e._from).key IN ${toRemove}
            REMOVE e IN user_tasks_edge
        `),
      )
    }

    await trx.commit()
  }
}
```

#### Transactions para Atomicidade

**Sempre use transactions ao criar/atualizar múltiplas edges:**

```typescript
const trx: Transaction = await database.connection.beginTransaction({
  write: ['user_tasks_edge'],
})

await trx.step(async () => {
  // Operação 1
})

await trx.step(async () => {
  // Operação 2
})

await trx.commit()
```

**Por que?**

- **Atomicidade**: Todas as operações são executadas ou nenhuma é
- **Consistência**: Evita estados intermediários inconsistentes

#### PARSE_IDENTIFIER() para Extrair Keys

**PARSE_IDENTIFIER()** extrai componentes de um Document ID:

```typescript
// _from = "users/user_abc123"
PARSE_IDENTIFIER(_from).key    // "user_abc123"
PARSE_IDENTIFIER(_from).collection  // "users"

// Uso em query
FOR e IN user_tasks_edge
  FILTER e._to == ${taskKey}
  RETURN PARSE_IDENTIFIER(e._from).key  // Retorna apenas a key do usuário
```

### 9.3 Traversals

#### DOCUMENT() Lookup Simples

**DOCUMENT()** busca documento por ID completo:

```typescript
const q = aql`
  FOR d IN ${this.aql}
    LET contract = DOCUMENT("contracts", d.contract)
    FILTER contract != null
  RETURN { deal: d, contract }
`
```

**Sintaxe:**

- **`DOCUMENT("collection", key)`**: Busca documento por collection e key
- **`DOCUMENT(documentId)`**: Busca documento por ID completo (`"contracts/contract_abc123"`)

#### FOR...IN para Percorrer Edges

**Buscar tarefas de um usuário:**

```typescript
const q = aql`
  FOR e IN user_tasks_edge
    FILTER e._from == ${'users/' + userKey}
    LET task = DOCUMENT(e._to)
    FILTER task != null
    FILTER task.archived != true
  RETURN task
`
```

**Buscar usuários de uma tarefa:**

```typescript
const q = aql`
  FOR e IN user_tasks_edge
    FILTER e._to == ${'tasks/' + taskKey}
    LET user = DOCUMENT(e._from)
    FILTER user != null
  RETURN user
`
```

#### Profundidade de Traversal

Para traversals complexos, use `FOR...IN...INBOUND/OUTBOUND`:

```typescript
// Buscar tarefas de um usuário (1 hop)
FOR e IN 1..1 OUTBOUND ${'users/' + userKey} user_tasks_edge
  RETURN e

// Buscar projetos e tarefas de um usuário (2 hops)
FOR e IN 1..2 OUTBOUND ${'users/' + userKey} user_projects_edge, project_tasks_edge
  RETURN e
```

**Sintaxe:**

- **`1..1`**: Profundidade mínima e máxima (1 hop)
- **`OUTBOUND`**: Segue edges de `_from` para `_to`
- **`INBOUND`**: Segue edges de `_to` para `_from`
- **`ANY`**: Segue edges em ambas as direções

---

## Transactions

### Quando Usar Transactions

Use transactions quando precisar **atomicidade** em múltiplas operações:

- Criar/atualizar múltiplas edges
- Operações interdependentes (criar deal + criar contract)
- Evitar estados intermediários inconsistentes

### beginTransaction() com Write Collections

```typescript
const trx: Transaction = await database.connection.beginTransaction({
  write: ['user_tasks_edge', 'tasks'],
})
```

**Parâmetros:**

- **`write`**: Array de collections que serão modificadas
- **`read`**: Array de collections que serão apenas lidas (opcional)

### trx.step() para Operações Sequenciais

```typescript
const trx: Transaction = await database.connection.beginTransaction({
  write: ['user_tasks_edge'],
})

// Operação 1
await trx.step(async () => {
  return database.connection
    .collection('user_tasks_edge')
    .save({ _from: 'users/user_abc', _to: 'tasks/task_xyz' })
})

// Operação 2
await trx.step(async () => {
  return database.connection
    .collection('user_tasks_edge')
    .save({ _from: 'users/user_def', _to: 'tasks/task_xyz' })
})

await trx.commit()
```

**Cada `step()` é uma operação atômica dentro da transaction.**

### trx.commit() / trx.abort()

```typescript
try {
  const trx = await database.connection.beginTransaction({
    write: ['user_tasks_edge'],
  })

  await trx.step(async () => {
    // Operação 1
  })

  await trx.step(async () => {
    // Operação 2
  })

  await trx.commit()  // ✅ Confirma todas as operações
} catch (error) {
  await trx.abort()  // ❌ Reverte todas as operações
  throw error
}
```

**Importante:**

- **`commit()`**: Confirma todas as operações
- **`abort()`**: Reverte todas as operações (rollback)
- Sem `commit()`, as operações NÃO são aplicadas

### Exemplo: Criação de Edges com Transaction

**Arquivo:** `packages/api/src/tasks/cases/assignee-edge.ts`

```typescript
const trx: Transaction = await database.connection.beginTransaction({
  write: ['user_tasks_edge'],
})

// Adicionar edges faltantes
for (const userKey of toAdd) {
  await trx.step(async () => {
    const fromVertex = `users/${userKey}`
    const toVertex = edgeTaskKey

    // Verificar se edge já existe
    const existingEdge = await database.connection
      .query(
        aql`
          FOR e IN user_tasks_edge
            FILTER e._from == ${fromVertex} AND e._to == ${toVertex}
            LIMIT 1
          RETURN e
        `,
      )
      .then(cursor => cursor.next())

    if (isNotEmptyValue(existingEdge)) {
      return
    }

    // Criar edge
    return database.connection
      .collection('user_tasks_edge')
      .save({ _from: fromVertex, _to: toVertex })
  })
}

// Remover edges obsoletas
if (toRemove.length > 0) {
  await trx.step(() =>
    database.connection.query(aql`
      FOR e IN user_tasks_edge
        FILTER e._to == ${edgeTaskKey}
          AND PARSE_IDENTIFIER(e._from).key IN ${toRemove}
        REMOVE e IN user_tasks_edge
    `),
  )
}

await trx.commit()
```

---

## Performance e Otimização

### 11.1 DataLoader Pattern

**DataLoader** evita N+1 queries ao buscar múltiplos documentos relacionados.

#### Problema: N+1 Queries

```typescript
// ❌ EVITAR - N+1 queries
const deals = await dealService.find(query)  // 1 query

for (const deal of deals) {
  const contract = await contractService.get(deal.contract)  // N queries
  // ...
}
```

**Resultado:** 1 + N queries (se N = 100 deals, são 101 queries!)

#### Solução: DataLoader

**Arquivo:** `packages/api/src/infra/service.ts`

```typescript
dataLoader = new DataLoader<string, T>(this.findByKeys.bind(this))

async findByKeys<T>(keys: readonly string[]): Promise<T[]> {
  const query = aql`
    FOR doc IN ${this.aql}
      FILTER doc._key IN ${keys}
    RETURN doc
  `
  const data = (await this.query(query)) as T[]
  const resultMap = new Map<string, T>(
    data.map((document: T) => [
      (document as ArangoObject)._key,
      document,
    ]) as [string, T][],
  )
  return keys.map(key => resultMap.get(key) ?? ({} as T)) as T[]
}
```

#### Uso

```typescript
// ✅ USAR - 2 queries (1 para deals, 1 para todos os contracts)
const deals = await dealService.find(query)  // 1 query

const contracts = await Promise.all(
  deals.map(deal => contractService.dataLoader.load(deal.contract))
)  // 1 query com IN
```

**Resultado:** 2 queries independente de N!

### 11.2 Redis Cache

#### Cache Automático em Métodos de Query

**Service base implementa cache automático:**

```typescript
async get<R extends T>(key: string | undefined): Promise<R | undefined> {
  const cachedResponse = await this.getCached(key)

  if (isNotEmptyValue(cachedResponse)) {
    return JSON.parse(cachedResponse) as R
  }

  const data = await this.database.get(this.collection, key)

  if (isNotEmptyValue(data)) {
    await this.setCache(key, JSON.stringify(data))
  }

  return data as R
}
```

#### TTL Configurável

**Padrão:** 5 minutos

```typescript
await redis.set(cacheKey, JSON.stringify(data), 'EX', 300)  // 300 segundos = 5 minutos
```

#### Invalidação em Updates

**Cache é automaticamente invalidado em create/update:**

```typescript
async create<T>(payload: Partial<T>): Promise<CreateUpdateResponse<T>> {
  const response = await this.database.insertOne<T>(this.collection, payload)

  this.dataLoader.clearAll()
  await this.clearCache()  // ✅ Invalida cache

  return response
}
```

#### Controller-Level Caching

Para queries customizadas, use cache em nível de controller:

```typescript
async searchDeals(filter: DealFilter, user: string): Promise<Deal[]> {
  const cacheKey = `deals:search:${JSON.stringify(filter)}:${user}`

  const cached = await redis.get(cacheKey)
  if (cached) {
    return JSON.parse(cached)
  }

  const deals = await this.search(filter, user)
  await redis.set(cacheKey, JSON.stringify(deals), 'EX', 300)

  return deals
}
```

### 11.3 Query Optimization

#### LIMIT para Restringir Resultados

**Sempre use LIMIT em queries de listagem:**

```typescript
const q = aql`
  FOR d IN ${this.aql}
    FILTER d.workspace == ${workspace}
    SORT d.createdAt DESC
    LIMIT ${offset}, ${limit}  // ✅ LIMIT obrigatório
  RETURN d
`
```

#### Explain para Analisar Planos de Execução

```typescript
const query = aql`
  FOR d IN ${this.aql}
    FILTER d.workspace == ${workspace}
    FILTER d.archived == false
  RETURN d
`

const plan = await database.connection.query({
  query: query.query,
  bindVars: query.bindVars,
  options: { profile: 2 }
})

console.log(plan.getExtra())
```

**Procure por:**

- **`EnumerateCollectionNode`**: Full collection scan (lento!)
- **`IndexNode`**: Usa índice (rápido!)

#### Evitar Full Collection Scans

**❌ Full collection scan:**

```typescript
FOR d IN deals
  FILTER d.workspace == ${workspace}  // Sem índice
RETURN d
```

**✅ Index scan:**

```typescript
FOR d IN deals
  FILTER d.workspace == ${workspace}  // Com índice idx_deals_workspace
RETURN d
```

### 11.4 Helpers da Service Base

#### buildSort() - Ordenação Determinística

```typescript
buildSort(sort: Sort[] | undefined): GeneratedAqlQuery {
  if (!sort || sort.length === 0) {
    return aql`SORT doc.createdAt DESC, doc._key DESC`
  }

  const sortParts: GeneratedAqlQuery[] = []
  for (const s of sort) {
    sortParts.push(aql`${literal(`doc.${s.field}`)} ${literal(s.order)}`)
  }
  // Adiciona _key para ordenação determinística
  sortParts.push(aql`${literal('doc._key')} DESC`)

  return aql`SORT ${join(sortParts, ', ')}`
}
```

**Por que adicionar `_key`?**

- Garante **ordenação determinística** (mesma ordem a cada query)
- Evita resultados aleatórios em paginação

#### buildLimit() - Paginação

```typescript
buildLimit(limit?: number, offset?: number): GeneratedAqlQuery {
  if (limit === undefined || limit === 0) {
    return aql`LIMIT 100`  // Limite padrão
  }

  if (offset !== undefined && offset > 0) {
    return aql`LIMIT ${offset}, ${limit}`
  }

  return aql`LIMIT ${limit}`
}
```

#### getDatesFilterString() - Filtros de Data

```typescript
getDatesFilterString(
  filter: DateFilter,
  field: string,
): GeneratedAqlQuery {
  const filters: GeneratedAqlQuery[] = []

  if (filter.startDate) {
    filters.push(aql`${literal(field)} >= ${filter.startDate}`)
  }

  if (filter.endDate) {
    filters.push(aql`${literal(field)} <= ${filter.endDate}`)
  }

  return filters.length > 0 ? aql`FILTER ${join(filters, ' FILTER ')}` : aql``
}
```

---

## Error Handling

### 12.1 Retry Logic

**Arquivo:** `packages/api/src/infra/database.ts`

#### Lock Errors (18, 28, 29)

Ocorrem quando dois writes simultâneos tentam modificar o mesmo documento:

```typescript
const lockErroNums = [18, 28, 29]

if (error_.errorNum && lockErroNums.includes(error_.errorNum) && retryCount < 3) {
  Logger.warn(`Retrying query (attempt ${retryCount + 1}/3)`)
  await new Promise(resolve =>
    setTimeout(resolve, 1000 * (retryCount + 1)),
  )
  return this.query(query, retryCount + 1)
}
```

**Erros:**

- **18**: Write conflict (document locked)
- **28**: Transaction conflict
- **29**: Deadlock

#### Connection Errors (1200, 1204, 1205)

Perda de conexão temporária com ArangoDB:

```typescript
const connectionErroNums = [1200, 1204, 1205]

if (error_.errorNum && connectionErroNums.includes(error_.errorNum) && retryCount < 3) {
  Logger.warn(`Retrying query (attempt ${retryCount + 1}/3)`)
  await new Promise(resolve =>
    setTimeout(resolve, 1000 * (retryCount + 1)),
  )
  return this.query(query, retryCount + 1)
}
```

**Erros:**

- **1200**: HTTP connection error
- **1204**: No connection available
- **1205**: Connection timeout

#### Socket Errors

Network issues como `ECONNRESET` ou `socket hang up`:

```typescript
if (
  (error_.message.includes('socket hang up') ||
    error_.message.includes('ECONNRESET')) &&
  retryCount < 2
) {
  Logger.warn(
    `Socket hang up detected, retrying query (attempt ${retryCount + 1}/2)`,
  )
  await new Promise(resolve =>
    setTimeout(resolve, 2000 * (retryCount + 1)),
  )
  return this.query(query, retryCount + 1)
}
```

#### Backoff Exponencial

```typescript
await new Promise(resolve =>
  setTimeout(resolve, 1000 * (retryCount + 1)),
)
```

**Delays:**

- **Tentativa 1**: 1 segundo
- **Tentativa 2**: 2 segundos
- **Tentativa 3**: 3 segundos

**Por que progressivo?**

- Evita sobrecarga no server
- Dá tempo para recursos liberarem

#### Máximo de 3 Tentativas

```typescript
if (retryCount < 3) {
  return this.query(query, retryCount + 1)
}

throw error_  // Re-throw após retries esgotados
```

### 12.2 Error Types

#### ArangoError com errorNum

```typescript
catch (error: unknown) {
  const error_ = error as ArangoError

  if (error_.errorNum === 1202) {
    // Document not found
    return undefined
  }

  if (error_.errorNum === 18) {
    // Lock error - retry
    return this.query(query, retryCount + 1)
  }

  throw error_
}
```

#### Logging Estruturado

```typescript
Logger.error('ArangoDB Query Error', {
  errorNum: error_.errorNum,
  message: error_.message,
  query: query.query,
  bindVars: query.bindVars,
  retryCount,
})
```

#### Re-throw Após Retries Esgotados

```typescript
if (retryCount >= 3) {
  Logger.error('Max retries exceeded', { error_, query })
  throw error_  // ✅ Re-throw para camadas superiores tratarem
}
```

---

## Collection Initialization

**Arquivo:** `packages/api/src/infra/arangodb/collections-init.ts`

### Script de Inicialização

```typescript
export async function ensureCollections(): Promise<void> {
  const db = database
  const existingCollections = await db.connection.listCollections()
  const collectionNames = new Set(existingCollections.map(c => c.name))

  // Verificar e criar collection
  if (!collectionNames.has('seniorities')) {
    await db.connection.createCollection('seniorities')
    Logger.info('Collection "seniorities" created')
  }

  // Criar índice
  await db.ensureIndex('seniorities', {
    type: 'persistent',
    fields: ['name'],
    unique: true,
  })
}
```

### Verificar Collections Existentes

```typescript
const existingCollections = await db.connection.listCollections()
const collectionNames = new Set(existingCollections.map(c => c.name))

if (!collectionNames.has('my_collection')) {
  await db.connection.createCollection('my_collection')
}
```

### Criar Collections Faltantes

```typescript
for (const collectionName of requiredCollections) {
  if (!collectionNames.has(collectionName)) {
    await db.connection.createCollection(collectionName)
    Logger.info(`Collection "${collectionName}" created`)
  }
}
```

### Criar Índices Iniciais

```typescript
await db.ensureIndex('deals', {
  type: 'persistent',
  name: 'idx_deals_workspace',
  fields: ['workspace'],
  cacheEnabled: true,
})

await db.ensureIndex('deals', {
  type: 'persistent',
  name: 'idx_deals_assignees',
  fields: ['assignees[*]'],
  cacheEnabled: true,
})
```

---

## Best Practices - Checklist

### ✅ Queries

- [ ] Usar `aql` tag sempre (nunca string concatenation)
- [ ] Construir filtros incrementalmente com array + `join()`
- [ ] Usar `join()` para combinar filtros
- [ ] Adicionar `_key` ao sort para ordenação determinística
- [ ] Usar `LIMIT` em todas as queries de listagem
- [ ] Preferir `SEARCH` a `FILTER` para full-text search
- [ ] Usar `BM25()` para ordenação por relevância
- [ ] Usar `BOOST()` para ponderar campos

### ✅ Indexing

- [ ] Ordem de campos: igualdades → ranges → sort
- [ ] Naming convention: `idx_{collection}_{fields}`
- [ ] Cache enabled para queries frequentes (`cacheEnabled: true`)
- [ ] Covering indexes quando aplicável (`storedValues`)
- [ ] Sparse indexes para campos opcionais
- [ ] Array indexes com `field[*]` sintaxe
- [ ] Verificar índices existentes antes de criar

### ✅ Services

- [ ] Extender `Service<T>` base class
- [ ] Usar `this.aql` para referência à collection
- [ ] Emitir eventos em create/update (use `dontEmit: true` para evitar)
- [ ] Archive ao invés de delete físico
- [ ] Implementar cache em queries customizadas
- [ ] Usar DataLoader para batch loading
- [ ] Usar singleton pattern para services

### ✅ Performance

- [ ] DataLoader para relacionamentos (evitar N+1 queries)
- [ ] Redis cache em queries frequentes
- [ ] Índices apropriados para queries
- [ ] LIMIT em queries de listagem
- [ ] Evitar N+1 queries
- [ ] Usar `buildSort()`, `buildLimit()`, `getDatesFilterString()` helpers

### ✅ Error Handling

- [ ] Retry logic implementado para lock errors (18, 28, 29)
- [ ] Retry logic para connection errors (1200, 1204, 1205)
- [ ] Retry logic para socket errors (hang up, ECONNRESET)
- [ ] Logging estruturado com contexto
- [ ] Re-throw após retries esgotados
- [ ] Tratamento de erro 1202 (document not found)

### ✅ Naming Conventions

- [ ] Collections: `lowercase_snake_case` (plural)
- [ ] Índices: `idx_{collection}_{fields}`
- [ ] Campos: `camelCase`
- [ ] IDs: `{collection}_{nanoId()}`
- [ ] Edge collections: `{entity1}_{entity2}_edge`

### ✅ Data Model

- [ ] Todos os documentos implementam `ArangoObject`
- [ ] Campos obrigatórios: `_key`, `createdAt`, `updatedAt`
- [ ] Campo `workspace` para multi-tenancy
- [ ] Soft delete com `archived`, `archivedAt`, `archivedBy`
- [ ] Timestamps em ISO 8601 UTC
- [ ] Auditoria com `createdBy`, `archivedBy`

---

## Anti-Patterns - O Que Evitar

### ❌ Queries

- **String concatenation**: NUNCA use string concatenation para queries
  ```typescript
  // ❌ INCORRETO
  const query = `FOR doc IN ${collectionName} FILTER doc._key == "${key}" RETURN doc`

  // ✅ CORRETO
  const query = aql`FOR doc IN ${this.aql} FILTER doc._key == ${key} RETURN doc`
  ```

- **Full collection scans sem filtros**:
  ```typescript
  // ❌ INCORRETO
  FOR d IN deals RETURN d

  // ✅ CORRETO
  FOR d IN deals
    FILTER d.workspace == ${workspace}
    LIMIT 100
  RETURN d
  ```

- **Queries sem LIMIT**:
  ```typescript
  // ❌ INCORRETO
  FOR d IN deals FILTER d.workspace == ${workspace} RETURN d

  // ✅ CORRETO
  FOR d IN deals
    FILTER d.workspace == ${workspace}
    LIMIT 100
  RETURN d
  ```

- **Ordenação sem `_key` (não determinística)**:
  ```typescript
  // ❌ INCORRETO
  SORT d.createdAt DESC

  // ✅ CORRETO
  SORT d.createdAt DESC, d._key DESC
  ```

- **Nested template literals**:
  ```typescript
  // ❌ INCORRETO
  const query = aql`FOR doc IN ${`collection_${name}`}`

  // ✅ CORRETO
  const collectionName = `collection_${name}`
  const query = aql`FOR doc IN ${collectionName}`
  ```

### ❌ Indexing

- **Índices com ordem errada de campos**:
  ```typescript
  // ❌ INCORRETO
  fields: ['wonAt', 'workspace', 'archived']

  // ✅ CORRETO
  fields: ['workspace', 'archived', 'wonAt']
  ```

- **Índices sem naming convention**:
  ```typescript
  // ❌ INCORRETO
  name: 'deals_workspace_index'

  // ✅ CORRETO
  name: 'idx_deals_workspace'
  ```

- **Índices duplicados**:
  ```typescript
  // ❌ INCORRETO - Verificar antes de criar
  await col.ensureIndex({ name: 'idx_deals_workspace', fields: ['workspace'] })
  await col.ensureIndex({ name: 'idx_deals_ws', fields: ['workspace'] })

  // ✅ CORRETO
  const existing = await col.indexes()
  if (!existing.some(idx => idx.name === 'idx_deals_workspace')) {
    await col.ensureIndex({ name: 'idx_deals_workspace', fields: ['workspace'] })
  }
  ```

- **Cache disabled em queries frequentes**:
  ```typescript
  // ❌ INCORRETO
  { name: 'idx_deals_workspace', fields: ['workspace'], cacheEnabled: false }

  // ✅ CORRETO
  { name: 'idx_deals_workspace', fields: ['workspace'], cacheEnabled: true }
  ```

### ❌ Services

- **CRUD manual ao invés de usar Service base**:
  ```typescript
  // ❌ INCORRETO
  const deal = await database.connection.collection('deals').document(key)

  // ✅ CORRETO
  const deal = await dealService.get(key)
  ```

- **Delete físico ao invés de archive**:
  ```typescript
  // ❌ INCORRETO
  await dealService.delete(key)

  // ✅ CORRETO
  await dealService.archive(key, user)
  ```

- **Não emitir eventos em updates**:
  ```typescript
  // ❌ INCORRETO
  await database.connection.collection('deals').update(key, payload)

  // ✅ CORRETO
  await dealService.update(key, payload) // Emite evento automaticamente
  ```

- **Ignorar cache**:
  ```typescript
  // ❌ INCORRETO
  const data = await this.database.query(query) // Sem cache

  // ✅ CORRETO
  const data = await this.find(query) // Com cache
  ```

### ❌ Performance

- **N+1 queries sem DataLoader**:
  ```typescript
  // ❌ INCORRETO
  for (const deal of deals) {
    const contract = await contractService.get(deal.contract)
  }

  // ✅ CORRETO
  const contracts = await Promise.all(
    deals.map(deal => contractService.dataLoader.load(deal.contract))
  )
  ```

- **Queries sem índices apropriados**:
  ```typescript
  // ❌ INCORRETO - Sem índice
  FOR d IN deals FILTER d.customField == ${value} RETURN d

  // ✅ CORRETO - Criar índice primeiro
  await col.ensureIndex({ name: 'idx_deals_customField', fields: ['customField'] })
  ```

- **Materialização desnecessária de documentos**:
  ```typescript
  // ❌ INCORRETO
  FOR d IN deals
    FILTER d.workspace == ${workspace}
  RETURN d // Retorna documento completo

  // ✅ CORRETO - Covering index
  FOR d IN deals
    FILTER d.workspace == ${workspace}
  RETURN { _key: d._key, name: d.name } // Usa storedValues
  ```

- **Ignorar covering indexes**:
  ```typescript
  // ❌ INCORRETO
  { name: 'idx_deals_workspace', fields: ['workspace'] }

  // ✅ CORRETO - Covering index para queries frequentes
  {
    name: 'idx_deals_workspace',
    fields: ['workspace'],
    storedValues: ['name', 'status', 'archived']
  }
  ```

---

## Exemplos Práticos Completos

### Exemplo 1: Service Completo

**DealService** com todos os métodos, queries complexas, filtros dinâmicos, agregações.

**Arquivo:** `packages/api/src/deals/service.ts`

```typescript
export class DealService extends Service<Deal> {
  private static instance: DealService | undefined

  public constructor() {
    super()
    this.collection = 'deals'
    this.model = 'deal'
    this.dataLoader = new DataLoader<string, Deal>(this.findByKeys.bind(this))
  }

  public static getInstance(): DealService {
    DealService.instance ??= new DealService()
    return DealService.instance
  }

  // Query complexa com fallback Elasticsearch → ArangoDB
  async search(
    filter: Partial<DealFilter>,
    user: string | undefined,
  ): Promise<Deal[]> {
    let esAvailable = false
    try {
      esAvailable = await elasticService.isAvailable()
    } catch (error) {
      Logger.warn('Falha ao verificar disponibilidade do Elasticsearch', error)
    }

    if (esAvailable) {
      try {
        return await dealElasticService.searchDeals(filter, user)
      } catch (error) {
        Logger.warn('Elasticsearch error, falling back to ArangoDB', { error })
      }
    }

    return this.searchWithArango(filter, user)
  }

  // Full-text search com ArangoSearch
  searchByName(filter: Partial<DealFilter>, user: string): Promise<Deal[]> {
    const filters = this.buildDealFilters(filter, user)
    const limit = this.buildLimit(filter.limit, filter.offset)

    const q = aql`
      FOR d IN deals_search
        SEARCH (
          BOOST(ANALYZER(d.name IN TOKENS(${filter.name}, "maia::pt_br_text_search"), "maia::pt_br_text_search"), 2.0) OR
          ANALYZER(d.organizationChallenges IN TOKENS(${filter.name}, "maia::pt_br_text_search"), "maia::pt_br_text_search")
        )
        ${filters.length > 0 ? aql`FILTER ${join(filters, ' FILTER ')}` : aql``}
        SORT BM25(d) DESC
        ${limit}
      RETURN d
    `
    return this.query<Deal>(q)
  }

  // Agregação com COLLECT
  countDealsByStatus(
    filter: Partial<DealFilter>,
    user: string,
  ): Promise<{ status: string; count: number; amount: number }[][]> {
    const filters = this.buildDealFilters(filter, user)

    const q = aql`
      FOR d IN ${this.aql}
        ${filters.length > 0 ? aql`FILTER ${join(filters, ' FILTER ')}` : aql``}
        COLLECT status = d.status
        AGGREGATE total = COUNT(d), amount = SUM(d.value)
      RETURN { status, count: total, amount }
    `

    return this.query<{ status: string; count: number; amount: number }[]>(q)
  }

  // Construção de filtros dinâmicos
  buildDealFilters(
    filter: Partial<DealFilter>,
    user: string,
  ): GeneratedAqlQuery[] {
    const filters: GeneratedAqlQuery[] = []

    this.addWorkspaceFilters(filters, filter)
    this.addAssigneeFilters(filters, filter, user)
    this.addBasicPropertyFilters(filters, filter)

    return filters
  }

  private addWorkspaceFilters(
    filters: GeneratedAqlQuery[],
    filter: Partial<DealFilter>,
  ): void {
    if (filter.workspace !== undefined) {
      filters.push(aql`d.workspace == ${filter.workspace}`)
    }
  }

  private addAssigneeFilters(
    filters: GeneratedAqlQuery[],
    filter: Partial<DealFilter>,
    user: string,
  ): void {
    if (filter.assignees !== undefined && filter.assignees.length > 0) {
      if (filter.assignees.includes('$me') && user) {
        filter.assignees = filter.assignees.map(assignee =>
          assignee === '$me' ? user : assignee,
        )
      }
      const arrAqls: GeneratedAqlQuery[] = []
      for (const [index, assignee] of filter.assignees.entries()) {
        if (index === 0) {
          arrAqls.push(aql`${assignee} IN d.assignees`)
        } else {
          arrAqls.push(aql`OR ${assignee} IN d.assignees`)
        }
      }
      filters.push(aql`${join(arrAqls, ' OR ')}`)
    }
  }
}

export const dealService = DealService.getInstance()
```

### Exemplo 2: Full-Text Search

**Criar analyzer, criar view, query com SEARCH e BOOST, ordenação por relevância.**

#### Passo 1: Criar Analyzer

```typescript
await db.connection.createAnalyzer('maia::pt_br_text_search', {
  type: 'text',
  properties: {
    locale: 'pt_BR',
    case: 'lower',
    accent: false,
    stemming: true,
    edgeNgram: {
      min: 1,
      max: 15,
      preserveOriginal: true,
    },
  },
  features: ['frequency', 'position', 'norm'],
})
```

#### Passo 2: Criar View

```typescript
await db.createView('deals_search', {
  type: 'arangosearch',
  primarySort: [{ field: 'createdAt', direction: 'desc' }],
  links: {
    deals: {
      fields: {
        name: { analyzers: ['maia::pt_br_text_search'] },
        organizationChallenges: { analyzers: ['maia::pt_br_text_search'] },
      },
      includeAllFields: false,
      storeValues: 'none',
      trackListPositions: false,
    },
  },
})
```

#### Passo 3: Query com SEARCH e BOOST

```typescript
const q = aql`
  FOR d IN deals_search
    SEARCH (
      BOOST(ANALYZER(d.name IN TOKENS(${searchTerm}, "maia::pt_br_text_search"), "maia::pt_br_text_search"), 2.0) OR
      ANALYZER(d.organizationChallenges IN TOKENS(${searchTerm}, "maia::pt_br_text_search"), "maia::pt_br_text_search")
    )
    SORT BM25(d) DESC
    LIMIT 10
  RETURN d
`
```

### Exemplo 3: Graph Operations

**Criar edge collection, update edges com transaction, query de traversal, diff pattern.**

#### Passo 1: Criar Edge Collection

```typescript
await db.connection.createCollection('user_tasks_edge', {
  type: 3, // Edge collection
})

await db.connection.collection('user_tasks_edge').ensureIndex({
  type: 'persistent',
  fields: ['_from'],
  unique: false,
})

await db.connection.collection('user_tasks_edge').ensureIndex({
  type: 'persistent',
  fields: ['_to'],
  unique: false,
})
```

#### Passo 2: Update Edges com Transaction (Diff Pattern)

```typescript
const edgeTaskKey = 'tasks/' + current._key

// 1. Buscar assignees atuais
const currentAssignees: string[] = await database.connection
  .query(
    aql`
      FOR e IN user_tasks_edge
        FILTER e._to == ${edgeTaskKey}
        RETURN PARSE_IDENTIFIER(e._from).key
    `,
  )
  .then(c => c.all())

// 2. Diff
const toAdd = current.assignees?.filter(u => !currentAssignees.includes(u)) ?? []
const toRemove = currentAssignees.filter(
  u => (current.assignees && !current.assignees.includes(u)) ?? false,
)

// 3. Transaction
const trx: Transaction = await database.connection.beginTransaction({
  write: ['user_tasks_edge'],
})

for (const userKey of toAdd) {
  await trx.step(async () => {
    const fromVertex = `users/${userKey}`
    const toVertex = edgeTaskKey
    return database.connection
      .collection('user_tasks_edge')
      .save({ _from: fromVertex, _to: toVertex })
  })
}

if (toRemove.length > 0) {
  await trx.step(() =>
    database.connection.query(aql`
      FOR e IN user_tasks_edge
        FILTER e._to == ${edgeTaskKey}
          AND PARSE_IDENTIFIER(e._from).key IN ${toRemove}
        REMOVE e IN user_tasks_edge
    `),
  )
}

await trx.commit()
```

#### Passo 3: Query de Traversal

```typescript
// Buscar tarefas de um usuário
const q = aql`
  FOR e IN user_tasks_edge
    FILTER e._from == ${'users/' + userKey}
    LET task = DOCUMENT(e._to)
    FILTER task != null
    FILTER task.archived != true
  RETURN task
`
```

### Exemplo 4: Agregação Complexa

**Múltiplos LETs, subqueries, COLLECT AGGREGATE, cálculos com documentos relacionados.**

**Arquivo:** `packages/api/src/projects/service.ts`

```typescript
buildContractFilters(filter: Partial<ProjectFilter>): GeneratedAqlQuery {
  const minAmount = filter.minAmount ?? null
  const maxAmount = filter.maxAmount ?? null

  if (minAmount !== null || maxAmount !== null) {
    return aql`
      LET contract = DOCUMENT("contracts", d.contract)
      FILTER contract != null

      LET contractValue = SUM(
        FOR it IN contract.items || []
          RETURN it.amount * it.quantity
      )

      LET pp = contract.paymentPlan || null
      LET couponSource = pp != null && pp.subscription != null
        ? pp.subscription.coupons
        : contract.coupons

      LET couponPercent = SUM(FOR c IN couponSource || [] RETURN c.percentOff)
      LET couponAmount = SUM(FOR c IN couponSource || [] RETURN c.amountOff)

      LET valorContrato = contractValue * (100 - couponPercent) / 100 - couponAmount

      ${minAmount === null ? aql`` : aql`FILTER valorContrato >= ${minAmount}`}
      ${maxAmount === null ? aql`` : aql`FILTER valorContrato <= ${maxAmount}`}
    `
  }

  return aql``
}
```

**Como funciona:**

1. **`LET contract = DOCUMENT(...)`**: Busca documento relacionado
2. **`LET contractValue = SUM(...)`**: Subquery para somar valores de items
3. **`LET couponSource = ... ? ... : ...`**: Condicional para determinar origem de cupons
4. **`LET couponPercent = SUM(...)`**: Subquery para somar percentuais de desconto
5. **`LET valorContrato = ...`**: Cálculo final com aplicação de descontos
6. **`FILTER valorContrato >= ...`**: Filtro condicional por range de valores

---

## Troubleshooting

### Query Lenta - Usar Explain

**Problema:** Query demorando muito tempo.

**Solução:** Use `explain` para analisar plano de execução:

```typescript
const query = aql`
  FOR d IN ${this.aql}
    FILTER d.workspace == ${workspace}
    FILTER d.archived == false
  RETURN d
`

const plan = await database.connection.query({
  query: query.query,
  bindVars: query.bindVars,
  options: { profile: 2 }
})

console.log(plan.getExtra())
```

**Procure por:**

- **`EnumerateCollectionNode`**: Full collection scan (adicionar índice!)
- **`IndexNode`**: Usando índice (bom!)
- **`estimatedNrItems`**: Número estimado de documentos processados
- **`executionTime`**: Tempo de execução

**Fix:** Criar índice apropriado.

### Lock Errors - Aumentar Retries

**Problema:** Erro 18, 28 ou 29 (lock errors).

**Solução:** Aumentar número de retries ou delay:

```typescript
public query = async <T>(
  query: GeneratedAqlQuery,
  retryCount = 0,
): Promise<T[]> => {
  try {
    return await connection.query(query).then(c => c.all())
  } catch (error: unknown) {
    const error_ = error as ArangoError
    const lockErroNums = [18, 28, 29]

    if (error_.errorNum && lockErroNums.includes(error_.errorNum) && retryCount < 5) {
      Logger.warn(`Retrying query (attempt ${retryCount + 1}/5)`)
      await new Promise(resolve => setTimeout(resolve, 2000 * (retryCount + 1)))
      return this.query(query, retryCount + 1)
    }

    throw error_
  }
}
```

**Mudanças:**

- **Retries:** 3 → 5
- **Delay:** 1s, 2s, 3s → 2s, 4s, 6s, 8s, 10s

### Connection Errors - Verificar Pool

**Problema:** Erro 1200, 1204 ou 1205 (connection errors).

**Solução:** Verificar configuração do pool:

```typescript
agentOptions: {
  keepAlive: true,
  timeout: 120_000,      // Aumentar se queries longas
  maxSockets: 50,        // Aumentar se alta concorrência
  maxFreeSockets: 10,    // Aumentar para reutilização
}
```

### Índice Não Usado - Verificar Ordem de Campos

**Problema:** Query lenta mesmo com índice.

**Solução:** Verificar ordem de campos no índice vs query:

```typescript
// Query
FILTER d.workspace == ${workspace}
FILTER d.archived == false
FILTER d.wonAt >= ${startDate}

// Índice INCORRETO
fields: ['wonAt', 'workspace', 'archived']

// Índice CORRETO
fields: ['workspace', 'archived', 'wonAt']
```

**Regra:** Igualdades → Ranges → Sort

### Full Collection Scan - Criar Índice Apropriado

**Problema:** `EnumerateCollectionNode` no explain.

**Solução:** Criar índice para os campos filtrados:

```typescript
await col.ensureIndex({
  type: 'persistent',
  name: 'idx_deals_workspace_archived',
  fields: ['workspace', 'archived'],
  cacheEnabled: true,
})
```

**Depois de criar índice, rodar query novamente e verificar explain.**

---

## Referências

### Documentação Oficial

- **Documentação Oficial:** [https://docs.arangodb.com/](https://docs.arangodb.com/)
- **AQL Tutorial:** [https://docs.arangodb.com/stable/aql/tutorial/](https://docs.arangodb.com/stable/aql/tutorial/)
- **Indexing:** [https://docs.arangodb.com/stable/index-and-search/indexing/](https://docs.arangodb.com/stable/index-and-search/indexing/)
- **ArangoSearch:** [https://docs.arangodb.com/stable/index-and-search/arangosearch/](https://docs.arangodb.com/stable/index-and-search/arangosearch/)
- **Graph Queries:** [https://docs.arangodb.com/stable/aql/graphs/](https://docs.arangodb.com/stable/aql/graphs/)
- **arangojs Documentation:** [https://arangodb.github.io/arangojs/](https://arangodb.github.io/arangojs/)

### Documentos Relacionados

- [CLAUDE.md](../../CLAUDE.md) - Handbook completo da Maia API
- [TypeScript Guide](./typescript.md) - Guia de TypeScript da Berry
- [Testing Guide](../quality/automated-testing.md) - Guia de testes automatizados

### Arquivos de Referência no Codebase

#### Configuração
- `packages/api/src/infra/database.ts` - Conexão e retry logic
- `packages/api/src/infra/collections.ts` - Definições de collections

#### Base Classes
- `packages/api/src/infra/service.ts` - Service base class completo

#### Inicialização
- `packages/api/src/infra/arangodb/collections-init.ts` - Criação de collections
- `packages/api/src/infra/arangodb/analyzers.ts` - Analyzers customizados
- `packages/api/src/infra/arangodb/views.ts` - Views de full-text search

#### Indexes
- `packages/api/src/scripts/indexes/add-new-indexes.ts` - Criação de índices

#### Exemplos Práticos
- `packages/api/src/deals/service.ts` - Queries complexas, search, agregações
- `packages/api/src/users/service.ts` - Full-text search simples
- `packages/api/src/projects/service.ts` - Cálculos complexos, subqueries
- `packages/api/src/tasks/cases/assignee-edge.ts` - Graph operations, transactions

---

**Fim do Documento**

Este guia documenta e padroniza a forma como a Berry/Maia API utiliza ArangoDB. Para dúvidas ou sugestões, consulte o Tech Lead ou abra uma issue no repositório.
