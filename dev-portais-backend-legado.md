---
name: dev-portais-backend-legado
description: "Convenções dos backends ANTIGOS dos portais NEXDOM (SGU Portais, grupo SGU3.0 no git.fesc.io, pasta <portal>/backend: administracao_empresa, administracao_beneficiario, administracao_operadora, movimentacao, sgupeb-components). Stack: Java 11, Spring Boot 2.2.2 + Spring Cloud Hoxton, Spring MVC REST + GraphQL SPQR ainda vivo (consumido pelos fronts Vue 2), Spring Data JPA/Data REST sobre Oracle + Flyway, OpenFeign + lib io.fesc.webservice para os webservices do SGU, segurança JWT e tenant pela lib interna io.fesc:core, Lombok, JUnit 4 + Mockito, Checkstyle 8.27, pipeline .gitlab-ci.yml. Use sempre que a tarefa (chamado, bug, feature, refatoração, revisão) tocar controller, resolver, service, repository, entidade/DTO, migration ou client Feign nesses repos, mesmo sem citar Spring. NÃO usar para os frontends (dev-portais-frontend-legado), nem RESSUS/NEXDOM DS (dev-spring-boot) ou CRM (dev-crm-backend)."
---

# Backend SGU Portais (legado, repos SGU3.0)

As regras do time estão na seção "Regras do time" abaixo; se o workspace tiver `AGENTS.md`/`CLAUDE.md`, ele complementa e prevalece em conflito. Se uma classe citada aqui não existir mais, avise em vez de assumir o padrão.

## Stack e repositórios

| Repo (pasta `<portal>/backend`) | Serviço (`spring.application.name`) | Pacote raiz | Front que consome | Particularidades |
|---|---|---|---|---|
| `administracao_empresa/backend` (`ape`) | `portal-administracao-empresa` | `io.fesc.portal.empresa` | `empresa/frontend`, `administracao_empresa/Frontend` | `api/` guarda documentos/tags servidos ao components; Redis; `pl-api-connector` (`@ComponentScan` inclui `br.com.zitrus`) |
| `administracao_beneficiario/backend` (`apb`) | `portal-administracao-beneficiario` | `io.fesc.portal.beneficiario` | `beneficiario/frontend`, `administracao_beneficiario/Frontend` | `@Cacheable` com Redis, `plapiconnect`, `@EnableScheduling` |
| `administracao_operadora/backend` (`apo`) | `portal-administracao-operadora` | `io.fesc.portal.operadora` | `operadora/frontend`, `administracao_operadora/frontend` | Data REST sem restrição (ver Acesso a dados); stubs do `administracao` em teste |
| `movimentacao/backend` (`amc`) | `movimentacao` | `io.fesc.movimentacao` | `movimentacao/frontend` e os portais (`apiDomain('movimentacao')`) | o maior: RabbitMQ (`app/config/rabbit*`), Spring Batch, `@Audited` (io.fesc.core.audit), Freemarker + html2pdf, Spring Cloud Contract, `@EnableAsync` |
| `sgupeb-components/backend` (`components`) | `sguweb-components` | `io.fesc.portal.components` | `sgupeb-components/frontend` (lib usada pelos portais) | sem datasource/Flyway; Feign para os três portais e para o amc; `io.fesc:core` 0.0.143-SNAPSHOT (os outros usam 1.0.2) |

Comum: Maven **sem wrapper**, Spring Boot `2.2.2.RELEASE`, Java 11, Spring Cloud `Hoxton.SR1` (Eureka, Kubernetes Ribbon, Sleuth), `io.fesc:core`, `io.fesc.webservice:client` (1.0.29 a 1.0.48 conforme o repo), `graphql-spqr-spring-boot-starter` 0.0.4, `ojdbc8`, Lombok, `spring-boot-starter-test` + H2. **Não existe** springdoc/Swagger, `spring-boot-starter-validation` explícito (vem pelo web do Boot 2.2), `.githooks`, `.cicd/` nem suíte de testes. Repos remotos: `git.fesc.io/SGU3.0/portal/<portal>/backend` (amc em `SGU3.0/SistemasdeValorAgregado/movimentacao/backend`); o `kubernetes/` é submódulo de `infrastructure/sgusuite`.

