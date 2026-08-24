# Regressivo definitivo do Agent — roteiro unico (2026-08-24)

> **DONO:** @Natasha (QA) | **EXECUTOR:** Marcos | **MAQUINA DE REFERENCIA:** `DESKTOP-VMSM6LE`
> **SUBSTITUI E APAGA** sete documentos de teste, e marca um oitavo como historico. O de-para esta na secao 18.
> **Escrito em 24/08**, depois de conferir a maquina, o codigo dos tres repos e o backend de staging.

**Este e o unico roteiro de teste de maquina do Agent.** Se voce estava procurando um teste antigo
por numero (`C7`, `R3.1.7`, `T4`, `R10.11.8`, `P4.5`), a secao 18 diz onde ele foi parar — ou por
que saiu.

---

## 0. O que este documento e, e o que ele nao e

**E:** a lista curada do que precisa ser executado para alguem olhar o Agent e dizer "funciona".
Executado do inicio ao fim, sem sobra e sem buraco, ele fecha a liberacao de um canario.

**Nao e:** arquivo. Cenario que nao faz mais sentido **nao esta aqui** — esta na secao 19, com uma
linha dizendo por que saiu. Foram **45 cortes**. Se voce discordar de um, a linha esta la para ser
discutida; o que nao acontece e um teste sumir sem ninguem saber se foi decisao ou descuido.

### Duas regras de metodo, e elas valem para o documento inteiro

**1. Criterio descreve comportamento medido, nunca intencao.** Oito defeitos de 24/08 nasceram de
documento que afirmava o que o codigo *deveria* fazer. Onde eu **nao** conferi o caminho que
executa, esta escrito **"nao verificado"** — e "nao verificado" nao vale como criterio de
aprovacao, vale como pergunta.

**2. Sem evidencia em arquivo, o cenario nao aconteceu.** Nao aceito print sem corpo de resposta,
teste unitario verde, nem relato de que rodou.

---

## 1. As duas raias — e por que elas nao se misturam

Ha cenarios que **destroem a instalacao real**: instalam versao, sobem servidor falso, apagam a
copia de seguranca ou derrubam a captura. Rodar isso na maquina de trabalho no meio de um teste de
captura invalida os dois.

| Raia | O que e | Onde roda | Cenarios |
|---|---|---|---|
| **A — maquina de trabalho** | captura de verdade, sessao, privacidade, leitura de estado | `DESKTOP-VMSM6LE`, no expediente | 74 |
| **B — maquina dedicada** | instala versao, usa servidor falso, mexe em `bin.previous` | maquina/VM separada, ou fim de expediente com as redes de seguranca a mao | 24 |
| **C — VM descartavel** | termina com a maquina parada **por desenho** | so VM. Pedido meu, e nao mudou | 2 |

**Nunca execute B ou C entre dois cenarios da raia A.** E **ninguem roda `dotnet test` nesta
maquina enquanto qualquer cenario corre** — a suite chega a parar os servicos reais por ~17s, e o
cenario em andamento deixa de valer.

### Redes de seguranca antes de abrir a raia B

| Rede | Arquivo |
|---|---|
| Primeira | `manager-srv-agent\instalador\ManagerAgent-Setup-v1.5.16.exe` (a propria candidata) |
| Segunda | `manager-srv-agent\instalador\ManagerAgent-Setup-v1.5.13.exe` (rodou a tarde de 24/08) |

**Se `sosMode` virar `true` em qualquer momento, pare tudo.** Ele nao sai sozinho, e dali em diante
nenhum cenario mede o que promete. O unico jeito de sair e trocar para `false` no
`watchdog-state.json` e reiniciar os dois servicos.

> **Desligar o auto-update pelo `appsettings.json` nao funciona.** `AutoUpdate.Habilitado` **nao e
> lido por codigo nenhum**. O unico freio real e o modo SOS.

### Uma armadilha que ja custou uma instalacao

**Nao rode `tests\e2e\scenarios\e1-alt-rollback-crash.ps1` na maquina de trabalho.** A primeira
linha dele chama o `teardown.ps1`, que **desinstala o Agent e apaga `C:\ProgramData\ManagerAgent`
inteira** — vinculacao, buffers e logs. Ele tambem grava `MANAGER_STALE_MINUTES_OVERRIDE`
permanente na maquina e nunca remove. Foi escrito para maquina descartavel. O AT-15 faz o mesmo
teste sem destruir a instalacao.

---

## 2. Evidencia — como registrar, e onde ela ja esta

```powershell
$ev = "C:\Temp\ManagerAgent-Tests\2026-08-24"; New-Item -ItemType Directory -Path $ev -Force
```

Ao fim de cada cenario, trocando `XX-00` pelo identificador dele:

```powershell
$hoje = Get-Date -Format yyyyMMdd
Copy-Item "C:\ProgramData\ManagerAgent\watchdog-state.json"               "$ev\XX-00-state.json"
Copy-Item "C:\ProgramData\ManagerAgent\logs\service-$hoje.log"            "$ev\XX-00-service.log"
Copy-Item "C:\ProgramData\ManagerAgent\logs\session-worker-S1-$hoje.log"  "$ev\XX-00-session.log"
```

**Regra de credencial, e ela nao e formalidade.** O device token e uma credencial. Nao imprima na
tela, nao cole no chat, nao salve na pasta de evidencia. O item 4 da linha de corte existe porque
uma senha passou pelo chat duas vezes.

### O que ja tem prova — nao se refaz

| Cenario | Data | Evidencia | Situacao |
|---|---|---|---|
| CS-01 estado inicial | 24/08 14:11 | `C:\Temp\ManagerAgent-Tests\2026-08-24\C1-estado-inicial.txt` | passou (foto envelheceu; ver CS-01) |
| CS-07 reboot | 24/08 14:14 | `...\C2-reboot.txt` | passou |
| CS-06 lockscreen | 18/08 | placar 1.5.5, T4 | passou — 1 LOCK / 1 UNLOCK, zero janela de bloqueio |
| CS-13 kill do worker | 18/08 | placar 1.5.5, T2+T3 | passou — 2 kills, 0 LOGIN novo, eventos 49250/49251 fechados |
| CS-15 parada graciosa | 18/08 | placar 1.5.5, T5 | passou — `GOODBYE PendingEvents=0`, sem `killing survivor` |
| CS-17 janela de 1ms | 18/08 | placar 1.5.5, T7 | **REPROVOU** — A-30 reaberto |
| CS-18 shell sem titulo | 18/08 | placar 1.5.5, T8 | **REPROVOU** — A-31 reaberto |
| IF-01 icone e versao | 18/08 | placar 1.5.5, T1 | passou — zero `compat legado` (aparecia em 35 de 35) |
| AT-01 auto-update fim a fim | 24/08 | 4x `Reason=POST_UPDATE` no `service-20260824.log` (14:08, 14:43, 15:38, 15:58) | passou 4x, com checksum. **Prova em log corrente, nao copiada para a pasta** |
| AT-05 update com usuario logado | 19/08 09:52 | `service-20260819.log` + `update-script.log`, transcrito no anexo A | **APROVADO** — aceite do A-39/A-40 |
| AT-16 auto-recuperacao do SCM | 24/08 16:07 | `sc.exe qfailure` nos dois servicos, medido por mim | passou depois de duas restauracoes reais. **Nao salvo em arquivo** |
| AT-17 sticky | 24/08 15:58 | a 1.5.16 entrou numa maquina com sticky da 1.5.15 ativa | **parcial** — uma checagem, o cenario pede tres |
| AT-18 registro da versao quebrada | 24/08 | pastas `bin.failed-1.5.14+793e743...` e `bin.failed-1.5.15+57ae0c6...` na maquina | passou 2x, com o numero certo e nao `unknown` |
| AT-22 telemetria do Watchdog | 24/08 15:42:59 | `...\item5-telemetria.txt` — 401 as 14:56, **202** as 15:42 | passou na 1.5.15 |
| RC-01 recall de frota | 24/08 | `...\C11-recall.txt` | passou 2x (42s e 12s sem captura) |

**15 cenarios com prova: 12 aprovados, 2 reprovados, 1 parcial.**

> **Ressalva que precisa estar escrita.** A prova do AT-01 e do AT-16 vive no log corrente e na
> minha medicao, **nao em arquivo na pasta de evidencia**. Vale como prova para nao refazer hoje;
> nao vale como evidencia arquivada. Quem for fechar a release copia as duas para a pasta.

### Quando uma prova antiga deixa de valer

O criterio nao e "ja rodou": e **o codigo mudou depois?** Conferido por mim em 24/08 16:05:

```
git diff --stat 57ae0c6(1.5.15) -> 1161326(1.5.16)
  7 arquivos .csproj  (so o numero da versao)
  instalador/checksum-v1.5.15.txt
  ZERO arquivos de codigo-fonte
```

**A 1.5.16 e a 1.5.15 com outro numero.** Entre a **1.5.13** e a **1.5.16**, os unicos arquivos de
produto que mudaram sao tres, todos do R-04: `Shared/Config/ConfigManager.cs`,
`Shared/Config/LeitorConfigSomenteLeitura.cs` (novo) e
`Watchdog/Services/HttpWatchdogAuditReporter.cs`. **Nada** de SessionWorker, decider de sessao,
rollback, recall, update ou lancamento de worker.

---

## 3. Defeitos abertos — os cenarios que DEVEM reprovar hoje

**Leia antes de executar.** Sete cenarios abaixo vao reprovar, e a reprovacao e **esperada**. Quem
executar sem saber disso abre achado duplicado e manda gente cacar defeito ja mapeado.

| Cenario | O que acontece | Dono | Segura o canario? |
|---|---|---|---|
| **CS-10** | **A correcao do A-62 nao roda em producao.** O decider tem cinco respostas; o `switch` do consumidor (`SessionWorker/Capture/SessionMonitor.cs:171-198`) tem **quatro `case` e nenhum `default`**. Quando a decisao e `Emitir_FlagDeOutroLogon`, o codigo **nao faz nada e nao loga nada**. Conferido por mim no arquivo, e confirmado de forma independente pelo @Tony | @Bucky | **nao** — ver secao 4 |
| **CS-17** | Evento de janela com 1ms na retomada da ociosidade (A-30, reaberto em 18/08) | @Bucky | nao — dado sujo |
| **CS-18** | `Program Manager` vira trabalho: titulo fora da lista de placeholders (A-31, reaberto em 18/08) | @Bucky | nao — dado sujo |
| **RC-08** | **Atraso de ate 6h + ate 6h.** O Service pergunta ao backend a cada 6h; se existir `update-failed.flag` recente ele **dorme o resto da janela antes da primeira pergunta** (`UpdateCheckerWorker.cs:100-108`, `await Task.Delay(restante)`). Na pior combinacao a ordem demora quase o dobro do pior caso previsto — e a maquina que acabou de falhar um update e justamente a candidata ao recall | @Bucky | nao |
| **RC-09** | **`pausar-versao` nao tem escopo de sistema operacional.** O `PausarVersaoBodyDto` tem so `versao` e `motivo` — conferido no arquivo. Pausar a "1.5.13" pausa Windows **e** Android. O `revogar-versao` ganhou `sistemaOperacional` obrigatorio em 24/08; o `pausar-versao` **nao ganhou**, e isso esta escrito no proprio DTO do recall | @Shuri | nao |
| **BK-03** | **O health do backend nao diz qual build esta no ar.** `resolveVersion()` le `APP_GIT_SHA` e devolve `unknown` quando a variavel nao esta setada (`health.controller.ts:97-103`) — e o staging responde `version: "unknown"`. Sem isso nao da para afirmar qual commit produziu um resultado | @Vision / @Shuri | nao, **mas muda como se le uma reprovacao** — ver BK-01 |

### Dois limites de desenho que NAO sao defeito — e que enganam quem cronometra

| Limite | Por que e assim | Consequencia para quem executa |
|---|---|---|
| **Maquina em SOS nunca recebe recall** | A guarda **retorna antes da chamada ao backend** (`UpdateCheckerWorker.cs:234`, `if (estadoWatchdog is { SosMode: true }) ... return`). Conferido por mim. SOS significa "nao encoste" | Passa. Mas quem medir "quantas maquinas voltaram" **precisa contar as em SOS a parte** — senao a conta da menos que a frota e a conclusao errada e "o recall nao funcionou" |
| **O recall volta uma versao, so** | `bin.previous` guarda exatamente uma copia. Se a anterior tambem estiver ruim, o recall nao resolve | Passa. Tem de estar visivel para quem aperta o botao, nao para quem executa o teste |

### Um terceiro, que e falta de cobertura e nao defeito

**O caminho de migracao de config anterior a v1.3.0 nunca rodou em maquina, em versao nenhuma.** O
R-04 reescreveu 253 linhas do `ConfigManager.cs`, e dentro dele ha o ramo que migra token em texto
puro para DPAPI. Esse ramo so dispara em maquina que sobe de uma config velha — a maquina de
referencia ja esta migrada, entao ele **nunca vai rodar la, por mais que se reteste**. E o AT-14, e
ele vira condicao de escolha do canario (secao 4).

### Uma anomalia que conferi antes de chamar de defeito — e nao e defeito

O `watchdog-state.json` diz `versaoAtual: 1.5.13` enquanto a instalada e a 1.5.16. O campo so e
escrito dentro do proprio rollback; o auto-update nao o carimba. Logo fica velho entre um rollback
e outro, **em toda a frota**. **Nao contamina nada:** no instante do rollback o
`RollbackOrchestrator` **le a versao do executavel em disco** (`LerVersaoInstalada()`,
`RollbackOrchestrator.cs:186`) e sobrescreve o campo antes de qualquer decisao — foi por isso que
as duas voltas de 24/08 gravaram `1.5.14` e `1.5.15` corretamente. Fica registrado porque quem
abrir o arquivo para saber em que versao a maquina esta vai ler errado. Faxina com o @Bucky, fora
desta entrega.

---

## 4. O portao — o que precisa estar verde para liberar

**Verde nos seis. Nao em cinco.**

| # | Precisa estar assim | O que eu aceito como prova |
|---|---|---|
| **1** | **BK-01 aprovado** (lote com item ruim) | `R1-item1.txt` com o **corpo cru** da resposta: 202, `totalEventos=2`, uma entrada em `motivosIgnorados` com **`indice=1`** |
| **2** | **Raia A inteira aprovada**, com os criterios **deste** documento | O trecho de log de cada cenario, em arquivo |
| **3** | **Nenhum LOGOUT com dono vazio** e **nenhum banco somente-leitura na sessao 2**, em cenario nenhum | Busca no log das duas sessoes. Sao as duas regressoes que custam dado do colaborador |
| **4** | **AT-15 e AT-17 aprovados** na raia B | Sem eles a frota nao tem caminho de volta — e a regra 5.2 do `REGRAS-RELEASE` |
| **5** | **Maquina fecha o dia no estado aceitavel** | CS-01, momento 2 |
| **6** | **Empresa do canario escolhida com criterio** | Ver abaixo |

**Sobre o item 6.** O AT-14 (migracao de config anterior a v1.3.0) nao roda na maquina de referencia, e a
primeira maquina do canario e exatamente uma candidata a exercita-lo. Peco uma destas duas, e
qualquer uma serve: **(a)** a @Shuri confirmar no banco que as maquinas das empresas escolhidas ja
estao em config >= v1.3.0; **ou (b)** o Marcos aceitar por escrito que esse caminho pode ser
exercitado em producao, com o botao de pausa a mao.

**O que para o dia na hora, sem discussao de lista:** `sosMode` virar `true`; LOGOUT com dono vazio
no MU-03; banco somente-leitura na sessao 2 no MU-04. Os dois ultimos sao perda de dado do
colaborador — criterio A da linha de corte.

---

## 5. Ordem e tempo

**Raia A — 74 cenarios, ~9h30.** Sequencia: PR -> CS -> IF -> DS -> VC -> RS -> MU -> BK -> a
leitura dos cenarios de recall que nao mexem na maquina.

**Raia B — 24 cenarios, ~15h.** So depois da raia A fechada, e nunca no meio dela.

**Raia C — 2 cenarios, ~1h30, em VM.**

**Total, se alguem executar tudo: ~26 horas de trabalho**, distribuidas em pelo menos tres dias
por causa das esperas (o Watchdog exige duas confirmacoes de proposito, e nao da para encurtar).

### Se o dia for curto — a raia minima que fecha o canario

`BK-01` -> `CS-01` -> `CS-08` -> `CS-09` -> `MU-03` -> `MU-04` -> `MU-05` -> `CS-01` (fechamento).
**~2h05.** E o recorte que eu assinaria hoje, com AT-15 e AT-17 vindo em seguida na raia B.

---

# 6. CAPTURA E SESSAO — um usuario · 19 cenarios · raia A · ~4h

O bloco que prova que o dado do colaborador sai certo. Tudo aqui roda na maquina de trabalho.

**Comando de leitura padrao deste bloco** (use onde estiver escrito "leia o log"):

```powershell
$hoje = Get-Date -Format yyyyMMdd
Select-String -Path "C:\ProgramData\ManagerAgent\logs\service-$hoje.log",
                    "C:\ProgramData\ManagerAgent\logs\session-worker-S1-$hoje.log" -Pattern 'LOGOUT|LOGIN|SESSAO_INTERROMPIDA|clean-shutdown|attempting re-launch|Worker launched'
```

---

### CS-01 · Foto do estado — partida e fechamento · 5 min cada · risco nenhum
**Origem:** C1 do roteiro 1.5.13 + R0/R6 do regressivo 1.5.16 + Preparacao do PLANO
**Prova:** 24/08 14:11 (`C1-estado-inicial.txt`) — a foto envelheceu, precisa ser refeita

Nao e teste: e a foto de antes e a de depois. Sem a primeira, qualquer coisa estranha que apareca
nao tem com o que ser comparada. Sem a segunda, o dia nao fecha.

**Pre-requisito:** PowerShell como administrador.

**Passos** (identicos nos dois momentos):

```powershell
Get-Service ManagerAgent, ManagerAgentWatchdog | Select-Object Name, Status, StartType
(Get-Item 'C:\Program Files\ManagerAgent\ManagerAgent.Service.exe').VersionInfo.ProductVersion
Test-Path 'C:\Program Files\bin.previous'
(Get-Item 'C:\Program Files\bin.previous\ManagerAgent.Service.exe').VersionInfo.ProductVersion
Get-ChildItem 'C:\Program Files' -Filter 'bin.failed-*' -Directory | Select-Object Name
Get-Content 'C:\ProgramData\ManagerAgent\watchdog-state.json'
Get-Process ManagerAgent.SessionWorker | Select-Object Id, SessionId, StartTime
sc.exe qfailure ManagerAgent
sc.exe qfailure ManagerAgentWatchdog
```

**Observar:** os dois servicos `Running` com inicio automatico; **um** SessionWorker por sessao
logada; `sosMode: false`; auto-recuperacao em `5s / 10s / 30s` com reset 86400 nos **dois**
servicos.

> A foto do `sc.exe qfailure` **e o unico ponto de comparacao do AT-16**. Sem ela, o AT-16 nao tem
> como ser julgado depois. E um comando, leva 30 segundos.

**Passou se:** dois servicos `Running`, um worker por sessao, `sosMode: false`, `bin.previous`
existe **com a versao anterior dentro** (nao a instalada), SCM em 5s/10s/30s nos dois, e os ciclos
de upload fechando com `FailedBatches=0, Descartados=0`.

**Reprovou se:** `sosMode: true`; faltar um dos dois servicos; mais de um SessionWorker na mesma
sessao; ou `bin.previous` com a versao **instalada** dentro — significa que o RC-04 nao foi
desfeito, e a maquina esta sem rede de seguranca.

**Se reprovar:** **pare o roteiro.** SOS ligado significa que a maquina ja nao atualiza nem se
protege, e nenhum cenario abaixo mede o que promete. Chame o @Bucky.

---

### CS-02 · Captura de janela ativa · 10 min · risco nenhum
**Origem:** PLANO 3.1 · roteiro 08-18 PASSO 4 · marco 2.x · automatizado R1
**Prova:** nenhuma nesta versao

**Pre-requisito:** agent capturando, sessao ativa.

**Passos:** abra Chrome num site conhecido (10s), Bloco de Notas (10s) e VS Code com um arquivo
`.cs` (10s), alternando entre eles. Espere 2 min e leia o log do worker procurando
`Window event opened` e `Window event closed`.

**Observar:** o worker loga cada evento **ja serializado em JSON** — nao e preciso abrir o banco
para conferir.

**Passou se, e sao seis coisas:** os tres processos aparecem com o nome certo; o titulo da janela
vem preenchido; a duracao de um evento em que voce ficou 30s e maior ou igual a 30; o campo de
usuario traz o nome de usuario Windows; o carimbo de tempo esta em ISO 8601; e
`dispositivoTipo = WINDOWS` esta no payload.

**Reprovou se:** faltar um dos processos; titulo vazio para janela que tem titulo; duracao zerada
ou negativa; ou evento carimbado com o usuario errado.

**Se reprovar:** guarde o trecho do `session-worker-S1-*.log` com os tres eventos e mande ao
@Bucky. Nao impede os proximos.

---

### CS-03 · Deteccao de ociosidade · 15 min · risco nenhum
**Origem:** PLANO 3.2 · roteiro 08-18 PASSO 4 · automatizado R2
**Prova:** nenhuma nesta versao

**Pre-requisito:** ninguem pode tocar no mouse nem no teclado durante o bloco. Qualquer toque zera
o contador e invalida a medicao.

**Passos:** gere atividade normal por 1 min. Pare completamente por **6 minutos**. Volte a usar.
Leia o log do worker procurando `Idle event`.

**Passou se:** ha transicao para ocioso depois do limiar configurado; os eventos de janela **param**
durante a ociosidade; ha retorno registrado ao resumir; e os eventos de janela voltam.

**Reprovou se:** a ociosidade nao for detectada; eventos de janela continuarem sendo emitidos
durante o bloco parado; ou a ociosidade nao fechar na retomada.

**Se reprovar:** confirme antes que ninguem encostou na maquina — e a causa mais comum. Depois,
log para o @Bucky.

---

### CS-04 · Status ATIVO / PAUSA / AUSENTE · 25 min · risco nenhum
**Origem:** roteiro 08-18 PASSO 5 (P5.1 a P5.4)
**Prova:** nenhuma

**Pre-requisito:** 20 minutos sem tocar na maquina. **Faca junto do CS-03 e do CS-16** — e o mesmo
bloco parado, lido com tres perguntas diferentes. Separa-los seria gastar uma hora para medir
vinte minutos.

**Passos:** trabalhe normalmente por 2 min, pare por 20 min sem encostar em nada, e volte. Leia os
eventos de transicao de status.

**Observar:** os limiares vigentes (PR4) sao **ATIVO abaixo de 5 min · PAUSA de 5 a 15 min ·
AUSENTE acima de 15 min**.

**Passou se, e sao quatro coisas:** ATIVO para PAUSA em ~5 min; PAUSA para AUSENTE em ~15 min (nao
30); as strings serializadas sao exatamente `ATIVO` / `PAUSA` / `AUSENTE`; e o retorno de input
volta para ATIVO **fechando a janela anterior**.

**Reprovou se:** aparecerem as strings antigas `Online` / `Ausente` / `Inativo`; ou o AUSENTE so
chegar aos 30 min.

**Se reprovar:** e regressao do A-58 — **desde a 1.5.11 quem calcula o status e o Agent**, nao o
backend. Log para o @Bucky.

---

