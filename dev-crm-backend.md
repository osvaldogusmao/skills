---
name: dev-crm-backend
description: "Convenções de desenvolvimento backend do produto NEXDOM CRM (repositório crm-backend, ecossistema UnimedSC — módulos crm-entities/crm-service/crm-web). Stack própria, NÃO compartilhada com RESSUS/NEXDOM DS: Java 8, Spring Boot 1.5.16, Jersey/JAX-RS (não Spring MVC), Oracle + Hibernate + QueryDSL, RabbitMQ, Redis, Feign para os serviços internos Z* (zworkspace/zpermission/zlogin/zbff/zprinter), multi-tenancy manual por coluna COD_EMP (não por schema/datasource). Use sempre que a tarefa envolver criar/alterar endpoints, Services, DAOs/Repositories QueryDSL, Entities/DTOs, integrações RabbitMQ/Feign, cache Redis, lógica multi-tenant, correção de bug, refatoração ou troubleshooting neste backend — mesmo que o pedido não mencione Jersey/QueryDSL explicitamente. NÃO usar para o frontend deste produto (ver nexdom-crm-frontend-dev) nem para outros produtos do ecossistema NEXDOM (RESSUS, NEXDOM DS, portais) — stacks incompatíveis."
---

# Backend NEXDOM CRM (crm-backend)

Convenções reais deste repositório e da lib compartilhada `service-core`/`project-core` (dependência direta) — não é documentação genérica de Spring Boot, é o que este projeto de fato faz, inconsistências incluídas. Se um caminho/classe citado aqui não existir mais, sinalize ao usuário em vez de assumir que o padrão ainda vale.

## Stack e módulos

Maven multi-módulo, dependência linear `crm-entities` ← `crm-service` ← `crm-web` (deployável). Java 8 / Spring Boot 1.5.16 — nada de streams/API Java 9+ nem recursos Spring Boot 2.x-only.

- **`crm-entities`**: `@Entity`, DTOs, VOs, enums, PKs — sem lógica de negócio. Pacote legado `entities/crm/vo/giu/*` (integração GIU/ANS) tem naming lowerCamelCase incomum — específico desse contexto legado, não replicar.
- **`crm-service`**: DAOs, Repositories QueryDSL, `impl/*ServiceImpl`, config (inclui RabbitMQ), `external/zprinter`, `omnichannel/*` (feature packages recentes).
- **`crm-web`**: camada HTTP (Jersey/JAX-RS), Feign clients (`webservice/*`), migrações Flyway (`db/migration`), Redis, config do Jersey.

**O framework HTTP é Jersey/JAX-RS (`javax.ws.rs.*`), não Spring MVC.** Controllers são `@Component` com `@Path`/`@GET`/`@POST`, registrados em `CrmJerseyConfig` (um `ResourceConfig` que varre beans `@Path`). Nunca use `@RestController`/`@RequestMapping` em código novo.

## Regra obrigatória #1: padrão "V2" para toda funcionalidade nova, sem exceção

Existe uma reescrita paralela e incompleta de entidades centrais (Protocol, Account, Category, Department, User...):

| | Legado (`XController`) | V2 (`XController2`) |
|---|---|---|
| Path | `/x` | `/x2` |
| Base class | `ServiceControllerAbstract` | `AbstractExceptionController` |
| Injeção | `@Inject` | `@Autowired` |
| Paginação | `Pagination` própria | `Page<T>` Spring Data real |
| Erro | ad-hoc | `toExceptionResponse(e)` centralizado |
| Camada de dados | `XxxDAO extends DAO<PK,T>` | `XxxService2` + `XxxQDSLRepository extends BaseQDSLRepository` |

**Confirmado pelo usuário**: o padrão V2 (`XController2`/`AbstractExceptionController`/`XService2`/`XQDSLRepository`/`Page<T>`) é obrigatório para toda funcionalidade nova — tanto para entidade que já tem versão "2" (não adicione endpoint/método/query nova na versão legada) quanto para entidade **totalmente nova**, que nasce direto no padrão V2, nunca no `DAO<PK,T>` legado — mesmo este ainda sendo majoritário em volume (~159 DAOs legados vs ~18 QDSLRepository). Manutenção pontual (correção, não feature nova) numa entidade só-legada ainda usa o padrão legado existente.

