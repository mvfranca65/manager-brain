# Proposta — ciclo de vida do SO no Agent Windows (A-56, A-61, A-62)

> **De:** @Bucky | **Para:** @Tony | **Aprovador:** Marcos
> **Data:** 2026-08-21
> **Escopo:** proposta tecnica. Nenhuma linha de codigo de producao foi escrita nesta rodada.
> **Base:** leitura do repo `manager-srv-agent`, branch `staging`, arvore de trabalho da 1.5.10
> (150 arquivos alterados e nao commitados).

---

## Estado da implementacao (atualizado em 2026-08-21)

Proposta aprovada pelo Marcos. **Passos 0 a 4 implementados**, mais A-58 e A-60. Nada commitado.

| Passo | Estado | Testes |
|---|---|---|
| 0 — `PipeServer` + reativar os 3 arquivos de teste | **feito** | 753 -> 794 |
| 1 — `SessionLifecycleTracker` + janela de confirmacao | **feito** | 794 -> 810 |
| 2 — relogio que nao conta tempo dormido | **feito** | 810 -> 819 |
| 4 — rejeicao ativa / invariante do Marcos | **feito** | 819 -> 853 |
| 3 — Service emite LOGOUT + flag por LogonId | **feito** | 887 -> 935 |

Fora desta secao, na mesma leva: A-58 (ATIVO/PAUSA/AUSENTE restaurado, 853 -> 887) e A-60
(buffer por sessao + varredura de orfaos, 887 -> 908).

O passo 4 introduziu o motivo de shutdown `DUPLICATE_WORKER`
(`src/ManagerAgent.Shared/Pipe/ShutdownReasons.cs`), que exige o SessionWorker em **1.5.11 ou
superior**. Worker anterior a isso e encerrado direto, sem o pedido educado — perde o buffer
daquele orfao, uma vez, na transicao de versao. **A versao dos csproj ainda esta em 1.5.10 e
precisa subir para 1.5.11 antes do build de validacao**, senao o proprio agent novo se declara
antigo e nunca recebe o pedido.

A armadilha da secao 5.4 (worker excedente gravando LOGOUT falso) esta coberta por teste
dedicado em `WorkerShutdownTests`. Verificada por sabotagem: invertendo a regra de
`ExigeDespedidaDeSessao`, exatamente os dois testes da armadilha ficam vermelhos e os dois de
drenagem/encerramento seguem verdes.

Decisoes do @Tony incorporadas: `QueryUnbiasedInterruptTime` como correcao principal e
`OnPowerEvent` fora desta rodada (secao 2.1); janela de confirmacao em vez de "ensinar o
watchdog" (secoes 2.2/3.3); flag comparada por `logonId` e nao apagada (secao 4.3); processo
vivo nao se relanca (secao 5.5).

**Sobre o passo 3.** A @Shuri confirmou que `eventos_sessao` **nao tem UNIQUE nenhuma**. As tres
camadas da secao 4.4 entraram: emissao no fim do `HandleLogoffAsync`, marca de GOODBYE por
`(sessao, pid)`, e consulta ao proprio buffer do Service imediatamente antes de inserir. A
**brecha que sobra** — LOGOUT preso no buffer autonomo do worker com o pipe caido, que so a
UNIQUE fecha — esta registrada em teste executavel,
`LogoutDuplicadoTests.BRECHA_CONHECIDA_LOGOUT_preso_no_buffer_do_worker_nao_e_visto`, para que
seja decisao registrada e nao esquecimento.

A regra da flag foi verificada por sabotagem: trocando a comparacao de `logonId` por "flag
existe, nao emite", exatamente os dois testes que protegem o alarme legitimo ficam vermelhos.


---

## Resposta curta

Nao existe **uma** correcao que resolva os tres. Existem **duas**, e a terceira sai de graca:

1. **A autoria do fim de sessao passa do worker para o Service.** Resolve o A-62 inteiro e o
   falso SESSAO_INTERROMPIDA do A-61.
2. **O relogio do watchdog para de contar o tempo em que a maquina dormiu.** Resolve o A-56.
3. **O relancamento espera confirmar que a sessao continua viva.** Resolve o ruido do A-61.

O que une os tres nao e a causa — sao causas diferentes. O que une e a **lacuna**: o Service
nao tem nenhum modelo do que o sistema operacional esta fazendo. Ele sabe que o pipe caiu; nao
sabe se caiu porque o usuario deslogou, porque a maquina dormiu, porque esta desligando ou
porque o worker realmente morreu. Todas as tres correcoes moram no mesmo componente novo, e por
isso vale trata-las como uma entrega so.

O item do Marcos ("sempre um agent so") e **independente dos tres** e e o de maior risco. Ele
tem secao propria, porque encontrei um defeito no pipe que muda o tamanho do problema.

---

## 1. O que confirmei no codigo

| Afirmacao do refinamento | Confere? | Evidencia |
|---|---|---|
| `HandleLogoff` manda `ShutdownMessage` com `SESSION_LOGOFF`/3000ms | sim | `ManagerAgentService.cs:179-204` (metodo se chama `HandleLogoffAsync`, nao `HandleLogoff`) |
| `Worker.HandleShutdownAsync` emite LOGOUT, drena e marca a flag | sim | `Worker.cs:506-586` — LOGOUT em 519, drain em 534, flag em 546, drain forcado em 562 |
| Watchdog nao sabe nada de suspensao | sim | `grep PowerMode\|Suspend\|Resume` em `src/ManagerAgent.Service/` retorna zero linhas |
| `HeartbeatTimeout` = 90s fixos | sim | `WorkerWatchdog.cs:35` |
| O watchdog remove do registro mas nao mata o processo | sim | `WorkerWatchdog.cs:410-482` — `HandleDeadWorkerAsync` faz pipe cleanup (417), `Remove` (426) e relanca (462). Nenhum `Kill` |
| A rejeicao por PID mismatch nao encerra nada | sim | `PipeMessageHandler.cs:95-101` — loga e da `return` |
| A flag de shutdown limpo so tem um autor, o worker | sim | `grep CleanShutdown` no Service so acha o registro no DI (`Program.cs:183-187`) — **registrado e nunca consumido** |
| SESSAO_INTERROMPIDA sai quando ha LogonId anterior e a flag falta | sim | `SessaoInterrompidaDecider.cs:32-47` + `SessionMonitor.cs:148-199` |

