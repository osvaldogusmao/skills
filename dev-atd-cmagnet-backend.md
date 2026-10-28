---
name: dev-atd-cmagnet-backend
description: "Backend do CMAGNET (SGU-Card): a API REST nova no módulo sgucard/sgucard-api (Java 1.8, UTF-8, JAX-RS/RESTEasy com registro por @AutoService, envelope {data, error}, paginação 1-based, permissões @RequerPermissao/@RequerPermissaoMenuPai, Swagger SGU-Card, Javadoc, DAO com SqlBuilder e CRUD simples pelo atd-persistence, JUnit 5 + JaCoCo de 90%) e o Java legado de src/br (sgucard-kernel, Java 1.6, ISO-8859-1). Use para criar ou alterar endpoint, service, DAO, DTO, Swagger, Javadoc ou teste no cmagnet, ou Java legado fora das páginas CodeCharge. NÃO cobre o frontend Vue (dev-atd-cmagnet-frontend), as páginas CodeCharge (dev-atd-codecharge), banco e menu (dev-atd-liquibase, no cmagnet-util) nem o AUTSC2."
---

# CMAGNET — backend

O WAR do cmagnet junta três camadas: o legado CodeCharge (`ccs/` e `ccs/CCSBuild/`), a API REST nova (`sgucard/sgucard-api`) e a SPA Vue (`sgucard/sgucard-frontend`). Esta skill cobre a API e o Java legado; as páginas CodeCharge estão na `dev-atd-codecharge`.

## Leia antes

Os docs do repositório são a fonte de verdade; esta skill é o resumo. Antes de planejar ou implementar, leia `docs/migracao-vue/padroes-vigentes.md`. Ele define a ordem de autoridade: ele mesmo, depois `docs/arquitetura-front-back-sgucard.md`, `docs/padrao-nomenclatura-codigo.md`, `docs/migracao-vue/padrao-migracao-tela-legado.md` e, por último, as páginas de telas já migradas em `docs/migracao-vue/pages/**`, que são referência funcional e não normativa. Depois leia só o doc do tema:

| Tarefa | Doc |
|---|---|
| Registro, paths, segurança, `web.xml` | `docs/configuracao-resteasy-sgucard.md` |
| Pacotes da feature, envelope, paginação | `docs/arquitetura-front-back-sgucard.md` |
| Swagger | `docs/migracao-vue/swagger-sgucard-api.md` |
| Testes e JaCoCo | `docs/testes-rest-backend-sgucard-api.md` |
| Acesso a dados | `docs/migracao-vue/padrao-atd-persistence-sgucard-api.md` e a skill `dev-atd-persistence` |
| Javadoc | `docs/migracao-vue/padrao-javadoc-backend.md` |
| Nomes de classe, método e teste | `docs/padrao-nomenclatura-codigo.md` |
| Upload `multipart/form-data` | `docs/migracao-vue/upload-multipart-sgucard-api.md` |
| Filtros dinâmicos (`NamedFilter`), `tieBreaker` | `docs/migracao-vue/padrao-migracao-tela-legado.md`, seções 3 e 5.1 |

- Se doc e código divergirem, siga o doc vigente e registre a divergência no resumo. Padrão novo só vale com o doc atualizado na mesma entrega (`padroes-vigentes.md`, "Como promover um novo padrão"): se a tarefa criar um, isso entra no plano.
- **Divergência conhecida**: os docs ainda mandam mapear entidades do atd-persistence com `@SqlTable`/`@SqlColumn`/`@SqlId`. Essas anotações não existem mais: a lib usa `@Entity`/`@Table`/`@Column`/`@Id` com `@SqlIdOrder`. Siga a `dev-atd-persistence` e o README da lib.

## Onde fica cada coisa

| Área | Caminho | Java | Encoding |
|---|---|---|---|
| API REST nova | `sgucard/sgucard-api/src/main/java/br/com/sgucard/api/<modulo>/` | 1.8 | UTF-8 |
| Configuração JAX-RS | `sgucard-api/.../br/com/sgucard/config/` (`RestApplication`, `RestResources`, `ResourceRegistry`) | 1.8 | UTF-8 |
| Java legado: framework (`br.com.dynamix.framework.*`), servlets, classes chamadas pelas telas | `src/br/` | 1.6 | ISO-8859-1 |
| Java gerado pelo CodeCharge | `ccs/CCSBuild/src/<área>/` — ver `dev-atd-codecharge` | 1.6 | ISO-8859-1 |
| API do homecare-webapp (`/servlet/capturaExterna/v1/*`) | `src/br/com/dynamix/framework/cmagnet/ajax/integracao/captura/` — ver "API do homecare" | 1.6 | ISO-8859-1 |
| Descritor do WAR | `ccs/WEB-INF/web.xml` (template do CodeCharge: `ccs/CCSBuild/config/web.xml`) | — | — |

