> **STATUS:** EM EXECUCAO
> **DATA:** 2026-08-18
> **DONO:** @Natasha (QA) | **EXECUTOR:** Marcos | **APOIO:** @Bucky
> **BASE:** PLANO-TESTES-REGRESSIVOS.md v2.0.0 (2026-06-11) + adicoes desta rodada

# Roteiro Regressivo — Agent Windows v1.5.x (rodada 2026-08-18)

Passo a passo executavel. Cada passo aponta os IDs `Rx.y.z` do plano oficial que voce marca.
Blocos **[NOVO]** sao adicoes desta rodada — nao existem no plano v2.0.0.

**Ambiente desta maquina (verificado):** Windows 11 26200 - PowerShell 5.1 - .NET SDK 8.0.424 - servico nao instalado - `Program Files\ManagerAgent` e `ProgramData\ManagerAgent` existem porem VAZIAS (residuo) - 66,9 GB livres.

---

## ANTES DE COMECAR — 4 decisoes

1. **Backend.** O `appsettings.json` do pacote aponta para **PRODUCAO** (`api-events.imanagerportal.com` / `api-admin.imanagerportal.com`). Se nao trocar no Configurator, os eventos desta rodada vao para prod. Defina: staging, local (:8083/:8084 — hoje **down**) ou prod com tenant de teste.
2. **Chave de ativacao** da empresa de teste (o instalador pede identificador + chave e ja vincula).
3. **Rebuild obrigatorio.** O instalador pronto e o `ManagerAgent-Setup-v1.5.1.exe` de **12/08**. Existem 2 commits depois dele (`592bd33` watchdog stale-recovery, `eb0de6e` menuVisivel) — sem rebuild, esses dois nao entram na rodada.
4. **Segundo usuario local** no Windows, se for rodar o bloco multi-sessao (Passo 12).

**Defeito conhecido em aberto (nao bloqueia):** `UpdateApplier.cs:792` tem em-dash em comentario do PS1 gerado; quebra 2 testes de guarda ASCII. Outras 4 falhas unitarias (`AgentLinkService` x3, `UpdateCheckerWorker` x1) estao em investigacao — os Passos 2 e 13 deste roteiro validam empiricamente se sao teste velho ou regressao.

---

## PASSO 0 — Preparacao (15 min)

```powershell
# 0.1 Pasta de trabalho e evidencias
New-Item -ItemType Directory -Path "C:\Temp\ManagerAgent-Tests\evidencias" -Force

# 0.2 Execution Policy
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned -Force

# 0.3 Limpar residuo das pastas vazias (evita falso positivo no R1.1.2/R1.1.7)
Remove-Item "C:\Program Files\ManagerAgent" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item "C:\ProgramData\ManagerAgent" -Recurse -Force -ErrorAction SilentlyContinue

# 0.4 Confirmar estado zero
Get-Service ManagerAgent,ManagerAgentWatchdog -ErrorAction SilentlyContinue
Get-Process -Name "ManagerAgent.*","AthenaAgent.*" -ErrorAction SilentlyContinue
```

Esperado: nada nas duas ultimas linhas.

```powershell
# 0.5 Build do pacote (nao precisa de terminal admin)
cd "C:\Users\NoisyTech\Documents\Manager\manager-srv-agent"
.\scripts\build\build-pacote-v2.ps1
# informe a chave da empresa de teste quando pedido

# 0.6 Hash do instalador gerado (anotar na evidencia)
Get-FileHash ".\instalador\ManagerAgent-Setup-v1.5.*.exe" -Algorithm SHA256
```

- [ ] **P0.1** Build conclui sem erro
- [ ] **P0.2** `.exe` gerado com a versao esperada no nome
- [ ] **P0.3** Repo limpo apos build (chave nao ficou em `appsettings.json`) — conferir com `git status`

---

## PASSO 1 — Instalacao limpa (Bloco 1 do plano)

```powershell
Start-Process "C:\Users\NoisyTech\Documents\Manager\manager-srv-agent\instalador\ManagerAgent-Setup-v1.5.1.exe" -Verb RunAs
```

No assistente: informe **identificador do colaborador** e **chave**. O instalador roda o `ManagerAgent.Configurator.exe` e ja tenta vincular.

Validar `R1.1.1` a `R1.1.15` do plano, **com estas correcoes:**

