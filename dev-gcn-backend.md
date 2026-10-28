---
name: nexdom-gcn-backend-dev
description: "Convenções de desenvolvimento backend do SGU GCN — Gestão de Cartas Negativas (repositório sgu-gcn-gitlab, pasta server/). Node 11 + TypeScript 3.4 (sem ?. e ??): Express 4 + TypeORM 0.2 + Oracle (todos os ambientes), carta PDF via Vue SSR + wkhtmltopdf com templates por Unimed, fila de e-mail com node-cron, integração AutSC/ATD, multi-tenancy manual por operadora. Use sempre que a tarefa envolver endpoints, controllers, services, models/migrations, layout da carta negativa, e-mail, jobs, sync de operadoras, autenticação, bug, refatoração ou troubleshooting no backend deste repositório, mesmo sem citar a stack. NÃO usar para as telas Nuxt/Vue deste produto (ver nexdom-gcn-frontend-dev) nem para outros produtos NEXDOM (CRM, RESSUS, NEXDOM DS)."
---

# Backend SGU GCN — Gestão de Cartas Negativas (sgu-gcn-gitlab)

Convenções reais deste repositório, inconsistências incluídas — não é documentação genérica de Express/TypeORM. Se um arquivo/função citado aqui não existir mais, sinalize ao usuário em vez de assumir que o padrão ainda vale.

> Todos os caminhos citados nesta skill são relativos à **raiz do repositório** (`sgu-gcn-gitlab/`), não à pasta da skill.

> O GCN é **um único app**: o Express (`server/`) serve o frontend Nuxt (`pages/`, `components/`...) no mesmo processo e no mesmo repositório. As telas têm skill própria (`nexdom-gcn-frontend-dev`). Quando uma mudança atravessa os dois lados (permissão nova, `operationStatus` novo, campo novo exibido na tela, `utils/` da raiz), faça a parte do backend e **descreva no seu resumo o que o frontend precisa fazer** (contrato, chave de permissão, valor novo).

## Stack e restrições de versão

| Camada | Tecnologia |
|---|---|
| Runtime | **Node 11.15.0** (`.nvmrc`, `Dockerfile`), npm 6 |
| Linguagem | **TypeScript 3.4.5**, `target: es5`, `module: commonjs`, `strict` com `noImplicitAny: false` |
| HTTP | Express 4 + body-parser (limite 50mb) + cors |
| ORM | TypeORM 0.2.x (`getRepository` global, sem DI) |
| Banco | **Oracle em todos os ambientes** (container Docker local, HML compartilhada e produção) via `oracledb`, Instant Client 18.3 no Docker — ver *Acesso a dados* sobre os resquícios de PostgreSQL |
| PDF | Vue SSR (`vue-server-renderer`) + `wkhtmltopdf` 0.12.5 |
| E-mail / jobs | nodemailer 5 + node-cron 2 |
| Log | winston 2 + winston-daily-rotate-file + morgan |
| Testes | Jest 23 + ts-jest, supertest, node-mocks-http no repositório, **sem uso** — ver *Lint, formatação e testes* |

**Regras de sintaxe obrigatórias (TS 3.4 / Node 11):**
- Proibido `?.` e `??` — use `a && a.b` e `a !== undefined && a !== null ? a : b` (padrão usado em todo o código).
- APIs de runtime no backend (Node 11). O `tsconfig.json` usa `lib: ["esnext"]`, então o TypeScript **não acusa** API inexistente — o erro só aparece em execução (`... is not a function`):
  - **Disponíveis:** `Array.prototype.flat`/`flatMap`, `Object.entries`/`values`, `async`/`await`, `fs.promises` (já usado em `server/services/pdf/html-render.ts`).
  - **Indisponíveis:** `Object.fromEntries` e `Promise.allSettled` (Node 12), `String.prototype.replaceAll` (Node 15), `Array.prototype.at` (Node 16.6). Use `reduce`, `Promise.all` com `.catch` por item, `replace(/x/g, ...)` e `arr[arr.length - 1]`.
  - Na dúvida sobre outra API, confira a versão mínima em node.green antes de usar.
