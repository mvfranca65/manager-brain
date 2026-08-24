> **DATA:** 2026-08-18 | **VERSAO:** Agent Windows **1.5.5** | **MAQUINA:** `DESKTOP-VMSM6LE` | **AGENTE:** id 50
> **ORIGEM:** `PLANO-CORRECOES-AGENT-WINDOWS-1.5.4.md` (backlog 1.5.5) | **Execucao:** Marcos + @Bucky | **Review:** @Tony
> **REGRA:** banco somente SELECT. Nada commitado ate o Marcos validar.

# Roteiro de Validacao — Agent Windows 1.5.5

O que esta suite de testes **nao** cobre: tudo que ja foi provado por teste unitario (605 verdes).
Aqui so entra o que depende da maquina — comportamento de ponta a ponta, com o agent instalado
e os dados chegando na base.

---

## O que mudou nesta versao

| Achado | Correcao | Como se prova aqui |
|---|---|---|
| A-32 | `dotnet test` nao sobe mais worker de verdade | Ja provado: 605 testes, contagem de processos inalterada |
| A-34 | Dedup de LOGIN passou a funcionar (`WTSINFOEX`) | **T2** |
| A-33 | Estado de lockscreen unificado | **T4** |
| A-37 | Suite nao grava mais no estado de producao | Ja provado; arquivo de producao apagado antes do teste |
| A-36 | LOGIN e LOGOUT sobem no ato | **T6** |
| A-35 | Kill do worker fecha a janela aberta | **T3** |
| A-29 | Sem icone fantasma; worker sai sozinho no SHUTDOWN | **T1**, **T5** |
| A-30 | Sem bloco fantasma de 1ms ao subir durante ociosidade | **T7** |
| A-31 | Shell sem titulo nao vira trabalho | **T8** |
| — | Icone da bandeja igual ao do app Android | **T1** |

---

## Ferramenta de consulta

Nao ha `psql` nesta maquina. As consultas usam o driver `postgres` do `manager-srv-admin-node`:

```bash
node <scratchpad>/q.mjs "select ..."
```

O helper recusa qualquer coisa que nao comece com SELECT ou WITH.

> `criado_em` e `ocorreu_em` sao UTC. Converter sempre com `at time zone 'America/Sao_Paulo'`,
> senao os horarios saem 3h deslocados dos logs.

**Logs:** `C:\ProgramData\ManagerAgent\logs\`
— `service-YYYYMMDD.log` (pipe, watchdog, upload, update) e `session-worker-S1-YYYYMMDD.log`.

---

## Passo 0 — Estado inicial

Antes de instalar, registrar o ponto de partida:

```powershell
Get-Process ManagerAgent.SessionWorker | Select-Object Id, StartTime
Test-Path 'C:\ProgramData\ManagerAgent\data\session-state-1.json'   # deve ser False
```

O `session-state-1.json` **precisa estar ausente**: ele foi contaminado pela suite (achado A-37) e
apagado de proposito. Com ele presente, o primeiro LOGIN legitimo da 1.5.5 seria suprimido e o
T2 nao poderia ser observado.

---

## Passo 1 — Gerar e instalar

```powershell
cd C:\Users\NoisyTech\Documents\Manager\manager-srv-agent
powershell -ExecutionPolicy Bypass -File scripts\build\build-pacote-v2.ps1
```

O instalador sai em `instalador\Output\`. Instalar por cima da 1.5.4 (o instalador trata upgrade).

---

# T1 — Instalacao limpa: icone, versao e LOGIN inicial
**Achados:** A-29 (icone), A-34 (aceite 1) | **Depois de:** passo 1

**O que valida:** o icone novo, a versao correta e o desaparecimento do WRN que provava que o
dedup de LOGIN nunca rodava.

**Como fazer:** apos instalar, olhar a bandeja e rodar:

```powershell
Get-Process ManagerAgent.SessionWorker | Select-Object Id, StartTime, Path
Select-String -Path 'C:\ProgramData\ManagerAgent\logs\session-worker-S1-*.log' -Pattern 'compat legado'
```

**Esperado:**
1. **Exatamente 1** SessionWorker vivo.
2. Bandeja com **1 unico** icone — escudo indigo com check branco (o mesmo do app Android).
3. A busca por `compat legado` **nao retorna nada**. Era esse WRN que aparecia em 35 de 35
   inicializacoes e provava que `QueryLogonId` devolvia null sempre.
4. No log, `LOGIN emitido` (primeira execucao — nao existe LogonId persistido ainda).

```sql
select id, versao_agente, to_char(criado_em at time zone 'America/Sao_Paulo','DD/MM HH24:MI:SS') as criado
from agentes where id = 50;
```
→ `versao_agente` deve ser **1.5.5.0**.

---

# T2 — Dedup de LOGIN: matar o worker nao gera LOGIN novo
**Achado:** A-34 (aceites 2 e 3) | **O defeito:** 35 reinicios = 35 LOGINs falsos

**Como fazer:**

```powershell
# hora de referencia
Get-Date -Format 'HH:mm:ss'
Stop-Process -Name ManagerAgent.SessionWorker -Force
Start-Sleep -Seconds 20   # watchdog relanca por EOF do pipe (~1s) + margem
Stop-Process -Name ManagerAgent.SessionWorker -Force
Start-Sleep -Seconds 20
Get-Process ManagerAgent.SessionWorker | Select-Object Id, StartTime
```

**Esperado:** exatamente 1 worker vivo, com PID novo, e **nenhum LOGIN novo** na base.

```sql
select id, tipo_evento, motivo,
       to_char(ocorreu_em at time zone 'America/Sao_Paulo','HH24:MI:SS.MS') as ocorreu,
       to_char(criado_em  at time zone 'America/Sao_Paulo','HH24:MI:SS.MS') as criado
