# Code Review na Berry

O objetivo do code review na Berry é garantir **qualidade**, **manutenibilidade** e **segurança** do código, além de **compartilhar conhecimento** entre o time.

## Papéis no code review

- **Autor**
  - Abre o PR seguindo o guia de PR.
  - Explica claramente a implementação técnica.
  - Garante que testes relevantes foram escritos e estão passando.
  - Responde comentários, ajusta o código e marca conversas como resolvidas.

- **Reviewer**
  - Lê a tarefa para entender o contexto do problema ou proposta.
  - Lê o PR com foco no impacto da mudança, não só em detalhes de sintaxe.
  - Usa os níveis de severidade abaixo para dar feedback claro.
  - Cobra testes, clareza e aderência aos padrões da Berry.
  - Busca reduzir retrabalho: aponta problemas de arquitetura, não só “nits”.

- **Tech Lead**
  - Atua em todos PRs com foco em arquitetura, segurança, performance crítica e débito técnico.
  - Destrava discussões quando houver divergência entre autor e reviewer.
  - Garante que padrões do time estão sendo seguidos.


## Fluxo de code review

1. **Autor abre o PR** Seguir [pull-requests.md](./pull-requests.md) (template, descrição, checklist, screenshots, etc.).

2. **Revisão técnica**
   - Mínimo de **2 aprovações**.
   - Mesmo que já tenha 2 aprovações, quem acessar deve revisar
   - Pelo menos uma das revisões deve partir de um sêniors para prosseguir
   - Os comentários devem ser claros e não repetir o que já foi comentado anteriormente (se quiser reforçar comente com um +1)
   - Os comentários devem ser adicionados na linha especifica do ponto a ser apontado, ou no topo do arquivo se for algo mais abrangente
   - Ao solicitar `Changes Requested`, o autor deve justificar o motivo nos comentários de forma resumida
   - Lembre-se de elogiar boas implementações ao aprovar

3. **Merge**
   - Após approvals e testes de validação do QA, o PR é mergeado pelo Tech Lead ou um Sênior definido por tal.
   - Se algo crítico aparecer, abrir **novo PR** ou seguir fluxo de **hotfix** 


## Checklist rápido para o revisor

Ao revisar, passe mentalmente por este checklist:

   - A descrição da tarefa deixa claro problema e contexto da situação a ser abordada?
   - A descrição do PR condiz com a solução implementada?
   - Os nomes de variáveis, funções e arquivos são autoexplicativos e não causam dualidades?
   - A lógica aplicada resolve o problema descrito?
   - Essa lógica introduz novos problemas?
   - O código é legível e fácil de entender?
   - Há casos de borda sem cobertura?
   - O código é simples o suficiente? Pode ser dividido em funções menores?
   - Há duplicação desnecessária que poderia virar helper/service?
   - Segue os guias de tecnologia (ex.: TypeScript, Legend, etc.)?
   - Segue padrões de código e arquitetura do projeto?
   - Existem testes unitários/de integração relevantes?
   - Os testes estão claros e focados em comportamento, não implementação?
   - Inputs são validados?
   - Há risco de leaks, N+1 queries, loops desnecessários, etc.?


## Boas práticas de comunicação

- Foque no **código**, não na pessoa.
- Seja **específico**:
  - Em vez de “isso está ruim”, prefira “podemos extrair essa lógica para um helper X por causa de Y”.
- Sugira **alternativas concretas** quando possível (linkar docs internos ajuda).
- Use conversas síncronas (call rápida) quando o PR estiver travado em muitas trocas assíncronas.
- Como autor, evite responder defensivamente:
  - Explique o contexto, reconheça pontos válidos, negocie o que pode ficar para um PR futuro.


## Hotfixes

Para bugs críticos em produção que precisam de correção urgente:

- **2 aprovações** são obrigatórias, com prioridade máxima.
- PR deve ser criado direto de `main` para `main`
- Após merge em `main`, fazer merge para `development` também
- Comunicar time imediatamente

