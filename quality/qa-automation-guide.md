# 🤖 QA Guide Automation - Documentação

## 📋 Visão Geral

Este documento descreve o sistema de automação que gera guias de teste QA automaticamente quando um Pull Request é merged.

**Workflow:** [`.github/workflows/qa-guide-on-merge.yml`](../.github/workflows/qa-guide-on-merge.yml)

---

## 🎯 O Que Faz

Quando um PR é **merged** no repositório:

1. ✅ **Detecta** o evento de merge automaticamente
2. ✅ **Coleta** todos os commits do autor do PR
3. ✅ **Analisa** os diffs e mudanças de código
4. ✅ **Gera** um guia completo de testes QA usando Claude AI
5. ✅ **Posta** o guia como comentário no PR
6. ✅ **Salva** o guia como artifact no GitHub Actions

---

## 🔧 Como Funciona

### Trigger

```yaml
on:
  pull_request:
    types: [closed]
```

O workflow é acionado quando um PR é **fechado**. Ele verifica se foi um merge real:

```yaml
if: github.event.pull_request.merged == true
```

### Etapas do Workflow

#### 1. **Checkout do Código**
```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0  # Pega todo o histórico
```

#### 2. **Coleta Informações do Autor**
- Nome do autor
- Email do autor
- Login do GitHub

#### 3. **Lista Todos os Commits do PR**
Usa `git log` para pegar commits entre a branch base e a branch do PR:
```bash
git log origin/main..origin/feature-branch --author="author@email.com"
```

#### 4. **Gera Análise Detalhada**
Para cada commit:
- Hash, mensagem, data
- Lista de arquivos modificados
- Estatísticas (linhas adicionadas/removidas)
- Diff do código (primeiras 200 linhas)

Tudo salvo em `commits-analysis.md`.

#### 5. **Claude AI Analisa e Gera Guia**
O arquivo `commits-analysis.md` é enviado para Claude com um prompt estruturado que pede:
- Executive summary
- Onde testar (URLs, módulos)
- Casos de teste detalhados
- Priorização
- Checklist de aceitação
- Edge cases

#### 6. **Posta no PR**
O guia gerado é postado como comentário no PR usando GitHub Script.

#### 7. **Salva como Artifact**
O guia fica disponível para download por 90 dias nos artifacts do workflow.

---

## 📝 Estrutura do Guia Gerado

O guia de QA inclui:

### 1. Sumário Executivo
- Total de commits
- Features implementadas
- Áreas afetadas
- Impacto geral

### 2. Onde Testar
- URLs específicas
- Módulos do sistema
- Permissões necessárias
- Ambiente de teste

### 3. Casos de Teste Detalhados
Formato padrão:
```markdown
##### TC-XXX: [Título do Teste]
**Objetivo:** O que valida

**Passos:**
1. Passo 1
2. Passo 2
3. Passo 3

**Resultado Esperado:**
- ✅ Resultado 1
- ✅ Resultado 2

**Dados de Teste:**
- [Dados necessários]

**Prioridade:** 🔴 Alta | 🟡 Média | 🟢 Baixa
```

### 4. Priorização
- 🔴 **Alta:** Funcionalidades críticas
- 🟡 **Média:** Importantes mas não bloqueantes
- 🟢 **Baixa:** Melhorias, polish

### 5. Checklist de Aceitação
Critérios para aprovar o PR.

### 6. Edge Cases
Casos de borda identificados automaticamente.

---

## 🚀 Como Usar

### Para Desenvolvedores

**Ao criar um PR:**
1. Faça commits claros e descritivos
2. Siga as convenções de commit do projeto
3. Documente mudanças significativas

**Quando o PR for merged:**
1. Aguarde ~2-5 minutos para o guia ser gerado
2. Revise o guia postado no PR
3. Adicione informações faltantes se necessário
4. Notifique o time de QA

### Para QA Engineers

**Após o merge:**
1. Acesse o PR mergido
2. Encontre o comentário "🧪 Guia de Testes QA - Gerado Automaticamente"
3. Leia o guia completo
4. **Priorize testes 🔴 Alta Prioridade primeiro**
5. Siga os passos exatamente como descritos
6. Marque checkboxes ao completar
7. Reporte bugs encontrados

**Se o guia estiver incorreto:**
- Comente no PR mencionando o autor
- Peça esclarecimentos
- Gere manualmente se necessário

---

## ⚙️ Configuração

### Secrets Necessários

O workflow usa este secret (já configurado):
- `ANTHROPIC_API_KEY`: API key do Claude AI

### Permissões

O workflow precisa de:
```yaml
permissions:
  contents: read          # Ler código
  pull-requests: write    # Postar comentários
  id-token: write         # Autenticação Claude
```

### Customização

#### Alterar Timeout
```yaml
timeout_minutes: '15'  # Padrão: 15 minutos
```

#### Modificar Prompt
Edite a seção `direct_prompt` no workflow para ajustar o formato do guia.

#### Filtrar por Branch
Adicione condição:
```yaml
if: |
  github.event.pull_request.merged == true &&
  github.event.pull_request.base.ref == 'main'
```