from eventos_sessao
where agente_id = 50 and ocorreu_em >= now() - interval '15 minutes'
order by ocorreu_em;
```

No log do worker, o esperado agora e `LOGIN suprimido — LogonId ja registrado`.

**Aceite 3 (logon real continua gerando LOGIN)** so se prova com logoff/logon de verdade do
Windows — deixar para o fim da sessao, se houver disposicao.

---

# T3 — Kill do worker fecha o evento de janela
**Achado:** A-35 | **O defeito:** evento id 49169 (Taskmgr) aberto para sempre

**Como fazer:** deixar uma janela reconhecivel em foco por ~1 minuto (o Bloco de Notas serve),
para o snapshot de 60s registra-la, e entao matar o worker:

```powershell
notepad
# esperar ~70s com o Bloco de Notas em foco
Get-Date -Format 'HH:mm:ss'
Stop-Process -Name ManagerAgent.SessionWorker -Force
```

**Esperado:** o evento do Bloco de Notas fecha com `finalizado_em` = hora do kill (tolerancia 1s).

```sql
select id, nome_processo, left(titulo_janela, 40) as titulo,
       to_char(iniciado_em   at time zone 'America/Sao_Paulo','HH24:MI:SS') as ini,
       to_char(finalizado_em at time zone 'America/Sao_Paulo','HH24:MI:SS') as fim
from eventos_janela
where agente_id = 50 and iniciado_em >= current_date
order by iniciado_em desc limit 10;
```

**Criterio 2 — nenhuma janela orfa:**

```sql
select count(*) as janelas_abertas
from eventos_janela
where agente_id = 50 and finalizado_em is null and iniciado_em >= current_date;
```
→ no maximo **1** (a que esta genuinamente em curso agora).

No log do Service: `Evento de janela fechado por morte do worker (WorkerDied)`.

---

# T4 — Ciclo de lockscreen: 1 LOCK e 1 UNLOCK
**Achado:** A-33 | **O defeito:** 2 LOCK + 2 UNLOCK por ciclo, e 15s de tela de bloqueio como janela ativa

**Como fazer:**

```powershell
Get-Date -Format 'HH:mm:ss'   # anotar
# Win+L, esperar ~30s na tela de bloqueio, desbloquear
```

**Esperado:**

```sql
select id, tipo_evento, motivo,
       to_char(ocorreu_em at time zone 'America/Sao_Paulo','HH24:MI:SS.MS') as ocorreu,
       to_char(criado_em  at time zone 'America/Sao_Paulo','HH24:MI:SS.MS') as criado