### CS-05 · Heartbeat e cadencia efetiva · 10 min · risco nenhum
**Origem:** PLANO 3.4 · marco 5.x · automatizado R3 (aceite do A-50)
**Prova:** nenhuma nesta versao

**Passos:** deixe a maquina capturando por 5 minutos e leia o log procurando `Heartbeat`.

**Passou se:** pelo menos 3 heartbeats em 3 minutos; o uptime cresce de forma consistente;
`versaoAgente` presente e igual a instalada; `eventosPendentes` reflete a fila; e **o intervalo
medido bate com o `appsettings`**. Este ultimo e o aceite do A-50, e existe porque a secao
`Capture` do config chegou a ser letra morta — o registro em `Program.cs` era um lambda vazio.

**Reprovou se:** o intervalo medido ignorar o configurado; ou `versaoAgente` vier vazio ou errado.

**Se reprovar:** e o A-50 de volta — a config nao esta chegando ao worker. @Bucky.

---

### CS-06 · Bloqueio e desbloqueio de tela · 10 min · risco nenhum
**Origem:** PLANO 3.3 · T4 do roteiro 1.5.5 (achado A-33) · marco 4.x
**Prova:** 18/08 — **PASSOU** (1 LOCK em 4,1s, 1 UNLOCK em 2,0s, zero janela de tela de bloqueio)

**Passos:** anote a hora. `Win+L`, espere 30s na tela de bloqueio, desbloqueie. Leia o log.

**Passou se, e sao tres coisas:** **exatamente um** LOCK e **um** UNLOCK, nao dois de cada; o
atraso entre acontecer e ser gravado fica abaixo de 5s nos dois; e **nenhuma janela de tela de
bloqueio** entra como janela ativa. Antes entravam 15 segundos de "Tela de Bloqueio padrao do
Windows" contando como trabalho.

**Reprovou se:** dois LOCK ou dois UNLOCK por ciclo; ou qualquer evento de janela com `LogonUI`,
`Windows Operating System`, ou titulo com "bloqueio"/"lock".

**Se reprovar:** e o A-33 de volta. @Bucky.

---

### CS-07 · Reboot · 15 min · risco ~2,5 min sem captura
**Origem:** C2 do roteiro 1.5.13 · PLANO 7.1
**Prova:** 24/08 14:14 (`C2-reboot.txt`) — **PASSOU na 1.5.13**, e nenhum arquivo de SessionWorker,
decider, LOGIN/LOGOUT ou lancamento de worker mudou de la para ca. **Nao refaz para o canario.**

**Pre-requisito:** feche seus programas.

**Passos:** anote a hora. Reinicie pelo menu Iniciar (reinicio normal, nao forcado). Faca login e
espere **5 minutos** sem mexer. Leia o log.

**Observar:** o Agent sobe **cerca de 2 minutos e meio depois do boot**. E configuracao — inicio
atrasado, para nao concorrer com o boot do Windows — e **nao e defeito**. Ja medido em 2min35s.

**Passou se, e sao tres coisas:** **um** LOGIN emitido — um segundo LOGIN **suprimido** com
`LogonId ja registrado` nao conta como duplicado, e a deduplicacao funcionando; os dois servicos
rodando; e nenhum evento de janela de antes do reboot continuando aberto.

> **Sobre a `SESSAO_INTERROMPIDA` num reboot limpo: ela NAO deve sair, e isso esta certo.** A sessao
> anterior alcanca gravar a flag de desligamento limpo com o `LogonId` daquele logon, e no login
> seguinte o decider compara e suprime. **Nao use este cenario para exercitar a emissao** — o
> caminho que emitiria e o CS-10, e ele esta quebrado.

**Reprovou se:** dois ou mais LOGIN **emitidos**; nenhum LOGIN; o Agent nao subir em ate 5 min; ou
sair `SESSAO_INTERROMPIDA`.

**Se reprovar:** guarde `service-*.log` e `startup-trace.log` do dia e siga para o CS-08 — nao
impede os proximos.

---

### CS-08 · Logoff e login · 10 min · risco segundos
**Origem:** C3 do roteiro 1.5.13, criterio confirmado no regressivo 1.5.16
**Prova:** nenhuma — pendente desde 21/08

**Passos:** anote a hora. Faca **logoff** (sair da conta, nao bloquear a tela). Entre de novo e
espere **2 minutos**. Leia o log.

**Passou se, e sao quatro coisas:** exatamente **um** LOGOUT; com o horario **do logoff**, nao o do
processamento; um LOGIN na volta; e **nenhum** `attempting re-launch` entre a queda do worker e o
logoff.

> **A `SESSAO_INTERROMPIDA` nao aparecer aqui esta certo, e eu conferi o porque.** No logoff limpo o
> worker grava a flag com o LogonId daquele logon; no login seguinte o decider compara e suprime.
> **E o A-62 funcionando** — no unico ramo em que ele funciona.

**Reprovou se:** nenhum LOGOUT (era o A-62); dois LOGOUT; ou `SESSAO_INTERROMPIDA` no login.

**Se reprovar:** copie a janela de log entre o logoff e o login e mande ao @Bucky.

---

### CS-09 · Parar o Service com o usuario logado · 15 min · risco ~6 min sem captura
**Origem:** C4 do roteiro 1.5.13, **com os criterios reescritos** no regressivo 1.5.16
**Prova:** nenhuma — pendente

> ## O criterio antigo deste cenario estava ERRADO, e o erro era meu.
>
> Duas afirmacoes circularam sobre o C4, e **as duas sao falsas**. Conferi no codigo e no log da
> propria maquina:
>
> **Falso 1 — "sai uma linha de LOGOUT".** Nao sai. `Stop-Service ManagerAgent` manda ao worker o
> motivo `SERVICE_STOP`, e esse motivo **nao emite LOGOUT de proposito**
> (`Shared/Pipe/ShutdownReasons.cs:24-34`): a pessoa **nao saiu**, o logon do Windows continua vivo,
> e emitir LOGOUT ali criava um LOGOUT sem LOGIN de volta. A pessoa "saia" as 14:11 e continuava
> produzindo evento ate as 15:09, e toda soma de tempo logado errava para menos.
> **Reaproveitar o LOGOUT era o defeito; nao emitir e a correcao.**
>
> **Falso 2 — "e o jeito de exercitar a `SESSAO_INTERROMPIDA`".** Nao e. No `SERVICE_STOP` o worker
> **grava sim** a flag de desligamento limpo, e no start seguinte o decider **suprime**.
>
> **Nao e teoria.** Aconteceu nesta maquina as 15:57:34 de 24/08, e a linha esta no
> `session-worker-S1-20260824.log`: `SHUTDOWN sem fim de sessao (Motivo=SERVICE_STOP)...`, seguida
> 5 segundos depois de `SESSAO_INTERROMPIDA suprimida — clean-shutdown flag presente`.
>
> **Executado com o criterio antigo, este cenario reprova um produto que esta certo** e manda o
> @Bucky cacar defeito que nao existe.

**Passos:** anote a hora e confirme o icone na bandeja. `Stop-Service ManagerAgent`. Espere **1
minuto** e confirme que parou. `Start-Service ManagerAgent`. Espere **2 minutos** e leia o log.

**Passou se, e sao cinco coisas:**
1. Sai a linha `SHUTDOWN sem fim de sessao (Motivo=SERVICE_STOP)`.
2. **NENHUM LOGOUT** e emitido para a sua sessao. Sair LOGOUT aqui e **reprovacao** — e a volta do
   defeito que o A-65 corrigiu.
3. Sai `clean-shutdown flag persistida`, e `C:\ProgramData\ManagerAgent\data\clean-shutdown-1.json`
   fica com carimbo novo.
4. Na volta, `SESSAO_INTERROMPIDA suprimida — clean-shutdown flag presente`. **Nao emitida.**
5. Na volta, o LOGIN e **suprimido** com `LogonId ja registrado`, porque e o mesmo logon. Isso e
   dedup correto, nao LOGIN faltando.

**Reprovou se:** aparecer LOGOUT; a flag nao for gravada; `SESSAO_INTERROMPIDA` for **emitida**; ou
o Service nao voltar.

**Se reprovar:** `Start-Service ManagerAgent` resolve o servico. Guarde o log e siga.

> **Sobre o tempo:** se voce **nao** subir o servico na mao, o Watchdog leva ~**6 minutos** (medido:
> 5min21s). Ele exige duas confirmacoes antes de agir, para nao reagir a falso alarme. **Nao e
> defeito.** Para exercitar tambem esse caminho, pule o `Start-Service` e espere.

---

### CS-10 · Queda de energia — emissao de SESSAO_INTERROMPIDA · 15 min · risco medio · **DEVE REPROVAR**
**Origem:** cenario novo, criado a partir do achado N-01 (secao 3)
**Prova:** nenhuma — e o **unico** cenario que exercita a emissao

**Por que existe.** Nenhum outro cenario do roteiro exercita a emissao da `SESSAO_INTERROMPIDA`.
Quando fui procurar qual exercitaria, encontrei o defeito. Conferi as tres pontas:

1. `Emitir_FlagDeOutroLogon` aparece em `src\` **so no proprio decider** — definicao e `return`.
   Nenhum consumidor no produto inteiro o trata.
2. Os dois testes que cobrem o caso (`FlagPorLogonIdTests.cs:48` e `:130`) chamam `Decide()`
   **direto** e nunca passam pelo `SessionMonitor`. Por isso a suite esta verde. Um deles se chama
   `Reboot_com_flag_de_outro_logon_ainda_reporta_a_interrupcao` — e ele **nao** reporta.
3. `ClearAsync`, a funcao que apagaria a flag, existe, tem teste, e **nao tem um unico chamador no
   produto**. A flag nunca e apagada: fica no disco carregando o LogonId de um logon velho.

Junte o ponto 3 com o defeito e a queda de energia — o caso que o evento existe para marcar — cai
justamente no ramo morto:

| Momento | Estado |
|---|---|
| Pessoa encerra o dia direito (logon L1) | flag gravada com **L1** |
| Dia seguinte, entra (logon L2) | LOGIN sai; anterior passa a **L2**; a flag continua **L1** |
| **Cai a energia no meio do expediente** | ninguem grava flag nenhuma |
| Pessoa liga e entra (logon L3) | compara flag **L1** com anterior **L2**, diferentes, `Emitir_FlagDeOutroLogon`, **silencio** |

O ramo `Emitir`, que funciona, so e alcancado quando **nao existe flag nenhuma** no disco — maquina
nova, ou que nunca teve um desligamento limpo. **Na frota real, e quase inalcancavel.**

**Pre-requisito:** faca **por ultimo** no dia. Desligamento sujo pode corromper o buffer da sessao.

**Passos:** com a sessao aberta, **reset na marra** (segure o botao de forca). Religue e entre.

```powershell
$hoje = Get-Date -Format yyyyMMdd
Select-String -Path "C:\ProgramData\ManagerAgent\logs\session-worker-S1-$hoje.log" -Pattern 'SESSAO_INTERROMPIDA suprimida|Detectada sessao interrompida'
```

**O resultado que eu PREVEJO, pela leitura do codigo: nenhuma das duas linhas.** Silencio total —
nem emissao, nem supressao.

**"Passou" hoje significa reprovar:** o silencio **confirma o achado**, e nao e falha do teste.
Anexe ao chamado do @Bucky e siga.

**Se sair `Detectada sessao interrompida`:** **a minha leitura do codigo esta errada** e eu quero
saber no mesmo dia. Me chame antes de concluir qualquer coisa.

> **O que NAO aceito como equivalente:** apagar o `clean-shutdown-1.json` na mao e reiniciar. Isso
> cai no ramo `Emitir` (flag ausente), que **funciona** — voce provaria o caminho que nao esta
> quebrado e concluiria que esta tudo bem. Se fizer assim mesmo, registre como **"ramo de flag
> ausente conferido"**, nunca como CS-10 executado.

**Nota para quem for corrigir:** o comportamento observavel e **identico ao da 1.5.12, que ja esta
em producao** — nao e regressao. O que e novo e acreditarmos que foi consertado. A correcao e um
`case` faltando, mas quem escreveu o `switch` incompleto ja tinha teste verde: **a correcao precisa
vir com teste no `SessionMonitor`**, nao so no decider, senao repetimos o mesmo erro.

---

### CS-11 · Suspender e acordar · 15 min · risco alguns minutos
**Origem:** roteiro 08-18 P4.8 · roteiro simples secao 3
**Prova:** nenhuma

**Passos:** trabalhe por 2 min, coloque a maquina para dormir, espere 5 min, acorde e use por 2 min.
Leia o log.

**Passou se:** o periodo dormindo e registrado; **nenhuma hora trabalhada e inventada** no
intervalo; e a captura retoma sozinha ao acordar, com flush na virada.

**Reprovou se:** o intervalo dormindo aparecer como trabalho continuo; ou a captura nao retomar sem
intervencao.

**Se reprovar:** e o pior tipo de defeito de dado — tempo a mais no relatorio de alguem. Log para o
@Bucky **e** @Tony.

---

### CS-12 · Virada do dia · 5 min de leitura · risco nenhum
**Origem:** PLANO 13.3 · marco 8.x
**Prova:** nenhuma

**Oportunista, nao vigilia.** Nao fique acordado ate meia-noite: deixe a maquina ligada e leia no
dia seguinte.

**Passos:** na manha seguinte, liste `C:\ProgramData\ManagerAgent\logs` e confira os arquivos dos
dois dias.

**Passou se:** arquivo de log novo criado na virada; o do dia anterior fechado corretamente;
sessoes de captura em curso **nao interrompidas** pela virada; e os eventos do novo dia com carimbo
de tempo correto.

**Reprovou se:** o log do dia anterior ficar aberto ou truncado; ou uma sessao morrer na virada.

**Se reprovar:** guarde os dois arquivos de log inteiros — o defeito esta na fronteira e some se
voce so copiar o trecho.

---

### CS-13 · Matar o worker: a janela fecha e o LOGIN nao duplica · 10 min · risco segundos
**Origem:** T2 + T3 do roteiro 1.5.5 (achados A-34 e A-35) · automatizado R5a
**Prova:** 18/08 — **PASSOU nos dois criterios**

Duas verificacoes na mesma acao. Separa-las era gastar o tempo duas vezes.

**Passos:**

```powershell
notepad
Start-Sleep -Seconds 70
Get-Date -Format 'HH:mm:ss'
Stop-Process -Name ManagerAgent.SessionWorker -Force
Start-Sleep -Seconds 20
Stop-Process -Name ManagerAgent.SessionWorker -Force
Start-Sleep -Seconds 20
Get-Process ManagerAgent.SessionWorker | Select-Object Id, SessionId, StartTime
```

**Passou se, e sao quatro coisas:**
1. Exatamente **um** worker vivo, com PID novo.
2. **Nenhum LOGIN novo** — o log traz `LOGIN suprimido — LogonId ja registrado` nas duas vezes.
   Era o A-34: 35 reinicios geravam 35 LOGINs falsos.
3. O evento do Bloco de Notas **fecha** com a hora do kill, tolerancia 1s, e o log do Service traz
   `Evento de janela fechado por morte do worker (WorkerDied)`. Era o A-35.
4. No maximo **uma** janela sem fechamento — a que esta genuinamente em curso agora.

**Reprovou se:** LOGIN novo emitido a cada kill; ou janela ficando aberta para sempre.

**Se reprovar:** e A-34 ou A-35 de volta. @Bucky.

---

### CS-14 · Parada graciosa: o worker sai sozinho, sem icone fantasma · 10 min · risco segundos
**Origem:** T5 do roteiro 1.5.5 (achados A-29 e A-36)
**Prova:** 18/08 — **PASSOU** (`GOODBYE PendingEvents=0`, EOF 71ms depois, sem `killing survivor`)

**Por que existe:** `StopApplication()` parava os hosted services mas o loop do WinForms seguia
bloqueado. O processo nunca saia sozinho, era morto de fora, e o icone ficava desenhado na bandeja.

**Passos:** anote a hora. `Restart-Service ManagerAgent`. Olhe a bandeja **sem passar o mouse por
cima** e leia o log.

**Passou se, e sao tres coisas:** o worker antigo **desaparece sozinho** — no log, `SHUTDOWN`
seguido de `goodbye`, **sem** `killing survivor worker`; a bandeja fica com **um** icone, nao dois;
e o LOGOUT chega em menos de 5s.

**Reprovou se:** aparecer `killing survivor worker`; ou sobrar icone fantasma que so some ao passar
o mouse.

**Se reprovar:** e o A-29 de volta. @Bucky.

---

### CS-15 · Evento de reuniao · 10 min · risco nenhum
**Origem:** roteiro 08-18 P4.5 · roteiro simples secao 3
**Prova:** nenhuma

**Pre-requisito:** Teams, Meet ou Zoom instalado, e uma reuniao de verdade. A deteccao e por WASAPI
render por processo — nao da para simular.

**Passos:** entre numa reuniao, fique 3 minutos, saia.

**Passou se:** o inicio e confirmado em ~1 min; o fim e registrado em ~2 min; e sai **uma** linha de
reuniao, nao duas — o A-57 duplicava.

**Reprovou se:** a reuniao nao for detectada; ou virar duas linhas. Cruze com o BK-04.

**Se reprovar:** anote o app e o horario exato. @Bucky, e @Shuri se a duplicata so aparecer no banco.

---

### CS-16 · Janela de 1ms na retomada da ociosidade · 15 min · **DEVE REPROVAR**
**Origem:** T7 do roteiro 1.5.5 (achado A-30)
**Prova:** 18/08 — **REPROVOU.** Evento 49253 com 1ms as 14:48:26, worker subindo dentro da
ociosidade 4664. **A-30 reaberto.**

**Faca junto do CS-03 e do CS-04** — e o mesmo bloco de ociosidade, lido com outra pergunta.

**Passos:** apos ~10 minutos parado, mexa e abra uma janela nova. Procure eventos de janela com
duracao abaixo de 1 segundo no instante da retomada.

**Passou se:** nenhum evento de janela com duracao de milissegundos no instante da retomada.

**Reprovou se — e e o esperado hoje:** evento de duracao ~1ms colado na retomada.
**Registre como confirmacao do A-30, nao como achado novo.**

---

### CS-17 · Shell sem titulo vira trabalho · 10 min · **DEVE REPROVAR**
**Origem:** T8 do roteiro 1.5.5 (achado A-31)
**Prova:** 18/08 — **REPROVOU.** Evento 49268, `Program Manager`, titulo fora da lista de
placeholders. **A-31 reaberto.**

**Passos:** minimize tudo, fique ~1 minuto na area de trabalho, volte a uma janela. Procure eventos
com processo de shell (`explorer`) ou titulo vazio/placeholder.

**Passou se:** zero eventos desse tipo.

**Reprovou se — e e o esperado hoje:** `Program Manager` ou area de trabalho entrando como trabalho.
**Confirmacao do A-31, nao achado novo.**

---

### CS-18 · Worker morto volta sozinho · 10 min · risco segundos
**Origem:** PLANO 13.1 e 13.2 · automatizado R5
**Prova:** nenhuma nesta versao

**Passos:** mate o SessionWorker a forca tres vezes seguidas, com 10s de intervalo. Confira que ele
volta a cada vez.

**Passou se, e sao quatro coisas:** o Service detecta e relanca em **ate 15s** a cada vez; o log
registra cada morte e cada relancamento; o Service **nao para de tentar** depois de varias; e ele
permanece estavel durante a sequencia. **Nenhuma ociosidade duplicada no restart** — aceite do
A-48.

**Reprovou se:** o Service desistir de relancar; ou a ociosidade duplicar no restart.

**Se reprovar:** @Bucky. Se for a ociosidade duplicando, e o A-48 de volta.

---

### CS-19 · Rate limit de relancamento de worker · 15 min · risco ate 5 min sem captura na sessao
**Origem:** PLANO 13.2 · automatizado R11 (`-IncluirRateLimit`)
**Prova:** nenhuma

**Fora do padrao de proposito:** deixa a sessao ate 5 min sem worker. Nao rode no meio de um teste
de captura.

**Passos:** mate o worker repetidamente, rapido, ate o rate limit disparar. Depois espere a janela
vencer sem tocar em nada.

**Passou se:** o rate limit **dispara** — o Service para de relancar e diz por que no log; **e** o
worker **volta** sozinho depois que a janela do rate limit vence.

**Reprovou se:** o Service relancar em laco sem limite; **ou** o worker nao voltar depois da janela.
Este segundo e o pior dos dois, porque e falha silenciosa: a sessao fica sem captura e ninguem
percebe.

**Se reprovar:** @Bucky, no mesmo dia se for o segundo caso.

---

# 7. PRIVACIDADE E LGPD — 7 cenarios · raia A · ~1h

**Nenhum cenario deste bloco foi cortado, e nao vai ser.** E o unico bloco onde errar custa
cliente, nao tempo. Uma unica ocorrencia aqui e critica e para o lancamento — nao passa por
discussao de lista, nao vira "achado a avaliar".

**Preparacao — o canario.** Antes de comecar, escolha um texto que voce vai digitar de verdade e
que nao existe em lugar nenhum:

```powershell
$canario = "CANARIO-LGPD-$((Get-Date).ToString('HHmmss'))-SENHA-FALSA"
$canario
```

Digite esse texto num campo de senha, num email e num gerenciador de senhas. **Se ele aparecer em
qualquer evento, log ou arquivo de diagnostico, o Agent esta capturando conteudo digitado.**

**Dump para conferir os seis primeiros cenarios de uma vez:**

```powershell
$ev = "C:\Temp\ManagerAgent-Tests\2026-08-24"
$hoje = Get-Date -Format yyyyMMdd
Get-Content "C:\ProgramData\ManagerAgent\logs\session-worker-S1-$hoje.log" -Tail 2000 > "$ev\PR-dump.txt"
Select-String -Path "$ev\PR-dump.txt" -Pattern "$canario|senha|password|@gmail"
```

---

### PR-01 · Conteudo digitado nao e capturado · 10 min · **critico**
**Origem:** roteiro 08-18 P6.1 · PLANO R3.1.10 · automatizado R4
**Prova:** nenhuma nesta versao

**Passos:** digite o canario num campo de senha e um paragrafo longo num email. Espere 3 minutos e
rode a busca acima.

**Passou se:** **zero** ocorrencias do canario, e zero ocorrencias de qualquer trecho do texto
digitado, em evento, log ou banco local.

**Reprovou se:** uma unica ocorrencia, em qualquer lugar.

**Se reprovar:** **pare tudo.** Nao continue o roteiro, nao rode mais nada nesta maquina, guarde o
dump e chame o Marcos e o @Tony no mesmo minuto. E vazamento, e vazamento nao espera a proxima
reuniao.

---

### PR-02 · Nenhum screenshot · 10 min · **critico**
**Origem:** roteiro 08-18 P6.2 · roteiro simples secao 5
**Prova:** nenhuma nesta versao

**Passos:** use a maquina normalmente por 10 minutos. Depois procure imagem em disco e no payload:

```powershell
Get-ChildItem 'C:\ProgramData\ManagerAgent' -Recurse -Include *.png,*.jpg,*.jpeg,*.bmp,*.webp
```

**Passou se:** nenhuma imagem de tela gerada em disco, e nenhum campo de imagem ou base64 grande no
payload dos eventos.

**Reprovou se:** qualquer imagem, em qualquer formato, ou campo de payload com base64 acima de
alguns KB que voce nao consiga explicar.

**Se reprovar:** mesmo procedimento do PR-01. Pare tudo.

---

### PR-03 · URL: so o dominio raiz, nunca o caminho · 15 min · **critico**
**Origem:** roteiro 08-18 P6.3 e P4.6 · automatizado R4 (achado A-55)
**Prova:** nenhuma nesta versao

**Este e o unico cenario que exercita o recorte de URL.** Nao se corta por parecer redundante com o
PR-04: sao campos diferentes, e o A-55 nasceu justamente da URL completa chegando pelo titulo da
janela.

**Passos:** abra um navegador e navegue por **varias paginas internas** de um mesmo site (nao so a
home). Espere 3 minutos e procure, nos eventos, qualquer coisa com barra depois do dominio.

**Passou se:** o campo de dominio traz **apenas o dominio raiz**; e **nenhuma URL com caminho**
aparece em campo nenhum — inclusive no titulo da janela, que e por onde o A-55 vazou.

**Reprovou se:** qualquer caminho, parametro de consulta ou fragmento de URL em qualquer campo.

**Se reprovar:** guarde o evento inteiro e pare. Trate como o PR-01.

---

### PR-04 · Titulo de janela sem conteudo · 10 min · **critico**
**Origem:** roteiro 08-18 P6.4 · automatizado R4 (achado A-47)
**Prova:** nenhuma nesta versao

**Passos:** abra um email longo e uma conversa de mensageiro, com texto reconhecivel no corpo.
Espere 3 minutos e leia os titulos capturados.

**Passou se:** o titulo capturado e **so o titulo da janela** — nunca corpo de mensagem, nunca
trecho do email, nunca nome de contato que so aparece no corpo.

**Reprovou se:** qualquer pedaco do corpo aparecer no titulo.

**Se reprovar:** e o A-47 de volta, e e vazamento. Pare.

---

### PR-05 · Logs contem so metadados · 10 min · **critico**
**Origem:** roteiro 08-18 P6.5 · PLANO R9.3.4
**Prova:** nenhuma nesta versao

**Por que e separado do PR-01:** o PR-01 olha o **evento**; este olha o **log**. Ja aconteceu de o
evento sair limpo e o log de depuracao imprimir o conteudo inteiro do payload. Sao dois caminhos de
escrita diferentes.

**Passos:** com o canario ja digitado, leia os logs do Service, do worker e do Watchdog do dia
inteiro procurando o canario e conteudo de evento.

**Passou se:** os logs trazem so metadados — nome de processo, contagem, carimbo de tempo, id de
sessao. Nenhum conteudo de evento, nenhum titulo completo de janela com corpo, nenhum canario.

**Reprovou se:** qualquer conteudo de evento impresso em log.

**Se reprovar:** pare. Trate como o PR-01.

---

### PR-06 · Diagnostico mascara token e chave · 10 min · **critico**
**Origem:** roteiro 08-18 P6.6 · PLANO R4.3.4 · automatizado R9
**Prova:** nenhuma nesta versao

**Por que importa mais do que parece:** o ZIP de diagnostico e o arquivo que **sai da maquina do
cliente** e chega ao suporte por email. E o unico artefato do produto que viaja.

**Passos:** rode `coletar-diagnostico.ps1` (ou Menu > Ferramentas > Exportar Diagnostico). Extraia o
ZIP e procure, dentro dele, o device token, o refresh token e a chave de ativacao da empresa.

**Passou se:** token e chave aparecem **mascarados**; o `config.json` incluido nao traz credencial
legivel; e nao ha canario nem conteudo de evento no pacote.

**Reprovou se:** qualquer credencial legivel dentro do ZIP.

**Se reprovar:** pare, e **nao envie o ZIP a ninguem**. Apague o arquivo gerado.

---

### PR-07 · Resumo de input: so contadores, zero conteudo · 10 min · **critico**
**Origem:** roteiro 08-18 P4.4
**Prova:** nenhuma nesta versao

**Este e o unico cenario que exercita o agregador de input.** E o lugar mais provavel de um
keylogger nascer por acidente, porque a funcao precisa **ver** as teclas para conta-las.

**Passos:** digite intensamente por 4 minutos (o resumo agrega a cada 180s), incluindo o canario.
Leia o evento de resumo.

**Passou se:** o resumo traz **apenas contadores** — quantidade de teclas, quantidade de cliques,
intervalo. Nenhuma tecla identificada, nenhuma sequencia, nenhum conteudo.

**Reprovou se:** qualquer campo que permita reconstruir o que foi digitado, inclusive
indiretamente.

**Se reprovar:** pare. Trate como o PR-01.

---

# 8. MULTIUSUARIO — 5 cenarios · raia A · ~2h

**Pre-requisito de todos:** a conta `Marcos` (segundo usuario Windows local) com a senha em maos.
Conferido em 24/08 16:03: **a conta existe e esta habilitada.**

**Ruido conhecido neste bloco, e nao e falha:**

- `Task Scheduler launch succeeded but worker process not found for session N` — aparece enquanto a
  pessoa esta digitando a senha. Mensagem enganosa, nao erro.
- `Falha ao persistir session-state ... Access to the path is denied` na sessao 2 — e o **A-66**,
  fora da lista de bloqueadores por decisao registrada: so acontece em maquina compartilhada, e a
  do cliente e individual. **Anote quantas vezes apareceu e siga.**

---

### MU-01 · Dois usuarios simultaneos · 15 min · risco nenhum
**Origem:** PLANO 8.1
**Prova:** nenhuma nesta versao

**Passos:** logue com a sua conta, use "Trocar usuario" para abrir a sessao da conta `Marcos`, e
deixe as duas ativas. Gere atividade em cada uma.

```powershell
Get-Process ManagerAgent.SessionWorker | Select-Object Id, SessionId, UserName
Get-ChildItem 'C:\ProgramData\ManagerAgent\data' | Select-Object Name, Length
```

**Passou se, e sao quatro coisas:** **dois** processos SessionWorker, um por sessao; um Named Pipe
por sessao; **um arquivo de buffer por sessao** (`autonomous-buffer-S1.db`, `-S2.db`) — foi assim
que o A-60 foi resolvido; e os eventos de cada sessao carimbados com o usuario que de fato os
gerou.

**Reprovou se:** um unico worker para as duas sessoes; dois workers na mesma sessao; ou buffer
compartilhado.

**Se reprovar:** e regressao do A-60. @Bucky, e nao siga para o MU-04.

---

### MU-02 · Logoff de um, o outro continua · 10 min · risco nenhum
**Origem:** PLANO 8.2
**Prova:** nenhuma nesta versao

**Pre-requisito:** MU-01 concluido, duas sessoes vivas.

**Passos:** faca logoff da conta `Marcos` e volte para a sua. Espere 2 minutos e leia o log.

**Passou se:** o worker da sessao que saiu e encerrado corretamente; o worker da outra sessao
**permanece vivo, com o mesmo PID**; o Service permanece ativo; e a captura da sessao que ficou
continua sem interrupcao.

**Reprovou se:** o logoff de um derrubar o worker do outro; ou a sessao que ficou passar mais de 90s
sem worker.

**Se reprovar:** copie a sequencia inteira de log da saida e mande ao @Bucky.

---

### MU-03 · Troca rapida entre dois usuarios · 20 min · risco nenhum
**Origem:** C6 do roteiro 1.5.13 (achado A-63)
**Prova:** nenhuma — **e o unico cenario que exercita de verdade a corrida do A-63**

Na bateria da 1.5.12 a corrida nao aconteceu, e o registro daquele dia diz com todas as letras: "o
teste prova que nada regrediu, nao que a correcao agiu". Este cenario existe para fechar isso.
**Nao se corta.**

**Passos:**
1. Troque para a conta `Marcos` e **use de verdade por 2 minutos**.
2. Volte para a sua e use por 2 minutos.
3. Repita a troca mais **duas vezes, rapido** — o objetivo e a segunda sessao demorar a subir.
4. Faca **logoff** da conta `Marcos` e volte para a sua.
5. Leia o log procurando `LOGOUT|LOGIN|Usuario=|Worker removed|CONNECT received|Failed to launch`.

**Passou se, e sao tres coisas:** **nenhum LOGOUT com o nome do dono vazio** — e a falha que este
cenario procura; nenhuma sessao ficou sem worker por mais de 90s; e um worker por sessao, nunca
dois na mesma.

**Reprovou se:** **qualquer LOGOUT sem nome.**

**Se reprovar:** **para o dia.** LOGOUT sem dono e perda de dado do colaborador, criterio A da
linha de corte. Copie a sequencia inteira da troca e mande ao @Bucky **e** ao @Tony, porque fecha
ou reabre o A-63.

---

### MU-04 · Servico derrubado a forca, dois usuarios · 25 min · risco ~8 min sem captura
**Origem:** C7 do roteiro 1.5.13, **com o criterio reescrito** no regressivo 1.5.16 (achado A-60)
**Prova:** nenhuma — pendente desde a 1.5.10

Prova que, com o Service caindo com duas pessoas trabalhando, **cada uma preserva os seus eventos**.
Faca **depois** do MU-03, com as duas contas logadas (a sua ativa, a `Marcos` desconectada).

**ANOTE OS PIDs ANTES — o criterio depende deles:**

```powershell
Get-Process ManagerAgent.SessionWorker | Select-Object Id, SessionId, StartTime
```

**Passos:**
1. Gere atividade nas duas contas por 2 minutos cada.
2. Na sua conta: `Stop-Process -Name ManagerAgent.Service -Force`
3. Espere **8 minutos** sem mexer — o Watchdog precisa das duas confirmacoes.
4. Confira `Get-Service ManagerAgent` e **os mesmos PIDs** do comando acima.
5. Confira os buffers: `Get-ChildItem 'C:\ProgramData\ManagerAgent\data' | Select Name, Length`.

> **O criterio antigo dizia "os dois workers voltam", e isso induz ao erro.** Matar o Service a
> forca **nao mata os workers**: cada um e processo separado, ve o cano cair e fica tentando
> reconectar (1s ate 30s). Quando o Service volta, ele **readota os orfaos** antes de lancar
> qualquer coisa. Os workers **nao voltam — eles nunca sairam**, e os PIDs tem de ser **os mesmos**
> do passo inicial. Quem esperar PID novo marca reprovacao errada.

**Passou se, e sao cinco coisas:**
1. O Service volta sozinho, levantado pelo Watchdog.
2. **Os mesmos PIDs** de worker do passo inicial continuam vivos — um por sessao.
3. **Nenhum worker duplicado** na mesma sessao. Se a readocao falhar, o Service lanca um segundo e
   ficam dois na mesma sessao — **isso e reprovacao**, e e o defeito que este passo caca.
4. Existe **um buffer por sessao**, e os eventos das duas sobem depois da volta, cada um com o seu
   dono.
5. **Nenhum** `attempt to write a readonly database` na sessao 2.

**Reprovou se:** banco somente-leitura na sessao 2; dois workers na mesma sessao; ou um worker
tendo morrido junto com o Service.

**Se reprovar:** banco somente-leitura e regressao do A-60 — **pare o dia e chame o @Bucky.** E
criterio A da linha de corte.

> **O que esperar de evento de sessao aqui: nada.** Como os workers sobrevivem, o `SessionMonitor`
> nao e construido de novo, e **nao sai LOGIN, nem LOGOUT, nem `SESSAO_INTERROMPIDA`** para
> ninguem. Ausencia total de evento de sessao e o **acerto** neste cenario, nao omissao. Se sair
> LOGOUT, e reprovacao: ninguem saiu.

---

### MU-05 · Desligar com a segunda conta desconectada · 20 min · risco ~3 min sem captura
**Origem:** C5 do roteiro 1.5.13 (achado A-65, caminho 3)
**Prova:** nenhuma — **e o unico cenario que exercita o caminho 3 do A-65**

Faca **por ultimo** entre os de sessao: termina com um desligamento.

**Passos:**
1. Deixe a conta `Marcos` logada e **desconectada** — troque para ela, use 3 min, e volte para a
   sua **sem fazer logoff dela**.
2. Confirme dois workers: `Get-Process ManagerAgent.SessionWorker | Select Id, SessionId, StartTime`.
3. **Desligue** a maquina (Desligar, nao reiniciar). Confirme o aviso de que ha outra pessoa logada.
4. Ligue, entre e espere **5 minutos**.
5. Leia o log do dia e o do dia anterior, procurando LOGOUT e `SESSAO_INTERROMPIDA` nas duas
   sessoes.

**Passou se, e sao tres coisas:** **as duas** sessoes fecham — inclusive a desconectada, que era
exatamente o defeito; **nenhum worker nasce durante o desligamento** (se nascer, e o A-61 de volta);
e nenhuma sessao fica com evento posterior ao proprio encerramento.

> **O que esperar da `SESSAO_INTERROMPIDA` aqui:** o desligamento manda `MACHINE_SHUTDOWN`, que **e**
> fim de sessao para todo mundo, inclusive a desconectada. Entao **deve sair LOGOUT das duas** e a
> flag deve ser gravada; no login seguinte, **suprimida**. Se sair LOGOUT de uma so, e reprovacao.

**Reprovou se:** a sessao da conta `Marcos` nao gerar LOGOUT; ou worker nascendo no meio do
desligamento.

**Se reprovar:** anote os horarios exatos das linhas e passe ao @Bucky.

> **O sexto cenario multiusuario e o AT-19** — a volta de versao com duas contas logadas. Ele vive
> na raia B porque instala versao quebrada, mas o que ele prova e multiusuario: o script de volta
> precisa matar os dois workers para liberar os arquivos.

---

# 9. INTERFACE E DIAGNOSTICO — 6 cenarios · raia A · ~1h

O que o colaborador ve e o que o suporte usa. Barato, e o unico jeito de pegar defeito cosmetico
que vira chamado.

---

### IF-01 · Icone na bandeja: aparencia, estabilidade e reacao · 15 min · risco nenhum
**Origem:** PLANO 2.1 · T1 do roteiro 1.5.5 · roteiro 08-18 P3.3
**Prova:** 18/08 — **PASSOU** (1 worker, icone novo, zero `compat legado`, LOGIN unico em 4,7s)

**Passos:** olhe a bandeja por alguns minutos. Reinicie o Explorer. Depois pare o Service e observe.

**Passou se, e sao cinco coisas:** o icone aparece em ate 15s apos o Service subir; e o escudo
indigo com check branco, **o mesmo do app Android**; a dica de tela mostra `iManager - vX.Y.Z` com
a versao instalada; **um unico icone**, nunca dois; e ele **nao some** ao reiniciar o Explorer.
Ao parar o Service, o estado do icone reage — nao fica congelado mostrando saude.

**Reprovou se:** dois icones; icone que so some ao passar o mouse (icone fantasma, A-29); icone
sumindo com o Explorer; ou versao errada na dica de tela.

**Se reprovar:** icone fantasma e o A-29 — cruze com o CS-14 antes de abrir achado.

---

### IF-02 · Menu, submenu Ferramentas e `menuVisivel` · 15 min · risco nenhum
**Origem:** PLANO 2.2 e 2.3 · roteiro 08-18 P3.1 e P3.2
**Prova:** nenhuma nesta versao

**Atencao:** desde o commit `eb0de6e` o menu obedece o `menuVisivel` que vem de
`GET /api/agente/config`, por plataforma. **O conjunto de itens pode divergir legitimamente do
PLANO antigo** — e o backend que manda.

**Passos:** clique com o botao direito no icone. Abra o submenu Ferramentas. Depois derrube o
backend (ou desligue a rede) e abra o menu de novo.

**Passou se, e sao quatro coisas:** o menu abre sem atraso perceptivel; o cabecalho traz
`iManager - vX.Y.Z` desabilitado como item; os itens presentes **batem com o `menuVisivel`
retornado pelo backend**; e com o backend fora do ar o menu **cai num padrao coerente — nao some e
nao trava**.

**Reprovou se:** o menu travar ou sumir com o backend fora; ou os itens ignorarem o `menuVisivel`.

**Se reprovar:** anote o que o backend devolveu e o que o menu mostrou. @Bucky e @Shuri.

---

### IF-03 · Limpar Dados e Reiniciar — a confirmacao · 10 min · risco perde o buffer local
**Origem:** PLANO 2.5 e 4.4
**Prova:** nenhuma nesta versao

**E acao destrutiva, e o teste e sobre a porta, nao sobre a acao.** Eventos ainda nao enviados
somem para sempre.

**Passos:** Menu > Ferramentas > Limpar Dados e Reiniciar. **Responda Nao na primeira vez.** Depois
repita e responda Sim.

**Passou se, e sao seis coisas:** o dialogo aparece **antes de qualquer acao**; descreve com
clareza o que sera feito; avisa que nao pode ser desfeito; **"Nao" e o padrao** (o que Enter
seleciona); clicar Nao **cancela de verdade** — o buffer nao e tocado e o Service continua rodando;
e clicar Sim para o Service, encerra o worker, remove o buffer, preserva log e config, reinicia e
o icone volta.

**Reprovou se:** a acao acontecer antes da confirmacao; "Sim" ser o padrao; ou clicar Nao ainda
assim mexer no buffer.

**Se reprovar:** e defeito de porta destrutiva — vai para o @Bucky no mesmo dia, mesmo sem bloquear
release.

---

### IF-04 · Sobre · 5 min · risco nenhum
**Origem:** PLANO 2.6
**Prova:** nenhuma nesta versao

**Mantido apesar de barato:** e por esta janela que o suporte identifica **qual maquina** esta
falando. Sem ela, a primeira pergunta de todo chamado fica sem resposta.

**Passos:** Menu > Sobre.

**Passou se:** abre; mostra a versao no formato X.Y.Z, igual a instalada; mostra o identificador da
instalacao; mostra o identificador do colaborador configurado; e o botao fecha.

**Reprovou se:** versao divergindo da instalada; ou identificador vazio.

---

### IF-05 · health-check.ps1 · 10 min · risco nenhum
**Origem:** PLANO 4.1 · roteiro 08-18 P7.1 · automatizado R9
**Prova:** nenhuma nesta versao

**Passos:**

```powershell
cd "C:\Program Files\ManagerAgent\scripts"
.\health-check.ps1
```

**Passou se, e sao cinco coisas:** executa sem erro e responde em menos de 45s; confere o servico
`ManagerAgent` **e o `ManagerAgentWatchdog`** — o segundo era a lacuna do PLANO antigo; confere os
dois processos, o config, o buffer e o Named Pipe; testa conectividade com as APIs; e devolve
**80% ou mais** numa instalacao normal, com resumo ao final.

**Reprovou se:** o script nao verificar o Watchdog; ou devolver abaixo de 80% numa maquina que
esta comprovadamente saudavel — nesse caso o defeito e do script, nao do Agent, e vale igual.

**Se reprovar:** guarde a saida inteira. @Bucky.

---

### IF-06 · coletar-diagnostico.ps1 e monitorar-logs.ps1 · 15 min · risco nenhum
**Origem:** PLANO 4.2 e 4.3
**Prova:** nenhuma nesta versao

**A conferencia de mascaramento do ZIP e o PR-06** — aqui so se testa se as ferramentas funcionam.

**Passos:** rode `coletar-diagnostico.ps1`. Depois `monitorar-logs.ps1 -Filter Error` e
`-Source Service`, saindo com Ctrl+C.

**Passou se:** o ZIP e criado com nome carimbado, abre sem erro, e traz status dos servicos, os dois
processos, config, os logs recentes de cada origem, status do buffer e o resultado do health-check;
o monitor mostra Service e worker ao vivo, com cores por nivel, os filtros funcionam, e Ctrl+C
encerra sem excecao.

**Reprovou se:** o ZIP nao abrir; faltar log de uma das origens; ou Ctrl+C deixar processo orfao.

---

# 10. PESO NA MAQUINA — 3 cenarios · raia A · ~1h

**O limite mudou, e o numero antigo reprova produto correto.** O PLANO pedia 100MB por processo. Esse
criterio e **anterior ao build self-contained**, cujo working set inclui paginas compartilhadas do
runtime embutido e nasce naturalmente mais alto. **O limite vigente e 150MB por processo**, decidido
pelo Marcos em 19/08.

---

### DS-01 · Memoria e CPU dos tres processos · 15 min · risco nenhum
**Origem:** PLANO 5.1 e 5.2 · roteiro 08-18 PASSO 8 · automatizado R7 e R10
**Prova:** nenhuma nesta versao

**Passos:** com a maquina em uso normal, rode por 10 minutos:

```powershell
$fim = (Get-Date).AddMinutes(10)
while ((Get-Date) -lt $fim) {
  Get-Process -Name "ManagerAgent.Service","ManagerAgent.SessionWorker","ManagerAgent.Watchdog" -ErrorAction SilentlyContinue |
    ForEach-Object { "{0} {1} = {2} MB / {3} threads" -f (Get-Date -Format HH:mm:ss), $_.Name, [math]::Round($_.WorkingSet64/1MB,1), $_.Threads.Count }
  Start-Sleep -Seconds 30
}
```

**Passou se, e sao quatro coisas:** cada um dos tres processos abaixo de **150MB**; o Watchdog
abaixo de **60MB**; nenhum cresce continuamente ao longo dos 10 minutos; e a CPU media combinada
fica abaixo de 15%, praticamente zero com a maquina parada.

**Reprovou se:** crescimento continuo sem estabilizar — e o unico sintoma de vazamento que da para
ver em 10 minutos; ou pico constante de CPU.

**Se reprovar:** repita com 30 minutos antes de abrir achado. Um pico isolado nao e vazamento.

---

### DS-02 · Crescimento do buffer local · 10 min de leitura · risco nenhum
**Origem:** PLANO 5.4 e 12.3 · automatizado R10
**Prova:** nenhuma nesta versao

**Passos:**

```powershell
Get-ChildItem 'C:\ProgramData\ManagerAgent\data' | Select-Object Name, Length, LastWriteTime
```

**Passou se:** o buffer de cada sessao fica **abaixo de 50MB** em uso normal; e apos um ciclo de
upload bem-sucedido o tamanho para de crescer.

**Reprovou se:** buffer acima de 50MB numa maquina que esta enviando normalmente — significa que a
fila nao esta drenando, e o proximo passo e o RS-01.

---

### DS-03 · Cadencia de upload · 15 min · risco nenhum
**Origem:** roteiro 08-18 P8.2 · automatizado R8
**Prova:** nenhuma nesta versao

**Passos:** deixe a maquina capturando por 15 minutos e leia os ciclos de upload no log do Service.

**Passou se:** o ciclo respeita o intervalo configurado; o lote respeita o tamanho maximo; o numero
de lotes por ciclo respeita o teto do `appsettings`; e os ciclos fecham com `FailedBatches=0` e
`Descartados=0`.

**Reprovou se:** `Descartados` diferente de zero em operacao normal — **isso e perda de dado do
colaborador** e vira criterio A; ou a cadencia ignorando o config, que e o mesmo sintoma do A-50
visto no CS-05.

**Se reprovar:** se houver descarte, guarde os ciclos inteiros e cruze com o BK-01 — o descarte
apos cinco recusas e exatamente o caminho de perda que o item 1 fecha.

---

# 11. VINCULO, IDENTIDADE E CREDENCIAIS — 5 cenarios · raia A · ~1h30

Bloco que **nao existia no PLANO** e entrou no roteiro de 18/08. Sem vinculo nao ha captura, e a
credencial que sustenta tudo expira sozinha.

---

### VC-01 · Vinculacao e Device JWT · 20 min · risco nenhum
**Origem:** roteiro 08-18 P2.1 a P2.4
**Prova:** nenhuma nesta versao

**Passos:**

```powershell
cd "C:\Program Files\ManagerAgent\scripts"
.\test-vinculacao.ps1
$hoje = Get-Date -Format yyyyMMdd
Select-String -Path "C:\ProgramData\ManagerAgent\logs\service-$hoje.log" -Pattern "vincul|link|token|refresh|401|403"
```

**Passou se, e sao quatro coisas:** a vinculacao conclui; o `config.json` tem identificador da
instalacao, identificador do colaborador, device token e refresh token; a config **nao e salva em
laco** — olhe o log, era a suspeita por tras das falhas unitarias do `AgentLinkService`; e config
incompleto (remova o identificador da instalacao e reinicie o Service) produz **erro claro, nao
retry infinito**.

**Reprovou se:** gravacao de config em laco; ou retry infinito com config incompleto.

**Se reprovar:** devolva o `config.json` de antes — guarde uma copia **antes** de comecar.

---

### VC-02 · Token expirado dispara renovacao · 20 min · risco nenhum
**Origem:** roteiro 08-18 P2.5 · automatizado `n7-device-jwt-expirado.ps1`
**Prova:** nenhuma em maquina

**Este e o unico cenario manual que cobre a renovacao proativa.** O automatizado n7 existe e cobre
o mesmo caminho — se ele puder rodar, rode-o e marque este como coberto.

**Passos:** provoque a expiracao (ou espere a renovacao natural, a cada ~2h) e observe o ciclo.

**Passou se:** a sequencia 401 -> renovacao -> nova tentativa acontece **sem perder evento**; e o
log traz `Device JWT renovado com sucesso`.

**Reprovou se:** eventos descartados durante a renovacao; ou laco de 401 sem renovar.

---

### VC-03 · Dispositivo revogado para de enviar · 20 min · risco nenhum
**Origem:** roteiro 08-18 P2.6 · automatizado `n8-agente-revogado.ps1`
**Prova:** nenhuma em maquina

**Passos:** peca a revogacao do dispositivo no backend e observe dois ciclos de upload.

**Passou se:** o Agent **para de enviar** e **loga o motivo com todas as letras**; e nao entra em
laco de retry contra um servidor que ja disse nao.

**Reprovou se:** o Agent continuar tentando indefinidamente; ou parar **em silencio** — parar sem
dizer por que e tao ruim quanto nao parar, porque o suporte nao tem como diagnosticar.

**Ao terminar:** desfaca a revogacao antes de seguir, ou todos os cenarios seguintes medem errado.

---

### VC-04 · Sem vinculo nao captura · 15 min · risco nenhum
**Origem:** automatizado `n9-sem-vinculo-nao-captura.ps1` (aceite da ADR-001)
**Prova:** nenhuma em maquina

**Rode o modo de diagnostico primeiro** — ele e o padrao do script, nao muda nada na maquina e
responde a unica pergunta que importa antes de qualquer coisa.

**Passou se:** sem vinculo ativo com colaborador, o Agent **nao captura e nao envia** — e diz por
que no log.

**Reprovou se:** captura ou envio acontecendo sem vinculo. **E violacao da ADR-001**, e nao e
detalhe: e dado de pessoa sendo coletado sem o vinculo que o autoriza.

**Se reprovar:** pare e chame o @Tony. Isto tem peso de LGPD, nao so de bug.

---

### VC-05 · Token nunca em texto claro · 5 min · risco nenhum
**Origem:** roteiro 08-18 P1.19 · PLANO R1.1.8
**Prova:** nenhuma nesta versao

```powershell
$cfg = Get-Content 'C:\ProgramData\ManagerAgent\config.json' -Raw | ConvertFrom-Json
$cfg.PSObject.Properties | Where-Object { $_.Name -match 'token|chave|key|secret' } | Select-Object Name
```

**Passou se:** o campo de device token esta **cifrado (DPAPI, escopo de maquina)** — uma string
base64 longa, nunca um JWT legivel comecando por `eyJ`.

**Reprovou se:** qualquer credencial legivel no `config.json`.

**Se reprovar:** trate como o PR-01 — e vazamento em disco. Pare.

---

# 12. RESILIENCIA — 14 cenarios · raia A · ~3h

Rede fora, disco cheio, arquivo corrompido, processo morto. O bloco que prova que o Agent **nao
perde o dado que ja capturou** quando o mundo em volta falha.

---

### RS-01 · Servidor fora do ar · 20 min · risco nenhum
**Origem:** PLANO 6.1 · marco 10.1
**Prova:** nenhuma nesta versao

**Passos:** aponte o endereco de eventos para um host inexistente no `config.json`, reinicie o
Service, use a maquina por 10 minutos, e devolva o endereco correto. **Guarde uma copia do config
antes.**

**Passou se, e sao seis coisas:** o Service continua rodando, sem travar nem parar; o worker
continua ativo e o icone na bandeja permanece; a captura continua; os eventos ficam no buffer local;
o log registra **falha de upload, nao erro critico**; e o reenvio e tentado periodicamente. Nenhuma
excecao nao tratada.

**Reprovou se:** o Service cair; a captura parar; ou aparecer excecao nao tratada.

**Ao terminar:** devolva o `config.json` e confirme que a fila drenou.

---

### RS-02 · Sem rede · 20 min · risco nenhum
**Origem:** roteiro 08-18 P9.1 · marco 10.1 e 10.2 · roteiro simples secao 6
**Prova:** nenhuma nesta versao

**Diferente do RS-01, e por isso os dois existem:** ali o servidor recusa, aqui **nao ha caminho**.
Sao pilhas de erro diferentes, e a primeira ja passou enquanto a segunda falhava.

**Passos:** desligue o Wi-Fi. Use a maquina por 10 minutos gerando atividade. Religue.

**Passou se:** a captura continua; o buffer cresce; e ao voltar a rede **o acumulado sobe sem
duplicar nada**.

**Reprovou se:** eventos duplicados apos a volta; ou eventos perdidos no intervalo.

**Se reprovar:** duplicata e o pior dos dois — infla o relatorio de horas de alguem. Cruze com o
BK-02 antes de fechar o diagnostico.

---

### RS-03 · Config invalido · 15 min · risco ate restaurar
**Origem:** PLANO 6.2
**Prova:** nenhuma nesta versao

**Guarde uma copia do config antes.** Depois grave um JSON malformado, reinicie o Service e espere
10 segundos.

**Passou se:** o Service **detecta** o config invalido; o log traz erro de configuracao **claro**;
e ele nao trava indefinidamente — encerra ou usa valores padrao, mas decide.

**Reprovou se:** o Service ficar preso sem log; ou subir fingindo que esta tudo bem.

**Ao terminar:** restaure o config e confirme que voltou a capturar.

---

### RS-04 · Buffer corrompido · 15 min · risco perde o buffer da sessao
**Origem:** PLANO 6.3
**Prova:** nenhuma nesta versao

**Passos:** pare o Service, sobrescreva o arquivo de buffer da sessao com lixo, suba o Service e
espere 10 segundos.

**Passou se:** o Service detecta o banco corrompido; o log diz isso; ele **recria o banco
automaticamente**; e Service e worker voltam a funcionar normalmente.

**Reprovou se:** o Service cair em laco de partida; ou subir sem conseguir gravar evento nenhum,
em silencio.

---

### RS-05 · Buffer no limite e disco apertado · 20 min · risco nenhum
**Origem:** PLANO 5.4 e 12.3
**Prova:** nenhuma nesta versao

**Passos:** com a rede desligada (aproveite o RS-02), deixe o buffer crescer o maximo que conseguir
e observe o comportamento no limite configurado.

**Passou se:** o worker **nao consome memoria excessiva** durante um periodo autonomo longo; ao
atingir o limite, os eventos **mais antigos** sao descartados, e **o log registra o descarte com a
contagem** — descarte silencioso e reprovacao mesmo que o limite esteja correto.

**Reprovou se:** descarte sem registro; ou o worker crescendo em memoria proporcionalmente ao
buffer.

---

### RS-06 · Crash do Service: o SCM levanta · 10 min · risco segundos
**Origem:** PLANO 7.3 · roteiro 08-18 P10 · marco 10.3
**Prova:** nenhuma nesta versao

**Passos:** `Stop-Process -Name ManagerAgent.Service -Force`. Espere 10 segundos e confira.

**Passou se:** o SCM detecta e reinicia em **ate 5 segundos** (a configuracao e 5s/10s/30s); o
worker e readotado ou relancado; o log registra o reinicio apos falha; e **o buffer nao e perdido**.

**Reprovou se:** o Service nao voltar sozinho; ou o buffer da sessao ficar corrompido pelo kill.

---

### RS-07 · Service travado: o Watchdog forca a recuperacao · 15 min · risco ~6 min
**Origem:** roteiro 08-18 P10.1 e P10.2
**Prova:** nenhuma nesta versao

**Diferente do RS-06:** ali o processo **morre** e o SCM ve. Aqui ele fica **vivo e mudo**, e o SCM
nao ve nada. So o Watchdog pega, pelo heartbeat parado.

**Passos:** suspenda o processo do Service (nao mate). Espere ate 8 minutos sem mexer.

**Passou se:** o Watchdog detecta o heartbeat parado e **forca a recuperacao**; e o evento de
recuperacao **e postado ao backend** pelo reporter HTTP — nao basta o log local.

**Reprovou se:** o Watchdog nao agir em 8 minutos; ou agir sem reportar.

---

### RS-08 · Modo SOS quando a recuperacao falha em serie · 20 min · risco **entra em SOS**
**Origem:** roteiro 08-18 P10.3
**Prova:** nenhuma nesta versao

**Atencao:** este cenario **liga o SOS de proposito**, e o SOS **nao sai sozinho**. Faca por ultimo
no bloco, e desfaca antes de qualquer outra coisa.

**Passou se:** apos falhas repetidas de recuperacao, o Watchdog entra em SOS, grava `sosSince`, e
**registra o motivo** — nao entra em silencio.

**Reprovou se:** entrar em SOS sem motivo registrado; ou nunca entrar, ficando em laco de tentativa.

**OBRIGATORIO ao terminar:** troque `sosMode` para `false` no `watchdog-state.json`, reinicie os
dois servicos e **confirme que voltou a `false`** antes de seguir.

---

### RS-09 · Estado do Watchdog sobrevive a reboot · 5 min · risco nenhum
**Origem:** roteiro 08-18 P10.4
**Prova:** nenhuma nesta versao

**Aproveite o reboot do CS-07** — nao gaste outro.

**Passos:** anote o conteudo do `watchdog-state.json` antes do reboot do CS-07 e compare depois.

**Passou se:** o estado persiste, e o arquivo esta **legivel** — escrita atomica, nunca truncado,
sem arquivo temporario orfao ao lado.

**Reprovou se:** estado zerado no boot; ou arquivo truncado.

---

### RS-10 · Matar o Watchdog · 5 min · risco segundos
**Origem:** roteiro 08-18 P10.5
**Prova:** nenhuma nesta versao

**Passos:** mate o processo do Watchdog a forca e espere 10 segundos.

**Passou se:** o SCM o reinicia em ate 5 segundos.

**Reprovou se:** o Watchdog nao voltar. **E o pior caso silencioso do produto:** a maquina segue
capturando e ninguem percebe que ela ficou sem quem a levante na proxima queda.

---

### RS-11 · Named Pipe: fluxo normal e reconexao · 20 min · risco segundos
**Origem:** PLANO 9.1 e 9.2 · PLANO 12.1 e 12.2
**Prova:** nenhuma nesta versao

Juntos de proposito: um sem o outro engana. O primeiro prova que entrega; o segundo, que nao perde
quando o cano cai.

**Passos:** gere eventos e confirme a entrega. Depois `Restart-Service ManagerAgent`, gere eventos
**durante** a janela sem Service, e espere a volta.

**Passou se, e sao cinco coisas:** o log traz a conexao do pipe apos o worker subir; os eventos
chegam ao Service em menos de 1 segundo; ao cair o pipe, o worker **entra em modo autonomo e diz
isso no log**; ele **continua capturando** sem Service; e ao voltar, os eventos bufferizados sobem
**sem duplicar** e o log registra a saida do modo autonomo.

**Reprovou se:** evento perdido na janela; evento duplicado apos a reconexao; ou o worker parando
de capturar quando o pipe cai.

---

### RS-12 · Named Pipe: quem pode conectar · 15 min · risco nenhum
**Origem:** PLANO 9.3
**Prova:** nenhuma — **e o unico cenario que cobre a permissao do canal**

**Nao se corta por parecer teorico.** O pipe carrega o evento bruto do colaborador antes de
qualquer mascaramento. Quem consegue ler o pipe le tudo.

**Passos:** com duas sessoes vivas (aproveite o MU-01), tente conectar ao pipe da sessao 1 a partir
da sessao 2.

**Passou se:** o pipe aceita o usuario **da sessao correspondente** e o SYSTEM; **rejeita** a
tentativa de outro usuario; e o log da tentativa registra **so metadados de conexao**, nunca
conteudo de evento.

**Reprovou se:** um usuario conseguir ler o pipe de outro. **Trate como o PR-01: pare tudo.**

---

### RS-13 · Reenvio nao duplica · 15 min · risco nenhum
**Origem:** marco 10.4 · PLANO R12.2.4 · roteiro simples secao 6
**Prova:** nenhuma nesta versao

**Consolidado**: a nao-duplicacao e criterio dentro do RS-02 e do RS-11, mas ninguem nunca conferiu
**do outro lado**. Este cenario e a conferencia final, e depende do BK-02.

**Passos:** depois do RS-02 e do RS-11, com os horarios anotados, confira no banco que os eventos
daquele intervalo aparecem **uma vez so**.

**Passou se:** nenhuma duplicata por id ou por par (evento, carimbo de tempo).

**Reprovou se:** qualquer duplicata. **Marque "nao conferido" se o acesso ao banco nao sair** —
nunca "aprovado".

---

### RS-14 · Ciclo de vida do Service: boot, parada graciosa, reinicio · 20 min · risco ~3 min
**Origem:** PLANO 7.1, 7.2 e 7.4
**Prova:** parcial — o boot foi coberto pelo CS-07 em 24/08

**Passos:** confira o boot (ja feito no CS-07). Depois: conte os eventos pendentes,
`Stop-Service ManagerAgent`, espere 5s, leia o fim do log. Depois `Restart-Service ManagerAgent`.

**Passou se, e sao cinco coisas:** o Service para **sem estourar o tempo do SCM** (menos de 30s); o
worker e encerrado pelo Service **antes** de ele parar; o icone some da bandeja; o log indica
encerramento gracioso, sem crash; e o buffer nao e corrompido. No reinicio, o worker volta, o icone
volta e a captura retoma.

**Reprovou se:** timeout do SCM na parada; buffer corrompido no shutdown; ou o worker sobrevivendo
a uma parada graciosa.

**Se reprovar:** perda de dado no shutdown normal e criterio A. Pare e chame o @Bucky.

---

# 13. INSTALACAO E DESINSTALACAO — 6 cenarios · ~2h30

**Cinco destes seis sao raia B.** Eles destroem a vinculacao, os buffers e a historia da maquina.
Nao rode nenhum antes de fechar a raia A inteira.

**Pre-requisito comum:** identificador do colaborador e **chave de ativacao** da empresa de teste em
maos. Sem a chave, a maquina fica instalada e **sem vinculo — ou seja, sem capturar**.

**Regra herdada, e vale para o bloco inteiro:** teste que monta a propria arvore de diretorios nao
prova nada sobre a instalacao real. **Todo cenario roda contra o que o instalador produziu**, nunca
contra pasta montada a mao. Foi exatamente essa distancia que escondeu o defeito original do
rollback: os testes unitarios passavam porque concordavam com o codigo, e os dois discordavam do
instalador.

---

### IN-01 · Instalacao limpa · 30 min · **raia B** · risco: a maquina fica sem captura ate concluir
**Origem:** PLANO 1.1 · roteiro 08-18 PASSO 1 (P1.16 a P1.19)
**Prova:** nenhuma nesta versao

**Passos:** confirme que nao existe instalacao previa. Rode o instalador como administrador. Informe
identificador e chave no assistente. Espere 3 minutos.

**Passou se, e sao onze coisas:**
1. O instalador inicia e conclui sem erro.
2. `C:\Program Files\ManagerAgent` criada, com `ManagerAgent.Service.exe`,
   `ManagerAgent.SessionWorker.exe` e `ManagerAgent.Configurator.exe`.
3. **Seis scripts** em `scripts\` — health-check, monitorar-logs, coletar-diagnostico, limpar-reset,
   test-vinculacao e desvincular. *(O PLANO antigo dizia cinco. Sao seis desde que
   `test-vinculacao` e `desvincular` entraram.)*
4. `C:\ProgramData\ManagerAgent` criada, com `config.json`, buffer e pasta de logs.
5. **Dois** servicos registrados: `ManagerAgent` e `ManagerAgentWatchdog`. *(O PLANO antigo so citava
   um. O Watchdog e servico proprio.)*
6. `ManagerAgent` com inicio `Automatic (Delayed)`; `ManagerAgentWatchdog` com `Automatic`.
7. **Auto-recuperacao do SCM em `5s / 10s / 30s`.** *(O PLANO antigo dizia 1s/5s/30s. O `.iss`
   configura 5/10/30 — conferido no roteiro de 18/08. O numero antigo reprova instalacao correta.)*
8. Os dois servicos sobem sozinhos apos a instalacao.
9. O worker e lancado pelo Service e o icone aparece na bandeja.
10. `DOTNET_BUNDLE_EXTRACT_BASE_DIR` presente em `HKLM` — requisito de multiusuario, ausente do
    PLANO antigo.
11. O identificador da instalacao e um GUID valido e o token esta **cifrado** (ver VC-05).

**Reprovou se:** faltar um dos dois servicos; SCM com valores diferentes de 5/10/30; menos de seis
scripts; ou a vinculacao nao concluir.

**Se reprovar:** reinstale com a rede de seguranca. Se o instalador recusar por instalacao
existente, remova `C:\Program Files\ManagerAgent` a mao e repita.

---

### IN-02 · Configurator: configuracao e troca de ambiente · 20 min · **raia B**
**Origem:** PLANO 1.2 · roteiro 08-18 decisao 1
**Prova:** nenhuma nesta versao

> **O pacote instalado aponta para PRODUCAO por padrao.** Quem troca e o Configurator. Se voce
> instalar e nao trocar, **os eventos desta rodada vao para producao.** Decida o ambiente **antes**
> de instalar, e confirme no `config.json` depois.

**Passos:** abra `ManagerAgent.Configurator.exe`. Preencha os dois enderecos de backend, a chave de
ativacao e o identificador do colaborador. Salve e reinicie o Service.

**Passou se:** o Configurator grava tudo no `config.json`; o Service reinicia apos salvar; o worker
passa a usar a configuracao nova; e os eventos passam a sair para o ambiente escolhido — confirme
no log, nao no formulario.

**Reprovou se:** o config nao refletir o que foi digitado; ou o Service continuar apontando para o
endereco antigo depois do reinicio.

---

### IN-03 · Reinstalar por cima: o upgrade preserva a identidade · 20 min · **raia B**
**Origem:** PLANO 1.3 · roteiro simples secao 1
**Prova:** nenhuma nesta versao

**Passos:** com o Agent instalado e vinculado, rode o mesmo instalador de novo e escolha
sobrescrever.

**Passou se, e sao quatro coisas:** o instalador **detecta** a instalacao existente e oferece
sobrescrever ou cancelar; o `config.json` existente e **preservado**; o **identificador da
instalacao NAO e regerado**; e o Service reinicia com a configuracao antiga intacta.

**Reprovou se:** identificador novo apos upgrade. **Isso e o A-49:** cada upgrade criando uma linha
nova em `agentes` para a mesma maquina — a base chegou a ter 12 linhas para a `DESKTOP-VMSM6LE`,
com 12 identificadores distintos e **um unico** hardware fingerprint.

**Se reprovar:** cruze com o IN-06 antes de abrir achado.

---

### IN-04 · Desinstalar e instalar do zero · 40 min · **raia B** · risco: apaga a vinculacao e a historia
**Origem:** C8 do roteiro 1.5.13 · PLANO 14.1 e 14.2
**Prova:** nenhuma — pendente desde a 1.5.10

**Faca depois de todo o resto.** Apaga a vinculacao e os buffers, e todos os cenarios anteriores
perdem a historia.

**Antes de desinstalar**, anote o identificador da instalacao e guarde uma copia dos logs e do
`watchdog-state.json`.

**Passos:** desinstale pelo Painel de Controle. Confira a limpeza. Instale de novo, informando
identificador e chave. Espere 3 minutos.

**Passou se, e sao seis coisas:** os **dois** servicos saem do SCM; a pasta de programa some;
nenhum processo `ManagerAgent.*` sobra; a chave de ambiente `DOTNET_BUNDLE_EXTRACT_BASE_DIR` sai do
registro; a reinstalacao conclui e a maquina volta **vinculada e capturando**; e o novo
identificador de instalacao **e diferente do anterior** — aqui a mudanca e o comportamento correto,
ao contrario do IN-03.

**Reprovou se:** sobrar servico registrado, processo vivo ou a chave de ambiente no registro; ou a
vinculacao nao concluir.

**Se reprovar:** a maquina fica sem captura. Reinstale com a rede de seguranca; se o instalador
recusar, remova a pasta a mao.

---

### IN-05 · Desinstalacao limpa: o dispositivo fica desvinculado · 20 min · **raia B**
**Origem:** roteiro 08-18 P15.1 a P15.3 · PLANO 14.1
**Prova:** nenhuma nesta versao

**Faca junto do IN-04** — e a mesma desinstalacao, olhada do lado do backend.

**Passou se:** o `desvincular.ps1` roda durante a desinstalacao; e o dispositivo **some da lista de
vinculados no portal**. O `ManagerAgentWatchdog` tambem e desregistrado, e a opcao de preservar os
dados do usuario e oferecida.

**Reprovou se:** o dispositivo continuar vinculado no portal apos desinstalar. **E o pior caso de
dado orfao:** a empresa continua vendo um colaborador ligado a uma maquina que nao existe mais.

**Se reprovar:** @Shuri, com o identificador da instalacao anotado antes de desinstalar — sem ele
nao da para reproduzir.

---

### IN-06 · Identificador da instalacao e estavel · 10 min · raia A
**Origem:** automatizado `n6-instalacao-id-estavel.ps1` (diagnostico do A-49)
**Prova:** medido no banco em 19/08 — 12 linhas para a mesma maquina, 12 identificadores, **um**
hardware fingerprint, nenhuma com data de desvinculacao

**So leitura.** Nao muda nada na maquina.

**Passos:** compare o identificador atual com o de antes do ultimo update, e confira o fingerprint
de hardware.

**Passou se:** troca de binario **preservando o `ProgramData` nao cria linha nova** — o
identificador e reaproveitado. Foi o que aconteceu de 1.5.7 para 1.5.8: o id 51 foi reusado.

**Reprovou se:** identificador novo apos um update. Cruze com o IN-03.

---

# 14. ATUALIZACAO E VOLTA DE VERSAO — 22 cenarios · ~10h

**Dezesseis sao raia B e dois sao VM.** Este e o bloco mais caro do roteiro, e o unico que exige
gerar pacotes.

## Preparacao da raia B — a ordem nao pode ser invertida

1. Definir o numero novo com o @Vision (chame de **N**).
2. Gerar o pacote **N** com o codigo atual e instalar por cima, **preservando a vinculacao**.
3. Confirmar que **N** subiu e esta capturando por pelo menos 15 minutos.
4. **So entao** gerar o pacote **N+1 propositalmente quebrado** — o servico principal nao sobe. A
   receita e a do `e1-alt-rollback-crash.ps1`: trocar o `ManagerAgent.Service.exe` por um binario
   que sai com codigo 1 no startup. **Use so a receita; nao rode aquele script** (secao 1).
5. Publicar **N+1 apenas para esta maquina**, pelo servidor local de teste — so o endereco de
   administracao aponta para o stub; os eventos continuam indo para staging.

> **Nao publique a versao quebrada no backend de verdade.** Se publicar, qualquer outra maquina que
> perguntar vai baixa-la.

**Se um pacote N nao existir, os cenarios de raia B nao rodam** — e isso e pre-requisito de outra
pessoa, nao problema deste roteiro. Marque **"bloqueado por pre-requisito"**, nunca "reprovado".

---

### AT-01 · Auto-update fim a fim (Plan A / WMI) · 30 min · **raia B**
**Origem:** PLANO 10.1 · roteiro 08-18 P13
**Prova:** 24/08 — **passou 4x**, com checksum, nunca falhou. A quarta (15:58) foi para a propria
1.5.16

> **O PLANO antigo chamava o Plan A de "Script PS1 via schtasks". Esta errado.** O Plan A e **WMI
> `Win32_Process.Create`** desde a v1.3.10; o `schtasks` virou Plan B, justamente porque o Task
> Scheduler atrasava o disparo em ate ~6 minutos. Quem procurar `schtasks` no log de um Plan A
> bem-sucedido nao vai achar, e vai marcar reprovacao errada.

**Passos:** com **N+1 bom** publicado no stub, `Restart-Service ManagerAgent` e acompanhe o log ao
vivo.

**Passou se, e sao seis coisas:** o Service detecta a versao nova; baixa para a pasta de updates; o
**checksum e verificado antes de aplicar**; o log traz `Plan A (WMI) - Create returned 0`; os dois
servicos voltam na versao nova; e o log de partida traz `Reason=POST_UPDATE`.

**Reprovou se:** aplicar sem verificar checksum; ou nao voltar na versao nova em ate 15 minutos.

---

### AT-02 · Caminhos de recuperacao do update: Plan B, Plan C e falha total · 60 min · **raia B**
**Origem:** PLANO 10.2, 10.3 e 10.4 · automatizado `e2-wmi-failure-fallback.ps1`
**Prova:** nenhuma em maquina

**Tres testes num cenario so, porque a preparacao e a mesma e cada um so faz sentido depois do
anterior falhar.** **Rode o `e2` primeiro** — ele cobre o Plan B automaticamente, sabotando o
servico do WMI. Se ele passar, so o Plan C e a falha total ficam para a mao.

**Passou se, e sao cinco coisas:** com o WMI sabotado, o log registra o Plan A falhando e o Plan B
(schtasks) sendo acionado **automaticamente**, e o update completa; com A e B indisponiveis, o Plan
C (rename em processo) e acionado e o Service novo sobe; **o log diz qual plano teve sucesso**; e
com os tres falhando, o Service **continua rodando na versao atual, capturando**, com o motivo de
cada falha no log.

**Reprovou se:** o Agent parar de capturar por causa de um update que nao deu certo. **E o criterio
central deste cenario:** update que falha nao pode custar o dia de trabalho de ninguem.

**Ao terminar:** restaure o servico do WMI e a politica de execucao.

---

### AT-03 · Checksum SHA-256 errado · 20 min · **raia B**
**Origem:** automatizado `e5-sha256-mismatch.ps1` · roteiro 08-18 P13 · marco 9.3
**Prova:** nenhuma em maquina — o automatizado existe

**Passos:** publique um update com o SHA-256 declarado errado e deixe o ciclo acontecer.

**Passou se:** o update **nao e aplicado**; o Agent segue na versao anterior, capturando; e o log
diz que o checksum nao bateu.

**Reprovou se:** o pacote ser aplicado mesmo com o checksum errado. **Isso e falha de seguranca**,
nao de atualizacao: significa que qualquer pacote adulterado entra.

**Se reprovar:** pare tudo e chame o @Tony no mesmo minuto.

---

### AT-04 · Versao igual e recusada · 20 min · **raia B**
**Origem:** automatizado `n1-versao-igual-recusada.ps1` (aceite do A-46)
**Prova:** observado em 19/08 — o ciclo seguinte ao E8 respondeu `no update available`

**Por que existe:** ate a 1.5.7 o Agent confiava cegamente no backend. Se a resposta dissesse
disponivel, ele baixava e aplicava — **mesmo que a versao oferecida fosse identica a instalada**.
Backend com bug de comparacao ou cache servindo resposta velha viravam laco de update.

**Passou se:** oferecendo a versao **ja instalada**, o Agent **recusa**, diz por que, e nao baixa
nada.

**Reprovou se:** qualquer download comecar. O pacote tem ~310MB; download iniciado ja e reprovacao,
mesmo que a instalacao seja barrada depois.

---

### AT-05 · Update com usuario logado · 40 min · **raia B**
**Origem:** roteiro de validacao 1.5.7 · automatizado `e8-update-com-usuario-logado.ps1`
**Prova:** 19/08 09:52 — **APROVADO.** Aceite do A-39 e do A-40. Log transcrito no **anexo A**

**Por que existe:** e exatamente o cenario que falhou **11 vezes seguidas** em 18/08, e o unico que
nunca tinha sido testado — porque as maquinas sempre atualizavam com ninguem logado. Sem usuario na
sessao nao existe worker, sem worker nao existe lock, e o defeito ficava invisivel.

**Suite verde e leitura de codigo nao fecham este aceite. So a maquina fecha.**

**Pre-requisito:** sessao interativa logada com o icone na bandeja, PowerShell elevado, .NET 8 SDK,
espaco em disco (~300MB por versao, e sao duas). O cenario **nao** chama o `teardown.ps1` — ele
preserva a vinculacao.

**Passos:** `.\tests\e2e\scenarios\e8-update-com-usuario-logado.ps1`. Ele confere que voce esta
elevado e que **existe SessionWorker vivo** — sem worker ele aborta, em vez de passar por vacuo.
Sobe um stub local na porta 18081, aponta **so** o endereco de administracao para o stub (os eventos
continuam indo para staging), desliga o SOS temporariamente, reinicia o Service e roda as
assercoes. No `finally` restaura o config e o SOS **como estavam**.

**Passou se, e sao seis coisas:** a versao troca em ate 180s; **nenhum** `CONNECT received` nem
`Worker launched` entre o `UPDATE_APPLYING` e o fim da copia; a linha `worker launch gate closed`
esta presente; nenhum erro de lock nem rollback no `update-script.log`; o Service `Running` no fim;
e **o SessionWorker volta em ate 90s**.

> **O criterio 6 e tao importante quanto o 2.** O portao fecha por TTL de 5 min. Se algo o deixasse
> fechado, a sessao ficaria sem captura — falha silenciosa, pior que o bug original.

**Reprovou se:** `CONNECT received` logo apos o `GOODBYE` (A-39 voltou: o portao nao segura);
`sendo usado por outro processo` (A-40 voltou: o script nao matou o worker); ou nenhum
`gate closed` — nesse caso o binario em uso e anterior a 1.5.7 e o passo de instalacao nao pegou.

**Ao terminar:** **confira o modo SOS.** O cenario restaura o estado anterior, entao se o SOS estava
ligado ele continua ligado — e nao sai sozinho.

---

### AT-06 · Cooldown de 6h sobrevive a restart · 30 min · **raia B**
**Origem:** automatizado `n2-cooldown-sobrevive-restart.ps1` (aceite do A-41)
**Prova:** nenhuma em maquina

**Este defeito quebrou duas vezes, de jeitos opostos** — por isso o cenario existe:
1. Ate a 1.5.6, `Program.Main` apagava a flag de update falho **antes** de o checker le-la. O
   cooldown era codigo morto e o Agent reentrava no ciclo na hora. Foi assim que 11 tentativas
   couberam em 12 minutos.
2. Apagar a flag **so depois** de uma espera de 6h e igualmente ruim: se o Service reiniciar no meio,
   a espera recomeca do zero e a flag nunca e apagada. Numa maquina que reinicia todo dia, o Agent
   nunca mais atualizaria.

**A solucao vigente, conferida por mim no codigo:** a idade e medida **no filesystem**
(`File.GetLastWriteTimeUtc`), a decisao e imediata, e a flag e apagada assim que a janela vence —
independente de quantos reinicios aconteceram no meio (`UpdateCheckerWorker.cs:95-120`).

**Passou se:** apos um update falho, o Agent **espera o restante da janela** contada pelo relogio de
parede; reinicios no meio **nao reiniciam a contagem**; e assim que a janela vence a flag e apagada
e o ciclo volta ao normal.

**Reprovou se:** o Agent reentrar no ciclo imediatamente; ou a espera recomecar a cada reinicio.

---

### AT-07 · Retencao de backups · 30 min · **raia B**
**Origem:** automatizado `n3-retencao-backups.ps1` (aceite do A-42)
**Prova:** nenhuma em maquina

**O defeito era de caminho, nao de valor.** O `appsettings` tem um maximo de backups, mas quem lia
esse campo era o `UpdateApplier` **da Tray** — um caminho que **nao roda no auto-update**. O caminho
que roda de verdade nunca podava nada. Em 18/08 isso virou 3,4GB de tentativas falhas em disco.

**Passou se:** apos updates sucessivos, o numero de backups guardados respeita o maximo
configurado, e a poda acontece **no caminho que o auto-update de fato executa**.

**Reprovou se:** backups acumulando sem limite.

**Se reprovar:** meca o espaco ocupado antes de abrir o achado — o numero e o argumento.

---

### AT-08 · Buffer sobrevive ao update · 30 min · **raia B**
**Origem:** automatizado `n5-buffer-sobrevive-update.ps1`
**Prova:** nenhuma em maquina

**Lacuna sem achado anterior.** O buffer vive separado dos binarios **justamente** para sobreviver
ao update, e o comentario no codigo diz isso com todas as letras. Mas ninguem nunca conferiu.

**Passos:** gere eventos, **nao deixe subirem** (derrube a rede), dispare o update, e confira depois.

**Passou se:** os eventos que ainda nao tinham subido **continuam la** depois do update, e sobem
quando a rede volta.

**Reprovou se:** um unico evento perdido. **Criterio A da linha de corte.**

---

### AT-09 · Artefatos de update: limpeza sem destruir o que esta em curso · 30 min · **raia B**
**Origem:** roteiro 08-18 P13.4
**Prova:** nenhuma

**Duas metades, e uma sem a outra engana:** o faxineiro precisa remover artefato velho **e**
preservar o script de update **recente** — apagar um PS1 que esta prestes a rodar sabota a
atualizacao no pior momento.

**Passou se:** artefatos antigos sao removidos; e um `run-update.ps1` com menos de 30 minutos
**nao e deletado**.

**Reprovou se:** qualquer das duas metades. A segunda e a que valida a falha unitaria conhecida do
`UpdateCheckerWorker`.

---

### AT-10 · Update travado: recuperacao de flag stale · 30 min · **raia B**
**Origem:** automatizados `e3-stuck-ps1-recovery.ps1` e `e4-reboot-mid-update.ps1`
**Prova:** nenhuma em maquina

**Dois automatizados, o mesmo caminho, gatilhos diferentes:** o `e3` simula flag velha com PS1 orfao
e staged; o `e4` simula crash no meio (o processo saiu limpo mas o script nunca rodou).

**Passou se:** o Watchdog detecta o travamento dentro da janela configurada; emite a auditoria de
recuperacao de update travado; **os artefatos sao apagados**; e o Service volta a `Running`.

**Reprovou se:** a maquina ficar em limbo contando falha em silencio; ou os artefatos sobrarem,
porque o proximo ciclo tropeca neles.

> **Cuidado com o `e4`:** ele usa `MANAGER_STALE_MINUTES_OVERRIDE` para encurtar a janela. **Remova
> a variavel ao terminar** — ela e permanente na maquina e nenhum script a apaga.

---

### AT-11 · Flag de update corrompida · 15 min · **raia B**
**Origem:** automatizado `e6-corrupted-flag.ps1`
**Prova:** nenhuma em maquina

**Passou se:** com a flag JSON ilegivel, o leitor cai no plano B (data de escrita do arquivo) e o
Agent segue normal.

**Reprovou se:** excecao nao tratada; ou o Agent parando por causa de um arquivo de controle
corrompido.

---

### AT-12 · O script gerado e ASCII puro · 10 min · raia A
**Origem:** roteiro 08-18 P13.6 (achado A-05) · teste `PowerShellScriptsAreAsciiTests`
**Prova:** nenhuma nesta versao — **estado atual nao verificado**

**So leitura.** Nao muda nada na maquina.

**Por que importa:** caractere fora de ASCII **em comentario passa; dentro de string quebra** o
parser do PowerShell 5.1 — no meio de uma maquina que acabou de receber ordem de atualizar ou
reverter. O A-05 ja apareceu tres vezes, e chegou a impedir **3 dos 7 cenarios E2E** de parsear.

**Passos:** apos um update ou uma volta, abra o script gerado em `C:\ProgramData\ManagerAgent` e
procure caractere fora de ASCII.

**Passou se:** zero caracteres fora de ASCII no script gerado, comentarios inclusive.

**Reprovou se:** qualquer um. **Nao verificado:** havia um em-dash em comentario reportado em
`UpdateApplier.cs:792` em 18/08. **Nao conferi se continua.** Se aparecer, confira o arquivo antes
de abrir achado novo — pode ser o mesmo.

---

### AT-13 · Update a partir de uma versao antiga da frota · 40 min · **raia B**
**Origem:** automatizado `e7-current-prod-version.ps1`
**Prova:** nenhuma em maquina

**Mantido, e nao e redundante com o AT-01.** E o unico cenario que parte de uma versao **muito
atras** da atual — e o canario e exatamente isso: maquina velha da frota atualizando. Uma versao
antiga usa `schtasks`, com o atraso conhecido, entao o tempo esperado e maior.

**Passou se:** o update completa a partir da versao antiga, com tolerancia de tempo maior por causa
do `schtasks`.

**Reprovou se:** o caminho antigo travar. **Este cenario e o mais proximo que temos do primeiro dia
do canario.**

---

### AT-14 · Migracao de config anterior a v1.3.0 · 40 min · **raia B**
**Origem:** achado N-03 (secao 3), levantado no regressivo 1.5.16
**Prova:** **nenhuma, em versao nenhuma, em maquina nenhuma**

**Este e o unico cenario que exercita a migracao de token em texto puro para DPAPI.** O R-04
reescreveu 253 linhas do `ConfigManager.cs`, que e o carregador de config do **Service**, e dentro
dele mora esse ramo. Ele **so dispara em maquina que sobe de uma config velha** — a maquina de
referencia ja esta migrada, e o ramo **nunca vai rodar la, por mais que se reteste**. Hoje esta
coberto por 12 testes unitarios, e so.

**Pre-requisito:** uma maquina (ou VM) com config anterior a v1.3.0 — token em texto puro.

**Passou se:** o Service sobe, **migra o token para DPAPI**, preserva o vinculo e o identificador da
instalacao, e registra a migracao no log.

**Reprovou se:** o Service nao subir; ou subir perdendo o vinculo; ou deixar o token em texto puro.

**Se nao houver maquina candidata:** marque **"nao conferido"**, nunca "aprovado" — e leia a secao 4,
item 6: sem este cenario, a escolha da empresa do canario passa a ser uma decisao com risco
declarado.

---

### AT-15 · A maquina volta sozinha para a versao anterior · 60 min · **raia B** · risco **alto**
**Origem:** C12 do roteiro 1.5.13 (fundido com o Teste 10.5 do @Bucky)
**Prova:** nenhuma pelo caminho automatico — **as duas voltas de 24/08 vieram do recall**

**E a peca que a regra 5.2 do `REGRAS-RELEASE` exige, e ela nunca foi executada pelo caminho
automatico.** Ate 24/08 de manha o mecanismo nem funcionava: quem le a copia de seguranca procurava
uma pasta um nivel acima de onde quem escreve a colocou.

**Pre-requisito:** os cinco passos da preparacao da raia B. **A copia de seguranca tem de conter N
neste momento** — se ainda contiver a versao anterior, o passo 2 nao aconteceu; pare aqui.

**Passos:** guarde a foto de antes. Confirme `sosMode: false`. `Restart-Service ManagerAgent` e
acompanhe o log ao vivo. Espere ate 15 minutos: baixa N+1, aplica, o servico tenta subir e **falha
tres vezes**, o Watchdog dispara a volta, o script para tudo, troca os arquivos e religa.

**Passou se, e sao sete coisas:**
1. A versao instalada voltou a ser **N**.
2. Os dois servicos `Running`.
3. A captura voltou — eventos novos no log depois da volta.
4. Existe uma pasta `bin.failed-...` **ao lado** da instalacao, em `C:\Program Files`, nao dentro.
5. `sosMode` continua **false**, e o contador de falhas de partida voltou a **0**.
6. O `rollback-result.json` **nao existe mais** ao final — foi consumido pelo Watchdog.
7. O audit `UPDATE_ROLLBACK_TRIGGERED` chegou ao backend, **com esse nome, nunca com nome de
   recall**. *(Depende do staging no ar; sem ele, marque "nao conferido", nao "reprovado".)*

> **Sobre o item 6, para nao virar reprovacao falsa.** O `rollback-result.json` **aparece** durante a
> volta e **some** ao final. Sao dois momentos, nao uma contradicao: e um arquivo de recado, escrito
> pelo script e apagado por quem o le. Lidos como uma lista unica de conferencia final, os dois
> criterios se contradizem e alguem marcaria reprovacao.

**Reprovou se:** o SOS ligou; ou a versao instalada continua sendo a N+1; ou o servico nao volta a
rodar. **Nos tres a maquina esta sem capturar.**

**Se reprovar — nesta ordem:** leia o motivo **antes** de consertar, porque reinstalar apaga a
evidencia (`rollback-script.log`, `rollback-result.json`, `watchdog-state.json`, copia da pasta de
logs). Depois reinstale a rede de seguranca. **Confira o `sosMode` depois de reinstalar** — ele nao
sai sozinho.

| Sintoma no log | O que significa |
|---|---|
| "backup nao encontrado" | o endereco da copia de seguranca voltou a divergir — e o defeito original |
| erro de arquivo em uso | algo travou os binarios; o script deveria ter parado o Watchdog e os workers antes. Ver AT-19 |
| o script nunca rodou | o disparo falhou. Depois de 15 min sem resposta isso vira erro e liga o SOS, de proposito |
| SOS ligado sem registro de qual versao quebrou | e o buraco conhecido do caminho "sem backup": a captura da versao nao chega a ser gravada. Mapeado, nao corrigido |

---

### AT-16 · A auto-recuperacao do SCM voltou no lugar · 5 min · raia A
**Origem:** C21 do roteiro 1.5.13 (Teste 10.6 do @Bucky, o mesmo que o E2E E6 cobriria)
**Prova:** 24/08 16:07 — **PASSOU** depois de duas restauracoes reais

**Rode logo depois do AT-15, antes de qualquer outra coisa.** E so leitura, leva 30 segundos, e mede
um estrago que **nao aparece em lugar nenhum ate a proxima vez que o servico cair**.

**Por que importa:** o script de volta **desliga a auto-recuperacao dos dois servicos** para
conseguir trocar os arquivos. Se ele falhar em restaurar, a maquina fica **sem auto-recuperacao** —
pior do que o defeito que o rollback conserta. Ha um `finally` que cobre, e ha teste unitario da
ordem; **faltava a prova em maquina, e e esta.**

**Passos:** `sc.exe qfailure ManagerAgent` e `sc.exe qfailure ManagerAgentWatchdog`.

**Passou se:** os **dois** servicos voltaram a `restart 5000 / 10000 / 30000` com reset 86400 —
exatamente o que o CS-01 mediu antes de tudo comecar.

**Reprovou se:** qualquer um dos dois sem acoes de recuperacao, ou com valores diferentes dos do
CS-01.

**Se reprovar:** achado serio, @Bucky no mesmo dia. Anote se o AT-15 passou ou reprovou junto: se
ele falhou no meio, o `finally` pode nao ter chegado a rodar, e isso muda o diagnostico.

> **Ressalva honesta sobre a prova de 24/08:** as duas voltas daquele dia vieram do **recall**, nao
> de falha de partida. As duas origens disparam **o mesmo script de restauracao**, e e o script que
> mexe no SCM — entao a cobertura vale. O que continua sem prova em maquina e a **entrada** pelo
> caminho automatico, e essa e o AT-15.

---

### AT-17 · Nao reinstala a versao quebrada · 40 min · **raia B** · risco alto
**Origem:** C13 do roteiro 1.5.13 (fundido com o Teste 10.7 do @Bucky)
**Prova:** **parcial**, 24/08 15:58 — a 1.5.16 entrou numa maquina com sticky da 1.5.15 ativa

Era o buraco do R-01: a maquina voltava, e no ciclo seguinte instalava **de novo** a mesma versao
que a tinha quebrado. Laco infinito.

**Pre-requisito:** AT-15 aprovado, e o servidor de teste **continuando a oferecer a N+1 quebrada**.
Se voce desligar o stub depois do AT-15, este cenario nao tem o que provar — e isso e de proposito:
e o cenario de madrugada em que ninguem apertou o botao de pausa.

**Passos:** com a maquina na **N** restaurada, force uma nova checagem com `Restart-Service
ManagerAgent`, espere 5 minutos e leia. **Repita mais duas vezes, com intervalo** — e teste de laco,
uma vez so nao prova.

> **Alternativa que exercita outro gatilho.** Da para encurtar a janela com o override de intervalo
> no `config.json`. Os dois caminhos servem e **exercitam gatilhos diferentes**: o reinicio testa a
> checagem que o Service faz **ao subir**; o override testa o **laco periodico**. Se der tempo, faca
> os dois — foi o gatilho de boot que o RC-06 mostrou ser o perigoso. **Devolva o valor original ao
> terminar**, senao a maquina fica perguntando fora da cadencia de producao e as medicoes seguintes
> nao valem.

**Passou se, e sao quatro coisas:** nas **tres** checagens a N+1 foi barrada, com aviso no log
dizendo **qual versao e ate quando** — o bloqueio dura 24h; **nenhum download comecou** (o pacote
tem ~310MB: download iniciado ja e reprovacao mesmo que a instalacao seja barrada depois); a maquina
continuou na N capturando; e o **Service seguiu `Running` o tempo todo**, nas tres.

**Reprovou se:** a maquina baixar a N+1 de novo, em qualquer uma das tres.

**Se reprovar:** desligue o stub **imediatamente**, senao a maquina entra em laco. E o R-01 nao
fechado, e e bloqueador de producao.

**Nota que evita leitura errada:** a protecao depende de a maquina ter **gravado qual versao
quebrou**. Numa maquina que nunca reverteu antes — que era o caso de toda a frota — esse campo
nascia vazio e a protecao nao dispararia. Quem fecha isso e o R-02, e e o AT-18 que confere. **Se o
AT-17 reprovar, leia o AT-18 antes de concluir qualquer coisa:** pode ser que a protecao esteja
certa e o que faltou foi o registro.

> **Uma afirmacao antiga que NAO foi absorvida.** Circulava que "sem o R-01 a maquina baixa 310MB,
> reinstala a quebrada, cai de novo — e ai a sticky **impede** o segundo rollback". A primeira
> metade e o risco real e ficou. **A segunda nao se sustenta:** o bloqueio de 24h impede **instalar**
> a versao ruim, e nao impede a maquina de **voltar** dela. Fica como **pergunta para o @Bucky**,
> nao como criterio.

**Como a guarda de fato funciona, conferido por mim:** ela so barra quando **as duas** coisas valem
— a sticky esta no futuro **e** a versao oferecida e **a mesma** que quebrou
(`UpdateCheckerWorker.cs:555-567`). **A trava e por versao, nao congelamento.** Foi por isso que a
1.5.16 entrou em 24/08 numa maquina com sticky da 1.5.15 ativa: a 1.5.15 continua barrada, a 1.5.16
passou. Quem ler `stickyVersion` no estado e concluir "a maquina esta travada" vai errar.

---

### AT-18 · O registro guarda qual versao quebrou · 5 min · raia A
**Origem:** C14 do roteiro 1.5.13 (era o R-02)
**Prova:** 24/08 — **PASSOU 2x.** As pastas `bin.failed-1.5.14+793e743...` e
`bin.failed-1.5.15+57ae0c6...` estao na maquina, com o numero certo e nao `unknown`

**So leitura.** Vale tendo o AT-15 passado ou reprovado.

**Passos:** leia o `watchdog-state.json` e liste `C:\Program Files -Filter 'bin.failed-*'`.

**Passou se:** o campo da ultima versao quebrada traz **N+1**; o nome da pasta e
`bin.failed-<N+1>-<data>`; e o bloqueio esta carimbado 24 horas a frente.

**Reprovou se:** o campo vazio, ou a pasta saindo como `bin.failed-unknown-...`.

**Detalhe que nao e defeito:** ao abrir o arquivo no bloco de notas, o sinal `+` do numero da versao
aparece escapado, com uma sequencia estranha no lugar. E so a forma de gravar; o programa le certo.
**Nao registre como achado.**

**Se reprovar:** anote qual foi o desfecho do AT-15. Se ele terminou no caminho "sem backup", este
resultado **ja e esperado e nao e defeito novo** — e a lacuna conhecida do R-02, ainda aberta com o
@Tony.

---

### AT-19 · A volta acontece com dois usuarios logados · 60 min · **raia B** · risco alto
**Origem:** C22 do roteiro 1.5.13 (Teste 10.9 do @Bucky, **com os criterios corrigidos**)
**Prova:** nenhuma

**Por que existe:** o script de volta precisa **matar os `SessionWorker`** para liberar os arquivos
travados. O A-40 registra que esquecer isso ja derrubou o rollback em **11 tentativas seguidas**.
**Este e o unico cenario que exercita esse passo com duas sessoes vivas.**

**Pre-requisito:** AT-15 aprovado (repita a preparacao para ter de novo uma N+1 quebrada) e as duas
contas logadas, cada uma com o seu worker e com atividade real por 2 minutos.

**Passou se, e sao quatro coisas:**
1. A volta **conclui** — nao falha por arquivo em uso.
2. **Um worker por sessao** volta a subir depois.
3. **Nenhum evento perdido:** o buffer de cada sessao sobe depois do retorno, cada um com o seu dono.
4. **Nenhum LOGOUT** gravado para as duas pessoas. **Ninguem saiu.** Uma volta de versao nao e um fim
   de expediente e nao pode aparecer no relatorio como se fosse.

> **CRITERIO CORRIGIDO.** O original pedia so *"nenhum LOGOUT gravado"*. Isso diz o que **nao** pode
> aparecer e nao diz o que **deve** — e um teste que so proibe passa por omissao. O script **mata**
> os workers: uma `SESSAO_INTERROMPIDA` nas duas sessoes seria **legitima e esperada** aqui, pelo
> mesmo motivo que seria num reboot. **Portanto:** `SESSAO_INTERROMPIDA` pode aparecer; **LOGOUT nao
> pode**; e nenhuma sessao pode ficar aberta com evento posterior ao encerramento. Sem essa terceira
> linha, o cenario deixaria passar exatamente o defeito do A-65 que o MU-05 caca.
>
> **Ressalva, e ela e minha:** com o CS-10 em aberto, a `SESSAO_INTERROMPIDA` **provavelmente nao vai
> sair** aqui tambem — pelo mesmo ramo morto. **Ausencia dela nao e criterio de reprovacao deste
> cenario**; registre e cruze com o CS-10.

**Reprovou se:** a volta falhar por arquivo em uso (o A-40 de volta); LOGOUT para qualquer das duas
contas; ou um worker nao voltar.

---

### AT-20 · Volta sem copia de seguranca: entra em SOS, e o esperado · 45 min · **VM** · **termina com a maquina parada**
**Origem:** C23 do roteiro 1.5.13 (Teste 10.8 do @Bucky)
**Prova:** nenhuma

> **Este cenario termina com a maquina parada. E o resultado esperado, nao uma falha.** O unico
> caminho de volta e alguem reinstalar a mao. **Preferencia minha, e nao mudou: rodar em VM, nunca
> na maquina de trabalho.** Se o Marcos autorizar por escrito na maquina real, que seja o ultimo ato
> do dia, com as duas redes de seguranca conferidas antes.

**O caso real:** cliente novo, que instalou e nunca atualizou. Nao tem copia de seguranca, logo nao
tem para onde voltar. **O produto nao pode travar nem lancar excecao** — tem de desistir de forma
limpa, ligar o SOS e registrar o motivo.

**Nao confunda com o RC-05.** Mesmo ponto de partida, desfecho **oposto**, e essa e a graca dos dois:

| | Caminho | Sem backup, o esperado e |
|---|---|---|
| **AT-20** | rollback **automatico** (a versao nova nao subiu) | **liga SOS** — a maquina esta quebrada de verdade e precisa de gente |
| **RC-05** | **recall** (ordem do backend) | **NAO liga SOS** — a maquina esta saudavel, so nao pode obedecer |

**Se os dois se comportarem igual, a origem nao esta sendo distinguida — e esse e o defeito que a
dupla procura.**

**Passou se, e sao cinco coisas:** o log registra que o backup nao foi encontrado; `sosMode: true`
com `sosSince` preenchido; **nenhuma excecao nao tratada**; o **Watchdog continua de pe** (o
processo nao morre); e os ciclos de update seguintes sao **pulados com o motivo no log**, nao em
silencio.

**Reprovou se:** excecao nao tratada; o Watchdog cair; ou os ciclos serem pulados sem dizer por que.

**Procedimento de saida — OBRIGATORIO:** guarde a evidencia **antes** de consertar (reinstalar apaga
tudo). Reinstale com a rede de seguranca. **Confira o `sosMode` depois de reinstalar** — se
continuar `true`, troque para `false` e reinicie os dois servicos. **So encerre depois de ver evento
novo subindo no log.**

---

### AT-21 · Maquina desligada no meio da volta · 45 min · **VM** · **termina com a maquina parada**
**Origem:** C24 do roteiro 1.5.13 (Teste 10.10 do @Bucky)
**Prova:** nenhuma

**O pior momento possivel.** A maquina perde energia exatamente enquanto os arquivos estao sendo
trocados. O boot seguinte tem de **concluir** o que ficou pela metade, ou desistir com motivo — o
que nao pode e ficar em limbo, contando falha em silencio.

**Procedimento:** desligar na tomada assim que o `rollback-script.log` registrar a parada dos
servicos.

**Passou se, e sao quatro coisas:**
1. No boot, o Watchdog **registra rollback pendente e conclui**, ou aguarda dentro da folga de 15
   minutos.
2. Passados 15 minutos sem resultado, **entra em SOS** — e nao fica contando falha em silencio.
3. O `watchdog-state.json` esta **legivel** — escrita atomica, nunca truncado, sem temporario orfao.
4. **Nenhum evento do colaborador perdido.**

> **O item 4 nao e conferivel no log.** Provar que nenhum evento se perdeu exige comparar o que a
> maquina bufferizou com o que chegou do outro lado, e isso e **conferencia no banco** — o mesmo
> pre-requisito do BK-02. **Enquanto o acesso ao banco nao sair, o item 4 se marca como "nao
> conferido", nunca como aprovado.** Marcar aprovado sem o banco e afirmar exatamente o que nao foi
> olhado.

**Reprovou se:** o `watchdog-state.json` sair truncado ou ilegivel; ou a maquina ficar contando
falha sem entrar em SOS nem concluir.

**Procedimento de saida:** o mesmo do AT-20.

---

### AT-22 · Telemetria do Watchdog chega ao backend · 10 min · raia A
**Origem:** item 5 da linha de corte · roteiro 08-18 P13.3 · marco 9.6
**Prova:** 24/08 15:42:59 — **202** (`item5-telemetria.txt`). Antes disso, 401 as 14:56

**Oportunista: sai de graca no primeiro recall ou rollback que acontecer.** Nao peca execucao
dedicada.

**Passou se:** apos um rollback ou recall, o reporter HTTP **posta a auditoria** e recebe 202.

**Reprovou se:** 401 persistente (era o defeito), ou nenhuma tentativa de post.

> **Ressalva que precisa estar escrita:** o 202 que fechou este item saiu com o Watchdog na
> **1.5.15**. Desde que a 1.5.16 subiu (15:58:16) **nao houve recall nem rollback, entao o canal nao
> foi exercitado nesta versao.** Eu aceito assim mesmo, porque a arvore e identica (secao 2) — mas
> **nao escrevo que "foi provado na 1.5.16", porque nao foi.**

---

# 15. RECALL — 9 cenarios · ~3h

Mandar de volta quem **ja tem** a versao ruim instalada. Diferente do botao de pausa, que so impede
novas entregas.

## Como a ordem viaja de um lado ao outro — vale para todos os nove

Conferido por mim no codigo. **E a primeira coisa a olhar em qualquer cenario deste bloco:**

- Quem **le** a resposta do backend e o **Service**, no ciclo de 6h.
- Quem **executa** a restauracao e o **Watchdog**, no ciclo de 60s.
- O Service so **marca** os campos de recall pendente no `watchdog-state.json`.

Depois de o Service ver o recall, some **ate 60s** para a restauracao comecar. **Se o pedido nao
aparecer no JSON, o problema esta no Service; se aparecer e nada acontecer, esta no Watchdog.**
Conferir o `watchdog-state.json` e o passo 1 de todo cenario de recall.

---

### RC-01 · Recall de frota: a maquina volta sozinha · 30 min · raia A · risco medio
**Origem:** C11 do roteiro 1.5.13 (fundido com R10.11.1, .3 e .4 do @Bucky)
**Prova:** 24/08 — **PASSOU 2x** (`C11-recall.txt`), 42s e 12s sem captura

**Pre-requisito:** staging no ar, e a maquina rodando uma versao **cujo Agent entende recall**. A
1.5.16 entende.

**Passos:** peca ao @Vision para disparar `POST /api/admin/fleet/revogar-versao` com a versao
instalada e um motivo escrito — **o motivo e obrigatorio, minimo 10 caracteres**, e vai parar no log
da maquina do cliente explicando por que ela voltou sozinha. Espere o proximo ciclo (ate 6h, ou
reinicie o Service para forcar). **Confira primeiro o `watchdog-state.json`.** Espere ate 60s e leia
os dois logs.

**Passou se:** a maquina volta para a versao anterior, **ou** recusa com motivo registrado — e em
nenhum dos dois casos entra em SOS. O motivo escrito pelo operador aparece no log em **dois
momentos**: quando a resposta chega e quando o Watchdog age.

**Reprovou se:** entrar em SOS por causa de um recall; ou repetir a ordem em laco a cada reinicio.

**Se reprovar:** desligue o recall no backend **antes** de mexer na maquina, ou o laco continua.

**Tres limites que mudam o que este teste significa — e nao sao reprovacao:**
1. **O recall so alcanca maquina cujo Agent ja entende o comando.** Quem roda versao anterior
   **ignora**, sem erro e sem aviso. O recall protege as versoes lancadas **depois** de ele existir;
   **ele nunca desfaz a versao que estiver rodando no dia em que for publicado.**
2. **Maquina em SOS nunca recebe recall** (secao 3).
3. **Volta uma versao, so** (secao 3).

---

### RC-02 · Quem o recall NAO alcanca · 40 min · raia A · **desfazer obrigatorio**
**Origem:** C20 do roteiro 1.5.13 (junta R10.11.2, .6 e .16 do @Bucky)
**Prova:** nenhuma

**Vem antes dos outros de proposito:** e o mais barato, quase nao mexe na maquina, e evita a leitura
errada mais provavel do bloco inteiro — concluir que "o recall nao funcionou" quando ele fez
exatamente o que devia.

| Caso | Situacao | Esperado | Por que |
|---|---|---|---|
| **a** | Agent **anterior ao recall** | **ignora, sem erro e sem aviso**, e segue o fluxo normal de update | O campo e novo. Um Agent que nao o conhece nao quebra por causa dele — e o requisito de compatibilidade |
| **b** | Versao **diferente** da revogada | **ignora** | O recall e por versao. Revogar a 1.5.13 nao pode mexer em quem esta na 1.5.14 |
| **c** | Maquina em **modo SOS** | **nao pergunta ao backend, logo nao ouve o recall** | A guarda retorna antes da chamada. Conferido: `UpdateCheckerWorker.cs:234` |

**Passos:** rode os casos **a** e **b** primeiro (nao mudam nada). O **caso c** por ultimo, porque
muda a maquina: guarde o estado, ponha `sosMode: true`, reinicie o Service, peca a revogacao da
versao instalada e observe por dois ciclos.

**Passou se:** nos tres casos **nada acontece na maquina** e nenhum erro aparece no log. No caso c,
o log tem de mostrar que a maquina **nem chegou a perguntar** ao backend.

**Reprovou se:** o caso a gerar erro ou excecao — compatibilidade quebrada; ou o caso b disparar uma
volta de versao — recall pegando quem nao devia.

**OBRIGATORIO ao terminar o caso c:** desfaca o SOS. **Ele nao sai sozinho.** Troque para `false`,
reinicie os dois servicos e **confirme** antes de seguir. Se esquecer, a maquina fica instalada, sem
procurar update, e **todo cenario seguinte mede errado**.

---

### RC-03 · Onde o recall aparece: painel, audit e logs · 20 min · raia A
**Origem:** C18 do roteiro 1.5.13 (junta R10.11.5, .13, .14 e .15 do @Bucky)
**Prova:** nenhuma

**So leitura.** Roda **depois** de qualquer cenario de recall ter acontecido, sobre a evidencia que
ele deixou.

**Passou se, e sao sete coisas:**
1. Os eventos no painel se chamam `UPDATE_RECALL_TRIGGERED`, `_SKIPPED` ou `_FAILED` — **nunca**
   `UPDATE_ROLLBACK_TRIGGERED` para um recall.
2. Os tres saem em nivel **aviso**.
3. O `_FAILED` sai como **aviso, nao erro** — `ERROR` neste repo e o nivel do SOS, e recall que
   falha **nao liga SOS**.
4. O audit do disparo traz **autor e motivo** de quem revogou.
5. O motivo digitado pelo operador aparece no log **do Service** (quando a resposta chegou) **e do
   Watchdog** (quando ele agiu).
6. Quando o backend nao manda motivo, os dois logs dizem isso **com todas as letras**. Silencio nao
   serve.
7. O `run-rollback.ps1` gerado continua **ASCII puro**, **sem** o texto do operador.

> **Sobre o item 7, para nao virar achado errado:** o motivo *nao* estar no script e proposital. O
> motivo e texto livre com acento — o exemplo do proprio contrato e *"Captura de tela saindo em
> branco na 1.5.13"* — e dentro do PS1 ele quebraria o parser do PowerShell 5.1 (A-05), no meio de
> uma maquina que acabou de receber ordem de reverter. **Motivo no log do C# e no audit; o PS1
> continua ASCII.** Se voce achar o motivo dentro do PS1, **isso** e o achado.

> **Uma instrucao antiga foi REMOVIDA daqui, e vale saber por que.** Circulava a orientacao de
> "consultar o painel sem filtro de nivel", porque os tres eventos chegavam como rotineiros e
> sumiriam de qualquer consulta filtrada. **Isso foi corrigido em 24/08** pelo commit `e86124d`: os
> tres entraram no mapa de tipos conhecidos como aviso. Se aquela instrucao tivesse sido absorvida
> como estava, mandaria o testador **desligar exatamente o filtro que agora prova que a correcao
> funcionou.** O item 2 e o que a substitui: conferir o nivel **e** parte do teste.

**Reprovou se:** um evento de recall sair com nome de rollback (item 1 — mistura os dois caminhos e
estraga o criterio de "telemetria zero" do rollout); ou os eventos nao aparecerem numa consulta
filtrada por aviso (item 2 — a correcao de 24/08 nao pegou).

---

### RC-04 · Guarda A: backup na mesma versao · 40 min · **raia B** · risco medio
**Origem:** C15 do roteiro 1.5.13 (R10.11.7 do @Bucky, **com o preparo corrigido**)
**Prova:** nenhuma

**Por que existe:** sem esta guarda, uma maquina cujo backup tem a **mesma** versao da instalada
**reverteria a cada ciclo, para sempre** — copiando ~160MB e deixando uma pasta `bin.failed-*` de
~160MB a cada volta. **E um cenario de disco cheio**, nao so de teste falhando.

> **ESTE CENARIO DESARMA O AT-15 SE RODAR NA HORA ERRADA.** O preparo original mandava *"copiar a
> instalacao por cima do backup"*. Isso **destroi o `bin.previous`**, que e a unica copia de
> seguranca da maquina e e exatamente para onde o AT-15 retorna. Rodar antes dele apagaria a rede
> de seguranca do teste que bloqueia producao — **em silencio**, e sem ninguem perceber ate o AT-15
> falhar por "backup nao encontrado" e o diagnostico apontar para o defeito errado.
>
> **Duas correcoes:** (1) este cenario vai para **depois** de AT-15, AT-17 e AT-18; (2) o preparo
> **guarda e devolve** o backup, em vez de sobrescreve-lo.

**Preparo:**

```powershell
Copy-Item 'C:\Program Files\bin.previous' 'C:\Program Files\bin.previous.guardado' -Recurse
Copy-Item 'C:\Program Files\ManagerAgent\*' 'C:\Program Files\bin.previous\' -Recurse -Force
```

**Passos:** confirme que backup e instalacao estao **na mesma versao**. Peca a revogacao dessa
versao. Espere o ciclo. Leia o log e liste `C:\Program Files`.

**Passou se, e sao cinco coisas:** **nenhum servico para**; **nenhum `bin.failed-*` novo** aparece; o
log traz a recusa pela guarda A; o audit sai como pulado com motivo de backup na mesma versao; e a
maquina **continua capturando**.

**Reprovou se:** a maquina disparar a volta mesmo assim. Nesse caso desligue o recall no backend
**imediatamente**, antes que o ciclo seguinte repita.

**OBRIGATORIO ao terminar — devolva a copia de seguranca de verdade:**

```powershell
Remove-Item 'C:\Program Files\bin.previous' -Recurse -Force
Move-Item 'C:\Program Files\bin.previous.guardado' 'C:\Program Files\bin.previous'
(Get-Item 'C:\Program Files\bin.previous\ManagerAgent.Service.exe').VersionInfo.ProductVersion
```

A ultima linha tem de mostrar a versao **anterior**, nao a instalada. Se mostrar a instalada, **a
maquina esta sem rede de seguranca.**

---

### RC-05 · Guarda B: sem backup nao liga SOS · 30 min · **raia B**
**Origem:** C16 do roteiro 1.5.13 (R10.11.8 do @Bucky)
**Prova:** nenhuma

Protege a frota contra o pior efeito colateral possivel do recall: **uma maquina que, por nao ter
para onde voltar, sai do ar e deixa de receber a correcao que ia consertar o problema que motivou o
recall.**

**Preparo:** `Move-Item 'C:\Program Files\bin.previous' 'C:\Program Files\bin.previous.guardado'`

**Passos:** confirme que `bin.previous` **nao existe**. Peca a revogacao da versao instalada. Espere
o ciclo. Leia o log do Watchdog e o `watchdog-state.json`.

**Passou se, e sao quatro coisas:** o log traz a recusa pela guarda B; o audit sai como pulado com
motivo de ausencia de backup; **`sosMode` continua `false`**; e a maquina segue capturando e segue
elegivel ao auto-update.

**Conferir o `sosMode` e obrigatorio.** Nao vale "o log disse que recusou".

**Reprovou se:** `sosMode` virar `true`. E o oposto do desenho: SOS aqui tiraria a maquina do
proprio conserto. **Compare com o AT-20** — mesmo ponto de partida, desfecho oposto, de proposito.

**OBRIGATORIO ao terminar:** devolva o `bin.previous`. Sem isso o AT-15 fica sem rede de seguranca.

---

### RC-06 · Guarda C: o recall nao se repete a cada reinicio · 30 min · raia A
**Origem:** C17 do roteiro 1.5.13 (junta R10.11.9 e .10 do @Bucky)
**Prova:** nenhuma

**Juntos porque um sem o outro engana:** o primeiro prova que a ordem **para de sair**, o segundo
prova que ela **nao para demais**.

**Por que existe:** o gatilho perigoso **nao** e o ciclo de 6h. E a checagem que o Service faz
**assim que sobe**. Numa maquina que reinicia em laco — que e exatamente a maquina que acabou de
falhar um recall — sem um carimbo a ordem sairia a cada boot.

**Pre-requisito:** um recall ja recusado (RC-04 ou RC-05).

**Parte 1:** reinicie o Service **duas ou tres vezes seguidas**, dentro de 1 hora. **Passou se:** o
log diz guarda C, o pedido **nao** e remarcado, e os campos de ultima tentativa e ultima versao
tentada estao preenchidos no JSON.

**Parte 2:** com a guarda C carimbada para a versao X, ponha a maquina numa versao **Y, tambem
revogada**. **Passou se:** a ordem sai **na hora**, sem esperar a janela de 1 hora. **A chave da
guarda e por versao, nao global.**

**Reprovou se:** a parte 1 mostrar a ordem saindo a cada boot (laco); **ou** a parte 2 mostrar a
maquina presa por 1 hora numa segunda versao ruim. **As duas metades sao reprovacao** — e por isso
viraram um cenario so.

---

### RC-07 · Recall que falha nao liga SOS · 40 min · **raia B** · risco medio-alto
**Origem:** C19 do roteiro 1.5.13 (R10.11.11 do @Bucky)
**Prova:** nenhuma

**E o par assincrono do RC-05, e o unico jeito de exercita-lo e com o script rodando de verdade.**
No RC-05 a maquina recusa **antes** de tentar; aqui ela **tenta, e falha no meio**. Sao dois
caminhos de codigo diferentes, e so este prova o segundo.

**Risco:** a restauracao vai falhar de proposito. O criterio e justamente que a maquina **nao** saia
do ar por causa disso — mas se ela sair, e uma reprovacao com a maquina parada. Tenha as duas redes
de seguranca a mao.

**Passos:** force a restauracao a falhar — por exemplo, deixando um arquivo do `bin.previous`
travado por outro processo. Dispare o recall e deixe acontecer. Reinicie a maquina. No boot
seguinte, leia o log e o `watchdog-state.json`.

**Passou se, e sao tres coisas juntas:** o log diz que o recall nao recuperou a maquina; o audit sai
como falhou; e **`sosMode` continua `false`**.

**Reprovou se:** `sosMode` virar `true`. **A diferenca entre este cenario e o AT-15 e exatamente
esta:** falha de rollback automatico **liga** SOS, e esta certo; falha de recall **nao liga**, e
tambem esta certo. **Se os dois se comportarem igual, a origem nao esta sendo distinguida na volta —
e esse e o defeito que este cenario procura.**

**Ao terminar:** solte o arquivo travado, confira o `sosMode` e o estado dos dois servicos.

---

### RC-08 · Quanto tempo a ordem leva para chegar · 30 min de leitura · raia A · **DEVE REPROVAR**
**Origem:** limitacao 4 do C11, promovida a cenario proprio
**Prova:** nenhuma em maquina — **o caminho esta conferido no codigo**

**Por que virou cenario:** enquanto era so uma nota de rodape, ninguem cronometrava. E o numero
importa: e a diferenca entre "a frota volta hoje" e "a frota volta amanha".

**O que o codigo faz, conferido por mim** (`UpdateCheckerWorker.cs`, bloco de cooldown):
- A maquina pergunta ao backend a cada **6 horas**.
- Se existir uma flag de update falho **recente**, o Service **dorme o restante da janela antes de
  fazer a primeira pergunta** — `await Task.Delay(restante)`. Ele **nem chega a falar** com o
  backend.
- Na pior combinacao — update falhou ha 5 minutos, operador revoga a versao agora — a ordem pode
  demorar **quase o dobro do pior caso previsto**.

**E justamente a maquina que acabou de falhar um update que e a candidata ao recall.**

**Passos:** cronometre do disparo do RC-01 ate a maquina agir, com e sem flag de update falho
recente.

**"Passou" hoje significa reprovar:** o atraso de ate 6h + ate 6h **e o comportamento atual**.
Registre o tempo medido e anexe ao chamado. **Nao abra achado novo.**

**Reprovaria de verdade se:** a ordem nunca chegasse; ou o Service nao registrasse por que esta
esperando — a espera silenciosa e que faz o operador achar que o recall nao funcionou.

---

### RC-09 · `pausar-versao` sem escopo de sistema operacional · 20 min · raia A · **DEVE REPROVAR**
**Origem:** cenario novo, conferido no codigo do backend em 24/08
**Prova:** nenhuma — **conferido no arquivo**

**O que esta errado, e eu li o DTO.** O `PausarVersaoBodyDto` tem **dois campos: `versao` e
`motivo`**. Nao tem `sistemaOperacional`. **Pausar a "1.5.13" pausa o Agent Windows e o Agent
Android ao mesmo tempo** — sao duas linhas diferentes na tabela, e uma unica chamada acerta as duas.

O `revogar-versao` **ganhou** o campo em 24/08, e obrigatorio, com o caminho "sem escopo"
**removido, nao desativado**. O racional esta escrito no proprio DTO do recall: *um recall e
disparado sob pressao, no meio de um incidente; um padrao que amplia o alcance em silencio significa
que a mao que erra digitando revoga tambem o build que estava bom.* **O `pausar-versao` nao ganhou o
mesmo tratamento**, e isso tambem esta escrito la.

**Passos:** no Swagger de staging, pause uma versao que exista para mais de um sistema operacional.
Confira no banco quais linhas foram afetadas.

**"Passou" hoje significa reprovar:** o escopo amplo **e o comportamento atual**. Registre e anexe
ao chamado da @Shuri. **Nao abra achado novo.**

**Este cenario tambem cobre o kill switch de update** — se a pausa impede novas instalacoes da
versao pausada, o kill switch funciona. **Reprovaria de verdade se** a pausa nao impedisse novas
entregas.

---

# 16. BACKEND E INGESTAO — 4 cenarios · raia A · ~1h30

O que sai da maquina tem de chegar inteiro do outro lado. Sem este bloco, todos os anteriores
provam so metade do caminho.

---

### BK-01 · Um item ruim nao derruba o lote · 25 min · raia A · **OBRIGATORIO**
**Origem:** item 1 da linha de corte · C9 do roteiro 1.5.13 · R1 do regressivo 1.5.16
**Prova:** nenhuma — **e o unico item da linha de corte sem prova em maquina**

**O que se prova, em uma frase:** um lote com um evento invalido no meio **e aceito**, os eventos
bons **entram**, e o invalido **volta identificado**. Hoje, sem a correcao, o lote inteiro seria
recusado e depois de cinco recusas o Agent **descarta ate 100 eventos** — perda silenciosa de dado
do colaborador.

**Pre-requisito:** o codigo do backend commitado e o staging no ar. Em 24/08 os dois cairam:
`8217492` commitado, `HEAD dccd1a0` em `staging`, arvore limpa; `GET /api/health` respondendo 200
com `database: ok`.

> **Um detalhe que muda como voce le um resultado ruim.** O health de staging responde
> `version: "unknown"` — **o ambiente nao diz qual build esta rodando** (e o BK-03). O que da para
> afirmar e circunstancial e e forte: o merge foi as **13:46:09** e o processo no ar subiu as
> **13:48:38**, 2min29s depois. **Se o teste der 400, isso NAO e automaticamente reprovacao** — pode
> ser build velho. Confirme com o @Vision qual commit esta no ar **antes** de registrar reprovacao.

**Endereco e formato — conferidos no codigo do backend, nao supostos:**
- `POST https://staging-api-events.imanagerportal.com/api/agent/events`
- Autenticacao: `Bearer <device token>`. **A chave de ativacao da empresa NAO serve** — este servico
  so aceita o JWT de dispositivo.
