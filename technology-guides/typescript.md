# Guia TypeScript - Berry

## 1. Introdução

### 1.1 Por que TypeScript na Berry?

TypeScript é a linguagem fundamental em toda a stack da Berry, tanto no backend quanto no frontend. A escolha de TypeScript permite:

- **Segurança de Tipos**: Detecta erros em tempo de desenvolvimento, antes que cheguem à produção
- **Manutenibilidade**: Código autodocumentado com tipos explícitos
- **Refatoração Segura**: IDEs podem realizar refatorações complexas com confiança
- **Produtividade**: Autocompletar e IntelliSense aceleram o desenvolvimento
- **Qualidade**: Reduz bugs em ~15% segundo estudos da indústria

### 1.2 Escopo deste Guia

Este documento cobre:
- Configuração TypeScript para projetos Berry (backend e frontend)
- Regras e padrões específicos da Berry
- Padrões arquiteturais usados no código
- Exemplos práticos do codebase Berry
- Troubleshooting e best practices

---

## 2. Configuração do Projeto

### 2.1 Backend (Maia API)

**Arquivo**: `packages/api/tsconfig.json`

```json
{
  "extends": "@tsconfig/node22/tsconfig.json",
  "compilerOptions": {
    "target": "esnext",
    "module": "esnext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "allowSyntheticDefaultImports": true,
    "baseUrl": ".",
    "paths": {
      "@app/*": ["src/*"],
      "arangojs/aql": ["./node_modules/arangojs/aql.js"]
    },
    "outDir": "build",
    "rootDir": "src"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "build"]
}
```

**Características principais**:
- **Node.js 22+**: Versão moderna com suporte ESNext
- **Strict Mode**: Todas as verificações estritas ativas
- **ESM**: ES Modules nativos (`"module": "esnext"`)
- **Path Aliases**: `@app/*` mapeia para `src/*`

### 2.2 Frontend (Maia Vite)

**Arquivo**: `packages/app/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "jsx": "react-jsx",
    "esModuleInterop": true,
    "skipLibCheck": true,
    "allowSyntheticDefaultImports": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "baseUrl": ".",
    "paths": {
      "@app/*": ["src/*"]
    }
  },
  "include": ["src"]
}
```

**Características principais**:
- **React 19**: JSX com `react-jsx` transform
- **Vite Bundler**: Resolução de módulos otimizada
- **Strict Mode**: Mesma rigorosidade do backend

### 2.3 Build Process

**Backend**:
```bash
# Compilação com resolução de aliases
tsc && tsc-alias
```

**Frontend**:
```bash
# Vite já resolve aliases nativamente
vite build
```

---

## 3. Tipos Fundamentais

### 3.1 Tipos Primitivos

```typescript
// String
const name: string = 'Berry Consultoria';
const email: string = 'contato@berryconsultoria.com';

// Number
const age: number = 25;
const revenue: number = 150000.50;

// Boolean
const isActive: boolean = true;
const hasPermission: boolean = false;

// Array
const tags: string[] = ['crm', 'sales', 'consulting'];
const scores: number[] = [1, 2, 3, 5];

// Tuple
const coordinate: [number, number] = [10.5, 20.3];
const userRole: [string, number] = ['admin', 1];
```

### 3.2 Tipos Especiais

```typescript
// Undefined (PREFERIDO na Berry)
let value: string | undefined = undefined;

// Never para funções que nunca retornam
function throwError(message: string): never {
  throw new Error(message);
}

// Unknown para valores desconhecidos (melhor que any)
function processValue(value: unknown): string {
  if (typeof value === 'string') {
    return value.toUpperCase();
  }
  return String(value);
}

// Void para funções sem retorno
function logMessage(message: string): void {
  console.warn(message);
}
```

---

## 4. Regras Específicas da Berry

### 4.1 Use `undefined` ao invés de `null`

❌ **Incorreto**:
```typescript
interface User {
  name: string;
  email: string | null; // Evite null
}

const user: User = {
  name: 'John',
  email: null,
};
```