**Não vale aqui**: Java 17, Boot 2.7, QueryDSL, JUnit 5, `jakarta.*`, `getReferenceById()` (use `getOne()`), `z-audit-lib`, serviços `Z*`. Java 11 não tem `record`, `sealed`, text block, `switch` expressão, pattern matching, `Stream.toList()`, `String.formatted()`.

## Regras do time

- **Menor alteração segura**: entender o fluxo e procurar o padrão existente antes de editar; sem mudança cosmética junto de correção funcional; sem atualizar dependência, plugin ou versão; sem abstração nova quando uma existente resolve.
- **Coesão**: cada campo do payload é preenchido num lugar só (se o método que monta o objeto tem os dados, a decisão fica nele, não no chamador); uma regra, um `if`, sem repetir a decisão em outro ponto do fluxo.
- **Defensividade na medida**: antes de espalhar checagem de nulo, confira se o campo pode ser nulo ali. Nada de `Boolean.TRUE.equals` e `Boolean.FALSE.equals` no mesmo fluxo (três caminhos para um domínio de dois): trate o nulo na origem e use `if/else`. Não remonte valor que já chega pronto (sem `substring` + concatenação para reconstruir chave).
- **Nome de domínio**: condição com mais de um termo vira método privado com nome de domínio (`isInclusaoTitular`, `isContratoVendaAlterado`); comparação de id/código usa o método do enum quando existe (`TiposMovimento.isInclusao(id)`, não `TiposMovimento.INCLUSAO.id().equals(id)`); nada de `data`, `item`, `obj`, `temp` quando há nome melhor.
- **Não duplicar** lógica, DTO, validação ou utilitário; comentário só quando explica o porquê, nunca repetindo o código.
- **Erros e dados**: não ocultar erro, não capturar `Exception` genérica, não usar `Optional.get()` sem checar, não logar token, senha, dado pessoal ou payload sensível.
- **Editar arquivo existente**: ao adicionar import, inserir só a linha nova na posição certa, sem reordenar o bloco. Antes de criar método em classe existente, confira se o nome já está em uso (sobrecarga com propósito diferente confunde: use nome distinto). A mesma operação em vários controllers tem o mesmo nome de método em todos. Edição em lote por script exige conferir o diff (formatação e posição).

## Estrutura de pacotes

Um pacote por domínio: `<raiz>/<dominio>/` com `XxxController`, `XxxResolver`, `service/XxxService` (interface) + `XxxServiceImpl`, `payload/` (`XxxRequest`/`XxxResponse`, nunca `Dto`), entidade `Xxx`, `XxxRepository`, `XxxFactory`, `exception/`, `enumerator/`. `app/` tem a `*Application` e `config/` (`WebSecurityConfiguration`, `RepositoryRestConfig`, `RedisConfig`). `webservice/` tem os clients, um subpacote por serviço (`administracao`, `movimentacao`, `wsadministration/<recurso>`, `wsfesc`) e `WebServiceExceptionHandler`. No amc, `api/**` são endpoints de integração protegidos por `IntegracaoTokenInterceptor` (`app/config/WebConfig`) e o `OriginInterceptor` guarda `efetivar`/`auditar`. A `*Application` fixa `@ComponentScan`/`@EnableFeignClients`/`@EnableJpaRepositories`/`@EntityScan("io.fesc")` e `@ImportAutoConfiguration(DefaultFeignRequestConfiguration.class)` (components não tem). Não mexer nisso junto de outra tarefa.

## GraphQL x REST: como se faz hoje

