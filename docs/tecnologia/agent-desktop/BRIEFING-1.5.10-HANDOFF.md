# Briefing 1.5.10 — Agent Windows

Handoff para a sessao que vai executar os testes manuais e as correcoes que sairem deles.
Escrito para ser lido do zero, sem contexto anterior.

> **Data:** 2026-08-19 | **Autor:** @Tony | **Executor:** Marcos na `DESKTOP-VMSM6LE`

---

## 1. Onde estamos em tres linhas

- A **1.5.10 esta instalada** na maquina de teste e passou por bateria automatizada completa.
- Ela **nao foi publicada**: `versoes_agente` so tem 1.5.6, 1.5.5 e 1.5.4 para WINDOWS.
- **Nada foi commitado**: 118 arquivos pendentes em `manager-srv-agent`, 5 em `manager-srv-admin-node`.

---

## 2. Onde os arquivos ficam de verdade

A regra de sessao manda ler `.brain/...`. **Esse diretorio nao existe.** O conteudo vive em
`manager-brain/docs/`. Varios arquivos obrigatorios (`_shared/banco-dados.md`,
`_shared/seguranca.md`, `_shared/lgpd-operacional.md`, `tecnologia/services/*/README.md`)
**nao existem em repositorio nenhum**.

Repositorios presentes: `manager-srv-agent`, `manager-srv-agent-android`,
`manager-srv-admin-node`, `manager-srv-events-node`, `manager-brain`.
Nao existem aqui: `manager-srv-portal-node`, srv-portal Java, fed-portal.

Documentos desta rodada:
- `agent-desktop/ACHADOS-REGRESSIVO-2026-08-18.md` — **fonte da verdade**, A-01 a A-51
- `agent-desktop/ROTEIRO-VALIDACAO-1.5.7.md` — aceite do auto-update, com execucao registrada
- `registro/2026-08-19-handoff-shuri-backend-node.md` — o que e do backend Node

---

## 3. Estado da maquina

```
versao instalada : 1.5.10.0
servicos         : ManagerAgent + ManagerAgentWatchdog Running
worker           : 1 SessionWorker
modo SOS         : DESLIGADO
agente no banco  : id=51, DESKTOP-VMSM6LE
```

Em staging so existem 3 agentes com heartbeat recente: esta maquina (1.5.10), um Android
(1.0.26-staging, outro `sistema_operacional`) e uma linha antiga da propria maquina (A-49).
**Publicar a 1.5.10 em staging nao afeta ninguem** — nao ha maquina Windows em versao anterior.

---

## 4. O que foi corrigido nesta rodada

Tudo comecou com um incidente: a 1.5.6 foi publicada e **nenhum agent conseguiu aplica-la**.
Onze tentativas, 285MB cada, ~3GB de banda queimada em 12 minutos.

| # | Achado | O que era |
|---|---|---|
| A-39 | CRITICO | `WorkerWatchdog` relancava o worker no meio do update; o worker novo segurava o file lock do `.exe` e a copia falhava |
| A-40 | CRITICO | O script de update parava o Watchdog e o Service, mas **nunca matava o SessionWorker** antes de copiar. O rollback falhava pelo mesmo lock |
| A-41 | ALTA | O cooldown de 6h era codigo morto: `Program.Main` apagava a flag antes de o checker le-la. Corrigido para relogio de parede (a primeira correcao, por `Task.Delay`, deixaria o agent sem nunca mais atualizar apos reinicios) |
| A-42 | ALTA | `MaximoBackups` so valia no caminho da Tray; 11 tentativas deixaram 3,4GB em disco |
| A-46 | ALTA | O agent aplicava **qualquer** versao que o backend mandasse, inclusive a que ja rodava. Um backend com bug poria a frota em reinstalacao continua |
| A-48 | ALTA | Reinicio do worker durante ociosidade duplicava o evento (13-23ms de diferenca furavam a UNIQUE) e deixava linha aberta para sempre. **Corrompia os pilares Atividade e Saude** |
| A-50 | ALTA | A secao `Capture` do `appsettings.json` era letra morta **por inteiro** — `services.Configure` era um lambda vazio. A frota mandava o dobro de batimentos e o triplo de resumos de input do que a config pedia |
| A-51 | CRITICA | Apos 3 crashes do worker em 5min, a sessao saia do registro e **ninguem voltava a olhar para ela**. A captura parava para sempre, em silencio, ate logoff ou restart do servico |

