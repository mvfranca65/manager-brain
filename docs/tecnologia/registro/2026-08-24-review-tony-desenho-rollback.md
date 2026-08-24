# Review do @Tony — desenho do rollback assincrono (2026-08-24)

> **De:** @Tony | **Autor:** @Bucky | **Aprovador:** Marcos
> **Desenho:** `registro/2026-08-24-desenho-bucky-rollback-assincrono.md`
> **Refinamento:** `registro/2026-08-24-refinamento-bucky-rollback.md`

## Veredito: **APROVADO**, com uma correcao de premissa e um item que eu acrescento ao escopo

O desenho pode virar codigo. As tres perguntas dele estao respondidas na secao 3.

---

## O que eu conferi, e nao apenas li

**O Watchdog volta sozinho — confirmado.** O PS1 tem `Start-Service ManagerAgentWatchdog` no
caminho de sucesso **e** no `catch`. Os dois existem. A premissa central do desenho se sustenta.

**O recovery dos dois servicos — confirmado, e ele achou algo que eu nao tinha visto.**
`UpdateApplier` desliga recovery so do `{ServiceName}` (`failure ManagerAgent reset= 0`). O
`ManagerAgentWatchdog` fica com recovery ligado o tempo todo, e o PS1 lhe aplica
`Stop-Process -Force` como segunda camada. **Isso e um risco latente no proprio caminho de update,
hoje** — nao so no rollback. Nunca mordeu porque o `Stop-Service` limpo costuma resolver antes, e
ai o kill forcado nao encontra processo. E sorte, nao desenho.

**As duas medicoes estao certas.** `PruneOldBackups` so varre `ConfigPaths.PastaBackups` e nunca
toca `bin.previous` — conferi o corpo do metodo. A janela do `Remove-Item`/`Rename-Item` existe e o
argumento de que ela e inofensiva esta correto: naquele instante a instalacao ainda e a versao boa,
porque a copia dos arquivos novos so acontece depois. **Concordo em nao mexer.**

**O comentario que mente sobre o `.building` — confirmado.** `UpdateArtifactsCleaner` so olha
`C:\ProgramData\ManagerAgent` (`run-update.ps1`, `run-update.bat`, `updates\staged-*`). Nada nele
alcanca `C:\Program Files`. Vai para a varredura de textos como nono item.

---

## 1. Correcao de premissa — e ela reforca a conclusao dele

O desenho diz que o `watchdog-state.json` tem *"um unico dono, o `WatchdogStateStore` em C#"*.

**Nao tem.** O arquivo e escrito por **dois processos**: o `ManagerAgent.Service`
(`Program.cs`, `UpdateCheckerWorker.cs`) e o `ManagerAgent.Watchdog` (`Program.cs`,
`WatchdogService.cs`, `RollbackOrchestrator.cs`).

O que ele tem e **um unico serializador** — os dois processos passam pela mesma classe, com os
mesmos `JsonPropertyName`. E isso que sustenta o argumento, e sustenta melhor do que a versao
dele: o risco de acrescentar o PowerShell nao e virar de um para dois escritores, e sim **virar de
dois escritores que concordam para tres em que um nao compartilha o serializador**.

Corrigir a frase no codigo e nos comentarios. Premissa errada, mesmo levando a conclusao certa,
vira folclore depois.

---

## 2. Item que eu acrescento ao escopo — e digo que sou eu acrescentando

`WatchdogStateStore.Save` faz `File.WriteAllText`. **Sem temp + rename.** Com dois processos ja
escrevendo, uma queda de energia no meio da escrita deixa JSON truncado.

Consequencia: no boot seguinte o Watchdog nao consegue ler o estado e cai no default —
**`sosMode` volta a `false`**. O SOS e o freio que impede a maquina de reinstalar a versao
quebrada. Perde-se o freio exatamente na maquina que ja estava em apuros.

Isto **nao estava na linha de corte** e eu o estou colocando no escopo do 3a. Justificativa:

- o @Bucky ja vai abrir esse arquivo para gravar `rollbackDispatchedAtUtc`;
- e escrita atomica em arquivo pequeno: temp no mesmo volume + `File.Move(..., overwrite: true)`;
- o modo de falha e perder o freio de auto-update, que e o criterio B da linha de corte.

**Limite:** so o `Save`. Nao mexer no formato, nem nos campos existentes, nem no `Load`.

---

## 3. As tres perguntas

**1. Arquivo proprio de resultado em vez do `watchdog-state.json` — APROVADO.**
Um escritor (script), um leitor (Watchdog no boot), apagado depois de lido. E o mesmo padrao de
recado entre processos do `update-in-progress.flag`. Mantenha os cinco campos como estao: **so
fato, nenhuma decisao**. Se aparecer no script uma linha calculando sticky ou versao, reprovo.

**2. `rollbackDispatchedAtUtc` + folga — APROVADO, com a folga ajustada.**
O fecho esta certo e e a melhor parte do desenho: sem ele, o script que nao roda vira silencio, que
e o modo de falha que esta base ja pagou tres vezes.

**Folga: 15 minutos, nao 10.** Motivo medido: o Service tem partida atrasada e no boot de 21/08
levou **2min19s** so para subir; o ciclo do Watchdog e de 60s e o `GracePeriod` e de 3min. Dez
minutos deixa pouca margem sobre isso numa maquina lenta, e o custo de errar para menos e um SOS
falso — que desliga o auto-update de um cliente que estava bem. Errar para mais custa 5 minutos.

**3. Recovery dos dois servicos — APROVADO, e entra no escopo do 3a.**
Ele nao estava escrito no refinamento, ele achou e tem razao. Duas exigencias:

- desligar antes da troca e **restaurar no `finally`**, nao no caminho feliz. Deixar a maquina com
  recovery desligado e pior que o defeito que estamos corrigindo, e vale para os dois servicos;
- restaurar com os mesmos valores que estao la hoje, medidos na maquina:
  `reset= 86400 actions= restart/5000/restart/10000/restart/30000`. **Nao inventar numero novo.**

---

## O que continua valendo do refinamento

Nada do que ele traz muda os limites. Continuam de pe:

- `UpdateApplier` e o PS1 de update **intactos**;
- `HeartbeatMonitor`, `ScmMonitor`, criterio de 3 falhas, sticky 24h e `RecoveryOrchestrator` **nao
  se toca**;
- os quatro desfechos do `RollbackResult` continuam existindo e alcancaveis;
- testes unitarios existentes do `RollbackOrchestrator` **nao se reescreve**;
- caminho de instalacao vindo de **um unico ponto**, nunca de constante repetida — foi a duplicacao
  que causou tudo isto.

## O que vou cobrar no review do codigo

- o teste E2E falhando com a correcao desfeita — conferido por ele, como no A-63;
- nenhum teste existente reescrito sem justificativa dentro do proprio teste;
- nenhum `catch` novo que engula falha em silencio;
- o `Save` atomico com teste de que arquivo truncado nao vira `sosMode = false` silencioso;
- o caminho de recovery restaurado tambem quando o script falha no meio.

---

## Estado

**3a liberado para desenvolvimento.** O 3b depende do 3a e segue como estava.

Arvore em 1010 testes verdes, conferido hoje antes de comecar. Nada commitado.
