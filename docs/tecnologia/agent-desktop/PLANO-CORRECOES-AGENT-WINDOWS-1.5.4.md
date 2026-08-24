> **DATA:** 2026-08-18 | **ORIGEM:** `ACHADOS-REGRESSIVO-2026-08-18.md` (25 achados + D-01)
> **DESTINO:** Agent Windows **1.5.4** | **Dono:** @Bucky | **Review:** @Tony | **Aprovacao:** Marcos
> **REGRA:** nada vai para main sem o Marcos validar. Commits em Conventional Commits.

# Plano de Correcoes — Agent Windows 1.5.4

Refinamento dos achados da rodada regressiva de 2026-08-18 em lotes executaveis.
A propria entrega da 1.5.4 serve como **teste de auto-update** (decisao do Marcos): publicar a 1.5.4
com a 1.5.3 instalada e observar o update automatico + rollback BAA.

---

## Backlog da 1.5.5 — encontrado validando a propria 1.5.4

Nao entra na 1.5.4 (ja esta instalada e rodando em staging). Fechar em lote depois.

| # | Achado | Estado do codigo | O que falta |
|---|---|---|---|
| 1 | **A-29** — icones fantasma na bandeja | **FEITO (2026-08-18)** — review @Tony OK | A causa era outra: o `UpdateApplier` **ja** mandava SHUTDOWN pelo pipe e esperava 10s. O que faltava era do lado de quem recebe — nada ligava o fim do host ao fim do loop de mensagens do WinForms. `StopApplication()` parava os hosted services e o processo seguia vivo ate o `Kill`, pulando o `Dispose` que remove o icone. Agora o `TrayApplicationContext` encerra o loop no `ApplicationStopping`, com `Dispose` explicito apos o `Application.Run`. Mais `TrayNotificationArea.RemoveGhostIcons()` no startup, para os fantasmas de morte forcada (crash, taskkill do suporte, instalador). 7 testes na parte pura da varredura; o resto e validacao na maquina |
| 2 | **A-30** — bloco fantasma de 1ms ao subir durante ociosidade | **REABERTO (2026-08-18)** — vai para a **1.5.6** | Validado na maquina e **reprovado**: o evento 49253 nasceu com 1ms as 14:48:26 quando o worker subiu dentro da ociosidade 4664. A ordem do loop cobre a ociosidade que comeca com o worker rodando, nao o worker que sobe dentro dela. Causa raiz agora conhecida: o `IdleMonitorService` fecha a janela com `endedAt` = inicio da ociosidade, anterior ao inicio da janela, e o clock-jump guard do `WindowActivityService` fabrica o 1ms. Ver achado A-30 REABERTO |
| 3 | **A-31** — shell sem titulo virando 50s de trabalho | **REABERTO (2026-08-18)** — vai para a **1.5.6** | Validado na maquina e **reprovado**: evento 49268, `Sistema operacional Microsoft(R) Windows(R)` / **`Program Manager`**. O filtro exige processo de shell **e** titulo placeholder; o processo bate, o titulo nao — a area de trabalho se identifica como `Program Manager`, nao "sem título". Correcao de uma linha em `TitulosPlaceholder` + teste. Ver achado A-31 REABERTO |
| 4 | **A-27** — `publicado_em` 3h no futuro | Nao e do Agent | @Shuri: gravar UTC no endpoint de publicacao e corrigir as linhas existentes |
| 5 | **A-32** — `dotnet test` sobe workers reais na instalacao | **FEITO (2026-08-18)** — review @Tony OK | Corrigido na raiz: `WorkerLauncher` valida a sessao via `IWtsSessionQuery` antes de qualquer plano; varredura de worker desgarrado no Plano C sob semaforo global; testes reescritos com dublê; `.runsettings` com `Category!=RequiresIsolation` aplicado via `RunSettingsFilePath`. **Aceite comprovado:** 253 testes verdes, contagem de `ManagerAgent.SessionWorker` inalterada (1, o legitimo), nenhum log sintetico novo |
| 6 | **A-33** — duas fontes de estado de lockscreen | **FEITO (2026-08-18)** — review @Tony OK | `LockScreenDetector` e o dono unico: `SessionSwitch`, `PowerModeMonitor` (3a fonte, encontrada no caminho) e o polling so alimentam. Transicao unica faz flush + emissao + `Pause()` na ordem certa; dedup por estado, sem janela de tempo. Polling passou a usar `SessionFlags` (A-34) no lugar do `OpenInputDesktop`. Resume de sleep usa `RefreshState` em vez de afirmar UNLOCK. 15 testes novos. **Aceite (1 LOCK e 1 UNLOCK por ciclo, sem janela de lockscreen) so na maquina com a 1.5.5** |
| 10 | **A-37** — suite grava no estado da instalacao de producao | **FEITO (2026-08-18)** — review @Tony OK | Achado novo, encontrado corrigindo o A-34. `ConfigPaths.BaseFolderOverride` (internal, so testes) + sandbox por teste em `SessionMonitorRegistrationTests`. **Pendencia na maquina:** apagar `C:\ProgramData\ManagerAgent\data\session-state-1.json` antes de validar a 1.5.5 |
| 7 | **A-34** — dedup de LOGIN nunca funcionou (`WTSLogonTime` nao suportado) | **FEITO (2026-08-18)** — review @Tony OK | `WtsHelpers.QuerySessionInfoEx` le a classe 25 (`WTSINFOEX`, layout ANSI de 160 bytes) e alimenta `QueryLogonTimeTicks` + o novo `IsSessionLocked` (insumo do A-33). **Aceite 1 e 4 comprovados nesta maquina:** `QueryLogonId(1) = 1:134314716844003445:NoisyTech` (era null), `IsSessionLocked(1) = False`, 9 testes novos incluindo trava de layout. **Aceites 2 e 3 (matar worker nao gera LOGIN / logon real continua gerando) so na maquina com a 1.5.5 instalada** |
| 8 | **A-35** — kill do worker deixa janela aberta para sempre | **FEITO (2026-08-18)** — review @Tony OK | `OrphanWindowEventCloser` novo no Service: le o ultimo `WindowActivityEvent` da sessao no buffer (`IServiceEventBuffer.FindLastEventAsync`) e, se estiver aberto, reemite com `finalizadoEm` = instante do EOF. Chamado no `OnWorkerDisconnectedAsync` **antes** do retorno por logoff — o logoff e justamente um caso em que ninguem mais fecharia. `motivo=WorkerDied` foi para o log, nao para o payload: `EventoJanela` nao tem esse campo e cria-lo exigiria mudanca de contrato e schema (@Thor/@Shuri). 18 testes novos |
| 9 | **A-36** — LOGIN/LOGOFF sem upload imediato | **FEITO (2026-08-18)** — review @Tony OK | Eram **dois** pontos, nao um: `PipeMessageHandler.ContainsSessionBoundary` (Service, fura o ciclo de 60s) e `SessionEventService` (SessionWorker, fura o dreno de 10s). So o primeiro estava no achado; sem o segundo o aceite de 5s nao fecharia. Regra virou fonte unica em `Shared/Session/SessionEventTypes.cs`. **O tipo e `LOGOUT`, nao `LOGOFF`** — o Agent nunca emitiu essa string, e ha teste travando a divergencia. 12 + 3 testes novos |

