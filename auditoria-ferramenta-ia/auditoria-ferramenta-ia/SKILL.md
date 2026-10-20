---
name: auditoria-ferramenta-ia
description: Audita, revisa o código, testa e corrige uma ferramenta (app, site, painel) — segurança, funcionamento, código, desempenho, conformidade (LGPD, licenças, OWASP, regras do projeto), clareza e visual — em modo completo ou rápido, corrigindo só após aprovação.
---

# Auditoria e revisão de código de ferramenta

Use quando o usuário pedir para auditar, revisar o código, fazer code review, testar, "deixar pronto para produção", "conferir antes de publicar" ou "tirar a cara de IA" de um app, site, painel ou ferramenta, principalmente se foi gerado com IA (Lovable, Bolt, v0, Cursor, Claude, etc.).

Outras skills citadas ao longo do texto (como `supabase` ou `design:accessibility-review`) são **opcionais**: use se estiverem instaladas; se não estiverem, siga só as instruções desta skill.

O objetivo é uma ferramenta segura, que funciona em todos os caminhos, mostra números certos, tem código que dá para manter e que qualquer pessoa entende sem explicação. A ordem de prioridade nunca muda: **segurança e disponibilidade → funcionamento e números certos → conformidade → desempenho → qualidade do código → clareza → visual.**

## Primeiro passo: escolher o modo

Pergunte ao usuário, uma vez, qual modo ele quer (se ele já disse, não pergunte):

| Modo | Quando usar | O que cobre |
|---|---|---|
| **Completo** | Primeira auditoria, antes de lançar, a cada mudança grande ou de tempos em tempos | Todas as fases (0 a 7) |
| **Rápido** | Antes de cada publicação, depois de uma mudança pontual | Só o que mudou desde a última entrega — ver "Modo rápido" |

Se a pessoa não estiver disponível para responder, use o **completo** e diga no topo do relatório qual modo foi usado.

## Regras que valem em todas as fases e nos dois modos

- **Evidência antes de afirmar.** Cada achado tem onde está (arquivo:linha, tela, rota, consulta) e como foi verificado (comando rodado, teste feito, medição, print). Sem evidência, o item vai como "suspeita", nunca como fato.
- **Desempenho se mede, não se adivinha.** Lentidão, consumo de memória e loop pesado só viram achado confirmado com medição (tempo cronometrado, `EXPLAIN ANALYZE`, perfil do navegador, contagem de chamadas). Sem medição, é "suspeita — medir".
- **Falha conhecida se confere na hora, não de memória.** Versão vulnerável de biblioteca, licença, limite de plano, preço de serviço: consulte a fonte oficial (aviso do fabricante, base CVE/GHSA, página da licença, página de preços) no momento da auditoria e coloque o link no relatório. Se não der para consultar, marque como "suspeita — conferir".
- **Não quebrar nada nem tocar em dado real.** Trabalhe numa branch ou cópia. Nunca rode testes que escrevem dados contra produção ou contra dados reais de clientes sem permissão explícita. Para testar sem produção, use as saídas da seção "Como testar sem produção".
- **Nunca exiba segredos.** Ao achar uma chave ou senha, mostre só o nome da variável e os 4 primeiros caracteres, e recomende trocar a chave.
- **Relatório antes de correção — sempre.** Nada é corrigido antes de o usuário ver a análise completa e **aprovar** quais itens corrigir. Isso vale também para melhorias de código "óbvias" (renomear, extrair função, remover código morto).
- **Mudanças pequenas.** Corrija em lotes curtos, re-teste depois de cada lote. Nunca reescreva o app inteiro. Refatoração grande de arquitetura: proponha um plano em etapas e peça aprovação de cada etapa.
- **Diga o que não deu para verificar** (sem acesso ao banco, sem ambiente para rodar, sem login de teste, serviço fora do ar, trecho do código não lido).
- **Urgência fala na hora.** Se aparecer algo que está quebrando a ferramenta agora (site ou banco fora do ar, chave vazada, dado de cliente exposto), avise o usuário imediatamente e continue a auditoria em paralelo.

