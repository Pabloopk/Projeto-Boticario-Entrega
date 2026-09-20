# O Boticário Brussels — estudo de caso
![Uploading image.png…]()


**Vitrine multilíngue com catálogo extraído de PDF, painel próprio e conversão por WhatsApp.**

Uma revendedora brasileira de produtos O Boticário em Bruxelas vendia por Instagram e conversa
direta. O catálogo dela era um PDF de 148 páginas e 460 MB. Não havia site, não havia estoque
digital, e cada pedido nascia de um print de página de catálogo mandado no privado.

Este é o registro de como esse PDF virou uma loja online em quatro idiomas que ela mesma
administra — e das decisões técnicas que a forma do problema exigiu.

| | |
|---|---|
| **Papel** | Desenvolvedor único — levantamento, arquitetura, implementação e entrega |
| **Período** | 7 a 18 de setembro de 2026 |
| **Stack** | Next.js 16 · React 19 · TypeScript · Tailwind CSS 4 · Neon Postgres · Drizzle ORM |
| **Escala** | 625 produtos · 9 categorias · 4 idiomas · 589 fotos oficiais |
| **Cliente** | Marília Alves · Bruxelas, Bélgica |

---

## O problema, em uma frase

**Mostrar 625 produtos que só existiam num PDF, em quatro idiomas, para um público que compra por
WhatsApp — e deixar que a dona atualize estoque e preço sozinha, sem tocar em código.**

Três restrições moldaram tudo:

1. **Sem checkout.** A venda acontece na conversa. O site é uma vitrine com uma função: levar o
   cliente ao WhatsApp já sabendo o que ele quer.
2. **Sem equipe.** Quem opera o site é uma pessoa só, que não é técnica. Qualquer fluxo que exigisse
   deploy, terminal ou planilha estaria morto na primeira semana.
3. **Orçamento de infraestrutura próximo de zero.** A solução tinha que caber em planos gratuitos ou
   quase.

---

## Fase 1 — Arrancar 625 produtos de um PDF

O ponto de partida era `CATALOGO_10.pdf`: 148 páginas, 460 MB, layout de revista com preço, nome e
código de um mesmo produto espalhados por colunas diferentes.

`pdftotext -layout` preserva a posição das colunas, mas entrega uma leitura embaralhada — o nome de
um produto e o preço dele podem aparecer separados por dezenas de linhas. A extração foi dividida em
oito blocos processados em paralelo, com **uma regra explícita: omitir qualquer produto cuja
associação preço–nome fosse ambígua.**

> Preferir a lacuna ao dado errado é o que torna o resultado utilizável. Um produto faltando é um
> produto que alguém acrescenta depois. Um preço errado é uma venda com prejuízo.

Resultado da primeira passagem: **410 produtos únicos, todos com preço.** O Ciclo 11, publicado uma
semana depois, foi somado por um script que **só acrescenta códigos novos** — os produtos que a dona
já tinha ajustado ficaram intocados. O catálogo chegou a 625.

### O problema das variantes de cor

A extração achatou os batons: um registro por código, sem o nome da cor. Na grade, isso são nove
cards idênticos de "Batom Cremoso Hidratante".

Uma segunda extração dirigida ao PDF recuperou **51 nomes** de cor e fragrância — *warm cashmere*,
*douradex*, *Pessegura* — cada um com grau de confiança registrado para auditoria posterior.

Três resultados foram **descartados de propósito**: dois devolveram "Malbec", que é a linha e não a
cor, e um devolveu "refil", contaminação de layout. Sobraram sete produtos marcados `needsReview`,
que renderizam como cards separados identificados pelo código — funcional, apenas menos elegante,
até alguém conferir no impresso.

### Os três formatos de preço

O catálogo não usa um formato só, e isso definiu boa parte da arquitetura. O card, a página de
produto e a mensagem de WhatsApp precisam tratar os três:

| Formato | Exemplo |
|---|---|
| Simples | € 19,99 |
| Com desconto | € 23,99 → € 20,39, economize € 3,60 |
| Escalonado por quantidade | € 14,99 · 2 un € 11,99/un · 3+ un € 10,49/un |

