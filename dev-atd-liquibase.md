---
name: dev-atd-liquibase
description: "Changesets Liquibase dos repositórios do ATD (sgu-atd) nos dois modelos vivos: o central do cmagnet-liquibase (cmagnet-util, pastas DEV-AA-MM-0-0, usado pelo CMAGNET e pelo AUTSC2) e o SAPIA-NI do fdsc-rules (pastas SAPIA-NI-AA-MM-0-0). Use para criar ou revisar changeset de tabela, coluna, constraint, índice, sequence/trigger, parâmetro CM_PARAM, menu (CM_MENUS/AUTSC2_MENUS) e permissões, procedure/function/view canônicas, pasta de mês nova, espelho de ALTER TABLE no SAPIA-NI e para entender falhas do LiquibaseChanceLogTest, LiquibaseJediRDATest ou LiquibaseValidationTest. NÃO aplica nada em banco e NÃO serve para Liquibase de outros produtos NEXDOM nem para os modelos parados do capa/servicos-online-benef."
---

# Liquibase do ATD (central + SAPIA-NI)

Dois modelos vivos, em dois repositórios. O changelog **nunca é aplicado por você**: ele roda no startup do WAR de cada ambiente (listeners no `web.xml` do cmagnet, do autsc2 e do fdsc-rules: `DynamixChangesRegister` e o listener Liquibase de cada app). Seu trabalho termina no XML revisado e nos testes offline passando.

O capa e o servicos-online-benef tinham changelogs próprios (`CAPA-*`, `SOB-*`), mas o listener saiu do `web.xml` dos dois em 2026-09-22 (`c9117a0` e `9611dfd`), para evitar concorrência nas migrations do banco compartilhado. As pastas continuam nos repositórios e não rodam mais: banco desses produtos vai no central.

| | Central | SAPIA-NI |
|---|---|---|
| Onde | `cmagnet-util/cmagnet-liquibase/src/main/resources/liquibase/central/` | `fdsc-rules/dynamix-rules-server/Resources/database/liquibase/` |
| Quem usa | cmagnet **e autsc2** (o autsc2 não tem pastas próprias e roda o central) | servidor de regras (owner `UD976`, com réplicas de tabelas `CM_*`/`AUTSC2_*`) |
| Pasta do mês | `DEV-AA-MM-0-0` | `SAPIA-NI-AA-MM-0-0` (só existe nos meses que precisaram) |
| Master | `db-changelog-central.xml`: pastas antigas comentadas; ativas as `DEV-26-*`, depois `function/`, `procedure/`, `view/` | `db-changelog-master.xml`: um include por pasta, em ordem |
| Changelog da pasta | pastas recentes: só `<includeAll path="changes" relativeToChangelogFile="true"/>` e os arquivos em `changes/`; pastas até `DEV-26-08`: includes explícitos | sempre includes explícitos, na ordem numérica |
| Start-version | `id="1_start_version_AA.MM.0.0"` | `id="1.1_start_version_AA.MM.0.0"` |
| Engine | fork interno 4.0.3 | fork interno 2.1.3 |

## Mapa de fontes (leia só o que a tarefa pedir)

| Tarefa | Leia |
|---|---|
| Qualquer changeset novo | os 2–3 arquivos mais recentes da pasta do mês, do mesmo tipo (`alter-table`, `create-param`, `Create-Menu`…) — copie o estilo deles |
| Falha do `LiquibaseJediRDATest` / espelho no SAPIA-NI | `cmagnet-util/cmagnet-liquibase/doc/liquibase-jedi-rda-test.md` (passo a passo e modelos ADD COLUMN, ADD CONSTRAINT, MODIFY NULL) |
| Falha do `LiquibaseChanceLogTest` | `cmagnet-util/cmagnet-liquibase/doc/liquibase-changelog-test.md` |
| Menu de tela Vue do AUTSC2 | `autsc2/docs/integracao-menu-spa.md` (seção "Migration Liquibase") |
| Tabelas monitoradas pelo espelho | `MONITORED_TABLES` em `cmagnet-liquibase/src/test/java/br/com/zitrus/liquibase/LiquibaseJediRDATest.java` (179 tabelas `CM_*`, `AUTSC2_*`, `SCE_*`, `TISS_*`, `PTU_*` e algumas outras) |

