> **DATA:** 2026-08-18 | **VERSAO TESTADA:** Agent Windows 1.5.3 | **AMBIENTE:** staging
> **STATUS:** rodada encerrada em 2026-08-18 — 25 achados + 1 decisao. Correcoes na 1.5.4 (ver `PLANO-CORRECOES-AGENT-WINDOWS-1.5.4.md`)

# Achados do Regressivo — Agent Windows 1.5.3

Maquina: `DESKTOP-VMSM6LE` | Agente na base: **id 50** | Colaborador: `i888777` (usuario 3336, empresa 8)

---

## A-01 — Desvinculacao no uninstall nunca funcionou — `desvincular.ps1` nao existe

**Severidade:** ALTA | **Dono:** @Bucky | **Review:** @Tony

O `ManagerAgent-v2.iss` chama o script como PRIMEIRA acao do `[UninstallRun]`:

```
Filename: "powershell.exe"; Parameters: "-ExecutionPolicy Bypass -File ""{app}\scripts\desvincular.ps1"""; Flags: runhidden waituntilterminated; RunOnceId: "Desvincular"
```

Mas o arquivo **nunca existiu no repositorio** — nenhum commit em todo o historico do git tocou nele.
O `scripts/build/build-pacote-v2.ps1` procura em `instalador\Pacote\desvincular.ps1` e `instalador\desvincular.ps1`,
nao acha em nenhum dos dois e **segue sem erro** (nao ha `else`).

**Efeito:** no uninstall o PowerShell falha silenciosamente (`runhidden`) e o dispositivo fica vinculado para sempre no backend.
**Evidencia:** agente **id 48** (1.5.1, mesma maquina, desinstalado antes da 1.5.3) esta com `desvinculado_em = NULL` na base.

**Sao 2 defeitos:**
1. Script ausente
2. Build silencioso — deveria abortar quando arquivo obrigatorio do pacote nao e encontrado

**Correcao proposta:**
- Escrever `instalador/Pacote/desvincular.ps1`: le `config.json`, decripta o Device JWT (DPAPI),
  chama `POST /api/agente/dispositivos/desvincular` (autenticado `@RequireDevice`, ja existe no srv-admin-node),
  best-effort com timeout curto (nao pode travar o uninstall), loga o resultado.
- Adicionar guard no `build-pacote-v2.ps1`: arquivo obrigatorio ausente = build falha.
- Depois: limpar na base os agentes orfaos ja desinstalados (ex.: id 48) — tarefa @Thor/@Vision.

---

## A-02 — `usuario_sessao` grava a conta que elevou o instalador, nao o usuario da maquina

**Severidade:** MEDIA | **Dono:** @Bucky | **Review:** @Tony

Na base, o agente id 50 gravou `usuario_sessao = "Raquel"`.
- Usuario real desta maquina: `NoisyTech`
- Identificador informado na instalacao: `i888777` (usuario 3336 = "Teste Colaborador")
- `Raquel` e outra conta local do Windows (dono registrado do SO)

**Causa provavel:** `ManagerAgent.Configurator/Program.cs:162` usa `Environment.UserName`. O Configurator roda
elevado durante a instalacao, entao pega a conta usada no UAC, nao a sessao do colaborador.
Comparar com `ManagerAgent.Service/Linking/AgentLinkService.cs:162`, que envia `"SYSTEM"` fixo, e com
`ManagerAgent.Tray/Linking/AgentLinkService.cs:217`, que usa `Environment.UserName` no contexto certo.

**Efeito:** campo de diagnostico errado; em maquina compartilhada aponta a pessoa errada.
**Correcao proposta:** o `usuario_sessao` deve vir do SessionWorker (sessao interativa real), nao do Configurator elevado.

---

## A-03 — `hardware_fingerprint` chega nulo

**Severidade:** MEDIA | **Dono:** @Bucky

Feature entrou na v1.4.1 (`e05ad5f`, Eixo 5) e a coluna existe em `agentes`, mas o agente 1.5.3 registrou
`hardware_fingerprint = NULL` e `hardware_fingerprint_atualizado_em = NULL`.

**CAUSA RAIZ (encontrada em 2026-08-18):** o campo foi adicionado **so** ao payload do
`ManagerAgent.Service/Linking/AgentLinkService.cs`. Mas quem faz a vinculacao inicial e o
`ManagerAgent.Configurator`, durante a instalacao — e o payload dele (`Program.cs`, `LinkDeviceAsync`)
nunca carregou `hardwareFingerprint`. O backend, portanto, nunca recebeu o valor: o Service so
re-vincula se o agente perder o token, o que nao aconteceu.

Sem o fingerprint o backend nao consegue deduplicar por identidade de hardware — dedup que deveria
sobreviver a rename da maquina, troca de IP e reinstalacao do agent.

**Correcao (1.5.4):** o Configurator passa a calcular e enviar o fingerprint, pelo mesmo
`HardwareFingerprintProviderFactory`. Best-effort: falha de WMI nao impede a vinculacao.

---

## A-04 — Janela "Sobre" nao mostra o `instalacaoId`

**Severidade:** BAIXA | **Dono:** @Bucky

O plano exige (R2.6.3). Hoje a janela mostra versao, usuario e status.
Sem o `instalacaoId` na tela, o suporte nao casa a maquina com o registro do backend pelo print do colaborador.
Tambem vale expor o identificador do colaborador (`i888777`), nao a conta Windows.

---

## A-05 — Em-dash em script PowerShell gerado (quebra 2 testes de guarda)

**Severidade:** BAIXA | **Dono:** @Bucky

`src/ManagerAgent.Service/Update/UpdateApplier.cs:792` — comentario do PS1 gerado contem `—` (U+2014).
Introduzido em `f774606`. Quebra `GeneratePowerShellScript_ContainsOnlyAsciiCharacters` e `..._DoesNotContainEmDash`.
Esta em linha de comentario, entao nao quebra execucao — mas viola a regra de PS1 somente-ASCII.
**Correcao:** trocar por hifen simples.

---

## A-06 — 4 falhas unitarias em investigacao (teste velho x regressao)

**Severidade:** A DEFINIR | **Dono:** @Bucky

Suite em `staging`: 506 passam, 6 falham. Alem das 2 do A-05:

| Teste | Sintoma |
|---|---|
| `AgentLinkServiceTests.EnsureLinkedAsync_ReturnsLinked_OnFullSuccess` | `IConfigManager.Salvar()` chamado 2x, esperado 1x |
| `AgentLinkServiceTests.EnsureLinkedAsync_ReturnsLinked_WhenLinkResponseHasNullTokens` | idem |
| `AgentLinkServiceTests.EnsureLinkedAsync_ReturnsConfigIncomplete_WhenInstallIdMissing` | retorna `Retry`, esperado `ConfigIncomplete` |
| `UpdateCheckerWorkerTests.ExecuteAsync_PsRunUpdateRecente_NaoDeleta` | `run-update.ps1` recente sendo deletado (spec 1.3.10 item 5) |

Hipotese do Marcos: testes desatualizados apos `592bd33` / `f119d0c`. Confirmar nos passos de vinculacao e auto-update.

---

## A-07 — Consumo de memoria acima do criterio (a confirmar)

**Severidade:** A CONFIRMAR | **Dono:** @Bucky

Medicao logo apos a instalacao: Service 139 MB, SessionWorker 164 MB, Watchdog 81 MB.
Criterio do plano: < 100 MB por processo. **Remedir com o agent estabilizado** antes de tratar como falha.

---

## A-08 — `eventos_transicao_status` sem registros novos desde 16/08

**Severidade:** A CONFIRMAR | **Dono:** @Bucky

Todas as outras tabelas de evento receberam dados hoje; essa parou em 16/08 20:24.
O agente 50 ainda nao gerou nenhuma transicao. Validar no teste de status ATIVO/PAUSA/AUSENTE.

---

## A-09 — Plano de testes regressivos desatualizado (13 correcoes)

**Severidade:** MEDIA | **Dono:** @Natasha

`PLANO-TESTES-REGRESSIVOS.md` v2.0.0 e de 2026-06-11 e nao cobre 8 meses de mudancas.
Lista completa (C1–C13) no fim do `ROTEIRO-REGRESSIVO-2026-08-18.md`. Resumo do que falta:
servico `ManagerAgentWatchdog`, vinculacao/Device JWT, eventos v1.4.x, `dispositivoTipo`, `menuVisivel`,
thresholds ATIVO/PAUSA/AUSENTE, rollback BAA, bloco LGPD dedicado, `desvincular.ps1`.
Correcoes factuais: sao 6 scripts e nao 5, recovery e 5s/10s/30s e nao 1s/5s/30s,
`monitorar-performance.ps1` nao existe mais no pacote.

---

## A-10 — `health-check.ps1` sem elevacao produz 3 falsos negativos

**Severidade:** ALTA (engana o suporte) | **Dono:** @Bucky | **Review:** @Tony

O script le `C:\ProgramData\ManagerAgent\config.json`, que tem ACL restrita (protecao correta do agent).
Rodando sem elevacao, o `Get-Content` lanca `UnauthorizedAccessException` — o erro **nao e tratado** e o script
conclui coisas erradas:

| Passo | Reportou | Realidade |
|---|---|---|
| [4/10] Configuracao | "incompleta: baseUrlAdmin, baseUrlEvents, chaveAtivacaoEmpresa, identificadorColaborador" | config completa; agent vinculado e enviando |
| [7/10] API Admin | "inacessivel — verificar internet/firewall" | acessivel; heartbeat as 11:31 |
| [8/10] API Events | "inacessivel" | acessivel; eventos chegando na base |

Score exibido: 60% (ATENCAO). Score real, descontados os falsos negativos: 100%.
Efeito pratico: manda o suporte cacar problema de firewall que nao existe.

**Decisao do Marcos (2026-08-18):** o script deve **solicitar elevacao antes de rodar** — auto-elevar via UAC
(relaunch com `Start-Process -Verb RunAs`) em vez de so avisar. Aplicar o mesmo padrao aos demais scripts
que leem `config.json` (`coletar-diagnostico.ps1`, `test-vinculacao.ps1`).

Complemento: tratar `UnauthorizedAccessException` explicitamente — "sem permissao para ler" nunca pode
virar "campo ausente" ou "API fora do ar".

---

## A-11 — `health-check.ps1` procura o log do SessionWorker no arquivo errado

**Severidade:** MEDIA | **Dono:** @Bucky

Linha 225 do script: procura entradas com a tag `[SessionWorker]` dentro de `service-*.log`
(comentario no codigo: "V2: SessionWorker escreve no mesmo service-*.log").

Mas o worker tem arquivo proprio: **`session-worker-S1-20260818.log`** (23 KB, escrito em tempo real).
Resultado: `[AVISO] Nenhuma entrada do SessionWorker no log` + issue "workers podem nao estar iniciando",
quando o worker esta rodando e logando normal.