---

## Fase 2 — As fotos oficiais que não tinham URL

O requisito era usar as fotos oficiais da marca. A hipótese natural — montar a URL da imagem a
partir do código do catálogo — foi testada e **reprovada**:

| Código | Resultado |
|---|---|
| `4482` | 200 OK |
| `63794` | 404 |
| `87237` | 404 |

A loja portuguesa roda em Shopify, e existem três esquemas de numeração concorrentes: SKU da
Shopify, sufixo do handle e código da sessão fotográfica. Quando o código do catálogo bate, é
coincidência.

### A solução: a API de busca pública, e o campo que ninguém olha

A Shopify expõe um endpoint JSON sem autenticação:

```
GET https://oboticario.pt/search/suggest.json?q=<nome>&resources[type]=product
```

Duas armadilhas encontradas na prática: **só funciona sem `www`** (com o prefixo, 404), e **não
aceita busca por código** — pesquisar `63794` não devolve nada. A busca precisa ser por nome.

Mas buscar por nome erra. As cinco variantes de Lily têm títulos quase idênticos — Regular, Absolu,
Lumière, Cashmere, Love Lily. O achado que fez o método funcionar foi o campo **`price`** da
resposta: a Cashmere custa € 27,99 contra € 23,99 das demais. Cruzando **nome + tamanho + preço**, o
candidato certo é identificado com segurança.

O script pontua cada candidato e classifica a confiança do match:

```
cobertura de tokens do nome    55%
precisão (penaliza extras)     15%
preço exato                   +25%
tamanho confere                +5%

≥ 0,80 alta · ≥ 0,60 média · ≥ 0,50 baixa · abaixo: sem match
```

Ele faz pausa de 400 ms entre requisições, grava o progresso a cada 10 produtos — dá para
interromper e retomar — e **gera um relatório em Markdown com todos os matches de confiança média ou
baixa**, para conferência humana. Automatizar o que é determinístico e entregar o resto numa lista
revisável é mais honesto do que fingir 100% de acerto.

**589 fotos** resolvidas. As imagens são baixadas, não hotlinkadas: são as oficiais, mas passam a
viver no repositório, e o site não quebra se a marca reorganizar o CDN.

### Uma regra dura que vale registrar

A API devolve um campo de preço. Ele é usado **apenas** como sinal para escolher a foto certa —
nunca é gravado no catálogo nem exibido. A loja online tem promoções próprias, diferentes das do
ciclo impresso, e os dois valores podem divergir. A fonte única de verdade dos preços é o catálogo.

É o tipo de confusão que só aparece três meses depois, num preço errado que ninguém sabe explicar.
Por isso está escrito na especificação em maiúsculas.

---

## Fase 3 — O compositor de mensagens

O pedido central do projeto: *ler o produto que o cliente quer, descrevê-lo brevemente e enviar por
WhatsApp.*

Sem backend, isso é um **montador determinístico** — não é IA. Ele compõe texto a partir dos dados
estruturados do produto, com a sintaxe de formatação do próprio WhatsApp:

```
Olá! Tenho interesse neste produto:

*Ácido Poliglutâmico Acqua Gel Hidratante Antissinais*
Botik · 50 g
Ref: 4480
Preço: *€ 17,99*
2 unidades: € 14,39 cada
3+ unidades: € 12,59 cada

Hidratação intensiva desde a 1ª aplicação.

Está disponível?
```

Há também uma **lista de pedido** que acumula produtos no navegador e envia tudo numa mensagem só,
com os descontos por quantidade já aplicados no total.

Três decisões sustentam o módulo:

- **Nunca inventar texto.** 103 produtos não têm descrição no catálogo. Para esses, a frase é montada
  a partir de linha, categoria e tamanho. Escrever copy de marketing por conta própria seria pôr
  palavras na boca da marca.
- **Degradar antes de quebrar.** URLs de `wa.me` falham **em silêncio** acima de ~2.000 caracteres —
  o pior modo de falha possível, porque o cliente só vê nada acontecer. Passando do limite, a
  mensagem remove as descrições; ainda longa, resume os itens excedentes.