- Não suba versões de dependência (typeorm, nuxt, typescript, winston) como efeito colateral de uma tarefa — a stack inteira está presa ao Node 11.

## Estrutura e ciclo de inicialização

Um único processo: `server/index.ts` sobe Express e injeta `nuxt.render` como último middleware. Ordem de boot (falha em qualquer etapa aborta o servidor):

1. `startConnection()` (`server/database-connect.ts`) — conecta via `TYPEORM_*` do `.env`; models/migrations/subscribers carregados por **glob** (`models/*.{ts,js}` etc.) — arquivo novo nessas pastas entra automaticamente. Cria `admin@sgugcn.com` se a tabela `users` estiver vazia.
2. `startSync()` (`server/sync.ts`) — rotinas idempotentes que propagam defaults para **todas** as operadoras (modelo de negativa padrão, informação normativa, config de e-mail, perfis padrão).
3. `startJobs()` (`server/jobs/index.ts`) — cron de envio de e-mails pendentes e limpeza de logs antigos.
4. Nuxt (build em dev).

```
server/
  router/index.ts        -> /api (router/api.ts) e /auth (router/auth.ts)
  router/api/<recurso>.ts -> mapeia rota -> método do controller
  controllers/<recurso>.ts -> objeto literal `xxxController` com métodos async (req, res)
  services/<recurso>.ts  -> classe `XxxService` (instanciada com `new` a cada uso)
  models/<entidade>.ts   -> @Entity TypeORM + class-validator
  migrations/            -> <timestamp>-<Nome>.ts + type-converter.ts
  subscribers/           -> @EventSubscriber (ex.: limpa cache de header/footer ao atualizar Operator)
  sync/                  -> rotinas de propagação executadas no boot
  jobs/                  -> node-cron
  services/pdf/          -> geração da carta (html-render, letter-generator, pdf-generator) + templates HTML
  interfaces/            -> interfaces compartilhadas (IDownloadLetterRequest, modelNegativeDefault)
pages/ components/ layouts/ middleware/ plugins/ store/  -> frontend Nuxt (skill nexdom-gcn-frontend-dev)
utils/                   -> código compartilhado FRONT + BACK (string-utils.js, date-util.ts, ckeditor-utils-pdf.js)
```

> `utils/` na raiz é importado pelo backend (`server/logger.ts` usa `date-util`, `pdf-generator.ts` usa `ckeditor-utils-pdf`, `server/controllers/requests.ts` usa `string-utils`). Alterar algo ali afeta os dois lados.

## Padrão de camadas

Para um recurso novo, replicar exatamente esta cadeia:

1. **Rota** em `server/router/api/<recurso>.ts` exportando `xxxRoutes = Router()`; montar em `server/router/api.ts` **sempre com `checkJwtTokenMiddleware`**: `apiRoutes.use('/xxx', checkJwtTokenMiddleware, xxxRoutes)`. Sub-recursos via `router.use('/:parentId/filhos', filhosRoutes)` (padrão de `requests.ts`).
2. **Controller** `export const xxxController = { async index(req, res) {...}, ... }` com constante `const XXX_CONTROLLER = 'xxxController'`. Cada método:
   ```ts
   async index(req, res) {
     try {
       logInformationController(XXX_CONTROLLER, 'index', req.user)
       const result = await new XxxService().findAll(req.query, req.user)
       logTotalQueryRecords(result ? result.length : 0, req.user)
       return res.json({ type: 'success', message: 'Lista de ... selecionadas', result })
     } catch (error) {
       logError(error, req, res)
       return res.status(405).json({ type: 'error', message: 'Erro ao buscar ...', result: error })
     }
   }
   ```
   Controller só valida entrada, chama service e monta a resposta. Lógica de negócio nova vai no **service** — `requestsController.upsert` acessa repositório direto e é legado, não modelo.
