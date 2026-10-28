---
name: dev-atd-persistence
description: "Como usar e como estender a biblioteca atd-persistence do ATD (persistência JDBC com anotações JPA como metadado, usada pelo sgucard-api/sgucard-kernel do cmagnet e pelo autsc2): facade AtdPersistence, DAOs de entidade (SqlReadOnlyEntityDAO/SqlMutableEntityDAO), EntityKey/EntityCriteria/EntityPatch, paginação, @Version, transação e conexão, e a receita para criar entidade + DAO + método na facade no próprio repositório da lib, com os testes e o JaCoCo. Use ao mapear tabela, criar DAO de CRUD simples ou mexer no atd-persistence. NÃO decide QUANDO usar a lib num produto (isso está na skill de backend do produto) e NÃO é Hibernate nem Spring Data JPA."
---

# atd-persistence

Biblioteca de persistência compartilhada do ATD (`br.com.atd:atd-persistence`, repositório `atd-persistence`). É um **executor JDBC** que usa as anotações JPA 2.2 só como metadado: não há Hibernate, `EntityManager` nem Spring Data JPA — só os tipos `Pageable`/`Sort`/`Page`/`Slice` do `spring-data-commons`. Consumidores: `sgucard-api` e `sgucard-kernel` (cmagnet) e autsc2.

Fonte de verdade: o `README.md` do repositório (≈2.100 linhas). Esta skill é o resumo para o dia a dia; para o detalhe, leia só a seção que a tarefa pedir (mapa abaixo). Se um doc de produto descrever outra coisa — os docs do CMAGNET ainda citam `@SqlTable`/`@SqlColumn`/`@SqlId`, que foram removidos —, vale o README e o código da lib; reporte a divergência.

## Mapa do README

| Tarefa | Seção |
|---|---|
| Configurar provider / obter DAO | "Configuracao no consumidor", "Instanciando e usando uma DAO" |
| Buscar, listar, paginar, projetar | "Find e Exists", "List", "Projecoes tipadas", "EntityKey" |
| Gravar | "Insert", "Update", "EntityInsert", "EntityPatch", "Delete", "Escritas em lote" |
| Concorrência | "Concorrencia otimista com `@Version`" |
| CLOB/BLOB | "Gravando CLOB/BLOB", "Lendo CLOB/BLOB grandes" |
| Transação | "Operacoes com Connection explicita" |
| Relacionamentos | "Relacionamentos JDBC explicitos" (e "Risco em relacionamentos de alta cardinalidade") |
| Entidade nova | "Levantando os metadados de uma tabela Oracle", "Criando uma nova entidade", "Chave composta e `@IdClass`", "Anotacoes interpretadas", "De/Para de tipos SQL para Java", "Criando uma nova DAO", "Validando uma nova entidade" |

## O contrato da lib

- **Transação é do chamador.** A lib nunca chama `setAutoCommit`, `commit` ou `rollback`. Para várias operações numa transação, abra a `Connection`, passe-a às sobrecargas que recebem `Connection` e faça commit/rollback no consumidor.
- **Conexão:** se a DAO abriu a conexão pelo `SqlConnectionProvider`, ela fecha; se recebeu uma `Connection` por parâmetro, quem fecha é o chamador.
- **Validação antes do SQL:** Bean Validation nas anotações da entidade (`SqlEntityValidationException`); erro de mapeamento aparece no **primeiro uso** do tipo (`IllegalArgumentException`) — por isso todo mapeamento novo tem teste de metadata.
- **Concorrência otimista:** campo `@Version` (Short/Integer/Long); conflito lança `SqlOptimisticLockException`.

## Usar no consumidor (cmagnet, autsc2)

- A facade já configurada com o provider do produto vem de `SqlUtil.getPersistence()` (`br.com.dynamix.framework.autsc.SqlUtil`, nos dois). Use o método dedicado (`persistence.parametroEntityDAO()`) ou, se ainda não existir, `persistence.dao(MinhaEntityDAO::new)`.
- `SqlReadOnlyEntityDAO`: `find` (por `EntityKey`, com `EntityFetch` ou projeção), `exists`, `count`, `findFirst`, `list`, `slice`, `page` (com `Pageable`/`Sort`).
- `SqlMutableEntityDAO` acrescenta: `insert`, `insertReturningKey`, `insertAndFind`, `update` (entidade ou `EntityKey` + `EntityPatch`), `updateWhere`, `delete` (opcionalmente com a versão esperada), `deleteWhere` e lote `insertAll`/`updateAll`/`deleteAll`/`deleteAllById` (lotes de 100, com log).
- Filtros: `EntityCriteria` (`equal`, `in`, `like` com escape, `between`, `isNull`, `anyOf`/`allOf`/`or`/`not`, `where(DynaSqlBuilder)` para o que o builder não cobre, `whereRelated` para filtrar por entidade relacionada).
- **Quando** usar a lib em vez de SQL próprio é regra do produto (está na skill de backend dele). A forma de instanciar e testar a DAO de feature também (no `sgucard-api`, o construtor público delega para `SqlUtil.getPersistence().<dao>()` e existe um construtor injetável para os testes).

## Estender a lib (no repositório atd-persistence)

Entidade/DAO nova para um consumidor é mudança **neste** repositório. Receita:

1. Levante os metadados da tabela no Oracle com as consultas `ALL_*` da seção "Levantando os metadados…" (colunas, PK, checks, FKs, triggers/sequences, índices). Nunca adivinhe tipo, nulabilidade ou ordem da PK.
2. `src/main/java/br/com/atd/persistence/mapping/<dominio>/<Nome>Entity.java`: `implements SqlMappedEntity<<Nome>Entity>`, Lombok `@FieldNameConstants @Getter @Setter @NoArgsConstructor`, `@Entity` + `@Table(name = …)`, `@Column` com nome, nulabilidade e tamanho/precisão iguais aos do banco, validações (`@NotNull`, `@Size`, `@Digits`) coerentes com a coluna e um `static key(...)` que monta o `EntityKey` com as constantes `Fields`. Chave composta: `<Nome>Id` + `@IdClass` + `@SqlIdOrder(n)` em cada `@Id`. Base comum de várias tabelas: `Abstract<Nome>Entity` / `@MappedSuperclass`.
3. Anotações aceitas: `@Entity`, `@Table`, `@Column`, `@Id`, `@IdClass`, `@GeneratedValue` (AUTO/IDENTITY, quando o banco gera a chave por trigger/default), `@Version`, `@Enumerated`, `@Convert`, `@ManyToOne`/`@OneToMany`/`@JoinColumn(s)`, `@MappedSuperclass`, `@Lob`, e as próprias `@SqlColumnOptions(selectable, hasDefault)` e `@SqlIdOrder`. Coluna com default do banco: `hasDefault = true`; coluna pesada (CLOB/BLOB) fora do select padrão: `selectable = false`.
4. `<Nome>EntityDAO.java` no mesmo pacote: `extends SqlMutableEntityDAO<<Nome>Entity>` (ou `SqlReadOnlyEntityDAO` para tabela só de leitura), um único construtor `(SqlConnectionProvider connectionProvider)` chamando `super(<Nome>Entity.class, connectionProvider)`.
5. Se os consumidores vão pegar a DAO pela facade (o caso comum), método em `AtdPersistence`: `public <Nome>EntityDAO <nome>EntityDAO() { return dao(<Nome>EntityDAO::new); }`. Sem método, o consumidor usa `persistence.dao(<Nome>EntityDAO::new)`.
6. Testes (JUnit 5 + Mockito, `Connection` mockada, sem banco): teste de metadata da entidade (tabela, colunas, PK, geração) em `src/test/java/br/com/atd/persistence/metadata/` — erro de anotação só aparece no primeiro uso, e o teste faz o build falhar antes; teste do SQL com `SqlEntityCommandBuilder` em `.../sql/` quando houver chave composta, coluna com default, campo não atualizável ou critério dinâmico; e, se criou o método na facade, a linha `assertDao(<Nome>Entity.class, persistence.<nome>EntityDAO())` no `AtdPersistenceTest`, como as DAOs que já estão lá.
7. `mvn -q verify` verde e, para o consumidor enxergar localmente, `mvn -q install`.

## Regras

| ID | Nível | Regra | Como verificar |
|---|---|---|---|
| ATD-PER-001 | MUST | A lib não importa nada de consumidor (`WorkSql`, exceções REST do `sgucard-api`, `Resource`, `Service`, sessão, permissão). Pode depender de `DynaSqlBuilder`/`IResultQuery` | imports do diff |
| ATD-PER-002 | MUST | Só anotações JPA + `@SqlColumnOptions`/`@SqlIdOrder`; nunca `@SqlTable`/`@SqlColumn`/`@SqlId` | compilação e teste de metadata |
| ATD-PER-003 | MUST | Nome de tabela, colunas e ordem da PK iguais aos do banco | consultas `ALL_*` do README |
| ATD-PER-004 | MUST | Entidade nova sempre com teste de metadata; teste de SQL nos casos sensíveis (chave composta, default, campo não atualizável, critério dinâmico); método novo na facade com `assertDao` | `mvn -q verify` |
| ATD-PER-005 | MUST | Transação no chamador; a lib nunca faz commit/rollback | revisar o diff |
| ATD-PER-006 | SHOULD | Relacionamento de alta cardinalidade carregado sob demanda (`EntityFetch`), nunca em lote cego | seção de risco do README |

## Testes, cobertura e build

- `mvn -q verify` roda os testes e o gate do JaCoCo: **≥ 90% de linha** no total, com `annotation`, `connection`, `exception`, `mapping` e `dao` fora da conta. Entidade e DAO novas não contam; o método novo na facade conta, e quem o cobre é o `assertDao` do `AtdPersistenceTest`.
- Java 1.8 e **UTF-8** (diferente da maioria dos repositórios do ATD).

## Versão e publicação

- `HOMOLOG` em `26.00.0.0-SNAPSHOT`; o CI faz `install` em `feature/`/`bugfix/` (nada publicado) e `deploy` nas demais branches.
- **Não está no `pipeline-batch-docker`**: precisa ser publicada à parte antes de autsc2 e cmagnet.
- O autsc2 segue a versão pela propriedade `version.cmagnet-pom` do `fdsc-pom`; o `sgucard-api` e o `sgucard-kernel` fixam a versão à mão (trocada por release).

## Fora deste repositório

- Consumidor (cmagnet/autsc2) precisando de entidade, DAO ou método novo na facade → no plano: primeiro o atd-persistence (`/dev-ticket` próprio, mesma branch `feature/SAT-I####`, `mvn -q install` local), depois o consumidor; o MR da lib entra na `HOMOLOG` antes. Mapa completo em `dev-atd-ecossistema`.

## NUNCA

- Nunca use Hibernate, `EntityManager`, repositório Spring Data JPA ou JPQL: a lib não é JPA em runtime.
- Nunca feche uma `Connection` recebida do chamador nem abra transação dentro da lib.
- Nunca mapeie "de cabeça": sem os metadados reais da tabela, pare e peça.
- Nunca desligue nem rebaixe o gate do JaCoCo para o build passar.
