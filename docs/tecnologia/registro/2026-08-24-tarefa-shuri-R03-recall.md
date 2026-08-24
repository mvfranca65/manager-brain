# Tarefa para @Shuri — R-03: recall de frota, lado backend (2026-08-24)

> **De:** @Tony | **Aprovador:** Marcos
> **Para ser executado numa sessao nova.** Este documento e autossuficiente: leia so ele.
> **Nao commite. Nao faca push. Ao terminar, @Tony revisa.**
>
> **Repo desta tarefa: `manager-srv-admin-node`.**

---

# 1. Contexto — leia antes de abrir qualquer arquivo

## 1.1 Onde as coisas estao

Diretorio de trabalho: `C:\Users\NoisyTech\Documents\Manager\`

| Repo | O que e |
|---|---|
| `manager-srv-admin-node` | **o desta tarefa.** Vinculo, config e updates do Agent (NestJS + Drizzle) |
| `manager-srv-events-node` | ingestao de eventos — seu tambem, nao mexe aqui |
| `manager-srv-agent` | Agent Desktop C# — @Bucky |
| `manager-brain` | documentacao |

**Atencao:** o prompt de sessao manda ler arquivos em `.brain/`. **Essa pasta nao existe** — o
cerebro esta em `manager-brain/docs/`. E `time.md` diz que voce nao mexe em `srv-admin`; esta
desatualizado, o `manager-srv-admin-node` e seu. Sigo a realidade do repo.

## 1.2 O problema que esta tarefa resolve

Hoje **nao existe** jeito de mandar a frota voltar de versao.

| Mecanismo | O que faz | O que NAO faz |
|---|---|---|
| `versoes_agente.pausada = true` (kill switch) | para de entregar a quem ainda nao pegou | **nao mexe em quem ja atualizou** |
| `rollout_percent` | limita exposicao daqui para frente | idem |
| Rollback automatico no Agent (feito em 24/08) | volta a versao anterior **quando o Service nao sobe 3x** | nao age se a versao sobe e esta errada de outro jeito |

E **republicar a versao antiga nao funciona**: seu `agent-update.service.ts` so aceita candidata
que passe em `isVersaoMaior(candidata, versaoAtual)` (linha ~183), e o Agent tem trava equivalente
(A-46). As duas estao certas — evitam laco de reinstalacao — e sao justamente o que impede um
downgrade.

**Cenario que fica descoberto:** versao sobe normalmente, passa pelo rollout, chega a 100% da
frota, e so entao se descobre que ela captura errado. Hoje isso so se desfaz maquina a maquina,
a mao.

## 1.3 O que ja existe do outro lado

Em 24/08 o Agent ganhou o mecanismo de restauracao local:

- cada maquina guarda a instalacao anterior em `C:\Program Files\bin.previous`, gravada pelo
  script de update antes de sobrescrever;
- `run-rollback.ps1` para os servicos, restaura o backup por cima e sobe de volta;
- o resultado e reconciliado no boot seguinte.

**A peca dificil ja esta pronta e testada.** Falta o gatilho vindo do servidor — que e esta tarefa.

## 1.4 Estado do repo

```powershell
cd C:\Users\NoisyTech\Documents\Manager\manager-srv-admin-node
```

| | |
|---|---|
| Ultimo commit | `f306951` |
| Testes | `pnpm test` (vitest). **Rode antes de comecar e anote o numero** |

Se o `pnpm` nao estiver no PATH, use `node_modules\.bin\vitest.cmd run`.

---

# 2. Decisoes ja tomadas — nao reabrir

Estao no refinamento `registro/2026-08-24-refinamento-recall-de-frota.md`, aprovadas pelo Marcos.

| Decisao | Por que |
|---|---|
| **No Windows: reverter local**, nao baixar versao antiga | o backup ja esta em cada maquina; baixar custaria ~310MB por maquina e exigiria relaxar `isVersaoMaior`, que tem paridade com o Java |
| **Canal: a resposta do `verificar`** | os dois Agents ja consultam esse endpoint. Sem canal novo |
| **Latencia de ate 6h aceita** | palavra do Marcos: *"o importante e voltar a funcionar sem precisar de atuacao manual de alguem"*. **Nao invente polling mais curto** |
| **Nao usar o heartbeat do `srv-events`** | e o caminho quente da ingestao; misturar controle de frota com ingestao acopla duas coisas que precisam falhar separado |
| **Escopo: so Windows** | o app fica de fora — decisao do Marcos em 24/08, motivo na secao 3.2 |

---

# 3. O que fazer

## 3.1 Coluna nova em `versoes_agente`

**Uma so.**

| Campo | Tipo | Significa |
|---|---|---|
| `pausada` *(ja existe)* | BOOLEAN | nao entregue a quem ainda nao tem |
| `revogada` *(nova)* | `BOOLEAN NOT NULL DEFAULT false` | **quem ja tem, volte a versao anterior** |

> **Nota de escopo — 2026-08-24, decisao do Marcos.** Cheguei a especificar aqui uma coluna
> `versao_alvo`, para a mesma API servir ao Agent Android. **Ela saiu.** O motivo esta na secao
> 3.2: no Android um "recall" e, tecnicamente, **uma atualizacao normal** — publicar um build com
> `versionCode` maior contendo o comportamento antigo. Isso ja funciona hoje pelo caminho de update
> que existe, sem campo novo e sem codigo novo. O alvo explicito nao servia nem ao Android nem ao
> Windows; era complexidade sem uso. **Se um dia voltar, e uma coluna nulavel — migration trivial.**
>
> **Esta tarefa e so Windows.**

**Migration** em `src/common/db/migrations/`, seguindo a convencao dos vizinhos
(`20260808120003_add_sistema_operacional_versoes_agente.sql` e um bom molde: cabecalho explicando
contexto, regra de auditoria previa quando aplicavel, DDL comentado).

Schema TS: `src/common/db/schema/versoes-agente.schema.ts` — acrescente o campo e a linha
correspondentes no bloco de comentario que lista as colunas (ele documenta cada uma).

**Nao ha saneamento a fazer** — coluna nova com default seguro, nenhuma linha existente muda de
comportamento.

## 3.2 Por que esta tarefa e so Windows

O Agent Android fica de fora, e o motivo e util registrar — ele explica por que nao ha campo de
alvo na API.

Conferido no repo `manager-srv-agent-android` em 24/08:

- a instalacao do APK e por `ACTION_VIEW` (`ApkDownloadWorker.kt:102`), o fluxo padrao de instalar
  app de origem desconhecida, com o colaborador tocando para confirmar — como tem de ser em BYOD
  sem MDM;
- por esse caminho o Android **recusa instalar APK com `versionCode` menor** que o instalado;
- desinstalar antes apagaria os dados do app — buffer de eventos nao enviados, vinculo, permissoes;
- e nao ha copia do APK anterior guardada no aparelho.

**Ou seja: no app, "voltar" e publicar um build com `versionCode` MAIOR contendo o comportamento
antigo.** Isso e uma **atualizacao normal**, que o caminho de update ja entrega hoje, sem campo
novo e sem codigo novo dos dois lados.

**Decisao do Marcos, 24/08:** o app fica de fora desta rodada; quando precisar, ele publica a
versao pelo processo de release de sempre. O contexto completo, se um dia isto voltar, esta em
`registro/2026-08-24-android-recall-descartado.md` — **que nao esta em execucao**.

## 3.3 O `verificar` passa a poder mandar reverter


Arquivo: `src/modules/agent-update/agent-update.service.ts`, metodo `verificarAtualizacao`.

**A regra:** antes de procurar candidata de update, checar se a **versao atual do agente**
(`input.versaoAtual`, para o `input.sistemaOperacional` dele) esta marcada como `revogada`. Se
estiver, a resposta manda reverter em vez de oferecer update.

**Ponto de atencao — o formato da versao.** `input.versaoAtual` chega do Agent como tres partes
(`1.5.13`), porque o `ObterVersaoSemVer()` dele monta `major.minor.build`. Mas o Agent ja mandou
formato de quatro partes no passado, e isso quebrou o update inteiro em silencio — e o achado
**A-28**, que voce ja conhece. **Normalize antes de comparar** e escreva teste com `1.5.13`,
`1.5.13.0` e `1.5.13+hash`. O @Bucky esta criando um normalizador equivalente do lado dele pelo
mesmo motivo.

**Ordem importa:** a checagem de recall vem **antes** do laco de candidatas. Uma maquina numa
versao revogada nao deve receber oferta de update — ela tem de voltar primeiro.

## 3.4 O campo novo na resposta

`src/modules/agent-update/dto/verificar-atualizacao.response.dto.ts`.

Campos **opcionais**, aditivos. Sugestao de forma (decida os nomes, mas o sentido e este):

```
recallSolicitado?: boolean     // esta versao foi revogada; saia dela
recallMotivo?: string           // texto do operador, para log e diagnostico
```

- Nao ha URL nem checksum: o Windows restaura a copia que ja esta no disco dele.
- Agent antigo ignora campo desconhecido — nao quebra nada.
- Quando ha recall, `atualizacaoDisponivel` deve ser **false**: nao ha update a oferecer, ha uma
  saida a pedir. Documente isso no `@ApiProperty`.
- **O contrato exato que voce fechar aqui e o que o @Bucky vai implementar.** Escreva-o no
  handoff da secao 8 — ele nao pode adivinhar.

## 3.5 O endpoint para disparar

Molde pronto: `fleet-health.controller.ts`, `POST pausar-versao` (linha ~252) e
`fleet-health.service.ts`, `pausarVersao` (linha ~283). Copie a forma, nao o conteudo.

Exigencias:

- `@RequireAdmin()` — como o `pausar-versao`;
- **motivo obrigatorio** no corpo;
- **audit** com autor e motivo, no padrao do `KILL_SWITCH_APPLIED`;
- **idempotente**: versao ja revogada devolve sem efeito colateral, como o `already-paused` faz;
- versao inexistente -> 404, como o `pausar-versao` ja faz.

## 3.6 Revogar tem de pausar junto — regra dura

**Revogar sem pausar cria um laco:** a maquina volta para a versao anterior e o ciclo seguinte
reinstala a revogada, porque ela continua sendo entregue.

Faca as duas coisas **na mesma transacao**. Se por algum motivo isso nao for possivel, **pare e me
chame** — nao entregue com as duas separadas confiando em quem dispara lembrar de apertar os dois
botoes.

## 3.7 Paridade

O Java nao tem nada disto. **Registre como divergencia intencional** em
`src/modules/agent-update/agent-update-verificar.parity.ts`, no bloco `intentionalDiffs`, com o
motivo — o arquivo ja tem o padrao e ja lista outras. **Nao deixe o runner vermelho e nao mexa no
Java.**

---

# 4. O que NAO mexer

| Area | Por que |
|---|---|
| `isVersaoMaior` e a comparacao SemVer do laco de candidatas | esta certa e tem paridade byte-a-byte com o Java. O recall e um caminho **separado**, antes do laco |
| Filtros de `canary_only` e `rollout_percent` | provados, e o recall nao passa por eles — quem esta na versao revogada volta, canario ou nao |
| `pausada` e o `pausarVersao` existentes | continuam como estao. Voce **usa**, nao altera |
| `agent-update.repository.ts`, query base `WHERE ativa=true AND pausada=false` | nao mexer. O recall consulta a versao **atual do agente**, que pode estar pausada — e uma consulta diferente, nao um relaxamento desta |
| `manager-srv-events-node` | outro repo, fora do escopo |
| Qualquer coisa no `manager-srv-agent` | e do @Bucky. Voce entrega o contrato; ele obedece |

---

# 5. Ordem de deploy — e a consequencia que precisa estar escrita

`REGRAS-RELEASE` secao 3: **backend primeiro, Agent depois.**

**Diga isto na documentacao do endpoint, com todas as letras:** o recall so funciona em maquina
cujo Agent entende o comando. A frota de hoje (1.5.12) vai ignorar. **O recall protege as versoes
lancadas DEPOIS de ele existir, nunca a que estiver rodando quando ele for publicado.**

Quem for usar o botao precisa saber disso antes de contar com ele.

---

# 6. Testes

Cenarios completos em
`manager-brain/docs/tecnologia/agent-desktop/TESTES-ROLLBACK-E-RECALL.md`, secao 2.3. Os que sao
seus:

| # | Cenario | Esperado |
|---|---|---|
| S1 | Agent na versao revogada consulta `verificar` | resposta manda reverter, `atualizacaoDisponivel = false` |
| S2 | Agent em **outra** versao | fluxo normal de update, sem campo de recall |
| S3 | Versao revogada **e** ha versao mais nova publicada | manda reverter; **nao** oferece a nova |
| S4 | Formatos de versao (`1.5.13`, `1.5.13.0`, `1.5.13+hash`) | todos reconhecidos como a mesma versao |
| S5 | Versao revogada de **outro SO** | nao afeta o agente Windows, e vice-versa |
| S6 | Endpoint sem Admin JWT | 401/403 |
| S7 | Endpoint sem motivo | 400 |
| S8 | Revogar versao inexistente | 404 |
| S9 | Revogar duas vezes | idempotente, sem efeito colateral duplicado |
| S10 | Revogar marca `pausada` junto | as duas no banco, mesma transacao |
| S11 | Audit emitido com autor e motivo | registro presente |
| S12 | Nenhuma versao revogada (caso normal) | resposta byte-a-byte igual a de hoje |
| S13 | Revogar versao de SO **Android** | **400** — fora de escopo nesta rodada, e no app o caminho e outro (secao 3.2) |

**O S12 e o que protege o resto da frota.** Se ele quebrar, voce mudou o caminho normal.

**A conferencia que eu vou cobrar:** desfaca a sua correcao, rode os testes, confirme que falham,
restaure, e **diga no relatorio quantos falharam e quais**. Sem isso eu nao aceito o "passou".

Guardas da base: `pnpm typecheck` e `pnpm lint` limpos. Ha um teste
(`test/unit/swagger/conformance.test.ts`) que **ja falha por ambiente** — falha de setup, nao sua.
Confirme que ele ja falhava antes de voce comecar, e ignore.

---

# 7. Aceite

- [ ] Coluna `revogada` criada, com migration comentada e schema TS atualizado
- [ ] Revogar versao de SO Android e recusado com 400 (fora de escopo nesta rodada)
- [ ] Revogar marca `pausada` junto, na mesma transacao
- [ ] Agent na versao revogada recebe ordem de reverter, com `atualizacaoDisponivel = false`
- [ ] Agent em outra versao nao vê diferença nenhuma
- [ ] Formatos de versao diferentes reconhecidos como a mesma
- [ ] Endpoint com Admin JWT, motivo obrigatorio, audit, idempotente, 404 para versao inexistente
- [ ] Divergencia registrada no parity runner
- [ ] Limitacao "so protege versoes futuras" documentada no endpoint
- [ ] Suite verde, fora a falha de ambiente do swagger
- [ ] Conferencia da secao 6 feita e relatada

---

# 8. Ao terminar

Escreva `manager-srv-admin-node/HANDOFF-R03-RECALL.md` com:

- o que mudou, arquivo por arquivo;
- **o contrato exato da resposta** — nome e forma dos campos. O @Bucky vai implementar contra isso,
  e ele nao pode adivinhar;
- contagem de testes antes e depois;
- resultado da conferencia da secao 6;
- o que voce encontrou e **nao** corrigiu, e por que;
- qualquer ponto em que este documento estava errado. **Isso e util, nao e critica** — nas duas
  ultimas rodadas o @Bucky achou coisas que eu tinha escrito errado no refinamento, e nas duas ele
  tinha razao.

**Pare e chame o @Tony** se: revogar e pausar nao puderem ser atomicos; o parity runner nao souber
marcar divergencia esperada; ou se a checagem de recall exigir mexer em algo da secao 4.

**Nao commite. Nao faca push.**

---

# 9. O que vem depois — nao e seu

| Item | Dono |
|---|---|
| Agent obedecer a ordem, reusando `run-rollback.ps1` | @Bucky, **depois** do seu backend estar em staging |
| **R-01** — o Agent nao reinstalar a versao que quebrou. **Pre-requisito do recall**: sem ele a maquina volta e reinstala a revogada no ciclo seguinte | @Bucky, `registro/2026-08-24-tarefa-bucky-R01-sticky.md` |
| Testes de maquina 10.11 | @Natasha / Marcos |

**Se o R-01 ainda nao estiver feito quando voce terminar, tudo bem** — o seu lado nao depende dele.
Mas o recall **nao vai ser ligado em producao** antes do R-01 estar de pe.
