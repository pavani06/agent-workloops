# agent-workloops

Repositório de workflows para agentes de IA: os processos que fazem um agente executar trabalho
longo, encadeado e verificável, sob gates humanos.

Cada workflow aqui saiu de uma execução real, com a evidência do que funcionou e do que falhou.
Nada é hipótese de processo: se está documentado, rodou.

O primeiro é o **workloop**: playbook completo em [`playbooks/workloop.md`](playbooks/workloop.md),
mapa visual do processo em [`docs/anatomia.html`](docs/anatomia.html), e o caso de origem com os
números em [`docs/caso-llm-council.md`](docs/caso-llm-council.md).

## Em linguagem de negócio

**O que é.** Um kit de processo pronto para contratar trabalho longo de IA sem perder o controle:
o agente executa dezenas de tarefas encadeadas — correções, migrações, evoluções — com revisão
independente a cada entrega, e o humano decide apenas nos pontos que importam: aprovar, gastar,
publicar.

**O problema que resolve.** Delegar a agentes desanda de formas conhecidas: eles esquecem decisões
tomadas no meio do projeto, o trabalho acaba aprovado por quem mesmo o fez, o dinheiro é gasto sem
que ninguém autorizasse, e meses depois ninguém consegue dizer por que aquele código existe. O
workloop mantém **todo o estado em documentos persistidos** — a tarefa, a diretiva de execução, a
memória que uma tarefa deixa para a próxima — que qualquer sessão nova lê sozinha, sem depender de
memória ou de quem estava presente. E cerca o agente de gates: o que ele não pode fazer sozinho
está enumerado.

**O que você recebe.** Templates prontos para instanciar (contrato do repo, épico com grafo de
dependências, diretiva por tarefa, handoff entre tarefas, runbook do modo em lote, frases exatas
de aprovação), o playbook completo das três camadas e o caso de origem com números — incluindo a
seção de onde o processo falhou.

**Resultado medido no caso real** (15 tarefas encadeadas em um produto em produção): zero regressão
no que já funcionava; cerca de 30 defeitos de desenho pegos pelo revisor independente — classes que
nenhum teste automatizado pegaria; e a supervisão humana caiu para ~4 mensagens por tarefa no modo
assistido e, no modo delegado, uma frase de lançamento mais a leitura do relatório por **lote** de
tarefas.

**Governança embutida.** O que nunca se delega ao agente: gastar dinheiro (aprovação por nome, com
orçamento explícito, antes de gastar), decidir empate de revisão, mudar plano, redimensionar
escopo, qualquer ação irreversível e o "ship it" final. Aprovação vale quando nomeia o objeto e
carrega data e hora — "ok" não autoriza nada além do que foi imediatamente proposto.

**Quando vale e quando não vale.** O processo tem custo: uma diretiva, uma revisão e um handoff por
unidade de trabalho. Paga-se quando há mais de cinco unidades encadeadas, dependências entre elas
e algo que não pode quebrar no caminho. Não se paga em tarefa única, nem em exploração sem destino,
nem quando fiscalizar custa mais que refazer.

## O problema

Você tem um trabalho de quinze unidades encadeadas, com dependências entre elas e um invariante
que não pode quebrar no caminho. Um agente dá conta de cada unidade isolada. O que ele não dá
conta é de lembrar da unidade 3 quando chega na 11, de saber que a decisão tomada na 6 mudou a
premissa da 9, e de parar sozinho antes de gastar seu dinheiro ou mexer na sua branch principal.

As saídas usuais falham de formas conhecidas. Contexto longo esquece o meio. Resumo de sessão
apaga exatamente o detalhe que a próxima unidade precisava. Revisar tudo à mão anula o ganho.
Não revisar nada produz onze merges e uma surpresa.

## A ideia central

**Todo o estado vive em artefatos persistidos, nunca na memória da sessão.**

Uma sessão nova, sem contexto nenhum, recebe `execute a #7` e chega ao trabalho certo lendo,
nesta ordem: o contrato do repo, o épico com o grafo de dependências, os handoffs das issues que
bloqueavam a #7, e a diretiva postada na própria issue. Nenhum vault, nenhuma memória, nenhuma
sessão anterior.

Se essa rota falhar, o framework está quebrado, por melhor que a sessão atual esteja indo. É um
teste que se roda de propósito, em sessão limpa, uma vez por fase.

---

## Guia passo a passo

Uma volta completa, da instalação do contrato ao fechamento da fase. Os exemplos são reais: vêm
da execução que originou o playbook, um projeto de linha de comando em Python com 15 issues.

### 1. Instalar o contrato no repo

O `AGENTS.md` na raiz do repo alvo é o que permite ao operador dizer só `execute a #N`. Copie o
template e preencha:

```bash
cp templates/AGENTS.template.md /caminho/do/repo/AGENTS.md
# preencha: comando de teste, constraints da fase, convenção de commit,
# aplicabilidade por issue, mapa dos documentos
```

O template traz sete seções fixas. Duas merecem cuidado:

- **O comando de teste tem de ser literal e copiável**, com o interpretador certo. Comando
  ambíguo aqui vira "o agente rodou o teste errado e reportou verde" três issues depois.
- **As constraints da fase são poucas e mecanicamente verificáveis.** Quatro é um bom teto. A
  mais valiosa costuma ser a de não-regressão. No caso de origem ela era uma linha:

  > O comportamento atual do `council ask` é intocável: prompts default byte-idênticos, provados
  > pelo golden (seção 15 do teste) após cada tarefa.

  Essa linha sobreviveu a sete refatorações do caminho crítico e a dez merges, sem uma quebra.

### 2. Formar o grafo

Antes de qualquer código: investigação com evidência `file:line`, sessão de interrogatório para
extrair a fronteira de decisões, plano escrito com critério de verificação por tarefa, **revisão
adversarial do plano por um segundo agente**, e só então o épico.

O épico ([`templates/epico.md`](templates/epico.md)) carrega o grafo, o checklist de fechamento e
a rotina de bastão:

```
#2 golden            (sem blockers)
#3 parser            (sem blockers)
#4 config       ← #2
#5 prompts      ← #2, #4
#8 registro     ← #6, #7        [MAIOR RISCO: toca o núcleo]
#13 skill A     (sem blockers)  [artefato fora do repo]
#15 gate final  ← #9, #11, #12  [custo real: aprovação por nome antes de gastar]
```

Marque no próprio grafo os três tipos que mudam o ciclo: maior risco, artefato fora do repo e
custo real. Ordene por desbloqueio, não por facilidade: no caso de origem as duas primeiras
issues foram o golden e o parser, sem blockers, porque uma protegia todo o resto de regressão e
a outra era pré-requisito de metade do grafo.

### 3. Rodar a primeira issue inteira, no modo individual

Não delegue nada ainda. A primeira issue é onde você descobre o que o contrato esqueceu.

O diálogo real tem quatro mensagens suas. Começa com o briefing, que custa uma mensagem e evita
descobrir no meio da execução que faltava uma decisão:

```
você:    para lançar a #7, há alguma instrução específica?
agente:  <o que muda nessa issue: modo de ciclo, gates que voltam a ser seus,
          custo estimado, dependências abertas, formas de lançar>

você:    execute a #7
agente:  <pré-voo: blockers fechados, handoffs lidos; escreve a diretiva,
          posta na issue e PARA>

você:    aprovo a diretiva da #7, 26/08 8:55
agente:  <implementa, suite verde, PR, review adversarial, e PARA no gate>

você:    ship it
agente:  <merge, fecha a issue, handoff no PR, limpa worktree e labels>
```

A aprovação leva data e hora. O campo de tempo é o que distingue um ato de aprovação de um
trecho de texto que menciona aprovação. As frases todas estão em
[`templates/frases-do-operador.md`](templates/frases-do-operador.md).

A diretiva ([`templates/diretiva.md`](templates/diretiva.md)) é postada no tracker, não no chat,
porque precisa sobreviver ao fim da sessão. Sete seções fixas: Missão, Estado de chegada, Passos,
Invariantes, Validação, Fora de escopo, Entrega. Seção vazia é sinal, e "Fora de escopo" vazia
significa que o escopo não foi pensado.

### 4. Ler o handoff que a issue deixou

Cada issue de código deixa no PR um arquivo de no máximo 35 linhas
([`templates/handoff.md`](templates/handoff.md)):

```markdown
# Handoff — issue #12
PR #27 · branch issue/12-perfis (código em a4984ff4189f) · 2026-08-26

## O que mudou no repo
## Decisões tomadas em voo (fora do plano)
## Pegadinhas descobertas
## O que a próxima issue precisa saber
## Pendências deixadas
```

A regra que dá sentido ao arquivo: **a próxima issue lê o handoff ANTES do plano.** O plano diz o
que se pretendia; o handoff diz o que aconteceu. Divergência entre os dois sobe ao operador em
vez de ser absorvida em silêncio, e handoff ausente ou incompleto é finding BLOCKING no review.

### 5. Delegar o lote

Com duas ou três issues rodadas e o ciclo estável, instancie o runbook
([`templates/runbook-loop.md`](templates/runbook-loop.md)), defina os segmentos e delegue os
gates. O lançamento é a frase mais longa do framework, porque é a que delega:

```
Executo o loop de execucao do epico #1 segundo o runbook (docs/plans/2026-08-25-loop-relay.md)
e a issue #18, a partir do estado atual do master, segmento 3.
Autorizo por issue do segmento: aprovacao de diretivas fieis ao plano/epico e "ship it"
pos-review PASS. Paradas obrigatorias: condicoes 1-8 do runbook e o fim do segmento.
Nao delegadas: #13, #14, #15.
26/08 07:10
```

Ela nomeia o objeto (runbook, issue e segmento), o que está delegado e o que não está, e termina
com data e hora. **Sem a frase completa, é execução individual, não lote:** um spec colado sem o
ato de aprovação é cópia do spec.

O agente devolve uma linha por issue no tracker do loop:

```
#12 perfis reais — MERGED a4984ff4189f · PR #27 · suite 313 checks exit 0 ·
review PASS na 1a rodada · handoff docs/handoffs/issue-12.md · avisos: nenhum
```

E um bloco por segmento, com a fronteira que abriu. No caso de origem, o segmento que fechou
quatro issues consumiu do operador a frase de lançamento e a leitura do relatório final.

### 6. Quando o review reprova

Um FAIL é o gate funcionando, não um problema. Exemplo real, do gate final da fase: o revisor
reprovou o PR com dois bloqueios, ambos de precisão, nenhum deles detectável por teste.

```
1. BLOCKING: "conselho dividido declara ENCALHADO" (README.md:131) promete enforcement
   inexistente. A flag apenas instrui o prompt (prompts.py:129-140); o parser aceita
   também DECIDIDO (structured.py:142-168) e o engine não cruza status com divided
   (engine.py:427-450). Reescreva como "o decisor é instruído a declarar ENCALHADO".

2. BLOCKING: o parágrafo exigido omite o ponto de MCP stateless (README.md:127-133).
   A informação existe, mas fora do lugar que a diretiva pediu.
```

Corrigido, o segundo passe deu PASS. O que vale copiar dessa amostra: o revisor cita `file:line`,
verifica a afirmação **contra a implementação** e não contra o relatório do executor, e devolve
veredito binário.

Duas regras cercam esse momento. **Dissenso não se absorve:** um FAIL entra no consolidado com
refutação evidenciada, e o tie-break é do operador, nunca de quem escreveu o código. E há um
**teto de duas rodadas**: se depois de duas correções o review ainda reprova, o problema é de
desenho e sobe, em vez de virar correção circular.

### 7. O gate final da fase

A última issue prova o trabalho inteiro contra o mundo real, e é onde entra o gate de custo:
aprovação por nome, com o orçamento explícito, **antes** de gastar.

```
execute a #15, aprovando o custo do smoke: ~18 chamadas, 4 delas GLM da cota trimestral
```

Sem a pré-aprovação o agente para antes de gastar e apresenta o orçamento. Com ela, tudo corre em
uma sessão só, parando no "ship it".

O fechamento da fase também tem cláusula: marcar o checklist do épico e **verificar que nenhuma
issue da faixa ficou aberta, parando e reportando se ficou**. No caso de origem ele parou, com
duas issues abertas, e o operador escolheu explicitamente o caminho. Sem essa cláusula, o épico
teria sido fechado com trabalho aberto dentro.

### 8. Testar o bootstrap frio

Antes de fechar a fase, abra uma sessão limpa e diga apenas `execute a #N`. Ela precisa chegar ao
trabalho correto lendo só os artefatos. Se precisar que você explique qualquer coisa, o contrato
tem um buraco, e o lugar de corrigir é o `AGENTS.md`.

---

## As três camadas

| Camada | O que faz |
|---|---|
| **Formação** | Investigação → interrogatório → plano → revisão adversarial do plano → épico com grafo → contrato do repo |
| **Execução** | Modo individual e modo lote, mais o bootstrap de sessão fria e a retomada após crash |
| **Verificação e memória** | Suite e golden, CI, review adversarial com teto de 2 rodadas, handoff lido antes do plano, evidência endereçada por sha |

## Os três verificadores

| Verificador | O que pega | Custo |
|---|---|---|
| Suite e golden | Regressão mecânica, quebra de invariante | Grátis, roda sempre |
| CI | O que a sessão esqueceu de rodar, ou rodou em ambiente sujo | Grátis depois de configurar |
| Review adversarial | O que teste nenhum vê: desenho errado, promessa falsa na documentação, escopo estourado, handoff fraco | Caro, obrigatório |

O terceiro é o que se paga. No caso de origem produziu cerca de 30 bloqueios legítimos em 15
issues, incluindo classes que nenhum teste pega: avaliadores que eram também autores do material
avaliado, identificadores vazando em um caminho que deveria ser cego, escrita não idempotente
onde o contrato exigia imutabilidade, e drift de descrição em um componente que a issue nem
tocava.