Arquivos reais em `C:\ProgramData\ManagerAgent\logs\`:
`service-YYYYMMDD.log` - `session-worker-S{sessionId}-YYYYMMDD.log` - `watchdog-YYYYMMDD.log` - `startup-trace.log`

**Verificar o mesmo padrao em** `monitorar-logs.ps1` e `coletar-diagnostico.ps1` — o plano de testes tambem
cita `worker-YYYYMMDD.log`, nome que nao existe (soma ao A-09).

**Bonus:** o health-check nao verifica o servico `ManagerAgentWatchdog` nem o `watchdog-*.log`, mesmo existindo.

---

## A-12 — Janela em uso nunca e gravada — so entra na base depois da troca

**Severidade:** ALTA (afeta calculo de horas e foco) | **Dono:** @Bucky | **Review:** @Tony

O agent so persiste o evento de janela **depois** que o usuario troca de janela — o registro nasce ja fechado,
com `iniciado_em` e `finalizado_em` preenchidos. Nao existe linha com `finalizado_em = NULL` representando
a janela corrente.

**Evidencia (2026-08-18):** ultimo evento do agente 50 terminou 11:46:50; as 11:49:46, com o VS Code em foco
havia ~3 min, `select count(*) from eventos_janela where agente_id=50 and finalizado_em is null` = **0**.

**Efeito:** quem passa horas no mesmo aplicativo nao gera evento nenhum durante esse periodo. Atividade,
foco e fragmentacao ficam subestimados; o portal nao mostra "o que a pessoa esta fazendo agora".

**Quebra de paridade:** o Agent Android (agente 49, v1.0.15) grava normalmente — as 12 linhas abertas da tabela
sao todas dele. Android faz certo, Windows nao.

**Correcao proposta:** enviar a janela corrente como registro aberto (`finalizado_em = NULL`) e fecha-lo na
troca, ou emitir keep-alive periodico da janela ativa. Definir com @Tony qual das duas, considerando o que o
srv-events espera e o que o Android ja faz (contrato tem de ser o mesmo).

---

## A-13 — `status_usuario` chega NULL em todos os eventos de janela

**Severidade:** ALTA | **Dono:** @Bucky

Todos os eventos do agente 50 tem `status_usuario = NULL`. Somado ao A-08
(`eventos_transicao_status` sem registro novo desde 16/08), indica que a maquina de estados
ATIVO/PAUSA/AUSENTE **nao esta operando** nesta versao.

Confirmar no teste de ociosidade e no de transicao de status. Se confirmado, o PR4 de thresholds
(5min PAUSA / 15min AUSENTE) nao esta em producao de fato.

---

## A-14 — Agent captura a si mesmo e a bandeja do sistema como janela ativa

**Severidade:** BAIXA (polui metricas) | **Dono:** @Bucky

Aparecem como janela ativa, em blocos de 1-2 segundos:
- `ManagerAgent SessionWorker` / titulo "sem titulo"
- `Sistema operacional Microsoft® Windows®` / titulo "Janela de estouro da bandeja do sistema."

Sao ruido: inflam a contagem de trocas de janela (pilar Fragmentacao) e sujam o historico do colaborador.
**Correcao proposta:** blocklist de processos/janelas do proprio agent e de shell do Windows
(bandeja, alt-tab, menu iniciar), com duracao minima para registrar um bloco.

---

## D-01 — DECISAO: maquina de estados sai do Agent e vai para o backend

**Decidido por:** Marcos, 2026-08-18 | **Desenho:** @Tony | **Candidato a ADR**

`eventos_transicao_status` foi um erro de desenho. O Agent **nao** classifica status. Ele envia fatos crus;
o backend deriva ATIVO / PAUSA / AUSENTE.

**Regra de negocio (fica so no backend):**
- `>= 5 min` sem input -> PAUSA
- `>= 15 min` sem input -> AUSENTE

**Contrato de eventos exigido pelo Marcos:** os eventos tem de refletir a realidade **no momento certo**.
Ao entrar em ociosidade, o Agent precisa:
1. **fechar** o evento de janela em andamento (ex.: VS Code) com `finalizado_em` = instante da ociosidade;
2. **abrir** o evento de ociosidade com `finalizado_em = NULL` (em andamento);
3. ao voltar o input, fechar a ociosidade e abrir a nova janela ativa, tambem com `finalizado_em = NULL`.

**Ponto de arquitetura a fechar (@Tony):** se o Agent fechasse a janela so aos 5 min, o limiar de negocio
estaria duplicado em dois lugares (agent + backend) e vai divergir com o tempo. Recomendacao:

> O Agent mantem **um unico limiar tecnico de deteccao** (hoje 60s) e, ao detecta-lo, fecha a janela e abre
> a ociosidade em andamento. O backend recebe o intervalo de ociosidade e aplica sozinho a regra de negocio:
> < 5 min = micro-pausa (tempo reatribuido ao bloco de janela anterior), >= 5 min = PAUSA, >= 15 min = AUSENTE.

Assim os numeros 5 e 15 existem em um lugar so, e o Agent nao precisa saber o que e "pausa".
**Contrapartida:** com deteccao a 60s os blocos fragmentam mais, e o backend passa a ser obrigado a
mesclar micro-pausas — senao o pilar Fragmentacao infla. Decisao final do formato do contrato precisa
de @Thor / @Shuri (srv-events) antes de o @Bucky implementar.

**Desdobramentos:**
- Depreciar `eventos_transicao_status` e o codigo de status no Agent (`UserStatusManager`, `status_usuario`
  nos payloads) — coordenar remocao da tabela com @Thor.
- O PR4 de thresholds (`pr4-threshold-inatividade-bucky.md`) fica **obsoleto** na parte de Agent:
  os thresholds 5/15 nao devem viver no C#.
- Substitui os achados A-08 e A-13 — o campo NULL deixa de ser bug e passa a ser o comportamento correto.

---

## A-15 — Limiar de ociosidade: 60s no codigo vs "5+ minutos" na documentacao

**Severidade:** MEDIA (doc) | **Dono:** @Bucky + @Steve (doc de produto)

`appsettings.json` usa `LimiteOciosidadeSegundos: 60`. Comprovado no teste: 5 ociosidades curtas
registradas (77s, 78s, 88s, 97s, 153s) so de pausas normais de leitura.
`produto.md` e o README do agent dizem "sem input de teclado/mouse por **5+ minutos**".

Com o **D-01**, os 60s passam a ser limiar **tecnico de deteccao** (correto), e o "5 min" vira regra de
negocio do backend. **Acao:** corrigir a documentacao de produto para descrever o novo desenho — nao mexer
no limiar do Agent.

---

## A-16 — Janela ativa nao e fechada durante a ociosidade (tempo inflado)

**Severidade:** ALTA | **Dono:** @Bucky | **Resolve junto com D-01 e A-12**

Evidencia (agente 50, 2026-08-18):

| Evento | Intervalo |
|---|---|
| Janela VS Code (id 48368) | 11:46:50 -> 11:49:35 (2min45s) |
| Ociosidade (id 4355) | 11:47:05 -> 11:49:38 |

O bloco de janela cobre quase integralmente um periodo de ociosidade e foi gravado como janela ativa.
O plano exige (R3.2.2) que os eventos de janela parem durante a ociosidade.
Se o backend nao subtrair a ociosidade, o tempo entra como produtivo.

Na ociosidade longa (11:51:36 -> 11:58:34, 6min57s, registrada corretamente com motivo `IdleTimeout`),
o ultimo evento de janela e das 11:50:47 — o VS Code ficou 8 minutos em foco sem gerar registro (A-12).

---

## A-17 — Paridade Windows <-> Android no ciclo de vida dos eventos (implementacao de referencia)

**Severidade:** ALTA — engloba A-12, A-13, A-16 | **Dono:** @Bucky | **Review:** @Tony
**Decisao do Marcos (2026-08-18):** "quero que funcione igual ao agent-android — ele esta funcionando 100%".

O Agent Android e a referencia canonica. Copiar o comportamento, nao reinventar.

### Referencias de codigo (repo `manager-srv-agent-android`)

- `app/src/main/kotlin/com/trivion/manageragent/collector/WindowActivityCollector.kt`
- `app/src/main/kotlin/com/trivion/manageragent/collector/IdleDetector.kt`
- `app/src/main/kotlin/com/trivion/manageragent/event/model/AgentEvent.kt`

### Comportamento esperado (Android hoje x Windows 1.5.3)

| Momento | Android (correto) | Windows 1.5.3 (hoje) |
|---|---|---|
| Janela entra em foco | emite `WindowActivity` com `finalizadoEm = null` | nao emite nada |
| Janela segue em foco | reemite snapshot aberto a cada `snapshotIntervalMillis` = 60s | nada |
| Troca de janela | fecha a anterior (`finalizadoEm` = inicio da nova) e abre a nova | so aqui grava, ja fechada |
| Ociosidade detectada (60s) | emite `Idle` com `finalizadoEm = null` e `iniciadoEm` = ultimo instante de interacao | grava depois, ja fechada |
| Usuario volta | reemite `Idle` com o MESMO `iniciadoEm` + fim real | — |
| Lock de tela | `resetSessionBoundary(unlocked=false)` fecha janela e ociosidade em andamento antes de zerar estado | a validar |
| Ociosidade longa | **nao** reemite snapshot (decisao consciente — "ainda ausente" e derivavel da linha aberta) | — |
| `statusUsuario` | manda `"ATIVO"` / `"AUSENTE"` (nunca `PAUSA`) | manda NULL (A-13) |

### O backend ja suporta — nao precisa de mudanca

`manager-srv-events-node/src/modules/ingestion/handlers/`:
- `window-activity.handler.ts` — UNIQUE `(agente_id, nome_processo, iniciado_em)`
  (migration `20260815000000_uq_eventos_janela_sessao.sql`) + `onConflictDoUpdate`
- `idle.handler.ts` — UNIQUE `(agente_id, iniciado_em)`
  (migration `20260815010000_uq_eventos_ociosidade_sessao.sql`) + upsert
- Regra de merge nos dois: **primeiro `finalizadoEm` nao-nulo vence** (fechamento e imutavel)

O proprio comentario do `window-activity.handler.ts` registra: *"Windows so emite o evento fechado 1x:
chave inedita, upsert degenera em INSERT puro"* — o backend ja foi escrito prevendo esta mudanca no Agent.

### Serializacao

O Android omite o campo quando nulo (`explicitNulls = false`) — o contrato Zod usa `.optional()`, nao
`.nullable()`. O C# precisa fazer o mesmo: **omitir** `finalizadoEm` em vez de mandar `"finalizadoEm": null`.

### Armadilhas ja resolvidas no Android (nao repetir no C#)

Ambas achadas ao vivo em staging no agente 49, em 2026-08-15:
1. O tick de ociosidade emitia `finalizadoEm = clock()` no inicio da ausencia — fabricava fechamento falso
   e toda ausencia virava 2 linhas nao reconciliadas. Correto: inicio sai com `finalizadoEm = null`.
2. `resetSessionBoundary` zerava o estado sem emitir o fechamento — ausencia em andamento no momento de um
   LOCK ficava aberta para sempre (id 4269 ficou >2h aberto). Correto: emitir o fechamento antes de zerar.

### Fora de escopo

`isMediaActive()` (video sem toque na tela nao conta como ausencia) e especifico de mobile. No Windows o
equivalente seria audio/video em foreground — **nao implementar agora**, avaliar depois com @Tony.

---

## A-18 — Boundary de LOCK cria janela fantasma `unknown` de 1ms

**Severidade:** MEDIA | **Dono:** @Bucky | **Amarrado ao A-17**

No instante do LOCK o agent gravou `eventos_janela` id **48376**:
`nome_processo = "unknown"`, `iniciado_em = 12:19:56.311`, `finalizado_em = 12:19:56.312` — **1 milissegundo**.

Em vez de **fechar a janela real** que estava em foco (comportamento do Android em `resetSessionBoundary`),
o Windows fabrica um registro fantasma de processo desconhecido. Entra na contagem de trocas de janela e
polui o pilar Fragmentacao com um bloco que nunca existiu.

**Correcao (junto com A-17):**
- **No LOCK:** fechar a janela em andamento com `finalizadoEm` = instante do lock. Se havia ociosidade em
  andamento, fechar tambem (armadilha 2 do A-17, ja corrigida no Android). Nao criar registro novo.
- **No UNLOCK:** fechar o que restou aberto e abrir o evento da janela que ganhou foco, com
  `finalizadoEm = null` (a pessoa vai focar em algo — decisao do Marcos, 2026-08-18).

**Cuidado de corrida:** no instante do unlock o Windows reporta a lock screen / `unknown` como janela ativa
por fracao de segundo. Abrir o evento cedo demais grava lixo igual a este achado.
O Android resolve com `debounceMillis = 1_500L` antes de promover a janela pendente a committed
(`WindowActivityCollector.promotePending`). O C# precisa do mesmo debounce.

---

## A-19 — LOCK/UNLOCK devem ser enviados no ato, nao no ciclo de 60s

**Severidade:** MEDIA | **Dono:** @Bucky
**Decisao do Marcos (2026-08-18)** — e **ja e o comportamento do Android**, entao e paridade, nao regra nova.

Medido nesta rodada:

| Evento | Ocorreu | Subiu | Atraso |
|---|---|---|---|
| LOCK (id 4681) | 12:19:55 | 12:20:21 | 26s |
| UNLOCK (id 4682) | 12:20:21 | 12:21:25 | **64s** |

**Referencia Android** — comentario em `sender/UploadCoordinator.kt`:

> "Desde A4B, o ticker de 60s e o LOCK/UNLOCK **nao** chamam mais esta classe — eles chamam
> `EventUploader.drain` direto, no escopo do service, para nao depender do JobScheduler no caminho quente."

Ou seja: no Android o LOCK/UNLOCK dispara flush imediato, fora do agendador. O Windows precisa de um
caminho equivalente — drain direto no `UploadWorker`/pipeline, disparado pelo evento de sessao.

**Requisito de robustez (@Tony):** o envio imediato **nao pode** travar o lock nem propagar excecao.
Sem rede ou com backend fora do ar, degrada silenciosamente para o buffer e sobe no proximo ciclo.
O caminho quente precisa de timeout curto e ser fire-and-forget.

---

## A-20 — Relancamento do SessionWorker leva 97s: ignora o EOF do pipe e espera timeout de heartbeat

**Severidade:** ALTA | **Dono:** @Bucky | **Review:** @Tony
**Criterio do plano (R13.1.4):** worker relancado em <= 15s. **Medido: 97s.**

Linha do tempo (`service-20260818.log`, worker morto a forca pelo Gerenciador de Tarefas):

| Hora | Log |
|---|---|
| 09:32:07.246 | `PipeServer: worker disconnected (EOF). Session=1` — morte detectada **na hora** |
| 09:33:44.701 | `WorkerWatchdog: dead worker detected. Session=1, PID=17908, Reason=Heartbeat timeout: last ping was 104s ago` |
| 09:33:44.714 | `WorkerWatchdog: attempting re-launch for Session=1.` |
| 09:33:44.734 | `Worker launched via WTSQueryUserToken. Session=1, PID=8888, Attempt=1` |
| 09:33:45.363 | `PipeServer: worker connected. Session=1` |

**Causa:** o sinal de morte chega imediato pelo EOF do Named Pipe, mas o relancamento so e disparado pelo
`WorkerWatchdog`, que depende de um timeout de heartbeat (~104s). O `PipeServer` loga o EOF e nao aciona
`handling worker death`.

**Impacto:** 97 segundos sem captura nenhuma — janela, ociosidade e sessao. Pior: o Service continua
enviando batimentos nesse periodo, entao no portal o agente aparece saudavel enquanto nao coleta nada.

**Correcao proposta:** disparar `handling worker death` + re-launch direto no EOF do pipe.
Manter o `WorkerWatchdog` por heartbeat como **fallback** — ele continua sendo o unico sinal quando o
worker trava vivo, sem fechar o pipe (processo existe, para de responder).

**Funcionou bem (nao mexer):** o relancamento em si — `WTSQueryUserToken`, primeira tentativa,
pipe reconectado em menos de 1s.

---

## A-21 — Race na inicializacao do Service: adota o worker orfao E lanca outro (2 workers na mesma sessao)

**Severidade:** ALTA | **Dono:** @Bucky | **Review:** @Tony

Ao reiniciar apos kill do `ManagerAgent.Service`, o Service adotou o worker orfao existente e, 32ms depois,
lancou um segundo worker para a **mesma sessao**:

```
09:38:29.260 [INF] Orphan worker adopted. Session=1, PID=8888
09:38:29.270 [INF] Adopted orphan worker. Session=1, PID=8888
09:38:29.273 [INF] PipeServer: worker connected. Session=1
09:38:29.292 [INF] Worker launched via WTSQueryUserToken. Session=1, PID=19424, Attempt=1
09:38:29.293 [INF] Worker registered for session 1, PID=19424
```

Confirmado no SO: PIDs **8888** (09:33:44) e **19424** (09:38:29) rodando lado a lado na sessao 1.
Sintoma colateral no log: `worker connected` seguido de `worker disconnected (EOF)` no mesmo milissegundo
(09:38:29.896 / .897) — os dois workers disputando o mesmo Named Pipe.

**Causa provavel:** o caminho de launch nao consulta o registro antes de criar o worker, ou
`Worker registered for session 1` sobrescreve a entrada do adotado — o PID 8888 fica solto, sem gestor,
e nem o watchdog nem o shutdown o alcancam.

**Impacto:**
- Dois processos capturando a mesma sessao (duplicacao de eventos depende do upsert do backend salvar —
  nesta rodada nao houve duplicata visivel, gracas a chave `(agente_id, nome_processo, iniciado_em)`,
  mas isso e sorte do backend, nao desenho do agent)
- ~290 MB de RAM em vez de ~145 MB (agrava o A-07)
- Worker zumbi sobrevive ao ciclo de vida do Service

**Correcao proposta:** o launch deve ser idempotente por sessao — verificar o registro (e o adotado)
antes de criar; se ja existe worker vivo e conectado para a sessao, nao lancar. Cobrir com teste de
`Service restart com worker vivo`.

**Acao tomada na rodada:** PID 8888 encerrado manualmente para o regressivo seguir limpo (restou so o 19424).

**Funcionou bem (nao mexer):**
- SCM reiniciou o Service dentro do criterio (PID novo 20404), `AGENT_STARTUP` com `Reason=SERVICE_START`
- Adocao de orfao em si funciona
- Upload retomou sem perda: 10 eventos no 1o ciclo, 5 no seguinte, `FailedBatches=0`
- `UpdateCheckerWorker` consultou **staging** (`staging-api-admin.imanagerportal.com`) e respondeu
  "no update available" — config apontando para o ambiente correto
- `WorkerWatchdog started. CheckInterval=15s, HeartbeatTimeout=90s, MaxRelaunches=3/5min`

---

## A-22 — Falha de upload sem rede e logada como erro fatal com stack trace

**Severidade:** BAIXA (ruido de log) | **Dono:** @Bucky

Durante o teste de queda de rede, cada ciclo de upload gerou no `service-*.log`:

```
[ERR] Erro ao enviar lote de eventos
System.Net.Http.HttpRequestException: No such host is known. (staging-api-events...)
 ---> System.Net.Sockets.SocketException (11001): No such host is known.
   at System.Net.Http.ConnectHelper.ConnectAsync(...)
   ... (stack completo)