- Resposta esperada: **202**, com `motivosIgnorados`, cada entrada com `indice` (posicao no lote
  **original**), `tipoEvento` e `motivo`.

**Passo 1 — abrir o token da propria maquina** (DPAPI, escopo de maquina). Campos confirmados em
`ConfigLocal.cs:46` e `ConfigManager.cs:378-380`:

```powershell
Add-Type -AssemblyName System.Security
$cfg   = Get-Content 'C:\ProgramData\ManagerAgent\config.json' -Raw | ConvertFrom-Json
$blob  = [Convert]::FromBase64String($cfg.deviceTokenDpapi)
$plain = [System.Security.Cryptography.ProtectedData]::Unprotect($blob, $null, [System.Security.Cryptography.DataProtectionScope]::LocalMachine)
$token = [System.Text.Encoding]::UTF8.GetString($plain)
"token com $($token.Length) caracteres"
```

> **O token vive na variavel `$token` desta janela e morre quando voce fechar.** Nao imprima, nao
> cole no chat, nao salve na pasta de evidencia.

**Passo 2 — montar o lote: dois eventos bons e um ruim no meio.** O evento ruim tem de ser
**valor com o tipo errado** — `nomeProcesso = 123`, quando o campo e texto.

> **Por que nao "campo faltando".** Campo faltando **o backend aceita**: os campos daquele evento sao
> opcionais e o validador descarta chave desconhecida. Isso ja foi medido, e ate um teste nasceu
> errado por causa disso. **Se voce trocar por campo faltando, o lote passa inteiro e voce vai
> concluir que o teste passou sem ter testado nada.**

