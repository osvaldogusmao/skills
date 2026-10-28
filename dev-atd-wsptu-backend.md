---
name: dev-atd-wsptu-backend
description: "Backend do wsptu, os web services SOAP do PTU (intercâmbio entre Unimeds) do ATD: JBossWS-CXF no WildFly, 8 módulos e as versões PTU V60 a V100. Cobre a cadeia PortTypeImpl → WSActionImpl → DAO por versão, os enums de transação (TransacaoPTU, VersaoTransacaoPTU, TransacaoActionEnum), a procedure SP_PRCSSA_MSG_PTU_V* do banco central, as duas cópias de XSD, o model JAXB e os SEIs gerados e commitados, as receitas de coluna nova, ajuste de schema e versão PTU nova, acesso a dados com dynamix-dbutil, o encoding misto e a validação sem suíte de testes. Use para criar ou alterar código no wsptu. NÃO cobre o wstiss3 (TISS, use dev-atd-wstiss3-backend), o cmagnet-laudos nem o corpo da procedure (use dev-atd-liquibase)."
---

# wsptu (web services do PTU)

O wsptu recebe por SOAP as transações PTU das outras Unimeds (JBossWS-CXF no WildFly). Ele grava a mensagem nas tabelas `PTU_*`, chama a procedure da versão no banco central e devolve a resposta montada com o que a procedure gravou. O `web.xml` tem 99 servlets, um por serviço e versão (V60_00 a V100_00). O repositório não tem docs: esta skill e o código são a referência.

## Módulos

| Módulo | O que vai nele |
|---|---|
| `wsptu-web` (WAR) | `web.xml`; por serviço e versão, o SEI `<Servico>PortType` e o `<Servico>PortTypeImpl` em `br.coop.unimed.ptu.<servico>.webservices.v<NNN>_00`; WSDL e XSD em `src/main/resources/wsdl/v<NNN>_00/` |
| `wsptu-action` | `WSPTUBaseAction` (o fluxo), `TransacaoActionEnum`; por versão, `v<NNNNN>/WSPTUv<NNNNN>AbstractAction` e `impl/<Servico>WSActionImpl` |
| `wsptu-dao` | `sql/DBPool`, `procedure/SenderProcedure`; por versão, `v<NNNNN>/<Bloco>DAO` e, na v10000, `rules/` |
| `wsptu-model` | classes JAXB por versão (`br.com.unimedsc.ptu.esb.model.v<NNNNN>`) e a segunda cópia do XSD em `src/main/resources/META-INF/XSDs/PTU/V<NNN>_00/` |
| `wsptu-flow` | `PTUMessageV<NNNNN>` por versão, `auth/` e `router/` (encaminhamento HTTP para outras Unimeds, `WSPTUDynamicHttpRouter`) |
| `wsptu-common` | enums `TransacaoPTU` e `VersaoTransacaoPTU` |
| `wsptu-converter`, `wsptu-client` | conversões e cliente |

A versão aparece em três grafias: `V100_00`/`v100_00` (WSDL, XSD e URL), `v10000` (pacotes Java de action, dao, model e flow) e `PTU100`/`_100` (enums).

## Fluxo de uma mensagem

1. `<Servico>PortTypeImpl` tem `@WebService(wsdlLocation = "wsdl/v100_00/...")` e `@SchemaValidation(type = BOTH)`: o CXF valida pedido e resposta contra a cópia do XSD em `wsptu-web`. O Impl só chama `new <Servico>WSActionImpl("<Servico>WS", "V100_00").process(obj, null, null)`.
2. O `WSPTUBaseAction` converte o objeto para XML, valida de novo contra a cópia do XSD em `wsptu-model` (`getXSDFilePath()` da versão) e confere o hash MD5.
3. Em `doProcessarMensagemPTURecebimento`, abre a conexão do `DBPool`, faz `setAutoCommit(false)`, chama o `processarMensagemPTURecebimento(obj, con)` do `WSActionImpl` e faz `commit`. O `WSActionImpl` grava a mensagem (`PTUMsgDAO`, com `TransacaoPTU.getByCdTransacao`) e depois os DAOs dos blocos.
4. `doChamarProcedure`: o `SenderProcedure` executa `{CALL <procedure>(?,?)}`, com a procedure definida na constante de `TransacaoPTU` (ex.: `SP_PRCSSA_MSG_PTU_V10`). No envio síncrono, as chamadas da mesma chave são serializadas.
5. Lê o retorno pelos `consultar(...)` dos DAOs, converte para XML e acrescenta o hash.

