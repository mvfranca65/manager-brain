# Refinamento para @Shuri — lote recusado por conteudo (2026-08-24)

> **De:** @Tony | **Para:** @Shuri | **Aprovador:** Marcos
> **Origem:** item 1 de `registro/2026-08-21-linha-de-corte-producao.md`
> **Repo:** `manager-srv-events-node`
>
> **Regras:** nao commite, nao faca push. Ao terminar, **@Tony revisa**.

---

## Antes de comecar

**O que NAO mexer.** O caminho de ingestao esta bom e tem paridade provada. Nao encoste em:

| Area | Por que |
|---|---|
| `BatchDedupService` | cobre o replay byte-a-byte em 10min; nao e o alvo aqui |
| `RateLimitService`, `EmpresaResolverService`, `DeviceGuardsService` | portoes de entrada, sem relacao com os dois itens |
| Handlers individuais (`window-activity`, `idle`, `meeting`, ...) | ja tratam rejeicao por item via `motivosIgnorados` — e o mecanismo que vamos reusar, nao reescrever |
| Migration `20260821000000_uq_eventos_reuniao_sessao.sql` | esta em pe, aprovada, aguardando deploy. Nao mexer |
| Normalizacao canonica de `dispositivoTipo` | fora de escopo |

**Nota de dominio.** `time.md` diz que voce nao mexe em `srv-events`. Esta desatualizado: o
`manager-srv-events-node` existe, e seu, e a migration da reuniao de 21/08 e sua. Sigo a
realidade do repo. Vou corrigir o `time.md` em separado.

---

# Item 1 — um lote recusado por conteudo perde ate 100 eventos

**Severidade: ALTA.** Prioridade 1. Segura producao.

## Por que virou blocker agora

Ate a 1.5.11 o Agent nao desistia de lote nenhum: recusado com 400, ele voltava no ciclo
seguinte, para sempre. Ruim, e **visivel** — 93 recusas do mesmo lote numa tarde.

Na 1.5.12 o Agent ganhou teto (achado A-64): apos **5 recusas por conteudo** o lote sai da fila e
some. Foi a troca certa, porque a fila travada queimava a janela de emergencia de todo
desligamento. Mas ela converteu um travamento visivel em **perda de dado silenciosa**. Provado em
maquina em 21/08 as 16:57 — dois lotes descartados, `Eventos=1` cada.

Enquanto o backend responder 400 ao lote inteiro por causa de conteudo, essa perda e real e
cresce com o tamanho do lote (ate 100 eventos por lote).

## Causa 1 — medida, e nao e "item invalido"

`ingestion.service.ts:103` faz `AgentEventBatchSchema.safeParse(bodyRaw)`. A recusa medida foi:

```
{"codigo":"PAYLOAD_INVALIDO","issues":[{"path":"windowsUsername",
 "code":"invalid_type","message":"Invalid input: expected string, received null"}]}
```

`path: windowsUsername` — campo **do lote**, nao de um evento. E o schema diz
(`agent-event-batch.dto.ts:397`):

```ts
windowsUsername: z.string().max(255).optional(),
```

`.optional()` aceita **ausente**, e recusa **`null` explicito**. O Agent serializa o campo como
`"windowsUsername": null` quando nao tem o nome. Ou seja: **o mesmo dado, escrito de duas formas
equivalentes em JSON, era aceito de uma e recusado da outra.**

E o proprio servico ja trata null logo abaixo (`ingestion.service.ts`):

```ts
windowsUsername: batch.windowsUsername ?? null,
```

A coluna `windows_username` e nullable em todas as tabelas `eventos_*`. **Quem recusa null e so o
schema.** O resto do caminho ja o aceita.

### O que fazer

Trocar por `.nullish()` (aceita `null` e ausente). Depois disso `batch.windowsUsername` pode ser
`null`, e o `?? null` abaixo continua correto sem mudanca.

