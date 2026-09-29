# Assistente de Ordens XP

Ferramenta web estática (Vite + JS puro) que transforma solicitações de clientes em texto livre
nos 4 formatos de saída padronizados do back office: **Ordem por E-mail**, **Auditoria por E-mail**,
**Lote Simples (TSV)** e **Lote TWAP (TSV)**.

O usuário é operador de back office de escritório credenciado à XP. Ele cola a solicitação como
recebeu (texto, print transcrito, tabela de posições) e copia a saída pronta.

## A regra que domina todas as outras

**Nada é adivinhado.** O sistema só emite o que foi informado. Quando falta dado crítico, ou quando
há ambiguidade, a saída é **bloqueada** e o que falta é exibido — jamais preenchido por inferência,
arredondamento ou "melhor esforço". Um número errado numa ordem de renda variável é dinheiro real
do cliente; uma saída ausente é só um clique a mais.

Ao mexer em parser, validators ou formatters, o teste mental é sempre: *este valor veio do texto do
usuário, ou eu o inventei?*

## Vocabulário do domínio

Use estes termos no código, nos testes e nas conversas — eles são os mesmos que o usuário usa.

- **Conta** — código numérico da conta XP (5 a 8 dígitos). É o único valor aceito na coluna/campo
  `Cliente`. Nome de pessoa nunca entra na saída, mesmo quando digitado (`80000005 - Fulana`
  → `80000005`).
- **Quantidade** — número inteiro de ativos. Sem separador de milhar, sem `R$`.
- **Financeiro** — valor em reais a ser executado. Sempre formatado `R$ X.XXX,XX`. Chega com `R$`,
  mas também sem ele, e estas formas contam como financeiro (decidido com o operador): centavos
  (`3.000,00`, quando é o único número da linha), `3.000 reais`, `valor 3.000`, `3 mil`, `20k`.
  `3 mil cotas` é quantidade. Número sem nada (`3.000`) continua quantidade.
- **Quantidade e Financeiro são mutuamente exclusivos** numa mesma ordem: uma ordem é *por
  quantidade* ou *por financeiro*, nunca as duas. Confundi-los é o erro mais caro do sistema.
- **Ordem** — uma linha de execução: conta + ativo + operação + (quantidade | financeiro) + preço.
- **Solicitação** — o texto colado pelo usuário, que gera uma ou várias ordens.
- **Cesta** — várias ordens na mesma solicitação, compartilhando conta e operação, cada ativo com
  seu próprio valor. Gera uma linha por ativo.
- **Preço** — o preço informado, preservado **exatamente** como digitado (sem arredondar, sem
  reformatar). Ausente ⇒ literalmente `A mercado`.
- **Operação** — `Compra` ou `Venda` no preview e no e-mail; `C` ou `V` nos TSVs. O grupo também
  escreve `aplicação` e `aporte` para compra, `resgate` e `saída` para venda — decisão do operador.
  Um cabeçalho (`Venda:`) vale para as linhas de baixo até o próximo; a operação escrita na
  própria linha vale mais que ele.
- **Cesta mista** — cesta com ordem por financeiro e ordem por quantidade lida de número solto
  (`BTLG11 3.000,00` / `XPML11 3.000`). Quase sempre são centavos esquecidos. Bloqueia com dois
  botões: **Tudo em R$** (mesmo número, só troca o tipo) e confirmar. `100 cotas`, escrito assim, não
  dispara: é mistura de propósito.
- **Bloqueio** — alerta que impede a geração da saída, quase sempre com um botão de confirmar.
  Distinto de um **aviso**, que só acompanha a saída sem segurá-la.
- **Descarte** — linha que não pôde virar ordem e sai listada com o motivo, para o operador
  tratar à mão. Nunca some calada.

## Fora de escopo

O escritório já trata estes casos à mão, e automatizá-los exigiria justamente a adivinhação que o
projeto existe para evitar. O sistema os reconhece e os **descarta com o motivo à vista**:

- **Ordem sem quantidade nem financeiro** — `V - HGCR11 (Total)`, onde "Total" é a única indicação
  de tamanho. Não há número a preservar, então não há ordem a gerar.
- **Push** — é canal do Hub da XP, alheio ao fluxo de e-mail e lote.
- **Observações entre parênteses** coladas ao valor ("R$ 34.800(não ultrapassar esse valor)"): o
  valor é lido, a observação não é reproduzida.

## Fundos cetipados

Fundos fechados e feeder não têm ticker: chegam escritos por extenso, do jeito que o assessor
digitou. `validate/fundosXP.js` guarda a prateleira de distribuição da XP e `validate/fundo.js`
faz a busca por nome.

