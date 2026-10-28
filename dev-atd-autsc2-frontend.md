---
name: dev-atd-autsc2-frontend
description: "Frontend Vue do AUTSC2: a SPA em frontend/ (Vue 3, TypeScript, Vite, Pinia, Vue Router, Axios, VeeValidate/Zod), publicada direto em /autsc2/app/, sem iframe e sem app.do. Cobre features e rotas por módulo com meta.recurso, cliente @/core/http com cd_menu/cd_modulo, estado de grid na query (page/pageSize/sort/direction), navegação entre telas Vue e para telas .do, permissões na UI com v-permissao e usePermissao, componentes compartilhados, fidelidade visual ao Magneto, acessibilidade, JSDoc e testes Vitest. Use para criar ou alterar tela, rota, store, cliente de API, componente ou teste no frontend do autsc2. NÃO cobre o backend REST (dev-atd-autsc2-backend), as páginas CodeCharge (dev-atd-codecharge) nem o frontend do CMAGNET, que roda em iframe."
---

# AUTSC2 — frontend Vue

Uma SPA só, em `frontend/`, publicada **direto** em `/autsc2/app/<rota>`. Não há wrapper, iframe nem `app.do`: o menu aponta para a rota Vue, e o `SpaFallbackFilter` atende deep link e refresh. O build (`vue-tsc` e `vite build`) gera `ccs/app/`, que fica fora do git. O Maven roda só `npm install` e `npm run build`, então **o CI não roda o Vitest**.

## Leia antes

Primeiro `docs/padroes-vigentes.md`, que é o índice normativo e define a ordem de autoridade dos docs. Depois, só o doc do tema:

| Tarefa | Doc |
|---|---|
| Rotas, features, HTTP, componentes, estado de grid | `docs/padroes-vigentes.md` (Organização de frontend, Estado de grid na URL, Componentes) |
| Navegação entre telas Vue e para `.do`; lista → detalhe | `docs/agente-migracao-codecharge-vue.md`, seção "Rotas e Integração entre Telas Vue" |
| Permissão na UI | `docs/permissoes-granulares-frontend.md` |
| Menu (`DS_URL_APP`, `meta.recurso`) | `docs/integracao-menu-spa.md` |
| Acessibilidade e fidelidade visual | `docs/acessibilidade-frontend.md` |
| Controle CodeCharge → Vue | `docs/de-para-codecharge-vue-rest.md` |
| Roteiro prático e checklist | `docs/guia-proximas-migracoes.md` |

Os helpers do SGU-Card para wrapper e iframe **não se aplicam aqui**: `app.do`, `resizeReporter`, `ccsStyle`, `navigateTop`, `replaceTopAppUrlWithDynaHash` e a modal aberta sobre `app.do`. O catálogo de componentes do `padroes-vigentes.md` e o "sem suíte Vitest" do doc de permissões estão atrás do código: confira `frontend/src/shared` antes de afirmar que algo falta.

## Estrutura

- **`frontend/src/`:**
  - `core/` é a infraestrutura global: `auth/permissoes`, `http`, `legacy/legacyBridge.ts`, `menu/menuContext.ts`, `theme`;
  - `shared/` guarda o que é reutilizável de verdade;
  - `modules/<modulo>/` tem `index.ts`, `routes.ts` e `features/`. Hoje existem `admin`, `auditoria`, `consultas`, `setup` e `utilitarios`.
- **Feature:**
  - simples: `features/<feature>/{api,components,pages,store,validations,types.ts}`;
  - composta: `core/` mais as subfeatures no primeiro nível;
  - modal: `modules/<modulo>/modal/<nome>` (ex.: `utilitarios/modal/relatorio-situacao`).
- **Rotas:**
  - ficam no `routes.ts` do módulo; subrota mais específica vem antes da rota-pai;
  - o nome da rota é pontuado e espelha o caminho (`admin.fases-autorizacao.niveis`);
  - toda rota de negócio tem `meta.title`, `meta.requiresAuth: true` e, quando há permissão, `meta.recurso`. O `meta.recurso` é a mesma chave do `@RequiresPermission` e de `AUTSC2_MENUS.DS_CHAVE_PERMISSAO`;
  - o `DS_URL_APP` do menu precisa bater **exatamente** com a rota (kebab-case, prefixo `/autsc2/app`, sem barra final). Rota desconhecida cai no catch-all, que redireciona para `/consultas/pacotes`;
  - nunca crie `main.ts`, `index.html` ou SPA por tela.

## Contratos

- **HTTP:**
  - use só `@/core/http` (o alias `@core/http` também funciona), com paths `/rest/api/v1/...`;
  - nada de `fetch`, cliente próprio ou URL absoluta;
  - o interceptor acrescenta `cd_menu` e `cd_modulo`, que o `menuContext.ts` lê da URL e guarda no `sessionStorage`. Nunca os monte no `api/*.ts`;
  - o `api/*.ts` desempacota `response.data?.data` com fallback (`items: []`, `total: 0`, `totalPages: 0`) e nunca vaza `AxiosResponse` nem o envelope;
  - mensagens de erro vêm de `extractApiError` ou `getHttpErrorMessage` (`@/core/http/errors`);
  - **401 e 403 levam ao `/autsc2/Login.do`**. Por isso, ação sem permissão deve ficar bloqueada na tela (ver "Permissão na UI").