```powershell
# Dois servicos, nao um (plano v2.0.0 so cita ManagerAgent)
Get-Service ManagerAgent, ManagerAgentWatchdog | Select-Object Name,Status,StartType
# Esperado: ManagerAgent = Running / Automatic (Delayed) ; ManagerAgentWatchdog = Running / Automatic

# Recovery do SCM (plano diz 1s/5s/30s; o .iss configura 5s/10s/30s)
sc.exe qfailure ManagerAgent
sc.exe qfailure ManagerAgentWatchdog

# Processos
Get-Process -Name "ManagerAgent.Service","ManagerAgent.SessionWorker","ManagerAgent.Watchdog" -ErrorAction SilentlyContinue

# Config
$cfg = Get-Content "C:\ProgramData\ManagerAgent\config.json" | ConvertFrom-Json
$cfg | Format-List
$cfg.instalacaoId      # GUID valido
$cfg.deviceToken       # base64 DPAPI, NUNCA um JWT em claro

# Scripts empacotados: sao 6, nao 5 (plano R1.1.6 desatualizado)
Get-ChildItem "C:\Program Files\ManagerAgent\scripts"
# Esperado: health-check, monitorar-logs, coletar-diagnostico, limpar-reset, test-vinculacao, desvincular

# Multi-usuario (requisito Marcos, ausente do plano)
Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Environment" -Name DOTNET_BUNDLE_EXTRACT_BASE_DIR -ErrorAction SilentlyContinue
```

- [ ] `R1.1.1`–`R1.1.15`
- [ ] **[NOVO] P1.16** `ManagerAgentWatchdog` registrado, `start=auto`, rodando
- [ ] **[NOVO] P1.17** 6 scripts em `scripts\`
- [ ] **[NOVO] P1.18** `DOTNET_BUNDLE_EXTRACT_BASE_DIR` presente em HKLM
- [ ] **[NOVO] P1.19** `deviceToken` nao esta em plaintext

---

## PASSO 2 — Vinculacao e Device JWT **[NOVO — ausente do plano]**

```powershell
cd "C:\Program Files\ManagerAgent\scripts"
.\test-vinculacao.ps1
Get-Content "C:\ProgramData\ManagerAgent\logs\service-$(Get-Date -Format yyyyMMdd).log" |
    Select-String "vincul|link|token|refresh|401|403"
