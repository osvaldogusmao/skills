---
name: dev-atd-proc-tarefas-backend
description: "Backend do proc-tarefas, o motor de tarefas batch do ATD: um WAR com Quartz 1.7.3 e um job só, que lê a lista de tarefas de um XML externo (XStream) e executa cada classe ITarefa ou ITarefaTemporizada com transação própria, com cron e propriedades em arquivos do servidor. Cobre a estrutura das tarefas (autsc, cmagnet, cargalogin, procxml), a receita de tarefa nova com a pendência de configuração do servidor, o envio de SMS e e-mail pelos senders do cmagnet-sms, acesso a dados com dynamix-dbutil e WorkSql, Java 1.6, encoding e a validação sem suíte de testes. Use para criar ou alterar uma tarefa no proc-tarefas. NÃO cobre os jobs do proc-guias (use dev-atd-proc-guias-backend), que têm agenda por parâmetro no banco e outro modelo de job."
---

# proc-tarefas (motor de tarefas batch)

O proc-tarefas é um WAR com Quartz 1.7.3 e **um job só** (`JobProcessarTarefas`). A cada execução, o job lê uma lista de tarefas de um XML **fora do WAR** e roda cada uma, na ordem, com transação própria. Não confunda com o proc-guias: lá cada job é um arquivo Quartz e a agenda vem do banco (`CM_PROC_PARAM`); aqui a agenda e a lista de tarefas vêm de arquivos do servidor. O `readme.md` só descreve os scripts de pipeline (é o mesmo do capa).

## Como o job roda

1. `quartz.properties` carrega `jobs/jobsTarefas.xml` pelo plugin `JobInit`. O cron do XML (`0 */1 * ? * *`) é só o padrão.
2. `JobProperties.getConfig()` busca por JNDI o caminho do arquivo de configuração do servidor e o lê a cada execução. As chaves são:
   - `jndi-name` (conexão) e `jndi-transactional` (fora do modelo; só o valor `false` desliga a transação);
   - `proc.tarefas.executar` (liga e desliga tudo);
   - `proc.tarefas.lista.tarefas` (caminho do XML de tarefas);
   - `proc.tarefas.properties.tarefas` (pasta das properties de cada tarefa);
   - `lock-file-do` e `lock-file-path` (arquivo de trava);
   - `proc-tarefas-cron-expression` (o cron real; o `RescheduleJob` reagenda o job com ele).
3. `TarefasFactory` lê o XML de tarefas com XStream (`<lista-tarefas>` com `<TarefaBean>`: `idTarefa`, `nmTarefa`, `nmClasseImpl`, `nmProperties`, `nrOrdem`, `executar`).
4. Para cada tarefa com `executar = true`, o job:
   - instancia `nmClasseImpl` por reflexão (construtor público sem argumentos);
   - passa a conexão, o `TarefaBean` e a pasta de properties;
   - chama `init()`, que carrega `nmProperties` em `getParametros()`, depois `run()` e `finalizer()`;
   - faz `commit` depois de cada tarefa e `rollback` se ela lançar exceção.

O `TarefaBeanComparator` não ordena de verdade (nunca devolve negativo): na prática, a ordem é a do arquivo. Mantenha `nrOrdem` coerente com a posição e não "conserte" o comparador de passagem.

Os modelos versionados ficam em `Resources/proc-tarefas/`: `config.properties`, `listaTarefas.xml` e `props/<produto>/<Tarefa>.properties`. O que roda é a cópia de cada servidor, que o repositório não controla.

## Tarefas

| Pasta (`src/main/java/br/com/unimedsc/proctarefas/tarefas/impl/`) | O que tem |
|---|---|
| `autsc/` | `AuditoriaPendenciasImpl` (avisos por e-mail e SMS das pendências de autorização), avisos de agenda e os senders de SMS em `sms/` |
| `cmagnet/` | cargas e populates (data de atendimento, favoritos, CID, medicamentos) |
| `cargalogin/` | carga de login e envio de e-mail de acesso |
| `procxml/` | envio de e-mail do processamento de XML |

- `ITarefa` (abstrata, `implements Runnable`): `newInstance()`, `run()`, `getConnection()`, `getParametros()` (as properties da tarefa; `get(chave, padrão)` aceita valor padrão) e `getTarefaBean()`.
- `ITarefaTemporizada` estende `ITarefa` e acrescenta a janela de execução: `listaDias()` (dias da semana separados por `;`, 1 = domingo), `horaInicial()` e `horaFinal()` (`HH:mm`), e `executarPopulate()` para saber se está dentro dela.

## Receita: tarefa nova

