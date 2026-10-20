# Auditoria e revisão de código de ferramenta

Uma skill para o Claude que audita, revisa o código, testa e corrige apps, sites e painéis — principalmente os gerados com IA (Lovable, Bolt, v0, Cursor, Claude e afins).

Ferramentas feitas com IA costumam ficar bonitas rápido, mas falham sempre nos mesmos pontos: chave exposta, regra de acesso que só existe na tela, botão que grava duas vezes, número que não bate com a fonte, termo que só quem fez entende e visual com "cara de IA". Somam-se a isso os problemas que só aparecem lendo o código por dentro: regra de negócio duplicada, estado que vaza de um registro para outro, consulta lenta, dependência com licença arriscada. Esta skill dá ao Claude um roteiro fixo para encontrar tudo isso com prova, priorizar e corrigir — só depois da sua aprovação — sem quebrar o que já funciona.

## Dois modos

| Modo | Quando usar | O que cobre |
|---|---|---|
| **Completo** | Primeira auditoria, antes de lançar, mudança grande, revisão periódica | Todas as fases |
| **Rápido** | Antes de cada publicação, depois de uma mudança pontual | Só o que mudou desde a última entrega e o que depende disso |

O Claude pergunta qual modo você quer no início.

## O que ela faz

A auditoria segue sempre a mesma ordem de prioridade: **segurança e disponibilidade → funcionamento e números certos → conformidade → desempenho → qualidade do código → clareza → visual.**

| Fase | O que verifica |
|---|---|
| **0. Reconhecimento** | Versão no ar, tecnologias, promessa, jornadas do usuário, **premissas do projeto** (as regras que o código precisa respeitar, lidas da documentação), mapa do código e infraestrutura (site e banco no ar, pausa por inatividade, espaço × limite do plano, backup, domínio). |
| **1. Segurança** | Chaves expostas, autenticação e sessão, regras de acesso do banco (RLS), autocadastro como admin, troca de ID na URL (IDOR), injeção de código (XSS) e de SQL, injeção de fórmula em planilhas exportadas, bibliotecas com falhas conhecidas, CSP, limite de tentativas — cada achado ligado à categoria do OWASP Top 10. |
| **2. Funcionamento** | Cada jornada de ponta a ponta, números conferidos contra uma fonte real, estados de vazio/erro/rede lenta, duplo clique, F5 no meio, troca rápida entre registros, restos de geração por IA. |
| **3. Revisão do código** | Leitura por partes (com revisões independentes em código grande): erros de lógica (estado que vaza, corrida entre chamadas assíncronas, erro virando sucesso, fuso horário), qualidade (funções gigantes, duplicação de regra de negócio, código morto, acoplamento) e **desempenho medido** (consultas N+1 e sem índice, loops pesados, vazamento de memória, loops sem saída). |
| **4. Conformidade** | **Regras do projeto** conferidas uma a uma no código (cumprida / violada / não verificável), **LGPD** (dados pessoais, minimização, exposição, retenção, política), **licenças** das dependências e **OWASP Top 10**. |
| **5. Clareza** | Em 5 segundos a pessoa sabe onde está, o que fazer e o que acontece ao clicar? Termos consistentes, formato brasileiro, mensagens de erro úteis, estados vazios que orientam, acessibilidade. |
| **6. Visual sem cara de IA** | Gradiente roxo, emoji como ícone, glassmorphism, textos de marketing vazios, números inventados — e como trocar por um sistema visual simples e coerente. |
| **7. Relatório e aprovação** | Relatório priorizado com evidência de cada achado, premissas, conformidade, o que está OK e o que não deu para verificar. **Você aprova o que corrigir**; as correções vêm em lotes pequenos, re-testados, com teste automático para cada regra do projeto que estava quebrada. |
| **Reauditoria** | Numa próxima rodada, confere o que foi corrigido, o que voltou e o que apareceu de novo. |

### Regras que ela segue o tempo todo

- **Nada é afirmado sem prova.** Cada achado tem onde está e como foi verificado; sem isso, vai como "suspeita".
- **Falha conhecida é conferida na fonte oficial** no dia da auditoria, com link.
- **Não mexe em dado real.** Testes que gravam rodam em cópia ou simulação; em produção, só leitura.
- **Nunca mostra segredos** — só o nome da variável e os 4 primeiros caracteres.
- **Desempenho se mede, não se adivinha** — lentidão só vira achado com número cronometrado.
- **Relatório e aprovação antes de qualquer correção** — inclusive melhorias de código "óbvias".
- **Urgência é avisada na hora** (site fora do ar, chave vazada), sem esperar o relatório.

