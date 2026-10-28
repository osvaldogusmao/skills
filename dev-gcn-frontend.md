---
name: nexdom-gcn-frontend-dev
description: "Convenções de desenvolvimento frontend do SGU GCN — Gestão de Cartas Negativas (repositório sgu-gcn-gitlab, pastas pages/components/layouts/middleware/plugins/store). Nuxt 2 em mode spa + Vue 2.6 Options API em JavaScript (sem TypeScript nos .vue), Bulma, vue-tables-2, vuelidate, vue-izitoast, CKEditor 5, HTTP via this.$axios, permissões de tela por chaves MENU_* (middleware admin + $hasPermission), data-testid obrigatório. Servido pelo mesmo processo Express do backend (Node 11, sem ?. e ??). Use ao criar ou editar páginas, componentes, validações de formulário, tabelas, permissões de menu/tela, chamadas à API ou estilos deste repositório, mesmo sem citar a stack. NÃO usar para o backend deste produto — server/, migrations, carta PDF, e-mail, jobs (ver nexdom-gcn-backend-dev) — nem para o frontend de outros produtos NEXDOM (CRM, RESSUS, NEXDOM DS)."
---

# Frontend SGU GCN — Gestão de Cartas Negativas (sgu-gcn-gitlab)

Convenções reais deste repositório, inconsistências incluídas — não é documentação genérica de Nuxt/Vue. Se um arquivo/função citado aqui não existir mais, sinalize ao usuário em vez de assumir que o padrão ainda vale.

> Todos os caminhos citados nesta skill são relativos à **raiz do repositório** (`sgu-gcn-gitlab/`), não à pasta da skill.

> O GCN é **um único app**: o Nuxt é servido pelo Express (`server/`) no mesmo processo e no mesmo repositório. O backend tem skill própria (`nexdom-gcn-backend-dev`). Quando a tarefa também exige mudança no `server/` (endpoint, migration de permissão, valor novo de `operationStatus`), não implemente essa parte por conta própria: use o contrato informado pelo agente de backend no mesmo ticket ou, se não houver, sinalize no seu resumo o que falta do lado do servidor.

## Stack e restrições de versão

| Camada | Tecnologia |
|---|---|
| Runtime | **Node 11.15.0** (`.nvmrc`, `Dockerfile`), npm 6 — o mesmo processo do backend |
| Framework | Nuxt 2 `mode: 'spa'`, Vue 2.6.14 Options API em **JavaScript** (nenhum `.vue` usa `lang="ts"`) |
| UI | Bulma + `assets/main.scss`, vue-tables-2, vuelidate, vue-izitoast (`this.$toast`), CKEditor 5 |
| HTTP | `this.$axios` (módulo do Nuxt) |
| Testes | Jest 23 + @vue/test-utils (`jest-vue.config.js`) no repositório, **sem uso** — ver *Lint, formatação e testes* |

**Regras de sintaxe obrigatórias:**
- Proibido `?.` e `??` — use `a && a.b` e `a !== undefined && a !== null ? a : b` (padrão usado em todo o código).
- `utils/` na raiz é **compartilhado com o backend** (`server/logger.ts` usa `date-util`, `pdf-generator.ts` usa `ckeditor-utils-pdf`, `server/controllers/requests.ts` usa `string-utils`) e roda no Node 11: ali não use `Object.fromEntries`, `Promise.allSettled`, `String.prototype.replaceAll` nem `Array.prototype.at`. Alterar algo em `utils/` afeta os dois lados — avise no seu resumo.
- Não suba versões de dependência (nuxt, vue, bibliotecas de UI) como efeito colateral de uma tarefa — a stack inteira está presa ao Node 11.

## Estrutura

```
pages/<recurso>/{index.vue,new.vue,_id/...}  -> telas (rotas geradas pelo Nuxt)
components/              -> componentes reutilizáveis (PascalCase), ex.: Notification.vue
layouts/                 -> layout e menu
middleware/admin.js      -> checagem de permissão de página
plugins/                 -> ex.: $hasPermission
store/                   -> Vuex do Nuxt
assets/main.scss         -> estilos globais
utils/                   -> código compartilhado FRONT + BACK (string-utils.js, date-util.ts, ckeditor-utils-pdf.js, validators/)
server/                  -> backend Express (skill nexdom-gcn-backend-dev)
```

## Páginas e componentes

