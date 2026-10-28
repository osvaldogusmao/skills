---
name: dev-atd-autsc2-backend
description: "Backend do AUTSC2: a API REST em src/br/com/autsc2/api, dentro do WAR (Java 1.8 com fontes em ISO-8859-1, JAX-RS/RESTEasy 2.3.4 com registro manual no ServiceLoader e nunca @AutoService, envelope {data, error}, paginação 1-based com ou sem COUNT, @RequiresPermission com cd_menu/cd_modulo, @PublicEndpoint, Swagger, Javadoc, DAO com SqlBuilder e CRUD simples pelo atd-persistence, JUnit 4 + JaCoCo de 90%) e o Java legado de src/br. Use para criar ou alterar endpoint, service, DAO, DTO, Swagger, Javadoc ou teste no autsc2, ou Java legado fora das páginas CodeCharge. NÃO cobre o frontend Vue (dev-atd-autsc2-frontend), as páginas CodeCharge (dev-atd-codecharge), banco e menu (dev-atd-liquibase, no central do cmagnet-util) nem o CMAGNET."
---

# AUTSC2 — backend

O autsc2 é um WAR só, com três partes: o legado CodeCharge (`ccs/` e `ccs/CCSBuild/`), o Java de `src/` (o legado e a API nova em `src/br/com/autsc2/api`) e a SPA Vue (`frontend/`, publicada em `/autsc2/app/`). Esta skill cobre a API e o Java de `src/`; as páginas `.do` estão na `dev-atd-codecharge`.

## Leia antes

Os docs do repositório são a fonte de verdade; esta skill é o resumo. Leia primeiro `docs/padroes-vigentes.md`. A ordem de autoridade é: ele mesmo, `docs/arquitetura-front-back.md`, `docs/padrao-nomenclatura-codigo.md`, `docs/agente-migracao-codecharge-vue.md` com `docs/guia-proximas-migracoes.md` e, por último, `docs/pages/**`, que é referência funcional e não normativa. Depois, só o doc do tema:

| Tarefa | Doc |
|---|---|
| Registro, paths, segurança, `web.xml`, Swagger | `docs/configuracao-resteasy.md` |
| Pacotes da feature, envelope, listagens | `docs/padroes-vigentes.md` (Organização de backend REST, Listagens) e `docs/arquitetura-front-back.md` |
| DAO e paginação com ou sem `COUNT` | `docs/guia-proximas-migracoes.md` (seção DAO) |
| Testes e JaCoCo | `docs/testes-rest-backend-autsc2-api.md` |
| Nomes e encoding | `docs/padrao-nomenclatura-codigo.md` |
| Permissão e o endpoint `/menu/permissoes` | `docs/permissoes-granulares-frontend.md`, seção 3 |
| Menu e `DS_CHAVE_PERMISSAO` | `docs/integracao-menu-spa.md` |
| atd-persistence | skill `dev-atd-persistence`. O autsc2 ainda não tem doc própria; o `padroes-vigentes.md` aponta para o `docs/migracao-vue/padrao-atd-persistence-sgucard-api.md` do cmagnet |

**Divergências conhecidas entre doc e código.** Siga o código e registre no resumo:
- **`SqlBuilder`:** o `padroes-vigentes.md` manda usar `br.com.dynamix.framework.cmagnet.sql.SqlBuilder`, que **não existe** no autsc2. A API usa `br.com.dynamix.framework.autsc.sql.SqlBuilder`.
- **atd-persistence:**
  - o `padroes-vigentes.md` diz que a integração ainda não foi feita, mas ela existe: a dependência está no `pom.xml` e há `SqlUtil.getPersistence()`;
  - o doc também ensina `@SqlTable`/`@SqlColumn`/`@SqlId`, que não existem mais na lib.
- **Registries:** o `configuracao-resteasy.md` e o `arquitetura-front-back.md` listam 2 registries no `ServiceLoader`; hoje são 7.
- **Catálogos:** os de componentes e de testes também estão atrás do código. Confira o que existe antes de afirmar que falta.

