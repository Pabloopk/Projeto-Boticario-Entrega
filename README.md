# Vitrine Multilíngue com Catálogo Automatizado e Venda por WhatsApp

<img width="1890" alt="O Boticário Brussels" src="./Captura%20de%20tela%202026-09-20%20091929.png" />



## Projeto em Produção
 https://marilia-boticario.vercel.app

---

##  Contexto

Desenvolvimento e implantação de uma loja online completa para uma revendedora de produtos O Boticário em Bruxelas, cujo catálogo existia apenas em um PDF de 148 páginas e 460 MB.

O objetivo foi transformar esse material estático em uma **vitrine digital em quatro idiomas**, administrada pela própria cliente, que leva o visitante direto ao WhatsApp já sabendo o que quer comprar.

 Registro técnico detalhado das decisões: [ESTUDO-DE-CASO.md](./ESTUDO-DE-CASO.md)

---

## O que foi implementado

- Extração automatizada de 625 produtos a partir do catálogo em PDF  
- Resolução de 589 fotos oficiais da marca via API pública de busca  
- Vitrine completa com grade, filtros, categorias e páginas de produto  
- Compositor determinístico de mensagens de WhatsApp com lista de pedido  
- Painel administrativo próprio para estoque, preço e cadastro de produtos  
- Internacionalização em quatro idiomas com rotas dedicadas e SEO  
- Design system documentado e validado por contraste WCAG  
- Deploy completo em ambiente de produção  

---

## Arquitetura e Infraestrutura

A aplicação foi estruturada com foco em **performance, autonomia da cliente e custo de operação próximo de zero**, incluindo:

- Renderização dinâmica com cache e invalidação por tag  
- Banco de dados Postgres em nuvem com alta disponibilidade  
- Armazenamento de imagens com hash de conteúdo e cache imutável  
- Camada de acesso a dados tipada de ponta a ponta  
- Deploy do frontend em ambiente serverless  
- Configuração segura de parâmetros em ambiente de produção  

---

##  Impacto no Negócio

✔ Catálogo de 625 produtos disponível publicamente, sem intermediários  
✔ Atualização de estoque e preço feita pela cliente, sem depender de desenvolvedor  
✔ Alcance ampliado para o público francófono, anglófono e hispanófono  
✔ Entrada no WhatsApp já com produto, referência e preço definidos  
✔ Redução do atrito entre a descoberta do produto e o pedido  

---

##  Tecnologias Utilizadas

<p align="left">

<!-- Frontend / Deploy -->
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/typescript/typescript-original.svg" width="40"/>
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/react/react-original.svg" width="40"/>
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/nextjs/nextjs-original.svg" width="40"/>
<img src="https://assets.vercel.com/image/upload/front/favicon/vercel/180x180.png" width="40"/>

<!-- Estilo -->
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/tailwindcss/tailwindcss-original.svg" width="40"/>

<!-- Database / Services -->
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/postgresql/postgresql-original.svg" width="40"/>
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/nodejs/nodejs-original.svg" width="40"/>

</p>

---

##  Segurança

- Autenticação com hash memory-hard e sessões persistidas em banco  
- Cookies com prefixo `__Host-`, `httpOnly`, `Secure` e `SameSite`  
- Freio de força bruta por IP, sem dependência de serviço externo  
- Trilha de auditoria com estado anterior e posterior de cada edição  
- Cabeçalhos de segurança, CSP e painel fora do índice de busca  
- Validação de sessão em cada Server Action, não apenas no middleware  

---

##  Domínio e Hospedagem

- Aplicação disponível em domínio público com HTTPS ativo  
- Deploy contínuo integrado ao repositório  
- Ambiente com certificado SSL e cache de borda  
- Estrutura desacoplada entre vitrine pública e painel administrativo  

---

##  Diferencial

Essa solução vai além da implementação técnica:

 transforma um catálogo impresso em vitrine digital pesquisável  
 devolve à cliente o controle total sobre estoque e preço  
 escala o alcance para novos idiomas sem aumentar a operação  

---

##  Contato

Se você busca digitalizar um catálogo, internacionalizar um produto ou escalar sua operação:

 Entre em contato para desenvolver uma solução sob medida.
