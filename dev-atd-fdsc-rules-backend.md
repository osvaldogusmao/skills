---
name: dev-atd-fdsc-rules-backend
description: "Backend do fdsc-rules, o servidor de regras do ATD (Drools 5 via dynamix-rules): expor dados às regras — atributo novo num fato, fato de sessão ou global novo, campo novo na mensagem de regra, consulta remota nova — com MyBatis, classes JAXB e os registros que precisam ficar em sincronia (TypeTask, ruletask/querytask.properties, sql-map-config.xml, XSD). Use para qualquer mudança Java/XML nos módulos fdsc-rules-util, -model, -mapper, -core e dynamix-rules-server, ou para entender como cmagnet-util, proc-guias, autsc2 e cmagnet chamam o servidor de regras. As regras (DRL) ficam no Guvnor, fora do git. NÃO cobre o Liquibase SAPIA-NI (use dev-atd-liquibase)."
---

# fdsc-rules (servidor de regras do ATD)

O `dynamix-rules-server` (WAR, um por Unimed — cada "ponta" roda o seu) recebe pedidos SOAP (`executeRules(router, xml)` e `executeQuery(router, query)`, endpoint `/dynamix-rules-server/server`, classe `DynamixRulesEntry` do framework `dynamix-rules`), monta os **fatos** a partir do pedido e do banco e roda as regras. Os clientes chamam por `ExecuteRemote.execRule/execQuery` (`dynamix-rules-client`) usando os nomes de `TypeTask`.

## As regras ficam no Guvnor

Os change-sets em `dynamix-rules-server/Resources/properties/*.xml` apontam para pacotes `LATEST.drl` num Guvnor. Não há `.drl` versionado no repositório (os do `config-atd` são stubs para ambiente local). Então, aqui:

- o trabalho é **expor dados às regras**: fatos, atributos, consultas;
- criar ou alterar a regra é **pendência para o dev**, feita no Guvnor — descreva no resumo qual pacote, qual fato e quais atributos a regra vai usar;
- o jar `fdsc-rules-model` é o modelo que os autores de regra usam no Guvnor: atributo novo precisa de getter público.

## Módulos (ordem de build: util → model → mapper → core → server)

| Módulo | Conteúdo |
|---|---|
| `fdsc-rules-util` (Java 1.8) | `br.com.unimedsc.access.TypeTask` (nomes das tasks), classes JAXB de `datamodel.rule`/`datamodel.query` (`Rule`, `Query`, `QueryBody`, `ObjectFactory`, `*Parameter` — versionadas e editadas à mão, sem `xjc`), `rules.actions`, `rules.configs`. **É o único módulo consumido por outros repositórios** |
| `fdsc-rules-model` (Java 1.6) | fatos, todos `extends SapiaFact`: `remote` (vêm no pedido), `session` (carregados do banco por pedido), `global` (pools em cache), `result`, `function`, `internal` |
| `fdsc-rules-mapper` (Java 1.6) | MyBatis 3.0.1: interfaces `query/*QMapper`, `rule/perform/*RMapper`, `global/*GMapper`; XML em `src/main/resources/datamodel/{query,rule,global,price,config}/*Map.xml`; `datamodel/sql-map-config.xml`; XSDs `RuleComplexType.xsd` e `QueryComplexType.xsd`; datasource JNDI `java:comp/env/jdbc/dynamix-rules-server` |
| `fdsc-rules-core` (Java 1.6) | `server.task.rule.impl` (um `*RuleTask` por pacote de regras, `RuleTaskUtil`, `SapiaGuiaTask`), `task.result.impl`, `task.query.impl.RemoteQueryTask`, `server.global.impl` (loader + truth maintainer de cada pool global) |
| `dynamix-rules-server` (WAR) | descritores e propriedades: `src/main/resources/specificationproperties/{ruletask,querytask,ruleproperties}.properties`, `Resources/properties` (change-sets do Guvnor, implantados fora do WAR via JNDI) e o Liquibase SAPIA-NI |
| `fdsc-rules-service` (EAR) | legado; o CI não o usa |

