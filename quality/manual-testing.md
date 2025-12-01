# Guia de Testes Manuais - Berry

**Versão:** 1.0.0
**Última atualização:** 27 de Novembro de 2025

---

## Índice

1. [Introdução](#introdução)
2. [Setup do Ambiente](#setup-do-ambiente)
3. [Matriz de Priorização](#matriz-de-priorização)
4. [Integração com Code Review](#integração-com-code-review)
5. [Módulos de Teste](#módulos-de-teste)
6. [Fluxos End-to-End](#fluxos-end-to-end)
7. [Casos de Erro](#casos-de-erro)
8. [Template de Bug Report](#template-de-bug-report)
9. [Checklist de Aprovação de PR](#checklist-de-aprovação-de-pr)

---

## Introdução

Este documento consolida os **casos de teste manuais** para a plataforma Berry, cobrindo funcionalidades críticas do CRM, projetos, contratos, pagamentos, leilões e integrações.

### Quando Testar

- **Durante Code Review**: Smoke tests em ambiente temporário (Coolify)
- **Antes de Merge**: Testes completos de funcionalidades modificadas
- **Antes de Deploy**: Regressão de fluxos críticos P0

### Legenda

- ✅ **Aprovado**: Teste passou
- ❌ **Reprovado**: Teste falhou, requer correção
- ⚠️ **Atenção**: Sugestão ou comportamento não crítico

---

## Setup do Ambiente

### Ambientes

**Staging (Coolify):**
- URL temporária gerada por PR
- Dados isolados por ambiente

**Credenciais de Teste:**
- Admin: `qa@berry.com.br`
- BDR: `bdr@berry.com.br`
- Franchisee: `franchisee@berry.com.br`

### Dados de Teste

**Stripe:**
- Sucesso: `4242 4242 4242 4242`
- Falha: `4000 0000 0000 0002`
- CVV: Qualquer 3 dígitos

**WhatsApp:**
- Número: `+55 11 99999-0000` (staging)

---

## Matriz de Priorização

### P0 - Crítico
Impedem uso do sistema. Testar **toda sprint**.
- Login e autenticação
- Criação/edição de Deals
- Geração de contratos
- Pagamentos Stripe
- Leilões

### P1 - Alto
Críticas para negócio, mas com workarounds.
- Transições de status CRM
- Tarefas
- WhatsApp
- Dashboards

### P2 - Médio
Melhoram experiência, não bloqueiam.
- Chat interno
- Notificações
- Filtros avançados

### P3 - Baixo
UI/UX e performance.
- Responsividade mobile
- Animações

---

## Integração com Code Review

### Fluxo

```
Code Review (2 approvals) → Ambiente temporário → QA testa → QA aprova → Merge
```

### Quando Executar

**Smoke Tests (5-10 min):**
- Mudanças pequenas (< 100 linhas)
- Apenas happy path

**Testes Completos (20-30 min):**
- Novas features
- Mudanças em lógica crítica (P0)

**Regressão (1-2h):**
- Antes de release
- Mudanças em infraestrutura

---

## Módulos de Teste

---

## ID 00001 - Autenticação & Login

**Teste geral:** P0 - Crítico
**Testes em:** Chrome, Firefox, Mobile

### Passo a passo

1. Acesse ambiente staging
2. Clique em "Login com Google"
3. Selecione conta @berry.com.br
4. Aguarde redirecionamento

**Resultado esperado:** Dashboard carregado com nome/foto do usuário.

### Cases

**00001.1 Caso:** Login com Google OAuth (Happy Path)
**Funcionalidade:** Deve permitir login com @berry.com.br
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Token JWT armazenado, redirecionamento para `/`, nome e foto no header.
**Tarefa:** N/A
**Prints:** _____

---

**00001.2 Caso:** Login com domínio inválido
**Funcionalidade:** Bloquear contas que não sejam @berry.com.br
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Erro "Apenas @berry.com.br podem fazer login", usuário permanece em `/login`.
**Tarefa:** N/A
**Prints:** _____

---

**00001.3 Caso:** Token expirado
**Funcionalidade:** Redirecionar para login quando token expirar
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Após 14 dias, redirecionar para `/login` com mensagem "Sessão expirada".
**Tarefa:** N/A
**Prints:** _____

---

**00001.4 Caso:** Logout do sistema
**Funcionalidade:** Desconectar e limpar sessão
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Remover token do localStorage, redirecionar para `/login`.
**Tarefa:** N/A
**Prints:** _____

---

**00001.5 Caso:** Acesso a rota protegida sem login
**Funcionalidade:** Redirecionar para login se não autenticado
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Ao acessar `/crm` sem login, redirecionar automaticamente.
**Tarefa:** N/A
**Prints:** _____

---

## ID 00002 - CRM & Leads

**Teste geral:** P0 - Crítico
**Testes em:** Chrome, Firefox

### Passo a passo

1. Login com @berry.com.br
2. Menu lateral → "CRM"
3. Visualize pipeline de deals

**Resultado esperado:** Lista de deals por status (Not Started, MQL, SQL, Opportunity, Won, Lost).

### Cases

**00002.1 Caso:** Criação de lead via formulário
**Funcionalidade:** Criar lead e calcular score automaticamente
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Formulário em `/contact` cria deal com status "Not Started", score 1-5 calculado.
**Tarefa:** N/A
**Prints:** _____

---

**00002.2 Caso:** Validação de campos obrigatórios
**Funcionalidade:** Impedir criação sem campos obrigatórios
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Erro em vermelho abaixo de "Nome", "Email", "Mensagem" se vazios.
**Tarefa:** N/A
**Prints:** _____

---

**00002.3 Caso:** Transição Not Started → MQL
**Funcionalidade:** Marcar lead como MQL
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Botão "Marcar como MQL" atualiza status, cria task para BDR, envia notificação.
**Tarefa:** N/A
**Prints:** _____

---

**00002.4 Caso:** Atribuição automática de BDR
**Funcionalidade:** Atribuir BDR com menor carga
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** BDR com menos deals atribuídos automaticamente, nome aparece em "Responsável".
**Tarefa:** N/A
**Prints:** _____

---

**00002.5 Caso:** Transição MQL → SQL → Opportunity → Won
**Funcionalidade:** Permitir avançar deal pelo pipeline
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Cada transição atribui role correto (SDR, Closer), registra no histórico, envia notificações.
**Tarefa:** N/A
**Prints:** _____

---

**00002.6 Caso:** Registrar outcome: Won
**Funcionalidade:** Marcar deal como ganho e gerar contrato
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Status = "Won", contrato gerado, fluxo de pagamento iniciado.
**Tarefa:** N/A
**Prints:** _____

---

**00002.7 Caso:** Filtro e busca de deals
**Funcionalidade:** Filtrar por status e buscar por texto
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Dropdown filtra por status, campo de busca encontra por nome/organização/CNPJ.
**Tarefa:** N/A
**Prints:** _____

---

## ID 00003 - Projetos & Tarefas

**Teste geral:** P0 - Crítico
**Testes em:** Chrome, Firefox

### Passo a passo

1. Login → Menu "Projetos"
2. Visualize lista de projetos

**Resultado esperado:** Projetos com status, cliente, progresso de tarefas.

### Cases

**00003.1 Caso:** Criação automática de projeto após deal ganho
**Funcionalidade:** Deal Won cria projeto automaticamente
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Projeto criado com status "Awaiting Payment", nome da organização, contatos.
**Tarefa:** N/A
**Prints:** _____

---

**00003.2 Caso:** Criação de tarefa manual
**Funcionalidade:** Adicionar tarefa ao projeto
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Modal com Título, Responsável, Data, Prioridade. Tarefa aparece em "To Do".
**Tarefa:** N/A
**Prints:** _____

---

**00003.3 Caso:** Arrastar tarefa no kanban
**Funcionalidade:** Mover entre To Do, In Progress, Done
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Drag and drop atualiza status, registra no histórico com timestamp.
**Tarefa:** N/A
**Prints:** _____

---

**00003.4 Caso:** Criação de subtarefas
**Funcionalidade:** Adicionar subtarefas com checkbox
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Progresso da tarefa (2/5 subtarefas) aparece no card.
**Tarefa:** N/A
**Prints:** _____

---

**00003.5 Caso:** Comentários e anexos
**Funcionalidade:** Adicionar comentários e arquivos
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Comentário com avatar/timestamp. Upload de PDF/imagens para S3.
**Tarefa:** N/A
**Prints:** _____

---

## ID 00004 - Contratos

**Teste geral:** P0 - Crítico
**Testes em:** Chrome, Firefox

### Cases

**00004.1 Caso:** Geração automática após deal ganho
**Funcionalidade:** Criar contrato com dados do deal
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Contrato em `/contracts` com status "Draft", dados preenchidos.
**Tarefa:** N/A
**Prints:** _____

---

**00004.2 Caso:** Edição de valores e cupom
**Funcionalidade:** Editar itens, valores, aplicar cupom
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Total recalculado automaticamente. Cupom válido aplica desconto.
**Tarefa:** N/A
**Prints:** _____

---

**00004.3 Caso:** Envio para assinatura (ZapSign)
**Funcionalidade:** Enviar para assinatura digital
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Email enviado, status = "Pending Signature", link de acompanhamento.
**Tarefa:** N/A
**Prints:** _____

---

**00004.4 Caso:** Webhook de assinatura concluída
**Funcionalidade:** Atualizar status ao receber webhook
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Status = "Signed", invoice gerada no Stripe, fluxo de pagamento iniciado.
**Tarefa:** N/A
**Prints:** _____

---

## ID 00005 - Pagamentos (Stripe)

**Teste geral:** P0 - Crítico
**Testes em:** Chrome, Firefox

### Cases

**00005.1 Caso:** Criação de subscription após assinatura
**Funcionalidade:** Criar subscription no Stripe
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Webhook de contrato assinado cria subscription, envia invoice.
**Tarefa:** N/A
**Prints:** _____

---

**00005.2 Caso:** Pagamento bem-sucedido
**Funcionalidade:** Processar pagamento e ativar projeto
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Cartão `4242...` aprovado, webhook recebido, projeto = "Active", email enviado.
**Tarefa:** N/A
**Prints:** _____

---

**00005.3 Caso:** Pagamento falhado e retry
**Funcionalidade:** Tratar falha e agendar retry
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Cartão `4000 0000 0000 0002` falha, invoice = "Failed", retry em 3 dias.
**Tarefa:** N/A
**Prints:** _____

---

**00005.4 Caso:** Atualização de método de pagamento
**Funcionalidade:** Cliente atualiza cartão
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Modal Stripe Elements salva novo cartão, tenta cobrar invoice pendente.
**Tarefa:** N/A
**Prints:** _____

---

## ID 00006 - Leilões (Auction)

**Teste geral:** P0 - Crítico
**Testes em:** Chrome, Firefox

### Cases

**00006.1 Caso:** Criação automática para deal alto valor
**Funcionalidade:** Deals > R$ 10k geram leilão
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Leilão de 3h criado, lance inicial R$ 1.000, franchisees notificados.
**Tarefa:** N/A
**Prints:** _____

---

**00006.2 Caso:** Fazer lance válido
**Funcionalidade:** Lance >= atual + R$ 100
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Lance registrado, histórico atualizado, notificações enviadas.
**Tarefa:** N/A
**Prints:** _____

---

**00006.3 Caso:** Validação de lance inválido
**Funcionalidade:** Impedir lance < incremento mínimo
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Erro "Lance mínimo deve ser R$ X", impede envio.
**Tarefa:** N/A
**Prints:** _____

---

**00006.4 Caso:** Encerramento e cobrança de CAC
**Funcionalidade:** Após 3h, cobrar vencedor e transferir deal
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Status = "Closed", invoice de CAC criada, deal transferido após pagamento.
**Tarefa:** N/A
**Prints:** _____

---

## ID 00007 - WhatsApp & Chat

**Teste geral:** P1 - Alto
**Testes em:** Chrome, Firefox

### Cases

**00007.1 Caso:** Recebimento de mensagem
**Funcionalidade:** Criar conversa ao receber mensagem
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Mensagem "Olá" cria conversa com status "New", aparece no inbox.
**Tarefa:** N/A
**Prints:** _____

---

**00007.2 Caso:** Resposta automática da Maia
**Funcionalidade:** IA responde automaticamente
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Maia responde em < 5s com boas-vindas, indicador "IA".
**Tarefa:** N/A
**Prints:** _____

---

**00007.3 Caso:** Escalação para humano
**Funcionalidade:** Detectar dúvida complexa e escalar
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Status = "Escalated", BDR notificado, pode assumir conversa.
**Tarefa:** N/A
**Prints:** _____

---

**00007.4 Caso:** Criar deal da conversa
**Funcionalidade:** Gerar deal com dados do WhatsApp
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Deal criado com nome/telefone extraídos, link para conversa.
**Tarefa:** N/A
**Prints:** _____

---

## ID 00008 - Dashboards

**Teste geral:** P1 - Alto
**Testes em:** Chrome, Firefox

### Cases

**00008.1 Caso:** Sales Dashboard
**Funcionalidade:** Exibir métricas de vendas
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Funil de deals, revenue, taxa de conversão, top closers. Carrega em < 3s.
**Tarefa:** N/A
**Prints:** _____

---

**00008.2 Caso:** Filtro de data
**Funcionalidade:** Filtrar por período
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Dropdown (Hoje, 7 dias, 30 dias, Custom). Gráficos recarregam.
**Tarefa:** N/A
**Prints:** _____

---

**00008.3 Caso:** Drill-down em gráfico
**Funcionalidade:** Clicar em estágio mostra lista de deals
**Status:** [ ] ✅ / [ ] ❌
**Comentário:** Modal com lista de deals do estágio clicado.
**Tarefa:** N/A
**Prints:** _____

---

## Fluxos End-to-End

---

### Fluxo E2E 1: Lead → Projeto Completo

**Prioridade:** P0
**Duração:** ~25 min
**Frequência:** Toda sprint

**Cenário:** Lead capturado até projeto ativado.

| # | Ação | Validação | Status |
|---|------|-----------|--------|
| 1 | Preencher formulário `/contact` | Lead criado, score calculado | [ ] |
| 2 | Marcar como MQL | BDR atribuído, task criada | [ ] |
| 3 | Qualificar como SQL | SDR atribuído | [ ] |
| 4 | Converter para Opportunity | Closer atribuído | [ ] |
| 5 | Agendar Sales Meeting | Evento no Google Calendar | [ ] |
| 6 | Marcar como Won | Contrato gerado | [ ] |
| 7 | Enviar para assinatura | Email enviado, status atualizado | [ ] |
| 8 | Simular assinatura (webhook) | Invoice gerada no Stripe | [ ] |
| 9 | Processar pagamento | Pagamento aprovado | [ ] |
| 10 | Verificar projeto criado | Projeto "Active", tarefas criadas | [ ] |

**Critérios de Sucesso:**
- ✅ Todos os passos sem erro
- ✅ Tempo < 30 min
- ✅ 0 erros em console
- ✅ Dados sincronizados entre módulos

---

### Fluxo E2E 2: Leilão de Deal Alto Valor

**Prioridade:** P0
**Duração:** ~20 min

| # | Ação | Validação | Status |
|---|------|-----------|--------|
| 1 | Marcar deal > R$ 10k como Won | Leilão criado (3h) | [ ] |
| 2 | Fazer lance (Franchisee A: R$ 1.500) | Lance registrado | [ ] |
| 3 | Fazer lance (Franchisee B: R$ 2.000) | Franchisee A notificado | [ ] |
| 4 | Encerramento automático | Vencedor determinado | [ ] |
| 5 | Processar pagamento CAC | Invoice paga | [ ] |
| 6 | Transferir deal para vencedor | Deal no workspace do vencedor | [ ] |

---

### Fluxo E2E 3: WhatsApp → CRM

**Prioridade:** P1
**Duração:** ~15 min

| # | Ação | Validação | Status |
|---|------|-----------|--------|
| 1 | Enviar "Olá" para WhatsApp Berry | Conversa criada | [ ] |
| 2 | Verificar resposta da Maia | IA responde em < 5s | [ ] |
| 3 | Enviar mensagem complexa | Maia escala para BDR | [ ] |
| 4 | BDR cria deal da conversa | Deal com dados do WhatsApp | [ ] |

---

## Casos de Erro

**Erro 1:** Rede indisponível
**Validação:** Mensagem "Sem conexão. Dados salvos quando restaurada", retry automático
**Status:** [ ] ✅ / [ ] ❌

---

**Erro 2:** Timeout (> 30s)
**Validação:** Spinner + "Processando...". Após 30s: "Tempo esgotado. Tente novamente"
**Status:** [ ] ✅ / [ ] ❌

---

**Erro 3:** Token expirado durante uso
**Validação:** Renovar automaticamente. Se falhar, redirecionar com "Sessão expirada"
**Status:** [ ] ✅ / [ ] ❌

---

**Erro 4:** Stripe webhook duplicado
**Validação:** Processar apenas 1x (idempotência), sem duplicação no histórico
**Status:** [ ] ✅ / [ ] ❌

---

**Erro 5:** Acesso a deal de outro workspace
**Validação:** HTTP 403, mensagem "Sem permissão"
**Status:** [ ] ✅ / [ ] ❌

---

**Erro 6:** Upload > 10MB
**Validação:** Erro "Arquivo muito grande. Máximo: 10MB", não enviar para S3
**Status:** [ ] ✅ / [ ] ❌

---

## Template de Bug Report

**Título:** [Módulo] Descrição curta

**ID Test Case:** 00XXX.X
**Prioridade:** P0 / P1 / P2 / P3
**Ambiente:** Staging URL, Chrome/Firefox, Data

**Descrição:**
_Descrição clara do bug_

**Passos para Reproduzir:**
1. Passo 1
2. Passo 2
3. Passo 3

**Resultado Esperado:** _O que deveria acontecer_

**Resultado Atual:** _O que aconteceu_

**Evidências:**
- Screenshots: _____
- Logs: _____

**Impacto:**
- [ ] Bloqueia fluxo crítico
- [ ] Perda de dados
- [ ] Afeta UX (tem workaround)
- [ ] Cosmético

---

## Checklist de Aprovação de PR

### Setup
- [ ] Notificação de PR recebida
- [ ] Ambiente temporário funcionando
- [ ] Login realizado
- [ ] Feature/bugfix compreendida

### Testes Funcionais
- [ ] Test cases P0 relevantes executados
- [ ] Happy path testado
- [ ] Edge cases principais testados
- [ ] Validações de formulários OK
- [ ] Permissões (diferentes roles) OK

### UI/UX
- [ ] Chrome funcional
- [ ] Firefox funcional
- [ ] Sem erros de layout
- [ ] Mensagens de erro claras
- [ ] Loading states corretos

### Performance
- [ ] Páginas < 5s
- [ ] Console sem erros críticos
- [ ] Network sem erros 500

### Aprovação
**APROVAR se:**
- ✅ P0 passaram
- ✅ Sem erros críticos
- ✅ Performance OK

**REJEITAR se:**
- ❌ P0 falhou
- ❌ Bug crítico
- ❌ Regressão
- ❌ Performance > 10s

---

**Última atualização:** 27 de Novembro de 2025
**Versão:** 1.0.0