- O GraphQL SPQR é contrato **vivo**: cada backend expõe `/graphql` e os fronts montam a query à mão (`.graphql(api.graphql, query)` com a lista de campos). Só `@GraphQLQuery`, nenhuma `@GraphQLMutation`: escrita sempre foi REST.
- **Feature nova = endpoint REST** (controller + service + payload). Nenhum resolver novo desde 2025. Exemplo: ape `EmpresaController` `GET empresa/paginado` (PRT-I189), que substituiu uma query no front.
- Não criar resolver novo e **não remover** resolver ou query existente (quebra tela que ainda usa), nem migrar resolver para REST sem pedido. Resolver é casca: `@Component @GraphQLApi`, injeção por construtor, método `@GraphQLQuery` que só delega ao service (modelo: `agrupamento/AgrupamentoResolver`). No amc também há `@GraphQLApi` direto no `ServiceImpl` com `@GraphQLQuery(name = "...")` (`critica/service/CriticaServiceImpl`).
- Bug em dado que a tela lê por GraphQL: corrija no service, que resolver e controller compartilham. Antes de mexer num tipo devolvido por query, procure o nome da query nos fronts da tabela (`grep -rn "nomeDaQuery" <portal>/frontend/src` em cada front da tabela). Campo novo é compatível (o front pede se quiser); renomear, remover, mudar tipo ou pôr `@NotNull` em campo quebra a query.
- Argumento `@NotNull` em método de resolver vira non-null no schema; paginação de query é `Integer page, Integer size, String order` + `Pagination.of`, ou `Filter` do io.fesc.core.

## REST

- `@RestController` + `@RequestMapping` em snake_case, quase sempre plural (`inclusao_alteracao_beneficiarios`, `movimento_criticas`, `exclusoes_solicitadas_empresa`); há exceções legadas (`empresa`, `valor`). Siga o controller do domínio; sub-rota também snake_case (`{movimentoId}/efetivar`).
- Controller fino, delega ao service e devolve payload, `List` ou `Page` tipado. Injeção por construtor (`@Autowired` em campo no amc é legado, não replicar). Sem sobrecarga de método no controller.
- Paginação: `@RequestParam(defaultValue = "0") Integer page`, `size`, `order` + `io.fesc.core.search.Pagination.of(page, size, order)`; alguns recebem `Pageable`. Data: `@RequestParam @DateTimeFormat(pattern = "yyyy-MM-dd") LocalDate`. Upload: `consumes = MediaType.MULTIPART_FORM_DATA_VALUE` + `MultipartFile`.
- Endpoint novo precisa ser chamado pelo front legado à mão (`api.js` da view): informe rota, método, parâmetros e corpo no resumo.

## DTOs e validação

- `payload/` com `@Getter @Setter @NoArgsConstructor`; `@Data` só aparece em payload de relatório, nunca em entidade.
- `javax.validation.*`: `@NotNull`/`@NotBlank` com `message` em português no campo (ex.: `usuario/payload/UsuarioCreateRequest`, `parametrovalor/payload/ParametroValorRequest`) e `@Valid @RequestBody` no controller (`InclusaoAlteracaoBeneficiarioController`). O erro vira 400 pelo `io.fesc.core.validation.ResponseExceptionHandler`.
- Nenhum repo usa `@Validated` nem tem handler de `ConstraintViolationException`: restrição em `@RequestParam`/`@PathVariable` ou em elemento de lista não funciona sem isso. Use `@RequestParam` obrigatório (default do Spring) e valide o resto no service com exceção de domínio.
- Não reaproveite como request uma classe que também é devolvida por resolver (validação nela muda o schema).

## Acesso a dados