## Fase 0: Reconhecimento (antes de qualquer crítica)

1. **Versão auditada:** identifique exatamente o que está no ar (commit, data do deploy, URL) e confirme que é a mesma versão que você vai ler. Anote no relatório.
2. **Stack:** framework, banco, autenticação, hospedagem, serviços externos e APIs de IA usadas.
3. **Promessa:** escreva em 2 linhas o que a ferramenta promete e para quem.
4. **Jornadas:** liste de 3 a 6 jornadas principais do usuário (ex.: cadastrar, entrar, criar um registro, importar, pagar, exportar).
5. **Premissas do projeto (obrigatório):** leia a documentação do projeto (README, documento de estado/visão, especificações, CLAUDE.md, decisões registradas, regras de negócio combinadas com o usuário) e monte a lista numerada das **regras que o código precisa respeitar** — ex.: "a importação nunca soma dados de outra filial", "o relatório sempre ganha do lançamento manual", "só o admin vê o módulo X". Cada premissa será conferida na Fase 4. Se não houver documentação, pergunte ao usuário as 3 a 5 regras mais importantes; se ele não estiver disponível, deduza do código e marque como "deduzida — confirmar".
6. **Mapa do código:** liste os arquivos/módulos principais com o tamanho (linhas), o que cada um faz e de quem depende. Em arquivo único grande, liste os blocos/objetos principais com o intervalo de linhas. Esse mapa decide como dividir a leitura na Fase 3.
7. **Infraestrutura e limites do plano (sempre):**
   - O site e o banco estão no ar agora? (status do projeto no painel/API do provedor, uma requisição real à URL)
   - O plano tem pausa por inatividade? Quando foi o último uso?
   - Espaço usado × limite do plano (banco, storage, banda) e quanto o crescimento previsto consome.
   - Existe backup que o dono consegue restaurar ou baixar? Com que frequência?
   - Domínio e certificado válidos e com vencimento conhecido.
8. **Como rodar:**
   - **Projeto com estrutura (package.json etc.):** rode `install`, `build`, `lint`, `typecheck` e os testes existentes; anote o resultado.
   - **App sem estrutura (HTML único, scripts por CDN, no-code):** liste cada biblioteca carregada de fora, com a versão; confira se a versão é fixa (não `@latest`/`@2`) e se tem verificação de integridade (atributo `integrity`/SRI); leia a política de segurança do navegador (CSP) e anote o que ela libera.
9. Se faltar algo essencial (acesso ao código, URL, login de teste), peça uma vez, de forma objetiva.

## Fase 1: Segurança

Apps gerados por IA costumam falhar sempre nos mesmos pontos. Verifique cada um e, no relatório, indique a categoria OWASP Top 10 correspondente (ex.: A01 Controle de acesso, A03 Injeção, A05 Configuração, A06 Componentes vulneráveis, A07 Autenticação).

**Segredos**
- Chaves no código ou no repositório: procure `sk-`, `service_role`, `SECRET`, `PRIVATE_KEY`, `password`, tokens de API e arquivos `.env` versionados (inclusive no histórico do git).
- Chave secreta exposta no navegador: variáveis `NEXT_PUBLIC_`, `VITE_` ou `REACT_APP_` com segredo, ou a chave `service_role` do Supabase usada no cliente.
- Chamadas a APIs de IA (OpenAI, Anthropic etc.) feitas direto do navegador com a chave do dono.
- Repositório público: confirme que nada além do que é público por natureza (URL do banco, chave anônima) aparece.

**Autenticação e sessão**
- Login, logout, expiração e renovação de sessão; logout limpa dados locais (cache, IndexedDB, localStorage) para o próximo usuário do mesmo computador não ver dados do anterior.
- Recuperação de senha, links mágicos e convites não permitem tomar a conta de outra pessoa.
- Restrição de domínio/convite aplicada no servidor, não só na tela.