Eles usam o **mesmo modelo de e-mail das ações**. Lote, TWAP e a tabela da auditoria ficam
bloqueados: a coluna `Ativo` identifica o papel por código, e — de todo modo — esses fundos não são
executados em bolsa. O operador reconfirmou a tabela em 28/09, já com a coluna `Financeiro` pronta:
fundo fica só no e-mail em texto. Por isso "via email" num pedido com fundo não marca a tabela.

Um nome erra de um jeito que um ticker não erra: **por uma palavra**. `XP Habitat Renda Imobiliária
Feeder` e `XP Habitat Renda Imobiliária II Feeder` são fundos diferentes separados por um numeral.
Por isso os bloqueios `fundo-ambiguo` e `fundo-parcial` **não são confirmáveis**: confirmar não
decide qual dos três XP Habitat o cliente pediu. A saída é o operador clicar num dos candidatos
oferecidos no preview — sem esses botões o bloqueio seria um beco sem saída.

A prateleira não tem fonte pública para baixar. A CVM publica o cadastro de todos os fundos do
Brasil, mas ninguém publica o que a XP distribui, e não dá para inferir pelo administrador nem
pelo gestor. A lista vem copiada do portal e `npm run fundos:update -- arquivo.txt` regenera o
módulo. Guardamos só nome, ticker e quantidade mínima: preço e disponibilidade mudam todo dia, e
guardá-los seria carregar dado vencido dentro de um sistema cuja regra é não inventar dado.

## A entrada real

O operador cola **direto do grupo de WhatsApp** onde as ordens chegam, com o carimbo
`[13:46, 21/09/2026] Nome:` na frente e as confirmações da mesa no meio. `limparEntrada` descarta
tudo isso antes que qualquer parser veja o texto.

O descarte do carimbo é o mais importante de todos: ele contém um horário, e um horário lido dali
viraria `Hora Inicial` de uma ordem TWAP — um dado inventado dentro de uma ordem real. Há um teste
só para isso, e ele deve continuar passando.

Normalmente vem uma conta por bloco, mas duas ou mais são possíveis: a linha com o número da conta
abre uma nova solicitação, e misturar a conta de um cliente com o ativo de outro é o pior erro que
o sistema poderia cometer.

A linha da conta traz às vezes o pedido junto (`7000002 - compra via email`): o que vem depois do
número é lido como contexto, mas só com palavra inteira — o `V` de `Carlos V. Silva` não é venda.
Linha com ticker ou unidade depois do número (`10000 - PETR4`, `50000 reais cada`) não é conta.
Numeração de lista, marcadores e negrito do WhatsApp são tirados na limpeza; o tachado (`~`) não,
porque riscar costuma ser cancelar.

O valor às vezes vem **na linha de baixo** do ativo (`Riza Malls` / `R$ 16.000,00`), sobretudo em
fundo cetipado, cujo nome já ocupa a linha. `casarValores` liga cada valor sozinho ao ativo logo
acima, se ele não trouxe valor próprio. Um valor sem ativo logo acima **cancela o casamento do
bloco inteiro** e tudo vai para os descartes: é o sinal de que o valor pode vir *acima* do nome, e
casar de cima para baixo daria a cada fundo o valor do vizinho — erro que conferência nenhuma pega.

## Invariantes

Cada uma vale um teste. Violá-las é um bug de negócio, não de estilo.

1. `Cliente` é sempre a conta numérica.
2. Quantidade nunca vira financeiro, financeiro nunca vira quantidade.
3. `Compra` → `C`, `Venda` → `V`.
4. Preço informado é preservado literalmente; ausente é `A mercado`.
5. Ticker suspeito de erro de digitação gera **aviso pedindo confirmação** — a correção automática
   é proibida.
6. Ativo repetido na mesma solicitação **bloqueia** pedindo confirmação; somar automaticamente é
   proibido.
7. Linha sem quantidade nem financeiro não vira ordem: é descartada com o motivo à vista, nunca
   completada por inferência.
8. Conta ausente, operação ausente e ticker suspeito **bloqueiam** a geração.
9. A ordem de digitação dos ativos é preservada na saída.
10. Financeiro sai como `R$ X.XXX,XX` (pt-BR).
11. `Preço Would`, `Hora Inicial` e `Hora Final` só saem preenchidos com informação explícita.

Quase todo bloqueio traz um botão de confirmar-e-gerar. Um bloqueio sem saída faria o operador
contornar o sistema no meio do expediente, que é exatamente o risco que ele existe para evitar. As
exceções são os bloqueios que confirmar não resolve:

- **conta ausente** — sem conta não existe ordem para confirmar;
- **fundo ambíguo ou parcial** — confirmar não escolhe o fundo; a saída são os candidatos no
  preview;