✅ **Correto**:
```typescript
interface User {
  name: string;
  email?: string; // Automaticamente string | undefined
}

const user: User = {
  name: 'John',
  email: undefined, // Ou omita a propriedade
};
```

**Por quê?**
- Consistência: JavaScript usa `undefined` por padrão
- Menos confusão: Um único tipo para "ausência de valor"
- Melhor integração com operadores modernos

### 4.2 Use Nullish Coalescing (`??`)

❌ **Incorreto**:
```typescript
const displayName = user.name || 'Anonymous';
// Problema: "" (string vazia) também retorna 'Anonymous'

const count = items.length || 0;
// Problema: 0 também retorna 0 (comportamento inesperado)
```

✅ **Correto**:
```typescript
const displayName = user.name ?? 'Anonymous';
// Apenas undefined ou null retornam 'Anonymous'

const count = items.length ?? 0;
// Apenas undefined ou null retornam 0
```

### 4.3 Use Utility Functions para Validação

❌ **Incorreto**:
```typescript
if (value !== undefined && value !== null) {
  // Process value
}
```

✅ **Correto**:
```typescript
import { isNotEmptyValue, isEmptyValue } from '@app/utils/validator';

if (isNotEmptyValue(value)) {
  // Process value
}

if (isEmptyValue(data)) {
  throw AppGraphqlError(400, 'Data is required');
}
```

### 4.4 Tipos de Retorno Explícitos Sempre

❌ **Incorreto**:
```typescript
function getUser(id: string) { // Tipo de retorno implícito
  return userService().get(id);
}
```

✅ **Correto**:
```typescript
function getUser(id: string): Promise<User> {
  return userService().get(id);
}

async function getUserSync(id: string): Promise<User> {
  return userService().get(id);
}
```

### 4.5 Proibido Default Exports

❌ **Incorreto**:
```typescript
// src/users/service.ts
export default class UserService {
  // ...
}
```

✅ **Correto**:
```typescript
// src/users/service.ts
export class UserService extends Service<User> {
  collection: CollectionName = 'users';
}

export const userService = new UserService();
```

**Por quê?**
- Clareza: Nome explícito na importação
- Refatoração: IDEs conseguem rastrear melhor
- Consistência: Padrão único em toda a codebase

### 4.6 Proibido `any` (use `unknown`)

❌ **Incorreto**:
```typescript
function processData(data: any): void { // eslint-disable-line @typescript-eslint/no-explicit-any
  console.log(data.name); // Sem type safety
}
```

✅ **Correto**:
```typescript
function processData(data: unknown): void {
  if (typeof data === 'object' && data !== null && 'name' in data) {
    console.log((data as { name: string }).name);
  }
}

// Ou use Type Guard
function isUser(data: unknown): data is User {
  return (
    typeof data === 'object' &&
    data !== null &&
    'name' in data &&
    'email' in data
  );
}

function processUser(data: unknown): void {
  if (isUser(data)) {
    console.log(data.name); // Type-safe
  }
}
```

### 4.7 Evite Template Literals Aninhados

❌ **Incorreto**:
```typescript
const query = aql`
  FOR doc IN ${`collection_${name}`}
  RETURN doc
`;
```

✅ **Correto**:
```typescript
const collectionName = `collection_${name}`;
const query = aql`
  FOR doc IN ${collectionName}
  RETURN doc
`;
```

---

## 5. Utility Types

### 5.1 Partial e Required

```typescript
interface User {
  _key: string;
  name: string;
  email: string;
  role: string;
}

// Partial: Todos os campos opcionais
type UserUpdate = Partial<User>;
const update: UserUpdate = { name: 'John' }; // Válido

// Required: Todos os campos obrigatórios
type UserCreate = Required<User>;
const newUser: UserCreate = { // Todos os campos necessários
  _key: '123',
  name: 'John',
  email: 'john@example.com',
  role: 'admin',
};
```

### 5.2 Pick e Omit

