# Diretiva de issue (template)

O contrato de execução de UMA issue. Postada como comentário na própria issue, antes de
qualquer código, e executada só depois de aprovação nomeada do operador.

Por que no tracker e não no chat: a diretiva precisa sobreviver ao fim da sessão. Uma sessão
fria que pega a issue amanhã lê a diretiva postada e sabe exatamente o que foi combinado,
inclusive o que ficou de fora.

As sete seções são fixas. Seção vazia é sinal: se "Fora de escopo" está vazia, o escopo não
foi pensado.

---

```markdown
# Diretiva — issue #<N>: <título curto>

**Repo/base:** `<owner>/<repo> @ <branch> (<sha12>)` · **Ciclo:** <individual | loop segmento S>
· <aprovações já dadas no lançamento, se houver>

## Missão

<Uma ou duas frases. O que esta issue existe para provar ou entregar. Não é a lista de passos.>

## Estado de chegada

<O que já é verdade quando esta issue começa: issues fechadas relevantes, o que os handoffs dos
blockers disseram, o que está em voo em outro lugar. Divergência entre handoff e plano é
declarada AQUI, não resolvida em silêncio.>

## Passos

1. <Passo atômico, com o artefato que produz.>
2. <...>
3. <Verificação, explícita como passo.>
4. <Review, gate e finish.>

## Invariantes

- <Constraints da fase que esta issue pode ameaçar, nomeadas uma a uma.>
- <Invariantes locais: o que não pode ser tocado, o que não pode ser reescrito.>
- <Se a issue toca o núcleo: os invariantes ficam mais estritos aqui, não mais vagos.>

## Validação

- <Comando de teste e resultado esperado, literal.>
- <Checagens específicas que o revisor precisa reconferir por conta própria.>
- <Se há evidência externa (registro, artefato gerado): onde ela estará e como conferir.>

## Fora de escopo

- <O que esta issue explicitamente NÃO faz, mesmo que seja tentador.>
- <O que pertence a outra issue, com o número dela.>

## Entrega

- <PR com o quê; handoff em qual caminho; o que fecha a issue.>
- <Se há custo real: o orçamento em números, e a nota de que ele exige aprovação por nome
  ANTES de gastar.>
```

---

## Diretiva leve

Para issues cujo artefato vive fora do repo (skills, configuração de ferramenta, automação
pessoal): sem PR e sem handoff, o contrato encolhe para três seções.

```markdown
# Diretiva — issue #<N>: <título curto>

## Missão
<o que a coisa faz, e para quem>

## Critérios
<o que precisa ser verdade no artefato para ele estar pronto; onde ele vive>

## Gate
<qual sessão real do operador prova que funciona; que evidência fecha a issue>

## Fora de escopo
<mudanças no repo; o que pertence a outra issue>
```

O ponto que costuma ser esquecido: **o agente não consegue fechar esta issue sozinho.** Se o
gate é o operador usando o artefato, a issue fica aberta até isso acontecer. Escreva isso na
diretiva para que a expectativa fique explícita.

---

## Exemplo real (gate final de fase, com custo)

Da execução que originou este playbook, reduzido:

```markdown
# Diretiva — issue #15: gate E2E, suite completa, README e smoke run pago

**Repo/base:** `owner/repo @ master (a4984ff4189f)` · **Ciclo:** individual (gates do operador;
custo do smoke PRÉ-APROVADO no lançamento: ~18 chamadas, 4 de cota paga) · **Decisão do
operador**: caminho (a), o PR completa e o fechamento do épico fica pendente de #13/#14.

## Missão
Provar a fase de ponta a ponta contra serviços reais, documentar e registrar a evidência.

## Estado de chegada
11/14 issues fechadas. O master tem <capacidades>. #13/#14 abertas (gates do operador);
este PR não depende delas, o fechamento do épico sim.

## Passos
1. Diretiva postada; claim; branch `issue/15-gate-final`; worktree.
2. README: <quatro pontos exatos>.
3. Suite verde no branch; PR draft.
4. Smoke pago (aprovado), a partir do master da árvore principal: <dois comandos, com o
   assert de cada um>. Evidência (saídas, tokens, shas) no corpo do PR.
5. Review → GATE HUMANO ("ship it") → finish.

## Invariantes
- Nenhuma mudança de código: este PR é documentação mais evidência.
- O smoke roda contra o produto real, não contra o branch.

## Validação
- Suite exit 0 no branch e no master.
- Os dois registros do smoke existem e conferem com o que o PR afirma.

## Fora de escopo
- Qualquer correção de código descoberta pelo smoke vira issue nova.

## Entrega
- PR com README e handoff; evidência do smoke no corpo; checklist do épico atualizado.
```

O que esse exemplo demonstra e vale copiar: o cabeçalho registra **qual gate já foi assinado e
com que número**, o estado de chegada nomeia a decisão de caminho do operador, e a seção "Fora
de escopo" fecha a porta para a tentação clássica de corrigir, na mesma issue, o que a prova
final descobriu.
