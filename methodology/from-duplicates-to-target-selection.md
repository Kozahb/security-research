# Metodologia: como cheguei a duplicatas e o que elas mudaram no meu processo

> **Sobre este documento.** Peça de portfólio. Descreve técnica e processo, não alvos: nenhum
> programa, empresa, host, endpoint real, número de report ou conta aparece aqui. Os nomes de rota
> são genéricos (`/bff/<namespace>/...`) e servem só para mostrar a forma do problema. Todos os
> números citados são medições reais do meu próprio trabalho, em engajamento autorizado e com
> contas minhas.
>
> Escopo: dois achados meus foram fechados como **duplicata** - bug real, já conhecido pelo
> programa. Este documento explica (a) o método que produziu cada um, (b) por que o método levava a
> terreno disputado, e (c) a mudança de processo que veio depois, com o procedimento executável.

---

## 1. O que duplicata significa, tecnicamente

Duplicata não é erro de medição. É a confirmação de que o bug existe, somada à informação de que
alguém chegou antes. As consequências práticas são outras:

- a **classe** foi validada pelo triador, então o enquadramento estava certo;
- a **causa raiz fica queimada**: pela regra usual de "várias issues com a mesma causa raiz = um
  pagamento", os endpoints irmãos da mesma família caem junto e não valem report novo;
- o custo é só o tempo. Na plataforma onde reporto, duplicata não soma nem subtrai reputação.

Isso torna a duplicata um dado de calibração barato: ela não diz que a técnica é ruim, diz que a
**superfície** era previsível. A pergunta correta depois de uma duplicata não é "o que fiz de
errado", é **"por que esse pedaço de superfície era óbvio para outras pessoas também"**.

---

## 2. Caso 1 - inconsistência de autorização entre dois serviços sobre o mesmo objeto

### A forma do bug

Um identificador de objeto era lido por dois serviços diferentes. O serviço "dono" da feature
devolvia **404** para um id de outra conta, corretamente. Um segundo serviço, de metadados em lote,
devolvia **200** com os metadados do mesmo objeto, para a mesma sessão, sem checar relação. A
inconsistência é inarguível justamente porque o alvo demonstra o comportamento correto num caminho
e o incorreto no outro, para o mesmo id.

### Como foi achado

1. **Inventário de endpoints por tipo de identificador, não por feature.** Agrupei as rotas pelo
   *tipo de id* que aceitam. Se dois endpoints aceitam o mesmo tipo de id, eles são comparáveis,
   mesmo pertencendo a produtos diferentes.
2. **Replay do id negado.** Peguei um id que um endpoint recusava com 404 e repeti o mesmo id em
   todos os outros endpoints do inventário que aceitam aquele tipo. O 404 de um serve de **baseline
   de comportamento correto**: qualquer 200 em outro serviço é um desvio medido, não uma suposição.
3. **Enumeração pelo parâmetro de corpo, não pelo path.** O endpoint vulnerável era um `:bulk`.
   Existia um único path com esse sufixo, mas os parâmetros de corpo (listas de ids de item)
   revelavam a família real. Enumerar pelo **shape do corpo** achou irmãos que enumerar o path não
   achava.
4. **Amplificação medida, não alegada.** O parâmetro aceitava lista separada por vírgula, logo uma
   requisição testa muitos ids. Isso transforma vazamento pontual em primitiva de varredura. Medi
   quantos ids por requisição eram aceitos antes de escrever a palavra "enumeração" no report.

### Por que duplicou

Endpoint em lote com array de ids é **a primeira coisa** que qualquer pessoa testa num BFF: ele
anuncia IDOR na própria assinatura. O achado dependia de observação cuidadosa, mas a superfície era
das mais visitadas do alvo.

### Limite que eu mesmo documentei antes de submeter

