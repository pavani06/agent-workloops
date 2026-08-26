# Handoff de issue (template)

A memória que uma issue deixa para a próxima. Vive em `docs/handoffs/issue-N.md`, dentro do PR
da própria issue. Máximo de 35 linhas.

A regra que dá sentido ao arquivo: **a próxima issue lê o handoff ANTES do plano.** O plano diz
o que se pretendia; o handoff diz o que aconteceu. Quando os dois divergem, quem executa a
próxima issue precisa ver a divergência e subi-la ao operador, em vez de descobrir o problema
depois de meio dia de implementação em cima de uma premissa morta.

---

```markdown
# Handoff — issue #<N>
PR #<n> · branch <nome> (código em <sha12 do commit de código>) · <data>

## O que mudou no repo
<Arquivos e o que cada um passou a fazer. Prosa curta, não changelog de linha.>

## Decisões tomadas em voo (fora do plano)
<Tudo que o plano não previa e a execução decidiu. Se não houve, escreva "nenhuma".
Esta seção é a mais valiosa do arquivo: é onde o desvio fica visível.>

## Pegadinhas descobertas
<O que custou tempo e não estava documentado: comportamento inesperado de ferramenta,
ordem que importava, armadilha de ambiente. Se não houve, escreva "nenhuma".>

## O que a próxima issue precisa saber
<Endereçado às issues que dependem desta, por número. Contratos criados, nomes escolhidos,
onde as coisas ficaram.>

## Pendências deixadas
<O que ficou aberto de propósito, com o motivo e o dono.>
```

---

## Regras que o review verifica

1. **Máximo de 35 linhas.** Handoff longo não é lido, e handoff não lido não é memória.
2. **O cabeçalho só afirma o que é verdade antes do merge.** O arquivo viaja dentro do PR,
   então nunca declare "merged" nele. O estado de merge vive no PR, que é a fonte.
3. **O sha do cabeçalho é o do commit de código**, não o do commit que adicionou o handoff.
   Quem for reconstituir o que rodou precisa do primeiro.
4. **Seções vazias são preenchidas com "nenhuma"**, nunca deletadas. Uma seção ausente é
   ambígua entre "não houve" e "esqueci".
5. **Handoff ausente ou incompleto é finding BLOCKING no review.** Não é acessório do PR:
   é entregável dele.

## Truque de preenchimento

O sha e o número do PR só existem depois do commit e da criação do PR, o que gera a tentação de
escrever o handoff com placeholders e nunca voltar. Resolva isso mecanicamente: crie o handoff
com o marcador, commite o código, e no mesmo encadeamento de comandos substitua o marcador pelo
sha real e pelo número do PR, com um commit de correção. Assim o arquivo nunca chega ao review
com `PREENCHER` dentro.