Para consulta dinâmica/relatório (filtros variáveis, paginação, ordenação), sempre `XxxQDSLRepository extends BaseQDSLRepository` (`crm-service/.../repository/`), independente da entidade ter ou não versão "2" de controller/service. Feature nova e complexa (ex. novo domínio de atendimento): pacote de feature próprio `omnichannel/<feature>/{controller,service,factory,payload,validator,exceptions}` (padrão `omnichannel/protocolo/`).

## Regra obrigatória #2: multi-tenancy por coluna COD_EMP — risco crítico

O tenant é o claim customizado **`empresa`** de um token OAuth2 (`SecurityContextHolder` → `OAuth2Authentication`), nunca header/path param. Para obter o tenant da requisição atual: `SessionUtil.getInstance().getEnterpriseId()` — não implemente uma segunda forma de extrair tenant. Isolamento é por coluna `COD_EMP`/`enterpriseId` (via `CompositeEnterprisePK<Long>`), não por schema/datasource — não há `AbstractRoutingDataSource`/`@Filter` do Hibernate nesta linhagem (`br.com.unimedsc.core.*`, via `service-core`→`project-core`; a linhagem paralela `io.fesc.core.*` usada por outro repo **não** é usada aqui — não misture as duas).

> **`DAO<PK,T>` genérico (project-core) NÃO filtra por tenant em nenhum método** — `findAll()`, `findAllPaged()`, query nativa via `returnResultList()`/`returnSingleResult()` rodam sem `COD_EMP`. Chamar `findAll()` numa entidade tenant-scoped **retorna dados de todos os tenants**. Sempre filtre explicitamente `enterpriseId`/`COD_EMP` (em QueryDSL: `entity.pk.enterpriseId.eq(tenantId)`), obtendo o tenant via `SessionUtil` (contexto HTTP) ou recebendo-o já resolvido a montante (contexto assíncrono, ver RabbitMQ abaixo).

- Processamento fora da thread HTTP (`@Async`, `ExecutorService`, scheduled job): não há `DelegatingSecurityContextExecutor` configurado — `SecurityContextHolder` não propaga. Capture `SessionUtil.getInstance().getEnterpriseId()` **antes** de sair da thread da requisição e passe o valor explicitamente.
- `Enterprise`/`COD_EMP` (tenant) e "Unimed" **não são sinônimos** — uma Enterprise pode ter sub-unidades Unimed com integrações próprias; se a tarefa mencionar "Unimed" sem deixar claro se é o mesmo escopo do tenant, confirme antes de implementar.

## Acesso a dados (Oracle/Hibernate/QueryDSL)

- **Geração de PK não é `@GeneratedValue`/sequence simples**: entidades tenant-scoped (`EntityAbstractEnterprise<CompositeEnterprisePK<Long>>`) geram ID no `@PrePersist` via function PL/SQL `K_<PREFIXO>_CONTROLE_NUMERACAO.F_BUSCA_PROX_NUMERO(COD_EMP, tabela)`, executada via JDBC direto (`KeyGenerator.newKey`, fora do Hibernate). IDs são únicos por `(COD_EMP, tabela)`, não globalmente. Entidade tenant-scoped nova precisa da tabela de controle correspondente.
- `@Transactional` na camada de Service/Impl (quase nunca em DAO); para código novo prefira `org.springframework.transaction.annotation.Transactional` (mais usado e integrado ao resto da config Spring) em vez do `javax.transaction.Transactional` legado.
- Paginação: padrão legado usa `Pagination`/`Filter`/`Node`/`Comp`/`FilterHelper`/`Order` (tudo em `project-core`); padrão QueryDSL usa `Page`/`Pageable` do Spring Data + `fetchPage(query, filter)` de `BaseQDSLRepository` + `Projections.constructor(...)`. Não misture os dois estilos numa mesma feature.
- QueryDSL 4.2.2 — classes `Q*` são geradas em `target/generated-sources`, nunca editar manualmente.
- `DAO.executeFunction(...)` monta SQL nativo por concatenação de string — risco de SQL injection se parâmetro vier de input de usuário sem validação; valide sempre que usar esse método com dado de request.

## RabbitMQ (padrão consolidado, integração `zprinter`)