```typescript
interface Deal {
  _key: string;
  title: string;
  value: number;
  status: string;
  createdAt: string;
  updatedAt: string;
}

// Pick: Seleciona campos específicos
type DealSummary = Pick<Deal, '_key' | 'title' | 'value'>;
const summary: DealSummary = {
  _key: '123',
  title: 'New Deal',
  value: 50000,
};

// Omit: Remove campos específicos
type DealInput = Omit<Deal, '_key' | 'createdAt' | 'updatedAt'>;
const input: DealInput = {
  title: 'New Deal',
  value: 50000,
  status: 'open',
};
```

### 5.3 Record

```typescript
// Mapear chaves para tipos
type UserPermissions = Record<string, boolean>;
const permissions: UserPermissions = {
  'user:read': true,
  'user:write': false,
  'deal:read': true,
};

// Com chaves tipadas
type DealStatus = 'open' | 'won' | 'lost';
type StatusConfig = Record<DealStatus, { color: string; icon: string }>;
const config: StatusConfig = {
  open: { color: 'blue', icon: 'clock' },
  won: { color: 'green', icon: 'check' },
  lost: { color: 'red', icon: 'x' },
};
```

### 5.4 Awaited

```typescript
// Extrai tipo de Promise
type User = Awaited<ReturnType<typeof userService.get>>;

async function getUser(id: string): Promise<User> {
  return userService().get(id);
}

// Equivalente a:
type User = {
  _key: string;
  name: string;
  email: string;
};
```

---

## 6. Generics (Padrão Service<T>)

### 6.1 Service Base Class

A Berry usa um padrão de **Service Genérico** para todos os módulos:

```typescript
// src/infra/service.ts
export abstract class Service<T extends BaseEntity> {
  abstract collection: CollectionName;
  abstract model: string;

  async get(key: string): Promise<T> {
    const query = aql`
      FOR doc IN ${this.aql}
      FILTER doc._key == ${key}
      RETURN doc
    `;
    const cursor = await db.query(query);
    return cursor.next();
  }

  async create(data: Partial<T>): Promise<T> {
    const doc = await db.collection(this.collection).save(data);
    this.emitEvent('document_updated', doc);
    return doc;
  }

  async update(key: string, data: Partial<T>): Promise<T> {
    const doc = await db.collection(this.collection).update(key, data);
    this.emitEvent('document_updated', doc);
    return doc;
  }

  // DataLoader para batch loading
  dataLoader = new DataLoader<string, T>(async (keys: string[]) => {
    return this.getMany(keys);
  });
}
```

### 6.2 Implementação de Service

```typescript
// src/users/entities.ts
export interface User extends BaseEntity {
  _key: string;
  name: string;
  email: string;
  role: UserRole;
  workspaceKey: string;
}

export type UserRole = 'admin' | 'user' | 'client';

// src/users/service.ts
export class UserService extends Service<User> {
  collection: CollectionName = 'users';
  model = 'User';

  // Métodos específicos do User
  async findByEmail(email: string): Promise<User | undefined> {
    const query = aql`
      FOR user IN ${this.aql}
      FILTER user.email == ${email}
      RETURN user
    `;
    const cursor = await db.query(query);
    return cursor.next();
  }
}

export const userService = new UserService();
```

### 6.3 Benefícios do Generic Service

- **Reuso**: CRUD básico implementado uma vez
- **Type Safety**: Todos os métodos retornam o tipo correto
- **Extensibilidade**: Adicione métodos específicos por módulo
- **DataLoader**: Batch loading automático para performance
- **Events**: Emissão automática de eventos para listeners

---

## 7. Tipos Avançados

### 7.1 Union Types

```typescript
// Status com valores literais
type DealStatus = 'open' | 'won' | 'lost';
type PaymentMethod = 'credit_card' | 'boleto' | 'pix';

function processDeal(status: DealStatus): void {
  if (status === 'won') {
    // Type narrowing automático
  }
}
```

### 7.2 Intersection Types

```typescript
interface Timestamped {
  createdAt: string;
  updatedAt: string;
}

interface Workspaced {
  workspaceKey: string;
}

type Deal = Timestamped & Workspaced & {
  _key: string;
  title: string;
  value: number;
};

const deal: Deal = {
  _key: '123',
  title: 'New Deal',
  value: 50000,
  createdAt: '2025-11-24',
  updatedAt: '2025-11-24',
  workspaceKey: 'workspace123',
};
```