3. **Service** `export class XxxService` com métodos `public async`, cada um começando com `logContext('nomeMetodo', user)` e, no `catch`, `logger.error('Erro ao ...: ', error)` seguido de `throw error`. Validação de argumento obrigatório no início: `if (!id) { throw new Error('Xxx ID is required.') }`.
4. **Model** em `server/models/` + **migration** correspondente (ver seção de migrations).

### Contrato de resposta HTTP (o frontend depende dele)

- Sucesso: `{ type: 'success', message, result }` (listas paginadas acrescentam `count`). Auth usa `data` em vez de `result` (`auth/login` → `data.token`, `auth/me` → `data.me`, configurado em `nuxt.config.js`).
- Erro: `{ type: 'error', message, result|error }`. **Status 405 é a convenção histórica para erro de negócio/validação** em quase todos os controllers — não "corrija" para 400/422 em endpoints existentes (quebra telas). Em endpoint novo, siga o vizinho do mesmo recurso.
- Erro de validação exibível: `result: [{ property: 'campo', constraints: { chave: 'Mensagem em PT-BR' } }]` — formato de `class-validator`, renderizado pelo componente `components/Notification.vue`.
- Mensagens ao usuário em **português**. Exceção: API externa (`/api/requests/download`, `/info`, `/export`) responde em inglês — mantenha o idioma do contrato existente.

## Autenticação e autorização — pontos de atenção

- JWT HS256 com segredo `JWT_TOKEN`, validade 7d, emitido em `sessionsController.login` assinando **o objeto User inteiro** (sem senha). `checkJwtTokenMiddleware` (`server/utils.ts`) recarrega o usuário do banco e popula `req.user` (entidade `User` completa: `id`, `operatorId`, `unimedId`, `isAdmin`, `isOperatorAdmin`, `profileId`).
- **O backend não verifica permissões de perfil.** Chaves `MENU_*` (tabela `permissions`, via `profile_permissions`) são checadas **somente no frontend** (`middleware/admin.js` + `meta.permissions`, plugin `$hasPermission`). No backend só existe checagem ad hoc de `req.user.isAdmin` dentro de alguns controllers (ex.: criar/excluir operadora). Se uma funcionalidade nova precisa ser restrita, faça a checagem explícita no controller/service — não confie no menu escondido.
- Permissão nova: a parte do backend é a migration de insert em `permissions` (e a checagem explícita, se a funcionalidade for sensível); `meta` da página e menu são do frontend — informe a chave `MENU_XXX` criada no seu resumo.
- Rotas de API externa usam `allowMethods(['POST'])` (`server/middlewares/allowMethods.ts`) antes do router JWT. Para nova rota exposta a sistemas externos, siga o mesmo par (`allowMethods` + JWT).
- Senha: `bcryptjs` no `@BeforeInsert/@BeforeUpdate` de `User`; coluna `password` tem `select: false`. O login do front envia a senha já em `sha256` (`pages/login.vue`), e o seed de `database-connect.ts` segue o mesmo formato.
- Campos sensíveis são mascarados no log por `SENSITIVE_FIELDS` em `server/logger.ts` — **ao criar ou tocar em campo sensível no body (senha, token, credencial SMTP/integração), confira se ele está nessa lista e adicione-o se não estiver.**

## Regra obrigatória: escopo por operadora (multi-tenant por coluna)

O "tenant" é a **Operadora** (`Operator`, uma Unimed). Não há isolamento por schema; o filtro é manual em cada query:

- `user.operatorId` — operadora do usuário logado; `user.unimedId` — código da Unimed; `user.isAdmin` — admin global (vê todas as operadoras); `user.isOperatorAdmin` — admin da própria operadora.
- `Operator.id` (PK interna) ≠ `Operator.code` (código da Unimed, ex.: 58, 72, 192…). **A integração envia códigos**: em `POST /api/requests`, `operatorOwnerId`/`operatorWorkerId` do body são `code` e o controller converte para `id` via `findByCodeOrFail`. `operatorWorker.code` também seleciona a pasta de template PDF.
- `Request` tem duas operadoras: **`operatorOwner`** (Unimed do beneficiário — filtro "Unimed do beneficiário" na tela) e **`operatorWorker`** (Unimed executora, cujos dados/assinatura/SMTP/templates emitem a carta). Listagem de solicitações filtra `operatorOwnerId = :operatorId OR operatorWorkerId = :operatorId`.
- Toda consulta nova de listagem/busca **deve** filtrar por `user.operatorId` (e por `unimedId` onde o recurso já faz — reasons, templates, users), liberando só para `isAdmin` quando o padrão do recurso já o fizer. Referências: `OperatorService.findAll`, `TemplateService` (query builder), `UserService`, `ReasonService`.
- Os `findById` dos services (ex.: `RequestService.findById`, `OperatorService.findById`) não devem ser assumidos como filtrados por operadora — leia a implementação antes de reutilizar. Em endpoint novo que recebe ID do usuário, valide que o registro pertence à operadora de `req.user` (a menos que seja admin).

## Acesso a dados (TypeORM 0.2 + Oracle)

- Use `getRepository(Entity)` e `createQueryBuilder` com **parâmetros nomeados** (`.where('request.id = :id', { id })`). Nunca interpole valor de request em SQL.
- **O banco é Oracle em todos os ambientes.** SQL cru pode usar funções Oracle (`TO_DATE`, `NVL`, `SYSDATE`), como já faz `RequestService.getSearchQuery`. Os ramos PostgreSQL que ainda existem no código (`server/migrations/type-converter.ts`, `import { Client } from 'pg'` e `startTestConnection` em `server/database-connect.ts`, `.env.sample`, serviço `pg` do `docker-compose.yml`) são **resquício legado**: não os use como referência de ambiente e não os remova sem pedido explícito.
- Particularidades do Oracle a lembrar: string vazia `''` é gravada como `NULL` (filtre com `IS NULL`, não `= ''`); identificador entre aspas é case-sensitive (`"requestedAt"` ≠ `requestedAt`); `LIKE` diferencia maiúsculas — o código usa `lower(...) LIKE lower(:param)`.
- Colunas camelCase legadas exigem aspas em SQL cru (`"request"."requestedAt"`); dentro do QueryBuilder use `alias.propriedade`.
- Nomenclatura: tabelas legadas em camelCase plural (`requestItems`, `reasonsData`), novas em snake_case (`message_jobs`, `profile_permissions`, `request_histories`). **Colunas novas: snake_case no banco com propriedade camelCase** — `@Column({ name: 'negative_date', nullable: true }) public negativeDate?: Date`. Membros de classe sempre com modificador explícito (`public`/`private`) — exigido pelo ESLint.
- Boolean no Oracle é `NUMBER` 0/1 — use `migrationsTypes.boolean/trueValue/falseValue`.
- Paginação: manual, `skip((page - 1) * 30).take(30)` + `getCount()` (ver `RequestService.findAll`); o front usa `vue-tables-2` `ServerTable` com `responseAdapter` lendo `result` e `count`.
- Ordenação dinâmica: mapa de estratégias por coluna permitida (`orderStrategies` em `RequestService`) — nunca passe `orderBy` do request direto ao QueryBuilder.
- Para expor `Request` ao front, selecione só colunas necessárias de `Operator` (`getSelectColumnsRequest`) e **remova `emailConfig`** da resposta (credenciais SMTP) — ver `requestsController.show`.

## Migrations