Tambem: A-35 reavaliado (funciona; o teste e que nao exercitava), A-44 corrigido em 3
ocorrencias (paralelismo de teste sobre estado estatico), A-27 corrigido no `srv-admin-node`.

**Suite: 659 testes no agent, 0 falhas. 1105 no admin-node, 0 falhas.**

---

## 5. O que o script automatizado cobre

```powershell
cd C:\Users\NoisyTech\Documents\Manager\manager-srv-agent

# bateria local (~4min, ou ~12min com -IncluirRateLimit)
.\tests\e2e\regressivo\run-regressivo-local.ps1
.\tests\e2e\regressivo\run-regressivo-local.ps1 -IncluirRateLimit

# conferencia na base, usando a janela de tempo da rodada acima
$env:MANAGER_DB_URL = "postgres://usuario:senha@host:5432/postgres?sslmode=require"
node tests\e2e\regressivo\verificar-eventos-base.js

# aceite do auto-update (PRECISA de terminal elevado)
.\tests\e2e\scenarios\e8-update-com-usuario-logado.ps1
```

**Nao precisa de elevacao**, exceto o e8. **Nao encoste no teclado** durante o bloco de
ociosidade — o script avisa e faz contagem regressiva.

| ID | Cenario | Ultima execucao na 1.5.10 |
|---|---|---|
| R1 | Captura de janela ativa (charmap, Paint da Store, titulos) | PASSOU |
| R5a | Janela aberta e fechada na morte do worker (A-35) | PASSOU |
| R2 | Deteccao de ociosidade | PASSOU |
| R3 | Heartbeat enviado | PASSOU |
| R3 | Intervalo do heartbeat respeita o appsettings (A-50) | PULADO — precisa de 3 batidas e a janela de 4min so cabe 2, agora que o intervalo e 120s |
| R4 | Digitacao nao e capturada (keylogger) | PASSOU |
| R4 | Titulo de janela sem conteudo (A-47) | PASSOU |
| R4 | URL so dominio raiz | PULADO — navegador nao foi usado |
| R5 | Worker morto volta sozinho | PASSOU |
| R5 | Ociosidade nao duplica no restart (A-48) | PASSOU |
| R7 | Memoria por processo < 150MB | PASSOU |
| R8 | Buffers SQLite presentes e fila drenando ate zero | PASSOU |
| R9 | `health-check.ps1` executa e **nao expoe segredo** | PASSOU |
| R9 | 6 scripts do pacote instalados | PASSOU |
| R10 | CPU media e crescimento do buffer | PASSOU |
| R11 | Rate limit dispara **e a sessao volta** (A-51) | PASSOU |
| Base | Eventos chegam, LGPD, sem evento orfao | 10 PASSOU, 0 falhou |
| e8 | Auto-update com usuario logado | `[E8 PASS]` em 36,1s |

Duas melhorias faceis de fazer, se sobrar tempo: aumentar a janela do R3 para ~7min (para
medir o intervalo de 120s) e abrir um navegador em 2-3 paginas do mesmo site no R4.

---

## 6. O que PRECISA ser testado a mao

O script marca esses como PENDENTE de proposito, em vez de deixar sumir num verde enganoso.

### Prioridade ALTA — codigo que mexemos nesta leva

**M1 — Bloqueio e desbloqueio de tela**

Por que importa: o `LockScreenDetector` foi **reescrito** no A-33, porque havia tres fontes
de estado de lockscreen que nao se falavam e geravam 16 LOCK e 26 UNLOCK falsos numa rodada.
Codigo novo, caminho nunca validado a mao depois da correcao.

1. Anote a hora. Aperte `Win + L`
2. Espere **30 segundos** com a tela bloqueada
3. Desbloqueie

O que conferir (log em `C:\ProgramData\ManagerAgent\logs\session-worker-S1-*.log` e tabela
`eventos_sessao` do `agente_id=51`):