Único caso real de RabbitMQ hoje (geração assíncrona de documentos via serviço externo `zprinter`). Infra a replicar para uma fila nova: `RabbitConfig` (exchange direct durável, filas `durable`+`x-queue-mode=lazy`, DLQ, `RetryInterceptor` com 3 tentativas restrito a `RabbitBusinessException`, `RepublishMessageRecoverer` na DLQ ao esgotar), `CrmRabbitPostConfig` (`@PostConstruct @Profile("!test")` que declara a infra e injeta o retry interceptor), `QueueProperties` (nomes de fila/routing key/exchange via `@Value`, nunca hardcoded), `CustomRabbitTemplate` (wrapper com `Jackson2JsonMessageConverter`). Producer/consumer de exemplo: `PrinterService.sendToPrint` / `PrinterConsumer.printerOut`.

**Propagação de tenant em mensagem assíncrona — regra obrigatória para qualquer consumer novo que acesse dado de negócio**:
1. Producer envia `empresaId`/`COD_EMP` num header dedicado (seguir o padrão do header `zrouter`).
2. Listener chama `AuthenticationContext.setTenantId(...)` **antes** de qualquer chamada a repository/service.
3. Limpar em `finally` com `AuthenticationContext.clearAuthentication()`, sempre, mesmo em exceção — evita vazamento de tenant entre mensagens na mesma thread do listener container.

Anti-pattern: retry manual dentro do listener — quem decide retry/DLQ é o `retryInterceptor` central; o listener deve relançar a exceção. Não há idempotência (dedup por message-id) nos consumers hoje — se a tarefa envolver reprocessamento que não pode duplicar (ex. gerar protocolo duas vezes), implemente idempotência explícita (padrão similar ao `IdempotencyInterceptor` do Redis, abaixo).

## Feign clients

Padrão a seguir para cliente novo (serviços internos "Z*" — zworkspace, zpermission, zlogin, zbff): `@FeignClient(name = "z-workspace", url = "${z-workspace.url}", configuration = NexdomAutorizedClientConfig.class)`, URL sempre externalizada, `configuration` própria estendendo `BaseOAuthConfig` (resolve headers/URL de auth via `IntegrationManagerService.getConfig(integrationType)`) — subclasses só implementam `integrationType()`. Não replicar o padrão legado sem `configuration` e com retorno `Object` não tipado (ex. `ProtocoloClient`), mesmo que o serviço de destino já tenha um client nesse formato. Não há Hystrix/`Retryer` customizado/`fallback=` — resiliência (timeout, circuit breaker) não tem precedente local, é decisão nova a discutir, não convenção a replicar.

## Redis / Cache

`@Cacheable` extensivo (26+ classes) — **regra obrigatória**: toda key de cache tenant-scoped inclui `enterpriseId`, ex. `@Cacheable(value = "crm/category", key = "{ #categoryId, #enterpriseId })`. Cache sem essa chave vaza dado entre tenants. Convenção de nome: `"<domínio>/<entidade>"`. Idempotência de requisição via `IdempotencyInterceptor` (chave MD5 de `method:uri:params:body`, prefixo `protocolador/idempotencia:`, TTL configurável) — hoje só na rota `/wsc/geraprotocolo`; replique esse interceptor para outra rota que precise da mesma proteção, não implemente verificação manual no Service. **Não replicar**: `UtilsCacheable.java` tem credenciais hardcoded como fallback — falha de segurança existente, sinalize se tocar no arquivo.

## APIs REST (Jersey) e erros

- `@GET`/`@POST`/`@PUT`/`@DELETE`, `@QueryParam`, `@PathParam` de `javax.ws.rs.*`; path em camelCase, verbos em inglês. Maioria usa `@POST` mesmo para busca com filtro complexo (critérios no corpo); `@GET` só para busca simples por id/path param.
- Controllers V2 retornam `Page<T>`; controllers legados usam `Pagination` própria — não misture estilos numa mesma resposta.
- Swagger: `swagger-jaxrs2` registrado manualmente em `CrmJerseyConfig` (não é `springdoc-openapi-starter`, que pressupõe Spring MVC) — anote controllers novos com `@Tag(name=...)` mesmo que o vizinho legado não tenha.
- Sem `@ControllerAdvice`/`@ExceptionHandler` (não é Spring MVC). Handler global `CrmExceptionMapper implements ExceptionMapper<Exception>` cobre poucos tipos (rede de segurança, não mecanismo principal). Em controller novo (padrão V2), envolva a lógica em try/catch e retorne `AbstractExceptionController.toExceptionResponse(e)` no catch — não é automático. Formato de erro: `ErrorResponse { status, code, error, message }`.

