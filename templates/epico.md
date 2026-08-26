# Épico (template)

A issue-raiz do trabalho. Carrega três coisas que não vivem em nenhum outro lugar: o **grafo de
dependências**, o **checklist de fechamento** e a **rotina de bastão** (como um agente pega o
trabalho).

O épico é o único documento que uma sessão fria lê depois do `AGENTS.md`. Ele precisa responder,
sem ambiguidade: o que pode começar agora, e o que ainda não pode.

---

## Corpo do épico

```markdown
# <Nome da fase>

<Duas ou três frases: o que esta fase entrega e por quê. O problema, não a solução.>

**Plano aprovado:** `docs/plans/<arquivo>.md` (revisão adversarial OKAY em <data>).
**Contrato de execução:** `AGENTS.md` na raiz.

## Grafo de dependências

    #2 golden            (sem blockers)
    #3 parser            (sem blockers)
    #4 config       ← #2
    #5 prompts      ← #2, #4
    #6 engine       ← #3, #5
    #7 destilacao   ← #6
    #8 registro     ← #6, #7        [MAIOR RISCO: toca o núcleo]
    #9 audit        ← #8
    #10 cli         ← #8
    #11 servidor    ← #8
    #12 perfis      ← #4, #10
    #13 skill A     (sem blockers)  [artefato fora do repo]
    #14 skill B     ← #11, #12      [artefato fora do repo]
    #15 gate final  ← #9, #11, #12  [custo real: aprovação por nome antes de gastar]

<O grafo é a fonte da verdade sobre o que pode começar. Blockers também são declarados na
própria issue, mas em caso de divergência entre issue e grafo, o operador decide.>

## Checklist de fechamento

- [ ] #2 <titulo>
- [ ] #3 <titulo>
- ...
- [ ] #15 <titulo>

O épico só fecha com todas as caixas marcadas E nenhuma issue da faixa aberta. Se alguma ficou
aberta na hora do fechamento, a execução para e reporta, em vez de fechar mesmo assim.

## Rotina de bastão

Como um agente pega o trabalho, do zero:

1. **Pré-voo**: os blockers desta issue estão todos CLOSED? Ler os handoffs deles em
   `docs/handoffs/`, ANTES do plano. Divergência entre handoff e plano sobe ao operador.
2. **Diretiva**: escrever e postar como comentário nesta issue, no formato de 7 seções.
   PARAR para aprovação nomeada do operador.
3. **Executar**: branch e worktree isolados a partir da base; edição cirúrgica; suite verde.
4. **Review**: segundo agente, adversarial, contra o `AGENTS.md` e a diretiva. FAIL volta para
   correção, com teto de 2 rodadas.
5. **GATE**: "ship it" do operador. Só então merge, fechamento e limpeza.
6. **Handoff**: dentro do PR, obrigatório, lido pela próxima issue antes do plano.

O estado vive aqui e no PR, nunca em memória de sessão. Uma sessão nova retoma lendo
`AGENTS.md` → este épico → os handoffs → a diretiva postada.

## Modo em lote

Quando autorizado: runbook em `docs/plans/<runbook>.md`, tracker do loop na issue #<L>.
Exige a frase de lançamento do operador nomeando runbook, issue e segmento. Sem ela, cada
issue roda no modo individual.
```

---

## Como desenhar o grafo

**Uma issue é um tracer bullet**: atravessa o sistema inteiro em uma fatia fina e deixa algo
verificável funcionando. Não é uma camada horizontal ("todos os models"), porque camada
horizontal não é verificável sozinha e o handoff dela não tem o que dizer.

**Blockers são poucos e reais.** Se quase tudo bloqueia quase tudo, o trabalho não foi
decomposto: foi listado. Se nada bloqueia nada, provavelmente há dependências escondidas que
vão aparecer como retrabalho na quinta issue.

**Marque os três tipos especiais no próprio grafo**, porque eles mudam o ciclo:

- **Maior risco**: toca o núcleo. Invariantes mais estritos, review mais rigoroso, parada
  obrigatória antes de começar.
- **Artefato fora do repo**: sem PR e sem handoff; o gate é uma sessão real do operador.
- **Custo real**: exige aprovação nomeada, com o orçamento em números, antes de gastar.

**Ordene por desbloqueio, não por facilidade.** A primeira issue deve ser a que mais abre
fronteira. No caso real, as duas primeiras foram o golden e o parser, sem blockers: uma protegia
todo o resto de regressão, a outra era pré-requisito da metade do grafo.