- **Total sempre "estimado".** Não inclui frete nem confirmação de estoque, e o texto diz isso.

> **Verificado:** um pedido de 25 itens gera uma URL de 1.677 caracteres, dentro do limite de 1.800.
> Os descontos por quantidade conferem — três unidades do código 4480 somam € 37,77, não € 53,97.

O módulo é de funções puras `Produto → string`, isolado e testável. Foi construído assim porque é a
peça de que depende toda a receita do negócio.

---

## Fase 4 — De site estático a site editável

A primeira versão era 100% estática (`output: 'export'`). Rápida, gratuita de hospedar, e com um
defeito fatal: **qualquer mudança de estoque exigia um rebuild.** Na prática, exigia eu.

O objetivo da fase foi eliminar isso. Duas rotas possíveis:

| | **A — Dinâmico com cache** | **B — Estático + rebuild por webhook** |
|---|---|---|
| Mudança aparece em | ~1 segundo | 1 a 3 minutos |
| Risco | Baixo | Build falha e a edição "some" sem explicação para ela |

**Escolhi A.** Com invalidação por tag, o site público continua servido de cache — a mesma velocidade
de antes, na prática — e a edição aparece na hora. O caminho B trocava um segundo por três minutos e
adicionava um modo de falha silencioso na mão de quem não sabe ler log de build.

### Por que Neon Postgres

| Alternativa | Por que não |
|---|---|
| Supabase | Bom, e traz auth pronta. Mas traz junto realtime, storage e edge functions que não uso, e a auth acopla o projeto à plataforma |
| SQLite / Turso | Mais barato, mas Postgres tem `numeric` para preço, arrays para badges e `jsonb` para as faixas de quantidade — tipos que o domínio pede |
| **Neon** | Postgres puro, sem lock-in. Se um dia precisar sair, é um `pg_dump` |

### A decisão de produto que desenhou a tela

Conversando sobre o uso real, ficou claro que **o que ela mexe todo dia é estoque.** Preço e texto
são exceção; produto novo, de vez em quando.

Isso decidiu a tela principal: **o estoque se edita direto na lista**, num campo numérico que salva
sozinho — sem abrir o editor do produto, sem botão "salvar" geral. Abrir um formulário inteiro para
trocar "3" por "2" seria atrito na única tarefa que se repete.

O painel cobre produto e nada mais. `/admin/conteudo` e `/admin/destaques` estavam planejados e
foram **dispensados** depois de conversar: textos do site mudam uma vez por ano, e cada tela a mais é
uma tela a manter.

### Imagens dentro do Postgres

Uma escolha pouco ortodoxa, feita de propósito. O `id` de cada imagem é o **SHA-256 do conteúdo**,
o que dá duas coisas de graça:

- a mesma foto enviada duas vezes ocupa uma linha só;
- a URL é imutável por construção — mudou a foto, mudou a URL — então o cache pode ser eterno
  (`max-age=31536000, immutable`) e o banco quase nunca é consultado.

O navegador reduz a foto para 1200 px antes de subir: 5 MB de câmera de celular viram ~200 KB. O
servidor confere pelos *magic bytes* que é imagem de verdade, porque o tipo declarado pelo navegador
pode ser forjado.

Isso dispensou criar conta em mais um serviço e manter mais um token. Cem fotos ocupam uma fração do
plano gratuito. Migrar para um storage dedicado, se um dia fizer sentido, é um script.

### Preço promocional é regra, não valor

O banco guarda o preço normal e a **regra** de desconto — não o resultado:

```
priceSale = preco_promo ?? (desconto_pct ? arredondar(price × (1 − desconto_pct/100), 2) : null)
```

Ela mexe no percentual sem calcular nada na mão, mas ainda pode cravar um preço específico quando
quiser. A conversão acontece num lugar só, `paraProduto()`, e um "desconto" que não baixa o preço é
descartado — para nunca aparecer preço riscado à toa.

---

## Segurança: o pedido explícito da cliente

"Meus dados não podem vazar." O que foi implementado, e por quê:

| Medida | Razão |
|---|---|
| **scrypt** (N=32768, r=8, p=1) para senha | Memory-hard, encarece muito ataque com GPU, e vem no Node — zero dependências para falhar no deploy. `bcryptjs` puro em JS levaria mais de 1 s por login |
| **Só o SHA-256 do token de sessão no banco** | Quem lê a tabela `sessoes` não consegue montar um cookie válido |
| Cookie `__Host-` + `httpOnly` + `Secure` + `SameSite=Lax` | O prefixo `__Host-` impede que um subdomínio comprometido escreva o cookie do painel; `httpOnly` impede XSS de roubar a sessão |
| Sessão em tabela, 7 dias, logout apaga a linha | A sessão morre de verdade — diferente de um JWT, que continua válido até expirar |
| **Freio de força bruta** na própria tabela | 5 falhas do mesmo IP em 15 min → atraso crescente e bloqueio. Sem Redis, sem serviço externo |
| Mesma resposta e mesmo tempo para e-mail errado e senha errada | Um atacante não descobre o e-mail dela testando |
| Sem rota de cadastro e sem "esqueci minha senha" | Recuperação por e-mail é a superfície mais explorada em painéis pequenos. Contas se criam por CLI |
| Trilha de **auditoria** com estado antes/depois | Se algo for editado errado, dá para ver o que era |
| `noindex` + `X-Frame-Options: DENY` + CSP + `nosniff` | O painel não existe para o Google, nem para um iframe alheio |
| **Nenhuma variável com `NEXT_PUBLIC_`** | O prefixo embute o valor no JS enviado ao navegador. É o erro clássico que vaza credencial em projeto Next, e está escrito como regra do projeto |

**O ponto mais importante:** o `middleware.ts` faz apenas a triagem barata — tem cookie? não tem,
vai para o login. **A validação real acontece dentro do layout do painel e de cada Server Action**,
consultando a tabela de sessões. Tratar o middleware como a barreira é uma falha conhecida em
projetos Next; aqui ela está documentada no código para não ser reintroduzida.

O trabalho foi verificado com **12 checagens automatizadas** — entre elas: sessão expirada não vaza
conteúdo do painel, conta desativada perde acesso na hora, cookie forjado para na tela de login, e
apagar a conta apaga as sessões em cascata. Todas passando.

---

## Multilíngue: quatro idiomas sem esquecer nenhum

O público é diverso — brasileiros na Europa, mas também belgas francófonos. A loja fala **pt-BR, fr,
en e es**; o painel continua só em português, porque quem o usa é uma pessoa só.

Toda página vive sob um prefixo (`/fr/produto/malbec-icon/`). A raiz e links antigos são
redirecionados nesta ordem: cookie da última visita → `Accept-Language` do navegador → português.
Cada página declara suas traduções em `hreflang` e o `sitemap.xml` lista as quatro versões — é assim
que o Google entende que são traduções, não conteúdo duplicado.

### O detalhe de que mais me orgulho

O tipo `Dicionario` é **derivado** do arquivo português. Consequência: adicionar uma chave em
`pt-br.ts` **quebra a compilação** de `fr.ts`, `en.ts` e `es.ts` até que alguém traduza.

Não é possível esquecer uma tradução. O compilador não deixa.

### O que deliberadamente **não** é traduzido

As mensagens de WhatsApp saem sempre em português, com formatação de preço brasileira, mesmo quando
o visitante navega em francês. **Quem lê do outro lado é a dona** — um pedido em espanhol seria um
pedido a decifrar antes de separar.

Isso aparece num detalhe fino do formulário de contato: o `<select>` guarda o **índice** do assunto,
não o texto. Quem escolhe "Question sur un produit" no site francês manda "Dúvida sobre um produto".
Só o texto livre que o visitante digitou vai como ele escreveu.

Preços, esses sim, seguem a convenção de cada idioma — `€ 20,39` em pt-BR, `20,39 €` em fr,
`€20.39` em en.

---

## Redesign: quando medir muda o desenho

O visual inicial foi herdado do tema Shopify do site antigo: cantos retos, uma fonte só, seções
empilhadas sem hierarquia. Funcionava, mas lia como catálogo, não como marca de cosmético.