[ERR] Falha no ciclo de upload
```

Tres problemas:
1. **Nivel errado:** ausencia de rede e condicao esperada e ja tratada (buffer). Deve ser `[WRN]`, nao `[ERR]`.
2. **Stack trace inutil:** o agent **ja sabia** que estava offline — o proprio heartbeat do mesmo minuto
   registra `RedeDisponivel: false`. Logar 30 linhas de stack de DNS por ciclo enche o log.
3. **Linha duplicada:** o mesmo ciclo loga o erro no envio do lote **e** de novo no ciclo, dobrando o volume.

**Efeito:** log de producao poluido; `[ERR]` perde valor como sinal (quem monitora por nivel vira ruido).
Em 10 minutos offline o arquivo cresceu com ~10 blocos de stack identicos.

**Correcao proposta:** classificar falha de transporte (DNS/socket/timeout) como `[WRN] Upload adiado — sem rede,
N eventos no buffer`, sem stack; reservar `[ERR]` + stack para respostas 5xx/contrato invalido/erro inesperado.
Consultar `RedeDisponivel` antes de tentar o ciclo e sair cedo.

---

## A-23 — `EventosPendentes` sempre 0 no heartbeat, mesmo com buffer cheio

**Severidade:** MEDIA | **Dono:** @Bucky

Durante o apagao de rede o buffer subiu 21 -> 23 -> 26 eventos, mas todos os batimentos do periodo
reportaram `EventosPendentes: 0`.

**Efeito:** o campo e o unico indicador remoto de acumulo. Zerado, o portal nao consegue distinguir
"agent ocioso, nada a enviar" de "agent offline ha horas com 5000 eventos represados". Perde-se o alerta
mais util de saude do agent.

**Causa provavel:** o heartbeat le um contador em memoria zerado a cada ciclo, em vez de consultar
`COUNT(*)` da tabela de buffer no SQLite.

**Correcao proposta:** popular `EventosPendentes` a partir da contagem real do buffer. Como o heartbeat
tambem so sobe quando ha rede, avaliar com @Shuri incluir tambem `bufferMaisAntigoEm` para o backend
calcular o atraso real na primeira reconexao.

---

## A-24 — Hooks reinstalados durante ociosidade legitima

**Severidade:** MEDIA | **Dono:** @Bucky

No `session-worker-S1-*.log`, durante o periodo em que o usuario estava legitimamente ausente:

```
[WRN] Hooks appear inactive (3 zero summaries). Reinstalling...
```

A heuristica de saude dos hooks de teclado/mouse conclui "hook morreu" a partir de N resumos consecutivos
com zero eventos. Mas **zero input e exatamente o que define ociosidade** — o detector de ociosidade e a
heuristica de saude leem o mesmo sinal e chegam a conclusoes opostas.

**Efeito:** desinstala e reinstala hooks de baixo nivel sem necessidade, justamente no momento em que o
agent precisa detectar com precisao o retorno do usuario. Risco de perder o primeiro input da volta e
adiar o fechamento da ociosidade.

**Correcao proposta:** suprimir a heuristica enquanto o `IdleDetector` estiver com ociosidade em andamento.
Validar saude do hook por um sinal proprio (ex.: o hook responde a um ping/`CallNextHookEx` de teste, ou
verificar se o handle ainda esta registrado via `GetLastError`), nunca por ausencia de input.

---

## A-25 — ~~Log do SessionWorker com codificacao errada (mojibake)~~ **NAO PROCEDE**

**Status:** DESCARTADO na revisao do @Tony (2026-08-18) | **Severidade:** — | **Dono:** —

Reportado a partir de `UsuÃ¡rio entrou em ociosidade` visto no `session-worker-S1-20260818.log`.
**Era erro de leitura, nao de escrita.** O hexdump dos bytes reais mostra `U s u \303\241 r i o` —
`C3 A1` e o UTF-8 correto de "á". O mojibake veio do terminal em que o log foi lido (locale Latin-1),
nao do agent. O sink Serilog do SessionWorker esta configurado exatamente igual ao do Service.

**Fica so a parte legitima:** as mensagens de log do agent estao em portugues, contrariando a regra do
projeto (codigo e log em ingles, documentacao em portugues). Tratado dentro do L7 do plano, sem
severidade propria — na 1.5.4 as mensagens de janela e ociosidade ja foram convertidas.

**Licao:** verificar bytes antes de abrir achado de encoding.

---
## A-26 — `ConfigManager.TentarCarregar` devolve "nao existe" quando na verdade e "sem permissao"

**Severidade:** MEDIA | **Dono:** @Bucky | **Mesma classe do A-10**

`C:\ProgramData\ManagerAgent\config.json` tem ACL restrita (protecao correta). Lido por um processo
sem elevacao, o `Get-Content`/`File.ReadAllText` lanca `UnauthorizedAccessException`, que e engolido:
`TentarCarregar` retorna `false` — indistinguivel de "arquivo nao existe".

**Descoberto ao rodar a suite:** `ConfigManagerTests.TentarCarregar_WhenFileDoesNotExist_ReturnsFalseAndDefaultConfig`
falha nesta maquina. O teste ja previa "se o arquivo existir, deve carregar" — o arquivo **existe** e o
carregamento **falha**, porque o processo de teste nao e elevado.

Em producao o Service roda como SYSTEM e le sem problema, entao o impacto e em ferramentas de suporte e
diagnostico — exatamente onde o A-10 ja morde. E o mesmo defeito de fundo nas duas pontas (C# e PowerShell):
**erro de permissao virando "campo ausente"**.

**Correcao proposta:** tratar `UnauthorizedAccessException` explicitamente e propagar um resultado distinto
(`ConfigLoadResult.SemPermissao` vs `NaoEncontrado`), logando o motivo real. Corrigir junto com o A-10.

---

## A-34 — Dedup de LOGIN nunca funcionou: `WTSLogonTime` nao e suportado pelo Windows

**Severidade:** ALTA | **Dono:** @Bucky | **Review:** @Tony | **Para a 1.5.5**
**Regressao de defeito ja corrigido:** e o mesmo bug que a spec `2026-07-09-agent-v1.4.0`
(item 4) fechou. A correcao foi escrita, entrou no codigo e **nunca executou**.

### Sintoma

Matar o SessionWorker duas vezes (teste do A-20) gerou dois LOGIN falsos. A sessao Windows
do usuario nunca foi interrompida — ele nao deslogou nem uma vez.

| id | tipo | ocorreu_em (-03) | origem |
|---|---|---|---|
| 4863 | LOGIN | 11:50:44.001 | restart do worker apos kill #1 |
| 4864 | LOGIN | 11:50:50.548 | restart do worker apos kill #2 |

Vale para **todo** restart do worker: kill, crash, auto-update, fast user switch. Hoje o
`session-worker-S1` registrou **35** inicializacoes — e 35 LOGINs.

### Causa raiz

`SessionMonitor.MaybeEmitInitialLogin()` tem o caminho de dedup certo: monta um `LogonId`
estavel (`SessionId:LogonTime:UserName`), compara com o persistido e so emite LOGIN se divergir.
Mas se `QueryLogonId` devolver `null` ele cai no fallback conservador:

```
[WRN] Nao foi possivel resolver LogonId via WTS. Emitindo LOGIN em modo compat legado.
```

Esse WRN aparece **35 vezes em 35 inicializacoes**. O fallback e o unico caminho que roda.

O `null` vem de `WtsHelpers.QueryLogonTimeTicks`, que chama:

```csharp
NativeMethods.WTSQuerySessionInformation(..., WtsInfoClass.WTSLogonTime, out buffer, out bytes)
```

`WTSLogonTime` (classe 18) **nao e suportada** por `WTSQuerySessionInformation` — a Microsoft
documenta que ela sempre retorna FALSE com `ERROR_NOT_SUPPORTED`. Nao e falha de ambiente nem
de permissao: e a API se comportando como especificada.

Medido nesta maquina (`scratchpad/wts.ps1`, sessao 1):

| classe | valor | ok | bytes | LastError |
|---|---|---|---|---|
| WTSUserName | 5 | True | 10 | — |
| **WTSLogonTime** | **18** | **False** | **0** | **50 = ERROR_NOT_SUPPORTED** |
| WTSSessionInfo | 24 | True | 144 | — |
| WTSSessionInfoEx | 25 | True | 160 | — |

O comentario no codigo chama o `null` de "raro". Ele e universal — o caminho feliz nunca rodou
em maquina nenhuma.

### Correcao

Trocar a classe 18 pela **25 (`WTSSessionInfoEx`)**, que devolve `WTSINFOEX` → `WTSINFOEX_LEVEL1`
com `LogonTime` como `LARGE_INTEGER`. Alternativa equivalente: classe 24 (`WTSSessionInfo` →
`WTSINFO`). Ambas confirmadas funcionando acima.

**Ganho de tabela:** `WTSINFOEX_LEVEL1` traz tambem `SessionFlags`
(`WTS_SESSIONSTATE_LOCK` / `WTS_SESSIONSTATE_UNLOCK`) — estado de bloqueio autoritativo, vindo
do proprio Windows. E exatamente o que falta no [A-33](#a-33): o `OpenInputDesktop` de hoje nao
consegue abrir o input desktop enquanto o Winlogon esta no controle, e por isso o polling
demorou 17s para perceber o lock. Ao implementar o A-34 vale ler o mesmo struct para o A-33 e
resolver os dois com uma P/Invoke so.

### Criterio de aceite

1. `QueryLogonId` devolve nao-null nesta maquina (o WRN "compat legado" some do log).
2. Matar o SessionWorker **nao** gera LOGIN novo em `eventos_sessao`.
3. Logoff/logon de verdade **continua** gerando LOGIN.
4. Teste unitario cobrindo `QueryLogonId != null` para a sessao corrente.

---

## A-35 — Kill do worker deixa o evento de janela aberto para sempre

**Severidade:** ALTA | **Dono:** @Bucky | **Review:** @Tony | **Para a 1.5.5**

### Sintoma

No instante do kill, a janela em foco era o Gerenciador de Tarefas. O worker morreu sem fechar
o evento, e ninguem fechou depois:

| id | processo | iniciado_em (-03) | finalizado_em |
|---|---|---|---|
| 49169 | Taskmgr | 11:50:32 | **ABERTO** (kill foi 11:50:43) |

O registro fica aberto indefinidamente. Para o backend, que calcula duracao por
`finalizado_em - iniciado_em`, isso e um bloco de duracao indefinida na conta do colaborador —
ou, dependendo de como o relatorio trata o null, atividade silenciosamente descartada.

Vale para qualquer morte nao-graciosa do worker: kill, crash, `taskkill /F`.

### Causa raiz

`_currentEvent` do `WindowActivityService` so existe em memoria. O flush acontece em
`StopAsync` / no lock / na troca de janela — todos caminhos que exigem o processo vivo.
`SIGKILL` nao passa por nenhum deles.

O worker novo tambem nao resolve: ele sobe com `_currentEvent = null` e nao tem como saber o
que o anterior tinha aberto.

### Correcao proposta

O Service **ja sabe** a hora exata da morte — ele detecta o EOF do pipe (A-20):

```
11:50:43.519 [INF] PipeServer: worker disconnected (EOF). Session=1
```

E ja conhece o ultimo evento de janela aberto daquela sessao, porque o snapshot de 60s (A-12)
o reemite periodicamente. Entao: ao tratar a morte do worker, o Service fecha o ultimo evento
de janela conhecido com `FinalizadoEm = instante do EOF`, com motivo `WorkerDied`.

Isso mantem a correcao no lado que sobrevive — o Service — em vez de tentar persistir estado
no processo que acabou de morrer.

**Alternativa descartada:** persistir `_currentEvent` no `autonomous-buffer.db` a cada troca.
Custa escrita em disco a cada foco novo e ainda perde o intervalo entre a ultima escrita e o
kill. O EOF do pipe da o instante exato, de graca.

### Criterio de aceite

1. `taskkill /F` no SessionWorker → o evento de janela em curso fecha com `finalizado_em` = hora do EOF (tolerancia 1s).
2. Nenhum `eventos_janela` com `finalizado_em is null` sobra alem do que esta genuinamente em curso.
3. Parada graciosa do worker continua fechando pelo caminho normal (sem fechar duas vezes).

---

## A-36 — LOGIN e LOGOFF nao disparam upload imediato

**Severidade:** MEDIA | **Dono:** @Bucky | **Review:** @Tony | **Para a 1.5.5**

O A-19 fez LOCK/UNLOCK subirem na hora. LOGIN e LOGOFF ficaram de fora e ainda esperam o ciclo
de 60s:

| evento | ocorreu_em | criado_em | atraso |
|---|---|---|---|
| LOCK / UNLOCK (pos A-19) | — | — | 1,5s a 4,0s |
| LOGIN (id 4863) | 11:50:44.001 | 11:51:39.108 | **55,1s** |
| LOGIN (id 4864) | 11:50:50.548 | 11:51:39.108 | **48,6s** |

Causa: `PipeMessageHandler.ContainsSessionBoundary` so reconhece `LOCK` e `UNLOCK`.

O caso que importa e o **LOGOFF**: ele acontece quando a maquina esta desligando, e 60s de
espera e mais tempo do que o Windows costuma dar antes de matar os processos. Um LOGOFF perdido
deixa a sessao do dia sem fechamento.

**Correcao:** incluir `LOGIN` e `LOGOFF` na lista. O A-34 reduz o volume de LOGIN a quase zero,
entao o custo de upload extra e desprezivel.

**Criterio de aceite:** LOGIN e LOGOFF com `criado_em - ocorreu_em` abaixo de 5s.

---

## A-33 — Duas fontes de estado de lockscreen que nao se falam (LOCK/UNLOCK duplicado + tela de bloqueio capturada)

**Severidade:** ALTA | **Dono:** @Bucky | **Review:** @Tony | **Para a 1.5.5**
**Medido com a maquina limpa** (apos o A-32), entao os numeros valem.

O agent tem **duas** fontes independentes de estado de lockscreen:

| Fonte | Quando dispara | Onde |
|---|---|---|
| `SystemEvents.SessionSwitch` do Windows | **na hora** (evento do SO) | `SessionEventService` |
| Polling WTS (`IsInputDesktopLocked` + `QueryConnectState`) | a cada 5s | `LockScreenDetector` |

Elas nao compartilham estado. O unico acoplamento e o `NotifySessionSwitchLock`, com janela de dedup
de **2 segundos**. Isso produz dois defeitos.

### Sintoma 1 — LOCK e UNLOCK duplicados

Um unico ciclo de Win+L gerou 2 de cada:

| Evento | Fonte | Horario | Distancia |
|---|---|---|---|
| LOCK | SessionSwitch | 14:44:51.354 | — |
| LOCK | polling WTS | 14:45:08.458 | **+17,1s** |
| UNLOCK | SessionSwitch | 14:45:08.826 | — |
| UNLOCK | polling WTS | 14:45:13.601 | **+4,8s** |

A janela de dedup de 2s nao cobre nem de longe. E o **UNLOCK nao tem dedup nenhum** —
`OnExitLockscreen` emite direto, sem consultar nada.

Como `eventos_sessao` nao tem UNIQUE (diferente de `eventos_janela` e `eventos_ociosidade`), tudo entra.

### Sintoma 2 — a tela de bloqueio vira janela ativa

Pior que a duplicata:

```
11:44:52.016  LOCK detectado - flushando  (SessionSwitch: fecha a janela, mas NAO suspende a captura)
11:44:55.083  Window event opened: "Microsoft(R) Windows(R) Operating System"
                                 / "Tela de Bloqueio padrao do Windows"