## Onde fica cada coisa

| Área | Caminho | Encoding |
|---|---|---|
| API REST | `src/br/com/autsc2/api/modules/<modulo>/<feature>/` | ISO-8859-1 |
| Configuração JAX-RS | `src/br/com/autsc2/api/config/` (`RestApplication`, `RestResources`, `ResourceRegistry`) | ISO-8859-1 |
| Registro dos módulos | `src/META-INF/services/br.com.autsc2.api.config.ResourceRegistry` | — |
| Java legado: framework (`br.com.dynamix.framework.autsc.*`), DAOs antigos, classes chamadas pelas telas | `src/br/` fora de `br/com/autsc2/api` | ISO-8859-1 |
| Testes | `test/br/com/autsc2/api/...`, no mesmo pacote da classe | ISO-8859-1 |
| Descritores do WAR | `ccs/WEB-INF/web.xml` e o template do Studio `ccs/CCSBuild/config/web.xml` (mantenha os dois equivalentes ao mexer em REST) | — |

Tudo compila num `pom.xml` só (`src`, `test` e `ccs/CCSBuild/src`), em Java 1.8 e com `project.build.sourceEncoding=ISO-8859-1` herdado do `fdsc-pom`. Pacote de feature vai em minúsculas e sem separador (ex.: `fasesautorizacao`, `avisosbeneficiario`); o `setup/versaoIntegracao` foge do padrão.

## Encoding: leia antes de editar qualquer `.java`

- **Tudo em ISO-8859-1:** todo `.java` de `src/` e de `test/` é lido pelo javac como ISO-8859-1. Arquivo salvo em UTF-8 vira mojibake no Swagger, nas mensagens e nos DTOs.
- **Nunca edite parcialmente um `.java` ISO-8859-1 com ferramenta que grava UTF-8.** Os acentos que já existiam viram `U+FFFD`, e essa perda é irreversível (`guia-proximas-migracoes.md`, seção Encoding). Isso já aconteceu na `HOMOLOG`: `api/docs/SwaggerTags.java` e `modules/setup/versaoIntegracao/dao/VersoesIntegracaoDAO.java` têm `�` no lugar dos acentos.
- **Procedimento do time:** gravar o arquivo inteiro em ISO-8859-1 e conferir por bytes que não há `EF BF BD` (U+FFFD) nem pares UTF-8. No Claude Code, isso se faz com os passos da subseção abaixo, validados no piloto.
- **Onde vai acento:** texto humano (mensagens REST e de `ApiException`, Swagger, validações) leva acento, no arquivo em ISO-8859-1. Identificadores, comentários e logs técnicos ficam sem acento. Javadoc é em português sem acento.

### Como editar um arquivo ISO-8859-1 (validado no piloto)

O Edit e o Write gravam sempre em UTF-8. No piloto, um Edit numa linha **sem** acento de um `.java` ISO-8859-1 regravou o arquivo inteiro e trocou todos os acentos que já existiam por `�` (`EF BF BD`). Por isso:

1. **Descubra o encoding antes de editar:** se `iconv -f UTF-8 -t UTF-8 <arquivo> > /dev/null` falhar, o arquivo é ISO-8859-1. Se passar, ele é UTF-8 ou só ASCII, e o Edit não estraga o que já existe. Um arquivo UTF-8 com acento numa área ISO-8859-1 é um achado: não converta de passagem, reporte.
2. **Arquivo ISO-8859-1: nunca use o Edit nele direto.** Edite uma cópia UTF-8 ao lado e só converta de volta se a conversão passar:
   ```bash
   iconv -f ISO-8859-1 -t UTF-8 <arquivo> > <arquivo>.utf8
   # leia <arquivo>.utf8 com o Read e edite com o Edit
   iconv -f UTF-8 -t ISO-8859-1 <arquivo>.utf8 > <arquivo>.iso && mv <arquivo>.iso <arquivo> && rm <arquivo>.utf8
   ```
   Se o `iconv` de volta falhar, o texto tem um caractere que não existe em ISO-8859-1 (travessão `—`, aspas curvas): troque por `-` ou `"` e repita. Para uma troca pequena, também serve um script Node que lê e grava o arquivo com a codificação `'latin1'`.
