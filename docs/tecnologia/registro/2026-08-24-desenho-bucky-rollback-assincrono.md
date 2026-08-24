# Desenho do @Bucky — quem confirma o rollback quando o Watchdog sai do ar (2026-08-24)

> **De:** @Bucky | **Para:** @Tony (aprovar antes de eu escrever codigo)
> **Pedido em:** `registro/2026-08-24-refinamento-bucky-rollback.md`, item 3a
>
> Nada implementado. Este documento e so o desenho e as duas medicoes que voce pediu.

---

## Resposta curta

**Opcao (a), com uma mudanca: o script escreve um arquivo proprio de resultado, nao o
`watchdog-state.json`.** O Watchdog volta (o script o reinicia, como ja faz no update), le esse
arquivo, aplica o desfecho no estado dele e apaga o arquivo.

**Recusei a opcao (b)** — trocar so o Service e deixar o Watchdog na versao quebrada. Motivos
abaixo.

---

## Por que (a) funciona: o Watchdog volta sozinho

O PS1 de update ja para e religa o Watchdog, e isso esta provado em maquina no update de 21/08:

```
16:52:58  UpdateApplier: Watchdog service stop requested
16:53:06  [Watchdog] ManagerAgent Watchdog starting. Version=1.5.12.0
```

O script termina com `Start-Service -Name 'ManagerAgentWatchdog'`, e ha um segundo
`Start-Service` no `catch`, para o caminho de erro. Ou seja: **o Watchdog voltar faz parte do
padrao que ja roda em producao.** O rollback so precisa deixar o recado.

## Por que arquivo proprio e nao o `watchdog-state.json`

O `watchdog-state.json` e escrito hoje por um unico dono, o `WatchdogStateStore` em C#, com
`JsonPropertyName` em cada campo e `WhenWritingNull`. Se o PS1 passar a escrever nele:

- viram **dois escritores com serializadores diferentes** no arquivo que governa o modo SOS;
- um campo com o nome errado, ou uma escrita parcial por queda de energia, deixa o Watchdog sem
  saber se esta em SOS — e SOS e o freio do auto-update da maquina inteira;
- ha a chance de o Watchdog ainda estar vivo por alguns instantes escrevendo o mesmo arquivo.

O arquivo proprio (`rollback-result.json`) tem **um escritor so** (o script) e **um leitor so** (o
Watchdog no boot, que apaga em seguida). E o mesmo padrao de recado entre processos que ja usamos
com `update-in-progress.flag` e com a flag de falha do update.

**Conteudo minimo — so fato, nenhuma decisao:**

| Campo | Para que |
|---|---|
| `attemptedAtUtc` | quando o script rodou |
| `versaoQuebrada` | a que estava instalada antes da troca |
| `swapDone` | conseguiu trocar os arquivos? |
| `serviceRunning` | o Service subiu depois da troca? |
| `erro` | mensagem, quando houve |

**A decisao continua em C#.** O script nao calcula sticky, nao decide SOS, nao le versao. O
Watchdog, ao subir, le o arquivo, le a versao instalada e mapeia para os desfechos que **ja
existem hoje** — nenhum sai do contrato:

| Arquivo diz | `RollbackResult` | O que o Watchdog faz |
|---|---|---|
| `swapDone=true`, `serviceRunning=true` | `Success` | sticky 24h, `startupFailures = 0` |
| `swapDone=true`, `serviceRunning=false` | `RolledBackButStillFailing` | SOS |
| `swapDone=false` | `Error` | SOS |
| backup nao existia (nem chega a escrever script) | `BackupNotFound` | SOS — caso legitimo do primeiro update |

## O buraco que esse desenho abre, e como fecho

**Se o script nunca rodar** — falha do WMI, schtask que nao dispara —, nao ha arquivo de
resultado. O Watchdog voltaria, nao acharia nada e simplesmente continuaria contando falhas, sem
nunca concluir nada. Silencioso, que e o modo de falha que esta base ja pagou caro.

**Fecho gravando a intencao antes de disparar:** `rollbackDispatchedAtUtc` no
`watchdog-state.json`, escrito pelo C#, pelo dono de sempre. No boot:

- ha intencao **e** ha resultado -> aplica o desfecho e limpa os dois;
- ha intencao e **nao** ha resultado, e ja passou uma folga (proponho 10 min) -> trata como
  `Error` -> SOS. **O rollback que nao aconteceu nao passa despercebido.**

---

## Por que recusei a opcao (b) — nao parar o Watchdog

Tres motivos, e o terceiro sozinho ja mata:

1. **O `SessionWorker.exe` tambem esta travado**, por cada worker vivo. Trocar so o Service nao
   evita ter de matar processo — o PS1 de update mata os workers antes de copiar, com o comentario
   do A-40 dizendo que foi exatamente isso que derrubou o rollback nas 11 tentativas de 18/08.
   Ou seja: a opcao (b) nao me poupa o script; so me poupa uma linha dele.

2. **Ficariamos com binarios de versoes diferentes convivendo.** Ja tratamos isso como risco real
   noutro ponto: o `ShutdownReasons.VersaoMinimaDuplicateWorker` existe porque worker antigo com
   Service novo interpreta mensagem errado. Deixar Watchdog quebrado + Service antigo cria a mesma
   classe de problema, de proposito.

3. **O binario quebrado pode ser o proprio Watchdog.** Se a versao ruim quebrou o Watchdog, a
   opcao (b) nunca o conserta — e ele e quem faz o rollback. A maquina fica sem rede de seguranca
   ate alguem ir la.

---

## Uma coisa que o refinamento nao previu, e que quebraria o script

**Os dois servicos tem recovery automatico no SCM.** Medido na maquina agora:

```
ManagerAgent          : REINICIAR 5000ms / 10000ms / 30000ms
ManagerAgentWatchdog  : REINICIAR 5000ms / 10000ms / 30000ms
```

`Stop-Service` limpo nao dispara recovery. Mas o PS1 de update usa `Stop-Process -Force` como
segunda camada para o Watchdog e para os workers — e **kill forcado o SCM le como queda**. Nesse
instante o SCM pode religar o Watchdog no meio da troca, e ele volta a travar o proprio `.exe`.

O update ja resolve isso, e **so para o Service principal**:

```
UpdateApplier: SCM recovery disabled temporarily.
```

**O script de rollback tem de desligar o recovery dos DOIS antes da troca e restaurar depois**,
inclusive no caminho de erro — senao uma falha no meio deixa a maquina com recovery desligado, que
e pior que o defeito que estamos corrigindo.

---

## As duas medicoes que voce pediu

### 1. `PruneOldBackups` — nao tem relacao com o rollback

Medido no codigo: `PruneOldBackups` so varre `ConfigPaths.PastaBackups`
(`C:\ProgramData\ManagerAgent\backups`, as pastas `pre-*`). **Nunca toca `bin.previous`**, que
mora em `C:\Program Files\`. Sao dois backups independentes, com propositos diferentes — o `pre-*`
serve ao rollback dentro do proprio script de update; o `bin.previous` serve ao Watchdog.

**Nao ha a janela que voce temia por esse caminho.**

### 2. Ha uma janela, em outro lugar, e ela e inofensiva

No PS1 de update:

```powershell
if (Test-Path $binPreviousFinal) { Remove-Item -Path $binPreviousFinal -Recurse -Force }
Rename-Item -Path $binPreviousBuilding -NewName 'bin.previous' -Force
```

Entre apagar o backup antigo e renomear o novo, a maquina fica sem `bin.previous`. Dura o tempo de
um rename.

**E inofensiva, e o motivo importa:** nesse instante a instalacao **ainda e a versao antiga, que
funciona** — a copia dos arquivos novos so acontece depois. Uma queda ali deixa a maquina com um
Agent bom e sem backup, e o proximo update reconstroi o backup. Nao ha perda de caminho de volta.

**Nao mexo nisso.**

---

## Achado de lado: mais um comentario que mente

O PS1 diz:

> *"Rename atomico via .building evita bin.previous corrompido se PC desligar (...) se PC desligar
> durante Copy, .building fica orfa e Watchdog limpa"*

**O Watchdog nao limpa.** O `UpdateArtifactsCleaner` so mexe em `C:\ProgramData\ManagerAgent` —
`run-update.ps1`, `run-update.bat` e `updates\staged-*`. Nada nele olha para `C:\Program Files`.

Efeito: queda durante a copia do backup deixa `bin.previous.building` com ~160MB em
`C:\Program Files\` **para sempre**. Nao vi a pasta na maquina hoje, entao nunca aconteceu aqui.

Nao vou corrigir junto — nao segura producao e mexer no cleaner nao esta no escopo. **Registro para
entrar na varredura das mensagens defasadas**, que ja tem oito itens. Este e o nono, e o segundo
que encontro no mesmo PS1.

---

## O que peco para prosseguir

1. Aprovar o arquivo proprio de resultado em vez do `watchdog-state.json`.
2. Aprovar a `rollbackDispatchedAtUtc` + folga de 10 min como fecho do buraco do script que nao
   roda.
3. Confirmar que desligar e religar o recovery dos dois servicos entra no escopo do 3a — nao
   estava escrito.

Sem essas tres, nao comeco.
