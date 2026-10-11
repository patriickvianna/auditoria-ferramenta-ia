---
name: auditoria-ferramenta-ia
description: Audita, testa e corrige uma ferramenta feita com IA (app, site, painel) — segurança, funcionamento, números certos, clareza para o usuário e visual sem cara de IA — com relatório priorizado e correções em lotes.
---

# Auditoria de ferramenta feita com IA

Use quando o usuário pedir para auditar, revisar, testar, "deixar pronto para produção" ou "tirar a cara de IA" de um app, site, painel ou ferramenta, principalmente se foi gerado com IA (Lovable, Bolt, v0, Cursor, Claude, etc.).

Outras skills citadas ao longo do texto (como `supabase` ou `design:accessibility-review`) são **opcionais**: use se estiverem instaladas; se não estiverem, siga só as instruções desta skill.

O objetivo é uma ferramenta segura, que funciona em todos os caminhos, que mostra números certos e que qualquer pessoa entende sem explicação. A ordem de prioridade nunca muda: **segurança e disponibilidade, depois funcionamento e números certos, depois clareza, depois visual.**

## Regras que valem em todas as fases

- **Evidência antes de afirmar.** Cada achado tem onde está (arquivo:linha, tela, rota) e como foi verificado (comando rodado, teste feito, print). Sem evidência, o item vai como "suspeita", nunca como fato.
- **Falha conhecida se confere na hora, não de memória.** Versão vulnerável de biblioteca, limite de plano, preço de serviço: consulte a fonte oficial (aviso do fabricante, base CVE/GHSA, página de preços) no momento da auditoria e coloque o link no relatório. Se não der para consultar, marque como "suspeita — conferir".
- **Não quebrar nada nem tocar em dado real.** Trabalhe numa branch ou cópia. Nunca rode testes que escrevem dados contra produção ou contra dados reais de clientes sem permissão explícita. Para testar sem produção, use uma das saídas da seção "Como testar sem produção".
- **Nunca exiba segredos.** Ao achar uma chave ou senha, mostre só o nome da variável e os 4 primeiros caracteres, e recomende trocar a chave.
- **Mudanças pequenas.** Corrija em lotes curtos, re-teste depois de cada lote. Nunca reescreva o app inteiro. Mudanças grandes de fluxo ou de identidade visual: proponha e pergunte antes.
- **Relatório antes de correção.** Nada é corrigido antes de o usuário ver o quadro completo, exceto se ele pedir.
- **Diga o que não deu para verificar** (sem acesso ao banco, sem ambiente para rodar, sem login de teste, serviço fora do ar).
- **Urgência fala na hora.** Se aparecer algo que está quebrando a ferramenta agora (site ou banco fora do ar, chave vazada), avise o usuário imediatamente e continue a auditoria em paralelo.

## Fase 0: Reconhecimento (antes de qualquer crítica)

1. **Versão auditada:** identifique exatamente o que está no ar (commit, data do deploy, URL) e confirme que é a mesma versão que você vai ler. Anote no relatório.
2. **Stack:** framework, banco, autenticação, hospedagem, serviços externos e APIs de IA usadas.
3. **Promessa:** escreva em 2 linhas o que a ferramenta promete e para quem.
4. **Jornadas:** liste de 3 a 6 jornadas principais do usuário (ex.: cadastrar, entrar, criar um registro, importar, pagar, exportar).
5. **Infraestrutura e limites do plano (sempre):**
   - O site e o banco estão no ar agora? (status do projeto no painel/API do provedor, uma requisição real à URL)
   - O plano tem pausa por inatividade? Quando foi o último uso?
   - Espaço usado × limite do plano (banco, storage, banda) e quanto o crescimento previsto consome.
   - Existe backup que o dono consegue restaurar ou baixar? Com que frequência?
   - Domínio e certificado válidos e com vencimento conhecido.
6. **Como rodar:**
   - **Projeto com estrutura (package.json etc.):** rode `install`, `build`, `lint`, `typecheck` e os testes existentes; anote o resultado.
   - **App sem estrutura (HTML único, scripts por CDN, no-code):** liste cada biblioteca carregada de fora, com a versão; confira se a versão é fixa (não `@latest`/`@2`) e se tem verificação de integridade (atributo `integrity`/SRI); leia a política de segurança do navegador (CSP) e anote o que ela libera.
