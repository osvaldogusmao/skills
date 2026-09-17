---
name: dev-crm-frontend
description: "Convenções de desenvolvimento frontend do produto NEXDOM CRM (Frontend-Workspace, ecossistema UnimedSC). Stack própria, NÃO compartilhada com RESSUS/NEXDOM DS: Vue 2.6 (100% Options API) + Vue Router 3 + Vuex 3, microfrontend single-spa (single-spa-vue, build SystemJS, libs @zitrus/* externalizadas), UI em Bulma (CSS pré-compilado vindo de @unimed/unijs, não Vuetify) + componentes internos @unimed/unijs/@unimed/sguweb-components/@unimed/forms, HTTP via @unimed/unijs/api (sem axios próprio), WebSocket via SocketIoClientPlugin ou STOMP/SockJS. Use ao criar ou editar componentes Vue, módulos Vuex, rotas, chamadas HTTP/websocket ou estilos neste microfrontend — em tarefas como criar tela/CRUD, criar módulo de store, adicionar rota, consumir API, estilizar componente. NÃO usar para o backend deste produto (ver nexdom-crm-backend-dev) nem para o frontend de outros produtos NEXDOM (RESSUS/NEXDOM DS usam Vuetify, não Bulma, e não têm as libs @unimed/*)."
---

# Frontend NEXDOM CRM (Vue 2 + single-spa)

Convenções reais deste repositório, extraídas do código-fonte — não do padrão "de livro" do Vue 2. Onde um padrão daqui divergir do arquivo que você está editando, siga o vizinho mais próximo: este projeto tem inconsistências históricas conhecidas, listadas abaixo.

## Stack e arquitetura

- Vue 2.6, 100% Options API (sem Composition API em nenhum lugar do projeto), Vue Router 3 (`mode: 'history'`), Vuex 3 + `vuex-persistedstate`.
- **Microfrontend single-spa**: builda como módulo SystemJS (`libraryTarget: 'system'`, `library: '@zitrus/main/crm'`) e depende em runtime de pacotes `@zitrus/*` (ex. `@zitrus/utils-service`), que são `externals` do webpack — **nunca instale/bundle esses pacotes localmente** (não aparecem em `package.json`/`node_modules`, são resolvidos pelo shell).
- UI: **Bulma** (CSS já compilado, vem de `@unimed/unijs/static/bulma.css` — não Vuetify) + componentes internos `@unimed/unijs` (uso massivo, registrado globalmente via `Vue.use(Unijs)`, usados direto no template: `<uni-toolbar>`, `<uni-button>`, `<uni-input>`, `<uni-form>`, `<uni-grid>`...), `@unimed/sguweb-components` e `@unimed/forms` (uso pontual). `@unimed/shared-components` está em `package.json` mas sem uso confirmado — não assuma que está em uso.
- HTTP via `@unimed/unijs/api` (`this.$api` ou `api` importado) — **não existe instância axios própria neste repo**, nunca importe `axios` diretamente. WebSocket via dois mecanismos coexistentes e não integrados: `SocketIoClientPlugin` (`Vue.prototype.$socket`, usado só pelo `OmniChannel`) e um cliente STOMP/SockJS próprio (`SocketService`, usado só por `ProtocoloNotificacao`) — não crie um terceiro.
- Sem testes automatizados (nem unitários nem e2e ativos — job Cypress do CI está comentado). Lint é bloqueante no build; commits validados por commitlint/husky.
- Registries privados por escopo: `@unimed/*` e `@zitrus/*` não estão em registry público (`.npmrc`).

## Entry points e integração single-spa

Três arquivos (`src/main.js`, `src/dev.js`, `src/staging.js`) montam via `singleSpaVue`+`singleSpaCss`, registrando os mesmos plugins globais (`Unijs`, `OmniDirectives`, `Templates`, `EasyAuth`, `UserPlugin`, `SocketIoClientPlugin`). `src/main.js` precisa continuar exportando `bootstrap`/`mount`/`unmount` gerados por `singleSpaVue({...})` — nunca remova/renomeie/contorne esse contrato, é o que o shell usa para carregar/descarregar o módulo. Ao adicionar um plugin Vue global novo, registre-o nos três arquivos, mantendo a ordem relativa existente — não crie um quarto entry point.

`vue.config.js` mantém `output.libraryTarget: 'system'` + `library: '@zitrus/main/crm'` e lista `externals: [/^@zitrus\/.+$/]` — nunca remova uma entrada de `externals` nem adicione dependência nova sem avaliar se deveria ser compartilhada pelo shell (bundlar algo que o shell já fornece duplica código/gera conflito de versão em runtime). Para montar URL absoluta de asset estático em runtime, use `runtimePublicPath()` (`src/utils/public-path.js`) — nunca hardcode um path relativo, pois hospedado via SystemJS o CRM pode estar em qualquer origem.

