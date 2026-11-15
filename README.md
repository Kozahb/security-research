# Security Research

Methodology notes and write-ups on web application security research, focused on authorization flaws
(BOLA/IDOR), API surface discovery and target selection.

I research public bug bounty programs under the handle [`kozahb`](https://hackerone.com/kozahb) on
HackerOne.

🇧🇷 [Versão em português](pt/README.md)

## The principle behind this repository

Everything here describes **technique and process, never targets**. No program, company, host, real
endpoint, report number or account appears in these texts. Route names are generic and serve only to
show the shape of the problem. The numbers cited are real measurements from my own work, under
authorised engagement and on accounts of mine.

Each text is published in accordance with the disclosure policy of the program concerned.

## Contents

### Methodology

- [From duplicates to target selection](methodology/from-duplicates-to-target-selection.md) - what a
  duplicate teaches about surface visibility, and the procedure for reconstructing an application's
  module graph (module federation) to find API surface the interface never advertises.

## Tools

- [kozarecon](https://github.com/Kozahb/kozarecon) - a pluggable reconnaissance toolkit, async and
  scope-enforced at its core.

## Contact

- HackerOne: [`kozahb`](https://hackerone.com/kozahb)
- LinkedIn: [Thaleson Eduardo](https://www.linkedin.com/in/thalesoneduardo/)