**Banco e autorização** (com Supabase, siga também a skill `supabase` e use os advisors de segurança)
- RLS ativado em todas as tabelas, com policies que restringem por usuário (`auth.uid()` ou e-mail do token). Policy `using (true)` em tabela com dado pessoal é crítica.
- **Autocadastro:** o usuário consegue criar o próprio registro já como aprovado/admin? Confira o `WITH CHECK` de INSERT das tabelas de usuário e papel.
- **Usuário bloqueado ou pendente** ainda consegue ler dados por alguma função ou policy?
- Funções `SECURITY DEFINER` conferem quem chama; EXECUTE revogado de `public`/`anon` quando não devem ser públicas.
- Toda rota de API, server action e função do banco (RPC) confere quem está logado e se aquele usuário pode acessar aquele registro. Teste trocar o ID na URL, no corpo da requisição ou no parâmetro da função (IDOR).
- Regras de negócio (preço, permissão, limite de uso, hierarquia) aplicadas no servidor, não só escondidas na interface. **Tudo que dá para fazer pelo console do navegador (F12) deve ser exatamente o que a tela permite.**

**Entrada de dados**
- Validação no servidor (ex.: zod, constraint, policy) em formulários, APIs e uploads.
- XSS: `dangerouslySetInnerHTML`, `innerHTML`, template de texto montando HTML ou markdown renderizado sem sanitizar. Faça o teste dinâmico da seção "Receitas de teste".
- SQL montado com concatenação de texto; filtros dinâmicos sem parâmetro.
- Upload e leitura de arquivo: tipo, tamanho, onde fica público e **versão da biblioteca que lê o arquivo** (Excel, PDF, imagem) conferida contra falhas conhecidas.
- Exportação para planilha: campos de texto começando com `=`, `+`, `-`, `@` neutralizados (injeção de fórmula).

**Dependências e cadeia de fornecimento**
- Com gerenciador de pacotes: `npm audit` (ou equivalente), listando só alto e crítico.
- Sem gerenciador: cada biblioteca de CDN conferida contra a base de falhas, versão fixa, SRI, e CSP restrita (de preferência hospedar as bibliotecas no próprio site e deixar `script-src 'self'`).

**Exposição e abuso**
- CORS aberto (`*`) em rotas autenticadas.
- Falta de limite de tentativas em login, cadastro, recuperação de senha e rotas que chamam IA (custo).
- Mensagens de erro ou logs que mostram stack trace, dados pessoais ou chaves.

## Fase 2: Funcionamento e números certos

1. Para cada jornada da Fase 0, percorra de ponta a ponta. Se houver navegador disponível, teste de verdade; se não, leia o código do fluxo inteiro.
2. **Números conferidos contra a fonte.** Em ferramenta que calcula ou analisa, escolha pelo menos um caso real (um relatório, um extrato, uma planilha oficial) e feche o número da ferramenta com a fonte, item a item. Toda diferença é explicada ou vira achado. Esse costuma ser o achado mais valioso.
3. Em cada tela, force estes estados e veja se existem e fazem sentido: **vazio, carregando, erro, sucesso, texto muito longo, muitos itens, rede lenta ou sem internet.**
4. Teste o que a IA costuma esquecer: clique duplo em botão de enviar, voltar do navegador, recarregar no meio do fluxo, sessão expirada, duas abas abertas, troca rápida entre registros (resposta atrasada de um sobrescrevendo o outro). Use as receitas abaixo.
5. Abra o console: erros e avisos em vermelho entram no relatório (separe o que é do app do que é do ambiente de teste).
6. Procure restos de geração por IA: dados falsos fixos no código, `TODO`, funções mock, botões que não fazem nada, links para `#`, páginas duplicadas.
7. Se o projeto já tem testes, rode todos e anote o placar. Para cada bug corrigido, escreva antes um teste que falha e depois passa (veja `superpowers:test-driven-development`).

## Fase 3: Revisão do código (qualidade, arquitetura e desempenho)

