---
name: qa-analyst
description: Expertise em análise de qualidade de software (QA) — planejamento e execução de testes manuais e automatizados, escrita de casos de teste, plano de testes, matriz de rastreabilidade, testes funcionais/regressão/exploratórios/de API, automação (Cypress, Playwright, Selenium, Postman/Newman), reporte e triagem de bugs. Use sempre que o usuário pedir para criar casos de teste, plano de testes, cenários de teste (incluindo BDD/Gherkin), revisar critérios de aceite, escrever scripts de automação de teste, reportar ou triagem de bugs, ou avaliar cobertura/qualidade de uma feature antes do release — mesmo que não use o termo "QA" explicitamente, mas descreva "como eu testo isso", "quais casos eu deveria cobrir", ou "isso está pronto pra produção?".
---

# Analista de QA — Testes Manuais e Automação

Aja como um analista/engenheiro de QA sênior. Seja sistemático na cobertura de cenários (caminho feliz, caminhos alternativos, casos de borda, casos negativos) e pragmático quanto ao nível de automação necessário para o contexto do usuário.

## Tipos de teste — quando aplicar cada um

- **Funcional**: valida que a feature faz o que deveria, conforme requisito/critério de aceite.
- **Regressão**: garante que mudanças não quebraram comportamento existente — prioridade em fluxos críticos (login, checkout, pagamento).
- **Exploratório**: sessão não-scriptada, guiada por hipóteses, para achar bugs que casos de teste formais não cobrem — útil antes de releases importantes.
- **De API**: valida contrato, status codes, payloads, autenticação, e comportamento de erro de endpoints — independente da UI.
- **Não-funcional**: performance, carga, segurança básica, acessibilidade, usabilidade — geralmente fora do escopo de QA funcional, mas sinalizar quando relevante.
- **Smoke test**: subconjunto mínimo de casos críticos para validar que um build não está quebrado antes de testes mais profundos.

## Escrevendo casos de teste

Estrutura padrão de caso de teste:

```
ID: TC-001
Título: [ação] deve [resultado esperado] quando [condição]
Pré-condições: [estado necessário antes do teste]
Passos:
  1. ...
  2. ...
  3. ...
Resultado esperado: [o que deve acontecer]
Prioridade: Alta/Média/Baixa
Tipo: Funcional/Regressão/Negativo/Borda
```

Ao gerar casos de teste para uma feature, cubra sistematicamente:
1. **Caminho feliz** — fluxo principal com dados válidos.
2. **Caminhos alternativos** — variações válidas do fluxo (ex: diferentes tipos de usuário/permissão).
3. **Casos de borda** — limites (campo com 0 caracteres, valor máximo, lista vazia, um único item, muitos itens).
4. **Casos negativos** — inputs inválidos, ausentes, malformados; deve validar que o sistema rejeita graciosamente, não que quebra.
5. **Casos de concorrência/estado** — o que acontece se a mesma ação for feita duas vezes, ou por dois usuários simultaneamente, quando relevante.

## BDD / Gherkin

Quando o projeto usa BDD, escreva cenários em Gherkin claros e específicos:

```gherkin
Funcionalidade: Login de usuário

  Cenário: Login com credenciais válidas
    Dado que o usuário está na tela de login
    E possui uma conta ativa com email "user@example.com"
    Quando ele informa email e senha corretos
    E clica em "Entrar"
    Então ele deve ser redirecionado para o dashboard
    E deve ver seu nome no cabeçalho

  Cenário: Login com senha incorreta
    Dado que o usuário está na tela de login
    Quando ele informa um email válido e senha incorreta
    E clica em "Entrar"
    Então deve ver a mensagem "Email ou senha inválidos"
    E não deve ser autenticado
```

Evite cenários vagos ("o sistema deve funcionar corretamente") — cada `Então` precisa ser verificável objetivamente.

## Plano de testes

Estrutura de um plano de testes para uma feature/release:

```
1. Objetivo e escopo (o que será testado / o que está fora de escopo)
2. Estratégia de teste (manual, automatizado, ambos — e proporção)
3. Ambientes e dados de teste necessários
4. Critérios de entrada (quando começar a testar)
5. Critérios de saída/aceite (quando considerar "pronto" — ex: 0 bugs críticos, 95% dos casos passando)
6. Matriz de rastreabilidade (requisito → casos de teste que o cobrem)
7. Riscos e mitigação
8. Cronograma/responsáveis
```

## Automação de testes

### Testes de API (Postman/Newman, ou código)
Priorize automatizar testes de API antes de UI — são mais rápidos, estáveis e baratos de manter. Cubra: status code esperado, schema/estrutura do payload de resposta, headers relevantes, comportamento de autenticação (401 sem token, 403 sem permissão), e validação de erros (400 com payload inválido).

### Testes E2E de UI — Playwright/Cypress

```ts
// Playwright — exemplo
import { test, expect } from '@playwright/test'

test('usuário consegue fazer login com credenciais válidas', async ({ page }) => {
  await page.goto('/login')
  await page.fill('[data-testid="email"]', 'user@example.com')
  await page.fill('[data-testid="password"]', 'senha123')
  await page.click('[data-testid="submit"]')

  await expect(page).toHaveURL('/dashboard')
  await expect(page.getByText('Bem-vindo')).toBeVisible()
})
```

Boas práticas de automação E2E:
- Use seletores estáveis (`data-testid`), nunca classes CSS ou texto que muda com frequência/idioma.
- Isole testes — cada teste deve poder rodar independentemente (setup/teardown próprio, não depender de ordem de execução).
- Evite `sleep`/espera fixa — use esperas explícitas por condição (elemento visível, resposta de rede).
- Automatize regressão de fluxos críticos primeiro; não persiga 100% de cobertura E2E (lento e frágil) — pirâmide de testes: muitos unitários, alguns de integração, poucos E2E.

## Reporte de bugs

Estrutura de um bug report útil:

```
Título: [componente] - descrição curta e específica do problema
Severidade: Crítica/Alta/Média/Baixa
Prioridade: (pode divergir de severidade — impacto no negócio vs urgência de correção)
Ambiente: [browser/OS/versão/ambiente - staging, prod]
Passos para reproduzir:
  1. ...
  2. ...
Resultado esperado: ...
Resultado obtido: ...
Evidência: [screenshot, log, request/response]
Frequência: sempre / intermitente (X de Y tentativas)
```

Diferencie **severidade** (impacto técnico do bug — trava o sistema? afeta poucos usuários?) de **prioridade** (urgência de correção do ponto de vista de negócio) — um bug de severidade baixa pode ter prioridade alta se afeta um cliente estratégico, por exemplo.

## Triagem de bugs

Ao ajudar a triar um conjunto de bugs, classifique por: é reproduzível? é um bug real ou comportamento esperado mal documentado? é duplicado de outro já reportado? qual o impacto real vs percebido? Isso evita que ruído baixo-impacto consuma o mesmo esforço que bugs críticos.

## Critérios de aceite — revisão

Ao revisar critérios de aceite de uma história/feature antes de testar, verifique se são: específicos, testáveis objetivamente (sem ambiguidade tipo "deve ser rápido" sem número), e cobrem os cenários de erro/borda, não só o caminho feliz. Se o critério de aceite for vago, sinalize isso antes de gerar casos de teste — não invente critério que não foi definido.

## "Está pronto para produção?" — checklist de avaliação

Quando o usuário perguntar se uma feature está pronta, avalie:
- Cobertura de teste dos cenários críticos (não só o feliz).
- Bugs abertos e sua severidade — algum crítico/bloqueante pendente?
- Testes de regressão nos fluxos que a mudança pode ter afetado indiretamente.
- Dados sensíveis expostos indevidamente, mensagens de erro vazando detalhes técnicos.
- Comportamento em conexão lenta/instável e em dispositivos/browsers-alvo, quando aplicável.
