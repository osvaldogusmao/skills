---
name: dev-atd-homecare-frontend
description: "Frontend do homecare-webapp do ATD: a SPA Vue 3 (TypeScript strict, Vite, Pinia, Vue Router, Axios; sem UI kit e sem VeeValidate/Zod), servida por nginx em /homecare/, que os prestadores de homecare usam para capturar, executar e finalizar guias de exame. Cobre a estrutura (views, components, services, stores, composables), serviço como função async com useApi, contextStore e modalStore, o fluxo de autenticação JWT com refresh em sessionStorage, os guards de rota, a configuração em runtime (BACKEND_BASE_URL), a API nos servlets legados capturaExterna do cmagnet e a validação por type-check e build. Use para criar ou alterar tela, serviço, store ou componente no homecare-webapp. NÃO cobre o backend desses servlets (dev-atd-cmagnet-backend) nem as SPAs do CMAGNET e do AUTSC2."
---

# homecare-webapp (SPA Vue)

O homecare-webapp é a SPA dos prestadores de homecare. Com ela, o prestador:
- escolhe o local de atendimento e busca o beneficiário;
- captura a guia de exame/SADT;
- executa as sessões, com timer e alarme;
- finaliza a guia, total ou parcialmente.

Ela não tem backend próprio: chama os servlets legados do **cmagnet** em `<BACKEND_BASE_URL>/servlet/capturaExterna/v1/*`. O repositório não tem docs (o README é o do create-vue), então esta skill e o código são a referência.

## Stack

- **Base:** Vue 3.5 com `<script setup lang="ts">` (sem Options API), TypeScript strict (`@vue/tsconfig`), Vite 7 (`base: '/homecare/'`, alias `@` → `src`), Pinia 3, Vue Router 4 e Axios.
- **O que não tem:** UI kit, VeeValidate/Zod e i18n. Componentes e CSS são próprios (`<style scoped>`, sem SCSS), e os textos ficam em pt-BR no código. Não introduza biblioteca nova sem o chamado pedir.
- **Formatação e lint:** Prettier (sem ponto e vírgula, aspas simples, largura 100) e ESLint flat (`eslint.config.ts`).
- **Testes:** Cypress (componente e e2e). Não há Vitest.

## Estrutura

| Pasta | O que vai nela |
|---|---|
| `src/views/*View.vue` | uma tela por rota (`ExamesEmAbertoView`, `CapturarGuiaView`, `DetalheGuiaCapturadaView`…) |
| `src/components/` | componentes reutilizáveis (`SimpleTable`, `SectionCard`, `Button`, `FloatButton`, `Loader`, `NavBar`, `Timer`, `InputCarteira`) e as modais `*Modal.vue` |
| `src/services/` | um `*Service.ts` por assunto, com funções `async` nomeadas e um objeto default que as agrega; `api.ts` (Axios); `types/index.ts` (DTOs) |
| `src/stores/` | `authStore`, `careFacilityStore`, `parametersStore`, `modalStore` e a fachada `contextStore` |
| `src/composables/` | `useApi`, `useAudio` |
| `src/router/` | `index.ts` (rotas e guard) e `vue-router.d.ts` (`RouteMeta` tipado) |
| `src/utils/` | `filters`, `sorters`, `storage`, `assert` e formatação de datas (`index.ts`) |

Os arquivos de scaffold sem uso (`HelloWorld`, `TheWelcome`, `WelcomeItem`, os ícones de exemplo, `layouts/DefaultLayout.vue`, `assets/logo.svg`) não servem de modelo. Não os apague fora do plano.

## Padrões

- **Serviço:** é uma função `async` que chama `api.post('/servlet/capturaExterna/v1/<endpoint>', payload)` e devolve `response.data`, tipado com o DTO de `services/types` (modelo: `GuiaService.getGuiasFinalizadas`). Tipo novo vai em `services/types/index.ts`.
- **Tela de dados:** usa `const { data, error, loading, execute } = useApi(GuiaService.getX)`. O `useApi` controla o carregamento e o erro e só aceita erro `AxiosError`; outro tipo de erro dispara `assert`. Modelo: `ExamesFinalizadosView.vue`. Só as telas do fluxo de login (`LoginView` e `SelectCareFacilityView`) controlam o estado à mão, com try/catch.
- **Contexto:** o `useContextStore()` dá acesso a `auth`, `careFacility` (`cdLocal`…), `params` (`getGrupoDeItens()`), `modal` e `systemLogout()`.
- **Modal:**
  - `context.modal.open(Componente, props)` devolve uma Promise que resolve quando a modal fecha. Há `confirm(title, message, ConfirmLabel)` e `alert(title, message)` prontos.
  - Uma modal de cada vez; abrir outra por cima dispara `assert`.
- **Filtro e ordenação em memória:** `utils/filters` (`and`, `filterIncludes`) e `utils/sorters`.
- **Datas:** o backend espera `dd/MM/yyyy` (ex.: `toLocaleDateString('pt-BR')`). Os helpers ficam em `@/utils`: `formatAsISODate`, `manipulateDate` e `isValidaDate`.

## Autenticação, sessão e rotas

- **Login:** o `LoginService` usa o `loginApi`, que não passa pelo refresh, e devolve `token`, `refreshToken`, `exp` e `refreshExp`. Toda chamada pelo `api` leva `Authorization: Bearer`.
- **401:**
  - o `api.ts` chama uma vez `/servlet/capturaExterna/v1/refresh`, enfileira as chamadas concorrentes (`RefreshInterceptorService`) e repete;
  - se o refresh falhar, faz `systemLogout()` e `location.reload()`;
  - não implemente outro tratamento de 401 na tela.