11:45:08.458  Window event closed on flush   <- so agora o polling suspende
```

**15 segundos de tela de bloqueio gravados como janela ativa.** O `SessionEventService` trata o LOCK
fazendo flush da janela e da ociosidade, mas **nao chama `_windowActivityService.Pause()`** — quem
pausa e so o `LockScreenDetector`, que chegou 17s depois. No intervalo, a captura seguiu rodando e
pegou a lock screen.

Numa maquina que fica bloqueada a noite inteira, o atraso do polling define quanto de "trabalho"
fantasma e contabilizado.

**Por que o polling atrasa tanto:** o LOCK so foi detectado as 14:45:08.458 — **0,37s antes do usuario
desbloquear**. Ou seja, durante o bloqueio inteiro o `QueryLockedState` nao reconheceu o estado.
O `IsInputDesktopLocked` depende de abrir o input desktop, o que o processo do usuario nao consegue
enquanto o Winlogon esta no controle. O sinal confiavel e o `SessionSwitch`; o polling e o fallback
para auto-lock por politica (que nao dispara SessionSwitch) — nunca deveria ser o caminho principal.

**Correcao proposta — estado unico, sem janela de tempo:**

1. O `LockScreenDetector` passa a ser o **dono unico** do estado de lock. O `SessionSwitch` deixa de
   emitir por conta propria e passa a **alimentar** o detector (`NotifyLock(instante)` /
   `NotifyUnlock(instante)`).
2. A transicao acontece uma vez so, em um lugar so: emite o SessionEvent, fecha janela e ociosidade,
   suspende os monitores. Quem chegar depois com o mesmo estado vira no-op — **sem janela de 2s**,
   porque a dedup passa a ser por estado, nao por tempo. Janela de tempo aqui e chute; o atraso medido
   (17s) mostra que qualquer valor escolhido ia errar.
3. `Pause()` da captura de janela tem de acontecer **junto com o flush**, na mesma transicao. Hoje o
   flush esta em um caminho e o pause em outro — e essa separacao que abre a fresta de 15s.
4. Avaliar UNIQUE `(agente_id, tipo_evento, ocorreu_em)` em `eventos_sessao` com @Shuri: os outros dois
   eventos ja tem chave de reconciliacao e foi ela que segurou o estrago do A-32. **Defesa em
   profundidade, nao substituto** da correcao no Agent.

**Nao mexer (funcionou):** o fechamento da janela no instante exato do LOCK (A-18) e o envio imediato
(A-19) — ver secao de cobertura.

---

## A-32 — CRITICO: `dotnet test` sobe SessionWorkers de verdade na instalacao de producao

**Severidade:** CRITICA (contamina dados e mascara defeitos) | **Dono:** @Bucky + @Natasha | **Review:** @Tony

`tests/ManagerAgent.Service.Tests/WorkerLauncherTests.cs` instancia o **`WorkerLauncher` real** e chama:

```csharp
var result = await launcher.LaunchWorkerAsync(sessionId: 9999, CancellationToken.None);
```

O comentario do teste assume:

> "On macOS / without elevated privileges, WTS + explorer tokens fail, and Task Scheduler is not
> available. The launcher must return null — not throw."

**Nao e verdade numa maquina Windows com sessao interativa.** O `TryLaunchViaExplorerToken` acha o
`explorer.exe` da sessao ativa, duplica o token e **sobe um `ManagerAgent.SessionWorker.exe` real**.
Sao 3 testes assim (`sessionId: 9999`, `0` e `uint.MaxValue`).

**Comprovado em 2026-08-18:** apos ~8 execucoes da suite durante a rodada, a maquina tinha **9**
SessionWorkers vivos (1 legitimo + 8 orfaos, todos iniciados entre 11:34:15 e 11:34:40, batendo com a
execucao do `dotnet test`). Arquivos de log orfaos comprovam os session ids sinteticos:

```
session-worker-S0-20260818.log
session-worker-S9999-20260818.log
session-worker-S4294967295-20260818.log   <- uint.MaxValue
```

**Estrago:**

| Efeito | Detalhe |
|---|---|
| Dados contaminados | Cada worker orfao detecta LOCK/UNLOCK e emite eventos por conta propria. O teste de lock/unlock registrou **16 LOCK e 26 UNLOCK** para um unico ciclo — quase tudo dos orfaos |
| Icones fantasma | Cada worker desenha o seu na bandeja — a maior parte dos 17 icones do A-29 eram **processos vivos**, nao fantasmas |
| Memoria | ~140 MB cada, ~1,1 GB no total |
| Diagnostico envenenado | Os orfaos escrevem na pasta de log de producao e conectam no Named Pipe de producao |
| Falso positivo de regressao | A duplicacao de LOCK/UNLOCK parecia regressao do L3.1 (drain imediato). Nao era |

O Service se defendeu certo — rejeitou todos com `CONNECT rejected: PID mismatch. Expected=8904` — mas os
processos continuam vivos, capturando e enfileirando por fora.

**Correcao proposta:**
1. `WorkerLauncherTests` nao pode exercitar o launcher real. Extrair uma interface para as 3 estrategias
   de launch e testar com dublê; ou marcar os testes com `Skip` quando houver sessao interativa
   (`Environment.UserInteractive` + agent instalado).
2. Se algum teste precisar mesmo lancar processo, tem de **matar o que subiu** no `Dispose` do fixture.
3. Guard no `WorkerLauncher`: recusar `sessionId` que nao corresponda a uma sessao WTS existente —
   `9999` e `uint.MaxValue` nunca deveriam chegar a lancar nada. Isso conserta o defeito na raiz e vale
   para producao tambem, nao so para teste.

**Acao tomada na rodada:** os 8 orfaos foram encerrados; sobrou o PID 8904 (o legitimo, registrado no
Service). O teste de LOCK/UNLOCK precisa ser refeito com a maquina limpa — os numeros coletados
antes disso nao valem.

**Licao:** rodar a suite na mesma maquina que hospeda o ambiente sob teste misturou os dois. Enquanto
o item 1 nao for corrigido, **nao rodar `dotnet test` nesta maquina durante uma rodada de regressivo**.

---

## A-30 — Janela aberta durante ociosidade ja em curso vira bloco fantasma de 1ms

**Severidade:** MEDIA | **Dono:** @Bucky | **CORRIGIDO na 1.5.4** | **Encontrado validando a propria 1.5.4**

Primeiro evento de janela do worker 1.5.4 (id **49067**):

| Campo | Valor |
|---|---|
| `nome_processo` | `Windows Terminal` |
| `iniciado_em` | 14:26:19.893 |
| `finalizado_em` | 14:26:19.894 |
| duracao | **0,001s** |

E o mesmo sintoma do A-18 reaparecendo por outro caminho.

**Causa:** o worker subiu as 14:26:19 (troca de binarios do auto-update) enquanto o usuario **ja estava
ausente desde 14:26:12** — ele estava parado olhando o update acontecer. O loop de captura chamava
`WindowActivityService.HandleAsync` **antes** de `IdleMonitorService.CheckAsync`, entao a janela foi
aberta antes de alguem perceber a ausencia. Quando a ociosidade foi detectada (14:27:16, com
`idleTime = 63s`), o `FlushCurrentEventAsync` recebeu `14:26:12` — **anterior** ao `IniciadoEm` da
janela — e o guard de clock jump entrou, gravando `FinalizadoEm = IniciadoEm + 1ms`.

**O guard mascarou o defeito.** Ele existe para relogio que anda para tras, e aqui converteu um erro de
ordem em um dado plausivel. Vale a lição: guard que "conserta" silenciosamente esconde a causa.

**Correcao:** inverter a ordem no loop do `Worker` — ociosidade avaliada **antes** da captura de janela.
`GetLastInputInfo` e do sistema e sobrevive ao restart do processo, entao no primeiro tick apos um
update o worker ja sabe que o usuario esta ausente e suspende a captura. Nenhuma janela e aberta, nao
ha o que fechar, nao ha fantasma.

**Por que nao simplesmente descartar no flush:** o registro **aberto ja subiu** para o backend. Descartar
o fechamento deixaria a linha aberta para sempre — a armadilha 2 do Android (id 4269, >2h aberto).
Prevenir e a unica saida correta.

---

## A-31 — Shell do Windows sem titulo registrado como 50 segundos de trabalho

**Severidade:** MEDIA | **Dono:** @Bucky | **CORRIGIDO na 1.5.4** | **Complementa o A-14**

Evento id **49080**, gravado pelo worker 1.5.4:

| Campo | Valor |
|---|---|
| `nome_processo` | `Sistema operacional Microsoft® Windows®` |
| `titulo_janela` | `sem título` |
| duracao | **50,4 segundos** |

O usuario estava mexendo na bandeja do sistema (limpando os icones fantasma do A-29). Cinquenta
segundos de area de trabalho entraram como janela ativa — vira tempo produtivo no calculo.

**Causa:** a blocklist do L1.3 cobriu o titulo `"Janela de estouro da bandeja do sistema"`, mas o A-14
listava **dois** casos e o segundo era justamente titulo vazio / `"sem título"`. Generalizei pelo eixo
errado: bloqueei o titulo especifico em vez da combinacao shell + sem titulo.

**Correcao:** filtro exige as **duas** condicoes — processo de shell **E** titulo ausente/placeholder.
Bloquear o shell inteiro seria errado: o Explorador de Arquivos roda no mesmo processo, e trabalho de
verdade e **tem** titulo (o nome da pasta). Coberto por teste nos dois sentidos: shell sem titulo e
descartado, Explorador com titulo continua sendo capturado.

Listas configuraveis em `appsettings.json` (`ProcessosShell`, `TitulosPlaceholder`).

---

## A-29 — Icones fantasma na bandeja: cada morte forcada do worker deixa um icone orfao

**Severidade:** MEDIA (muito visivel para o colaborador) | **Dono:** @Bucky | **Review:** @Tony

Apos a rodada de hoje, a bandeja do sistema acumulou **17 icones do iManager**. Conferido no SO no
mesmo instante: existia **1 unico** `ManagerAgent.SessionWorker` (PID 8904). Os outros 16 eram
fantasmas — o Windows mantem o icone desenhado ate alguem passar o mouse por cima, quando ele consulta
o processo dono, ve que morreu e remove.

**Causa:** `TrayIconManager.Dispose()` faz o certo (`_notifyIcon.Visible = false` + `Dispose()`) e e
chamado pelo `TrayApplicationContext.Dispose`. Mas isso **so roda em saida graciosa**. Toda morte
forcada pula o caminho inteiro:

| Origem da morte forcada | Onde |
|---|---|
| `p.Kill(entireProcessTree: true)` | `UpdateApplier.KillSurvivorWorkers()` — **todo update** |
| `taskkill /F /IM ManagerAgent.SessionWorker.exe` | `ManagerAgent-v2.iss`, `CurStepChanged` e `[UninstallRun]` |
| Relancamento do watchdog apos worker travado | `WorkerWatchdog.HandleDeadWorkerAsync` |
| Gerenciador de Tarefas (usuario ou suporte) | — |

Hoje foram muitas: os testes A-20 e A-21 mataram o worker de proposito, mais 3 reinicios do Service
para o teste de update, mais a troca de binarios. Um fantasma por morte.

**Efeito no cliente:** nao e so cosmetico. O colaborador ve uma fileira de icones do iManager e conclui
que "o agente abriu varias vezes" ou que ha algo errado com o monitoramento. Vira chamado de suporte.
E **cada auto-update deixa pelo menos um** — ou seja, acumula sozinho ao longo do tempo, sem ninguem
fazer nada de errado.

**Correcao proposta (2 frentes):**

1. **Encerrar com graça antes de matar.** O `Worker.HandleShutdownAsync` ja implementa o caminho
   completo (LOGOUT, flush, goodbye, `StopApplication`) e o `TrayApplicationContext.Dispose` remove o
   icone. Falta usa-lo: `KillSurvivorWorkers` e o instalador devem mandar o SHUTDOWN pelo pipe e
   aguardar o grace period, deixando o `taskkill /F` como **ultimo recurso**, nao como primeira acao.
   Beneficio extra: os eventos em andamento sao fechados corretamente em vez de ficarem abertos.
2. **Varrer fantasmas no startup.** Morte forcada sempre vai existir (crash, energia, `taskkill` do
   suporte). Ao subir, o worker deve forcar um refresh da area de notificacao (broadcast de
   `WM_MOUSEMOVE` sobre a notification area) para que o Windows recolha os orfaos de execucoes
   anteriores.

**Contorno imediato:** passar o mouse sobre os icones — o Windows remove os mortos na hora.

---

## A-28 — CRITICO: o auto-update do Windows nunca funcionou (versao de 4 partes vs parser de 3)

**Severidade:** CRITICA | **Dono:** @Bucky (agent) + @Shuri (backend) | **Review:** @Tony
**Encontrado em 2026-08-18** ao tentar testar o auto-update para a 1.5.4.

O Agent envia `GET /api/agente/atualizacoes/verificar?versaoAtual=1.5.3.0`. O `.NET`
`Assembly.GetName().Version.ToString()` tem **quatro** componentes.

O `parseVersao` do `manager-srv-admin-node/src/modules/agent-update/agent-update.service.ts` faz:

```ts
const parts = versao.split('.');
if (parts.length !== 3) { throw new Error(`Formato versao invalido: ${versao}`); }
```

e o `isVersaoMaior` engole a excecao (`catch { return false }`), respondendo
`atualizacaoDisponivel: false`.

**Comprovado no endpoint real de staging (2026-08-18):**

| Requisicao | Resposta |
|---|---|
| `?versaoAtual=1.5.3.0` (o que o Agent manda) | `{"atualizacaoDisponivel":false}` |
| `?versaoAtual=1.5.3` (3 partes) | `{"atualizacaoDisponivel":true,"versaoNova":"1.5.4",...}` |

**Efeito:** o backend responde "voce esta atualizado" para **toda** versao que o Agent Windows
pergunta. O auto-update nunca entregou uma versao sequer. O defeito ficou mascarado porque todas as
versoes ate a 1.5.3 foram instaladas manualmente — e a instalacao manual apaga o ProgramData e
revincula, entao ninguem sentiu falta.

**Sao 2 defeitos, um em cada ponta:**

1. **Agent (@Bucky, corrigido na 1.5.4):** enviar SemVer de 3 partes.
   `UpdateCheckerWorker.ObterVersaoSemVer()` monta `Major.Minor.Build`. Coberto por 2 testes,
   um deles guarda direta contra a volta do `Assembly.GetName().Version.ToString()`.
2. **Backend (@Shuri, PATCH PRONTO — ver `PATCH-A28-PARSEVERSAO-SHURI.md`):** duas correcoes.
   - `parseVersao` deve tolerar 4 componentes (ignorar o quarto) em vez de lancar.
   - A falha **nao pode ser silenciosa**: hoje versao mal-formada vira "esta atualizado", que e o
     pior default possivel. Deve logar `WARN` com o valor recebido. Se tivesse esse log, o defeito
     teria aparecido no primeiro deploy em vez de passar despercebido por meses.

**BLOQUEIO PARA O TESTE DE UPDATE:** a maquina roda a **1.5.3**, que continua mandando 4 partes.
Corrigir o Agent na 1.5.4 nao ajuda a sair da 1.5.3 — quem faz a pergunta e o binario velho.
Sem a correcao do backend (item 2), a unica saida da 1.5.3 e instalacao manual, que **nao testa o
auto-update** e ainda destroi o ProgramData (novo `agente_id`).

**Nota de parity:** o comentario do codigo diz "byte-a-byte com Java `parseVersao` linhas 356-363",
entao o backend Java tinha o mesmo defeito. A paridade replicou o bug fielmente. Vale rever se outros
pontos de parity carregam defeitos herdados.

---

## A-27 — `versoes_agente.publicado_em` gravado 3h no futuro (hora local rotulada como UTC)

**Severidade:** BAIXA hoje, ALTA se alguem filtrar por data | **Dono:** @Shuri (srv-admin-node)
**Fora do escopo do Agent** — achado de tabela, encontrado ao publicar a 1.5.4.

Com relogio da maquina e do banco **sincronizados** (`now()` = 13:57:26Z nos dois), as publicacoes ficaram:

| Versao | `publicado_em` gravado | Momento real (local -03:00) | Desvio |
|---|---|---|---|
| 1.5.4 (id 35) | `16:54:37Z` | 10:54 | **+3h** |
| 1.5.3 (id 34) | `14:21:56Z` | 08:21 | **+3h** |

Os dois casos batem com "hora local escrita como se fosse UTC" — o deslocamento e exatamente o offset de
Brasilia. Para comparacao, `eventos_janela.criado_em` grava UTC correto, entao o problema esta no caminho
de publicacao de release, nao no banco.

**Por que nao quebrou agora:** `agent-update.repository.ts` consulta
`WHERE ativa=true AND pausada=false AND sistema_operacional=$so ORDER BY publicado_em DESC` — nao ha filtro
`publicado_em <= now()`. A versao mais nova continua vindo primeiro.

**Por que importa:** basta alguem adicionar esse filtro (agendamento de release, rollout por janela de
horario) para toda versao nova ficar **invisivel por 3 horas** depois de publicada. Alem disso, qualquer
relatorio ou auditoria de release mostra hora errada.

**Correcao proposta:** gravar `publicado_em` em UTC no endpoint de publicacao e corrigir as linhas
existentes. Confirmar com @Shuri se a origem e o painel de release ou o script de deploy.

---

## A-30 — REABERTO: o bloco fantasma de 1ms sobrevive ao conserto da 1.5.4

**Severidade:** BAIXA (dado sujo, nao altera totais) | **Dono:** @Bucky | **Review:** @Tony
**Para a 1.5.6** | **Reaberto em 2026-08-18 na validacao da 1.5.5**

O conserto da 1.5.4 (ordem do loop no `Worker`) cobre a ociosidade que **comeca** com o worker
ja rodando. Nao cobre o worker que **sobe dentro** de uma ociosidade em curso — que e o caso
que o proprio titulo do achado descreve.

### Evidencia (agente 51, validacao da 1.5.5)

| Fato | Instante |
|---|---|
| Ociosidade `4664` comeca | 14:48:20.912 |
| Worker morto (`taskkill`) | 14:48:25 |
| Worker sobe de novo | 14:48:26 |
| `eventos_janela` **49253** aberto | 14:48:26.202 |
| Mesmo evento fechado | 14:48:26.**203** — 1ms |
| Ociosidade `4664` termina | 14:56:04.786 |

### Causa raiz (agora conhecida — o achado original nao a tinha)

`WindowActivityService.FlushCurrentEventAsync`:

```csharp
// Clock jump guard: clamp FinalizadoEm if it's somehow before IniciadoEm
if (_currentEvent.FinalizadoEm < _currentEvent.IniciadoEm)
    _currentEvent.FinalizadoEm = _currentEvent.IniciadoEm.AddMilliseconds(1);