O redesign trouxe profundidade de cor, título serifado, formas arredondadas e respiro entre seções —
**preservando o verde `#2B926A`**, que foi extraído do site real da cliente. Trocá-lo por um teal
genérico descartaria identidade já construída.

Mas a medição de contraste revelou um problema: **`#2B926A` com texto branco rende 3,87:1 e reprova
o AA da WCAG.** A cor da marca não pode carregar texto.

A solução foi derivar uma escala em volta dela e atribuir papéis explícitos:

| Token | Hex | Papel | Contraste |
|---|---|---|---|
| `brand-deep` | `#12443A` | Fundo de CTA, header, overlay | 10,97:1 — AAA |
| `brand-dark` | `#1C6B4F` | Verde para texto sobre claro | 6,43:1 — AA |
| `brand` | `#2B926A` | **Forma e cor, nunca leitura** | 3,87:1 |
| `brand-soft` | `#6FB89C` | Só sobre fundo escuro | 2,33:1 sobre branco |

A mesma disciplina separou `line` (divisor decorativo) de `line-strong` (borda de controle): a WCAG
1.4.11 exige 3:1 para qualquer borda que seja a única indicação de um input. É a distinção que mais
se erra em design system.

O redesign foi executado sob **restrições duras escritas antes de começar**: nenhuma linha de `lib/`,
de Server Action, de middleware ou de contexto de estado poderia ser tocada. Só classes de
apresentação. Isso manteve um redesign visual completo sem risco de regressão funcional — e o
escopo escrito é o que impede o "já que estou aqui".

O sistema virou documentação (`DESIGN-SYSTEM.md`) e uma skill de agente que aplica os tokens
automaticamente em qualquer trabalho futuro de interface.

---

## Linha do tempo

| Data | Entrega |
|---|---|
| 07/09 | Extração do PDF, 410 produtos, vitrine estática com hero, abas, cards e páginas de produto |
| 08–09/09 | Regra de desconto unificada · migração para Neon + painel de administração |
| 10/09 | Tradução para francês, inglês e espanhol · catálogo completo com 510 produtos |
| 13/09 | Design system documentado e redesign visual aplicado em toda a vitrine |
| 16/09 | Ciclo 11 incorporado (625 produtos) e imagens faltantes resolvidas |
| 18/09 | Ajustes finais de botão de compra e lista de pedido |

---

## Números

| | |
|---|---|
| Produtos no catálogo | **625** |
| Categorias | **9** |
| Idiomas | **4** |
| Fotos oficiais resolvidas | **589** |
| Tabelas no banco | **6** (produtos, imagens, conteúdo, admins, sessões, auditoria) |
| Dependências de produção | **6** |
| Vulnerabilidades | **0** |
| Origem do catálogo | PDF de **460 MB**, 148 páginas |

---

## O que eu levo deste projeto

**Preferir a lacuna ao dado errado.** A regra que governou a extração do PDF é a mesma que governa o
matching de imagens e o compositor de mensagens. Um produto faltando é visível e alguém conserta. Um
preço errado ou uma descrição inventada passam despercebidos até virarem prejuízo.

**O modo de falha importa mais que a taxa de falha.** URLs de `wa.me` quebram em silêncio; por isso o
compositor degrada em dois estágios antes do limite. Builds automáticos falham em silêncio para quem
não lê log; por isso o site é dinâmico com cache, e não estático com webhook.

**Desenhar para a tarefa que se repete.** O painel inteiro foi organizado em volta da descoberta de
que o uso diário é ajustar estoque. Duas telas planejadas foram cortadas depois de perguntar o que
ela realmente faria com elas.

**Automatizar o determinístico e entregar o resto revisável.** O matcher de imagens não finge 100% de
acerto: pontua, classifica a confiança e gera uma lista de conferência humana. Fingir certeza onde
não há é o que transforma automação em dívida.

**Medir antes de decidir estética.** A decisão mais estrutural do redesign — o CTA não usa a cor da
marca — não veio de gosto nem da referência visual. Veio de um número: 3,87:1.

---

*Pablo Nunes · setembro de 2026*