Se o doc ou o código do repositório contradisser esta skill, o repositório vence — siga-o e reporte a divergência.

## Regras

| ID | Nível | Regra | Como verificar |
|---|---|---|---|
| ATD-LQB-001 | MUST | Todo changeSet com `failOnError="false"` e `preConditions onFail="MARK_RAN"` que confere o estado atual (tabela/coluna/constraint/linha existe ou não) | a precondição falha se o objeto já estiver no estado final |
| ATD-LQB-002 | MUST | Nunca alterar um changeSet já aplicado em algum ambiente; correção = changeSet novo | `git log` do arquivo; na dúvida, pergunte |
| ATD-LQB-003 | MUST | Nomes Oracle em caixa alta, `_`, sem aspas, sem acento, até **30 bytes** (tabela, coluna, constraint, índice, sequence, trigger, procedure, view) | contar caracteres; ASCII = 1 byte |
| ATD-LQB-004 | MUST | Coluna nova com `COMMENT ON COLUMN`; tabela nova com `COMMENT ON TABLE` e comentário em todas as colunas — explicando o significado de negócio, não repetindo o nome | revisar o arquivo |
| ATD-LQB-005 | MUST | Arquivo novo `NNN-SAT-I<numero>-<descricao>.xml`; `NNN` = próximo número livre na pasta | listar a pasta |
| ATD-LQB-006 | MUST | Central, pasta com `includeAll`: arquivo em `changes/`, só `.xml`. SQL longo vai num `sql/` irmão referenciado por `<sqlFile path="../sql/..." relativeToChangelogFile="true">` | um `.sql` dentro de `changes/` roda como changeSet cru |
| ATD-LQB-007 | MUST | SAPIA-NI e pastas de include explícito: acrescentar o `<include>` do arquivo novo no `db-changelog.xml` da pasta, na ordem | nenhum teste pega include esquecido no SAPIA-NI |
| ATD-LQB-008 | MUST | `ALTER TABLE` no central numa tabela de `MONITORED_TABLES` exige o mesmo `ALTER TABLE` no SAPIA-NI, na pasta do **mesmo mês** (é outro repositório: ver "Fora deste repositório") | `LiquibaseJediRDATest` com o fdsc-rules clonado ao lado |
| ATD-LQB-009 | MUST | `CREATE TABLE` no SAPIA-NI com coluna `ID_ORIGEM`, ou um changeSet `ALTER TABLE <tabela> ADD ID_ORIGEM NUMBER(22)` no mesmo arquivo | `LiquibaseValidationTest` |
| ATD-LQB-010 | MUST | Procedure/function/view: o código vive no arquivo canônico (`procedure/`, `function/`, `view/`, um `.sql` por objeto, `runOnChange="true"`, `author="ATD"`, `id="proc-<NOME>"`/`func-`/`view-`, `sqlFile encoding="UTF-8" endDelimiter="--GO"`); na pasta do mês vai só um XML de rastreio com `SELECT 1 FROM DUAL` e o comentário da entrega | `procedure/db-changelog-procedures.xml` |
| ATD-LQB-011 | MUST | Trigger (ex.: de sequence) com `<createProcedure procedureName="T_<TABELA>" catalogName="N">`: `N` = 1 para objeto novo e a última versão + 1 para alterar um existente (controle da tabela `DATABASECHANGECUSTOM`) | `LiquibaseChanceLogTest` acusa `catalogName` repetido ou fora de ordem |
| ATD-LQB-012 | SHOULD | Constraint: no espelho do SAPIA-NI, `DROP CONSTRAINT` condicionado à existência antes do `ADD CONSTRAINT`; flags `FG_*` com check `IN ('S','N')` | doc do JediRDA |
| ATD-LQB-013 | SHOULD | Prefixos: `PK_`, `FK_`, `CK_`, `UK_`, `IDX_`/`IX_`; colunas `CD_`, `DS_`, `DT_`, `FG_`, `TP_`, `NR_`, `VL_`, `QT_`, `ID_`, `NM_` | revisar |
| ATD-LQB-014 | SHOULD | Sem `<rollback>` (não é prática nos dois modelos); nunca inventar rollback destrutivo | — |