O dado vazado era fraco: um UUID e um id de usuário, e esse programa lista id de usuário como não
confidencial. Escrevi isso na análise interna **antes** de submeter e ordenei a fila de reports do
mais forte para o mais fraco. É a mesma disciplina de **delta de impacto**: campo que a feature
legítima já devolve para aquele mesmo ator não é vazamento.

---

## 3. Caso 2 - controle de segurança que a API pública respeita e a API interna não

### A forma do bug

A aplicação tem uma configuração de conta que exige aceite manual de convites de colaboração. Com
ela ligada, o convite fica `pending` e não concede acesso nenhum. O endpoint interno que a UI do
**convidado** usa para aceitar convites pendentes não verificava que o chamador era o convidado.
Quem enviou o convite podia chamar esse endpoint com o id da própria colaboração e marcar o convite
como aceito no lugar da vítima, sem ação nem notificação dela.

### Como foi achado

Esta é a parte que vale como processo, porque o valor está na ordem dos passos:

1. **Partir da resposta do triador, não do endpoint.** Um achado anterior meu tinha sido fechado
   como informativo, com o argumento de que o comportamento padrão é documentado e que um report
   precisaria mostrar um **controle de segurança sendo burlado**. Isso define o alvo seguinte: não
   mais o comportamento padrão, e sim o controle que o usuário liga explicitamente.
2. **Ligar o controle e atacar primeiro pela API pública documentada.** Cinco variantes, todas
   mantiveram o controle: convite por e-mail, convite por id numérico de usuário, convite com
   supressão de notificação, injeção de `"status": "accepted"` no corpo da criação, e update direto
   do status da colaboração (que respondeu `400 invalid_status`). **Os cinco negativos são o valor
   do teste**: provam que o controle é real e estreitam o problema a um único caminho possível.
3. **Perguntar como a UI legítima executa a ação privilegiada.** Observei o fluxo do convidado
   aceitando o convite e peguei a rota interna que a interface chama, cujo corpo é uma lista de
   objetos de colaboração.
4. **Verificar quem alcança o identificador.** O convidante lê o id da colaboração pendente na
   própria lista de colaboradores. Não é adivinhação nem bruteforce: o atacante já tem o id de
   forma legítima. Essa checagem é o que separa bug explorado de bug hipotético.
5. **Chamar a rota do lado errado.** Mesma requisição, sessão do **convidante**. 200, convite
   aceito, acesso concedido. Reproduzido nas duas direções, com as duas contas alternando o papel de
   vítima, para descartar contaminação de estado.
6. **Limpar e restaurar.** Colaborações removidas, pastas apagadas e purgadas nas duas contas, e as
   configurações de sharing restauradas ao valor original - inclusive numa conta que já estava fora
   do padrão antes do teste, o que registrei para não confundir baseline com efeito meu.

### Por que duplicou

Authz em endpoint de BFF **que tem botão equivalente na UI** é superfície pisada. A interface
documenta a existência da rota para qualquer pessoa que abra o DevTools. A medição foi limpa e o
enquadramento passou a barra que o próprio triador tinha definido - e ainda assim era terreno
disputado.

---

## 4. O sinal que prediz duplicata

Das duas, extraio um preditor que hoje aplico antes de investir tempo:

> **Se a ação tem botão equivalente na interface, assuma superfície pisada.**

O corolário é operacional, não filosófico: existe botão => a rota aparece na aba de rede de
qualquer pessoa logada, em dez segundos, sem ferramenta nenhuma. A dificuldade do bug não protege
contra duplicata; a **visibilidade da superfície** é que decide. Ordem de valor que adotei:

| prioridade | superfície | por que |
|---|---|---|
| alta | namespace que minha sessão alcança mas que minha UI não expõe | ninguém acha sem colher bundle |
| alta | serviço que responde ao meu plano, mas cuja feature meu plano não tem | descompasso entre entitlement e authz |
| média | RPC legado / parâmetro de rota antigo ainda roteado | sem UI e sem documentação |
| baixa | endpoint de BFF com botão correspondente | todo mundo já passou por ali |