- Exatamente **um** `LOCK` e **um** `UNLOCK` — nao varios
- A janela ativa **fecha** no bloqueio (`Window event closed`), senao tela bloqueada vira
  tempo trabalhado
- **Nenhum `LOGIN`** — bloquear nao e logar (regressao do A-34)
- Nenhuma ociosidade duplicada (A-48)
- `IdleMonitorService suspenso (lockscreen)` e depois `retomado`

**M2 — Logoff e login**

Por que importa: o dedup de LOGIN foi corrigido no A-34 (`WTSLogonTime` nao e suportado, o
caminho feliz nunca rodou em maquina nenhuma) e o upload imediato de fronteira de sessao no
A-36. Ambos novos.

1. Anote a hora. Faca logoff do Windows
2. Faca login de novo
3. Espere ~1min

O que conferir:

- Um `LOGOUT` e um `LOGIN`, **um de cada**
- O `LOGOUT` sobe **rapido** (A-36: fura o ciclo de 60s). Se so aparecer no boot seguinte, e
  o A-38 regredindo
- O evento de janela em aberto **fecha** no logoff (A-35)
- No log: `LOGIN suprimido — LogonId ja registrado` **nao** deve aparecer num login de
  verdade; deve aparecer em restart de worker

> **ATENCAO:** o logoff derruba a sessao do Windows e mata os processos do Claude Code. Faca
> o M2 por ultimo, ou aceite reabrir a sessao depois.

### Prioridade MEDIA — codigo nao tocado nesta leva

- **M3 Reboot** — agent sobe sozinho, icone volta em ate 30s, sem LOGIN duplicado
- **M4 Reuniao** (Teams/Meet/Zoom em primeiro plano por 3min) — `eventos_reuniao` com inicio e fim
- **M5 Suspender e acordar** — periodo dormindo nao vira hora trabalhada
- **M6 Sem internet** (10min com Wi-Fi desligado, usando o PC) — continua registrando,
  e ao voltar envia o acumulado **sem duplicar**
- **M8 Instalacao e desinstalacao** pelo instalador — deixar por ultimo, destroi a maquina de teste
- **M9 Icone e menu da bandeja** — verificacao visual
- **M10 Dois usuarios simultaneos** — precisa de segunda conta
- **M11 Ciclo de vida do Service** (parada graciosa, auto-restart do SCM) — precisa de elevacao
- **M12 AutonomousBuffer** (derrubar o Service e ver o worker bufferizar sozinho) — elevacao

**M7 — Status ATIVO/PAUSA/AUSENTE:** nao e testavel no Agent. Desde a v1.5.0 quem deriva e o
**backend**; o agent nao emite mais `StatusTransitionEvent`. Verificar no portal ou na tabela
`eventos_transicao_status`.

---

## 7. Aberto, com dono

| Item | Dono | Situacao |
|---|---|---|
| **A-47** titulo de janela carrega conteudo digitado | — | **ACEITO COMO ESTA** pelo Marcos: "erro do bloco, nao nosso". Teste continua no harness caso a decisao mude |
| **A-49** 12 agentes para a mesma maquina | @Shuri | Handoff pronto. `hardware_fingerprint` e estavel nas 12 e nao e usado para deduplicar |
| **A-27** `publicado_em` no futuro | @Shuri | **CORRIGIDO** no `srv-admin-node`. O dado sempre esteve certo; o defeito era de leitura (driver aplicando fuso do processo) |
| Passivo: 33 ociosidades + 27 janelas abertas na base | @Shuri + @Tony | O fix impede novas, nao fecha as antigas. Precisa decidir o criterio |
| `health-check.ps1` pendura 135s em I/O | @Bucky | O harness ja o chama com teto de 45s; a causa no script continua |
| 3 melhorias de teste (janela do R3, navegador no R4) | @Bucky | Opcional |

---

## 8. Publicacao — o que falta

**Nao publicado.** Para publicar:

1. **Pausar a 1.5.6 primeiro** — ela esta `ativa=true, obrigatoria=true`, e qualquer maquina
   em 1.5.5 tenta aplica-la e cai no loop do A-39/A-40:
   ```sql
   UPDATE versoes_agente SET pausada = true WHERE versao = '1.5.6';
   ```
