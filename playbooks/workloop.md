# Workloop: execução de trabalho longo por agente sob gates humanos

Playbook portátil, extraído de uma execução real (ver `docs/caso-llm-council.md`).
Resolve um problema específico: **fazer um agente executar dezenas de unidades de trabalho
encadeadas, ao longo de várias sessões, sem que o operador precise revisar cada linha e sem
que o agente decida o que só o operador pode decidir.**

O truque central não é o prompt. É que **todo o estado vive em artefatos persistidos**, nunca
na memória da sessão: uma sessão de zero contexto lê os artefatos e retoma o trabalho no ponto
exato. Isso é o que torna o trabalho maior do que uma sessão possível.

## Quando usar e quando não

Use quando o trabalho tem: mais de 5 unidades encadeadas, dependências entre elas, um invariante
que não pode quebrar no caminho, e um humano que quer assinar decisões sem revisar implementação.

Não use para: tarefa única, exploração sem destino definido, trabalho onde o custo do processo
(diretiva, review, handoff por unidade) excede o custo da unidade em si.

## Vocabulário

| Termo | Definição |
|---|---|
| **Operador** | O humano. Assina gates, decide o que é irreversível ou caro, dá tie-break. |
| **Executor** | O agente que conduz o ciclo de uma unidade de trabalho. |
| **Revisor** | Um segundo agente, independente do executor, que reprova por evidência. |
| **Épico** | O tracker do trabalho inteiro: grafo de dependências e checklist de fechamento. |
| **Issue** | Uma unidade de trabalho com blockers declarados e critério de verificação próprio. |
| **Contrato do repo** | `AGENTS.md`: as regras que valem para toda issue, escritas antes da primeira. |
| **Diretiva** | O contrato de execução de UMA issue, postado no tracker, aprovado por nome. |
| **Handoff** | A memória que uma issue deixa para a próxima, viajando dentro do PR. |
| **Runbook do loop** | O documento que governa o modo em lote (segmentos, gates delegados, paradas). |
| **Frase de lançamento** | O ato do operador que nomeia objeto, gates delegados e data/hora. |
| **Gate** | Ponto onde a execução para e espera assinatura humana. |
| **Parada** | Condição que interrompe o loop imediatamente, mesmo com gates delegados. |
| **Constraint de fase** | Invariante cuja violação reprova a issue, independente de testes verdes. |
| **Golden** | Prova byte a byte de que o comportamento existente não mudou. |
| **Bootstrap frio** | A rota que uma sessão de zero contexto percorre para retomar o trabalho. |

---

## Camada 1: Formação

Nada de código antes desta camada terminar. Ela produz o grafo que a Camada 2 executa.

### 1.1 Investigar o território

Levantamento do que existe hoje, com evidência `file:line`. Sem propostas ainda. O produto é
um mapa do que o trabalho vai tocar, não uma opinião sobre o que fazer.

### 1.2 Interrogar antes de planejar