---

## 5. A contramedida: achar a superfície que não tem botão

Depois da segunda duplicata, mudei o alvo da fase de recon. Em aplicação de micro-frontends, os
bundles que a página **carrega** são uma fração do código: o host carrega sob demanda *remotes* que
têm os próprios chunks, e boa parte dos paths de API existe **só** lá dentro.

### 5.1 Procedimento - reconstruir as URLs dos chunks sem executar o JS

1. **Detectar módulos federados** na página autenticada: objeto de manifesto injetado no HTML
   (`webpackRemotesManifest`, `__federation`, `mf-manifest.json`), função `__webpack_require__.u`
   nos bundles do host, CDN segmentado por produto.
2. **Extrair o manifesto** do HTML de uma página logada. É o único passo que precisa de sessão.
   Cada entrada dá hostname, path, versão e nome do arquivo de entrada do remote.
3. **Baixar as entries** (anônimo, direto no CDN). Elas são runtime de module federation: servem
   para achar o **mapa de chunks**, não para grep de endpoint. No meu caso as entries tinham
   **zero** ocorrências do prefixo do BFF - quem grepa só a entry conclui, errado, que não há
   superfície ali.
4. **Reconstruir as URLs por parsing, nunca com `eval`.** É código de terceiro baixado da internet;
   dar `eval` nele é executar código arbitrário na própria máquina. Três regras que custaram rodada
   para descobrir:
   - a função certa se acha por **pontuação**, não por regex de `u=`: a expressão tem de conter
     `.js` **e** um mapa com pelo menos 3 valores hexadecimais de 6 a 24 chars;
   - prefixo = o **último** literal terminado em `/` antes do primeiro mapa (há URLs hardcoded antes
     da função, e concatenar todos os literais produz lixo);
   - sufixo = o **último** literal que começa com ponto e termina em `.js`.
5. **Validar 1 URL por remote com HEAD antes do download em massa.** Layouts coexistem no mesmo app
   (`.chunk.js`, `.bundle.js`, `.js` puro, com prefixos diferentes por remote). Na minha execução, 3
   de 26 remotes estavam com o sufixo errado e **só** esse teste mostrou.
6. **Filtrar tradução antes de baixar** (`locale|messages|lang|i18n|translation` pelo mapa de
   nomes): cortou 394 de 2494 chunks sem perder nada, porque chunk de tradução não tem endpoint.
7. **Baixar com throttle** (poucas conexões, pausa curta). É conteúdo estático no CDN, não no app do
   alvo, mas volume desnecessário é barulho desnecessário. Hash de chunk é imutável, logo cache
   local vale: só rebaixar quando a versão do remote mudar.
8. **Grep pelo prefixo do BFF capturando o namespace**, não por `/api/`. O produto da fase não é uma
   lista de endpoints: é o **diff dos namespaces contra a lista que eu já conhecia**.

**Resultado medido:** 26 remotes, 2494 chunks reconstruídos, 2100 depois do filtro, 1951 baixados
com 200 e 148 com 404 (ids que só têm CSS - normal). Cinco namespaces de BFF que nenhuma sessão
anterior tinha visto apareceram **exclusivamente** nos chunks dos remotes. Um deles era o subsistema
de cobrança, com 24 rotas e zero botão correspondente na minha interface.

### 5.2 Ponto cego honesto desta fase

Existem endpoints que não estão em bundle nenhum. Três rotas de lock/save de um editor não apareciam
no bundle principal, nem nos 427 chunks do host, nem nos 27 chunks do próprio remote: foram achadas
por **captura de tráfego** ao exercitar a feature. Análise estática e captura são complementares,
não substitutas. Se uma feature importa, exercite-a com a rede gravando.

---

## 6. Triagem antes do teste: três decisões de custo