### 7.3 Type Guards

```typescript
// Type Predicate
function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    '_key' in value &&
    'email' in value
  );
}

// Uso
function processValue(value: unknown): void {
  if (isUser(value)) {
    console.log(value.email); // Type-safe
  }
}
```

### 7.4 Discriminated Unions

```typescript
interface SuccessResult {
  success: true;
  data: User;
}

interface ErrorResult {
  success: false;
  error: string;
}

type Result = SuccessResult | ErrorResult;

function handleResult(result: Result): void {
  if (result.success) {
    console.log(result.data.name); // Type narrowing
  } else {
    console.error(result.error);
  }
}
```

---

## 8. Padrões de Código Berry

### 8.1 Entities (Definições de Tipos)

```typescript
// src/deals/entities.ts
export interface Deal extends BaseEntity {
  _key: string;
  createdAt: string;
  updatedAt: string;
  workspaceKey: string;
  title: string;
  value: number;
  status: DealStatus;
  organizationKey?: string;
  contactKey?: string;
}

export type DealStatus =
  | 'not_started'
  | 'mql'
  | 'sql'
  | 'opportunity'
  | 'won'
  | 'lost';

export interface DealFilter {
  search?: string;
  status?: DealStatus[];
  minValue?: number;
  maxValue?: number;
  workspaceKey?: string;
}
```

### 8.2 GraphQL Types

```typescript
// src/deals/graphql.ts
import { gql } from 'apollo-server-core';

export const DealGraphqlDefs = gql`
  type Deal {
    _key: String!
    createdAt: String!
    updatedAt: String!
    workspaceKey: String!
    title: String!
    value: Float!
    status: DealStatus!
    organization: Organization
    contact: Contact
  }

  enum DealStatus {
    not_started
    mql
    sql
    opportunity
    won
    lost
  }

  input DealFilter {
    search: String
    status: [DealStatus!]
    minValue: Float
    maxValue: Float
  }

  type Query {
    getDeal(key: String!): Deal
    listDeals(filter: DealFilter): [Deal!]!
  }

  type Mutation {
    createDeal(input: CreateDealInput!): Deal!
    updateDeal(key: String!, input: UpdateDealInput!): Deal!
  }
`;
```

### 8.3 Controllers (GraphQL Resolvers)

```typescript
// src/deals/controllers/get-deal.ts
import { dealService } from '@app/deals/service';
import { Deal } from '@app/deals/entities';
import { GraphqlContext } from '@app/infra/context';
import { AppGraphqlError } from '@app/infra/errors';
import { isEmptyValue } from '@app/utils/validator';

export async function getDealController(
  parent: unknown,
  args: { key: string },
  context: GraphqlContext,
): Promise<Deal> {
  if (isEmptyValue(args.key)) {
    throw AppGraphqlError(400, 'Deal key is required');
  }

  const deal = await dealService().get(args.key);

  if (isEmptyValue(deal)) {
    throw AppGraphqlError(404, 'Deal not found');
  }

  // Workspace isolation check
  if (deal.workspaceKey !== context.workspace) {
    throw AppGraphqlError(403, 'Access denied');
  }

  return deal;
}
```

### 8.4 Event Types

```typescript
// src/event/types.ts
export interface DocumentUpdatedEvent {
  type: 'document_updated';
  collection: CollectionName;
  key: string;
  operation: 'create' | 'update' | 'delete';
  data: Record<string, unknown>;
  previousData?: Record<string, unknown>;
}

export interface WhatsAppIncomingEvent {
  type: 'whatsapp_incoming';
  conversationKey: string;
  messageKey: string;
  from: string;
  body: string;
}

export type ListenerJob =
  | DocumentUpdatedEvent
  | WhatsAppIncomingEvent
  | StripeWebhookEvent;
```

### 8.5 Use Cases (Business Logic)

