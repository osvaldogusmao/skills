---
name: dev-portais-frontend-legado
description: "Convenções do frontend LEGADO dos Portais SGU (repos SGU3.0 em git.fesc.io: empresa, movimentacao, operadora, captura, administracao_*), em manutenção até o front novo, que terá skill própria. Vue 2.6 (Options API, JavaScript) + Vuex 3 + Vue Router 3 + Vue CLI 4, UI Bulma 0.5.3 pré-compilado + componentes <uni-*> de @unimed/unijs, HTTP via Vue.api/this.$api da unijs, app standalone com EasyAuth ou módulo UMD do shell SGU (sem single-spa). Consome os backends legados dos portais (GraphQL SPQR + REST), sem padrão novo nem refatoração ampla. Use ao criar ou alterar tela, rota, módulo Vuex, service/URL de API ou estilo nesses repos. NÃO usar para o backend dos portais (dev-portais-backend-legado), para RESSUS/NEXDOM DS (Vuetify, single-spa: dev-vue3-microfrontend) nem para o CRM (dev-crm-frontend). O repo beneficiario/frontend é exceção de stack (Vuetify 1.5, CLI 3)."
---

# Frontend legado dos Portais SGU (Vue 2 + Bulma + @unimed/unijs)

Front em manutenção: vai ser substituído por um front novo. Trabalho aqui é correção pontual e evolução pequena, seguindo o contrato do backend (ver "Consumir API"; backend: `dev-portais-backend-legado`). Não introduza padrão novo, lib nova ou refatoração além do necessário. Se este guia divergir do arquivo que você está editando, siga o vizinho mais próximo no mesmo repo. Referência principal: `empresa/frontend` e `movimentacao/frontend`, os mais ativos.

## Stack e repositórios

Comum aos portais de negócio: `vue` 2.6.10, `vuex` 3.0.0, `vue-router` 3.0.1, `@vue/cli-service` 4.0.0 (webpack 4), ESLint 5 + Prettier 1.18, `node-sass` 4.11, `moment` 2.19, Bulma 0.5.3 (o CSS vem pronto de `@unimed/unijs/static/bulma.css`, carregado no `index.html`, junto com o fontawesome). O CI builda com `node:10.16`. Não há lockfile (`preinstall: npm config set package-lock false`), então as versões transitivas variam. `@unimed/*` vem de registry privado.

| Repo | Pacote | Papel | unijs |
|---|---|---|---|
| `empresa/frontend` | `@unimed/pem` | Portal Empresa, standalone com login próprio, rotas `/pem/...` | 2.8.0 (+ sguweb-components) |
| `movimentacao/frontend` | `@unimed/amc` | módulo de auditoria de movimentos, carregado pelo shell SGU, rotas `/movimentacao/...` | 2.8.0 (+ sguweb, shared-components) |
| `operadora/frontend` | `@unimed/operadora` | standalone com login, mesmo desenho do empresa | 2.16.0 |
| `captura/frontend` | `@unimed/captura` | standalone (`main.js` usa `App.dev` + `router/index.dev`) | 1.1.84 |
| `administracao_{empresa,operadora,beneficiario}` | `@unimed/portal-administracao-*` | módulos do shell, com store mínima | 2.6.0 |
| `beneficiario/frontend` | `beneficiario-portal` | **exceção**: Vuetify 1.5 (`<v-*>`), CLI 3, `vue-axios`, `vue-apollo` declarado mas sem uso em `src`, aspas duplas, Jest com 1 spec de exemplo | 2.0.200 |
| `sgupeb-components/frontend` | `@unimed/sguweb-components` | **lib**, não portal: `build:lib`, Jest + Cypress, ESLint airbnb + `import/order`, stylelint | — |

## Bootstrap (sem single-spa)

