# Arquitetura e Tecnologia - BerryMax

**Versão:** 1.0.0  
**Última atualização:** 01 de Dezembro de 2025  
**Objetivo:** Documentar a arquitetura do sistema BerryMax de forma concisa, focando nos conceitos principais e padrões arquiteturais.

---

## Índice

1. [Introdução](#introdução)
2. [Visão Geral da Plataforma](#visão-geral-da-plataforma)
3. [Arquitetura Multi-Tenant](#arquitetura-multi-tenant)
4. [Event-Driven Architecture](#event-driven-architecture)
5. [Integrações Externas](#integrações-externas)
6. [Arquitetura de Módulos](#arquitetura-de-módulos)
7. [Infraestrutura e Deploy](#infraestrutura-e-deploy)
8. [Segurança](#segurança)
9. [Performance e Escalabilidade](#performance-e-escalabilidade)
10. [Best Practices](#best-practices)
11. [Anti-Patterns](#anti-patterns)
12. [Troubleshooting](#troubleshooting)
13. [Referências](#referências)

---

## Introdução

A **BerryMax** é uma plataforma B2B de consultoria que automatiza todo o ciclo de vida do cliente, desde a geração de leads até a entrega de projetos. A plataforma utiliza uma arquitetura **multi-tenant** com **event-driven architecture**, permitindo que múltiplos workspaces (franchiser e franchisees) operem de forma isolada enquanto compartilham a mesma infraestrutura.

### Stack Tecnológico

- **Backend**: Node.js 24+, TypeScript, Fastify, Apollo GraphQL Server
- **Frontend**: React 19, Vite, TanStack Router, Legend State
- **Databases**: ArangoDB (multi-model), Redis (cache/queue), Elasticsearch (search), Qdrant (vectors)
- **Infraestrutura**: Docker Swarm, Nginx, BullMQ, WebSockets
- **Integrações**: Stripe, Google Workspace, WhatsApp Business API

### Arquitetura de Alto Nível

```
┌─────────────┐
│   Frontend  │ (React + Vite)
└──────┬──────┘
       │ GraphQL / REST
┌──────▼──────┐
│  Maia API   │ (Fastify + GraphQL)
└──────┬──────┘
       │
   ┌───┴───┬──────────┬──────────┐
   │       │          │          │
┌──▼──┐ ┌─▼───┐  ┌───▼──┐  ┌───▼──┐
│Arango│ │Redis│  │Elastic│ │Qdrant│
│  DB  │ │Queue│  │Search │ │Vector│
└──────┘ └─────┘  └───────┘ └──────┘
```

---

## Visão Geral da Plataforma

### Componentes Principais

#### Backend (Maia API)

A API principal é construída com **Fastify** e **Apollo GraphQL Server**, fornecendo uma interface GraphQL para a maioria das operações. A arquitetura segue **Domain-Driven Design** com módulos auto-contidos (`deals`, `projects`, `users`, `whats`, etc.).

**Características principais:**
- GraphQL como API principal (`/graphql`)
- REST endpoints para webhooks e uploads
- WebSocket para comunicação em tempo real
- Event-driven architecture com BullMQ

#### Frontend (Maia App)

Aplicação React moderna construída com **Vite**, utilizando **TanStack Router** para roteamento e **Legend State** para gerenciamento de estado reativo.

**Características principais:**
- TypeScript strict mode
- Componentes funcionais com `observer()`
- TanStack Query para data fetching
- Design system com Tailwind CSS e shadcn/ui

#### Infraestrutura

A infraestrutura utiliza **Docker Swarm** para orquestração, com serviços containerizados para API, Frontend, e todos os bancos de dados. **Nginx** atua como reverse proxy e load balancer.

### Fluxo de Dados

#### Request Flow

```
Frontend → GraphQL Query/Mutation → Fastify → Apollo Server
    ↓
Resolver → Controller → Service → ArangoDB
    ↓
Response ← Cache (Redis) ← Data
```

#### Event Flow

```
Database Change (create/update) → Service.emit()
    ↓
document_updated Event → BullMQ Queue
    ↓
Queue Worker → Filter Listeners → Execute Listeners
    ↓
Background Jobs → External APIs / Notifications
```

### Modelo de Negócio

A BerryMax opera com dois tipos de workspaces:

- **Franchiser Workspace**: Workspace central da Berry Consultoria (`FRANCHISER_WORKSPACE`) que gera leads e gerencia o sistema de leilões.
- **Franchisee Workspaces**: Unidades de consultoria individuais que fazem lances em leads, gerenciam CRM e projetos.

**Revenue Model:**
- **CAC (Customer Acquisition Cost)**: Franchisees pagam pelo deal fechado pelo franchiser
- **Application Fee**: Taxa de 24% dos franchisees
- **Subscription Revenue**: Receita recorrente mensal de contratos de clientes

---

## Arquitetura Multi-Tenant

### Conceito

A BerryMax utiliza **workspace** como unidade fundamental de isolamento multi-tenant. Cada workspace representa uma organização (franchiser ou franchisee) com dados completamente isolados. O isolamento é garantido em múltiplas camadas: banco de dados, autenticação, autorização e cache.

**Tipos de Workspace:**
- `franchiser`: Workspace central da Berry Consultoria
- `franchisee`: Workspaces de unidades de consultoria

### Isolamento de Dados

Todos os documentos multi-tenant incluem obrigatoriamente o campo `workspace`:

```typescript
export interface Deal extends ArangoObject {
  _key: string
  workspace: string // OBRIGATÓRIO
  name: string
  status: string
  // ...
}
```

Os **Services** aplicam filtros automáticos por workspace em todas as queries:

```typescript
buildDealFilters(filter: Partial<DealFilter>, user: string): GeneratedAqlQuery[] {
  const filters: GeneratedAqlQuery[] = []
  
  // Filtro automático por workspace do usuário
  if (filter.workspace !== undefined) {
    filters.push(aql`d.workspace == ${filter.workspace}`)
  }
  
  return filters
}
```

### Autenticação e Autorização

O **context** é criado por requisição e inclui `user`, `workspace` e `locale`:

**Arquivo:** `packages/api/src/infra/context.ts`

```typescript
export const createAppContext = async (
  req: FastifyRequest,
): Promise<GraphqlContext> => {
  const token = extractToken(req.headers.authorization)
  const user = await userFromToken(token)
  const workspace = await workspaceService().get(user.workspace)
  
  return {
    user: user._key,
    workspace: workspace._key,
    locale: workspace.locale,
  }
}
```

**Autorização:**
- JWT tokens com 14 dias de expiração
- Workspace extraído do token e injetado no context
- Role-based access control (RBAC) por workspace
- Permissions verificadas via GraphQL Shield

### Exemplo Prático

**Service com filtro automático de workspace:**

```typescript
// packages/api/src/deals/service.ts
export class DealService extends Service<Deal> {
  collection = 'deals'
  
  async search(filter: Partial<DealFilter>, user: string): Promise<Deal[]> {
    const filters = this.buildDealFilters(filter, user)
    // Filtro automático por workspace já aplicado em buildDealFilters
    
    const q = aql`
      FOR d IN ${this.aql}
        FILTER ${join(filters, ' FILTER ')}
      RETURN d
    `
    
    return this.query<Deal>(q)
  }
}
```

---

## Event-Driven Architecture

### Visão Geral

A BerryMax utiliza uma arquitetura **event-driven** baseada em **BullMQ** e **Redis** para processar eventos de forma assíncrona. Quando um documento é criado ou atualizado no banco de dados, um evento `document_updated` é emitido e enfileirado, disparando listeners que executam lógica de negócio automatizada.

**Fluxo básico:**
1. Database change (create/update) → Service emite evento
2. Event enfileirado em BullMQ (`document_updated`)
3. Queue worker processa evento
4. Listeners são filtrados e executados em ordem de prioridade
5. Background jobs executam ações (notificações, integrações, etc.)

### Sistema de Eventos

**Event Emission:**

Todos os métodos `create()` e `update()` do Service base emitem eventos automaticamente:

```typescript
// packages/api/src/infra/service.ts
async update<T>(payload: Partial<T>, options?: QueryOptions) {
  const updated = await this.database.updateOne(this.collection, payload)
  
  if (options?.dontEmit !== true) {
    await handleObjectUpdate({
      collection: this.collection,
      current: updated.new,
      previous: updated.old,
      author: requestUser,
    })
  }
  
  return updated
}
```

**Queue Processing:**

**Arquivo:** `packages/api/src/event/queue.ts`

```typescript
export const eventQueue = new Queue('event', {
  connection: getBullMQConnection(),
  defaultJobOptions: {
    attempts: 5,
    backoff: { type: 'exponential', delay: 15000 },
    removeOnComplete: true,
    removeOnFail: false,
  },
})

export const queueWorker = new Worker('event', async job => {
  if (job.name === 'document_updated') {
    await sendListenersToQueue(job.data, eventListeners)
  }
  // ... outros job types
})
```

### Listeners

**Base Class:**

**Arquivo:** `packages/api/src/event/base.ts`

```typescript
export abstract class EventListener<T extends ArangoObject> {
  priority: number = 0 // Quanto menor, maior a prioridade
  name: string = ''
  
  abstract filter(payload: ChangePayload<T>): Promise<boolean> | boolean
  abstract execute(payload: ChangePayload<T>): Promise<void>
}
```

**Priority System:**
- **0-1**: Infraestrutura crítica (edge collections, computed values)
- **10-50**: Lógica de negócio (CRM, contracts, payments)
- **100+**: Processos secundários (notificações, analytics)

**Listener Registration:**

**Arquivo:** `packages/api/src/listeners.ts`

```typescript
export const eventListeners: MaiaEventListener<ArangoObject>[] = [
  new UpdateUserTaskEdgeCollection(), // Priority 0
  new CrmLeadAnalysisUseCase(),       // Priority 10
  new CrmDealIsMQL(),                 // Priority 10
  new CrmDealIsSQL(),                 // Priority 10
  new CrmDealClose(),                 // Priority 20
  // ... mais listeners
]
```

### Job Processing

**Job Types Principais:**
- `document_updated`: Dispara listeners
- `whatsapp_incoming`: Processa mensagens WhatsApp
- `stripe_webhook`: Processa eventos Stripe
- `listener`: Executa um listener específico
- `file_transcription`: Transcreve arquivos de áudio
- `pdf_parsing`: Processa PDFs
- `websocket_broadcast`: Broadcast para clientes WebSocket

**Retry Logic:**
- 5 tentativas por padrão
- Exponential backoff (15 segundos inicial)
- Failed jobs mantidos por 24 horas para análise
- Notificações enviadas após falha final

### Exemplo Prático

**Fluxo completo: Deal → MQL → SQL → Opportunity**

```typescript
// 1. Deal criado/atualizado
await dealService.update({ _key: 'deals_123', status: 'inbound_analysis' })
// → Emite document_updated

// 2. CrmLeadAnalysisUseCase (priority 10) executa
// → Analisa lead, calcula score, cria tasks

// 3. Task de MQL completada
await taskService.update({ 
  _key: 'tasks_456', 
  processOutcome: 'is_mql' 
})
// → Emite document_updated

// 4. CrmDealIsMQL (priority 10) executa
// → Atualiza deal.status = 'mql'
// → Atribui BDR
// → Cria task de SQL

// 5. Task de SQL completada
await taskService.update({ 
  _key: 'tasks_789', 
  processOutcome: 'is_sql' 
})

// 6. CrmDealIsSQL (priority 10) executa
// → Atualiza deal.status = 'sql'
// → Atribui SDR
// → Agenda call de qualificação

// 7. CrmDealConvertoToOpportunity (priority 20) executa
// → Atualiza deal.status = 'opportunity'
// → Atribui Closer
```

---

## Integrações Externas

### Stripe Integration

#### Arquitetura

A BerryMax utiliza **Stripe Connect** para gerenciar pagamentos de múltiplos workspaces (franchisees). O sistema mantém:
- **Platform Account**: Conta principal Berryconsul (franchiser)
- **Connect Accounts**: Contas de cada franchisee workspace

**Webhook Endpoints:**
- `/stripe`: Webhooks da conta platform
- `/stripe/connect`: Webhooks das contas Connect

#### Webhook Processing

**Arquivo:** `packages/api/src/infra/stripe/webhook.ts`

```typescript
export async function stripeWebhookController(
  req: FastifyRequest,
  reply: FastifyReply,
): Promise<void> {
  const event = await constructStripeEvent(req, false)
  await eventQueue.add('stripe_webhook', event)
  reply.send({ ok: true })
}

export async function processStripeWebhook(job: Job<StripeEvent>): Promise<void> {
  const event = job.data
  const workspace = await workspaceService().byStripeAccountId(event.account)
  await webhookRouter.route(event.type, workspace, event.data.object)
}
```

**Webhook Router:**

**Arquivo:** `packages/api/src/infra/stripe/handlers/webhook-router.ts`

O router centraliza o roteamento de eventos Stripe para handlers específicos:

```typescript
class StripeWebhookRouter {
  private static readonly ROUTES: WebhookRoutes = {
    ...invoiceHandlers,      // invoice.payment_succeeded, etc.
    ...subscriptionHandlers,  // customer.subscription.created, etc.
    ...checkoutHandlers,     // checkout.session.completed, etc.
    ...miscHandlers,         // payment_intent.*, etc.
  }
  
  async route(eventType: string, workspace: Workspace, data: unknown): Promise<void> {
    const handler = StripeWebhookRouter.ROUTES[eventType]
    if (handler) {
      await handler(workspace, data)
    }
  }
}
```

#### Casos de Uso Principais

- **Subscription Creation**: Criação de assinaturas recorrentes para contratos
- **Invoice Payment**: Processamento de pagamentos de faturas
- **CAC Charging**: Cobrança de CAC para leilões vencidos
- **Payment Failed**: Retry automático e notificações

#### Exemplo

**Handler de invoice payment succeeded:**

```typescript
// packages/api/src/infra/stripe/handlers/invoice-handlers.ts
export const invoiceHandlers = {
  'invoice.payment_succeeded': async (
    workspace: Workspace,
    data: unknown,
  ): Promise<void> => {
    const invoice = data as Stripe.Invoice
    const subscription = await getSubscription(invoice.subscription)
    
    // Ativa projeto relacionado
    await projectService.update({
      _key: subscription.metadata.projectKey,
      status: 'active',
    })
    
    // Notifica workspace
    await sendNotification(workspace, 'Payment succeeded')
  },
}
```

### Google Workspace Integration

#### Autenticação

A BerryMax utiliza **Google OAuth 2.0** para autenticação de usuários internos (domínio `berry.com.br`). O fluxo suporta tanto **token ID** quanto **authorization code**.

**Arquivo:** `packages/api/src/auth/cases/token.ts`

```typescript
export async function authenticateWithToken(input: OAuthInput): Promise<User> {
  const oAuth2 = new google.auth.OAuth2(
    process.env.GOOGLE_CLIENT_ID,
    process.env.GOOGLE_CLIENT_SECRET,
  )
  
  const ticket = await oAuth2.verifyIdToken({
    idToken: input.token,
    audience: process.env.GOOGLE_CLIENT_ID,
  })
  
  const payload = ticket.getPayload()
  
  // Validação de domínio
  if (payload.hd !== 'berry.com.br') {
    throw new Error('invalid_domain')
  }
  
  return {
    id: payload.sub,
    email: payload.email,
    name: payload.name,
    // ...
  }
}
```

#### APIs Utilizadas

- **Google Calendar**: Criação e atualização de eventos, sincronização de reuniões
- **Google Drive**: Armazenamento de arquivos, criação de pastas
- **Google Docs**: Geração de documentos (contratos, propostas)
- **Google Sheets**: Processamento de dados, relatórios
- **Gmail**: Envio de emails automatizados
- **Google Admin**: Sincronização de usuários do workspace

#### Service Account Pattern

Para operações server-to-server, a BerryMax utiliza **Service Account** com JWT authentication:

**Arquivo:** `packages/api/src/infra/google.ts`

```typescript
export function getGoogleApi(api: GoogleApi, subject: string): GoogleApiClient {
  const auth = new google.auth.JWT({
    subject, // Email do usuário para impersonation
    keyFile: 'google-service.json',
    scopes: [
      'https://www.googleapis.com/auth/calendar',
      'https://www.googleapis.com/auth/drive',
      // ... outros scopes
    ],
  })
  
  switch (api) {
    case 'calendar':
      return google.calendar({ version: 'v3', auth })
    case 'drive':
      return google.drive({ version: 'v3', auth })
    // ... outros APIs
  }
}
```

#### Exemplo

**Criação de evento no Google Calendar:**

```typescript
// packages/api/src/google/calendar/create.ts
export async function createGoogleCalendarEvent(
  calendarId: string,
  event: calendar_v3.Params$Resource$Events$Insert,
): Promise<calendar_v3.Schema$Event> {
  const calendarApi = getGoogleApi('calendar', calendarId)
  const response = await calendarApi.events.insert(event)
  return response.data
}
```

### WhatsApp Business API Integration

#### Arquitetura

A integração com WhatsApp Business API permite comunicação automatizada com leads e clientes. O sistema processa mensagens recebidas via webhook e responde automaticamente usando a IA Maia quando apropriado.

**Webhook Endpoint:** `/whatsapp/webhook`

#### Webhook Processing

**Arquivo:** `packages/api/src/whats/controllers/webhook.ts`

```typescript
export async function whatsappWebhookController(
  req: FastifyRequest,
  reply: FastifyReply,
): Promise<void> {
  const payload = req.body as WhatsAppWebhookPayload
  await eventQueue.add('whatsapp_incoming', payload)
  reply.send(challenge) // Para validação do webhook
}

export async function processWhatsappPayload(
  job: Job<WhatsAppWebhookPayload>,
): Promise<void> {
  const payload = job.data
  for (const entry of payload.entry) {
    for (const change of entry.changes) {
      if (change.value.messages) {
        await processMessages(change, phone_number_id, display_phone_number)
      }
    }
  }
}
```

#### Message Flow

1. **Mensagem recebida** → Webhook enfileirado
2. **Conversation criada/inicializada** → 24h window atualizado
3. **Mensagem salva** → Banco de dados
4. **AI Response (Maia)** → Se dentro da janela de 24h e condições atendidas
5. **Mensagem enviada** → WhatsApp API
6. **Status updates** → WebSocket broadcast para frontend

**Arquivo:** `packages/api/src/whats/cases/incoming.ts`

```typescript
export async function processIncomingMessages(
  messages: WhatsappMessageModel[],
): Promise<void> {
  for (const message of messages) {
    const waConvo = new WappConversation()
    await waConvo.initializeWithMessage(message)
    await waConvo.createIncomingMessage(message)
    
    // Atualiza janela de 24h
    const expiration = dayjs().add(24, 'hours').unix()
    await waConvo.update24hConversationWindow(expiration)
  }
}
```

#### Real-time Updates

O frontend recebe atualizações em tempo real via **WebSocket**:

```typescript
// Frontend: packages/app/src/modules/whats/pages/chats.state.ts
export function setupWhatsappWebsocket() {
  WsService.subscribe('whatsapp::{chat}::messages', (data) => {
    // Atualiza estado local com nova mensagem
    whatsappState$.messages.merge([data])
  })
}
```

#### Exemplo

**Fluxo completo de mensagem recebida:**

```typescript
// 1. Webhook recebido
POST /whatsapp/webhook
{
  "entry": [{
    "changes": [{
      "value": {
        "messages": [{ "from": "5511999999999", "text": { "body": "Olá" } }]
      }
    }]
  }]
}

// 2. Job enfileirado
await eventQueue.add('whatsapp_incoming', payload)

// 3. Mensagem processada
await processIncomingMessages(messages)
// → Conversation criada/atualizada
// → Mensagem salva no banco
// → Event document_updated emitido

// 4. MaiaRespondToWhatsappMessage listener executa
// → Analisa mensagem
// → Gera resposta com IA
// → Envia via WhatsApp API

// 5. Frontend atualizado via WebSocket
WsService.broadcast({
  channel: 'whatsapp::{chat}::messages',
  type: 'create',
  data: newMessage,
})
```

---

## Arquitetura de Módulos

### Domain-Driven Design

A BerryMax segue **Domain-Driven Design** com módulos auto-contidos que representam domínios de negócio específicos. Cada módulo contém sua própria lógica de negócio, acesso a dados e definições GraphQL.

**Estrutura de módulo típica:**

```
src/
├── deals/              # Domínio: CRM e Sales
│   ├── entities.ts     # Type definitions
│   ├── service.ts      # Data access layer
│   ├── graphql.ts      # GraphQL schema
│   ├── defs.ts         # Resolver mappings
│   ├── controllers/    # Business logic
│   └── cases/          # Use cases (listeners, etc.)
├── projects/           # Domínio: Project Management
├── tasks/              # Domínio: Task Management
├── users/              # Domínio: User Management
└── whats/              # Domínio: WhatsApp Integration
```

### Service Pattern

Todos os módulos estendem a classe base `Service<T>` que fornece:

- **CRUD operations**: `get()`, `create()`, `update()`, `archive()`
- **Query building**: `find()`, `findOne()`, `query()`
- **Caching**: Cache automático com Redis
- **DataLoader**: Batch loading para evitar N+1 queries
- **Event emission**: Eventos automáticos em create/update

**Arquivo:** `packages/api/src/infra/service.ts`

```typescript
export class Service<T extends ArangoObject> {
  collection: CollectionName = 'none'
  dataLoader = new DataLoader<string, T>(this.findByKeys.bind(this))
  
  async get<R extends T>(key: string): Promise<R | undefined> {
    // Cache check → Database query → Cache set
  }
  
  async create<T>(payload: Partial<T>): Promise<CreateUpdateResponse<T>> {
    // Auto-timestamps, event emission
  }
  
  async update<T>(payload: Partial<T>): Promise<CreateUpdateResponse<T>> {
    // Auto-timestamps, event emission, cache invalidation
  }
}
```

### GraphQL Architecture

A API utiliza **code-first approach** com `apollo-server-core` e `gql` template literals:

**Schema Definition:**

```typescript
// packages/api/src/deals/graphql.ts
export const DealGraphqlDefs = gql`
  type Deal {
    _key: String!
    workspace: String!
    name: String!
    status: String!
    # ...
  }
  
  extend type Query {
    deals(filter: DealFilter!): [Deal]
    deal(key: String!): Deal
  }
`
```

**Resolver Mapping:**

```typescript
// packages/api/src/deals/defs.ts
export const DealGraphqlQueries = {
  deals: all([isAuthenticated], getDealsController),
  deal: all([isAuthenticated], getDealController),
}
```

**Context Injection:**

Todos os resolvers recebem `context` com `user`, `workspace` e `locale`:

```typescript
export async function getDealsController(
  _: unknown,
  args: { filter: DealFilter },
  context: GraphqlContext,
): Promise<Deal[]> {
  return dealService().search(args.filter, context.user)
}
```

### Module Structure

Cada módulo segue esta estrutura padrão:

- **`entities.ts`**: Interfaces TypeScript e tipos
- **`service.ts`**: Classe Service estendendo `Service<T>`
- **`graphql.ts`**: Definições de schema GraphQL
- **`defs.ts`**: Mapeamento de queries/mutations para controllers
- **`controllers/`**: Lógica de negócio (resolvers GraphQL)
- **`cases/`**: Use cases (listeners, background jobs, etc.)

---

## Infraestrutura e Deploy

### Containerização

A BerryMax utiliza **Docker** para containerização de todos os serviços:

- **API**: `Dockerfile` (produção), `Dockerfile.Dev` (desenvolvimento)
- **Frontend**: `Dockerfile` (produção), `Dockerfile.Dev` (desenvolvimento)
- **Databases**: Docker Compose para desenvolvimento local

### Orquestração

**Docker Swarm** é utilizado para orquestração em produção:

- **Service scaling**: Replicação horizontal de serviços
- **Health checks**: Verificação automática de saúde dos containers
- **Rolling updates**: Atualizações sem downtime
- **Service discovery**: Descoberta automática de serviços

**Arquivo:** `packages/api/deploy/swarm.yml`

```yaml
services:
  api:
    image: berrymax/api:latest
    replicas: 3
    deploy:
      update_config:
        parallelism: 1
        delay: 10s
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
```

### Reverse Proxy

**Nginx** atua como reverse proxy e load balancer:

- **SSL/TLS termination**: Certificados Let's Encrypt
- **Load balancing**: Distribuição de requisições entre réplicas
- **Rate limiting**: Proteção contra abuso
- **Static files**: Servir assets do frontend

### CI/CD

O processo de deploy utiliza:

- **Build**: Compilação TypeScript, build do frontend
- **Testing**: Execução de testes (90% coverage threshold)
- **Docker build**: Criação de imagens Docker
- **Deploy**: Push para registry e atualização do Swarm

---

## Segurança

### Autenticação

- **JWT tokens**: Tokens com 14 dias de expiração
- **Google OAuth**: Autenticação via Google para usuários internos
- **Domain validation**: Apenas domínio `berry.com.br` permitido
- **Token refresh**: Renovação automática de tokens

### Autorização

- **Role-based access control (RBAC)**: Permissões por role (admin, member, guest)
- **Workspace isolation**: Acesso restrito aos dados do próprio workspace
- **GraphQL Shield**: Middleware de autorização para resolvers
- **Permission checks**: Verificação de permissões em operações críticas

### API Security

- **GraphQL Armor**: Proteção contra query complexity attacks
- **Rate limiting**: Limitação de requisições por IP/usuário
- **Input validation**: Validação de inputs com Zod
- **CORS**: Configuração restritiva de CORS

### Data Protection

- **Encryption at rest**: Dados criptografados no banco de dados
- **Encryption in transit**: HTTPS/TLS para todas as comunicações
- **LGPD compliance**: Conformidade com Lei Geral de Proteção de Dados
- **Audit logs**: Registro de todas as operações críticas

---

## Performance e Escalabilidade

### Caching Strategy

**Redis** é utilizado em múltiplas camadas:

- **Service-level cache**: Cache de queries frequentes (TTL-based)
- **Controller-level cache**: Cache de respostas GraphQL
- **Session cache**: Cache de sessões de usuário
- **Cache invalidation**: Invalidação automática em updates

**Arquivo:** `packages/api/src/infra/service.ts`

```typescript
async get<R extends T>(key: string): Promise<R | undefined> {
  // 1. Check cache
  const cached = await getFromCache({ collection: this.collection, key })
  if (cached) return JSON.parse(cached)
  
  // 2. Database query
  const data = await this.database.get(this.collection, key)
  
  // 3. Set cache
  if (data) {
    await setToCache({ collection: this.collection, key, data: JSON.stringify(data) })
  }
  
  return data
}
```

### Database Optimization

- **ArangoDB indexing**: Índices persistentes e covering indexes
- **Query optimization**: AQL queries otimizadas com índices apropriados
- **Connection pooling**: Pool de conexões reutilizáveis
- **DataLoader**: Batch loading para evitar N+1 queries

### Horizontal Scaling

- **Stateless services**: API e Frontend são stateless, permitindo escalonamento horizontal
- **Load balancing**: Nginx distribui carga entre múltiplas réplicas
- **Database sharding**: Planejado para futuro (quando necessário)
- **Queue workers**: Múltiplos workers processando jobs em paralelo

---

## Best Practices

### Multi-Tenant Isolation

- ✅ Sempre incluir campo `workspace` em documentos multi-tenant
- ✅ Filtrar queries automaticamente por workspace do context
- ✅ Validar workspace em operações cross-tenant
- ✅ Usar `context.workspace` em vez de hardcode

### Event-Driven Patterns

- ✅ Usar `dontEmit: true` apenas quando necessário (evitar loops infinitos)
- ✅ Organizar listeners por prioridade (infraestrutura → negócio → secundário)
- ✅ Implementar `filter()` eficiente para evitar execuções desnecessárias
- ✅ Não executar tarefas longas em listeners (usar queue jobs)

### Integration Patterns

- ✅ Validar webhooks com assinaturas (Stripe, WhatsApp)
- ✅ Enfileirar webhooks para processamento assíncrono
- ✅ Implementar retry logic com exponential backoff
- ✅ Logar todos os eventos de integração para debugging

### Error Handling

- ✅ Usar `AppGraphqlError` para erros GraphQL
- ✅ Logar erros com contexto (user, workspace, request ID)
- ✅ Implementar retry automático para jobs falhados
- ✅ Notificar sobre falhas críticas (Ntfy)

---

## Anti-Patterns

### Multi-Tenancy

- ❌ **Não filtrar por workspace**: Sempre aplicar filtros de workspace
- ❌ **Hardcode de workspace**: Usar `context.workspace` sempre
- ❌ **Cross-tenant queries**: Evitar queries que cruzam workspaces
- ❌ **Workspace em cache keys**: Incluir workspace em chaves de cache

### Event-Driven

- ❌ **Emitir eventos em listeners**: Usar `dontEmit: true` para evitar loops
- ❌ **Tarefas longas em listeners**: Mover para queue jobs
- ❌ **Listeners sem filter()**: Sempre implementar filter eficiente
- ❌ **Prioridades inconsistentes**: Seguir padrão 0-1, 10-50, 100+

### Integrações

- ❌ **Processar webhooks síncronamente**: Sempre enfileirar
- ❌ **Ignorar validação de assinatura**: Validar todos os webhooks
- ❌ **Sem retry logic**: Implementar retry para operações críticas
- ❌ **Logs insuficientes**: Logar todos os eventos de integração

---

## Troubleshooting

### Problemas Comuns

#### Multi-Tenancy

**Problema**: Dados de um workspace aparecendo em outro

**Solução**: Verificar se filtros de workspace estão sendo aplicados em todas as queries. Usar `context.workspace` em vez de valores hardcoded.

**Problema**: Context sem workspace

**Solução**: Verificar se token JWT contém workspace válido. Validar se usuário pertence a workspace ativo.

#### Event-Driven

**Problema**: Listener não executa

**Solução**: 
1. Verificar se `filter()` retorna `true`
2. Verificar prioridade do listener
3. Verificar logs do queue worker
4. Verificar se evento foi emitido (`dontEmit: true`?)

**Problema**: Loop infinito de eventos

**Solução**: Usar `dontEmit: true` ao atualizar documentos dentro de listeners para evitar re-emissão de eventos.

#### Integrações

**Problema**: Webhook Stripe não processa

**Solução**: 
1. Verificar assinatura do webhook
2. Verificar se workspace foi encontrado pelo `stripeAccountId`
3. Verificar logs do queue worker
4. Verificar se handler existe no router

**Problema**: WhatsApp mensagens não chegam

**Solução**:
1. Verificar validação do webhook (challenge)
2. Verificar se job foi enfileirado
3. Verificar logs de `processWhatsappPayload`
4. Verificar se conversation foi criada

### Debugging Strategies

1. **Logs estruturados**: Usar `Logger` com contexto (user, workspace, request ID)
2. **Queue inspection**: Usar Bull Board (`/admin/queues`) para inspecionar jobs
3. **Database queries**: Verificar diretamente no ArangoDB se dados foram salvos
4. **Event tracing**: Adicionar logs em pontos críticos do fluxo de eventos

---

## Referências

### Arquivos Críticos do Codebase

**Multi-Tenant:**
- `packages/api/src/infra/context.ts` - Context creation
- `packages/api/src/workspace/entities.ts` - Workspace entity
- `packages/api/src/infra/service.ts` - Base Service class

**Event-Driven:**
- `packages/api/src/event/base.ts` - EventListener base class
- `packages/api/src/event/queue.ts` - Queue e worker
- `packages/api/src/listeners.ts` - Listener registration
- `packages/api/src/event/helpers.ts` - Helper functions

**Stripe:**
- `packages/api/src/infra/stripe/webhook.ts` - Webhook handlers
- `packages/api/src/infra/stripe/handlers/webhook-router.ts` - Router
- `packages/api/src/infra/stripe/handlers/*.ts` - Event handlers

**Google:**
- `packages/api/src/auth/cases/token.ts` - OAuth authentication
- `packages/api/src/infra/google.ts` - Google API clients

**WhatsApp:**
- `packages/api/src/whats/controllers/webhook.ts` - Webhook handler
- `packages/api/src/whats/cases/incoming.ts` - Message processing

**Service Pattern:**
- `packages/api/src/infra/service.ts` - Base Service
- `packages/api/src/deals/service.ts` - Exemplo de Service

### Documentação Oficial

- **ArangoDB**: https://docs.arangodb.com/
- **BullMQ**: https://docs.bullmq.io/
- **Stripe Connect**: https://stripe.com/docs/connect
- **Google Workspace APIs**: https://developers.google.com/workspace
- **WhatsApp Business API**: https://developers.facebook.com/docs/whatsapp

### Links Úteis

- **CLAUDE.md**: Documentação completa do desenvolvedor (backend + frontend)
- **ArangoDB Guide**: `docs/technology-guides/arangodb.md`
- **TypeScript Guide**: `docs/technology-guides/typescript.md`
- **Legend App State Guide**: `docs/technology-guides/legend-app-state.md`

---

**Última atualização:** 01 de Dezembro de 2025  
**Versão:** 1.0.0