```typescript
// src/crm/cases/deal-is-mql.ts
import { EventUseCase } from '@app/event/use-case';
import { DocumentUpdatedEvent } from '@app/event/types';
import { Deal } from '@app/deals/entities';
import { taskService } from '@app/tasks/service';

export class CrmDealIsMQL extends EventUseCase {
  priority = 20;
  model = 'Deal';

  async shouldRun(event: DocumentUpdatedEvent): Promise<boolean> {
    if (event.collection !== 'deals') {
      return false;
    }

    const deal = event.data as Deal;
    const previousDeal = event.previousData as Deal | undefined;

    return (
      deal.status === 'mql' &&
      previousDeal?.status !== 'mql'
    );
  }

  async run(event: DocumentUpdatedEvent): Promise<void> {
    const deal = event.data as Deal;

    // Create BDR assignment task
    await taskService().create({
      title: `Qualificar lead: ${deal.title}`,
      description: 'Realizar primeiro contato e qualificação inicial',
      dealKey: deal._key,
      workspaceKey: deal.workspaceKey,
      status: 'to_do',
      priority: 'high',
    });

    console.warn('CrmDealIsMQL: Task created', { dealKey: deal._key });
  }
}
```

---

## 9. Async/Await e Promises

### 9.1 Async Functions com Tipos

```typescript
// Sempre especifique Promise<T> como retorno
async function getUser(id: string): Promise<User> {
  return userService().get(id);
}

async function getUsers(filter: UserFilter): Promise<User[]> {
  return userService().list(filter);
}

// Void para funções sem retorno
async function sendEmail(to: string, subject: string): Promise<void> {
  await emailService().send({ to, subject });
}
```

### 9.2 Error Handling

```typescript
async function createUser(data: UserInput): Promise<User> {
  try {
    const user = await userService().create(data);
    return user;
  } catch (error) {
    console.error('Failed to create user', { error, data });
    throw AppGraphqlError(500, 'Failed to create user');
  }
}
```

### 9.3 Promise.all para Paralelização

```typescript
async function getUserWithRelations(userId: string): Promise<UserWithRelations> {
  // Busca paralela
  const [user, workspace, tasks] = await Promise.all([
    userService().get(userId),
    workspaceService().get(userWorkspaceKey),
    taskService().list({ assignedTo: userId }),
  ]);

  return {
    user,
    workspace,
    tasks,
  };
}
```

---

## 10. Type Safety em Runtime (Zod)

### 10.1 Validação de Input

```typescript
import { z } from 'zod';

// Schema Zod
const CreateUserSchema = z.object({
  name: z.string().min(1, 'Name is required'),
  email: z.string().email('Invalid email'),
  role: z.enum(['admin', 'user', 'client']),
  workspaceKey: z.string().min(1, 'Workspace is required'),
});

// Inferir tipo TypeScript do Zod
type CreateUserInput = z.infer<typeof CreateUserSchema>;

// Uso no controller
export async function createUserController(
  parent: unknown,
  args: { input: unknown },
  context: GraphqlContext,
): Promise<User> {
  // Validação em runtime
  const input = CreateUserSchema.parse(args.input);

  // Agora input é type-safe
  return userService().create(input);
}
```

### 10.2 Validação de Environment

```typescript
// src/infra/env.ts
import { z } from 'zod';

const EnvSchema = z.object({
  NODE_ENV: z.enum(['development', 'staging', 'production']),
  PORT: z.string().transform(Number),
  ARANGO_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  OPENAI_API_KEY: z.string().min(1),
});

export const env = EnvSchema.parse(process.env);

// Type-safe e validado
console.log(env.PORT); // number
console.log(env.NODE_ENV); // 'development' | 'staging' | 'production'
```

---

## 11. Best Practices e Anti-Patterns

### 11.1 Best Practices

✅ **Use Type Inference quando óbvio**:
```typescript
const name = 'John'; // TypeScript infere string
const age = 25; // TypeScript infere number
const users = await userService().list({}); // Infere User[]
```

✅ **Tipos de Retorno Explícitos em Funções Públicas**:
```typescript
export function calculateTotal(items: Item[]): number {
  return items.reduce((sum, item) => sum + item.price, 0);
}
```