2. Upload de `dist\ManagerAgent-Installer-v1.5.10.zip` para
   `.../manager-agent-releases/releases/WINDOWS/1.5.10/`
   - sha256 `16e278dae640ac02d9132c418a4812d273eddba5ffd9cd6dfbb0b39b71d9bc1e`
   - 164643210 bytes
3. INSERT em `versoes_agente` com `publicado_em = now() at time zone 'utc'` (a coluna e
   `TIMESTAMP WITHOUT TIME ZONE` e o padrao e UTC; `now()` puro gravaria local e recriaria o A-27)

**A-45 — restricao de rollout que vale para producao:** quem aplica o update e o applier da
versao **instalada**. Maquinas em 1.5.5/1.5.6 carregam o defeito e so conseguem aplicar a
1.5.10 **quando ninguem estiver logado**. Com usuario logado, entram em retentativa. O campo
`obrigatoria` **nao protege**: o agent nunca le esse campo, aplica o que o backend oferecer.

---

## 9. Armadilhas que ja custaram tempo

1. **`.ps1` tem que ser ASCII puro.** Em-dash dentro de string quebra o parser (achado A-05,
   ja reincidiu 3x). Conferir com
   `perl -ne 'print if /[^\x00-\x7F]/' arquivo.ps1`. Existe teste travando isso:
   `PowerShellScriptsAreAsciiTests`.
2. **Config que parece ligada e nao esta.** `AutoUpdate.Habilitado` **nao e lido por ninguem**;
   `MaximoBackups` so vale na Tray; `StickyVersion` e gravado e nunca lido;
   `sosRecoveryHeaderCount` nunca e incrementado. Antes de confiar num campo, `grep` quem le.
   O unico freio real do auto-update e `sosMode: true` em `watchdog-state.json`.
3. **Timestamp `WITHOUT TIME ZONE`:** comparar sempre o texto cru contra
   `now() AT TIME ZONE 'UTC'` antes de concluir que o dado esta errado. Dois relatos de
   "publicado_em no futuro" nasceram de ler a coluna com cliente que aplica fuso do processo.
4. **Automacao nao gera "atividade".** `Start-Process` e `AppActivate` nao mexem no
   `GetLastInputInfo`; a sessao entra em ociosidade e o agent **pausa a captura de janela** de
   proposito. Para testar captura, sintetize input (`SendKeys '{F15}'`).
5. **Bloco de Notas do Win11 e instancia unica com abas.** Abrir arquivo cria aba no processo
   do usuario — `SendKeys` acabaria digitando no documento **dele**.
6. **A suite mexe em estado estatico de processo** (`ConfigPaths.BaseFolderOverride`). O
   paralelismo do xUnit esta desligado nos dois assemblies que tocam isso; nao religar.
7. **A sessao do Claude Code roda sem elevacao.** Instalar, parar servico e o e8 exigem UAC.
   O padrao que funcionou: `Start-Process powershell -Verb RunAs` com um script que loga num
   arquivo, e o Marcos so clica "Sim".

---

## 10. Credencial

A string de conexao do banco de staging foi passada em chat e esta no historico daquela
sessao. **Vale trocar a senha.** Enquanto isso, ela nao foi gravada em nenhum arquivo do
repositorio.

---

## 11. Cenarios novos N1..N8 (2026-08-20, @Tony)

Automacao das lacunas levantadas sobre a 1.5.10. Nada commitado — @Bucky revisa antes.

### Como rodar

```powershell
cd C:\Users\NoisyTech\Documents\Manager\manager-srv-agent

# leves (sem elevacao, sem efeito colateral)
.\tests\e2e\regressivo\run-cenarios-novos.ps1 -SomenteLeves

# leves + elevados (reiniciam Service, mexem no config, restauram no fim)
.\tests\e2e\regressivo\run-cenarios-novos.ps1

# inclui os dois que aplicam update real e deixam a maquina na 1.5.99
.\tests\e2e\regressivo\run-cenarios-novos.ps1 -IncluirUpdatesReais
```

### O que cada um cobre

