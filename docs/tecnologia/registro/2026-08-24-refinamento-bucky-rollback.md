# Refinamento para @Bucky — rollback do auto-update (2026-08-24)

> **De:** @Tony | **Para:** @Bucky | **Aprovador:** Marcos
> **Origem:** itens 3a e 3b de `registro/2026-08-21-linha-de-corte-producao.md`
> **Diagnostico completo:** `registro/2026-08-24-rollback-nao-esta-ligado.md`
> **Repo:** `manager-srv-agent`
>
> **Regras:** nao commite, nao faca push. Ao terminar, **@Tony revisa**.

---

## Antes de comecar: o que NAO mexer

O auto-update funciona e foi exercitado ao vivo em 21/08 (1.5.11 -> 1.5.12, ~45s do download ao
Service de pe). **O defeito e so no caminho de VOLTA.**

| Area | Por que |
|---|---|
| `UpdateApplier` e o PS1 gerado | aplicam update e criam o backup corretamente. **O backup ja existe e esta certo** |
| `HeartbeatMonitor`, `ScmMonitor` | detectam a falha corretamente — o contador chega a 3 |
| Criterio de 3 falhas + sticky 24h (`WatchdogService`) | a decisao de quando reverter esta certa |
| `RecoveryOrchestrator`, modo SOS | comportamento correto para o caso em que nao ha volta possivel |
| Estrutura do `RollbackResult` e o audit `UPDATE_ROLLBACK_TRIGGERED` | contrato bom, mantenha |
| Testes unitarios existentes do `RollbackOrchestrator` | passam com o construtor de caminho customizado e continuam validos. **Nao reescreva** — o que falta e um teste do caminho de producao, nao a troca dos que existem |

**O defeito e de endereco, nao de decisao.** Quem decide reverter acerta; quem executa procura no
lugar errado, restauraria no lugar errado, e nao conseguiria executar o movimento.

---

# 3a — ligar o rollback ao backup que existe

**Severidade: ALTA.** Segura producao (criterio B da linha de corte: pode derrubar a frota sem
caminho de volta).

## Os tres defeitos, em ordem de descoberta

### Defeito 1 — le um nivel acima do que deveria

O script de update grava (log da maquina, duas vezes):

```
2026-08-21 16:53:00 [UPDATE-A] BAA backup created at C:\Program Files\bin.previous
```

O orquestrador procura:

```csharp
private const string DefaultInstallBinDir = @"C:\Program Files\ManagerAgent\bin";
var parentDir = Path.GetDirectoryName(_installBinDir);      // C:\Program Files\ManagerAgent
var backupBinDir = Path.Combine(parentDir, "bin.previous"); // ...\ManagerAgent\bin.previous
```

