# AGENTS.md do repo (template)

Copie para a raiz do repo alvo como `AGENTS.md` e preencha os `<placeholders>`.
Este arquivo é o que permite ao operador dizer apenas `execute a #N`: ele carrega, sozinho,
tudo o que uma sessão de zero contexto precisa saber.

Mantenha-o curto. Se passar de duas telas, algo aqui é detalhe que pertence ao plano ou à
diretiva, não ao contrato.

---

```markdown
# AGENTS.md — <nome-do-repo>

Regras para qualquer agente ou operador executando trabalho neste repo.
Valem para toda issue do épico #<N> (<nome da fase>).

## Sempre

- Teste: `<comando exato>` — exit 0 obrigatório antes de qualquer PR.
  Nenhuma checagem existente é editada, enfraquecida ou removida.
  <Se não houver CI: registre "no required checks configured" em vez de fingir verde.>
- Commit: <convenção: idioma, caixa, prefixo de tema, um commit por issue, squash no merge>.
  Nunca sem pedido explícito do operador.
- Dependências: <política. Ex.: stdlib pura; nenhuma dependência nova sem issue própria>.
- Falha nunca silenciosa: erro nomeado, nunca warning vago, nunca resultado inventado.
- Edição cirúrgica: só o que a issue pede; código adjacente não se "melhora".

## Constraints da fase (violou uma = issue reprovada)

1. <O comportamento existente que é intocável, e a prova mecânica dele. Ex.: "prompts default
   byte-idênticos, provados pelo golden da seção 15 do teste, após cada tarefa".>
2. <Invariante de arquitetura. Ex.: "o servidor permanece sem estado de sessão".>
3. <Invariante de proveniência. Ex.: "tudo que entra na execução entra selado por hash".>
4. <Invariante de compatibilidade. Ex.: "registros antigos jamais reescritos; campos novos são
   apenas aditivos".>

Constraints são poucas, verificáveis mecanicamente e caras de violar por acidente.
Se uma delas não pode ser checada por um comando, ela é intenção, não constraint.

## Rotina de execução (relay)

1. **Pré-voo**: blockers da issue todos CLOSED? Ler `docs/handoffs/issue-N.md` dos blockers
   (comece por eles, não pelo plano). Divergência handoff↔plano sobe ao operador, nunca é
   absorvida em silêncio.
2. **Diretiva**: comentário na issue, formato fixo (Missão / Estado de chegada / Passos /
   Invariantes / Validação / Fora de escopo / Entrega), aprovação nomeada do operador antes
   de executar.
3. **Ciclo**: start → implementar → suite exit 0 → review com segundo agente →
   GATE HUMANO ("ship it" explícito) → finish.

## Bootstrap de sessão fria (zero contexto)

O operador diz apenas `execute a #N`. Desta raiz, em ordem:

1. `git pull origin <branch-default>`; confirmar repo e branch default.
2. Ler este `AGENTS.md` inteiro.
3. Ler o épico #<N> (grafo de dependências + rotina) e a issue #N; verificar se a diretiva
   já está postada nos comentários.
4. Pré-voo (acima). Sem diretiva postada: escrever e PARAR para aprovação nomeada.
   Com diretiva postada e o operador citando a issue: implementar.
5. Ciclo completo (start → implementar → suite exit 0 → review com segundo agente → PARAR
   no "ship it" → finish + cleanup).

Nada além disso é preciso: as camadas AGENTS.md → épico → handoffs → diretiva carregam todo
o estado. Nenhuma sessão anterior, vault ou memória é requerida.

## Handoff obrigatório

Todo PR de código carrega `docs/handoffs/issue-N.md` (≤ 35 linhas):

    # Handoff — issue #N
    PR #<n> · branch <nome> (código em <sha12 do commit de código>) · <data>

    ## O que mudou no repo
    ## Decisões tomadas em voo (fora do plano)
    ## Pegadinhas descobertas
    ## O que a próxima issue precisa saber
    ## Pendências deixadas

O cabeçalho só afirma o que é verdadeiro antes do merge: o handoff viaja dentro do PR, então
nunca declare "merged" nele. O estado de merge vive no PR, que é a fonte.

Handoff ausente ou incompleto = finding BLOCKING no review. A próxima issue lê este arquivo
ANTES do plano.

## Aplicabilidade por issue

- **<faixa de issues de código>**: ciclo completo (diretiva + handoff obrigatório).
- **<issue de gate final>**: <o que ela carrega; se há custo real, a aprovação explícita do
  operador é exigida ANTES de gastar; ao final, marcar o checklist do épico e verificar que
  nenhuma issue ficou aberta, parando e reportando se ficou>.
- **<issues cujo artefato vive fora do repo>**: sem ciclo, sem PR, sem handoff. Diretiva leve
  (missão + critérios + gate); o gate é uma sessão real do operador; evidência em comentário
  de fechamento da issue. Ainda leem os handoffs dos blockers.
- **<issue de maior risco>**: toca <o núcleo>. Invariantes mais estritos na diretiva e review
  mais rigoroso (<o que exatamente o revisor precisa reconferir>).

## Mapa

- Plano aprovado (revisão adversarial OKAY): `docs/plans/<arquivo>.md`.
- Loop de execução em lote: runbook `docs/plans/<runbook>.md` + issue #<L> (label `loop`).
  Lançamento exige a frase do operador que nomeia runbook, issue e segmento, e os gates
  delegados; sem ela, é execução individual, não lote.
- Épico e grafo de dependências: issue #<N>. Cópia local do plano pode estar atrás do tracker;
  em dúvida, o tracker manda.
```

---

## Notas de preenchimento

**Sobre a seção "Sempre"**: o comando de teste precisa ser literal e copiável, incluindo o
interpretador correto. Um comando ambíguo aqui vira "o agente rodou o teste errado e reportou
verde" três issues depois.

**Sobre as constraints**: quatro é um bom teto. Elas competem por atenção; se tudo é constraint,
nada é. A constraint mais valiosa costuma ser a de não-regressão do comportamento existente,
porque protege o usuário atual do sistema enquanto ele é reformado por baixo.

**Sobre o bootstrap frio**: escreva-o na ordem literal de execução, com os comandos. Ele será
lido por uma sessão que não sabe nada, inclusive por você mesmo daqui a três semanas.

**Sobre a aplicabilidade**: esta seção existe porque nem toda issue merece o ciclo completo.
Declarar isso por escrito, antes de começar, evita a negociação caso a caso que corrói o gate.
