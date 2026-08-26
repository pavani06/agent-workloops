# Runbook do loop (template)

Governa o modo em lote: várias issues por sessão, com gates delegados e paradas cirúrgicas.
Vive em `docs/plans/<data>-loop.md` no repo alvo, e é apontado pelo `AGENTS.md`.

Instancie-o só depois de 2 ou 3 issues rodadas no modo individual. O loop amplifica o processo
que existe; se o ciclo individual ainda está sendo ajustado, o lote amplifica o ajuste também.

---

```markdown
# Loop de execução do épico #<N> — Runbook

**Tracker/autorização:** issue #<L> (label `loop`). **Escopo:** issues #<a> a #<b>.
**Não delegadas:** #<x>, #<y>. O bootstrap de sessão fria está no `AGENTS.md`; este runbook
governa o modo em lote.

## 1. Objetivo

Executar a cadeia principal restante do épico #<N> em segmentos, com gates delegados e paradas
cirúrgicas, de forma que uma sessão fria (zero contexto) conduza o loop lendo apenas artefatos
persistidos.

## 2. Etapa 0 (opcional, recomendada): CI mínimo

- [ ] `<arquivo de workflow>`: roda em push e pull request; executa `<comando de teste>`.
- <O que o CI consegue e o que não consegue rodar no ambiente dele, explicitamente.>
- Verde na base = gate mecânico ganho de graça em todos os PRs seguintes. Sem ele, o revisor
  adversarial é o único gate independente do loop inteiro.

## 3. Disciplina de seleção e execução

1. **Pick:** menor número ABERTO do épico com todos os blockers CLOSED. Conferir blockers no
   tracker, nunca por memória.
2. **Serial:** uma issue em voo por vez. Após cada merge: atualizar a base, suite exit 0, e só
   então o pick da próxima.
3. **Ciclo por issue:** o mesmo do modo individual (diretiva → start → implementar → suite →
   review → ship it → finish + handoff no PR). Nada do ciclo é dispensado no lote; o que muda
   é quem assina os gates.
4. **Budget:** nunca iniciar uma issue nova se a sessão não tem fôlego para terminá-la. Fim de
   sessão para em fronteira de issue, nunca no meio, e comenta o estado na #<L>.

## 4. Segmentos e gates

| Segmento | Issues | Gates delegados por issue | Fim de segmento |
|---|---|---|---|
| 0 (opcional) | CI mínimo | nenhum (operador) | CI verde na base |
| 1 | #<a>, #<b>, #<c> | diretiva fiel ao plano/épico + ship it pós review PASS | PARADA 1: relatório na #<L>, aguardar |
| 2 | #<d> | idem, com diretiva estrita (maior risco) | PARADA 2: relatório, aguardar |
| 3 | #<e>...#<f> | idem | PARADA FINAL: relatório, fechar a #<L> |

O operador lança UM segmento por vez com a frase de lançamento (§7). "Continua" após uma parada
é um novo lançamento nomeando o próximo segmento.

## 5. Condições de parada IMEDIATA

1. Divergência handoff↔plano (sobe ao operador, nunca absorvida em silêncio).
2. Review FAIL após 2 rodadas de correção na mesma issue: depois disso o problema é de desenho,
   não de execução.
3. Suite vermelha ou regressão de invariante sem causa imediata óbvia e corrigível na hora.
4. Necessidade de sair do escopo da diretiva (scope creep não se aprova sozinho).
5. Diretiva que exija divergir do plano ou do épico (emenda de plano é ato do operador).
6. Infra persistente (rede, tracker, git) após retries razoáveis.
7. Fim de segmento.
8. Imediatamente antes de iniciar #<d>, sempre.

Em qualquer parada: comentar o estado na #<L> (o que parou, por quê, o que falta) e aguardar.

## 6. Recuperação de crash / retomada

1. Sessão nova roda o bootstrap frio (`AGENTS.md`) e lê os últimos comentários da #<L>.
2. Procurar órfãos no histórico da base, na lista de PRs e nas worktrees:
   - PR aberto de issue em voo → retomar do ponto (review ou finish).
   - Worktree com trabalho sem PR → perguntar ao operador antes de qualquer coisa.
   - Nada em voo → pick normal.
3. Regra de ouro: o estado vive no tracker (claim, PR, handoff, label de trabalho em
   andamento), nunca em memória de sessão.

## 7. Frase de lançamento (template do operador)

    Executo o loop de execucao do epico #<N> segundo o runbook (<caminho do runbook>)
    e a issue #<L>, a partir do estado atual da base, segmento <S>.
    Autorizo por issue do segmento: aprovacao de diretivas fieis ao plano/epico e "ship it"
    pos-review PASS. Paradas obrigatorias: condicoes 1-8 do runbook e o fim do segmento.
    Nao delegadas: #<x>, #<y>.
    <data e hora>

A frase nomeia o objeto (runbook, issue e segmento) e os gates delegados; o campo de data e
hora fecha o requisito de aprovação nomeada. Sem ela preenchida, é cópia do spec, não
autorização.

## 8. Relatórios

Por issue (comentário na #<L>, uma linha):

    #<n> <titulo-curto> — MERGED <sha12> · PR #<pr> · suite <X> checks exit 0 ·
    review PASS em <k> rodada(s) · handoff docs/handoffs/issue-<n>.md · avisos: <nenhum|lista>

Por segmento: bloco com as linhas das issues, a fronteira resultante (o que abriu) e qualquer
sinal que mereça olho humano retrospectivo.

## 9. Riscos e mitigações

| Risco | Mitigação |
|---|---|
| Erosão de gate (muitos merges sem olho humano) | Etapa 0; revisor obrigatório sem exceção; paradas segmentadas; relatórios auditáveis na #<L> |
| Erro composto na cadeia inicial propagando adiante | Constraint de não-regressão + suite verde como invariante duro por issue; parada antes da issue de maior risco |
| Review sem convergência | Máximo de 2 rodadas por issue, depois parada |
| Scope creep aprovado pelo próprio loop | Paradas 4 e 5; diretiva fiel ao plano é condição da delegação |
| Fronteira mal computada | Blockers conferidos no tracker a cada pick; execução serial elimina corrida |
| Handoff fraco quebrando a cadeia seguinte | Handoff ausente ou incompleto já é BLOCKING no review |
| Custo e cota | Issues caras ficam fora do loop; fim de segmento é fronteira natural de custo |

## 10. Fechamento

A #<L> fecha ao fim do último segmento (ou quando superseded), com relatório final: issues
executadas, merges, pendências não delegadas, lições para o próximo lote.
```

---

## Por que os segmentos existem

A tentação é delegar o loop inteiro de uma vez. O segmento resolve três coisas ao mesmo tempo:

- **Custo**: cada parada é um ponto natural para o operador decidir se continua gastando.
- **Erosão de gate**: uma sequência longa de merges sem olho humano degrada a qualidade do
  julgamento do próprio revisor, porque o contexto do que "normal" significa vai derivando.
- **Recorte de risco**: a issue que toca o núcleo ganha um segmento só dela, com parada
  obrigatória antes de começar. Isso não é desconfiança do agente; é onde o custo de errar
  é assimétrico.

## Sobre a condição 2 (teto de 2 rodadas)

Ela espelha a regra dos três erros, mas mais apertada, porque o revisor já é a segunda tentativa.
Um FAIL é normal e saudável: é o gate funcionando. Dois FAILs seguidos na mesma issue significam
que executor e revisor não estão convergindo sobre o que a issue deveria ser, e isso é um
problema de diretiva ou de plano. O operador resolve em uma mensagem o que o loop resolveria em
cinco rodadas de correção circular.