- **ordem que não cabe no formato** — financeiro no Lote Simples, fundo cetipado no lote, no TWAP
  ou na auditoria. Não há o que atestar: a coluna não existe, e confirmar escrevia `null` em
  `Qtd. Total`. A saída é outro formato, marcado a um clique.

A comporta (`gerar`, em `ui/saidas.js`) só aceita confirmação de bloqueio confirmável — ela não
depende de o botão estar escondido.

A **Auditoria por E-mail** — na tela, **E-mail em tabela**, que é como o operador a chama — é o
e-mail da ordem com a tabela do Lote Simples no corpo, copiada em HTML e em texto com TAB, os dois
de uma vez. Desde 28/09 a coluna de valor **acompanha a cesta**: só quantidade, `Qtd. Total`, igual
ao Lote Simples; só reais, `Financeiro` no lugar dela; mista, **duas tabelas** — a de financeiro
primeiro, a de quantidade depois, duas linhas em branco entre elas. Isso reverteu uma decisão
anterior de manter a tabela sem financeiro — o operador mudou de ideia com um pedido real na mão.
O Lote Simples em TSV, colado na planilha da XP, continua sem financeiro. "Tabela igual Excel", para o operador, é **grade
simples**: borda fina em todas as células e nada mais — sem cor, sem negrito, sem fonte declarada.
Ele cola no Outlook na web, que dá ao que chega sem fonte a fonte da mensagem; a primeira versão
declarava Aptos e pintava o cabeçalho de cinza, e foi recusada por destoar do resto do e-mail.

Os textos e cabeçalhos exatos das 4 saídas estão em [`docs/FORMATOS-DE-SAIDA.md`](docs/FORMATOS-DE-SAIDA.md).
Leia esse arquivo antes de alterar qualquer formatter ou template — ele é a fonte da verdade dos
literais, e um caractere fora do lugar quebra a colagem no Excel ou o padrão do e-mail.

## Arquitetura

O fluxo é um pipeline de uma direção, e cada etapa é uma função pura testável isoladamente:

```
texto colado → limparEntrada → parseSolicitacoes → Solicitacao[] → validar → Diagnostico[]
                                                         ↓                        ↓
                                                  preview editável ──→ formatadores → texto
```

- `src/core/` — o pipeline. Sem DOM, sem `localStorage`, sem `window`. É aqui que vivem os testes.
  - `texto/limparEntrada.js` — tira o carimbo do WhatsApp e o ruído da mesa
  - `parse/parseSolicitacoes.js` — quebra por conta e monta as ordens; `parse/valores.js` decide
    quantidade × financeiro
  - `validate/ticker.js` — reconhece o ticker contra `validate/tickersB3.js` e levanta suspeita
  - `validate/validar.js` — produz bloqueios e avisos
  - `format/formatadores.js` — escreve o e-mail e os dois TSVs
- `src/ui/` — render e edição. `preview.js` monta o cartão e aplica as edições; `saidas.js` decide
  entre mostrar a saída ou o bloqueio.
- `src/platform/` — adaptadores de browser (clipboard, histórico), isolados para o core não
  depender deles.
- `src/main.js` — só estado de tela, eventos e render. Nenhum julgamento de negócio mora aqui.

`validate/tickersB3.js` é **gerado**, com 1.624 tickers da B3 por classe. Ele existe para uma
coisa só: `KCNR11` é um ticker perfeitamente bem formado, e nenhuma regra de formato revela que
ele é `KNCR11` digitado errado. Só a comparação contra a lista revela.

**O preview é a comporta.** Nenhuma saída é gerada direto do parser: o parser propõe, o usuário
confere e corrige campo a campo no preview, e só então os formatters rodam sobre os dados
confirmados. Os formatters recebem dados já validados e não fazem inferência alguma.

O botão **+** do cartão acrescenta uma linha em branco — nem a operação da cesta ela herda. Por
isso a validação cobre o que o parser nunca produz: ativo vazio e ordem sem valor bloqueiam sem
confirmação, e nome que não é ticker nem fundo da prateleira bloqueia com confirmação. O tipo
escolhido no controle Qtd/R$ fica em `ordem.tipo`, porque numa linha vazia não há valor de onde
deduzi-lo.

## Sistema visual

`style.css` é construído sobre tokens (`--fundo`, `--superficie`, `--texto`, `--marca`, …)
redefinidos por `:root[data-tema='claro']` e `:root[data-tema='escuro']`. **Escreva sempre pelo
token, nunca por um valor literal** — um `#fff` cravado no CSS quebra um dos dois modos, e só se
descobre quando alguém troca de tema no meio do expediente.