✅ **Use Const Assertions para Literais**:
```typescript
const DEAL_STATUSES = ['open', 'won', 'lost'] as const;
type DealStatus = typeof DEAL_STATUSES[number]; // 'open' | 'won' | 'lost'
```

✅ **Prefira Interfaces para Objetos**:
```typescript
interface User {
  _key: string;
  name: string;
}
```

✅ **Use Type Guards para Validação**:
```typescript
if (isNotEmptyValue(user.email)) {
  sendEmail(user.email);
}
```

### 11.2 Anti-Patterns

❌ **Type Assertions Desnecessários**:
```typescript
const user = data as User; // Evite se possível
```

✅ **Validação Adequada**:
```typescript
if (isUser(data)) {
  processUser(data);
}
```

❌ **any em Todo Lugar**:
```typescript
function process(data: any): any { // Muito ruim
  return data;
}
```

✅ **Generics ou Unknown**:
```typescript
function process<T>(data: T): T {
  return data;
}
```

❌ **Type vs Interface Inconsistente**:
```typescript
type User = { name: string }; // Use interface
interface Config = { port: number }; // Sintaxe inválida
```

✅ **Consistente**:
```typescript
interface User { name: string }
type Status = 'open' | 'closed'; // Type para unions
```

---

## 12. Exemplos Práticos

### 12.1 Backend Service Completo

```typescript
// src/projects/entities.ts
export interface Project extends BaseEntity {
  _key: string;
  createdAt: string;
  updatedAt: string;
  workspaceKey: string;
  title: string;
  status: ProjectStatus;
  dealKey?: string;
  organizationKey: string;
}

export type ProjectStatus = 'active' | 'on_hold' | 'completed' | 'cancelled';

export interface ProjectFilter {
  search?: string;
  status?: ProjectStatus[];
  workspaceKey?: string;
}

// src/projects/service.ts
export class ProjectService extends Service<Project> {
  collection: CollectionName = 'projects';
  model = 'Project';

  async listByWorkspace(
    workspaceKey: string,
    filter: ProjectFilter,
  ): Promise<Project[]> {
    const statusFilter = filter.status ?? [];
    const searchTerm = filter.search ?? '';

    const query = aql`
      FOR project IN ${this.aql}
      FILTER project.workspaceKey == ${workspaceKey}
      ${statusFilter.length > 0 ? aql`FILTER project.status IN ${statusFilter}` : aql``}
      ${searchTerm ? aql`FILTER LIKE(project.title, ${`%${searchTerm}%`}, true)` : aql``}
      SORT project.createdAt DESC
      RETURN project
    `;

    const cursor = await db.query(query);
    return cursor.all();
  }
}

export const projectService = new ProjectService();

// src/projects/controllers/list-projects.ts
export async function listProjectsController(
  parent: unknown,
  args: { filter: ProjectFilter },
  context: GraphqlContext,
): Promise<Project[]> {
  if (isEmptyValue(context.workspace)) {
    throw AppGraphqlError(401, 'Workspace required');
  }

  return projectService().listByWorkspace(
    context.workspace,
    args.filter ?? {},
  );
}
```

### 12.2 Frontend Component com Legend State

