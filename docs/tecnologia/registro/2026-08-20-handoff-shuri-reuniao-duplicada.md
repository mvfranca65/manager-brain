# Handoff para @Shuri — A-57: reuniao vira duas linhas (2026-08-20)

> **De:** @Tony | **Para:** @Shuri | **Origem:** teste manual M4 da bateria da 1.5.10
> **Severidade:** ALTA pelo efeito em relatorio | **Lado:** backend Node (ingest)

## O que foi medido

Uma reuniao real no Google Meet, com o Marcos na maquina, gerou **duas linhas** em
`eventos_reuniao`:

```
id=3156  iniciado_em=2026-08-21T01:55:26.171Z  finalizado_em=NULL
id=3157  iniciado_em=2026-08-21T01:55:26.171Z  finalizado_em=2026-08-21T02:02:30.508Z
```

Mesmo `iniciado_em`, mesmo `titulo_reuniao` ("Google Meet - Meet: kea-kazm-mhz"), mesmo
`agente_id`. O evento de encerramento **criou linha nova** em vez de fechar a que estava
aberta.

## O agent fez a parte dele

Conferido no log do SessionWorker:

```
22:56:26  Reuniao confirmada: Google Meet - reuniaoInstalacaoId=fcde4d53-e980-4536-b144-56424c3207b1
23:02:30  Reuniao encerrada:  Google Meet - duracao 7 min - reuniaoInstalacaoId=fcde4d53-...
```

Mesmo `reuniaoInstalacaoId` no inicio e no fim. Os lotes subiram com sucesso, sem erro. E os
6 snapshots da reuniao foram todos gravados com esse mesmo id de correlacao, na tabela
`eventos_reuniao_snapshot`.

## Por que isto deveria estar resolvido

E o mesmo defeito que voce ja corrigiu para as outras duas tabelas — os commits estao no
`manager-srv-events-node`:

- `fix: reconcilia snapshot em andamento de eventos_janela (upsert em vez de INSERT duplicado)`
- `fix: reconciliar snapshot de ausencia aberta em eventos_ociosidade`

`eventos_reuniao` ficou de fora.

## Pista que pode ser a causa

`eventos_reuniao_snapshot` tem a coluna `reuniao_instalacao_id`. A tabela `eventos_reuniao`,
pelas colunas que li, **nao tem** — sao elas:

```
id, criado_em, ip_origem, maquina_id, usuario_id, aplicativo, camera_ativa, finalizado_em,
iniciado_em, microfone_ativo, participantes_detectados, titulo_reuniao, agente_id,
empresa_id, usuario_ref_id, windows_username, render_ativo, render_confianca,
dispositivo_tipo, dispositivos_participantes
```

Sem a coluna de correlacao, o ingest nao tem por onde reconciliar e so consegue inserir. Se
for isso, a correcao envolve schema — e ai vale alinhar comigo antes, porque mexer em chave
de reconciliacao e decisao de arquitetura, nao refactor local. A alternativa e reconciliar
por `(agente_id, aplicativo, iniciado_em)`, que ja e o suficiente para este caso e nao pede
coluna nova.

## Impacto

- Toda reuniao deixa uma linha aberta permanente, engordando o mesmo passivo que ja tem 33
  ociosidades e 27 janelas abertas no seu handoff de 19/08.
- Qualquer soma de tempo em reuniao conta dobrado, ou trata a linha aberta como reuniao que
  nunca terminou.
- O pilar de Foco e a leitura de tempo em reuniao ficam inflados.

## Antes de mexer

Alinhar comigo (@Tony). Se a saida for reconciliar por chave nova, e decisao de arquitetura.
E vale conferir se o mesmo furo existe em `eventos_sessao` e `eventos_chamada_telefonica`,
que tambem tem pares de inicio/fim.
