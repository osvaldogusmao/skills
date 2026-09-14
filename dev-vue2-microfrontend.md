---
name: dev-vue2-microfrontend
description: "Convenções de desenvolvimento frontend Vue 2 usadas nos produtos NEXDOM (ex. RESSUS, NEXDOM DS) — cada produto é um micro-frontend registrado via single-spa (single-spa-vue) dentro de um shell comum, com Vuex para estado, Vue Router, Vuetify 2 para componentes visuais, Tailwind para utilitários, bibliotecas internas @zitrus/* compartilhadas pelo shell, ESLint (airbnb) + Prettier, testes unitários com Jest e e2e com Cypress. Use sempre que for implementar, revisar ou corrigir código frontend Vue desses produtos — mesmo que o pedido não mencione \"single-spa\" ou \"micro-frontend\" explicitamente."
---

# Frontend Vue 2 (micro-frontend) — produtos NEXDOM

## Stack

- Vue 2.6 (não Vue 3), Vue CLI 5, Vuex 3 (estado), Vue Router 3.
- Vuetify 2 como base de componentes visuais (Material Design); Tailwind CSS para ajustes utilitários pontuais ao lado do Vuetify — não recrie com Tailwind um componente que o Vuetify já oferece.
- TypeScript está disponível no projeto (`tsconfig.json`), mas o código é majoritariamente JavaScript — não force conversão para `.ts` de arquivos existentes sem pedido explícito.

## Arquitetura: isto é um micro-frontend, não uma SPA standalone

O projeto **não roda sozinho** — ele é registrado dentro de um shell (host) via [single-spa](https://single-spa.js.org/), usando `single-spa-vue`. Isso muda regras que, num Vue app comum, não existiriam:

- `src/main.js` **precisa** exportar `bootstrap`, `mount` e `unmount`, gerados por `singleSpaVue({...})` — nunca remova, renomeie ou contorne esse contrato, é o que o shell usa para carregar/descarregar o módulo.
- `vue.config.js` precisa manter `configureWebpack.output.libraryTarget: 'system'` (formato SystemJS, exigido pelo single-spa) e listar em `externals` as bibliotecas que o shell já fornece (`@zitrus/*`, `vue`, `vue-router`, `vuetify`, etc.) — **nunca remova uma entrada de `externals` nem adicione uma dependência nova sem avaliar se ela deveria ser compartilhada pelo shell**; bundlar algo que o shell já fornece duplica código e pode gerar conflito de versões em runtime.
- Variáveis como `RESSUS_URL`, `WORKSPACE_URL`, `STORAGE_URL`, `ZPERMISSION_URL`, etc. são injetadas em build time via `webpack.DefinePlugin` a partir de variáveis de ambiente — nunca hardcode essas URLs no código.

## Plugins e serviços compartilhados

- Autenticação/permissões vêm do pacote interno `@zitrus/utils-service` (`UserPlugin`, `UserAppPermissionPlugin`) — nunca implemente checagem de permissão ou parsing de token própria; use os plugins já registrados em `main.js`.
- Chamadas HTTP usam a instância axios já configurada em `services/instanceAxios` (`Vue.prototype.$http`) — nunca crie uma instância axios nova solta num componente.

## Estrutura de pastas

- `views/`: telas/rotas (organizadas por área, ex.: `processos`, `cadastros`, `relatorios`, `painel`).
- `components/`: componentes de UI reutilizáveis (subpastas para componentes complexos com múltiplos arquivos).
- `business-components/`: componentes com lógica de negócio embutida — diferente de `components/`, que deve ficar mais genérico/visual.
- `store/`: módulos Vuex, um por domínio (padrão `store/<dominio>/`).
- `api/` e `services/`: chamadas HTTP e integrações.
- `mixins/`, `plugins/`, `constants/`, `utils/`, `events/`: conforme o nome sugere — não crie um novo local para o mesmo tipo de conteúdo.

## Padrões de UI e formulário

- Componentes visuais: Vuetify primeiro; Tailwind só para utilitário pontual.
- Máscaras de input: `v-mask` (campos gerais) e `v-money` (valores monetários) — não implemente máscara manual com regex.
- Modais/overlays entre componentes distantes na árvore: `portal-vue`.
- Assinatura digital/PKI, quando aplicável: `web-pki` — não implemente fluxo de assinatura próprio.

## Qualidade

- ESLint com `@vue/airbnb` + Prettier + `eslint-plugin-vuejs-accessibility` (regras de acessibilidade fazem parte do padrão, não são opcionais) — rode `npm run lint` antes de finalizar e corrija os apontamentos, não desabilite a regra via comentário sem justificativa forte.
- Commits seguem Conventional Commits, com commitlint validando (`npm run cm` para o fluxo assistido via commitizen).
- Testes unitários: Jest + `@vue/test-utils` (`npm run test:unit`, `npm run test:unit:coverage` para cobertura). Testes e2e: Cypress (`npm run cy:run`/`cy:open`), com relato via ReportPortal quando configurado — nunca remova um teste e2e existente sem substituí-lo por outro que cubra o mesmo fluxo.