- Criar: `npx typeorm migration:create -n AddXxxAtInTabela` (requer `TYPEORM_MIGRATIONS_DIR=server/migrations`). Arquivo `<timestamp>-<Nome>.ts`, classe `<Nome><timestamp> implements MigrationInterface`.
- Padrões de nome em uso: `Create<Tabela>Table`, `Add<Coluna>AtIn<Tabela>`, `Alter<Coluna>...AtIn<Tabela>`, `Insert...`.
- **Todos os tipos via `server/migrations/type-converter.ts`** (`migrationsTypes.bigint | string | boolean | longText | now | trueValue | falseValue`) — é o padrão do projeto e o valor que vale é o ramo Oracle. Se precisar de tipo novo, adicione um getter lá seguindo o mesmo formato.
- Sempre implemente `down()`. Use a API `queryRunner.addColumn/createTable/dropColumn` com `TableColumn`/`Table`; `queryRunner.query` só para carga de dados (ex.: `InsertDataAtInPermissions`), com identificadores entre aspas duplas.
- Cabeçalho `/* eslint-disable prettier/prettier */` é usado nas migrations recentes.
- Model e migration devem mudar juntos. Os ambientes rodam com `TYPEORM_SYNCHRONIZE=false` e `TYPEORM_MIGRATIONS_RUN=true`: **a migration é executada no boot**, e uma migration com erro impede o servidor de subir. Não use o `TYPEORM_SYNCHRONIZE=true` do `.env.sample` legado.
- Como a migration roda no boot, **subir o servidor com uma migration nova aplica essa migration no banco para onde o `.env` aponta**. Teste migration nova só no Oracle local do container — nunca com o `.env` apontando para a HML compartilhada (ver *Ambiente local*).
- Dado padrão por operadora (perfil, modelo, config) não é só migration: crie/ajuste a rotina em `server/sync/` e chame-a também em `OperatorService.save` para operadoras novas (padrão de `syncProfilesToOperator`/`syncNormativeToOperator`).

## Domínio — conceitos fáceis de errar

- **Request (Solicitação)**: guia negada recebida por integração (`POST /api/requests` = upsert). Campos-chave: `protocol`, `guideNumber`, `code` (`cd_solicitacao`, usado no AutSC), `deniedStatus` (`denied`/`parcial` — o template compara com `'parcial'`), `negativeDate` e `mail` (e-mail do beneficiário).
- **`operationStatus`**: a lista canônica é `operationStatusAvailable` em `server/utils.ts`. Ao adicionar/alterar um status, atualize **todos os pontos**: `operationStatusAvailable` e `getOperationStatus` em `server/utils.ts`, `IDownloadLetterRequest.operationStatus` e `operationStatusList` em `pages/requests/index.vue` (este último é do frontend — informe o valor novo no seu resumo).
- **RequestItem**: itens da guia; só itens `status === 'denied'` entram na carta, agrupados por `reasonId`.
- **Reason (Motivo)**: por operadora/unimed; `reasonId` é o código externo do motivo de negativa. **Modelo padrão** = `modelNegativeDefault.reasonId = 999999999999999` (`server/interfaces/model-negative-default.ts`) — fallback quando o motivo recebido não existe.
- **Template (Modelo de negativa)**: texto CKEditor por motivo/operadora; flags `modelDefault`, `automaticSend` (envio automático ao beneficiário) e `downloadAvailable` (marca solicitação como `available`). A decisão fica em `RequestService.sendMailAutomaticToNegativeOrUpdateAvailable` — exige que **todos** os templates dos motivos tenham a flag.
- **ReasonData**: *snapshot* do texto do template gravado na solicitação — a carta usa o ReasonData, não o template atual. Editar template não altera cartas já geradas.
- **History**: registro de envio/download da carta (`sendType` `'Externo'`, `'Download'`…, anexo em `uploads/<uuid>.pdf`). `RequestHistory` / `TemplateHistory` / `ReasonHistory`: auditoria de alterações de campos.
- **MessageJob**: fila de mensagens em banco (ver E-mail).
- **Profile / Permission**: perfis por operadora; perfis padrão `ProfilesDefault` (`Administrador Operadora`, `Usuário Operadora`) sincronizados no boot. Chaves de permissão (`MENU_*`): a fonte da verdade são as migrations que inserem em `permissions` (`grep -rn "MENU_" server/migrations`), não o enum `PermissionsDefault` de `server/utils.ts`.

## Geração da carta negativa (PDF)

Fluxo: `RequestService.createLetter` → `LetterGenerator.render()` → `HtmlRender` (Vue SSR) → `PDFGenerator` (wkhtmltopdf A4) → **`/tmp/request-<id>.pdf`**.