## Backlog da 1.5.6 — encontrado validando a 1.5.5 (2026-08-18)

Versao dos `.csproj` elevada para **1.5.6**. Suite: **616 testes, 0 falhas**. Nada commitado.

| # | Achado | Estado | O que foi feito |
|---|---|---|---|
| 1 | **A-30 REABERTO** — bloco de 1ms | **PARCIAL (2026-08-18)** — review @Tony OK | O bloco deixou de ser fabricado: fim anterior ao inicio agora fecha com **duracao zero** e loga WARN, em vez de inventar 1ms em silencio. **A causa estrutural continua aberta** — ver ressalva abaixo |
| 2 | **A-31 REABERTO** — area de trabalho e `Program Manager` | **FEITO (2026-08-18)** — review @Tony OK | `Program Manager` somado a `TitulosPlaceholder`, com teste do par exato capturado em campo e guarda contra bloquear um app comum de mesmo titulo. Confirmado que nao ha override no `appsettings` |
| 3 | **A-38** — LOGOUT da parada so sobe no proximo boot | **FEITO (2026-08-18)** — review @Tony OK | `ManagerAgentService.StopAsync` faz uma tentativa final de upload antes do checkpoint, com teto de 5s e CTS proprio. `RunBoundedAsync` extraido e coberto por 5 testes — travar ali faria o SCM matar o Service |

### Ressalva do A-30 — por que ficou parcial

A correcao proposta ("nao abrir janela enquanto a sessao ja esta ociosa") **e impossivel como
escrita**. A cronologia medida na maquina prova:

| Instante | Fato |
|---|---|
| 14:48:20.912 | Ultima interacao do usuario |
| 14:48:26.202 | Janela abre — ociosidade de **5s**, limiar e 60s. A sessao **nao** esta ociosa |
| 14:49:23.161 | `User became idle (idle >= 62s)` — o limiar cruza |
| — | `_idleStartUtc` retroage para 14:48:20.912, **antes** do inicio da janela |

No momento da abertura a ociosidade ainda nao existe: ela e datada retroativamente na ultima
interacao e so descoberta 60s depois. **E nao e especifico de restart de worker** — qualquer
janela que ganhe foco no intervalo silencioso entre a ultima interacao e o limiar sofre o mesmo,
inclusive um aplicativo roubando foco sozinho.

Fechar o evento tambem nao e opcional: ele ja foi emitido como **aberto** quando nasceu, e engolir
o fechamento o deixaria orfao para sempre (achado A-35).

**O que resta decidir** (precisa de rodada propria, nao de remendo):

1. **No Agent** — so emitir o evento de janela quando houver interacao do usuario no periodo,
   em vez de emitir na abertura. Mexe no modelo de emissao e toca A-12, A-16 e A-18.
2. **No backend** (@Shuri/@Jarvis) — descartar blocos de duracao zero no pipeline de relatorio.
   Com o fim deste lote, esses blocos passaram a ser identificaveis por `finalizado_em = iniciado_em`,
   o que antes nao era possivel (1ms parecia dado real).

A opcao 2 e barata e resolve o efeito hoje; a 1 resolve a causa e merece refinamento.

---

**A-28 (backend)** ja foi para producao de staging pelo Marcos e esta **validado em campo**: o
auto-update rodou de ponta a ponta pela primeira vez (1.5.3 -> 1.5.4, 2026-08-18 11:24-11:26).
A parte do Agent (`ObterVersaoSemVer`) esta na 1.5.4.

---

## Status de execucao (2026-08-18)

Versao dos `.csproj` ja elevada de 1.5.3 para **1.5.4**. Suite: **529 testes, 0 falhas**
(era 506/6 antes da rodada). Nada commitado — Marcos valida primeiro.