- Páginas em `pages/<recurso>/{index.vue,new.vue,_id/...}`; painéis/modais auxiliares da página ficam na própria pasta (ex.: `pages/requests/_id/show-request-modal.vue`); componentes reutilizáveis em `components/` (PascalCase).
- Options API em JavaScript (`export default { components, data, computed, mounted, methods }`). Não introduza `nuxt-property-decorator`/TS em `.vue` (está no `package.json`, mas não é usado).
- Validação de formulário: `vuelidate` (`validations: {}`) + validadores em `utils/validators/`.
- Tabelas: `vue-tables-2` (`ServerTable` para listas paginadas no servidor, `ClientTable` para listas pequenas) + `addTableTestIds` de `utils/vue-tables-data-testid.js`.
- **`data-testid` obrigatório** em inputs/botões interativos, padrão `<contexto><entidade>-<campo>` (ex.: `filterrequest-numeroguia`, `filterprofile-btn-filter`, `table-request-btn-view`).
- Estilo: Bulma + `assets/main.scss`; classe `is-force-black-dark-mode` nos `box` de tela.
- Mensagens ao usuário em **português**.

## Consumir a API (contrato definido pelo backend)

- HTTP: `this.$axios` com caminho relativo sem barra (`'api/operators?select=true'`); lê `response.data.result`. Feedback com `this.$toast.success/warning/error`.
- Sucesso: `{ type: 'success', message, result }` (listas paginadas acrescentam `count`). Auth usa `data` em vez de `result` (`auth/login` → `data.token`, `auth/me` → `data.me`, configurado em `nuxt.config.js`).
- Erro: `{ type: 'error', message, result|error }`. **Status 405 é a convenção histórica para erro de negócio/validação** — trate 405 como erro esperado de validação, não como falha de rede.
- Erro de validação exibível: `result: [{ property: 'campo', constraints: { chave: 'Mensagem em PT-BR' } }]` (formato `class-validator`) — renderize com `<Notification :errors="errors">` (`components/Notification.vue`), não monte a mensagem na mão.
- Listas paginadas no servidor: o backend pagina de 30 em 30 e devolve `result` + `count`; o `ServerTable` lê os dois via `responseAdapter`.
- `Request` vem com só parte das colunas de `Operator` e **sem `emailConfig`** (credenciais SMTP) — não peça ao backend para incluir esse campo numa tela.

## Permissões e autenticação

- **Permissões de página**: `middleware: 'admin'` + `meta: { permissions: ['MENU_XXX'] }`; itens de menu/botões com `this.$hasPermission('MENU_XXX')`. `isAdmin` passa em tudo.
- **O backend não verifica permissões de perfil** — as chaves `MENU_*` são checadas **somente no frontend**. Esconder menu/botão não protege nada: se a funcionalidade for sensível, ela também precisa de checagem no backend (sinalize no seu resumo se o plano não previu isso).
- Permissão nova = migration de insert em `permissions` (backend) + `meta` da página + menu no layout (frontend). A fonte da verdade das chaves são as migrations (`grep -rn "MENU_" server/migrations`), não o enum `PermissionsDefault` de `server/utils.ts`.
- Usuário logado (`auth/me`): `operatorId`, `unimedId`, `isAdmin` (admin global, vê todas as operadoras), `isOperatorAdmin` (admin da própria operadora), `profileId`.
- O login (`pages/login.vue`) envia a senha já em `sha256` — mantenha esse formato.

## Domínio — o que a tela precisa saber

- **Operadora** (`Operator`, uma Unimed) é o "tenant"; o filtro por operadora é feito no backend. `Operator.id` (PK interna) ≠ `Operator.code` (código da Unimed, ex.: 58, 72, 192…).
- **Request (Solicitação)**: guia negada recebida por integração. Tem duas operadoras: **`operatorOwner`** (Unimed do beneficiário — filtro "Unimed do beneficiário" na tela) e **`operatorWorker`** (Unimed executora, que emite a carta).
- **`operationStatus`**: a lista canônica fica em `server/utils.ts` (`operationStatusAvailable`). A tela tem sua própria `operationStatusList` em `pages/requests/index.vue` — status novo exige atualizar os dois lados (os pontos do servidor são do backend).
- **Template (Modelo de negativa)**: texto CKEditor por motivo/operadora, com flags `modelDefault`, `automaticSend` e `downloadAvailable`. A carta usa o snapshot **ReasonData** gravado na solicitação — editar o modelo não altera cartas já geradas (não é bug de tela).
- **Profile / Permission**: perfis por operadora; perfis padrão `Administrador Operadora` e `Usuário Operadora`.

