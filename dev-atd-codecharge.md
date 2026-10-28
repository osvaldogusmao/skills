---
name: dev-atd-codecharge
description: "Manutenção das páginas CodeCharge Studio (telas .do em Java) do CMAGNET, do AUTSC2 e do servicos-online-benef (SOB): onde fica cada parte (.ccs, .ccp em UTF-8, .html e Java gerado em ISO-8859-1, classes escritas à mão; no SOB, tudo sob WebContent/), onde mexer para cada tipo de mudança, como alterar o código de evento (ações OneLine/MultiLine no .ccp e no Java gerado; Custom Code só no Java), SQL com parâmetro, compilação e a pendência obrigatória de regenerar no Studio e conferir o diff, incluindo o web.xml, cujo template está atrás do implantado nos três produtos. Use para corrigir ou alterar uma tela .do, um evento, um data source ou a classe Java que a tela chama. NÃO cobre a migração da tela para Vue (dev-atd-codecharge-migration, dev-atd-cmagnet-backend/frontend) nem banco e menu (dev-atd-liquibase)."
---

# CodeCharge (CMAGNET, AUTSC2 e servicos-online-benef)

As telas legadas `.do` do cmagnet e do autsc2, e o servicos-online-benef (SOB) inteiro, são páginas CodeCharge Studio com saída Java (Servlet). O Studio é uma IDE gráfica do Windows: **você não regenera nada**. Quem edita no Studio e regenera é o dev, então toda mudança precisa ficar certa nos arquivos que você toca e voltar com a pendência de regenerar.

## Onde fica cada coisa

| Arquivo | O que é | Encoding |
|---|---|---|
| `ccs/CMagnet.ccs` (cmagnet) / `ccs/autsc.ccs` (autsc2) | projeto do Studio: páginas, conexões, `codePage` | — |
| `ccs/<área>/<página>.ccp` | definição da página: componentes, data sources, eventos e o código das ações `OneLine`/`MultiLine` | **UTF-8**, sem declaração XML |
| `ccs/<área>/<página>.html` | template, lido em tempo de execução (`tmplPath` da `View` gerada) | ISO-8859-1 (`charset=iso-8859-1`) |
| `ccs/CCSBuild/src/<área>/<página>/` | Java gerado: `<página>Action`/`Model`/`View`, `<componente>DataObject`, `…DataObjectHandler`, `…RecordHandler`, `…Row`, `<componente><controle>Handler` | ISO-8859-1 |
| `src/br/com/dynamix/ccs/cmagnet/<área>/` (cmagnet) / `src/br/com/dynamix/ccs/autsc/<área>/` (autsc2) | classes escritas à mão que os eventos chamam | ISO-8859-1 |

Um texto acentuado de uma ação é o mesmo caractere no `.ccp` (UTF-8) e no Java gerado (ISO-8859-1), mas com **bytes diferentes**.

No SOB os caminhos são outros (raiz `WebContent/`); ver a seção "servicos-online-benef (SOB)". Os encodings são os mesmos.

## Onde mexer (escolha o menor lugar que resolve)