## Testes: não existem no repositório hoje

Nenhum teste automatizado (JUnit/Mockito/integração) existe em `crm-entities`/`crm-service`/`crm-web`. É um gap real, não uma convenção respeitada — não afirme "os testes passaram", não existe suíte para rodar. Se a tarefa pedir teste explicitamente ou uma correção crítica justificar teste de regressão, proponha um teste mínimo (JUnit 4 + Mockito, compatível com Java 8/Spring Boot 1.5.16 — `spring-test` 4.3.19 está no classpath, sem evidência de JUnit 5), deixando claro que não há padrão pré-existente a seguir. Para validar sem testes: compile o módulo afetado (`mvn -pl <módulo> compile`), revise manualmente o fluxo, e descreva ao usuário exatamente o que foi validado e o que não foi.

## Domínio — conceitos fáceis de errar

- **Protocol** é o núcleo (entidades satélite: Movement, Access, Archive, Follow, Form, Mail, Task, + Omnichannel). `StatusProtocol` tem **três** estados de encerramento distintos (`ENCERRADO`, `ENCERRADO_INTERNAMENTE`, `ENCERRADO_EXTERNAMENTE`) — ao checar "protocolo está fechado?", verifique os três explicitamente; não existe helper único que já trate isso.
- **AttendanceGroup** (fila de atendimento) liga `Category`/`Channel`/`Reason`/`SubReason`/`Department` — árvore de classificação de roteamento.
- **Enterprise vs Unimed**: ver regra na seção de multi-tenancy acima.

## Anti-patterns identificados neste projeto

- `findAll()`/`findAllPaged()`/query nativa via `DAO<PK,T>` numa entidade tenant-scoped sem filtro de `COD_EMP`/`enterpriseId`.
- Criar um novo mecanismo de identificação de tenant (header custom, path param) em vez de usar o claim `empresa` via `SessionUtil`.
- Retry manual dentro de `@RabbitListener`, ou criar exchange/fila do zero em vez de seguir `RabbitConfig`/`QueueProperties`.
- Consumer RabbitMQ que acessa dado de negócio sem restaurar tenant (`AuthenticationContext.setTenantId`) e sem limpar em `finally`.
- Cachear dado tenant-scoped sem `enterpriseId`/`COD_EMP` na chave do `@Cacheable`.
- `@RestController`/Spring MVC em vez de JAX-RS/Jersey.
- Introduzir Hystrix/Resilience4j/WebClient reativo sem necessidade comprovada — sem precedente local, e Spring Boot 1.5.16/Java 8 limita opções.
- Assumir que `SessionUtil`/`SecurityContextHolder` funcionam fora da thread HTTP original (`@Async`, `ExecutorService`) sem capturar o tenant antes.
- Duplicar Feign client/DAO já existente para o mesmo domínio só porque a implementação encontrada é "legada" — avalie estender/tipar antes de reescrever do zero.
- Copiar credenciais para código-fonte (há um caso legado em `UtilsCacheable` — falha existente, não padrão a seguir).
- Adicionar endpoint/método/query nova na versão legada (`XController`/`XDAO`) de uma entidade que já tem versão "2".

## Troubleshooting rápido

- **Dado de outro tenant aparecendo**: causa mais provável é `findAll()`/`findAllPaged()`/query nativa sem filtro de `COD_EMP` — cheque o DAO/Repository da entidade. Segunda causa mais provável: chave de `@Cacheable` sem `enterpriseId`.
- **Erro/latência em carteirinha/protocolo (fluxo `zprinter`)**: checar nessa ordem — fila DLQ (`crm-workspace.queue.dlq`, esgotou as 3 tentativas?), tipo de exception no `PrinterConsumer` (`RabbitBusinessException` é retryable, outra não), propagação do header `zrouter`/`empresaId`.
- **Requisição duplicada processada duas vezes**: verificar se a rota está coberta pelo `IdempotencyInterceptor` (hoje só `/wsc/geraprotocolo`).
- **Falha de chamada a outro serviço (Feign)**: sem timeout/circuit breaker configurado, a falha sobe direto como exception até `AbstractExceptionController`/`CrmExceptionMapper` — checar `configuration` do client (`BaseOAuthConfig`/subclasses) para erro de autenticação entre serviços.