```typescript
// src/modules/project/project-list.tsx
import { observer, useObservable } from '@legendapp/state/react';
import { useQuery } from '@tanstack/react-query';
import { projectService } from '@app/modules/project/service';
import { ProjectFilter, Project } from '@app/modules/project/entities';
import { Button } from '@app/components/ui/button';
import { Input } from '@app/components/ui/input';

export const ProjectList = observer(() => {
  const filter$ = useObservable<ProjectFilter>({
    search: '',
    status: ['active'],
  });

  const { data: projects, isLoading } = useQuery({
    queryKey: ['projects', filter$.get()],
    queryFn: () => projectService().list(filter$.get()),
  });

  const handleSearch = (value: string): void => {
    filter$.search.set(value);
  };

  const handleStatusToggle = (status: Project['status']): void => {
    const currentStatuses = filter$.status.get() ?? [];
    const newStatuses = currentStatuses.includes(status)
      ? currentStatuses.filter(s => s !== status)
      : [...currentStatuses, status];
    filter$.status.set(newStatuses);
  };

  return (
    <div className="bg-card rounded-lg p-6">
      <div className="mb-4 flex gap-2">
        <Input
          value={filter$.search.get()}
          onChange={e => handleSearch(e.target.value)}
          placeholder="Buscar projetos..."
          className="flex-1"
        />
        <Button onClick={() => handleStatusToggle('active')}>
          Active
        </Button>
      </div>

      {isLoading ? (
        <div>Carregando...</div>
      ) : (
        <table className="w-full">
          <thead>
            <tr>
              <th>Título</th>
              <th>Status</th>
              <th>Criado em</th>
            </tr>
          </thead>
          <tbody>
            {projects?.map(project => (
              <tr key={project._key}>
                <td>{project.title}</td>
                <td>{project.status}</td>
                <td>{project.createdAt}</td>
              </tr>
            ))}
          </tbody>
        </table>
      )}
    </div>
  );
});
```

---

## 13. Troubleshooting

### 13.1 Erros Comuns

**Erro**: `Cannot find module '@app/...'`
- **Causa**: Path alias não configurado
- **Solução**: Verifique `tsconfig.json` e `vite.config.ts` (frontend) ou build process (backend)

**Erro**: `Type 'X' is not assignable to type 'Y'`
- **Causa**: Tipos incompatíveis
- **Solução**: Use Type Guards ou validação adequada

**Erro**: `Object is possibly 'undefined'`
- **Causa**: Strict null checks
- **Solução**: Use optional chaining (`?.`) ou validação (`isNotEmptyValue`)

### 13.2 Performance

**Build lento**:
- Use `skipLibCheck: true` em `tsconfig.json`
- Exclua `node_modules` do `include`

**Type checking lento em IDE**:
- Restart TypeScript server no VSCode
- Use `.vscode/settings.json` para cache:
```json
{
  "typescript.tsserver.maxTsServerMemory": 4096
}
```

---

## 14. Referências

### 14.1 Documentação Oficial

- [TypeScript Handbook](https://www.typescriptlang.org/docs/)
- [TypeScript Deep Dive](https://basarat.gitbook.io/typescript/)

### 14.2 Ferramentas

- **TypeScript Playground**: [https://www.typescriptlang.org/play](https://www.typescriptlang.org/play)
- **TSConfig Reference**: [https://www.typescriptlang.org/tsconfig](https://www.typescriptlang.org/tsconfig)

### 14.3 Recursos Internos

- **CLAUDE.md**: Guia completo de desenvolvimento Berry
- **packages/api/src/infra/**: Utilities e base classes
- **packages/app/src/lib/**: Utilities frontend

---

## 15. Controle de Versão

### 15.1 Versão TypeScript

- **Backend**: TypeScript 5.7.3+
- **Frontend**: TypeScript 5.7.3+

### 15.2 Target

- **Node.js**: ESNext (Node 22+)
- **Browsers**: ESNext (Vite transpila para navegadores modernos)

### 15.3 Atualizações

Para atualizar TypeScript:

```bash
# Backend
cd packages/api
pnpm update typescript

# Frontend
cd packages/app
pnpm update typescript
```

**Atenção**: Sempre teste após atualizar, especialmente em major versions.

---

## Conclusão

TypeScript é a espinha dorsal da Berry, permitindo desenvolvimento seguro, rápido e manutenível. Siga estas regras:

1. **Strict mode sempre ativo**
2. **Tipos de retorno explícitos**
3. **Use `undefined` ao invés de `null`**
4. **Prefira `??` ao invés de `||`**
5. **Nenhum default export**
6. **Evite `any`, use `unknown`**
7. **Valide com Zod em runtime**
8. **Use path aliases (`@app/*`)**
9. **Siga padrão `Service<T>` para módulos**
10. **Type Guards para validação**

Com essas práticas, você escreverá código TypeScript de qualidade, alinhado com os padrões da Berry.
