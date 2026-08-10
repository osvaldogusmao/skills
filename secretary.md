---
name: secretary
description: Gera atas de reuniões técnicas a partir de arquivos de transcrição .vtt exportados do Microsoft Teams, estruturando o conteúdo em temas abordados, compromissos assumidos e próximos passos. Use esta skill sempre que o usuário mencionar "ata de reunião", "minuta de reunião", "resumo da reunião", enviar ou referenciar um arquivo .vtt, ou pedir para transformar a transcrição/gravação de uma reunião do Teams em um documento estruturado — mesmo que não use a palavra "ata" explicitamente.
---

# Ata de Reunião Técnica a partir de Transcrição

## Objetivo

Transformar a transcrição bruta de uma reunião técnica (arquivo `.vtt` exportado do Microsoft Teams) em uma ata estruturada, cobrindo três blocos obrigatórios:

1. **Temas abordados** — os assuntos discutidos, organizados por tópico.
2. **Compromissos** — o que foi assumido, por quem, durante a reunião.
3. **Próximos passos** — o que fica definido para depois da reunião.

Mantenha o nível técnico da discussão — não simplifique termos técnicos, decisões de arquitetura, nomes de sistemas/ferramentas citados na reunião.

## Quando usar

Sempre que o usuário fornecer ou referenciar um arquivo `.vtt` de uma reunião do Teams e pedir uma ata, minuta, resumo estruturado, lista de compromissos ou de próximos passos da reunião.

## Passo 1 — Ler e entender o arquivo .vtt

Um arquivo `.vtt` (WebVTT) segue este formato básico:

```
WEBVTT

00:00:03.500 --> 00:00:07.200
<v João Silva>Bom dia pessoal, vamos começar a reunião.</v>

00:00:07.500 --> 00:00:12.000
<v Maria Souza>Bom dia! Eu queria falar sobre o status da integração com o Zoho.</v>
```

- Cada bloco tem um intervalo de tempo e o texto falado.
- O nome do falante pode vir dentro de uma tag `<v Nome do Falante>...</v>`, como prefixo antes de dois-pontos, ou pode não estar presente — depende da configuração de exportação do Teams. **Inspecione o arquivo real antes de assumir um formato fixo.**
- Reuniões longas geram arquivos grandes; leia o arquivo inteiro antes de processar, sem assumir truncamento.

## Passo 2 — Consolidar a transcrição

Antes de gerar a ata, produza uma versão consolidada e legível da transcrição:

- Agrupe falas consecutivas do mesmo falante em um único bloco (elimina a fragmentação típica de legendas automáticas).
- Descarte os timestamps na versão consolidada — mantenha-os só como referência interna, caso precise localizar um trecho específico depois.
- Se os nomes dos falantes não estiverem disponíveis no `.vtt`, não invente nomes — use "Participante 1", "Participante 2" etc., e avise o usuário dessa limitação logo no início da resposta.

Prefira um pequeno script Python (executado via ferramenta de terminal) para fazer esse parsing de forma determinística, em vez de processar o arquivo bruto manualmente — reduz erro e é mais rápido em arquivos grandes.

## Passo 3 — Extrair o conteúdo da ata

A partir da transcrição consolidada, identifique:

### Temas abordados
- Divida a reunião em tópicos/blocos de assunto — não é preciso seguir uma pauta pré-definida; os temas emergem do que foi de fato discutido.
- Para cada tema: um título curto + um resumo objetivo do que foi discutido, decisões tomadas, e pontos de discordância relevantes, se houver.
- Preserve termos técnicos, nomes de sistemas, ferramentas e siglas exatamente como usados na reunião.

### Compromissos
- Qualquer trecho onde alguém assume fazer algo (ex: "eu vou verificar", "fico responsável por", "vou enviar até sexta").
- Formato: responsável (quando identificável) + o que foi comprometido + prazo, se mencionado.
- Não infira compromissos que não foram verbalizados explicitamente. Se houver dúvida se algo é de fato um compromisso, deixe de fora ou marque como "a confirmar" em vez de assumir.

### Próximos passos
- Itens de ação combinados para depois da reunião, decisões sobre a próxima reunião, dependências entre equipes, pendências.
- Pode se sobrepor parcialmente com compromissos — quando isso acontecer, coloque no bloco mais específico: compromisso individual e claro vai em "Compromissos"; algo combinado em grupo ou sem responsável único vai em "Próximos passos".

## Passo 4 — Gerar a ata

Use este template como estrutura do documento final (ajuste os títulos das seções se o usuário pedir algo diferente):

```markdown
# Ata de Reunião — [Título/tema da reunião, inferido do conteúdo]

**Data:** [extrair da reunião, se disponível no nome do arquivo ou no conteúdo; caso contrário, perguntar ou deixar em branco]
**Participantes:** [lista de falantes identificados]

## Temas abordados

### [Tema 1]
[resumo objetivo]

### [Tema 2]
[resumo objetivo]

## Compromissos

- [Responsável] — [compromisso] [(prazo, se houver)]

## Próximos passos

- [item]
```

Gere o arquivo como `.md` por padrão. Se o usuário pedir explicitamente um documento Word, use a skill de docx para produzir o `.docx` a partir do mesmo conteúdo. Ao final, apresente o arquivo gerado ao usuário.

## Cuidados

- Nunca invente informação que não está na transcrição — se um bloco (ex: compromissos) ficar vazio porque a reunião não gerou nenhum, diga isso explicitamente na ata em vez de forçar itens artificiais.
- Se o `.vtt` vier sem identificação de falante, avise o usuário logo no início e ofereça associar as falas caso ele tenha a lista de participantes em mãos.
- Reuniões técnicas tendem a ter jargão específico do domínio — não troque por explicações genéricas; mantenha o vocabulário usado pelos participantes.
