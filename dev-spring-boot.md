---
name: dev-spring-boot
description: "Convenções de desenvolvimento backend Java/Spring Boot usadas nos produtos NEXDOM (confirmado no RESSUS e no NEXDOM DS) — Java 17, Spring Boot 2.7, organização por feature (package-by-feature), Spring Data JPA + QueryDSL sobre Oracle com migrações Flyway, REST (e GraphQL quando o produto expõe), Spring Cloud OpenFeign para integração com os microsserviços internos da plataforma (prefixo Z, ex. zworkspace, zstorage, zpermission, zlogin, zbff, zprinter, znotifica, zdata), RabbitMQ para eventos assíncronos, OAuth2, auditoria via z-audit-lib, e Checkstyle obrigatório no build. Use sempre que for implementar, revisar ou corrigir código backend Java desses produtos — mesmo que o pedido não mencione \"Spring Boot\" explicitamente, mas descreva um endpoint, uma regra de negócio, uma migração de banco ou uma integração com outro serviço da plataforma."
---

# Backend Java/Spring Boot — produtos NEXDOM

## Stack

- Java 17+, Spring Boot 2.7.x, build com Maven.
- Persistência: Spring Data JPA + QueryDSL (queries type-safe complexas) sobre Oracle (driver `ojdbc10`); testes de repositório/integração usam H2 em memória.
- API: REST sempre; **alguns produtos** (confirmado no RESSUS) também expõem GraphQL lado a lado — verifique se existe `src/main/resources/graphql/schemas.graphqls` antes de assumir que o produto tem GraphQL (o NEXDOM DS, por exemplo, é só REST). REST é documentado com springdoc-openapi em ambos os casos.
- Integrações específicas de produto (não universais, mas confirmadas em pelo menos um produto): geração de PDF (`pdfbox`), assinatura digital (`signer-client`, pacote `external/signer`) — não implemente PDF/assinatura na mão se essas libs já estiverem no `pom.xml` do projeto.
- Mensageria: RabbitMQ (`spring-boot-starter-amqp`) para eventos assíncronos entre serviços.
- Segurança: OAuth2 (`spring-security-oauth2-resource-server` + `spring-boot-starter-oauth2-client`) — nunca implemente autenticação/validação de token própria, sempre delegue ao Spring Security.
- Integração entre serviços: Spring Cloud OpenFeign (`@FeignClient`) para chamar os demais microsserviços internos da plataforma.
- Observabilidade: métricas via Micrometer/Prometheus, logs estruturados via `loki-logback-appender` — nunca use `System.out.println` para log.
- Auditoria: biblioteca interna `z-audit-lib` — nunca reimplemente log de auditoria na mão.
- Lombok para reduzir boilerplate (getters/setters/builders), seguindo o padrão já usado no código do projeto.

## Arquitetura: package-by-feature, não layer-by-type

Cada domínio de negócio (ex.: `beneficiario`, `prestador`, `procedimento`, `atendimento`) é um pacote próprio, com subpacotes internos por responsabilidade — não agrupe por camada no nível raiz (nunca crie um pacote `controllers` global com todos os controllers do sistema):

```
br.com.<empresa>.<produto>.<dominio>/
├── controller/           # REST
├── (Entidade)GraphQLController.java   # GraphQL, quando aplicável — nunca misture anotacoes REST e GraphQL na mesma classe
├── service/
├── repository/
├── dto/ ou payload/      # Request/Response
├── factory/              # construcao de entidades/DTOs a partir de outras representacoes
├── filter/               # QueryDSL/specification de filtros de listagem
├── exception/            # excecoes de negocio especificas do dominio
├── enumeration/
├── validator/
├── component/            # componentes auxiliares que nao sao service nem repository
└── consumer/             # consumers de fila (RabbitMQ), quando o dominio consome eventos
```

Dois pacotes especiais, sempre presentes:

- **`core`**: cross-cutting concerns compartilhados por todos os domínios — `annotation`, `constant`, `context`, `error`/`error.payload`, `exception.handler` (handler global de exceções), `excel` (geração de planilhas), `util`/`util.converter`.
- **`external`**: clients para os microsserviços internos da plataforma (`zworkspace`, `zstorage`, `zpermission`, `zlogin`, `zlicense`, `zbff`, `zprinter`, `znotifica`/`znotification`) e para APIs externas regulatórias (ex.: ANS). Cada integração tem tipicamente `<Nome>Client` (o `@FeignClient`), `<Nome>Service` (a lógica que usa o client), `<Nome>Config` (configuração do Feign), `<Nome>Payload` (DTOs da integração), e às vezes um `<Nome>ClientFake` para permitir testar sem depender do serviço real.

## Convenções de nomenclatura

- `<Entidade>Controller` para REST; `<Entidade>GraphQLController` para GraphQL, nos produtos que expõem GraphQL — o sufixo `GraphQLController` é obrigatório para diferenciar, nunca reaproveite o nome de um controller REST existente.
- `<Entidade>Request` / `<Entidade>Response` para payloads (não `Dto` genérico quando já existe uma convenção de Request/Response no domínio).
- `<Condição>Exception` para exceções de negócio (ex.: `AlegacaoAnsNuloException`, `VisualizacaoFiltroExistenteException`) — a exceção descreve a condição de erro, não a operação que falhou.
- `<Entidade>Factory` para lógica de construção/transformação de entidades e DTOs.
- `<Entidade>Filter`/`<Entidade>FilterBuilder` para filtros de listagem via QueryDSL.

## Banco de dados

- Toda alteração de schema (nova tabela, coluna, índice) precisa de uma **migração Flyway nova** em `src/main/resources/db/migration` — nunca altere uma migração já aplicada em produção, sempre crie uma nova.
- Prefira QueryDSL a `@Query` com JPQL/SQL nativo escrito à mão para consultas com múltiplos filtros dinâmicos — mais seguro contra erro de digitação e mais fácil de compor.

## Qualidade e build

- O build falha se o Checkstyle reprovar (`maven-checkstyle-plugin` configurado) — rode `mvn checkstyle:check` (ou `mvn test`/`mvn package`, que já disparam o plugin) antes de finalizar, e corrija qualquer violação em vez de suprimir a regra.
- Testes: JUnit 5 + Mockito (via `spring-boot-starter-test`) para unidade; H2 para testes de repositório; `spring-graphql-test` para testar resolvers GraphQL, só nos produtos que tem GraphQL — não confunda os dois tipos de teste.
- Nunca hardcode URL, usuário/senha ou token de outro serviço no código — sempre via `application.yml`/properties e variáveis de ambiente.