| Lote | Status | Revisao @Tony |
|---|---|---|
| L1 — Ciclo de vida dos eventos | **FEITO** | OK, com 1 correcao (snapshot lia a janela 2x no mesmo tick) |
| L2 — Disponibilidade dos workers | **FEITO** | OK, com 1 correcao (guard de sessao ativa para nao relancar em logoff) |
| L3 — Envio imediato e telemetria | **FEITO** | OK, com 1 correcao (`PendingCount` acessava SQLite sem o lock de escrita) |
| L4 — Desvinculacao no uninstall | **FEITO** | OK |
| L5 — Diagnostico confiavel | **FEITO** | OK, com 1 correcao (em-dash violava a regra de PS1 somente-ASCII — o mesmo A-05) |
| L6 — Identidade e metadados | **FEITO** | OK — causa raiz do A-03 encontrada no caminho |
| L7 — Higiene | **PARCIAL** | A-05, A-06 e A-24 feitos; A-25 descartado como falso achado |
| L8 — Documentacao | **PENDENTE** | @Natasha (A-09) e @Steve (A-15) |

**Falta validar em maquina:** todo o L1–L6 esta coberto por teste unitario, mas o comportamento
de ponta a ponta so se confirma com a 1.5.4 instalada e os dados chegando na base.

---

## Ordem de execucao

| Lote | Tema | Achados | Risco | Bloqueia release? |
|---|---|---|---|---|
| **L1** | Ciclo de vida dos eventos (paridade Android) | A-12, A-16, A-17, A-18, A-14 | ALTO | **SIM** |
| **L2** | Disponibilidade dos workers | A-20, A-21 | ALTO | **SIM** |
| **L3** | Envio imediato e telemetria de fila | A-19, A-22, A-23 | MEDIO | **SIM** |
| **L4** | Desvinculacao no uninstall | A-01 | ALTO | **SIM** |
| **L5** | Diagnostico confiavel (scripts) | A-10, A-11 | MEDIO | Nao |
| **L6** | Identidade e metadados | A-02, A-03, A-04 | MEDIO | Nao |
| **L7** | Higiene: hooks, log, ASCII, testes | A-24, A-25, A-05, A-06 | BAIXO | Nao |
| **L8** | Documentacao | A-09, A-15, D-01 | BAIXO | Nao |

L1 e L2 sao independentes entre si e podem ir em paralelo. L3 depende de L1 (o drain imediato
precisa dos eventos abertos ja existindo).

---

# L1 — Ciclo de vida dos eventos (paridade com o Agent Android)

**Achados:** A-12, A-16, A-17, A-18, A-14 | **Referencia canonica:** `manager-srv-agent-android`

## Principio

O Agent emite **fatos crus**. Todo evento nasce **aberto** (`finalizadoEm` ausente) e e fechado por um
evento posterior com o **mesmo `iniciadoEm`**. O backend reconcilia por upsert — as chaves UNIQUE
`(agente_id, nome_processo, iniciado_em)` e `(agente_id, iniciado_em)` ja existem e a regra de merge
"primeiro `finalizadoEm` nao-nulo vence" ja esta implementada. **Nenhuma mudanca de backend e necessaria
para L1.**

## L1.1 — Janela abre imediatamente e faz snapshot a cada 60s

`src/ManagerAgent.SessionWorker/Capture/WindowActivityService.cs`

Hoje `HandleAsync` cria `_currentEvent` em memoria e **so chama `_eventBuffer.AddAsync` na troca**.
Resultado: quem fica 3h no mesmo app nao gera evento nenhum (A-12).

| Momento | Comportamento exigido |
|---|---|
| Janela ganha foco | `AddAsync` do evento com `FinalizadoEm = null` |
| Janela segue em foco | reemitir o **mesmo** evento aberto a cada `SnapshotJanelaSegundos` (default **60**) |
| Troca de janela | `AddAsync` da anterior com `FinalizadoEm` = inicio da nova, depois abre a nova |
| Encerramento (`FlushCurrentEventAsync`) | ja correto — manter |

- O `iniciadoEm` **nunca muda** entre o open, os snapshots e o close — e a chave de reconciliacao.
- Novo campo em `appsettings.json`: `SnapshotJanelaSegundos: 60`.
- `JsonDefaults.IgnoreNull` ja omite o campo nulo — **nao trocar** por `null` explicito.

## L1.2 — Debounce antes de promover a janela

Portar `WindowActivityCollector.promotePending` (Android, `debounceMillis = 1_500L`).
Uma janela so vira evento apos permanecer em foco por `DebounceJanelaMs` (default **1500**).
Resolve o fantasma `unknown` de 1ms do A-18 e reduz o ruido do A-14.

## L1.3 — Blocklist de janelas do proprio agent e do shell (A-14)

Ignorar, antes de qualquer emissao:
- processos do proprio agent (`ManagerAgent*`)
- shell do Windows: bandeja ("Janela de estouro da bandeja do sistema"), alt-tab, menu iniciar,
  `Windows.UI.Core.CoreWindow`, `ApplicationFrameHost` sem titulo
- `nome_processo` vazio ou `"unknown"`

Blocklist em `appsettings.json` (`ProcessosIgnorados`), com defaults no codigo — configuravel sem rebuild.

## L1.4 — Ociosidade abre e fecha como o Android

`src/ManagerAgent.SessionWorker/Capture/IdleMonitorService.cs`
`src/ManagerAgent.SessionWorker/Capture/Storage/IdleEventRecord.cs`

Hoje o idle so e persistido **fechado** (`PersistIdleEventAsync`), no retorno do usuario.

1. `IdleEventRecord.EndUtc` passa a `DateTimeOffset?` e `DurationSeconds` a `long?`; ambos **omitidos**
   no JSON quando nulos.