```

O `+1ms` **nao e o defeito** — e um guard contra salto de relogio absorvendo em silencio um erro
de logica. A sequencia:

1. O worker sobe dentro da ociosidade e o `WindowActivityService` abre um evento de janela,
   porque ainda nao sabe que a sessao esta ociosa.
2. O `IdleMonitorService` reconhece a ociosidade **ja em curso** e manda fechar a janela com
   `endedAt` = inicio da ociosidade (14:48:20.912).
3. Esse instante e **anterior** ao inicio da janela (14:48:26.202). O guard dispara e fabrica o
   bloco de 1ms.

**Efeito:** um evento de lixo por reinicio de worker dentro de ociosidade. Nao move total de
relatorio (1ms), mas suja a tabela — e acontece a cada auto-update que pegue a maquina ociosa.

**Correcao proposta:** nao abrir evento de janela enquanto a sessao ja esta ociosa, em vez de
abrir e fechar com carimbo invalido. O guard de relogio continua, mas deve **descartar** o evento
(e logar WARN) em vez de fabricar duracao — se `FinalizadoEm < IniciadoEm`, o evento nunca foi
valido e emitir 1ms so troca um dado errado por outro.

**Licao:** um guard defensivo que "conserta" dado invalido esconde o defeito que o gerou. Preferir
descartar e logar a fabricar valor plausivel.

---

## A-31 — REABERTO: a area de trabalho se chama "Program Manager", nao "sem titulo"

**Severidade:** BAIXA hoje (o bloco sai com 0s), MEDIA se cair fora de ociosidade | **Dono:** @Bucky
**Review:** @Tony | **Para a 1.5.6** | **Reaberto em 2026-08-18 na validacao da 1.5.5**

### Evidencia (agente 51)

```
49268  'Sistema operacional Microsoft(R) Windows(R)'  'Program Manager'  15:06:50 -> 15:06:50  (0.0s)
```

### Causa raiz

O filtro do A-31 exige **duas** condicoes — `IsShellProcess(processName) && IsPlaceholderTitle(windowTitle)`.
O acerto foi deliberado: o mesmo processo serve o Explorador de Arquivos, que e trabalho de verdade
e tem o nome da pasta no titulo. So que a lista de placeholders e:

```csharp
public string[] TitulosPlaceholder { get; set; } =
{
    "sem título", "sem titulo", "untitled", "(sem título)",
};
```

O processo bateu (`Sistema operacional Microsoft`), o titulo **nao**: a area de trabalho do Windows
se identifica como **`Program Manager`** — nome da janela da classe `Progman`, que o Windows nao
traduz. O conserto da 1.5.4 partiu do que apareceu no log daquele dia ("sem título") em vez do
nome canonico da janela.

**Por que saiu com 0s desta vez:** a ociosidade `4670` comecou as 15:06:49, um segundo antes de a
janela abrir — o mesmo clamp do [A-30 REABERTO](#a-30-reaberto) fabricou a duracao. **Fora de
ociosidade o bloco teria a duracao real da area de trabalho em foco**, que e o defeito original.

**Correcao:** somar `Program Manager` a `TitulosPlaceholder` — uma linha — mais teste cobrindo o
titulo real. Vale conferir tambem `Progman` e `Shell_TrayWnd` como nomes de janela.

**Licao:** a lista de placeholders foi montada a partir do sintoma observado num dia, nao do nome
canonico da janela. Filtro por lista literal precisa da fonte canonica, senao cobre so o caso que
alguem ja viu.

---

## A-38 — O LOGOUT da parada do Service nao sobe antes do Service morrer

**Severidade:** MEDIA | **Dono:** @Bucky | **Review:** @Tony | **Para a 1.5.6**
**Encontrado em 2026-08-18 validando a 1.5.5 (T5)**

O A-36 fez LOGIN e LOGOUT acordarem o UploadWorker na hora. Funciona — exceto justamente quando
quem para e o **proprio Service**.

### Evidencia (agente 51)

| Fato | Instante |
|---|---|
| `Graceful shutdown initiated` | 16:11:07.256 |
| LOGOUT emitido pelo worker (`SERVICE_STOP`) | 16:11:07.271 |
| `GOODBYE. PendingEvents=0` | 16:11:09.808 |
| Worker sai sozinho (EOF) | 16:11:09.879 |
| LOGOUT chega no banco | **+38,7s** |

### Causa raiz

`ManagerAgentService.StopAsync` faz, nesta ordem:

1. Broadcast de SHUTDOWN para os workers
2. Espera cada worker sair (5s, senao `Kill`)
3. `_eventBuffer.CheckpointAsync()` — WAL para disco
4. `base.StopAsync`

**Nao ha tentativa final de upload.** O LOGOUT entregue pelo worker no passo 1 cai no SQLite e
espera o proximo ciclo do `UploadWorker` — que so existe depois que o Service voltar. O pedido de
upload imediato do A-36 e emitido, mas o `UploadWorker` esta sendo desmontado no mesmo instante.

**Neste teste** o atraso foi 38,7s porque o Service reiniciou em seguida. **Num desligamento real
da maquina, o LOGOUT so sobe no proximo boot** — potencialmente no dia seguinte. O dado nao se
perde (esta persistido), mas a sessao do dia fica sem fechamento no backend ate la, que e
exatamente o risco que o A-36 pretendia eliminar.

**Correcao proposta:** no `StopAsync`, entre o passo 2 e o 3, uma tentativa final de upload
limitada por tempo (3 a 5s, best-effort). Precisa ser **bounded**: em desligamento real a rede
pode ja estar caindo, e travar o `StopAsync` faz o SCM matar o Service — trocando um atraso por
uma parada suja. O checkpoint continua sendo a rede de seguranca.

**Nao confundir com o A-36:** aquele esta correto e provado (LOGIN 4,7s, LOCK 4,1s, UNLOCK 2,0s).
Este e o caminho que sobra, e so aparece quando o Service e o proprio alvo da parada.

---

## A-37 — Suite de testes grava no estado da instalacao de producao

**Severidade:** MEDIA | **Dono:** @Bucky | **Review:** @Tony | **Encontrado e corrigido em 2026-08-18**
**Mesma familia do [A-32](#a-32)** — teste que toca a maquina sob regressivo em vez de um sandbox.

Encontrado durante a correcao do A-34. O teste
`SessionMonitorRegistrationTests.SessionMonitor_GetAndClearEvents_ReturnsAtLeastLoginOnWindows`
constroi um `SessionMonitor` **real**, e o construtor persiste o LogonId via `SessionStateStore`,
que resolve o caminho por `ConfigPaths.PastaDadosDb` — ou seja,
`C:\ProgramData\ManagerAgent\data\session-state-1.json`, **o mesmo arquivo do agent instalado**.

Evidencia: apos rodar a suite as 12:37:33, o arquivo de producao continha o valor produzido pelo
codigo em desenvolvimento:

```json
{ "lastLogonId": "1:134314716844003445:NoisyTech", "updatedAt": "2026-08-18T15:37:33Z" }
```

**Como se revelou:** o proprio teste passou na primeira execucao e falhou na segunda. Ele afirmava
que o construtor sempre emite LOGIN — verdade ate a 1.5.4, quando o dedup nunca funcionava (A-34).
Com o dedup corrigido, a segunda execucao encontrou o LogonId ja persistido **pela primeira** e
suprimiu o LOGIN, como devia. O teste codificava o bug, nao o contrato.

**Estrago potencial:** um valor gravado por teste no arquivo de producao suprime o LOGIN legitimo do
agent instalado, ou o contrario. Hoje sem efeito pratico porque a 1.5.4 nunca le esse arquivo (cai
sempre no fallback do A-34), mas passa a valer assim que a 1.5.5 for instalada.

**Correcao aplicada:**
1. `ConfigPaths.BaseFolderOverride` — redirecionamento **internal** (visivel so aos projetos de teste
   via `InternalsVisibleTo`), sem superficie de ataque em producao.
2. `SessionMonitorRegistrationTests` passa a apontar para um diretorio temporario proprio por teste,
   com limpeza no `Dispose`.
3. O teste que codificava o bug foi reescrito em dois: um cobrindo a primeira execucao (emite LOGIN)
   e outro cobrindo o criterio de aceite 2 do A-34 — segunda instancia no mesmo logon **nao** emite
   LOGIN de novo.

**Pendencia para a maquina:** o `session-state-1.json` de producao esta com o valor escrito pelo teste.
Apagar antes da validacao da 1.5.5, senao o primeiro LOGIN legitimo sera suprimido e o aceite 3 do
A-34 nao podera ser observado.

**Licao (a mesma do A-32):** teste que constroi objeto real herda os caminhos reais. Todo teste que
toca disco precisa de sandbox explicito.

---

## Observacoes (nao sao defeitos)

- `nome_processo` grava o nome amigavel do produto ("Google Chrome", "Notepad", "Visual Studio Code"),
  nao o executavel ("chrome.exe"). O plano de testes descreve o executavel — corrigir o plano (soma ao A-09).
- `windows_username` vem correto nos eventos (`NoisyTech`). Confirma que o A-02 esta isolado no
  Configurator elevado, nao na captura.
- `url_dominio` gravou `drive.google.com`, sem path — **LGPD OK** neste ponto.
- `categoria_aplicativo` vem NULL; confirmar se e preenchido depois pelo backend (pipeline) ou se falta.
- Upload confirmado a cada ~60s, conforme `appsettings.json`.

---

## Cobertura da rodada

| Bloco | Resultado | Achados |
|---|---|---|
| Instalacao e vinculacao | PASSOU | A-01, A-02, A-03 |
| Bandeja, menu e Sobre | PASSOU | A-04 |
| Verificacao de saude (health-check) | **FALHOU** | A-10, A-11 |
| Captura de janelas | **PARCIAL** — ciclo de vida errado | A-12, A-14, A-16, A-17 |
| Ociosidade | PASSOU (com desenho a mudar) | A-15, D-01 |
| Sessao LOCK/UNLOCK | PASSOU | A-18, A-19 |
| Kill do SessionWorker | **PASSOU na 1.5.4** — 0,55s e 0,80s (criterio 15s) | A-20 fechado; A-34, A-35, A-36 abertos |
| Kill do Service | **FALHOU** — 2 workers na mesma sessao | A-21 |
| Queda de rede e resync | **PASSOU** — sem perda, sem duplicata | A-22, A-23, A-24, A-25 |
| Suite unitaria | 506 passam / 6 falham | A-05, A-06 |

**Detalhe do teste de rede (2026-08-18):** Wi-Fi desligado ~10 min com uso ativo + ociosidade de 5 min.
Captura seguiu normal (buffer 21 -> 26). Ao religar, subiram de uma vez 10 eventos de janela (ids 48394-48403,
todos com `criado_em = 12:50:07`), 2 ociosidades (ids 4365 e 4366, 302s e 303s) e 14 batimentos.
`select nome_processo, iniciado_em, count(*) ... having count(*) > 1` retornou vazio em `eventos_janela` e
em `eventos_ociosidade` — **zero duplicatas**. Cadeia temporal integra: cada `finalizado_em` e exatamente o
`iniciado_em` do bloco seguinte, sem buraco entre 12:42:40 e 12:49:03.
Reconfirmou o **A-16**: o bloco `Windows Terminal` 12:43:57 -> 12:48:59 cobre integralmente a ociosidade
12:43:56 -> 12:48:58 — 5 minutos de ausencia contabilizados como janela ativa.

**Detalhe do teste de kill do SessionWorker (2026-08-18, maquina limpa pos A-32):** dois `taskkill /F`
seguidos. O Service detectou os dois pelo **EOF do pipe**, nao pelo heartbeat:

| # | EOF do pipe | worker novo lancado | CONNECT recebido | total |
|---|---|---|---|---|
| 1 | 11:50:43.519 | 11:50:43.529 (+10ms) | 11:50:44.072 | **0,55s** |
| 2 | 11:50:49.829 | 11:50:49.836 (+7ms) | 11:50:50.630 | **0,80s** |

Era 97s (o worker morto so era percebido no vencimento do heartbeat). Criterio era 15s.
**Exatamente 1** SessionWorker vivo ao final — o guard de `Launching` travado nao gerou duplicata.
Efeitos colaterais do restart viraram A-34 (LOGIN falso), A-35 (janela orfa) e A-36 (atraso de upload).

---

## Ainda nao testado

| # | Bloco | Por que ficou de fora | Quando testar |
|---|---|---|---|
| P-1 | Auto-update e rollback BAA | Precisa de uma versao nova publicada | **Na 1.5.4** — a propria entrega das correcoes vira o teste (decisao do Marcos) |
| P-2 | Performance estabilizada (A-07) | Medido logo apos instalar, com 2 workers vivos (A-21) | Depois da 1.5.4, com o agent rodando ha 4h+ |
| P-3 | Scripts de diagnostico | So o health-check foi rodado | Depois de A-10/A-11 corrigidos |
| P-4 | Multi-sessao (2 usuarios logados) | Nao executado | 1.5.4 |
| P-5 | Named Pipe e AutonomousBuffer | Coberto so indiretamente pelos kills | 1.5.4 |
| P-6 | Logoff / suspensao / reinicio do Windows | Nao executado | 1.5.4 |
| P-7 | Deteccao de reuniao (Teams/Meet/Zoom) | Nao executado | 1.5.4 |
| P-8 | Bloco LGPD dedicado (keylog, screenshot, URL, titulo, diagnostico) | Verificado so por amostragem (`url_dominio` OK) | 1.5.4 — obrigatorio antes de release |
| P-9 | Backend fora do ar (5xx, nao so DNS) | So o cenario sem rede foi testado | 1.5.4 |
| P-10 | Desinstalacao e reinstalacao | Deixado por ultimo de proposito | **Depois de A-01** — hoje desvincula nada |
| P-11 | Status ATIVO/PAUSA/AUSENTE derivado no backend | Depende de D-01 estar implementado | Apos backend + agent 1.5.4 |

---

# Incidente do auto-update — 2026-08-18 17:19 a 17:31

A 1.5.6 foi publicada em staging e **nenhum agent conseguiu aplica-la**. Onze tentativas
seguidas, todas travando no mesmo arquivo. O agent ficou na 1.5.5 e, no fim da sequencia,
com os dois servicos parados.

Nao e problema do `srv-admin-node`: o endpoint respondeu certo em todas as tentativas.

```
17:27:12 UpdateCheckerWorker: checking. CurrentVersion=1.5.5
17:27:14 UpdateCheckerWorker: update available. NewVersion=1.5.6, Url=...1.5.6/ManagerAgent-Installer-v1.5.6.zip
17:27:27 UpdateCheckerWorker: download verified. Applying update 1.5.6.
```

Versao parseada, URL pre-assinada valida, ZIP baixado e SHA-256 conferido. O contrato de
`/api/agente/atualizacoes/verificar` foi honrado.

---

## A-39 — WorkerWatchdog relanca o worker no meio do update

**Severidade:** CRITICO. **Causa raiz do incidente.**

**Arquivos:**
- `src/ManagerAgent.Service/Resilience/WorkerWatchdog.cs`
- `src/ManagerAgent.Service/SessionManagement/WorkerLauncher.cs`
- `src/ManagerAgent.Service/Update/UpdateApplier.cs`

**Sintoma:** toda tentativa de update morre em

```
[UPDATE-A] Copying staged files...
[UPDATE-A] ERROR: O processo nao pode acessar o arquivo
           'C:\Program Files\ManagerAgent\ManagerAgent.SessionWorker.exe'
           porque ele esta sendo usado por outro processo.