## Receitas

**Pasta do mês nova.** A pasta é a da **versão da entrega** (`AA.MM` da `DESENVOLVIMENTO-AA-MM-0-0`/release; uma `RELEASE-AA-MM-0-N` reaproveita a pasta do mês) — confirme no plano. Se ela ainda não existe:
- Central: `DEV-AA-MM-0-0/db-changelog.xml` só com o `includeAll`, `DEV-AA-MM-0-0/changes/001-start-version.xml` e o `<include file="DEV-AA-MM-0-0/db-changelog.xml" relativeToChangelogFile="true" />` depois da última linha `DEV-*` do `db-changelog-central.xml` (antes de `function/`). Nunca deixe `changes/` vazio: derruba o startup.
- SAPIA-NI: `SAPIA-NI-AA-MM-0-0/001-start-version.xml`, `SAPIA-NI-AA-MM-0-0/db-changelog.xml` com o include dele e o include da pasta no fim do `db-changelog-master.xml`.
- Start-version: `<comment>liquibase de inicio da versao AA.MM.0.0</comment>` e `<sql>select 1 from dual</sql>`.

**Coluna nova (central).** ChangeSet 1: precondição `USER_TABLES` = 1 e `USER_TAB_COLUMNS` da coluna = 0 (ou `<not><columnExists/></not>`), `ALTER TABLE <T> ADD <C> <tipo>`. ChangeSet 2: precondição `<columnExists>`, `COMMENT ON COLUMN`. Se `<T>` estiver em `MONITORED_TABLES`, registre o espelho.

**Tabela nova.** `CREATE TABLE` com precondição `USER_TABLES` = 0; changeSet de comentários (tabela e todas as colunas; vários `COMMENT ON` separados por `;` num `<sql>`); PK (`ALTER TABLE … ADD CONSTRAINT PK_… PRIMARY KEY (…) using index`, precondição em `USER_CONSTRAINTS`); FKs, checks e índices; sequence + trigger `T_<TABELA>` via `createProcedure` se o ID for gerado.

**Parâmetro.** `INSERT INTO CM_PARAM (NM_PARAM, SG_CATEGORIA, NM_TITULO, DS_EXPLICACAO, DM_PARAM, SG_NIVEL, VL_PARAM)` com precondição `COUNT(*) … WHERE NM_PARAM = '<nome>'` = 0. Arquivo `NNN-SAT-I<n>-create-param-<nome>.xml`.

**Menu de tela Vue.**
- Tela migrada (menu legado já existe): `UPDATE AUTSC2_MENUS` (ou `CM_MENUS`) `SET DS_URL_APP = '<rota Vue>', DS_CHAVE_PERMISSAO = '<modulo>.<tela>', FG_USA_APP = 'S' … AND DS_URL_APP IS NULL`. **Nunca** remova ou sobrescreva `DS_URL` — é o fallback quando `FG_USA_APP` não está ativo. `DS_URL_APP` casa exatamente com a rota do Vue Router (`/autsc2/app/...` ou `/cmagnet/app/...`, kebab-case, sem barra final) e `DS_CHAVE_PERMISSAO` com o recurso exigido pelo endpoint REST.
- Tela nova: `INSERT INTO <MENUS> (…) SELECT … FROM <MENUS> AM WHERE AM.DS_URL = '<menu de referência>' AND ROWNUM = 1` (copia módulo e flags; `NR_ORDEM` = máximo do módulo + 10), com precondição pelo `DS_URL` novo = 0; depois as permissões em `AUTSC2_GRUPOS_FUNCOES`/`CM_GRUPOS_FUNCOES` copiadas do menu de referência.

**Procedure/function/view.** Edite o `.sql` canônico (um statement, sem `/` final; o arquivo não usa `--GO`) e acrescente o changeSet dele em `db-changelog-<tipo>s.xml` se for objeto novo; na pasta do mês, o XML de rastreio. Antes de editar um `.sql` canônico existente, confira o encoding real: vários estão em ISO-8859-1 apesar do `encoding="UTF-8"` — não converta de passagem, reporte.

