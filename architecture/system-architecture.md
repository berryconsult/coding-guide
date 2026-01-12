# Arquitetura e Tecnologia - BerryMax

## Índice

1. [Visão Geral](#visão-geral)
2. [Arquitetura Multi-Tenant](#arquitetura-multi-tenant)
3. [Event-Driven Architecture](#event-driven-architecture)
4. [Integrações Externas](#integrações-externas)
5. [Arquitetura de Módulos](#arquitetura-de-módulos)
6. [Infraestrutura](#infraestrutura)
7. [Segurança](#segurança)
8. [Best Practices e Anti-Patterns](#best-practices-e-anti-patterns)
9. [Referências](#referências)

---

## Visão Geral

A **BerryMax** é uma plataforma B2B de consultoria que automatiza o ciclo de vida do cliente. Utiliza arquitetura **multi-tenant** com **event-driven architecture**.

### Stack Tecnológico

| Camada | Tecnologias |
|--------|-------------|
| Backend | Node.js 24+, TypeScript, Fastify, Apollo GraphQL |
| Frontend | React 19, Vite, TanStack Router, Legend State |
| Databases | ArangoDB, Redis, Elasticsearch, Qdrant |
| Infra | Docker Swarm, Nginx, BullMQ, WebSockets |
| Integrações | Stripe, Google Workspace, WhatsApp Business API |

### Arquitetura de Alto Nível

```
Frontend (React) → GraphQL/REST → Maia API (Fastify)
                                       ↓
                    ┌──────┬──────┬───────┬──────┐
                    │Arango│Redis │Elastic│Qdrant│
                    └──────┴──────┴───────┴──────┘
```

## Arquitetura Multi-Tenant

### Conceito

**Workspace** é a unidade de isolamento multi-tenant. Cada workspace (franchiser ou franchisee) tem dados completamente isolados em múltiplas camadas: banco, autenticação, autorização e cache.

### Isolamento de Dados

Todos os documentos multi-tenant incluem `workspace`:

```typescript
export interface Deal extends ArangoObject {
  workspace: string // OBRIGATÓRIO
  // ...
}
```

Services aplicam filtros automáticos por workspace em todas as queries.

### Autenticação e Autorização

**Context** criado por requisição (`packages/api/src/infra/context.ts`):

```typescript
export const createAppContext = async (req: FastifyRequest): Promise<GraphqlContext> => {
  const token = extractToken(req.headers.authorization)
  const user = await userFromToken(token)
  const workspace = await workspaceService().get(user.workspace)
  
  return { user: user._key, workspace: workspace._key, locale: workspace.locale }
}
```

- JWT tokens (14 dias expiração)
- Role-based access control (RBAC) por workspace
- Permissions via GraphQL Shield

---

## Event-Driven Architecture

### Visão Geral

Arquitetura baseada em **BullMQ** e **Redis**. Quando um documento é criado/atualizado, evento `document_updated` é emitido e enfileirado, disparando listeners.

**Fluxo:**
1. Database change → Service emite evento
2. Event enfileirado em BullMQ
3. Queue worker processa
4. Listeners filtrados e executados por prioridade
5. Background jobs executam ações

### Sistema de Eventos

Métodos `create()` e `update()` do Service base emitem eventos automaticamente (`packages/api/src/infra/service.ts`).

**Queue Processing** (`packages/api/src/event/queue.ts`):
- 5 tentativas por padrão
- Exponential backoff (15s inicial)
- Failed jobs mantidos 24h

### Listeners

**Base Class** (`packages/api/src/event/base.ts`):

```typescript
export abstract class EventListener<T extends ArangoObject> {
  priority: number = 0
  abstract filter(payload: ChangePayload<T>): Promise<boolean> | boolean
  abstract execute(payload: ChangePayload<T>): Promise<void>
}
```

**Priority System:**
- **0-1**: Infraestrutura crítica
- **10-50**: Lógica de negócio
- **100+**: Processos secundários

**Job Types:** `document_updated`, `whatsapp_incoming`, `stripe_webhook`, `listener`, `file_transcription`, `pdf_parsing`, `websocket_broadcast`

---

## Integrações Externas

### Stripe Integration

**Stripe Connect** para pagamentos multi-workspace:
- Platform Account: Conta principal Berryconsul
- Connect Accounts: Contas de cada franchisee

**Webhooks:** `/stripe` (platform), `/stripe/connect` (Connect)

Webhook router centraliza roteamento (`packages/api/src/infra/stripe/handlers/webhook-router.ts`).

**Casos de uso:** Subscription Creation, Invoice Payment, CAC Charging, Payment Failed

### Google Workspace Integration

**OAuth 2.0** para autenticação (domínio `berry.com.br`).

**APIs:** Calendar, Drive, Docs, Sheets, Gmail, Admin

**Service Account** para operações server-to-server (`packages/api/src/infra/google.ts`).

### WhatsApp Business API

**Webhook:** `/whatsapp/webhook`

**Fluxo:**
1. Mensagem recebida → Webhook enfileirado
2. Conversation criada → 24h window atualizado
3. Mensagem salva → AI Response (Maia) se condições atendidas
4. Status updates → WebSocket broadcast

---

## Arquitetura de Módulos

### Domain-Driven Design

Módulos auto-contidos representando domínios de negócio:

```
src/
├── deals/              # CRM e Sales
│   ├── entities.ts     # Types
│   ├── service.ts      # Data access
│   ├── graphql.ts      # Schema
│   ├── defs.ts         # Resolver mappings
│   ├── controllers/    # Business logic
│   └── cases/          # Use cases
├── projects/           # Project Management
├── tasks/              # Task Management
├── users/              # User Management
└── whats/              # WhatsApp Integration
```

### Service Pattern

Classe base `Service<T>` (`packages/api/src/infra/service.ts`):
- CRUD operations
- Query building
- Caching automático (Redis)
- DataLoader (evita N+1)
- Event emission automático

### GraphQL Architecture

Code-first approach com `apollo-server-core`. Todos os resolvers recebem `context` com `user`, `workspace` e `locale`.

---

## Infraestrutura

### Containerização e Orquestração

- **Docker**: Containerização de todos os serviços

### Performance

**Caching (Redis):**
- Service-level cache
- Invalidação automática em updates

**Database:**
- ArangoDB indexing
- Query optimization
- Connection pooling


---

## Segurança

| Área | Implementação |
|------|---------------|
| Autenticação | JWT (14 dias), Google OAuth, Domain validation |
| Autorização | RBAC, Workspace isolation, GraphQL Shield |
| API | GraphQL Armor, Rate limiting, Input validation (Zod), CORS |
| Dados | Encryption at rest/transit, Audit logs |

---

## Best Practices e Anti-Patterns

### ✅ Best Practices

**Multi-Tenant:**
- Sempre incluir `workspace` em documentos multi-tenant
- Filtrar queries automaticamente por workspace do context
- Usar `context.workspace` em vez de hardcode

**Event-Driven:**
- Usar `dontEmit: true` para evitar loops infinitos
- Organizar listeners por prioridade
- Implementar `filter()` eficiente
- Tarefas longas em queue jobs, não em listeners

**Integrações:**
- Validar webhooks com assinaturas
- Enfileirar webhooks para processamento assíncrono
- Implementar retry com exponential backoff

### ❌ Anti-Patterns

**Multi-Tenant:**
- Não filtrar por workspace
- Hardcode de workspace
- Cross-tenant queries

**Event-Driven:**
- Emitir eventos em listeners sem `dontEmit: true`
- Tarefas longas em listeners
- Listeners sem filter()

**Integrações:**
- Processar webhooks síncronamente
- Ignorar validação de assinatura

---

## Referências

### Arquivos Críticos

| Área | Arquivos |
|------|----------|
| Multi-Tenant | `infra/context.ts`, `workspace/entities.ts`, `infra/service.ts` |
| Event-Driven | `event/base.ts`, `event/queue.ts`, `listeners.ts` |
| Stripe | `infra/stripe/webhook.ts`, `infra/stripe/handlers/` |
| Google | `auth/cases/token.ts`, `infra/google.ts` |
| WhatsApp | `whats/controllers/webhook.ts`, `whats/cases/incoming.ts` |

### Documentação Oficial

- [ArangoDB](https://docs.arangodb.com/)
- [BullMQ](https://docs.bullmq.io/)
- [Stripe Connect](https://stripe.com/docs/connect)
- [Google Workspace APIs](https://developers.google.com/workspace)
- [WhatsApp Business API](https://developers.facebook.com/docs/whatsapp)

### Guias Internos

- [ArangoDB](../technology-guides/arangodb.md)
- [TypeScript](../technology-guides/typescript.md)
- [Legend State](../technology-guides/legend-app-state.md)