- **Estado de grid na URL** (modelo: `useFasesAutorizacaoStore`):
  - a store declara o mapa de query params e quais params ela gerencia, para não apagar `cd_menu`/`cd_modulo` ao reescrever a URL;
  - filtro usa o nome legado quando existe (`s_CD_FASE`);
  - paginação e ordenação usam `page`, `pageSize`, `sort` e `direction`, diferente do SGU-Card, que usa `pagina`;
  - o `sort` lido da URL é validado contra a whitelist que espelha os `@SortColumn` do DTO; valor inválido volta ao padrão sem quebrar a tela.
- **Navegação:**
  - **Vue → Vue:** `RouterLink` ou `router.push` com rota nomeada. Para `<a>` comum (nova aba), use `router.resolve(...)`.
    - Preserve `cd_menu`/`cd_modulo` e os nomes de parâmetro legados que forem contrato (`CD_PACOTE`, `CD_UNIMED`).
    - A tela de destino carrega pela rota e pela própria API, sem depender da store da origem.
    - No retorno lista → detalhe, a URL é a fonte de verdade do estado.
  - **Vue → tela ainda legada:** link `.do` com `BASE_URL`. O `BASE_URL` serve só para assets, login e telas legadas, nunca para navegar dentro da SPA.
  - **Popup ou callback legado:** a ponte mínima fica em `core/legacy/legacyBridge.ts`. O tipo de callback global vai em `src/core/legacy/window.d.ts`, que **ainda não existe**: crie-o quando a primeira tela precisar, sem espalhar `declare global` pelas telas.
- **Integração entre telas:** se a tela que você está migrando é destino de link, botão ou popup de uma tela Vue já migrada, troque o `.do` da origem pela rota Vue. Faça isso na mesma entrega e documente o contrato novo nas duas telas.

## Permissão na UI

O mapa vem de `GET /rest/api/v1/menu/permissoes`, carregado uma vez no `main.ts` e guardado só em memória. É UX: quem garante a segurança é o backend.

- **`v-permissao="'INSERT'"`:** bloqueia a ação, sem tirar o elemento do DOM: põe `aria-disabled`, esmaece, barra o clique e mostra tooltip. O recurso vem de `meta.recurso`; a forma explícita é `{ recurso, funcao }`.
- **`usePermissao().can('UPDATE')`:** para renderizar diferente, como link quando pode editar e texto quando não pode.
- **Ação dentro de componente compartilhado:** só por prop opt-in (ex.: `link-register-permissao` da `GridPaginada`), nunca com `v-permissao` fixo no componente.
- **Tokens:** use `ACCESS`, `INSERT`, `UPDATE` e `DELETE`. Sem `meta.recurso` ou sem o mapa carregado, a ação fica bloqueada (fail-closed).

## Componentes compartilhados

Verifique `frontend/src/shared` antes de criar outro:
- **Grid e paginação:**
  - `GridPaginada` tem o modo `pages` (com total) e o `adjacent` (só `hasNext`/`hasPrevious`), o link "Inserir Novo" (`link-register`) e a prop `link-register-permissao`;
  - passe `link-register-query` com a query atual, senão o link perde `cd_menu`/`cd_modulo`;
  - junto dela: `PaginacaoGrid` (slot `#acoes`), `ColunaOrdenavel` e `usePaginacao`.
- **Controles:**
  - `LegacyDatePicker` exige `id` e usa o formato `dd/MM/yyyy`;
  - `LegacyTextInput`, `LegacyButton` e `LegacyIcon`;
  - `SelectField` para `ListBox` simples e `AutocompleteBusca` para lista grande, no estilo Select2;
  - `CarteirinhaInput`.
- **Outros:** `WizardPassos`, `PainelRetratil`/`usePainelRetratil`, `RodapeVersao`, `useTermos` (termos `{term:...}` com fallback) e as diretivas `v-mask-telefone` e `v-tooltip`.
- **Componente que ainda falta** (checkbox, radio, textarea, upload): crie em `shared/components` copiando o do SGU-Card **sem adaptar**. Adaptar cria um fork silencioso.
- **JSDoc do componente:**
  - bloco acima do `defineProps` (propósito, o contrato legado que encapsula e o que não faz) com `@example` real;
  - JSDoc em cada prop e `defineEmits` com `@event`;
  - `defineOptions({ inheritAttrs: false })` com `v-bind="$attrs"` quando encapsular controle nativo;
  - prefira `export interface <Componente>Props`.

## Migração não é redesign