[UPDATE-A] Rolling back...
[UPDATE-A] CRITICAL: rollback failed - (mesmo lock)
```

**Causa raiz:** quem segura o arquivo e um SessionWorker que o proprio agent sobe durante o
update. O `WorkerWatchdog` nao tinha nocao nenhuma de update em andamento — ele ve o pipe
cair, confirma que o processo saiu, ve a sessao ainda ativa, e relanca. E o que foi
programado para fazer. Cronologia medida na maquina:

| Instante | Fato |
|---|---|
| 17:27:27.630 | `UpdateApplier: notified all workers (UPDATE_APPLYING)` |
| 17:27:32.657 | `GOODBYE. Session=1` — o worker obedeceu e saiu |
| 17:27:33.585 | `CONNECT received. Pid=10312` — **watchdog ja relancou** |
| 17:27:37.650 | `killing survivor worker PID=10312` — o applier mata uma vez |
| 17:27:38.446 | `CONNECT received. Pid=14272` — **watchdog relanca de novo** |
| 17:27:40.573 | applier comeca a copiar; 14272 segura o `.exe` |

O mesmo aparece no log do usuario, em portugues:

```
17:29:52  Recebeu comando de parada: Update
17:29:57  Monitor da sessao 1 parou de responder. Reiniciando...
17:30:00  Monitor de atividade iniciando (versao 1.5.5.0)
17:30:02  Monitor da sessao 1 parou de responder. Reiniciando...
```

`KillSurvivorWorkers()` mata **uma vez**, antes da extracao. A extracao leva ~3s e o
watchdog tem orcamento de 3 relancamentos por 5min. Sempre perde a corrida.

**Correcao (feita):** `UpdateGate` — singleton no DI, compartilhado por `UpdateApplier`,
`WorkerLauncher` e `WorkerWatchdog`. O applier fecha o portao **antes** do broadcast de
shutdown; o launcher recusa lancar com o portao fechado. O guard fica no `WorkerLauncher`
porque ele e o ponto unico por onde todo worker nasce — `ManagerAgentService` (scan inicial
e novas sessoes) e `WorkerWatchdog` (relancamento) passam os dois por la.

O portao fecha **por tempo**, com TTL de 5min, nao por chamada de abertura. O caminho feliz
termina em `Environment.Exit(0)` e ninguem sobrevive para reabrir; se o apply morrer antes
disso, sem TTL a sessao ficaria para sempre sem worker — sem captura e sem ninguem para
religar. No caminho de falha conhecido (`finally` do applier) o portao reabre na hora.

**Aceite:**
1. Update completo sem nenhum `CONNECT received` entre o SHUTDOWN e a copia.
2. `Copying staged files` sem erro de lock.
3. Matar o worker fora de update continua relancando normalmente.
4. Testes: `UpdateGateTests` (7 casos, incluindo expiracao de TTL e recusa do launcher).

---

## A-40 — o script de update nunca matou o SessionWorker

**Severidade:** CRITICO. Independente do A-39 e igualmente necessario.

**Arquivo:** `src/ManagerAgent.Service/Update/UpdateApplier.cs` (geradores de script)

**Sintoma:** o mesmo lock do A-39, mas por um caminho que o portao nao cobre.

**Causa raiz:** o PS1 gerado para os Planos A e B para o `ManagerAgentWatchdog`, mata
`ManagerAgent.Watchdog.exe` e para o servico principal — e **nunca toca no
`ManagerAgent.SessionWorker`**. O comentario no proprio codigo explica que pararam o
Watchdog porque ele segurava file lock no proprio exe; ninguem fez o mesmo raciocinio para
o worker. Parar o Service nao mata worker: cada um e processo separado, lancado via
`WTSQueryUserToken`. O batch do Plano C tinha o mesmo buraco (`taskkill` so do Watchdog).

**Agravante:** o rollback falha pelo mesmo lock. Nas 11 tentativas deu sorte — travou no
primeiro arquivo e nada foi corrompido. Se o lock pegasse no meio da copia, a instalacao
ficaria quebrada **e** o rollback nao conseguiria desfazer.

**Correcao (feita):** bloco de kill do `ManagerAgent.SessionWorker` antes do backup, no PS1
(Planos A e B) e no batch (Plano C), com espera de ate 15s e WARN no log se sobrar alguem.
O Plano D nao precisa: ele renomeia para `.old` em vez de sobrescrever, e rename funciona
com o arquivo aberto.

---

## A-41 — o cooldown de 6h nunca rodou

**Severidade:** ALTA. E o que transformou uma falha em onze.

**Arquivos:** `src/ManagerAgent.Service/Program.cs`, `Update/UpdateCheckerWorker.cs`

**Causa raiz:** o `UpdateCheckerWorker` tem cooldown de 6h para update que falhou:

```csharp
if (File.Exists(failedFlag)) { ... await Task.Delay(CheckInterval); }
```

So que `Program.Main` roda `CheckPostUpdateFlags()` antes do host subir, e ali:

```csharp
Log.Warning("Post-update: previous update FAILED. Reason: {Reason}", reason);
File.Delete(ConfigPaths.ArquivoUpdateFailed);   // <- apaga
```

Quando o checker vai olhar, a flag ja nao existe. O cooldown era **codigo morto**. Cada
falha reiniciava o Service, `VerificarAoIniciar=true` disparava outro ciclo, e o loop
fechava. **11 tentativas em 12 minutos, cada uma baixando 285MB — cerca de 3GB de banda.**

**Correcao (feita):** `Program.Main` continua lendo a flag para o `_startupReason`, mas nao
apaga mais. Quem apaga e o checker, depois de cumprir o cooldown — senao cada restart
cobraria mais 6h e o agent nunca mais updataria.

---

## A-42 — `MaximoBackups` nao vale para o caminho que roda

**Severidade:** ALTA (era MEDIA; virou ALTA por risco de encher disco do cliente).

**Sintoma:** 11 pastas `pre-1.5.6-*` em `C:\ProgramData\ManagerAgent\backups`, **3,4 GB**.
O `appsettings.json` diz `MaximoBackups: 3`.

**Causa raiz:** `MaximoBackups` so e lido pelo `UpdateApplier` da **Tray**. O caminho que
roda de verdade e o do Service, que nunca podou nada. Mesma familia do
`AutoUpdate.Habilitado`, que **nao e lido por ninguem** — desligar auto-update pelo
appsettings nao tem efeito algum. Foi por isso que a contencao do incidente usou modo SOS,
e nao o appsettings.

**Correcao (feita):** `PruneOldBackups()` no `UpdateApplier` do Service, chamado antes de
criar o backup novo. Constante `MaxBackups = 3` espelhando o appsettings, com comentario
explicando por que nao e config.

---

## A-43 — `UpdateCheckerWorker` instanciava o WatchdogStateStore na mao

**Severidade:** MEDIA.

**Causa raiz:** `var stateStore = new WatchdogStateStore(...)` dentro de
`CheckAndApplyAsync`, com o caminho default fixo em
`C:\ProgramData\ManagerAgent\watchdog-state.json`. A interface `IWatchdogStateStore` ja
existia justamente para isso e nao estava sendo usada.

**Como apareceu:** ao ligar o modo SOS para conter o incidente, **6 testes da suite
quebraram** — eles liam o estado real da maquina.

**Correcao (feita):** store injetado por DI, registrado no `Program.cs`. Novo teste
`ExecuteAsync_SkipsCycle_WhenSosModeIsActive` cobre o branch de SOS explicitamente — antes
ele so era exercitado por acidente.

---

## A-44 — suite escrevia na pasta do agent instalado

**Severidade:** MEDIA. Parente do A-37.

**Sintoma:** 3 testes com `UnauthorizedAccessException` em
`C:\ProgramData\ManagerAgent\run-update.ps1` — arquivo do SYSTEM, deixado pelas tentativas
falhas. Os que passavam estavam mexendo no estado da maquina sob regressivo.

**Correcao (feita):** `UpdateCheckerWorkerTests` redireciona `ConfigPaths.BaseFolderOverride`
para um sandbox por classe. Como o override e estatico e o xUnit paraleliza classes, as tres
classes que tocam `ConfigPaths` (`UpdateCheckerWorkerTests`, `ConfigPathsTests`,
`ConfigManagerTests`) entraram na colecao serial `ConfigPaths`.

---

## A-45 — a correcao do auto-update nao consegue se auto-instalar

**Severidade:** ALTA. **Nao e bug de codigo — e restricao de rollout. Decisao de @Vision/@Steve.**

Quem aplica o update e o `UpdateApplier` da versao **instalada**, e o PS1 e gerado por ela.
Logo, todo agent em 1.5.5 ou 1.5.6 no campo carrega o defeito e **nao consegue aplicar a
1.5.7 enquanto houver usuario logado** — o fix so passa a valer para updates aplicados
*pela* 1.5.7 em diante.

Janela que funciona hoje: maquina **sem sessao interativa** (ninguem logado). Sem worker,
sem lock, a copia passa. Foi por isso que o defeito nao apareceu antes.

Opcoes a decidir:
1. Publicar a 1.5.7 e deixar aplicar sozinha na primeira janela com ninguem logado.
2. Reinstalar via instalador (manual ou push) nas maquinas que precisam do fix ja.

---

## Contencao aplicada em 2026-08-18 17:38

Modo SOS ligado a mao em `C:\ProgramData\ManagerAgent\watchdog-state.json`:

```json
{ "versaoAtual": "1.5.5", "sosMode": true, "ultimaVersaoQuebrada": "1.5.6" }
```

Confirmado no log:

```
[WRN] UpdateCheckerWorker: modo SOS ativo desde "2026-08-18T20:38:23Z".
      Pulando ciclo (ultima versao quebrada: 1.5.6).
