# Code Review na Berry

O objetivo do code review na Berry é garantir **qualidade**, **manutenibilidade** e **segurança** do código, além de **compartilhar conhecimento** entre o time.

> Este guia foca *no ato de revisar código*.  
> Para fluxo completo de PR, veja `processes/pull-requests.md`.  
> Para gestão de tarefas, veja `processes/task-management.md`.
> Para política de merge, rebase e naming de branches/commits, veja `processes/git-workflow.md`.

---

## Papéis no code review

- **Autor**
  - Abre o PR seguindo o guia de PR.
  - Explica claramente o *porquê* da mudança (contexto, problema e solução).
  - Garante que testes relevantes foram escritos e estão passando.
  - Responde comentários, ajusta o código e marca conversas como resolvidas.

- **Reviewer**
  - Lê o PR com foco no impacto da mudança, não só em detalhes de sintaxe.
  - Usa os níveis de severidade abaixo para dar feedback claro.
  - Cobra testes, clareza e aderência aos padrões da Berry.
  - Busca reduzir retrabalho: aponta problemas de arquitetura, não só “nits”.

- **Tech Lead**
  - Atua em PRs sensíveis (arquitetura, segurança, performance crítica).
  - Destrava discussões quando houver divergência entre autor e reviewer.
  - Garante que padrões do time estão sendo seguidos.

- **QA**
  - Valida a funcionalidade no ambiente temporário.
  - Verifica cenários de negócio, regressões e fluxos críticos.
  - Dá o “ok” final para merge quando o processo exigir.

---

## Fluxo de code review

1. **Autor abre o PR**
   - Seguir `processes/pull-requests.md` (template, descrição, checklist, screenshots, etc.).
   - Garantir que o PR é pequeno o suficiente para ser revisado com qualidade.

2. **Revisão técnica**
   - Mínimo de **2 approvals** (a combinar por projeto/time).
   - Comentários usando os níveis de severidade.

3. **Ambiente temporário + QA**
   - PR aprovado dispara deploy em ambiente temporário (quando existir).
   - QA valida cenários de uso e registra o resultado.

4. **Merge**
   - Após approvals + validação QA (quando aplicável), o PR é mergeado seguindo o padrão definido (ex.: *Squash and Merge*).
   - Se algo crítico aparecer depois, abrir **novo PR** ou seguir fluxo de **hotfix** (documentado em `git-workflow.md`/`pull-requests.md`).

---


## Checklist rápido para o revisor

Ao revisar, passe mentalmente por este checklist:

1. **Entendimento**
   - A descrição do PR deixa claro problema → solução?
   - Os nomes de variáveis, funções e arquivos são autoexplicativos?

2. **Correção**
   - A lógica resolve o problema descrito?
   - Há casos de borda importantes sem cobertura?

3. **Qualidade / Manutenibilidade**
   - O código é simples o suficiente? Pode ser dividido em funções menores?
   - Há duplicação desnecessária que poderia virar helper/service?

4. **Padrões da Berry**
   - Segue os guias de tecnologia (ex.: TypeScript, Legend, etc.)?
   - Segue padrões de código e arquitetura do projeto?

5. **Testes**
   - Existem testes unitários/de integração relevantes?
   - Os testes estão claros e focados em comportamento, não implementação?

6. **Segurança e performance (quando aplicável)**
   - Inputs são validados?
   - Há risco de leaks, N+1 queries, loops desnecessários, etc.?

> SLA e prioridade de `Changes Requested` estão em `task-management.md`. Template e pré-requisitos de PR estão em `pull-requests.md`.

---

## Boas práticas de comunicação

- Foque no **código**, não na pessoa.
- Seja **específico**:
  - Em vez de “isso está ruim”, prefira “podemos extrair essa lógica para um helper X por causa de Y”.
- Sugira **alternativas concretas** quando possível (linkar docs internos ajuda).
- Use conversas síncronas (call rápida) quando o PR estiver travado em muitas trocas assíncronas.
- Como autor, evite responder defensivamente:
  - Explique o contexto, reconheça pontos válidos, negocie o que pode ficar para um PR futuro.