## Lint, formatação e testes

- Prettier: **sem ponto e vírgula, aspas simples** (`.prettierrc`). ESLint: `plugin:vue/recommended` + prettier + `@typescript-eslint/recommended`; variável não usada só com prefixo `_`. Rode `npm run lint` (ou `npm run fixl`) nos arquivos alterados.
- **Não use `console.log`** — o ESLint trata como erro em produção. Se encontrar `console.log` de depuração no código, não replique.
- Arquivos do repositório usam LF — o Git no Windows avisa conversão para CRLF; não converta finais de linha de arquivos inteiros.
- **Testes: não há suíte em uso.** `jest-vue.config.js`/`test/pages/`, `test/component/`, `test/middleware/`, `test/store/` (e `jest-server.config.js`/`test/server/` do backend) existem no repositório, mas o projeto é legado e os testes não acompanharam as melhorias e customizações — não rode `npm test` como validação, não trate falha dessas suítes como regressão da sua mudança e não tente "consertá-las". Não afirme "os testes passaram". Se a tarefa pedir teste explicitamente, proponha um teste mínimo deixando claro que não há suíte ativa a seguir. Para validar: `npm run lint` nos arquivos alterados, revisão manual da tela, e descreva ao usuário exatamente o que foi validado e o que não foi.

## Ambiente local

- `npm run dev` (nodemon + ts-node em `server/index.ts`, porta 3000, build Nuxt em memória) — o frontend só roda junto com o backend. `APP_FRONTEND_URL` é usado nos links de e-mail de recuperação de senha.
- Banco: Oracle em todos os ambientes, configurado no `.env` do time (o `.env.sample` e o serviço `pg` do `docker-compose.yml` são legado). O `.env` pode apontar para o Oracle local (container Docker próprio) ou para a **base HML compartilhada** com outras pessoas — e como as migrations rodam no boot, `npm run dev` com uma migration nova do backend apontando para a HML aplica essa migration lá. O Oracle local é `TYPEORM_HOST=localhost` (`TYPEORM_SID=ORCLCDB.localdomain`); qualquer outro host trate como HML. Se o `.env` apontar para a HML, ou se houver dúvida, confirme com o usuário antes de subir o servidor com migration nova ou de testar na tela algo que grave dados.

## Git e entrega

- Branches `feature/SAT-I<nnnn>` e `bugfix/SAT-I<nnnn>[-descricao]` a partir de `develop`; MR para `develop` no GitLab. Branches de release `DESENVOLVIMENTO-AA-MM-0-0`; `master` é produção. CI via template `nexdom/devops/ci-templates` (`.gitlab-ci.yml`).

## Anti-patterns — checklist final

Revisão rápida antes de entregar; o detalhe está na seção indicada.

- `?.`, `??`, ou API de Node ≥ 12 em `utils/` → *Stack e restrições de versão*.
- TypeScript, `nuxt-property-decorator` ou Composition API em `.vue` → *Páginas e componentes*.
- Input/botão interativo sem `data-testid` no padrão → *Páginas e componentes*.
- Funcionalidade sensível protegida só por `meta.permissions`/menu, sem avisar que falta checagem no backend → *Permissões e autenticação*.
- Mensagem de erro do backend montada na mão em vez de `<Notification>`, ou 405 tratado como falha genérica → *Consumir a API*.
- `operationStatus` novo só na tela (ou só no servidor) → *Domínio*.
- `console.log` ou mensagem ao usuário em inglês → *Lint* / *Páginas e componentes*.

## Troubleshooting rápido

- **Tela vazia ou dados não aparecem**: confira se a tela lê `response.data.result` (e `count` nas listas paginadas) — auth usa `data`, não `result`.
- **Menu/página não aparece para o usuário**: a chave `MENU_XXX` precisa existir em `permissions` (migration) e estar vinculada ao perfil dele (`profile_permissions`); `isAdmin` vê tudo.
- **403 "Provided token is invalid"**: header sem `Bearer` ou `JWT_TOKEN` diferente entre ambientes (configuração do backend).
- **Carta com texto antigo após editar o modelo**: esperado — a carta usa o snapshot `ReasonData` da solicitação.