- `vue.config.js` escolhe a config por `BUILD_ENV`. `dev` usa `config/build/dev.conf.js`, com entry `src/dev.js`. `main` e `staging` usam `build.conf.js`, com entry `src/<BUILD_ENV>.js` e saída UMD (`library` = nome do pacote sem `@unimed/`). `config/` e `kubernetes/` são submódulos git (preset de build compartilhado): não edite lá. Aqui não existe `singleSpaVue`, `bootstrap`/`mount` nem `@zitrus/*` externals, ao contrário do CRM.
- **Standalone com login** (empresa, operadora): `main.js` monta com `el: '#app'` e `Vue.use(EasyAuth, { store, router, loginOptions, product: 'portal-empresa' })`. As rotas de login (`views/login/*`) ficam em `routes.js`. No empresa, `validacao-termos-utilizacao.js` entra como `customValidation`.
- **Módulo do shell** (movimentacao, administracao_*): `main.js` não tem `el` e exporta a instância, usa `App.prod.vue` (ou `App.vue`) e `EasyAuth` com `router: null`. O `dev.js` roda isolado com `router/index.dev.js`, que usa `EasyNavigation` como layout.
- Código morto ou quebrado: `empresa/src/staging.js` importa `./App.dev` e `./router/index.dev`, que não existem; `movimentacao/src/router/index.js` importa `./factory`, que não existe. O que vale é `main.js` + `router/routes.js`. Plugin global novo vai em `main.js`, `dev.js` e `staging.js`, na mesma ordem.
- URL de backend: `import { apiDomain } from 'env'` (alias para `config/build/env.js`), resolvido de `localStorage.domains` (`.env.js` em dev, `.env.example.js` copiado para `dist`). Nunca use host fixo.

## Criar ou alterar tela

1. **URL**: função ou constante em `src/api/<recurso>.js`, com default export de objeto, que só monta a string `${apiDomain('portal/empresa')}/<recurso_com_underscore>`. Exemplos: `empresa/src/api/portal-empresa.js` e `api/movimentos.js`; `movimentacao/src/api/api.js`, `api/api-movimentos.js`, `api/provider.js` (exporta `uri`) e `views/parametros/api.js`. Não monte URL dentro do componente.
2. **Chamada**: no empresa, service em `src/services/<recurso>.js` (`export const x = () => Vue.api.get(api.x()).then(({ data }) => data)`). No movimentacao, `createCommandService({ method, uri, payload, onSuccess, onCustomError, onFinally })` de `src/services/index.js`, agrupado por feature como em `views/ocorrencias/service/index.js`.
3. **View**: `src/views/<feature-kebab>/index.vue` com `grid.vue`, `filtro.vue`, `toolbar.vue`, `edit.vue`/`form.vue` e `modal-*.vue`. A raiz é `<uni-section is-page>` com `uni-breadcrumb` (empresa) ou `<amc-breadcrumb slot="breadcrumb">` (movimentacao). Os mixins `EasyForm/EasyGrid/EasyCrud/EasyEntity` e os templates `<t-page-*>` da unijs **não são usados nos portais** (0 ocorrências). Reaproveite os mixins locais de `src/easy/` pelo barrel `@/easy`: `EasyEmpresa`, `EasyContrato`, `EasyDominio`, `EasySidebar`, `EasyOperations` e `EasyVisor` no empresa; `EasyAuditoria` e `EasyGridAuditoria` no movimentacao.
4. **Rota** em `src/router/routes.js`, com import estático e sem guard por rota (a autenticação é do `EasyAuth`). No empresa e no operadora, a rota é filha de `Application` com `path: '/pem/<rota>'`, `name`, `meta: { title }` (o `Application.vue` aplica o `document.title`) e `props: true` quando recebe params. No movimentacao, `path: '/movimentacao/<rota>'`, `name` e `meta: { breadcrumb, doc }`. Use `name` único: o empresa repete `name: 'entrada'`, não repita isso.

## Vuex