**Espelho no SAPIA-NI** (quando você estiver no fdsc-rules): pasta do mesmo mês da mudança no central; próximo `NNN` livre; costuma-se repetir o nome do arquivo e o id do changeSet do central; precondições com `MARK_RAN`; constraint com DROP antes do ADD; include no `db-changelog.xml`. Replique o `ALTER TABLE` como está no central (o teste normaliza comentários, espaços, caixa e prefixo de schema, mas compara a estrutura).

## Encoding

XML e SQL das pastas recentes em **UTF-8 sem BOM**, nos dois modelos. Preserve o encoding de arquivo existente que você alterar (há `.sql` canônicos em ISO-8859-1 e arquivos antigos com `encoding="latin1"`). Depois de editar, procure `�` e sequências como `Ã£`/`Ã§` no diff.

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

## Validação (offline)

- cmagnet-util, da raiz: `mvn -q -pl cmagnet-liquibase "-Dtest=br.com.zitrus.liquibase.LiquibaseChanceLogTest,br.com.zitrus.liquibase.LiquibaseJediRDATest" test`. O JediRDA só valida de verdade com o fdsc-rules clonado ao lado do cmagnet-util (ou com `-Dsapia.ni.liquibase.dir=<raiz liquibase do fdsc-rules>`); sem ele, avisa e passa — é o que acontece no CI.
- fdsc-rules, da raiz: `mvn -q -pl dynamix-rules-server -am "-Dtest=LiquibaseValidationTest" "-Dsurefire.failIfNoSpecifiedTests=false" test`.
- Se existir `scripts/validate-cmagnet-liquibase.ps1` no repositório, rode-o também.
- Pontos cegos — confira à mão: o `LiquibaseChanceLogTest` só olha as 3 últimas pastas `DEV-26-*` (pastas `DEV-27-*` passam sem validação), e não confere o include no master, as pastas canônicas, os alvos de `sqlFile`, ids ou precondições; o `LiquibaseValidationTest` olha as 5 últimas pastas SAPIA-NI e pode falhar falsamente quando há parênteses antes do `ID_ORIGEM` (ex.: `VARCHAR2(10)`) — se acontecer, reporte em vez de reescrever a tabela para agradar o teste.

## Fora deste repositório

- Mudança no central que pede espelho no SAPIA-NI → item em "Riscos/dúvidas" do plano: repetir no fdsc-rules (mesmo mês, mesmo chamado, `/dev-ticket` próprio). O contrário também: tabela nova no SAPIA-NI que o app usa precisa existir no central.
- Tela que depende de menu ou tabela nova → o changeset é deste repositório (cmagnet-util) e a tela é do cmagnet/autsc2: o plano registra a ordem (banco antes do app).
- Ver o mapa completo em `dev-atd-ecossistema`.

## NUNCA

- Nunca aplique changelog em banco: nada de `liquibase update`, `mvn liquibase:*`, `config-atd/liquibase/lib/launcher.bat`, `ExecuteLiquibaseFile_*.bat` ou a UI do `config-atd/atd-dev`. O POM tem a configuração do plugin com uma conexão de DEV: invocar um goal do plugin aplicaria nela.
- Nunca use `runOnChange`/`runAlways` para contornar checksum de changeSet versionado comum.
- Nunca use `cmagnet-util/scripts/node/liquibase.js` (regera o changelog da pasta só com includes explícitos e apaga o `includeAll`), o `Resources/database/liquibase/db-changelog-master.xml` do cmagnet nem o `sgucard-liquibase` (nenhum dos dois é usado).
- Nunca apague, mova ou mostre o conteúdo de `cmagnet-liquibase/src/main/resources/ExecuteLiquibaseFile.bat`: os testes dependem dele e ele tem credenciais. O mesmo vale para qualquer credencial que aparecer em POM ou script.
- Os arquivos `*exemplo*` da `DEV-26-09-0-0/changes/` rodam de verdade (criam e removem objetos): não os trate como documentação nem os copie sem entender.
- Nunca crie changeset no capa nem no servicos-online-benef: o Liquibase deles foi desligado em 2026-09-22, e um arquivo novo nas pastas `CAPA-*`/`SOB-*` nunca rodaria. A mudança vai no central.
