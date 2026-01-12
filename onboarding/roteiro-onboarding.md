# Roteiro de Onboarding - Berry

Bem-vindo(a) à Berry! Este roteiro vai te guiar pelos principais aspectos da empresa, processos, tecnologias e arquitetura do sistema.

---

## Estrutura do Onboarding

| Etapa | Duração | Descrição |
|-------|---------|-----------|
| **1. Reunião de Apresentação** | 1 hora | Dorli apresenta empresa/negócio, Tech Lead apresenta processos, tecnologia e arquitetura |
| **2. Estudo Individual** | 2 dias | Leitura da documentação disponibilizada |
| **3. Tasks de Adaptação** | ~1 semana total | Tarefas básicas para familiarização com o sistema |
| **4. Inclusão na Sprint** | Próxima sprint | Participação ativa no ciclo de desenvolvimento |

**Responsável pelo onboarding:** Tech Lead (Pedro Pacheco)

---

## Índice

1. [Reunião de Apresentação (1h)](#reunião-de-apresentação-1h)
2. [Estudo Individual - Processos](#estudo-individual---processos)
3. [Estudo Individual - Tecnologia](#estudo-individual---tecnologia)
4. [Estudo Individual - Arquitetura](#estudo-individual---arquitetura)
5. [Checklist de Conclusão](#checklist-de-conclusão)

---

## Reunião de Apresentação (1h)

> **Formato:** Reunião presencial ou remota

Esta reunião cobre os 4 pilares do onboarding de forma resumida. Após a reunião, o novo membro terá 2 dias para estudar a documentação detalhada.

---

### Parte 1: Empresa e Negócio (~15 min)

> **Responsável:** Dorli

#### Sobre a Berry

A **Berry** é uma empresa de consultoria B2B que desenvolveu a plataforma **BerryMax** - um sistema completo para automatizar o ciclo de vida do cliente, desde a captação de leads até a gestão de projetos.

#### O Produto: BerryMax

- **CRM Completo:** Gestão de leads, deals e pipeline de vendas
- **Automação de Processos:** Fluxos automatizados de qualificação e conversão
- **Gestão de Projetos:** Acompanhamento de projetos e tarefas
- **Integrações:** WhatsApp Business, Stripe, Google Workspace
- **IA (Maia):** Assistente inteligente para análise e comunicação

#### Modelo de Negócio

- **Multi-tenant:** Cada cliente (workspace) tem dados completamente isolados
- **Franchiser/Franchisee:** Suporte a estruturas de franquias
- **Pagamentos:** Integração com Stripe Connect para cobrança e repasses

#### Fluxo Principal do Negócio

```
Lead → MQL → SQL → Won → Contrato → Pagamento → Projeto Ativo
```

---

### Parte 2: Processos (~15 min)

> **Responsável:** Tech Lead

#### Git Workflow

| Branch | Propósito |
|--------|-----------|
| `main` | Produção (protegida) |
| `development` | Integração (protegida) |

**Nomenclatura:** `[ID-DA-TAREFA]` (ex: `MAIA-45`, `DEAL-78`)

#### Task Management

| Pontos | Duração | Exemplo |
|--------|---------|---------|
| **1** | < 1 hora | Ajustar texto |
| **2** | 1-2 horas | Componente simples |
| **3** | 2-4 horas | Filtro em lista |
| **5** | 1 dia | Feature completa |

#### Fluxo de Status

```
Backlog → To Do → In Progress → Code Review → QA → Approved → Ready to Deploy → Completed
```

🔴 **Changes Requested = Prioridade MÁXIMA**

---

### Parte 3: Tecnologia (~15 min)

> **Responsável:** Tech Lead

#### Stack Tecnológico

| Camada | Tecnologias |
|--------|-------------|
| **Backend** | Node.js 24+, TypeScript, Fastify, Apollo GraphQL |
| **Frontend** | React 19, Vite, TanStack Router, Legend State |
| **Databases** | ArangoDB, Redis, Elasticsearch, Qdrant |
| **Infra** | BullMQ, WebSockets |

#### Regras Principais

- **TypeScript:** Strict mode, sem `any`, sem `null` (usar `undefined`)
- **Frontend:** `useState` proibido → usar Legend State (`useObservable`)
- **Backend:** Sempre usar `aql` tag para queries ArangoDB
- **Testes:** Vitest, coverage 90%, padrão AAA

---

### Parte 4: Arquitetura (~15 min)

> **Responsável:** Tech Lead

#### Arquitetura de Alto Nível

```
Frontend (React) → GraphQL/REST → Maia API (Fastify)
                                       ↓
                    ┌──────┬──────┬───────┬──────┐
                    │Arango│Redis │Elastic│Qdrant│
                    └──────┴──────┴───────┴──────┘
```

#### Conceitos-Chave

- **Multi-Tenant:** Workspace é a unidade de isolamento
- **Event-Driven:** BullMQ + Redis para processamento assíncrono
- **Integrações:** Stripe Connect, Google Workspace, WhatsApp Business API

---

## Estudo Individual - Processos

> **Duração:** ~4 horas de leitura

### Documentos para Ler

| Documento | Descrição | Link |
|-----------|-----------|------|
| Git Workflow | Branches, commits, merge | [git-workflow.md](../processes/git-workflow.md) |
| Pull Requests | Template, checklist, requisitos | [pull-requests.md](../processes/pull-requests.md) |
| Code Review | Papéis, checklist, boas práticas | [code-review.md](../processes/code-review.md) |
| Task Management | Story points, fluxo, prioridades | [task-management.md](../processes/task-management.md) |
| QA Guide | Testes manuais e E2E | [qa-guide.md](../processes/qa-guide.md) |

### Pontos Importantes

#### Git Workflow
- Branches sempre com ID da tarefa: `MAIA-45`
- Commits: `MAIA-45: Add lead analysis`
- Merge sempre via PR para `development`

#### Pull Requests
- 2 aprovações obrigatórias (pelo menos 1 sênior)
- Screenshots obrigatórios para mudanças visuais
- CI/CD verde antes do merge

#### Code Review
- Foco no código, não na pessoa
- Sugerir alternativas concretas
- Elogiar boas implementações

#### Task Management
- Tarefas > 5 pontos devem ser divididas
- Máximo 1 tarefa In Progress por dev
- Changes Requested = prioridade máxima

---

## Estudo Individual - Tecnologia

> **Duração:** ~6 horas de leitura

### Documentos para Ler

| Documento | Descrição | Link |
|-----------|-----------|------|
| TypeScript | Padrões e regras Berry | [typescript.md](../technology-guides/typescript.md) |
| Legend App State | Estado no frontend | [app-state.md](../technology-guides/app-state.md) |
| ArangoDB | Queries e data model | [arangodb.md](../technology-guides/arangodb.md) |
| Testes Automatizados | Vitest, mocking, padrões | [automated-testing.md](../technology-guides/automated-testing.md) |
| Frontend Styling | Tailwind, shadcn/ui | [frontend-styling.md](../technology-guides/frontend-styling.md) |

### Pontos Importantes

#### TypeScript
```typescript
// ✅ Use undefined (não null)
// ✅ Use ?? (não ||)
// ✅ Tipos de retorno explícitos
// ❌ Proibido: any, default exports
```

#### Legend App State
```typescript
// ❌ PROIBIDO
const [name, setName] = useState('')

// ✅ CORRETO
const state$ = useObservable({ name: '' })
```

#### ArangoDB
```typescript
// ✅ Sempre usar aql tag
const q = aql`FOR doc IN ${this.aql} FILTER doc._key == ${key} RETURN doc`

// ✅ Sempre incluir workspace em documentos multi-tenant
```

#### Testes
```typescript
// Padrão AAA
it('should work', () => {
  // Arrange - preparar dados
  // Act - executar ação
  // Assert - verificar resultado
})
```

---

## Estudo Individual - Arquitetura

> **Duração:** ~3 horas de leitura

### Documento Principal

[system-architecture.md](../architecture/system-architecture.md)

### Pontos Importantes

#### Multi-Tenant
- Workspace é a unidade de isolamento
- Todos os documentos incluem `workspace`
- Context criado por requisição com user e workspace

#### Event-Driven
- BullMQ + Redis para filas
- Listeners com sistema de prioridade
- Jobs para tarefas longas (não em listeners)

#### Integrações
- **Stripe Connect:** Pagamentos multi-workspace
- **Google Workspace:** OAuth 2.0, Calendar, Drive
- **WhatsApp:** Webhook para mensagens, AI response

#### Estrutura de Módulos (DDD)
```
src/
├── deals/              # CRM e Sales
├── projects/           # Project Management
├── tasks/              # Task Management
├── users/              # User Management
└── whats/              # WhatsApp Integration
```

---

## Checklist de Conclusão

### Reunião de Apresentação
- [ ] Participou da reunião com Dorli e Tech Lead
- [ ] Entendeu o produto BerryMax
- [ ] Entendeu o fluxo Lead → Projeto
- [ ] Conheceu a stack tecnológica

### Estudo Individual - Processos (2 dias)
- [ ] Leu git-workflow.md
- [ ] Leu pull-requests.md
- [ ] Leu code-review.md
- [ ] Leu task-management.md
- [ ] Leu qa-guide.md

### Estudo Individual - Tecnologia (2 dias)
- [ ] Leu typescript.md
- [ ] Leu app-state.md (Legend State)
- [ ] Leu arangodb.md
- [ ] Leu automated-testing.md
- [ ] Leu frontend-styling.md

### Estudo Individual - Arquitetura (2 dias)
- [ ] Leu system-architecture.md
- [ ] Entendeu arquitetura multi-tenant
- [ ] Entendeu event-driven architecture
- [ ] Conheceu as integrações

### Setup do Ambiente
- [ ] Acesso ao repositório GitHub
- [ ] Acesso ao Plane (task management)
- [ ] Docker e Docker Compose instalados
- [ ] Node.js 24+ instalado
- [ ] pnpm instalado
- [ ] Projeto rodando localmente
- [ ] Extensões do VS Code configuradas

---

## Tasks de Adaptação (~1 semana)

Após os 2 dias de estudo, o novo membro receberá tarefas básicas para se familiarizar com o sistema:

- Tarefas de **1-2 pontos** do backlog
- Foco em mudanças pequenas e isoladas
- Pair programming com dev sênior quando necessário
- Participação em code reviews como observador

---

## Inclusão na Sprint

Após completar as tasks de adaptação, o novo membro será incluído na próxima sprint com:

- Participação no planning
- Tarefas de até **3 pontos** inicialmente
- Aumento gradual de complexidade
- Mentoria contínua do Tech Lead

---

**Bem-vindo(a) ao time! 🍓**

*Qualquer dúvida, fale com o Tech Lead ou com o time.*