A regra de negócio da transação costuma estar na procedure, não no Java. As procedures ficam no banco central (ver `dev-atd-liquibase`), mas só a da v10 tem arquivo no `cmagnet-liquibase` (`central/procedure/SP_PRCSSA_MSG_PTU_V10.sql`). As das versões V6 a V9 (V9 vale para V90, V91 e V92) não têm changeset: mudar uma delas exige a definição atual do banco, então registre a dúvida no plano.

## Receitas

**Coluna nova numa tabela PTU** (`580f2a28`, SAT-I2409, só o `OrdemServicoDAO` da v10000):
- a coluna entra no `INSERT` (lista de colunas e mais um `?`) e no `inserir`, com `setNull` quando o campo vier vazio;
- e no `consultar`, na leitura do `ResultSet` para o objeto do model;
- a coluna no banco é um changeset no central, que vem antes.

**Ajuste de schema numa versão** (`9192a96e`, SAT-I2393, CNPJ alfanumérico na V92):
- as **duas** cópias do XSD, com a mesma mudança: `wsptu-model/.../META-INF/XSDs/PTU/V92_00/` e `wsptu-web/.../wsdl/v92_00/`. Uma valida no CXF e a outra na action;
- nas versões V90 a V100 as duas cópias são iguais. Na V60, V70 e V80 elas já divergem hoje: há enumerações diferentes no `ptu_SimpleTypes` e, na V60, um `minOccurs` diferente no `ptu_Transacoes`. Não iguale de passagem; se o chamado mexer numa dessas versões, registre no plano qual cópia vale;
- a classe do model afetada, ajustada à mão para bater com o XSD;
- o DAO que grava ou lê o campo.

**Versão PTU nova** (`d7abd432`, SAT-I2392, PTU v10): parte da cópia da versão anterior, renomeada. Mudam juntos:
- `wsptu-web`: WSDL e XSD em `wsdl/v<NNN>_00/`, o SEI e o Impl de cada serviço e o bloco de servlets e `url-pattern` no `web.xml`;
- `wsptu-model`: as classes JAXB e o `package-info.java` da versão, mais a cópia do XSD em `META-INF/XSDs/PTU/V<NNN>_00/`;
- `wsptu-action`: `WSPTUv<NNNNN>AbstractAction`, `WSPTUv<NNNNN>ExceptionsManager` e os `impl/`;
- `wsptu-dao`: os DAOs, o `WSDAOV<NNNNN>Model` e, se houver, `rules/`;
- `wsptu-flow`: `PTUMessageV<NNNNN>`;
- enums: `VersaoTransacaoPTU`, as constantes da versão em `TransacaoPTU` (com a procedure) e o mapa de `TransacaoActionEnum`;
- banco: a procedure nova no central.

Os scripts `scripts/novas_versoes/*.sh`, `generate-files.bat` (wsimport) e `jaxb.model.bat` (xjc) fazem essa cópia e a geração com versões e caminhos fixos no próprio arquivo. Só os use se o plano disser qual e como. No caso comum, faça as cópias e as renomeações à mão e confira o resultado com `git status`.

**`package-info.java`:** o `.gitignore` tem `package*`, e os 7 `package-info.java` do model entraram com `git add -f` (`b9a64fdc`). Numa versão nova, o dela também precisa de `git add -f`; sem ele, as classes da versão ficam sem o `@XmlSchema` do namespace.

## Acesso a dados