[INF] Batch uploaded successfully. Events=6 ... Attempt=1
```

O `sosRecoveryHeaderCount` nunca e incrementado em codigo de producao, entao o SOS **nao sai
sozinho** — tem que ser desligado a mao quando a 1.5.7 estiver instalada. `StickyVersion` /
`StickyUntil` sao gravados pelo `RollbackOrchestrator` e **nao sao lidos por ninguem**: os
"24h sem tentar upgradar" que o comentario promete nao existem. Fica como pendencia.

---

## A-41 REVISADO — o cooldown tinha que ser por relogio de parede

**Encontrado revisando a propria correcao do A-41, antes de qualquer teste na maquina.**

A primeira versao da correcao parou de apagar a flag no `Program.Main` e passou a apaga-la
so **depois** de cumprir `await Task.Delay(6h)`. Isso troca um defeito por outro pior:

- Se o Service reiniciar dentro da janela — update, reboot, recovery do Watchdog — o
  `Task.Delay` recomeca do zero e a flag **nunca** e apagada.
- Numa maquina que reinicia todo dia, o agent entraria em cooldown eterno: **nunca mais
  atualizaria**, silenciosamente.

Trocar "o agent atualiza demais" por "o agent nunca mais atualiza" nao e conserto.

**Correcao final:** a idade vem do `LastWriteTimeUtc` da flag, decisao imediata, e a flag e
apagada assim que a janela vence — independente de quantos restarts aconteceram no meio.
Se a flag for ilegivel, trata como recem-criada e cumpre a janela inteira: errar para o
lado de esperar demais e mais seguro que reentrar no loop.

---

## A-46 — o agent aplica qualquer versao que o backend mandar

**Severidade:** ALTA. **Encontrado montando o cenario de E2E, nao na maquina.**

**Arquivo:** `src/ManagerAgent.Service/Update/UpdateCheckerWorker.cs`

O agent manda `?versaoAtual=1.5.7` e **nunca compara a resposta com a versao que roda**. Se
o backend responder `atualizacaoDisponivel: true` com `versaoNova` igual a instalada, ele
baixa e aplica de novo. Depois do restart, checa outra vez, recebe a mesma resposta, e
aplica outra vez.

Basta uma linha errada em `versoes_agente`, um cache servindo resposta velha ou um bug de
comparacao no backend para a frota inteira entrar em reinstalacao continua — centenas de MB
por ciclo, com restart de Service a cada volta. Exatamente o formato do incidente de
2026-08-18, so que disparado do lado do servidor e atingindo todos os clientes de uma vez.

Nao e hipotetico: **o primeiro rascunho do cenario E8 caiu nisso.** O stub respondia "1.5.99
disponivel" para sempre; assim que o agent aplicasse a 1.5.99, ele reaplicaria em loop. O
defeito do produto so apareceu porque escrever o teste obrigou a pensar como o backend.

**Correcao (feita):** se `versaoNova` for igual a versao corrente, loga WARN e pula o ciclo.
Comparacao por igualdade apenas — downgrade continua permitido, porque rollback dirigido
pelo backend e cenario legitimo.

**Testes:** `Does_not_download_when_the_offered_version_is_the_running_one` (checa uma vez,
nao baixa nada) e `Still_downloads_when_the_offered_version_is_different` — o segundo existe
para o guard nao ficar largo demais e bloquear update legitimo.

O stub de E2E tambem passou a ler `?versaoAtual` e responder `atualizacaoDisponivel: false`
quando o agent ja esta na versao oferecida, como o backend real faz.

---

# Regressivo automatizado — 2026-08-19, 10:50 (agent 1.5.7, DESKTOP-VMSM6LE)

Primeira rodada do harness `tests/e2e/regressivo/`. Duas camadas: o que o agent capturou
(log do worker) e o que chegou na base (staging, somente leitura).

Resultado local: passou=6 falhou=2 atencao=1 pendente=7 pulado=1.
Resultado na base: passou=10 falhou=0.

**Os eventos chegam.** As contagens da base batem com o que o agent registrou. O que a rodada
achou nao foi perda de evento — foi evento a mais, e conteudo a mais.

---

## A-47 — o titulo da janela carrega conteudo digitado

**Severidade:** ALTA. **LGPD.** Decisao de produto: @Steve + @Tony.

Registro real, colhido de `eventos_janela.titulo_janela` no banco de staging:

```
"* Possuo solidos conhecimentos em ati - Bloco de notas"
```

Isso e a primeira linha do texto que o usuario digitou. O Bloco de Notas do Windows 11
nomeia a aba nao-salva com o inicio do conteudo, e o Agent captura o titulo da janela --
comportamento legitimo e documentado.

O Agent nao le conteudo. O Windows entrega conteudo no titulo. Para o cliente e para a LGPD
o efeito e o mesmo: **senha, dado de cliente ou informacao sensivel digitada na primeira
linha de um bloco de notas nao salvo vai parar no banco.**

Nao e hipotese: esta gravado em staging agora.

Atinge qualquer app que titule pela primeira linha do conteudo (Bloco de Notas do Win11,
Notepad++ com arquivo novo, alguns editores).

**Encaminhamento:** decisao de produto sobre o tratamento (sanitizar por lista de processos,
truncar, substituir o titulo pelo nome do app em editores conhecidos). Nao implementar sem
o @Steve, porque perder titulo de editor tem custo de produto.

---

## A-48 — reinicio do worker durante ociosidade duplica o evento

**Severidade:** ALTA. **Corrompe metrica.** Dono: @Bucky, review @Tony.

Colhido no regressivo: o passo que mata o SessionWorker (cenario legitimo -- crash,
auto-update, fast user switch) produziu **duas linhas para a mesma ociosidade**:

```
id=4798  inicio=13:50:50.807Z  fim=13:58:49  dur=478s  motivo=IdleTimeout
id=4797  inicio=13:50:50.820Z  fim=-         ABERTO    motivo=IdleStarted
```

**13 milissegundos de diferenca.** A UNIQUE de `eventos_ociosidade` e
`(agente_id, iniciado_em)`, entao nao colidem e as duas entram.

Mecanismo: o worker A emite a abertura da ociosidade; o worker morre; o worker B sobe,
recalcula o inicio a partir do `GetLastInputInfo` e chega a um valor alguns ms diferente;
emite outra abertura. A linha do worker A **nunca fecha**.

Nao e caso isolado. Varredura de 7 dias na base inteira:

| Sintoma | Ocorrencias |
|---|---|
| Ociosidades abertas ha mais de 30min | **33** |
| Janelas abertas ha mais de 30min | **27** |
| Pares de ociosidade iniciados a menos de 2s um do outro | **12** (deltas de 2ms a 54ms) |

Os pares se concentram em `2026-08-18T20:35`, o horario exato do loop de update falho: cada
uma das 11 tentativas reiniciou o worker e gerou outra duplicata. O A-39/A-40 alimentava este
defeito.

**Impacto no produto:** ociosidade contada em dobro e linha que nunca fecha alimentam os
pilares Atividade e Saude -- ou seja, o Score IA sai errado.

**Correcao sugerida:** o instante de inicio da ociosidade tem que ser recuperado do estado
persistido, e nao recalculado apos restart. Alternativa complementar: arredondar o
`iniciado_em` para o segundo antes de emitir, para a UNIQUE poder fazer o trabalho dela.
Parente direto do A-35 (janela orfa), que ja tem `OrphanWindowEventCloser` -- falta o
equivalente para ociosidade.

---

## A-49 — uma linha em `agentes` por versao instalada

**Severidade:** ALTA. Dono: @Thor / @Shuri (backend de vinculacao).

A maquina de regressivo tem **12 registros** em `agentes`, um por versao, nenhum desvinculado:

```
id=39 1.1.0.0 (03/07)   id=45 1.3.7.0   id=48 1.5.1.0 (12/08)
id=40 1.1.1.0           id=46 1.3.9.0   id=50 1.5.4.0 (18/08)
id=41 1.3.0.0           id=47 1.4.0.0   id=51 1.5.7.0 (18/08)
...
```

Se valer em producao, cada atualizacao de frota infla a contagem de agentes e **fragmenta o
historico do colaborador entre varios ids** -- afeta relatorio, nao so cosmetica. Existem
`instalacao_id` e `hardware_fingerprint` justamente para deduplicar; investigar por que nao
seguram.

---

## A-50 — `IntervaloSinalVidaSegundos` do appsettings nao chega na opcao

**Severidade:** MEDIA.

`appsettings.json` da instalacao diz `"IntervaloSinalVidaSegundos": 120`. O codigo le a
propriedade (`HeartbeatService`), com default 60. O observado no log e **61s**: o valor do
arquivo nao esta sendo aplicado, o default vence.

Efeito: o dobro do trafego de batimento em relacao ao configurado. Mesma familia do
`AutoUpdate.Habilitado` e do `MaximoBackups` -- config que parece ligada e nao esta.

---

## Memoria acima do criterio do proprio roteiro

| Processo | Working set |
|---|---|
| ManagerAgent.Service | 138,1 MB |
| ManagerAgent.SessionWorker | 137,8 MB |
| ManagerAgent.Watchdog | 78,6 MB |

O `ROTEIRO-REGRESSIVO-SIMPLES` exige "cada parte abaixo de 100 MB". Dois processos passam.

Ressalva antes de tratar como regressao: `WorkingSet64` inclui paginas compartilhadas, e
build self-contained tem working set naturalmente alto. Medir `PrivateMemorySize64` e
acompanhar por algumas horas antes de concluir se e vazamento, criterio desatualizado, ou
custo real do self-contained.

---

## Defeitos do proprio harness (corrigir antes da proxima rodada)

| Problema | Efeito na rodada |
|---|---|
| Matching por nome de executavel | O agent grava o nome amigavel (`Paint`, `Notepad`, `Google Chrome`), nao `mspaint`. O R1 do Paint reprovou por comparacao errada, nao por falha do produto |
| Bloco de Notas do Win11 e por abas | `Start-Process notepad` abre uma aba no processo existente; o titulo capturado foi o da aba que o usuario ja tinha. O canario da LGPD nunca chegou a virar titulo, entao o R4 passou sem exercitar o caminho do A-47 |
| `AppActivate` sem verificar retorno | Se o foco falha, o teste segue e reprova como se fosse o produto |
| Expectativa de heartbeat | Calculada com 2min; o intervalo real e 60s (ver A-50) |
| `Set-Content -Encoding UTF8` grava BOM | Quebrou o `JSON.parse` do Node na primeira execucao da camada 2. Corrigido nos dois lados |

---

## A-35 — reavaliado: funciona; o teste e que nao exercitava

**Encerrado sem correcao de codigo (2026-08-19).**

Tres rodadas seguidas marcaram ATENCAO em "janela orfa fechada", e a leitura obvia seria que
o `OrphanWindowEventCloser` nao dispara. Nao e o caso.

O passo que mata o worker rodava DEPOIS do bloco de ociosidade. Quando a ociosidade comeca, o
`WindowActivityService` ja fecha a janela em curso (`Window event closed on flush`) e e
pausado. Logo, no instante da morte nao havia janela aberta -- e o fechador corretamente nao
fazia nada, saindo por um caminho de log Debug que nem aparece no arquivo.

Somando a isso, a assercao procurava por `OrphanWindowEventCloser|orphan window|janela orfa`,
e a mensagem real e `Evento de janela fechado por morte do worker (WorkerDied)`. Ou seja: dois
erros meus se somando para produzir um alarme sobre codigo que estava certo.

**Correcao (no teste):** novo passo **R5a**, que mata o worker com uma janela ativa em primeiro
plano, antes do bloco de ociosidade. Resultado da primeira execucao:

```
[PASSOU] R5a - 1 evento(s) fechado(s) pelo Service
```

O passo antigo (morte durante a ociosidade) deixou de afirmar coisa sobre o A-35 e passou a
verificar o **A-48**, que e o que ele de fato exercita.

---

## Rodada final — 2026-08-19, agent 1.5.8

```
passou=11  falhou=0  atencao=0  pendente=7  pulado=1     (local, 4.2 min)
passou=10  falhou=0  atencao=0  pulado=5                 (base)
```

| Item | Resultado |
|---|---|
| R1 janela ativa (charmap, Paint da Store, titulos) | PASSOU |
| R5a janela fechada na morte do worker (A-35) | PASSOU |
| R2 ociosidade | PASSOU |
| R3 heartbeat | PASSOU |
| R4 digitacao nao capturada (keylogger) | PASSOU |
| R4 titulo sem conteudo | PASSOU |
| R5 worker volta sozinho | PASSOU |
| R5 ociosidade nao duplica no restart (A-48) | PASSOU -- "inicio reaproveitado do estado persistido" |
| R7 memoria < 150MB | PASSOU |
| Base: eventos_janela / ociosidade / batimentos / LGPD | PASSOU |

**Criterio de memoria revisado para 150MB** (decisao do Marcos, 2026-08-19). O limite de 100MB
do roteiro antigo e anterior ao build self-contained, cujo working set inclui as paginas
compartilhadas do runtime embutido.

Os 7 PENDENTE continuam exigindo gente: desbloqueio de tela, logoff/login, reboot, reuniao,
suspender/acordar, sem internet e status ATIVO/PAUSA/AUSENTE (este ultimo derivado no backend
desde a v1.5.0).

---

## A-50 CORRIGIDO — a secao `Capture` inteira era letra morta

**Severidade revisada: ALTA** (era MEDIA quando parecia so o heartbeat).

O registro das opcoes no `SessionWorker/Program.cs` era um lambda **vazio**:

```csharp
services.Configure<ManagerAgentUploadOptions>(opt =>
{
    // Defaults are fine; will be overridden by config when available
});
```

O comentario diz que a config sobrescreve. Nada vinculava a configuracao. O agent sempre usou
os defaults da classe, e **a secao `Capture` do appsettings.json nao tinha efeito nenhum** --
nem o heartbeat, nem o limiar de ociosidade, nem os limites de reuniao. Qualquer valor que o
backend injetasse no instalador era decorativo.

**Correcao:** `services.Configure<ManagerAgentUploadOptions>(context.Configuration.GetSection("Capture"))`.
6 testes novos, incluindo os dois casos que impedem regressao para o outro extremo: secao
ausente mantem defaults, e campo ausente na secao nao zera o campo.

### O que muda de comportamento ao vincular

Comparando o appsettings instalado com os defaults da classe, dois valores diferem:

| Config | Efetivo ate a 1.5.8 | A partir da correcao |
|---|---|---|
| `IntervaloSinalVidaSegundos` | 60s | **120s** |
| `IntervaloResumoEntradaSegundos` | 60s | **180s** |

Os demais campos ja coincidiam. Ou seja, a frota vinha enviando **o dobro de batimentos e o
triplo de resumos de input** do que a configuracao pedia -- 114 resumos em 2h numa unica
maquina, medido em 2026-08-19.

**Atencao para @Jarvis/@Steve:** `resumos_atividade_input` alimenta os pilares. Triplicar o
intervalo muda a granularidade do dado, nao so o volume. Se 180s prejudicar algum calculo, a
decisao agora e simples -- basta editar o appsettings, que finalmente e obedecido.

---

## Modo SOS desligado — 2026-08-19

`watchdog-state.json` com `sosMode: false`, `versaoAtual: 1.5.8`. `ultimaVersaoQuebrada` fica
como registro historico.

**Verificado antes de desligar, porque havia risco real de downgrade:** a ultima versao
Windows publicada e a **1.5.6**, `ativa=true` e `obrigatoria=true`; a maquina roda 1.5.8, que
e build local nao publicado. Se o backend oferecesse a 1.5.6, seria um downgrade obrigatorio
revertendo todas as correcoes -- e o guard A-46 do Agent so bloqueia versao *igual*, nao
anterior.

Conferido no codigo do `srv-admin-node` (`isVersaoMaior`): a comparacao e SemVer e
`1.5.6 > 1.5.8` e falso, entao o backend responde `atualizacaoDisponivel: false`. Seguro.

O desligamento so vale a partir do proximo ciclo de checagem (startup do Service ou 6h).

---

## A-51 — sessao ativa fica sem worker para sempre, em silencio

**Severidade: CRITICA.** Encontrado em 2026-08-19 pelo cenario novo do plano 13.2 (multiplos
crashes do worker), na primeira execucao. **Bloqueou a publicacao da 1.5.9.**

### Medido na maquina

```
12:49:43  [ERR] WorkerWatchdog: re-launch limit reached for Session=1. Max 3 re-launches in 5 min
12:59:25        workers vivos: 0
                log do Service apos 12:49:44: VAZIO