- `src/store/index.js` é vazio (`strict` fora de produção). Os módulos são registrados sob demanda no `created` da view ou do mixin local, com `register('<nome>Store', modulo, this.$store)`, e consumidos com `const { mapGetters, mapMutations, mapActions, mapForm } = createNamespace('<nome>Store')`, tudo de `@unimed/unijs/store`. O nome do `register` e o do `createNamespace` precisam ser idênticos. Confira se o nome já existe.
- Módulo em `src/store/<feature>.js` com `namespaced: true, state, getters, mutations, actions` (ex.: `store/contrato.js`, `store/criticas-manuais.js`). Para form e grid simples, use o módulo genérico `crud` da unijs (`views/parametros/edit.vue`; os administracao_* usam só ele). `vuex-map-fields` (`getField`/`updateField`) vem transitivo da unijs.
- Declare o estado inteiro no módulo. Paginação segue o formato `{ page, totalElements, itemsPerPage, totalPages }`. Mutation é síncrona, e o modo `strict` acusa em dev qualquer alteração do state fora dela.

## Componentes e utilitários

- `<uni-*>` são globais via `Vue.use(Unijs)`. Os mais usados: `uni-grid` + `uni-grid-column` (`<template slot-scope="props">`), `uni-button`, `uni-input`, `uni-select`, `uni-checkbox`, `uni-datepicker`, `uni-modal`, `uni-collapse`, `uni-row`, `uni-tabs`/`uni-tab`, `uni-paginate`, `uni-loading` e `uni-icon` (nomes do fontawesome).
- Slots usam a sintaxe antiga `slot="x"`/`slot-scope` (0 `v-slot` em empresa e movimentacao). Siga o arquivo.
- Componente local em `src/components/<Nome>/index.vue` ou `<Nome>.vue`. Onde existe `src/require-components.js` (empresa, movimentacao, operadora, administracao_empresa e administracao_beneficiario), todo `.vue` dessa pasta é registrado **globalmente** com o `name` do componente ou o nome do arquivo. Por isso, declare sempre `name` em PascalCase, senão um `index.vue` vira componente "index". Nos demais repos, importe localmente.
- Componentes de negócio compartilhados são importados por nome: `@unimed/sguweb-components` (`FormularioExclusaoBeneficiario`, `CriaRascunhoExclusaoBeneficiario`) e `@unimed/shared-components` (`VisorBeneficiario`, `DemonstrativoIrModal`), este só nos repos que o declaram (o empresa não declara). Alterar um deles exige mudar a lib `sgupeb-components` e subir a versão no portal, o que fica fora do escopo sem pedido.
- Feedback: `this.$toast.error/success/warning/info` ou `this.$toast2({ title, description, status })`. No movimentacao, `toast()` de `@/utils/toast`. Datas com `moment` e `@/utils/date.js`. Constantes em `src/constants/index.js` (UPPER_SNAKE, ex.: `PARAMETROS_ID`, `sistemaApi`), nunca número mágico.
- `?.`/`??` só dentro de `<script>` (o empresa e o movimentacao declaram `@babel/plugin-proposal-optional-chaining`). Nunca use no `<template>`: o compilador do Vue 2.6 não aceita, e o código tem 0 ocorrências.

## Consumir API: GraphQL e REST

**Como o GraphQL é chamado hoje** (para seguir o padrão):
- Endpoint `${apiDomain('portal/empresa')}/graphql` (constante `graphql` em `api/portal-empresa.js` e `api/movimentos.js`) ou `${apiDomain('movimentacao')}/graphql` (`graphQLServiceURI` em `api/provider.js`).
- Query montada como template string com valores interpolados, chamada com `Vue.api.graphql(apiPortal.graphql, query).then(({ data }) => data.data.<query>)`. Aparece em `services/movimento.js`, `services/beneficiario.js`, `services/complemento.js` e direto em views (ex.: `views/dashboard/importacoes/index.vue`, `views/movimentacoes/sidebar/index.vue`). Para achar tudo: `grep -rn "\.graphql(" src`.
- No movimentacao, `createGraphQLService({ queryString })` devolve `{ payload, error, isSuccess }` via `resolveGraphqlResponse`, e as queries ficam em `*.query.js` (`views/ocorrencias/service/`).
- Paginação no GraphQL: `page: ${page - 1}, size, order: "id"`, com retorno `{ content, totalElements }`.

**Qual contrato seguir**
- GraphQL é contrato vivo, mas desde 2025 nenhuma query nova foi criada. Funcionalidade nova é REST. Tela que já usa GraphQL continua com a query dela: campo novo entra na lista de campos da query existente (o backend só acrescenta o campo no tipo devolvido), sem criar query nova e sem migrar a tela para REST sem pedido.