**Confira os irmaos na mesma passada** — mesma assimetria, mesmo risco, sem mudar semantica:
`identificadorColaborador` e `instalacaoId`, ambos `.optional()` hoje. Se o Agent puder mandar
null neles, tem o mesmo defeito latente. **Meça no codigo do Agent antes de mudar** (
`manager-srv-agent/src/ManagerAgent.Service/Upload/AgentEventBatchDto.cs`) e mude so o que o
Agent de fato emite como null. Nao saia trocando tudo para `nullish` por precaucao.

**Nao relaxe o resto do schema de lote.** `maquinaId`, `versaoAgente`, `descricaoSo`,
`dispositivoTipo` e `enviadoEm` continuam estritos: sem eles o lote e inatribuivel, e ai 400 e a
resposta certa.

## Causa 2 — um evento invalido derruba os outros 99

`agent-event-batch.dto.ts:398`:

```ts
eventos: z.array(AgentEventItemSchema).min(1).max(1000)
```

`AgentEventItemSchema` e uma discriminated union por `tipoEvento`. Um item fora do shape — ou com
`tipoEvento` que este backend ainda nao conhece — **falha o parse do array inteiro**, e o lote
todo vira 400.

Nao ha ocorrencia medida disto. Entra junto porque e a mesma classe de defeito e porque o teto do
Agent tornou o custo permanente.

### O que fazer

**A regra passa a ser: erro de LOTE e 400; erro de ITEM nunca e 400.**

Item invalido vai para `motivosIgnorados`, que **ja existe**, ja esta no contrato de resposta e ja
e usado pelos handlers (ex.: `MeetingEvent` sem aplicativo). Nao invente canal novo.

Desenho sugerido — decida voce, mas o resultado tem de ser este:

1. `AgentEventBatchSchema` valida os campos de lote e recebe `eventos` como array cru
   (`z.array(z.unknown())`), so com `min`/`max`.
2. Um passo seguinte roda `AgentEventItemSchema.safeParse` **por item**, preservando o indice
   original (o `dispatch` ja carrega `originalIndex` — reuse).
3. Item que falha entra em `motivosIgnorados` com indice, `tipoEvento` (quando legivel) e motivo.
4. Os que passam seguem para o `dispatch` como hoje.

### Decisao minha, para nao ficar em aberto

**Lote em que TODOS os itens falham responde 202, nao 400**, com todos em `motivosIgnorados`.

Parece errado e nao e: 400 faz o Agent retentar, e apos 5 tentativas ele **descarta**. Com 202 o
lote sai da fila do Agent na primeira resposta, com registro do que foi ignorado dos dois lados.
Nos dois casos o evento invalido nao entra no banco; a diferenca e que um deles nao gasta cinco
minutos de retentativa nem corre risco de levar evento bom junto.

### Ganho de lado — compatibilidade para frente

Com isto, um Agent mais novo que mande um `tipoEvento` que este backend ainda nao conhece deixa de
derrubar o lote: o tipo desconhecido e ignorado, o resto entra. Hoje derruba tudo. Isso reduz o
risco da ordem de deploy do `REGRAS-RELEASE` secao 3 — **nao a dispensa**, a regra de enum novo
continua valendo.

## Aceite

- Lote com `windowsUsername: null` e **aceito** (202), e grava `null` na coluna.
- Lote com `windowsUsername` ausente continua aceito — sem regressao.
- Lote com `maquinaId` invalido continua 400.
- Lote com 1 item invalido e 99 validos: **202, os 99 gravados**, 1 em `motivosIgnorados` com
  indice e motivo.
- Lote com `tipoEvento` desconhecido: 202, item ignorado, resto gravado.
- Lote com todos os itens invalidos: 202 com todos ignorados, nada gravado.
- Teste com o payload **exato** da recusa medida (esta em
  `registro/2026-08-21-bateria-1.5.12.md`) — o que hoje da 400 tem de passar.

## Paridade

Isto **diverge do Java** de proposito: la o lote inteiro cai. Registre a divergencia no
`manager-parity-runner` como esperada, com o motivo, em vez de deixar o runner vermelho ou de
mudar o Java. Se o runner nao tiver como marcar divergencia esperada, me chame antes de improvisar.

