---
name: dev-atd-cmagnet-frontend
description: "Frontend Vue do CMAGNET (SGU-Card): a SPA única em sgucard/sgucard-frontend (Vue 3, TypeScript, Vite, Pinia, Vue Router, Axios, VeeValidate/Zod), publicada em /cmagnet/app/ dentro do iframe do ccs/app.html. Cobre features e rotas por módulo, cliente @core/http, navegação Vue ↔ CodeCharge (app.do, .do carimbado, popup, modal Vue em nova janela), grid com estado na query (pagina), componentes Legacy* e fidelidade visual ao Magneto, acessibilidade, JSDoc e testes Vitest. Use para criar ou alterar tela, rota, store, cliente de API, componente ou teste no sgucard-frontend. NÃO cobre o backend REST (dev-atd-cmagnet-backend), as páginas CodeCharge (dev-atd-codecharge) nem o frontend do AUTSC2, que é SPA direta sem iframe."
---

# CMAGNET — frontend Vue

Uma SPA só, em `sgucard/sgucard-frontend`, publicada em `/cmagnet/app/<rota>`. Ela roda dentro do iframe do `ccs/app.html`: cabeçalho e menu continuam no CodeCharge, e o tema chega por `ccsStyle`. O Maven faz o build (`npm install` e `npm run build`), mas **o CI não roda o Vitest**: se você não rodar os testes, ninguém roda.

## Leia antes

Primeiro `docs/migracao-vue/padroes-vigentes.md`, que é o índice normativo e define a ordem de autoridade dos docs. Depois, só o doc do tema:

| Tarefa | Doc |
|---|---|
| Rotas, features, `core`, cliente HTTP, iframe | `docs/arquitetura-front-back-sgucard.md` (seções Frontend Vue, Fluxo HTTP, Integração com legado) |
| Navegação Vue ↔ CodeCharge ↔ `app.do`, `dynaHash`, popups | `docs/migracao-vue/padrao-navegacao-vue-codecharge.md` (resumo: `resumo-navegacao-vue-codecharge.md`) |
| Modal Vue | `docs/migracao-vue/padrao-estrutura-modais-vue-rest.md` |
| Acessibilidade e fidelidade visual | `docs/acessibilidade-frontend.md` |
| Controle CodeCharge → Vue (`Record`, `Grid`, `ListBox`, termos, datas, carteirinha) | `docs/migracao-vue/de-para-codecharge-vue-rest.md` |
| JSDoc | `docs/migracao-vue/padrao-jsdoc-frontend.md` |
| Nomes | `docs/padrao-nomenclatura-codigo.md` (seção Frontend) |
| Lista grande, travamento | `padroes-vigentes.md`, seção "Volume de dados e custo de renderização" |

Tela já migrada (`docs/migracao-vue/pages/**`) é referência funcional, não de estrutura: algumas são anteriores aos padrões atuais.

## Estrutura

- **`src/`:**
  - `src/core` é a infraestrutura global: `auth`, `http`, `iframe`, `legacy`, `navigation`, `theme`;
  - `src/shared` guarda o que é reutilizável de verdade;
  - `src/modules/<modulo>/` tem `index.ts`, `routes.ts` e `features/`. Hoje existem `admin`, `notas-cobranca` e `utilitarios`.
- **Feature:**
  - simples: `features/<feature>/{api,components,pages,store,validations,types.ts}`, criando só o necessário;
  - composta: `core/` mais as subfeatures no primeiro nível;
  - o caminho segue a hierarquia funcional pedida pela demanda (ex.: `admin/cadastros/locais-atendimento`), não o caminho do legado.
- **Rotas:**
  - ficam no `routes.ts` do módulo, que o `index.ts` registra com `router.addRoute`;
  - subrota mais específica vem antes da rota-pai;
  - toda rota de negócio tem `meta.title` e `meta.requiresAuth: true`;
  - nunca crie `main.ts`, `index.html` ou SPA por tela.
- **Modal:** o path tem o segmento `modal` e o nome da rota tem `.modal.`. A pasta `modal/<nome>` fica em `src/shared`, no módulo ou na feature, conforme quem usa a modal.

## Contratos

- **HTTP:**
  - use só `@core/http` (base `/cmagnet`, sessão pelo cookie), com paths `/rest/api/v1/...`;
  - o `api/*.ts` desempacota `response.data?.data` com fallback (`items: []`, `total: 0`, `totalPages: 0`) e nunca vaza `AxiosResponse` ou o envelope;
  - erro vira mensagem por `extractApiError`. O `401` desloga (interceptor); o `403` mostra a mensagem do backend e mantém o usuário na tela;
  - nunca monte `CD_MENU`/`cd_menu` na API da feature: o interceptor propaga o valor da URL.