**REST**
- URL no `api/*.js` e chamada em `Vue.api`/`this.$api` (empresa) ou `createCommandService` (movimentacao). O contrato vem do controller do backend: rota, método, parâmetros, DTO e obrigatoriedade. A chamada é escrita à mão.
- `$api.get(url)` devolve a resposta axios, com o corpo em `data` (e não em `data.data`). `List` chega como array. O `Page` do Spring chega como `{ content, totalElements, totalPages, number, size }`, com `page` começando em 0 e `sort=campo,asc`. O `uni-paginate` começa em 1, então envie `page - 1`.
- Se o chamado pedir para trocar uma query por REST, os campos podem mudar de nome ou forma: a query escolhia campos, o DTO REST vem completo. Confira o contrato, ajuste o mapeamento da view e remova a query e o service ou a constante `graphql` que ficarem sem uso.
- Exemplo real no empresa, commit `85b909ed`: `services/reajuste-contrato.js` (query `reajustesContrato`) foi removido, e entraram `getReajusteContrato` em `api/portal-empresa.js` e `getReajustesContrato` em `services/portal-empresa.js`.
- Erros: vários services do empresa fazem `.catch(({ response }) => response)`, o que devolve o erro como se fosse dado. Não repita isso: exiba `this.$toast.error`/`Vue.toast.error` e retorne um valor explícito ou relance o erro. No movimentacao, o padrão é `applyCatchError` (`src/interceptors/index.js`, que são wrappers de promise e não interceptors do axios), usado por default no `createCommandService`.
- Não importe `axios` direto nem crie outra instância. A única exceção existente é `movimentacao/src/services/index.js`, no caminho `bearerTokenCustom`.

## Estilo

- Ordem `<template>` → `<script>` → `<style lang="scss" scoped>` (padrão majoritário).
- Layout com as classes do Bulma 0.5.3: `columns`/`column is-N`, `level`, `title is-6`, `has-text-primary`, `is-mobile`. Utilitários de página em `@/style/page-style.scss` (`page-full-height`, `page-padding`) no movimentacao e nos administracao_*.
- Os portais não têm arquivo de tokens de cor (diferente do `color.scss` do CRM). O `.sass-lint.yml` tem `no-color-literals` só como aviso, e o sass-lint não roda em script nem no CI. Prefira as classes do Bulma a hex novo e, se precisar de cor, reaproveite a já usada na feature. Não tente sobrescrever variáveis do Bulma: o CSS já vem compilado.

## Qualidade, testes e commits

- `npm run lint` (`vue-cli-service lint --no-fix`, com `plugin:vue/essential` + `@vue/prettier`; Prettier com `singleQuote` e `trailingComma: all`). Hoje passa limpo em empresa e movimentacao. `npm run lint:fix` corrige formatação. `build:main` e `build:staging` rodam o lint antes e falham com erro. No empresa, `dev.conf.js` tem `lintOnSave: false` ("TEMP"), então o dev server não acusa: rode o lint na mão.
- **Não há testes nos portais de negócio**: `@vue/cli-plugin-unit-jest` está nas devDependencies, mas não há script `test` nem spec. Só `sgupeb-components` (`npm run test:unit`, `test:e2e`) e `beneficiario` (`npm run test:unit`) têm Jest. Criar teste no portal é decisão nova: combine antes.
- O hook `commit-msg` (husky → `node node_modules/@unimed/commitlint`) exige `^(feat|fix|refactor)\((product|tech)\): (adicionado|alterado|removido|corrigido) <o quê> para <objetivo>$`, tudo minúsculo, com até 100 caracteres (`Merge branch` passa). Ele não usa o `commitlint.config.js` local (que lista outros types e 120 caracteres). Nunca adicionar `Co-Authored-By:`, `Claude-Session:`, `Generated with` ou equivalente. Por isso `PRT-IXXX: ...` é recusado: o chamado vai na branch (`feature/PRT-I<n>`, `fix/PRT-I<n>`) e no MR (template em `.gitlab/merge_request_templates/`). `npm run commit` abre o git-cz.