3. **Acento novo num arquivo novo, ou num arquivo que era só ASCII, numa área ISO-8859-1:** grave com o Write ou o Edit e depois converta com `iconv -f UTF-8 -t ISO-8859-1`, para um `.iso` e `mv`, como acima.
4. **Confira antes de concluir:**
   - `LC_ALL=C grep -c $'\xef\xbf\xbd' <arquivo>` dá `0`;
   - `git diff <arquivo> | iconv -f ISO-8859-1 -t UTF-8 | grep -E '^\+.*(Ã|Â)'` não mostra nada;
   - o `git diff` só tem as linhas pretendidas;
   - não sobrou `.utf8` nem `.iso` no `git status`.

## Receita: endpoint novo

Modelos: `admin/fasesautorizacao` (listagem com total, ordenação, permissão e subpágina) e `consultas/pacotes` (paginação sem `COUNT`). O `pacotes` ainda tem `dao/`, `docs/` e `dto/` no mesmo nível da subfeature `detalhe/`: não copie esse arranjo.

1. **Path:** constante no `<Modulo>ApiPath` (`ROOT = ApiVersion.V1 + "/<modulo>"`), usada no `@Path`.
2. **Pacote `br.com.autsc2.api.modules.<modulo>.<feature>`:**
   - `Resource` e `Service` ficam na raiz;
   - `DAO` e `Filtro` também ficam na raiz quando únicos; com dois ou mais, vão para `dao/` e `filter/`;
   - DTOs vão em `dto/`; os modelos Swagger (`*Doc`) sempre em `docs/`;
   - ponte com o legado vai num `<Feature>LegacySupport`, com implementação `Default<Feature>LegacySupport`;
   - subfuncionalidade com qualificador próprio ganha subpacote (ex.: `pacotes/detalhe/`); feature composta usa `core/` mais as subfeatures.
3. **Registro, sem `@AutoService`:**
   - inclua o resource no `<Modulo>Resources`, que implementa `ResourceRegistry`;
   - módulo novo ganha uma linha no arquivo de `ServiceLoader` (`src/META-INF/services/...ResourceRegistry`);
   - atualize o `RestConfigurationTest`.
4. **`@Produces(MediaType.APPLICATION_JSON)` puro.** Com `;charset=UTF-8`, este RESTEasy (2.3.4) responde 406. O `ResponseUtil` já põe o charset.
5. **Swagger 1.5:**
   - tag em `SwaggerTags`, usada no `@Api`;
   - `@ApiOperation` com `response`;
   - `@ApiResponses` com `ApiErrorResponseDoc` nos erros;
   - `@ApiParam`, com `hidden = true` nos parâmetros internos;
   - envelopes documentados em `docs/*Doc`.
6. **Testes:** no mesmo pacote, em `test/` (ver "Testes e validação").

## Contrato REST

- **Path e resposta:** URL em `/autsc2/rest/api/v1/...`. O retorno passa por `ResponseUtil` (`ok`, `badRequest`, `unauthorized`, `forbidden`, `notFound`, `internalError`, `fromException`), que monta o envelope `{data, error}`. No DTO, data é `String` no formato `dd/MM/yyyy`; a conversão fica no DAO ou no `Service`.
- **Listagem:**
  - `page` começa em 1; `pageSize` tem padrão 20 e máximo 200 (`PageableFactory`);
  - o `PageResponse` traz `items`, `total`, `page`, `pageSize`, `totalPages`, `first`, `last`, `hasNext` e `hasPrevious`;
  - sort só por campo com `@SortColumn`: `sortBy` é o nome do campo no DTO, e o valor da anotação é a coluna SQL;
  - **legado com total** (Navigator "de N"): `PaginationQuery`, que faz `COUNT` e paginação por `ROW_NUMBER()` com `ORDER BY RN`. Filtro com mais de um `?` no mesmo trecho usa `filterAll(...)`;
  - **sem total exato:** peça `pageSize + 1` à origem, devolva `pageSize` itens e preencha `hasNext`/`hasPrevious`. Nada de `COUNT` só para montar a navegação;
  - consulta que não é SQL (servidor de regras): um helper da feature traduz o campo de ordenação, com a mesma whitelist (ex.: `PrestadorOrdenacao`).