- Cada portal tem schema Oracle próprio, `ddl-auto=validate`: mudança em entidade exige migration. Entidade sem `@Table`, PK `@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "x")` + `@SequenceGenerator(name = "x", sequenceName = "s_<tabela>", allocationSize = 1)` (modelo: `usuarioemail/UsuarioEmail`), ou UUID (`uuid2`).
- Dado do SGU (beneficiário, contrato, fatura, plano, empresa contratante) não vem por JPA, vem dos webservices.
- Repository `JpaRepository` + `JpaSpecificationExecutor` para filtro dinâmico: `repository.findAll(Filter.search(filter), Filter.page(filter))` (`parametro/service/ParametroServiceImpl`). Em `@Query`, `@Param` de `org.springframework.data.repository.query.Param`, não `feign.Param` (erro presente em `EmpresaContratoRepository`).
- Regras do time: nada de `findAll()` sem limite em tabela que cresce (os existentes são de domínio pequeno, como `ParametroTipoValor`), consulta ou chamada Feign em loop (N+1), `FetchType.EAGER`, `save()` em loop (use `saveAll`). Prefira projeção, `JOIN FETCH`, agregação no banco e `getOne()`.
- `@Transactional` (Spring) no service, `readOnly = true` em consulta. É raro (1 classe em ape/apb/apo, 23 no amc), mas escrita em mais de uma entidade precisa dele. Não segure transação durante chamada Feign.
- Spring Data REST: ape, apb e amc têm `detection-strategy=annotated` + `RepositoryRestConfig` que desliga **só o GET**; quem tem `@RepositoryRestResource` fica aberto para POST/PUT/PATCH/DELETE sem regra (ape: `usuario_emails`, `empresa_contratos`, usados pelo front de admin; e vários no amc). O apo tem o starter sem `annotated` nem `RepositoryRestConfig`, então pela regra padrão todo repository público é exportado, inclusive GET (não verificado em runtime: avise, não corrija junto). Nunca anote repository novo: controller + payload.

## Feign e webservices

- Serviços da plataforma: `@FeignClient(name = "movimentacao", url = "${movimentacao.url:}", contextId = "movimentacao", configuration = DefaultFeignRequestConfiguration.class)` (ape `webservice/movimentacao/MovimentacaoClient`, `webservice/administracao/AdministracaoClient`; components `webservice/portaladministracaoempresa/PortalAdministracaoEmpresaClient`, que por usar o core antigo declara sem `configuration`). URL local em `application-development.properties` (`${URL_MOVIMENTACAO:...}`), timeout em `feign.client.config.<nome>.*`.
- Webservices do SGU por tenant (WsAdministration/WsFesc): classes `XxxWebservice` da lib `io.fesc.webservice.client.*`, chamadas com `.from(administracaoClientService.getWsAdministrationUrl())`. Prefira `getWsAdministrationUrl()` a `getEndpoint()`: só ele cai no `TenantContext` quando não há usuário. Modelo: `webservice/wsadministration/beneficiario/service/BeneficiarioApeServiceImpl`.
- O domínio chama o service do webservice, nunca o client. O service captura `FeignException` e segue o padrão do próprio arquivo: `validateError(e)` com `HttpResponseValidator` → `ErrorResponseException` (`MovimentacaoClientServiceImpl`), ou `webServiceExceptionHandler.getMessage(e.status(), "Servico - metodo")` + `WebServiceException(WebService.WS_ADMINISTRATION.getOrigem(), ...)`.
- Reutilize os DTOs de `io.fesc.webservice.client` antes de criar. Sem `RestTemplate`/`HttpClient`. Não logar payload com dado pessoal nem token; não disparar escrita contra serviço de dev.

## Segurança, tenant e autorização

- `io.fesc.core.security.WebSecurityConfiguration` autentica tudo (JWT, stateless). Cada repo (menos components) tem `app/config/WebSecurityConfiguration` com bean nomeado (`@Configuration("EmpresaWebSecurityConfiguration")`) só para `web.ignoring()` de rota pública (login, logout, recuperar senha, primeiro acesso). Não libere rota sem pedido explícito.
- `io.fesc.core.security.Authentication` injetado no construtor: `getUser()`/`getTenant()` lançam `AuthenticationException` sem contexto; `findUser()`/`findTenant()` para client_credentials; `isAdmin()`, `isCompanyAdmin()`, `isClientOnly()`. Estático: `AuthenticationContext`.
- Tenant é a empresa do SGU ligada à Unimed (`AdministracaoClientService.getUnimed(tenantId)`), **não** a empresa contratante que chega como `empresaId`/`codigoEmpresa`. Fora de request, `TenantContext`; em `@Async` no amc, `app/async/TenantAwareTaskDecorator`. `tenant.enabled` (`TENANT_ENABLED`, default `false`): um datasource por serviço.
- Autorização de dado é manual no service, sem `@PreAuthorize`. Código vindo do request é validado contra o usuário logado: ape `EmpresaUsuarioService.validaAcessoCodigoEmpresa(codigoEmpresa)` (`EmpresaUsuarioAccessUnauthorizedException`) e `EmpresaContratoService.getContratosUsuario(usuarioId, empresaId)`; apb `UsuarioService.validaAcessoCodigoBeneficiario` e `validaAcessoDadosPessoais`.

