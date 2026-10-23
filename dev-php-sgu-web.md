---
name: dev-php-sgu-web
description: Convencoes de desenvolvimento do SGU Web - PHP 5.6 legado (Apache) com Oracle (oci8/ADOdb), Smarty e JS/jQuery. Use ao implementar ou revisar telas, consultas SQL e JS nas pastas sys/sgu-web, sys/cont_benef e hb_lib.
---

# SGU Portal do Prestador - convencoes de desenvolvimento

## Stack (nao assuma nada mais novo)

- **PHP 5.6** (imagem `php:5.6-apache`). Proibido: `??`, type hints escalares, return types, `[...]` com spread, `random_int`, closures com `fn`, namespaces modernos, `declare(strict_types)`. Short array `[]` e `::class` sao OK.
- **Oracle** via `oci8` + ADOdb (`adodb5`) e as classes proprias de `hb_lib/classes` (`hb_oci8.inc.php`, `hb_db.inc.php`). Compatibilidade de SQL/PLSQL: **11g como piso**.
- **Smarty** para templates (`$smarty->assign(...)`), **JS/jQuery** puro em `js/` ao lado de cada tela. Sem build, sem npm, sem framework SPA.
- Rodando em Docker (`Dockerfile`, `docker-compose.yaml`). Variaveis de ambiente via envplate (`ep`).

## Estrutura

- `hb_lib/` - biblioteca compartilhada (conexao, sessao, seguranca, template, PDF, e-mail). **Alteracao aqui afeta todas as telas**: so mexa se o plano pedir explicitamente.
- `sys/sgu-web/` - telas do portal (`matmed*.php`, `sguweb_*.php`, `hb_perfil.php`...). Uma tela = um `.php` + um `.js` + um template Smarty.
- `sys/cont_benef/` - modulo de contas/beneficiarios, com `dbconfig.php` e `adodb5` proprios.
- Bibliotecas de terceiros (`adodb5`, `jquery*.js`, `DatePicker`) **nao se editam**.

## Padrao de uma tela (`sys/sgu-web/*.php`)

1. `require ('lib/hb_config.php');`
2. `$pag_atual = '<nome_da_tela>'; valida_perfil($pag_atual);` - nunca pule a validacao de perfil.
3. Monta dados com `$DB->Execute($sql)` / `$DB->GetRow($sql)`, itera com `NumRows()`, `field("col")`, `Next()`.
4. Entrega ao template com `$smarty->assign(...)`, incluindo `INCLUDES` (`addinc_new('./js/<tela>.js?v='.$rand, 'js')`), `TITULO` e `LEGENDA`.
5. Mantenha o comentario de versao do topo (`/* Versao: NNN */`) se a tela tiver um, incrementando-o.

Siga o estilo do arquivo que esta editando (indentacao, nomes em portugues/snake_case, SQL em maiusculas). Nao reformate o arquivo inteiro: diff pequeno e focado.

## SQL / Oracle

- **Sempre bind variables** para qualquer valor vindo de `$_GET/$_POST/$_REQUEST/sessao`. Nunca concatene entrada do usuario em SQL. Se o trecho legado concatena, nao replique o padrao em codigo novo.
- Valores inteiros vindos de entrada: valide/converta (`(int)`) antes de usar.
- Compatibilidade 11g: sem `FETCH FIRST`, sem `OFFSET`, sem identity columns, sem `LISTAGG ... ON OVERFLOW`; paginacao com `ROWNUM`.
- Prefira views/funcoes existentes (`V_*`, `f_busca_valor_param_2(...)`) a recriar regra de negocio na tela.
- DDL e alteracao de objetos de banco ficam com o papel de PL/SQL e precisam estar no plano aprovado. Nada de `DROP`/`TRUNCATE`/`DELETE` sem `WHERE`.
- Colunas de data inativas usam o sentinela `01010001` (`TO_CHAR(col,'DDMMYYYY') = '01010001'`); preserve esse padrao.

## Seguranca (obrigatorio)

- **Nunca grave credenciais, strings de conexao ou chaves em codigo.** Hoje existem credenciais legadas em `sys/cont_benef/dbconfig.php`: nao copie, nao exiba em logs/saida e, se tocar no arquivo, sinalize no resumo que devem migrar para variavel de ambiente.
- Escape toda saida em HTML (`htmlspecialchars($v, ENT_QUOTES, 'UTF-8')` ou o escape do Smarty) para evitar XSS.
- Respeite os headers definidos em `hb_lib/hb_init.php` (CSP, X-Frame-Options, etc.); nao os afrouxe.
- Upload/download de arquivo: valide extensao e tamanho, nunca use nome vindo do cliente como caminho.
- Nao logue dados de beneficiario/prestador (CPF, carteirinha, dados clinicos).

## JavaScript

- Arquivo `js/<tela>.js` ao lado da tela, jQuery ja carregado; sem novas dependencias externas (a CSP so permite `self` e `cdn.datatables.net`).
- Sem ES modules e sem sintaxe que navegadores corporativos antigos nao suportem; na duvida, ES5.
- Chamadas AJAX tratam erro e nao assumem resposta bem-sucedida.

## Encoding e texto

- Saida em UTF-8 (`Content-Type: text/html; charset=utf-8`). Preserve a codificacao original do arquivo ao editar (verifique antes de salvar) e nao introduza BOM.
- Textos de interface em portugues do Brasil.

## Verificacao

O projeto nao tem suite de testes automatizados nem lint configurados. Portanto:

1. Rode `php -l <arquivo>` em todo `.php` alterado (idealmente na imagem PHP 5.6: `docker compose run --rm <servico> php -l <arquivo>`).
2. Se o ticket pedir, suba o ambiente com `docker compose up` e valide a tela no navegador.
3. No resumo final, declare explicitamente o que foi e o que **nao** foi verificado.