```

Dez minutos sem worker e sem **uma linha** de log. Nao e a janela de 5min demorando: o Service
nunca mais tenta.

### Causa

```csharp
// HandleDeadWorkerAsync
_registry.Remove(sessionId);                 // remove ANTES de checar o limite
if (!IsRelaunchAllowed(sessionId)) return;   // estourou -> desiste

// CheckAllWorkersAsync, o loop periodico de 15s
var workers = _registry.GetAll();
if (workers.Count == 0) return;              // registro vazio -> nada a iterar
```

A sessao sai do registro antes da checagem do rate-limit. Quando o limite estoura, ela fica
fora do registro e o loop periodico -- que so itera o registro -- nao tem mais o que olhar. A
janela de 5min de `IsRelaunchAllowed` nunca e reavaliada porque nenhum caminho de codigo volta
a consulta-la.

Unicas recuperacoes possiveis: um evento WTS de sessao (logoff/login) ou restart do Service.

### Impacto

Se o worker de um colaborador morrer 3 vezes em 5 minutos, **a captura daquela sessao para
ate ele deslogar**. Sem erro, sem alerta, sem log. O portal mostra a pessoa como ausente
enquanto ela trabalha, e o relatorio semanal a penaliza por isso.

Nao e teorico: o auto-update mata worker, e o loop de update falho de 18/08 matou worker
repetidamente. Pode ja ter acontecido em cliente.

### Correcao

Reconciliacao no `WorkerWatchdog`: a cada ciclo de 15s, compara as sessoes que **deveriam**
ter worker (`IWtsSessionQuery.GetActiveSessionIds()`, que ja filtra sessao 0 e mantem Active,
Connected e Disconnected) com as que **tem**, e recupera a diferenca.

Nao conserta so este caso -- cobre qualquer causa que deixe uma sessao orfa.

Respeita o portao de update (A-39), para nao reabrir por outro caminho a porta que faz a copia
do update falhar, e o mesmo rate-limit, para nao virar laco de lancamento quando a sessao esta
genuinamente com problema.

6 testes novos, incluindo robustez: falha ao enumerar sessoes ou retorno nulo nao podem
derrubar o ciclo -- isso pararia tambem a checagem dos workers vivos.

---

## Achados menores da mesma rodada

**`health-check.ps1` pendura.** Medido: 135s bloqueado em I/O com 0,5s de CPU, provavelmente
numa chamada de rede que nao respeita o proprio `TimeoutSec`. Uma ferramenta de suporte que
trava e uma ferramenta que ninguem usa. O harness passou a chama-la com teto de 45s e a
reprovar o item se estourar; **a causa no script continua aberta** (dono: @Bucky).

**R3 ficou PULADO por sucesso.** Com o A-50 corrigido, o batimento passou de 60s para os 120s
configurados, e cabem menos batidas na janela de teste do que as 3 necessarias para medir o
intervalo. E sinal de que a correcao pegou, nao de defeito.

---

# 1.5.10 — bateria completa, tudo verde (2026-08-19)

Primeira versao com a bateria fechada de ponta a ponta. **Candidata a publicacao.**

## Correcoes que entram

| Achado | O que era |
|---|---|
| A-39 | WorkerWatchdog relancava worker no meio do update, segurando o lock do .exe |
| A-40 | Script de update nunca matava o SessionWorker antes de copiar |
| A-41 | Cooldown de 6h era codigo morto; depois, por relogio de parede |
| A-42 | `MaximoBackups` nao valia para o caminho que roda |
| A-46 | Agent aplicava qualquer versao que o backend mandasse, inclusive a corrente |
| A-48 | Reinicio do worker durante ociosidade duplicava o evento e deixava linha aberta |
| A-50 | Secao `Capture` do appsettings era letra morta por inteiro |
| A-51 | Sessao ativa ficava sem worker para sempre, em silencio |

Suite: **659 testes, 0 falhas**, estavel em execucoes repetidas.

## Regressivo local (22 itens)

```
passou=20  falhou=0  atencao=0  pendente=7  pulado=2
```

| Item | Resultado |
|---|---|
| R1 janela ativa (charmap, Paint da Store, titulos) | PASSOU |
| R5a janela fechada na morte do worker (A-35) | PASSOU |
| R2 ociosidade | PASSOU |
| R3 heartbeat | PASSOU |
| R4 digitacao nao capturada (keylogger) | PASSOU |
| R4 titulo sem conteudo | PASSOU |
| R5 worker volta sozinho | PASSOU |
| R5 ociosidade nao duplica no restart (A-48) | PASSOU |
| R7 memoria < 150MB | PASSOU |
| R8 buffers SQLite + fila drenando ate zero | PASSOU |
| R9 diagnostico executa e nao expoe segredo | PASSOU |
| R9 6 scripts do pacote instalados | PASSOU |
| R10 CPU 7,98% e buffer 1,8MB | PASSOU |
| **R11 rate limit dispara e a sessao VOLTA (A-51)** | **PASSOU** |

Evidencia do A-51 corrigido:

```
13:37:40  re-launch limit reached for Session=1
13:39:21  WorkerWatchdog: sessao 1 esta ativa e sem worker. Relancando (reconciliacao).
13:39:26  worker de volta
```

Na 1.5.9, a mesma sequencia deixou a sessao morta por mais de 10 minutos sem uma linha de log.

## Conferencia na base (staging)

```
passou=10  falhou=0  atencao=0
```

`eventos_janela: 8` · `eventos_ociosidade: 1` (sem duplicata) · `batimentos: 5` ·
canario nao chegou · URL so dominio · nenhum evento sem fechamento.

## Auto-update

```
[E8 PASS] update aplicado com usuario logado em 36.1s
[OK] nenhum erro de lock nem rollback
[OK] nenhum worker subiu durante o update
[OK] SessionWorker de volta
```

## Pendencias que exigem gente (nao bloqueiam)

Bloqueio/desbloqueio de tela, logoff/login, reboot, reuniao (Teams/Meet/Zoom),
suspender/acordar, sem internet, status ATIVO/PAUSA/AUSENTE (derivado no backend),
instalacao/desinstalacao pelo instalador, icone e menu da bandeja, dois usuarios
simultaneos, ciclo de vida do Service (7.x) e AutonomousBuffer (12.x).

## Aberto, fora do Agent

A-47 (titulo com conteudo -- @Steve), A-49 (12 agentes por maquina -- @Shuri),
A-27 (`publicado_em` no futuro -- @Shuri), passivo de 33 ociosidades e 27 janelas abertas
na base, e a causa do `health-check.ps1` pendurar (@Bucky).

---

# Decisoes do Marcos — 2026-08-19

## A-47 — ACEITO COMO ESTA

> "Segue como esta... erro do bloco nao nosso."

O titulo de janela pode carregar conteudo digitado porque o Bloco de Notas do Windows 11
nomeia a aba nao-salva pela primeira linha do texto. O Agent captura o titulo, que e
comportamento documentado do produto; quem coloca conteudo ali e o sistema operacional.

**Nao havera mitigacao no Agent.** Fica o registro do risco residual, para quando o assunto
voltar por auditoria, LGPD ou pergunta de cliente:

- O dado chega em `eventos_janela.titulo_janela` e fica no banco como qualquer outro titulo.
- Se um colaborador digitar senha ou dado sensivel na primeira linha de um bloco de notas
  nao salvo, isso e persistido.
- Confirmado em staging: `"* Possuo solidos conhecimentos em ati - Bloco de notas"`.
- Vale para qualquer editor que titule pela primeira linha do conteudo.

O teste automatizado `R4 - Titulo de janela sem conteudo (A-47)` continua no harness. Nao
foi removido de proposito: se um dia a decisao mudar, a verificacao ja existe.

## A-49 — @SHURI REFINA E IMPLEMENTA

Handoff em `registro/2026-08-19-handoff-shuri-backend-node.md`, secao 1. O dado que fecha o
diagnostico: 12 linhas para a mesma maquina, 12 `instalacao_id` distintos e **1** unico
`hardware_fingerprint`. O campo que existe para deduplicar e estavel e nao esta sendo usado.

Restricao de escopo ja levantada: a troca de binario in-place (1.5.7 -> 1.5.8 -> 1.5.10) **nao**
cria linha nova. O gatilho e algo que regenera o `instalacao_id`, nao "toda versao".

## A-27 — CORRIGIDO. E o diagnostico anterior estava errado

> "Vamos colocar ja mas nao pode quebrar os que nao tiverem dado hoje que ja existem la."

**A condicao esta satisfeita por construcao: nao ha migracao a fazer.**

O achado original dizia "`publicado_em` gravado 3h no futuro". Medicao de 2026-08-19,
comparando o texto cru da coluna com `now() AT TIME ZONE 'UTC'`:

```
1.0.27  gravado=2026-08-19 12:49:51.499   agora_utc=2026-08-19 17:07:46   futuro=false
1.0.26  gravado=2026-08-19 12:42:52.174   agora_utc=2026-08-19 17:07:46   futuro=false
```

**O dado no banco sempre esteve certo.** O defeito e de LEITURA: `publicado_em` e
`TIMESTAMP WITHOUT TIME ZONE`, e o driver `postgres.js` pega o texto sem fuso e aplica o do
processo (America/Sao_Paulo), devolvendo um `Date` 3h adiantado. Quem consome a API via ISO
8601 ve a publicacao no futuro.

A escrita foi medida no mesmo dia e esta correta: com `TZ=America/Sao_Paulo`, o driver
serializa `new Date()` para o texto UTC.

**Correcao:** `readUtc()` aplicado nos tres pontos de leitura do `release-admin.repository`
(`buscarPorVersaoSo`, `listar` e o `RETURNING` do `inserir`, que volta pelo mesmo caminho do
driver). Linha sem dado continua sem dado -- `readUtc(null)` devolve null.

Melhora a paridade com o Java, que devolve o valor gravado como esta.

Suite do `srv-admin-node`: 1095 passando. As 11 falhas remanescentes sao **anteriores a esta
mudanca** (medido com o patch guardado) e estao em `agent-installer-download`,
`agent-auth.service` e `swagger/conformance` -- sem relacao com release-admin.

### Nota sobre o metodo

O relato de "3h no futuro" de 18/08 e o meu de "+0,9h no futuro" de 19/08 vieram da **mesma
armadilha que o achado descreve**: ambos leram a coluna com um cliente que aplica fuso do
processo. Ao investigar timestamp `WITHOUT TIME ZONE`, comparar sempre o texto cru contra
`now() AT TIME ZONE 'UTC'` antes de concluir que o dado esta errado.
