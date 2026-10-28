---
name: dev-atd-ecossistema
description: "Mapa dos repositórios do ATD (sgu-atd: cmagnet, autsc2, cmagnet-util, fdsc-rules, atd-persistence, proc-guias e os demais): o que cada um contém, quem depende de quem, ordem de build e publicação, versões e branches, e onde cada tipo de mudança acontece (banco central e SAPIA-NI, menu, parâmetros, regras no Guvnor, entidades do atd-persistence, jobs). Use ao planejar ou analisar qualquer chamado num repositório do ATD, principalmente para descobrir se a mudança atravessa repositórios e em que ordem. NÃO traz convenções de código de um repositório — para isso use a skill dev-atd-* do repositório."
---

# Ecossistema de repositórios do ATD (sgu-atd)

Mapa para **planejar**: o que mora em cada repositório, quem depende de quem e onde cada tipo de mudança acontece. Convenções de código (encoding de cada área, registro REST, testes, padrões de DAO) ficam na skill de cada repositório. Tudo aqui foi conferido nos POMs, nos CIs e no histórico de merges dos repositórios em `gitlab.com/nexdom/produto/sgu-atd/`; se algo divergir do que você encontrar no repositório, o repositório vence — reporte a divergência em vez de seguir este mapa.

## Repositórios

| Repositório | O que é | Skill no kit |
|---|---|---|
| `fdsc-pom` | POM pai de quase todos: versões (`version.<projeto>-pom`), plugins, encoding ISO-8859-1, JUnit 4.11, Mockito 1.10.19 | — |
| `dynamix-project` | super-POM (registry de publicação) | — |
| `dynamix-rules` | framework do motor de regras (Drools 5): client, core, model, util, cache — versão `6.01.H`, publicado à parte | — |
| `fdsc-rules` | servidor de regras (`dynamix-rules-server`, SOAP) + Liquibase **SAPIA-NI** | `dev-atd-fdsc-rules-backend`, `dev-atd-liquibase` |
| `cmagnet-util` | 31 módulos compartilhados (libs, domínio JDBC, clientes SOAP, auth, busca) + `cmagnet-liquibase` (banco **central** e menus) | `dev-atd-cmagnet-util-backend`, `dev-atd-liquibase` |
| `atd-persistence` | persistência JDBC com metadados JPA, usada por cmagnet e autsc2 | `dev-atd-persistence` |
| `proc-guias` | processamento batch de guias (Quartz) | `dev-atd-proc-guias-backend` |
| `proc-tarefas` | motor de tarefas batch (Quartz, um job só; lista de tarefas e properties em arquivos do servidor) | `dev-atd-proc-tarefas-backend` |
| `cmagnet` | SGU-Card: `sgucard-api` (REST), `sgucard-frontend` (Vue 3 em iframe) e o legado CodeCharge | `dev-atd-cmagnet-backend`, `dev-atd-cmagnet-frontend`, `dev-atd-codecharge` |
| `autsc2` | autorizador: API REST, Vue 3 servido direto pelo WAR e o legado CodeCharge | `dev-atd-autsc2-backend`, `dev-atd-autsc2-frontend`, `dev-atd-codecharge` |
| `cmagnet-laudos` | WS de laudos (`wslaudos`) e biblioteca de laudos usada por autsc2, cmagnet, proc-guias e servicos-online-benef | `dev-atd-cmagnet-laudos-backend` |
| `capa` | portal JSP de consulta e impressão de autorizações (só leitura; lê as tabelas do autsc2) | `dev-atd-capa` |
| `servicos-online-benef` | serviços ao beneficiário (CodeCharge puro, em `WebContent/`) | `dev-atd-codecharge` |
| `wsptu` | web services SOAP do PTU (intercâmbio entre Unimeds), versões V60 a V100 | `dev-atd-wsptu-backend` |
| `wstiss3` | web services SOAP do TISS (ANS), processados por Smooks, sem DAO Java | `dev-atd-wstiss3-backend` |
| `app_web` | módulos externos em servlet/JSP (sem `fdsc-pom`) | — |
| `homecare-webapp` | SPA Vue 3 avulsa (nginx, `/homecare/`) dos prestadores de homecare; a API são os servlets legados `capturaExterna` do cmagnet | `dev-atd-homecare-frontend` |
| `config-atd`, `pipeline-batch-docker` | ambiente local do time (scripts, versionamento) e orquestrador de pipelines | — |

