# O rollback do auto-update nao esta ligado (2026-08-24)

> **Achado e verificacao:** @Tony | **Maquina:** DESKTOP-VMSM6LE, Agent 1.5.12
> **Origem:** item 3 de `registro/2026-08-21-linha-de-corte-producao.md`
>
> **Muda o item 3 da linha de corte.** Nao era escrever o teste — o mecanismo nao funciona.
> **Refinamento para o @Bucky:** `registro/2026-08-24-refinamento-bucky-rollback.md`.

---

## Correcao da primeira versao deste documento

A primeira redacao dizia que o `UpdateApplier` **nao cria** `bin.previous` e que o backup so
existia como `pre-*`. **Errado.** O script de update cria `bin.previous` sim, e o log da maquina
prova:

```
2026-08-21 11:45:52 [UPDATE-A] BAA backup created at C:\Program Files\bin.previous
2026-08-21 16:53:00 [UPDATE-A] BAA backup created at C:\Program Files\bin.previous
```

Eu tinha lido so o lado C# e concluido cedo demais — o mesmo erro que esta base ja documentou tres
vezes (`docs do Agent descrevem intencao, nao codigo`). O diagnostico correto esta abaixo e e
**mais estreito e mais facil de corrigir** do que o que escrevi antes.

---

## Diagnostico correto

O backup existe, esta completo e esta certo. **Quem le procura um nivel de diretorio acima do que
quem escreve usou.**

| | Caminho | Existe? |
|---|---|---|
| Script de update **escreve** | `C:\Program Files\bin.previous` | **sim** — 8 itens, `ManagerAgent.Service.exe` 1.5.11.0, criado 21/08 16:53 |
| `RollbackOrchestrator` **le** | `C:\Program Files\ManagerAgent\bin.previous` | nao, e nunca vai existir |

### De onde vem a diferenca

**Script** (`UpdateApplier.cs`, gerador do PS1):

```powershell
$parentDir = Split-Path 'C:\Program Files\ManagerAgent' -Parent   # -> C:\Program Files
$binPreviousFinal = Join-Path $parentDir 'bin.previous'           # -> C:\Program Files\bin.previous
```

**Orquestrador** (`RollbackOrchestrator.cs`):

```csharp
private const string DefaultInstallBinDir = @"C:\Program Files\ManagerAgent\bin";
var parentDir = Path.GetDirectoryName(_installBinDir);            // -> C:\Program Files\ManagerAgent
var backupBinDir = Path.Combine(parentDir, "bin.previous");       // -> ...\ManagerAgent\bin.previous
```

Os dois calculam "o pai do diretorio de instalacao". **Discordam sobre qual e o diretorio de
instalacao:** o script usa `C:\Program Files\ManagerAgent`; o C# assume
`C:\Program Files\ManagerAgent\bin`.

**O `bin\` nao existe.** Confirmado na maquina — a unica subpasta da instalacao e `scripts\`, e os
executaveis ficam na raiz:

```
PathName do servico ManagerAgent          : C:\Program Files\ManagerAgent\ManagerAgent.Service.exe
PathName do servico ManagerAgentWatchdog  : C:\Program Files\ManagerAgent\ManagerAgent.Watchdog.exe
Test-Path 'C:\Program Files\ManagerAgent\bin' -> False
```

A constante descreve um layout que o instalador nunca produziu.

---

## Consequencia hoje

3 falhas seguidas de startup pos-update -> `TryRollbackAsync` -> `Directory.Exists` no caminho
errado -> `BackupNotFound` -> `WatchdogService` liga **modo SOS**.

O SOS desliga o auto-update. **Nao devolve o Agent ao ar.** A maquina fica sem captura ate alguem
reinstalar a mao. Numa frota, e o estrago que a regra 5.2 do `REGRAS-RELEASE` existe para impedir.

---

## Nao e um defeito, sao tres

**1. Le no nivel errado** — acima.

**2. Restauraria no lugar errado, mesmo achando o backup.** O passo 5 faz
`Directory.Move(backupBinDir, _installBinDir)`, ou seja, colocaria os binarios em
`C:\Program Files\ManagerAgent\bin\` — pasta de onde nada roda. O SCM continuaria apontando para
a raiz, com os binarios quebrados intactos.

**3. Nao poderia executar o movimento de qualquer forma.** O passo 4 faz
`Directory.Move(_installBinDir, failedDir)`. Corrigidos os itens 1 e 2, `_installBinDir` passa a
ser a raiz da instalacao — que contem o **`ManagerAgent.Watchdog.exe` em execucao, que e o proprio
processo que esta rodando o rollback**. Windows nao move diretorio com executavel travado.

Isto nao e teoria: o proprio script de update documenta o mesmo obstaculo, resolvido parando o
Watchdog antes de copiar —

> *"Stop Watchdog FIRST (Item 3 fix): ManagerAgent.Watchdog.exe fica com file lock (...) Sem parar
> aqui, o Copy-Item falha com 'file being used by another process'."*

e o comentario do A-40 no mesmo script registra que **isso ja derrubou rollback antes**:

> *"(...) e o rollback falha pelo mesmo motivo, que foi o que aconteceu nas 11 tentativas de
> 2026-08-18."*

---

## Por que ninguem viu

O `e1-alt-rollback-crash.ps1` existe (141 linhas, BAA Fase 4) e **nunca rodou no CI**. Os helpers
de setup montam a estrutura `bin\` que o orquestrador espera. **O teste e o codigo concordam entre
si, e os dois discordam do instalador real.**

Os testes unitarios do `RollbackOrchestrator` passam porque usam o construtor com caminho
customizado — o de producao nunca foi exercitado.

---

## O que muda na linha de corte

O item 3 vira dois: **3a** ligar o rollback ao backup que existe (@Bucky), **3b** o teste E2E no
CI contra a instalacao real (@Bucky + @Vision). O 3b sem o 3a e teatro.

---

## Nota de higiene, nao bloqueia

`bin.previous` mora em `C:\Program Files\` — fora da pasta do produto, no raiz de Program Files.
Nome generico, em local que nao e nosso. Vale mover para dentro da instalacao ou para
`ProgramData`, mas **nao nesta rodada**: mover o backup e mudar o caminho de update, que esta
funcionando. Anotado no refinamento como decisao adiada.