- Templates em `server/services/pdf/<operatorWorker.code>/{body,header,footer}.html`, com fallback para `default/`. **Mudança de layout/campo na carta precisa ser replicada em `default/` e em todas as pastas customizadas** que tenham o mesmo trecho — liste `server/services/pdf/` para ver as pastas existentes e confira cada uma (ex.: `grep -ln "trecho" server/services/pdf/*/body.html`).
- Sintaxe dos `.html` de carta é **template Vue 2** (`{{ }}`, `v-if`, `v-for`) com dados `request`, `sortedItems`, `renderModelReason`, `moment`, `today`, `currentDate`, `textoPreview`. Campos calculados que o template usa (ex.: `request.downloadNegativeDate`, `request.reasonsDataMap`) são preenchidos no service antes do render — propriedade não-coluna declarada no model sem `@Column`.
- Em template de carta, diretiva condicional é `v-if` (sem `:`) — `:v-if` vira atributo HTML comum e o bloco sempre renderiza.
- Placeholder `#ITENS_DESCRICAO` dentro do texto do template é substituído pela descrição dos itens (`renderModelReason`).
- Outros HTML (`mail-recover-password.html`, `negative-information.html`, `model-negative.html`) usam `${variavel}` com `replaceTemplateVariables` de `server/utils.ts` — sintaxe diferente, não misture.
- Header/footer são cacheados em `/tmp/header-<operatorWorkerId>.html` / `footer-...`; o subscriber `server/subscribers/operator.ts` os apaga no `afterUpdate` de `Operator`. Alterou `header.html`/`footer.html` em dev? Apague os arquivos de `/tmp` para ver o efeito.
- Conteúdo do CKEditor é normalizado por `utils/ckeditor-utils-pdf.js` antes do wkhtmltopdf.
- Dependências de sistema: binário `wkhtmltopdf` no PATH e diretório `/tmp` (o caminho é fixo — em Windows sem WSL/Docker a geração falha).

## E-mail, jobs e integrações

- **Nunca envie e-mail de forma síncrona na requisição.** Enfileire com `new MessageJobService().save({ recipient, operatorWorkerId, requestId, historyId, attachments })` → cria `MessageJob` `PENDING`, `maxRetries: 2`, remetente/assunto/corpo da `EmailConfig` da operadora. O cron `server/jobs/send-pending-messages.ts` (`EMAIL_SCHEDULE_TIMER_MINUTES`, padrão 5) envia com o SMTP da **operatorWorker**, atualiza `Request.operationStatus` (`sended`/`automaticSended`/`pending`) e `History.sentAt`.
- Erros de SMTP: traduza com `mapNodemailerError` (`server/utils.ts`); serialize com `serializeError` antes de gravar/logar.
- Operadora sem SMTP válido: `requestsController.validMailConfigOperator` — reutilize em vez de reimplementar.
- **AutSC / ATD**: `IntegrationAtd.sendFileATD` (`server/services/integration-atd.ts`) — multipart com Basic Auth usando `operator.integrationUrl/User/Password`, exige `request.code`, timeout 30s, mensagens de erro já traduzidas para o usuário. Novas chamadas ao AutSC devem passar por essa classe.
- Job novo: arquivo em `server/jobs/` exportando `startXxx`/`stopXxx`, registrar em `jobs/index.ts` (start e stop — o `stop` é chamado no SIGTERM/SIGINT).
- Relatório: `RequestService.export` gera CSV (`json2csv`, BOM) ou XLSX (`createXlsxFile` em `server/file-utils.ts`); MIME por `getMimeByExtension`.

## Logging

- Controller: `logInformationController(CONTROLLER, 'acao', req.user)` no início, `logTotalQueryRecords(n, req.user)` após consultas, `logError(error, req, res)` em todo `catch` (inclui dados do request sanitizados).
- Service: `logContext('metodo', user)` no início; `logger.info/error` do `server/logger.ts`. Mensagens de log em português.
- **Não use `console.log`** — o ESLint trata como erro em produção e o output não vai para o arquivo rotacionado. Se encontrar `console.log` de depuração no código, não replique.
- Logs em `logs/` (`server.log` + rotação diária, retenção `LOG_RETENTION_DAYS`, padrão 60 dias).