Migração de tela CodeCharge → Vue/REST (cmagnet e autsc2): skill `dev-atd-codecharge-migration`.

## Quem depende de quem

Ordem de build pelos POMs:

```
fdsc-pom → fdsc-rules → { cmagnet-util, atd-persistence }
        → { proc-guias, proc-tarefas, capa, servicos-online-benef, cmagnet-laudos, wsptu }
        → { autsc2, cmagnet }
(wstiss3 herda o fdsc-pom, mas não usa cmagnet-util nem fdsc-rules; app_web não usa o fdsc-pom;
 dynamix-project e dynamix-rules ficam por baixo, publicados à parte)
```

Do cmagnet-util, o wsptu usa só o `cmagnet-commons`, e o cmagnet-laudos usa `commons`, `parameter`, `rda` e `tiss`.

Artefatos que mais atravessam repositórios — mudar a assinatura de algo aqui é mudança em **vários** repositórios:

| Artefato (repositório de origem) | Quem consome |
|---|---|
| `fdsc-rules-util` (fdsc-rules): `TypeTask`, classes JAXB de `Rule`/`Query` | cmagnet-util (`rda`, `price`), proc-guias (10 módulos), autsc2, `sgucard-kernel` |
| `cmagnet-commons`, `cmagnet-parameter`, `cmagnet-rda` (cmagnet-util) | até 8 repositórios (commons), 6 (parameter, rda); autsc2 usa 21 módulos do cmagnet-util e o cmagnet 23 |
| `atd-persistence` | `sgucard-api`, `sgucard-kernel`, autsc2 |
| `proc-remoteutil` (proc-guias) | autsc2, cmagnet |
| `cmagnet-laudos-*` (fixados em `6.01.H`/`6.01.H-WILDFLY`) | autsc2, cmagnet, proc-guias, servicos-online-benef |
| `dynamix-rules` client/util/cache (`6.01.H`) | fdsc-rules, cmagnet-util, proc-guias, autsc2, cmagnet |

## Onde cada tipo de mudança acontece