2. Ao detectar `idleTime >= _idleThreshold` (60s): persistir **imediatamente** o registro aberto, com
   `StartUtc = nowUtc - idleTime` (instante real da ultima interacao) e `EndUtc = null`.
3. Ao voltar o input: persistir de novo com o **mesmo `StartUtc`** e `EndUtc` real. O upsert do backend fecha.
4. `MinIdleDurationSeconds = 60` deixa de ser filtro de descarte no fechamento — o registro ja foi enviado
   aberto. Manter o filtro **apenas** no caminho sintetico (`WallClockGap`), que nasce fechado.
5. **Nao** reemitir snapshot durante ociosidade longa (decisao consciente do Android: "ainda ausente" e
   derivavel da linha aberta).

**Armadilha do Android, nao repetir:** o inicio da ausencia **nunca** pode sair com
`finalizadoEm = clock()` — isso fabrica fechamento falso e gera duas linhas nao reconciliadas.

## L1.5 — Ociosidade fecha a janela em andamento (A-16)

No instante em que a ociosidade e detectada:
1. `WindowActivityService.FlushCurrentEventAsync(inicioDaOciosidade)` — fecha a janela com
   `finalizadoEm` = inicio da ociosidade (nao o instante da deteccao);
2. abre a ociosidade (L1.4);
3. `WindowActivityService.Pause()` enquanto ocioso;
4. no retorno: fecha a ociosidade, `Resume()`, e a proxima `HandleAsync` abre a nova janela.

Evidencia do problema: bloco `Windows Terminal` 12:43:57 -> 12:48:59 sobreposto integralmente a
ociosidade 12:43:56 -> 12:48:58 — 5 min de ausencia contados como janela ativa.

## L1.6 — Boundaries de LOCK / UNLOCK (A-18)

Portar `IdleDetector.resetSessionBoundary` do Android.

**LOCK:** fechar a janela em andamento com `finalizadoEm` = instante do lock; se havia ociosidade em
andamento, fechar tambem; **nao criar registro novo** (hoje cria o fantasma `unknown` de 1ms, id 48376).

**UNLOCK:** fechar o que restou aberto e abrir o evento da janela que ganhou foco, com `finalizadoEm = null`
(decisao do Marcos: "a pessoa vai focar em algo"). O debounce de L1.2 e **obrigatorio** aqui — no instante
do unlock o Windows reporta a lock screen como janela ativa por fracao de segundo.

## Criterios de aceite de L1

Com o agent rodando em staging:

- [ ] `select count(*) from eventos_janela where agente_id=<id> and finalizado_em is null` = **1** enquanto
      houver janela em foco (nunca 0, nunca >1)
- [ ] Ficar 5 min no mesmo app gera **1 linha** (aberta, com snapshots reconciliados), nao 0 e nao 5
- [ ] Nenhum bloco de `eventos_janela` sobrepoe um bloco de `eventos_ociosidade`
- [ ] Entrar em ociosidade: a linha da janela fecha e a linha de ociosidade nasce com `finalizado_em is null`
- [ ] LOCK nao gera linha `nome_processo = 'unknown'`
- [ ] Zero linhas com `nome_processo like 'ManagerAgent%'` ou titulo de bandeja do sistema
- [ ] Zero duplicatas: `group by nome_processo, iniciado_em having count(*) > 1` retorna vazio
- [ ] Nenhum payload contem `"finalizadoEm": null` (campo omitido, nunca nulo explicito)

---

# L2 — Disponibilidade dos workers

## L2.1 — Relancar no EOF do pipe, nao no timeout de heartbeat (A-20)

`src/ManagerAgent.Service/Pipe/PipeServer.cs:147`

O EOF ja e detectado **na hora** (09:32:07) mas so loga; o relancamento so sai pelo `WorkerWatchdog`
por timeout de heartbeat (09:33:44). **97s sem captura nenhuma**, com o Service enviando batimentos
normalmente — no portal o agente aparece saudavel enquanto nao coleta nada.

No `break` do EOF em `ReadLoopAsync`, invocar `HandleWorkerDeathAsync(conn.SessionId)`
(ja existe em `PipeServer.cs:247`).

- Manter o `WorkerWatchdog` por heartbeat como **fallback** — e o unico sinal quando o worker trava vivo
  sem fechar o pipe.
- Cuidado: EOF tambem ocorre em shutdown legitimo e em logoff. Nao relancar se o Service esta parando
  (`ct.IsCancellationRequested`) nem se a sessao nao esta mais ativa.
- O rate-limit existente (`MaxRelaunches = 3/5min`) continua valendo como protecao contra crash-loop.

**Aceite:** matar o SessionWorker pelo Gerenciador de Tarefas -> worker de volta em **<= 15s** (criterio
R13.1.4). Medido hoje: 97s.

## L2.2 — Launch idempotente por sessao (A-21)

`src/ManagerAgent.Service/SessionManagement/WorkerRegistry.cs`

`RegisterOrphan` grava `State = WorkerState.Launching`, mas o guard de `TryLaunchAsync` (linha 58) so
aborta quando `State is Connected or Paused`. Resultado: ao reiniciar o Service, ele **adota** o orfao e
**32ms depois lanca outro** para a mesma sessao (PIDs 8888 e 19424 confirmados no SO).

Correcao: o guard passa a abortar sempre que houver processo vivo registrado para a sessao —
`existing.Process is { HasExited: false }` — independente do `State`, com um teto de idade para
`Launching` travado (ex.: `LaunchedAt` ha mais de 60s sem conectar = considerar morto e substituir,
matando o PID antigo antes).