**Como ler sem deixar passar:** divida o código pelo mapa da Fase 0 em trechos de no máximo ~1.500 linhas, com fronteira em módulo/função. Revise cada trecho de forma **independente** (se houver subagentes disponíveis, um por trecho, cada um com esta fase e as premissas do projeto) e depois consolide, eliminando repetidos. Em arquivo grande, nunca faça uma leitura única de ponta a ponta. Anote no relatório quais trechos foram lidos.

**Erros de lógica (os que mais quebram em produção)**
- Estado compartilhado/global alterado por um fluxo e lido por outro (ex.: dados de um registro "vazando" para outro, soma onde deveria substituir).
- Corrida entre chamadas assíncronas: resposta antiga sobrescrevendo a nova, `await` faltando, promessa sem tratamento de erro, salvamento atrasado gravando no registro errado.
- `catch` vazio ou que transforma erro em sucesso ("salvo" mostrado quando falhou).
- Datas e fuso horário (cálculo pelo fuso do aparelho em vez do fuso do negócio), arredondamento de dinheiro, divisão por zero, números com vírgula/ponto lidos errado.
- Caminho novo que substituiu um antigo sem herdar as proteções do antigo (compare lado a lado).

**Qualidade e organização (manutenibilidade)**
- Funções/métodos muito longos (> ~80 linhas) ou com muitas responsabilidades; arquivos gigantes sem divisão interna clara.
- **Duplicação:** blocos quase iguais em lugares diferentes (mesma regra de negócio implementada duas vezes é risco de as duas divergirem — achado de severidade maior que duplicação "estética").
- **Código morto:** funções, variáveis, CSS e módulos sem uso; meça quantas linhas.
- Nomes que não dizem o que a coisa faz; padrões misturados (dois jeitos de fazer a mesma coisa); números mágicos sem explicação.
- Comentários desatualizados que contradizem o código.
- Acoplamento: módulos mexendo no estado interno de outros; regra de negócio espalhada pela interface.
- Em React/Next.js, aplique `vercel-react-best-practices` e `vercel-composition-patterns`.

**Desempenho (sempre com medição)**
- **Banco:** consultas N+1 (uma por item dentro de loop), `select *` trazendo colunas pesadas, falta de paginação, filtros sem índice (confira com `EXPLAIN`/`EXPLAIN ANALYZE`), policies de RLS que chamam função por linha em tabela grande.
- **Navegador/servidor:** loops aninhados sobre listas grandes (O(n²)), recálculo pesado a cada tecla ou a cada renderização, filtros/ordenações refeitos sem necessidade, bundle pesado, bibliotecas inteiras importadas para usar uma função.
- **Memória:** listas enormes mantidas em memória sem necessidade, ouvintes de evento e `setInterval` nunca removidos, cópias repetidas de grandes volumes, cache que só cresce.
- **Loops sem saída:** `while` que depende de condição externa sem limite/timeout, recursão sem caso base, re-renderização em cadeia.
- Para cada suspeita, meça com dados de tamanho real (ou o maior volume disponível) e registre o número: "importar 10 mil linhas: 4,2 s", "consulta X: 380 ms, sem índice".

## Fase 4: Conformidade

**1. Regras do projeto (obrigatório)** — para cada premissa da Fase 0, encontre no código onde ela é garantida e classifique: **cumprida** (com o arquivo:linha e, se possível, um teste que a prova), **violada** (achado, com o caminho que quebra a regra) ou **não verificável** (por quê). Sugira um teste automático para cada premissa que não tiver um.

**2. LGPD (dados pessoais)**
- Inventário: quais dados pessoais são coletados (nome, e-mail, CPF, placa, telefone, localização…), onde ficam (banco, cache local, logs, exportações), quem acessa.
- Minimização: o dado é necessário para a finalidade? Colunas pessoais sem uso são achado.
- Exposição: dado pessoal em log, URL, mensagem de erro, planilha exportada ou repositório.
- Retenção e exclusão: existe prazo de guarda? Dá para apagar os dados de uma pessoa?
- Transparência: existe política de privacidade/termos de uso compatível com o que o código faz?