## Regras do time

- **Menor alteração segura**: entender o fluxo e seguir o padrão do módulo; sem mudança cosmética junto de correção funcional; sem subir versão, trocar lib ou introduzir TypeScript, Composition API, `@vue/composition-api`, Pinia ou lib de formulário.
- **Vue 2 / Options API**: `data` como função, com todas as propriedades declaradas de início; `computed` para valor derivado e sem efeito colateral; `watch` só para efeito colateral inevitável, observando a propriedade específica em vez de `deep`; lógica de `created`/`mounted` delegada a métodos nomeados; `mounted` só quando depende do DOM; limpeza de listener, timer e subscription em `beforeDestroy` (nunca `beforeUnmount`/`unmounted`).
- **Reatividade**: chave nova em objeto reativo com `this.$set`, remoção com `this.$delete`, ou troca do objeto/array inteiro; nunca alterar índice ou `length` de array direto.
- **Props e eventos**: prop com tipo, obrigatoriedade e default (objeto/array por função); nunca mutar prop nem copiá-la para `data` só para espelhar; comunicação com o pai por evento; `v-model` em componente próprio usa `value` + `input` (não `modelValue`).
- **Template**: expressão simples, condição composta vira `computed`; sem método pesado no template; nunca `v-if` e `v-for` no mesmo elemento; `:key` estável do domínio (não índice quando a ordem muda); `v-html` só com conteúdo confiável; `ref` em vez de acesso direto ao DOM.
- **Estado e API**: estado local fica no componente; store só para o que é compartilhado, sem duplicar a mesma fonte de verdade; mutation síncrona, assíncrono em action; chamada HTTP na camada de `api/` + `services/`, não espalhada no componente; em busca concorrente, resposta antiga não sobrescreve a nova; sem fallback para formatos de payload que o contrato não tem; autorização é do backend, nunca do front.
- **Formulário e UX**: estados de loading, vazio, erro e sucesso tratados; botão de envio bloqueado enquanto a requisição roda; valores do usuário preservados quando a requisição falha; validação num lugar só (o backend é a validação canônica); erro exibido perto do campo; layout sem largura fixa que cause overflow em mobile ou zoom.
- **JavaScript**: `===`/`!==`; `null`, `undefined`, `''`, `0` e `false` tratados de forma consciente; sem mutar objeto ou array compartilhado.
- **Coesão e nomes**: uma decisão num lugar só; condição com mais de um termo vira `computed` ou método com nome de domínio; não remontar valor que já chega pronto; nada de `data`, `item`, `obj`, `temp` quando há nome melhor; não duplicar lógica, validação ou utilitário; comentário só quando explica o porquê.
- **Erros e dados**: não ocultar erro, não logar token, senha ou dado pessoal.
- **Editar arquivo existente**: ao adicionar import, inserir só a linha nova, sem reordenar o bloco; conferir se o nome de método/componente já existe antes de criar; edição em lote por script exige conferir o diff.

## Anti-patterns neste legado

- Criar query GraphQL nova, manter GraphQL ao lado de REST "por garantia" ou ler `data.data` em resposta REST.
- Montar URL no componente ou fixar host em vez de usar `apiDomain` e `api/*.js`.
- Copiar padrão do CRM ou de outra stack: `singleSpaVue`, `@zitrus/*`, `<t-page-*>`, `EasyCrud`/`EasyForm`, `color.scss`, Vuetify fora do `beneficiario`.
- Editar `staging.js` ou `router/index.js` achando que vale em produção, ou editar os submódulos `config/` e `kubernetes/`.
- Componente em `src/components` sem `name` (quebra o registro global) ou `name` de rota duplicado.
- `?.` no template, `v-slot` misturado num arquivo que usa `slot-scope`, ou alterar o state fora de mutation.
- `.catch(({ response }) => response)`, `axios` direto ou nova instância HTTP.
- Refatorar tela inteira, trocar lib ou subir versão de `@unimed/*` para resolver um chamado pontual.