Ou seja: o diagnostico do refinamento esta correto no essencial. O que segue e onde ele fica
curto.

---

## 2. Onde discordo do refinamento

### 2.1 `SystemEvents.PowerModeChanged` nao e o caminho certo para o Service

O refinamento sugere que "o Service pode ouvir `SystemEvents.PowerModeChanged`". Isso e o que o
`PowerModeMonitor` faz **dentro do worker** (`PowerModeMonitor.cs:73`), e la funciona porque o
worker roda em sessao interativa com bomba de mensagens (`Application.Run`, comentado em
`PowerModeMonitor.cs:26-28`). O Service roda na sessao 0, sem desktop e sem janela de topo. O
caminho suportado para servico e outro:

```
ServiceBase.CanHandlePowerEvent = true;
protected override bool OnPowerEvent(PowerBroadcastStatus status)
```

E ja existe o lugar exato para por isso: `SessionAwareServiceLifetime` deriva de
`WindowsServiceLifetime` e ja liga `CanHandleSessionChangeEvent = true` no construtor
(`SessionAwareServiceLifetime.cs:35`) e ja sobrescreve `OnSessionChange` (linha 45). A simetria
e perfeita.

**Mas mesmo isso nao basta**, e por isso nao proponho o evento como correcao principal: em
maquina com Modern Standby (praticamente todo notebook novo) o `PBT_APMSUSPEND` frequentemente
nao dispara. A correcao que nao depende de evento nenhum esta na secao 3.2.

### 2.2 O watchdog JA tem guarda de logoff — e ela falhou

O refinamento trata o relancamento durante o logoff como se o watchdog nao soubesse nada. Ele
sabe: `WorkerWatchdog.cs:248-255` recusa relancar quando a sessao nao esta mais ativa, com o
comentario certo ("Logoff tambem produz EOF, e ali o worker DEVE morrer"). A guarda existe e
nao pegou o caso medido no dia 20.

Por dois motivos, os dois no codigo:

- `IsSessionStillActive` aceita `Disconnected` como ativa (`WorkerWatchdog.cs:280`), de
  proposito (tela bloqueada / RDP solto). Uma sessao em logoff passa por `Disconnected`.
- No instante do EOF o Windows ainda nao mudou o estado da sessao. O log prova: EOF as
  23:44:11.983, notificacao WTS de logoff as 23:44:12.964. A guarda foi consultada **981ms
  antes de existir a informacao que ela precisa**.

Isso muda a correcao: nao e ensinar o watchdog, e **dar tempo a ele**. Detalhe na secao 3.3.

### 2.3 O worker ja tem um segundo caminho de LOGOUT, e ele tambem nao funciona

