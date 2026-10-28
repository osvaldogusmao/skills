---
name: dev-atd-proc-guias-backend
description: "Backend do proc-guias, o processamento batch de guias do ATD (WAR proc-job com Quartz 1.7.3 e 13 módulos: importação/validação de XML TISS, precificação, auditoria, validação de conteúdo, câmara, limpeza, WSD, webin, autsc, laudos, ActiveMQ): job novo, agenda de job (parâmetro CM_PROC_PARAM, não código), passo de processamento com transação manual por guia e chamada ao servidor de regras, acesso a dados com dynamix-dbutil, integrações e validação sem suíte de testes. Use para criar ou alterar código no proc-guias. NÃO cobre proc-tarefas (outro repositório) nem o servidor de regras (use dev-atd-fdsc-rules-backend)."
---

# proc-guias (processamento batch de guias)

`proc-job` é o WAR (Quartz 1.7.3, `RAMJobStore`, 3 threads) que orquestra os jobs; os outros 13 módulos são jars. Depende de 9 módulos do cmagnet-util, do `fdsc-rules-util`/`dynamix-rules` (servidor de regras) e do `cmagnet-laudos`; o módulo `proc-remoteutil` é consumido pelo autsc2 e pelo cmagnet.

## Módulos

| Módulo | O que faz |
|---|---|
| `proc-job` (WAR) | Quartz, jobs e orquestração (`JobProcessaGuia`) |
| `proc-xml` | importação e validação de XML TISS com Smooks 1.3 (uma configuração por versão TISS em `src/main/resources/br/com/unimedsc/rda/xml/tiss/smooks/config/v*/`) |
| `proc-precificacao` | precificação de guias (`PrecificacaoGuias`, sobre o `AbstractPrecificadorGuias` do `cmagnet-price`) |
| `proc-auditoria` | auditoria de guias (`AuditorGuias`) |
| `proc-validcontent` | validação de conteúdo (`ValidarConteudoGuias`) |
| `proc-camara` | câmara de compensação (glosas, impostos, notas, demonstrativos) |
| `proc-cleaner` | ações sobre as guias já processadas (`ICleanerAction`/`EnumCleanerAction`: glosa automática, cancelamento, entrega, fluxo, protocolos…) |
| `proc-wsd-transacoes` | importação de arquivos PTU A1100 |
| `proc-autsc`, `proc-webin`, `proc-laudos` | solicitações AutSC (desbloqueio, integrações pendentes, PTU inconsistente), auditoria WebIN e importação de laudos |
| `proc-broker` | ActiveMQ |
| `proc-remoteutil` | consultas remotas (`*Finder`) — **API do autsc2 e do cmagnet** |
| `proc-util` | `DataBaseUtil` e utilitários |

## Como os jobs rodam

- `proc-job/src/main/resources/quartz.properties` carrega pelo plugin `br.com.unimedsc.job.JobInit` os arquivos listados em `org.quartz.plugin.jobInitializer.fileNames` (`jobs/*.xml`).
- **A agenda real vem dos parâmetros**: cada constante de `JobsEnum` aponta para um parâmetro de cron (ex.: `proc.guias.cron.expression`) e um de ativação. Na subida, `JobInit.start()` reagenda cada job com o cron do parâmetro; a cada execução, `JobConfiguration.configureJob()` recarrega os parâmetros, pula se o job estiver inativo ou já rodando e, depois de rodar, reagenda se o cron mudou. O `<cron-expression>` do XML é só o padrão.
- Por isso **mudar a agenda de um job é mudar parâmetro, não código** — no plano, é pendência para o time (como o parâmetro é alterado no ambiente é decisão deles).
- O parâmetro `configuracoes` (lista separada por vírgula) faz o job rodar `start()` uma vez por configuração; cada execução lê `<configuracao>.<parametro>` (`ProcParamUtilLoader`) e, se não houver, o `<parametro>` sem prefixo.
- `JobProcessaGuia.processarGuias()` liga cada etapa do job principal por parâmetro (importar, precificar, validar conteúdo, WSD, webin, autsc…): etapa nova no fluxo principal = parâmetro novo + o `if` correspondente.

## Receita: job novo (5 arquivos — ex.: commit `89ca9a71`)

Modelo: `proc-job/…/job/limpeza/JobLimparProtocoloAtendimentoLogs.java`.

1. Constante em `JobsEnum`: `NOME("<nome>-trigger", "<Grupo>", "<parametro.do.cron>", "<nome>-job", "<parametro de ativacao>")`.
2. `<Nome>Config extends JobConfiguration implements Job`: construtor `super(JobsEnum.NOME)`; `execute(context)` chama `configureJob(context)`; `start(ProcParamUtilLoader p)` faz `new <Nome>(p).execute()`.
3. `<Nome> extends JobProcessor`: construtor recebendo `ProcParamUtilLoader`; `execute()` com a conexão de `DataBaseUtil.getConnection(JobProperties.JNDI_NAME_DATA_BASE)`, `setAutoCommit(false)`, `commit()` no sucesso e `rollback()` no erro, com log por `DynaLog`.
4. `proc-job/src/main/resources/jobs/job<Nome>.xml` no formato Quartz 1.5 dos vizinhos: `job-detail` (nome, grupo, descrição, `job-class` = a classe `Config`) e `trigger/cron` com os mesmos nomes do `JobsEnum` e o cron padrão.
5. O arquivo novo acrescentado em `org.quartz.plugin.jobInitializer.fileNames` do `quartz.properties`.