**Configure o CI cedo.** Sem ele, o revisor adversarial é o único gate mecânico do loop inteiro,
e ele é caro e falível.

## As oito condições de parada

Valem mesmo com gates delegados: divergência handoff↔plano; review FAIL após 2 rodadas na mesma
issue; suite vermelha ou regressão de invariante sem causa óbvia; necessidade de sair do escopo
da diretiva; diretiva que exija divergir do plano; infra persistente após retries razoáveis; fim
de segmento; e imediatamente antes de iniciar a issue de maior risco, sempre.

Em qualquer parada: comentar o estado no tracker do loop e aguardar.

## A economia de gates

O que nunca se delega, em qualquer modo: **custo pago** (aprovado por nome, com o orçamento
explícito, antes de gastar), **tie-break de review**, **emenda de plano**, **recorte de escopo**,
**ações irreversíveis** e o **"ship it"**.

E a regra que sustenta as demais: **aprovação nomeia o objeto.** "Ok" e "pode seguir" autorizam o
que foi imediatamente proposto, nada além disso.

## Como a decisão sai do operador

Três formatos, escolhidos pelo tamanho da fronteira:

- **Uma questão por vez, com recomendação**, quando são poucas e a resposta de uma muda a
  seguinte. O operador responde ou diz "aceito a recomendação".
- **Fronteira em blocos**, acima de dez questões: bloco A é o que a análise recomenda com
  convicção, validado em bloco ou vetado por número; bloco B é o que é genuinamente do operador,
  uma linha cada. No caso de origem, 26 questões viraram 11 de bloco A mais 5 de bloco B, e o
  operador resolveu tudo em duas mensagens.
- **Julgamento cego**, para empate que a análise não quebra: duas alternativas sem autoria, em
  ordem sorteada, e a escolha é gravada e endereçada ao material original.

## Mapa dos arquivos

    playbooks/
      workloop.md                  o playbook completo, das três camadas
    templates/
      AGENTS.template.md           contrato do repo alvo (o que permite dizer só "execute a #N")
      epico.md                     corpo do épico: grafo, checklist, rotina de bastão
      diretiva.md                  contrato de execução de uma issue (7 seções), e a versão leve
      handoff.md                   a memória que uma issue deixa para a próxima
      runbook-loop.md              o modo em lote: segmentos, gates delegados, paradas
      frases-do-operador.md        as frases exatas de lançamento, aprovação, gate e tie-break
    docs/
      anatomia.html                o mapa visual do processo
      caso-llm-council.md          o caso de origem, com os números e as falhas

## O caso de origem

| Métrica | Valor |
|---|---|
| Issues de execução | 15 (mais um tracker de loop) |
| Suite de testes | 189 → 313 checagens |
| Regressão do comportamento existente | zero: golden byte a byte verde em todos os merges |
| Bloqueios legítimos do review adversarial | cerca de 30 |
| Checagens existentes editadas em todo o épico | 1, declarada e justificada em três lugares |
| Teto de 2 rodadas de review acionado | 1 vez, resolvido por tie-break do operador |
| Sessões de revisor mortas por timeout | 2, substituídas preservando a independência |

O estudo completo, incluindo **onde o framework falhou**, está em
[`docs/caso-llm-council.md`](docs/caso-llm-council.md). Vale a seção das falhas: processo pago
morto pelo teto de tempo do harness sem deixar artefato nenhum, contabilidade de custo feita à
mão, e a fronteira de 26 questões que não cabia no formato de uma por vez.

## Quando não usar

O custo do processo é real: uma diretiva, um review e um handoff por unidade de trabalho. Isso se
paga quando há mais de cinco unidades encadeadas, dependências entre elas e um invariante a
proteger. Não se paga em tarefa única, em exploração sem destino definido, nem quando o custo do
processo excede o custo da unidade.

## Estado e próximos passos

Versão 0.1: o playbook, os templates e o mapa, extraídos de um caso único. O que falta, em ordem
de valor:

- **Segundo caso**, em domínio diferente, para separar o que é framework do que era
  particularidade daquele projeto.
- **Skill operacional** que instancie o workloop em um repo, em vez de copiar templates à mão.
- **Contabilidade de custo mecânica**, derivada de artefato e não de prosa: hoje o orçamento das
  sessões é somado à mão, e o caso de origem mostra que isso quebra quando um processo morre no
  meio.
- **Checklist de bootstrap frio automatizado**, para testar a retomada de zero contexto sem
  depender de disciplina.
- **Workflows irmãos**: o de interrogatório e o de triagem, hoje descritos apenas dentro do
  playbook.

O framework é auto-hospedável: a evolução deste repo deve ser executada com ele mesmo.
