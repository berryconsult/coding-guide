# Documentação Técnica da Berry

Bem-vindo ao repositório de documentação técnica da Berry! Este é o hub central para todos os processos, guias e padrões de desenvolvimento da empresa.

## Objetivo

Este repositório centraliza toda a documentação técnica e de processos da Berry, garantindo que todos os membros do time tenham acesso padronizado e atualizado às práticas e procedimentos da empresa.

## Índice de Documentos

### Processos de Desenvolvimento

- **[Git Workflow](./processes/git-workflow.md)** ✅
  - Estratégia de branches (main, development, feature branches)
  - Padrões de commits e mensagens
  - Processo de Pull Requests e Code Review
  - Fluxo completo de desenvolvimento
  - Troubleshooting e boas práticas

- **Code Review** 🚧 (Em breve)
  - Responsabilidades de revisores e autores
  - Checklist de revisão
  - Boas práticas de feedback
  - Ferramentas e automações

- **Task Management** 🚧 (Em breve)
  - Sistema de pontuação (Story Points)
  - Processo de estimativa
  - Ciclo de vida das tarefas
  - Gestão de Sprints

### Arquitetura e Tecnologia

- **Arquitetura do Sistema** 🚧 (Em breve)
  - Visão geral da plataforma BerryMax
  - Arquitetura multi-tenant
  - Event-driven architecture
  - Integrações

### Qualidade e Testes

- **QA Guidelines** 🚧 (Em breve)
  - Processo de testes manuais
  - Testes automatizados (E2E)
  - Critérios de aceitação
  - Registro de bugs

### Deploy e CI/CD

- **Deploy Process** 🚧 (Em breve)
  - Ambientes (dev, staging, production)
  - Processo de deploy
  - Rollback e contingência
  - Monitoramento

### Onboarding

- **Novo Desenvolvedor** 🚧 (Em breve)
  - Setup do ambiente
  - Primeiros passos
  - Recursos essenciais
  - Contatos importantes

## Links Importantes

- **[CLAUDE.md (Raiz do Projeto)](../CLAUDE.md)**: Guia técnico completo do BerryMax (API + Frontend)
- **[GitHub - BerryMax](https://github.com/berry/berrymax)**: Repositório principal
- **[Plane.so - Board](https://plane.so/berry/projects)**: Gestão de tarefas

## Padrões da Berry

### Sistema de IDs de Tarefas

Toda tarefa deve ter um ID único no formato: `[PREFIXO]-[NÚMERO]`

**Prefixos Utilizados**:

| Prefixo | Contexto | Exemplo |
|---------|----------|---------|
| `BRY` | Tarefas gerais da Berry | `BRY-123` |
| `MAIA` | Assistente AI | `MAIA-45` |
| `DEAL` | Sistema de deals | `DEAL-78` |
| `PROJ` | Sistema de projetos | `PROJ-90` |
| `FIX` | Bugs urgentes | `FIX-89` |

### Padrão de Commits

```
[ID-da-tarefa]: <tipo>: <descrição>

Exemplo:
[MAIA-45]: feat: adiciona análise de leads com IA
```

**Tipos aceitos**: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`

### Padrão de Pull Requests

- **Título**: `[ID-da-tarefa]: <tipo>: <descrição>`
- **Aprovações**: 2 obrigatórias
- **Merge**: Squash and Merge
- **Destino**: Sempre `development`

## Stack Tecnológica

### Backend (API)
- Node.js 24+ com TypeScript
- Fastify + Apollo GraphQL
- ArangoDB, Redis, Elasticsearch, Qdrant
- BullMQ para background jobs

### Frontend (App)
- React 19.1.0 com TypeScript
- Vite 6.3.1
- TanStack Router + Query
- Legend State
- Tailwind CSS 4.1.5

## Contribuindo com a Documentação

Encontrou algo desatualizado ou faltando? Siga este processo:

1. Abra uma tarefa no Plane.so com prefixo `BRY`
2. Crie uma branch: `BRY-XXX`
3. Faça as alterações na documentação
4. Crie um PR seguindo os padrões
5. Solicite revisão do Tech Lead

## Contatos

**Dúvidas sobre**:
- **Git Workflow**: Tech Lead
- **Processos**: Tech Lead + Product Owner
- **Código**: Tech Lead + Time de Dev
- **QA**: Time de QA

## Controle de Versão

| Versão | Data | Autor | Mudanças |
|--------|------|-------|----------|
| 1.0 | 2025-01-19 | Tech Lead | Criação inicial da documentação |

---

**Legenda**:
- ✅ Documento completo e revisado
- 🚧 Em desenvolvimento
- 📅 Planejado para próximas sprints