**Router**: apesar do nome, `src/router/index.dev.js` é usado por **todos** os entry points, inclusive produção (`src/router/index.js` é código morto). Edite `index.dev.js` para mudar guard/layout raiz/redirect padrão — a auth global vem de `configureBeforeEachRouter` (`@zitrus/utils-service`), não crie guard por rota nem use `meta.auth`. `routes.js` importa todos os componentes estaticamente (zero `import()` dinâmico) — não introduza lazy loading isolado numa rota nova, quebraria a convenção sem trazer benefício real.

## Criar uma tela CRUD (view)

1. `src/views/<nome-em-kebab-case>/api.js` estendendo o helper `entity()`/`entity-adm.js`/`entity-cad.js`/`entity-erp.js` com os endpoints do recurso — "entities" são URL builders, **não models**: retornam apenas strings de URL, nunca construa a URL inline no componente.
2. `index.vue` (grid, `<t-page-grid>`) e `edit.vue` (form, `<t-page-edit>` — a mesma tela costuma servir insert e edit), usando mixins `Easy*` de `@unimed/unijs/easy` (`EasyForm`, `EasyGrid`, `EasyCrud`, `EasyEntity`) + mixins locais de domínio, na ordem `mixins: [EasyX, ...domainMixins]`. Para sobrescrever `save`/`insert`/`update`/`remove`/hooks de lifecycle vindos de um mixin `Easy*`, chame a implementação base via `this.super.<metodo>()` — não copie a lógica base.
3. Registre rotas em `src/router/routes.js` (centralizado, views nunca se auto-registram): `path`, `alias` (se houver nome legado em português, ex. `/crm/contas` alias de `/crm/account`), `component`, `name`, `meta: { applicationKey: 'crm', title, doc }`. Sem guard por rota.
4. Se a tela precisa de estado compartilhado, crie `src/views/<nome>/store/` no padrão split-namespaced (ver Vuex abaixo), não o padrão flat legado.

## Vuex — dois padrões coexistem, use split-namespaced para código novo

- **Flat (legado)**: arquivo único, uma mutation genérica `(state, {key, value}) => state[key] = value`, sem `types.js`, sem `namespaced: true` (acessado no namespace global: `mapGetters(['getProtocolo'])`). Só estenda esse padrão para um módulo que já é flat (`protocol`, `account`, `agendamento`); não crie um módulo flat novo.
- **Split-namespaced (padrão para código novo)**: pasta com `actions.js`/`getters.js`/`mutations.js`/`state.js`/`types.js`/`index.js`, `namespaced: true` explícito no `index.js`. Consuma sempre com o nome do módulo: `mapGetters('NomeDoModulo', [...])`/`mapActions('NomeDoModulo', [...])`. **Cuidado**: `volatileFilter` usa essa mesma organização de pastas mas **não** declara `namespaced: true` (é acessado no namespace global apesar da aparência) — não repita essa inconsistência num módulo novo.
- Registre o módulo em `src/store/index.js`. Antes de nomear um módulo de view novo (`src/views/<feature>/store/`), confira se já existe módulo flat homônimo — se existir, use uma chave que não colida (ex. sufixo `View`/plural, como já feito com `ProtocolView`/`Agendamentos`).
- `createPersistedState({ key, paths })` só para dado de filtro/contexto de negócio que deve sobreviver a reload (filtros de listagem, último contexto de navegação) — nunca para estado de UI efêmero (abas abertas, loading) nem dado de auth/token (auth fica em `EasyAuth`, fora do Vuex).
- Não adicione constantes novas em `src/store/types.js` (raiz, não-namespaced) — é resquício do padrão flat antigo; use um `types.js` dentro da pasta do próprio módulo.

## Componentes, mixins, enums

- Componentes reutilizáveis em `src/components/`: majoritariamente `PascalCase.vue` com prefixo `Crm` (ex. `CrmBreadcrumb.vue`), registrados **localmente** (`components: { CrmBreadcrumb }`) — nunca via `Vue.component` global. Só diretivas (`OmniDirectives`, via `Vue.use`) são globais.
- Estrutura do `.vue`: sempre `<template>` → `<script>` → `<style lang="scss" scoped>`, nessa ordem.
- Mixins em `src/mixins/`/`src/views/mixins/` combinam um mixin de `@unimed/unijs/easy` com mixins locais de domínio.
- Enums em `src/enums/modules/<Nome>.js`, um por arquivo, `Object.freeze({ CHAVE: { id, value } })`, barrel `src/enums/index.js`, import `import { Flag } from '@/enums'` — não crie enum inline num componente nem use magic strings.
- Datas: `moment` direto no componente, ou reaproveite um helper local (`helpers/moment.js`) já existente no módulo — não existe util central de formatação de data no projeto, não crie um "definitivo".