---

# Item 2 — UNIQUE em eventos_sessao: SAIU DESTA RODADA

**Decisao do Marcos, 2026-08-24: nao segura producao, fica para o futuro. Nao desenvolva.**

Mantenho aqui o que apurei, para quando voltar — e porque parte disso e correcao de uma coisa que
eu afirmei errado e voce nao deve herdar.

**O que eu errei.** Coloquei este item na linha de corte dizendo que ele fecha a brecha do D5 — o
LOGOUT em duplicata quando o pipe cai no instante do logoff. **Fui conferir o codigo do Agent e
nao fecha.** Os dois LOGOUT daquele cenario carregam `ocorreuEm` **diferentes**:

- o do **Service** usa o instante do EOF do pipe (`SessionLifecycleTracker.UltimoEof`) — medido em
  21/08: `20:02:58.8375020Z`;
- o do **worker**, se existisse, usaria o `DateTimeOffset.UtcNow` do momento em que ele tratou o
  SHUTDOWN — outro milissegundo.

UNIQUE por igualdade exata de `ocorreu_em` **nao junta os dois**. A chave funcionou em
`eventos_janela`, `eventos_ociosidade` e `eventos_reuniao` porque naqueles casos abertura e
fechamento compartilham o mesmo `iniciado_em`. Aqui nao ha campo compartilhado.

**E a duplicata do D5 nunca foi observada.** As tres classes medidas ja estao corrigidas.

**Quando voltar, o desenho e:**

- Chave `(agente_id, tipo_evento, ocorreu_em)` pega **reenvio** — buffer que sobe duas vezes,
  replay fora da janela de 10min do `BatchDedupService`. Classe real, menor.
- Fechar o D5 de verdade exige `logon_id` vindo do Agent: campo novo no contrato, coluna nova e a
  ordem de deploy do `REGRAS-RELEASE` secao 3. So com incidente medido.
- **Nunca** janela de tolerancia — mesmo criterio que voce escreveu na migration da reuniao.
- Antes de qualquer migration, **medir** duplicatas existentes; havendo, parar e levar criterio de
  saneamento para o Marcos aprovar, como no D1.
- `agente_id` e nullable em `eventos_sessao` (diferente de `eventos_reuniao`) — linha com
  `agente_id` nulo nao seria coberta, e isso teria de estar no cabecalho da migration.
- No handler, `ON CONFLICT DO NOTHING` e nao `DO UPDATE`: evento de sessao e pontual e imutavel,
  nao ha o que reconciliar.

# Achado de lado, para o seu backlog — nao e desta rodada

`eventos_sessao.motivo` e `varchar(50)`, e `session.handler.ts:27` trunca em 50. Consistente.

Mas eu disse ao @Bucky em 21/08 que o campo era "255, texto livre" quando ele parametrizou o
`motivo` do LOGOUT do Service. **Estava errado sobre o numero** — acertei sobre ser texto livre e
nao enum, que era o que importava para a regra de enum novo. Os valores que ele usa
(`OsLogoff`, `MachineShutdown`) tem menos de 20 caracteres, entao nao ha efeito. Registro para nao
virar folclore.

---

# Ordem e limites

1. **Causa 1** (o `nullish`) — e uma linha e desarma o defeito medido
2. **Causa 2** (validacao por item + `motivosIgnorados`)

**So isso.** O item 2 saiu por decisao do Marcos; nao desenvolva.

**Pare e me chame:** se o parity runner nao souber marcar divergencia esperada; se a validacao por
item exigir mexer em handler.

**Nao commite. Nao faca push.** Ao terminar, avise que o **@Tony revisa**.

**O que o review vai cobrar:**

- teste com o payload exato que foi recusado em producao, nao um payload inventado
- teste do caminho contrario: erro de lote continua 400
- nenhum teste existente reescrito para acomodar a resposta nova sem justificativa no proprio teste
- nenhum `catch` novo que engula falha em silencio
- a divergencia de paridade registrada, nao escondida
