---
name: dev-atd-cmagnet-util-backend
description: "Backend do cmagnet-util, os 31 módulos Java compartilhados do ATD (libs comuns, domínio JDBC de guias/TISS/RDA, clientes SOAP, auth, busca) consumidos por cmagnet, autsc2, proc-guias e outros: catálogo dos módulos, acesso a dados com dynamix-dbutil (DynaSqlBuilder, BaseDAO + DynaDBUtil), DAO que consulta o servidor de regras (WSLDAO + TypeTask), clientes JAX-WS, testes JUnit 4 e o impacto em quem consome. Use para criar, alterar ou testar código nesses módulos. NÃO cobre o módulo cmagnet-liquibase (use dev-atd-liquibase) nem o código dos produtos que consomem estes módulos."
---

# cmagnet-util (módulos compartilhados)

Reator Maven com 31 módulos (`groupId br.com.unimedsc.cmagnet`, nome da pasta = `artifactId`, versão `26.00.0.0-SNAPSHOT` na `HOMOLOG`). Quase todo o ecossistema depende dele: o autsc2 usa 21 módulos, o cmagnet 23, o proc-guias 9. Mudança aqui raramente fica só aqui — veja "Impacto em quem consome" antes de mudar qualquer assinatura pública.

## Catálogo

| Tipo | Módulos |
|---|---|
| Libs comuns | `cmagnet-commons` (formatadores, validadores, conversores, SQL; base de 14 módulos irmãos), `cmagnet-security` (JWT, hash), `cmagnet-crypto` (criptografia versionada dos dados clínicos do PEP), `cmagnet-parameter` (parametrização do sistema), `cmagnet-cache`, `cmagnet-report` (Jasper/iText/PDFBox), `cmagnet-monitor`, `cmagnet-server-info`, `cmagnet-accesscontrol`, `cmagnet-notifier` |
| Domínio (JDBC, DAO/BO) | `cmagnet-rda` (domínio central: ~165 DAOs, DAOs via servidor de regras, RabbitMQ), `cmagnet-tiss` (guias e auditoria), `cmagnet-webin`, `cmagnet-pep`, `cmagnet-price` (precificação), `cmagnet-agenda`, `cmagnet-solic`, `cmagnet-protocolo`, `cmagnet-gestao` (integração Gestão TISS) |
| Clientes SOAP/externos | `cmagnet-tiss-exportacao` (JAXB do padrão ANS TISS v2.02–v4.02, ~3,5 mil classes geradas, e clientes de WS TISS), `cmagnet-integracao` (CFM, Interall, SCCard), `cmagnet-unimedbr-chat`/`-sso` (clientes CXF gerados), `cmagnet-sms` (gateways de SMS) |
| Auth | `cmagnet-auth` (filtros e gerenciadores de token; tipos JAX-RS e Swagger, sem resources `@Path`) |
| Busca | `cmagnet-search-model`/`-indexer`/`-client`, `cmagnet-search-configs` (cores Solr), `cmagnet-search-job` (WAR do indexador Quartz — a única imagem Docker do repositório) |
| Banco | `cmagnet-liquibase` — **não é deste papel**: ver `dev-atd-liquibase` |

Não há servidor SOAP próprio aqui: as classes `*PortTypeImpl` são esqueletos gerados.

## Padrões de código

**Acesso a dados** (`dynamix-dbutil`, Oracle, sem ORM):
- Código novo: `DynaSqlBuilder` — `sql.append("… WHERE CD_CM_PREST = ? ", cdCmPrest)` amarra o parâmetro junto do trecho; depois `sql.executeUpdate(connection)` ou `IResultQuery rs = sql.executeResultQuery(connection)` (modelo: `cmagnet-rda/…/prest/PrestadorEventualDAO.java`).
- Legado: `extends BaseDAO` com `DynaDBUtil.parametrizedPs(con, sql, parametros…)` e `DynaDBUtil.doFinallyClose(ps)` no `finally` (ex.: `cmagnet-tiss/…/guia/solicitacao/dao/AtendimentoSolicitanteGuiaDAO.java`). Ao mexer num DAO legado, siga o estilo dele.
- **Quem chama é dono da `Connection` e da transação**: DAO e BO recebem a conexão por parâmetro e nunca a fecham, nem fazem commit (ex.: `cmagnet-solic/…/SolicBO.java`).
- Parâmetro sempre amarrado (`?`), nunca concatenado no SQL.

