# Task Management - Berry

## Story Points

**Escala Fibonacci:** 1, 2, 3, 5, 8, 13
~
| Pontos | Esforço | Duração | Exemplo |
|--------|---------|---------|---------|
| **1** | Mínimo | < 1 hora | Ajustar texto, corrigir typo |
| **2** | Baixo | 1-2 horas | Componente simples, validação |
| **3** | Médio | 2-4 horas | Filtro em lista, endpoint CRUD |
| **5** | Alto | 1 dia | Feature completa (UI + backend) |

### Regra: Tarefas > 5 pontos devem ser divididas

```markdown
# Original (13 pontos)
MAIA-100: Implementar análise de leads com IA

# Dividida:
MAIA-101: (3 pts) Criar schema e migration
MAIA-102: (5 pts) Implementar LeadAnalysisService
MAIA-103: (3 pts) Criar endpoint GraphQL
MAIA-104: (2 pts) Adicionar componente UI
```

---

## Status do Fluxo

```
Backlog → To Do → In Progress → Code Review → QA → Approved → Ready to Deploy → Completed
```

| Status | Responsável | Ação |
|--------|-------------|------|
| **Backlog** | PO | Priorizar e refinar |
| **To Do** | Dev | Pegar tarefa |
| **Changes Requested** | Dev | Resolver IMEDIATAMENTE |
| **In Progress** | Dev | Desenvolver (máx 1 por dev) |
| **Code Review** | Reviewers + Lead | Revisar em até 4h |
| **QA** | QA | Testar em até 24h |
| **Approved** | Tech Lead | Tech Lead faz o merge |
| **Ready to Deploy** | Tech Lead | Deploy para produção |

---

## Prioridades

🔴 **Changes Requested** = Prioridade MÁXIMA (resolver em 1h)

> Pausar qualquer outra tarefa para resolver feedback de code review.

---

## Anatomia de Tarefa

### Título
```
MAIA-45 implementa análise de leads
```

### Descrição
```markdown
## Contexto
Por que esta tarefa existe?

## Objetivo
O que queremos alcançar?

## Critérios de Aceitação
- [ ] Critério específico e testável
- [ ] Outro critério
```

### Labels obrigatórias
- **Tipo:** `feature`, `bug`, `refactor`
- **Módulo:** `deals`, `projects`, `maia`
- **Prioridade:** `high`, `medium`, `low`

---

## Regras

### WIP (Work in Progress)
- Máximo **1 tarefa In Progress** por desenvolvedor
- Se bloqueado, comunicar imediatamente os responsáveis

### Estimativas
- Na dúvida, arredondar para cima
- Incerteza alta = pontos maiores ou spike primeiro

### Capacidade
- 50 pontos por sprint por dev
- Considerar: code reviews, meetings, imprevistos (margem de 20 a 30%)

---

## FAQ

**Posso pegar nova tarefa se tenho uma em Code Review?**
- Aguardando aprovação: Sim
- Changes Requested: Não, resolver primeiro
- Aguardando QA: Sim, mas fique disponível

**O que fazer quando bloqueado?**
1. Notificar o Tech Lead
2. Propor soluções adequadas se possível