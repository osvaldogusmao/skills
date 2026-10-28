---
name: dev-atd-capa
description: "Portal CAPA do ATD: WAR com páginas JSP model-1 (scriptlets chamando classes Java) para consultar e imprimir a capa de uma autorização, prorrogar a senha e entrar no autsc2 ou no cmagnet. Cobre a estrutura (WebContent com consulta.jsp, imprimir.jsp, prorroga_senha.jsp e index.jsp; classes em br.com.unimedsc.capa e cópias de classes do autsc2), o acesso a dados só de leitura com o SqlBuilder local, os relatórios Jasper só compilados, a receita de campo novo na consulta (query + bean + JSP), o encoding por arquivo, o Liquibase desligado e a validação, em que JSP só falha em tempo de execução. Use para criar ou alterar página, consulta ou classe no capa. NÃO cobre o autsc2 nem o cmagnet, para onde o login e a prorrogação de senha encaminham."
---

# capa (portal de consulta de autorizações)

O capa é um WAR pequeno de páginas JSP no estilo model-1: cada página chama classes Java direto em scriptlets (`<% … %>`, `<%= … %>`), sem framework. Com ele se consulta e imprime a capa de uma autorização, se prorroga a senha e se entra no autsc2 ou no cmagnet. O `readme.md` só descreve os scripts de pipeline: esta skill e o código são a referência.

## Estrutura

| Onde | O que vai nele |
|---|---|
| `WebContent/*.jsp` | `index.jsp` (entrada e login), `consulta.jsp`, `imprimir.jsp`, `prorroga_senha.jsp`, `header*.jsp`, `footer.jsp` |
| `WebContent/scripts/`, `style/`, `images/` | JS (jQuery 3.7.1, jquery-migrate, inputmask, `capa.js`, `customDatepicker.js`), CSS e imagens |
| `WebContent/WEB-INF/web.xml` | servlets de inicialização (log, cliente do servidor de regras, singletons), `ServletController` em `*.do`, `AJAXVerifySystem` em `/servlet/verifysystem` e o taglib `components` (prefixo `dyna`, `tags/termo.tld` → `TermoTag`) |
| `src/main/java/br/com/unimedsc/capa/` | `consulta/` (`Consulta`, `ConsultaRetornoBean`, `ConsultaRetornoItemBean`), `connection/SqlBuilder`, `controller/` (`Parametros`, `ServletController`), `impressao/ImpressaoGuia`, `ajax/` |
| `src/main/java/br/com/unimedsc/rda/dao/` | DAOs de beneficiário e prestador por WS (`AutSC*WSDAOImpl`, `SCS3*WSDAOImpl`) |
| `src/main/java/br/com/dynamix/framework/` | `autsc/` (relatórios, domínios, modelo, segurança), `commons/` (`ParameterInit`…), `tags/` |
| `src/main/resources/.../autsc/reports/guias/jasper/` | relatórios Jasper **só compilados** (`.jasper`, sem `.jrxml`) |

**Cópias do autsc2:** 43 das 61 classes em `br/com/dynamix/framework/` têm o mesmo caminho e o mesmo nome no autsc2. São cópias que podem ter divergido. Não sincronize com o autsc2 nem copie de lá sem o plano dizer; se a correção também vale lá, registre o autsc2 no plano.

## Acesso a dados (só leitura)

- O capa só **lê** do banco: não há `INSERT`, `UPDATE` nem chamada de procedure no Java ou nos JSP. Mantenha assim. Escrita, quando existe, é delegada: a prorrogação de senha chama por `jQuery.getJSON` um servlet do autsc2, cuja URL vem do parâmetro `capa.prorroga.senha.autorizacao.servlet.path`.
- Consultas com o `SqlBuilder` local (`extends DynaSqlBuilder`). O padrão é o de `Consulta`:
  - `conn = sql.createConnection()`, com o JNDI de `Parametros` (`jndiName`);
  - `sql.append(" … = ? ", valor)`, com o valor sempre amarrado;
  - `IResultQuery rs = sql.executeResultQuery(conn)`;
  - `sql.closeConnection(conn)` no `finally`.
- As tabelas lidas são as do autsc2 (`AUTSC2_*`), no banco central. Parâmetros vêm de `ParameterInit` e dos `*ParametersUtil` do `cmagnet-parameter` (cmagnet-util).

