---
name: dev-atd-wstiss3-backend
description: "Backend do wstiss3, os web services SOAP do padrão TISS 3/4 (ANS) do ATD: CXF no WildFly, 14 versões TISS publicadas (v3_02_00 a v4_03_00) com 11 a 12 operações cada, processadas por Smooks 1.4 sem DAO Java. Cobre o fluxo WSTiss3Process → SmooksHttpAction → EMessageType, o layout por versão (WSDL, config/<versão>/<operação> com smooks.xml, binding.xml, executor.xml e response.ftl, SEI e Impl, model JAXB, bloco no web.xml), o SQL nos executors e as procedures e views SP_WS_TISS_*/V_WSTISS_* do banco central, as receitas de correção replicada entre versões e de versão TISS nova, o package-info com @XmlNs e a validação sem suíte de testes. Use para criar ou alterar código ou configuração no wstiss3. NÃO cobre o wsptu (PTU, use dev-atd-wsptu-backend), a importação de XML TISS do proc-guias nem o corpo das procedures (use dev-atd-liquibase)."
---

# wstiss3 (web services do TISS)

O wstiss3 recebe por SOAP as mensagens TISS dos prestadores (CXF no WildFly). Não há DAO em Java: cada operação é uma configuração **Smooks 1.4** que grava a mensagem nas tabelas `WS_TISS_MSG*`, chama a procedure de avaliação no banco central e monta a resposta com FreeMarker a partir de SELECTs em views `V_WSTISS_*`. O `web.xml` tem 159 servlets: 14 versões (v3_02_00 a v4_03_00), com 11 ou 12 operações cada. A versão `v3_02_02` tem só WSDL e config, sem model, Impl nem endpoint no `web.xml`. O repositório não tem docs: esta skill e o código são a referência.

## Módulos e layout por versão

| Onde | O que vai nele |
|---|---|
| `wstiss3-web/src/main/java/br/gov/ans/tiss/ws/tipos/<operacao>/v<NNNNN>/` | `Tiss<Op>PortType` (SEI gerado) e `Tiss<Op>PortTypeImpl` |
| `wstiss3-web/src/main/resources/wsdl/v<V>/` | WSDL e XSD da versão (ex.: `v4_03_00`) |
| `wstiss3-web/src/main/resources/config/v<V>/<op>_v<V>/` | `smooks.xml`, `bindings/binding.xml`, `executors/executor.xml`, `response.ftl` (o `loteguias` tem também `clean.xml`) |
| `wstiss3-web/src/main/webapp/WEB-INF/web.xml` | um servlet e um `url-pattern` `v<V>/tiss<Op>` por operação e versão |
| `wstiss3-model` | JAXB gerado com `xjc -no-header`: `br.gov.ans.padroes.tiss.schemas.v<NNNNN>` e `org.w3._2000._09.xmldsig_.v<NNNNN>` |
| `wstiss3-action` | `WSTiss3Process`, `EMessageType`, `esb/SmooksHttpAction`, `utils/database/DynaDataSource` |

A versão aparece como `v4_03_00` (WSDL, config e URL) e como `v40300` (pacotes Java).

## Fluxo de uma mensagem

1. O `Tiss<Op>PortTypeImpl` (`@WebService(wsdlLocation = "wsdl/v4_03_00/...")`, `@SchemaValidation(type = BOTH)`) chama `WSTiss3Process.processar("tiss<Op>_v4_03_00", pedido, new <Resposta>())`.
2. `EMessageType.getByName` separa o nome da operação e a versão a partir do sufixo `_v3…`/`_v4…`. `getConfigPath()` monta `/config/<versão>/<configPath>_<versão>/smooks.xml`. Uma versão nova **não** muda o enum; só uma operação nova cria constante (nome, pasta de config e nome do serviço no banco).
3. O `SmooksHttpAction`:
   - valida o XML e o hash;
   - grava o registro de entrada em `WS_TISS_MSG`, que vira `idEntrada`, e põe `idEntrada` e `prfx` no contexto;
   - abre a conexão com `setAutoCommit(false)` e a injeta como o datasource `DBExtractTransformLoadDS` (`DynaDataSource`);
   - roda o Smooks e depois remove os acentos da resposta;
   - acrescenta o hash de saída e faz `commit`.
4. No `smooks.xml`:
   - importa o `binding.xml`, que monta `HashMap`s com `jb:bean`, `jb:wiring` e `jb:value` por elemento, e o `executor.xml`, que faz os `INSERT` em `WS_TISS_MSG_*` com valores `${bean.caminho}`;
   - depois `COMMIT` e `{call SP_WS_TISS_AVALIA_MSG(${idEntrada})}`;
   - depois os SELECTs em `WS_TISS_MSG*` e nas views `V_WSTISS_*`, guardados com `db:resultSet name="…RS"`;
   - o `response.ftl` monta o envelope SOAP a partir desses `…RS`, com o prefixo `${prfx}`.

O `<import file="/config/datasources.xml"/>` está comentado em todos os `smooks.xml`. A conexão vem do Java, e esse arquivo tem credenciais: não o descomente nem o copie.