```

- [ ] **P2.1** Vinculacao concluiu (log indica `linked`)
- [ ] **P2.2** `config.json` tem `instalacaoId`, `identificadorColaborador`, `deviceToken` e `refreshToken`
- [ ] **P2.3** Config nao e salvo repetidamente em loop (olhar log — relacionado as falhas unitarias do `AgentLinkService`)
- [ ] **P2.4** Config incompleto (remover `instalacaoId` e reiniciar o Service) resulta em erro claro, nao em retry infinito
- [ ] **P2.5** Token expirado dispara refresh (401 -> refresh -> retry) sem perder eventos
- [ ] **P2.6** Dispositivo revogado no backend (403 + `X-Agent-Revoked`) faz o agent parar de enviar e logar o motivo

---

## PASSO 3 — Tray e menu (Bloco 2 do plano)

Validar `R2.1.x` a `R2.6.x`, **com estas ressalvas:**
- O submenu **Ferramentas** listado no plano (`R2.3.2`–`R2.3.6`) pode divergir: desde `eb0de6e` o menu obedece `menuVisivel` vindo de `GET /api/agente/config` por plataforma.
- O plano cita `monitorar-performance.ps1` (`R4.5`, `R5.x`): **esse script nao existe no pacote.** Use o loop manual do Passo 8.

- [ ] `R2.1.1`–`R2.1.5`, `R2.2.1`–`R2.2.7`, `R2.4.x`, `R2.5.x`, `R2.6.x`
- [ ] **[NOVO] P3.1** Menu reflete o `menuVisivel` retornado pelo backend
- [ ] **[NOVO] P3.2** Com backend offline, menu cai num default coerente (nao some nem trava)
- [ ] **[NOVO] P3.3** Estado do icone/notificacao reage em tempo real quando o Service e parado

---

## PASSO 4 — Captura de dados (Bloco 3 do plano)

```powershell
# Reset limpo
Stop-Service ManagerAgent; Remove-Item "C:\ProgramData\ManagerAgent\buffer.db*" -Force; Start-Service ManagerAgent
Start-Sleep -Seconds 10
```

Roteiro manual: Chrome em github.com (10s) -> Notepad (10s) -> VS Code com um .cs (10s) -> Win+L -> desbloquear -> 6 min sem tocar em nada -> voltar a usar.

```powershell
$db = "C:\ProgramData\ManagerAgent\buffer.db"
sqlite3 $db "PRAGMA journal_mode;"
sqlite3 $db "SELECT tipo, COUNT(*) FROM eventos GROUP BY tipo;"
sqlite3 $db "SELECT json FROM eventos ORDER BY timestamp DESC LIMIT 15;"
```

- [ ] `R3.1.1`–`R3.1.10`, `R3.2.1`–`R3.2.4`, `R3.3.1`–`R3.3.4`, `R3.4.1`–`R3.4.4`, `R3.5.1`–`R3.5.7`
- [ ] **[NOVO] P4.1** `dispositivoTipo = WINDOWS` presente no payload (multi-dispositivo, `f119d0c`)
- [ ] **[NOVO] P4.2** `sistema_operacional` preenchido
- [ ] **[NOVO] P4.3** HardwareFingerprint enviado e estavel entre reinicios (`e05ad5f`)
- [ ] **[NOVO] P4.4** Resumo de input agregado a cada 180s — **so contadores, zero conteudo** (`65a0c84`)
- [ ] **[NOVO] P4.5** Evento de reuniao ao entrar em call (Teams/Meet/Zoom) via WASAPI render por processo; confirma em ~1 min, encerra em ~2 min (`3e40767`)
- [ ] **[NOVO] P4.6** `url_dominio` traz **apenas o dominio raiz**, nunca o path
- [ ] **[NOVO] P4.7** LOGOUT explicito e SESSAO_INTERROMPIDA gerados corretamente (`a7f0f90`)
- [ ] **[NOVO] P4.8** Sleep/hibernacao: evento de sleep + flush na virada do dia (`3320121`)

---

## PASSO 5 — Transicoes de status ATIVO/PAUSA/AUSENTE **[NOVO]**

Thresholds vigentes (PR4): ATIVO < 5 min - PAUSA de 5 a 15 min - AUSENTE > 15 min.

```powershell
# ficar 20 minutos sem input, depois:
sqlite3 "C:\ProgramData\ManagerAgent\buffer.db" "SELECT json FROM eventos WHERE tipo LIKE '%status%' ORDER BY timestamp DESC LIMIT 10;"
```

- [ ] **P5.1** ATIVO -> PAUSA em ~5 min
- [ ] **P5.2** PAUSA -> AUSENTE em ~15 min (nao 30)
- [ ] **P5.3** Strings serializadas sao exatamente `ATIVO` / `PAUSA` / `AUSENTE` (nao `Online`/`Ausente`/`Inativo`)
- [ ] **P5.4** Retorno de input volta para ATIVO e fecha a janela anterior

---

## PASSO 6 — LGPD **[NOVO — bloco dedicado]**

```powershell
# Abrir um gerenciador de senhas, digitar num campo de senha, abrir um email longo. Depois:
$db = "C:\ProgramData\ManagerAgent\buffer.db"
sqlite3 $db "SELECT json FROM eventos ORDER BY timestamp DESC LIMIT 200;" > "C:\Temp\ManagerAgent-Tests\evidencias\dump-lgpd.txt"
Select-String -Path "C:\Temp\ManagerAgent-Tests\evidencias\dump-lgpd.txt" -Pattern "senha|password|@gmail|http.*/.*/"
```

- [ ] **P6.1** Nenhum conteudo digitado no buffer
- [ ] **P6.2** Nenhum screenshot em disco ou no payload
- [ ] **P6.3** Nenhuma URL com path — so dominio raiz
- [ ] **P6.4** Titulos de janela sao apenas titulo (sem corpo de mensagem/email)
- [ ] **P6.5** Logs contem so metadados, nunca conteudo de evento
- [ ] **P6.6** `deviceToken` e chave mascarados no ZIP do `coletar-diagnostico.ps1`

---

## PASSO 7 — Scripts diagnosticos (Bloco 4 do plano)

```powershell
cd "C:\Program Files\ManagerAgent\scripts"
.\health-check.ps1
.\monitorar-logs.ps1 -Filter Error      # Ctrl+C para sair
.\coletar-diagnostico.ps1
.\limpar-reset.ps1 -KeepConfig          # responder NAO na primeira vez, SIM na segunda
```

- [ ] `R4.1.1`–`R4.1.13`, `R4.2.1`–`R4.2.7`, `R4.3.1`–`R4.3.10`, `R4.4.1`–`R4.4.13`
- [ ] **[NOVO] P7.1** `health-check.ps1` verifica tambem o servico `ManagerAgentWatchdog`
- [ ] ~~`R4.5.x`~~ **REMOVIDO** — `monitorar-performance.ps1` nao existe no pacote

---

## PASSO 8 — Performance (Bloco 5 do plano, adaptado)

```powershell
$fim = (Get-Date).AddMinutes(10)
while ((Get-Date) -lt $fim) {
  Get-Process -Name "ManagerAgent.Service","ManagerAgent.SessionWorker","ManagerAgent.Watchdog" -ErrorAction SilentlyContinue |
    ForEach-Object { "{0} {1} = {2} MB / {3} threads" -f (Get-Date -Format HH:mm:ss), $_.Name, [math]::Round($_.WorkingSet64/1MB,1), $_.Threads.Count }
  Start-Sleep -Seconds 30
}
```

- [ ] `R5.1.x`, `R5.2.x`, `R5.3.x`, `R5.4.x`
- [ ] **[NOVO] P8.1** `ManagerAgent.Watchdog` < 60 MB e CPU ~0% em idle
- [ ] **[NOVO] P8.2** Upload respeita 60s / lote 100 / max 10 batches por ciclo (`appsettings.json`)

---

## PASSO 9 — Erro e recuperacao (Bloco 6 do plano)

- [ ] `R6.1.1`–`R6.1.7` (servidor offline)
- [ ] `R6.2.1`–`R6.2.4` (config invalido)
- [ ] `R6.3.1`–`R6.3.4` (buffer corrompido)
- [ ] **[NOVO] P9.1** Sem rede (desligar Wi-Fi) por 10 min: captura continua, buffer cresce, reenvio ao voltar sem duplicar

---

## PASSO 10 — Ciclo de vida + Watchdog (Blocos 7 e 13 do plano + adicoes)

```powershell
# 10.1 Kill do Service -> SCM reinicia
Stop-Process -Id (Get-Process ManagerAgent.Service).Id -Force; Start-Sleep 10; Get-Service ManagerAgent