- **Estado de tela:** Pinia guarda o estado; filtro, página e ordenação vão para a query por `useRouteBackedGridState`.
  - A página é `pagina`, nunca `page`, que é reservado pelo `app.do`.
  - Grid: `GridPaginada`, `PaginacaoGrid`, `ColunaOrdenavel`.
  - Lista grande vai em `shallowRef`, bloco retrátil usa `v-if` e linhas em volume são montadas em blocos.
- **Navegação** (tudo de `@core/navigation`):
  - **Vue → Vue:** rota nomeada, preservando `route.query`.
  - **Vue → CodeCharge:**
    - use `href` (nunca `router.push`), montado com `toCodeChargeUrl`;
    - carimbe com `signCodeChargeUrls`, a não ser que a URL já venha carimbada do backend;
    - use `target="_top"` quando a tela legada substitui a atual.
  - **Popup CodeCharge:** `openCodeChargePopup`.
  - **Modal Vue:** abre com `openVueModalWindow` e fecha com `closeVueModalWindow`; quem abriu recebe o retorno por `registerVueModalResultHandler`, no formato `{action, refresh, payload?, message?}`.
  - **CodeCharge → Vue:** `app.do?page=/cmagnet/app/<rota>`, com filtros fora de `page`.
  - **Callback global para o legado:** o tipo fica só em `src/core/legacy/window.d.ts`. A tela atribui o callback e o restaura ao desmontar.
- **Iframe:** o `resizeReporter` é global. Nunca faça resize por tela.

## Migração não é redesign

- **Visual Magneto:**
  - reproduza estrutura, classes e espaçamento do HTML legado: tabelas de layout, `colspan`, `width`, `nowrap` e o `&nbsp;` entre controles;
  - nada de cards, `flex`, `grid` ou CSS local para "limpar" a tela;
  - radio continua radio e `<input>` de botão continua botão;
  - divergência visual só com motivo, registrada em "Diferenças conhecidas" no doc da tela.
- **`id` e `name`:**
  - preserve todo `id` do legado e todo `name` que faça parte de contrato (POST, query, popup, callback, teste de QA);
  - não crie `id` ou `name` que o legado não tinha;
  - `id` duplicado herdado: documente, não renomeie.
- **Nome acessível:**
  - todo controle visível tem nome acessível, por texto, `aria-labelledby` com id existente ou `aria-label`;
  - não introduza `<label>` onde o tema estiliza `label`;
  - ação só com ícone leva `aria-label`.
- **Componentes compartilhados** (verifique antes de criar outro):
  - controles: `LegacyTextInput`, `LegacySelect`, `LegacyCheckbox`, `LegacyRadioGroup`, `LegacyFileInput`, `LegacyTextarea`, `LegacyDatePicker` e `LegacyButton`;
  - navegação e ícones: `LegacyIcon`, `LinkNavegacao` e `AcaoGrid` (com exatamente um destino, `href` ou `to`);
  - outros: `CarteirinhaInput`, `PainelRetratil` e `useTermos` (`{term:...}` com fallback local).
- **Formulário migrado:**
  - valida no submit, não ao montar;
  - mostra todas as mensagens juntas no topo (`MagnetoErrorDataTD`, `role="alert"`, foco no resumo);
  - a mensagem de campo obrigatório parte do `caption` do controle no `.ccp`.
- **Fica como pendência:** a validação manual no navegador, que você não faz (DOM final, teclado, Lighthouse Accessibility no tema ativo). Registre no resumo e no doc da tela.

## Texto e JSDoc

- **Encoding:** `.vue` e `.ts` em UTF-8, com texto humano acentuado. Isso inclui `meta.title`, `aria-label`, fallbacks de termo, mensagens e strings verificadas em teste. Identificadores, rotas, query params, ids e names ficam sem acento.
- **Varredura antes de concluir:** procure `Ã|Â|�|\\u[0-9a-fA-F]{4}|&[A-Za-z]+;` nos arquivos alterados. `&nbsp;` só é aceito como espaçamento herdado.
- **JSDoc:** em português com acento.
  - Em `src/core`, `src/router` e `src/shared`: `@param`, `@returns` e um `@example` curto nos contratos reutilizáveis.
  - Em `src/modules`: `@param` e `@returns` nas funções com regra, sem `@example` obrigatório.
  - Nada de comentário linha a linha.