#### Notificar Time de QA
Adicione step:
```yaml
- name: Mention QA Team
  run: |
    gh pr comment ${{ github.event.pull_request.number }} \
      --body "@qa-team 👆 Por favor revisar o guia acima"
```

---

## 📊 Monitoramento

### Ver Execuções
1. Acesse: **Actions** tab no GitHub
2. Selecione workflow: **Generate QA Testing Guide on PR Merge**
3. Veja histórico de execuções

### Logs
Cada step tem logs detalhados:
- Commits coletados
- Análise gerada
- Resposta da IA
- Status de postagem

### Artifacts
Baixe os guias gerados:
1. Entre na execução do workflow
2. Scroll até "Artifacts"
3. Download: `qa-guide-pr-XXX.zip`

---

## 🐛 Troubleshooting

### Guia Não Foi Gerado

**Possíveis causas:**
1. PR não foi merged (apenas fechado)
2. Nenhum commit do autor encontrado
3. Erro na API do Claude
4. Timeout (>15 minutos)

**Solução:**
1. Verifique logs do workflow
2. Confirme que PR foi merged
3. Verifique API key do Claude
4. Gere manualmente se necessário

### Guia Incompleto ou Incorreto

**Possíveis causas:**
1. Commits sem mensagem descritiva
2. Mudanças muito complexas
3. Falta de contexto no código

**Solução:**
1. Revise e complemente o guia manualmente
2. Melhore mensagens de commit
3. Adicione comentários no código
4. Ajuste o prompt do workflow

### Workflow Não Executou

**Possíveis causas:**
1. Workflow desabilitado
2. Permissões insuficientes
3. Erro de sintaxe no YAML

**Solução:**
1. Verifique se workflow está ativo em Settings > Actions
2. Confirme permissões do workflow
3. Valide YAML com linter

---

## 💰 Custos

### Claude API
Cada execução consome tokens:
- **Input:** ~5k-15k tokens (commits + diffs + prompt)
- **Output:** ~3k-8k tokens (guia gerado)
- **Custo estimado:** $0.15-0.50 por PR (depende do tamanho)

### GitHub Actions
- **Minutos:** ~3-5 minutos por execução
- **Storage:** Artifacts mantidos por 90 dias
- **Limite:** 2000 minutos/mês (plano Free) ou ilimitado (plano pago)

---

## 📈 Melhorias Futuras

### Curto Prazo
- [ ] Adicionar testes de regressão automatizados
- [ ] Integrar com ferramentas de teste (Playwright, Cypress)
- [ ] Notificações no Slack/Discord
- [ ] Templates de guia por tipo de mudança

### Médio Prazo
- [ ] Análise de cobertura de código
- [ ] Geração de dados de teste automaticamente
- [ ] Link para ambientes de staging
- [ ] Checklist interativo no PR

### Longo Prazo
- [ ] Integração com sistema de QA tracking
- [ ] Machine learning para melhorar prompts
- [ ] Geração de testes automatizados
- [ ] Dashboard de métricas de qualidade

---

## 📚 Recursos

### Documentação
- [GitHub Actions](https://docs.github.com/en/actions)
- [Claude API](https://docs.anthropic.com/claude/reference/getting-started-with-the-api)
- [GitHub Script](https://github.com/actions/github-script)

### Exemplos
- [Exemplo de guia gerado](../joao-pacheco-qa-testing-guide.md)
- [Análise de commits](../joao-pacheco-commits-report.md)

### Contato
- **Time de DevOps:** Para problemas com o workflow
- **Time de QA:** Para feedback sobre os guias
- **Autor Original:** @macpedro

---

## 🎓 FAQ

### P: O guia é sempre preciso?
**R:** Não. É gerado por IA e pode ter erros. Sempre revise e ajuste conforme necessário.

### P: Posso desabilitar para alguns PRs?
**R:** Sim. Adicione label `skip-qa-guide` e modifique o workflow para checar labels.

### P: Funciona para PRs de forks?
**R:** Depende. Pode ter limitações de permissão. Teste antes.

### P: Quanto tempo leva?
**R:** Normalmente 2-5 minutos. Pode chegar a 15 minutos em PRs grandes.

### P: Posso usar outra IA?
**R:** Sim! Substitua o step do Claude por OpenAI, ou outra IA de sua escolha.

### P: E se não tiver CLAUDE.md?
**R:** O workflow funciona sem. O CLAUDE.md apenas adiciona contexto do projeto.

---

## ✅ Checklist de Implementação

- [x] Workflow criado (`.github/workflows/qa-guide-on-merge.yml`)
- [x] Secret `ANTHROPIC_API_KEY` configurado
- [x] Permissões do workflow configuradas
- [x] Documentação criada
- [ ] Primeiro teste com PR real
- [ ] Ajustes no prompt baseado em feedback
- [ ] Time de QA treinado
- [ ] Processo documentado no handbook

---

## 📝 Changelog

### v1.0.0 - 2025-10-09
- ✨ Versão inicial do workflow
- ✨ Geração automática de guia QA
- ✨ Postagem em comentário do PR
- ✨ Salvamento como artifact
- ✨ Tratamento de erros
- ✨ Documentação completa

---

**Última atualização:** 09/10/2025
**Versão:** 1.0.0
**Autor:** Claude Code + @macpedro
