---
name: dev-atd-cmagnet-laudos-backend
description: "Backend do cmagnet-laudos do ATD: o WAR wslaudos (CXF no WildFly, 2 operações SOAP V01_00 de resultados de exames) e 7 jars de biblioteca de laudos (validação de templates, anexos, importação, DAOs) usados por autsc2, cmagnet, proc-guias e servicos-online-benef. Cobre a cadeia PortTypeImpl → WSActionImpl → DAO, o SQL em constantes das classes dao/base com dynamix-dbutil, quem controla a transação, o model JAXB mantido à mão, as duas cópias do XSD, a versão fixada nos consumidores (version.cmagnet-laudos-pom = 6.01.H-WILDFLY no fdsc-pom) e a validação sem suíte de testes. Use para criar ou alterar código no cmagnet-laudos. NÃO cobre as telas de laudos do cmagnet (CodeCharge, use dev-atd-codecharge) nem o job de importação do proc-guias (use dev-atd-proc-guias-backend)."
---

# cmagnet-laudos (WS e biblioteca de laudos)

O repositório tem duas partes:
- o WAR `cmagnet-laudos-web` (`wslaudos`), com 2 operações SOAP na versão V01_00;
- 7 jars de biblioteca de laudos, usados por outros 4 repositórios.

A última mudança funcional é de 2023 (`5e20f14`); depois disso, só pipeline e versionamento. Mudança aqui é rara e quase sempre pede decisão de versão (ver "Versão nos consumidores"). O repositório não tem docs: esta skill e o código são a referência.

## Módulos

| Módulo | O que vai nele | Quem consome |
|---|---|---|
| `cmagnet-laudos-web` (WAR) | `web.xml`, SEI e `*PortTypeImpl` em `br.com.unimedsc.datamodel.laudos.<mensagem>.v0100`, WSDL e XSD em `src/main/resources/wsdl/v01_00/` | — |
| `cmagnet-laudos-action` | `esb/action/WSLaudosAbstractRequisicaoAction` (fluxo), `WSLaudosBaseAction`, `v01_00/WSLaudosV0100AbstractAction`, `v01_00/impl/*WSActionImpl` | — |
| `cmagnet-laudos-dao` | `dao/*DAO` (ex.: `LaudosResultadoDAO`), SQL em `dao/base/*BaseDAO`/`*DAOBase`, `dao/esb/` (`DBPool`, `LaudosMsgDAO`), `dao/rda/` | proc-guias, servicos-online-benef |
| `cmagnet-laudos-core` | validação de laudos e templates (`validator/impl`, `validador/digester`, `templates/dominios`) e anexos (`anexo/LaudosResultadoAnexosControl`) | autsc2, cmagnet (telas CodeCharge de laudos), servicos-online-benef |
| `cmagnet-laudos-proc` | importação de arquivos de laudo (`ImportadorLaudos`, `UnzipLaudo`, `LaudoFilesManager`) | proc-guias |
| `cmagnet-laudos-commons`, `-error` | beans, enums, exceções, `UtilXML` | cmagnet (commons), servicos-online-benef (error) |
| `cmagnet-laudos-model` | JAXB `model/v0100` (40 classes) e `model/estrutura/xml`; segunda cópia do XSD em `src/main/resources/xsd/v01_00/` | — |

Depende de `cmagnet-commons`, `cmagnet-parameter`, `cmagnet-rda` e `cmagnet-tiss` (cmagnet-util) e de `dynamix-rules-util` (fdsc-rules).

## Fluxo do WS

1. O `web.xml` mapeia `V01_00/MensagemConsultaResultados` e `V01_00/MensagemResultadosExames` para os `*PortTypeImpl`, que têm `@WebService(wsdlLocation = "wsdl/v01_00/...")`.
2. O Impl chama `new MensagemResultadosExamesWSActionImpl("MensagemResultadosExamesWS", "V01_00").process(...)`.
3. `WSLaudosAbstractRequisicaoAction.process`:
   - abre a conexão do `DBPool`;
   - grava a entrada (`processarMensagemLaudosRecebimento`, com `LaudosMsgDAO` e os DAOs);
   - gera o id de saída e monta a resposta (`processarMensagemLaudosResposta`);
   - fecha a conexão no `finally`.
4. Erro de regra de negócio volta como `LaudosFault` com a lista de restrições (`CtListaErroRegraNegocio`).

## Acesso a dados e transação