**O legado é compilado por links.** O reator `sgucard/` tem 17 módulos, e 15 deles recebem o legado por links simbólicos: `src/br` entra no `sgucard-kernel`, e cada área de `ccs/CCSBuild/src` entra num módulo `sgucard-*` (ex.: `exames` e `internacoes` no `sgucard-guias`). Os links são criados por `Resources/deploy/cm-links.bat` (Windows) ou `linux-mklinks.sh`.

Sem os links, o módulo do legado compila vazio e **sem erro**. Quem depende dele falha, ou pega a versão do repositório Maven. Antes de confiar num build que envolve legado, confira que `sgucard/sgucard-kernel/CCSBuild/src/br` existe. Se não existir, pare e registre a pendência. Não rode os scripts: eles apagam e recriam links, usam caminho fixo e exigem administrador ou Modo de Desenvolvedor no Windows.

## Receita: endpoint novo

Modelos: `admin/usuarioslogados` (feature simples) e `admin/cadastros/locaisatendimento` (feature composta).

1. **Path:** constante no `<Modulo>ApiPath` (`ROOT = ApiVersion.V1 + "/<modulo>"`, mais o path da feature), usada no `@Path`.
2. **Pacote da feature `br.com.sgucard.api.<modulo>.<feature>`:**
   - `Resource` e `Service` ficam na raiz da feature;
   - `DAO`, `DTO` e `Filter` ficam na raiz quando há só um de cada tipo; com mais de um, vão para `dao/`, `dto/` e `filter/`;
   - `docs/<Feature>ResponseDoc` (Swagger) existe sempre;
   - ponte com o legado (CodeCharge, `CarimboHash`, `WorkSql`, auditoria, popups `.do`) vai num `<Feature>LegacySupport` com implementação `Default<Feature>LegacySupport`;
   - feature com subfeatures fica com `core/` mais as subfeatures no primeiro nível. Nunca crie `core/` em feature simples.
3. **Registro:** inclua o resource no `<Modulo>Resources`, anotado com `@AutoService(ResourceRegistry.class)`, e atualize o `ResourceRegistryTest`. Módulo novo ganha um `<Modulo>Resources` próprio. Nunca escreva `META-INF/services` à mão e nunca registre na `RestApplication`.
4. **Swagger:**
   - tag nova em `SwaggerTags`, usada em `@Api(value, tags)`;
   - `@ApiOperation(value, notes, response)` em cada método;
   - `@ApiResponses` com pelo menos 200, 401 e 500, usando `ApiErrorResponseDoc` nos erros;
   - `@ApiParam` nos parâmetros, com `hidden = true` nos internos (`@Context`);
   - `@ApiModel`/`@ApiModelProperty` nos DTOs.
5. **Testes:** no mesmo pacote, em `sgucard-api/src/test/java` (ver "Testes e validação").

## Contrato REST

- **Path e resposta:** URL em `/cmagnet/rest/api/v1/...`. O retorno passa por `ResponseUtil` (`ok`, `badRequest`, `notFound`), que monta o envelope `{data, error}` com `application/json;charset=UTF-8` e `Cache-Control: no-store`. SQL e regra ficam fora do `Resource`.
- **Listagem:**
  - `page` começa em 1 (padrão 1); `pageSize` tem padrão 20 e máximo 200; use `PageableFactory`;
  - devolva `PageResponse` com `items`, `total`, `page`, `pageSize` e `totalPages`;
  - `sortBy`/`sortDir` só para campos com `@SortColumn`;
  - se a coluna ordenada pode repetir valores, use `PaginationQuery.tieBreaker(<PK>)`.
- **Permissão:**
  - `@RequerPermissao(recurso, funcao = FuncaoOperacao.ACCESS)` vai na classe; `pagina` é usado em item de catálogo administrativo;
  - escrita vai em métodos separados, cada um com uma única função (`INSERT`, `UPDATE` ou `DELETE`);
  - tela compartilhada por vários menus usa `@RequerPermissaoMenuPai` com `paginasPai`; `permiteSemMenu`/`recursoSemMenu` só quando a tela também abre fora de menu;
  - endpoint público só com `@Publico`;
  - **nunca invente chave de permissão**: sem evidência no menu ou no legado, registre a pendência.
- **URL `.do` devolvida pela API:** sai carimbada, com `CarimboHash.doCarimbo(url, request)` no `Service` ou no `LegacySupport`, e há teste conferindo `dynaHash=`.
- **Outro módulo SGU-Card:** chame por dependência Maven e chamada Java direta, nunca por reflection (`Class.forName`, `Method.invoke`).

## API do homecare

