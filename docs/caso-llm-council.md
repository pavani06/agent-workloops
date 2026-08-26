# Caso de origem: uma fase inteira executada sob o workloop

O playbook em `playbooks/workloop.md` não foi desenhado no abstrato. Ele é a destilação de uma
execução real: a Fase 1 de um projeto de linha de comando em Python (`llm-council`), 15 issues
de trabalho, executadas ao longo de três dias e várias sessões de agente, com um único operador
humano assinando os gates.

Este documento registra o que aconteceu, com os números, para que o playbook possa ser julgado
por evidência e não por plausibilidade.

## O arco completo

Uma pergunta inicial sobre "como evoluir a arquitetura deste projeto" virou, em ordem:

1. Investigação da arquitetura existente, com evidência no código.
2. Sessão de interrogatório cruzando a investigação com o objetivo.
3. Plano escrito, submetido a revisão adversarial, aprovado.
4. Épico com grafo de dependências e 14 issues de execução, mais uma de gate final.
5. Rotina de bastão documentada no épico, contrato do repo escrito no `AGENTS.md`.
6. Execução: primeiro individual, depois em lote sob runbook, com gates delegados por segmento.
7. CI mínimo configurado como etapa 0 do lote.
8. Gate final com prova real paga.
9. Duas issues cujo gate era uma sessão do próprio operador.
10. Fechamento do épico, com checklist verificado issue por issue.

## Os números

| Métrica | Valor |
|---|---|
| Issues de execução | 15 (mais um tracker de loop) |
| Suite de testes | 189 → 313 checagens |
| Regressão do comportamento existente | zero: golden byte a byte verde em todos os merges |
| Bloqueios legítimos levantados pelo review adversarial | cerca de 30 |
| Checagens existentes editadas em todo o épico | 1, declarada e justificada em três lugares |
| Teto de 2 rodadas de review acionado | 1 vez (resolvido por tie-break do operador) |
| Sessões de revisor mortas por timeout | 2 (substituídas preservando a independência) |
| Dependências novas introduzidas | zero (constraint da fase) |

## O que o review adversarial pegou que os testes não pegariam

Esta é a linha do orçamento mais difícil de justificar antes de rodar, e a mais fácil depois.
Amostra real dos bloqueios:

- Avaliadores que eram também autores do material avaliado, quebrando o cegamento que o desenho
  exigia.
- Identificadores reais vazando em um caminho que deveria ser cego.
- Escrita não idempotente onde o contrato exigia imutabilidade de registro.
- Drift de descrição em um componente que a issue nem tocava, detectado porque o revisor
  conferiu o diff inteiro e não só o que a diretiva prometia.
- Documentação afirmando garantia que o código não fazia: o texto dizia que o sistema "declara"
  um estado, quando na verdade apenas instrui um modelo a declará-lo, sem verificação. Corrigido
  para "é instruído a declarar, e quem confere é você".

O último merece atenção porque é o mais insidioso: nenhum teste falha quando a documentação
promete demais. Só um leitor adversarial pega.

## Onde o framework foi testado sob estresse

**O teto de 2 rodadas foi acionado uma vez.** Executor e revisor não convergiram sobre o que uma
issue deveria entregar. A regra parou o loop e subiu ao operador, que decidiu em uma mensagem.
Sem a regra, teriam sido mais rodadas de correção circular.

**Duas sessões de revisor morreram por timeout.** A resposta correta não é o executor aprovar o
próprio trabalho: é substituir o revisor por outra sessão independente. A independência é a
propriedade que dá valor ao review; preservá-la sob falha é o teste real.

**Uma checagem existente precisou ser editada.** O contrato do repo proibia editar checagens
existentes. A exceção foi declarada na diretiva, justificada no PR e registrada no handoff, em
três lugares. Uma exceção declarada em três lugares é rastreável; uma exceção silenciosa é
erosão de constraint.

**O fechamento do épico encontrou issues abertas.** O gate final tinha, no próprio texto, a
instrução de verificar que nenhuma issue da faixa ficou aberta e **parar e reportar** se ficasse.
Ele parou. O operador escolheu explicitamente o caminho (o PR completa, o épico espera), e a
decisão ficou registrada. Sem essa cláusula, o épico teria sido fechado com duas issues abertas
dentro.

## Onde o framework falhou

Honestidade importa mais do que a vitrine.

**Processo caro morto pelo teto de tempo do harness.** Uma rodada de deliberação paga levava
mais tempo do que o teto de um comando de shell. O processo foi morto, o serviço já tinha sido
cobrado, e **nenhum artefato foi produzido**: nem registro, nem registro parcial, nem
contabilidade. Aconteceu duas vezes. A correção operacional foi rodar processos longos em
background ou em multiplexador de terminal, fora do teto do comando, e anotar à mão o custo
perdido. A correção estrutural (escrita incremental por estágio) virou achado documentado para
a fase seguinte.

**Contabilidade de custo derivada de prosa.** O sistema subcontava aproximadamente metade do
consumo, porque um dos estágios não registrava uso. Todo o orçamento das sessões foi feito à
mão, no documento da sessão. Funcionou, mas é frágil: o número que autoriza gasto não deveria
depender de alguém lembrar de somar.

**Fronteira grande demais para o formato de uma questão por vez.** A primeira rodada de
interrogatório devolveu 26 questões. Uma por vez seriam 26 idas e voltas. A saída foi o corte em
blocos (o que a análise recomenda com convicção contra o que é genuinamente do operador), e o
operador resolveu tudo em duas mensagens. O formato de uma questão por vez continua correto para
fronteiras pequenas, e vira desperdício acima de dez.

## O momento que mais diz sobre o método

No gate final, a prova real consistia em usar o produto contra serviços de verdade, com custo
aprovado por nome antes de gastar. Uma das execuções pedia ao próprio sistema que decidisse qual
deveria ser o próximo passo do projeto, com o estado real da fase como evidência.

A proposta eleita, por unanimidade dos avaliadores cegos, foi a que dizia: *"o que a evidência
prova é mais estreito do que a frase 'infraestrutura concluída' sugere: as 313 checagens verdes
são offline e provam consistência interna, não que o caminho real funciona de ponta a ponta"*.

O sistema, ao ser usado pela primeira vez para valer, elegeu a crítica à validade da própria
prova que estava sendo produzida. E a execução daquela prova era, ela mesma, a resposta à
crítica.

Isso não é anedota bonita: é a demonstração de que o gate final tem de ser **uso real**, não
suite verde. Uma bateria de testes offline nunca teria produzido aquela informação.