# 10.2 Kill do SessionWorker -> Service relanca
Stop-Process -Id (Get-Process ManagerAgent.SessionWorker).Id -Force; Start-Sleep 20; Get-Process ManagerAgent.SessionWorker

# 10.3 [NOVO] Service travado (suspenso, nao morto) -> Watchdog age
Get-Content "C:\ProgramData\ManagerAgent\logs\*watchdog*.log" -Tail 40
```

- [ ] `R7.1.x` (boot), `R7.2.x` (stop graceful), `R7.3.x` (auto-restart), `R7.4.x` (restart manual)
- [ ] `R13.1.x`, `R13.2.x`, `R13.3.x` (Service vigia Worker)
- [ ] **[NOVO] P10.1** `ManagerAgentWatchdog` detecta heartbeat stale do Service e forca recovery
- [ ] **[NOVO] P10.2** Stale-recovery consolidado em `/evento` (`592bd33`) — o reporter HTTP realmente posta a auditoria
- [ ] **[NOVO] P10.3** Modo SOS entra quando o recovery falha repetidamente
- [ ] **[NOVO] P10.4** `WatchdogState` sobrevive a reboot (estado persistido)
- [ ] **[NOVO] P10.5** Matar o Watchdog: SCM reinicia em <= 5s

---

## PASSO 11 — Named Pipe e AutonomousBuffer (Blocos 9 e 12 do plano)

- [ ] `R9.1.x`, `R9.2.x`, `R9.3.x`
- [ ] `R12.1.x`, `R12.2.x`, `R12.3.x`

---

## PASSO 12 — Multi-sessao (Bloco 8 do plano) — requer 2o usuario local

- [ ] `R8.1.1`–`R8.1.5`, `R8.2.1`–`R8.2.4`

---

## PASSO 13 — Auto-update + rollback BAA (Bloco 10 do plano + adicoes)

Pre-requisito: publicar uma versao maior no backend de teste.

- [ ] `R10.1.x` (Plan A / PS1 via schtasks), `R10.2.x` (Plan B / BAT), `R10.3.x` (Plan C / rename), `R10.4.x` (falha total)
- [ ] **[NOVO] P13.1** Backup completo criado em `bin.previous` antes de aplicar (`f774606`)
- [ ] **[NOVO] P13.2** Update com binario quebrado dispara **rollback automatico pelo Watchdog** e volta a versao anterior
- [ ] **[NOVO] P13.3** Telemetria do rollback chega ao backend (`HttpWatchdogAuditReporter`)
- [ ] **[NOVO] P13.4** `UpdateArtifactsCleaner` remove artefatos velhos; `run-update.ps1` **recente (< 30 min) NAO e deletado** — valida a falha unitaria do `UpdateCheckerWorker`
- [ ] **[NOVO] P13.5** Kill switch de update funciona
- [ ] **[NOVO] P13.6** Script PS1 gerado nao tem caractere fora de ASCII — **hoje falha** (`UpdateApplier.cs:792`)

---

## PASSO 14 — Migracao V1 -> V2 (Bloco 11) — so se houver maquina com V1

- [ ] `R11.1.x`, `R11.2.x`, `R11.3.x` — senao marcar **N/A** no relatorio

---

## PASSO 15 — Desinstalacao e reinstalacao (Bloco 14 do plano) — POR ULTIMO

```powershell
# Desinstalar pelo Painel de Controle, depois:
Get-Service ManagerAgent, ManagerAgentWatchdog -ErrorAction SilentlyContinue   # esperado: vazio
Test-Path "C:\Program Files\ManagerAgent"                                       # esperado: False
Get-Process -Name "ManagerAgent.*" -ErrorAction SilentlyContinue                # esperado: vazio
```

- [ ] `R14.1.1`–`R14.1.7`, `R14.2.1`–`R14.2.4`
- [ ] **[NOVO] P15.1** `desvincular.ps1` roda no uninstall e o dispositivo fica desvinculado no backend
- [ ] **[NOVO] P15.2** Servico `ManagerAgentWatchdog` tambem e desregistrado
- [ ] **[NOVO] P15.3** `DOTNET_BUNDLE_EXTRACT_BASE_DIR` removido do HKLM

---

## Smoke test (15 min) — opcional antes da rodada longa

1. Instalar -> `ManagerAgent` e `ManagerAgentWatchdog` Running, icone na bandeja
2. Menu abre com cabecalho `iManager - vX.Y.Z`
3. Alternar 3 janelas -> eventos no `buffer.db`
4. `health-check.ps1` >= 80%
5. Service < 100 MB, Worker < 100 MB, Watchdog < 60 MB
6. Kill do SessionWorker -> relancado em <= 15s
7. Kill do Service -> SCM reinicia em <= 5s
8. Desinstalar limpo

---

## Correcoes pendentes no plano oficial v2.0.0

Aplicar no `PLANO-TESTES-REGRESSIVOS.md` depois desta rodada:

| # | Item | Acao |
|---|---|---|
| C1 | Servico `ManagerAgentWatchdog` ausente | Criar bloco proprio |
| C2 | `R1.1.6` diz 5 scripts | Sao 6 (inclui `test-vinculacao` e `desvincular`) |
| C3 | `R1.1.13` diz recovery 1s/5s/30s | O `.iss` configura 5s/10s/30s |
| C4 | `R4.5` e `R5.x` usam `monitorar-performance.ps1` | Script nao empacotado — trocar pelo loop manual |
| C5 | Vinculacao via Configurator no install | Nao coberta |
| C6 | Device JWT / refresh / revogacao | Nao coberto |
| C7 | Eventos v1.4.1–v1.4.5 | Nao cobertos |
| C8 | `dispositivoTipo` / multi-dispositivo | Nao coberto |
| C9 | `menuVisivel` por plataforma | Nao coberto |
| C10 | Thresholds ATIVO/PAUSA/AUSENTE | Nao cobertos |
| C11 | Rollback BAA / `bin.previous` | Nao coberto |
| C12 | Bloco LGPD dedicado | So existe `R3.1.10` solto |
| C13 | `desvincular.ps1` no uninstall | Nao coberto |

---

## Relatorio final

Use o template da secao "Relatorio Final" do plano oficial, acrescentando as categorias novas
(Vinculacao/JWT, Status, LGPD, Watchdog service, Rollback BAA).

**Criterio global:** >= 90%. Bloqueiam release: Service ou Watchdog nao sobem, Worker nao e lancado,
Named Pipe nao conecta, perda de dados no shutdown, vazamento LGPD (qualquer ocorrencia = critico),
rollback BAA nao restaura versao anterior, impossibilidade de desinstalar.
