# Pull Requests

## Criando Pull Requests

Ao concluir o desenvolvimento em uma branch, crie um Pull Request (PR) para integrar as mudanças na branch `development`.

**Passos**:
1. Acesse o repositório no GitHub
2. Clique em "New Pull Request"
3. Selecione a branch de origem (ex: `ABCD-18`) e a branch de destino (`development`)
4. Preencha o título e a descrição do PR conforme template abaixo (em inglês)
5. Volte ao Ticket/Issue e preencha: WWW, passos para testar, critérios de validação
6. Faça um code review no seu próprio Pull Request e corrija o que encontrar
7. Mova para o estágio `Code review` e envie o link no canal do projeto

---

## Template de PR

### Título
```
[TICKET_ID]: tipo: Resumo claro da mudança
```

### Descrição

```markdown
[Explicação clara do que foi implementadoo]


## Screenshots/GIFs

[OBRIGATÓRIO para mudanças visuais]

## Notas Adicionais
[Decisões técnicas, débitos técnicos, etc]
Closes Task [MAIA-89](Link para a task)
```

---

## Exemplo Completo

**No Pull Request:**
```markdown
Adds Google authentication functionality, allowing users to log in using their Google accounts.

![Screenshot 1](https://example.com/screenshot1.png)

Closes [ABCD-18](task_link)
```

**No Ticket da tarefa:**
```markdown
### O que foi feito, por que foi feito e qual problema foi resolvido?
- Corrigido erro ao tentar acessar bottom sheet de Transferência de Pontos
- Corrigido erro de comunicação com a API ao tentar transferir pontos

### Passos para testar
- Acesse [/link] e faça login
- Clique no item de menu: a > b > c
- Preencha o formulário e tente enviar

### Critérios de validação
- Deve validar os campos obrigatórios
- Deve redirecionar para a lista após envio
```

---

## Referências

> Para detalhes completos, consulte os documentos específicos:

- **Padrão de commits, branches e merge**: [git-workflow.md](./git-workflow.md)
- **Processo de revisão e aprovações**: [code-review.md](./code-review.md)
- **Prioridades e status do fluxo**: [task-management.md](./task-management.md)