O homecare-webapp (SPA à parte, skill `dev-atd-homecare-frontend`) não usa o `sgucard-api`: chama servlets legados do cmagnet, com autenticação JWT própria.
- **Código:** `src/br/com/dynamix/framework/cmagnet/ajax/integracao/captura/`, com um servlet `AJAX<Nome>` por endpoint, beans de entrada em `request/`, saídas em `dto/` e a regra em `service/<área>/`. Os serviços variam por tipo de guia (consulta, exame, internação) e são criados por factories; a cadeia de regras de execução e finalização fica em `service/executaefinaliza/regrasnegocio/`.
- **Mapeamento e filtros:** os endpoints são mapeados nos dois `web.xml` (servlet e `url-pattern` em `/servlet/capturaExterna/v1/<nome>`). Passam pelos filtros `CapturaExternaCorsFilter` e `HomecareJwtFilter`; o token é emitido pelo `TokenJWTCapturaExternaService`.
- **Receita de endpoint novo** (SAT-I1906, `db98769191`): servlet, `request/`, `dto/`, interface e implementações em `service/<área>/`, e os dois `web.xml`. Mudar o contrato de um endpoint existente quebra o homecare. O plano lista a mudança no homecare-webapp, no mesmo SAT-I, com o cmagnet antes.
- **Linguagem:** é Java legado (Java 1.6 e ISO-8859-1), compilado no `sgucard-kernel`.

## Acesso a dados

- **Consulta de tela** (join, view, função, agregação, DTO de tela, paginação): DAO de feature com `br.com.dynamix.framework.cmagnet.sql.SqlBuilder` ou `PaginationQuery`, com parâmetro sempre amarrado. Nada de `connection.prepareStatement(...)` em código novo.
- **`insert`, `update`, `delete` ou `select` simples de uma tabela só:** use **obrigatoriamente** a DAO de entidade do atd-persistence.
  - A DAO de feature recebe a DAO de entidade no construtor público, por `SqlUtil.getPersistence().<metodoDaFacade>()`, e tem um construtor injetável para teste.
  - No CMAGNET, o método dedicado na facade `AtdPersistence` é obrigatório; `persistence.dao(X::new)` não vale como estado final.
  - Se a entidade, a DAO ou o método não existirem, a mudança começa no repositório `atd-persistence` (ver "Fora deste repositório").
  - Uso da lib: `.claude/skills/dev-atd-persistence/SKILL.md`. Se essa skill não estiver instalada, leia o README do atd-persistence e registre a pendência.
- **Transação composta:** a `Connection` é da feature (`WorkSql.getConnection()`, `setAutoCommit(false)`, commit/rollback/close) e vai para os overloads da DAO de entidade que recebem `Connection`. Exceções da lib viram `ApiBadRequestException` ou `ApiDatabaseException`.
- DAOs antigos com `PreparedStatement` (`BeneficiarioExcecaoDAO`, `ExcecaoBiometriaFacialDAO`, `CabecalhoRodapeLocalDAO`) não são modelo; a lista está em `padrao-atd-persistence-sgucard-api.md`.

## Texto, encoding e Javadoc

- **`sgucard-api` (UTF-8):**
  - texto humano leva acento correto: mensagens REST e de `ApiException`, Swagger, validações e asserts que reproduzem texto de tela;
  - ficam sem acento os identificadores, paths, chaves, aliases SQL, comentários Java e logs técnicos.
- **Javadoc:** em português sem acento nos `.java`, exceto texto literal de tela. Vai em `Service`, `DAO`, `Filter`, `LegacySupport` e helpers. `Resource`, DTO exposto e `docs/*Doc` já são documentados pelo Swagger; não repita em Javadoc.
- **Legado `src/br`:** ISO-8859-1 e Java 1.6 de sintaxe e de API. O CI compila com JDK 8, então API nova passa no build e quebra a regra. Preserve o encoding do arquivo editado.
- **Varredura:** antes de concluir, procure `Ã|Â|�|\\u[0-9a-fA-F]{4}|&[A-Za-z]+;` nos arquivos UTF-8 alterados (o `rg` dos docs ou a busca equivalente). Nos ISO-8859-1, confira que o diff não trouxe `Ã` nem `�` novos.

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

## Testes e validação

- **Stack de teste:** JUnit 5.10.2 e Mockito 3.12.4, sem container nem banco.
  - Resource: instancie direto, com service fake.
  - Service: teste com fake do `LegacySupport`.
  - Métodos `final` da DAO de entidade não são mockáveis: injete uma função pequena, com method reference em produção e lambda no teste.
- **Mínimo esperado:**
  - paginação: `page` nulo ou menor que 1, `pageSize` padrão e teto de 200, resposta com `page` começando em 1;
  - sort permitido e sort recusado, e `tieBreaker` quando usado;
  - classes `docs` do Swagger;
  - registro do resource (`ResourceRegistryTest`);
  - `dynaHash=` nas URLs `.do`.
