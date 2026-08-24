# Roteiro de validacao da 1.5.7 — aceite do A-39 e A-40

> **Dono:** @Tony | **Executor:** Marcos na `DESKTOP-VMSM6LE` | **Data:** 2026-08-18
> **Contexto:** `ACHADOS-REGRESSIVO-2026-08-18.md`, secao "Incidente do auto-update"

---

## O que este roteiro prova

Que um agent **1.5.7 aplica um update com usuario logado**, ou seja, com um SessionWorker
vivo segurando `ManagerAgent.SessionWorker.exe`.

Esse e exatamente o cenario que falhou 11 vezes seguidas em 18/08 — e o unico que nunca
tinha sido testado, porque as maquinas sempre atualizaram com ninguem logado. Sem usuario
na sessao nao existe worker, sem worker nao existe lock, e o defeito ficava invisivel.

Suite verde e leitura de codigo **nao fecham este aceite**. So a maquina fecha.

---

## Antes de comecar

| Item | Como conferir |
|---|---|
| Sessao interativa logada | Voce na maquina, com o icone do agent na bandeja |
| PowerShell **elevado** | O roteiro mexe em servico e em `C:\ProgramData\ManagerAgent` |
| .NET 8 SDK | `dotnet --version` |
| Espaco em disco | O build gera ~300MB por versao; serao duas |

**Antes de tudo, libere os 3,4GB das tentativas falhas** (nao e obrigatorio, mas o build
precisa de espaco):

```powershell
Get-ChildItem 'C:\ProgramData\ManagerAgent\backups' -Directory |
    Sort-Object LastWriteTime -Descending |
    Select-Object -Skip 3 |
    Remove-Item -Recurse -Force
Remove-Item 'C:\ProgramData\ManagerAgent\updates\1.5.6' -Recurse -Force
```

---

## Passo 1 — Build da 1.5.7

```powershell
cd C:\Users\NoisyTech\Documents\Manager\manager-srv-agent
.\scripts\build\build-pacote-e2e.ps1 -Version 1.5.7
```

Roda a suite, publica self-contained em `dist\v1.5.7\` e valida que os EXEs reportam 1.5.7.
Restaura a versao dos `.csproj` no fim, mesmo se falhar no meio.

**Esperado:** `Pacote v1.5.7 pronto` com sha256.

---

## Passo 2 — Instalar a 1.5.7 na mao

Este salto **tem que ser manual**. E o achado A-45: quem aplica um update e o
`UpdateApplier` da versao instalada, e a 1.5.5 que esta na maquina carrega o defeito. Ela
nao consegue aplicar a 1.5.7 com voce logado — e justamente o bug que estamos corrigindo.

```powershell
.\scripts\build\install-build-local.ps1 -Version 1.5.7
```

Para os servicos, mata os workers, troca so os binarios. **`C:\ProgramData\ManagerAgent`
e preservado** — vinculacao (agente 50 / `i888777`), buffers e logs continuam.

**Esperado:** `Instalado: 1.5.7.0` e os dois servicos `Running`.

Confirme:

```powershell
(Get-Item 'C:\Program Files\ManagerAgent\ManagerAgent.Service.exe').VersionInfo.FileVersion
Get-Service ManagerAgent, ManagerAgentWatchdog | Select-Object Name, Status
```

---

## Passo 3 — Build da versao alvo do teste

```powershell
.\scripts\build\build-pacote-e2e.ps1 -Version 1.5.99 -SkipTests
```

`1.5.99` de proposito: numerica (senao o `Program.cs` marca o update como falho no
`StartsWith`) e obviamente falsa, sem risco de colidir com uma 1.5.8 real.

---

## Passo 4 — Rodar o cenario

```powershell
.\tests\e2e\scenarios\e8-update-com-usuario-logado.ps1
```

O que ele faz:

1. Confere que voce esta elevado, que a instalada e >= 1.5.7 e que **existe SessionWorker
   vivo** — sem worker ele aborta, em vez de passar por vacuo.
2. Sobe um stub local do `srv-admin` na porta 18081 oferecendo a 1.5.99.
3. Aponta so o `baseUrlAdmin` pro stub. **Os eventos continuam indo pro staging.**
4. Desliga o modo SOS temporariamente.
5. Reinicia o Service, o que dispara a checagem de startup.
6. Espera aplicar, roda as assercoes.
7. No `finally`: restaura `config.json`, restaura o SOS **como estava** e reinicia.

Ele **nao** chama o `teardown.ps1` dos cenarios E1..E7 — aquele desinstala o agent e apaga
`C:\ProgramData\ManagerAgent` inteiro, o que aqui destruiria a vinculacao.

---

## Criterios de aceite

| # | Criterio | Quem verifica |
|---|---|---|
| 1 | Versao passa de 1.5.7 para 1.5.99 em ate 180s | o proprio cenario |
| 2 | Nenhum `CONNECT received` nem `Worker launched` entre o `UPDATE_APPLYING` e o fim da copia | `assert-no-worker-launch-during-update.ps1` |
| 3 | Linha `worker launch gate closed` presente no log | idem (prova que o portao rodou) |
| 4 | Nenhum erro de lock nem rollback em `update-script.log` | `assert-no-file-lock-error.ps1` |
| 5 | Service `Running` no fim | `assert-service-running.ps1` |
| 6 | SessionWorker **volta** em ate 90s | o proprio cenario |

O criterio 6 e tao importante quanto o 2: o portao fecha por TTL de 5min. Se algo o
deixasse fechado, a sessao ficaria sem captura — falha silenciosa, pior que o bug original.

**Saida esperada:** `[E8 PASS] update aplicado com usuario logado em Ns`

---

## Se falhar

```powershell
# o que o script de update fez
Get-Content 'C:\ProgramData\ManagerAgent\logs\update-script.log' -Tail 30

