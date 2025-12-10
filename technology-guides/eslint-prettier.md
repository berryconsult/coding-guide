# ESLint & Prettier - API e App

**Versão:** 1.0.0  
**Última atualização:** 10 de Dezembro de 2025  
**Responsável:** Equipe de Plataforma Berry  
**Objetivo:** Padronizar lint (ESLint 9 + TypeScript-ESLint) e formatação (Prettier 3) na Maia API (Node 24, Fastify) e Maia App (React 19, Vite) de forma rápida e repetível.  
**Escopo:** `packages/api` e `packages/app` no monorepo `berrymax` (pnpm workspaces).

---

## TL;DR (5 passos)
1) Instale as dependências na raiz:  
`pnpm add -D @eslint/js typescript-eslint eslint-config-prettier eslint-plugin-react-hooks eslint-plugin-react-refresh prettier globals`
2) Crie `eslint.config.mjs` na raiz com base compartilhada + overrides de API e App (flat config).
3) Crie `prettier.config.js` + `.prettierignore` compartilhados.
4) Adicione scripts de lint/format em `packages/api` e `packages/app`.
5) VS Code: habilite `editor.formatOnSave` e `source.fixAll.eslint` (settings locais do repositório).

---

## Estrutura esperada
```
berrymax/
├── package.json          # pnpm workspace
├── eslint.config.mjs     # flat config único
├── prettier.config.js
└── packages/
    ├── api/              # Backend Node 24 + TS
    └── app/              # Frontend React 19 + TSX
```

## ESLint (flat config)
Crie `eslint.config.mjs` na raiz.

```js
// ESLint 9 + flat config
import js from '@eslint/js'
import tseslint from 'typescript-eslint'
import eslintConfigPrettier from 'eslint-config-prettier'
import globals from 'globals'
import reactHooks from 'eslint-plugin-react-hooks'
import reactRefresh from 'eslint-plugin-react-refresh'

export default [
  { ignores: ['**/dist/**', '**/build/**', '**/.turbo/**', '**/.cache/**', '**/coverage/**'] },

  // Base JS + TS type-checked
  js.configs.recommended,
  ...tseslint.configs.recommendedTypeChecked,
  ...eslintConfigPrettier,

  // Maia API (Node 24)
  {
    files: ['packages/api/**/*.{ts,tsx}'],
    languageOptions: {
      parserOptions: {
        project: './packages/api/tsconfig.json',
        tsconfigRootDir: new URL('.', import.meta.url),
      },
      globals: { ...globals.node, ...globals.es2024 },
    },
    rules: {
      '@typescript-eslint/explicit-function-return-type': 'error',
      '@typescript-eslint/no-explicit-any': 'error',
      '@typescript-eslint/no-floating-promises': 'error',
    },
  },

  // Maia App (React 19 + Vite)
  {
    files: ['packages/app/**/*.{ts,tsx}'],
    languageOptions: {
      parserOptions: {
        project: './packages/app/tsconfig.json',
        tsconfigRootDir: new URL('.', import.meta.url),
      },
      globals: { ...globals.browser, ...globals.es2024 },
    },
    plugins: { 'react-hooks': reactHooks, 'react-refresh': reactRefresh },
    rules: {
      ...reactHooks.configs.recommended.rules,
      'react-refresh/only-export-components': 'warn',
    },
  },
]
```

Notas rápidas:
- Ajuste os caminhos de `files`/`project` se a estrutura for diferente.
- Para lint rápido (sem type-check), troque `recommendedTypeChecked` por `recommended`.
- Evite regras de estilo duplicadas: `eslint-config-prettier` já desliga as conflitantes.

## Prettier
`prettier.config.js` na raiz.

```js
export default {
  semi: false,
  singleQuote: true,
  trailingComma: 'all',
  printWidth: 100,
  tabWidth: 2,
  bracketSpacing: true,
}
```

`.prettierignore` (sugestão):

```
dist
build
coverage
node_modules
package-lock.json
pnpm-lock.yaml
```

## Scripts recomendados
Em `packages/api/package.json`:

```json
{
  "scripts": {
    "lint": "eslint src --max-warnings=0",
    "lint:fix": "eslint src --fix",
    "format": "prettier --write \"src/**/*.{ts,tsx,js,json,md}\""
  }
}
```

Em `packages/app/package.json`:

```json
{
  "scripts": {
    "lint": "eslint src --max-warnings=0",
    "lint:fix": "eslint src --fix",
    "format": "prettier --write \"src/**/*.{ts,tsx,js,json,md,css}\""
  }
}
```

Opcional na raiz (executa ambos):  
`"lint": "pnpm --filter \"./packages/**\" lint"`

## VS Code (recomendado)
`.vscode/settings.json` na raiz ou por pasta:

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": { "source.fixAll.eslint": true },
  "eslint.useFlatConfig": true
}
```

## CI / pre-commit (opcional)
- `pnpm add -D husky lint-staged` na raiz.
- `.husky/pre-commit` → `pnpm lint-staged`.
- `lint-staged.config.mjs`:

```js
export default {
  '*.{ts,tsx,js,jsx}': ['eslint --fix', 'prettier --write'],
  '*.{md,json,css}': ['prettier --write'],
}
```

## Troubleshooting rápido
- **Parsing error / Cannot read config file**: confira `tsconfigRootDir` e caminho do `project`.
- **Lint lento**: use `recommended` em vez de `recommendedTypeChecked` para checagens sem type-check.
- **Regra duplicada de estilo**: confirme que `eslint-config-prettier` está no final do array.
- **Extensões VS Code sem efeito**: verifique `eslint.useFlatConfig: true` e reabra o workspace.

## Referências
- Stack: Node 24, React 19, TypeScript (guia: `technology-guides/typescript.md`).
- ESLint flat config: https://eslint.org/docs/latest/use/configure/configuration-files-new
- Prettier 3: https://prettier.io/docs/en/options