Os parâmetros de cron e de ativação precisam existir em `CM_PROC_PARAM` no ambiente — registre isso como pendência no resumo.

## Passo típico de processamento

Modelo: `proc-auditoria/…/processaguias/auditoria/AuditorGuias.java`.
1. Seleciona as guias a processar com `DynaDBUtil.parametrizedPs` (situação e limite vindos de parâmetro).
2. Para cada guia, transação própria: `setAutoCommit(false)`, limpa os erros anteriores e monta o `Rule` (JAXB do `fdsc-rules-util`) a partir dos SELECTs.
3. Chama o servidor de regras: `ExecuteRemote.execRule(route, TypeTask.<X>, UtilXML.getXML(rule))`.
4. Lê o `RuleTaskResultSet`, converte cada linha por `DomainActions` e grava os erros.
5. Recalcula a situação da guia e faz `commit` — ou `rollback` na falha, sem derrubar o lote.
Variantes (distribuidor, executor, MQ) escolhidas por enum a partir do parâmetro `tipo_execucao`.

## Acesso a dados e integrações

- `dynamix-dbutil`: `DynaDBUtil.parametrizedPs`/`createPs`/`doFinallyClose`/`recoverSequence` e `DynaSqlBuilder` (`append(sql, parametros…)` + `executeResultQuery`/`executeUpdate`). SQL Oracle em constantes, sempre com parâmetro amarrado.
- Conexão: `DataBaseUtil.getConnection("datacenter")` → JNDI `java:/datacenter`, **sem JTA**: a transação é manual. Método que recebe `Connection` nunca a fecha.
- Integrações: servidor de regras (SOAP, `ExecuteRemote`; URL e timeouts no `client.properties` externo, achado por JNDI); HTTP para o autsc2 e o cmagnet; ActiveMQ; sistema de arquivos (XML/zip TISS, arquivos WSD, anexos, arquivo de lock).
- Configuração (`JobProperties`), recarregada a cada execução, nesta ordem e com o último vencendo: a tabela `CM_PROC_PARAM`, o `proc-job.properties` do WAR (cujo `version` acompanha o POM e é trocado pelo script de versionamento) e o `client.properties` externo.

## Stack e encoding

- **Java 1.6 de sintaxe e de API**: nada de lambda, `<>`, try-with-resources, multi-catch, `switch` de String, `java.nio.file`, `java.time`, `Objects`, streams. O CI compila com JDK 8 e `-source 1.6`: sintaxe nova quebra o build, mas API nova passa — e quebra a regra do projeto.
- Fontes em **ISO-8859-1**. Literais SQL com acento são frequentes: um arquivo salvo em UTF-8 (`proc-validcontent/…/ValidarConteudoGuias.java` está assim) gera texto quebrado no banco, e o histórico já tem correções de "Ã£o" no `ImportadorArquivoTISS.java`. Preserve o encoding e procure `�`/`Ã` no diff.

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

## Validação (não há suíte automatizada)

- `mvn test` não roda nada: os arquivos de teste existentes são utilitários `main()` ou estão fora da raiz de teste.
- Valide compilando o WAR: `mvn -q -pl proc-job -am package`. Um módulo: `mvn -q -pl <modulo> -am install -DskipTests`.
- Os artefatos de cima (na `HOMOLOG`: cmagnet-util e fdsc-rules `26.00.0.0-SNAPSHOT`, dynamix-rules `6.01.H`, cmagnet-laudos `6.01.H-WILDFLY`) precisam estar instalados ou publicados; se a resolução falhar, reporte — não contorne.
- Se o chamado justificar um teste de regressão, proponha um mínimo em JUnit 4 (4.11 + Mockito 1.10.19, do `fdsc-pom`), sem banco, deixando claro que não há padrão de teste anterior no repositório. Descreva sempre o que foi validado e o que não foi.

## Regras

| ID | Nível | Regra |
|---|---|---|
| ATD-PRG-001 | MUST | Transação manual por unidade de trabalho (guia): `setAutoCommit(false)`, `commit` no sucesso, `rollback` na falha |
| ATD-PRG-002 | MUST | Método que recebe `Connection` não a fecha |
| ATD-PRG-003 | MUST | Java 1.6 de sintaxe e de API |
| ATD-PRG-004 | MUST | Job novo com os 5 arquivos; agenda e ativação por parâmetro em `CM_PROC_PARAM` |
| ATD-PRG-005 | MUST | SQL com parâmetro amarrado |
| ATD-PRG-006 | SHOULD | Mudança em `proc-remoteutil` aditiva (é API do autsc2 e do cmagnet) |

## Fora deste repositório

- Campo novo no pedido ou consulta nova no servidor de regras → fdsc-rules primeiro (`dev-atd-fdsc-rules-backend`).
- Módulo do cmagnet-util que precisa mudar → cmagnet-util primeiro, `install` local.
- Mudança em `proc-remoteutil` → avaliar autsc2 e cmagnet.
- Parâmetro novo em `CM_PROC_PARAM` → pendência para o time.
- Mapa completo: `dev-atd-ecossistema`.

## NUNCA

- Nunca rode os utilitários `main()` do repositório (ex.: `proc-xml/src/test/…/TISSConnection.java`): conectam em bancos reais e há credenciais gravadas no código.
- Nunca copie credenciais para código, plano ou relatório.
- Nunca troque versão de POM nem o `version` do `proc-job.properties` à mão.