- **Permissão:**
  - `@RequiresPermission(resource = "<modulo>.<tela>", operations = FuncaoOperacao.ACCESS)` vai na classe (`FuncaoOperacao` de `br.com.dynamix.framework.autsc.permission`);
  - escrita vai em métodos separados, cada um com uma única operação;
  - a chave é a mesma de `AUTSC2_MENUS.DS_CHAVE_PERMISSAO` e do `meta.recurso` da rota Vue;
  - o `PermissionApiFilter` exige `cd_menu` e `cd_modulo`, que o front propaga;
  - enquanto o menu não tiver `FG_USA_APP='S'` e a chave, a API responde 403 para todo mundo;
  - endpoint público só com `@PublicEndpoint`. Nunca invente chave de permissão.
- **Outro módulo:** chame por dependência Maven e chamada Java direta, nunca por reflection.

## Acesso a dados

- **SQL novo:** use `br.com.dynamix.framework.autsc.sql.SqlBuilder`, com parâmetro amarrado e a mesma `Connection` dentro de uma transação. Nada de `prepareStatement` em código novo.
- **DAO da API:** isola banco e legado. Pode reaproveitar DAO legado permanente (ex.: `AutSCFactoryDAO.getInstance().getListaPacoteDAO()`), fecha recursos no `finally` e converte exceção técnica em `ApiIntegrationException`.
- **`LegacySupport`:** só para dependência permanente (jars, parâmetros, domínios, serviços remotos), nunca para o handler CodeCharge da tela que está sendo substituída. Nesse caso, leve o SQL para uma DAO da feature e registre a duplicação no doc da tela (modelo: `dao/PrestadorRdaDAO` em `modules/consultas/prestadores`).
- **CRUD simples de uma tabela** (sem join, view, função ou paginação de tela): use a DAO de entidade do atd-persistence.
  - A DAO de feature a recebe no construtor público por `SqlUtil.getPersistence().<metodoDaFacade>()` (`br.com.dynamix.framework.autsc.SqlUtil`) e tem um construtor injetável para teste.
  - Precisa haver método dedicado na facade. Se faltar a entidade, a DAO ou o método, a mudança começa no atd-persistence.
  - **Não copie o `VersoesIntegracaoDAO`:** ele cria o próprio `SqlConnectionProvider` e instancia a DAO de entidade direto, o que o `padroes-vigentes.md` proíbe.
  - Uso da lib: `.claude/skills/dev-atd-persistence/SKILL.md`. Exceções dela viram `ApiBadRequestException`, `ApiIntegrationException` ou `ApiNotFoundException`.

## Testes e validação

- **Stack de teste:** JUnit 4 (4.11, `@Test`, `@Before`) e `mockito-all` 1.10.19, sem servidor e sem banco.
  - Resource: instancie direto, com service fake.
  - DAO: fake como classe privada no próprio teste.
  - Exceção: `try/catch` conferindo a mensagem.
- **Mínimo esperado:**
  - paginação: página e tamanho recebidos, normalização, limites, sort, resultado vazio e envelope;
  - sem `COUNT`: o intervalo pedido ao DAO, o `pageSize + 1` e o `hasNext`/`hasPrevious` nas páginas inicial, intermediária e final;
  - `RestConfigurationTest`, quando o registro mudar;
  - DTO que nenhum service ou resource exercita ganha teste dedicado em `dto/`.