Adicionalmente: ao adotar um orfao que ja conectou o pipe, promover para `Connected`.

**Aceite:** `taskkill` no `ManagerAgent.Service` -> apos o restart do SCM existe **exatamente 1**
`ManagerAgent.SessionWorker.exe` por sessao. Cobrir com teste `ServiceRestart_ComWorkerVivo_NaoLancaSegundo`.

**Nao mexer (funcionou):** restart pelo SCM, `AGENT_STARTUP` com `Reason=SERVICE_START`, adocao de orfao
em si, retomada do upload sem perda, `UpdateCheckerWorker` apontando para staging.

---

# L3 — Envio imediato e telemetria de fila

## L3.1 — Drain imediato em LOCK/UNLOCK (A-19)

Medido: LOCK 26s de atraso, UNLOCK **64s**. No Android o caminho quente ja e direto —
`sender/UploadCoordinator.kt`: *"o ticker de 60s e o LOCK/UNLOCK nao chamam mais esta classe — eles chamam
`EventUploader.drain` direto, no escopo do service, para nao depender do JobScheduler"*.

O evento de sessao dispara drain direto no `UploadWorker`, fora do timer de 60s.

**Requisito de robustez (@Tony):** o envio imediato **nao pode travar o lock nem propagar excecao**.
Timeout curto, fire-and-forget; sem rede degrada em silencio para o buffer e sobe no proximo ciclo.

**Aceite:** `criado_em - ocorreu_em` <= **5s** para LOCK e UNLOCK.

## L3.2 — Falha de transporte e aviso, nao erro (A-22)

`src/ManagerAgent.Service/Upload/UploadWorker.cs` e `HttpEventUploader.cs`

Hoje cada ciclo offline emite `[ERR]` + stack completo de `SocketException`, duas vezes (lote e ciclo).

- Falha de transporte (DNS / socket / timeout) -> `[WRN] Upload adiado — sem rede, N eventos no buffer`,
  **sem stack**.
- `[ERR]` + stack fica reservado para 5xx, contrato invalido e erro inesperado.
- Consultar `RedeDisponivel` e sair cedo do ciclo quando offline.
- Eliminar a linha duplicada (logar no ciclo **ou** no lote, nao nos dois).

## L3.3 — `EventosPendentes` real no heartbeat (A-23)

`src/ManagerAgent.SessionWorker/Capture/HeartbeatService.cs:42` — hardcoded `EventosPendentes = 0`
(mesmo bug em `ManagerAgent.Tray/Services/HeartbeatService.cs:42`).

Popular a partir do `COUNT(*)` real do buffer SQLite. Buffer com 26 eventos reportou 0 — o campo e o
unico indicador remoto de acumulo, e zerado nao distingue "nada a enviar" de "offline ha horas".

**Complemento a alinhar com @Shuri:** incluir `bufferMaisAntigoEm` no payload, para o backend calcular
o atraso real na primeira reconexao. Requer campo novo no contrato do srv-events — **nao bloqueia** L3.3.

---

# L4 — Desvinculacao no uninstall (A-01)

**Dois defeitos.**

1. **`instalador/Pacote/desvincular.ps1` nunca existiu** — nenhum commit no historico tocou nele, mas o
   `ManagerAgent-v2.iss` o chama como primeira acao do `[UninstallRun]`, com `runhidden`. Falha em silencio
   e o dispositivo fica vinculado para sempre.
2. **`scripts/build/build-pacote-v2.ps1` nao valida** — procura o arquivo em dois caminhos, nao acha e
   segue sem erro (nao ha `else`).

**Escrever o script:** le `config.json`, decripta o Device JWT (DPAPI), chama
`POST /api/agente/dispositivos/desvincular` (`@RequireDevice`, ja existe no srv-admin-node).
Best-effort com **timeout curto** — nao pode travar o uninstall. Loga o resultado em arquivo proprio
(com `runhidden` nao ha console). Sem rede: falha silenciosa e segue a desinstalacao.

**Guard no build:** arquivo obrigatorio do pacote ausente = build **falha**. Vale para os 6 scripts.

**Aceite:** desinstalar pelo Painel de Controle -> `agentes.desvinculado_em` preenchido em staging.

**Divida tecnica separada (@Thor/@Vision):** limpar os agentes orfaos ja desinstalados — ex.: **id 48**
(1.5.1, mesma maquina) esta com `desvinculado_em = NULL`.

---

# L5 — Diagnostico confiavel

## L5.1 — Auto-elevacao dos scripts (A-10)

**Decisao do Marcos:** o script deve **solicitar elevacao antes de rodar** — relaunch automatico via
`Start-Process -Verb RunAs`, nao apenas avisar.

Sem elevacao o `health-check.ps1` nao le `config.json` (ACL restrita, protecao correta), o
`UnauthorizedAccessException` **nao e tratado**, e o script conclui:

| Passo | Reportou | Realidade |
|---|---|---|
| [4/10] Configuracao | "incompleta: baseUrlAdmin, baseUrlEvents, chaveAtivacaoEmpresa, identificadorColaborador" | config completa, agent vinculado e enviando |
| [7/10] API Admin | "inacessivel — verificar internet/firewall" | acessivel, heartbeat as 11:31 |
| [8/10] API Events | "inacessivel" | acessivel, eventos na base |

Score exibido **60% (ATENCAO)**; score real **100%**. Manda o suporte cacar firewall que nao existe.

Aplicar o mesmo padrao a `coletar-diagnostico.ps1` e `test-vinculacao.ps1`.
**E tratar `UnauthorizedAccessException` explicitamente** — "sem permissao para ler" nunca pode virar
"campo ausente" ou "API fora do ar", mesmo com a auto-elevacao no lugar.