### Testes práticos incluídos

- **XSS dinâmico:** preenche todos os campos com código malicioso inofensivo e conta se algo executou.
- **Duplo clique:** simula rede lenta e conta quantas gravações um clique duplo gera.
- **Contraste:** calcula a razão WCAG de cada cor de texto × fundo (nada "no olho").
- **Celular:** abre a 360 px de largura e mede se a página estoura a tela.
- **Troca rápida de registro:** confere que nada de um registro aparece ou é gravado em outro.
- **Volume:** roda o fluxo mais pesado com o maior volume real e cronometra.
- **F5 no meio:** confere se a pessoa volta de onde parou.
- **Testar sem produção:** banco simulado no navegador e cópia local das regras de acesso, para atacar as permissões sem tocar em dado real.

## O que você recebe

1. **Relatório** com modo, versão auditada, resumo dos 3 problemas mais graves e uma tabela de achados (área, severidade, certeza, onde, o que acontece, evidência, correção proposta).
2. **Tabela das premissas do projeto** (cumprida / violada / não verificável) e **conformidade** (LGPD, licenças, OWASP).
3. **Lista do que foi verificado e está OK** e do que não foi possível verificar.
4. **Ordem sugerida de correção** em lotes e o pedido de aprovação.
5. Depois da sua aprovação, **as correções**, testadas de novo, com a lista do que mudou e do que ficou pendente.

## O que precisa

**Obrigatório**
- Claude com suporte a skills (app Claude ou Claude Code).
- Acesso ao **código** da ferramenta (arquivos, repositório ou o HTML).

**Recomendado — quanto mais, mais completa a auditoria**
- **URL** da ferramenta no ar.
- **Login de teste** (de preferência um usuário comum, não o admin).
- **Acesso de leitura ao banco** (por exemplo, o conector do Supabase), para conferir regras de acesso, tamanho e avisos de segurança.
- **Documentação do projeto** (README, especificações, regras de negócio) — é de onde saem as premissas conferidas na conformidade. Sem ela, o Claude pergunta as regras principais.
- **Um caso real para conferir os números** (um relatório, extrato ou planilha que a ferramenta deveria reproduzir).
- Ambiente onde o Claude consiga rodar código e um navegador automatizado (Playwright), para os testes práticos. Sem isso, a auditoria é feita pela leitura do código e marca o que ficou sem teste.

**Opcional**
- Outras skills citadas no texto (`supabase`, `design:accessibility-review`, `ui-ux-pro-max` etc.) melhoram partes específicas, mas a auditoria funciona sem elas.

## Como instalar

**No app Claude:** baixe o `auditoria-ferramenta-ia.zip` deste repositório e envie na área de Skills das configurações do Claude. O recurso de skills precisa estar ativado na sua conta.

**No Claude Code:** copie a pasta `auditoria-ferramenta-ia` (a que tem o `SKILL.md`) para `~/.claude/skills/` (vale para todos os projetos) ou para `.claude/skills/` dentro do projeto.

## Como usar

Basta pedir em linguagem normal. Exemplos:

- "Audita essa ferramenta." (o Claude pergunta: completo ou rápido?)
- "Faz uma revisão de código completa do projeto."
- "Revisão rápida do que mudou antes de eu publicar."
- "Revisa esse app antes de eu colocar em produção."
- "Tira a cara de IA desse painel."
- "Faz uma reauditoria comparando com o relatório anterior."

O Claude segue as fases, entrega o relatório e espera você escolher o que corrigir.

## Limites

- A auditoria **não substitui um teste de invasão profissional** em sistemas com dado sensível ou dinheiro envolvido.
- A checagem de **licenças e LGPD não é parecer jurídico**: aponta riscos para você levar a um advogado quando for o caso.
- O que não pôde ser verificado (sem acesso ao banco, sem login de teste, sem ambiente para rodar) aparece no relatório como não verificado — não como aprovado.
- Login real com provedores (Google, etc.) e celular físico normalmente são simulados.

## Licença

MIT — use, adapte e compartilhe à vontade, mantendo o aviso de autoria. Veja [LICENSE](LICENSE).