- **JaCoCo:**
  - meta de linha (`LINE/COVEREDRATIO`) em 0.90, sobre `br/com/autsc2/api/**`;
  - fora da conta: `*Doc*`, `*ApiPath*`, `docs/*`, `ApiVersion*`, `*DAO*` e `*LegacySupport*`. **DTO conta.**
- **Comandos, da raiz do repositório:**
  - `mvn -q -Dfrontend.maven.skip=true verify`: testes e JaCoCo, sem o build do front;
  - um teste: `mvn -q -Dfrontend.maven.skip=true -Dtest=<Classe>Test test`.
- O CI (`mvn clean install`) roda os testes e o JaCoCo em qualquer branch, mas não roda o Vitest.
- **Fica como pendência:** conferir o Swagger (`/autsc2/rest/sguatd/swagger.json`, só localhost) e testar com o servidor de pé.

## Regras

| ID | Nível | Regra | Como verificar |
|---|---|---|---|
| ATD-AUB-001 | MUST | Resource no `<Modulo>Resources`, registry no arquivo do `ServiceLoader` e `RestConfigurationTest` atualizado; nunca `@AutoService` | diff e teste |
| ATD-AUB-002 | MUST | `.java` em ISO-8859-1; sem edição parcial com ferramenta que grava UTF-8; conferência por bytes | bytes do arquivo |
| ATD-AUB-003 | MUST | `@Produces(MediaType.APPLICATION_JSON)` sem charset | diff |
| ATD-AUB-004 | MUST | Envelope por `ResponseUtil`; `page` começando em 1, `pageSize` 20/200, sort por `@SortColumn`; sem total exato, `pageSize + 1` | testes de paginação |
| ATD-AUB-005 | MUST | `ACCESS` na classe e uma operação por método de escrita; chave igual a `DS_CHAVE_PERMISSAO` e a `meta.recurso` | diff |
| ATD-AUB-006 | MUST | SQL novo com `br.com.dynamix.framework.autsc.sql.SqlBuilder`; CRUD simples pelo atd-persistence via `SqlUtil.getPersistence()` | diff |
| ATD-AUB-007 | MUST | `verify` verde com o JaCoCo igual ou acima de 0.90 | comando |
| ATD-AUB-008 | MUST | Swagger completo (tag, `response`, erros com `ApiErrorResponseDoc`, `@ApiParam`) | diff |
| ATD-AUB-009 | SHOULD | `LegacySupport` só para dependência permanente | diff |
| ATD-AUB-010 | SHOULD | Subpacote técnico só com duas ou mais classes do tipo; `core/` só em feature composta | diff |

## Fora deste repositório

- **Banco e menu:** o autsc2 não tem pastas Liquibase. Banco e menu vão no **central do cmagnet-util** (skill `dev-atd-liquibase`), inclusive `AUTSC2_MENUS` (`DS_URL_APP`, `FG_USA_APP`, `DS_CHAVE_PERMISSAO`) e `AUTSC2_GRUPOS_FUNCOES`. As pastas novas do central usam `includeAll`, então o passo "registrar no `db-changelog.xml`" do `integracao-menu-spa.md` não vale para elas. No plano, o banco vem antes do app.
- **Entidade, DAO ou método na facade:** começam no atd-persistence, com `mvn -q install` local antes de compilar o autsc2.
- **Módulos de outros repositórios:** os do `cmagnet-util` (o autsc2 usa 21), as consultas ao servidor de regras (`fdsc-rules`) e o `proc-remoteutil` (`proc-guias`) mudam primeiro lá.
- **Mapa completo:** `dev-atd-ecossistema`.

## NUNCA

- Nunca use `@AutoService` nem registre resource direto na `RestApplication`.
- Nunca aplique o UPDATE de menu no banco, nem mesmo "para testar", como o doc sugere: isso é do dev.
- Nunca suba o servidor nem aplique Liquibase.
- Nunca copie para código, plano ou relatório o conteúdo de `Resources/deploy/oracle-autsc2-ds.xml`, `Resources/deploy/site.properties` ou dos `.ccs`: têm credenciais.
- Nunca troque versão de POM à mão.