## Lint, formatação e testes

- Prettier: **sem ponto e vírgula, aspas simples** (`.prettierrc`). ESLint: `plugin:vue/recommended` + prettier + `@typescript-eslint/recommended`; modificador de acesso explícito obrigatório em membros de classe; retorno explícito recomendado; variável não usada só com prefixo `_`. Rode `npm run lint` (ou `npm run fixl`) nos arquivos alterados.
- Arquivos do repositório usam LF — o Git no Windows avisa conversão para CRLF; não converta finais de linha de arquivos inteiros.
- **Testes: não há suíte em uso.** `jest-server.config.js`/`test/server/` (e `jest-vue.config.js`/`test/pages/`, `test/component/`, `test/middleware/`, `test/store/` do frontend) existem no repositório, mas o projeto é legado e os testes não acompanharam as melhorias e customizações — não rode `npm test` como validação, não trate falha dessas suítes como regressão da sua mudança e não tente "consertá-las". Não afirme "os testes passaram" para o backend. Se a tarefa pedir teste explicitamente, proponha um teste mínimo deixando claro que não há suíte ativa a seguir — e sem `startTestConnection()`, que cria banco PostgreSQL e não funciona com o Oracle (mocke os repositórios). Para validar: `npm run lint` e `npx tsc --noEmit -p .` nos arquivos alterados, revisão manual do fluxo, e descreva ao usuário exatamente o que foi validado e o que não foi.

## Ambiente local

- `.env` com Oracle (`TYPEORM_CONNECTION=oracle`, `TYPEORM_HOST`, `TYPEORM_PORT`, `TYPEORM_USERNAME`, `TYPEORM_PASSWORD`, `TYPEORM_DATABASE`, `TYPEORM_SID`, `TYPEORM_MIGRATIONS_DIR=server/migrations`, `TYPEORM_MIGRATIONS_RUN=true`, `TYPEORM_SYNCHRONIZE=false`, `TYPEORM_LOGGING`): use o `.env` do time — nunca copie usuário/senha para código, log ou resumo — o `.env.sample` está desatualizado (aponta para PostgreSQL). Variáveis relevantes: `JWT_TOKEN`, `TYPEORM_*`, `TYPEORM_MIGRATIONS_RUN`, `SMTP_*`, `EMAIL_SCHEDULE_TIMER_MINUTES`, `APP_FRONTEND_URL` (links de e-mail de recuperação), `LOG_RETENTION_DAYS`, `HTTPS` (usa `https/server.key|crt`), `EXPRESS_LOGGER`.
- **O `.env` pode apontar para dois bancos diferentes:** o Oracle local, num container Docker próprio e descartável, ou a **base HML compartilhada**, usada por outras pessoas para debug e análise. Antes de qualquer ação que escreva no banco, confira para onde o `.env` aponta: o Oracle local do container é `TYPEORM_HOST=localhost`, `TYPEORM_PORT=1521`, `TYPEORM_SID=ORCLCDB.localdomain`. **Qualquer outro host trate como HML compartilhada.** Se for a HML, ou se houver dúvida, **pare e confirme com o usuário** antes de: subir o servidor com migration nova, rodar `db-seed`, usar `TYPEORM_DROP_SCHEMA`, rodar script de carga ou executar `INSERT`/`UPDATE`/`DELETE`/DDL à mão. Na HML, só leitura sem confirmação.
- `npm run dev` (nodemon + ts-node em `server/index.ts`, porta 3000, build Nuxt em memória). Produção: `npm run build` (`build.sh` → `builded/`) e `npm start`.
- Docker: a imagem (`Dockerfile`) inclui Oracle Instant Client e wkhtmltopdf. O `docker-compose.yml` ainda declara um serviço `pg` (PostgreSQL 11) que não é usado — o Oracle local roda num container Docker próprio, fora do `docker-compose.yml`, e é apontado pelo `.env`.
- Seeds: `npm run db-seed`, **só no Oracle local do container**. O fluxo usa `TYPEORM_DROP_SCHEMA=true`, que **apaga o schema inteiro** — nunca rode contra a HML.