**Passo 3 — enviar e salvar a resposta inteira** em `$ev\R1-item1.txt`, **incluindo o corpo**.

**Passou se, e sao quatro coisas juntas:**
1. A resposta e **202**.
2. `totalEventos` = **2** — os dois bons entraram; o campo conta o que foi **gravado**, nao o que
   foi enviado.
3. `motivosIgnorados` tem **exatamente uma** entrada.
4. Nessa entrada, `indice` = **1** — o indice do item ruim no lote **original**, nao 0 nem 2 — e o
   motivo cita o campo `dados.nomeProcesso`.

**O item 4 e o coracao do teste.** O indice certo foi a parte mais delicada da correcao: sem ele, o
backend devolveria o numero do item vizinho e ninguem conseguiria dizer qual evento caiu. **Se vier
202 mas com o indice errado, e reprovacao**, nao detalhe.

**Reprovou se:** 400 com o lote inteiro recusado — **antes de registrar**, confirme o commit no ar;
ou 202 com o indice apontando para o item errado, e essa e a que interessa. **401 nao e reprovacao
do item 1**: o token expirou ou nao abriu.

**Se o token falhar (401 ou erro ao abrir):** o mais provavel e token vencido — o Service renova a
cada ~2h. Espere a renovacao (procure `Device JWT renovado com sucesso` no log do dia) e repita.
**Nao invente outro campo do `config.json` no chute.** Alternativas, da mais barata para a mais
cara: o Swagger de staging; pedir **o comando pronto** a @Shuri, que tem o lote misto de teste
(`test/parity/fixtures/batch-misto.json`) — **peca o comando, nunca o token**; ou o @Vision rodar de
outra maquina, o que serve porque o item 1 e defeito **de backend** e o que se prova e o
comportamento do servidor.