- **Visual Magneto:** reproduza a estrutura, as classes e o espaçamento do HTML legado, inclusive o `&nbsp;` entre botões. O tema é carregado por `core/theme` (`/Themes/<tema>/Style.css`).
- **`id` e `name`:** preserve todo `id` legado e todo `name` que faça parte de contrato. Não invente `id` nem `name` que o legado não tinha.
- **Nome acessível:** todo controle visível tem nome acessível; ação só com ícone leva `aria-label`; não introduza `<label>` onde o tema estiliza `label`.
- **Formulário:** VeeValidate e Zod, com schema e composable em `validations/` da feature.
- **Lista grande:** `shallowRef`, `v-if` em bloco retrátil e linhas montadas em blocos. Modelos: `RelatorioSituacaoSecao.vue` e `useAcompanhamentoParamsStore.ts`.
- **Fica como pendência:** a validação manual no navegador (DOM, teclado, foco, tema). Registre no resumo e no `docs/pages/<modulo>-<tela>.md`.

## Texto e JSDoc

- **Encoding:** `.vue`, `.ts` e `.js` em UTF-8 sem BOM, com texto humano acentuado. Isso inclui `meta.title`, `aria-label`, fallbacks, mensagens e as strings verificadas em teste. Identificadores, rotas e params ficam sem acento.
- **Varredura antes de concluir:** procure `Ã|Â|\\u[0-9a-fA-F]{4}|&[A-Za-z]+;` nos arquivos alterados.
- **JSDoc:** em português.
  - Em `core`, `router` e `shared`: `@param`, `@returns` e `@example` nos contratos reutilizáveis.
  - Em `modules`: `@param` e `@returns` nas funções com regra.

## Testes e validação

- **Stack:** Vitest 2 com jsdom, configurado no `vite.config.ts`, e `@vue/test-utils`. Specs ficam em `__tests__/`, ao lado do código.
- **Mínimo esperado:**
  - schema e validação;
  - helper ou componente compartilhado criado ou alterado;
  - rota e query de grid com estado na URL;
  - texto com a string acentuada final.
- **Comandos, da raiz do repositório:**
  - `npm --prefix frontend test` (`vitest run`). Obrigatório, porque o CI não roda;
  - `npm --prefix frontend run build` (`vue-tsc --noEmit` e `vite build`, que escreve em `ccs/app/`, fora do git). Pega erro de tipo;
  - sem `node_modules`, rode antes `npm --prefix frontend ci` (precisa de rede).
- **Lint:** não há configuração de ESLint no repositório, então o `npm run lint` não funciona. Não crie a configuração fora do plano; registre a limitação.

## Regras

| ID | Nível | Regra | Como verificar |
|---|---|---|---|
| ATD-AUF-001 | MUST | Tela entra como rota no `routes.ts` com `meta.title`, `meta.requiresAuth` e, com permissão, `meta.recurso` igual a `DS_CHAVE_PERMISSAO`; sem wrapper, iframe ou SPA por tela | diff |
| ATD-AUF-002 | MUST | HTTP só por `@/core/http`; `api/*.ts` desempacota `data` com fallback; sem `cd_menu`/`cd_modulo` manual | diff e teste |
| ATD-AUF-003 | MUST | Estado de grid na query (`page`, `pageSize`, `sort`, `direction`), preservando params de terceiros e validando `sort` | teste de rota |
| ATD-AUF-004 | MUST | Navegação Vue → Vue por rota nomeada preservando contexto de menu e params legados; destino migrado na mesma entrega atualiza a origem | diff e doc das telas |
| ATD-AUF-005 | MUST | Ação sem permissão bloqueada por `v-permissao`/`usePermissao` (o 403 leva ao login) | diff |
| ATD-AUF-006 | MUST | Migração preserva visual Magneto, `id` legados e `name` de contrato, com nome acessível em todo controle | comparação com o HTML legado |
| ATD-AUF-007 | MUST | UTF-8 sem BOM e texto humano acentuado | varredura |
| ATD-AUF-008 | MUST | `test` e `build` verdes, rodados por você | comandos |
| ATD-AUF-009 | SHOULD | Componente que falta é copiado do SGU-Card sem adaptar, com JSDoc completo | diff |
| ATD-AUF-010 | SHOULD | Lista grande com `shallowRef`, `v-if` e montagem em blocos | teste |

## Fora deste papel

- **Endpoint novo ou alterado:** papel `dev-backend` (`dev-atd-autsc2-backend`). No plano, vem antes do front.
- **Menu apontando para a rota Vue** (`AUTSC2_MENUS`: `DS_URL_APP`, `FG_USA_APP`, `DS_CHAVE_PERMISSAO`): changeset no central do `cmagnet-util` (`dev-atd-liquibase`). Até ele ser aplicado, a API da tela responde 403.
- **Página `.do` que precisa mudar:** papel `dev-codecharge` (`dev-atd-codecharge`).

## NUNCA

- Nunca use helper de wrapper ou iframe do SGU-Card, nem crie página `.do` intermediária para abrir tela Vue.
- Nunca use Vuetify ou outro UI kit, nem biblioteca nova sem registrar o motivo.
- Nunca use `BASE_URL` para navegar dentro da SPA.
- Nunca tire do DOM a ação sem permissão quando o padrão é bloquear com `v-permissao`.