Classe para entender o fluxo de uma task: `fdsc-rules-core/.../server/task/rule/impl/AutorizacaoRuleTask.java`.

## Receitas (tiradas do histórico do repositório)

**1. Atributo novo num fato existente** (ex.: commit `097cc501`):
- no fato (`fdsc-rules-model`): campo privado, valor padrão no construtor (`DefaultValues.getInteger()`, `getString()`…), getter público e setter com `bind(valor, Tipo.class)`;
- em cada `*Map.xml` que carrega o fato (o `global/<Fato>GMap.xml` e/ou o `rule/<Fato>RMap.xml`): `<result property="<atributo>" column="<COLUNA>"/>` e a coluna no `SELECT`.

**2. Fato de sessão novo** (ex.: `e3c8da91`):
- classe em `fdsc-rules-model/.../rules/model/session/<Fato>.java`;
- `fdsc-rules-mapper/.../datamodel/mapper/rule/perform/<Fato>RMapper.java`: `@FactAnnotation(typeFact = TipoFatoEnum.SESSION, name = "<Fato>")` numa interface que `extends IRuleTaskMapper`;
- `datamodel/rule/<Fato>RMap.xml` com o `select`;
- no `sql-map-config.xml`: `<typeAlias alias="<Fato>" type="…session.<Fato>"/>` e `<mapper resource="datamodel/rule/<Fato>RMap.xml" />`.
O framework registra o fato sozinho pela anotação (varredura do classpath).

**Fato global novo** (ex.: `97c4cbc4`, que criou a versão global e a de sessão do mesmo fato): o fato em `fdsc-rules-model/.../rules/model/global/`; `<Fato>GMapper extends IGlobalMapper` com `@FactAnnotation(typeFact = TipoFatoEnum.GLOBAL, name = "<Fato>")` e `@GlobalFactAnnotation(loader = "…<Fato>GlobalFactLoader", maintainer = "…<Fato>GlobalFactTruthMaintainer")`; `<Fato>GlobalFactLoader extends AbstractDBGlobalFactLoader` (devolve a classe do `GMapper`) e o truth maintainer em `fdsc-rules-core/.../server/global/impl/`; `global/<Fato>GMap.xml`; `sql-map-config.xml`. Não precisa mexer no `ruletask.properties`: desde 2017 (FDSC-710), todo fato global registrado pela anotação já entra no pool de todas as tasks (`FactoryRuleTaskSpecification`, no `dynamix-rules`).

**3. Campo novo na mensagem de regra** (ex.: `6b151274`, `442514a9`):
- elemento opcional (`minOccurs="0"`) no `RuleComplexType.xsd`;
- campo, getter e setter na classe JAXB `<…>Parameter.java` (`fdsc-rules-util`), batendo com o XSD;
- atributo no fato `model/remote` correspondente;
- atribuição sem estourar nulo no `RuleTaskUtil`, no `SapiaGuiaTask` ou no `*RuleTask` que monta o fato a partir do pedido.
Quem envia o pedido (proc-guias, autsc2, cmagnet-util…) precisa preencher o campo — é mudança em outro repositório.

**4. Consulta remota nova** (ex.: `3d107184`, 9 arquivos):
- `QueryComplexType.xsd`; `<X>Parameter.java`, `QueryBody` e `ObjectFactory` (JAXB, `fdsc-rules-util`);
- constante nova no fim de `TypeTask` (`NOMEX("NomeX")`);
- `fdsc-rules-mapper/.../mapper/query/<X>QMapper.java` (`extends IQueryTaskMapper`) e `datamodel/query/<X>QMap.xml`;
- `sql-map-config.xml`; e `NomeX=…query.<X>QMapper` no `querytask.properties`.