- **Sessão:** fica no `sessionStorage`, com prefixo `[homecare]`, e só pelas chaves de `STORAGE_KEYS`, via `HomecareStorage` (`utils/storage.ts`). Nunca use `localStorage`: a sessão saiu de lá no SAT-I2415.
- **Rotas:**
  - ficam no `router/index.ts`, com `meta` tipado: `requiresAuth`, `isGuest`, `selectCareFacility` e `description`, que vira o título da página;
  - o guard leva ao login, à seleção do local ou a `loading`, que recarrega os parâmetros e volta pelo `redirect`;
  - rota nova leva `meta.requiresAuth` e `description`, com o componente carregado por `import()`.
- **Configuração em runtime:**
  - em produção, o `BACKEND_BASE_URL` vem do JSON `APPLICATION_CONFIG` do `index.html`, preenchido por envsubst no container nginx;
  - em desenvolvimento, é `http://localhost:8080/cmagnet/` (`src/env.ts`);
  - nunca escreva URL de ambiente no código.
- **Asset público:** monte o caminho com `import.meta.env.BASE_URL`, porque o app roda sob `/homecare/`. Nada de caminho fixo.

## A API fica no cmagnet

- **Onde fica:** os endpoints são servlets legados do cmagnet, em `src/br/com/dynamix/framework/cmagnet/ajax/integracao/captura/AJAX*.java`. Eles são mapeados nos dois `web.xml` (`ccs/WEB-INF/web.xml` e `ccs/CCSBuild/config/web.xml`), passam pelos filtros `CapturaExternaCorsFilter` e `HomecareJwtFilter` e **não** estão no `sgucard-api`.
- **Endpoint novo ou contrato alterado:** a mudança começa no cmagnet, no mesmo chamado (SAT-I) e com o mesmo nome de branch.
  - Exemplo: SAT-I1906, `db98769191` no cmagnet, com servlet, `request/`, `dto/`, `service/` e os dois `web.xml`.
  - No plano, o cmagnet vem antes do homecare, e o MR do cmagnet entra na `HOMOLOG` primeiro.
  - Ver `dev-atd-cmagnet-backend` e `dev-atd-ecossistema`.
- **Tipos:** o tipo em `services/types` reflete o DTO devolvido pelo servlet, não a tabela.

## Texto e encoding

UTF-8 com LF e 2 espaços (`.editorconfig`), e texto humano em português com acento. Antes de concluir, procure `Ã`, `Â` e `�` nos arquivos alterados.

## Validação

- **Instalar:** rode `npm ci` com `CYPRESS_INSTALL_BINARY=0`, porque o script `prepare` baixa o binário do Cypress, que é grande.
- **Checagem:** `npm run type-check` (`vue-tsc --build`) e `npm run build` (type-check e `vite build`, que escreve em `dist/`, fora do git). É o mesmo que o CI roda.
- **Lint sem alterar arquivo:** use `npx eslint .`. O `npm run lint` usa `--fix` e o `npm run format` reescreve os arquivos com o Prettier: só rode se o plano pedir formatação.
- **Testes:**
  - os 2 specs que existem são de scaffold e estão quebrados: `src/components/__tests__/HelloWorld.cy.ts` e `cypress/e2e/example.cy.ts`;
  - teste novo, só se o chamado pedir, como teste de componente Cypress em `src/components/__tests__/*.cy.ts` (`npm run test:unit`, que exige o binário do Cypress);
  - na prática, a validação é o type-check, o build e o teste manual.
- **Fica como pendência:** abrir a tela contra um cmagnet de pé (local ou HML) e testar o fluxo.

## Regras

| ID | Nível | Regra | Como verificar |
|---|---|---|---|
| ATD-HCF-001 | MUST | Chamada HTTP só pelo `api` de `services/api.ts`, numa função de `*Service.ts`, com o tipo em `services/types` | diff |
| ATD-HCF-002 | MUST | Tela de dados usa `useApi` para carregamento e erro; nenhuma tela trata 401 por conta própria | diff |
| ATD-HCF-003 | MUST | Sessão só por `HomecareStorage` (`sessionStorage`, prefixo `[homecare]`) | diff |
| ATD-HCF-004 | MUST | Rota com `meta` tipado (`requiresAuth`, `description`) e componente carregado por `import()` | diff |
| ATD-HCF-005 | MUST | Endpoint novo ou alterado nasce no cmagnet, no mesmo chamado, antes do homecare | plano |
| ATD-HCF-006 | MUST | `type-check` e `build` verdes | comandos |
| ATD-HCF-007 | SHOULD | Reusar os componentes e utils existentes; nada de UI kit ou biblioteca nova sem pedido | diff |
| ATD-HCF-008 | SHOULD | Sem URL de ambiente nem caminho fixo de asset no código | diff |

## NUNCA

- Nunca rode `npm run lint` nem `npm run format` sem o plano pedir: eles reescrevem arquivos.
- Nunca guarde token em `localStorage` nem o escreva em log.
- Nunca altere `Dockerfile`, `nginx.conf` ou `.gitlab-ci.yml` sem o chamado pedir: eles definem a imagem de todos os ambientes.
- Nunca aponte o código para a URL de um ambiente (HML ou produção).