| Mudança | Onde | Observação |
|---|---|---|
| Tabela, coluna, índice, parâmetro `CM_PARAM*` do schema principal (owner `DATACENTER`) | `cmagnet-util/cmagnet-liquibase`, central (`DEV-AA-MM-0-0`) | vale para o **autsc2 também**: ele não tem pastas Liquibase próprias e roda o central. Vale também para o capa e o servicos-online-benef, cujo Liquibase próprio foi desligado em 2026-09-22 (as pastas `CAPA-*`/`SOB-*` não rodam mais) |
| Menu (`CM_MENUS`, `AUTSC2_MENUS`, `CM_ITENS_ADMIN`) e permissões de menu (`CM_GRUPOS_FUNCOES`, `AUTSC2_GRUPOS_FUNCOES`) | changeset no central | essas tabelas só mudam por changeset (exceção conhecida: o PEP tem menu próprio em código, no `cmagnet-pep`) |
| Tabela do lado das regras (owner `UD976`) e espelho de `ALTER TABLE` em tabela monitorada | `fdsc-rules`, `SAPIA-NI-AA-MM-0-0` | o espelho fica na pasta do **mesmo mês** da mudança no central |
| Regra de negócio (DRL) | **Guvnor, fora do git** | no fdsc-rules o trabalho é expor dados às regras (fatos, atributos, consultas); criar/alterar a regra é pendência para o dev |
| Entidade/DAO de CRUD simples usada por cmagnet ou autsc2 | `atd-persistence` | o consumidor só compila depois do `mvn install` da lib |
| Procedure de transação PTU (`SP_PRCSSA_MSG_PTU_V*`) e procedures e views do WS TISS (`SP_WS_TISS_*`, `V_WSTISS_*`) | central (`procedure/`, `view/` e pastas `DEV-AA-MM-0-0`) | é onde costuma estar a regra de negócio do wsptu e do wstiss3; vem antes do WS. Algumas views antigas do wstiss3 não têm changeset |
| Biblioteca de laudos (validação, anexos, importação) | `cmagnet-laudos` | os consumidores fixam `version.cmagnet-laudos-pom` (`6.01.H-WILDFLY`) no `fdsc-pom`: a mudança só chega a eles com versão nova e bump da propriedade, decisão do time |
| Endpoint do homecare (`/servlet/capturaExterna/v1/*`) | `cmagnet`: servlets legados em `ajax/integracao/captura/`, mapeados nos dois `web.xml` | o homecare-webapp muda depois, no mesmo SAT-I e com o mesmo nome de branch |
| Job de processamento de guias | `proc-guias` | a **agenda** do job vem do parâmetro `CM_PROC_PARAM` (banco), não do código — combine com o time como alterá-la |
| Tarefa batch avulsa (avisos, cargas, envio de e-mail e SMS) | `proc-tarefas` | a lista de tarefas, as properties de cada uma e o cron ficam em arquivos de cada servidor: incluir a tarefa nova neles é pendência para o time |
| Domínio compartilhado, utilitário, cliente SOAP de terceiros | `cmagnet-util` | ver quem consome antes de mudar assinatura |
| Versão de qualquer projeto | `fdsc-pom` + script de versionamento | nunca à mão, nunca pelo agent |

## Mudança que atravessa repositórios

O `/dev-ticket` trabalha num repositório por vez — isso é proposital (ampliar o escopo é decisão humana). Quando o chamado precisa de mais de um:

1. No plano, em "Riscos/dúvidas", liste cada repositório afetado, o que muda nele e a **ordem** pelo grafo acima (base antes do consumidor; banco antes do app que depende dele no startup).
2. O dev roda `/dev-ticket <mesmo chamado>` em cada repositório, na ordem. O agente `nexdom-gitflow-atd` dá o mesmo nome de branch (`feature/SAT-I####` ou `bugfix/SAT-I####`) em todos.
3. Localmente, instale a base antes de compilar o consumidor: `mvn -q -pl <módulo> -am install` no repositório da base. O CI de `feature/`/`bugfix/` só faz `install` no cache do job — **nada é publicado** —, então o consumidor não enxerga a mudança da base por outro caminho.
4. O MR da base é mergeado na `HOMOLOG` primeiro; o CI do consumidor só passa depois que a base for publicada a partir da `HOMOLOG`.

Se a compilação do consumidor falhar por classe ou método ausente de um módulo base, confira a versão resolvida antes de mexer no código: `mvn -q help:evaluate -Dexpression=version.cmagnet-pom -DforceStdout` (troque a propriedade conforme o projeto).

## Versões e branches

- `HOMOLOG` é a linha de desenvolvimento e o destino dos MRs; versão `26.00.0.0-SNAPSHOT` (pai `fdsc-pom` `26.00.0.0-SNAPSHOT-CONFIG`).
- `DESENVOLVIMENTO-AA-MM-P-B` → versão `AA.MM.P.B-SNAPSHOT`; `RELEASE-AA-MM-P-B` → `AA.MM.P.B` (ex.: `RELEASE-26-09-0-1` → `26.09.0.1`). Quem cria essas branches e troca as versões é o script `config-atd/linux/gitlab-versionamento.sh` (lista de arquivos por projeto em `versionamento-atd-projetos.conf`).
- O mês da versão casa com as pastas Liquibase: `DEV-AA-MM-0-0` (central) = `SAPIA-NI-AA-MM-0-0` (fdsc-rules) = versão `AA.MM.0.0`.
- Chamado: `bugfix/SAT-I####` (Bug) ou `feature/SAT-I####` a partir da `HOMOLOG`, MR para a `HOMOLOG` — feito pelo agente `nexdom-gitflow-atd`. Levar uma correção também para uma `DESENVOLVIMENTO-*` (já aconteceu, ex.: `feature/SAT-I660-26-09`) é decisão manual do time.