## Git e entrega

- Branches `feature/SAT-I<nnnn>` e `bugfix/SAT-I<nnnn>[-descricao]` a partir de `develop`; MR para `develop` no GitLab. Branches de release `DESENVOLVIMENTO-AA-MM-0-0`; `master` é produção. CI via template `nexdom/devops/ci-templates` (`.gitlab-ci.yml`).
- `server/config.ts` tem `appVersion = 'dev'`, substituído por `change-version.sh` no build Docker — não edite manualmente.

## Anti-patterns — checklist final

Revisão rápida antes de entregar; o detalhe está na seção indicada.

- `?.`, `??` ou API de Node ≥ 12 → *Stack e restrições de versão*.
- Query sem filtro de `user.operatorId` ou busca por ID sem checar a operadora → *Escopo por operadora*.
- Funcionalidade sensível sem checagem no backend (confiando só em `meta.permissions`/menu do front) → *Autenticação e autorização*.
- Tipo de coluna literal fora de `migrationsTypes`; `orderBy`/valor do request interpolado em SQL → *Acesso a dados* / *Migrations*.
- Migration sem `migrationsTypes` ou sem `down()` → *Migrations*.
- Escrita no banco (migration nova no boot, `db-seed`, `TYPEORM_DROP_SCHEMA`, DML/DDL manual) sem conferir se o `.env` aponta para a HML compartilhada → *Ambiente local*.
- E-mail enviado direto, sem `MessageJob` → *E-mail, jobs e integrações*.
- Layout da carta alterado só em `default/` → *Geração da carta negativa*.
- `emailConfig`/credencial na resposta, ou campo sensível fora de `SENSITIVE_FIELDS` → *Acesso a dados* / *Autenticação e autorização*.
- Regra de negócio nova no controller → *Padrão de camadas*.
- Status HTTP ou formato `{ type, message, result }` alterado em endpoint existente → *Contrato de resposta HTTP*.
- `operationStatus` novo sem atualizar todos os pontos (ou sem avisar o frontend) → *Domínio*.
- `console.log` ou mensagem ao usuário em inglês → *Logging* / *Contrato de resposta HTTP*.

## Troubleshooting rápido

- **Carta não gera / "a Operadora não possui template configurado"** (`ENOENT` no preview): falta template para `operatorWorker.code` e em `default/`, `/tmp` inexistente, ou `wkhtmltopdf` fora do PATH.
- **Carta com cabeçalho/rodapé antigo**: cache em `/tmp/header-<operatorWorkerId>.html` — o subscriber só limpa em `afterUpdate` de `Operator`; apague manualmente em dev.
- **Carta com texto antigo após editar o modelo**: esperado — a carta usa o snapshot `ReasonData` da solicitação.
- **E-mail não sai**: verifique `message_jobs` (`status`, `retry_count`, `log`), se a operadora tem `EmailConfig` válida (`validMailConfigOperator`) e o intervalo `EMAIL_SCHEDULE_TIMER_MINUTES`; logs "Não existem mensagens pendentes" confirmam que o cron roda.
- **Solicitação não fica `available`/não envia automático**: `sendMailAutomaticToNegativeOrUpdateAvailable` exige `request.mail` preenchido e **todos** os templates dos motivos com `automaticSend`/`downloadAvailable`.
- **403 "Provided token is invalid"**: header sem `Bearer` ou `JWT_TOKEN` diferente entre ambientes. Token válido de usuário removido do banco **não** retorna 403: `findOneOrFail` rejeita dentro do callback de `jwt.verify` em `checkJwtTokenMiddleware` e o handler de `unhandledRejection` encerra o processo — se o servidor cair após uma requisição autenticada, verifique isso.
- **Servidor não sobe**: erro em `startConnection`/`startSync`/`startJobs` aborta o boot — `EMAIL_SCHEDULE_TIMER_MINUTES` inválido lança exceção; migration pendente com `TYPEORM_MIGRATIONS_RUN=true` falhando.