| ID | Grupo | Achado | O que prova |
|---|---|---|---|
| N1 | ELEVADO | A-46 | backend oferece a versao ja instalada e o agent recusa sem baixar |
| N2 | ELEVADO | A-41 | cooldown de 6h por relogio de parede, preservado entre reinicios e encerrado ao vencer |
| N3 | DESTRUTIVO | A-42 | poda de backups no caminho real, por idade, com backup novo preservado |
| N4 | LEVE | A-50 | a secao Capture vale na pratica: heartbeat e limiar de ociosidade medidos no log |
| N5 | DESTRUTIVO | — | evento pendente sobrevive ao update **e continua entregavel** depois |
| N6 | ELEVADO | A-49 | InstalacaoId estavel; `-IdEsperado` fecha o M8 com evidencia |
| N7 | ELEVADO | — | Device JWT expirado: renovacao proativa e reativa (401) de ponta a ponta |
| N8 | ELEVADO | — | 403 + `X-Agent-Revoked`: fixa o comportamento atual e documenta o gap |

Todos exigem evidencia positiva antes de aprovar (o stub recebeu a pergunta, a fila tinha
evento, houve deteccao normal de ociosidade). Sem isso, FALHA em vez de verde vazio.

### Testes unitarios novos (Service.Tests, 12 casos, verdes)

- `BackupPruneTests` — poda por idade, keep=0, keep negativo, menos pastas que o limite,
  pasta inexistente, pasta travada. Cobre os ramos que o E2E do N3 nao alcanca.
- `AppSettingsKeysAreReadTests` — toda chave de appsettings tem propriedade que a recebe.
- `CaptureOptionsHaveConsumersTests` — toda chave de `Capture` tem consumidor no
  SessionWorker.

Os dois ultimos travam a divida atual numa lista com justificativa: chave morta **nova**
quebra o build.

### Tres achados desta rodada

**A-52 — `IntervaloResumoEntradaSegundos` nao chega no consumidor.** ALTA.
`Worker.cs:55` usa `InputSummaryInterval = TimeSpan.FromSeconds(60)`, constante. O
appsettings pede 180s. O agent emite **3x mais resumo de input** do que o configurado.
O briefing do A-50 falava em "o dobro de batimentos e o triplo de resumos": a vinculacao
consertou o dobro; o triplo continua na 1.5.10. Mesma familia: `IntervaloLoopWorkerSegundos`
(constante `LoopDelay`) e `IntervaloCheckHooksSegundos` (nem existe como propriedade, e o
JSON pede 60s contra `HookCheckInterval` de 30s). Correcao no Agent: @Bucky.

**A-53 — sem tratamento de revogacao no Windows.** MEDIA, decisao com @Steve.
403 cai no ramo generico do `HttpEventUploader` e o `UploadWorker` retenta para sempre.
Nao ha leitura de `X-Agent-Revoked` em lugar nenhum do codigo Windows. O Agent Android
tem `RevocationInterceptor`. As duas pontas divergem, e continuar coletando sem vinculo
ativo tem lado de LGPD.

**A-54 — `Remove-Job -Force` no stub trava 120s.** BAIXA, ja corrigido.
A thread do job fica presa em `HttpListener.GetContext()`. Medido: 120s por cenario, no
`finally`. Corrigido com a rota `/__parar` no stub e o helper `setup\stop-stub.ps1`
(0,1s). **Os cenarios e1..e8 continuam usando `Remove-Job` direto** — trocar pela chamada
ao helper economiza ~16min numa bateria completa.


### Decisoes de 2026-08-20

- **A-52 bloqueia a publicacao.** Handoff em
  `registro/2026-08-20-handoff-bucky-a52.md`. Correcao com @Bucky, review com @Tony.
- **A-53 (revogacao / vinculo) IMPLEMENTADO.** A regra "so capturar e enviar com vinculo
  ativo com colaborador" foi implementada por @Tony em 2026-08-20 e aguarda review do
  @Bucky. Ver `adr/ADR-001-captura-exige-vinculo-ativo.md`. Aceite: N8 (com
  -ExigirParadaNaRevogacao) e N9. Rode o N9 em modo diagnostico antes de instalar.
- **A-54 corrigido** no stub (`/__parar` + `setup\stop-stub.ps1`).
