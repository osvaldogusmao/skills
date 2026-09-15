---
name: dev-vue3-microfrontend
description: "Convenções de desenvolvimento frontend Vue 3 usadas nos produtos NEXDOM mais recentes (confirmado no NEXDOM DS) — Composition API com TypeScript, Pinia para estado, Vue Router 4, Vuetify 3, build com Vite (não Vue CLI/Webpack), testes unitários com Vitest e e2e com Cypress, registrado como micro-frontend via single-spa-vue v3. Não confundir com a skill nexdom-vue-microfrontend-dev (Vue 2 + Vuex + Vuetify 2 + Vue CLI/Webpack + Jest, usada em produtos mais antigos como o RESSUS) — são gerações diferentes e incompatíveis entre si. Use sempre que for implementar, revisar ou corrigir código frontend desses produtos mais novos."
---

# Frontend Vue 3 (micro-frontend, geração nova) — produtos NEXDOM

> Esta skill é para a geração **nova** dos frontends NEXDOM (Vue 3). Para produtos mais antigos (ex.: RESSUS), veja a skill `nexdom-vue-microfrontend-dev` (Vue 2) — as duas stacks não são intercambiáveis: Options API vs Composition API, Vuex vs Pinia, Vuetify 2 vs 3, Vue CLI/Webpack vs Vite, Jest vs Vitest. Confirme qual geração o projeto usa (olhe `package.json`: se tiver `vite.config.ts` e `pinia`, é esta) antes de aplicar qualquer convenção.

## Stack

- Vue 3 (Composition API, `<script setup>`/composables — não Options API), TypeScript como linguagem padrão (não apenas disponível — os arquivos são `.ts`/`.vue` com `<script setup lang="ts">`).
- Pinia para estado (não Vuex) — uma store por domínio em `store/<dominio>.ts`, usando `defineStore`.
- Vue Router 4, Vuetify 3 (Material Design, API diferente da v2 — não copie snippets da v2).
- Build com **Vite** (`vite.config.ts`) — não Vue CLI/Webpack. Testes unitários com **Vitest** (`vitest.config.js`, ambiente `jsdom`) — não Jest. E2e com Cypress.
- Lógica reutilizável via **composables** (`src/composables/use<Algo>.ts`, ex.: `useValidateCpf.ts`) — é o equivalente Vue 3 dos mixins do Vue 2; prefira composable a mixin.

## Arquitetura: micro-frontend via single-spa-vue v3

Assim como a geração anterior, isto **não roda sozinho** — é registrado num shell via single-spa. Mas a API de integração é diferente da v2:

```ts
const vueLifecycles = singleSpaVue({
  createApp,
  appOptions: {
    render(this: SingleSpaProps) {
      return h(AppView, { name: this.name, mountParcel: this?.mountParcel, singleSpa: this?.singleSpa });
    },
  },
  handleInstance: (app) => {
    // registro de plugins (Pinia, Vuetify, router, etc.) acontece AQUI, não em appOptions
    app.use(createPinia());
    app.use(router);
    app.use(Vuetify);
  },
});

export const { bootstrap, mount, unmount } = vueLifecycles;
```

- Registre plugins dentro de `handleInstance`, não tente passar `store`/`vuetify` direto em `appOptions` como na v2 — a v3 usa `app.use(...)` explicitamente.
- **Nunca remova ou renomeie** a exportação `bootstrap`/`mount`/`unmount` — é o contrato que o shell espera.

## Compartilhamento de dependências: diferente da geração Vue 2

Esta é a diferença mais fácil de errar: **não externalize `vue`, `vue-router`, `vuetify`, `pinia`** no `vite.config.ts` — ao contrário da geração Vue 2 (que externaliza tudo pro shell prover), aqui **cada micro-frontend empacota sua própria cópia** dessas libs. Motivo provável: outros micro-frontends no mesmo shell ainda podem estar na geração Vue 2, então não dá pra compartilhar uma instância global de Vue entre versões incompatíveis. Só externalize bibliotecas genuinamente compartilhadas e independentes de versão de framework, como `@zitrus/utils-service`:

```ts
build: {
  rollupOptions: {
    external: ['@zitrus/utils-service'],
    output: { format: 'system', globals: { '@zitrus/utils-service': '@zitrus/utils-service' } },
  },
},
```

Antes de adicionar uma nova entrada em `external`, confirme que a lib é realmente compartilhada pelo shell — errar aqui quebra o build ou duplica dependência sem necessidade.

## Estrutura de pastas

- `views/`: telas/rotas. `layouts/`: layouts compartilhados entre views.
- `components/`: componentes de UI. `business-components/`: componentes com lógica de negócio.
- `composables/`: lógica reutilizável (`use*`). `store/`: stores Pinia por domínio. `types/`: tipos TypeScript compartilhados.
- `api/`/`services/`: integrações HTTP. `config/`, `plugins/`, `utils/`.

## Qualidade

- TypeScript: rode `type-check` (script do `package.json`) antes de finalizar — não ignore erro de tipo com `// @ts-ignore` sem justificativa.
- Lint/format conforme os scripts do projeto (`lint`, `lint:fix`, `format`).
- Testes: unitários com Vitest (`npm run test`, `npm run coverage` para cobertura, `npm run test:snapshot` para snapshots) — não confunda com `test:unit` (nome usado na geração Vue 2/Jest). E2e com Cypress (`npm run cy:run`/`cy:open`).
- Commits seguem Conventional Commits (`npm run commit`, husky/commitizen via `prepare`).