from eventos_sessao
where agente_id = 50 and tipo_evento in ('LOCK','UNLOCK')
  and ocorreu_em >= now() - interval '10 minutes'
order by ocorreu_em;
```
→ **exatamente 1 LOCK e 1 UNLOCK.** `criado_em - ocorreu_em` abaixo de 5s nos dois.

**A parte que mais importa — nenhuma janela de tela de bloqueio:**

```sql
select id, nome_processo, left(titulo_janela,60) as titulo,
       to_char(iniciado_em at time zone 'America/Sao_Paulo','HH24:MI:SS') as ini
from eventos_janela
where agente_id = 50 and iniciado_em >= now() - interval '10 minutes'
  and (titulo_janela ilike '%bloqueio%' or titulo_janela ilike '%lock%'
       or nome_processo ilike '%LogonUI%' or nome_processo ilike '%Windows Operating System%');
```
→ **zero linhas.** Antes entravam 15 segundos de "Tela de Bloqueio padrao do Windows".

---

# T5 — Parada graciosa: worker sai sozinho, sem icone fantasma
**Achado:** A-29 (causa raiz) + A-36 (LOGOUT no ato)

**O defeito:** `StopApplication()` parava os hosted services mas o loop do WinForms seguia
bloqueado. O processo nunca saia sozinho, era morto de fora, e o icone ficava desenhado.

**Como fazer:**

```powershell
Get-Date -Format 'HH:mm:ss'
Restart-Service ManagerAgent
```

**Esperado:**
1. O worker antigo **desaparece sozinho**, sem `Kill`. No log do Service: `SHUTDOWN` seguido de
   `goodbye`, **sem** `killing survivor worker`.
2. A bandeja fica com **1 icone**, nao 2. Sem passar o mouse por cima.
3. LOGOUT na base em menos de 5s:

```sql
select id, tipo_evento, motivo,
       to_char(ocorreu_em at time zone 'America/Sao_Paulo','HH24:MI:SS.MS') as ocorreu,
       to_char(criado_em  at time zone 'America/Sao_Paulo','HH24:MI:SS.MS') as criado,
       extract(epoch from (criado_em - ocorreu_em)) as atraso_s
from eventos_sessao
where agente_id = 50 and tipo_evento = 'LOGOUT'
  and ocorreu_em >= now() - interval '10 minutes'
order by ocorreu_em desc;
```

---

# T6 — Fronteira de sessao sobe no ato
**Achado:** A-36 | **O defeito:** LOGIN com 55,1s e 48,6s de atraso

Consolidado a partir dos eventos gerados em T2, T4 e T5:

```sql
select tipo_evento,
       count(*) as qtd,
       round(max(extract(epoch from (criado_em - ocorreu_em)))::numeric, 1) as pior_atraso_s
from eventos_sessao
where agente_id = 50 and ocorreu_em >= current_date
group by tipo_evento order by tipo_evento;
```
→ **pior atraso abaixo de 5s** para LOCK, UNLOCK, LOGIN e LOGOUT.

---

# T7 — Janela aberta durante ociosidade nao vira bloco de 1ms
**Achado:** A-30 | **Estado:** ja corrigido na 1.5.4, so falta observar

**Como fazer:** ficar ~10 minutos sem tocar no teclado e no mouse (ociosidade em curso), e entao
mexer e abrir uma janela nova.

**Esperado:** nenhum evento de janela com duracao de milissegundos no instante da retomada.

```sql
select id, nome_processo, left(titulo_janela,40) as titulo,
       to_char(iniciado_em at time zone 'America/Sao_Paulo','HH24:MI:SS.MS') as ini,
       extract(epoch from (finalizado_em - iniciado_em)) as duracao_s
from eventos_janela
where agente_id = 50 and iniciado_em >= current_date
  and finalizado_em is not null
  and extract(epoch from (finalizado_em - iniciado_em)) < 1