## Consumir API e WebSocket

- Construa a URL via `api.js`/entity do recurso e chame com `this.$api.get/post(...)` (ou `api` importado de `@unimed/unijs/api` numa store action). Trate erro com `applyCatchResponseData`/`applyCatchError` (`src/interceptors/index.js`) — são wrappers manuais de promise, **não interceptors reais do axios**.
- Para fluxo mais complexo (precisa de `onSuccess`/erro customizado/`finally`), prefira o padrão `createCommandService` (`src/services/index.js`), já usado por `ProtocoloNotificacao`/`OmniChannel`, em vez de inventar um terceiro padrão de service. Existe também um padrão de Repository (classe, `src/repository/`) para um conjunto coeso de operações com `Filter` de `@unimed/unijs/core` — não recrie como funções soltas se já existir repository para o recurso.
- WebSocket: se o módulo já usa `Vue.prototype.$socket` (`SocketIoClientPlugin`, padrão `OmniChannel`), continue com `this.$socket.on(topic, callback)`. Se já usa o singleton STOMP/SockJS (`ProtocoloNotificacao`), continue com `SocketService.connect()/disconnect()`. Não crie um terceiro mecanismo.

## Estilizar um componente

`<style lang="scss" scoped>`. Para cor de marca (primária, sucesso, alerta, erro), importe `@/style/color.scss` e use a variável (`$primary`, `$green-theme`...) — **nunca hex literal** (regra `no-color-literals` do `.sass-lint.yml`, violada com frequência no código legado — não repita). Para status/feedback (tag/badge), use as classes utilitárias semânticas do Bulma direto no template (`has-background-{success,danger,warning,info}` + `has-text-white`, mapeadas por uma prop) em vez de criar uma variável Sass nova — só declare hex local quando for uma cor que o Bulma não oferece nativamente (ver `CrmStatusTag.vue`). Use grid do Bulma (`columns`/`column is-N`) para layout, não recrie com flexbox custom. O Bulma vem pré-compilado de `@unimed/unijs` — não existe pipeline de override de variáveis Sass do Bulma, não tente criar um.

## Qualidade, commits, CI

- ESLint (`plugin:vue/essential` + `@vue/prettier`) + Prettier (aspas simples, vírgula final, `trailingComma: all`). `npm run lint` é **bloqueante no build** (`build:main`/`build:staging` rodam lint antes do build) — rode antes de finalizar.
- Commitlint valida a mensagem (`commit-msg` hook via Husky, pacote interno `@unimed/commitlint`): scope obrigatório minúsculo (só `product` ou `tech`), type obrigatório minúsculo (`ci`/`docs`/`feat`/`fix`/`refactor`/`test`/`release`), subject sem ponto final e sem sentence/start/pascal/upper-case, máx. 120 caracteres no header. Formato: `type(scope): subject`. Use `npm run commit` (git-cz) para o fluxo assistido.
- CI (GitLab): `build` → `docker` → `deploy`; testes e2e (Cypress) definidos mas comentados, não executam. Sonar só existe como script npm, não roda automaticamente.
- **Não há testes automatizados no projeto** — nem unitário nem componente nem e2e ativo. Se pedirem para escrever testes, não existe convenção prévia a seguir (runner, localização), é decisão nova a tomar explicitamente com o usuário.

## Anti-patterns identificados neste projeto

- Instalar/bundlar um pacote `@zitrus/*` em `package.json` — é sempre externo, resolvido pelo shell single-spa.
- Editar `src/router/index.js` esperando efeito em produção — é código morto, o real é `index.dev.js`.
- Criar um módulo Vuex flat novo do zero, ou criar um módulo split sem declarar `namespaced: true` (repetindo a inconsistência do `volatileFilter`).
- Persistir (`vuex-persistedstate`) estado de UI efêmero ou dado de auth/token.
- Importar `axios` diretamente em vez de `@unimed/unijs/api`/`this.$api`.
- Criar um terceiro mecanismo de WebSocket além de `SocketIoClientPlugin` e STOMP/SockJS.
- Hex literal de cor de marca fora de `color.scss`, ou tentar sobrescrever variáveis Sass do Bulma (o CSS já vem compilado).
- Registrar componente globalmente via `Vue.component` (só diretivas são globais neste projeto).
- Introduzir lazy loading (`import()`) isolado numa única rota nova, quebrando a convenção de import estático do `routes.js` sem trazer benefício real de code-splitting.