- **JaCoCo:**
  - meta de linha (`LINE/COVEREDRATIO`) em 0.90, lida do `pom.xml` do `sgucard-api`;
  - fora da conta: `*Doc*`, `*ApiPath*`, `docs/*`, `ApiVersion*`, `*DAO*`, `*DTO*` e `*LegacySupport*`. Não mova regra para essas classes para fugir da meta;
  - abaixo da meta, a entrega não terminou: escreva os testes que faltam.
- **Comandos, da raiz do repositório:**
  - `mvn -q -f sgucard/pom.xml -pl sgucard-api verify`: testes e JaCoCo, obrigatório com backend REST novo ou alterado;
  - um teste: `mvn -q -f sgucard/pom.xml -pl sgucard-api -Dtest=<Classe>Test test`;
  - se mudou `src/br` ou outro módulo `sgucard-*` que a API usa, acrescente `-am`. Sem ele, o `sgucard-kernel` vem do repositório Maven (o instalado localmente ou o SNAPSHOT que o CI publica a partir de qualquer branch), e não do seu código;
  - só legado: `mvn -q -f sgucard/pom.xml -pl <modulo> -am -DskipTests compile` (ex.: `sgucard-kernel` para `src/br`).
- **Fica como pendência:** conferir o endpoint no Swagger UI (`/cmagnet/api-docs/`, só localhost) e testar com o servidor de pé. Você não sobe o WildFly.

## Regras

| ID | Nível | Regra | Como verificar |
|---|---|---|---|
| ATD-CMB-001 | MUST | Resource registrado pelo `<Modulo>Resources` com `@AutoService(ResourceRegistry.class)` e `ResourceRegistryTest` atualizado; sem `META-INF/services` à mão nem registro na `RestApplication` | diff e teste |
| ATD-CMB-002 | MUST | Envelope `{data, error}` por `ResponseUtil`; listagem com `page` começando em 1, `PageableFactory`, `pageSize` 20/200, sort só por `@SortColumn` | testes de paginação |
| ATD-CMB-003 | MUST | `ACCESS` na classe e uma função por método de escrita; chave de permissão só com evidência | diff |
| ATD-CMB-004 | MUST | DML e select simples pela DAO de entidade, com método dedicado em `AtdPersistence` obtido por `SqlUtil.getPersistence()`; sem `prepareStatement` em código novo | diff |
| ATD-CMB-005 | MUST | Swagger completo: tag em `SwaggerTags`, `response` em `@ApiOperation`, 200/401/500 com `ApiErrorResponseDoc`, `@ApiParam` | checklist do doc de Swagger |
| ATD-CMB-006 | MUST | `verify` verde com o JaCoCo igual ou acima da meta do POM | comando |
| ATD-CMB-007 | MUST | URL `.do` devolvida pela API sai carimbada, com teste de `dynaHash=` | teste |
| ATD-CMB-008 | MUST | `sgucard-api` em UTF-8 com texto humano acentuado; `src/br` em ISO-8859-1 e Java 1.6 | varredura |
| ATD-CMB-009 | SHOULD | Ponte com o legado em `<Feature>LegacySupport`, com a regra testada no `Service` por fake | testes |
| ATD-CMB-010 | SHOULD | Subpacote técnico só com mais de uma classe do tipo; `core/` só em feature composta | diff |

## Fora deste repositório

- **Banco e menu:** tabela, coluna, parâmetro, menu (`CM_MENUS`, `CM_ITENS_ADMIN`) e permissão de menu são changeset no central do `cmagnet-util` (skill `dev-atd-liquibase`). No plano, o banco vem antes do app. Uma migração de tela só fecha quando o menu aponta para a rota Vue.
- **Entidade, DAO ou método na facade:** começam no `atd-persistence`. Use o mesmo chamado e a mesma branch, rode o `mvn -q install` local da lib antes de compilar o cmagnet, e o MR da lib entra na `HOMOLOG` antes.
- **Módulo do `cmagnet-util`** (ex.: `cmagnet-rda`, `cmagnet-commons`): muda primeiro lá. **Consulta ao servidor de regras:** começa no fdsc-rules (`dev-atd-fdsc-rules-backend`).
- **Mapa completo:** `dev-atd-ecossistema`.

## NUNCA

- Nunca aplique Liquibase nem rode `mvn liquibase:*`. O `Resources/database/liquibase` e o `sgucard-liquibase` deste repositório não são o lugar de changeset novo; o lugar é o central do cmagnet-util.
- Nunca suba o servidor, nem rode `Resources/deploy/cm-links.bat` ou `linux-mklinks.sh`.
- Nunca copie para código, plano ou relatório o conteúdo de `Resources/deploy/oracle-cmagnet-ds.xml`, `Resources/deploy/site.properties` ou `docker-gitlab-login.bat`: têm credenciais.
- Nunca troque versão de POM à mão.