- SQL em constantes `static final` do DAO (`INSERT`, `SELECT`), executadas com `DynaDBUtil.createPs(conn, SQL)` e fechadas com `doFinallyClose(null, pst, rs)`. Parâmetro sempre amarrado.
- O DAO recebe a `Connection` e não a fecha nem faz `commit`: quem abre, comita e fecha é o `WSPTUBaseAction`. Não há `rollback` no `catch` dele: é um achado conhecido. Não corrija de passagem; se o chamado pedir, registre no plano.
- O DAO de versão estende `WSDAOV<NNNNN>Model` e implementa `IPTUKeyController` quando participa da chave da mensagem.

## Stack e encoding

- Java 1.8 (`fdsc-pom`), compilado com JDK 8 no CI (`maven:3.9.6-eclipse-temurin-8`). Há um `import jdk.nashorn...` num DAO da v10000 que só compila com JDK 8: não troque o JDK.
- Depende do `cmagnet-commons` (cmagnet-util) e do `dynamix-dbutil`.
- As fontes são declaradas ISO-8859-1, mas há `.java` gravados em UTF-8, e 171 arquivos já têm `�` (U+FFFD), quase todos em comentários dos DAOs. Preserve o encoding de cada arquivo, não "conserte" `�` existente e confira o diff atrás de `�` e `Ã` novos.

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

## Validação

- Não há teste automatizado. Valide com `mvn -q package` na raiz, que compila os 8 módulos e empacota o WAR. Os artefatos de cima (`fdsc-pom`, `cmagnet-commons`) precisam estar instalados ou publicados; se a resolução falhar, reporte, não contorne.
- O build não confere WSDL/XSD contra o model nem o SQL. Confira à mão: XML bem-formado nos arquivos alterados, a mesma mudança nas duas cópias do XSD e, no `INSERT`, o mesmo número de colunas, de `?` e de `set*`.
- **Fica como pendência:** chamar o serviço num ambiente com uma mensagem real da versão.

## Regras

| ID | Nível | Regra | Como verificar |
|---|---|---|---|
| ATD-PTU-001 | MUST | Mudança de schema aplicada igual nas duas cópias do XSD da versão | diff das duas pastas |
| ATD-PTU-002 | MUST | Coluna nova no `INSERT`, no `inserir` e no `consultar` do DAO, com `setNull` para vazio | diff |
| ATD-PTU-003 | MUST | DAO não fecha nem comita a `Connection` recebida | diff |
| ATD-PTU-004 | MUST | Versão nova com todos os arquivos da receita e os 3 enums | `git status` × receita |
| ATD-PTU-005 | MUST | `package-info.java` de versão nova adicionado com `git add -f` | `git ls-files` |
| ATD-PTU-006 | MUST | SQL com parâmetro amarrado | diff |
| ATD-PTU-007 | MUST | Encoding de cada arquivo preservado, sem `�` novo | diff |
| ATD-PTU-008 | SHOULD | Mudança numa versão não altera as outras, salvo pedido do chamado | diff |

## Fora deste repositório

- Tabela, coluna ou procedure `SP_PRCSSA_MSG_PTU_V*` → banco central (`dev-atd-liquibase`), antes do wsptu.
- Módulo do cmagnet-util que precisa mudar → cmagnet-util primeiro, com `install` local.
- Mapa completo: `dev-atd-ecossistema`.

## NUNCA

- Nunca rode `mvn deploy` nem o perfil `-Pdocker`: publicam no Nexus e no registry de imagens.
- Nunca rode `scripts/node` nem `scripts/sh`: chamam as APIs reais do Jira e do GitLab e fazem `git config --global` e `branch -D`.
- Nunca rode os scripts de geração (`scripts/novas_versoes/*.sh`, `generate-files.bat`, `jaxb.model.bat`) sem o plano dizer qual e como.
- Nunca copie para código, plano ou relatório o conteúdo dos `httpRouter*.properties`, do keystore `*.javaks` ou do `sonar-project.properties`: há segredos versionados neles.
- Nunca troque versão de POM à mão.