## Build e publicação

- CI de cada repositório: `mvn -q --settings ~/.m2/settings.xml clean install -U` em `feature/`/`bugfix/`; `clean deploy -U` nas demais branches, para o registry Maven do GitLab (definido no `dynamix-project`).
- Exceções: no autsc2, o job de build só faz `install`, em qualquer branch, e o WAR vira imagem Docker em outro job. No cmagnet, o job de build faz `deploy` dos módulos `sgucard-*` em **qualquer** branch e depois o `install` do WAR. Nenhum outro repositório consome esses módulos. Mas quem compila o `sgucard-api` sem `-am` pega o `sgucard-kernel` do repositório Maven (o instalado localmente ou o SNAPSHOT publicado pela última branch que passou no CI), e não o código local.
- `pipeline-batch-docker` compila tudo em ordem nas branches `DESENVOLVIMENTO-*`: fdsc-pom → fdsc-rules → cmagnet-util → proc-guias e proc-tarefas → autsc2 e cmagnet → app_web, capa, cmagnet-laudos, servicos-online-benef, wsptu, wstiss3.

## Atenção (inconsistências conhecidas)

- `atd-persistence` não está no `pipeline-batch-docker`: precisa ser publicado à parte antes de autsc2 e cmagnet.
- Seis repositórios usam o pai `fdsc-pom:26.00.0.0-CONFIG`, definido em duas branches do fdsc-pom com valores diferentes — a versão resolvida do cmagnet-util varia conforme o que foi publicado por último (confira com o `help:evaluate` acima).
- `sgucard-api` e `sgucard-kernel` fixam a versão do `atd-persistence` à mão; o autsc2 segue a propriedade do `fdsc-pom`.
- `cmagnet-laudos` é compilado depois dos consumidores no pipeline, mas eles fixam versões `6.01.H*` — a versão 26.x dele não é consumida.

## Encoding (visão geral)

Java em **ISO-8859-1** em quase todos os repositórios (padrão do `fdsc-pom`). Em UTF-8: `sgucard-api`, `atd-persistence`, os frontends Vue, os docs e as pastas Liquibase recentes. O detalhe por área e o procedimento seguro de edição (validado no piloto: o Edit e o Write do Claude Code gravam sempre UTF-8) estão na skill de cada repositório. Arquivos com `�` ou "Ã£" já existem em alguns repositórios: não "corrija" de passagem — reporte.

## NUNCA

- Nunca altere outro repositório de dentro do `/dev-ticket` deste — liste a mudança no plano e siga o passo a passo acima.
- Nunca troque versão de POM à mão.
- Nunca aplique Liquibase em banco (`liquibase update`, `mvn liquibase:*`, `config-atd/liquibase/lib/launcher.bat`, `ExecuteLiquibaseFile_*.bat`): o changelog roda no startup do WAR de cada ambiente.
- Nunca rode `mvn deploy` nem o perfil Maven `docker` (`-Pdocker`), que todo WAR do ATD tem: o primeiro publica no registry Maven e o segundo faz build e push da imagem com as credenciais do `settings.xml`. Publicar é papel do CI.
- Nunca rode `scripts/node` nem `scripts/sh`, que existem em quase todos os repositórios: chamam as APIs reais do Jira e do GitLab e fazem `git config --global` e `branch -D`.
- Nunca rode os `main()` de teste manual que existem em alguns repositórios: eles chamam serviços e bancos reais.
- Nunca copie para código, plano, relatório ou comentário as credenciais que existem em arquivos de alguns repositórios.