7. Se faltar algo essencial (acesso ao código, URL, login de teste), peça uma vez, de forma objetiva.

## Fase 1: Segurança

Apps gerados por IA costumam falhar sempre nos mesmos pontos. Verifique cada um:

**Segredos**
- Chaves no código ou no repositório: procure `sk-`, `service_role`, `SECRET`, `PRIVATE_KEY`, `password`, tokens de API e arquivos `.env` versionados.
- Chave secreta exposta no navegador: variáveis `NEXT_PUBLIC_`, `VITE_` ou `REACT_APP_` com segredo, ou a chave `service_role` do Supabase usada no cliente.
- Chamadas a APIs de IA (OpenAI, Anthropic etc.) feitas direto do navegador com a chave do dono.
- Repositório público: confirme que nada além do que é público por natureza (URL do banco, chave anônima) aparece.

**Banco e autorização** (com Supabase, siga também a skill `supabase` e use os advisors de segurança)
- RLS ativado em todas as tabelas, com policies que restringem por usuário (`auth.uid()` ou e-mail do token). Policy `using (true)` em tabela com dado pessoal é crítica.
- **Autocadastro:** o usuário consegue criar o próprio registro já como aprovado/admin? Confira o `WITH CHECK` de INSERT das tabelas de usuário e papel.
- **Usuário bloqueado ou pendente** ainda consegue ler dados por alguma função ou policy?
- Toda rota de API, server action e função do banco (RPC) confere quem está logado e se aquele usuário pode acessar aquele registro. Teste trocar o ID na URL, no corpo da requisição ou no parâmetro da função (IDOR).
- Regras de negócio (preço, permissão, limite de uso, hierarquia) aplicadas no servidor, não só escondidas na interface. **Tudo que dá para fazer pelo console do navegador (F12) deve ser exatamente o que a tela permite.**

**Entrada de dados**
- Validação no servidor (ex.: zod, constraint, policy) em formulários, APIs e uploads.
- XSS: `dangerouslySetInnerHTML`, `innerHTML` ou markdown renderizado sem sanitizar. Faça o teste dinâmico da seção "Receitas de teste".
- SQL montado com concatenação de texto.
- Upload e leitura de arquivo: tipo, tamanho, onde fica público e **versão da biblioteca que lê o arquivo** (Excel, PDF, imagem) conferida contra falhas conhecidas.

**Dependências e cadeia de fornecimento**
- Com gerenciador de pacotes: `npm audit` (ou equivalente), listando só alto e crítico.
- Sem gerenciador: cada biblioteca de CDN conferida contra a base de falhas, versão fixa, SRI, e CSP restrita (de preferência hospedar as bibliotecas no próprio site e deixar `script-src 'self'`).

**Exposição e abuso**
- CORS aberto (`*`) em rotas autenticadas.
- Falta de limite de tentativas em login, cadastro, recuperação de senha e rotas que chamam IA (custo).
- Mensagens de erro ou logs que mostram stack trace, dados pessoais ou chaves.
- Dados pessoais (LGPD): o que é coletado, onde fica, quem acessa, por quanto tempo é guardado, se dá para apagar, se existe política de privacidade.

## Fase 2: Funcionamento e números certos

1. Para cada jornada da Fase 0, percorra de ponta a ponta. Se houver navegador disponível, teste de verdade; se não, leia o código do fluxo inteiro.
2. **Números conferidos contra a fonte.** Em ferramenta que calcula ou analisa, escolha pelo menos um caso real (um relatório, um extrato, uma planilha oficial) e feche o número da ferramenta com a fonte, item a item. Toda diferença é explicada ou vira achado. Esse costuma ser o achado mais valioso.
3. Em cada tela, force estes estados e veja se existem e fazem sentido: **vazio, carregando, erro, sucesso, texto muito longo, muitos itens, rede lenta ou sem internet.**
4. Teste o que a IA costuma esquecer: clique duplo em botão de enviar, voltar do navegador, recarregar no meio do fluxo, sessão expirada, celular (360px de largura). Use as receitas abaixo.
5. Abra o console: erros e avisos em vermelho entram no relatório (separe o que é do app do que é do ambiente de teste).
6. Procure restos de geração por IA: dados falsos fixos no código, `TODO`, funções mock, botões que não fazem nada, links para `#`, páginas duplicadas, código morto (meça quantas linhas), componentes quase iguais copiados.
7. Se o projeto já tem testes, rode todos e anote o placar. Para cada bug corrigido, escreva antes um teste que falha e depois passa (veja `superpowers:test-driven-development`).
8. Em React ou Next.js, aplique `vercel-react-best-practices`: requisições em cascata, re-renderizações desnecessárias, bundle pesado.

