# Pull Requests

## Criando Pull Requests

Ao concluir o desenvolvimento em uma branch, crie um Pull Request (PR) para integrar as mudanças na branch `development`.

**Passos**:
1. Acesse o repositório no GitHub.
2. Clique em "New Pull Request".
3. Selecione a branch de origem (ex: `ABCD-18`) e a branch de destino (`development`).
4. Preencha o título e a descrição do PR, incluindo:
   - **Título**: `[TICKET_ID]: Resumo claro da mudança.`
   - **Descrição**: Detalhes sobre o que foi feito, por que foi feito e qualquer informação adicional relevante.
   - **Tarefas**: Liste as tarefas concluídas neste PR.
   - **Screenshots**: Inclua screenshots de quaisquer mudanças relevantes, principalmente para confirmação visual e mudanças de UI.
   - **Referências**: Links para issues ou tickets relacionados.
5. Volte ao Ticket/Issue e preencha estas informações:
   - **(WWW)**: O que foi feito, por que foi feito e qual problema foi resolvido?
   - **Passos para testar**: Descrição básica dos passos para executar um teste manual
   - **Critérios de validação**: Liste claramente o que deve ser validado
6. Mova para o estágio `Code review` e envie o link do Ticket/Issue na ferramenta de comunicação no respectivo canal do projeto

**Referências cruzadas**:
- Para o padrão de commits, naming de branch, método obrigatório de merge (*Squash and Merge*), rebase/force-with-lease e exceções de hotfix, consulte `git-workflow.md`.
- Para severidade/comunicação de revisão, regra das 2 aprovações e checklist do revisor, consulte `code-review.md`.
- Para prioridades de `Changes Requested` e status do fluxo (Code Review, Testing, Approved), consulte `task-management.md`.

**Exemplo de Descrição**:
```
// No pull request:

### Descrição
Adiciona funcionalidade de autenticação do Google, permitindo que usuários façam login usando suas contas Google.

### Screenshots

![Screenshot 1](https://example.com/screenshot1.png)
![Screenshot 2](https://example.com/screenshot2.png)

### Resolve/Fecha Tarefa/Issue
Fecha [ABCD-18](task_link)
```

```
// No ticket:

### O que foi feito, por que foi feito e qual problema foi resolvido?
- Corrigido erro vermelho ao tentar acessar bottom sheet de Transferência de Pontos;
- Corrigido erro de comunicação com a api ao tentar transferir pontos.

### Passos para testar
- Acesse [link] e faça login
- clique no item de menu da topbar: a > b > c
- preencha o formulário com as informações mínimas necessárias
- tente enviar

### Critérios de validação
- Deve validar os campos do formulário adequadamente por obrigatoriedade ou formato de dados
- Não deve habilitar o botão se os campos obrigatórios não estiverem preenchidos
- Deve enviar se preenchido
- Se enviado, deve redirecionar para a lista e o registro deve estar presente
```

## Revisando Pull Requests

Todos os PRs devem ser revisados por pelo menos dois membros da equipe antes de testar e fazer merge. A revisão deve focar em:

- **Qualidade do Código**: Garantir aderência aos padrões de codificação.
- **Funcionalidade**: Garantir que a funcionalidade atende aos requisitos.
- **Testes**: Confirmar a existência e cobertura dos testes.
- **Documentação**: Garantir que a documentação está atualizada.

**Checklist de Revisão**:
- [ ] Código segue as convenções do projeto.
- [ ] Sem bugs óbvios.
- [ ] Testes foram adicionados ou atualizados.
- [ ] Documentação foi atualizada conforme necessário.
- [ ] O PR resolve o issue/ticket referenciado.

## Merge

Após a aprovação do PR, o merge deve seguir estas diretrizes:

- **Squash e Merge**: Use "Squash and Merge" para consolidar todos os commits do PR em um único commit na branch `development`. Este commit deve seguir a convenção de mensagem com o ID do ticket.
- **Rebase**: Alternativamente, use rebase para manter um histórico linear, garantindo que a mensagem do commit principal inclua o ID do ticket.
- **Proteção de Branch**: Configure proteções na branch `development` para exigir revisões antes do merge e garantir que a suíte de testes passe. 