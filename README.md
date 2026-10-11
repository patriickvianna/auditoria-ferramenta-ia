# Auditoria de ferramenta feita com IA

Uma skill para o Claude que audita, testa e corrige apps, sites e painéis — principalmente os gerados com IA (Lovable, Bolt, v0, Cursor, Claude e afins).

Ferramentas feitas com IA costumam ficar bonitas rápido, mas falham sempre nos mesmos pontos: chave exposta, regra de acesso que só existe na tela, botão que grava duas vezes, número que não bate com a fonte, termo que só quem fez entende e visual com "cara de IA". Esta skill dá ao Claude um roteiro fixo para encontrar esses problemas com prova, priorizar e corrigir sem quebrar o que já funciona.

## O que ela faz

A auditoria segue sempre a mesma ordem de prioridade: **segurança e disponibilidade → funcionamento e números certos → clareza → visual.**

| Fase | O que verifica |
|---|---|
| **0. Reconhecimento** | Versão que está no ar, tecnologias usadas, o que a ferramenta promete, as jornadas principais do usuário e a infraestrutura: site e banco no ar, pausa por inatividade, espaço usado × limite do plano, backup, domínio. |
| **1. Segurança** | Chaves expostas, regras de acesso do banco (RLS), autocadastro como admin, usuário bloqueado que ainda lê dados, troca de ID na URL (IDOR), injeção de código (XSS), bibliotecas com falhas conhecidas, CDN sem versão fixa, CSP, limite de tentativas, dados pessoais (LGPD). |
| **2. Funcionamento** | Percorre cada jornada de ponta a ponta, confere os números da ferramenta contra uma fonte real (relatório, extrato, planilha), força estados de vazio, erro, rede lenta, duplo clique, F5 no meio e celular, e procura restos de geração por IA (dados falsos, botões mortos, código duplicado). |
| **3. Clareza** | Em 5 segundos a pessoa sabe onde está, o que fazer e o que acontece ao clicar? Termos consistentes, formato brasileiro de número e data, mensagens de erro úteis, estados vazios que orientam, contraste e acessibilidade básica. |
| **4. Visual sem cara de IA** | Gradiente roxo, emoji como ícone, glassmorphism, textos de marketing vazios, números inventados — e como trocar por um sistema visual simples e coerente. |
| **5. Relatório e correções** | Relatório priorizado (crítico, alto, médio, baixo) com evidência de cada achado, o que está OK e o que não deu para verificar. As correções vêm em lotes pequenos, só depois de você aprovar, cada lote testado de novo. |
| **6. Reauditoria** | Numa próxima rodada, confere o que foi corrigido, o que voltou e o que apareceu de novo. |

### Regras que ela segue o tempo todo

- **Nada é afirmado sem prova.** Cada achado tem onde está e como foi verificado; sem isso, vai como "suspeita".
- **Falha conhecida é conferida na fonte oficial** no dia da auditoria, com link.
- **Não mexe em dado real.** Testes que gravam rodam em cópia ou simulação; em produção, só leitura.
- **Nunca mostra segredos** — só o nome da variável e os 4 primeiros caracteres.
- **Relatório antes de correção** — você vê o quadro todo e decide o que corrigir.
- **Urgência é avisada na hora** (site fora do ar, chave vazada), sem esperar o relatório.

### Testes práticos incluídos

- **XSS dinâmico:** preenche todos os campos com código malicioso inofensivo e conta se algo executou.
- **Duplo clique:** simula rede lenta e conta quantas gravações um clique duplo gera.
- **Contraste:** calcula a razão WCAG de cada cor de texto × fundo (nada "no olho").
- **Celular:** abre a 360 px de largura e mede se a página estoura a tela.
- **F5 no meio:** confere se a pessoa volta de onde parou.
- **Testar sem produção:** banco simulado no navegador e cópia local das regras de acesso, para atacar as permissões sem tocar em dado real.

## O que você recebe

1. **Relatório** com versão auditada, resumo dos 3 problemas mais graves e uma tabela de achados (área, severidade, certeza, onde, o que acontece, evidência, correção proposta).
2. **Lista do que foi verificado e está OK** e do que não foi possível verificar.
3. **Ordem sugerida de correção** em lotes.
4. Depois da sua aprovação, **as correções**, testadas de novo, com a lista do que mudou e do que ficou pendente.

## O que precisa

**Obrigatório**
- Claude com suporte a skills (app Claude ou Claude Code).
- Acesso ao **código** da ferramenta (arquivos, repositório ou o HTML).

**Recomendado — quanto mais, mais completa a auditoria**
- **URL** da ferramenta no ar.
- **Login de teste** (de preferência um usuário comum, não o admin).
- **Acesso de leitura ao banco** (por exemplo, o conector do Supabase), para conferir regras de acesso, tamanho e avisos de segurança.
- **Um caso real para conferir os números** (um relatório, extrato ou planilha que a ferramenta deveria reproduzir).
- Ambiente onde o Claude consiga rodar código e um navegador automatizado (Playwright), para os testes práticos. Sem isso, a auditoria é feita pela leitura do código e marca o que ficou sem teste.

**Opcional**
- Outras skills citadas no texto (`supabase`, `design:accessibility-review`, `ui-ux-pro-max` etc.) melhoram partes específicas, mas a auditoria funciona sem elas.

## Como instalar

**No app Claude:** baixe o `auditoria-ferramenta-ia.zip` deste repositório e envie na área de Skills das configurações do Claude. O recurso de skills precisa estar ativado na sua conta.

**No Claude Code:** copie a pasta `auditoria-ferramenta-ia` (a que tem o `SKILL.md`) para `~/.claude/skills/` (vale para todos os projetos) ou para `.claude/skills/` dentro do projeto.

## Como usar

Basta pedir em linguagem normal. Exemplos:

- "Audita essa ferramenta."
- "Revisa esse app antes de eu colocar em produção."
- "Tira a cara de IA desse painel."
- "Faz uma reauditoria comparando com o relatório anterior."

O Claude segue as fases, entrega o relatório e espera você escolher o que corrigir.

## Limites

- A auditoria **não substitui um teste de invasão profissional** em sistemas com dado sensível ou dinheiro envolvido.
- O que não pôde ser verificado (sem acesso ao banco, sem login de teste, sem ambiente para rodar) aparece no relatório como não verificado — não como aprovado.
- Login real com provedores (Google, etc.) e celular físico normalmente são simulados.

## Licença

MIT — use, adapte e compartilhe à vontade, mantendo o aviso de autoria. Veja [LICENSE](LICENSE).