Os dois calculam "o pai do diretorio de instalacao" e discordam sobre qual e o diretorio de
instalacao. **`C:\Program Files\ManagerAgent\bin` nao existe** — a unica subpasta da instalacao e
`scripts\`, e os executaveis ficam na raiz. Confirmado na maquina.

### Defeito 2 — restauraria no lugar errado, mesmo achando

Passo 5: `Directory.Move(backupBinDir, _installBinDir)` colocaria os binarios em
`...\ManagerAgent\bin\`, pasta de onde nada roda. O SCM aponta para a raiz:

```
ManagerAgent          -> C:\Program Files\ManagerAgent\ManagerAgent.Service.exe
ManagerAgentWatchdog  -> C:\Program Files\ManagerAgent\ManagerAgent.Watchdog.exe
```

O Service subiria com os binarios quebrados intactos, e o rollback reportaria sucesso.

### Defeito 3 — o movimento nao pode ser feito por quem esta rodando

Passo 4: `Directory.Move(_installBinDir, failedDir)`. Corrigidos 1 e 2, `_installBinDir` passa a
ser a raiz da instalacao — **que contem o `ManagerAgent.Watchdog.exe` em execucao, o proprio
processo que executa o rollback**. Windows nao move diretorio com executavel travado.

Nao e teoria. O PS1 de update documenta o mesmo obstaculo e a solucao:

> *"Stop Watchdog FIRST (Item 3 fix): ManagerAgent.Watchdog.exe fica com file lock (...) Sem parar
> aqui, o Copy-Item falha com 'file being used by another process'."*

E o comentario do A-40, no mesmo script, registra que **isto ja derrubou rollback**:

> *"(...) e o rollback falha pelo mesmo motivo, que foi o que aconteceu nas 11 tentativas de
> 2026-08-18."*

## O que fazer

**Reuse o padrao que ja funciona: delegar a troca de arquivos a um processo externo.**

E o que o `UpdateApplier` faz — escreve um PS1, dispara via WMI e sai. O rollback tem exatamente o
mesmo problema (nao pode substituir o proprio binario em execucao) e deve ter a mesma forma. **Nao
invente um segundo mecanismo.**

Desenho pedido:

1. **Caminhos vem de `ConfigPaths`, nao de constante.** Foi a constante que criou a divergencia.
   O diretorio de instalacao e o mesmo `ConfigPaths.PastaInstalacao` que o `UpdateApplier` usa. O
   `bin.previous` e o pai desse diretorio + `bin.previous` — **calculado do mesmo jeito nos dois
   lados**. Melhor ainda: exponha o caminho num unico ponto do `Shared` e faca os dois lados
   consumirem dele, para nao poder divergir de novo.
2. **O orquestrador para de mover diretorio.** Ele passa a: verificar o backup, escrever o script
   de restauracao, dispara-lo e sair. O script faz o que o PS1 de update ja faz — para o Watchdog,
   para o Service, mata os SessionWorkers, move a instalacao quebrada para
   `bin.failed-{versao}-{timestamp}`, copia o backup por cima, sobe o Service.
3. **Preserve o que ja esta certo:** audit `UPDATE_ROLLBACK_TRIGGERED` **antes** de agir;
   `bin.failed-*` guardado para investigacao; sticky 24h; `StartupFailuresSinceLastUpdate = 0` no
   sucesso; escalada para SOS quando nao ha volta.
4. **`BackupNotFound` continua existindo e virando SOS.** E o caso legitimo do **primeiro update
   do cliente**, em que nao ha versao anterior. Nao invente fallback para ele.

### O ponto que exige desenho seu, e eu quero ver antes do codigo

**Quem confirma que o rollback deu certo, se quem sabia disso saiu do ar?**

Hoje o `TryRollbackAsync` faz `TryStartService` e le o resultado na hora — sincrono, no proprio
processo. Delegando a um script externo, o Watchdog **para junto** e nao ve o desfecho. Sem
resolver isso, some: o `RollbackResult.Success`, o sticky de 24h, o
`RolledBackButStillFailing` e a escalada para SOS. Ou seja, **os quatro desfechos que hoje
existem**.

Duas saidas plausiveis, e eu nao escolho por voce:

- **o script grava o desfecho no `watchdog-state.json`** e o Watchdog, ao subir, le e conclui
  (sticky ou SOS). Estado ja e o canal usado entre reinicios;
- **o Watchdog nao para** e so o Service e trocado, aceitando que o `ManagerAgent.Watchdog.exe`
  fique na versao quebrada ate o proximo update. Mais simples, e deixa binarios de versoes
  diferentes convivendo — precisa de argumento de que isso e seguro.

**Traga o desenho antes de escrever.** E a unica parte disto que e arquitetura.

### Ponto a medir, nao a supor

`PruneOldBackups(MaxBackups - 1)` roda **antes** de criar o backup novo. Confirme se existe janela
em que a maquina fica sem backup nenhum — se existir, um crash naquele intervalo tira o rollback
junto. **Meça no codigo e diga o que achou**; se nao houver janela, escreva isso e siga.

### Decisao adiada, de proposito

`bin.previous` mora em `C:\Program Files\` — fora da pasta do produto, nome generico no raiz de
Program Files. Deveria estar dentro da instalacao ou em `ProgramData`. **Nao mexa agora:** mover o
backup e mudar o caminho de update, que esta funcionando e acabou de ser exercitado. Fica anotado.

## Aceite do 3a

- Instalacao **real** (sem `bin\`, binarios na raiz), versao nova que sai com codigo 1 no startup:
  apos 3 ciclos, o Service volta sozinho na versao anterior e fica `Running`.
- A versao quebrada e preservada em `bin.failed-{versao}-{timestamp}`.
- `UPDATE_ROLLBACK_TRIGGERED` emitido antes de agir.
- `watchdog-state.json` com `stickyVersion` preenchido, `stickyUntil` = agora + 24h e
  `startupFailuresSinceLastUpdate = 0`.
- Sem backup nenhum (primeiro update): vai para SOS, sem travar e sem exception.
- Versao anterior tambem nao sobe: SOS, com `ultimaVersaoQuebrada` registrada.
- Rollback recente (sticky ativa): **nao** dispara de novo.

---

# 3b — o teste E2E rodando no CI

**Depende do 3a.** Sozinho, ele passaria no ambiente sintetico e continuaria mentindo — foi
exatamente isso que aconteceu.

## O que ha hoje

`tests/e2e/scenarios/e1-alt-rollback-crash.ps1`, 141 linhas, escrito na BAA Fase 4. **Nunca rodou
no CI.** Os helpers de setup montam a estrutura `bin\` que o orquestrador espera — por isso o teste
e o codigo concordam entre si e os dois discordam do instalador.

## O que fazer

1. **O cenario passa a rodar contra a instalacao gerada pelo instalador de verdade**, nao contra
   uma arvore montada pelo proprio teste. Se `install-agent-version.ps1` monta `bin\` na mao, e ele
   que muda.
2. Manter as assercoes que ja existem (versao, `bin.failed-*`, audit, `watchdog-state.json`) —
   elas estao certas.
3. Entrar no CI. Como exige servico Windows, elevacao e reboot de servico, **decida com o @Vision**
   se roda no `windows-latest` do GitHub Actions ou numa VM propria. Se nao couber no runner
   hospedado, **diga isso em vez de adaptar o teste ate caber** — um teste que passa sem exercitar
   o SCM real nos traria de volta ao ponto de partida.

## Aceite do 3b

- O cenario roda ponta a ponta contra instalacao real e passa.
- Ele **falha** se o 3a for desfeito. Confira isso do mesmo jeito que voce fez no A-63: desfaz a
  correcao, roda, ve falhar, restaura.
- Esta no CI como status check obrigatorio, ou ha registro escrito do porque nao da e qual e a
  alternativa.

---

# Ordem e limites

1. **Desenho do desfecho assincrono** — traga antes de escrever codigo
2. **3a** — caminhos por `ConfigPaths`, restauracao delegada, quatro desfechos preservados
3. **3b** — cenario contra instalacao real, depois CI com o @Vision

**Pare e me chame:** no desenho do item 1; se a medicao do `PruneOldBackups` achar janela sem
backup; se o CI nao suportar o cenario.

**Nao commite. Nao faca push.** Ao terminar, avise que o **@Tony revisa**.

**O que o review vai cobrar:**

- o teste falhando com a correcao desfeita — nao aceito "passou" sem essa conferencia
- nenhum teste existente reescrito para acomodar comportamento novo sem justificativa no proprio teste
- os quatro desfechos do `RollbackResult` ainda existindo e alcancaveis
- nenhum `catch` novo que engula falha em silencio
- caminho de instalacao vindo de um unico ponto, nao repetido em constante — foi a duplicacao que
  causou tudo isto
- o `UpdateApplier` e o PS1 de update **intactos**