Procedures e views (`SP_WS_TISS_*`, `V_WSTISS_*`) ficam no banco central (`dev-atd-liquibase`): as procedures em `central/procedure/` e parte das views em `central/view/`. Regra de negócio e dado novo na resposta costumam começar lá. Nem toda view usada tem changeset (ex.: `V_WSTISS_SOLIC_STAT_AUTRIZ_ITE` não aparece no `cmagnet-liquibase`): mudar uma dessas exige a definição atual do banco, então registre a dúvida no plano.

## Receitas

**Correção replicada entre versões** (`66975ae4`, SAT-I2921):
- a correção (ex.: `?string?left_pad(2, "0")` num campo do `response.ftl`) foi aplicada nas mesmas 2 operações das 15 pastas de versão (30 arquivos);
- o `1bc67251` (SAT-I1463, `ORDER BY` num SELECT do `smooks.xml`) corrigiu só as 2 versões em que o problema apareceu;
- quais versões entram é decisão do plano: liste-as e aplique a mesma mudança em todas.

**Dado novo na resposta:** a coluna na view `V_WSTISS_*` (central, antes), o SELECT no `smooks.xml` (se não for `*`) e o elemento no `response.ftl`, na ordem do XSD da versão.

**Versão TISS nova** (`ca8badb5`, SAT-I2017, 4.03.00). Mudam juntos:
- `wstiss3-model`: as classes geradas da versão (`schemas/v<NNNNN>` e `xmldsig_/v<NNNNN>`);
- `wsdl/v<V>/`: WSDL e XSD da ANS;
- `config/v<V>/`: cópia da versão anterior, com os nomes e os caminhos de import trocados;
- `wstiss3-web`: SEI e Impl de cada operação;
- o bloco de servlets e `url-pattern` no `web.xml`.

Os `generate_files.bat` (xjc/wsimport), o `update_config.bat` e o `update_version_2.0.bat` fazem essa cópia e a geração com versões e caminhos fixos no arquivo. Só os use se o plano disser qual e como.

**`package-info.java` do model:** o xjc gera sem prefixo. Depois de gerar, reaplique à mão o `@XmlNs(prefix = "ans", namespaceURI = "http://www.ans.gov.br/padroes/tiss/schemas")` no `@XmlSchema` (`baa4e3fa`, SAT-I1780). Sem ele, a resposta sai sem o alias `ans:`.

Código gerado (model e SEI) não se edita à mão, exceto esse `package-info.java`.

## Stack e encoding

- Java 1.8 (`fdsc-pom`), fontes ISO-8859-1; XML de config em UTF-8. Depende só do `dynamix-dbutil` e do `dynamix-logging` (não usa cmagnet-util nem fdsc-rules).
- Preserve o encoding de cada arquivo e confira o diff atrás de `�` e `Ã`.

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

- Não há teste automatizado. `mvn -q package` compila e empacota o WAR, mas **não valida** Smooks, FreeMarker, WSDL nem SQL.
- Confira à mão:
  - cada XML alterado é bem-formado; no PowerShell, `[xml](Get-Content -Raw -Encoding UTF8 <arquivo>)` falha se não for;
  - cada `${…RS…}` usado no `response.ftl` tem o `db:resultSet` correspondente no `smooks.xml`;
  - a mudança foi feita em todas as versões listadas no plano.
- **Fica como pendência:** chamar a operação num ambiente, com uma mensagem da versão.

## Regras

| ID | Nível | Regra | Como verificar |
|---|---|---|---|
| ATD-TIS-001 | MUST | Mesma correção em todas as versões listadas no plano | diff por versão |
| ATD-TIS-002 | MUST | SQL nos executors com valores `${…}` do contexto, como nos vizinhos; nunca concatenação de texto livre | diff |
| ATD-TIS-003 | MUST | Model e SEI gerados não são editados à mão, exceto o `@XmlNs` do `package-info.java` | diff |
| ATD-TIS-004 | MUST | Versão nova com model, WSDL, config, SEI/Impl e `web.xml` | `git status` × receita |
| ATD-TIS-005 | MUST | XML de config bem-formado e `response.ftl` coerente com os `db:resultSet` | leitura |
| ATD-TIS-006 | MUST | `config/datasources.xml` continua comentado e fora do diff | diff |
| ATD-TIS-007 | SHOULD | Regra de negócio na procedure ou view do central, não no FreeMarker | plano |

## Fora deste repositório

- Procedure, view ou tabela `WS_TISS_*` → banco central (`dev-atd-liquibase`), antes do wstiss3.
- Mapa completo: `dev-atd-ecossistema`.

## NUNCA

- Nunca rode `mvn deploy` nem o perfil `-Pdocker`: publicam no Nexus e no registry de imagens.
- Nunca rode `scripts/node` nem `scripts/sh`: chamam as APIs reais do Jira e do GitLab e fazem `git config --global` e `branch -D`.
- Nunca rode `generate_files.bat`, `update_config.bat` ou `update_version_2.0.bat` sem o plano dizer qual e como.
- Nunca copie para código, plano ou relatório o conteúdo de `config/datasources.xml` ou do `sonar-project.properties`: há segredos versionados neles.
- Nunca troque versão de POM à mão.
