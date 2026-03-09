# open-finance-poc — Project Instructions

## Project

- **Description**: TODO
- **Spec**: `conversation/resources/spec.md` (authoritative, when available)

## Execution Framework

### Key Rules

1. **Never start implementation without DoR** — all tasks must be specified with DoD and approved by user before execution begins.
2. **1 Story = 1 worktree = 1 feature-branch = 1 PR = 1 merge** — no exceptions.
3. **Autonomous execution** — after DoR approval, execute all tasks without interruption until PR creation. Only stop for genuine technical blockers.

### Story Workflow

```
1. Planning (collaborative)
   - Assistant proposes story scope + tasks with DoD
   - User refines and approves

2. Execution (autonomous)
   - Worktree created
   - Each task: implement → verify → commit
   - PR created when all tasks pass

3. Review (user)
   - Assistant submits a PR for approval with a manual QA checklist
   - User reviews PR on GitHub
   - User merges the PR or requests changes via PR comments
```

### PR & Handoff to User

Quando a execução autônoma termina e é hora de entregar ao user:

1. **PR title + description proativos** — forneça imediatamente o title e description sugeridos para o PR.
2. **Test plan para QA manual** — checklist que o user consiga executar manualmente.
3. **URL de teste explícita** — inclua a URL completa se aplicável.

### GitHub Credentials

Existem duas camadas de autenticação:

1. **Git operations** (push/pull/clone):
   - Funcionam via VS Code credential helper (injetado pelo devcontainer)
   - Nenhuma configuração necessária

2. **GitHub API** (criar PRs, ler comentários):
   - Use `gh` CLI com o token fine-grained do env `GITHUB_TOKEN`
   - Se `GITHUB_TOKEN` não estiver disponível, peça ao user

3. **Fallback quando `gh` não funciona**:
   - Forneça o link direto: `https://github.com/jco-automations/<repo>/pull/new/<branch>`
   - Inclua title, description e test plan completos na mensagem

### Worktree vs Repo Principal — onde salvar o quê

O worktree é efêmero (removido após merge). Regra:

- **No worktree**: apenas artefatos da story (código, testes, commits)
- **No repo principal**: tudo que transcende a story individual:
  - Insights, learnings, decisions (`conversation/logs/`)
  - Todo-list, memory (`conversation/state/`)
  - Spec-change drafts (`conversation/scratchpad/`)
