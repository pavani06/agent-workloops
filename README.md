# agent-workloops

Repositório de workflows para agentes de IA: os processos que fazem um agente executar trabalho
longo, encadeado e verificável, sob gates humanos.

Cada workflow aqui é extraído de uma execução real, com a evidência do que funcionou e do que
falhou. Nada é hipótese de processo: se está documentado, rodou.

## O que tem aqui hoje

**[Workloop](playbooks/workloop.md)**, o primeiro playbook. Resolve o problema de fazer um agente
executar dezenas de unidades de trabalho encadeadas, ao longo de várias sessões, sem que o
operador revise cada linha e sem que o agente decida o que só o operador pode decidir.

A ideia central: **todo o estado vive em artefatos persistidos, nunca na memória da sessão.**
Uma sessão de zero contexto lê os artefatos e retoma o trabalho no ponto exato. É isso que
torna possível o trabalho maior do que uma sessão.

Três camadas:

| Camada | O que faz |
|---|---|
| **Formação** | Investigação → interrogatório → plano → revisão adversarial do plano → épico com grafo de dependências → contrato do repo |
| **Execução** | Modo individual (diretiva → aprovação nomeada → ciclo → review → gate) e modo lote (runbook, segmentos, gates delegados, 8 condições de parada), mais o bootstrap de sessão fria |
| **Verificação e memória** | Suite e golden, CI, review adversarial com teto de 2 rodadas, handoff obrigatório lido antes do plano, evidência endereçada por sha |

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
      caso-llm-council.md          o caso de origem, com os números e as falhas

## Como usar em um repo novo

1. Copie `templates/AGENTS.template.md` para a raiz do repo alvo como `AGENTS.md` e preencha:
   comando de teste, constraints da fase, convenção de commit, aplicabilidade por issue, mapa.
2. Faça a Camada 1 (plano, revisão adversarial do plano, épico com grafo) usando
   `templates/epico.md`.
3. Rode a primeira issue no modo individual, inteira, do pré-voo ao finish. Não delegue nada
   ainda: a primeira issue é onde você descobre o que o contrato do repo esqueceu.
4. Com o ciclo estável, instancie `templates/runbook-loop.md`, defina os segmentos e delegue os
   gates com a frase de lançamento.
5. Antes de fechar a fase, teste o bootstrap frio de propósito, em uma sessão limpa.

As frases que você vai usar no dia a dia estão em `templates/frases-do-operador.md`.

## O que o framework protege

O que nunca se delega, em qualquer modo: **custo pago** (aprovado por nome, com o orçamento
explícito, antes de gastar), **tie-break de review**, **emenda de plano**, **recorte de escopo**,
**ações irreversíveis** e o **"ship it"**.

E a regra que sustenta as demais: **aprovação nomeia o objeto.** "Ok" e "pode seguir" autorizam
o que foi imediatamente proposto, nada além disso.

## Estado e próximos passos

Versão 0.1: o playbook e os templates, extraídos de um caso único. O que ainda não existe e faz
sentido construir, em ordem de valor:

- **Segundo caso**, em domínio diferente do primeiro, para separar o que é framework do que era
  particularidade daquele projeto.
- **Skill operacional** que instancie o workloop em um repo (criar contrato, épico, issues e
  runbook, e conduzir o ciclo), em vez de copiar templates à mão.
- **Contabilidade de custo mecânica**, derivada de artefato e não de prosa: hoje o orçamento das
  sessões é somado à mão, e o caso de origem mostra que isso quebra quando um processo morre no
  meio.
- **Checklist de bootstrap frio automatizado**, para testar a retomada de zero contexto sem
  depender de disciplina.
- **Workflows irmãos**: o de interrogatório (formação da fronteira) e o de triagem, hoje
  descritos apenas dentro do playbook.

O framework é auto-hospedável: a evolução deste repo deve ser executada com ele mesmo.