1. **Regra ou consulta de uma tela, já chamada por um evento:** fica na classe escrita à mão (`src/br/com/dynamix/ccs/...`; no SOB, `src/main/java/br/com/dynamix/framework/servicos/online/...`). É Java comum, e não precisa regenerar nada. Esse é o caso mais comum: no cmagnet há cerca de 4.800 ações `OneLine` e 1.600 `MultiLine`, quase sempre só chamando essas classes (ex.: `new br.com.dynamix.ccs.cmagnet.admin.beneficiario_excecao.CadastroBeneficiarioExcecaoHandler().loadTipoAtendimento(e)`).
2. **Layout, texto ou classe CSS, sem controle novo:** só o `.html`. Mantenha os marcadores `<!-- BEGIN … -->`/`<!-- END … -->` e os `{Campo}`.
3. **Código de evento:** é o trecho entre `//Event <Evento> Action <Tipo> @<id>-<hash>` e `//End Event <Evento> Action <Tipo>` no handler gerado. Onde mexer depende do tipo da ação:
   - **`OneLine` ou `MultiLine`** (categoria Dynamix): o código mora no `.ccp` (atributos `cmd_line` ou `cmd_line1`, `cmd_line2`… da `<Action>` com o mesmo `id`; no XML eles não vêm em ordem numérica) e é copiado para o Java. Mude os dois com o mesmo conteúdo. No `.ccp`, use escape XML (`&quot;`, `&amp;`, `&lt;`, `&gt;`), como o resto do arquivo. Mudança só no Java deixa o `.ccp` mostrando código que não é o que roda, e a regeneração pode desfazer a mudança.
   - **`Custom Code`** (categoria General): o `.ccp` guarda só `<Action actionName="Custom Code" … id="<id>"/>`, e o código existe apenas no bloco do Java gerado. Mude só o bloco.
   - Nunca altere os marcadores `//Event … @<id>-<hash>` e `//End Event …`, nem mova código para fora deles.
   - **Antes de editar uma ação `OneLine` ou `MultiLine`, compare os dois lados.** Hoje o código já difere entre o `.ccp` e o Java em cerca de 130 ações do cmagnet, 60 do autsc2 e 3 do SOB. Num caso, o `.ccp` tem `return` sem `;` e o Java tem `return;`. Em outro, o Java chama a classe de outra tela. Se a ação que você vai mudar já diverge, não escolha um lado: registre os dois conteúdos no resumo, como pendência para o dev decidir.
4. **Estrutura da página** (controle, componente, data source, SQL, evento ou página nova): o `.ccp`, o `.html` e o Java gerado precisam continuar coerentes entre si (ids, nomes, parâmetros); página nova entra também no `.ccs`. É a mudança mais arriscada. Prefira que o dev crie o esqueleto no Studio e ponha a regra na classe escrita à mão. Se você fizer, liste cada arquivo no resumo.

5. **JS da tela** (`ccs/js/` no cmagnet e no autsc2, `WebContent/js/` no SOB):
   - o `.html` carrega o arquivo com `<script … charset="iso-8859-1" src="…?v={VersaoSistema}">`. O Studio não gera esse arquivo, então mudança só no JS não precisa regenerar;
   - texto acentuado no JS vai em ISO-8859-1, o `charset` da tag, pelos passos da subseção de encoding;
   - prefira uma função nova e opcional, chamada só pelas telas do chamado, a mudar uma função compartilhada por várias telas (ex.: `functions.js`, `ajax_samples.js`);
   - handler de `change`, `input` ou `click` que chama algo que dispara outro evento (ex.: `clearComponents`, `.click()` num link de limpeza) precisa de trava de reentrada. No piloto, a primeira versão de uma correção entrou em laço infinito, e só a revisão pegou.

## Pendência obrigatória

Toda entrega termina com **"regenerar no CodeCharge Studio e conferir o diff"**, junto com o que conferir:

- Os trechos que você mudou continuam iguais depois de regenerar.
- **`web.xml`:** nos três produtos, o template do Studio está atrás do `web.xml` implantado. Se a regeneração reescrever o `web.xml` a partir dele, somem servlets que estão em uso, e essa parte do diff não pode ser comitada.
  - **cmagnet** (`ccs/CCSBuild/config/web.xml` × `ccs/WEB-INF/web.xml`): faltam no template 35 servlets, entre eles o RESTEasy (`/rest/*`), além do `resteasy.servlet.mapping.prefix`, do `SpaFallbackFilter` e do `SwaggerCorsFilter`. A API REST e a SPA parariam.
  - **autsc2** (mesmos caminhos): faltam no template 25 servlets AJAX (ex.: `ajaxGetPlano`). O RESTEasy, a SPA e os filtros do Swagger estão nos dois arquivos.
  - **SOB** (`WebContent/CCSBuild/config/web22.xml` e `web23.xml` × `WebContent/WEB-INF/web.xml`): faltam nos dois templates os 4 servlets de inicialização (`LoggingInit`, `SystemInit` do próprio SOB, `clientInit` do cliente do servidor de regras e `singletonInit`). Sem eles, a aplicação sobe sem log, sem a inicialização do sistema, sem o cliente de regras e sem os singletons.