## Sincronias (confira antes de entregar)

- Todo `TASKID` de `ruletask.properties` (`Task=<classe>[;<pools>]`) e de `querytask.properties` (`Task=<QMapper>`) tem constante em `TypeTask` com o mesmo nome. O contrário não vale: o `TypeTask` também tem nomes de alguns fatos globais (ex.: `ItensUnidadeMedida`) e constantes sem uso no repositório — não remova nenhuma (ATD-FRL-001).
- `sql-map-config.xml` lista todo `*Map.xml` e o `typeAlias` de cada fato.
- XSD e classes JAXB descrevem o mesmo campo, com o mesmo nome e tipo.
- Versões (os 7 POMs e o `versionSystem` do `web.xml`) só mudam pelo script de versionamento do time — nunca à mão.

## Stack e encoding

- `fdsc-rules-core`, `-mapper` e `-model` compilam em **Java 1.6**: nada de lambda, `<>`, try-with-resources, multi-catch, `switch` de String nem APIs do Java 7+ (o CI compila com JDK 8, então API nova passaria no build e quebraria a regra do módulo). `fdsc-rules-util` é 1.8.
- Fontes em **ISO-8859-1** (63 arquivos com acento). O `AutorizacaoRuleTask.java` já tem um `�` de uma edição antiga em UTF-8 — não "corrija" de passagem e não propague: preserve o encoding do arquivo que editar e procure `�`/`Ã` no diff.

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

- Testes JUnit 4 em `fdsc-rules-util` (3) e `fdsc-rules-model` (1), mais o `LiquibaseValidationTest` (JUnit 5) do server. Nenhum toca banco nem rede.
- Um módulo com o que ele depende: `mvn -q -pl <modulo> -am test`. Tudo: `mvn -q test`. O WAR: `mvn -q -pl dynamix-rules-server -am package`.
- Para outro repositório enxergar a mudança localmente: `mvn -q -pl fdsc-rules-util -am install` (o CI de `feature/`/`bugfix/` não publica).

## Regras

| ID | Nível | Regra |
|---|---|---|
| ATD-FRL-001 | MUST | Nunca renomear nem remover constante de `TypeTask`, campo de XSD/JAXB ou atributo de fato que outro repositório ou regra do Guvnor usa — acrescente; remoção é decisão do time |
| ATD-FRL-002 | MUST | Mapper novo sempre registrado no `sql-map-config.xml`; task nova sempre em `TypeTask` + `*.properties` |
| ATD-FRL-003 | MUST | Java 1.6 nos módulos core/mapper/model |
| ATD-FRL-004 | MUST | Getter público em todo atributo de fato (o Guvnor usa) |
| ATD-FRL-005 | SHOULD | Campo novo no pedido sempre opcional, para clientes antigos continuarem válidos |

## Fora deste repositório

- Mudança em `fdsc-rules-util` (`TypeTask`, classes JAXB) é API de cmagnet-util (`rda`, `price`), proc-guias (10 módulos), autsc2 e `sgucard-kernel`: no plano, fdsc-rules primeiro (`install` local, MR na `HOMOLOG` antes) e depois cada consumidor que precisar enviar o campo ou chamar a consulta.
- Regra nova ou alterada → pendência no Guvnor.
- Tabela ou coluna nova que o fato lê → changeset (SAPIA-NI no fdsc-rules e/ou central no cmagnet-util): ver `dev-atd-liquibase`.
- Mapa completo: `dev-atd-ecossistema`.

## NUNCA

- Nunca crie `.drl` no repositório como se fosse a regra, nem aponte os change-sets de `Resources/properties` para outro lugar.
- Nunca chame o Guvnor nem o endpoint `/reloadserver` (recarrega pools num servidor rodando).
- Nunca troque versão de POM ou do `web.xml` à mão.
- Nunca copie credenciais de arquivos de configuração para código, plano ou relatório.