**3. Licenças das dependências**
- Liste cada biblioteca com a licença (consulte a fonte oficial). MIT/BSD/Apache: ok. GPL/AGPL/LGPL, licença comercial ou "sem licença": atenção para uso comercial — explique o risco em uma frase.
- Código copiado de terceiros sem licença compatível.

**4. Padrões de segurança (OWASP)**
- Monte a tabela das categorias do OWASP Top 10 que se aplicam à ferramenta com o status de cada uma (ok / achado / não verificado), usando os resultados da Fase 1.

## Receitas de teste (use sempre que possível)

- **Injeção de código (XSS) dinâmica:** preencha todo campo que vem do banco ou de arquivo importado (nomes, descrições, observações, campos do arquivo) com `"><img src=x onerror="window.__xss=(window.__xss||0)+1">`. Percorra todas as telas e modais e, no fim, leia `window.__xss` e conte `img[src="x"]` no DOM. Qualquer valor acima de zero é crítico.
- **Duplo clique:** atrase as respostas de gravação em ~400 ms (rede lenta simulada), dê duplo clique em cada botão de criar/salvar e conte quantas gravações saíram. Mais de uma é achado.
- **Contraste:** calcule a razão WCAG de cada par cor de texto × cor de fundo da paleta (fórmula de luminância relativa). Texto normal precisa de 4,5:1; texto grande, 3:1. Não avalie no olho.
- **Celular:** abra em 360 px de largura e meça `document.documentElement.scrollWidth`. Se passar de 360, a página quebra no celular. Pule se o usuário disser que a ferramenta não é usada no celular.
- **Recarregar no meio:** faça a jornada principal, aperte F5 no meio e confira se o usuário continua de onde parou ou recebe aviso claro.
- **Troca rápida de registro:** abra o registro A, troque para o B antes de A terminar de carregar/salvar e confira que nada de A aparece ou é gravado em B.
- **Volume:** rode o fluxo mais pesado com o maior volume real disponível e cronometre (importação, relatório, tela com muitos itens).

## Como testar sem produção

- **Banco simulado no navegador:** substitua o cliente do banco por um falso em memória (ex.: um `createClient` que responde com tabelas em JavaScript) e rode a interface real com navegador automatizado (Playwright). Serve para telas, fluxos, XSS e duplo clique.
- **Cópia local das regras de acesso:** recrie em um Postgres local só a estrutura, as funções e as policies do banco (sem dados reais), com uma função que imita o token do usuário, e rode os ataques de permissão como usuários diferentes, dentro de transação revertida.
- **Leitura em produção, só leitura:** consultas `select`, `EXPLAIN` e advisors podem rodar contra produção; qualquer escrita, só com permissão explícita.

## Fase 5: Clareza para o usuário

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

## Fase 6: Visual sem cara de IA

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

## Fase 7: Relatório, aprovação e correções

**Relatório** (entregue antes de corrigir; salve junto da documentação do projeto — pasta de docs, projeto do Claude — com a data no nome, para a próxima auditoria comparar):

1. **Cabeçalho:** modo (completo/rápido), versão auditada (commit/data/URL), stack, promessa, jornadas e trechos de código lidos.
2. **Resumo de 3 linhas:** estado geral e os 3 problemas mais graves.
3. **Tabela de achados:**

| # | Área | Severidade | Certeza | Onde | O que acontece | Evidência | Correção proposta |
| --- | --- | --- | --- | --- | --- | --- | --- |

Área: Segurança · Funcionamento · Conformidade · Desempenho · Código · Clareza · Visual · Infra.
Certeza: **confirmado** (teste, medição ou print) ou **suspeita** (indício sem verificação completa).