1. Classe em `tarefas/impl/<produto>/<Nome>Impl.java`, estendendo `ITarefa` (ou `ITarefaTemporizada`, se tiver janela de horário), com construtor público sem argumentos, `newInstance()` e `run()`. Lógica grande vai numa classe auxiliar ao lado (ex.: o par `CargaDataAtendimentoImpl` + `PopulateCargaDataAtendimentoTable`, do `f376bb7`).
2. Toda configuração da tarefa vem de `getParametros().get("<produto>.<tarefa>.<chave>")`, nunca fixa no código.
3. Entrada nova no modelo `Resources/proc-tarefas/listaTarefas.xml`, no formato das vizinhas (`nmProperties` relativo à pasta de properties, ex.: `/cmagnet/Favorito.properties`), e o arquivo modelo `Resources/proc-tarefas/props/<produto>/<Nome>.properties` com as chaves e valores de exemplo, **sem** senha nem endereço real.
4. **Pendência para o time:** incluir a entrada no `listaTarefas.xml` e o `.properties` em cada servidor. Sem isso, a tarefa nunca roda.

## Tratamento de erro e transação

- `run()` não pode lançar exceção checada. Para a tarefa voltar atrás, deixe escapar uma `RuntimeException`: o job faz `rollback` e segue para a próxima tarefa.
- Exceção capturada e só logada dentro do `run()` faz o job comitar o que já foi gravado. É o padrão de quem processa item a item (`AuditoriaPendenciasImpl` loga e segue para o próximo); decida no plano qual comportamento a tarefa precisa.
- A tarefa usa a conexão de `getConnection()` e nunca a fecha nem faz `commit`; quem controla é o job.

## Acesso a dados e integrações

- SQL com `DynaDBUtil.parametrizedPs`/`createPs` e `doFinallyClose`, ou com o `WorkSql` local (`executeParametrized`, `getParametrizedRows`). Parâmetro sempre amarrado.
- SMS: os senders ficam em `tarefas/impl/autsc/sms/`, escolhidos por `TipoSender` num `*Creator` (`ICreator`), sobre as classes do módulo `cmagnet-sms` do cmagnet-util. Um provedor novo começa no cmagnet-util. No caso do Zenvia (S20ATDDES-26407), foi o `894269f3a` no cmagnet-sms e depois o `fa79d87` aqui, com o sender e o `case` no creator.
- E-mail: `DynaMailer` e `EmailAbstractBean`, com servidor e credenciais vindos das properties da tarefa.
- Depende de `cmagnet-commons`, `cmagnet-parameter`, `cmagnet-agenda`, `cmagnet-sms` e `cmagnet-rda` (cmagnet-util) e de `fdsc-rules-util` e do cliente do servidor de regras.

## Stack e encoding

- **Java 1.6 de sintaxe e de API** (`fdsc-pom`): nada de lambda, `<>`, try-with-resources, multi-catch, `switch` de String, `java.nio.file` ou `java.time`.
- Fontes em ISO-8859-1. Um arquivo já tem `�` (`JobProcessarTarefas.java`): não corrija de passagem. Preserve o encoding e confira o diff atrás de `�` e `Ã`.

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

- Não há teste automatizado. `mvn -q package` compila e empacota o WAR. O único arquivo em `src/test/java` é o `MainTeste`, que o surefire não executa.
- Os artefatos de cima (cmagnet-util, fdsc-rules) precisam estar instalados ou publicados; se a resolução falhar, reporte.
- Confira à mão que a entrada nova do `listaTarefas.xml` é XML bem-formado e que `nmClasseImpl` é o nome completo da classe criada.
- **Fica como pendência:** a configuração em cada servidor e a primeira execução acompanhada no log.

## Regras

| ID | Nível | Regra | Como verificar |
|---|---|---|---|
| ATD-PTR-001 | MUST | Tarefa nova estende `ITarefa`/`ITarefaTemporizada`, com construtor público sem argumentos | diff |
| ATD-PTR-002 | MUST | Configuração da tarefa só por `getParametros()`; modelos em `Resources/proc-tarefas/` sem segredo | diff |
| ATD-PTR-003 | MUST | Tarefa não fecha nem comita a conexão recebida; o comportamento em erro (rollback ou seguir) está decidido no plano | diff, plano |
| ATD-PTR-004 | MUST | Java 1.6 de sintaxe e de API | diff |
| ATD-PTR-005 | MUST | SQL com parâmetro amarrado | diff |
| ATD-PTR-006 | MUST | Configuração do servidor (lista de tarefas e properties) devolvida como pendência | resumo |
| ATD-PTR-007 | MUST | Encoding preservado, sem `�` novo | diff |

## Fora deste repositório

- Provedor de SMS, cliente SOAP ou domínio compartilhado → cmagnet-util primeiro, com `install` local.
- Tabela ou coluna nova → banco central (`dev-atd-liquibase`).
- Mapa completo: `dev-atd-ecossistema`.

## NUNCA

- Nunca rode o `MainTeste` (`src/test/java/.../teste/MainTeste.java`): ele conecta num banco real com credenciais no código.
- Nunca copie para código, plano ou relatório senhas, tokens ou endereços reais de properties de tarefa, de `Resources/deploy` ou do `settings.xml`.
- Nunca rode `mvn deploy` nem o perfil `-Pdocker`.
- Nunca rode `scripts/node` nem `scripts/sh`: chamam as APIs reais do Jira e do GitLab e fazem `git config --global` e `branch -D`.
- Nunca troque versão de POM à mão.
