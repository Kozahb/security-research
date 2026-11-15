# Security Research

Notas de metodologia e write-ups de pesquisa de segurança de aplicações web, com foco em falhas de autorização (BOLA/IDOR), descoberta de superfície de API e seleção de alvo.

Pesquiso em programas públicos de bug bounty sob o handle [`kozahb`](https://hackerone.com/kozahb) no HackerOne.

## Princípio deste repositório

Todo conteúdo aqui descreve **técnica e processo, nunca alvos**. Nenhum programa, empresa, host, endpoint real, número de report ou conta aparece nos textos. Nomes de rota são genéricos e servem apenas para mostrar a forma do problema. Os números citados são medições reais do meu próprio trabalho, em engajamento autorizado e com contas minhas.

Cada texto é publicado respeitando a política de divulgação do programa em questão.

## Conteúdo

### Metodologia

- [Das duplicatas à seleção de alvo](methodology/from-duplicates-to-target-selection.md) - o que uma duplicata ensina sobre visibilidade de superfície, e o procedimento de reconstrução do grafo de módulos de uma aplicação (module federation) para encontrar superfície de API que a interface não anuncia.

## Ferramentas

- [kozarecon](https://github.com/Kozahb/kozarecon) - toolkit de reconhecimento pluggável, assíncrono e com validação de escopo no núcleo.

## Contato

- HackerOne: [`kozahb`](https://hackerone.com/kozahb)
- LinkedIn: [Thaleson Eduardo](https://www.linkedin.com/in/thalesoneduardo/)
