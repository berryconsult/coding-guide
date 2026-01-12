# Git Workflow - Berry

## Branches

| Branch | Propósito | Proteção |
|--------|-----------|----------|
| `main` | Produção | Protegida, sem push direto |
| `development` | Integração | Protegida, apenas via PR |

### Nomenclatura

**Formato:** `[ID-DA-TAREFA]` (ex: `MAIA-45`, `DEAL-78`)

```bash
# ✅ Correto
MAIA-45
DEAL-78
AUTH-56

# ❌ Incorreto
feature/lead-analysis
maia_45_lead_analysis
```

### Criar branch

```bash
git checkout development
git pull origin development
git checkout -b MAIA-45
```

---

## Commits

**Formato:** `ID: tipo: descrição`

```bash
# Exemplos
MAIA-45:  Add lead analysis
DEAL-78: Fix CAC (Customer Aquisition Cost) calculation
BRY-12: Update README
```

**Regras:**
- Commits atômicos (uma mudança por commit)
- Verbo no imperativo ("adiciona", não "adicionado")
- Inglês simples e claro
- Máx 72 caracteres

---

## Pull Requests

### Quando criar

- Desenvolvimento concluído
- Testes passando localmente
- Código formatado (prettier, linter)
- Sem conflitos com `development`

### Título

```
MAIA-45: Implement lead analysis
```

### Destino

**Sempre `development`** (nunca direto para `main` (Salvo Hotfixes) )

### Aprovação

- **2 aprovações** obrigatórias
- CI/CD pipeline verde
- Todos os comentários resolvidos/respondidos

> Template e descrição detalhada em [pull-requests.md](./pull-requests.md)

---

## Merge

**Método:** Squash and Merge

```bash
# Mensagem final
MAIA-45 Implement lead analysis

- Add LeadAnalysisService
- Implement OpenAPI Integration
- Add Unit Tests
---

## Hotfixes

Para bugs críticos em produção, use o ID da tarefa:

```bash
# 1. Criar branch de main
git checkout main
git pull origin main
git checkout -b MAIA-89

# 2. Fazer correção mínima
git commit -m "MAIA-89 Fix validation expression"

# 3. PR para main (2 aprovações obrigatórias)

# 4. Após merge de main, sincronizar development
git checkout development
git merge main
git push origin development
```

---

## Conflitos

Em casos de conflitos, usar o merge para resolver com o seguinte método:

```bash
git checkout MAIA-45
git merge origin/development
# Resolver conflitos
git add .
git commit -m "MAIA-45 Fix conflict with Development before merge"
git push origin MAIA-45
```

---

## Troubleshooting

### Em casos de problemas com a branch entrar em contato com o Tech Lead

---

## Checklist antes de PR

- [ ] Branch atualizada com development
- [ ] Testes passando (`pnpm test`)
- [ ] Lint/Prettier aplicados
- [ ] Descrição do PR completa (com Print se envolver mudanças na UI)
- [ ] Screenshots (se mudança visual)

> Checklist completo em [pull-requests.md](./pull-requests.md)