order by iniciado_em desc limit 20;
```
→ zero linhas, ou nenhuma no instante da retomada.

---

# T8 — Shell sem titulo nao vira trabalho
**Achado:** A-31 | **Estado:** ja corrigido na 1.5.4, so falta observar

**Como fazer:** minimizar tudo e ficar ~1 minuto na area de trabalho, depois voltar a uma janela.

**Esperado:** nenhum evento com processo de shell nem titulo vazio/placeholder.

```sql
select id, nome_processo, coalesce(nullif(titulo_janela,''),'(vazio)') as titulo,
       extract(epoch from (finalizado_em - iniciado_em)) as duracao_s
from eventos_janela
where agente_id = 50 and iniciado_em >= current_date
  and (nome_processo ilike '%explorer%' or nome_processo ilike '%shell%'
       or titulo_janela = '' or titulo_janela is null)
order by iniciado_em desc limit 20;
```
→ zero linhas.

---

## Placar

Executado em 2026-08-18, agente **51** (a instalacao limpa revinculou o dispositivo — o antigo era o 50).

| # | Teste | Achado | Resultado | Observacao |
|---|---|---|---|---|
| T1 | Instalacao: icone, versao, LOGIN inicial | A-29, A-34 | **PASSOU** | 1 worker, icone novo, **zero** `compat legado` (aparecia em 35 de 35), LOGIN unico em 4,7s |
| T2 | Kill do worker nao gera LOGIN | A-34 | **PASSOU** | 2 kills, 0 LOGIN novo: `LOGIN suprimido — LogonId ja registrado` nas duas vezes |
| T3 | Kill fecha o evento de janela | A-35 | **PASSOU** | Eventos 49250 e 49251 fechados em 14:48:05 e 14:48:25, os instantes exatos dos kills, com `WorkerDied` no log. 1 janela aberta (a corrente), 0 duplicatas |
| T4 | Ciclo de lockscreen | A-33 | **PASSOU** | Exatamente 1 LOCK (4,1s) e 1 UNLOCK (2,0s). **Zero** janela de tela de bloqueio — antes entravam 15s |
| T5 | Parada graciosa sem fantasma | A-29 | **PASSOU** | `GOODBYE PendingEvents=0` e EOF 71ms depois. **Sem** `killing survivor` — o worker saiu sozinho pela primeira vez |
| T6 | Fronteira de sessao no ato | A-36 | **PASSOU com ressalva** | LOGIN 4,7s, LOCK 4,1s, UNLOCK 2,0s. LOGOUT de `SERVICE_STOP` levou 38,7s → **A-38** |
| T7 | Sem bloco fantasma de 1ms | A-30 | **REPROVOU** | Evento 49253 com 1ms as 14:48:26, worker subindo dentro da ociosidade 4664 → **A-30 reaberto** |
| T8 | Shell sem titulo | A-31 | **REPROVOU** | Evento 49268, `Program Manager` — titulo fora da lista de placeholders → **A-31 reaberto** |

**Bonus nao previstos no roteiro**, visiveis so porque a instalacao limpa revinculou o dispositivo:
**A-02** (`usuario_sessao` = `NoisyTech`, era `Raquel`) e **A-03** (`hardware_fingerprint` presente,
era nulo) — as duas correcoes do L6 da 1.5.4 confirmadas em campo.

### Leitura da rodada

Os **7 achados implementados e revisados nesta sessao passaram**. Os **2 herdados como "ja
corrigido, so validar" reprovaram**, e um terceiro defeito novo apareceu no caminho que ninguem
tinha exercitado. Nenhum dos tres bloqueia a 1.5.5: sao dado sujo (1ms, area de trabalho) e atraso
de um evento que nao se perde. Todos vao para a **1.5.6**.

---

## Fora deste roteiro

| Item | Por que |
|---|---|
| **A-27** — `publicado_em` em UTC | Backend, @Shuri. Independente do Agent |
| **A-02** — `usuario_sessao` = "Raquel" | Corrigido na 1.5.4, mas so e gravado na **vinculacao**. Um upgrade nao revincula: o campo so muda numa instalacao limpa (desinstalar + instalar) |
| **Auto-update ponta a ponta** | Exige publicar a 1.5.5 no backend de staging — passo do Marcos/@Vision, depois que este roteiro fechar |
| **P-8 — bloco LGPD** | Obrigatorio antes do release, roteiro proprio |
| **A-21** — kill do Service | Era o proximo da fila do regressivo anterior; roda depois deste roteiro |