`SessionMonitor.OnSessionSwitch` mapeia `SessionSwitchReason.SessionLogoff` para `"LOGOUT"`
(`SessionMonitor.cs:229`). Ou seja, a alternativa que o @Tony avaliou e descartou ("o worker
assinar `SystemEvents.SessionEnding`") ja existe numa variante, desde antes.

E nao funciona pelo motivo mais simples possivel: `AddEvent` so poe o evento numa `List` em
memoria (`SessionMonitor.cs:291-302`). Quem tira dali e o `SessionEventService.HandleAsync`,
que roda no laco de captura a cada `_loopDelay` e so entao grava no buffer
(`SessionEventService.cs:51-111`). Entre a notificacao e a gravacao ha pelo menos uma volta do
laco. O processo nao vive tanto.

**Concordo com descartar a opcao 2, e com um motivo mais forte que o do refinamento:** ela ja
foi tentada e o log de 20/08 e a prova de que perde a corrida. Nao ha ajuste que a salve, porque
o problema nao e receber a notificacao — e ter tempo de persistir depois de receber.

### 2.4 A rejeicao por PID mismatch nao e passiva: ela e ativamente danosa

Esta e a discordancia que mais muda o tamanho do problema. O refinamento descreve o worker
orfao como peso morto: 136MB, um icone a mais, CPU acumulada. Pela leitura do codigo, ele faz
bem pior que isso.

`PipeServer.AcceptLoopAsync` aceita a conexao **antes** de qualquer validacao e, ao aceitar,
descarta a conexao anterior da mesma sessao:

```csharp
// PipeServer.cs:84-90
if (_connections.TryRemove(sessionId, out var stale))
    await stale.DisposeAsync();
var conn = new PipeConnection(sessionId, pipe);
_connections[sessionId] = conn;
```

Sequencia: o orfao 20420 reconecta -> o Service **derruba a conexao do worker legitimo 23836**
-> le o CONNECT do orfao -> rejeita por PID mismatch e da `return`, sem WELCOME e sem fechar
nada -> a conexao do orfao **fica registrada como a conexao daquela sessao** -> o worker
legitimo detecta EOF (`PipeClient.cs:169-176`) e reconecta -> derruba o orfao -> o orfao
reconecta em backoff -> repete.

Dois efeitos, os dois ruins:

1. **Ping-pong no pipe.** Cada reconexao do orfao deixa o worker legitimo sem canal. Enquanto
   isso ele nao entrega evento, nao recebe LOCK/UNLOCK/SHUTDOWN e nao manda PING — o que
   alimenta o timeout de heartbeat do proprio watchdog.
2. **Corrida que apaga a conexao boa do registro.** O `finally` do read loop antigo faz
   `_connections.TryRemove(conn.SessionId, out _)` (`PipeServer.cs:196`), removendo **por
   chave**. Se ele rodar depois de `_connections[sessionId] = conn` da linha 90, remove a
   conexao **nova**. A partir dai o `SendAsync` daquela sessao devolve false para sempre
   (`PipeServer.cs:225-229`) — o pipe esta fisicamente de pe e o Service nao consegue mais
   falar com o worker.

O `TryRemove(conn.SessionId, out _)` da linha 196 tem de virar a sobrecarga que compara o valor
(`TryRemove(new KeyValuePair<...>(sessionId, conn))`). E uma linha, e vale independentemente do
resto desta proposta.

**Consequencia para o achado:** o orfao de 6h nao estava so gastando RAM. Ele estava brigando
pelo canal com o worker de verdade. Confirma-se no log procurando pares repetidos de
`PipeServer: worker connected. Session=1` seguidos de `CONNECT rejected: PID mismatch` depois
das 22:47:13 do dia 20 — se estiverem la, e este ciclo.

### 2.5 Os testes que cobririam exatamente estes caminhos estao desligados

`tests/ManagerAgent.Service.Tests/ManagerAgent.Service.Tests.csproj` termina com:

```xml
<ItemGroup>
  <Compile Remove="PipeMessageHandlerTests.cs" />
  <Compile Remove="WorkerWatchdogTests.cs" />
  <Compile Remove="UploadWorkerTests.cs" />
</ItemGroup>
```

Os arquivos existem no disco e nao compilam (o `WorkerWatchdogTests.cs` ainda constroi o
watchdog com 4 argumentos; o construtor real tem 8, `WorkerWatchdog.cs:83-91`).

Traduzindo: **os 753 verdes nao dizem nada sobre rejeicao de CONNECT nem sobre deteccao de
morte de worker.** Estes sao justamente os dois arquivos que esta proposta mexe. Nao e um
detalhe de higiene — e a mesma armadilha registrada na ADR-001, em que "693 suites verdes nao
significaram nada porque os testes concordavam com o codigo errado". Aqui e pior: eles nem
concordam, estao removidos da compilacao.

Reativar os tres arquivos (adaptando ao construtor atual) e pre-requisito, nao consequencia.

### 2.6 A flag de shutdown limpo e chaveada por SessionId, e o Windows recicla SessionId

`clean-shutdown-{sessionId}.json` (`CleanShutdownFlagStore.cs:170-171`) e
`session-state-{sessionId}.json` (`SessionStateStore.cs:133-134`).

O proprio registro de testes de 20/08 documenta o caso: *"o Windows trocou o SessionId do
NoisyTech de 1 para 2 (a sessao 2 era do Marcos)"*. Nesse momento o arquivo da sessao 2
carregava estado do Marcos e passou a ser lido como estado do NoisyTech. O LOGIN saiu certo por
acaso — porque o LogonId mudou. A decisao de SESSAO_INTERROMPIDA nao tem essa sorte: ela le a
presenca de um arquivo, nao o conteudo.

Isso e uma **quarta fonte de SESSAO_INTERROMPIDA falso**, independente dos tres achados, e ela
piora se o Service passar a escrever a flag (secao 4.3). Precisa entrar junto.

---

## 3. A correcao de fundo

### 3.0 O componente

Um objeto novo no Service, `SessionLifecycleTracker`, singleton, dono de duas coisas:

- **Estado da maquina:** `Running` | `Suspending` | `ShuttingDown`
- **Estado por sessao:** `Running` | `Ending` (logoff em curso ou confirmado) + o instante do
  ultimo EOF de worker daquela sessao + o `LogonId` vigente

Ele nao decide nada sozinho. Ele e consultado por quem decide:

| Quem pergunta | O que pergunta | Hoje |
|---|---|---|
| `WorkerWatchdog.OnWorkerDisconnectedAsync` | posso relancar? | so consulta o estado WTS, e cedo demais |
| `WorkerWatchdog.IsWorkerDead` | quanto tempo mesmo passou? | `DateTimeOffset.Now`, que conta sono |
| `ManagerAgentService.HandleLogoffAsync` | quando a sessao acabou de verdade? | assume "agora" |
| Emissor da flag e do LOGOUT | esta sessao terminou de forma esperada? | ninguem pergunta |

E o mesmo padrao do `UpdateGate` (`WorkerWatchdog.cs:296`, `WorkerLauncher.cs:77`) e do
`ILinkStatus` (`WorkerWatchdog.cs:300`, `WorkerLauncher.cs:90`): um portao pequeno consultado
por todos os caminhos que nascem ou matam worker. O padrao ja existe no codigo e funciona; esta
proposta o repete para um terceiro tipo de estado.

### 3.1 Parte 1 — a autoria do fim de sessao vai para o Service (A-62)

Quem sobrevive ao logoff e o Service. Entao e nele que o fim de sessao nasce.

Em `HandleLogoffAsync` (`ManagerAgentService.cs:179`), antes do que ja existe:

1. **Emitir o LOGOUT direto no buffer**, com `ocorreuEm` = o instante do EOF daquela sessao
   (guardado pelo tracker), nao "agora". O EOF e a hora real da morte, e o watchdog ja o captura
   hoje para outro fim: `var eofAtUtc = DateTimeOffset.UtcNow;` em `WorkerWatchdog.cs:202`.
   Se nao houver EOF registrado, cai para "agora".
2. **Marcar a flag de shutdown limpo** daquela sessao, com motivo `OsLogoff`.
3. So entao seguir com o `ShutdownMessage` que ja existe — ele continua util no caso em que o
   worker AINDA esta vivo (logoff lento, `shutdown /l` sem pressa, RDP). Nesse caso o LOGOUT do
   worker e o do Service coincidem, e o backend reconcilia: mesmo `tipoEvento` e mesmo
   `ocorreuEm` produzem o mesmo `client_dedup_key`
   (`SqliteEventBuffer.cs` / `EventDedupKey.ComputeEventKey`). Vale conferir com o @Shuri se a
   dedup de `eventos_sessao` e por `(agente_id, tipo_evento, ocorreu_em)`; se for, nao ha
   duplicata. **Se nao for, esta e a unica dependencia externa da proposta** e precisa de
   resposta antes de implementar.

**O mecanismo ja existe e esta provado em campo.** O `OrphanWindowEventCloser` fabrica um evento
completo no buffer do Service sem worker nenhum vivo
(`OrphanWindowEventCloser.cs:82-90`), e o log de 20/08 registra ele funcionando as 23:44:12.005
— 970ms **antes** do Service saber do logoff. Emitir um `SessionEvent` e mais simples que o que
ele ja faz: o payload e um objeto de tres campos (`EventoSessao.cs`), e o `windowsUser` vem do
`WorkerEntry.WindowsUser` (`WorkerRegistry.cs:157-164`) ou do
`WtsSessionMonitor.GetSessionUsername` (`WtsSessionMonitor.cs:115`).

E o `ICleanShutdownFlagStore` **ja esta registrado no DI do Service e nao tem consumidor**
(`Program.cs:183-187`). O encanamento da recomendacao do @Tony ja foi feito; falta ligar.

Efeito colateral bom: o LOGOUT entra no buffer do Service, `ContainsSessionBoundary` reconhece
LOGOUT (`PipeMessageHandler.cs:187-208` + `SessionEventTypes.cs`), e o upload imediato do A-36
volta a ter efeito — desde que o mesmo gatilho seja disparado no caminho novo (hoje ele so roda
dentro de `HandleEventsAsync`; o caminho do Service precisa chamar
`RequestImmediateUpload` explicitamente).

### 3.2 Parte 2 — o relogio que nao conta tempo dormido (A-56)

`IsWorkerDead` compara `DateTimeOffset.Now - entry.LastPingAt` (`WorkerWatchdog.cs:382`).
Enquanto a maquina dorme, esse relogio anda e o ping nao chega. Nao ha bug de logica; ha
escolha de relogio errada.

O Windows tem a primitiva exata para isto: **`QueryUnbiasedInterruptTime`** (kernel32,
Windows 7+). Ela conta apenas o tempo em que a maquina esteve **acordada** — sono e hibernacao
nao entram. `Environment.TickCount64` **nao** serve: mapeia `GetTickCount64`, que inclui o sono.

A mudanca e pequena e cirurgica: `WorkerEntry.LastPingAt` ganha um par
`LastPingUnbiased` (ticks), e a comparacao de heartbeat passa a usa-lo. `LaunchedAt` idem, para
o ramo "never pinged" (`WorkerWatchdog.cs:392-397`).

Por que prefiro isso ao `OnPowerEvent`:

- **Nao depende de evento chegar.** Modern Standby, hibernacao, pausa de VM, congelamento de
  container — todos passam a ser tratados certo, sem codigo especifico para cada um.
- **Nao tem janela de corrida.** Nao existe "o evento chegou depois que o watchdog ja decidiu".
- **E testavel sem Windows.** Basta injetar o provedor de relogio; hoje `DateTimeOffset.Now`
  esta hardcoded e por isso `WorkerWatchdogTests` so consegue testar com entradas fabricadas.

O `OnPowerEvent` da secao 2.1 continua valendo, mas como **complemento opcional**, nao como a
correcao. Ele daria um log explicito ("maquina suspendeu as X, acordou as Y"), que ajuda no
diagnostico e vale pouco alem disso. Sugiro deixar para depois.

**O mesmo defeito de relogio existe no Watchdog externo** — `HeartbeatMonitor.cs:67-70` compara
`DateTime.UtcNow` com o mtime do arquivo, limite de 6min. Um notebook fechado por mais de 6
minutos cai nisso. Hoje esta contido pela regra de duas camadas (o `ScmMonitor` ve o Service
`Running` e as duas precisam concordar — foi o que o M11 mediu). **Nao proponho mexer agora**,
mas fica registrado: a correcao e a mesma e o dia em que a regra de duas camadas for relaxada,
isso vira recovery espurio.

### 3.3 Parte 3 — confirmar antes de relancar (A-61)

O EOF chega ~1s antes da notificacao WTS de logoff. A guarda existente
(`WorkerWatchdog.cs:248`) pergunta cedo demais. A correcao e esperar.

Hoje ha uma espera, `ExitConfirmationGrace = 3s` (`WorkerWatchdog.cs:81`), mas ela e **pulada
justamente no caso do logoff**: o `if (!process.HasExited)` da linha 215 e falso, porque no
logoff o Windows ja matou o processo. O caminho rapido existe para o caso raro (EOF com processo
vivo) e o caminho lento nunca roda no caso comum.

Proposta: antes de relancar, esperar uma **janela de confirmacao de fim de sessao** de ~5s e
reconsultar o tracker. Se nesse intervalo chegar `SessionLogoff` daquela sessao, nao relanca.
Os dois casos medidos cabem na janela: 981ms no A-62; 5,0s e 2,0s no A-61.

Custo: em crash real de worker, o relancamento demora 5s a mais. O plano da 1.5.4 (A-20) tinha
como criterio 15s; 5s cabem com folga.

Falta o caso do desligamento da maquina, em que pode nem haver `SessionLogoff` a tempo. Duas
redes:

- Sobrescrever `OnShutdown()` em `SessionAwareServiceLifetime` e marcar
  `MachineState = ShuttingDown` antes de chamar `base.OnShutdown()`. `WindowsServiceLifetime` ja
  liga `CanShutdown` — o Service ja recebe o aviso, so nao faz nada com ele.
- A reconciliacao (`WorkerWatchdog.cs:294-344`) tambem tem de consultar o tracker. Ela roda a
  cada 15s e relanca qualquer sessao "ativa" sem worker; num desligamento demorado ela sozinha
  faz nascer worker.

**Honestidade sobre o A-61:** quem o resolve e a janela de confirmacao, nao a deteccao de
shutdown. O `OnShutdown` chega tarde (depois dos logoffs) e serve so como segunda rede.

---

## 4. Sobre o caminho recomendado pelo @Tony no A-62

**Concordo com o caminho 1 (o Service emite o LOGOUT).** E a decisao certa, pelo motivo certo:
quem sobrevive ao logoff e o Service. Adiciono tres ressalvas que o refinamento nao cobre e que
mudam o desenho.

### 4.1 O horario do LOGOUT tem de ser o do EOF, nao o de "agora"

Se o Service emitir com `DateTimeOffset.Now`, o LOGOUT sai ~1s depois do fim real da sessao — e
depois do fechamento do evento de janela que o A-35 ja gravou com o instante do EOF. A linha do
tempo ficaria fora de ordem. O instante certo ja e capturado em `WorkerWatchdog.cs:202`; falta
guarda-lo onde `HandleLogoffAsync` alcance. O registro nao serve: `HandleDeadWorkerAsync` ja
removeu a entrada (`WorkerWatchdog.cs:426`) antes de o logoff ser processado. Por isso o
instante do EOF fica no tracker, nao no `WorkerEntry`.

### 4.2 O `ShutdownMessage` tem de continuar existindo

Nao trocar um pelo outro. Ha caminhos em que o worker esta vivo e precisa fechar os eventos em
andamento e drenar o buffer — `StopAsync` do Service (`ManagerAgentService.cs:253-257`), logoff
lento, RDP. Retirar o `ShutdownMessage` trocaria o A-62 por perda de dado. A regra e: o Service
**garante** o LOGOUT; o worker **colabora** quando da.

### 4.3 Se o Service escrever a flag, o worker pode nao conseguir mais apaga-la

Este e o risco real da recomendacao, e o refinamento nao o menciona.

`SessionMonitor.TryClearCleanShutdownFlag` (`SessionMonitor.cs:201-216`) apaga a flag no startup
do worker, e o worker roda como usuario interativo. Se a flag passar a ser criada pelo Service
(SYSTEM), o `File.Delete` do worker pode falhar por permissao — e falha **em silencio**, porque
o metodo captura e so loga warning. Flag que nao e apagada e flag que **suprime um
SESSAO_INTERROMPIDA legitimo** no proximo crash. Trocariamos falso positivo por falso negativo,
que e pior: o evento existe justamente para detectar instabilidade real.

E exatamente a familia do A-60 (arquivo criado por um dono, ilegivel/imutavel para os outros).

**Correcao proposta, que resolve isto e o problema de reciclagem de SessionId da secao 2.6 de
uma vez: parar de apagar a flag e passar a compara-la.**

O `CleanShutdownFile` ganha um campo `logonId` (o mesmo formato ja usado no dedup de LOGIN:
`"{sessionId}:{logonTimeTicks}:{userName}"`, `WtsHelpers.cs:102-111`). A regra de
`SessaoInterrompidaDecider` passa de *"a flag existe?"* para *"a flag pertence ao LogonId
anterior?"*:

```
cleanShutdown != null && cleanShutdown.LogonId == previousLogonId  ->  nao emite
```

Com isso:

- **Ninguem precisa apagar nada.** Some o problema de ACL, e o `TryClearCleanShutdownFlag` pode
  sair. Escrita so pelo Service, leitura por ambos — que e o padrao de permissao mais simples
  que existe.
- **Flag velha vira inofensiva.** Ela nao bate com o LogonId novo e e ignorada. Nao ha janela de
  "flag antiga suprime evento novo".
- **A reciclagem de SessionId deixa de importar.** O LogonId carrega o nome do usuario e o
  horario de logon.
- O `SessaoInterrompidaDecider` continua sendo funcao pura e continua testavel sem Windows
  (`SessaoInterrompidaDecider.cs`, `SessaoInterrompidaDeciderTests`).

O Service consegue calcular o LogonId: `WtsHelpers` esta em `Shared` e o Service ja o usa
(`WorkerWatchdog.cs:275`). Para nao depender de a sessao ainda existir no instante do logoff, o
tracker guarda o LogonId no lancamento do worker, que acontece no `SessionLogon`
(`ManagerAgentService.cs:126-128`).

### 4.4 O caso feio: a GOODBYE chegando depois de o Service ja ter emitido

Pergunta do @Tony, com a confirmacao da @Shuri de que `eventos_sessao` nao tem UNIQUE nenhuma.
Resposta em tres camadas, da mais barata para a mais forte.

**Camada 1 — ordem: emitir no FIM do `HandleLogoffAsync`, nao no comeco.**

O metodo ja faz, hoje, exatamente a espera de que precisamos
(`ManagerAgentService.cs:191-199`): manda o `ShutdownMessage` e aguarda ate 3s o processo sair.
Decidir depois dessa espera e de graca — nao acrescenta latencia nenhuma, porque a espera ja
existe — e resolve os dois casos do enunciado:

- worker vivo e cooperativo: ele emite o LOGOUT, manda a GOODBYE e morre dentro dos 3s. O
  Service ve a GOODBYE e nao emite.
- worker ja morto (o caso medido em 20/08): nunca ha GOODBYE. O Service emite.

**Camada 2 — a marca de GOODBYE tem de ser por PID, nao por sessao.**

`HandleGoodbye` hoje so muda o estado no registro (`PipeMessageHandler.cs:232-239`), e o
`Remove` apaga esse estado. Precisa de campo proprio no tracker, chaveado por
`(sessionId, pid)`. Sem o PID, a GOODBYE de um worker anterior — o que foi substituido depois
de um update, por exemplo — suprimiria o LOGOUT do worker atual, e voltariamos ao A-62 por
outro caminho.

**Camada 3 — o que fecha a corrida de verdade: os dois LOGOUT passam pelo mesmo buffer.**

Este e o ponto que muda a resposta. O LOGOUT do worker nao vai direto para o backend: ele chega
ao Service pelo pipe e e gravado no `ServiceSqliteEventBuffer`
(`PipeMessageHandler.HandleEventsAsync` -> `InsertEventsAsync`). O LOGOUT que o Service emitir
vai para **o mesmo arquivo SQLite, no mesmo processo**.

Ou seja: nao precisamos coordenar dois emissores por ordem de chegada. Basta o Service, no
instante em que for inserir o seu, perguntar ao proprio buffer se ja existe um LOGOUT daquela
sessao nos ultimos segundos. `FindLastEventAsync(sessionId, tipoEvento)` ja existe e ja e usado
pelo `OrphanWindowEventCloser` (`OrphanWindowEventCloser.cs:55`) — falta uma variante que olhe
o `tipoEvento` de dentro do `dados`, que e onde o LOGOUT mora.

Isso e uma verificacao local, sincrona, sem rede e sem depender de quem chegou primeiro.
Fecha a corrida independentemente da ordem.

**O que continua em aberto, e nao e meu.** Se o pipe estiver caido no momento do logoff, o
LOGOUT do worker fica no buffer autonomo dele e pode subir muito depois — eventualmente so no
proximo boot. Nesse caso a camada 3 nao ve nada, o Service emite, e mais tarde chega a copia do
worker. Duas linhas.

Nenhuma verificacao do lado do agent fecha essa janela: os dois eventos nascem em processos
diferentes, em momentos diferentes, e um deles pode atravessar um reboot. **So a UNIQUE em
`eventos_sessao` fecha** — que e justamente o que o @Tony ja esta tratando com a @Shuri. As tres
camadas acima tornam o caso raro; a UNIQUE torna impossivel.

Recomendo implementar as tres camadas no passo 3 e **nao** esperar pela UNIQUE para liberar o
passo — mas registrar que, sem ela, sobra essa janela estreita.

---

## 5. "Eliminar e sempre ter apenas um agent rodando"

Este e o ponto mais delicado. Ele nao e um dos tres achados: e uma **invariante** que o sistema
hoje nao tem, e nenhuma das correcoes acima a estabelece sozinha.

### 5.1 O enunciado

*No maximo um processo `ManagerAgent.SessionWorker` vivo por sessao Windows, e o Service sabe
qual e.*

Nao basta o Service recusar o excedente. Recusar sem encerrar produz exatamente o que o Marcos
viu: dois icones. Pior, pela secao 2.4, produz disputa pelo pipe.

### 5.2 Os quatro caminhos que quebram a invariante hoje

| Caminho | Estado |
|---|---|
| Suspensao: worker declarado morto acorda e o novo ja nasceu | comprovado (A-56) |
| Revogacao seguida de re-vinculacao | previsto por mim no review da ADR-001, nao reproduzido |
| Reconciliacao (`WorkerWatchdog.cs:294`) lancando em sessao que ja tem worker fora do registro | contido no teste com `RequiresIsolation` (A-32), nao no produto |
| Plano C (Task Scheduler) lancando na sessao errada | ja tratado — `WorkerLauncher.cs:346-347` mata o intruso |

Note que o Plano C **ja faz** o que estamos pedindo aos outros caminhos: compara antes/depois e
mata o que sobrou (`KillStrayWorker`, `WorkerLauncher.cs:374-387`). E o `WorkerRegistry` ja mata
um `Launching` travado antes de substituir (`WorkerRegistry.cs:97`). O padrao existe em dois
lugares; falta nos outros dois.

### 5.3 A rejeicao vira encerramento — com uma regra de decisao explicita

Matar quem chega e perigoso: pode ser o worker certo com o registro errado (Service reiniciado,
orfao adotado em `AdoptOrphanWorkersAsync`, `ManagerAgentService.cs:210-244`). A regra tem de
olhar os dois lados antes de decidir quem sobra.

Em `HandleConnectAsync`, no lugar do `return` da linha 100:

```
CONNECT chega com PID X, registro espera PID Y, X != Y:

  Y esta vivo (Process.HasExited == false)?
     SIM -> X e o excedente. Encerrar X.
     NAO -> Y e fantasma. Adotar X: atualizar o registro, mandar WELCOME.

  Nao ha Y (registro vazio para a sessao)?
     -> Adotar X. E o caso do Service que reiniciou.

  Nao consigo ler o estado de Y?
     -> Adotar X e registrar warning. Duvida nao mata processo.
        (mesmo criterio conservador de IsWorkerDead, WorkerWatchdog.cs:372-377,
        e de IsSessionStillActive, WorkerWatchdog.cs:269-287)
```

O ramo "adotar" fecha um buraco que ninguem tinha visto: hoje, se o registro estiver com um PID
morto, **nenhum** worker consegue conectar naquela sessao, porque o mismatch rejeita todos.

### 5.4 Como encerrar o excedente, sem estragar o dado dele

Matar direto perde os eventos que ele capturou enquanto estava orfao (no A-56, seis horas
deles). O excedente esta **conectado no pipe naquele exato momento** — da para pedir educado.

Sequencia proposta:

1. Responder no lugar do WELCOME uma `ShutdownMessage` com `Reason = "DUPLICATE_WORKER"`.
2. Aguardar o grace (3s).
3. Se ainda vivo, `Process.Kill()`.

**E aqui esta a armadilha que precisa de codigo novo do lado do worker:** hoje
`HandleShutdownAsync` **sempre** emite LOGOUT e **sempre** marca a flag de shutdown limpo
(`Worker.cs:519` e `Worker.cs:546`). Se o excedente receber a mensagem como esta, ele vai:

- gravar um **LOGOUT falso** — a pessoa continua trabalhando, e o relatorio vai registrar fim de
  expediente;
- gravar a **flag de shutdown limpo** de uma sessao que nao terminou — e o proximo
  SESSAO_INTERROMPIDA legitimo daquela sessao sera suprimido.

Ou seja: a correcao ingenua do A-56 introduz um novo defeito na area do A-62. Por isso o
`Reason` precisa ser tratado explicitamente em `HandleShutdownAsync`: com
`DUPLICATE_WORKER`, pular os passos 1 e 3 (nada de LOGOUT, nada de flag), manter o passo 4
(drenar o buffer) e sair. Vai bem como um `switch` no motivo, no topo do metodo.

Compatibilidade de versao: durante um update os dois lados podem estar em versoes diferentes.
Um worker antigo recebendo `DUPLICATE_WORKER` cairia no comportamento atual — LOGOUT falso.
Defesa: o Service so manda a mensagem se `connect.WorkerVersion` (ja vem no CONNECT,
`PipeMessageHandler.cs:90-91`) for >= a versao que entende o motivo; abaixo disso, `Kill()` seco.
Perde o buffer daquele orfao, uma vez, na transicao. Aceito.

### 5.5 Matar antes de relancar

Sugestao 1 do refinamento, com um ajuste. Antes de `HandleDeadWorkerAsync` relancar
(`WorkerWatchdog.cs:458-481`), o processo tem de acabar. Mas nao com `Kill()` cego:

- Se `Process.HasExited` -> nada a fazer.
- Se vivo (falso positivo de heartbeat, que e o caso do A-56) -> **nao relancar**. Se o processo
  esta vivo, nao ha o que recuperar. Zerar o relogio de heartbeat dele e esperar o proximo
  ciclo. Se em dois ciclos ele nao voltar a pingar, ai sim encerrar e relancar.

Isso e mais conservador que a proposta do refinamento e resolve o A-56 pela raiz mesmo se a
correcao de relogio (3.2) falhar por algum motivo: o worker 20420 estava vivo, entao nada seria
lancado, entao o 23836 nunca teria nascido.

---

## 6. Riscos

Em ordem de gravidade.

**R1 — LOGOUT duplicado no banco.** O Service passa a emitir LOGOUT e o worker continua
emitindo quando da tempo. Se a dedup de `eventos_sessao` no `srv-events-node` nao for por
`(agente, tipo, ocorreu_em)`, toda parada graciosa gera duas linhas — e o A-57 mostra que
`eventos_reuniao` ficou de fora de uma correcao dessas antes. **Bloqueia a implementacao ate o
@Shuri confirmar.** Contencao se a resposta for negativa: o Service so emite quando nao houve
`GoodbyeMessage` daquela sessao (o handler ja existe, `PipeMessageHandler.cs:232-239`).

**R2 — LOGOUT falso pelo caminho do worker duplicado.** Descrito em 5.4. Se o
`DUPLICATE_WORKER` for implementado sem o tratamento no `HandleShutdownAsync`, trocamos "sem
LOGOUT nenhum" (A-62) por "LOGOUT no meio do expediente", que e pior para o relatorio. Precisa
de teste dedicado, nao de revisao visual.

**R3 — Flag de shutdown limpo que nao morre.** Se a ideia de comparar por LogonId (4.3) for
descartada e a flag continuar sendo apagada pelo worker, o Service escrevendo como SYSTEM cria
falha silenciosa de permissao e o SESSAO_INTERROMPIDA para de funcionar de vez. **Ou entra a
comparacao por LogonId, ou entra ACL explicita no arquivo** (ha precedente, o `HeartbeatWriter`
aplica ACL propria). Nao ha terceira opcao segura.

**R4 — Encerramento do worker errado.** Se a regra de decisao de 5.3 for simplificada para
"rejeitou, mata", uma falha transitoria de leitura de processo mata o worker bom e a sessao
fica sem captura ate a reconciliacao. Por isso o ramo "duvida -> adota, nao mata".

**R5 — 5 segundos a mais para recuperar de crash real.** Custo direto e assumido da janela de
confirmacao (3.3). Mensuravel, pequeno, e menor que o criterio de 15s do plano da 1.5.4.

**R6 — `QueryUnbiasedInterruptTime` em ambiente virtualizado.** Alguns hipervisores nao propagam
o unbiased time corretamente. Consequencia de falha: volta ao comportamento de hoje (conta o
sono). Nao regride nada. Contencao: se o valor vier zerado ou nao monotonico, cair para
`DateTimeOffset.Now` com um warning.

**R7 — A corrida do `_connections` (2.4) fica sem correcao.** Se ela ficar, o resto da secao 5
nao se sustenta: o Service pode mandar o `DUPLICATE_WORKER` e a mensagem ir para a conexao
errada, ou nao ir para nenhuma. **A linha 196 do `PipeServer` tem de ser corrigida primeiro.**

---

## 7. Testes

### 7.1 As quatro suites, e quem e afetada

| Suite | Testes | Impacto |
|---|---|---|
| `ManagerAgent.Service.Tests` | 396 | **Alto.** Watchdog, registry, pipe handler, ciclo de vida — quase toda a mudanca |
| `ManagerAgent.SessionWorker.Tests` | 224 | **Medio.** `HandleShutdownAsync` (motivo novo), `SessionMonitor` (decisao por LogonId) |
| `ManagerAgent.Watchdog.Tests` | 31 | **Nenhum** nesta rodada — nao proponho mexer no `HeartbeatMonitor` |
| `ManagerAgent.Tray.Tests` | 14 | **Nenhum** |

(A soma da 665 `[Fact]`/`[Theory]` declarados; os 753 executados incluem os casos de `[Theory]`
com varios `InlineData`.)

### 7.2 Pre-requisito: reativar os tres arquivos excluidos

`PipeMessageHandlerTests.cs`, `WorkerWatchdogTests.cs` e `UploadWorkerTests.cs` saem do
`<Compile Remove>` e sao adaptados aos construtores atuais. Sem isso, mexer em
`HandleConnectAsync` e em `IsWorkerDead` e mexer no escuro. Isso vai **subir** a contagem de
testes e pode revelar quebras pre-existentes — o que e o objetivo.

### 7.3 Testes novos, por achado

**A-62**
- Service recebe `SessionLogoff` com o worker ja morto -> emite exatamente um LOGOUT.
- O `ocorreuEm` do LOGOUT e o instante do EOF, nao o do processamento do logoff.
- Depois do logoff, a flag da sessao existe e carrega o LogonId correto.
- No proximo startup com esse LogonId como anterior, `SessaoInterrompidaDecider` nao emite.
- Flag de outro LogonId presente -> **emite** (o caso que a correcao nao pode quebrar).

**A-56**
- `IsWorkerDead` com relogio injetado: 200s de wall-clock e 10s de tempo acordado -> **vivo**.
- Worker declarado morto que volta a conectar antes do relancamento -> nenhum processo novo.
- Reconexao com PID mismatch e PID esperado vivo -> `ShutdownMessage(DUPLICATE_WORKER)` enviada.
- Reconexao com PID mismatch e PID esperado morto -> **adota**, manda WELCOME, nao encerra.
- Reconexao com PID mismatch e estado do processo ilegivel -> adota, nao encerra.
- `HandleShutdownAsync` com `Reason = "DUPLICATE_WORKER"` -> **nao** emite LOGOUT, **nao**
  marca flag, **drena** o buffer.

**A-61**
- EOF seguido de `SessionLogoff` dentro da janela -> nenhum relancamento.
- EOF sem `SessionLogoff` dentro da janela -> relanca (nao pode virar regressao do A-20).
- `MachineState = ShuttingDown` -> nem o EOF nem a reconciliacao lancam.

**Pipe (secao 2.4)**
- Read loop antigo terminando depois de uma conexao nova ser registrada -> a conexao nova
  **permanece** em `_connections` e o `SendAsync` continua funcionando.

### 7.4-bis O que falta provar na maquina (2026-08-21)

Os passos 0, 1, 2 e 4 estao verdes em 853 testes, e **nenhum deles prova comportamento de
suspensao real**. A lista abaixo esta na ordem em que deve ser executada — cada item depende do
anterior estar limpo. Antes de tudo: subir a versao dos csproj para 1.5.11 e gerar o pacote.

1. **Reboot limpo.** Continua saindo **um** SESSAO_INTERROMPIDA legitimo. E a regressao mais
   provavel de toda esta leva: as correcoes existem para calar o evento em situacao normal, e
   ele nao pode calar quando a situacao e anormal de verdade.
2. **`taskkill /F` no SessionWorker.** SESSAO_INTERROMPIDA continua saindo, e o worker volta em
   ate ~20s (a janela de confirmacao de 5s entrou no caminho).
3. **Suspender 3 minutos e acordar.** Um unico processo `ManagerAgent.SessionWorker`, nenhum
   SESSAO_INTERROMPIDA, nenhum segundo icone na bandeja. E o A-56 na origem.
4. **Suspender 30 minutos e acordar.** Passa do limite de 6min do Watchdog externo. Mesma
   verificacao, mais conferir no log que o Watchdog nao acionou recovery do Service.
5. **Logoff e novo login.** Com o passo 3 pronto, este item passou a ter o que verificar de
   verdade, e sao quatro coisas:
   - sai **exatamente um** LOGOUT, e o horario dele e o do logoff (o instante do EOF), nao o do
     processamento — tem de vir **antes** do fechamento do evento de janela, nao depois;
   - o login seguinte **nao** emite SESSAO_INTERROMPIDA;
   - o LOGOUT chega ao banco em segundos, sem esperar o proximo boot (o A-36 volta a valer);
   - nenhum "attempting re-launch" no log entre o EOF e o logoff (o A-61).

   Duas variacoes valem a pena, porque exercitam os dois emissores: **logoff normal** (o Windows
   mata o worker antes — quem emite e o Service) e **parar o servico com o usuario logado** (o
   worker esta vivo e se despede — quem emite e ele). Nos dois casos, **uma** linha, nunca duas.
6. **Desligar e ligar a maquina.** Nenhum worker nascendo durante o desligamento.
7. **Dois usuarios (troca rapida).** O logoff de um nao pode deixar o outro sem worker — e o
   caso da marca por SessionId reciclado.

Os itens 3, 4 e 7 sao os unicos que podem reprovar a correcao de forma silenciosa: o sintoma e
"faltou captura", nao "apareceu erro". Vale conferir no banco, nao so no log.

### 7.4 Na maquina

Os testes unitarios nao provam nada sobre suspensao real. Precisa de:

- Suspender 3 minutos, acordar: um unico processo worker, nenhum SESSAO_INTERROMPIDA.
- Suspender **30 minutos** (passa do limite de 6min do Watchdog externo): mesma verificacao,
  mais conferir que o Watchdog nao acionou recovery.
- Logoff e novo login: um LOGOUT com o horario do logoff, nenhum SESSAO_INTERROMPIDA.
- Reboot: continua havendo **um** SESSAO_INTERROMPIDA legitimo (esta e a regressao mais
  provavel de tudo isto — a correcao nao pode calar o evento no caso em que ele esta certo).
- Matar o worker com `taskkill /F`: SESSAO_INTERROMPIDA continua saindo.

---

## 8. Ordem sugerida

Cada passo termina verde e e reportado antes do seguinte.

| # | O que | Por que primeiro |
|---|---|---|
| 0 | Corrigir `PipeServer.cs:196` (remocao condicional) + reativar os 3 arquivos de teste | Sem isso o resto e cego |
| 1 | `SessionLifecycleTracker` + janela de confirmacao antes de relancar | Fecha o A-61, e a base dos outros |
| 2 | Relogio unbiased no heartbeat | Fecha o falso positivo do A-56 na origem |
| 3 | Service emite LOGOUT + marca flag; flag comparada por LogonId | Fecha o A-62 e o SESSAO_INTERROMPIDA falso |
| 4 | Rejeicao ativa + `DUPLICATE_WORKER` + nao relancar processo vivo | Fecha a invariante do Marcos |

O passo 3 foi implementado com a confirmacao da @Shuri de que `eventos_sessao` nao tem UNIQUE:
as tres camadas da secao 4.4 entraram, e a brecha que sobra esta registrada em teste executavel
(`LogoutDuplicadoTests.BRECHA_CONHECIDA_...`). Os passos 0, 1,
2 e 4 nao dependem de ninguem.

---

## 9. O que NAO entra nesta correcao

- **A-58 e A-60** — independentes, seguem na fila do refinamento.
- **`HeartbeatMonitor` do Watchdog externo** — mesmo defeito de relogio, hoje contido pela regra
  de duas camadas. Registrado na secao 3.2 para quando essa regra for revista.
- **`OnPowerEvent` no Service** — util para diagnostico, nao necessario para a correcao.
- **Sequestro do `windows_username`** (A-59) — decidido, fica como esta.