**DAO que consulta o servidor de regras** (fdsc-rules): `<X>WSDAOImpl extends WSLDAO<Bean, Query> implements <X>DAO`, singleton `getInstance()`, `getTask()` devolvendo a constante de `TypeTask`, roteamento por `RouteRemote(cdUnimed, cdContrato)`; interfaces em `…rda.dao`, implementações em `…rda.dao.impl`, entregues pelo `FactoryDAO` (lança `RulesServerException`). Modelo: `cmagnet-rda/…/rda/dao/impl/AvisoBeneficiarioWSDAOImpl.java`. Consulta remota nova nasce no fdsc-rules (`TypeTask` + `QMapper` — ver `dev-atd-fdsc-rules-backend`) e só depois é usada aqui.

**Cliente SOAP**: `Service` JAX-WS criado a partir da URL do WSDL, `BindingProvider` com handler chain e timeouts de conexão/leitura vindos da configuração (`cmagnet-integracao/…/cfm/v2/provider/CfmV2ServiceProvider.java`); variante CXF em `cmagnet-unimedbr-sso/…/ServiceFactory.java`. URL e timeouts sempre de configuração, nunca no código.

**Código gerado** (JAXB do TISS no `cmagnet-tiss-exportacao`, clientes CXF, `*PortTypeImpl`): nunca editar à mão — envolva ou estenda.

## Stack e encoding

- **Java 1.6** em todos os módulos (herdado do `fdsc-pom`): nada de lambda, `<>`, try-with-resources, multi-catch, `switch` de String nem APIs do Java 7+ (o CI compila com JDK 8 e não pega API nova).
- Fontes majoritariamente em **ISO-8859-1**, com alguns arquivos UTF-8: preserve o encoding de cada arquivo que editar e procure `�`/`Ã` no diff.

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

- JUnit 4 (`@Test`), alguns ainda no estilo JUnit 3 (`extends TestCase`); sem Mockito e sem teste de banco. Há testes em `commons`, `report`, `tiss-exportacao`, `agenda`, `tiss`, `security` e alguns outros — o CI roda todos.
- Um módulo com as dependências internas: `mvn -q -pl cmagnet-<modulo> -am test`. Tudo: `mvn -q test`.
- Para outro repositório enxergar a mudança localmente: `mvn -q -pl cmagnet-<modulo> -am install` (o CI de `feature/`/`bugfix/` não publica).
- Teste novo: JUnit 4, sem banco, no `src/test/java` do módulo, testando a regra (conversão, validação, montagem de SQL) e não a infraestrutura.

## Impacto em quem consome

| Módulo | Repositórios que consomem |
|---|---|
| `cmagnet-commons` | 8 |
| `cmagnet-parameter`, `cmagnet-rda` | 6 |
| `cmagnet-report` | 4 |
| `cmagnet-tiss`, `cmagnet-gestao`, `cmagnet-agenda`, `cmagnet-security`, `cmagnet-protocolo` | 3 |
| `cmagnet-webin`, `cmagnet-accesscontrol` | 2 |

Prefira mudança aditiva (método ou sobrecarga nova) a mudar assinatura existente. Se a assinatura precisar mudar, o plano lista cada repositório consumidor e a ordem (cmagnet-util primeiro, `install` local, MR na `HOMOLOG` antes) — mapa em `dev-atd-ecossistema`.

## Regras

| ID | Nível | Regra |
|---|---|---|
| ATD-CMU-001 | MUST | SQL com parâmetro amarrado; nunca concatenar valor de entrada |
| ATD-CMU-002 | MUST | DAO/BO nunca fecha nem comita a `Connection` recebida |
| ATD-CMU-003 | MUST | Java 1.6 de sintaxe e de API |
| ATD-CMU-004 | MUST | Código gerado (JAXB TISS, clientes CXF, `*PortTypeImpl`) não se edita à mão |
| ATD-CMU-005 | SHOULD | Código novo de acesso a dados com `DynaSqlBuilder`; legado mantém o estilo do arquivo |
| ATD-CMU-006 | SHOULD | Mudança de API pública aditiva; mudança de assinatura vira item multirrepositório no plano |

## NUNCA

- Nunca rode os `main()` de teste manual (`cmagnet-agenda/…/test/GenerateAgendamentos`, `GenerateHorariosPrestadores`, `cmagnet-report/…/ReportMainTest`, `cmagnet-tiss-exportacao/…/WSTest*`, `cmagnet-webin/…/MainWebin`): chamam serviços e bancos reais.
- Nunca copie para código, plano ou relatório as credenciais que existem em arquivos do repositório (há conexão de banco em código de teste e token em configuração de análise).
- Nunca troque versão de POM à mão.
