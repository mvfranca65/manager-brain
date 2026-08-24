# Briefing 1.5.5 — Agent Windows

Handoff para a sessao que vai implementar o backlog aberto na rodada regressiva de 2026-08-18.
Escrito para ser lido do zero, sem contexto anterior.

---

## 1. O que e isso

Rodada regressiva manual do **Agent Desktop V2** (Agent Windows, C#/.NET 8) contra **staging**.
A rodada achou 36 defeitos (A-01 a A-36 + D-01). A **1.5.4** ja fechou a maior parte e esta
**instalada e rodando** na maquina de teste. Este briefing cobre o que sobrou: **9 itens da 1.5.5**.

**Maquina de teste:** `DESKTOP-VMSM6LE` | agente **id 50** | colaborador `i888777` (usuario 3336, empresa 8)

### Repositorios

| Repo | O que e | Stack |
|---|---|---|
| `manager-srv-agent` | Agent Windows (**foco**) | .NET 8, xUnit + Moq, Inno Setup, Serilog |
| `manager-srv-admin-node` | Backend admin | NestJS + Drizzle + vitest, **pnpm** |
| `manager-srv-events-node` | Ingestao de eventos | NestJS |
| `manager-brain` | Documentacao | Markdown |

### Arquitetura em 5 linhas

- Dois servicos Windows: **`ManagerAgent`** (Sessao 0 / SYSTEM, `start=delayed-auto`) e
  **`ManagerAgentWatchdog`** (`start=auto`).
- Um **`ManagerAgent.SessionWorker`** por sessao interativa, lancado via `WTSQueryUserToken`.
- IPC por Named Pipe `ManagerAgent_S{sessionId}`.
- Buffers SQLite WAL: `buffer.db` (lado Service) e `autonomous-buffer.db` (lado worker).
- Ciclo de upload 60s, lote 100, max 10 lotes/ciclo. Checagem de update a cada 6h + uma no start.

### Modelo de eventos (paridade com o Agent Android)

O evento nasce **aberto** (`finalizadoEm` **ausente** do JSON, nao `null`) e e fechado por uma
emissao posterior com o **mesmo `iniciadoEm`**. O backend reconcilia por upsert:

- `eventos_janela`: UNIQUE `(agente_id, nome_processo, iniciado_em)`
- `eventos_ociosidade`: UNIQUE `(agente_id, iniciado_em)`
- `eventos_sessao`: **sem UNIQUE** — por isso duplicata aqui entra no banco (ver A-33)

Regra de merge: **o primeiro `finalizadoEm` nao-nulo vence**.

> Cuidado: `PipeEventBuffer` serializa com `DefaultIgnoreCondition = WhenWritingNull`.
> E isso que faz `finalizadoEm` sumir do JSON em vez de virar `null`. Nao mexer.

---

## 2. Regras que nao se negociam

1. **Nada de commit e nada de push.** O Marcos valida e commita. Quando ele autorizar: Conventional Commits.
2. **Banco e somente leitura.** SELECT apenas. Nenhuma escrita autorizada em staging.
3. **NAO rodar `dotnet test` na maquina sob teste** — ver A-32. E o item que mais custou tempo na rodada.
4. **LGPD (limites do produto, nao sugestoes):** sem keylog, sem screenshot (so no plano Plus futuro),
   sem audio/microfone/camera, sem leitura de conteudo, sem URL completa — **so o dominio raiz**.
5. **Codigo e mensagem de log em ingles. Documentacao em portugues.**
6. **Arquivos `.ps1` sao somente-ASCII.** Em-dash quebra o parser do PowerShell. Isso ja foi o
   achado A-05 e eu mesmo reintroduzi o bug durante a correcao. Conferir antes de salvar com
   `LC_ALL=C grep -nP '[^\x00-\x7F]' arquivo.ps1`
7. **Fala breve e objetiva.** Sem enrolacao, sem recapitular o que ja foi dito.

### Processo de revisao (regra do Marcos)

> "Quando o Bucky acabar cada tarefa quero que o Tony revise e de o OK, dando o ok pode seguir pra proxima."

**@Bucky implementa, @Tony revisa e da o OK, so entao vai para a proxima tarefa.** Uma por vez.
Na 1.5.4 o Tony pegou 4 defeitos reais nessa revisao — o processo funciona, nao pular.

---

## 3. Documentos a ler antes de comecar

Todos em `manager-brain/docs/tecnologia/agent-desktop/`:

| Arquivo | Para que |
|---|---|
| `ACHADOS-REGRESSIVO-2026-08-18.md` | **Fonte da verdade.** A-01..A-36 + D-01, com evidencia de cada um |
| `PLANO-CORRECOES-AGENT-WINDOWS-1.5.4.md` | Lotes L1..L8, criterios de aceite, tabela "Backlog da 1.5.5" e status de revisao |
| `PLANO-TESTES-REGRESSIVOS.md` | Plano de teste (desatualizado — e o proprio A-09) |
| `REGRAS-RELEASE.md` / `GUIA-RELEASE.md` | Como versionar e publicar |
| `PATCH-A28-PARSEVERSAO-SHURI.md` | Referencia do patch de backend que ja foi para staging |

Ler os achados **A-27, A-29 a A-36** por inteiro — sao os 9 itens deste briefing e cada um tem a
evidencia bruta (log, query, timestamp) que justifica a correcao.

---

## 4. Os 9 itens

Ordem sugerida: **A-32 primeiro** (destrava rodar testes com seguranca), depois o bloco de sessao
(A-34, A-33, A-36, que se reforcam), depois A-35, depois os dois de validacao (A-30, A-31),
depois A-29. O A-27 e do @Shuri, em paralelo.

---

### A-32 — CRITICO: `dotnet test` sobe SessionWorkers de verdade

**Arquivo:** `manager-srv-agent/tests/ManagerAgent.Service.Tests/WorkerLauncherTests.cs`

`WorkerLauncherTests` chama o lancador de verdade, que faz `WTSQueryUserToken` e sobe
`ManagerAgent.SessionWorker.exe` **da instalacao de producao**. A suite rodou ~8 vezes durante a
rodada e ficaram 8 workers vivos gravando eventos reais. Resultado: 16 LOCK / 26 UNLOCK falsos,
~17 icones na bandeja e um "sinal de regressao" que nao existia.

**Correcao:** marcar a classe com `[Trait("Category","RequiresIsolation")]` e excluir do
`dotnet test` padrao (`--filter "Category!=RequiresIsolation"` no CI e no `.runsettings`).
Idealmente tambem: apontar o lancador para um executavel dummy via injecao de caminho.

**Aceite:** `dotnet test` sem filtro nao deixa nenhum `ManagerAgent.SessionWorker` vivo depois de rodar.

**Enquanto nao estiver feito:** nao rodar a suite nesta maquina. Se rodar por engano, conferir e limpar:

```powershell
Get-Process ManagerAgent.SessionWorker | Select-Object Id, StartTime
```

Sobrevivente legitimo = o mais antigo, da sessao 1. Os logs
`session-worker-S0/S2/S3/S42/S9999/S4294967295` sao rastro dos falsos.

---

### A-34 — Dedup de LOGIN nunca funcionou (`WTSLogonTime` nao e suportado)

**Severidade:** ALTA. **Regressao de defeito ja fechado** pela spec `2026-07-09-agent-v1.4.0` (item 4).

**Arquivos:**
- `src/ManagerAgent.Shared/Runtime/WtsHelpers.cs` — `QueryLogonTimeTicks`, `QueryLogonId`
- `src/ManagerAgent.Shared/NativeMethods.cs` — `WtsInfoClass` (precisa dos structs novos)
- `src/ManagerAgent.SessionWorker/Capture/SessionMonitor.cs` — `MaybeEmitInitialLogin`

**Sintoma:** todo restart do worker (kill, crash, auto-update, fast user switch) emite um LOGIN
falso. Hoje: 35 inicializacoes = 35 LOGINs. A sessao Windows do usuario nunca foi interrompida.

**Causa raiz:** o dedup existe e esta certo — monta um `LogonId` estavel
(`SessionId:LogonTime:UserName`), compara com o persistido, so emite se divergir. Mas
`QueryLogonId` devolve `null` **sempre**, e cai no fallback conservador:

```
[WRN] Nao foi possivel resolver LogonId via WTS. Emitindo LOGIN em modo compat legado.
```

O `null` vem de `WTSQuerySessionInformation(..., WtsInfoClass.WTSLogonTime, ...)`. A classe
`WTSLogonTime` (18) **nao e suportada** por essa API — a Microsoft documenta que ela sempre
retorna FALSE com `ERROR_NOT_SUPPORTED`. Medido nesta maquina (sessao 1):

| classe | valor | ok | bytes | LastError |
|---|---|---|---|---|
| WTSUserName | 5 | True | 10 | — |
| **WTSLogonTime** | **18** | **False** | **0** | **50 = ERROR_NOT_SUPPORTED** |
| WTSSessionInfo | 24 | True | 144 | — |
| WTSSessionInfoEx | 25 | True | 160 | — |

O comentario no codigo chama o `null` de "raro". Ele e universal — o caminho feliz nunca rodou
em maquina nenhuma.

**Correcao:** trocar pela classe **25 (`WTSSessionInfoEx`)**, que devolve `WTSINFOEX` com
`WTSINFOEX_LEVEL1.LogonTime` como `LARGE_INTEGER`. Alternativa equivalente: classe 24
(`WTSSessionInfo` com `WTSINFO`).

**Ganho de tabela, usar no A-33:** `WTSINFOEX_LEVEL1` traz tambem `SessionFlags`
(`WTS_SESSIONSTATE_LOCK` / `WTS_SESSIONSTATE_UNLOCK`): estado de bloqueio autoritativo, vindo do
Windows. E exatamente o que falta no A-33. **Fazer o A-34 antes do A-33** e reaproveitar a P/Invoke.

**Aceite:**
1. O WRN "compat legado" some do log.
2. Matar o SessionWorker **nao** gera LOGIN novo em `eventos_sessao`.
3. Logoff/logon de verdade **continua** gerando LOGIN.
4. Teste unitario cobrindo `QueryLogonId != null` para a sessao corrente.

---

### A-33 — Duas fontes de estado de lockscreen que nao se falam

**Severidade:** ALTA.

**Arquivos:**
- `src/ManagerAgent.SessionWorker/Capture/LockScreenDetector.cs`
- `src/ManagerAgent.SessionWorker/Capture/SessionEventService.cs`
- `src/ManagerAgent.SessionWorker/Capture/WindowActivityService.cs`

Existem **duas** fontes independentes de estado de lock:

| Fonte | Quando dispara | Onde |
|---|---|---|
| `SystemEvents.SessionSwitch` | na hora (evento do SO) | `SessionEventService` |
| Polling WTS (`IsInputDesktopLocked`) | a cada 5s | `LockScreenDetector` |

Nao compartilham estado. O unico acoplamento e uma janela de dedup de **2 segundos**.

**Sintoma 1 — duplicata.** Um unico Win+L gerou 2 LOCK e 2 UNLOCK:

| Evento | Fonte | Horario | Distancia |
|---|---|---|---|
| LOCK | SessionSwitch | 11:44:51.354 | — |
| LOCK | polling | 11:45:08.458 | **+17,1s** |
| UNLOCK | SessionSwitch | 11:45:08.826 | — |
| UNLOCK | polling | 11:45:13.601 | **+4,8s** |

A janela de 2s nao cobre. E `OnExitLockscreen` **nao tem dedup nenhum**. Como `eventos_sessao`
nao tem UNIQUE, tudo entra no banco.

**Sintoma 2 (pior) — a tela de bloqueio virou janela ativa por 15s:**

```
11:44:52  LOCK (SessionSwitch) - fecha a janela, mas NAO chama Pause()
11:44:53  Window event opened: "Microsoft(R) Windows(R) Operating System"
                             / "Tela de Bloqueio padrao do Windows"
11:45:08  fechado - so agora o polling percebe e pausa
```

(evento `eventos_janela` id 49162, 11:44:53.032 ate 11:45:08.458)

O `SessionEventService` faz o flush no LOCK mas **nunca chama `_windowActivityService.Pause()`**.
Quem pausa e so o `LockScreenDetector`, que chegou 17s atrasado. Numa maquina bloqueada a noite
inteira, esse atraso define quanto de trabalho fantasma entra na conta do colaborador.

**Por que o polling atrasa 17s:** o `OpenInputDesktop` nao consegue abrir o input desktop enquanto
o Winlogon esta no controle. Ele so reconheceu o lock **0,37s antes do desbloqueio**. O sinal
confiavel e o `SessionSwitch`; o polling deve ser **so fallback** para auto-lock por politica.

**Correcao:** dedup **por estado**, nao por janela de tempo (qualquer valor escolhido erraria — os
17s provam).

1. `LockScreenDetector` vira **dono unico** do estado de lock.
2. `SessionSwitch` deixa de emitir por conta propria e passa a alimentar o detector via
   `NotifyLock` / `NotifyUnlock`.
3. A transicao de estado — e so ela — emite o evento, faz o flush de janela + ociosidade e chama
   `Pause()`/`Resume()`. Tudo num lugar so.
4. Usar `SessionFlags` do `WTSINFOEX_LEVEL1` (vem do A-34) como fonte do polling, no lugar do
   `OpenInputDesktop`.

**Defesa em profundidade (opcional, @Shuri):** UNIQUE `(agente_id, tipo_evento, ocorreu_em)` em
`eventos_sessao`.

**Aceite:** um ciclo Win+L e desbloqueio gera **exatamente 1 LOCK e 1 UNLOCK**, e **nenhum**
`eventos_janela` de shell/lockscreen entre eles.

---

### A-35 — Kill do worker deixa o evento de janela aberto para sempre

**Severidade:** ALTA.

**Arquivos:** `src/ManagerAgent.Service/Pipe/PipeServer.cs` (`OnWorkerDisconnected`),
`src/ManagerAgent.Service/Resilience/WorkerWatchdog.cs`

**Sintoma:** no instante do kill a janela em foco era o Gerenciador de Tarefas. Ninguem fechou o evento:

| id | processo | iniciado_em | finalizado_em |
|---|---|---|---|
| 49169 | Taskmgr | 11:50:32 | **ABERTO** (o kill foi 11:50:43) |

Para o backend, que calcula duracao por `finalizado_em - iniciado_em`, isso e um bloco de duracao
indefinida — ou atividade silenciosamente descartada, dependendo de como o relatorio trata o null.
Vale para qualquer morte nao-graciosa: kill, crash, `taskkill /F`.

**Causa raiz:** `_currentEvent` do `WindowActivityService` so existe em memoria. Os flushes
(`StopAsync`, lock, troca de janela) exigem o processo vivo, e o kill nao passa por nenhum deles.
O worker novo sobe com `_currentEvent = null` e nao tem como saber o que o anterior tinha aberto.

**Correcao:** resolver no lado que sobrevive — **o Service**. Ele ja sabe a hora exata da morte
pelo EOF do pipe:

```
11:50:43.519 [INF] PipeServer: worker disconnected (EOF). Session=1
```

E ja conhece o ultimo evento de janela aberto, porque o snapshot de 60s (A-12) o reemite. Entao:
ao tratar a morte, o Service fecha o ultimo evento conhecido com `FinalizadoEm` = instante do EOF,
motivo `WorkerDied`.

**Alternativa descartada:** persistir `_currentEvent` no `autonomous-buffer.db` a cada troca de foco.
Custa escrita em disco a cada foco novo e ainda perde o intervalo entre a ultima escrita e o kill.
O EOF da o instante exato, de graca.

**Aceite:**
1. `taskkill /F` no SessionWorker fecha o evento em curso com `finalizado_em` = hora do EOF (tolerancia 1s).
2. Nenhum `eventos_janela` com `finalizado_em is null` alem do que esta genuinamente em curso.
3. Parada graciosa continua fechando pelo caminho normal, **sem fechar duas vezes**.

---

### A-36 — LOGIN e LOGOFF nao disparam upload imediato

**Severidade:** MEDIA. **Arquivo:** `src/ManagerAgent.Service/Pipe/PipeMessageHandler.cs`,
metodo `ContainsSessionBoundary`

O A-19 fez LOCK/UNLOCK subirem na hora (1,5s a 4,0s). LOGIN e LOGOFF ficaram de fora e ainda
esperam o ciclo de 60s:

| evento | ocorreu_em | criado_em | atraso |
|---|---|---|---|
| LOGIN (id 4863) | 11:50:44.001 | 11:51:39.108 | **55,1s** |
| LOGIN (id 4864) | 11:50:50.548 | 11:51:39.108 | **48,6s** |

`ContainsSessionBoundary` so reconhece `LOCK` e `UNLOCK`.

O caso que importa e o **LOGOFF**: acontece com a maquina desligando, e 60s e mais do que o Windows
costuma conceder antes de matar os processos. LOGOFF perdido = sessao do dia sem fechamento.

**Correcao:** incluir `LOGIN` e `LOGOFF` na lista. O A-34 derruba o volume de LOGIN a quase zero,
entao o custo extra e desprezivel.

**Aceite:** LOGIN e LOGOFF com `criado_em - ocorreu_em` abaixo de 5s.

---

### A-30 e A-31 — ja corrigidos no codigo, falta validar na maquina

Nao reimplementar. Sao apenas validacao de ponta a ponta com a 1.5.4 instalada.

| # | O que era | Correcao ja aplicada | Como validar |
|---|---|---|---|
| **A-30** | Janela aberta durante ociosidade ja em curso virava bloco fantasma de 1ms | Ordem do loop no `Worker.cs`: `IdleMonitor`, `WindowActivity`, `SessionEvent`, `Heartbeat` | Ficar ocioso 2min, mexer o mouse, conferir que nao nasce `eventos_janela` de duracao ~0 |
| **A-31** | Shell do Windows sem titulo registrado como 50s de trabalho | Blocklist `ProcessosShell` + `TitulosPlaceholder` em `ManagerAgentUploadOptions`, aplicada em `WindowActivityService.IsIgnored` | Clicar na area de trabalho / menu iniciar e conferir que nao gera `eventos_janela` |

Cuidado no A-31: **nao da para bloquear o shell inteiro** — o mesmo processo serve o Explorador de
Arquivos, que e trabalho de verdade e **tem** titulo (o nome da pasta). O criterio e
`processo de shell` **E** `titulo placeholder`.

---

### A-29 — Icones fantasma na bandeja

**Severidade:** MEDIA. **NAO implementado.**

**Arquivo:** `src/ManagerAgent.Service/Update/UpdateApplier.cs`, metodo `KillSurvivorWorkers`,
e o instalador.

Workers mortos com `Kill()` deixam o icone da bandeja orfao ate o mouse passar por cima.
(Parte dos ~17 icones vistos na rodada foi do A-32, mas o defeito e real e independente.)

**Correcao:** enviar SHUTDOWN pelo pipe — que o `HandleShutdownAsync` **ja trata** — e aguardar
saida graciosa antes de partir para o `Kill`. Somar uma varredura de orfaos no startup do Service.

**Aceite:** apos um auto-update, zero icones fantasma na bandeja sem precisar passar o mouse.

---

### A-27 — `publicado_em` 3h no futuro (@Shuri, backend — nao e do Agent)

**Repo:** `manager-srv-admin-node`. Tabela `versoes_agente`, coluna `publicado_em`.

Grava hora local rotulada como UTC. Tanto a 1.5.3 quanto a 1.5.4 aparecem com **+3h**. Os relogios
estao sincronizados (verificado: `now()` do banco 13:57:26Z = maquina 13:57:26Z), entao e o
endpoint de publicacao.

Arquivos provaveis: `src/modules/release-admin/release-admin.service.ts` e `release-admin.repository.ts`.

**Correcao:** gravar UTC de verdade no endpoint **e** corrigir as linhas existentes.

---

## 5. Ja corrigido e validado em campo (nao mexer)

| Achado | O que era | Situacao |
|---|---|---|
| **A-28** | **Auto-update nunca funcionou.** `parseVersao` recebia `1.5.3.0` (4 partes), `parts.length !== 3` lancava, e o `catch { return false }` silenciava | **Corrigido dos dois lados e validado em campo.** Agent: `ObterVersaoSemVer` corta para `major.minor.build`. Backend: `parseVersao` valida `length < 3` e o `catch` agora loga WARN. O update 1.5.3 para 1.5.4 rodou de ponta a ponta em 2026-08-18, 11:24 as 11:26 |
| **A-20** | Worker morto so era percebido em 97s (criterio 15s) | **PASSOU: 0,55s e 0,80s.** Deteccao por EOF do pipe (`PipeServer.OnWorkerDisconnected` para `WorkerWatchdog.OnWorkerDisconnectedAsync`), com guard `IsSessionStillActive` para nao relancar em logoff. Exatamente 1 worker ao final |
| **A-18** | Janela nao fechava na fronteira do LOCK | **PASSOU.** Fecha no instante exato, sem bloco `unknown` de 1ms |
| **A-19** | LOCK/UNLOCK levavam 26s e 64s para subir | **PASSOU: 1,5s a 4,0s** |
| **A-12, A-14, A-16** | Ciclo de vida da janela, ruido, sobreposicao janela/ociosidade | **Confirmados no banco** apos a 1.5.4: 1 janela aberta, 0 ruido `ManagerAgent%`, 0 bandeja, 0 `unknown` |
| **A-25** | "Mojibake em UTF-8" | **FALSO ACHADO.** O hexdump mostrou `C3 A1` — UTF-8 correto. Era o terminal que lia em Latin-1. **Licao: conferir os bytes antes de abrir achado de encoding** |

---

## 6. Armadilhas conhecidas

| Armadilha | O que fazer |
|---|---|
| `dotnet test` sobe workers reais | Ver A-32. **Nao rodar a suite nesta maquina** ate isso estar feito |
| `npm ci` no `manager-srv-admin-node` | O repo usa **pnpm** (`pnpm-lock.yaml`). Use `npx --yes pnpm@9 install --frozen-lockfile` |
| `npx vitest` pega copia quebrada do cache | Use `node_modules/.bin/vitest` |
| Em-dash em `.ps1` | Quebra o parser. Somente-ASCII. Conferir com `LC_ALL=C grep -nP '[^\x00-\x7F]'` |
| Namespaces do WTS | Sao `ManagerAgent.Shared.Runtime.WtsHelpers` e `ManagerAgent.Shared.NativeMethods` |
| `Process.GetCurrentProcess()` em teste do guard | O guard chama `Kill()`, mataria o test runner. Suba um `cmd.exe /c timeout /t 60` descartavel |
| Scripts em `instalador/Pacote/` | A pasta e **apagada** no passo [2/9] do build. Colocar em `instalador/` |
| `python` | Nao disponivel nesta maquina |
| `psql` | Nao disponivel. Usar node com o driver `postgres` (secao 7) |
| Heredoc grande no bash | Quebra. Usar a ferramenta Write para documentos longos |

---

## 7. Como validar contra o banco (somente leitura)

Nao ha `psql`. O driver `postgres` esta em `manager-srv-admin-node/node_modules`. A URL de staging
esta em `manager-srv-admin-node/.env.local.bak-2026-08-12`.

```js
// q.mjs - uso: node q.mjs consulta.sql
import postgres from 'file:///C:/Users/<voce>/Documents/Manager/manager-srv-admin-node/node_modules/postgres/src/index.js'
import fs from 'fs'
const env = fs.readFileSync('.../manager-srv-admin-node/.env.local.bak-2026-08-12','utf8')
const url = env.match(/postgres:\/\/[^"'\s]+pooler\.supabase\.com[^"'\s]*/)[0]
const sql = postgres(url, { ssl: 'require', prepare: false })
console.log(JSON.stringify(await sql.unsafe(fs.readFileSync(process.argv[2],'utf8')), null, 1))
await sql.end()
```

> `criado_em` e `ocorreu_em` sao UTC. Converter sempre com `at time zone 'America/Sao_Paulo'`,
> senao os horarios saem 3h deslocados dos logs.

**Consultas uteis:**

```sql
-- Eventos de sessao recentes (A-33, A-34, A-36)
select id, tipo_evento,
       to_char(ocorreu_em at time zone 'America/Sao_Paulo','HH24:MI:SS.MS') as ocorreu,
       to_char(criado_em  at time zone 'America/Sao_Paulo','HH24:MI:SS.MS') as criado
from eventos_sessao
where agente_id = 50 and ocorreu_em >= now() - interval '30 minutes'
order by ocorreu_em;

-- Janelas orfas (A-35)
select id, nome_processo, left(titulo_janela,40),
       to_char(iniciado_em at time zone 'America/Sao_Paulo','HH24:MI:SS') as ini
from eventos_janela
where agente_id = 50 and finalizado_em is null
  and iniciado_em >= current_date
order by iniciado_em;

-- Duplicatas (deve voltar vazio)
select nome_processo, iniciado_em, count(*)
from eventos_janela where agente_id = 50
group by 1,2 having count(*) > 1;
```

**Logs:** `C:\ProgramData\ManagerAgent\logs\`

- `service-YYYYMMDD.log` — Service (pipe, watchdog, upload, update)
- `session-worker-S1-YYYYMMDD.log` — worker da sessao 1
- `watchdog-*.log`, `update-script.log`, `startup-trace.log`

---

## 8. O que ainda falta testar (depois das correcoes)

Blocos da rodada que nao foram executados. Detalhe em `ACHADOS-REGRESSIVO-2026-08-18.md`,
secao "Ainda nao testado".

| # | Bloco | Observacao |
|---|---|---|
| **A-21** | Kill do Service | **Era o proximo teste da fila.** Espera-se exatamente 1 SessionWorker apos o restart do SCM |
| P-2 | Performance estabilizada | Medir com o agent rodando ha 4h+ |
| P-3 | Scripts de diagnostico | So o health-check foi rodado |
| P-4 | Multi-sessao (2 usuarios logados) | — |
| P-5 | Named Pipe e AutonomousBuffer | Coberto so indiretamente pelos kills |
| P-6 | Logoff / suspensao / reinicio | Toca A-34 e A-36 diretamente |
| P-7 | Deteccao de reuniao (Teams/Meet/Zoom) | — |
| **P-8** | Bloco LGPD dedicado | **Obrigatorio antes do release** |
| P-9 | Backend fora do ar (5xx, nao so DNS) | So o cenario sem rede foi testado |
| P-10 | Desinstalacao e reinstalacao | Deixar por ultimo, depois do A-01 |
| P-11 | Status ATIVO/PAUSA/AUSENTE no backend | Depende do D-01 |

**L8 — Documentacao, ainda pendente:** A-09 (`PLANO-TESTES-REGRESSIVOS.md` desatualizado, @Natasha) e
A-15 (limiar de ociosidade: 60s no codigo vs "5+ minutos" na documentacao, @Steve).

---

## 9. Definicao de pronto

1. Os 9 itens implementados, cada um com **teste unitario** cobrindo a causa raiz.
2. Cada tarefa **revisada pelo @Tony** com OK explicito antes da proxima.
3. Suite verde (com o filtro do A-32 aplicado).
4. Versao elevada para **1.5.5** nos `.csproj`.
5. `ACHADOS-REGRESSIVO-2026-08-18.md` e `PLANO-CORRECOES-AGENT-WINDOWS-1.5.4.md` atualizados.
6. **Nada commitado** — o Marcos valida e commita.