Namespace novo no grep não quer dizer namespace vivo, e superfície nova não quer dizer severidade
alta. Três filtros que rodo antes de montar qualquer cenário:

### 6.1 Alcançabilidade - só GET, matriz anônimo x sessão, em todos os hosts

Aprender o **dialeto de erro do alvo** é o que dá significado aos códigos:

| resposta | leitura |
|---|---|
| 200 | vivo e respondendo a minha sessão. Prioridade |
| 401/403 com corpo de erro estruturado | o serviço existe e está autenticando. Vale cenário de authz |
| 401/403 **vazio** | em geral o namespace **não existe** naquele host. Descartou dois candidatos sem gastar teste |
| 404 numa rota que o bundle declara | existe mas falta parâmetro. Descobrir o param é o próximo passo |
| 400 com mensagem de validador | o validador está falando: usar a mensagem para extrair o schema |
| 200 com erro no corpo | *errors-as-data*. Nunca concluir "funcionou" pelo status - ler o corpo sempre |

A última linha já me custou dois falsos positivos próprios: mutations GraphQL que devolviam 200 com
"Authorization Failed" dentro do payload, lidas como sucesso porque a query pedia só o `__typename`.
Hoje toda mutation pede o campo de erros junto.

### 6.2 Teto de severidade, calculado antes de investir

Exemplo medido: um parâmetro de redirect era aceito **sem nenhuma validação** num fluxo de OAuth, o
que parecia caminho para comprometimento de conta. Mas o `redirect_uri` registrado no provedor é
**fixo**, logo o code nunca passa pelo domínio do atacante, e trocá-lo exige o `client_secret` do
alvo. Teto real: open redirect autenticado. Continua valendo medir porque é barato - mas não como
aposta principal, e isso se decide **antes** de montar o setup.

### 6.3 Ramos de erro antes de setup caro

Em fluxo de OAuth/redirect, o ramo de `error=` ou de code inválido às vezes já redireciona e entrega
a resposta de graça. Testei com um `state` válido recém-capturado: 400 sem header `Location` em
todos os ramos. Esse **negativo** é que provou que o setup caro era inevitável, ao custo de 10
requisições.

---

## 7. Checklist que uso hoje

Antes de investir num endpoint:

- [ ] Existe botão equivalente na UI? Se sim, assumir superfície pisada e procurar a variante sem botão.
- [ ] Qual o teto de severidade se o bug existir exatamente como imagino? Vale o setup?
- [ ] A matriz GET (anônimo x sessão, todos os hosts) diz que isso está vivo?
- [ ] Qual o **delta** sobre o que a feature legítima já devolve para o mesmo ator?
- [ ] O identificador necessário é alcançável de forma legítima, ou estou assumindo adivinhação?
- [ ] O baseline está isolado - cenário limpo, sem colaboração ou estado residual que explique o 200?
- [ ] Os ramos de erro já entregam a resposta sem o setup completo?

Durante:

- [ ] Nada de scanner automático; automação estreita, baixo volume, header de identificação do pesquisador.
- [ ] Só contas e objetos meus; exploração limitada ao mínimo que demonstra o impacto.
- [ ] Rota de escrita em cobrança não se exercita: mapear, ler o schema, medir authz só na leitura, e
      declarar no report que a escrita não foi exercitada por decisão de escopo.
- [ ] Reproduzir nas duas direções e restaurar o estado das contas no fim.

---

## 8. O que este documento não é

Não é um método para "evitar duplicatas" no sentido de garantia - num programa maduro, qualquer
superfície visível já foi olhada por centenas de pessoas. É uma mudança de **onde** gastar tempo: de
superfície que a interface anuncia para superfície que só aparece reconstruindo o grafo de módulos da
aplicação, com filtros de custo antes de cada investimento. As duas duplicatas são a evidência de que
a parte de medição funcionava; o que faltava era seleção de alvo.