A ferramenta fica aberta o dia inteiro, e isso governa a paleta:

- **Sem extremos.** Nada de preto puro no fundo do modo escuro, nem branco puro no texto; nada de
  preto puro no texto do modo claro. Os extremos cansam a vista porque forçam a pupila a se
  readaptar a cada troca de foco entre a tela e a sala.
- **Corpo de texto entre 7:1 e 8:1**, títulos por volta de 12:1, texto secundário nunca abaixo de
  4.5:1. Ao mexer nas cores, meça de novo — é fácil derrubar o secundário abaixo do mínimo.
- **Saturação baixa no escuro.** Cor saturada sobre fundo escuro vibra na retina. Os tons do modo
  escuro são suavizados, não os do claro reaproveitados.
- **Cor carrega significado.** Verde é compra, vermelho é venda, âmbar é atenção, e o âmbar da
  marca aparece em poucos lugares. Cor que decora tira força da cor que informa.

Movimento é curto (120–260 ms) e sempre com motivo: orientar de onde algo veio, confirmar um
clique. Nada pisca nem chama atenção duas vezes para a mesma coisa, e
`prefers-reduced-motion: reduce` desliga tudo.

Duas decisões do preview que vale manter:

- **Controle segmentado no lugar de `<select>`** quando a escolha precisa ser conferida de
  relance. Num `select` fechado só a opção atual aparece, e conferir uma cesta de dez ativos
  exigiria abrir linha por linha — justamente o que o preview existe para evitar.
- **O resumo da cesta soma financeiro e nunca quantidade.** A soma em reais é a exposição total e
  significa alguma coisa; 100 PETR4 mais 50 VALE3 não são 150 de nada, e esse número no topo do
  cartão convidaria a conferir a ordem contra um total sem sentido. Quando o total em reais não
  cobre a cesta inteira, ele se anuncia como parcial ("de 2 ordens").

O modo claro é um **neutro morno em quatro níveis** (`--fundo-sutil` < `--fundo` <
`--superficie-afundada` < `--superficie` < `--superficie-alta`), cada um com papel fixo: a página
afunda, o cartão sobe sobre ela, o campo afunda de novo dentro do cartão, e o branco puro fica
reservado ao que flutua. Cartão branco puro sobre fundo quase branco não cria hierarquia nenhuma —
a tela vira uma chapa só. O tom quente combina com o âmbar da marca e é percebido como mais macio
em leitura longa que o branco azulado de tela.

O tema vive em `document.documentElement.dataset.tema`, aplicado por um script inline no `<head>`
antes da primeira pintura — sem isso a tela pisca no tema errado a cada carregamento. O padrão é
**escuro**, gravado por navegador no `localStorage`; a preferência do sistema fica de fora de
propósito, porque a mesma pessoa pode ter o Windows no claro e ainda assim querer esta ferramenta
no escuro. Ao mexer no padrão, mexa nos **dois** lugares: `src/ui/tema.js` e o script inline.

## Testes

`npm test` (Vitest, ambiente node). O core é testado direto; a UI não é.

Trabalhe **test-first**: escreva o caso de negócio como teste em `*.test.js` ao lado do módulo,
veja-o falhar, então implemente. Cada invariante acima merece um caso próprio, e os que mais
quebram são quantidade × financeiro, TWAP por quantidade × TWAP por financeiro, e conta com nome
colado junto.

Quando o operador trouxer um pedido real do grupo, **transforme-o em teste antes de mexer no
código**. Os casos de `parseSolicitacoes.test.js` que citam contas reais vieram assim, e são os que
mais valem: eles descrevem como as ordens chegam de verdade, não como imaginamos que cheguem.

## Convenções

- JS puro com módulos ES, sem framework e sem dependência de runtime. Vitest e Vite são
  ferramentas de desenvolvimento, não vão para o bundle.
- Português nos nomes de domínio (`quantidade`, `financeiro`, `conta`), inglês nos termos técnicos
  (`parse`, `format`, `validate`).
- TSV usa TAB real (`\t`) e `\n`; nunca espaços alinhados nem markdown.
- Dados de exemplo e presets usam contas fictícias (`1234567`) e nunca dados de cliente real.
- **Funciona offline, aberto do disco.** Colegas do operador usam a ferramenta sem internet,
  abrindo `dist/index.html` com dois cliques (`file://`). O `vite.config.js` embute JS, CSS e
  ícone nesse arquivo, e `vite.config.test.js` garante que ele não aponta para arquivo nenhum.
  Isso proíbe recurso externo (Google Fonts, CDN), `import()` dinâmico, `fetch` de arquivo local
  e service worker. O plugin falha o build se sobrar algum arquivo fora do HTML.