## Erros e log

- Exceção de domínio em `<dominio>/exception/`: `extends RuntimeException` + `@ResponseStatus(HttpStatus.X)`, mensagem em português no construtor (`parametro/exception/JsonObjectException`). Genéricas do core: `NotFoundException` (404), `ErrorResponseException(msg)` (400).
- Não há `@ControllerAdvice` global nos repos (só `PLApiConnectClientExceptionHandler`); não crie formato de erro novo.
- Log com `@Log` (java.util.logging) e causa junto: `log.log(Level.SEVERE, mensagem, e)`. O legado `log.info(e.getMessage())` + `return null` (`UnimedServiceImpl.getOperadora`) não se copia. Sem `System.out`/`printStackTrace()`.

## Flyway, configuração e dev compartilhado

- `src/main/resources/db/migration/V000NNN__<verbo>_<objeto>.sql` (`create_x_table`, `alter_x_table`, `insert_parametro`, `insert_parametro_valor`), tabela `MIGRATIONS`, `out-of-order=true`, callback `afterMigrateError__delete_from_migrations.sql`. Confira a última na `develop` antes de numerar; nunca altere migration aplicada. DDL: `NUMBER(18)`, `VARCHAR2(n CHAR)`, constraints nomeadas (`<tabela>_pk`, `<tabela>_<col>_uk`), `COMMENT ON`, sequence `s_<tabela>` `NOCACHE ORDER` (ape `V000017__create_usuario_email_table.sql`). Parâmetro de funcionalidade (`Parametro`/`ParametroValor`) entra por migration (ape `d95e268e`: V000146 a V000150).
- `FLYWAY_ENABLED` é `true` por default: apontando para dev, suba com `FLYWAY_ENABLED=false` e rode `mvn clean` antes (cópia antiga em `target/classes` volta a rodar).
- Dev (`sgu3dev1`, `unim07h`, serviços em backend.dev) é compartilhado: só SELECT leve; nada de INSERT/UPDATE/DELETE/DDL/massa. Dado real: ler do dev e gravar num Oracle local (`.local-stack/<assunto>/`). Evidência com massa fictícia diz isso.
- Configuração nova: `${VARIAVEL:default}`, sem credencial nem default de dev. A variável vai nos `deployment*.yaml` do submódulo `kubernetes/sgusuite/templates/portal/<app>/` (amc: `templates/movimentacao/`; components: `k8s/` no próprio repo). Submódulo é outro repo: avise, não commite por aqui.

## Testes, checkstyle e pipeline