- A regeneração pode reescrever arquivos que você não tocou: só entra no commit o que é do chamado.

## SQL e segurança

- **Data source do `.ccp`:** valor de entrada vai como parâmetro declarado do componente (`SQLParameters`/`TableParameters`), nunca concatenado no texto do SQL.
- **Classe escrita à mão:** use o `SqlBuilder` do produto (`br.com.dynamix.framework.cmagnet.sql.SqlBuilder` no cmagnet, `br.com.dynamix.framework.autsc.sql.SqlBuilder` no autsc2; é o padrão das classes de tela) ou `DynaDBUtil.parametrizedPs`, com parâmetro amarrado. A conexão vem de `WorkSql.getConnection()` no cmagnet e de `SqlUtil.getConnection()` no autsc2, e é fechada no `finally` (`DynaDBUtil.doFinallyClose`).
- **SOB:** não há `SqlBuilder` próprio. As classes usam `DynaSqlBuilder` ou `DynaDBUtil` direto, com a conexão de `SqlUtil.getConnection()` (`SqlUtil extends WorkSql`, que pega a conexão do `JDBCConnectionFactory` do CodeCharge), fechada no `finally`. Modelo: `dao/FaseAutorizacaoDAO`.
- **Credenciais:** os `.ccs`, o `CCSBuild/config/site.properties` e os arquivos de ambiente têm dados de conexão. Nunca os copie para código, plano ou relatório.

## Encoding

- Preserve o encoding de cada arquivo que editar: o `.ccp` é UTF-8; o `.html`, o Java gerado e o `src/br` são ISO-8859-1. Gravar um arquivo ISO-8859-1 como UTF-8 quebra os acentos na tela e no banco.
- Ao levar um texto acentuado do `.ccp` para o Java, ou o contrário, confira os dois arquivos depois de editar: nenhum `Ã` ou `�` novo no diff. O `.ccp` (UTF-8) pode ser editado direto; o `.html` e o Java, só pelos passos da subseção abaixo. Nunca "corrija" um arquivo convertendo o encoding dele.
- Java 1.6 de sintaxe e de API no cmagnet e no SOB (o autsc2 compila em 1.8).

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

Não há teste automatizado de tela. Compile o que você mudou.

- **cmagnet:** o Java gerado e as classes de tela entram nos módulos `sgucard-*` por links (ver a seção sobre links da `dev-atd-cmagnet-backend`). Rode `mvn -q -f sgucard/pom.xml -pl <modulo> -am -DskipTests compile`.
  - Antes, confira que o link existe (`sgucard/<modulo>/CCSBuild/src/<área>`); sem ele, o módulo compila vazio e sem erro.
  - Área → módulo:
    - `admin`, `empresas` → `sgucard-admin`;
    - `consultas`, `exames`, `internacoes`, `honorarios` → `sgucard-guias`;
    - `auditoria`, `auditoria_webin` → `sgucard-webin`;
    - `modal`, `popup`, `utilitarios` → `sgucard-modal`;
    - `pep` → `sgucard-pep`;
    - `laudos` → `sgucard-laudos`;
    - `camara` → `sgucard-camara`;
    - `biometria` → `sgucard-biometria`;
    - `atendimentos`, `token` → `sgucard-atendimentos`;
    - `relatorios` → `sgucard-relatorios`;
    - `src/br` → `sgucard-kernel`.
  - Mapa completo: `Resources/deploy/linux-mklinks.sh`.