## L5.2 — Nomes de log corretos (A-11)

`health-check.ps1:225` procura a tag `[SessionWorker]` dentro de `service-*.log`. O worker tem arquivo
proprio. Arquivos reais em `C:\ProgramData\ManagerAgent\logs\`:

`service-YYYYMMDD.log` · `session-worker-S{sessionId}-YYYYMMDD.log` · `watchdog-YYYYMMDD.log` · `startup-trace.log`

Verificar o mesmo padrao em `monitorar-logs.ps1` e `coletar-diagnostico.ps1`.
**Bonus:** o health-check nao verifica o servico `ManagerAgentWatchdog` nem o `watchdog-*.log` — incluir.

---

# L6 — Identidade e metadados

| Item | Achado | Correcao |
|---|---|---|
| `usuario_sessao` grava a conta do UAC (`Raquel`) em vez do usuario real (`NoisyTech`) | A-02 | O valor deve vir do **SessionWorker** (sessao interativa real), nao do `Configurator` elevado (`Program.cs:162` usa `Environment.UserName` sob elevacao) |
| `hardware_fingerprint` e `hardware_fingerprint_atualizado_em` nulos | A-03 | Investigar: nao e enviado no vincular, e enviado por outro endpoint, ou o backend nao persiste. Feature entrou na v1.4.1 (`e05ad5f`, Eixo 5) |
| Janela "Sobre" sem `instalacaoId` | A-04 | Exigido por R2.6.3. Sem ele o suporte nao casa a maquina com o registro do backend pelo print. Expor tambem o identificador do colaborador (`i888777`), nao a conta Windows |

---

# L7 — Higiene

| Item | Achado | Correcao |
|---|---|---|
| Hooks reinstalados durante ociosidade legitima (`Hooks appear inactive (3 zero summaries)`) | A-24 | Suprimir a heuristica enquanto houver ociosidade em andamento. Zero input **e** a definicao de ociosidade — validar saude do hook por sinal proprio (handle registrado / ping), nunca por ausencia de input. Risco atual: perder o primeiro input da volta |
| ~~Mojibake no log do worker~~ — **achado descartado** na revisao do @Tony: o hexdump mostra `C3 A1`, UTF-8 correto; o mojibake era do terminal que leu o arquivo | A-25 | Fica so a parte legitima: mensagens de log em portugues violam a regra do projeto (codigo e log em ingles). Converter as mensagens tocadas na 1.5.4 |
| Em-dash em PS1 gerado (`UpdateApplier.cs:792`) | A-05 | Trocar por hifen simples. Esta em comentario, nao quebra execucao, mas viola a regra de PS1 somente-ASCII e derruba 2 testes de guarda |
| 4 falhas unitarias | A-06 | Reavaliar contra o codigo pos-`592bd33`/`f119d0c`: 3 de `AgentLinkServiceTests` (`Salvar()` 2x, `Retry` vs `ConfigIncomplete`) e 1 de `UpdateCheckerWorkerTests` (`run-update.ps1` recente deletado, spec 1.3.10 item 5). Se o comportamento novo esta certo, **atualizar o teste**; senao e regressao |

---

# L8 — Documentacao

- **A-09** — `PLANO-TESTES-REGRESSIVOS.md` v2.0.0 e de 2026-06-11 e nao cobre 8 meses. 13 correcoes
  (C1–C13) listadas no fim de `ROTEIRO-REGRESSIVO-2026-08-18.md`. Faltam: servico `ManagerAgentWatchdog`,
  vinculacao/Device JWT, eventos v1.4.x, `dispositivoTipo`, `menuVisivel`, thresholds, rollback BAA,
  bloco LGPD dedicado, `desvincular.ps1`. Corrigir tambem: sao **6** scripts e nao 5; recovery e
  **5s/10s/30s** e nao 1s/5s/30s; `monitorar-performance.ps1` nao existe mais no pacote;
  `nome_processo` grava o nome amigavel ("Google Chrome"), nao o executavel. **Dono: @Natasha.**
- **A-15** — `produto.md` e o README dizem "5+ minutos" de ociosidade; o codigo usa 60s. Com D-01 os 60s
  viram limiar **tecnico de deteccao** (correto) e o "5 min" vira regra de negocio do backend.
  **Corrigir a doc, nao o limiar.** Dono: @Steve.
- **D-01** — virar ADR: a maquina de estados sai do Agent e vai para o backend.

---

# D-01 — pendencias de arquitetura (fora de L1–L8)

A decisao esta registrada, mas **duas pecas nao sao do @Bucky** e precisam de @Tony + @Thor/@Shuri antes:

1. **Regra de negocio no srv-events:** `< 5 min` = micro-pausa (tempo reatribuido ao bloco de janela
   anterior), `>= 5 min` = PAUSA, `>= 15 min` = AUSENTE. Os numeros 5 e 15 passam a existir **em um lugar so**.
2. **Merge de micro-pausas:** com deteccao a 60s os blocos fragmentam mais. Se o backend nao mesclar, o
   pilar Fragmentacao infla. E contrapartida obrigatoria de L1, nao opcional.

**Desdobramentos no Agent (podem entrar em L1 ou ficar para 1.5.5):** depreciar `UserStatusManager` e o
campo `status_usuario` nos payloads; coordenar a remocao da tabela `eventos_transicao_status` com @Thor.
O PR4 de thresholds (`pr4-threshold-inatividade-bucky.md`) fica **obsoleto na parte de Agent** — os
thresholds 5/15 nao devem viver no C#.

Os achados **A-08** e **A-13** ficam substituidos por D-01: `status_usuario` nulo deixa de ser bug e
passa a ser o comportamento correto.

**Fora de escopo:** `isMediaActive()` do Android (video sem toque nao conta como ausencia) e especifico de
mobile. O equivalente Windows seria audio/video em foreground — avaliar depois com @Tony.

---

# Regressivo da 1.5.4

Alem de reexecutar os blocos que falharam, a 1.5.4 cobre o que ficou pendente (P-1 a P-11 em
`ACHADOS-REGRESSIVO-2026-08-18.md`): auto-update e rollback BAA (a propria entrega), performance
estabilizada com o agent rodando ha 4h+, scripts de diagnostico, multi-sessao, Named Pipe e
AutonomousBuffer, logoff/suspensao/reinicio, deteccao de reuniao, **bloco LGPD dedicado** (obrigatorio
antes de release), backend fora do ar com 5xx, e desinstalacao/reinstalacao **depois** de A-01.

## Backlog da 1.5.7 — incidente do auto-update (2026-08-18)

Versao dos `.csproj` elevada para **1.5.7** (a 1.5.6 ja esta publicada no bucket e aponta
para outro binario). Suite: **624 testes, 0 falhas**. Nada commitado.

Evidencia completa de cada item em `ACHADOS-REGRESSIVO-2026-08-18.md`, secao
"Incidente do auto-update".

| # | Achado | Sev | Estado | O que foi feito |
|---|---|---|---|---|
| 1 | **A-39** — WorkerWatchdog relanca worker no meio do update | CRITICO | **FEITO (2026-08-18)** — review @Tony OK | `UpdateGate` singleton no DI, fechado pelo `UpdateApplier` antes do broadcast de shutdown e consultado pelo `WorkerLauncher` — ponto unico por onde todo worker nasce. Fecha por TTL de 5min, nao por chamada de abertura: o caminho feliz termina em `Environment.Exit(0)` e ninguem sobrevive para reabrir. `WorkerWatchdog` tambem checa, para nao queimar o rate-limit de 3/5min. 7 testes novos |
| 2 | **A-40** — PS1/batch nunca mataram o SessionWorker | CRITICO | **FEITO (2026-08-18)** — review @Tony OK | Kill do `ManagerAgent.SessionWorker` antes do backup, no PS1 (Planos A e B) e no batch (Plano C), com espera de 15s e WARN se sobrar alguem. Corrige junto a falha de rollback, que travava no mesmo lock. Plano D dispensa: renomeia para `.old` em vez de sobrescrever |
| 3 | **A-41** — cooldown de 6h era codigo morto | ALTA | **FEITO (2026-08-18)** — review @Tony OK | `Program.Main` nao apaga mais a `update-failed.flag`; quem apaga e o checker, depois de cumprir o cooldown. Sem isso, cada restart cobraria mais 6h e o agent nunca mais updataria |
| 4 | **A-42** — `MaximoBackups` so vale para a Tray | ALTA | **FEITO (2026-08-18)** — review @Tony OK | `PruneOldBackups()` no `UpdateApplier` do Service, antes de criar o backup novo. 11 tentativas deixaram 3,4GB em disco |
| 5 | **A-43** — `WatchdogStateStore` instanciado na mao | MEDIA | **FEITO (2026-08-18)** — review @Tony OK | Store injetado por DI. Teste novo cobrindo o branch de SOS, que antes so era exercitado por acidente lendo o estado real da maquina |
| 6 | **A-44** — suite escrevia na pasta do agent instalado | MEDIA | **FEITO (2026-08-18)** — review @Tony OK | Sandbox via `ConfigPaths.BaseFolderOverride` em `UpdateCheckerWorkerTests` + colecao serial `ConfigPaths` para as 3 classes que tocam o override estatico |
| 7 | **A-45** — o fix nao consegue se auto-instalar | ALTA | **ABERTO — decisao de @Vision/@Steve** | Quem aplica o update e o applier da versao instalada. Agents em 1.5.5/1.5.6 no campo so aplicam a 1.5.7 numa janela **sem usuario logado**. Ou publica e espera a janela, ou reinstala via instalador |

### Pendencias na maquina de teste

- Modo SOS esta **ligado a mao** em `watchdog-state.json`. O `sosRecoveryHeaderCount` nunca e
  incrementado em codigo de producao, entao o SOS **nao sai sozinho** — desligar quando a 1.5.7
  estiver instalada, senao o agent nunca mais checa update.
- `C:\ProgramData\ManagerAgent\backups` com 3,4GB de tentativas falhas. O `PruneOldBackups` novo
  cuida das proximas, mas nao apaga retroativo: limpar a mao (manter as 3 mais recentes).
- `updates\1.5.6\update.zip` (285MB) pode ir junto.

### Ainda nao verificado na maquina

O aceite do A-39 e do A-40 **so fecha com a 1.5.7 instalada** aplicando um update de verdade.
Ate la, o que existe e suite verde e leitura de codigo. O teste que importa: publicar a 1.5.8
de mentira e ver a 1.5.7 aplicar com usuario logado, sem erro de lock.

### Pendencia de fora do Agent

`StickyVersion` / `StickyUntil` sao gravados pelo `RollbackOrchestrator` e **nao sao lidos por
ninguem**. Os "24h sem tentar upgradar" que o comentario promete nao existem. Somar ao backlog.

### Aceite na maquina — roteiro pronto (2026-08-18)

O aceite do A-39/A-40 tem roteiro proprio: `ROTEIRO-VALIDACAO-1.5.7.md`. Resumo:

```powershell
.\scripts\build\build-pacote-e2e.ps1 -Version 1.5.7      # build
.\scripts\build\install-build-local.ps1 -Version 1.5.7   # instala na mao (A-45)
.\scripts\build\build-pacote-e2e.ps1 -Version 1.5.99 -SkipTests
.\tests\e2e\scenarios\e8-update-com-usuario-logado.ps1   # o teste
```

O E8 e in-place: **nao** chama `teardown.ps1` (que desinstala o agent e apaga
`C:\ProgramData\ManagerAgent`, destruindo a vinculacao da maquina de regressivo).

### Correcoes na harness de E2E — ela nunca tinha rodado

| Problema | Efeito |
|---|---|
| Stub nao tinha rota de download | o ZIP caia no 404 do `else`; nenhum cenario baixaria update |
| `Stop-Job -Force` | parametro inexistente no PS 5.1; o `finally` de todos os cenarios lancava excecao |
| Nao-ASCII em `.ps1` | **3 dos 7 cenarios nao parseavam** (A-05 pela 3a vez) |
| `dist/` vazio | nenhum build do repo escrevia la; `build-v-teste.ps1` chama script inexistente |
| `MANAGER_UPDATE_CHECK_SECONDS` | usado por 3 cenarios, lido por ninguem |

Novos arquivos: `scripts/build/build-pacote-e2e.ps1`, `scripts/build/install-build-local.ps1`,
`tests/e2e/scenarios/e8-update-com-usuario-logado.ps1`,
`tests/e2e/assertions/assert-no-file-lock-error.ps1`,
`tests/e2e/assertions/assert-no-worker-launch-during-update.ps1`,
`tests/ManagerAgent.Service.Tests/PowerShellScriptsAreAsciiTests.cs`,
`tests/ManagerAgent.Service.Tests/UpdateScriptKillsWorkerTests.cs`.

Suite: **633 testes, 0 falhas**. Os 34 `.ps1` do repo parseiam.

### Revisao de 2026-08-18 (segunda passada, antes de instalar nada)

Marcos pediu certeza antes de gastar uma janela de teste. A revisao achou dois defeitos que
teriam custado exatamente a tentativa-e-erro que ele queria evitar:

| # | Achado | Sev | Estado |
|---|---|---|---|
| 8 | **A-41 REVISADO** — cooldown por `Task.Delay` recomeca a cada restart e a flag nunca e apagada: o agent **nunca mais atualizaria** | CRITICO | **FEITO** — idade lida do `LastWriteTimeUtc`, decisao imediata, flag apagada quando a janela vence |
| 9 | **A-46** — o agent aplica qualquer versao que o backend mandar, inclusive a que ja roda | ALTA | **FEITO** — guard de igualdade + 2 testes (um garante que update legitimo continua passando) |

O A-46 nao veio da maquina: veio de escrever o cenario de E2E. O primeiro rascunho do stub
respondia "ha update" para sempre e teria posto o agent em loop de reinstalacao — o mesmo
formato do incidente, agora disparavel pelo servidor contra a frota inteira.

Suite apos as duas correcoes: **638 testes, 0 falhas**. Os 34 `.ps1` do repo parseiam.

## Backlog da 1.5.8 — achados do regressivo automatizado (2026-08-19)

| # | Achado | Sev | Estado | O que foi feito |
|---|---|---|---|---|
| 1 | **A-48** — reinicio do worker durante ociosidade duplica o evento e deixa linha aberta | ALTA | **FEITO (2026-08-19)** — review @Tony OK | O inicio da ociosidade passou a ser persistido em `session-state-{sessao}.json` e reaproveitado quando o recalculo cai dentro de 5s do valor gravado. Antes, `agora - GetLastInputInfo()` era recalculado a cada processo e jitterizava ~15ms, o que furava a UNIQUE `(agente_id, iniciado_em)`. `ResetIdleState()` limpa o estado — ponto unico, cobre todos os caminhos de fechamento. 10 testes novos |
| 2 | **A-44 (2a ocorrencia)** — `BaseFolderOverride` estatico tambem em `SessionWorker.Tests` | MEDIA | **FEITO (2026-08-19)** | Colecao serial `ConfigPaths` criada tambem nesta suite. O sintoma e cruel: passa isolado, falha junto |
| 3 | **A-47** — titulo da janela carrega conteudo digitado | ALTA | **ABERTO — decisao de @Steve** | LGPD. Confirmado em staging. Nao implementar sem decisao de produto: sanitizar titulo de editor tem custo |
| 4 | **A-49** — uma linha em `agentes` por versao instalada | ALTA | **ABERTO — @Thor/@Shuri** | 12 registros para a maquina de regressivo. Fragmenta historico do colaborador |
| 5 | **A-50** — `IntervaloSinalVidaSegundos` do appsettings nao chega na opcao | MEDIA | **ABERTO** | Config diz 120s, efetivo e 60s |
| 6 | Memoria acima do criterio do roteiro | — | **A INVESTIGAR** | 138MB contra o limite de 100MB. Medir `PrivateMemorySize64` por horas antes de concluir |

Suite: **648 testes, 0 falhas**.

### Limpeza de dados pendente (nao feita — banco e somente leitura)

A varredura de 7 dias encontrou **33 ociosidades** e **27 janelas** abertas ha mais de 30min,
resquicio do A-48 e do A-35. O fix impede novas, mas nao fecha as existentes. Decidir com
@Thor se vale um script de fechamento retroativo — enquanto elas existirem, os pilares
Atividade e Saude leem tempo que nao aconteceu.
