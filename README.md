# Documentação Berry - Repositório Central

Bem-vindo ao repositório central de documentação da Berry! Este é o ponto de partida para toda a documentação técnica, processos de desenvolvimento, guias de tecnologia e padrões de qualidade.

---

## Índice

- [Processos de Desenvolvimento](#processos-de-desenvolvimento)
- [Guias de Tecnologia](#guias-de-tecnologia)
- [Qualidade e Testes](#qualidade-e-testes)
- [Arquitetura e Tecnologia](#arquitetura-e-tecnologia)

---

## Processos de Desenvolvimento

### Git Workflow

**Arquivo:** [processes/git-workflow.md](./processes/git-workflow.md)

Documentação completa do fluxo de trabalho Git na Berry, incluindo:
- Estratégia de branching (GitFlow adaptado)
- Convenções de commit messages
- Criação e gerenciamento de branches
- Resolução de conflitos
- Integração com ferramentas (GitHub, Coolify)
- Fluxo de hotfix para bugs críticos
- Best practices e troubleshooting

### Code Review

**Arquivo:** [processes/code-review.md](./processes/code-review.md)

Processo completo de code review integrado com QA:
- Fluxo: 2 approvals → Ambiente temporário → QA validation → Merge automático
- Papéis e responsabilidades (Author, Reviewer, QA, Tech Lead)
- Níveis de severidade ([Bloqueador], [Sugestão], [Nitpick], [Pergunta])
- Checklist de review (código, testes, performance, segurança)
- Exemplos de threads de review
- Integração com ambientes temporários para QA
- Boas práticas e anti-patterns

### Pull Requests

**Arquivo:** [processes/pull-requests.md](./processes/pull-requests.md) ✅

Guia completo de Pull Requests na Berry (~7.000 palavras):
- Anatomia de um PR (título, descrição WWW, template obrigatório)
- Checklist pré-PR (self-review, testes, formatação)
- Processo completo: Criação → Code Review → QA → Merge
- Durante code review (responder feedback, escalar Tech Lead)
- Ambiente temporário e validação QA
- Merge obrigatório: Squash and Merge
- Cenários especiais: Hotfixes, PRs grandes, PRs dependentes, conflitos
- Checklist completo (pré-PR, durante review, antes de merge)
- Boas práticas e anti-patterns
- 3 exemplos práticos completos com threads de review
- FAQ, ferramentas (GitHub CLI, Draft PRs), glossário

### Task Management

**Arquivo:** [processes/task-management.md](./processes/task-management.md)

Sistema completo de gerenciamento de tarefas com Light Scrum:
- Story Points (Fibonacci: 1, 2, 3, 5, 8, 13)
- **Regra de Ouro:** Tarefas > 5 pontos DEVEM ser divididas
- Workflow de 9 status (Backlog → Completed)
- Prioridades do desenvolvedor (Hotfixes > Changes Requested > Code Review > In Progress > To Do)
- Sistema de IDs (BRY, MAIA, DEAL, PROJ, FIX)
- Gestão de Sprints (2 semanas, Daily, Retro)
- RACI matrix para todos os papéis
- Exemplos práticos e templates

---

## Guias de Tecnologia

### TypeScript Guide

**Arquivo:** [technology-guides/typescript.md](./technology-guides/typescript.md) ✅

Guia completo de TypeScript para Berry (Backend + Frontend):
- Configuração TypeScript (Node.js 22+, React 19)
- Tipos fundamentais e avançados
- **7 regras específicas da Berry:**
  - Use `undefined` ao invés de `null`
  - Use `??` (nullish coalescing) ao invés de `||`
  - Tipos de retorno explícitos sempre
  - Proibido default exports
  - Proibido `any` (use `unknown`)
  - Evite template literals aninhados
  - Use utility functions (`isNotEmptyValue`, `isEmptyValue`)
- Padrão `Service<T>` e Generics
- GraphQL types e resolvers
- Async/Await e Promises
- Type Safety em Runtime (Zod)
- Best Practices e Anti-Patterns
- 12 exemplos práticos completos
- Troubleshooting

### ESLint & Prettier Guide

**Arquivo:** [technology-guides/eslint-prettier.md](./technology-guides/eslint-prettier.md) ✅

Guia objetivo de lint/format para Maia API (Node 24) e Maia App (React 19):
- Configuração flat do ESLint 9 + TypeScript-ESLint
- Prettier 3 e ignores compartilhados
- Scripts `lint`/`format`, VS Code e lint-staged opcional

### Frontend Styling Guide

**Arquivo:** [technology-guides/frontend-styling.md](./technology-guides/frontend-styling.md) ✅

Padrão de estilização do Maia App (React 19 + Vite + Tailwind + shadcn/ui):
- Reuso do design system antes de utilitários
- Tailwind para layout/spacing/cores; sem inline style
- Estados (hover/focus/disabled/loading) e responsividade mobile-first
- Checklist visual para PRs

### Legend App State Guide

**Arquivo:** [technology-guides/legend-app-state.md](./technology-guides/legend-app-state.md) ✅

Guia completo de boas práticas Legend App State para Berry (Frontend):
- Objetivo: Documentar e padronizar a forma como a Berry usa Legend App State
- **⚠️ REGRA FUNDAMENTAL: useState é PROIBIDO** - Todos os componentes devem usar `useObservable`
- Configuração e setup (estado global `state$`, estado local `useObservable`)
- **Padrões de Uso:**
  - Componentes com `observer()` (obrigatório)
  - Naming conventions (sufixo `$` obrigatório)
  - Estado local vs estado global
- **Operações com Observables:**
  - Leitura: `.get()` (reativo), `.peek()` (não-reativo), `use()` (hook)
  - Escrita: `.set()`, `.push()`, `.delete()`
  - Batch updates: `beginBatch()` / `endBatch()`
  - Merge: `mergeIntoObservable()` para updates parciais
- **Sincronização e Persistência:**
  - `syncToLocalStorage()` helper customizado
  - Integração com TanStack Query (server state vs UI state)
  - WebSocket real-time updates (exemplo WhatsApp)
- **Performance e Otimização:**
  - Fine-grained reactivity (re-render apenas o que mudou)
  - Evitar múltiplos `.get()` em loops
  - Computed values com `useComputed()`
  - Memoização quando necessário
- **Padrões Avançados:**
  - Estados modulares por feature (`whatsappState$`, `taskState$`, `crmPageState$`)
  - Helpers customizados (`maiaMergeIntoObservable`)
  - Integração TanStack Query + Legend State
  - WebSocket patterns completos
- **Best Practices Checklist:** Queries, estado, performance, naming, componentes
- **Anti-Patterns:** O que evitar (useState, sem sufixo `$`, sem `observer()`, `.get()` em loops)
- **4 Exemplos Práticos Completos:**
  - Componente simples com estado local (formulário)
  - Componente com estado global
  - Estado modular completo (WhatsApp state com WebSocket)
  - Integração TanStack Query + Legend State
- Troubleshooting (componente não re-renderiza, estado não atualiza, performance issues)
- Referências (documentação oficial, arquivos críticos do codebase)

**Total:** ~1.600 palavras (~6 páginas)

### ArangoDB Guide

**Arquivo:** [technology-guides/arangodb.md](./technology-guides/arangodb.md) ✅

Guia completo de boas práticas ArangoDB para Berry (Backend):
- Objetivo: Documentar e padronizar a forma como a Berry usa ArangoDB
- Configuração e conexão (singleton, connection pool, retry logic)
- Service Architecture Pattern (Base Service<T>, CRUD methods, auto-timestamps)
- **Queries AQL:**
  - Template literals com `aql` tag (type-safety)
  - Filtros dinâmicos com `join()`
  - Queries com arrays, agregações, cálculos
  - Full-text search com ArangoSearch
  - Graph queries com edge collections
- **Naming Conventions:** Collections (`lowercase_snake_case`), índices (`idx_{collection}_{fields}`), campos (`camelCase`), IDs (`{collection}_{nanoId()}`)
- **Data Model Padrão:** Interface `ArangoObject`, workspace isolation, soft delete, timestamps ISO 8601
- **Indexing:**
  - Tipos (persistent, unique, sparse, array)
  - Ordem de campos CRÍTICA (igualdades → ranges → sort)
  - Covering indexes com `storedValues`
  - Cache enabled para queries frequentes
- **ArangoSearch Views & Analyzers:**
  - Analyzers customizados para português brasileiro (`maia::pt_br_text_search`)
  - Views de full-text search
  - `BM25()` e `BOOST()` para relevância
- **Graph Operations:**
  - Edge collections (`{entity1}_{entity2}_edge`)
  - Diff pattern para sincronização
  - Transactions para atomicidade
- **Performance:** DataLoader (evitar N+1), Redis cache, query optimization
- **Error Handling:** Retry logic (lock errors, connection errors, socket errors)
- **Best Practices Checklist:** Queries, indexing, services, performance, error handling
- **Anti-Patterns:** O que evitar (string concatenation, full scans, delete físico)
- **4 Exemplos Práticos Completos:**
  - Service completo (DealService)
  - Full-text search (analyzer + view + query)
  - Graph operations (edge collection + transaction + traversal)
  - Agregação complexa (múltiplos LETs, subqueries)
- Troubleshooting (query lenta, lock errors, índice não usado)
- Referências (documentação oficial, arquivos críticos)

**Total:** ~18.000 palavras (~70 páginas)

### Outros Guias Planejados

- GraphQL & Apollo Guide
- TanStack Guide (Router + Query)
- Tailwind & Design System
- Node.js & Fastify
- Docker & Deploy

---

## Qualidade e Testes

### Testes Automatizados

**Arquivo:** [quality/automated-testing.md](./quality/automated-testing.md) ✅

Documento único cobrindo Backend e Frontend:

**Backend (Vitest 2.1.9, Node.js 22+):**
- 125 test files, ~50k linhas
- 90% coverage threshold obrigatório
- Padrões de mocking centralizados (`infra/test/`)
- Estrutura AAA (Arrange-Act-Assert)
- Factories para dados complexos
- 5 exemplos práticos completos:
  - Unit test simples (crypto)
  - Service test com mocks (DealService)
  - Event listener test (CRM flow)
  - Integration test (Elasticsearch)
  - Webhook test (Stripe)

**Frontend (Vitest 2.1.9, React 19):**
- Status atual: < 0.2% coverage (2 test files)
- **Fase 1 (Atual):** Testar utilities, services, cálculos
- Sem React Testing Library ainda
- 4 exemplos práticos:
  - Utility function (contract-utils)
  - Calculator (validação + cálculos)
  - Service test (AuthService mock)
  - Custom hook (useKeypress)

**Conteúdo:**
- Checklist para novos test files
- Troubleshooting e erros comuns
- Code review considerations
- Performance de testes
- Arquivos modelo para referência

**Meta Frontend:** 20% coverage nos próximos 3 meses (foco em utils, services, data hooks)

### Testes Manuais

**Arquivo:** [quality/manual-testing.md](./quality/manual-testing.md) ✅

Guia completo de testes manuais para QA da Berry:

**10 Módulos de Teste Documentados:**
- ID 00001: Autenticação & Login (7 test cases)
- ID 00002: CRM & Leads (15 test cases)
- ID 00003: Projetos & Tarefas (12 test cases)
- ID 00004: Contratos (10 test cases)
- ID 00005: Pagamentos - Stripe (10 test cases)
- ID 00006: Leilões - Auction (10 test cases)
- ID 00007: WhatsApp & Chat (10 test cases)
- ID 00008: Dashboards (8 test cases)
- ID 00009: Organizações & Contatos (10 test cases)
- ID 00010: Usuários & Permissões (10 test cases)

**4 Fluxos End-to-End Críticos:**
- Fluxo E2E 1: Lead → Projeto Completo (15 passos, ~30 min)
- Fluxo E2E 2: Leilão de Deal de Alto Valor (12 passos, ~25 min)
- Fluxo E2E 3: WhatsApp Lead → CRM → Atribuição (9 passos, ~20 min)
- Fluxo E2E 4: Falha de Pagamento → Retry → Sucesso (9 passos, ~15 min)

**Conteúdo Adicional:**
- Matriz de priorização (P0 - Crítico, P1 - Alto, P2 - Médio, P3 - Baixo)
- Integração com Code Review e processo de QA
- 12 casos de erro & edge cases
- Template de bug report
- Dados de teste e credenciais (Stripe, WhatsApp, ZapSign)
- Checklist de aprovação de PR
- Setup de ambiente de testes (Coolify staging)

**Total:** 102 test cases + 4 fluxos E2E completos

### Testes End-to-End (E2E)

**Arquivos:**
- [quality/e2e-setup.md](./quality/e2e-setup.md) ✅
- [quality/e2e-guide.md](./quality/e2e-guide.md) ✅

**E2E Setup (~3.500 palavras):**
Guia técnico de configuração da infraestrutura E2E:
- Playwright 1.40.0 (já instalado em packages/api)
- Estrutura de diretórios (`packages/e2e/`)
- Configuração completa (playwright.config.ts, tsconfig.json, package.json)
- Global setup e verificação de serviços
- Fixtures customizados (auth, API GraphQL, webhooks)
- Ambiente de testes (.env.example, dados de teste)
- Test data builders e helpers
- Troubleshooting e debugging
- Integração com CI/CD (planejado)

**E2E Practical Guide (~4.500 palavras):**
Guia prático com exemplos de código completos:

**3 Fluxos Críticos Documentados:**
1. **Lead → Projeto** (15 passos, 6 listeners):
   - CrmLeadAnalysisUseCase → CrmDealIsMQL → CrmDealIsSQL → CrmDealClose → GenerateContractSigningFromContract → CreateProjectFromDealUseCase
2. **Leilão de Deal Alto Valor** (12 passos):
   - AuctionEndedWithWinnerUseCase → ChargeWorkspaceForCAC
3. **WhatsApp → CRM** (9 passos):
   - WhatsappIncomingUseCase → MaiaRespondUseCase

**Conteúdo:**
- Anatomia de um teste E2E (estrutura AAA)
- Page Object Model completo (CRM, Auction, Contract, Project, Chat)
- Exemplos de testes completos (60+ linhas cada)
- Padrões de assertions customizados
- Mocking de webhooks (Stripe, ZapSign, WhatsApp)
- Teste de listeners e eventos
- Fixtures de autenticação (loginAsAdmin, loginAsBDR)
- GraphQL client para setup de dados
- Best practices e anti-patterns
- Estratégia de integração vs mocking

**Estrutura criada:**
- `packages/e2e/` (package.json, tsconfig.json, playwright.config.ts)
- `fixtures/` (test-base.ts, auth.ts, api.ts, webhooks.ts)
- Pronto para implementação dos testes

---

## Arquitetura e Tecnologia

### Arquitetura do Sistema

**Arquivo:** [architecture/system-architecture.md](./architecture/system-architecture.md) ✅

Documentação completa da arquitetura do sistema BerryMax:
- Visão geral da plataforma e componentes principais
- Arquitetura multi-tenant (workspace isolation, context injection)
- Event-driven architecture (BullMQ, listeners, job processing)
- Integrações externas (Stripe Connect, Google Workspace, WhatsApp Business API)
- Arquitetura de módulos (Domain-Driven Design, Service Pattern, GraphQL)
- Infraestrutura e deploy (Docker Swarm, Nginx, CI/CD)
- Segurança (autenticação, autorização, data protection)
- Performance e escalabilidade (caching, database optimization)
- Best practices e anti-patterns
- Troubleshooting de problemas comuns

**Total:** ~7.000 palavras (~30 páginas)

---

## Como Usar Esta Documentação

### Para Desenvolvedores Novos

1. **Comece com:** [Git Workflow](./processes/git-workflow.md)
2. **Depois:** [TypeScript Guide](./technology-guides/typescript.md)
3. **E então:** [Testes Automatizados](./quality/automated-testing.md)
4. **Finalmente:** [Code Review](./processes/code-review.md) e [Task Management](./processes/task-management.md)

### Para Desenvolvedores Experientes

Consulte os guias específicos conforme necessário:
- **Dúvidas de processo:** Veja a seção [Processos de Desenvolvimento](#processos-de-desenvolvimento)
- **Dúvidas técnicas:** Veja [Guias de Tecnologia](#guias-de-tecnologia)
- **Dúvidas de testes:** Veja [Qualidade e Testes](#qualidade-e-testes)

### Para Tech Leads

Use esta documentação como referência para:
- Onboarding de novos desenvolvedores
- Padronização de processos
- Code review checklist
- Planejamento de sprints

---

## Contribuindo para a Documentação

Ao adicionar ou atualizar documentação:

1. **Siga o padrão:**
   - Tom corporativo e profissional
   - Altamente detalhado com exemplos práticos
   - Português do Brasil (descrições e explicações)
   - Código em inglês (padrão do codebase)

2. **Estrutura:**
   - Introdução clara do propósito do documento
   - Seções bem organizadas com índice
   - Exemplos práticos e código real
   - Troubleshooting e referências

3. **Atualização:**
   - Mantenha este README.md atualizado
   - Marque documentos completos com ✅
   - Marque planejados com 🚧

---

## Contato e Suporte

Para dúvidas sobre a documentação:
- **Tech Lead:** Pedro
- **Repositório:** [BerryGitHub/berrymax](https://github.com/berryconsult/berrymax)
- **Issue Tracker:** Use GitHub Issues para reportar problemas ou sugerir melhorias

---

**Última atualização:** 10 de Dezembro de 2025
**Versão:** 1.7.0
**Documentos completos:** 14 ✅
**Documentos planejados:** 4+ 🚧