- **autsc2:** o WAR compila `src` e `ccs/CCSBuild/src` direto. Rode na raiz `mvn -q -Dfrontend.maven.skip=true -DskipTests compile`.
- **SOB:** o WAR compila `src/main/java` e, pelo `build-helper`, `WebContent/CCSBuild/src`. Rode na raiz `mvn -q -DskipTests compile`.
- **JS:** `node --check <arquivo>.js` confere a sintaxe. Para lógica nova de evento, simule o fluxo num script Node pequeno antes de concluir, como foi feito para validar a correção do laço no piloto.
- **Fica como pendência:** abrir a tela `.do` no navegador e testar o fluxo.

## servicos-online-benef (SOB)

A mecânica é a mesma: ações Dynamix `OneLine`/`MultiLine`, `Custom Code` e os marcadores `//Event`. Mudam os caminhos:

| O quê | cmagnet e autsc2 | SOB |
|---|---|---|
| Páginas (`.ccp` e `.html`) | `ccs/<área>/` | `WebContent/<área>/` (41 `.ccp`: `agenda`, `guias`, `laudos`, `sistema`, `usuario` e a raiz) |
| Projeto do Studio | `ccs/CMagnet.ccs` e `ccs/autsc.ccs` | `WebContent/ServicosOnlineBenef.ccs` |
| Java gerado | `ccs/CCSBuild/src/<área>/<página>/` | `WebContent/CCSBuild/src/<área>/<página>/` |
| Classes à mão | `src/br/com/dynamix/ccs/<produto>/<área>/` | `src/main/java/br/com/dynamix/framework/servicos/online/<assunto>/` (`parametros`, `laudos`, `autenticacao`, `impressao`…) |
| Template do `web.xml` | `ccs/CCSBuild/config/web.xml` | `WebContent/CCSBuild/config/web22.xml` e `web23.xml` |

- **Mudança estrutural de referência:** o `fc31e83` (S20ATDDES-28055, fase de autorização). O dev mudou no Studio o `.ccs`, três `.ccp` e três `.html`, e a regeneração mudou 22 arquivos Java juntos. É o tamanho típico de uma mudança de estrutura: prefira que o dev a faça no Studio.
- **Liquibase:** as pastas `src/main/resources/liquibase/SOB-*` continuam no repositório, mas o listener saiu do `web.xml` em 2026-09-22 (`9611dfd`). Elas não rodam mais. Nunca crie changeset no SOB: banco é no central (`dev-atd-liquibase`).

## Regras

| ID | Nível | Regra | Como verificar |
|---|---|---|---|
| ATD-CCH-001 | MUST | Código de ação `OneLine`/`MultiLine` alterado no `.ccp` e no Java gerado com o mesmo conteúdo; se os dois já divergiam, pendência em vez de escolher um lado | diff dos dois arquivos |
| ATD-CCH-002 | MUST | `Custom Code` alterado só dentro do bloco do Java gerado, com os marcadores intactos | diff |
| ATD-CCH-003 | MUST | Toda entrega devolve a pendência de regenerar no Studio e conferir o diff, incluindo o `web.xml`; nunca afirme que regenerou | resumo |
| ATD-CCH-004 | MUST | SQL com parâmetro, nunca valor concatenado | diff |
| ATD-CCH-005 | MUST | Encoding de cada arquivo preservado | diff sem `Ã`/`�` novo |
| ATD-CCH-006 | SHOULD | Regra nova na classe escrita à mão, não no Java gerado | diff |

## Fora deste papel

- **Migração da tela para Vue:** é planejada pela `dev-atd-codecharge-migration` e feita pelos papéis `dev-backend` e `dev-frontend`.
- **Banco e menu** (inclusive apontar o menu para uma tela Vue): `dev-atd-liquibase`, no cmagnet-util.
- **Mapa dos repositórios:** `dev-atd-ecossistema`.

## NUNCA

- Nunca tente abrir o CodeCharge Studio ou regenerar por script.
- Nunca troque o encoding de um arquivo para "consertar" acento.
- Nunca copie credencial de conexão do `.ccs` ou de arquivo de ambiente.