## Testes e validação

- **Stack:** Vitest 2 com jsdom e `@vue/test-utils`; specs em `src/**/*.{test,spec}.ts`.
- **Mínimo esperado:**
  - schema e validação;
  - helper ou componente compartilhado criado ou alterado;
  - rota e query de grid com `pagina`;
  - link Vue → CodeCharge (carimbo e `target="_top"`);
  - Vue → Vue preservando query;
  - resumo de validação no submit inválido.
  - Teste que depende de texto usa a string acentuada final. Não crie spec só para provar acessibilidade.
- **Comandos, da raiz do repositório:**
  - `npm --prefix sgucard/sgucard-frontend test` (`vitest run`). Obrigatório, porque o CI não roda;
  - `npm --prefix sgucard/sgucard-frontend run build` (`vue-tsc --noEmit` e `vite build`). Pega erro de tipo;
  - sem `node_modules`, rode antes `npm --prefix sgucard/sgucard-frontend install` (precisa de rede). Não use `npm ci`: não há `package-lock.json` versionado, e ele falha. O `install` é o mesmo que o Maven roda, e o `package-lock.json` que ele gera é ignorado pelo `.gitignore` da raiz.
- **Lint:** o `npm run lint` está quebrado hoje: o ESLint 9 procura `eslint.config.js`, e o projeto só tem `.eslintrc.cjs`. Para lint direcionado, rode o ESLint nos arquivos alterados, a partir de `sgucard/sgucard-frontend`, com `ESLINT_USE_FLAT_CONFIG=false`. Não migre a configuração fora do plano.

## Regras

| ID | Nível | Regra | Como verificar |
|---|---|---|---|
| ATD-CMF-001 | MUST | Tela entra como rota no `routes.ts` do módulo, com `meta.title` e `meta.requiresAuth`; sem `main.ts`/`index.html` por tela | diff |
| ATD-CMF-002 | MUST | HTTP só por `@core/http`; `api/*.ts` desempacota `data` com fallback; sem `CD_MENU` manual | diff e teste |
| ATD-CMF-003 | MUST | Estado de grid na query com `pagina` (nunca `page`), por `useRouteBackedGridState` | teste de rota |
| ATD-CMF-004 | MUST | Vue → CodeCharge por `href` com URL carimbada; popup por `openCodeChargePopup`; modal Vue por `openVueModalWindow` | teste e diff |
| ATD-CMF-005 | MUST | Migração preserva o visual Magneto, os `id` legados e os `name` de contrato, sem inventar `id`/`name` | comparação com o HTML legado |
| ATD-CMF-006 | MUST | Todo controle visível com nome acessível; ação só com ícone com `aria-label` | DOM |
| ATD-CMF-007 | MUST | UTF-8 com texto humano acentuado, sem mojibake, escape Unicode ou entidade | varredura |
| ATD-CMF-008 | MUST | `test` e `build` verdes, rodados por você | comandos |
| ATD-CMF-009 | SHOULD | Reusar `src/shared` antes de criar; extrair para `shared` só com repetição real | diff |
| ATD-CMF-010 | SHOULD | Lista grande com `shallowRef`, `v-if` e montagem em blocos | teste |

## Fora deste papel

- **Endpoint novo ou alterado:** papel `dev-backend` (`dev-atd-cmagnet-backend`). No plano, vem antes do front.
- **Menu apontando para a rota Vue** (`CM_MENUS.DS_URL_APP`, `CM_ITENS_ADMIN`): changeset no central do `cmagnet-util` (skill `dev-atd-liquibase`).
- **Página `.do` que precisa mudar:** papel `dev-codecharge` (`dev-atd-codecharge`).
- **Tema** (`ccs/Themes/<tema>/`): não mexa. Problema de contraste ou foco do tema é entrega separada, com autorização.

## NUNCA

- Nunca crie SPA, `main.ts` ou `index.html` por tela, nem cliente HTTP próprio.
- Nunca use Vuetify ou outro UI kit, nem biblioteca nova sem registrar o motivo.
- Nunca use `router.push` para `.do` nem `page` como parâmetro de rota Vue.
- Nunca declare `interface Window`/`declare global` fora de `src/core/legacy/window.d.ts`.
- Nunca altere `ccs/Themes/*` sem autorização explícita.