> **O que NAO aceito como prova:** teste unitario verde, print de tela sem o corpo da resposta, ou
> alguem dizendo que rodou. **Corpo da resposta salvo em arquivo, ou nao aconteceu.**

---

### BK-02 · Conferencia no banco · 40 min · raia A
**Origem:** C10 do roteiro 1.5.13 · T6 do roteiro 1.5.5 · marco 10.4
**Prova:** nenhuma — **nunca foi feita, por falta de acesso**

**E a unica forma de provar que o que saiu da maquina chegou inteiro do outro lado.** Sem ela, todo
o resto do roteiro prova que o Agent **emitiu**, nao que alguem **recebeu**.

**Pre-requisito:** acesso ao banco de staging. A senha esta pendente de troca (item 4 da linha de
corte) e passou pelo chat duas vezes. **Somente SELECT.**

**Passos:** com os horarios anotados nos cenarios de sessao, consulte a base para o periodo dos
testes.

| O que conferir | Esperado |
|---|---|
| LOGOUT do CS-08 | **uma unica linha**, com o nome do usuario preenchido |
| LOGOUT do CS-09 | **nenhuma** — o `SERVICE_STOP` nao emite LOGOUT (ver CS-09) |
| Sessoes do MU-05 | as duas fecham, nenhuma com evento posterior ao encerramento |
| Eventos do MU-03 | cada um no usuario Windows que de fato o gerou, **nenhum com dono vazio** |
| Reunioes | nenhuma virou duas linhas — e o BK-04 |
| Atraso de fronteira de sessao | **pior atraso abaixo de 5s** para LOCK, UNLOCK, LOGIN e LOGOUT |
| Eventos do RS-02 e do RS-11 | **uma vez so**, sem duplicata — e o RS-13 |