- Quase não há testes: smoke `*ApplicationTest` (`@SpringBootTest`) em todos, mais ape `RelatorioAnualDespesasTitularServiceImplTest` (pacote errado, `relatorioanualdespesas`) e amc `movimento/service/MovimentoServiceImplTest` + contratos em `src/test/resources/contracts`.
- Regra nova ou bug corrigido ganha teste unitário do service: JUnit 4 (`org.junit.Test`), `@RunWith(MockitoJUnitRunner.StrictStubs.class)`, `@Mock`/`@InjectMocks`, AssertJ, métodos `shouldXxx`, dados `makeXxx` fictícios, client/webservice mockado, no pacote espelho `src/test/java/io/fesc/...`. Controller, se testar, com `MockMvcBuilders.standaloneSetup` (não há exemplo no legado). Não introduza JUnit 5.
- Comandos: `mvn test -Dtest=XxxServiceImplTest` e `mvn checkstyle:check` (ambos rodam offline com o `~/.m2` local). O CI (`.gitlab-ci.yml`, job `build`) roda `mvn clean package -U` e depois `mvn checkstyle:check`; o checkstyle não roda no `package`, rode à parte. `mvn test` completo sobe o smoke com o contexto inteiro, e o `src/test/resources/application.properties` do ape tem defaults de Redis e URLs de dev: prefira `-Dtest`.
- Checkstyle 8.27, `checkstyle.xml` idêntico em ape/apb/apo/components (amc aceita `@SuppressWarnings` e complexidade 20; `suppression.xml` libera `MagicNumber` em `*Request.java`):
  - linha ≤ 120; linha em branco antes de `if`/`return` após statement e depois de `}`; sem linha em branco no início/fim de bloco nem dupla;
  - proibidos: ternário (`AvoidInlineConditionals`), `catch (Exception e)` (`IllegalCatch`), `HashMap`/`ArrayList` em declaração (`IllegalType`), `TODO`;
  - `MagicNumber` e `MultipleStringLiterals` viram constante; `FinalClass`, `DeclarationOrder`, `HiddenField`, `RequireThis`;
  - imports: terceiros em ordem alfabética (`io`, `org`...), linha em branco, `javax`, `java`, estáticos por último. Ao adicionar, insira só a linha nova.

## Commits e MR

Branch a partir de `develop`: `feature/PRT-IXXX`, `feat/PRT-IXXX` ou `fix/PRT-IXXX`. Commit de uma linha `PRT-IXXX: descrição curta` em minúsculas (apb `PRT-I181: corrige enumeracao de usuarios no primeiro acesso`); não há hook de commit-msg nem Conventional Commits no backend (o commitlint é só dos fronts). **Nunca** adicionar `Co-Authored-By:`, `Claude-Session:`, `Generated with` ou equivalente na mensagem. O MR entra com squash e o título costuma ser o nome da branch (`feature/PRT-I717`). Template `.gitlab/merge_request_templates/SGUWEB3_0.md`: **Atividade** (link do item PRT-IXXX) e **Realizado** (2 ou 3 bullets curtos).

## Anti-patterns

- Criar `@GraphQLQuery`/resolver novo, remover query que o front usa, ou mudar/remover campo de tipo devolvido por query sem conferir os fronts.
- Endpoint que aceita `empresaId`/`codigoEmpresa`/`codigoBeneficiario` sem validar contra o usuário logado, ou que confunde tenant com empresa contratante.
- Novo `@RepositoryRestResource`, entidade JPA exposta na API, ou "arrumar" Data REST/`*Application` junto de outra tarefa.
- Client Feign chamado direto do domínio, webservice em loop, segundo padrão de tratamento de `FeignException` num service que já tem um.
- `@Autowired` em campo, `@Data` em entidade, `feign.Param` em repository, `Date`/`SimpleDateFormat`, `new BigDecimal(double)`, recurso Java > 11, API de Boot > 2.2, JUnit 5.
- Flyway ligado apontando para dev, escrita em banco ou serviço de dev, credencial ou default de dev em properties.

## Troubleshooting rápido

- **401 em tudo local**: falta Bearer; sem token só respondem as rotas de `web.ignoring()`.
- **Tela quebrou após mudança no backend**: query GraphQL pedindo campo renomeado/removido. Procure a query no front e restaure o campo.
- **"O serviço (X) do sistema de gestão está indisponível"**: `WebServiceExceptionHandler` para 500/503 do SGU; confira a integração do tenant (`IntegracaoService`, `TipoIntegracao.WS_ADMINISTRATION`).
- **`AuthenticationException` em job, rota pública ou `@Async`**: sem usuário; use `find*()`, `getWsAdministrationUrl()` com `TenantContext`, e o `TenantAwareTaskDecorator` no amc.
- **"Schema-validation: missing table/column" ao subir**: `ddl-auto=validate` com migration faltando ou Flyway desligado na base local.
