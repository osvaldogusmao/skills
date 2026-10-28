---
name: dev-atd-codecharge-migration
description: "Roteiro para planejar a migração de uma tela CodeCharge (.do) para Vue + REST no CMAGNET ou no AUTSC2. Diz o que o plano precisa ter antes de qualquer código: as entradas obrigatórias, o inventário da tela legada (fato, inferência ou pendência), o contrato REST e Vue de cada produto, a divisão entre os papéis dev-backend e dev-frontend, a integração com telas Vue que já apontam para a tela, os itens em outros repositórios (menu e banco no cmagnet-util, entidade no atd-persistence) e quais docs do produto ler. Use ao planejar (nexdom-planner) um chamado que migra uma tela legada ou cria uma tela Vue no lugar de uma .do. Não é skill de implementação: o código segue as skills de backend e frontend do produto (dev-atd-cmagnet-* ou dev-atd-autsc2-*)."
---

# Migração de tela CodeCharge → Vue + REST (planejamento)

Esta skill não tem papel: o `nexdom-planner` a lê ao planejar, e os papéis a leem por caminho. Os dois produtos têm "agentes" escritos pelo time que fazem a migração inteira numa entrega só (`docs/agente-migracao-codecharge-vue.md` e, para tela nova sem legado, `docs/agente-criacao-feature-vue.md`). No kit, o trabalho se divide: o **plano** faz o levantamento e o contrato, e os **papéis** implementam em sequência.

## Docs do produto

| Produto | Leia primeiro | Roteiro detalhado | Doc da tela |
|---|---|---|---|
| CMAGNET | `docs/migracao-vue/padroes-vigentes.md` | `docs/agente-migracao-codecharge-vue.md`, `docs/migracao-vue/padrao-migracao-tela-legado.md`, `docs/migracao-vue/de-para-codecharge-vue-rest.md`, `docs/migracao-vue/guia-proximas-migracoes.md` | `docs/migracao-vue/pages/<modulo>/<tela>.md` |
| AUTSC2 | `docs/padroes-vigentes.md` | `docs/guia-proximas-migracoes.md` (o principal), `docs/agente-migracao-codecharge-vue.md`, `docs/de-para-codecharge-vue-rest.md`, `docs/integracao-menu-spa.md` | `docs/pages/<modulo>-<tela>.md` |

O `padroes-vigentes.md` de cada produto é o índice normativo, e cada um define a própria ordem de autoridade. Telas já migradas são referência funcional, não de estrutura. Para tela Vue nova sem legado, use a "Matriz de Decisão de Estrutura" de `docs/agente-criacao-feature-vue.md`.

## Diferenças que mudam o plano

| | CMAGNET | AUTSC2 |
|---|---|---|
| Publicação da SPA | iframe do `ccs/app.html`, aberta por `app.do?page=/cmagnet/app/<rota>` | direto em `/autsc2/app/<rota>`, sem wrapper |
| Página no estado da URL | `pagina` (o `page` é do `app.do`) | `page`, `pageSize`, `sort`, `direction` |
| Contexto de menu | `CD_MENU`, repassado pelo `@core/http` | `cd_menu` e `cd_modulo`, repassados pelo `menuContext.ts` |
| Permissão REST | `@RequerPermissao` / `@RequerPermissaoMenuPai` | `@RequiresPermission`, com a mesma chave do `meta.recurso` da rota |
| Registro do resource | `@AutoService(ResourceRegistry.class)` | arquivo do `ServiceLoader`; nunca `@AutoService` |
| Encoding da API | UTF-8 (`sgucard-api`) | ISO-8859-1 (`src/br/com/autsc2/api`) |
| Testes do backend | JUnit 5 | JUnit 4 |
| Menu | `CM_MENUS` ou `CM_ITENS_ADMIN` | `AUTSC2_MENUS` |
| Classes chamadas pelos eventos | `src/br/com/dynamix/ccs/cmagnet/<área>/` | `src/br/com/dynamix/ccs/autsc/<área>/` |

## Entradas obrigatórias (se faltar alguma, pergunte antes de planejar)

- **A tela de origem:** o caminho do HTML legado (ex.: `ccs/consultas/pacotes/lista.html`) e do código gerado (`ccs/CCSBuild/src/consultas/pacotes/lista/`), ou a URL `.do` ou o menu, a partir dos quais o plano acha os arquivos.
- **O contexto funcional de destino**, quando a tela pertence a uma feature-pai (ex.: `admin` → `cadastros` → `locais-atendimento`), e se a tela-pai já existe em Vue. Se não existir, não se cria página-pai vazia; a entrada continua pelo legado.
- **No AUTSC2, as telas Vue que já levam a esta tela:**
  - o componente de origem e a URL `.do` que ele chama hoje;
  - os parâmetros enviados;
  - o comportamento ao voltar;
  - se abre na mesma aba, em nova aba ou em popup;
  - se o fallback `.do` continua.
  - Se o chamado não disser, procure no `frontend/` as chamadas à tela legada.
- **O que não pode mudar**, conforme o chamado.

## O que o plano precisa conter

### 1. Inventário da tela legada

Classifique cada item:
- **fato**: lido no arquivo, com o caminho;
- **inferência**: dedução, com o motivo;
- **pendência**: algo que só o time confirma.

O que inventariar:
- **Arquivos:** `.html`, `.ccp`, Java gerado (`Action`, `Model`, `View`, `DataObject`, handlers), as classes escritas à mão que os eventos chamam, AJAX, servlets e JS inline.
- **Menu e permissão:** onde a tela aparece no menu e qual é a chave de permissão. Sem evidência, é pendência. Nunca invente.
- **Dados:**
  - o SQL de cada data source e evento, com a query original e o mapa alias → campo; não assuma o significado de um alias sem evidência;
  - as tabelas;
  - os parâmetros de URL e os hidden fields.
  - O que não é SQL também conta, como consulta ao servidor de regras ou DAO legado reaproveitado.