Severidade:
- **Crítico:** vaza dado, permite acesso indevido, expõe chave, viola uma premissa do projeto com dano real ou gera custo sem limite. Corrigir antes de qualquer outra coisa.
- **Alto:** quebra uma jornada principal, deixa a ferramenta fora do ar, perde ou duplica dados, mostra número errado ou tem lentidão medida que impede o uso.
- **Médio:** confunde o usuário, erro sem explicação, estado faltando, acessibilidade abaixo do mínimo, duplicação de regra de negócio, lentidão perceptível, licença de risco.
- **Baixo:** acabamento visual, texto, consistência, código morto, organização.

4. **Premissas do projeto:** tabela com cada premissa e o status (cumprida / violada / não verificável), com a evidência.
5. **Conformidade:** LGPD (inventário + achados), licenças (lista), OWASP (tabela de status).
6. **O que foi verificado e está OK** (com a evidência curta de cada item). Isso dá confiança e evita refazer a mesma checagem.
7. **O que não foi possível verificar** e por quê.
8. **Ordem sugerida de correção** em lotes.
9. **Pedido de aprovação:** pergunte quais itens corrigir (todos, por número, ou por severidade). Itens descartados ou adiados pelo usuário ficam registrados no relatório com a decisão dele.

**Correções (só depois da aprovação):**
- Ordem: críticos, altos, médios, baixos — dentro do que foi aprovado.
- Um lote por área, com descrição curta do que mudou e como foi re-testado.
- Toda correção de premissa violada vem com um teste automático que a protege daqui para frente.
- Ao final, rode de novo `build`, `lint`, `typecheck` e os testes (ou as receitas de teste, em app sem estrutura), repita as jornadas principais e meça de novo o que era de desempenho.
- Feche com a lista do que mudou e do que ficou pendente (e por quê).

## Modo rápido

Para revisar só o que mudou, antes de publicar:

1. **O que mudou:** compare com a última versão auditada ou publicada (`git diff`, ou o arquivo anterior ao lado do novo). Liste os trechos alterados.
2. **Raio de alcance:** inclua na revisão as funções que chamam e que são chamadas pelos trechos alterados, e as telas/jornadas que passam por eles.
3. **Revise esses trechos** com as Fases 1 (segurança), 3 (código e desempenho) e as premissas do projeto (Fase 4, item 1) que eles tocam.
4. **Dependências:** biblioteca nova ou versão nova? Confira falhas conhecidas e licença.
5. **Regressão:** rode os testes existentes e as receitas de teste ligadas ao que mudou (ex.: mexeu em gravação → duplo clique; mexeu em tela com dado importado → XSS).
6. **Relatório curto:** o que mudou, achados (mesma tabela e severidades), premissas tocadas e status, testes rodados. Pedido de aprovação antes de corrigir, como no modo completo.
7. **Sugira o modo completo** quando a mudança for grande (mais de ~20% do código), mexer em autenticação, permissões, regras do banco ou estrutura de dados, ou quando a última auditoria completa tiver mais de 3 meses.

## Reauditoria

Depois das correções (ou numa próxima rodada):
1. Abra o relatório anterior.
2. Refaça só os testes dos achados marcados como corrigidos e registre o resultado (resolvido / parcial / voltou).
3. Rode a Fase 0 de novo (infraestrutura, limites e premissas mudam com o tempo) e as receitas de teste.
4. Entregue um relatório curto de diferença: o que fechou, o que continua aberto, o que apareceu de novo.

## Skills que esta auditoria aciona quando estiverem disponíveis

- Segurança e banco: `supabase`, `supabase-postgres-best-practices`, `engineering:code-review`
- Bugs e testes: `superpowers:systematic-debugging`, `superpowers:test-driven-development`, `superpowers:verification-before-completion`
- Revisão em paralelo de trechos grandes: `superpowers:dispatching-parallel-agents`
- Performance React/Next: `vercel-react-best-practices`, `vercel-composition-patterns`
- UX e acessibilidade: `design:accessibility-review`, `design:ux-copy`, `web-design-guidelines`
- Visual: `ui-ux-pro-max`, `impeccable`, `emil-design-eng`
- Textos: `stop-slop`
