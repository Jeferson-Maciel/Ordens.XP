# Assistente de Ordens XP

Gerador de ordens de renda variável para back office. Você cola a solicitação do cliente como ela
chegou — texto corrido, print transcrito, tabela de posições — confere o que o sistema entendeu, e
copia a saída pronta em um dos 4 formatos padronizados.

Substitui o trabalho de reescrever ordem por ordem à mão num chat de IA, sem o risco de a saída
"melhorar" um dado que você não informou.

## Os 4 formatos

| Formato | Para quê | Saída |
| --- | --- | --- |
| **Ordem por E-mail** | pedir confirmação da execução ao cliente | texto |
| **E-mail em tabela** (auditoria) | o e-mail da ordem com a tabela do Lote Simples no corpo; ganha a coluna `Financeiro` quando a cesta é em reais, e vira duas tabelas quando mistura reais e quantidade | tabela em grade (+ texto com TAB) |
| **Lote Simples** | colar na planilha de lote | TSV |
| **Lote TWAP** | colar na planilha de TWAP | TSV |

Os TSVs usam TAB real e colam direto em Excel e Planilhas Google, uma coluna por campo.
A especificação literal de cada formato está em [`docs/FORMATOS-DE-SAIDA.md`](docs/FORMATOS-DE-SAIDA.md).

## O princípio

O sistema **não adivinha**. Preço não informado sai como `A mercado`; qualquer outro dado crítico
que falte bloqueia a geração e mostra o que falta. Ticker parecido com erro de digitação pede
confirmação, nunca corrige sozinho — a lista das 1.624 ações, FIIs, BDRs e ETFs da B3 está embutida
justamente para pegar um `KCNR11` no lugar de `KNCR11`. Ativo repetido não é somado sozinho.

Antes de gerar qualquer saída, um preview mostra o que foi interpretado, campo a campo, para você
corrigir na mão o que for preciso. Quase todo bloqueio tem um botão de confirmar ao lado: ele te
obriga a olhar, nunca te deixa sem saída. Os que não têm apontam a saída: escolher o fundo certo, ou
trocar para o formato que comporta a ordem.

## O que ele não faz

Estes casos continuam sendo feitos à mão, e o sistema os lista com o motivo em vez de tentar
adivinhar: ordem sem quantidade nem valor (`V - HGCR11 (Total)`) e Push, que é do Hub.

Fundos cetipados (sem ticker, identificados pelo nome) **são atendidos**, pelo mesmo modelo de
e-mail das ações, inclusive com o valor na linha de baixo do nome. Lote, TWAP e o E-mail em tabela
não: a coluna `Ativo` precisa de um código.

## Cole direto do WhatsApp

Pode colar o pedaço da conversa como ele está. O carimbo `[13:46, 21/09/2026] Fulano:`, os
telefones e os "Enviado mesa." são descartados — e o horário do carimbo nunca vira horário de
ordem. Duas contas no mesmo bloco viram duas solicitações separadas.

## Modo claro e escuro

O botão no canto superior direito alterna os dois. **Navegador novo abre sempre no escuro**, e a
partir do seu primeiro clique vale a sua escolha, gravada naquele navegador. Cada máquina guarda a
sua — o que é o esperado, já que cada estação tem sua tela e sua iluminação.

Os dois modos foram calibrados para quem passa o dia na ferramenta: sem preto puro no fundo, sem
branco puro no texto, contraste na faixa confortável em vez do máximo, cor saturada só onde ela
significa alguma coisa. O modo claro é um neutro morno em quatro níveis — a página afunda, o
cartão sobe sobre ela, o campo afunda de novo — e não uma chapa branca. As animações são curtas e
somem por completo se o sistema estiver configurado para reduzir movimento.

Cada classe de ativo tem sua cor no selo ao lado do ticker (ação, FII, BDR, ETF, Fiagro, fundo),
para achar um BDR no meio de uma lista de FIIs sem ler papel por papel. Ticker suspeito e ticker
fora da lista ficam com os únicos tons quentes da coluna, para continuarem sendo o que o olho
encontra primeiro.

## Como rodar

```bash
npm install
npm run dev      # servidor local de desenvolvimento
npm test         # testes do núcleo de parsing/validação/formatação
npm run build    # gera dist/index.html, um arquivo só, que funciona offline
```

Requer Node 20+. Não há dependências de runtime: o que vai para o ar é HTML, CSS e JS puro.

## Usar offline, sem instalar nada

`npm run build` gera **um único arquivo**, `dist/index.html`, com o JS, o CSS e o ícone dentro
dele. É esse arquivo que se entrega para quem vai usar: abre com dois cliques, sem internet, sem
servidor e sem Node. Pode ser renomeado, copiado para outra pasta, para um pendrive ou para a rede.

O `index.html` da raiz do projeto **não** funciona aberto com dois cliques: ele é a fonte que o
Vite lê, e o navegador bloqueia os scripts que ele carrega quando a página vem do disco. A tela
aparece, mas nenhum botão responde.

O tema e o histórico ficam gravados no navegador de cada pessoa, como na versão online.

## Deploy

Deploy estático na Netlify, já configurado em `netlify.toml` (build `npm run build`, publica `dist/`).
Qualquer host de estáticos serve igual.

## Uso

1. Cole a solicitação na área de texto.
2. Escolha o formato de saída, ou deixe a detecção automática por palavras-chave sugerir:
   `via email` marca os dois e-mails (texto e tabela; com fundo cetipado na cesta, só o texto),
   `auditar` marca só a tabela, e `lote` e `twap` marcam os seus.
3. Confira o preview e corrija o que estiver errado. O `×` tira uma linha; o **+ Adicionar ativo**
   acrescenta uma em branco, para preencher à mão.
4. Resolva os bloqueios, se houver.
5. Copie. E-mail e lote podem ser gerados na mesma passada, sem redigitar nada.

As últimas ordens geradas ficam no histórico local do navegador (`localStorage`), para reabrir e
reaproveitar.

## Estrutura

```
src/core/        parsing, validação e formatação — funções puras, sem DOM. É onde ficam os testes.
  texto/         limpeza da colagem do WhatsApp
  parse/         quebra por conta, leitura de quantidade × financeiro, preço e horário
  validate/      lista de tickers da B3, suspeita de digitação, bloqueios e avisos
  format/        e-mail, Lote Simples e Lote TWAP
src/ui/          preview editável e render das saídas
src/platform/    adaptadores de browser (clipboard, histórico)
docs/            especificação literal dos formatos de saída
```

São 449 testes cobrindo as regras de negócio, incluindo pedidos reais colados do grupo.

Convenções e regras de negócio para quem (ou o que) for mexer no código: [`CLAUDE.md`](CLAUDE.md).
Defeitos conhecidos e próximos passos: [`docs/ROADMAP.md`](docs/ROADMAP.md).

## Privacidade

Tudo roda no navegador. Nenhum dado de cliente sai da máquina: não há backend, não há telemetria,
não há chamada de rede. O histórico fica no `localStorage` do próprio navegador.
