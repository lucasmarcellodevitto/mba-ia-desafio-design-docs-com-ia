# Da Reunião ao Documento: Design Docs Gerados por IA


## Sobre o desafio

O desafio consiste em criar um conjunto completo de documentos de design docs a partir da transcrição de uma reunião sobre uma nova feature, utilizando a IA como principal ferramenta para ler e analisar o código existente e a transcrição da reunião, estruturar as informações e gerar o conteúdo final.

O resultado deverá contemplar os seguintes documentos:

| Documento | Papel | Pergunta que responde |
| --- | --- | --- |
| **PRD** | Problema, público, escopo e métricas de sucesso | *Por que e o quê?* |
| **RFC** | Proposta técnica da solução para revisão: abordagem geral, alternativas e questões em aberto | *Como pretendemos resolver, e o que ainda está em aberto?* |
| **ADRs** | Cada decisão arquitetural isolada, com contexto e consequências | *Por que decidimos exatamente assim?* |
| **FDD** | Especificação de implementação: fluxos, contratos, erros, integração com o código | *Como construir, em detalhe?* |
| **Tracker** | Rastreabilidade de cada item ao código ou à transcrição | *De onde veio cada coisa?* |

## Ferramentas de IA utilizadas

| Ferramenta | Uso |
| --- | --- |
| Claude Code (Anthropic) | Analise da transcrição e geração de todos os documentos. |

---

## Workflow adotado

A ordem seguida foi a mesma sugerida pelo enunciado original:

1 - Contextualização com IA: etapa para fornecer a IA acesso ao código (via Claude Code) e à transcrição. Exploração inicial para entender estrutura, padrões e o que a feature precisa endereçar.

2 - ADRs: etapa para identificar e produzir as principais decisões antes dos demais documentos. Essas decisões vão formar o esqueleto do "como implementar".

3 - RFC: consolidação da proposta técnica em cima das decisões. Referências aos ADR's escritos.

4 - FDD: após decisões formalizadas e a proposta consolidada, temos a etapa do detalhamento de implementação com a seção obrigatória "Integração com o sistema existente".

5 - PRD: por útimo temos o PRD. Documento mais alto nível, uma consolidação após a criação do RFC, FDD e ADRs.

6 - Tracker: montado em paralelo com os outros documentos.

7 - README: útima etapa. Feito quando o processo já estava completo.


## Prompts customizados

#### Prompt de contextualização

```
Leia o arquivo TRANSCRICAO.md do início ao fim e classifique cada tópico técnico discutido em uma das três categorias abaixo:

DECISÃO FECHADA — o grupo chegou a um consenso explícito sobre o tópico. Cite o timestamp [hh:mm] e o nome da pessoa que confirmou ou fechou a decisão.

DESCARTADA — a proposta ou alternativa foi levantada e posteriormente rejeitada de forma explícita. Informe o motivo do descarte e cite o timestamp [hh:mm] e o nome da pessoa que a descartou.

ADIADA / EM ABERTO — o tópico foi levantado, mas não foi decidido, ou foi explicitamente adiado para uma fase futura. Cite o timestamp [hh:mm] correspondente.

Não classifique um tópico como DECISÃO FECHADA por inferência. Essa classificação só deve ser utilizada quando houver uma frase ou manifestação explícita confirmando a decisão por parte de alguém do grupo.

Se não houver um timestamp claro associado à discussão ou à decisão, não inclua o item no resultado.

```

#### Prompt para validar os papéis e a coerência do conteúdo dos arquivos gerados

```
Revise os documentos ADR's, FDD, PRC, RFC e certifique que foram criados respeitando o papel de cada um.

Os documentos não se repetem, cada um opera em uma altura diferente. Verifique a fronteira entre eles: conteúdo duplicado entre documentos é um sinal de que algo esta no lugar errado.

Em uma frase: o RFC propõe e abre para revisão, os ADRs registram cada decisão fechada e o FDD detalha como construir. O RFC é conciso e fala em decisão; o FDD é profundo e fala em implementação. Não repita no RFC o nível de detalhe do FDD.

Use a seguinte tabela como apoio de revião:

| Documento | Papel | Camada | Pergunta que responde |
|---|---|---|---|
| **PRD** | Define o problema, o público, o escopo e as métricas de sucesso | Produto / Negócio | **Por que e o quê?** |
| **RFC** | Apresenta a proposta técnica para revisão, incluindo a abordagem geral, as alternativas consideradas e as questões em aberto | Arquitetura | **Como pretendemos resolver e o que ainda está em aberto?** |
| **ADRs** | Registra cada decisão arquitetural isolada, incluindo seu contexto e suas consequências | Decisão pontual | **Por que decidimos exatamente assim?** |
| **FDD** | Detalha a especificação de implementação, incluindo fluxos, contratos, tratamento de erros e integração com o código | Implementação | **Como construir, em detalhe?** |

```

## Iterações e ajustes

## Como navegar a entrega

Leia na ordem abaixo — ela vai do "por quê" até o detalhe de código, e cada documento
assume que você já viu o anterior:

1. **[docs/PRD.md](docs/PRD.md)** — o ponto de partida: qual dor de cliente originou a
   feature, quem são os afetados e como vamos saber que deu certo (objetivos e métricas).
2. **[docs/RFC.md](docs/RFC.md)** — a proposta de arquitetura submetida a revisão: o
   desenho geral da solução, o que foi avaliado e deixado de fora, e os pontos que
   seguem sem resposta.
3. **[docs/adrs/](docs/adrs/)** — o registro decisão a decisão. São 7 ADRs (ADR-001 a
   ADR-007) que sustentam o RFC; cada arquivo isola uma escolha e traz seu contexto, a
   decisão em si, o que foi descartado e o que ela custa.
4. **[docs/FDD.md](docs/FDD.md)** — o nível de "mãos no código": modelagem das tabelas,
   fluxos passo a passo, contratos HTTP, a matriz de erros `WEBHOOK_*` e a seção
   *Integração com o sistema existente*, que aponta onde tocar na base atual.
5. **[docs/TRACKER.md](docs/TRACKER.md)** — a referência cruzada. Se quiser conferir de
   onde saiu uma afirmação específica dos quatro documentos, aqui cada item está ligado
   à sua origem (um trecho da transcrição ou um arquivo do código).
6. **[TRANSCRICAO.md](TRANSCRICAO.md)** — a fonte primária. Útil para ler uma citação no
   contexto completo da conversa que a gerou.