# a janela critica no log do Service
Select-String -Path 'C:\ProgramData\ManagerAgent\logs\service-*.log' `
    -Pattern 'UPDATE_APPLYING|GOODBYE|CONNECT received|gate closed|Refusing to launch'
```

Leitura rapida:

| Sintoma | Significa |
|---|---|
| `CONNECT received` logo apos o `GOODBYE` | A-39 voltou: o portao nao esta segurando |
| `sendo usado por outro processo` | A-40 voltou: o script nao matou o worker |
| Nenhum `gate closed` | Binario em uso e anterior a 1.5.7 — refaca o passo 2 |
| Nao baixou nada | Stub fora do ar ou `baseUrlAdmin` nao aplicou |

---

## Depois do teste

A maquina fica na **1.5.99**, que e a 1.5.7 com outro numero — mesmo codigo. Para voltar:

```powershell
.\scripts\build\install-build-local.ps1 -Version 1.5.7
```

**Conferir o modo SOS.** O cenario restaura o estado anterior, entao se o SOS estava ligado
ele continua ligado. Enquanto estiver, o agent nao checa update nenhum, e ele **nao sai
sozinho** — `sosRecoveryHeaderCount` nunca e incrementado em codigo de producao:

```powershell
Get-Content 'C:\ProgramData\ManagerAgent\watchdog-state.json'
```

Para desligar quando a 1.5.7 estiver validada, trocar `"sosMode": true` por `false` e
reiniciar o Service.

---

## O que foi corrigido na harness para este roteiro existir

A harness de E2E (`tests/e2e/`) estava escrita mas nunca tinha rodado. Encontrado e
corrigido em 18/08:

| Problema | Efeito | Correcao |
|---|---|---|
| Stub nao servia o ZIP | `publish-fake-update.ps1` devolvia `/downloads/x.zip`, que caia no 404 do `else`. O download nunca aconteceria | rota `/downloads/*` + parametro `-ZipPath`, validada por SHA-256 |
| `Stop-Job -Force` | Parametro nao existe no PowerShell 5.1: o `finally` de **todos** os cenarios lancava excecao e o listener ficava vivo | trocado por `Remove-Job -Force`, que para e remove. `Stop-Job` sozinho trava, porque a thread do job fica bloqueada em `GetContext()` |
| Nao-ASCII em `.ps1` | **3 dos 7 cenarios nao parseavam** (`e1-alt`, `e3`, `e5`). Achado A-05 pela terceira vez: em comentario passa, dentro de string quebra | todos os 34 `.ps1` do repo limpos e parseando. Teste `PowerShellScriptsAreAsciiTests` trava a regressao no CI |
| `dist/` vazio | `publish-fake-update.ps1` e `install-agent-version.ps1` procuram la, mas o build existente escreve em `instalador\Pacote`, e termina em `Read-Host` (nao roda em cenario) | `build-pacote-e2e.ps1`, nao-interativo, com bump e restauracao de versao |
| `build-v-teste.ps1` | Chama `build-pacote-instalacao.ps1`, que **nao existe** no repo. E nao parseava (aspas nao fechadas na linha 106) | aviso no cabecalho apontando o script que funciona + linha corrigida |
| `MANAGER_UPDATE_CHECK_SECONDS` | Usado por 3 cenarios como se acelerasse a checagem. **Nao e lido por codigo nenhum** | o E8 nao usa: dispara pelo check de startup, que e real |

---

## Execucao — 2026-08-19, 09:52 (DESKTOP-VMSM6LE, Marcos logado)

**Resultado: APROVADO.** O aceite do A-39 e do A-40 esta fechado na maquina.

Sequencia registrada em `service-20260819.log`:

```
09:52:42.686  UpdateCheckerWorker: update available. NewVersion=1.5.99
09:52:44.291  UpdateCheckerWorker: download verified. Applying update 1.5.99.
09:52:44.301  UpdateApplier: starting update to version 1.5.99.
09:52:44.302  UpdateApplier: worker launch gate closed for 5min.        <- A-39
09:52:44.320  UpdateApplier: notified all workers (UPDATE_APPLYING).
09:52:49.373  GOODBYE. Session=1, Reason=Update to 1.5.99
09:52:54.713  UpdateApplier: extracting ZIP to staging dir
09:52:56.840  UpdateApplier: Plan A (WMI) launched. Exiting.
09:53:04.013  Post-update: successfully updated to version 1.5.99.
09:53:04.229  AGENT_STARTUP Version=1.5.99.0 Reason=POST_UPDATE
09:53:07.664  UpdateCheckerWorker: no update available. Agent is up to date.
09:53:08.503  CONNECT received. Session=1, Pid=6472, Version=1.5.99.0
```

E em `update-script.log`, onde a 1.5.6 morreu 11 vezes:

```
09:52:57  [UPDATE-A] Stopping SessionWorkers...       <- A-40
09:52:57  [UPDATE-A] SessionWorkers stopped.
09:52:59  [UPDATE-A] Copying staged files...
09:53:00  [UPDATE-A] Starting service...
```

### Criterios

| # | Criterio | Resultado |
|---|---|---|
| 1 | Versao 1.5.7 -> 1.5.99 | OK, em ~22s |
| 2 | Nenhum `CONNECT received` / `Worker launched` na janela critica | OK — **zero**. Na 1.5.5 eram dois, 1s e 6s apos o GOODBYE |
| 3 | Linha `worker launch gate closed` presente | OK |
| 4 | Nenhum erro de lock nem rollback | OK — `Copying staged files` seguido de `Starting service`, sem `ERROR` |
| 5 | Service Running no fim | OK |
| 6 | SessionWorker volta | OK — PID 6472 em 4s, ja na 1.5.99 |

Extras observados:

- **Aplicou uma unica vez.** `starting update to version` aparece 1x no log. O ciclo seguinte
  respondeu `no update available` — o stub ciente de versao e o guard A-46 funcionaram.
- **`killing survivor worker` nao aparece.** Na 1.5.5 era o sintoma de que o watchdog tinha
  relancado alguem; agora nao ha sobrevivente para matar.
- Ambiente restaurado pelo `finally`: stub encerrado (porta 18081 liberada), `config.json`
  devolvido e modo SOS de volta as 09:55:17.

### Estado da maquina apos o teste

Roda a **1.5.99**, que e o codigo da 1.5.7 com outro numero. Servicos `Running`, 1 SessionWorker,
modo SOS ligado (como estava antes). Para voltar o rotulo:
`.\scripts\build\install-build-local.ps1 -Version 1.5.7`.