> **O criterio de atraso vem do T6 da 1.5.5, e ja pegou um defeito:** naquela rodada o LOGOUT de
> `SERVICE_STOP` levou 38,7s e virou o A-38, enquanto LOGIN, LOCK e UNLOCK ficaram entre 2,0s e
> 4,7s. **Nao verificado se o A-38 foi corrigido** — meca antes de afirmar.

**Passou se:** cada evento aparece uma vez so, com o dono certo e o horario do **acontecimento**, nao
o da entrega.

**Reprovou se:** LOGOUT duplicado; LOGOUT sem nome; ou linha aberta que nunca fecha.

**Se reprovar:** guarde os identificadores das linhas — sem eles a @Shuri nao consegue reproduzir.

**Se o acesso nao sair:** marque **"nao conferido"** neste cenario e em todos os que dependem dele
(RS-13, AT-21 item 4, BK-04). **Nunca "aprovado".**

---

### BK-03 · O health do backend nao diz a versao · 10 min · raia A · **DEVE REPROVAR**
**Origem:** cenario novo, conferido no codigo em 24/08
**Prova:** observado em 24/08 16:02 — staging responde `version: "unknown"`

**Por que virou cenario e nao ficou como nota:** sem saber qual build esta no ar, **nenhum resultado
de teste contra o backend e conclusivo**. Um 400 no BK-01 pode ser o defeito ou pode ser build
velho, e hoje nao ha como distinguir sem perguntar a uma pessoa.

**O que o codigo faz, conferido por mim** (`health.controller.ts:97-103`): `resolveVersion()` le a
variavel de ambiente `APP_GIT_SHA` e devolve os primeiros caracteres. **Se a variavel nao estiver
setada, devolve `unknown`.** Nao e bug de codigo — e variavel de ambiente ausente no deploy.

**Passos:** `GET /api/health` no staging e leia o campo de versao.

**"Passou" hoje significa reprovar:** `unknown` **e o estado atual**. Registre e anexe ao chamado
do @Vision. **Nao abra achado novo.**

**Passaria se:** o campo trouxesse o SHA do commit no ar. **Custa uma variavel de ambiente no
deploy**, e fecha uma classe inteira de duvida.

---

### BK-04 · Reuniao nao vira duas linhas · 15 min · raia A
**Origem:** C10 do roteiro 1.5.13 (achado A-57)
**Prova:** nenhuma

**Depende do BK-02.** Se o acesso ao banco nao sair, marque "nao conferido".

**Passos:** apos o CS-15, consulte as reunioes do periodo.

**Passou se:** cada reuniao aparece **uma vez**, com inicio e fim.

**Reprovou se:** duplicata — e o A-57 de volta, e infla a agenda do colaborador.

---

# 17. OS AUTOMATIZADOS — o que eles cobrem, e o que eles NAO cobrem

**Isto nao e um bloco de cenarios. E o mapa que impede a confusao mais cara do projeto: achar que
suite verde e a mesma coisa que produto testado.**

Foi exatamente essa confusao que produziu o achado N-01 (secao 3): dois testes verdes chamando o
decider direto, e o consumidor sem tratar a resposta. **A assinatura ja e a nossa marca — o teste e
o codigo concordam entre si, e ninguem testa o consumidor.**

## Cenarios E2E que existem no repo

`manager-srv-agent/tests/e2e/scenarios/`

| Script | O que prova | Cenario manual correspondente | Ja rodou? |
|---|---|---|---|
| `e1-happy-path-wmi.ps1` | Update A->B feliz via WMI | AT-01 | nao verificado |
| `e1-alt-rollback-crash.ps1` | Versao nova crasha 3x, Watchdog reverte | AT-15 | **nunca rodou.** E o item 3b da linha de corte |
| `e2-wmi-failure-fallback.ps1` | WMI sabotado, cai para o Plan B | AT-02 | nao verificado |
| `e3-stuck-ps1-recovery.ps1` | Update travado, flag stale 45min | AT-10 | nao verificado |
| `e4-reboot-mid-update.ps1` | Crash no meio do update | AT-10 | nao verificado |
| `e5-sha256-mismatch.ps1` | Checksum errado nao aplica | AT-03 | nao verificado |
| `e6-corrupted-flag.ps1` | Flag JSON corrompida | AT-11 | nao verificado |
| `e7-current-prod-version.ps1` | Update a partir de versao antiga | AT-13 | nao verificado |
| `e8-update-com-usuario-logado.ps1` | Update com worker vivo (A-39/A-40) | AT-05 | **SIM — 19/08, APROVADO** |
| `n1-versao-igual-recusada.ps1` | A-46: nao reaplica a versao instalada | AT-04 | nao verificado |
| `n2-cooldown-sobrevive-restart.ps1` | A-41: cooldown por relogio de parede | AT-06 | nao verificado |
| `n3-retencao-backups.ps1` | A-42: poda de backups no caminho real | AT-07 | nao verificado |
| `n4-capture-config-efetiva.ps1` | A-50: a secao Capture vale campo a campo | CS-05 | nao verificado |
| `n5-buffer-sobrevive-update.ps1` | Buffer nao e descartado pelo update | AT-08 | nao verificado |
| `n6-instalacao-id-estavel.ps1` | A-49: identificador estavel entre updates | IN-06 | **medido no banco 19/08** |
| `n7-device-jwt-expirado.ps1` | Renovacao do Device JWT ponta a ponta | VC-02 | nao verificado |
| `n8-agente-revogado.ps1` | Revogacao para o envio | VC-03 | nao verificado |
| `n9-sem-vinculo-nao-captura.ps1` | ADR-001: sem vinculo nao captura | VC-04 | tem modo de diagnostico seguro |

> **"Nao verificado" acima significa exatamente isso:** eu **nao conferi** se cada um roda hoje.
> Sei que o `e8` rodou porque tenho o log. Nao vou afirmar o resto.

## O regressivo automatizado local

`manager-srv-agent/tests/e2e/regressivo/run-regressivo-local.ps1`

Dispara acoes reais e confere o que o Agent capturou, lendo os eventos que o worker grava **ja
serializados em JSON no proprio log**. Por isso **nao precisa de banco nem de elevacao**.

| ID | Cobre | Cenario manual |
|---|---|---|
| R1 | Janela capturada, titulo preenchido | CS-02 |
| R2 | Ociosidade detectada | CS-03 |
| R3 | Heartbeat e intervalo efetivo (A-50) | CS-05 |
| R4 | Digitacao nao capturada, titulo sem conteudo, URL so dominio (A-47, A-55) | PR-01, PR-03, PR-04 |
| R5, R5a | Worker morto volta; janela fecha na morte (A-35); ociosidade nao duplica (A-48) | CS-13, CS-18 |
| R7 | Memoria por processo | DS-01 |
| R8 | Buffers presentes, fila drena | DS-02, DS-03 |
| R9 | health-check e diagnostico sem segredo | IF-05, PR-06 |
| R10 | CPU media e tamanho do buffer | DS-01, DS-02 |
| R11 | Rate limit de relancamento | CS-19 |

**O proprio script declara o que NAO testa**, e marca como PENDENTE em vez de sumir num verde
enganoso: desbloqueio de tela, logoff/login e reboot (exigem a senha do usuario); reuniao (exige o
app real em primeiro plano); suspender/acordar (exige toque fisico); sem internet (exige regra de
firewall elevada); e status ATIVO/PAUSA/AUSENTE. **Sao os M1 a M12 dele — e sao exatamente os
cenarios CS-06, CS-07, CS-08, CS-11, CS-15, CS-04, RS-02, IN-01, IF-01, MU-01, RS-14 e RS-11 deste
roteiro.**

## O que continua por automatizar

| # | Cenario | Prioridade | Dono |
|---|---|---|---|
| A8 | `DispararScript` falha (WMI recusa) -> erro, intencao limpa, SOS | media | @Bucky |
| A9 | O script escrito no disco tem o conteudo esperado **e nada mais** — sem caminho absoluto fixo | baixa | @Bucky |
| A10 | `bin.failed-*` com versao nula vira `bin.failed-unknown-*` | baixa | @Bucky |
| E2 | E2E: rollback sem backup (o AT-20 na mao) | alta | @Bucky |
| E3 | E2E: a versao anterior tambem nao sobe | media | @Bucky |
| E4 | E2E: o script de volta nao roda | media | @Bucky |
| E5 | E2E do R-01 (sticky) | alta | @Bucky |
| **novo** | **Teste que atravessa o `SessionMonitor`, nao so o decider** | **alta** | @Bucky |

**O ultimo e o mais importante da lista, e ele nasceu hoje.** Enquanto nao existir um teste que
atravesse o **consumidor**, a familia de defeito do N-01 vai continuar aparecendo — e vai continuar
sendo encontrada por alguem lendo codigo, nunca pela suite.

---

# 18. DE-PARA — onde cada documento e cada teste antigo foi parar

## Os documentos

| Documento | Situacao | Para onde foi |
|---|---|---|
| `registro/2026-08-24-regressivo-final-1.5.16.md` | **APAGADO** | Criterios corrigidos do C4 e do C7 -> CS-09 e MU-04. Achado N-01 -> secao 3 e CS-10. Tabela de provas -> secao 2. Portao -> secao 4. Recorte do dia -> secao 5 |
| `registro/2026-08-24-roteiro-manual-1.5.13.md` | **APAGADO** | C1–C24 redistribuidos por natureza (tabela abaixo). Anexos B e C -> anexos B e C deste documento. De-para do @Bucky -> tabela abaixo |
| `agent-desktop/ROTEIRO-REGRESSIVO-2026-08-18.md` | **APAGADO** | Passos 1–15 redistribuidos. As 13 correcoes ao PLANO foram **aplicadas** nos cenarios, nao copiadas |
| `agent-desktop/ROTEIRO-REGRESSIVO-2026-08-18-SIMPLES.md` | **APAGADO** | A linguagem simples sobreviveu no **anexo D**; as tabelas eram duplicata dos cenarios tecnicos |
| `agent-desktop/ROTEIRO-VALIDACAO-1.5.5.md` | **APAGADO** | T1–T8 -> IF-01, CS-13, CS-06, CS-14, CS-16, CS-17; T6 -> criterio do BK-02. **O placar de 18/08 esta transcrito na secao 2** |
| `agent-desktop/ROTEIRO-VALIDACAO-1.5.7.md` | **APAGADO** | Aceite E8 -> AT-05. **O log da execucao de 19/08 esta transcrito no anexo A.** Correcoes da harness -> anexo C |
| `features/agent/testes-regressivo.md` | **APAGADO** | Marco de 27/03, era da V1. O que sobreviveu esta em CS-02, CS-03, CS-06, CS-12, bloco AT e bloco RS. O resto esta na secao 19 |
| `agent-desktop/PLANO-TESTES-REGRESSIVOS.md` | **MANTIDO como historico** | **Nao apaguei de proposito.** Seus ~500 identificadores `Rx.y.z` sao citados por numero em codigo de teste e em documentos historicos, e o bloco de migracao V1 (secao 19, corte 14) so existe la. **Leia o aviso que coloquei no topo dele.** Nao e mais o roteiro vigente |

## Os cenarios do roteiro de 24 (C1–C24)

| Antigo | Novo | Observacao |
|---|---|---|
| C1 | **CS-01** | ganhou o momento de fechamento (era o R6) |
| C2 | **CS-07** | criterio da `SESSAO_INTERROMPIDA` reescrito |
| C3 | **CS-08** | |
| C4 | **CS-09** | **criterios trocados** — os antigos reprovavam produto correto |
| C5 | **MU-05** | |
| C6 | **MU-03** | |
| C7 | **MU-04** | **criterio trocado** — "os dois workers voltam" induzia ao erro |
| C8 | **IN-04** | fundido com PLANO 14.1 e 14.2 |
| C9 | **BK-01** | fundido com o R1 do regressivo 1.5.16 |
| C10 | **BK-02** | ganhou o criterio de atraso do T6 da 1.5.5 |
| C11 | **RC-01** | os quatro limites foram para a secao 3 e para o RC-08 |
| C12 | **AT-15** | |
| C13 | **AT-17** | |
| C14 | **AT-18** | |
| C15 | **RC-04** | |
| C16 | **RC-05** | |
| C17 | **RC-06** | |
| C18 | **RC-03** | |
| C19 | **RC-07** | |
| C20 | **RC-02** | |
| C21 | **AT-16** | |
| C22 | **AT-19** | |
| C23 | **AT-20** | raia C (VM) |
| C24 | **AT-21** | raia C (VM) |

## Os testes do @Bucky (documento ja apagado em 24/08)

`agent-desktop/TESTES-ROLLBACK-E-RECALL.md` foi absorvido no roteiro de 24 e apagado. Esta cadeia
existe para quem procurar por `Teste 10.11` ou `R10.11.7`:

| @Bucky | Roteiro de 24 | **Aqui** |
|---|---|---|
| Teste 10.5 | C12 | **AT-15** |
| Teste 10.6 | C21 | **AT-16** |
| Teste 10.7 | C13 | **AT-17** |
| Teste 10.8 | C23 | **AT-20** |
| Teste 10.9 | C22 | **AT-19** |
| Teste 10.10 | C24 | **AT-21** |
| 10.11 cabecalho | Bloco 3 | **secao 15, "Como a ordem viaja"** |
| R10.11.1, .3, .4 | C11 | **RC-01** |
| R10.11.2, .6, .16 | C20 | **RC-02** |
| R10.11.5, .13, .14, .15 | C18 | **RC-03** |
| R10.11.7 | C15 | **RC-04** |
| R10.11.8 | C16 | **RC-05** |
| R10.11.9, .10 | C17 | **RC-06** |
| R10.11.11 | C19 | **RC-07** |
| R10.11.12 | C12 criterio 7 | **AT-15 criterio 7** |

