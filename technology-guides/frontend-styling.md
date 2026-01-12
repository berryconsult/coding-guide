# Guia de Estilização Frontend (Maia App)


## TL;DR
- Use **componentes do design system primeiro** (shadcn/ui + wrappers internos).
- Quando precisar de utilitários, prefira **Tailwind**; evite inline styles.
- Cores e espaçamentos: use tokens existentes (`text-slate-900`, `bg-slate-50`, `spacing 1-10`). Não invente hex ad-hoc.
- Estados: sempre trate `hover`, `focus-visible`, `disabled`, `loading`.
- Responsivo: comece mobile (`sm`, `md`, `lg`) e teste em 320px, 768px, 1280px.

---

## Stack e onde estilizar
- **Design System:** shadcn/ui (botões, inputs, cards, dialogs, dropdowns). Reuse antes de criar.
- **Tailwind CSS:** utilitários para layout, espaçamento, tipografia e estados.
- **CSS escopo local:** quando necessário, use módulos ou `style.ts` colocalizado; nunca global sem motivo.
- **Proibido:** `useState` para estado de UI (seguir `legend-app-state.md`), inline style exceto casos raros (ex.: CSS custom property dinâmica).

---

## Princípios
1) **Consistência > Criatividade:** siga tokens e componentes existentes.  
2) **Acessibilidade por padrão:** foco visível, contraste adequado, labels e aria em inputs.  
3) **Mobile-first:** implemente base mobile e eleve com breakpoints.  
4) **Estados claros:** disabled, loading, error, success devem ser visíveis.  
5) **Simplicidade:** prefira poucas classes Tailwind bem escolhidas a CSS customizado.

---

## Como estilizar no dia a dia

### 1) Use o design system primeiro
- Botões: `Button` do DS; variantes primária/ghost/destructive em vez de criar classes novas.
- Inputs/Forms: `Input`, `Textarea`, `Select`, `Label` do DS com mensagens de erro (`FormMessage`).
- Feedback: `Badge`, `Alert`, `Skeleton`, `Toast` do DS quando existir componente pronto.

### 2) Layout com Tailwind
- Grids/listas: `grid`, `gap-4`, `grid-cols-1 md:grid-cols-2`.
- Flex: `flex`, `items-center`, `justify-between`, `gap-2/4/6`.
- Contêiner: `max-w-screen-lg xl:max-w-screen-xl`, `mx-auto`, `px-4 md:px-6`.
- Scrollável: `overflow-y-auto` com `max-h-[calc(100vh-200px)]` quando preciso.

### 3) Tipografia e cores
- Títulos: `text-xl font-semibold md:text-2xl`; corpo: `text-sm md:text-base text-slate-700`.
- Links: `text-primary hover:underline`; manter `focus-visible:outline` do DS.
- Nunca usar hex direto; use classes de cor já definidas (`slate`, `primary`, `destructive`).

### 4) Espaçamento e bordas
- Margens e paddings: escala curta (`1-10`). Evite `mt-1 mb-7` sem padrão; alinhe a blocos de 4px (ex.: `mt-4`, `gap-4`).
- Borda e raio: `border`, `border-slate-200`, `rounded-md` (inputs), `rounded-lg` (cards), `rounded-full` (pills/avatars).

### 5) Estados e interações
- Focus: `focus-visible:ring-2 focus-visible:ring-primary focus-visible:ring-offset-2`.
- Disabled: use `aria-disabled`/`disabled` e classes `opacity-60 pointer-events-none`.
- Loading: use spinners do DS ou `aria-busy="true"`. Nunca deixe o botão clicável enquanto carrega.
- Error/success: cores semânticas (`text-destructive`, `text-emerald-600`), mensagens curtas e abaixo do campo.

### 6) Responsividade
- Base mobile (sem prefixo) → refine com `sm`, `md`, `lg`.
- Layouts de cards: `grid-cols-1 md:grid-cols-2 lg:grid-cols-3`.
- Tabelas largas: `overflow-x-auto` + `min-w-[720px]`.
- Testar em 320px, 768px, 1280px antes de abrir PR.

### 7) Componentes customizados
- Crie variantes com `class-variance-authority (cva)` se já adotado no projeto; mantenha API simples (`variant`, `size`, `state`).
- Coloque estilos próximos ao componente (`Component.tsx` + `component.styles.ts` se necessário).
- Prefira `clsx`/`cn` utilitário para combinar classes em vez de string manual longa.

---

## Exemplos rápidos

### Card com ações
```tsx
<section className="rounded-lg border border-slate-200 bg-white p-4 md:p-6 shadow-sm">
  <header className="flex items-center justify-between gap-3">
    <div>
      <p className="text-sm text-slate-500">Deal</p>
      <h2 className="text-xl font-semibold text-slate-900">Lead XPTO</h2>
    </div>
    <div className="flex items-center gap-2">
      <Button variant="outline" size="sm">Ver detalhes</Button>
      <Button size="sm">Converter</Button>
    </div>
  </header>
  <div className="mt-4 grid grid-cols-1 gap-3 md:grid-cols-3">
    <Stat label="Status" value="MQL" />
    <Stat label="Owner" value="Ana" />
    <Stat label="Score" value="82" />
  </div>
</section>
```

### Campo com erro
```tsx
<FormField
  control={form.control}
  name="email"
  render={({ field }) => (
    <FormItem>
      <FormLabel>Email</FormLabel>
      <FormControl>
        <Input type="email" placeholder="nome@empresa.com" {...field} />
      </FormControl>
      <FormMessage /> {/* usa cor semântica do DS */}
    </FormItem>
  )}
/>
```

---

## Checklist antes do PR (visual)
- Usa componentes do design system onde possível.
- Sem inline style arbitrário; Tailwind ou CSS local bem escopado.
- Estados `hover/focus/disabled/loading` cobertos.
- Responsividade verificada em 320/768/1280.
- Contraste e foco visível ok; inputs com label e aria.
- Screenshots/GIFs anexados no PR para mudanças visuais.

---

## Referências
- Design system (shadcn/ui) do projeto.
- Tailwind docs: https://tailwindcss.com/docs
- Legend state e hooks: `technology-guides/legend-app-state.md`

