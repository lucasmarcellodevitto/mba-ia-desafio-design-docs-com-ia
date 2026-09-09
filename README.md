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