- O SQL fica em constantes `protected static final String SQL_*` da classe base em `dao/base/`. O DAO concreto a estende, executa com `DynaDBUtil.createPs(conn, SQL_…)` e fecha com `doFinallyClose(null, ps, null)`. Parâmetro sempre amarrado.
- Métodos de biblioteca que recebem `Connection` (DAOs, `LaudosResultadoAnexosControl`) **não** controlam a transação: quem chama (a tela do cmagnet, o job do proc-guias, o SOB) faz `commit` ou `rollback`.
- Por isso o método propaga a `SQLException` em vez de só logar: é o ajuste do `5e20f14`, que trocou `catch (Exception)` com log por `catch (SQLException)` com `throw`. Não engula exceção num método que recebe `Connection`.

## Model e XSD

- O model JAXB `model/v0100` é **mantido à mão** (sem marcador de geração; o último ajuste de XSD, `b1d54b8`, mexeu no model e nos validadores juntos). Não rode xjc nele.
- O XSD existe em duas cópias que precisam ficar iguais: `cmagnet-laudos-model/src/main/resources/xsd/v01_00/` e `cmagnet-laudos-web/src/main/resources/wsdl/v01_00/`. A pasta `example/` do web é material de exemplo, fora do fluxo.
- Campo novo na mensagem mexe nos dois XSD, na classe do model, no validador do `core` que confere o campo e no DAO que o grava ou lê.

## Versão nos consumidores

- Os 4 consumidores usam `${version.cmagnet-laudos-pom}`, que o `fdsc-pom` fixa em `6.01.H-WILDFLY`. A versão 26.x construída a partir da `HOMOLOG` **não é consumida**.
- Uma mudança na biblioteca só chega a autsc2, cmagnet, proc-guias ou SOB com a publicação de uma versão nova e a troca dessa propriedade no `fdsc-pom`. Isso é decisão do time e vai no plano como pendência, com os consumidores afetados. O agent nunca muda a propriedade.

## Stack e encoding

- Java 1.8 (`fdsc-pom`), fontes ISO-8859-1. Um arquivo já tem `�` (`commons/esb/processor/UtilXML.java`): não corrija de passagem.
- Preserve o encoding e confira o diff atrás de `�` e `Ã`.

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

- `mvn -q package` na raiz compila os 8 módulos e empacota o WAR. O `mvn test` não roda nada: os arquivos de teste ficam em `src/main/test`, fora da raiz de teste do Maven.
- Esses arquivos são utilitários `main()` que conectam em bancos e serviços reais, com credenciais no código: nunca os rode nem os copie.
- Confira à mão as duas cópias do XSD iguais e, no SQL alterado, o mesmo número de colunas, de `?` e de `set*`.
- **Fica como pendência:** chamar a operação num ambiente e, se a biblioteca mudou, o bump de versão nos consumidores.

## Regras

| ID | Nível | Regra | Como verificar |
|---|---|---|---|
| ATD-LAU-001 | MUST | Método que recebe `Connection` não faz `commit`, `rollback` nem `close` e propaga a `SQLException` | diff |
| ATD-LAU-002 | MUST | SQL em constante da classe base, com parâmetro amarrado | diff |
| ATD-LAU-003 | MUST | Mudança de schema nas duas cópias do XSD e no model à mão, sem xjc | diff |
| ATD-LAU-004 | MUST | Mudança na biblioteca registra no plano o bump de `version.cmagnet-laudos-pom` e os consumidores afetados | plano |
| ATD-LAU-005 | MUST | Encoding preservado, sem `�` novo | diff |
| ATD-LAU-006 | SHOULD | Mudança aditiva em `core`, `dao`, `proc`, `commons` e `error` (são API de outros repositórios) | diff |

## Fora deste repositório

- Tabela ou coluna de laudos → banco central (`dev-atd-liquibase`), antes.
- Uso da biblioteca em autsc2, cmagnet, proc-guias ou SOB → no repositório consumidor, depois da versão nova publicada e do bump no `fdsc-pom`.
- Mapa completo: `dev-atd-ecossistema`.

## NUNCA

- Nunca rode os `main()` de `src/main/test` (ex.: `LaudoConnection`): conectam em bancos reais com credenciais gravadas.
- Nunca rode `mvn deploy` nem o perfil `-Pdocker`: publicam no Nexus e no registry de imagens.
- Nunca rode `scripts/node` nem `scripts/sh`: chamam as APIs reais do Jira e do GitLab e fazem `git config --global` e `branch -D`.
- Nunca copie para código, plano ou relatório o conteúdo de `wslaudos.properties`, `client.properties`, do keystore `*.javaks` ou do `sonar-project.properties`.
- Nunca troque versão de POM nem a propriedade `version.cmagnet-laudos-pom` à mão.
