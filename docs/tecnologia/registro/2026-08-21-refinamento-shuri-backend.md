# Refinamento para @Shuri — backend Node (2026-08-21)

> **De:** @Tony | **Para:** @Shuri | **Aprovador:** Marcos
> **Origem:** testes manuais da bateria 1.5.10, executados na `DESKTOP-VMSM6LE` em 20/08
>
> **Regras:** nao commite, nao faca push. Antes de mexer em chave de reconciliacao ou em
> schema, alinhe comigo — e decisao de arquitetura, nao refactor local.

---

## Resumo

Um item so: **A-57**.

O A-58 (status ATIVO/PAUSA/AUSENTE) tinha vindo para voce, e **saiu**. Explico no fim, porque
a conclusao interessa: **o backend nao precisa de nenhuma mudanca para o status voltar a
funcionar.**

---

# A-57 — uma reuniao vira duas linhas em `eventos_reuniao`

**Severidade: ALTA** pelo efeito em relatorio.

## O que foi medido

Uma reuniao real no Google Meet, com o Marcos na maquina, gerou **duas linhas**:

```
id=3156  iniciado_em=2026-08-21T01:55:26.171Z  finalizado_em=NULL              <- aberta para sempre
id=3157  iniciado_em=2026-08-21T01:55:26.171Z  finalizado_em=2026-08-21T02:02:30.508Z
```

Mesmo `iniciado_em`, mesmo `titulo_reuniao` ("Google Meet - Meet: kea-kazm-mhz"), mesmo
`agente_id`. O evento de encerramento **criou linha nova** em vez de fechar a que estava
aberta.

## O agent fez a parte dele

Log do SessionWorker:

```
22:56:26  Reuniao confirmada: Google Meet - reuniaoInstalacaoId=fcde4d53-e980-4536-b144-56424c3207b1
23:02:30  Reuniao encerrada:  Google Meet - duracao 7 min - reuniaoInstalacaoId=fcde4d53-...
```

Mesmo `reuniaoInstalacaoId` no inicio e no fim. Os lotes subiram sem erro. E os 6 snapshots
da reuniao foram todos gravados com esse mesmo id de correlacao, em
`eventos_reuniao_snapshot`.

## Por que isto ja deveria estar resolvido

E o mesmo defeito que voce ja corrigiu para as outras duas tabelas — os commits estao no
`manager-srv-events-node`:

- `fix: reconcilia snapshot em andamento de eventos_janela (upsert em vez de INSERT duplicado)`
- `fix: reconciliar snapshot de ausencia aberta em eventos_ociosidade`

**`eventos_reuniao` ficou de fora dessa leva.**

## Pista da causa

`eventos_reuniao_snapshot` tem a coluna `reuniao_instalacao_id`. A tabela `eventos_reuniao`
**nao tem** — estas sao as colunas dela:

```
id, criado_em, ip_origem, maquina_id, usuario_id, aplicativo, camera_ativa, finalizado_em,
iniciado_em, microfone_ativo, participantes_detectados, titulo_reuniao, agente_id,
empresa_id, usuario_ref_id, windows_username, render_ativo, render_confianca,
dispositivo_tipo, dispositivos_participantes
```

Sem coluna de correlacao, o ingest nao tem por onde reconciliar e so consegue inserir.

## Dois caminhos, e o segundo mexe em schema

1. **Reconciliar por `(agente_id, aplicativo, iniciado_em)`.** Ja e suficiente para este
   caso e nao pede coluna nova. E o mesmo tipo de chave que voce usou nas outras duas tabelas.
2. **Adicionar `reuniao_instalacao_id` a `eventos_reuniao`** e reconciliar por ela. Mais
   correto conceitualmente — o agent ja manda esse id —, mas **envolve schema**, e ai preciso
   aprovar antes.

Comece pela 1, a menos que encontre um motivo para a 2. Se for a 2, me chame.

## Cuidado ao definir a janela de reconciliacao

Voce mesmo levantou isso no handoff de 19/08, sobre o UNIQUE de `eventos_ociosidade`: uma
tolerancia larga demais funde eventos legitimos. Duas reunioes seguidas no mesmo aplicativo,
com poucos segundos entre elas, nao podem virar uma so.

## Impacto

- Toda reuniao deixa uma linha aberta permanente. Engorda o mesmo passivo que ja tem 33
  ociosidades e 27 janelas abertas, do seu handoff de 19/08.
- Qualquer soma de duracao conta a reuniao duas vezes, ou trata a linha aberta como reuniao
  que nunca terminou.
- O pilar de Foco e a leitura de tempo em reuniao ficam inflados.

## Verificar junto

`eventos_sessao` e `eventos_chamada_telefonica` tambem tem pares de inicio/fim. Confira se o
mesmo furo existe la — melhor descobrir agora do que em outro teste manual.

## Aceite

- Uma reuniao real gera **uma** linha, com `iniciado_em` e `finalizado_em` preenchidos.
- Duas reunioes seguidas no mesmo aplicativo continuam sendo duas linhas.
- As linhas ja duplicadas em staging: proponha o criterio de saneamento, **nao execute** —
  entra na mesma decisao do passivo de eventos abertos, que ainda esta aberta comigo e com o
  @Steve.
- Cobertura no `manager-parity-runner` ou no teste de ingest, conforme o padrao que voce ja
  usa para as outras duas tabelas.

---

# A-58 — por que saiu do seu escopo

O status ATIVO/PAUSA/AUSENTE **nao e gerado desde 17/05**. Medido:

```
agente 29: 244 transicoes, ultima em 2026-05-15
agente 31: 200 transicoes, ultima em 2026-05-17
agente 51: nenhuma
```

Minha primeira leitura foi que o backend deveria derivar isso a partir de
`eventos_ociosidade` e `eventos_janela`, que ja chegam — regra de negocio em um lugar so,
valendo para Windows, Android e o que vier.

**O Marcos decidiu diferente: comportamento igual ao do Android.** E no Android quem calcula
e emite e o proprio agent (`StatusTransitionCalculator.kt`, 5min pausa / 15min ausente).

Com isso:

- **Voce nao precisa fazer nada.** O `srv-events-node` ja tem `StatusTransitionHandler`, que
  persiste o evento assim que ele chegar. O caminho esta pronto e funcionando — a prova e que
  o Android grava por ele hoje.
- O trabalho e do **@Bucky**: restaurar no Agent Windows o `UserStatusManager` que foi
  removido no commit b45e181, em 12/08.

Fica registrado aqui so para voce nao ser pego de surpresa quando as transicoes voltarem a
chegar em volume. Se o handler tiver algum limite que va incomodar com a frota Windows
inteira emitindo de novo, e melhor descobrir antes — vale uma olhada rapida.

---

## Contexto

- Registro dos testes manuais: `registro/2026-08-20-achados-manuais-1.5.10.md`
- Handoff anterior, ainda aberto: `registro/2026-08-19-handoff-shuri-backend-node.md`
- Refinamento do @Bucky, com o A-58: `registro/2026-08-21-refinamento-bucky-agent.md`