## Receitas de teste (use sempre que possível)

- **Injeção de código (XSS) dinâmica:** preencha todo campo que vem do banco ou de arquivo importado (nomes, descrições, observações, campos do arquivo) com `"><img src=x onerror="window.__xss=(window.__xss||0)+1">`. Percorra todas as telas e modais e, no fim, leia `window.__xss` e conte `img[src="x"]` no DOM. Qualquer valor acima de zero é crítico.
- **Duplo clique:** atrase as respostas de gravação em ~400 ms (rede lenta simulada), dê duplo clique em cada botão de criar/salvar e conte quantas gravações saíram. Mais de uma é achado.
- **Contraste:** calcule a razão WCAG de cada par cor de texto × cor de fundo da paleta (fórmula de luminância relativa). Texto normal precisa de 4,5:1; texto grande, 3:1. Não avalie no olho.
- **Celular:** abra em 360 px de largura e meça `document.documentElement.scrollWidth`. Se passar de 360, a página quebra no celular. Tire print das telas principais.
- **Recarregar no meio:** faça a jornada principal, aperte F5 no meio e confira se o usuário continua de onde parou ou recebe aviso claro.

## Como testar sem produção

- **Banco simulado no navegador:** substitua o cliente do banco por um falso em memória (ex.: um `createClient` que responde com tabelas em JavaScript) e rode a interface real com navegador automatizado (Playwright). Serve para telas, fluxos, XSS e duplo clique.
- **Cópia local das regras de acesso:** recrie em um Postgres local só a estrutura, as funções e as policies do banco (sem dados reais), com uma função que imita o token do usuário, e rode os ataques de permissão como usuários diferentes, dentro de transação revertida.
- **Leitura em produção, só leitura:** consultas `select` e advisors podem rodar contra produção; qualquer escrita, só com permissão explícita.

## Fase 3: Clareza para o usuário

A pessoa precisa entender a ferramenta sem tutorial. Em cada tela, ela deve conseguir responder em 5 segundos:

- **Onde estou?** Título claro e item ativo na navegação.
- **O que faço aqui?** Uma ação principal por tela, visualmente mais forte que as outras.
- **O que acontece se eu clicar?** Botões com verbo e objeto ("Salvar cliente", não "OK" ou "Enviar").

Verifique também:
- **Termos consistentes:** a mesma coisa tem o mesmo nome em todas as telas. Monte um mini glossário e unifique. Marque jargão interno e abreviações ("Tkt", "base do aprendizado").
- **Formato regional:** números, porcentagens, moeda e datas no padrão do usuário (no Brasil: `6,2%`, `R$ 1.234,56`, `dd/mm/aaaa`) e iguais em todas as telas. Ponto e vírgula misturados entram no relatório.
- **Mensagens de erro** dizem o que aconteceu e o que fazer ("O CPF tem 11 números. Confira e tente de novo."), nunca só "Erro" ou códigos técnicos.
- **Estados vazios** explicam o que vai aparecer ali e oferecem o primeiro passo.
- **Formulários:** rótulo visível acima do campo (não só placeholder), validação ao sair do campo, campos obrigatórios marcados, só os campos necessários. Campo editável parece editável.
- **Ações destrutivas** (apagar, cancelar) pedem confirmação e, quando possível, permitem desfazer.
- **Feedback:** toda ação mostra que funcionou (aviso curto, mudança visível).
- **Navegação:** no máximo 5 itens principais; nada importante escondido em menu de três pontinhos.
- **Acessibilidade básica:** contraste calculado (receita acima), tudo funciona pelo teclado, foco visível, botões de ícone com `aria-label` (veja `design:accessibility-review`).
- **Textos da interface** sem jargão técnico e sem frases de marketing vazias (use `stop-slop` e `design:ux-copy`).

## Fase 4: Visual sem cara de IA

Marque cada sinal típico de interface gerada por IA que aparecer:

- Gradiente roxo e azul, fundo com bolhas desfocadas, glassmorphism sem motivo.
- Emoji usado como ícone, ícone de "brilhinho" para tudo que é IA.
- Cards com borda colorida só na esquerda, sombras pesadas, cantos muito arredondados em tudo.
- Tudo centralizado, hero gigante sem conteúdo útil, três cards de "benefícios" genéricos.
- Fonte padrão (Inter, Roboto, Arial) sem nenhuma decisão de tipografia.
- Textos como "Desbloqueie o poder de...", "de forma simples e rápida", "revolucione".
- Números, depoimentos ou logos de clientes inventados.
- Badges, selos e ícones decorativos que não informam nada.

Como corrigir:
1. Defina um sistema pequeno: 1 cor de fundo, 1 cor de texto, 1 cor de destaque (no máximo 2), 1 ou 2 fontes, escala de espaçamento de 4 ou 8 px. Se a marca já existe, use as cores e fontes dela.
2. Crie hierarquia com tamanho, peso e espaço, não com cor e sombra.
3. Troque textos genéricos por frases específicas do produto.
4. Use `ui-ux-pro-max` para escolher estilo, paleta e fontes adequados ao tipo de produto, e `impeccable` ou `emil-design-eng` para o acabamento de detalhes e interações.
5. Em interfaces com Next.js ou React, revise contra `web-design-guidelines`.

## Fase 5: Relatório e correções

**Relatório** (entregue antes de corrigir, para o usuário ver o quadro todo):

1. **Cabeçalho:** versão auditada (commit/data/URL), stack, promessa e jornadas.
2. **Resumo de 3 linhas:** estado geral e os 3 problemas mais graves.
3. **Tabela de achados:**

| # | Área | Severidade | Certeza | Onde | O que acontece | Evidência | Correção proposta |
| --- | --- | --- | --- | --- | --- | --- | --- |

Certeza: **confirmado** (teste ou print) ou **suspeita** (indício sem verificação completa).

Severidade:
- **Crítico:** vaza dado, permite acesso indevido, expõe chave ou gera custo sem limite. Corrigir antes de qualquer outra coisa.
- **Alto:** quebra uma jornada principal, deixa a ferramenta fora do ar, perde ou duplica dados, ou mostra número errado.
- **Médio:** confunde o usuário, erro sem explicação, estado faltando, acessibilidade abaixo do mínimo.
- **Baixo:** acabamento visual, texto, consistência, código morto.

4. **O que foi verificado e está OK** (com a evidência curta de cada item). Isso dá confiança e evita refazer a mesma checagem.
5. **O que não foi possível verificar** e por quê.
6. **Ordem sugerida de correção** em lotes.
7. **Salvar o relatório** junto da documentação do projeto (pasta de docs, projeto do Claude), com a data no nome, para a próxima auditoria comparar.

**Correções:**
- Ordem: críticos, altos, médios, baixos.
- Um lote por área, com descrição curta do que mudou e como foi re-testado.
- Ao final, rode de novo `build`, `lint`, `typecheck` e os testes (ou as receitas de teste, em app sem estrutura), e repita as jornadas principais.
- Feche com uma lista do que mudou e do que ficou pendente (e por quê).

## Fase 6: Reauditoria

Depois das correções (ou numa próxima rodada):
1. Abra o relatório anterior.
2. Refaça só os testes dos achados marcados como corrigidos e registre o resultado (resolvido / parcial / voltou).
3. Rode a Fase 0 de novo (infraestrutura e limites mudam com o tempo) e as receitas de teste.
4. Entregue um relatório curto de diferença: o que fechou, o que continua aberto, o que apareceu de novo.

## Skills que esta auditoria aciona quando estiverem disponíveis

- Segurança e banco: `supabase`, `supabase-postgres-best-practices`, `engineering:code-review`
- Bugs e testes: `superpowers:systematic-debugging`, `superpowers:test-driven-development`, `superpowers:verification-before-completion`
- Performance React/Next: `vercel-react-best-practices`, `vercel-composition-patterns`
- UX e acessibilidade: `design:accessibility-review`, `design:ux-copy`, `web-design-guidelines`
- Visual: `ui-ux-pro-max`, `impeccable`, `emil-design-eng`
- Textos: `stop-slop`