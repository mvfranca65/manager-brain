# Roteiro de teste manual na maquina — rodada 1.5.13 (2026-08-24)

> **STATUS:** PRONTO PARA EXECUTAR — parte hoje, parte travada por pre-requisito
> **DONO:** @Natasha (QA) | **EXECUTOR:** Marcos | **MAQUINA:** `DESKTOP-VMSM6LE`
> **APOIO:** @Bucky (Agent), @Shuri (backend), @Vision (numero de versao e publicacao)
>
> **Decisao do Marcos, hoje:** testar tudo a mao nesta maquina, com dado de verdade.
>
> **Decisao do @Tony, hoje (tarde):** existia um segundo documento de teste cobrindo o mesmo
> terreno — `agent-desktop/TESTES-ROLLBACK-E-RECALL.md`, do @Bucky. **Dois roteiros e como testar
> a lista errada.** Ele foi **incorporado neste documento e apagado**. Nao procure o arquivo: tudo
> que ele tinha esta aqui, e o de-para numero a numero esta na secao
> [**De-para com o documento do @Bucky**](#de-para-com-o-documento-do-bucky). Roteiro de teste de
> maquina agora e um so, e o dono e a @Natasha.
>
> **Base:** `registro/2026-08-21-linha-de-corte-producao.md` (com o estado de 24/08),
> `registro/2026-08-24-rollback-nao-esta-ligado.md`,
> `registro/2026-08-24-entrega-item1-e-rollback.md`, `agent-desktop/REGRAS-RELEASE.md` secao 5.2,
> `registro/2026-08-21-bateria-1.5.12.md`, `registro/2026-08-20-achados-manuais-1.5.10.md`,
> `manager-srv-agent/HANDOFF-R01-STICKY.md`, `HANDOFF-R02-VERSAO-QUEBRADA.md`,
> `HANDOFF-R03-RECALL-AGENT.md`, `DESENHO-R03-RECALL-AGENT.md`,
> `manager-srv-admin-node/HANDOFF-R03-RECALL.md`.

---

## LEIA ISTO ANTES DE QUALQUER COISA

**Esta e a sua maquina de trabalho, e alguns testes deste roteiro podem deixar ela sem captura.**

O teste da volta automatica de versao (Bloco 4) funciona assim: instala de proposito uma versao
que nao sobe, e espera a maquina voltar sozinha para a anterior. **Se a volta falhar, o Agent
entra em modo de emergencia (SOS) e para de capturar ate alguem reinstalar a mao.** O modo SOS
nao sai sozinho — nao existe no produto nada que o desligue automaticamente.

O **Bloco 5** e pior ainda: os dois cenarios de la deixam a maquina sem captura **de proposito** —
e o resultado esperado. Eles vieram do documento do @Bucky, que os escrevia sem aviso e sem
caminho de volta. **Nao rode o Bloco 5 nesta maquina sem autorizacao escrita do Marcos**, e
prefira uma VM descartavel. Detalhe no proprio bloco.

Por isso os Blocos 4 e 5 sao os **ultimos** de tudo, e por isso tem duas redes de seguranca
preparadas antes de comecar:

| Rede | Arquivo | Observacao |
|---|---|---|
| Primeira | `manager-srv-agent\instalador\ManagerAgent-Setup-v1.5.12.exe` | de 21/08 — **e anterior as correcoes de hoje** |
| Segunda | `manager-srv-agent\instalador\ManagerAgent-Setup-v1.5.11.exe` | de 21/08 |

Nao comece os Blocos 4 ou 5 num dia em que voce precise da maquina em seguida. Faca em fim de
expediente, com tempo de sobra para reinstalar.

**Aviso do @Tony, vale para o roteiro inteiro:** rodar a suite de testes automatizados nesta
maquina chegava a **parar os servicos reais dela por ~17s por vez**. O freio ja foi aplicado, mas
a regra e simples: **rodar suite e fazer teste manual nao se misturam.** Enquanto voce estiver
executando qualquer cenario deste roteiro, ninguem roda `dotnet test` nesta maquina — nem voce,
nem sessao de agente nenhuma. Se rodar, o resultado do cenario nao vale.

**Regra herdada do documento do @Bucky, e vale para tudo aqui:** teste que monta a propria arvore
de diretorios nao prova nada sobre a instalacao real. **Todo cenario deste roteiro roda contra o
que o instalador produziu** — nunca contra pasta montada a mao. Foi exatamente essa distancia que
escondeu o defeito original do rollback: os testes unitarios passavam porque concordavam com o
codigo, e os dois discordavam do instalador.

---

## O que mudou hoje, depois que a primeira versao deste roteiro foi entregue

Tres coisas mudaram, e as tres mexem no que da para executar.

**1. O codigo saiu da maquina. Foi commitado e empurrado para `staging` nos dois repos:**

| Repo | Commit em `staging` | Situacao |
|---|---|---|
| `manager-srv-agent` | `39d20e3` — *Agent obedece o recall de frota (R-03)* | empurrado, sincronizado com o `origin` |
| `manager-srv-admin-node` | `e86124d` — *eventos de recall sobem para WARN* | empurrado, sincronizado com o `origin` |
| `manager-srv-events-node` | **nenhum** | **NAO commitado.** Ver o P1 abaixo |

O pre-requisito antigo — *"o codigo esta so na maquina, sem publicacao"* — **caiu para o Agent e
para o admin**. Passou a ser outra coisa: **falta o deploy do `staging`**. Publicar codigo e
subir codigo publicado nao sao a mesma coisa, e nenhum cenario do Bloco 3 mede o que promete
enquanto o ambiente de staging estiver rodando a versao anterior.

**2. O recall ficou pronto dos dois lados.**

| Lado | Testes antes | Testes depois |
|---|---|---|
| Agent (`manager-srv-agent`) | 1092 | **1139** |
| Backend (`manager-srv-admin-node`) | 1199 | **1205** |

**3. Os avisos de recall subiram de nivel — o recall agora aparece no painel.** Os tres eventos
(`UPDATE_RECALL_TRIGGERED`, `_FAILED`, `_SKIPPED`) entravam no painel como evento rotineiro
(`INFO`) porque nao estavam no mapa de tipos conhecidos. O commit `e86124d` pos os tres como
**aviso (`WARN`)**. **Isso invalida um aviso que estava escrito nos dois roteiros** e que mandava
consultar o painel sem filtro de nivel — ver a correcao no C18.

**O que NAO mudou, e continua travando:**

- **A versao nao foi bumpada.** Os 7 projetos do Agent seguem em `1.5.12` — conferido arquivo por
  arquivo, e o mesmo numero que ja esta instalado nesta maquina.
- **Os instaladores da 1.5.13 nao existem.** A pasta `manager-srv-agent\instalador` vai ate
  `ManagerAgent-Setup-v1.5.12.exe`. Nem o pacote bom, nem o propositalmente quebrado.

---

## Estado da maquina, medido hoje (24/08, 11:58)

Confira antes de comecar; se algo aqui mudou, o roteiro precisa ser relido.

| O que | Valor medido |
|---|---|
| Servicos | `ManagerAgent` e `ManagerAgentWatchdog` — os dois `Running`, inicio automatico |
| Versao instalada | `1.5.12+b1d227ac93fcdbec72ece38755e9bca2f034f8a9` |
| Copia de seguranca da versao anterior | `C:\Program Files\bin.previous` **existe**, com a `1.5.11` dentro |
| Pasta de versao quebrada | nenhuma `bin.failed-*` em `C:\Program Files` |
| Modo SOS | **desligado** (`sosMode: false`) |
| Registro do Watchdog | `{ startupFailuresSinceLastUpdate: 0, sosMode: false, sosRecoveryHeaderCount: 0 }` |
| Auto-recuperacao do Windows (SCM) | `restart 5s / 10s / 30s`, reset 86400, nos **dois** servicos |
| Sobra da suite de hoje | `logs\rollback-script.log` escrito as 11:09 — residuo da rodada do @Bucky |

**Tres leituras importantes desse quadro:**

1. **O registro do Watchdog nao tem o campo que diz qual versao esta rodando.** E exatamente o
   que o R-02 previa. Numa maquina assim, hoje, a protecao contra reinstalar a versao quebrada
   nao teria como disparar. E o R-02 que fecha isso, e o Bloco 4 e quem prova.
2. **A maquina roda a 1.5.12 de 21/08 — que NAO tem nenhuma das correcoes de hoje.** O codigo de
   hoje esta empurrado para `staging`, mas empurrar codigo nao instala nada em maquina nenhuma. O
   numero de versao tambem nao mudou. Testar rollback ou recall hoje, sem instalar nada novo,
   testaria o mecanismo velho e quebrado.
3. **A auto-recuperacao do SCM esta intacta agora — e essa e a foto que o C21 vai comparar
   depois.** Sem esta medicao de hoje, o C21 nao teria com o que comparar. Ela nao existia em
   nenhum dos dois roteiros e entrou aqui de proposito.

---

## O que trava o que — pre-requisitos que nao dependem de voce

Nao ha contorno para nenhum destes. Estao escritos como pre-requisito, e nao como problema.

| # | Pre-requisito | Quem resolve | Trava qual cenario |
|---|---|---|---|
| P1 | **`manager-srv-events-node` nao foi commitado.** A correcao do item 1 (lote com item ruim) esta **so na arvore de trabalho** — ~1000 linhas em `ingestion.service.ts` e nos DTOs, sem commit e sem push. Os outros dois repos ja estao em `staging` | @Shuri commita, Marcos autoriza, @Vision publica | C9 |
| P2 | **Deploy do `staging`.** Os commits `39d20e3` (Agent) e `e86124d` (admin) estao empurrados, mas o ambiente de staging precisa **subir** essa versao. Codigo empurrado nao e codigo no ar | @Vision | C9, C10, C11, C15–C20 |
| P3 | **A versao nao foi bumpada.** Os 7 projetos continuam em `1.5.12`, o mesmo numero que ja esta instalado. Um pacote gerado da arvore de hoje nasceria com o mesmo numero da versao instalada, e o fluxo de update nao teria o que oferecer | bump coordenado com o @Vision | C11, C15–C20, Blocos 4 e 5 |
| P4 | **Um instalador bom da versao nova (N)**, gerado do codigo de hoje, instalado nesta maquina | @Bucky gera, voce instala | C11, C15–C20, Blocos 4 e 5 |
| P5 | **Um pacote propositalmente quebrado (N+1)** — em que o servico principal nao sobe. *Como sabotar, se alguem precisar: e o que o `e1-alt-rollback-crash.ps1` faz — trocar o `ManagerAgent.Service.exe` por um binario que sai com codigo 1 no startup. Use so a receita; **nao rode aquele script** (ver armadilha 1)* | @Bucky gera | Blocos 4 e 5 |
| P6 | **Acesso ao banco de staging.** A senha esta pendente de troca (item 4 da linha de corte) e passou pelo chat duas vezes | @Vision / Marcos | C10, C24, e a conferencia final de C9 |
| P7 | **Segundo usuario Windows local** com senha em maos (a conta `Marcos`, que ja foi usada nas rodadas anteriores) | voce | C6, C7, C22 |
| P8 | **Chave de ativacao** da empresa de teste, mais o identificador do colaborador | voce / Marcos | C8 |
| P9 | **Autorizacao escrita do Marcos**, ou uma VM descartavel, para os dois cenarios que derrubam a captura de proposito | Marcos | Bloco 5 (C23, C24) |

**O que caiu:** o pre-requisito antigo *"nada foi empurrado"* nao existe mais para o Agent e para
o admin — os dois estao em `staging`. E o *"lado Agent do recall ainda nao existe"* tambem caiu: o
recall esta completo dos dois lados.

**Consequencia pratica, dita com todas as letras:** dos 24 cenarios, **7 podem ser executados
hoje** (C1 a C7), do jeito que a maquina esta. Os outros 17 esperam pre-requisito de outra pessoa.
O numero de cenarios executaveis hoje **nao subiu** com o push de hoje, e isso e o ponto: o que
travava o Bloco 3 nunca foi o commit, foi o deploy e o numero de versao.

---

## Tres armadilhas que ja custaram caro — nao repita

**1. Nao rode `tests\e2e\scenarios\e1-alt-rollback-crash.ps1` nesta maquina.**

Ele parece ser exatamente o teste do Bloco 4, e nao e. A primeira linha dele chama o
`teardown.ps1`, que **desinstala o Agent e apaga a pasta `C:\ProgramData\ManagerAgent` inteira** —
levando junto a vinculacao, os buffers e todos os logs desta maquina. Ele tambem grava uma
variavel de ambiente permanente na maquina (`MANAGER_STALE_MINUTES_OVERRIDE`) e nunca a remove.
Ele foi escrito para maquina descartavel. O Bloco 4 deste roteiro faz o mesmo teste **sem**
destruir a instalacao, no modelo do roteiro da 1.5.7, que ja rodou bem aqui em 19/08.

**2. Desligar o auto-update pelo arquivo de configuracao nao funciona.**

`AutoUpdate.Habilitado` no `appsettings.json` **nao e lido por codigo nenhum**. O unico freio real
do auto-update e o modo SOS no `watchdog-state.json`. Se em algum momento voce precisar que a
maquina pare de procurar update, e por ali — e lembrando que ligar o SOS tambem para a captura de
voltar sozinha depois.

**3. Nao copie a instalacao por cima do `bin.previous` para preparar o teste da guarda A.**

O documento do @Bucky mandava fazer exatamente isso (era o setup do R10.11.7). **Nao faca sem ler
o C15.** O `bin.previous` e a unica copia de seguranca que a maquina tem, e e para ela que a volta
automatica do Bloco 4 retorna. Sobrescreve-la para preparar um teste de recall **desarma o Bloco
4** — e o Bloco 4 e o que bloqueia producao. O C15 traz o passo de guardar e devolver a copia, e a
ordem correta: **C15 so depois de C12, C13 e C14 concluidos.**

---

## Como registrar evidencia

Antes de comecar qualquer cenario:

```powershell
New-Item -ItemType Directory -Path "C:\Temp\ManagerAgent-Tests\2026-08-24" -Force
```

Ao fim de cada cenario, guarde nessa pasta:

```powershell
$ev = "C:\Temp\ManagerAgent-Tests\2026-08-24"
Copy-Item "C:\ProgramData\ManagerAgent\watchdog-state.json" "$ev\CN-watchdog-state.json"
Copy-Item "C:\ProgramData\ManagerAgent\logs\service-$(Get-Date -Format yyyyMMdd).log" "$ev\CN-service.log"
```

Trocando `CN` pelo numero do cenario. **Sem o log, o cenario nao vale** — a leitura de todos os
criterios abaixo e feita no log, nao na tela.

Quase tudo aqui precisa de **PowerShell aberto como administrador**.

---

## Mapa dos cenarios

**Leia de cima para baixo: esta e a ordem de execucao.** Vai do mais seguro para o mais
arriscado, e o que pode derrubar a captura fica por ultimo.

| # | Cenario | Precisa de | Risco para a captura | Origem |
|---|---|---|---|---|
| **BLOCO 1 — hoje, sem backend, sem risco** | | | | |
| C1 | Conferencia do estado inicial | nada | nenhum | @Natasha |
| C2 | Reboot | nada | ~2,5 min sem captura, esperado | @Natasha |
| C3 | Logoff e login | nada | segundos | @Natasha |
| C4 | Parar o Service com o usuario logado | nada | ~6 min, esperado | @Natasha |
| C5 | Desligar e ligar com a segunda conta desconectada | conta `Marcos` logada e desconectada | ~2,5 min | @Natasha |
| C6 | Troca rapida entre dois usuarios | P7 | nenhum | @Natasha |
| C7 | Servico derrubado com dois usuarios logados | P7 | ~6 min | @Natasha |
| **BLOCO 2 — hoje, sem backend, derruba a instalacao (recuperavel)** | | | | |
| C8 | Desinstalar e instalar do zero | P8 | ate reinstalar | @Natasha |
| **BLOCO 3 — precisa de staging no ar** | | | | |
| C9 | Lote nao cai inteiro por causa de um item ruim | P1 + P2 | nenhum | @Natasha |
| C10 | Conferencia no banco | P2 + P6 | nenhum | @Natasha |
| C20 | Quem o recall NAO alcanca | P2 | nenhum | @Bucky (10.11.2/.6/.16) |
| C18 | Onde o recall aparece: painel, audit e logs | P2 + C11 executado | nenhum (so leitura) | @Bucky (10.11.5/.13/.14/.15) |
| C16 | Recall sem copia de seguranca nao liga SOS (guarda B) | P2 + P3 + P4 | baixo | @Bucky (10.11.8) |
| C17 | O recall nao se repete a cada reinicio (guarda C) | P2 + P3 + P4 | baixo | @Bucky (10.11.9/.10) |
| C11 | Recall de frota — a maquina volta sozinha | P2 + P3 + P4 | medio | @Natasha + @Bucky (10.11.1/.3/.4) |
| C19 | Recall que falha nao liga SOS | P2 + P3 + P4 | medio-alto | @Bucky (10.11.11) |
| **BLOCO 4 — por ultimo: a volta automatica de versao** | | | | |
| C12 | A maquina volta sozinha para a versao anterior | P3 + P4 + P5 | **alto** | @Natasha + @Bucky (10.5) |
| C21 | A auto-recuperacao do Windows voltou no lugar | C12 executado | nenhum (so leitura) | @Bucky (10.6 / E6) |
| C13 | Nao reinstala a versao quebrada no ciclo seguinte | C12 aprovado | alto | @Natasha + @Bucky (10.7) |
| C14 | O registro guarda qual versao quebrou | C12 executado | nenhum (so leitura) | @Natasha |
| C22 | A volta acontece com dois usuarios logados | C12 aprovado + P7 | **alto** | @Bucky (10.9) |
| C15 | Recall recusado pela guarda A (backup na mesma versao) | C12–C14 concluidos + P2 + P3 + P4 | medio | @Bucky (10.11.7) |
| **BLOCO 5 — so com autorizacao (P9): derruba a captura DE PROPOSITO** | | | | |
| C23 | Volta sem copia de seguranca: entra em SOS, e o esperado | P9 + P5 | **maquina parada, por desenho** | @Bucky (10.8) |
| C24 | Maquina desligada no meio da volta | P9 + P5 + P6 | **maquina parada, por desenho** | @Bucky (10.10) |

---

# BLOCO 1 — da para fazer hoje, sem backend

Estes sete cenarios rodam com a maquina do jeito que ela esta. Nao dependem de deploy, nem de
numero de versao novo, nem de banco. Sao os manuais que ficaram pendentes das baterias da 1.5.10
e da 1.5.12.

**Faca na ordem.** Cada um deixa a maquina num estado limpo para o proximo.

---

## C1 — Conferencia do estado inicial

Nao e um teste, e a foto de antes. Sem ela, qualquer coisa estranha que aparecer depois nao tem
com o que ser comparada.

**Pre-requisito:** nenhum.

**Passos:**

1. Abra o PowerShell como administrador.
2. Rode:

```powershell
Get-Service ManagerAgent, ManagerAgentWatchdog | Select-Object Name, Status, StartType
(Get-Item 'C:\Program Files\ManagerAgent\ManagerAgent.Service.exe').VersionInfo.ProductVersion
Test-Path 'C:\Program Files\bin.previous'
(Get-Item 'C:\Program Files\bin.previous\ManagerAgent.Service.exe').VersionInfo.ProductVersion
Get-ChildItem 'C:\Program Files' -Filter 'bin.failed-*' -Directory
Get-Content 'C:\ProgramData\ManagerAgent\watchdog-state.json'
Get-Process ManagerAgent.SessionWorker -ErrorAction SilentlyContinue | Select-Object Id, StartTime
```

3. **Foto da auto-recuperacao do Windows** — este passo e novo, e existe so para o C21 ter com o
   que comparar depois:

```powershell
sc.exe qfailure ManagerAgent
sc.exe qfailure ManagerAgentWatchdog
```

4. Guarde a saida inteira em `C:\Temp\ManagerAgent-Tests\2026-08-24\C1-estado-inicial.txt`.

**O que observar:** os dois servicos rodando, um unico processo `SessionWorker` por usuario
logado, o modo SOS desligado, e os dois servicos com auto-recuperacao configurada em
`restart 5s / 10s / 30s`.

**Passou se:** o quadro bate com o "Estado da maquina" no topo deste documento.

**Reprovou se:** o modo SOS estiver ligado, ou faltar um dos dois servicos, ou existir mais de um
`SessionWorker` na mesma sessao.

**Se reprovar:** pare o roteiro. Modo SOS ligado significa que a maquina ja nao esta atualizando
nem se protegendo, e nenhum cenario abaixo mede o que promete. Chame o @Bucky.

---

## C2 — Reboot

Refaz o item 1 da lista dos 15. Ja passou na 1.5.10; precisa passar de novo porque o codigo mudou
tres vezes desde entao.

**Pre-requisito:** C1 guardado. Feche seus programas — a maquina vai reiniciar.

**Passos:**

1. Anote a hora.
2. Reinicie o Windows pelo menu Iniciar (reinicio normal, nao forcado).
3. Faca login normalmente e espere **5 minutos** sem mexer em nada.
4. Leia o log:

```powershell
Select-String -Path "C:\ProgramData\ManagerAgent\logs\service-$(Get-Date -Format yyyyMMdd).log" `
  -Pattern 'AGENT_STARTUP|LOGIN|LOGOUT|SESSAO_INTERROMPIDA|attempting re-launch|Worker launched'
```

**O que observar:**

- O Agent sobe sozinho, **cerca de 2 minutos e meio depois do boot**. Essa demora e por
  configuracao (inicio atrasado, para nao concorrer com o boot do Windows) e **nao e defeito** —
  ja foi medida em 2min35s na 1.5.10.
- Deve sair **um** LOGIN, nao dois. Um segundo LOGIN **suprimido** com a mensagem
  `LogonId ja registrado` nao conta como duplicado — e a deduplicacao funcionando.
- **A `SESSAO_INTERROMPIDA` depende de como a sessao anterior terminou** — leia o criterio
  corrigido abaixo antes de julgar.

> **Criterio corrigido em 2026-08-24 (@Tony), depois de medir na maquina.**
>
> A redacao anterior dizia: *"deve sair uma `SESSAO_INTERROMPIDA`, e aqui ela e legitima: reboot
> nao tem despedida limpa do worker"*. **Isso deixou de valer com o achado A-62.**
>
> Hoje o `SessaoInterrompidaDecider` **compara** a flag de desligamento limpo com o
> `LogonId` da sessao anterior, em vez de apenas verificar se a flag existe:
>
> | Situacao | O que deve acontecer |
> |---|---|
> | Flag do **mesmo** LogonId anterior | **suprime** — aquela sessao terminou limpa |
> | Flag de **outro** LogonId | **emite** — a flag nao diz nada sobre esta sessao |
> | Flag ausente | **emite** |
> | Flag **sem** LogonId (versao anterior a 1.5.11) | **suprime** — compatibilidade de frota |
>
> **Reinicio limpo pelo menu Iniciar normalmente NAO gera `SESSAO_INTERROMPIDA`**, porque a
> sessao anterior alcanca gravar a flag. Medido em 24/08: a flag lida era do logon de 21/08, que
> terminou limpo, e a supressao estava certa.
>
> Para exercitar a emissao de verdade, use o **C4** (parar o Service com o usuario logado) ou
> corte a energia — sao os casos em que a flag nao chega a ser escrita.

**Passou se:** um LOGIN emitido, os dois servicos rodando, nenhum evento de janela de antes do
reboot continuando aberto, e a decisao sobre `SESSAO_INTERROMPIDA` batendo com a tabela acima —
inclusive quando a decisao correta for **nao emitir**.

**Reprovou se:** dois ou mais LOGIN **emitidos**; nenhum LOGIN; o Agent nao subir em ate 5
minutos; ou a `SESSAO_INTERROMPIDA` divergir da tabela acima. Se a decisao divergir, **leia o log
do decider antes de abrir achado** — ele registra qual LogonId comparou com qual.

**Se reprovar:** guarde `service-*.log` e `startup-trace.log` do dia e siga para o C3 — este
achado nao impede os proximos.

---

## C3 — Logoff e login

Refaz o item 5. Passou na 1.5.12 (21/08, 17:02), mas naquela rodada a corrida que interessa nao
aconteceu — o registro foi limpo depois do LOGOUT, que e a ordem boa. Aqui a gente confirma que
continua assim.

**Pre-requisito:** C2 concluido.

**Passos:**

1. Anote a hora.
2. Faca **logoff** (sair da conta, nao bloquear a tela).
3. Faca login de volta e espere **2 minutos**.
4. Leia:

```powershell
Select-String -Path "C:\ProgramData\ManagerAgent\logs\service-$(Get-Date -Format yyyyMMdd).log" `
  -Pattern 'LOGOUT|LOGIN|SESSAO_INTERROMPIDA|clean-shutdown|attempting re-launch|confirmou logoff'
```

**O que observar, quatro coisas:**

| O que | Esperado |
|---|---|
| Quantidade de LOGOUT | exatamente **um** |
| Horario do LOGOUT | o instante do logoff (o momento em que o worker se desconecta), nao o do processamento |
| Login seguinte | emite LOGIN e **nao** emite `SESSAO_INTERROMPIDA` |
| Entre a queda do worker e o logoff | **nenhum** `attempting re-launch` |

**Passou se:** as quatro linhas acima batem.

**Reprovou se:** nao sair LOGOUT nenhum (era o A-62), sair dois, ou aparecer
`SESSAO_INTERROMPIDA` no login seguinte.

**Se reprovar:** copie a janela do log entre o logoff e o login e mande para o @Bucky. Nao impede
os proximos cenarios.

---

## C4 — Parar o Service com o usuario logado

E o caminho 1 do A-65, que **nunca foi testado em maquina**. Diferente do C3: aqui o worker esta
vivo e se despede sozinho — quem emite o LOGOUT e ele, nao o Service.

**Pre-requisito:** voce logado, com o icone do Agent na bandeja.

**Passos:**

1. Anote a hora.
2. Pare o servico:

```powershell
Stop-Service ManagerAgent
```

3. Espere **1 minuto** e confira que ele parou.
4. Suba de novo:

```powershell
Start-Service ManagerAgent
```

5. Espere **2 minutos** e leia o log com o mesmo comando do C3.

**O que observar:**

- Na parada, o Agent tenta um **ultimo envio** antes de morrer.
- Sai **uma** linha de LOGOUT, nunca duas — e a diferenca entre este cenario e o C3 e justamente
  quem a emitiu.
- Nao pode existir LOGOUT sem o LOGIN correspondente depois.

**Atencao ao tempo:** se voce **nao** subir o servico na mao, o Watchdog leva cerca de **6
minutos** para trazer de volta — e proposital (ele exige duas confirmacoes antes de agir, para
nao reagir a falso alarme). Ja foi medido: 5min21s. Isso nao e defeito.

**Passou se:** um LOGOUT com o horario da parada, um LOGIN na volta, e nenhuma sessao aparecendo
encerrada com evento posterior ao encerramento.

**Reprovou se:** dois LOGOUT, ou LOGOUT sem LOGIN de volta, ou o servico nao voltar.

**Se reprovar:** `Start-Service ManagerAgent` resolve o servico. Guarde o log e siga.

---

## C5 — Desligar e ligar com a segunda conta desconectada

E o caminho 3 do A-65, tambem **nunca testado**, somado ao item 6 da lista dos 15. O caso e:
alguem desliga a maquina enquanto a sessao de outro usuario ainda esta aberta, so que
desconectada.

**Pre-requisito:** conta `Marcos` disponivel (P7).

**Passos:**

1. Troque para a conta `Marcos` (Trocar usuario) e use por **3 minutos** — abra o navegador,
   troque de janela algumas vezes.
2. **Troque de volta** para a sua conta, sem fazer logoff da conta `Marcos`. Ela fica logada e
   desconectada.
3. Confirme que ha dois workers:

```powershell
Get-Process ManagerAgent.SessionWorker | Select-Object Id, SessionId, StartTime
```

4. **Desligue** a maquina (Desligar, nao reiniciar). O Windows vai avisar que outra pessoa esta
   logada — confirme.
5. Ligue de novo, faca login e espere **5 minutos**.
6. Leia o log do dia anterior e o do dia, procurando por LOGOUT e por `SESSAO_INTERROMPIDA`.

**O que observar:**

- A sessao desconectada tambem tem de ser **fechada** no desligamento. Era exatamente o defeito:
  ela ficava aparecendo como encerrada com eventos depois do encerramento.
- Nenhum worker pode **nascer** durante o desligamento. Se aparecerem linhas de relancamento no
  meio do shutdown, e o A-61 de volta.

**Passou se:** as duas sessoes fecham, e nenhum worker e lancado durante o desligamento.

**Reprovou se:** a sessao da conta `Marcos` nao gerar LOGOUT, ou aparecer worker nascendo no meio
do desligamento.

**Se reprovar:** anote os horarios exatos das linhas e passe ao @Bucky. Nao impede os proximos.

---

## C6 — Troca rapida entre dois usuarios

Item 7 da lista dos 15 — e o **unico cenario que exercita de verdade a corrida do A-63**. Na
bateria da 1.5.12 ela nao aconteceu, e o proprio registro daquele dia diz com todas as letras:
"o teste prova que nada regrediu, nao que a correcao agiu". Este cenario existe para fechar isso.

**Pre-requisito:** conta `Marcos` (P7). Reserve 15 minutos.

**Passos:**

1. Anote a hora.
2. Troque para a conta `Marcos` (Trocar usuario) e **use de verdade por 2 minutos**.
3. Troque de volta para a sua conta. Use por 2 minutos.
4. Repita a troca mais **duas vezes**, rapido — o objetivo e a segunda sessao demorar a subir.
5. Faca **logoff** da conta `Marcos` e volte para a sua.
6. Leia:

```powershell
Select-String -Path "C:\ProgramData\ManagerAgent\logs\service-$(Get-Date -Format yyyyMMdd).log" `
  -Pattern 'LOGOUT|LOGIN|Usuario=|Worker removed|CONNECT received|Failed to launch'
```

**O que observar:**

- **Todo LOGOUT tem de sair com o nome do dono preenchido.** Um LOGOUT com o nome vazio e a
  falha que este cenario procura.
- O logoff de um usuario **nao pode** deixar o outro sem worker.
- Um worker por sessao, nunca dois na mesma.

**Ruido conhecido, nao e falha deste cenario:**

- `Task Scheduler launch succeeded but worker process not found for session N` — aparece quando a
  pessoa ainda esta digitando a senha. Ja identificado, e mensagem enganosa, nao erro.
- `Falha ao persistir session-state ... Access to the path is denied` na sessao 2 — e o A-66, que
  **saiu da lista de bloqueadores por decisao registrada** (so acontece em maquina compartilhada,
  e o cliente e maquina individual). Anote quantas vezes apareceu e siga.

**Passou se:** nenhum LOGOUT com nome vazio, nenhuma sessao ficou sem worker, e cada evento saiu
carimbado com o usuario certo.

**Reprovou se:** qualquer LOGOUT sem nome, ou uma sessao ativa sem worker por mais de 90
segundos.

**Se reprovar:** este e um achado que interessa — copie a sequencia inteira de log da troca e
mande para o @Bucky **e** para o @Tony, porque fecha (ou reabre) o A-63.

---

## C7 — Servico derrubado com dois usuarios logados

Item 13 da lista dos 15. Prova que, quando o Service cai com duas pessoas trabalhando, **cada uma
preserva os seus eventos** — nenhum se perde e nenhum vai para a conta errada.

**Pre-requisito:** C6 concluido; as duas contas logadas ao mesmo tempo (a sua ativa, a `Marcos`
desconectada).

**Passos:**

1. Nas duas contas, gere atividade: abra e troque janelas por 2 minutos em cada uma.
2. Volte para a sua conta e derrube o Service **a forca**:

```powershell
Stop-Process -Name ManagerAgent.Service -Force
```

3. Espere **8 minutos** sem mexer (o Watchdog precisa das duas confirmacoes).
4. Confirme que voltou:

```powershell
Get-Service ManagerAgent
Get-Process ManagerAgent.SessionWorker | Select-Object Id, SessionId
```

5. Confira o tamanho dos buffers por sessao:

```powershell
Get-ChildItem 'C:\ProgramData\ManagerAgent\data' | Select-Object Name, Length
```

**O que observar:**

- Existe **um arquivo de buffer por sessao** (`autonomous-buffer-S1.db`, `-S2.db`, ...). Foi assim
  que o A-60 foi resolvido.
- Depois da volta, os eventos das duas sessoes precisam sair, cada um com o seu dono.
- Nenhum erro de "banco somente leitura" na sessao do segundo usuario — esse e o item 12 da lista,
  e vai junto neste cenario.

**Passou se:** o Service volta sozinho, os dois workers voltam, e nao ha erro de banco
somente-leitura no log da segunda sessao.

**Reprovou se:** aparecer `attempt to write a readonly database` na sessao 2, ou um dos workers
nao voltar.

**Se reprovar:** e regressao do A-60 — pare e chame o @Bucky antes de seguir.

---

# BLOCO 2 — hoje, sem backend, mas derruba a instalacao

## C8 — Desinstalar e instalar do zero

Ficou pendente desde a 1.5.10 ("por ultimo: destroi a maquina de teste"). **Faca depois de todo o
Bloco 1**, porque apaga a vinculacao e os buffers, e todos os cenarios acima perdem a historia.

**Pre-requisito:** P8 — identificador do colaborador e **chave de ativacao** da empresa de teste
em maos. Sem a chave, a maquina fica instalada e sem vinculo — ou seja, sem capturar.

**Passos:**

1. **Antes de desinstalar**, anote o identificador da instalacao e guarde uma copia dos logs:

```powershell
Copy-Item 'C:\ProgramData\ManagerAgent\logs' 'C:\Temp\ManagerAgent-Tests\2026-08-24\C8-logs-antes' -Recurse
Copy-Item 'C:\ProgramData\ManagerAgent\watchdog-state.json' 'C:\Temp\ManagerAgent-Tests\2026-08-24\C8-state-antes.json'
```

2. Desinstale pelo Painel de Controle.
3. Confira que nao sobrou nada:

```powershell
Get-Service ManagerAgent, ManagerAgentWatchdog -ErrorAction SilentlyContinue   # esperado: vazio
Test-Path 'C:\Program Files\ManagerAgent'                                       # esperado: False
Get-Process -Name 'ManagerAgent.*' -ErrorAction SilentlyContinue                # esperado: vazio
Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Environment' `
  -Name DOTNET_BUNDLE_EXTRACT_BASE_DIR -ErrorAction SilentlyContinue            # esperado: nada
```

4. Instale de novo, como administrador:

```powershell
Start-Process 'C:\Users\NoisyTech\Documents\Manager\manager-srv-agent\instalador\ManagerAgent-Setup-v1.5.12.exe' -Verb RunAs
```

5. Informe identificador e chave no assistente.
6. Espere 3 minutos e confira:

```powershell
Get-Service ManagerAgent, ManagerAgentWatchdog | Select-Object Name, Status, StartType
Get-ChildItem 'C:\Program Files\ManagerAgent\scripts'
```

**O que observar:**

- Na desinstalacao: os **dois** servicos saem, a pasta some, e o dispositivo fica desvinculado no
  backend (o desinstalador roda o script de desvinculacao).
- Na instalacao: dois servicos rodando, **6 scripts** na pasta `scripts`, e o token guardado
  **cifrado** no `config.json` — nunca em texto claro.

**Passou se:** desinstala limpo e instala vinculado, capturando de novo.

**Reprovou se:** sobrar servico registrado, sobrar a chave de ambiente no registro do Windows, ou
a vinculacao nao concluir.

**Se reprovar:** a maquina fica sem captura. O caminho de volta e reinstalar com o mesmo
instalador; se o instalador recusar por instalacao existente, remova a pasta
`C:\Program Files\ManagerAgent` a mao e repita.

**Depois deste cenario a maquina volta a 1.5.12 de 21/08** — o mesmo ponto de partida do Bloco 4.

---

# BLOCO 3 — precisa de staging no ar

**Nada deste bloco roda hoje.** O codigo do Agent e do admin ja esta em `staging` (commits
`39d20e3` e `e86124d`), mas **empurrar codigo nao e subir codigo**: enquanto o ambiente de staging
nao for redeployado (P2), qualquer cenario daqui mede o backend **antigo** e o resultado nao vale
— passar nao prova nada e reprovar acusa um defeito que ja foi corrigido. O C9 tem ainda o P1 por
cima: a correcao dele nem chegou a ser commitada.

**Os cenarios de recall (C11 e C15 a C20) precisam de mais uma coisa alem do staging:** a maquina
tem de estar rodando uma versao **cujo Agent entende o comando de recall** (P3 + P4). A 1.5.12
instalada aqui hoje **nao entende** — ela vai ignorar o recall em silencio, e isso e o
comportamento correto, nao uma reprovacao. Ver o C20.

### Como a ordem de recall viaja de um lado ao outro

Vale para todos os cenarios de recall, e e a primeira coisa a conferir em qualquer um deles:

- Quem **le** a resposta do backend e o **Service**, no ciclo de 6h.
- Quem **executa** a restauracao e o **Watchdog**, no ciclo de 60s.
- O Service so **marca** os campos `recallPendente*` no `watchdog-state.json`.

Ou seja: depois de o Service ver o recall, some **ate 60s** para a restauracao comecar.
**Se o pedido nao aparecer no JSON, o problema esta no Service; se aparecer e nada acontecer, esta
no Watchdog.** Conferir o `watchdog-state.json` e o passo 1 de todo cenario de recall.

---

## C9 — Um item ruim nao derruba o lote inteiro

E o item 1 da linha de corte, feito pela @Shuri. Hoje, um unico evento invalido faz o backend
recusar o lote inteiro; depois de cinco recusas o Agent **descarta** ate 100 eventos. Isso e perda
de dado silenciosa, e so o backend fecha.

**Pre-requisito:** P1 **e** P2. Atencao: este e o unico cenario cujo codigo **nao saiu da maquina
da @Shuri**. Diferente do Agent e do admin, o `manager-srv-events-node` esta com a correcao so na
arvore de trabalho, sem commit e sem push. A maquina precisa tambem estar apontando para staging
(confira no `config.json`, com PowerShell elevado: o pacote instalado aponta para **producao** por
padrao, e quem troca e o Configurator).

**Passos:**

1. Peca a @Shuri o comando pronto de envio de um lote de teste com **dois itens: um valido e um
   invalido** (tipo errado num campo, nao campo faltando — campo faltando o backend aceita, e isso
   ja esta medido).
2. Envie o lote contra o `/eventos` de staging.
3. Anote o codigo de resposta e o corpo.
4. Depois, deixe a maquina trabalhando normalmente por 30 minutos e leia:

```powershell
Select-String -Path "C:\ProgramData\ManagerAgent\logs\service-$(Get-Date -Format yyyyMMdd).log" `
  -Pattern 'lote recusado por conteudo|DESCARTADO|FailedBatches|Descartados'
```

**O que observar:**

- A resposta ao lote misto deve ser **202**, e o corpo deve trazer o item recusado com o
  **indice certo** do lote — nao o do item vizinho.
- O item valido tem de aparecer no banco (isso e o C10).
- Na operacao normal, os ciclos devem fechar com `FailedBatches=0` e `Descartados=0`.

**Passou se:** o lote misto responde 202, o item bom entra, o item ruim volta identificado, e
nenhum ciclo normal registra recusa por conteudo.

**Reprovou se:** o lote misto for recusado inteiro (400), ou o indice do item recusado apontar
para o item errado.

**Se reprovar:** e bloqueador de producao — item 1 da linha de corte continua aberto. Registre e
avise @Shuri e @Tony.

---

## C10 — Conferencia no banco

Ficou pendente da bateria da 1.5.12: **nunca foi feita**, por falta de acesso. E a unica forma de
provar que o que saiu da maquina chegou inteiro do outro lado.

**Pre-requisito:** P2 e P6 — staging no ar e acesso ao banco com senha nova.

**Passos:**

1. Com os horarios anotados nos cenarios C2 a C7, consulte a base de staging.
2. Confira, para o periodo dos testes:

| O que conferir | Esperado |
|---|---|
| `eventos_sessao`, LOGOUT do C3 | **uma unica linha**, com o nome do usuario preenchido |
| `eventos_sessao`, LOGOUT do C4 | uma unica linha — e nao duas somadas com a do C3 |
| Sessoes do C5 | as duas fecham, nenhuma com evento posterior ao encerramento |
| Eventos do C6 | cada um no usuario Windows que de fato o gerou |
| `eventos_reuniao` | se voce entrou em alguma reuniao, confira que nao viraram duas linhas (era o A-57) |

**Passou se:** cada evento aparece uma vez so, com o dono certo e o horario do acontecimento (nao
o da entrega).

**Reprovou se:** LOGOUT duplicado, LOGOUT sem nome, ou linha aberta que nunca fecha.

**Se reprovar:** guarde os `id` das linhas — sem eles o @Shuri nao consegue reproduzir.

---

## C20 — Quem o recall NAO alcanca

> **Origem:** absorve os casos R10.11.2, R10.11.6 e R10.11.16 do documento do @Bucky.

Este cenario prova o **limite** do recall, e vem antes dos outros de proposito: e o mais barato,
nao mexe em nada na maquina, e evita a leitura errada mais provavel do resto do bloco — concluir
que "o recall nao funcionou" quando ele fez exatamente o que devia.

**Pre-requisito:** P2 (staging no ar). **Nao** precisa de versao nova instalada — pelo contrario:
uma das tres pontas so da para observar com a 1.5.12 que ja esta aqui.

**Os tres casos, e o que cada um significa em portugues claro:**

| Caso | Situacao | Esperado | Por que |
|---|---|---|---|
| a | Maquina com Agent **anterior ao recall** (a 1.5.12 de hoje) | **ignora, sem erro e sem aviso**, e segue o fluxo normal de update | O campo e novo. Um Agent que nao o conhece nao quebra por causa dele — e o requisito de compatibilidade |
| b | Maquina rodando uma versao **diferente** da revogada | **ignora** | O recall e por versao. Revogar a 1.5.13 nao pode mexer em quem esta na 1.5.14 |
| c | Maquina em **modo SOS** | **nao pergunta ao backend, logo nao ouve o recall, e nada acontece** | A guarda de SOS retorna antes da chamada. SOS significa "nao encoste" |

**Passos:**

1. **Caso a** — com a maquina como esta hoje (1.5.12), peca ao @Vision para revogar a `1.5.12` em
   staging. Espere um ciclo e leia o log do Service. **Nao pode** aparecer nada de recall, nem
   erro de desserializacao, nem excecao. O ciclo de update tem de fechar normal.
2. **Caso b** — com a maquina numa versao N, peca a revogacao de uma versao **diferente** de N.
   Espere um ciclo. Nada pode acontecer, e `recallPendenteVersao` tem de continuar vazio no
   `watchdog-state.json`.
3. **Caso c** — este muda a maquina, entao faca por ultimo e desfaca em seguida:

```powershell
# antes: guarde o estado
Copy-Item 'C:\ProgramData\ManagerAgent\watchdog-state.json' "$ev\C20-state-antes.json"
```

   Ponha `sosMode: true` no `watchdog-state.json`, reinicie o Service, peca a revogacao da versao
   instalada e observe por dois ciclos.

**Passou se:** nos tres casos **nada acontece na maquina** e nenhum erro aparece no log. No caso
c, o log tem de mostrar que a maquina **nem chegou a perguntar** ao backend.

**Reprovou se:** o caso a gerar erro ou excecao (compatibilidade quebrada), ou o caso b disparar
uma volta de versao (recall pegando quem nao devia).

**OBRIGATORIO ao terminar o caso c:** desfaca o SOS, **ele nao sai sozinho**.

```powershell
# troque sosMode para false no watchdog-state.json e depois:
Restart-Service ManagerAgent
Restart-Service ManagerAgentWatchdog
```

   Confirme que voltou a `false` antes de seguir. **Se esquecer isso, a maquina fica instalada,
   sem procurar update e sem capturar direito** — e todo cenario seguinte mede errado.

**Nota que precisa chegar a quem aperta o botao (era a "limitacao conhecida" do @Bucky):** o
recall **nao alcanca a frota inteira**. As maquinas em SOS ficam de fora e continuam rodando a
versao revogada. Elas ja estavam fora do auto-update e ja precisavam de alguem indo la resolver na
mao. Quem for medir o efeito de um recall **precisa contar essas maquinas a parte** — senao a
conta de "quantas voltaram" da menos que a frota e a conclusao errada seria "o recall nao
funcionou".

---

## C18 — Onde o recall aparece: painel, audit e logs

> **Origem:** absorve R10.11.5, R10.11.13, R10.11.14 e R10.11.15 do documento do @Bucky.
> **Este cenario foi corrigido na absorcao — ver a caixa abaixo.**

E so leitura: nao mexe em nada na maquina. Roda **depois** de qualquer cenario de recall ter
acontecido (C11, C15, C16, C17 ou C19), sobre a evidencia que ele deixou.

**Pre-requisito:** P2, e pelo menos um cenario de recall ja executado.

**O que conferir:**

| # | O que | Esperado |
|---|---|---|
| 1 | Nome dos eventos no painel | `UPDATE_RECALL_TRIGGERED`, `UPDATE_RECALL_SKIPPED` ou `UPDATE_RECALL_FAILED` — **nunca** `UPDATE_ROLLBACK_TRIGGERED` para um recall |
| 2 | Nivel dos tres eventos | **`WARN` (aviso)** nos tres |
| 3 | Nivel do `_FAILED` | `WARN`, **nao** `ERROR` — `ERROR` neste repo e o nivel do `AGENT_SOS_MODE_ACTIVE`, e recall que falha **nao** liga SOS |
| 4 | Audit do disparo | traz **autor e motivo** de quem revogou |
| 5 | O motivo no log local, **nos dois processos** | o texto digitado pelo operador aparece no log do **Service** (quando a resposta chegou) **e** no log do **Watchdog** (quando ele agiu) |
| 6 | Quando o backend nao mandar motivo | os dois logs dizem isso **com todas as letras**. Silencio nao serve |
| 7 | O `run-rollback.ps1` gerado | continua **ASCII puro**, **sem** o texto do operador |

**Sobre o item 7, para nao virar achado errado:** o motivo *nao* estar no script e proposital. O
motivo e texto livre com acento (o exemplo do proprio contrato e *"Captura de tela saindo em
branco na 1.5.13"*), e no `run-rollback.ps1` ele quebraria o parser do PowerShell 5.1 (A-05) — no
meio de uma maquina que acabou de receber ordem de reverter. **Motivo no log do C# e no audit; o
PS1 continua ASCII.** Se voce achar o motivo dentro do PS1, **isso** e o achado.

> **CORRECAO NA ABSORCAO — o aviso do @Bucky sobre o filtro de nivel esta VENCIDO.**
> O R10.11.13 mandava "consultar sem filtro de nivel" porque os tres eventos chegavam ao painel
> como `INFO` e sumiriam de qualquer consulta filtrada por `WARN`. **Isso foi corrigido hoje**
> pelo commit `e86124d` do `manager-srv-admin-node`: os tres entraram no mapa de tipos conhecidos
> como `WARN`. **A instrucao antiga foi removida** — se ela tivesse sido absorvida como estava,
> mandaria o testador desligar exatamente o filtro que agora prova que a correcao funcionou.
> O item 2 desta tabela e o que substitui: agora conferir o nivel **e** parte do teste.

**Passou se:** os sete itens batem.

**Reprovou se:** um evento de recall sair com nome de rollback (item 1 — mistura os dois caminhos
e estraga o criterio de "telemetria zero" do rollout), ou os eventos nao aparecerem numa consulta
filtrada por aviso (item 2 — a correcao de hoje nao pegou).

---

## C16 — Recall sem copia de seguranca nao liga SOS (guarda B)

> **Origem:** R10.11.8 do documento do @Bucky.

Este e o cenario que protege a frota contra o pior efeito colateral possivel do recall: **uma
maquina que, por nao ter para onde voltar, sai do ar e deixa de receber a correcao que ia
consertar o problema que motivou o recall.**

**Pre-requisito:** P2 + P3 + P4 — staging no ar e a maquina rodando uma versao N que entende
recall. Precisa de uma instalacao **sem** `bin.previous` — ou seja, uma instalacao nova que nunca
atualizou.

**Antes de comecar, guarde a copia de seguranca de verdade:**

```powershell
$ev = "C:\Temp\ManagerAgent-Tests\2026-08-24"
Move-Item 'C:\Program Files\bin.previous' 'C:\Program Files\bin.previous.guardado'
```

**Passos:**

1. Confirme que `C:\Program Files\bin.previous` **nao existe**.
2. Peca ao @Vision para revogar a versao instalada.
3. Espere o ciclo (ate 6h, ou reinicie o Service para forcar a checagem imediata).
4. Leia o log do Watchdog e o `watchdog-state.json`.

**O que observar:**

- Log com `Recall RECUSADO pela guarda B`.
- Audit `UPDATE_RECALL_SKIPPED` com `motivo=sem_backup`.
- **E o ponto do teste:** `sosMode` continua **`false`** no `watchdog-state.json`.
- A maquina segue capturando e segue elegivel ao auto-update.

**Passou se:** as quatro coisas acima. **Conferir o `sosMode` e obrigatorio** — nao vale "o log
disse que recusou".

**Reprovou se:** `sosMode` virar `true`. E o oposto do desenho: SOS aqui tiraria a maquina do
proprio conserto.

**OBRIGATORIO ao terminar — devolva a copia de seguranca:**

```powershell
Move-Item 'C:\Program Files\bin.previous.guardado' 'C:\Program Files\bin.previous'
```

Sem isso, o Bloco 4 fica sem rede de seguranca.

---

## C17 — O recall nao se repete a cada reinicio (guarda C)

> **Origem:** R10.11.9 e R10.11.10 do documento do @Bucky, juntados num cenario so porque um sem o
> outro engana: o primeiro prova que a ordem para de sair, e o segundo prova que ela nao para
> demais.

**Pre-requisito:** P2 + P3 + P4, e um recall ja recusado (o C15 ou o C16).

**Por que existe:** o gatilho perigoso **nao** e o ciclo de 6h. E a checagem que o Service faz
**assim que sobe**. Numa maquina que reinicia em laco — que e exatamente a maquina que acabou de
falhar um recall — sem um carimbo a ordem sairia a cada boot.

**Parte 1 — a ordem para de sair (era o R10.11.9):**

1. Depois de um recall recusado, reinicie o Service **duas ou tres vezes seguidas**, dentro de 1
   hora.
2. Leia o log e o `watchdog-state.json`.

**Passou se:** o log diz `Guarda C`, o pedido **nao** e remarcado, e
`recallUltimaTentativaUtc` e `recallUltimaVersaoTentada` estao preenchidos no JSON.

**Parte 2 — a guarda C nao congela a maquina (era o R10.11.10):**

1. Com a guarda C carimbada para a versao X, coloque a maquina numa versao **Y, tambem revogada**.
2. Observe.

**Passou se:** a ordem sai **na hora**, sem esperar a janela de 1 hora. A chave da guarda e **por
versao**, nao global.

**Reprovou se:** a parte 1 mostrar a ordem saindo a cada boot (laco), **ou** a parte 2 mostrar a
maquina presa por 1 hora numa segunda versao ruim. **As duas metades sao reprovacao** — e por isso
elas viraram um cenario so.

---

## C11 — Recall de frota: a maquina volta sozinha

> **Origem:** cenario da @Natasha, ampliado com R10.11.1, R10.11.3 e R10.11.4 do @Bucky.

E o R-03: mandar de volta quem **ja tem** a versao ruim instalada. Diferente do botao de pausa,
que so impede novas entregas.

**Pre-requisito:** P2 + P3 + P4. **O lado Agent ficou pronto hoje** (commit `39d20e3`), entao o
que travava este cenario mudou: nao falta mais codigo, falta **numero de versao novo e um
instalador bom instalado nesta maquina**. A 1.5.12 que esta aqui nao entende recall.

**Limitacoes que precisam estar escritas, porque mudam o que o teste significa:**

1. **O recall so alcanca maquina cujo Agent ja entende o comando.** A frota que roda 1.5.12 hoje
   vai simplesmente **ignorar** — sem erro e sem aviso. O recall protege as versoes lancadas
   **depois** de ele existir; **ele nunca desfaz a versao que estiver rodando no dia em que for
   publicado.** Quem contar com ele para a 1.5.12 vai se frustrar: para essa, o caminho continua
   sendo maquina a maquina.
2. **Maquina em modo SOS nunca recebe recall.** Ela nao chega a perguntar ao backend, entao nao
   ouve a ordem. E o comportamento certo — SOS significa "nao encoste" — mas e um limite real do
   alcance do recall na frota. Detalhe e teste no C20.
3. **Volta uma versao, so.** A maquina guarda exatamente uma copia da versao anterior. Se a
   anterior tambem estiver ruim, o recall nao resolve. Isso e limitacao de desenho, nao lacuna de
   teste, e tem de estar visivel para quem dispara.
4. **Pode demorar ate 6 horas — e ate 6 horas A MAIS numa maquina que acabou de falhar uma
   atualizacao.** A maquina pergunta ao backend a cada 6 horas. Mas se existir um
   `update-failed.flag` recente, o Service **dorme o resto da janela antes de fazer a primeira
   pergunta** — ele nem chega a falar com o backend. Na pior combinacao (update falhou ha 5
   minutos, operador revoga a versao agora), a ordem pode demorar **quase o dobro do pior caso
   previsto**. E justamente a maquina que acabou de falhar um update que e a candidata ao recall.
   **Achado aberto, com tarefa propria** — nao e defeito deste teste, mas quem cronometrar o
   cenario precisa saber, senao registra reprovacao onde nao ha.

**Passos:**

1. Com a maquina rodando uma versao nova que entenda recall, peca ao @Vision para disparar:
   `POST /api/admin/fleet/revogar-versao`, com a versao instalada e um motivo escrito (o motivo e
   obrigatorio: minimo 10 caracteres).
2. Espere o proximo ciclo de checagem da maquina (ate 6 horas — ou reinicie o Service para forcar
   a checagem imediata).
3. **Confira primeiro o `watchdog-state.json`:** os campos `recallPendenteVersao`,
   `recallPendenteMotivo` e `recallPendenteEmUtc` tem de estar preenchidos. Se nao estiverem, o
   problema esta no Service e nao adianta olhar o Watchdog.
4. Espere ate **60 segundos** e leia o log do Service e o do Watchdog.

**O que observar:**

- O motivo escrito pelo operador aparece no log da maquina, em dois momentos: quando a resposta
  chega e quando o Watchdog age. Se o backend nao mandar motivo, o log tem de dizer isso com
  todas as letras — silencio nao serve.
- **Se nao houver copia da versao anterior**, a maquina **nao** pode entrar em modo SOS: ela loga,
  registra que pulou, e **continua capturando**. Isso e diferente do rollback automatico, e e de
  proposito. (Prova dedicada no C16.)
- **Depois de voltar, a maquina nao pode reinstalar a versao revogada** no ciclo seguinte. Duas
  travas independentes seguram isso: o `pausada=true` que o backend grava junto, e o bloqueio de
  24h do lado da maquina. (Prova dedicada no C13, pelo caminho automatico.)

**Passou se:** a maquina volta para a versao anterior, ou recusa com motivo registrado — e em
nenhum dos dois casos entra em SOS.

**Reprovou se:** a maquina entrar em SOS por causa de um recall, ou repetir a ordem de recall em
laco a cada reinicio.

**Se reprovar:** desligue o recall no backend (`revogada = false`) antes de mexer na maquina, ou o
laco continua.

---

## C19 — Recall que falha nao liga SOS

> **Origem:** R10.11.11 do documento do @Bucky.

E o par assincrono do C16, e o unico jeito de exercita-lo e com o script rodando de verdade. No
C16 a maquina recusa **antes** de tentar; aqui ela **tenta, e falha no meio**. Sao dois caminhos
de codigo diferentes, e so este prova o segundo.

**Pre-requisito:** P2 + P3 + P4, com `bin.previous` presente e valido.

**Risco:** medio-alto. A restauracao vai falhar de proposito. O criterio do teste e justamente que
a maquina **nao** saia do ar por causa disso — mas se ela sair, e uma reprovacao com a maquina
parada. Tenha as duas redes de seguranca a mao.

**Passos:**

1. Force a restauracao a falhar — por exemplo, deixando um arquivo do `bin.previous` travado por
   outro processo.
2. Dispare o recall e deixe acontecer.
3. Reinicie a maquina.
4. No boot seguinte, leia o log e o `watchdog-state.json`.

**Passou se, e sao tres coisas juntas:**

1. Log com `Recall NAO recuperou a maquina`.
2. Audit `UPDATE_RECALL_FAILED`.
3. `sosMode` continua **`false`**.

**Reprovou se:** `sosMode` virar `true`. **A diferenca entre este cenario e o C12 e exatamente
esta:** falha de rollback automatico **liga** SOS (e certo); falha de recall **nao liga** (e
tambem certo). Se os dois se comportarem igual, a origem nao esta sendo distinguida na volta — e
esse e o defeito que este cenario procura.

**Ao terminar:** solte o arquivo travado, confira `sosMode` e o estado dos dois servicos antes de
seguir.

---

# BLOCO 4 — POR ULTIMO: a volta automatica de versao

> **Este e o teste que pode deixar a maquina sem captura. Leia o aviso do topo antes.**

## Por que este bloco existe

E a peca que a regra 5.2 do `REGRAS-RELEASE` exige, e ela **nunca foi executada em maquina
nenhuma**. Ate hoje de manha o mecanismo nem funcionava: quem le a copia de seguranca procurava
uma pasta um nivel acima de onde quem escreve a colocou. O @Bucky corrigiu, e a correcao esta
coberta por teste — mas o rollback de verdade, com servico parando, arquivos trocando e servico
voltando, **nao foi exercitado nenhuma vez**. Foi exatamente essa distancia que escondeu o defeito
original.

## Pre-requisitos, os quatro

| # | O que | Sem isso |
|---|---|---|
| 1 | **Numero de versao novo** (P3), coordenado com o @Vision | um pacote da arvore de hoje nasce com `1.5.12`, o mesmo numero ja instalado, e o fluxo de update nao dispara |
| 2 | **Uma versao boa com o codigo de hoje instalada primeiro** (P4) | a maquina roda a 1.5.12 de 21/08, que **nao tem** as correcoes. Testar sem isso testaria o rollback quebrado |
| 3 | **Uma versao propositalmente quebrada** (P5), com numero maior que a boa | nao ha o que reverter |
| 4 | **Duas horas livres e as duas redes de seguranca a mao** | o pior caso e reinstalar do zero |

**Ordem obrigatoria dos passos preparatorios**, e ela nao pode ser invertida:

1. @Vision define o numero novo (chamemos de **N**, provavelmente `1.5.13`).
2. @Bucky gera o pacote **N** com o codigo de hoje, e voce instala por cima, preservando a
   vinculacao.
3. Confirme que **N** subiu e esta capturando por pelo menos 15 minutos.
4. So entao @Bucky gera o pacote **N+1 quebrado** (o servico principal nao sobe).
5. Publique **N+1** apenas para esta maquina — pelo servidor local de teste, no modelo do roteiro
   da 1.5.7 (so o endereco de administracao aponta para o servidor de teste; os eventos continuam
   indo para staging).

**Nao publique a versao quebrada no backend de verdade.** Se publicar, qualquer outra maquina que
perguntar vai baixa-la.

---

## C12 — A maquina volta sozinha para a versao anterior

> **Origem:** cenario da @Natasha, ampliado com o Teste 10.5 do @Bucky (itens R10.5.5 e R10.5.6).

**Pre-requisito:** os quatro acima. Versao **N** instalada, saudavel, e **N+1 quebrada**
publicada so para esta maquina.

**Passos:**

1. Guarde a foto de antes:

```powershell
$ev = "C:\Temp\ManagerAgent-Tests\2026-08-24"
Copy-Item 'C:\ProgramData\ManagerAgent\watchdog-state.json' "$ev\C12-state-antes.json"
(Get-Item 'C:\Program Files\ManagerAgent\ManagerAgent.Service.exe').VersionInfo.ProductVersion
(Get-Item 'C:\Program Files\bin.previous\ManagerAgent.Service.exe').VersionInfo.ProductVersion
```

   A copia de seguranca tem de conter **N** neste momento. Se ainda contiver a 1.5.11, o passo 2
   da preparacao nao aconteceu — pare aqui.

2. Confirme que o modo SOS esta **desligado**. Se estiver ligado, a maquina nao vai nem procurar
   update.
3. Dispare a checagem reiniciando o Service:

```powershell
Restart-Service ManagerAgent
```

4. **Acompanhe ao vivo**, em outra janela:

```powershell
Get-Content "C:\ProgramData\ManagerAgent\logs\service-$(Get-Date -Format yyyyMMdd).log" -Wait -Tail 20
```

5. Espere. A sequencia esperada leva **ate 15 minutos**: baixa N+1, aplica, o servico tenta subir
   e falha **tres vezes**, o Watchdog dispara a volta, o script para tudo, troca os arquivos e
   religa.

**O que observar, em ordem:**

| Momento | Onde | O que tem de aparecer |
|---|---|---|
| Update aplicado | `service-*.log` | a versao N+1 sendo instalada |
| Tres falhas de partida | `logs\` | o contador de falhas subindo ate 3 |
| Volta disparada | `rollback-script.log` | o script rodando de verdade — este arquivo e o que sobrevive ao reinicio |
| Desfecho, **durante** | `C:\ProgramData\ManagerAgent\rollback-result.json` | aparece **antes** de o Watchdog voltar |
| Desfecho, **depois** | o mesmo arquivo | **nao existe mais** — o Watchdog o consumiu ao concluir |
| Fim | servicos | os dois `Running`, versao de volta em **N** |

> **Nota de absorcao, para nao virar reprovacao falsa.** O documento do @Bucky pedia (R10.5.5) que
> o `rollback-result.json` **nao existisse mais** ao fim; este roteiro pedia que ele **aparecesse**
> antes de o Watchdog voltar. **Os dois estao certos, em momentos diferentes** — e um arquivo de
> recado, escrito pelo script e apagado por quem o le. Lidos como uma lista unica de conferencia
> final, eles se contradizem e alguem marcaria reprovacao. Por isso viraram duas linhas separadas,
> com o momento escrito em cada uma.

6. Ao fim:

```powershell
(Get-Item 'C:\Program Files\ManagerAgent\ManagerAgent.Service.exe').VersionInfo.ProductVersion
Get-Service ManagerAgent, ManagerAgentWatchdog | Select-Object Name, Status
Get-ChildItem 'C:\Program Files' -Filter 'bin.failed-*' -Directory | Select-Object Name
Get-Content 'C:\ProgramData\ManagerAgent\watchdog-state.json'
Test-Path 'C:\ProgramData\ManagerAgent\rollback-result.json'   # esperado: False
```

**Passou se, e sao sete coisas juntas:**

1. A versao instalada voltou a ser **N**.
2. Os dois servicos estao `Running`.
3. A captura voltou (aparecem eventos novos no log depois da volta).
4. Existe uma pasta `bin.failed-...` **ao lado** da instalacao, em `C:\Program Files` — nao dentro
   dela.
5. O modo SOS continua **desligado**, e `startupFailuresSinceLastUpdate` voltou a **0**.
6. O `rollback-result.json` **nao existe mais** (foi consumido).
7. **O audit `UPDATE_ROLLBACK_TRIGGERED` chegou ao backend** — e com esse nome, nunca com nome de
   recall. *(Este item veio do R10.5.6 e depende do P2: sem staging no ar, marque como "nao
   conferido" em vez de reprovado.)*

**Reprovou se:** o modo SOS ligou, ou a versao instalada continua sendo a N+1, ou o servico nao
volta a rodar. **Qualquer um dos tres e reprovacao — e nos tres a maquina esta sem capturar.**

**Se reprovar — o caminho de volta, nesta ordem:**

1. Leia o motivo antes de consertar, porque depois de reinstalar a evidencia some:

```powershell
Get-Content 'C:\ProgramData\ManagerAgent\logs\rollback-script.log' -Tail 40
Get-Content 'C:\ProgramData\ManagerAgent\rollback-result.json'
Get-Content 'C:\ProgramData\ManagerAgent\watchdog-state.json'
Copy-Item 'C:\ProgramData\ManagerAgent\logs' "$ev\C12-logs-falha" -Recurse
```

2. Reinstale a rede de seguranca:

```powershell
Start-Process 'C:\Users\NoisyTech\Documents\Manager\manager-srv-agent\instalador\ManagerAgent-Setup-v1.5.12.exe' -Verb RunAs
```

3. Se a 1.5.12 nao subir, use a `ManagerAgent-Setup-v1.5.11.exe`.
4. **Depois de reinstalar, confira o modo SOS.** Ele nao sai sozinho. Se `sosMode` estiver `true`
   no `watchdog-state.json`, troque para `false` e reinicie o Service — senao a maquina fica
   instalada e sem procurar update nenhum.

**Leitura rapida do que deu errado:**

| Sintoma no log | O que significa |
|---|---|
| "backup nao encontrado" | o endereco da copia de seguranca voltou a divergir — e o defeito original |
| erro de arquivo em uso | algo travou os binarios; o script deveria ter parado o Watchdog e os workers antes (ver C22) |
| o script nunca rodou | o disparo falhou; depois de 15 minutos sem resposta isso vira erro e liga o SOS, de proposito |
| SOS ligado sem registro de qual versao quebrou | e o buraco conhecido do caminho "sem backup": a captura da versao nao chega a ser gravada. Ja esta mapeado, ainda nao corrigido |

---

## C21 — A auto-recuperacao do Windows voltou no lugar

> **Origem:** Teste 10.6 do @Bucky (o mesmo que o E2E E6 cobriria).

**Rode logo depois do C12, antes de qualquer outra coisa.** E so leitura, leva 30 segundos, e
mede um estrago que nao aparece em lugar nenhum ate a proxima vez que o servico cair.

**Por que importa, nas palavras do proprio @Bucky:** o script de volta **desliga a
auto-recuperacao dos dois servicos** para conseguir trocar os arquivos. Se ele falhar em
restaurar, a maquina fica **sem auto-recuperacao** — pior do que o defeito que o rollback
conserta. Ha um `finally` no script que cobre, e ha teste unitario da ordem; **falta a prova em
maquina, e e esta.**

**Pre-requisito:** C12 executado (aprovado ou reprovado — vale nos dois casos).

**Passos:**

```powershell
sc.exe qfailure ManagerAgent
sc.exe qfailure ManagerAgentWatchdog
```

**Passou se:** os **dois** servicos voltaram a
`restart 5000 / restart 10000 / restart 30000`, com reset 86400 — exatamente o que o C1 mediu
antes de tudo comecar.

**Reprovou se:** qualquer um dos dois estiver sem acoes de recuperacao, ou com valores diferentes
dos do C1.

**Se reprovar:** e achado serio e vai para o @Bucky no mesmo dia — a maquina esta rodando, mas na
proxima queda ninguem a levanta. Anote se o C12 passou ou reprovou junto: se o C12 falhou no meio,
o `finally` pode nao ter chegado a rodar, e isso muda o diagnostico.

---

## C13 — Nao reinstala a versao quebrada no ciclo seguinte

> **Origem:** cenario da @Natasha, ampliado com o Teste 10.7 do @Bucky (itens R10.7.2 e R10.7.4).

Era o buraco do R-01: a maquina voltava, e no ciclo seguinte instalava **de novo** a mesma versao
que a tinha quebrado. Laco infinito.

**Pre-requisito:** C12 aprovado, e o servidor de teste **continuando a oferecer a N+1 quebrada**.
Se voce desligar o servidor de teste depois do C12, este cenario nao tem o que provar. Isso e de
proposito: e o cenario de madrugada em que ninguem apertou o botao de pausa.

**Passos:**

1. Deixe a maquina rodando na versao **N**, restaurada pelo C12.
2. Force uma nova checagem:

```powershell
Restart-Service ManagerAgent
```

3. Espere **5 minutos** e leia:

```powershell
Select-String -Path "C:\ProgramData\ManagerAgent\logs\service-$(Get-Date -Format yyyyMMdd).log" `
  -Pattern 'update available|barrada|sticky|no update available|starting update|download'
```

4. Repita o passo 2 mais duas vezes, com intervalo. E um teste de laco: uma vez so nao prova.

> **Alternativa ao `Restart-Service`, se voce quiser ver o ciclo natural e nao o de boot.** Da para
> encurtar a janela de 6h com `updateCheckIntervalSecondsOverride` no `config.json`. Os dois
> caminhos servem, e **exercitam gatilhos diferentes**: o reinicio testa a checagem que o Service
> faz ao subir; o override testa o laco periodico. Se der tempo, faca os dois — foi o gatilho de
> boot que o C17 mostrou ser o perigoso. **Devolva o valor original ao terminar**, senao a maquina
> fica perguntando ao backend fora da cadencia de producao e as medicoes seguintes nao valem.

**O que observar:**

- Tem de aparecer um aviso dizendo **qual versao foi barrada e ate quando** — o bloqueio dura 24
  horas.
- **Nao pode** aparecer nenhuma linha de inicio de update para a N+1.
- **Nenhum download pode comecar.** Este item veio do R10.7.2 e vale por si: o pacote tem ~310MB,
  e um download iniciado ja e reprovacao mesmo que a instalacao seja barrada depois.
- **O Service tem de seguir `Running` o tempo todo**, nas tres checagens (era o R10.7.4).

**Passou se:** nas tres checagens a N+1 foi barrada, com aviso no log, nenhum download comecou, e
a maquina continuou na N capturando, com o Service sempre no ar.

**Reprovou se:** a maquina baixar a N+1 de novo — em qualquer uma das tres.

**Se reprovar:** desligue o servidor de teste **imediatamente**, senao a maquina entra em laco. E
o R-01 nao fechado, e e bloqueador de producao.

**Nota que evita leitura errada do resultado:** a protecao do R-01 depende de a maquina ter
gravado qual versao quebrou. Numa maquina que nunca reverteu antes — que e o caso desta, e de toda
a frota — esse campo nascia vazio, e a protecao nao dispararia. Quem fecha isso e o R-02, e e o
C14 que confere. **Se o C13 reprovar, leia o C14 antes de concluir qualquer coisa:** pode ser que
a protecao esteja certa e o que faltou foi o registro.

> **Uma afirmacao do documento do @Bucky que NAO foi absorvida.** A caixa dele dizia: *"sem o R-01
> a maquina baixa 310MB, reinstala a versao quebrada, cai de novo — e ai a sticky **impede** o
> segundo rollback; fica quebrada ate a sticky vencer."* A primeira metade e o risco real e ficou.
> **A segunda metade nao se sustenta como escrita:** o bloqueio de 24h impede **instalar** a versao
> ruim, e nao impede a maquina de **voltar** dela. Absorver isso do jeito que estava faria o
> testador esperar um comportamento que o desenho nao promete, e registrar reprovacao se ele nao
> acontecesse. Fica registrado como **pergunta para o @Bucky**, nao como criterio.

---

## C14 — O registro guarda qual versao quebrou

Era o R-02. Antes dele, a auditoria reportava a versao quebrada como "desconhecida" e a pasta da
versao ruim saia com o nome `bin.failed-unknown-...`.

**Pre-requisito:** C12 executado (aprovado ou reprovado — este cenario e so leitura, e vale nos
dois casos).

**Passos:**

1. Leia o registro do Watchdog:

```powershell
Get-Content 'C:\ProgramData\ManagerAgent\watchdog-state.json'
```

2. Liste a pasta da versao quebrada:

```powershell
Get-ChildItem 'C:\Program Files' -Filter 'bin.failed-*' -Directory | Select-Object Name
```

**O que observar:**

| O que | Esperado | Antes do R-02 |
|---|---|---|
| Campo da ultima versao quebrada | traz **N+1** | vinha vazio |
| Nome da pasta preservada | `bin.failed-<N+1>-<data>` | `bin.failed-unknown-<data>` |
| Bloqueio ate quando | 24 horas a frente | — |

**Detalhe que nao e defeito:** ao abrir o arquivo no bloco de notas, o sinal `+` do numero da
versao aparece escapado, com uma sequencia estranha no lugar. E so a forma de gravar; o programa
le certo. Nao registre como achado.

**Passou se:** a versao quebrada esta gravada e o nome da pasta traz o numero da versao, nao a
palavra `unknown`.

**Reprovou se:** o campo estiver vazio, ou a pasta sair como `unknown`.

**Se reprovar:** anote junto qual foi o desfecho do C12. Se o C12 tiver terminado no caminho "sem
backup", este resultado ja e esperado e **nao e defeito novo** — e a lacuna conhecida, registrada
no handoff do R-02, ainda em aberto com o @Tony.

---

## C22 — A volta acontece com dois usuarios logados

> **Origem:** Teste 10.9 do @Bucky. **Os criterios foram corrigidos na absorcao — ver a caixa.**

**Por que existe:** o script de volta precisa **matar os `SessionWorker`** para liberar os
arquivos travados. O A-40 registra que esquecer isso ja derrubou o rollback em **11 tentativas
seguidas**. Este e o unico cenario que exercita esse passo com duas sessoes vivas.

**Pre-requisito:** C12 aprovado (repita a preparacao do Bloco 4 para ter de novo uma N+1
quebrada), P7 — duas contas logadas, cada uma com o seu worker.

**Risco:** o mesmo do C12, alto. Duas redes de seguranca a mao.

**Passos:**

1. Deixe as duas contas logadas (a sua ativa, a `Marcos` desconectada), com atividade real em cada
   uma por 2 minutos.
2. Confirme os dois workers e o tamanho dos buffers por sessao:

```powershell
Get-Process ManagerAgent.SessionWorker | Select-Object Id, SessionId
Get-ChildItem 'C:\ProgramData\ManagerAgent\data' | Select-Object Name, Length
```

3. Repita o C12 (dispare a checagem, deixe a N+1 quebrada ser instalada e a volta acontecer).
4. Depois da volta, confira os workers, os buffers e o log das duas sessoes.

**Passou se, e sao quatro coisas:**

1. A volta **conclui** — nao falha por arquivo em uso.
2. **Um worker por sessao** volta a subir depois.
3. **Nenhum evento perdido:** o buffer de cada sessao sobe depois do retorno, cada um com o seu
   dono.
4. **Nenhum LOGOUT** gravado para as duas pessoas — **ninguem saiu.** Uma volta de versao nao e um
   fim de expediente, e nao pode aparecer no relatorio como se fosse.

> **CORRECAO NA ABSORCAO.** O R10.9.4 original pedia so *"nenhum LOGOUT gravado"*. Isso diz o que
> **nao** pode aparecer e nao diz o que **deve**, e um teste que so proibe passa por omissao.
> O script **mata** os workers: uma `SESSAO_INTERROMPIDA` nas duas sessoes e **legitima e
> esperada** aqui, pelo mesmo motivo que ela e legitima no C2 (reboot nao tem despedida limpa).
> **Criterio corrigido:** `SESSAO_INTERROMPIDA` pode e deve aparecer; **LOGOUT nao pode**; e
> nenhuma sessao pode ficar aberta com evento posterior ao encerramento. Sem essa terceira linha o
> cenario deixaria passar exatamente o defeito do A-65 que o C5 caca.

**Reprovou se:** a volta falhar por arquivo em uso (o A-40 de volta), aparecer LOGOUT para
qualquer das duas contas, ou um worker nao voltar.

---

## C15 — Recall recusado pela guarda A (backup na mesma versao)

> **Origem:** R10.11.7 do @Bucky. **A ordem e o preparo foram corrigidos na absorcao — leia a
> caixa antes de tocar em qualquer coisa.**

**Por que existe:** sem esta guarda, uma maquina cujo `bin.previous` tem a **mesma** versao da
instalada **reverteria a cada ciclo, para sempre** — copiando ~160MB e deixando uma pasta
`bin.failed-*` de ~160MB a cada volta. E um cenario de disco cheio, nao so de teste falhando.

**Pre-requisito:** P2 + P3 + P4, **e C12, C13 e C14 ja concluidos.**

> **CORRECAO NA ABSORCAO — este cenario desarma o Bloco 4 se rodar na hora errada.**
> O preparo original era: *"basta copiar a instalacao por cima do backup"*. Isso **destroi o
> `bin.previous`**, que e a unica copia de seguranca da maquina e e exatamente para onde o C12
> retorna. Rodar este cenario antes do Bloco 4 apagaria a rede de seguranca do teste que bloqueia
> producao — em silencio, e sem ninguem perceber ate o C12 falhar por "backup nao encontrado" e o
> diagnostico apontar para o defeito errado.
> **Duas correcoes:** (1) este cenario vai para **depois** de C12, C13 e C14; (2) o preparo passa a
> **guardar e devolver** o `bin.previous`, em vez de sobrescreve-lo.

**Preparo — guarde a copia de seguranca de verdade primeiro:**

```powershell
$ev = "C:\Temp\ManagerAgent-Tests\2026-08-24"
Copy-Item 'C:\Program Files\bin.previous' 'C:\Program Files\bin.previous.guardado' -Recurse
# so agora sobrescreva, para forcar o caso de teste:
Copy-Item 'C:\Program Files\ManagerAgent\*' 'C:\Program Files\bin.previous\' -Recurse -Force
```

**Passos:**

1. Confirme que `bin.previous` e a instalacao estao **na mesma versao**:

```powershell
(Get-Item 'C:\Program Files\ManagerAgent\ManagerAgent.Service.exe').VersionInfo.ProductVersion
(Get-Item 'C:\Program Files\bin.previous\ManagerAgent.Service.exe').VersionInfo.ProductVersion
```

2. Peca ao @Vision para revogar essa versao.
3. Espere o ciclo (ou reinicie o Service para forcar).
4. Leia o log e liste `C:\Program Files`.

**Passou se, e sao cinco coisas:**

1. **Nenhum servico para.**
2. **Nenhum `bin.failed-*` novo** aparece.
3. Log com `Recall RECUSADO pela guarda A`.
4. Audit `UPDATE_RECALL_SKIPPED` com `motivo=backup_na_mesma_versao`.
5. **A maquina continua capturando.**

**Reprovou se:** a maquina disparar a volta mesmo assim — e ai desligue o recall no backend
(`revogada = false`) **imediatamente**, antes que o ciclo seguinte repita.

**OBRIGATORIO ao terminar — devolva a copia de seguranca de verdade:**

```powershell
Remove-Item 'C:\Program Files\bin.previous' -Recurse -Force
Move-Item 'C:\Program Files\bin.previous.guardado' 'C:\Program Files\bin.previous'
(Get-Item 'C:\Program Files\bin.previous\ManagerAgent.Service.exe').VersionInfo.ProductVersion
```

A ultima linha tem de mostrar a versao **anterior**, nao a instalada. Se mostrar a instalada, a
maquina esta sem rede de seguranca.

---

# BLOCO 5 — so com autorizacao: derruba a captura DE PROPOSITO

> **Estes dois cenarios terminam com a maquina parada. Isso e o resultado esperado, nao uma
> falha.** Nos dois, o unico caminho de volta e alguem reinstalar a mao.

**Pre-requisito de todos: P9 — autorizacao escrita do Marcos, ou uma VM descartavel.**

> **CORRECAO NA ABSORCAO.** No documento do @Bucky estes eram os testes **10.8** e **10.10**,
> soltos no meio da lista, **sem aviso de risco e sem caminho de volta** — o 10.8 tem
> `sosMode = true` como criterio de aprovacao, e o 10.10 manda *"desligar na tomada"*. Numa VM
> descartavel isso e razoavel. Nesta maquina, que e a maquina de trabalho do Marcos, cada um deles
> encerra o expediente. Foram separados num bloco proprio, com autorizacao explicita e o
> procedimento de saida escrito. **Preferencia da @Natasha: rodar os dois em VM, nunca aqui.**

---

## C23 — Volta sem copia de seguranca: entra em SOS, e o esperado

> **Origem:** Teste 10.8 do @Bucky.

**O caso real:** cliente novo, que instalou e nunca atualizou. Nao tem `bin.previous`, logo nao
tem para onde voltar. **O produto nao pode travar nem lancar excecao** — ele tem de desistir de
forma limpa, ligar o SOS e registrar o motivo.

**Nao confunda com o C16.** Sao o mesmo ponto de partida (sem backup) e o **oposto** no desfecho,
e essa e a graca dos dois:

| | Caminho | Sem backup, o esperado e |
|---|---|---|
| C23 | rollback **automatico** (a versao nova nao subiu) | **liga SOS** — a maquina esta quebrada de verdade e precisa de gente |
| C16 | **recall** (ordem do backend) | **NAO liga SOS** — a maquina esta saudavel, so nao pode obedecer |

Se os dois se comportarem igual, a origem nao esta sendo distinguida — e esse e o defeito que a
dupla procura.

**Pre-requisito:** P9 + P5, instalacao limpa, `C:\Program Files\bin.previous` **removido**.

**Passou se, e sao cinco coisas:**

1. Log registra `BackupNotFound`.
2. `sosMode = true` e `sosSince` preenchido.
3. **Nenhuma excecao nao tratada** no log.
4. O **Watchdog continua de pe** (o processo nao morre).
5. Os ciclos de update seguintes sao **pulados, com o motivo no log** — nao em silencio.

**Reprovou se:** aparecer excecao nao tratada, ou o Watchdog cair, ou os ciclos seguintes serem
pulados sem dizer por que.

**Procedimento de saida — OBRIGATORIO, e nao e opcional:**

1. Guarde a evidencia **antes** de consertar, porque reinstalar apaga tudo:

```powershell
Copy-Item 'C:\ProgramData\ManagerAgent\logs' "$ev\C23-logs" -Recurse
Copy-Item 'C:\ProgramData\ManagerAgent\watchdog-state.json' "$ev\C23-state.json"
```

2. Reinstale com a rede de seguranca (`ManagerAgent-Setup-v1.5.12.exe`).
3. **Confira o `sosMode` depois de reinstalar.** Ele **nao sai sozinho**. Se continuar `true`,
   troque para `false` e reinicie os dois servicos.
4. So encerre o dia depois de ver evento novo subindo no log.

---

## C24 — Maquina desligada no meio da volta

> **Origem:** Teste 10.10 do @Bucky.

**O pior momento possivel.** A maquina perde energia exatamente enquanto os arquivos estao sendo
trocados. O boot seguinte tem de **concluir** o que ficou pela metade, ou desistir com motivo — o
que nao pode e ficar em limbo, contando falha em silencio.

**Pre-requisito:** P9 + P5. **E, para o item 4, tambem o P6** (acesso ao banco) — ver a nota.

**Procedimento:** desligar na tomada assim que o `rollback-script.log` registrar a parada dos
servicos.

**Passou se, e sao quatro coisas:**

1. No boot, o Watchdog **registra rollback pendente e conclui**, ou aguarda dentro da folga de 15
   minutos.
2. Passados 15 minutos sem resultado, **entra em SOS** — e nao fica contando falha em silencio.
3. O `watchdog-state.json` esta **legivel** (escrita atomica) — nunca truncado, nem com arquivo
   temporario orfao no lugar.
4. **Nenhum evento do colaborador perdido.**

> **CORRECAO NA ABSORCAO — o item 4 nao e conferivel no log.** No documento do @Bucky ele estava
> na mesma lista dos outros tres, como se fosse so mais um `Get-Content`. Nao e: provar que nenhum
> evento se perdeu exige comparar o que a maquina bufferizou com o que chegou do outro lado, e isso
> e **conferencia no banco (P6)** — o mesmo pre-requisito do C10, que segue pendente por causa da
> senha. **Enquanto o P6 nao sair, o item 4 se marca como "nao conferido", nunca como aprovado.**
> Marcar aprovado sem o banco e afirmar exatamente o que nao foi olhado.

**Reprovou se:** o `watchdog-state.json` sair truncado ou ilegivel (a escrita atomica nao pegou),
ou a maquina ficar contando falha sem entrar em SOS nem concluir.

**Procedimento de saida:** o mesmo do C23 — guardar evidencia, reinstalar, conferir `sosMode`,
so encerrar depois de ver evento novo subindo.

---

# Depois de tudo — como a maquina tem de ficar

```powershell
Get-Service ManagerAgent, ManagerAgentWatchdog | Select-Object Name, Status, StartType
(Get-Item 'C:\Program Files\ManagerAgent\ManagerAgent.Service.exe').VersionInfo.ProductVersion
Get-Content 'C:\ProgramData\ManagerAgent\watchdog-state.json'
Get-Process ManagerAgent.SessionWorker | Select-Object Id, SessionId
Test-Path 'C:\Program Files\bin.previous'
sc.exe qfailure ManagerAgent
sc.exe qfailure ManagerAgentWatchdog
```

**Estado aceitavel para encerrar o dia:**

- Dois servicos `Running`.
- Um `SessionWorker` por usuario logado — nunca dois na mesma sessao.
- Modo SOS **desligado**.
- `bin.previous` **existe**, e com a versao **anterior** dentro (nao a instalada — se estiver com a
  instalada, o C15 nao foi desfeito).
- Auto-recuperacao do SCM nos dois servicos, igual ao que o C1 mediu.
- Fila de envio limpa (ciclos fechando com zero lotes com falha).

Se o modo SOS ficou ligado, **a maquina nao esta capturando e nao vai voltar sozinha**. Troque
para `false` e reinicie o Service, ou reinstale.

**Limpeza depois dos Blocos 4 e 5:** as pastas `bin.failed-*` em `C:\Program Files` ocupam cerca
de 160MB cada. Podem ser apagadas depois que o @Bucky ler o que precisava delas — nao antes.

---

# De-para com o documento do @Bucky

`agent-desktop/TESTES-ROLLBACK-E-RECALL.md` **foi incorporado neste roteiro e apagado em
24/08/2026**, por decisao do @Tony: roteiro de teste de maquina e um so. Nada dele se perdeu.
Esta tabela existe para quem procurar por numero (`Teste 10.11`, `R10.11.7`) achar onde a coisa
foi parar.

| No documento do @Bucky | Onde esta agora | Absorcao |
|---|---|---|
| Teste **10.5** — Rollback automatico completo | **C12** | fundido. Ficou a versao daqui (passo a passo, redes de seguranca, caminho de volta, acompanhamento ao vivo); entraram dele o R10.5.5 (`rollback-result.json` consumido) e o R10.5.6 (audit chegou ao backend) |
| Teste **10.6** — Recovery do SCM restaurado | **C21** (+ foto no C1) | cenario novo. Nao existia aqui, e mede um estrago invisivel ate a proxima queda |
| Teste **10.7** — Nao reinstala a que quebrou | **C13** | fundido. Ficou a versao daqui (tres checagens, teste de laco, ponte com o C14); entraram dele o R10.7.2 (nenhum download) e o R10.7.4 (Service sempre `Running`). **Uma afirmacao dele nao foi absorvida** — ver a caixa no C13 |
| Teste **10.8** — Rollback sem backup | **C23** (Bloco 5) | cenario novo, **corrigido**: ganhou aviso de risco, autorizacao (P9) e procedimento de saida |
| Teste **10.9** — Rollback com dois usuarios | **C22** | cenario novo, **corrigido**: o criterio "nenhum LOGOUT" era frouxo. Ver a caixa no C22 |
| Teste **10.10** — Desligada no meio | **C24** (Bloco 5) | cenario novo, **corrigido**: risco, autorizacao, e o item 4 amarrado ao acesso ao banco |
| **10.11** cabecalho — como a ordem viaja | **Bloco 3**, antes do C9 | absorvido inteiro. Vira o passo 1 de todo cenario de recall |
| R10.11.1, .3, .4 — recall basico | **C11** | fundido no cenario da @Natasha |
| R10.11.2, .6, .16 — quem o recall nao alcanca | **C20** | juntados num cenario so, e movidos para o **inicio** do bloco: sao os mais baratos e evitam a leitura errada mais provavel |
| R10.11.5, .13, .14, .15 — audit e log | **C18** | juntados. **R10.11.13 corrigido** — o aviso do filtro por nivel venceu hoje. Ver a caixa no C18 |
| R10.11.7 — guarda A | **C15** | **corrigido**: o preparo destruia o `bin.previous`. Ver a caixa no C15 |
| R10.11.8 — guarda B | **C16** | absorvido, com o passo de guardar/devolver o backup |
| R10.11.9 e .10 — guarda C | **C17** | juntados: um sem o outro engana |
| R10.11.11 — recall que falha | **C19** | absorvido, com caminho de volta |
| R10.11.12 — o caminho automatico nao mudou | **C12**, criterio 7 | virou o item do audit: nome de rollback para rollback, nunca nome de recall |
| Secao 1 — estado da cobertura | *anexo A* | **numeros vencidos**, atualizados no anexo |
| Secao 2 — automatizados a escrever | *anexo A* | nao sao cenarios manuais; ficam registrados para nao sumirem |
| Secao 3 — E2E em VM | *anexo A* | idem. O E6 virou o C21 |
| Secao 5 — o que nao da para testar | *anexo B* | absorvido inteiro |
| Secao 6 — ordem sugerida | **substituida** | era uma ordem por camada (unitario, depois E2E, depois maquina). Aqui a ordem e por **risco a captura**, que e o criterio desta rodada. O "Mapa dos cenarios" e a ordem que vale |
| Secao 7 — correcao de rota no plano | *anexo C* | absorvido inteiro |

**Nove arquivos ainda apontam para o arquivo apagado.** Nenhum foi alterado: cinco sao registros
historicos (registro do dia nao se reescreve), tres sao comentarios dentro de codigo de teste — e
mexer em codigo nao e desta tarefa — e um e o handoff do @Bucky. Ficam listados para a varredura
de textos defasados:

| Arquivo | O que |
|---|---|
| `registro/2026-08-21-decisoes-do-dia.md` | registro historico |
| `registro/2026-08-24-android-recall-descartado.md` | registro historico |
| `registro/2026-08-24-tarefa-bucky-R01-sticky.md` | registro historico |
| `registro/2026-08-24-tarefa-bucky-R03-recall-agent.md` | registro historico |
| `registro/2026-08-24-tarefa-shuri-R03-recall.md` | registro historico |
| `manager-srv-agent/HANDOFF-R03-RECALL-AGENT.md` | secao 2 — diz que ampliou o Teste 10.11 naquele arquivo |
| `manager-srv-admin-node/.../agent-update.service.test.ts:503` | comentario apontando o doc. **@Shuri**, na proxima vez que tocar o arquivo |
| `manager-srv-admin-node/.../fleet-health.repository.test.ts:561` | idem |
| `manager-srv-admin-node/.../revogar-versao.body.dto.test.ts:3` | idem — cita "secao 2.3", que agora e o *anexo A* deste roteiro |

---

## Anexo A — o que nao e cenario manual, e por isso nao virou C

Estava no documento do @Bucky e nao cabe num roteiro de maquina. Fica aqui para nao se perder.

**Numeros de cobertura, atualizados (os do documento dele estao vencidos):**

| Camada | No documento dele | Hoje |
|---|---|---|
| Suite do Agent, total | 1092 | **1139** |
| — Watchdog | 69 | **98** |
| — Service | 639 | **657** |
| — Tray / SessionWorker | 14 / 370 | inalterados |
| Suite do backend (`admin-node`) | 1199 | **1205** |
| E2E executados | **0** | **0** — `e1-alt-rollback-crash.ps1` existe e nunca rodou |
| Testes de maquina executados | **0** | **0** — e o que este roteiro existe para mudar |

**Automatizados que continuam por escrever (@Bucky):**

| # | Cenario | Prioridade |
|---|---|---|
| A8 | `DispararScript` falha (WMI recusa) → `Error`, intencao limpa, SOS | media |
| A9 | O script escrito no disco tem o conteudo esperado **e nada mais** — sem caminho absoluto fixo | baixa |
| A10 | `bin.failed-*` com versao nula vira `bin.failed-unknown-*` | baixa |

Os A1–A7 (sticky) e A11–A16 (recall) do documento dele foram entregues junto com o R-01 e o R-03.
**Conferir um a um contra o que foi entregue e do @Tony, na revisao dos handoffs — nao e QA de
maquina**, e por isso nao esta neste roteiro.

**E2E em VM que continuam por escrever:** E2 (sem backup), E3 (a anterior tambem nao sobe), E4
(script nao roda), E5 (o E2E do R-01). O **E1 existe e nunca rodou** — po-lo no CI e o item 3b da
linha de corte. O **E6** (recovery do SCM) virou o **C21** deste roteiro e sai da lista de
pendentes de automacao so quando o E2E existir.

---

## Anexo B — o que NAO da para testar, e por que

| O que | Por que | Como conviver |
|---|---|---|
| Rollback em maquina real dentro do CI hospedado | exige servico Windows, elevacao e reinicio de servico | decidir com o @Vision: VM propria ou runner auto-hospedado. **Se nao couber, dizer isso — nao adaptar o teste ate caber** |
| Queda de energia exata no meio da escrita do estado | nao se encena de forma repetivel | coberto por unitario (temp orfao) + o C24 na mao |
| Recall alcancando a frota inteira | exige varias maquinas | canario primeiro, e acompanhar por `agentes.versao_agente` |
| Volta de mais de uma versao | `bin.previous` guarda **uma** so | limitacao de desenho, nao lacuna de teste. Tem de estar visivel para quem dispara o recall |

---

## Anexo C — correcao de rota no `PLANO-TESTES-REGRESSIVOS.md`

Duas coisas, as duas herdadas do documento do @Bucky:

1. **O Teste 10.1 daquele plano chama o Plan A de "Script PS1 via schtasks".** O Plan A e **WMI
   `Win32_Process.Create`** desde a v1.3.10 — o `schtasks` virou Plan B, justamente porque o Task
   Scheduler atrasava o disparo em ate ~6 minutos. Entra na varredura de textos defasados, que ja
   tem nove itens.
2. **Os testes 10.5 a 10.11 nunca chegaram naquele plano.** O documento do @Bucky dizia que eles
   "entram na secao 10 do `PLANO-TESTES-REGRESSIVOS.md`" — **nao entraram**: aquela secao vai ate o
   **10.4** e para. Eles so existiam no arquivo apagado. **Se a absorcao nao tivesse sido feita,
   os sete testes teriam sumido junto com ele.**

---

# Relatorio final

Para cada cenario, uma linha:

| # | Cenario | Resultado | Evidencia | Observacao |
|---|---|---|---|---|
| C1 | Estado inicial (+ foto do SCM) | | | |
| C2 | Reboot | | | |
| C3 | Logoff e login | | | |
| C4 | Parar o Service logado | | | |
| C5 | Desligar com conta desconectada | | | |
| C6 | Troca rapida de usuarios | | | |
| C7 | Servico derrubado, dois usuarios | | | |
| C8 | Desinstalar e instalar | | | |
| C9 | Lote com um item ruim | | | |
| C10 | Conferencia no banco | | | |
| C20 | Quem o recall nao alcanca | | | |
| C18 | Onde o recall aparece | | | |
| C16 | Recall sem backup nao liga SOS (guarda B) | | | |
| C17 | Recall nao se repete (guarda C) | | | |
| C11 | Recall de frota | | | |
| C19 | Recall que falha nao liga SOS | | | |
| C12 | Volta automatica de versao | | | |
| C21 | Auto-recuperacao do SCM restaurada | | | |
| C13 | Nao reinstala a versao quebrada | | | |
| C14 | Registro da versao quebrada | | | |
| C22 | Volta com dois usuarios logados | | | |
| C15 | Recall recusado pela guarda A | | | |
| C23 | Volta sem backup: SOS de proposito | | | |
| C24 | Desligada no meio da volta | | | |

**O que bloqueia producao, e so isto:**

- **C12 reprovado** — a frota nao tem caminho de volta. E a regra 5.2, e o item 3a da linha de
  corte.
- **C13 reprovado** — a maquina reinstala a versao quebrada em laco.
- **C9 reprovado** — perda de dado silenciosa. E o item 1 da linha de corte.

**Um quarto entrou na lista com a absorcao:**

- **C21 reprovado** — a maquina fica **sem auto-recuperacao** depois de uma volta de versao. Passa
  o criterio da linha de corte pelo mesmo motivo que o C12: o produto ficaria pior depois de se
  proteger do que estava antes, e o estrago so aparece na proxima queda, quando ninguem estiver
  olhando. E o proprio @Bucky quem escreveu, sobre o E6: *"nao e detalhe"*. Concordo, e trago para
  a lista.

Os demais entram como achado da rodada e passam pelo criterio da linha de corte antes de virar
bloqueador. Nao inclua achado novo na lista de bloqueadores sem passar pelo criterio — foi
exatamente para isso que a linha foi tracada.

---

## Posicao da @Natasha

**O bloqueio de producao continua de pe, e o push de hoje nao o afrouxa.** A peca que protege a
frota nunca foi executada uma unica vez em maquina, e a regra 5.2 exige esse teste passando. Nada
neste roteiro muda isso ate o C12, o C13 e o C21 estarem aprovados com evidencia.

**O codigo ter ido para `staging` nao destravou nenhum cenario, e isso precisa ser dito com
todas as letras.** Continuam 7 executaveis hoje, os mesmos de manha. O que travava o Bloco 3 nunca
foi o commit — era o deploy (P2) e o numero de versao (P3), e os dois seguem abertos. Quem ler
"foi tudo empurrado" e concluir que a bateria destravou vai marcar reuniao para o dia errado.

**Duas coisas ficaram melhores hoje, e sao reais:** o recall existe dos dois lados, entao o C11
deixou de esperar codigo e passou a esperar so numero de versao; e os avisos de recall subiram
para `WARN`, entao o painel voltou a servir de evidencia — o C18 confere isso, e a instrucao
antiga de "consultar sem filtro" foi removida antes que alguem a seguisse.

**Recomendo executar o Bloco 1 hoje mesmo.** Sao sete cenarios que nao dependem de ninguem, e
cinco deles estao pendentes desde 20 e 21 de agosto. Fechar essa divida hoje deixa o dia do Bloco
4 livre para o que importa. O C1 ganhou a foto da auto-recuperacao do SCM: e um comando, leva 30
segundos, e sem ele o C21 nao tem com o que comparar depois.

**Os Blocos 4 e 5 nao devem ser tentados em cima do estado atual da maquina.** Ela roda a 1.5.12
de 21/08, que nao tem nenhuma das correcoes de hoje. Testar assim mediria o rollback quebrado e
produziria uma reprovacao que nao ensina nada — com o custo de deixar a maquina sem captura para
descobrir o que ja sabemos.

**Sobre o Bloco 5, uma posicao explicita:** os dois cenarios de la sao bons e precisam existir,
mas **nao nesta maquina**. Os dois terminam com a maquina parada por desenho, e o unico caminho de
volta e reinstalar. No documento em que eu os encontrei eles estavam soltos no meio da lista, sem
aviso e sem saida — quem executasse em ordem descobriria isso no meio do 10.8. **Peco VM para os
dois.** Se o Marcos autorizar nesta maquina mesmo assim, que seja o ultimo ato do dia, com as duas
redes de seguranca conferidas antes.