- **Comportamento:**
  - filtros, colunas, ordenação, tamanho de página (`setFetchSize`) e se a tela mostrava o total (Navigator "de N");
  - botões e links `.do`, popups e callbacks globais;
  - termos `{term:…}`;
  - validações (`caption`, `required`, `RequiredHandler`, `ValidateHandler`) e mensagens;
  - lista vazia, erro e `Group`s condicionais.
- **Volta preservando a busca:** é contrato funcional quando o legado tem hidden `s_busca`, valor padrão de busca no `RecordHandler`, `GridHandler.beforeShow` dependente de query param ou `Navigator` com URLs de página. Nesse caso, a busca vai para a query da rota da listagem.
- **Contrato com QA e acessibilidade:** os `id` e os `name` do HTML (o QA usa os ids) e os nomes acessíveis que já existem.
- **Visual:** a estrutura Magneto que a tela Vue vai reproduzir (tabelas de layout, classes, `&nbsp;`).

Checklist completo: "Checklist de levantamento" no `de-para-codecharge-vue-rest.md` do produto.

### 2. Contrato

- **Backend (papel `dev-backend`):**
  - pacote: `br.com.sgucard.api.<modulo>.<feature>` (CMAGNET) ou `br.com.autsc2.api.modules.<modulo>.<feature>` (AUTSC2), com `core/` se for feature composta;
  - cada endpoint, com método, path (`/cmagnet/rest/api/v1/...` ou `/autsc2/rest/api/v1/...`), query params e DTOs de entrada e saída;
  - paginação e sort: `page` começando em 1 e `@SortColumn`; com total ou só `hasNext`, conforme o legado;
  - erros e a permissão de cada endpoint (`ACCESS` na classe, uma função por método de escrita);
  - URLs `.do` que a API devolve (carimbadas, no CMAGNET) e a tag do Swagger;
  - quais operações vão pela DAO de entidade do atd-persistence, e se a entidade já existe.
- **Frontend (papel `dev-frontend`):**
  - rota, nome, pasta da feature e, no AUTSC2, `meta.recurso`;
  - query params preservados (contexto de menu, estado da grid, filtros com o nome legado);
  - navegação de ida e volta e componentes compartilhados;
  - termos e os `id`/`name` preservados.
- **Arquivos esperados** de cada papel.
- **Documento da tela**, no caminho da tabela "Docs do produto", com a estrutura da Etapa 5 do `agente-migracao-codecharge-vue.md`. Normalmente quem fecha é o `dev-frontend`.

### 3. Ordem dos papéis

1. **`dev-backend`:** API, Swagger e testes (com o JaCoCo).
2. **`dev-frontend`:** tela, rota e testes Vitest, consumindo o contrato da etapa anterior. No AUTSC2, também troca pela rota nova o link `.do` das telas Vue de origem e documenta o contrato novo nas duas telas.
3. **`qa-analyst`**, se estiver no roster.

O `dev-codecharge` só entra se a tela legada também mudar (ex.: o link da tela antiga para a nova).

### 4. Riscos e dúvidas: o que fica fora deste repositório

- **Menu:** a migração só está completa quando o menu aponta para a rota Vue. O changeset do `DS_URL_APP`, do `FG_USA_APP` e do `DS_CHAVE_PERMISSAO` vai no central do **cmagnet-util** (skill `dev-atd-liquibase`):
  - CMAGNET: em `CM_MENUS` ou `CM_ITENS_ADMIN`;
  - AUTSC2: em `AUTSC2_MENUS`, e até ele ser aplicado a API da tela responde 403 para todos;
  - ele roda num `/dev-ticket` próprio, no outro repositório, com o mesmo nome de branch; o banco entra antes do app.
- **Entidade ou DAO do atd-persistence que não existe:** começa no `atd-persistence` (`dev-atd-persistence`), com o `mvn install` local da lib antes de compilar o produto.
- **Tabela ou coluna nova:** vai no central do cmagnet-util, com espelho no SAPIA-NI se a tabela for monitorada.
- **Mapa dos repositórios e ordem:** `dev-atd-ecossistema`.

## O que dos agentes do time não vale dentro do `/dev-ticket`

- **Branch, commit e MR** são do fluxo do kit (agente `nexdom-gitflow-atd`, branch `feature/SAT-I<número>`). A "Regra de Branch" dos agentes do time (`feature/vue-<modulo>-<tela>`) vale só fora do kit.
- **Separação em issues:** vira as seções deste plano; não se criam issues.
- **Ficam como pendência do dev no resumo:** validação manual (comparar com a tela legada no navegador, teclado, Lighthouse) e conferência no Swagger. No AUTSC2, também o teste antes do menu, que o doc sugere fazer com UPDATE manual no banco: é o dev quem aplica.

## Preservação

Migração não é redesign nem evolução. Não acrescente campo, filtro, coluna ou ação que a tela legada não tinha, a menos que o chamado peça. Preserve o SQL e a semântica dos filtros, as mensagens, os `id` e os `name`. O que ficar diferente vai para "Diferenças conhecidas" no documento da tela.

## NUNCA

- Nunca implemente antes de o plano ser confirmado.
- Nunca planeje uma mudança no cmagnet-util ou no atd-persistence como se fosse deste repositório: é outro `/dev-ticket`, em outro repositório.
- Nunca invente chave de permissão, tabela ou regra que o inventário não mostrou.