## Receita: campo novo na consulta (`d359e9a`, S20ATDDES-27830, fase de autorização)

1. `Consulta`: um método `populate<Campo>(…, bean)` com a query no `SqlBuilder` (conexão própria, fechada no `finally`), chamado de `getValues`. Se o campo depende de parâmetro, a condição fica aqui.
2. `ConsultaRetornoBean`: o atributo, o getter e o setter (e um `isExibir<Campo>` quando a exibição é condicional).
3. `consulta.jsp`: o bloco no padrão das vizinhas (`<table class="MagnetoFormTABLERecord">`, `MagnetoFieldCaptionTD`, `MagnetoDataRecordTD`), dentro de `<% if (bean.isExibir<Campo>()) { %>` quando for condicional.

Mudança de layout ou texto de relatório: o repositório só tem o `.jasper` compilado, então isso é pendência. O fonte `.jrxml` precisa vir de fora, e o plano registra de onde.

## Segurança das páginas

- As páginas escrevem valores com `<%= … %>` sem escape de HTML, e o repositório não tem utilitário para isso. Numa saída nova de texto livre (vindo do banco ou do request), escape o HTML. Como criar o utilitário é decisão do plano.
- Valor do request só entra no SQL como parâmetro amarrado.

## Encoding

- Os JSP com acento (`consulta.jsp`, `header.jsp`, `imprimir.jsp`, `prorroga_senha.jsp`) e o `customDatepicker.js` estão em ISO-8859-1; as páginas não declaram `pageEncoding`, e o contêiner as lê como ISO-8859-1. O Java também é ISO-8859-1.
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

## Liquibase desligado

As pastas `Resources/database/liquibase/CAPA-*` continuam no repositório, mas o listener saiu do `web.xml` em 2026-09-22 (`c9117a0`), para evitar concorrência nas migrations do banco compartilhado. Elas não rodam mais. **Nunca crie changeset aqui**: banco é no central (`dev-atd-liquibase`).

## Validação

- `mvn -q package` compila o Java (Java 1.6 do `fdsc-pom`) e empacota o WAR. **Os JSP não são compilados no build**: erro de scriptlet ou de método inexistente só aparece quando a página abre.
- Confira à mão:
  - cada classe do `import` dos JSP alterados existe;
  - cada método chamado nos scriptlets existe com essa assinatura;
  - os `<% { %>` e `<% } %>` estão balanceados.
- Não há teste automatizado. **Fica como pendência:** abrir as páginas alteradas num ambiente.
- O `package.json` da raiz é dos scripts de pipeline, não do front: não rode `npm` nele.

## Regras

| ID | Nível | Regra | Como verificar |
|---|---|---|---|
| ATD-CAP-001 | MUST | O capa só lê do banco; escrita é delegada ao sistema dono do dado | diff |
| ATD-CAP-002 | MUST | SQL no `SqlBuilder` com parâmetro amarrado e conexão fechada no `finally` | diff |
| ATD-CAP-003 | MUST | Campo novo com query, bean e JSP juntos, no padrão do `d359e9a` | diff |
| ATD-CAP-004 | MUST | Imports e métodos usados pelos JSP alterados conferidos à mão | leitura |
| ATD-CAP-005 | MUST | Nenhum changeset Liquibase no capa | diff |
| ATD-CAP-006 | MUST | Encoding de cada arquivo preservado, sem `�` novo | diff |
| ATD-CAP-007 | SHOULD | Saída nova de texto livre com escape de HTML | diff |
| ATD-CAP-008 | SHOULD | Classe copiada do autsc2 alterada só com o plano dizendo se o autsc2 também muda | plano |

## Fora deste repositório

- Tabela, coluna ou parâmetro novo → banco central (`dev-atd-liquibase`).
- Prorrogação de senha, login ou classe original copiada → autsc2 (ou cmagnet, no login).
- Mapa completo: `dev-atd-ecossistema`.

## NUNCA

- Nunca copie para código, plano ou relatório o conteúdo de `Resources/oracle-capa-ds.xml`, `Resources/capa.properties`, `Resources/client.properties` ou do `settings.xml`.
- Nunca rode `mvn deploy` nem o perfil `-Pdocker`.
- Nunca rode `scripts/node` nem `scripts/sh`: chamam as APIs reais do Jira e do GitLab e fazem `git config --global` e `branch -D`.
- Nunca troque versão de POM à mão.
