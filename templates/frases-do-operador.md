# Frases do operador

O framework inteiro depende de uma coisa que parece burocrática e não é: **a aprovação nomeia o
objeto.** "Ok", "pode seguir" e "manda ver" autorizam o objeto imediatamente proposto, nada além
dele. Um spec colado sem a frase de aprovação é cópia do spec, não autorização.

Estas são as frases prontas. Copie, ajuste os números e cole.

---

## Antes de lançar: o briefing

Nem sempre você sabe se a issue tem particularidade (custo, dependência aberta, gate diferente).
Pergunte antes de gastar a sessão:

> para lançar a #<N>, há alguma instrução específica?

O agente responde com o que muda naquela issue: modo de ciclo, gates que voltam a ser seus,
custo estimado, dependências que ainda não fecharam, e as formas alternativas de lançar. Esta
pergunta custa uma mensagem e evita descobrir no meio da execução que faltava uma decisão sua.

---

## Lançamento individual

Forma mínima, quando a diretiva ainda não existe:

> execute a #<N>

O agente faz o pré-voo, escreve a diretiva, posta na issue e **para** para sua aprovação.

Forma com pré-aprovação de custo, quando a issue gasta recurso pago:

> execute a #<N>, aprovando o custo de <o que>: ~<X> chamadas, <Y> delas de <cota específica>

Aqui a diretiva, a implementação e a execução paga correm em uma sessão só, parando apenas no
"ship it". A diferença entre as duas formas é uma ida e volta de mensagem, e a segunda só é
segura quando você já sabe o que a issue vai fazer (ou seja: depois do briefing).

---

## Aprovação de diretiva

Com data e hora, sempre. O campo de tempo é o que distingue um ato de aprovação de um trecho
de texto que menciona aprovação:

> aprovo a diretiva da #<N>, <DD/MM HH:MM>

Cobrindo diretiva e custo na mesma frase:

> aprovo a diretiva da #<N> e o custo do gate: ~<X-Y> chamadas, <A-B> de <cota>

Se algo na diretiva estiver errado, aponte em vez de aprovar. Uma diretiva corrigida antes de
executar custa uma mensagem; corrigida depois, custa a issue inteira.

---

## Lançamento do loop (modo lote)

A frase mais longa do framework, porque é a que delega gates. Ela nomeia runbook, tracker,
segmento, o que está delegado e o que não está:

> Executo o loop de execucao do epico #\<N\> segundo o runbook (\<caminho\>) e a issue #\<L\>,
> a partir do estado atual da base, segmento \<S\>.
> Autorizo por issue do segmento: aprovacao de diretivas fieis ao plano/epico e "ship it"
> pos-review PASS. Paradas obrigatorias: condicoes 1-8 do runbook e o fim do segmento.
> Nao delegadas: #\<x\>, #\<y\>.
> \<data e hora\>

Depois de uma parada de segmento, "continua" não basta: é um novo lançamento, nomeando o
próximo segmento. Cada segmento é uma autorização própria.

---

## Gate de merge

> ship it

Só isso, e só depois do review PASS. Se quiser ajustes antes, peça os ajustes: o agente não
mergeia enquanto o "ship it" não vier.

---

## Aprovação de custo isolada

Quando o agente parou no meio pedindo orçamento:

> aprovo o custo: <X> chamadas, <Y> de <cota>, teto <Z>

O teto é opcional e útil: ele autoriza o agente a continuar até um limite sem voltar a
perguntar, e a parar quando o limite chegar.

---

## Respostas de fronteira

Quando o agente apresenta uma fronteira de questões, três formas de responder.

**Aceitar a recomendação de uma questão:**

> aceito a recomendação

**Responder com o seu raciocínio** (preferível quando a razão importa, porque ela fica gravada
no documento da sessão e passa a valer como precedente):

> eu acolho a decisão, porque <o motivo em uma frase>

**Bloco inteiro de uma vez**, quando a fronteira veio separada em blocos:

> 1 - valido o bloco A inteiro
> 2 - aceito as recomendações do bloco B

Vetos por número dentro de um bloco validado:

> valido o bloco A exceto o 8 e o 21, que <o motivo>

---

## Julgamento cego

Quando o agente apresenta duas alternativas sem autoria, em ordem sorteada:

> opção 1

ou `opção 2`, `empate`, `nenhuma`. Responda antes de olhar o material original: a ordem foi
sorteada e a autoria mascarada exatamente para que sua escolha não seja contaminada. O veredito
é gravado e endereçado ao material, e vira dado de calibração.

---

## Tie-break de review

Quando executor e revisor não convergem após duas rodadas:

> tie-break: <a decisão>, porque <o motivo>. <O que fica fora de escopo, se ficar.>

Esta é a frase que impede o loop circular. O executor não pode dar tie-break em si mesmo, e o
revisor não pode ser sobrescrito em silêncio: **dissenso não se absorve.**

---

## Recorte de escopo

Tirar algo do épico é decisão, não simplificação:

> decido tirar #<N> do escopo desta fase, porque <motivo>. Registre no épico e siga.

---

## Anti-padrões

| Frase | Por que falha |
|---|---|
| "ok" / "pode seguir" | Não nomeia objeto. Autoriza o que foi proposto por último, e só. |
| "aprovo tudo" | Não delimita. Se o agente descobrir três coisas novas, elas não estavam aprovadas. |
| Colar o spec de novo | Spec é conteúdo, não ato. A aprovação é um ato seu, fora do corpo do spec. |
| Aprovação sem data e hora | Fica indistinguível de uma citação da própria frase de aprovação. |
| "faz o que achar melhor" no gate de custo | Transfere ao agente a única decisão que ele não tem como tomar: quanto do seu dinheiro gastar. |