Sessão de grilling sobre o problema: o objetivo é extrair a **fronteira**, o conjunto de
questões cuja resposta muda o plano. Cada questão vem com recomendação (ver "Instrumentos de
decisão"). Questões que o próprio levantamento resolve não sobem ao operador: fatos o agente
busca, decisões são do operador.

O produto é um documento de entendimento compartilhado com as decisões do operador registradas
verbatim, endereçado ao material que as originou.

### 1.3 Escrever o plano

Tarefas atômicas, cada uma com critério de verificação concreto. Invariantes globais declarados
como constraints de fase: **violou uma, a issue é reprovada, mesmo com suite verde.**

Escolha as constraints com cuidado. Uma constraint boa é verificável mecanicamente e cara de
violar por acidente. O exemplo canônico: "o comportamento X existente é intocável, provado por
golden byte a byte após cada tarefa". Essa única linha sobreviveu a sete refatorações do caminho
crítico no caso real.

### 1.4 Revisar o plano adversarialmente

Um segundo agente critica o plano contra clareza, verificabilidade e completude, **antes de
existir qualquer código**. Plano reprovado volta para reescrita. Um plano aprovado nesta etapa
custa uma fração do que custa descobrir o buraco na issue 8 de 14.

### 1.5 Quebrar em issues com grafo

Cada issue declara seus **blockers explicitamente**. O grafo vive no corpo do épico e é a fonte
da verdade sobre o que pode começar. Regra dura: nenhuma issue nasce sem blockers declarados e
sem critério de verificação.

Ordene por risco, não por conveniência: a issue que toca o núcleo do sistema recebe invariantes
mais estritos e uma parada obrigatória antes de começar.

### 1.6 Escrever o contrato do repo

`AGENTS.md` (template em `templates/AGENTS.template.md`) com sete seções: Sempre, Constraints da fase,
Rotina de execução, Bootstrap de sessão fria, Handoff obrigatório, Aplicabilidade por issue,
Mapa. É este arquivo que permite ao operador dizer apenas `execute a #N` e nada mais.

---

## Camada 2: Execução

Dois modos. O individual é o padrão; o lote é uma delegação explícita e nomeada de gates.

### 2.1 Modo individual (relay)

O ciclo de uma issue:

1. **Pré-voo**: blockers todos fechados? Ler os handoffs dos blockers **antes** do plano.
   Divergência entre handoff e plano sobe ao operador, nunca é absorvida em silêncio.
2. **Diretiva**: postada como comentário na issue, formato fixo de 7 seções
   (`templates/diretiva.md`). Parar e aguardar **aprovação nomeada**.
3. **Implementar**: worktree isolada a partir do branch default, edição cirúrgica, só o que a
   diretiva pede.
4. **Verificar**: suite verde, golden verde, CI verde. Sem isso não há PR.
5. **Review adversarial**: segundo agente contra o contrato do repo e a diretiva. FAIL retorna
   para correção. Teto de 2 rodadas.
6. **GATE HUMANO**: o operador diz "ship it". Nada de merge antes disso.
7. **Finish**: merge, fechar issue, handoff no PR, limpar worktree e labels.

O que muda entre issues não é o ciclo: é quem assina os passos 2 e 6.

### 2.2 Modo lote (loop)

Para a cadeia principal, quando o operador quer várias issues por sessão sem parar em cada uma.
O runbook (`templates/runbook-loop.md`) define:

- **Segmentos**: grupos de issues com uma parada obrigatória ao fim de cada um. A parada não é
  falha, é fronteira natural de custo e de olho humano.
- **Gates delegados por segmento**: tipicamente "aprovação de diretivas fiéis ao plano" e
  "ship it pós review PASS". Tudo o mais continua do operador.
- **Disciplina de seleção**: menor número aberto com todos os blockers fechados, conferido no
  tracker e nunca por memória. Uma issue em voo por vez. Após cada merge, atualizar a base e
  rodar a suite antes do próximo pick.
- **Budget**: nunca iniciar uma issue que a sessão não tem fôlego para terminar. Fim de sessão
  para em fronteira de issue, nunca no meio, e comenta o estado no tracker do loop.
- **Relatório**: uma linha por issue no tracker do loop (merge, PR, checks, rodadas de review,
  handoff, avisos) e um bloco por segmento.

**Condições de parada imediata** (valem mesmo com gates delegados):

1. Divergência entre handoff e plano.
2. Review FAIL após 2 rodadas na mesma issue: depois disso o problema é de desenho, não de execução.
3. Suite vermelha ou regressão de invariante sem causa óbvia e corrigível na hora.
4. Necessidade de sair do escopo da diretiva: scope creep não se aprova sozinho.
5. Diretiva que exija divergir do plano: emenda de plano é ato do operador.
6. Infra persistente (rede, tracker, git) após retries razoáveis.
7. Fim de segmento.
8. Imediatamente antes de iniciar a issue de maior risco, sempre.

Em qualquer parada: comentar o estado no tracker do loop (o que parou, por quê, o que falta) e
aguardar.

### 2.3 A frase de lançamento

O modo lote só existe se o operador o autorizar com uma frase que **nomeia o objeto**: runbook,
tracker do loop, segmento, gates delegados e o que não está delegado, terminando com data e hora
preenchidas. Sem a frase completa, é execução individual, não lote. Um spec colado sem a frase
não é autorização, é cópia do spec.

Frases prontas em `templates/frases-do-operador.md`.

### 2.4 Bootstrap de sessão fria

O teste real do framework: uma sessão nova, sem nenhum contexto, recebe `execute a #N` e precisa
chegar ao trabalho correto lendo apenas artefatos. A rota:

1. Atualizar o repo, confirmar repo e branch default.
2. Ler o `AGENTS.md` inteiro.
3. Ler o épico (grafo + rotina) e a issue N; verificar se a diretiva já está postada.
4. Pré-voo. Sem diretiva postada: escrever e parar para aprovação. Com diretiva postada e o
   operador citando a issue: implementar.
5. Ciclo completo até o gate humano.

Se esta rota falha, o framework está quebrado, independente de quão bem a sessão atual esteja
indo. **Teste-a de propósito, em uma sessão limpa, pelo menos uma vez por fase.**

### 2.5 Retomada após crash

Sessão nova roda o bootstrap frio e lê os últimos comentários do tracker do loop. Depois procura
órfãos: PR aberto de issue em voo (retomar do ponto), worktree com trabalho sem PR (perguntar ao
operador antes de qualquer coisa), nada em voo (pick normal). A regra de ouro: o estado vive no
tracker (claim, PR, handoff, label de trabalho em andamento), nunca em memória de sessão.

---

## Camada 3: Verificação e memória

### 3.1 Três verificadores independentes

| Verificador | O que pega | Custo |
|---|---|---|
| **Suite + golden** | Regressão mecânica, quebra de invariante | Grátis, roda sempre |
| **CI** | O que a sessão esqueceu de rodar, ou rodou em ambiente sujo | Grátis após configurar |
| **Review adversarial** | O que teste nenhum vê: desenho errado, promessa falsa na documentação, escopo estourado, handoff fraco | Caro, obrigatório |

O terceiro é o que se paga. No caso real ele produziu cerca de 30 bloqueios legítimos em 15
issues, incluindo classes que nenhum teste sozinho pegaria: avaliadores que eram também autores,
identificadores vazando em um caminho que deveria ser cego, escrita não idempotente onde o
contrato exigia imutabilidade, e drift de descrição em um componente que a issue nem tocava.

**Configure o CI cedo.** Sem ele, o revisor adversarial é o único gate mecânico do loop inteiro,
e ele é caro e falível.

### 3.2 O review é adversarial de verdade

O revisor recebe: o diff, o contrato do repo, a diretiva, e a instrução de verificar a evidência
por conta própria em vez de acreditar no relatório do executor. Ele devolve findings com
`file:line` e um veredito binário.

**Dissenso não se absorve.** Um FAIL entra no consolidado com refutação evidenciada; o tie-break
é do operador, nunca do executor que escreveu o código.

**Teto de 2 rodadas.** Se depois de duas correções o review ainda reprova, o problema não é a
execução: é o desenho, e isso sobe.

### 3.3 Handoff obrigatório

Toda issue de código deixa um handoff de no máximo 35 linhas dentro do PR
(`templates/handoff.md`), com cinco seções: o que mudou, decisões tomadas em voo fora do plano,
pegadinhas descobertas, o que a próxima issue precisa saber, pendências deixadas.

Duas regras que parecem detalhe e não são:

- **A próxima issue lê o handoff ANTES do plano.** O plano é o que se pretendia; o handoff é o
  que aconteceu. Divergência entre os dois é sinal, não ruído.
- **O cabeçalho só afirma o que é verdade antes do merge.** O handoff viaja dentro do PR, então
  nunca declara "merged". Estado de merge vive no PR, que é a fonte.

Handoff ausente ou incompleto é finding BLOCKING no review.

### 3.4 Evidência endereçada

Toda afirmação de resultado carrega endereço verificável: sha do commit, número do PR, contagem
de checks, identificador do registro produzido. "Passou" sem endereço não é evidência, é
asserção. Relatórios de segmento existem para serem auditados depois, por alguém sem contexto.

---

## Economia de gates: o que nunca se delega

Delegar gates é o que torna o lote viável. Delegar os gates errados é o que torna o lote
perigoso. A lista do que permanece do operador, em qualquer modo:

1. **Custo pago**: aprovação por nome, com o orçamento explicitado, ANTES de gastar. Não depois,
   não implícito, não "aproveitando que já estamos aqui".
2. **Tie-break de review sem convergência**: quando revisor e executor não convergem.
3. **Emenda de plano ou substituição de etapa**: justificativa documenta, não autoriza.
4. **Recorte de escopo**: tirar algo do épico é decisão, não simplificação.
5. **Ações destrutivas ou irreversíveis**: merge, force-push, remoção, qualquer coisa visível a
   terceiros.
6. **"Ship it"**: no modo individual sempre; no lote, só dentro do segmento autorizado.

E a regra que sustenta todas: **aprovação nomeia o objeto**. "Ok" e "pode seguir" autorizam o
objeto imediatamente proposto, nada além dele.

---

## Instrumentos de decisão

Como extrair decisões do operador sem desperdiçar a atenção dele.

### Uma questão por vez, com recomendação

O formato padrão para fronteiras pequenas: uma questão, o contexto que a torna necessária, e
uma recomendação explícita com o porquê. O operador responde ou diz "aceito a recomendação".

Aplica-se quando as questões são poucas e a resposta de uma muda a seguinte.

### Fronteira em blocos

Para fronteiras grandes (10 questões ou mais), uma por vez é desperdício. Separe em dois blocos:

- **Bloco A, recomendações com convicção**: questões onde a análise já converge. O operador
  valida em bloco ou aponta vetos por número.
- **Bloco B, decisões genuinamente do operador**: questões que dependem de preferência, risco
  aceito ou informação que só ele tem. Uma linha cada, com recomendação.

No caso real, 26 questões viraram 11 de bloco A mais 5 de bloco B, e o operador resolveu tudo em
duas mensagens. Sem o corte, seriam 26 rodadas.

### Julgamento cego para desempate

Quando duas alternativas empatam na análise, apresente as duas **sem autoria e em ordem
sorteada**, e peça a escolha. O veredito é gravado e endereçado ao material original. Isso
quebra empates que a análise não quebra, e acumula calibração: com o tempo, você sabe o quanto
a análise concorda com quem decide de verdade.

---

## Aplicabilidade por tipo de trabalho

Nem toda issue merece o ciclo completo. Declare a aplicabilidade no `AGENTS.md`, por issue:

| Tipo | Ciclo | Gate |
|---|---|---|
| **Código no repo** | Completo: diretiva, PR, handoff, review | Ship it do operador |
| **Artefato fora do repo** (skill, config de ferramenta, automação pessoal) | Sem PR, sem handoff; diretiva leve com missão, critérios e gate | **Uma sessão real de uso** pelo operador; evidência fecha a issue |
| **Núcleo de alto risco** | Completo, com invariantes mais estritos e review mais rigoroso | Parada obrigatória antes de iniciar |
| **Gate final da fase** | Completo, mais prova real de ponta a ponta | Custo aprovado por nome; fase só fecha com todas as caixas marcadas |

O caso do artefato fora do repo tem uma consequência que costuma ser subestimada: **não existe
como o agente "concluir" sozinho**. Se o gate é o operador usando a coisa, a issue fica aberta
até isso acontecer. Isso é correto: é o único teste que importa para esse tipo de trabalho.

---

## Falhas observadas e o que as previne

Todas aconteceram na execução real que originou este playbook.

| Falha | Sintoma | Prevenção |
|---|---|---|
| Processo caro morto pelo teto de tempo do harness | Trabalho pago sem artefato nenhum | Rodar processos longos em background ou multiplexador de terminal, fora do teto do comando; registrar o custo perdido à mão |
| Agente revisor morto por timeout | Review sem veredito | Substituir por outra sessão preservando a independência (nunca deixar o executor se auto-aprovar) |
| Documentação prometendo garantia inexistente | Texto afirma enforcement que o código não faz | Review verificando afirmação contra implementação, `file:line` |
| Drift em componente "intocado" | Descrição alterada em ferramenta que a issue não tocava | Review conferindo o diff inteiro, não só o que a diretiva prometeu |
| Custo contado errado | Orçamento subcontado em cerca de metade | Contabilidade derivada de registro, não de prosa; enquanto não existir, anotar à mão e declarar |
| Fechamento de fase com pendências | Épico fechado com issues abertas | Checklist de fechamento verifica cada issue e para se alguma ficou |

---

## Como instanciar em um repo novo

1. Copie `templates/AGENTS.template.md` para o repo e preencha: comando de teste, constraints da fase,
   convenção de commit, aplicabilidade por issue, mapa dos documentos.
2. Escreva o plano (Camada 1), passe pelo revisor adversarial, e abra o épico com o grafo a
   partir de `templates/epico.md`.
3. Rode a primeira issue no modo individual, inteira, do pré-voo ao finish. Não delegue nada
   ainda: a primeira issue é onde você descobre o que o contrato do repo esqueceu.
4. Depois de 2 ou 3 issues, se o ciclo estiver estável, instancie
   `templates/runbook-loop.md`, defina os segmentos e delegue os gates com a frase de lançamento.
5. Antes de fechar a fase, teste o bootstrap frio de propósito, em sessão limpa.

O framework é auto-hospedável: use-o para evoluir ele mesmo.