## Ponteiros que ficaram para tras

Estes arquivos citam documentos que este roteiro apagou. **Nenhum foi alterado**: registro
historico nao se reescreve, e comentario dentro de codigo de teste nao e desta tarefa. Ficam
listados para a varredura de textos defasados.

| Arquivo | O que cita |
|---|---|
| `docs/INDEX.md` | **atualizado por mim** — aponta para este documento |
| `agent-desktop/ACHADOS-REGRESSIVO-2026-08-18.md` | as correcoes C1–C13 do roteiro de 18/08 |
| `agent-desktop/BRIEFING-1.5.10-HANDOFF.md` | o roteiro de validacao da 1.5.7 |
| `agent-desktop/PLANO-CORRECOES-AGENT-WINDOWS-1.5.4.md` | os dois roteiros de 18/08 |
| `registro/2026-08-24-recall-porta-de-mao-unica.md` | o roteiro manual da 1.5.13 |
| `registro/2026-08-24-retomada-pos-reboot.md` | o roteiro manual da 1.5.13 |
| `manager-srv-agent/HANDOFF-R03-RECALL-AGENT.md` | o documento do @Bucky, ja apagado antes |
| `manager-srv-admin-node/.../agent-update.service.test.ts:503` | idem. **@Shuri, na proxima vez que tocar o arquivo** |
| `manager-srv-admin-node/.../fleet-health.repository.test.ts:561` | idem |
| `manager-srv-admin-node/.../revogar-versao.body.dto.test.ts:3` | cita "secao 2.3", que hoje e a secao 17 deste documento |

---

# 19. O QUE FOI CORTADO, E POR QUE

**Esta tabela nao e o arquivo do cenario. E o registro da minha decisao.** Sem ela, daqui a dois
meses ninguem sabe se um teste sumiu por decisao ou por descuido — e a duvida traz tudo de volta.

**Nao cortei nada do bloco de privacidade, e nao cortei nenhum cenario que fosse o unico a
exercitar um caminho.** Onde a cobertura era unica, esta escrito no proprio cenario que ela e
unica.

| # | O que saiu | De onde | Por que saiu |
|---|---|---|---|
| 1 | Startup e `agent-tray.log` | marco 1.1–1.3 | O processo e o arquivo de log sao da V1. Nao existem no produto |
| 2 | Coleta de janela ativa | marco 2.1–2.4 | Duplicata de PLANO 3.1. A melhor redacao virou CS-02 |
| 3 | Idle com limiar de 60s | marco 3.1–3.3 | Limiar obsoleto (hoje 5 min). Substituido por CS-03 e CS-04 |
| 4 | LOCK/UNLOCK | marco 4.1–4.3 | Duplicata. A versao com criterio de atraso (T4) virou CS-06 |
| 5 | `Heartbeat AthenaAgent` no log | marco 5.1–5.3 | String da V1. Substituido por CS-05, que confere o intervalo efetivo |
| 6 | `events.db` com coluna `Uploaded` | marco 6.1–6.4 | Esquema obsoleto: hoje e um buffer por sessao. Substituido por DS-02 e RS-11 |
| 7 | Status `Online` / `Ausente` / `Inativo`, 30 min | marco 7.1–7.4 | **Produto nao tem mais esse comportamento.** Hoje e ATIVO/PAUSA/AUSENTE com 5/15 min. Substituido por CS-04 |
| 8 | Corte diario | marco 8.1–8.2 | Duplicata. Virou CS-12, reescrito como leitura no dia seguinte em vez de vigilia |
| 9 | Auto-update (7 itens) | marco 9.1–9.7 | Duplicata do bloco AT, com redacao anterior ao rollback e ao recall existirem |
| 10 | Resiliencia (4 itens) | marco 10.1–10.4 | Duplicata do bloco RS |
| 11 | Checklist pos-teste | marco | Duplicata de CS-01 e DS-01 |
| 12 | `monitorar-performance.ps1` | PLANO 4.5 e 5.x | **O script nao esta no pacote.** O roteiro de 18/08 ja o marcava removido. Substituido pelo laco manual do DS-01 |
| 13 | Tempo de inicializacao | PLANO 5.3 | Virou criterio dentro de IN-01 e RS-14. Cenario proprio nao pagava o tempo |
| 14 | Migracao V1 -> V2 (3 testes) | PLANO 11.1–11.3 | **Nao ha maquina V1 conhecida no escopo do canario**, e o caminho de migracao que importa hoje e o de config anterior a v1.3.0 (AT-14). **Nao verifiquei se resta alguma V1 na frota** — por isso o PLANO foi **mantido como historico**: se aparecer uma, o cenario esta la, intacto |
| 15 | "Nenhum item AthenaAgent no menu" | PLANO R2.2.7 | Legado V1. Nada no produto pode produzir esse item |
| 16 | SCM em `1s/5s/30s` | PLANO R1.1.13 | **Valor errado.** O instalador configura 5/10/30. Corrigido dentro do IN-01, nao removido |
| 17 | "5 scripts em `scripts\`" | PLANO R1.1.6 | Sao seis desde que `test-vinculacao` e `desvincular` entraram. Corrigido no IN-01 |
| 18 | "Plan A = PS1 via schtasks" | PLANO 10.1 | **Nome errado.** Plan A e WMI desde a v1.3.10. Corrigido no AT-01 |
| 19 | Consultas com `sqlite3` em `buffer.db` | PLANO 3.5 e outros | Banco unico e obsoleto, e o `sqlite3` nao esta garantido na maquina. Substituido pela leitura do JSON no proprio log, que e como o regressivo automatizado ja faz |
| 20 | Remocao automatica de eventos com mais de 7 dias | PLANO R5.4.2 | **Exige 7 dias de calendario.** Nao e executavel numa rodada. Vai para a lista de automacao |
| 21 | Taxa minima de sucesso por categoria (>= 90%) | PLANO "Metricas" | **Criterio percentual nao diz nada.** 95% de privacidade e reprovacao. Substituido pela linha de corte e pelo portao da secao 4 |
| 22 | Lista de "bugs nao-criticos" | PLANO "Metricas" | Lista sem criterio, e o criterio ja existe (linha de corte, com o @Tony) |
| 23 | Smoke test de 15 min | PLANO + roteiro simples | Substituido pela **raia minima** da secao 5, que fecha o canario em vez de so dar confianca |
| 24 | Referencia rapida dos scripts PowerShell | PLANO, secao final | Documentacao de ferramenta, nao cenario. Vive no `MANUAL-COMPLETO.md` |
| 25 | Limite do buffer autonomo, cenario proprio | PLANO 12.3 | Fundido no RS-05 — e a mesma medicao |
| 26 | Named Pipe normal e reconexao, separados | PLANO 9.1 e 9.2 | Fundidos no RS-11: um sem o outro engana |
| 27 | Parada graciosa e reinicio manual, separados | PLANO 7.2 e 7.4 | Fundidos no RS-14, mesma preparacao |
| 28 | Estrutura do menu e submenu, separados | PLANO 2.2 e 2.3 | Fundidos no IF-02, mesmo clique |
| 29 | Verificacao de saude pelo menu | PLANO 2.4 | Duplicata do IF-05. O caminho pelo menu virou criterio do IF-02 |
| 30 | `limpar-reset.ps1` pela linha de comando | PLANO 4.4 | Duplicata do IF-03 — mesmo dialogo, mesma confirmacao |
| 31 | Bufferizacao local e flush, separados | PLANO 12.1 e 12.2 | Fundidos no RS-11 |
| 32 | Multiplos crashes do worker, cenario proprio | PLANO 13.2 | Fundido no CS-18, que ja mata tres vezes |
| 33 | PASSO 0 — build e rebuild do pacote | roteiro 08-18 | Preparacao, nao cenario. Virou a preparacao da raia B (secao 14) |
| 34 | Tabela "correcoes pendentes no plano" (C1–C13) | roteiro 08-18 | Era uma lista de tarefas sobre um documento. **As 13 foram aplicadas** nos cenarios deste roteiro |
| 35 | As dez tabelas em linguagem simples | roteiro SIMPLES | Duplicata dos cenarios tecnicos. **A linguagem sobreviveu no anexo D** |
| 36 | "Como reportar um problema" | roteiro SIMPLES | Virou a regra de evidencia da secao 2, que e mais exigente |
| 37 | Numeros de cobertura (1139 / 1205 testes) | roteiro 1.5.13, anexo A | **Numeros vencidos no dia seguinte.** Nao sao cenario, e mantê-los so gera correcao de documento |
| 38 | Passo 0 da 1.5.5 (`session-state-1.json` ausente) | roteiro 1.5.5 | Preparacao especifica daquela rodada, para observar o T2. Sem valor fora dela |
| 39 | T6 — fronteira de sessao sobe no ato | roteiro 1.5.5 | Era uma consulta agregada sobre eventos que outros cenarios ja produzem. **Virou criterio dentro do BK-02**, com o limiar de 5s preservado |
| 40 | Passos 1–3 da 1.5.7 (build 1.5.7 e 1.5.99) | roteiro 1.5.7 | Preparacao de versao especifica. Generalizada na preparacao da raia B |
| 41 | "O que foi corrigido na harness" (6 itens) | roteiro 1.5.7 | Historia de como a harness passou a rodar. **Preservada no anexo C** por conter armadilhas que voltam |
| 42 | O recorte "o que rodar hoje" (R0–R6) | regressivo 1.5.16 | Datado: era o recorte de uma tarde. Substituido pela raia minima (secao 5) e pelo portao (secao 4) |
| 43 | "1.5.16 = 1.5.15 com outro numero" como secao | regressivo 1.5.16 | Virou a regra geral de quando uma prova antiga deixa de valer (secao 2), que serve para toda versao futura |
| 44 | Consulta ao painel "sem filtro de nivel" | roteiro 1.5.13, C18 | **Instrucao vencida em 24/08.** Absorvida como estava, mandaria desligar o filtro que agora prova a correcao. Ver a caixa no RC-03 |
| 45 | "A sticky impede o segundo rollback" | roteiro 1.5.13, C13 | **Afirmacao que o desenho nao sustenta.** O bloqueio impede **instalar** a versao ruim, nao impede **voltar** dela. Vira pergunta para o @Bucky, nao criterio. Ver a caixa no AT-17 |

**O corte de que mais me orgulho e o 21** — a taxa minima de sucesso por categoria. Ela parecia
rigor e era o contrario: dava para reprovar tres itens de privacidade e ainda "passar com 95%".
Um criterio que permite arredondar vazamento nao e criterio, e conforto. Trocado por uma linha:
**uma ocorrencia de privacidade para o lancamento.**

---

# 20. RELATORIO FINAL — uma linha por cenario

**Raia A (74) — maquina de trabalho**

| # | Cenario | Resultado | Evidencia | Obs |
|---|---|---|---|---|
| PR-01..07 | Privacidade e LGPD (7) | | | **uma ocorrencia para o dia** |
| CS-01 | Foto do estado (partida) | | | |
| CS-02..06 | Janela, ociosidade, status, heartbeat, lockscreen (5) | | | |
| CS-07 | Reboot | | | prova 24/08 |
| CS-08 | Logoff e login | | | |
| CS-09 | Parar o Service logado | | | criterios reescritos |
| CS-10 | Queda de energia | | | **deve reprovar** |
| CS-11..15 | Sleep, virada do dia, kill do worker, parada graciosa, reuniao (5) | | | |
| CS-16, CS-17 | 1ms na retomada; shell sem titulo | | | **devem reprovar** (A-30, A-31) |
| CS-18, CS-19 | Worker volta; rate limit | | | |
| IF-01..06 | Interface e diagnostico (6) | | | |
| DS-01..03 | Peso na maquina (3) | | | limite 150MB |
| VC-01..05 | Vinculo e credenciais (5) | | | |
| RS-01..14 | Resiliencia (14) | | | |
| MU-01..05 | Multiusuario (5) | | | MU-03 e MU-04 param o dia se reprovarem |
| AT-12, AT-16, AT-18, AT-22 | ASCII, SCM, versao quebrada, telemetria | | | so leitura |
| IN-06 | Identificador estavel | | | so leitura |
| RC-01, RC-02, RC-03, RC-06 | Recall: volta, alcance, painel, guarda C | | | RC-02 exige desfazer o SOS |
| RC-08, RC-09 | Atraso; escopo do pausar | | | **devem reprovar** |
| BK-01 | **Lote com item ruim** | | | **OBRIGATORIO** |
| BK-02..04 | Banco, health, reuniao (3) | | | dependem do acesso ao banco |
| CS-01 | Foto do estado (fechamento) | | | |

**Raia B (24) — maquina dedicada**

| # | Cenario | Resultado | Evidencia | Obs |
|---|---|---|---|---|
| IN-01..05 | Instalacao, Configurator, upgrade, do zero, desinstalacao (5) | | | |
| AT-01..11 | Update fim a fim e caminhos de recuperacao (11) | | | |
| AT-13, AT-14 | Versao antiga; migracao de config | | | AT-14 sem maquina candidata = "nao conferido" |
| AT-15, AT-17, AT-19 | Volta automatica; sticky; dois usuarios | | | AT-15 e AT-17 sao portao |
| RC-04, RC-05, RC-07 | Guardas A e B; recall que falha | | | devolver o backup ao terminar |

**Raia C (2) — VM descartavel**

| # | Cenario | Resultado | Evidencia | Obs |
|---|---|---|---|---|
| AT-20 | Volta sem backup: SOS de proposito | | | termina com a maquina parada |
| AT-21 | Desligada no meio da volta | | | item 4 exige banco |

**O que bloqueia producao, e so isto:**

- **AT-15 reprovado** — a frota nao tem caminho de volta. E a regra 5.2, e o item 3a da linha de corte.
- **AT-17 reprovado** — a maquina reinstala a versao quebrada em laco.
- **BK-01 reprovado** — perda de dado silenciosa. E o item 1 da linha de corte.
- **AT-16 reprovado** — a maquina fica **sem auto-recuperacao** depois de uma volta. Passa o criterio
  pelo mesmo motivo que o AT-15: o produto ficaria **pior depois de se proteger** do que estava
  antes, e o estrago so aparece na proxima queda, quando ninguem estiver olhando.
- **Qualquer ocorrencia do bloco PR** — vazamento nao passa por discussao de lista.

Os demais entram como achado da rodada e **passam pelo criterio da linha de corte com o @Tony antes
de virar bloqueador**. Nao inclua achado novo na lista sem passar pelo criterio — foi exatamente
para isso que a linha foi tracada.

---

# 21. POSICAO DA @NATASHA

**Este roteiro tem 100 cenarios, e eu cortei 45.** O que sobrou nao e o que da para testar: e o que
**precisa** ser testado para alguem olhar o Agent e dizer que funciona. Se o Marcos executar isto
inteiro e tudo passar, ele libera o canario sem sensacao de ter pulado alguma coisa. **Nem mais, nem
menos que isso** era o pedido, e e o que eu entreguei.

**O produto esta em condicao melhor do que qualquer versao que cogitamos liberar hoje de manha.** O
mecanismo que protege a frota saiu do papel: rodou duas vezes em maquina de verdade e voltou
capturando nas duas. A telemetria que estava muda voltou a falar. E o AT-16, que eu mesma tinha
posto como bloqueador, esta verde depois de duas restauracoes reais — eu medi.

**Falta um item, e ele e o mais barato de todos.** O BK-01 e o unico da linha de corte que nunca foi
exercitado contra o servidor, e o unico que perde dado do colaborador em silencio. O backend esta no
ar, o codigo esta commitado, o deploy bate com o horario. **Vinte e cinco minutos.**

**Tres coisas que eu peco que ninguem ignore por pressa.**

A primeira sao os **criterios do CS-09 e do MU-04**. Os que estavam escritos eram meus, e estavam
errados — conferi no codigo e no log da propria maquina. Executados como estavam, **reprovam um
produto correto** e mandam o @Bucky cacar defeito que nao existe.

A segunda e o **CS-10**. A correcao do A-62 nunca rodou em producao, porque falta um `case` no
`switch` que consome a decisao. Nao bloqueia o canario — o comportamento e identico ao da 1.5.12,
que ja esta em producao, e o efeito e contagem de tempo a mais, nao perda do dado ja capturado.
**Mas e o oitavo defeito do dia com a mesma assinatura**, e a assinatura ja e a nossa marca: *o
teste e o codigo concordam entre si, e ninguem testa o consumidor.* Enquanto nao existir um teste
que atravesse o consumidor, essa familia vai continuar aparecendo — e vai continuar sendo encontrada
por alguem lendo codigo, nunca pela suite. **Por isso ele entrou como o item de prioridade alta da
secao 17.**

A terceira e o **AT-14**, a migracao de config antiga. Nunca rodou em maquina nenhuma, nao roda na
de referencia, e a primeira maquina do canario e justamente uma candidata a exercita-la. Nao seguro
o canario por isso, mas **quero a escolha da empresa feita com esse dado na mesa, e nao por acaso.**

**Com o BK-01 verde e a raia A fechada, eu libero o canario** — 1 a 3 empresas, 48h, com o botao de
pausa a mao e alguem olhando a telemetria no primeiro dia. **Sem o BK-01, nao.**

**Sobre a raia C, uma posicao explicita que nao mudou:** os dois cenarios de la sao bons e precisam
existir, mas **nao na maquina de trabalho**. Os dois terminam com a maquina parada por desenho, e o
unico caminho de volta e reinstalar. No documento em que os encontrei estavam soltos no meio da
lista, sem aviso e sem saida — quem executasse em ordem descobriria isso no meio do cenario. **Peco
VM para os dois.**

---

# ANEXO A — evidencia transcrita: aceite do E8 (19/08, APROVADO)

Preservada aqui porque o documento que a continha foi apagado, e **estas linhas sao a prova do
AT-05**. Executado em 2026-08-19 09:52, `DESKTOP-VMSM6LE`, Marcos logado.

Em `service-20260819.log`:

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

**Os seis criterios, todos OK:** versao 1.5.7 -> 1.5.99 em ~22s; **zero** `CONNECT received` ou
`Worker launched` na janela critica (na 1.5.5 eram dois, 1s e 6s apos o GOODBYE); `gate closed`
presente; nenhum erro de lock nem rollback; Service `Running` no fim; worker de volta em 4s, PID
6472, ja na versao nova.

**Extras observados:** aplicou **uma unica vez** — o ciclo seguinte respondeu `no update available`,
com o stub ciente de versao e o guard A-46 funcionando. E **`killing survivor worker` nao aparece**:
na 1.5.5 era o sintoma de que o watchdog tinha relancado alguem; nao ha sobrevivente para matar.

---

# ANEXO B — o que NAO da para testar, e por que

| O que | Por que | Como conviver |
|---|---|---|
| Rollback em maquina real dentro do CI hospedado | exige servico Windows, elevacao e reinicio de servico | decidir com o @Vision: VM propria ou runner auto-hospedado. **Se nao couber, dizer isso — nao adaptar o teste ate caber** |
| Queda de energia exata no meio da escrita do estado | nao se encena de forma repetivel | coberto por unitario (temporario orfao) mais o AT-21 na mao |
| Recall alcancando a frota inteira | exige varias maquinas | canario primeiro, e acompanhar pela versao reportada de cada agente |
| Volta de mais de uma versao | a copia de seguranca guarda **uma** so | limitacao de desenho, nao lacuna de teste. Tem de estar visivel para quem dispara o recall |
| O ramo de migracao de config antiga, na maquina de referencia | ela ja esta migrada, e o ramo nunca vai disparar la | AT-14, em maquina candidata; ou a condicao de escolha do canario (secao 4, item 6) |

---

# ANEXO C — armadilhas da harness, e elas voltam

Preservadas do roteiro da 1.5.7. A harness de E2E estava escrita e **nunca tinha rodado**. Estes
seis problemas foram encontrados e corrigidos em 18/08 — e cada um deles e o tipo de coisa que
volta quando alguem escreve um cenario novo sem olhar para tras.

| Problema | Efeito | Correcao |
|---|---|---|
| O stub nao servia o ZIP | a rota caia no 404 do `else`. O download **nunca aconteceria** | rota de downloads mais parametro de caminho, validada por SHA-256 |
| `Stop-Job -Force` | o parametro **nao existe** no PowerShell 5.1: o `finally` de **todos** os cenarios lancava excecao e o listener ficava vivo | trocado por `Remove-Job -Force`. `Stop-Job` sozinho trava, porque a thread do job fica bloqueada |
| Nao-ASCII em `.ps1` | **3 dos 7 cenarios nao parseavam.** A-05 pela terceira vez: em comentario passa, dentro de string quebra | os 34 `.ps1` do repo limpos. O teste `PowerShellScriptsAreAsciiTests` trava a regressao no CI |
| `dist/` vazio | os scripts procuravam la, mas o build escrevia em outra pasta e terminava em `Read-Host` — nao roda em cenario | `build-pacote-e2e.ps1`, nao-interativo, com bump e restauracao de versao |
| `build-v-teste.ps1` | chamava um script que **nao existe** no repo, e nao parseava | aviso no cabecalho apontando o script que funciona |
| `MANAGER_UPDATE_CHECK_SECONDS` | usado por 3 cenarios como se acelerasse a checagem. **Nao e lido por codigo nenhum** | o E8 nao usa: dispara pelo check de startup, que e real |

**A ultima linha e a lição:** tres cenarios confiaram numa variavel de ambiente que ninguem le. E o
mesmo padrao do `AutoUpdate.Habilitado` (secao 1) e do `MaximoBackups` no caminho errado (AT-07).
**Antes de confiar num interruptor, ache quem o le.**

---

# ANEXO D — a leitura simples, para quem vai executar sem ler codigo

O mesmo roteiro, sem jargao. **Nao substitui os cenarios** — serve para saber, em uma linha, o que
cada bloco esta tentando descobrir.

| Bloco | A pergunta que ele responde |
|---|---|
| **Privacidade** | O agent grava alguma coisa que ele nao deveria ver? |
| **Captura e sessao** | O tempo de trabalho da pessoa sai certo — sem inventar hora e sem perder hora? |
| **Multiusuario** | Duas pessoas no mesmo computador tem os dados separados e nenhum se mistura? |
| **Interface** | O que a pessoa ve na tela funciona, e o suporte consegue diagnosticar de longe? |
| **Peso na maquina** | O agent atrapalha o trabalho de alguem? |
| **Vinculo** | O agent so coleta de quem foi autorizado, e para quando manda parar? |
| **Resiliencia** | Sem internet, sem servidor, com arquivo estragado ou processo morto — perde dado? |
| **Instalacao** | Instala, atualiza e sai limpo, sem deixar sujeira nem dispositivo fantasma? |
| **Atualizacao e volta** | Se a versao nova nao subir, o computador volta sozinho e continua trabalhando? |
| **Recall** | Se descobrirmos um problema depois de a versao ja estar instalada, conseguimos chamar a frota de volta? |
| **Backend** | O que saiu do computador chegou inteiro do outro lado? |

**Impede o lancamento:** qualquer vazamento de privacidade; o computador nao voltar sozinho depois
de uma atualizacao ruim; perder dado do colaborador; ou reinstalar em laco a versao que quebrou.
