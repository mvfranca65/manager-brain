> **STATUS:** DRAFT — aguardando aprovacao Marcos
> **DATA:** 2026-08-14
> **AUTOR:** @Tony
> **EXECUTOR:** @Sam (Android)
> **CONSULTORIA:** @Shuri (srv-events-node — só leitura de contrato, sem mudança de API), @Groot (UI da jornada), @Natasha (QA)
> **APROVADOR OBRIGATORIO:** Marcos
> **VERSAO ALVO:** Agent Android v1.1.0 — entrega única (A + B + C + D), 25 tasks em 7 fases, com gate bloqueante no meio (§10)
> **PRECONDICAO:** spec `2026-08-13-agent-android-entrega-confiavel-health-design.md` — esta spec **corrige e estende**, não substitui
> **DESIGN:** `.brain/tecnologia/features/agent-android-jornada-v2/` (@Groot, layout FECHADO) — avaliado e aprovado com 4 ajustes em §7
> **REQUISITO DESTACADO:** ciclo de envio de 60 s — cadeia completa, correção e protocolo de prova em §4.3
> **REGRA APLICADA:** mudança no Agent exige spec formal + investigação exaustiva antes de código (`feedback_agent_spec_obrigatoria`)

# Agent Android — Entrega de eventos (BUG) e Jornada v2 (EVOLUÇÃO)

---

## 0. Leitura rápida para o Marcos

| Bloco | Natureza | Estado | Onde |
|---|---|---|---|
| **A — Entrega de eventos + ciclo de 60 s** | 🔴 **BUG em produção** | Fechado, pronto para executar | §3, §4, tasks A1-A10 |
| **B — Dados da Tela de Status** | 🟡 Evolução | Fechado | §5, tasks B1-B4 |
| **C — Exportar diagnóstico** | 🟡 Evolução | Fechado | §6, tasks C1-C3 |
| **D — Jornada v2 (4 telas + identidade)** | 🟡 Evolução | **Design do @Groot fechado.** Aprovado com 4 ajustes | §7, tasks D0-D6 |

**Execução:** entrega única, **25 tasks em 7 fases**, com um gate bloqueante no meio (§10).

**Sobre os 60 s que você pediu — leia isto.** O loop de 60 s já existe no código desde 2026-08-13 (`ManagerAgentService.kt:175`), mas ele **apenas pede** o envio: quem decide *quando* enviar é o JobScheduler do Android, que agrupa e adia a seu critério. Ou seja, mesmo com o bug de contrato corrigido, os 60 s não seriam garantidos. A correção que fecha isso é a task **A4B** — a drenagem passa a acontecer **direto no foreground service**, e o WorkManager vira só rede de segurança para quando o service estiver morto (§4.3).

Com isso o compromisso fica:

| Situação | Entrega |
|---|---|
| Colaborador usando o aparelho (a jornada de trabalho) | **≤ 60 s** — garantido, laço determinístico |
| Bloqueio/desbloqueio de tela e botão "Verificar agora" | **< 2 s** |
| Fabricante suspendeu o app | **≤ 15 min** — piso da plataforma, não dá para baixar |
| Restrição agressiva de bateria (Samsung) | pior que isso — **número exato medido em E-11**, não estimado |
| Qualquer situação | **zero perda de evento** |

E o @Sam **prova com evidência, não com teste verde**: 30 minutos de uso real, duas consultas SQL no staging medindo o intervalo entre heartbeats no relógio do servidor, com 5 limites objetivos e gravação de tela anexada (§4.3.5). Se `gap_max_s` vier ≥ 300 s, o blackout continua vivo e a tanque para.

Referência de paridade: o Agent Windows, medido, entrega ~70 s (10 s de drain + 60 s de upload). A paridade real é "≈1 minuto" nos dois (§4.2).

**Resumo do bug em uma frase:** o Agent Android nunca conseguiu entregar um único evento — o formato do JSON que ele monta não é o formato que o `srv-events-node` valida; o servidor responde 400 em 100% dos batches; o classificador de falha do app ignora 400 e não registra diagnóstico; o worker entra em backoff exponencial de até 5 horas e a política `KEEP` do WorkManager descarta todos os pedidos de envio seguintes. Daí "acumula e não envia" + "Sem diagnóstico".

---

## 1. Contexto e origem

### 1.1 O que foi relatado

Marcos, sobre o APK staging 1.0.13 num Samsung:
- "os eventos não estão sendo enviados, está acumulando e não envia"
- Tela de status: "Fila local: 1 eventos" parada, "Sem diagnóstico"
- Pedido: paridade com o Windows — **envio a cada 60s**
- "acha que podemos mostrar mais algumas informações também? Pegando até o agent Windows como base"
- "Exportar diagnóstico salva em algum lugar que nem sei onde é"

### 1.2 O que já foi decidido antes desta spec

A spec `2026-08-13-agent-android-entrega-confiavel-health-design.md` (STATUS: REVISADA, execução autorizada) **já definiu e já teve implementada** a arquitetura de três camadas:

```text
Foreground service ticker (60 s, sessão UNLOCKED) ─┐
LOCK / UNLOCK (upload imediato) ───────────────────┼─► UploadCoordinator ─► BatchSenderWorker ─► POST /api/agent/events
WorkManager periódico (15 min, recuperação) ───────┘         ▲
                                              Room EventQueue (lease atômica)
```

Decisões herdadas que **permanecem válidas e não são reabertas aqui**:
- Não existe nem existirá `POST /api/agent/health`. O sinal operacional canônico é o próprio `HeartbeatEvent`.
- Não usar `ExistingWorkPolicy.REPLACE` — um upload aceito pelo servidor não pode ser cancelado antes do ACK local.
- Fila Room com `PENDING`/`IN_FLIGHT`, `leaseId`, `leaseUntil`, `tentativas`; ACK só remove IDs da lease correspondente.
- Categorias de diagnóstico são um allowlist — nunca payload, segredo ou stack trace bruto.
- Reversão de D7/D8 da spec BYOD: o ícone **permanece** na gaveta e existe tela de status (antes o app se auto-ocultava).

### 1.3 Objetivo desta spec

1. Fechar o bug de entrega com causa raiz provada e correção mínima (Bloco A).
2. Definir **quais** informações a tela de Status expõe e de onde cada dado vem (Bloco B).
3. Decidir a solução Android correta para o export de diagnóstico (Bloco C).
4. Registrar o que fica bloqueado aguardando o @Groot (Bloco D).

---

## 2. Escopo

### 2.1 Dentro

- Correção do contrato de envio Android → `srv-events-node` (**sem mudar o backend**).
- Correção da política de retry/coalescência de uploads.
- Fim do descarte silencioso de falhas e de eventos.
- Telemetria local de entrega (última tentativa, último sucesso, enviados hoje).
- Lista final de informações da tela de Status + origem de cada dado.
- Solução de exportação/compartilhamento de diagnóstico.

### 2.2 Fora

| Item | Motivo | Vai para |
|---|---|---|
| Qualquer mudança em `srv-events-node` | O backend está correto; quem diverge do contrato é o cliente | — |
| Endpoint/tabela de health | Já descartado em 2026-08-13 | — |
| Dashboards Grafana | Pausado por Marcos | Rodada futura @Vision |
| Desenho das 4 telas da jornada | Domínio do @Groot | Bloco D |
| Escopo/priorização de produto | Domínio do @Steve | — |
| Bump para `targetSdk 35` | Vira problema de FGS timeout (ver R-6) | Rodada futura |
| Multi-dispositivo no portal | Spec própria (`2026-08-11-multi-dispositivo-portal-design.md`) | — |

---

## 3. Diagnóstico de causa raiz — Frente 1

> Toda linha abaixo foi verificada lendo o código. Caminhos relativos à raiz de
> `/Users/Olimpo/Documents/Athena/Projetos/Agente/manager-srv-agent-android`
> e de `/Users/Olimpo/Documents/Athena/Projetos/Backend/manager-srv-events-node`.

### 3.1 CR-1 — 🔴 O envelope do evento não é o que o backend valida (causa primária)

O Android serializa o evento **flat**, com o discriminador no mesmo nível dos campos:

- `app/src/main/kotlin/com/trivion/manageragent/network/ApiClient.kt:35-39` → `Json { classDiscriminator = "tipoEvento" }`
- `app/src/test/kotlin/com/trivion/manageragent/event/model/AgentEventSerializationTest.kt:61` assere literalmente:
  ```json
  {"tipoEvento":"HeartbeatEvent","enviadoEm":"2026-08-08T14:00:00Z","eventosPendentes":42}
  ```

O backend exige o payload **aninhado sob `dados`**:

- `src/modules/ingestion/dto/agent-event-batch.dto.ts:340-370` → `z.discriminatedUnion('tipoEvento', [ z.object({ tipoEvento: z.literal(...), dados: <payload> }), ... ])`

Sem `dados`, o `safeParse` falha para **todo** item do array.

### 3.2 CR-2 — 🔴 Faltam 4 campos obrigatórios de topo no batch

- Android envia só 3 campos: `app/src/main/kotlin/com/trivion/manageragent/network/dto/EventBatchDtos.kt:11-16` → `agenteId`, `dispositivoTipo`, `eventos`.
- Backend exige adicionalmente `maquinaId`, `versaoAgente`, `descricaoSo`, `enviadoEm`: `src/modules/ingestion/dto/agent-event-batch.dto.ts:378-392`.
- `agenteId` **não existe** no schema do backend (o `agenteId` real vem do Device JWT — `src/modules/ingestion/ingestion.service.ts:169`). Como o `z.object` não é `.strict()`, o campo é descartado em silêncio: é campo morto, não erro.

**Consequência de CR-1 + CR-2:** `src/modules/ingestion/ingestion.service.ts:103-114` lança `ManagerValidationException(CODIGO_PAYLOAD_INVALIDO)` → **HTTP 400**, batch inteiro perdido, zero eventos gravados. Isso vale para **100% dos batches desde a primeira versão**. O Agent Android **nunca entregou um evento**.

Bom sinal colateral: todos os 8 `@SerialName` do Android existem na união do backend (incl. `HeartbeatEvent` e `PhoneCallEvent`), e `dispositivoTipo = "MOBILE_ANDROID"` é valor válido (`src/modules/ingestion/normalizar-dispositivo.ts:67`). O problema é só de envelope e campos de topo — não de vocabulário.

### 3.3 CR-3 — 🔴 400 é a única classe de falha que o app decidiu não enxergar

```kotlin
// app/src/main/kotlin/com/trivion/manageragent/sender/UploadCoordinator.kt:59-64
fun fromHttpStatus(status: Int): AgentDiagnosticCode? = when (status) {
    401 -> AgentDiagnosticCode.HTTP_401
    429 -> AgentDiagnosticCode.HTTP_429
    in 500..599 -> AgentDiagnosticCode.HTTP_5XX
    else -> null          // ← 400, 403, 404, 409, 413, 422 caem aqui
}
```

`BatchSenderWorker.kt:68-72` só grava diagnóstico quando o classificador devolve não-nulo — e **retorna `Result.retry()` de qualquer jeito**. E isso está **asserido como comportamento desejado** no teste: `app/src/test/kotlin/com/trivion/manageragent/sender/UploadCoordinatorTest.kt:37` → `assertEquals(null, UploadFailureClassifier.fromHttpStatus(400))`.

É exatamente o padrão vetado pela regra `feedback_enum_sem_coalescencia_silenciosa`, na versão pior: não coalesce para um default — apaga.

**É isto que produz "Sem diagnóstico" na tela do Marcos.** A tela está tecnicamente correta: nenhum diagnóstico foi jamais registrado. O app falhou 100% das vezes por uma classe de erro que ele foi programado para não registrar.

### 3.4 CR-4 — 🔴 `ExistingWorkPolicy.KEEP` + backoff exponencial de 5 min = blackout de até 5 horas

```kotlin
// app/src/main/kotlin/com/trivion/manageragent/sender/UploadCoordinator.kt:24-30
.setBackoffCriteria(BackoffPolicy.EXPONENTIAL, BACKOFF_MINUTES /* 5 */, TimeUnit.MINUTES)
workManager.enqueueUniqueWork(WORK_NAME, ExistingWorkPolicy.KEEP, request)
```

`KEEP` descarta a nova solicitação enquanto existir trabalho **não finalizado** com aquele nome. Um worker que retornou `Result.retry()` fica `ENQUEUED` durante toda a janela de backoff — ou seja, **não finalizado**. Logo, cada `request()` seguinte é jogada fora:

- o tick de 60 s (`HEARTBEAT`) — `service/CollectorTicker.kt:75`
- `SESSION_LOCK` / `SESSION_UNLOCK` — `service/SessionBroadcastReceiver.kt:45`

Backoff exponencial a partir de 5 min: 5 → 10 → 20 → 40 → 80 → 160 → 300 (teto de 5 h do WorkManager). Como a falha é **permanente** (CR-1/CR-2), o contador nunca reseta. Em poucas horas o app entra em janela de 5 h de silêncio e nunca sai.

**Achado B do scout: CONFIRMADO.** E é pior do que o descrito, porque a falha subjacente não é transitória.

Nota: o teste existente `app/src/test/kotlin/com/trivion/manageragent/sender/BatchSenderWorkerTest.kt:104-114` (`ten upload requests coalesce into one unique work request`) prova a coalescência no caminho feliz, mas **nunca exercita o caminho de backoff**. A coalescência em si é desejável; o defeito é usar o backoff do WorkManager como mecanismo de retry num uploader guiado por cadência.

### 3.5 CR-5 — 🟠 Resposta incompatível derruba até o caminho de sucesso

- Android desserializa `EventBatchResponse(aceitos: Int, ...)` — `network/dto/EventBatchDtos.kt:18-23`. `aceitos` **não tem default** → é obrigatório em kotlinx.serialization.
- Backend devolve `{ aceito, totalEventos, eventosJanela, ..., motivosIgnorados }` — `src/modules/ingestion/ingestion.service.ts:189-205`. Não existe `aceitos`.

`ignoreUnknownKeys = true` cobre campos extras, mas não campo obrigatório ausente → `MissingFieldException` → cai no `catch (exception: Exception)` de `BatchSenderWorker.kt:77-84` → `release()` + `Result.retry()` **mesmo com o servidor tendo gravado tudo**. Ou seja: corrigir só CR-1/CR-2 troca "400 eterno" por "reenvio eterno com duplicação". CR-5 é obrigatório no mesmo release.

### 3.6 CR-6 — 🟠 Sem header de dedup, todo retry duplica evento

O backend suporta `X-Client-Dedup-Batch` e responde **208** em batch repetido (`src/modules/ingestion/ingestion.controller.ts:180-197,276-278`). O Android nunca envia o header (`network/AuthInterceptor.kt:14-25`, `sender/BatchSenderWorker.kt:121-130`). Com a fila drenando pela primeira vez, um timeout de rede após o commit do servidor grava tudo duas vezes.

### 3.7 CR-7 — 🔴 A rede de segurança de coleta está desligada exatamente quando é necessária

```kotlin
// app/src/main/kotlin/com/trivion/manageragent/di/AppContainer.kt:74-76
fun sessionGate(): SessionCollectionGate = _sessionGate ?: synchronized(this) {
    _sessionGate ?: SessionCollectionGate().also { _sessionGate = it }   // initiallyUnlocked = false
}
```

`SessionCollectionGate.kt:15` → default `initiallyUnlocked = false`. O gate só vira `true` em `service/ManagerAgentService.kt:110`.

Quando o OEM mata o processo e o WorkManager o revive **sem** subir o Service (JobScheduler acorda só o worker), o `AppContainer` novo nasce com o gate **fechado**:

- `collector/UsageStatsReconcileWorker.kt:55` → `UsageStatsReconcileRunner.run()` → `collector/UsageStatsReconcileRunner.kt:14` → `gate.withUnlockedPermit { ... }` retorna `null` → **a reconciliação inteira vira no-op silencioso**.
- `event/EventQueue.kt:22` → `sessionGate.collectIfUnlocked { ... }` → qualquer evento não-`Session` enfileirado fora do processo do Service é **descartado sem log**.

O `UsageStatsReconcileWorker` existe, segundo seu próprio KDoc, para "rodar a cada 15min como fallback quando o AccessibilityService é morto por OEMs agressivos" (`collector/UsageStatsReconcileWorker.kt:38-41`). Ele está desabilitado precisamente no cenário que justifica sua existência. Num Samsung, esse é o cenário normal.

### 3.8 CR-8 — 🟠 O app nunca consegue pedir isenção de otimização de bateria

`oem/OEMIntentFactory.kt:143` documenta: *"Requer permissão REQUEST_IGNORE_BATTERY_OPTIMIZATIONS no Manifest (declarada no B3)"*. Ela **não está declarada** — `app/src/main/AndroidManifest.xml` lista apenas INTERNET, ACCESS_NETWORK_STATE, WAKE_LOCK, FOREGROUND_SERVICE, FOREGROUND_SERVICE_DATA_SYNC, POST_NOTIFICATIONS, RECEIVE_BOOT_COMPLETED, READ_PHONE_STATE, PACKAGE_USAGE_STATS.

Sem a permissão, `Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` (`oem/OEMIntentFactory.kt:149`) é rejeitado pelo sistema. O `catch` em `ui/PermissionDispatchActivity.kt:379-383` engole e **pula a etapa em silêncio**. No Samsung, se o intent proprietário (`com.samsung.android.lool/...BatteryActivity`) não resolver, não sobra fallback — o aparelho segue otimizando o app, "colocando para dormir", e matando o foreground service.

### 3.9 CR-9 — 🟡 Reprogramação impossível em app já instalado

`sender/BatchScheduler.kt:33-37` usa `ExistingPeriodicWorkPolicy.KEEP`. Um aparelho que já instalou o app **mantém o agendamento antigo para sempre**; mudar `intervalMinutes` numa versão nova não tem efeito nenhum. O teste em `BatchSenderWorkerTest.kt:177-183` só verifica que não lança exceção.

**Achado B (segunda metade): CONFIRMADO.** Correção: `ExistingPeriodicWorkPolicy.UPDATE` (disponível desde WorkManager 2.8; o projeto está em 2.9.0 — `gradle/libs.versions.toml`).

### 3.10 CR-10 — 🟡 Diagnóstico nunca se cura

`diagnostics/AgentDiagnosticStore.kt:28-37` só tem `record()`. Não existe `clear()`. Uma vez gravado um `HTTP_5XX`, a tela mostra "Atenção" para sempre, mesmo depois de tudo voltar ao normal. E "Sem diagnóstico" é ambíguo: não distingue *nunca falhou* de *nunca tentou*.

### 3.11 CR-11 — 🟡 Não existe watchdog do Service

`service/BootReceiver.kt` só cobre `BOOT_COMPLETED`/`LOCKED_BOOT_COMPLETED`. Não há `ACTION_MY_PACKAGE_REPLACED` (o próprio auto-update deixa o Service morto até o próximo boot) nem nada que religue o Service depois de o OEM matá-lo. `START_STICKY` não sobrevive a force-stop nem ao bucket `restricted` do App Standby.

### 3.12 CR-12 — 🟡 Três dos oito códigos de diagnóstico são letra morta

`diagnostics/AgentDiagnosticCode.kt` declara 8 códigos. Grep em `app/src/main`: `QUEUE_LIMIT_REACHED`, `PERMISSION_MISSING` e `TOKEN_REFRESH_FAILED` **nunca são gravados por ninguém**. (`TOKEN_REFRESH_FAILED` existe como *string* no `CrashReporter`, que é outro canal.)

Em particular `QUEUE_LIMIT_REACHED` sugere um teto de fila que **não existe**: `EventDao` não tem contador de tentativas por evento nem política de descarte, e `EventQueue.pendingCount()` é um `SELECT COUNT(*)` sem limite. Com o envio quebrado por semanas, a tabela `event` cresce sem teto no celular **pessoal** do colaborador. O Agent Windows V2 tem o mesmo buraco (o legado tinha `MaxPendingEvents = 50_000`, não portado).

### 3.13 Encadeamento — por que o sintoma é exatamente esse

```text
CR-1 + CR-2  →  HTTP 400 em 100% dos batches
      │
      ├─► CR-3: 400 não é classificado  →  nenhum diagnóstico gravado  →  tela mostra "Sem diagnóstico"
      │
      └─► BatchSenderWorker retorna Result.retry() (BatchSenderWorker.kt:72)
              │
              └─► CR-4: work fica ENQUEUED em backoff 5→10→20→...→300 min
                      │
                      └─► ExistingWorkPolicy.KEEP descarta os ticks de 60 s e os LOCK/UNLOCK
                              │
                              └─► "acumula e não envia"

CR-7 (em paralelo)  →  processo morto pelo OEM  →  gate fechado  →  coleta vira no-op
                          →  a fila para de crescer  →  "Fila local: 1 eventos" PARADA
```

O "1 evento parado" fecha o quadro: não é uma fila explodindo, é uma fila **congelada** — a coleta parou (CR-7) e o envio está em blackout (CR-4). Os dois sintomas que o Marcos viu vêm de duas causas diferentes, e ambas precisam ser corrigidas.

### 3.14 Veredito sobre os achados do scout

| Achado do scout | Veredito | Observação |
|---|---|---|
| **A** — "o intervalo de 60 s é impossível como está modelado" | ⚠️ **REFUTADO na conclusão, correto na premissa** | O mínimo de 15 min do `PeriodicWorkRequest` é real e é restrição dura da plataforma. Mas o envio de 60 s **não passa** por ele: já existe um ticker de 60 s no foreground service — `service/ManagerAgentService.kt:175` (`HEARTBEAT_INTERVAL_MS = 60_000L`) → `service/CollectorTicker.kt:69-76` → `uploadRequester.request(HEARTBEAT)`. A arquitetura que o scout propõe ("loop de 60 s no service + WorkManager como rede de segurança") **já é a arquitetura implementada**, e é a certa. Ela nunca funcionou por causa de CR-1..CR-4, não por causa do WorkManager. |
| **B** — "descarte silencioso por `ExistingWorkPolicy.KEEP`" | ✅ **CONFIRMADO** e agravado | Ver CR-4. E `ExistingPeriodicWorkPolicy.KEEP` de fato congela o intervalo em apps já instalados — CR-9. |
| "a fila persiste e nunca drena, ou o servidor rejeita?" | **O servidor rejeita** — 400 em todos os batches (CR-1/CR-2). E a fila também congelou, por outro motivo (CR-7). |
| "se for 401/403 a causa é outra (Device JWT)" | **Não é 401/403.** O caminho de auth está correto: `RefreshInterceptor` trata 401 renovando via `srv-admin`, `RevocationInterceptor` trata 403 + `X-Agent-Revoked`. A semântica do backend (401 = credencial ruim/expirada, 403 = revogado/bloqueado) casa com o cliente. |
| "'Sem diagnóstico' — ou não falhou, ou a falha não está sendo registrada" | **A segunda.** Falhou sempre; 400 foi deliberadamente excluído do allowlist (CR-3). |

---

## 4. Decisões arquiteturais

### 4.1 Tabela de decisões

| # | Decisão | Rationale |
|---|---|---|
| **D1** | O cliente se adapta ao contrato do `srv-events-node`. **Zero mudança no backend.** | O backend é o contrato compartilhado com o Agent Windows, que funciona em produção. Mudar o servidor para acomodar um cliente quebrado quebraria o cliente que funciona. Fora de escopo do @Sam e sem custo para a @Shuri. |
| **D2** | Manter a arquitetura de 3 camadas de 2026-08-13 (ticker 60 s → coordinator → WorkManager 15 min). | Ela está correta. O bug não é arquitetural. |
| **D3** | **O `BatchSenderWorker` para de retornar `Result.retry()` para falha de envio.** Retorna `Result.success()` (trabalho finalizado) e registra a falha + um *cooldown local*. | Remove a interação tóxica entre backoff do WorkManager e `ExistingWorkPolicy.KEEP`. Com o work sempre finalizando, `KEEP` volta a fazer só o que se espera dele: coalescer pedidos genuinamente concorrentes. A cadência de retry passa a ser a cadência natural do sistema (60 s com service vivo, 15 min sem), que é observável, testável e explicável ao usuário. Os eventos não se perdem — `release()` da lease já os devolve a `PENDING`. |
| **D4** | Backoff vira **estado do app**, não estado do WorkManager: `nextAttemptAtMillis` persistido; o worker sai com `success()` sem tentar enquanto o cooldown vigora. Cooldown: 60 s → 2 min → 5 min → 15 min (teto). | Backoff continua existindo (não vamos martelar servidor doente), mas fica legível, testável em unit test e exibível na tela ("próxima tentativa em ~2 min"). O estado interno do WorkManager é opaco e não pode ser mostrado ao colaborador. |
| **D5** | `UploadFailureClassifier` passa a classificar **toda** resposta não-2xx e **toda** exceção. Nada retorna `null` silenciosamente. | `feedback_enum_sem_coalescencia_silenciosa`. Um erro que o cliente não sabe nomear ainda assim precisa aparecer como erro. |
| **D6** | Novos códigos: `HTTP_400_CONTRATO`, `HTTP_403_BLOQUEADO`, `HTTP_4XX_OUTRO`, `RESPOSTA_INVALIDA`, `SEM_ENVIO_RECENTE`. | Cobrem exatamente os buracos encontrados. `SEM_ENVIO_RECENTE` é derivado (sem sucesso há > 30 min com service vivo e rede) e resolve a ambiguidade do "Sem diagnóstico". |
| **D7** | `AgentDiagnosticStore` ganha `clear()`, chamado em todo upload aceito. | Sem isso a tela nunca volta a "Funcionando corretamente" (CR-10). |
| **D8** | `AppContainer.sessionGate()` deriva o estado inicial de `KeyguardManager.isKeyguardLocked` + `PowerManager.isInteractive`. | Corrige CR-7. Qualquer ponto de entrada do processo (worker, accessibility service, activity) passa a nascer coerente com a realidade do aparelho, não com um default pessimista. |
| **D9** | `BatchScheduler` usa `ExistingPeriodicWorkPolicy.UPDATE`. | Corrige CR-9. Sem isso nenhuma mudança futura de intervalo chega a aparelho já instalado. |
| **D10** | Declarar `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` e **expor o estado da isenção na tela de Status**, com botão de ação. | Corrige CR-8. Em BYOD não dá para forçar; dá para tornar visível e acionável em um toque. Também é pré-requisito para o watchdog (D11). |
| **D11** | Watchdog: `ACTION_MY_PACKAGE_REPLACED` + verificação no `BatchSenderWorker` que religa o Service se ele estiver morto. | Corrige CR-11. `startForegroundService` a partir de background é permitido para app isento de otimização de bateria — por isso D10 vem antes. Se não estiver isento, o watchdog não insiste: apenas registra `SERVICE_MORTO_E_REINICIADO`/`PERMISSION_MISSING` e a tela mostra a pendência. |
| **D12** | Enviar `X-Client-Dedup-Batch` (hash estável dos IDs da lease) e `X-Installation-Id`. | Corrige CR-6. Primeira drenagem da fila acumulada vai reenviar; sem dedup, duplica dado em produção. |
| **D13** | Compartilhar diagnóstico via **share sheet** como ação primária; "Salvar em Downloads" via MediaStore como secundária. | Ver §6. |

### 4.2 O que "paridade de 60 s" significa em Android

#### 4.2.1 O que o Windows faz de fato

Auditei o Agent Windows V2 (`/Users/Olimpo/Documents/Athena/Projetos/Agente/manager-srv-agent`) antes de decidir. Os fatos, com evidência:

| Aspecto | Windows V2 | Evidência |
|---|---|---|
| Intervalo de upload | **60 s fixo, hardcoded** | `src/ManagerAgent.Service/Upload/UploadWorker.cs:26` — `TimeSpan.FromSeconds(60)`. Comentário na linha 24-25: *"Requisito do Marcos: eventos devem chegar ao backend em ate 60s"* |
| Mecanismo | `BackgroundService` + `Task.Delay` fixo. **Sem timer, sem PeriodicTimer** | `UploadWorker.cs:64-91` |
| Configurável? | **Não.** `appsettings.json:13` tem `IntervaloUploadSegundos: 60` mas **não é bindado** — não existe `Configure<ManagerAgentUploadOptions>` no `Service/Program.cs`. É valor decorativo | grep confirmado |
| Latência real ponta a ponta | **~70 s** — 10 s de drain SessionWorker→pipe + 60 s de upload | `SessionWorker/Worker.cs:51` + `UploadWorker.cs:26` |
| Batching por volume | **Não existe gatilho por volume.** Só teto por ciclo: 100 eventos/batch; numa máquina de 1 usuário isso dá **100 eventos/min de vazão máxima** | `UploadWorker.cs:27-29`, `Storage/SqliteEventBuffer.cs:261-324` |
| Retry / backoff | **Não existe.** Zero Polly, zero `WaitAndRetry`. Falha → `return false` sem espera; o próximo tick de 60 s tenta de novo. Único retry é 1 reenvio após refresh em 401 | `Upload/HttpEventUploader.cs:21,175-208`; grep por `Polly`/`AddPolicyHandler` → zero |
| Falha bloqueia os envios seguintes? | **Não.** O `foreach` sobre os batches **não faz `break`** no fracasso; segue para o próximo | `UploadWorker.cs:144-165` |
| Fila persistida | Sim — SQLite em `C:\ProgramData\ManagerAgent\data\events.db`, `uploaded=0` até HTTP 2xx/208 | `Storage/SqliteEventBuffer.cs:55,274` |
| Dedup no reenvio | Sim — header `X-Client-Dedup-Batch`, 208 tratado como sucesso | `Upload/HttpEventUploader.cs:150-151,159-165` |
| Flush em LOCK/UNLOCK | **Não funciona.** `_flushSignal.Signal()` é disparado, mas **ninguém consome o sinal no V2** — o único `WaitAsync` está no projeto legado que não é empacotado. O log diz "disparando flush imediato" e nada acontece | `SessionWorker/Capture/SessionEventService.cs:99-104` vs. `Tray/Workers/UploadWorker.cs:87` (legado) |
| Flush no shutdown | Chega ao SQLite local, **nunca ao backend**. `StopAsync` só faz checkpoint do WAL | `ManagerAgentService.cs:235-259` |

**Três conclusões que mudam a conversa:**

1. **O modelo do Windows é exatamente o que proponho em D3/D4 para o Android:** cadência fixa, sem backoff de framework, falha não bloqueia nada, próximo tick reprocessa a fila persistida. Não estou inventando um modelo novo — estou **removendo** do Android um mecanismo (backoff do WorkManager + `KEEP`) que o Windows nunca teve e que é a causa do blackout. A paridade fica mais próxima depois da correção, não mais distante.
2. **O "60 s" do Windows já é ~70 s na prática.** O compromisso real que o produto entrega hoje no desktop é "≈1 minuto", não "60 s cronometrados". O Android com esta spec entrega o mesmo.
3. **O Android já supera o Windows em dois pontos:** o flush imediato em LOCK/UNLOCK **funciona** no Android (`SessionBroadcastReceiver.kt:45`) e é código morto no Windows; e o `HeartbeatEvent` do Android carrega a contagem real de pendentes, enquanto o do Windows manda `EventosPendentes = 0` hardcoded (`SessionWorker/Capture/HeartbeatService.cs:42`).

#### 4.2.2 O compromisso do Android

A paridade que faz sentido é de **comportamento observável**, e é honesto declarar o que Android impõe e o Windows não:

| Situação | Comportamento do Agent Android após esta spec | Atraso máximo esperado |
|---|---|---|
| Aparelho em uso, tela desbloqueada, service vivo | Ticker de 60 s dispara heartbeat + upload | **≤ ~60 s** |
| LOCK / UNLOCK | Upload imediato, sem esperar o tick | **imediato** |
| Sem rede | Fila preservada em Room; constraint `NetworkType.CONNECTED` segura o worker até voltar | até a rede voltar |
| Tela bloqueada / aparelho no bolso | Ticker parado **por decisão de projeto** (session gate). Nada é coletado, logo nada há para enviar | n/a |
| Service morto pelo OEM (Samsung "colocar para dormir") | WorkManager periódico continua drenando; watchdog tenta religar o Service | **≤ 15 min** |
| Doze profundo (tela off + parado + sem carga) | WorkManager só roda nas janelas de manutenção do sistema | até a próxima janela do SO |

**Por que 15 min é o piso e não é negociável:** `PeriodicWorkRequest.MIN_PERIODIC_INTERVAL_MILLIS` é 15 minutos e é constante da plataforma, não configuração. Não existe `PeriodicWorkRequest` de 60 s em nenhuma versão do Android. O que existe de 60 s é o loop dentro de um foreground service — e é isso que o app já faz.

**Custo de bateria dos 60 s:** enquanto a sessão está desbloqueada, o aparelho está na mão do usuário, tela acesa, rádio já ativo. Um POST pequeno por minuto sobre conexão TLS já quente é ruído em cima de um consumo que já está acontecendo. Aproximadamente 480 requests numa jornada de 8 h — 1/60 do orçamento de rate limit do backend (60/min por `instalacaoId`, `src/modules/ingestion/services/rate-limit.service.ts:20-21`). O ponto crítico é que **o loop de 60 s nunca roda com a tela apagada**, que é exatamente quando Doze puniria. Doze e App Standby, portanto, quase não interagem com a cadência; interagem com a **sobrevivência do processo**, e é isso que D10/D11 endereçam.

**Resposta direta ao Marcos:** 60 s é viável e já está no código. Não vou prometer 60 s garantidos, porque nenhum app Android pode prometer isso. O compromisso defensável é: **≤ ~1 min em condição normal de uso, ≤ 15 min no pior caso de app adormecido pelo fabricante, e zero perda de evento em qualquer caso.** Se o Samsung puser o app para dormir e a isenção de bateria não estiver concedida, o pior caso passa a ser "até o usuário desbloquear o aparelho" — e é por isso que a isenção de bateria vira informação de primeira classe na tela de Status (§5).

### 4.3 A cadeia dos 60 s — o que muda no código para o ciclo **entregar**, não só pedir

Requisito do Marcos, tratado como requisito de primeira classe: **os eventos precisam sair a cada 60 s de verdade.** Rastreei a cadeia inteira, elo por elo, do tick até a linha gravada no Postgres. Existe um elo que a minha análise anterior não fechou.

#### 4.3.1 A cadeia hoje, elo por elo

| # | Elo | Onde | Entrega em 60 s? |
|---|---|---|---|
| 1 | Ticker dispara a cada 60 s | `ManagerAgentService.kt:175` → `CollectorTicker.kt:33-47` | ✅ sim, enquanto a sessão está desbloqueada |
| 2 | Heartbeat é enfileirado | `CollectorTicker.kt:70-74` | ✅ |
| 3 | Upload é **solicitado** | `CollectorTicker.kt:75` → `UploadCoordinator.kt:23-30` | ⚠️ **é só um pedido** |
| 4 | WorkManager decide quando rodar o worker | `enqueueUniqueWork` → JobScheduler | 🔴 **ELO QUEBRADO** |
| 5 | Worker drena e faz POST | `BatchSenderWorker.kt:46-93` | ✅ quando roda |
| 6 | Backend valida e grava | `ingestion.service.ts` | 🔴 400 hoje (CR-1/CR-2) |

**O elo 4 é o buraco.** `UploadCoordinator.request()` não envia nada — ele **enfileira um `OneTimeWorkRequest`** e entrega a decisão de *quando executar* ao JobScheduler do Android. JobScheduler não tem compromisso com 60 s: ele agrupa jobs, respeita o *standby bucket* do app e adia à vontade. Com o foreground service vivo o app costuma estar em bucket ativo e o job sai em segundos — mas isso é **comportamento observado, não contrato**. Construir um requisito de 60 s em cima da discricionariedade do JobScheduler é exatamente o tipo de promessa que não se cumpre.

Somando a isso o `ExistingWorkPolicy.KEEP` (CR-4), o elo 4 hoje não só atrasa: ele **descarta**.

#### 4.3.2 A correção — o service envia; o WorkManager vira só rede de segurança

A mudança é de responsabilidade, e é pequena:

**Extrair a lógica de drenagem do Worker para uma classe própria, sem dependência de WorkManager**, e passar a chamá-la **direto** do foreground service.

```text
ANTES
  ticker 60 s ──► UploadCoordinator.request() ──► [JobScheduler decide] ──► BatchSenderWorker ──► POST

DEPOIS
  ticker 60 s ──► EventUploader.drain() ──────────────────────────────────────────────────────► POST
                        ▲                                        (chamada direta, no escopo do service)
  LOCK/UNLOCK ──────────┤
  "Verificar agora" ────┘
                        │
  WorkManager 15 min ──►│  BatchSenderWorker vira casca fina que chama o MESMO EventUploader
  Watchdog ────────────►┘  (só para quando o service está morto)
```

Com isso:
- O caminho quente **deixa de passar pelo JobScheduler**. Enquanto o service vive, 60 s é determinístico — é um `delay()` num laço de corrotina, não uma sugestão ao SO.
- `ExistingWorkPolicy.KEEP` deixa de existir no caminho quente. Ele passa a governar só o caminho de recuperação, onde coalescer é justamente o comportamento certo.
- `BatchSenderWorker` e o ticker compartilham **exatamente o mesmo código** de drenagem — sem risco de divergirem.

**Concorrência:** `EventUploader` é singleton no `AppContainer` com um `Mutex` de processo, para que um drain do ticker e um drain do worker periódico não se sobreponham. A lease atômica do Room (`EventDao.claimLease`) já impede envio duplicado mesmo se isso acontecesse; o `Mutex` evita o trabalho redundante.

**Custo de bateria:** zero adicional. O laço só roda com a sessão desbloqueada — tela acesa, aparelho na mão, rádio já ativo. Nenhum `WakeLock` novo é necessário, e é por isso que o Doze praticamente não interage com esta cadeia (ver 4.3.4).

**Deriva do ticker:** `CollectorTicker` faz `onTick()` e depois `delay(60_000)` — ou seja, o período real é 60 s **mais** a duração do tick. Com a drenagem agora dentro do tick (DB + HTTP), isso pode virar 62-65 s e acumular deriva ao longo de uma jornada. Correção junto: calcular o próximo vencimento e dormir `deadline - now` (taxa fixa em vez de atraso fixo).

#### 4.3.3 Duas latências diferentes — e as duas precisam ser ditas

Confundir as duas é o que gera promessa quebrada:

- **Cadência do heartbeat ≈ 60 s.** O heartbeat é criado e drenado no mesmo tick, então a latência dele é ~0. O que os 60 s medem aqui é o **intervalo entre heartbeats** — é o sinal de que o laço está vivo.
- **Latência de um evento de atividade: 0 a 60 s, média 30 s.** Um evento de janela criado 5 s depois de um tick espera o próximo. Isso é inerente a qualquer envio em lote com cadência fixa — **o Agent Windows tem exatamente a mesma propriedade** (`UploadWorker.cs:26`), e lá a média é ainda maior por causa dos 10 s extras de drain pelo pipe.

O compromisso correto para o cliente é **"os eventos chegam em até 1 minuto"**, não "a cada 60 s cada evento é enviado".

#### 4.3.4 Onde os 60 s NÃO são alcançáveis — com número real e motivo

Escrito para não haver ambiguidade depois. `p95` medido no servidor.

| Cenário | Caminho efetivo | Latência real | Por quê |
|---|---|---|---|
| Tela desbloqueada, service vivo, rede OK | ticker → `EventUploader` direto | **≤ 60 s** (heartbeat p95 ~62 s; evento 0-60 s, média 30 s) | Laço de corrotina determinístico. **É o cenário da jornada de trabalho** |
| LOCK / UNLOCK | drain imediato | **< 2 s** | Disparo por broadcast, sem esperar tick |
| "Verificar agora" / "Tentar novamente" | drain imediato, ignora cooldown | **< 2 s** | Ação do usuário nunca é enfileirada |
| Sem rede | fila retida em Room, cooldown local | volta da rede **+ ≤ 60 s** | Nada se perde; a lease devolve tudo a `PENDING` |
| Tela bloqueada / aparelho no bolso | ticker parado **de propósito** | n/a | Session gate desliga a coleta. Não há evento para enviar — e é isso que torna o app barato em bateria |
| Service morto, app em bucket `ACTIVE`/`WORKING_SET` | WorkManager periódico | **≤ 15 min** | `MIN_PERIODIC_INTERVAL_MILLIS` = 15 min é constante da plataforma. **Não existe `PeriodicWorkRequest` de 60 s em nenhuma versão do Android** |
| Service morto, bucket `FREQUENT` | WorkManager | **~15-60 min** | App Standby adia jobs conforme o bucket |
| Service morto, bucket `RARE` ou "app em suspensão" da Samsung | WorkManager | **horas — na prática, até o colaborador abrir o app** | Restrição do fabricante, fora do controle do app. É o que A8 (isenção de bateria) e A9 (watchdog) atacam, sem poder eliminar |
| Doze profundo (tela off + parado + sem carga) | janelas de manutenção do SO | **≥ 1 h, crescendo** | Só ocorre com a tela apagada, quando não há coleta acontecendo |

**A frase honesta, que é a que vai para o cliente:**

> Enquanto o colaborador está usando o aparelho, os eventos chegam em **até 1 minuto**. Se o fabricante suspender o aplicativo, a entrega cai para **até 15 minutos**, e em aparelhos com restrição agressiva de bateria pode demorar mais — **sem perda de dado em nenhum caso**. A tela do app mostra ao colaborador quando a otimização de bateria está atrapalhando, com um toque para corrigir.

Os 60 s são reais e garantidos **no cenário que importa** — aparelho em uso durante a jornada. Fora dele, não são, e nenhum app Android consegue garanti-los. Prometer 60 s absolutos seria a promessa que o Marcos não aceita.

#### 4.3.5 Como o @Sam prova — evidência, não teste verde

Teste verde prova que a função foi chamada. **Não prova que o dado chegou.** A prova dos 60 s é uma consulta no banco de staging.

**A métrica.** Não medir `recebido_em − enviadoEm`: isso mistura o relógio do celular com o do servidor e qualquer desvio de relógio contamina o número. Medir o **intervalo entre heartbeats consecutivos, em tempo de servidor** — imune a desvio de relógio e mede exatamente a cadência entregue.

**Protocolo (E-0, cenário obrigatório do Gate 1):**

1. Samsung físico, APK novo, colaborador de teste vinculado.
2. Desbloquear a tela e manter o aparelho em uso normal por **30 minutos ininterruptos** (navegar, trocar de app — comportamento de jornada, não parado).
3. Não tocar em configuração de bateria durante a janela.
4. Ao fim, rodar no banco de staging:

```sql
-- Cadência entregue, medida no relógio do servidor
WITH hb AS (
  SELECT criado_em,
         LAG(criado_em) OVER (ORDER BY criado_em) AS anterior
  FROM batimentos
  WHERE agente_id = :agenteTeste
    AND criado_em >= :inicioJanela
)
SELECT COUNT(*)                                             AS total_heartbeats,
       ROUND(AVG(EXTRACT(EPOCH FROM (criado_em - anterior))))  AS gap_medio_s,
       ROUND(MAX(EXTRACT(EPOCH FROM (criado_em - anterior))))  AS gap_max_s,
       ROUND(PERCENTILE_CONT(0.95) WITHIN GROUP (
             ORDER BY EXTRACT(EPOCH FROM (criado_em - anterior))))  AS gap_p95_s
FROM hb
WHERE anterior IS NOT NULL;
```

**Critérios de aceite — todos obrigatórios:**

| Métrica | Limite | Por quê |
|---|---|---|
| `total_heartbeats` | **≥ 28** em 30 min | Tolera 2 perdas por variação de rede |
| `gap_medio_s` | **≤ 65** | 60 s + tick + rede |
| `gap_p95_s` | **≤ 75** | Margem para variação de rede móvel |
| `gap_max_s` | **≤ 120** | Nenhum buraco maior que 2 ciclos. **Um `gap_max` de 300 s ou mais significa que o blackout do CR-4 continua vivo** |

5. **Segunda consulta — latência de evento real** (não heartbeat), para provar a cadeia inteira e não só o laço:

```sql
SELECT COUNT(*) AS eventos,
       ROUND(MAX(EXTRACT(EPOCH FROM (recebido_em - iniciado_em)))) AS latencia_max_s
FROM eventos_janela
WHERE agente_id = :agenteTeste AND recebido_em >= :inicioJanela;
```
Aceite: `eventos > 0` e `latencia_max_s ≤ 90` (60 s de ciclo + duração da própria janela de atividade).

6. **Evidência anexada à release**, não apenas "passou":
   - saída bruta das duas consultas, colada na nota de release;
   - gravação de tela do aparelho durante a janela, mostrando o relógio;
   - print da Tela 4 ao fim, com "Última verificação: há menos de 1 min".

7. **Cenário de degradação (E-11), medido e não estimado:** repetir com "colocar app em suspensão" ativado na Samsung, por 60 min. Aceite: `gap_max_s ≤ 1000` (≈15 min + folga do JobScheduler) e **zero perda** — a soma de eventos gravados bate com o esperado. **Este é o número que sustenta a frase "até 15 minutos" dita ao cliente.** Se vier pior, o número da frase muda para o que foi medido — não o contrário.

### 4.4 Alternativas descartadas

| Alternativa | Por que foi descartada |
|---|---|
| Mudar `srv-events-node` para aceitar o envelope flat do Android | Fragmenta o contrato de ingestão entre dois clientes. O Windows funciona hoje com o formato aninhado. Custo real seria da @Shuri, para consertar um defeito que é do cliente. Rejeitada. |
| `ExistingWorkPolicy.REPLACE` no `UploadCoordinator` | Vetado explicitamente pela spec de 2026-08-13: cancelaria um upload já aceito pelo servidor antes do ACK local → perda de dado. |
| `ExistingWorkPolicy.APPEND_OR_REPLACE` | Enfileira uma corrente de N workers (um por tick). Com 60 s de cadência, cria backlog de workers e execuções redundantes sobre a mesma fila. |
| Manter `Result.retry()` e só reduzir o backoff para ~30 s | Não resolve o problema: durante *qualquer* janela de backoff, `KEEP` continua descartando LOCK/UNLOCK, que precisam sair na hora. Trata o sintoma. |
| Substituir WorkManager por `AlarmManager` exato | `setExactAndAllowWhileIdle` exige `SCHEDULE_EXACT_ALARM` (permissão especial, revisão de política, péssima justificativa para BYOD) e ainda assim é limitado a 1 disparo por 9 min em Doze. Pior que o que temos. |
| Substituir por FCM push-to-sync | Introduz Google Play Services como dependência dura e um serviço de push a operar, para um app sideloaded, distribuído fora da Play Store. Desproporcional. |
| Abandonar o foreground service e enviar só por WorkManager | Mataria o 60 s e a coleta contínua. É o oposto do pedido. |
| **Manter o WorkManager no caminho quente dos 60 s** (só consertando o `KEEP`) | Era o que a minha primeira versão desta spec propunha, e é insuficiente. Consertar o `KEEP` faz o pedido **parar de ser descartado**, mas o momento da execução continua sendo decisão do JobScheduler — que agrupa jobs e respeita o *standby bucket*. Vira "provavelmente uns 60 s", não 60 s. Requisito de cadência não se constrói sobre discricionariedade do escalonador. Ver §4.3.1. |
| `setExpedited()` no `OneTimeWorkRequest` | Cota de trabalho acelerado é limitada por app e por bucket; esgotada a cota, o WorkManager rebaixa o job para normal — silenciosamente. Cadência que degrada sem aviso é pior que cadência honesta. Além disso, gastar a cota de expedited a cada minuto a deixaria indisponível para o que realmente importa (recuperação após o service morrer). |

---

## 5. Frente 2 — o que a tela de Status deve mostrar

### 5.1 Estado atual

`ui/AgentStatusActivity.kt:40-56` exibe 6 linhas: versão/ambiente, saúde, serviço, sessão, permissões, fila, diagnóstico. Duas delas são inúteis como estão:

- `R.string.status_permissions_local` = *"Permissões: verifique nas configurações do aparelho"* — texto fixo, não olha nada (`res/values/strings.xml:65`).
- `status_queue` = *"Fila local: %1$d eventos"* — jargão de engenheiro num app instalado no **celular pessoal** de um colaborador.

E falta o que mais importa para diagnosticar o incidente atual: **quando foi o último envio bem-sucedido**.

### 5.2 "Pegando o Agent Windows como base" — o resultado da auditoria

Marcos pediu para usar o Windows como referência. Auditei tudo que o Windows expõe ao usuário final. **Copiar o Windows seria um retrocesso.** O que a bandeja do Windows mostra, em sua totalidade:

| Informação | Windows | Evidência |
|---|---|---|
| Versão do agente | ✅ Único dado real de diagnóstico | `SessionWorker/Tray/TrayIconManager.cs:56,132` |
| Estado | ⚠️ 5 estados definidos, **só 2 são atribuídos em runtime** (`Monitoring`, `WaitingLink`). `Error`, `Paused` e `Updating` nunca são setados | `Worker.cs:310` é o único `SetStatus` |
| Usuário | ⚠️ `Environment.UserName` — o usuário do **Windows**, não o colaborador do portal | `TrayIconManager.cs:133` |
| Último envio bem-sucedido | ❌ **Não existe.** `LastUploadStatus` é hardcoded na string `"OK"` | `Service/Pipe/PipeMessageHandler.cs:185` |
| Eventos na fila | ❌ Não exposto. O número existe, mas só vira `LogDebug` — nível que nem aparece no log padrão | `Worker.cs:324-327`, `appsettings.json:4` |
| Enviados hoje | ❌ Não existe nenhum contador | — |
| Estado da conexão | ❌ `BackendReachable` é hardcoded `true` — nunca é testado | `PipeMessageHandler.cs:183` |
| Vínculo / empresa / colaborador | ❌ Não existe na bandeja do V2 (o **legado** tinha, com CNPJ e colaborador mascarado — não foi portado) | `Tray/TrayApplicationContext.cs:220-251` (legado) |
| Ícone muda por estado | ❌ Ícone único | `TrayIconManager.cs:39-41` |

Onde o Windows **de fato** entrega diagnóstico útil é no script `scripts/diagnostico/health-check.ps1`, escondido atrás de "Ferramentas → Verificação de Saúde": ele mostra `Colaborador`, `InstalacaoId`, `Vinculado: Sim/Não` (linhas 104-110), tamanho do `events.db` com alerta acima de 10 MB (171-186) e conectividade com Admin e Events (190-222).

**Consequência para o desenho:** a referência correta não é a bandeja do Windows — é o `health-check.ps1` **trazido para dentro da tela**, mais os requisitos que o Windows não tem porque não é BYOD (transparência LGPD, otimização de bateria, permissões do Android). A tela do Android já nasce melhor que a do Windows; a meta é fechar as lacunas do `health-check`, não replicar a bandeja.

Isso valida diretamente as linhas #3 (último envio), #4 (fila), #7 (permissões — o análogo Android da saúde dos hooks, que no Windows nunca é exposta) e #9 (vínculo).

### 5.3 Lista final proposta — 10 linhas em 4 seções

Cabe no envelope que o @Groot assumiu (~8-10 linhas agrupadas em seções), contando cabeçalho de seção separadamente. Prioridade decrescente dentro de cada seção; itens marcados **(cortável)** são os que o @Groot pode remover se precisar de espaço.

| # | Seção | Linha | Origem do dado | Custo |
|---|---|---|---|---|
| 1 | **Proteção** | Estado geral — 4 estados já definidos (Funcionando / Em andamento / Atenção / Crítico) | `AgentStatusPresenter.healthLabel` — existe | zero |
| 2 | Proteção | Serviço ativo desde `{hora}` / Serviço parado | `isServiceRunning()` existe; `desde` = novo `serviceStartedAt` em prefs | baixo |
| 3 | Proteção | **Última sincronização: há {n} min** | **NOVO** — `DeliveryTelemetry.lastSuccessAt` (Bloco A, task A5) | baixo |
| 4 | **Envio** | Aguardando envio: `{n}` registros *(renomear "Fila local")* | `eventQueue().pendingCount()` — existe | zero |
| 5 | Envio | Enviados hoje: `{n}` **(cortável)** | **NOVO** — contador em prefs, reset por data (task A5) | baixo |
| 6 | Envio | Diagnóstico em linguagem humana — **só aparece quando ≠ OK** | `AgentDiagnosticStore` + novo mapa código→frase | baixo |
| 7 | **Aparelho** | Permissões: `{k}` de 4 concedidas, expansível item a item (Acessibilidade / Acesso a dados de uso / Notificações / Telefone) | `PermissionStepMachine.PermissionState` já modela os 4; a checagem existe em `PermissionDispatchActivity` e precisa ser **extraída** para um `PermissionInspector` reutilizável | médio |
| 8 | Aparelho | Otimização de bateria: **Ativa — pode interromper o monitoramento** / Desativada ✓ + **botão de ação** | `PowerManager.isIgnoringBatteryOptimizations` (existe) + permissão nova (D10) | baixo |
| 9 | Aparelho | Vínculo: `{nome do colaborador}` · `{empresa}` | Empresa: `AgentConfig.nomeEmpresa` — **já existe** (`config/AgentConfig.kt:12`). Nome: `ValidarColaboradorResponse.nomeCompleto` já chega na ativação (`auth/AuthManager.kt:35`); basta **persistir** no `TokenStorage` no sucesso de `vincular()`. **Sem API nova.** | baixo |
| 10 | **Privacidade** | "Este app não acessa mensagens, fotos, contatos, áudio, câmera, localização nem o conteúdo das suas telas." + link expansível "O que é coletado / O que nunca é coletado" | Texto estático derivado de `_shared/lgpd-operacional.md` §1 | zero |

### 5.4 Justificativas das escolhas menos óbvias

- **#3 é a linha mais importante da tela.** É a única que teria dito ao Marcos, na hora, que nada estava saindo. Uma tela de status que não mostra "última sincronização" não é uma tela de status.
- **#10 não é enfeite.** Em BYOD o aparelho é do colaborador. Declarar explicitamente o que o app **não** faz é o argumento de confiança que sustenta a base legal (`lgpd-operacional.md` §1 e §10, "Transparente + baixa visibilidade"). É a linha mais barata e a de maior retorno do produto.
- **#8 é diagnóstico e ação no mesmo lugar.** É a causa provável nº 1 de "o app parou" num Samsung, e o único ponto onde o colaborador pode agir sozinho, sem abrir chamado.
- **"Próximo envio previsto" foi rejeitado como timestamp.** O Android não permite prometer horário de execução. Em vez de um relógio que vai mentir, a linha #3 traz o texto honesto: *"a cada minuto enquanto o aparelho está em uso"*. O Marcos não aceita promessa que não se cumpre — então a tela também não faz.
- **Jargão eliminado.** "Fila local" → "Aguardando envio". Códigos de enum (`HTTP_5XX`) nunca aparecem na tela: viram frases ("Servidor indisponível — tentando de novo automaticamente"). O código continua no arquivo de diagnóstico exportado, para o suporte.

### 5.5 Dependências de outros times

**Nenhuma.** Todas as 10 linhas se resolvem no cliente. Especificamente: `empresa` sai do `config.json` embutido no APK e `nome do colaborador` já vem na resposta de validação — **não há dependência da @Shuri nem do `srv-events-node` para o Bloco B.**

Se o @Groot quiser exibir dados que só o servidor tem (ex.: "seu gestor viu seus dados pela última vez em..."), aí sim vira API nova e vira escopo da @Shuri — e não está proposto aqui.

---

## 6. Frente 3 — exportação de diagnóstico

### 6.1 O que acontece hoje

`log/DiagnosticoExporter.kt:36-80` copia os 3 logs mais recentes para:

```
context.getExternalFilesDir(Environment.DIRECTORY_DOWNLOADS) + "/ManagerAgent-logs"
=  /storage/emulated/0/Android/data/com.trivion.manageragent.staging/files/Download/ManagerAgent-logs/
```

O KDoc do próprio arquivo (linhas 12-14) chama isso de *"diretório público de Downloads"*. **Não é.** É o diretório **externo privado do app**. Desde o Android 11, `Android/data/` é bloqueado para navegação no app Arquivos e na maioria dos gerenciadores de arquivo. E `service/EnviarDiagnosticoReceiver.kt:39-49` mostra um Toast com o **caminho absoluto**, que o usuário não consegue digitar em lugar nenhum.

Marcos está certo: salva num lugar que ninguém acha. Além disso o botão se chama "Enviar diagnóstico" (`res/values/strings.xml:7`) e não envia nada — só copia.

Isso também descumpre o que a política declara: `lgpd-operacional.md` §10 promete *"escape hatch de logs — colaborador pode extrair logs pra pasta pública e enviar manualmente pro gestor"*.

### 6.2 Como o Windows resolve (e o que dá para reaproveitar)

O Windows tem "Ferramentas → Exportar Diagnóstico" na bandeja, que roda `scripts/diagnostico/coletar-diagnostico.ps1`:

- Gera **um `.zip`** em `%TEMP%\diagnostico-manager-agent-<yyyyMMdd-HHmmss>.zip` (linha 183-186).
- **Revela o arquivo para o usuário**: `explorer.exe "/select,$zipPath"` (linha 204). É o passo que o Android não tem análogo — e é exatamente por isso que o share sheet é a resposta certa lá.
- Conteúdo: `sistema.txt`, `config.json` **mascarado** (`chaveAtivacaoEmpresa`, `deviceToken`, `refreshToken` → `***MASCARADO***`; `instalacaoId` truncado em 10 chars — linhas 57-62), `logs/` (3 mais recentes por padrão), `buffer-info.txt` (só metadados do `events.db`), `servico.txt`, `conectividade.txt`, `health-check-result.txt`.
- **Entrega ao suporte é manual.** Não há upload em lugar nenhum; o `ErrorReportMessage` do pipe só vira warning no log e o `IErrorReportService` do SessionWorker está registrado como `NullErrorReportService` (`SessionWorker/Program.cs:162`).

**O que aproveitar:** o formato zip, o `resumo.txt` no espírito do `health-check-result.txt`, e — principalmente — **o mascaramento de segredos, que o Windows já faz e o Android não faz.** Isso é precedente direto para CG-1.

**O que melhorar:** o zip do Windows é infra-cêntrico e **não** inclui contagem de pendentes nem timestamp do último upload — as duas informações que teriam resolvido este incidente em 30 segundos. O `resumo.txt` do Android inclui as duas (temos elas da task A5).

### 6.3 Decisão

**Share sheet (`ACTION_SEND`) como ação primária. MediaStore/Downloads como ação secundária. SAF descartado.**

O objetivo final do arquivo é **chegar no suporte**, não morar no aparelho. O share sheet resolve o problema inteiro em um toque: o usuário escolhe WhatsApp/Gmail/o que usa, e o arquivo sai. Nenhuma caçada a pasta.

Fluxo:

1. Empacota em **um** `.zip`: os N logs mais recentes + um `resumo.txt` legível (versão, ambiente, `instalacaoId`, modelo/fabricante/API, estado das 4 permissões, isenção de bateria, tamanho da fila, último sucesso, últimos códigos de diagnóstico).
2. Grava em `cacheDir/diagnostico/` (limpável pelo sistema, não polui o armazenamento do colaborador).
3. Expõe via o **FileProvider que já existe** (`updater/src/main/AndroidManifest.xml`, authority `${applicationId}.fileprovider`), acrescentando `<cache-path name="diagnostico" path="diagnostico/"/>` em `updater/src/main/res/xml/updater_file_paths.xml`.
4. `Intent.createChooser(ACTION_SEND)` com `FLAG_GRANT_READ_URI_PERMISSION`, assunto pré-preenchido: `Diagnóstico ManagerAgent — {empresa} — {versão} — {data}`.
5. Ação secundária "Salvar em Downloads": `MediaStore.Downloads` (API 29+) / `DIRECTORY_DOWNLOADS` + `MediaScannerConnection` (API 26-28). Feedback mostra o **nome visível ao usuário** (`Downloads/ManagerAgent-diagnostico-2026-08-14.zip`), nunca um caminho absoluto.
6. Renomear o rótulo de "Enviar diagnóstico" para **"Compartilhar diagnóstico"** — o botão passa a fazer o que diz.

**Por que não SAF (`ACTION_CREATE_DOCUMENT`):** o usuário escolhe onde salvar e depois ainda precisa achar e encaminhar. Dois passos a mais para o mesmo destino. Mantido fora de escopo.

### 6.4 LGPD — o que o arquivo contém e o que precisa mudar

O zip contém `filesDir/logs/agent-*.log`, escrito por `log/FileLoggingTree.kt`. Auditei o que é logado:

- Não encontrei nenhuma chamada Timber que emita `tituloJanela`, `urlDominio` ou nome de pacote de terceiro. `ManagerAccessibilityService` loga apenas transições de ciclo de vida do serviço (linhas 51, 186, 190). Os collectors não logam conteúdo.
- ✅ Não há keylog, screenshot, áudio, câmera, GPS nem payload de evento — coerente com `lgpd-operacional.md` §1.

Mas o exportador de hoje é **inseguro por omissão**, não por construção — ele copia o arquivo inteiro, seja lá o que passar a ser escrito nele amanhã. Correções obrigatórias no Bloco C:

- **CG-1** — Filtro de redação na geração do zip: mascarar `Authorization`, `Bearer`, `chaveAtivacao`, cookies, refresh token e qualquer sequência tipo JWT. Aplicado na cópia, não na escrita — defesa em profundidade sobre uma garantia que hoje é só convenção.
- **CG-2** — `resumo.txt` nunca inclui CPF/matrícula. `instalacaoId` (UUID opaco) é suficiente para o suporte correlacionar.
- **CG-3** — Antes de abrir o share sheet, mostrar um diálogo dizendo **o que vai no arquivo** e para onde ele pode ir. Em BYOD o colaborador está prestes a mandar um arquivo do próprio celular para um terceiro; ele precisa consentir sabendo o conteúdo.
- **CG-4** — Teste que prova que um log com `Authorization: Bearer eyJ...` plantado sai mascarado do zip.

---

## 7. Frente 4 — Jornada v2 (@Groot) — **DESTRAVADA**

**Estado em 2026-08-14 (rodada 2):** `/Users/Olimpo/Documents/Athena/.brain/tecnologia/features/agent-android-jornada-v2/` existe com `design-spec.md` (371 linhas) e `mockups.html` (761 linhas). Layout **FECHADO**: Variante A + Refino 1 "Sem cartão", aplicado à Tela 1 (todos os estados) e Tela 2; Telas 3 e 4 congeladas e aprovadas. Único item em aberto: naming do produto.

**Veredito arquitetural: APROVADO com 4 ajustes.** Nada no design exige tecnologia fora da stack atual. Detalhe abaixo.

### 7.1 Viabilidade técnica em Android nativo — item a item

| Elemento do design | Viável? | Como |
|---|---|---|
| Refino 1 "Sem cartão" — conteúdo sem container, centralização vertical real | ✅ | `ConstraintLayout` + `app:layout_constraintVertical_bias="0.5"`. `constraintlayout:2.1.4` já é dependência. O diagnóstico do @Groot em §3.3 está correto: o `layout_gravity` atual dentro de `FrameLayout` é imprevisível com teclado aberto |
| Rodapé de versão ancorado fora do fluxo | ✅ | `app:layout_constraintBottom_toBottomOf="parent"` no root, irmão do bloco centralizado |
| Botão com spinner interno (Tela 2) | ✅ | `MaterialButton` + `app:icon` apontando para um `AnimatedVectorDrawable` + `app:iconGravity="textStart"`. **Recomendo a AVD, não o `CircularProgressIndicator` sobreposto em `FrameLayout`** que o @Groot cita como alternativa — a AVD não exige view extra nem hack de layout |
| Erro inline no campo | ✅ | `TextInputLayout.setError()` — já na dependência, hoje subutilizado. Uso correto do que existe |
| `ConfirmIdentityBottomSheet` (Tela 3) | ✅ | `BottomSheetDialogFragment` de `com.google.android.material:material:1.11.0`, já é dependência. `shapeAppearance` com canto superior 20 dp é suportado |
| Avatar circular com iniciais | ✅ | `TextView` circular com `background` drawable — mais simples que `ShapeableImageView` para iniciais |
| `HeroStatusCard` com cor por estado | ✅ | `MaterialCardView` + `strokeColor`/`cardBackgroundColor` trocados em runtime. `cardElevation=0dp` com `strokeWidth=1dp` é o padrão já usado em `activity_agent_status.xml` |
| Paleta de estado (verde/âmbar/vermelho) | ✅ | Novos recursos em `colors.xml`. Desvio do portal está justificado no §11 do @Groot e eu concordo |
| Tipografia Roboto, sem custom font | ✅ | Fonte padrão do sistema. Zero asset |
| Barra de progresso "Passo N de M" | ✅ | `LinearProgressIndicator` do Material, `4dp` |
| Checklist de permissões com ação inline | ✅ | `LinearLayout` de linhas ou `RecyclerView`. Com 4-5 itens fixos, `LinearLayout` + `include` é mais simples e testável |
| **Ícones Material Symbols** | ⚠️ **Não é dependência** | Não existe artefato Gradle de Material Symbols para Views. Cada ícone precisa virar `VectorDrawable` (`ic_*.xml`) exportado do site (Apache 2.0, sem custo de licença). **Pequeno bloqueio de asset** — ver AJ-4 |
| **Ícone do launcher (escudo, adaptive icon)** | ⚠️ **Asset não produzido** | Pendência #3 do @Groot. Não bloqueia código (o ícone atual segue como placeholder), bloqueia a identidade visual da release |
| Compose | — | **Não usado e não proposto.** O @Groot especificou tudo em Views + Material Components, que é a stack do app. Pergunta 7 da minha rodada anterior: respondida corretamente por omissão |

### 7.2 Ajustes obrigatórios ao design (AJ-1 a AJ-4)

Quatro pontos que o @Groot não tinha como saber, porque dependem de código ou de compliance:

**AJ-1 — 🔴 A tela nova de permissões (§6 do @Groot) NÃO pode ser uma `Activity` nova.**

O @Groot pediu revisão minha aqui (pendência #4 dele) e está certo em pedir. A tela é **aprovada**; a implementação como Activity separada é **rejeitada**.

Motivo: o encadeamento de permissões hoje é um laço fechado dentro de uma única Activity — `executeStep(step)` → `settingsLauncher.launch(intent)` → retorno → `avancarStep()` → `executeStep(próximo)` (`ui/PermissionDispatchActivity.kt:277-360`). Todo o estado vive na Activity: `currentStep`, `oem`, `authManager`, `tokenStorage`, e dois `ActivityResultLauncher` registrados nela. Separar em outra Activity exigiria serializar esse estado, tratar morte de processo no meio da cadeia e re-plumbar os callbacks de resultado — num fluxo que **já é frágil** (o `identificadorPendente` só existe em `TokenStorage` porque essa cadeia pode ser interrompida).

**Implementação correta:** a tela é um segundo *estado de view* dentro da `PermissionDispatchActivity` — `ViewFlipper` ou dois `<include>` alternando visibilidade. `executeStep` passa a **renderizar o checklist e esperar o toque em "Abrir configurações"** antes de disparar a `Intent`, em vez de disparar direto. Zero Activity nova, zero superfície de morte de processo nova, `PermissionStepMachine` intocada — exatamente como o @Groot previu ("não muda a máquina de estados").

**AJ-2 — 🟠 O denominador de "Passo N de M" é dinâmico, e a etapa OEM não pode exibir "concluído".**

Dois detalhes que quebram o checklist se implementados literalmente:

- **M varia por aparelho.** A sequência canônica é ACCESSIBILITY → USAGE_STATS → NOTIFICATION → PHONE → [OEM]. Em Android < 13, `NOTIFICATION` é concedida por padrão e `executeStep` chama `avancarStep()` na hora (`PermissionDispatchActivity.kt:301-308`) — se aparecer no checklist, o passo pisca e some. E `OEM` só existe quando `oem.isProblematico`. Logo M ∈ {3, 4, 5} e precisa ser calculado, não hardcoded.
- **A etapa OEM é inverificável.** `PermissionStepMachine.firstPending` documenta na linha 89: *"não há como verificar programaticamente se o autostart foi ativado"*. O checklist **não pode** mostrar `check_circle` verde para OEM — seria a UI afirmando um fato que o app não conhece. Renderizar como linha de estado neutro ("Recomendado") sem ícone de conclusão.

Isso vira uma função pura nova no `PermissionStepMachine`: `fun checklist(perms, oemProblematico, sdkInt): List<StepStatus>`, testável sem Android — mesma disciplina do resto da classe.

**AJ-3 — 🔴 O identificador do colaborador não existe mais no app quando a Tela 4 abre.**

O Bloco 2 da Tela 4 do @Groot mostra "Identificador vinculado (`joao.***@empresa.com`)". Mas `tokenStorage.clearIdentificadorPendente()` é chamado no sucesso do vínculo (`PermissionDispatchActivity.kt:~347`) — o dado é apagado de propósito.

**Solução, e ela é melhor que a original:** persistir **apenas a forma já mascarada**, nunca o identificador cru. `joao.silva@empresa.com` → grava `joao.***@empresa.com`; matrícula `123456` → grava `12***6`. Assim a tela mostra o que o @Groot desenhou **e** o app deixa de guardar um identificador funcional em repouso — ganho líquido de minimização de dado (LGPD Art. 6º). Mascaramento na escrita, não na leitura.

**AJ-4 — 🟡 Ícones precisam ser produzidos como `VectorDrawable`.**

Material Symbols não tem artefato Gradle para Views. Os ~8 ícones (`shield`, `check_circle`, `warning`, `error`, `sync`, `lock`, `visibility_off`, `wifi_off`) precisam ser exportados como `res/drawable/ic_*.xml`. Trabalho pequeno, mas é pré-requisito de compilação das telas. **Decisão:** @Sam exporta do site oficial (Apache 2.0) e o @Groot valida visualmente. Não bloqueia o início da implementação, bloqueia o fim.

### 7.3 Reconciliação com a minha lista da §5.3 — 3 lacunas

A Tela 4 do @Groot (4 blocos, ~10 linhas) cobre **7 das 10 linhas** que propus, e em dois casos resolve melhor que eu:

| Minha linha (§5.3) | No design do @Groot | Nota |
|---|---|---|
| #1 estado geral | Bloco 1, itens 2-3 (hero) | ✅ |
| #2 serviço | Bloco 4 ("Proteção em tempo real") | ✅ |
| #3 **última sincronização** | Bloco 1, item 4 ("Última verificação: há 2 min") | ✅ **Vocabulário dele vence** — "verificação", não "sincronização" (termo alinhado ao banimento de jargão da §1 dele) |
| #4 aguardando envio | Bloco 4 ("Dados aguardando envio: N") | ✅ |
| #5 enviados hoje | ❌ ausente | **Lacuna 1** — cortável, entra no Bloco 4 se couber |
| #6 diagnóstico em português | Bloco 1, item 3 (subtítulo do hero, dinâmico por causa) | ✅ **Melhor que a minha proposta** — vira o subtítulo do hero em vez de linha própria |
| #7 permissões item a item | Bloco 3 (checklist + "Corrigir" inline) | ✅ **Melhor que a minha proposta** — eu propus agregado expansível, ele resolveu com ação inline |
| #8 **otimização de bateria** | ❌ ausente | **Lacuna 2 — precisa entrar.** É a causa nº 1 de "o app parou" num Samsung (CR-8) |
| #9 vínculo | Bloco 2, itens 6-7 | ✅ e vai além (identificador mascarado — ver AJ-3) |
| #10 **bloco de privacidade LGPD** | ❌ ausente | **Lacuna 3 — obrigatória.** Ver §7.4 |

O próprio @Groot antecipou isso na pendência #5 dele ("quais dados novos entram no Bloco 3/4 depende do que @Tony está definindo") e declarou os Blocos 3 e 4 extensíveis. Então:

- **Lacuna 2 (bateria):** entra como **5ª linha do Bloco 3 "Permissões"**, com a mesma ação inline "Corrigir". Conceitualmente é o mesmo objeto — "o que falta configurar no aparelho" — e tem estado verificável (`isIgnoringBatteryOptimizations`), diferente do autostart OEM, que fica fora de qualquer checklist (AJ-2).
- **Lacuna 1 (enviados hoje):** entra como mais uma linha do Bloco 4, ou cai. Baixa prioridade.

### 7.4 Lacuna 3 — o bloco de privacidade e a notificação persistente

Esta é a única divergência real com o design, e é de compliance, não de gosto.

**O problema.** A §1 do @Groot bane "Monitor", "Monitoramento", "Observador", "Vigilância", "Rastreamento" de **qualquer texto novo**. Para a UI do app, concordo integralmente. Mas existe uma superfície onde esse banimento não pode valer: **a notificação persistente**.

`lgpd-operacional.md` §10 define a notificação como instrumento de compliance, não como elemento de marca: *"Notificação persistente única e não-ocultável — cumpre transparência LGPD (colaborador sempre sabe que está sendo monitorado)"*. É parte do que sustenta a base legal do produto (legítimo interesse + consentimento individual do titular, §3.2). Se o texto passar a dizer "iManager Proteção • protegendo seu aparelho", o colaborador deixa de saber que atividade de trabalho está sendo registrada — e a transparência que a política declara deixa de existir de fato.

Note que o @Groot **não listou** `notification_status_text` ("ManagerAgent • em execução") na tabela de vocabulário da §1 dele. Provavelmente porque a notificação não estava nos 4 mockups. Então isso não é discordância — é um vão entre os dois documentos.

**Minha decisão como TL:** o texto da notificação persistente é **controle de compliance, não superfície de design**. Ele mantém o token de marca novo e mantém a divulgação factual:

```
Título:    iManager Proteção          (= app_name, muda com o naming)
Corpo:     Registrando atividade de trabalho para {nomeEmpresa}
```

"Registrando atividade de trabalho" não usa nenhum termo banido pelo @Groot **e** diz a verdade material. É a formulação que satisfaz os dois documentos. Se o @Groot ou o @Steve quiserem outra redação, a exigência que não se negocia é: **um colaborador que ler só a notificação precisa entender que a atividade dele está sendo registrada para o empregador.**

**E o bloco de privacidade (minha linha #10)** é a contrapartida positiva disso, e é o que torna o reposicionamento "proteção" honesto em vez de eufemístico. Proposta que respeita o layout congelado: **Bloco 5, uma linha sempre visível + expansível**, usando o mesmo padrão colapsável que o @Groot já sancionou no Bloco 4:

> 🔒 **Seus dados pessoais não são acessados.** *(toque para ver o que é e o que não é registrado)*
> Expandido → duas colunas: **O que é registrado** (app em uso, tempo de foco, períodos de inatividade, início/fim de chamadas sem número) · **O que nunca é acessado** (mensagens, fotos, contatos, áudio, câmera, localização, conteúdo de telas, senhas)

Conteúdo derivado literalmente de `lgpd-operacional.md` §1. Custo de implementação: zero lógica, texto estático.

**Se o @Groot preferir não abrir um 5º bloco**, o fallback aceitável é um link no rodapé, junto de "Informações do aplicativo": *"O que registramos e o que nunca acessamos"*. Menos visível, ainda compliant. **O que não é aceitável é não existir.**

### 7.5 As 7 perguntas da rodada anterior — status

| # | Pergunta | Status |
|---|---|---|
| 1 | Identidade "segurança/antivírus": nome, ícone, texto da notificação | 🟡 **PARCIAL.** Nome: proposto ("iManager Proteção"), decisão do @Steve/Marcos pendente. Ícone: direção definida (escudo indigo, sem olho/lupa), **asset não produzido**. Notificação: **não endereçada pelo @Groot** → resolvida por mim na §7.4. ✅ **Ponto positivo:** o @Groot usou "antivírus" apenas como *referência de vocabulário* (§0 dele) e **nenhuma string proposta afirma detecção de vírus, malware ou ameaça**. Ele ficou do lado certo da linha. **Regra dura que fica registrada:** nenhuma string do app pode alegar detecção de vírus/malware/ameaça, porque o app não faz isso. |
| 2 | Onde entra a otimização de bateria | 🟡 **NÃO ENDEREÇADA** → resolvida na §7.3 (5ª linha do Bloco 3). Falta o @Groot confirmar se ela também vira etapa da jornada de configuração ou só aparece na Tela 4 |
| 3 | Permissões item a item ou agregado | ✅ **RESOLVIDA.** Item a item, com ação "Corrigir" inline (Bloco 3). Melhor que a minha proposta. Falta o tratamento visual de permissão **revogada depois do onboarding** — hoje o app não detecta isso; ver R-11 |
| 4 | Bloco de privacidade: fixo, modal ou tela própria | 🔴 **NÃO ENDEREÇADA** → proposta minha na §7.4, aguarda o @Groot |
| 5 | Linha de diagnóstico permanente ou condicional | ✅ **RESOLVIDA**, e melhor: virou o subtítulo dinâmico do hero, sempre presente, com texto que muda por causa |
| 6 | Cabe em ~10 linhas? | ✅ **RESOLVIDA.** 4 blocos, ~10 linhas, blocos declarados extensíveis. Minhas 3 lacunas cabem dentro da estrutura sem redesenho |
| 7 | Views + Material Components, não Compose | ✅ **RESOLVIDA** por omissão — tudo especificado em Views/Material. Nenhuma menção a Compose |

### 7.6 O que ainda bloqueia

| # | Item | Dono | Bloqueia |
|---|---|---|---|
| P-1 | **Nome final do produto** | @Steve + Marcos | `app_name` em `strings.xml`, título da notificação persistente, label do launcher, texto do convite do RH. @Sam usa `iManager Proteção` como placeholder e **isola todas as ocorrências num único recurso** para a troca ser de uma linha |
| P-2 | Redação final da notificação persistente | Marcos (DPO) | §7.4. Proposta minha em cima da mesa |
| P-3 | Placement do bloco de privacidade | @Groot | §7.4. Bloco 5 ou link de rodapé |
| P-4 | Ícone do launcher (adaptive icon, escudo) | @Groot | Identidade visual da release. Não bloqueia código |
| P-5 | Bateria também vira etapa da jornada? | @Groot | Só o fluxo de configuração; a Tela 4 já está resolvida |

Nenhum desses bloqueia o **início** da implementação. P-1 e P-2 bloqueiam o **fechamento** da release.

---

## 8. Tarefas

> Padrão TDD obrigatório em todas: escrever o teste que falha → provar RED → implementar → provar GREEN.
> Cobertura exigida (regra Marcos): **linha ≥ 80%, branch ≥ 95%** em toda classe alterada.
> Comandos: `./gradlew :app:testDebugUnitTest --tests "<filtro>"` para o ciclo, `./gradlew :app:assembleDebug` para compilar.
> **@Sam não commita.** Marcos commita. Mensagens sugeridas em Conventional Commits, português sem acento.

### BLOCO A — BUG (🔴 destravável sozinho, sem @Groot, sem @Steve)

---

**A1 — Envelope `{tipoEvento, dados}` e campos de topo do batch**

**Arquivos:**
- Modify: `app/src/main/kotlin/com/trivion/manageragent/network/dto/EventBatchDtos.kt`
- Modify: `app/src/main/kotlin/com/trivion/manageragent/network/ApiClient.kt`
- Modify: `app/src/main/kotlin/com/trivion/manageragent/sender/BatchSenderWorker.kt`
- Create: `app/src/main/kotlin/com/trivion/manageragent/network/dto/AgentEventEnvelope.kt`
- Test: `app/src/test/kotlin/com/trivion/manageragent/event/model/AgentEventSerializationTest.kt`, `.../network/dto/EventBatchRequestTest.kt` (novo)

**Aceite:** o JSON de `EventBatchRequest` casa byte a byte com `AgentEventBatchSchema`; cada item é `{"tipoEvento":"X","dados":{...}}`; o corpo traz `maquinaId`, `versaoAgente`, `descricaoSo`, `dispositivoTipo`, `enviadoEm`, `instalacaoId`, `eventos`; `agenteId` é removido do corpo (vem do JWT); `enviadoEm` é ISO-8601 com `Z`.

- [ ] Escrever teste que serializa um batch com um `HeartbeatEvent` e um `WindowActivityEvent` e assere o JSON **exato** esperado pelo backend, incluindo o nível `dados`.
- [ ] Escrever teste que assere presença e não-vazio de `maquinaId`, `versaoAgente`, `descricaoSo`, `enviadoEm` (fonte: `DeviceInfo.current()` + `BuildConfig.VERSION_NAME`).
- [ ] Rodar e verificar RED.
- [ ] Implementar o envelope. Recomendação: `AgentEventEnvelope(tipoEvento: String, dados: JsonObject)` construído a partir do `AgentEvent` pelo `Json` já configurado — mantém `AgentEvent` e o schema Room intactos (a fila persistida continua no formato atual, a conversão acontece só na borda de rede).
- [ ] Ajustar `AgentEventSerializationTest` — os asserts de formato flat de hoje passam a descrever a **persistência**, não o wire.
- [ ] Rodar focados + `:app:assembleDebug`. Commit sugerido: `fix: alinhar envelope de eventos Android ao contrato do srv-events-node`.

> ⚠️ Compatibilidade da fila: eventos já persistidos em Room no formato antigo continuam desserializando normalmente, porque o formato de persistência **não muda**. Não há migration.

---

**A2 — Resposta tolerante e header de dedup**

**Arquivos:**
- Modify: `network/dto/EventBatchDtos.kt`, `sender/BatchSenderWorker.kt`, `network/EventsApi.kt`
- Test: `.../sender/BatchSenderWorkerTest.kt`, `.../network/EventBatchResponseTest.kt` (novo)

**Aceite:** a resposta real do backend desserializa sem exceção; nenhum campo obrigatório sem default; `X-Client-Dedup-Batch` e `X-Installation-Id` são enviados; HTTP 208 é tratado como sucesso (ACK da lease).

- [ ] Teste que desserializa o JSON **real** do backend (`{aceito, totalEventos, eventosJanela, ..., motivosIgnorados}`) e prova que não lança.
- [ ] Teste que prova 208 → `UploadResult.Accepted` → `ack()`, não `release()`.
- [ ] Teste que prova que o header de dedup é estável para a mesma lease e diferente entre leases.
- [ ] RED → implementar (`EventBatchResponse` com todos os campos opcionais/`default`; dedup = hash estável de `leaseId` + IDs).
- [ ] GREEN + compile. Commit: `fix: tolerar resposta real de ingestao e enviar header de dedup`.

---

**A3 — Classificação total de falha (fim da coalescência silenciosa)**

**Arquivos:**
- Modify: `diagnostics/AgentDiagnosticCode.kt`, `sender/UploadCoordinator.kt`
- Test: `.../sender/UploadCoordinatorTest.kt`

**Aceite:** `fromHttpStatus` e `fromException` **nunca** retornam `null`; existem `HTTP_400_CONTRATO`, `HTTP_403_BLOQUEADO`, `HTTP_4XX_OUTRO`, `RESPOSTA_INVALIDA`, `SEM_ENVIO_RECENTE`; cada branch tem teste próprio.

- [ ] Reescrever `classifier maps only allowlisted remote failure categories`: `fromHttpStatus(400)` passa a exigir `HTTP_400_CONTRATO` (o assert atual de `null` na linha 37 é o bug, e vira o teste de regressão).
- [ ] Um teste por status: 400, 401, 403, 404, 409, 413, 422, 429, 500, 503 — branch coverage explícito, não cobertura de linha (`feedback_teste_branch_coverage`).
- [ ] Um teste por exceção: `UnknownHostException`, `SSLException`, `SocketTimeoutException`, `IOException`, `SerializationException`, genérica.
- [ ] RED → implementar → GREEN. Commit: `fix: classificar toda falha de envio sem descarte silencioso`.

> ⚠️ Os novos códigos viajam no `dados` do audit event `AGENT_DIAGNOSTIC_TRANSITION` (`di/AppContainer.kt:130-135`), que é mapa livre — **não** é enum de banco. Ainda assim, @Sam **confirma com a @Shuri** antes de codar que `audit_events.dados` aceita valor arbitrário. Regra `feedback_enum_novo_backend_antes`: valor novo no Agent nunca antes do backend suportar.

---

**A4 — Fim do `Result.retry()`; cooldown local explícito**

**Arquivos:**
- Modify: `sender/BatchSenderWorker.kt`, `sender/UploadCoordinator.kt`
- Create: `sender/UploadCooldown.kt`
- Test: `.../sender/BatchSenderWorkerTest.kt`, `.../sender/UploadCooldownTest.kt` (novo)

**Aceite:** o worker nunca retorna `Result.retry()`; falha → `release()` da lease + registro + agendamento de cooldown + `Result.success()`; dentro do cooldown o worker sai imediatamente sem tocar na rede; a progressão é 60 s → 2 min → 5 min → 15 min (teto) e **reseta em qualquer sucesso**. **Ação iniciada pelo usuário ("Verificar agora" / "Tentar novamente" da Tela 4) ignora o cooldown** — `UploadReason.MANUAL` força a tentativa. Backoff automático nunca pode fazer o botão parecer quebrado.

- [ ] Teste: 3 falhas consecutivas produzem cooldowns 60 s, 120 s, 300 s.
- [ ] Teste: sucesso após falha zera o cooldown.
- [ ] Teste: `doWork()` dentro da janela de cooldown retorna `success()` **sem** chamar `send()`.
- [ ] Teste de regressão do bug: com um work anterior finalizado, uma nova `request()` **não** é descartada por `KEEP` (`getWorkInfosForUniqueWork` mostra trabalho novo `ENQUEUED`).
- [ ] Teste: o caminho de backlog (`MAX_BATCHES_PER_RUN` atingido com fila restante) retorna `success()` e o próximo tick continua — sem `retry()`.
- [ ] Teste: `UploadReason.MANUAL` executa `send()` mesmo dentro da janela de cooldown; os demais motivos, não.
- [ ] RED → implementar → GREEN. Commit: `fix: substituir retry do WorkManager por cooldown local no envio`.

---

**A4B — 🔴 Drenagem direta no foreground service — é isto que faz os 60 s entregarem**

> Task mais importante do requisito de 60 s do Marcos. Sem ela, o ciclo apenas **solicita** o envio e o JobScheduler decide quando (ou se) ele acontece — ver §4.3.1, elo 4.

**Arquivos:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/sender/EventUploader.kt`
- Modify: `sender/BatchSenderWorker.kt`, `service/ManagerAgentService.kt`, `service/CollectorTicker.kt`, `service/SessionBroadcastReceiver.kt`, `di/AppContainer.kt`
- Test: `.../sender/EventUploaderTest.kt` (novo), `.../service/CollectorTickerTest.kt`, `.../sender/BatchSenderWorkerTest.kt`

**Aceite:** toda a lógica de drenagem sai do `BatchSenderWorker` para `EventUploader`, sem nenhuma dependência de WorkManager; o ticker de 60 s, o LOCK/UNLOCK e as ações manuais chamam `EventUploader.drain()` **direto** no escopo do service; `BatchSenderWorker` vira casca fina que chama o mesmo `EventUploader`; `EventUploader` é singleton no `AppContainer` com `Mutex` de processo; o `CollectorTicker` passa a ser de **taxa fixa** (calcula o próximo vencimento) em vez de atraso fixo.

- [ ] Teste: `EventUploader.drain()` executa o mesmo fluxo reservar → enviar → ACK que o worker fazia, com as mesmas asserções de lease.
- [ ] Teste: duas chamadas concorrentes de `drain()` não se sobrepõem (a segunda espera o `Mutex`) e não geram envio duplicado.
- [ ] Teste: `BatchSenderWorker.doWork()` delega ao `EventUploader` e não contém mais lógica de fila própria.
- [ ] Teste: o ticker chama `drain()` diretamente — **`UploadCoordinator` não está no caminho quente**. Teste de regressão do elo 4: nenhuma interação com `WorkManager` durante um tick com o service vivo.
- [ ] Teste de taxa fixa: com `onTick` levando 3 s, o intervalo entre inícios de tick permanece 60 s (não vira 63 s) e não acumula deriva em 10 ciclos.
- [ ] Teste: `drain()` respeita o cooldown de A4, exceto em `UploadReason.MANUAL`.
- [ ] **Orçamento de tempo (R-13):** `drain()` para limpo ao atingir **45 s de tempo de parede**, liberando a lease do batch em curso; o restante sai no próximo tick. Teste com envio artificialmente lento prova que o `drain()` retorna dentro do orçamento e que nenhum evento se perde. Sem isso, 5 POSTs lentos sequenciais (5 × 40 s de timeout) atropelam três ciclos de 60 s.
- [ ] RED → implementar → GREEN + `:app:assembleDebug`. Commit sugerido: `fix: drenar eventos direto no service para entregar o ciclo de 60s`.

---

**A5 — Telemetria de entrega e auto-cura do diagnóstico**

**Arquivos:**
- Modify: `diagnostics/AgentDiagnosticStore.kt`, `sender/BatchSenderWorker.kt`, `di/AppContainer.kt`
- Create: `diagnostics/DeliveryTelemetry.kt`
- Test: `.../diagnostics/AgentDiagnosticStoreTest.kt`, `.../diagnostics/DeliveryTelemetryTest.kt` (novo)

**Aceite:** existe `clear()` chamado em todo upload aceito; `DeliveryTelemetry` persiste `lastAttemptAt`, `lastSuccessAt`, `lastFailureCode`, `eventosEnviadosHoje`, `nextAttemptAt`; o contador diário zera na virada de data; `SEM_ENVIO_RECENTE` é derivado quando não há sucesso há > 30 min com service vivo e rede disponível.

- [ ] Teste: `record()` seguido de `clear()` faz `current()` voltar a `null`.
- [ ] Teste: 2 uploads aceitos no mesmo dia somam; no dia seguinte o contador reinicia.
- [ ] Teste: `SEM_ENVIO_RECENTE` aparece após 31 min sem sucesso e some no primeiro sucesso.
- [ ] **CR-12** — implementar teto de fila: acima de `MAX_PENDING_EVENTS = 50_000` (mesmo número do Agent Windows legado), descartar os **mais antigos** e gravar `QUEUE_LIMIT_REACHED`. Teste que prova o descarte por idade e o registro do código. Sem isso o código existe no enum e nunca é usado, e a tabela cresce sem teto no celular pessoal do colaborador.
- [ ] RED → implementar → GREEN. Commit: `feat: registrar telemetria de entrega e curar diagnostico no sucesso`.

---

**A6 — Estado inicial real do `SessionCollectionGate`**

**Arquivos:**
- Modify: `di/AppContainer.kt`, `service/SessionCollectionGate.kt`
- Test: `.../di/AppContainerTest.kt`, `.../collector/UsageStatsReconcileRunnerTest.kt`

**Aceite:** `AppContainer.sessionGate()` nasce com `initiallyUnlocked` derivado de `KeyguardManager.isKeyguardLocked` e `PowerManager.isInteractive`; `UsageStatsReconcileWorker` reconcilia normalmente num processo em que o `ManagerAgentService` **nunca** rodou.

- [ ] Teste de regressão do bug: processo sem Service + aparelho desbloqueado → `UsageStatsReconcileRunner.run()` **reconcilia**. Hoje esse teste falha (no-op silencioso).
- [ ] Teste: aparelho bloqueado → continua no-op (comportamento correto preservado).
- [ ] Teste: `EventQueue.enqueue` de evento não-`Session` funciona em processo sem Service com aparelho desbloqueado.
- [ ] RED → implementar → GREEN. Commit: `fix: derivar estado inicial da sessao do keyguard em vez de assumir bloqueado`.

---

**A7 — `BatchScheduler` reprogramável**

**Arquivos:** Modify `sender/BatchScheduler.kt` · Test `.../sender/BatchSenderWorkerTest.kt`

**Aceite:** `ExistingPeriodicWorkPolicy.UPDATE`; reagendar com intervalo diferente altera o `WorkInfo` do trabalho existente.

- [ ] Teste que agenda 15 min, reagenda 20 min e assere que o intervalo efetivo mudou (substitui o teste atual `nao lanca excecao`, que não prova nada).
- [ ] RED → implementar → GREEN. Commit: `fix: permitir reprogramacao do agendamento periodico de envio`.

---

**A8 — Isenção de otimização de bateria realmente solicitável**

**Arquivos:** Modify `app/src/main/AndroidManifest.xml`, `ui/PermissionDispatchActivity.kt` · Create `oem/BatteryOptimizationInspector.kt` · Test `.../oem/BatteryOptimizationInspectorTest.kt`

**Aceite:** `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` declarada; `ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` abre de fato; falha ao abrir a tela OEM **registra** `PERMISSION_MISSING` em vez de pular em silêncio; `BatteryOptimizationInspector.isExempt()` é consultável pela tela de status.

- [ ] Teste que prova que uma falha de intent OEM registra diagnóstico (hoje `PermissionDispatchActivity.kt:379-383` engole).
- [ ] Teste de `isExempt()` nos dois estados.
- [ ] RED → implementar → GREEN. Commit: `fix: declarar permissao de isencao de bateria e registrar falha de etapa OEM`.

---

**A9 — Watchdog do Service** *(recomendado no Bloco A; se apertar o prazo, pode sair em release próprio)*

**Arquivos:** Modify `service/BootReceiver.kt`, `AndroidManifest.xml`, `sender/BatchSenderWorker.kt` · Create `service/ServiceWatchdog.kt` · Test `.../service/ServiceWatchdogTest.kt`

**Aceite:** `ACTION_MY_PACKAGE_REPLACED` religa o Service após auto-update; o `BatchSenderWorker` verifica e religa o Service quando ele está morto **e** o app está isento de otimização de bateria; sem isenção, não insiste — registra `SERVICE_MORTO_E_REINICIADO` / `PERMISSION_MISSING`.

- [ ] Teste: service morto + isento → tenta `startForegroundService`.
- [ ] Teste: service morto + **não** isento → **não** tenta, registra diagnóstico (evitar `ForegroundServiceStartNotAllowedException` em Android 12+).
- [ ] Teste: service vivo → não faz nada.
- [ ] RED → implementar → GREEN. Commit: `feat: religar service apos update e morte por OEM`.

---

**A10 — Validação end-to-end contra staging**

- [ ] Rodar `:app:testDebugUnitTest` completo; corrigir as regressões introduzidas. As falhas **pré-existentes** registradas no handoff de 2026-08-13 (`UsageStatsCollectorTest` 3, `RefreshInterceptorTest` 2, `OEMIntentFactoryTest` 4) ficam fora — mas `OEMIntentFactoryTest` provavelmente é tocado por A8; se for, conserte.
- [ ] Verificar cobertura via Jacoco: linha ≥ 80%, branch ≥ 95% nas classes alteradas.
- [ ] Gerar APK `stagingRelease` assinado (keystore real em `$HOME/.manager/secrets/manageragent-staging.jks` — **não criar keystore nova**), bump para `1.0.14` / `versionCode 15`.
- [ ] Instalar no Samsung físico e validar os cenários de §9.2.
- [ ] Confirmar no banco de staging que `batimentos` e `eventos_*` receberam registros do `MOBILE_ANDROID`. **Este é o critério que fecha o bug** — teste verde não prova entrega.

### BLOCO B — Tela de Status (🟡 depende de aprovação Marcos; layout depende do @Groot)

**B1 — `PermissionInspector` extraído e testável**
Modify `ui/PermissionDispatchActivity.kt` · Create `ui/PermissionInspector.kt` · Test `.../ui/PermissionInspectorTest.kt`
Aceite: a checagem das 4 permissões sai da Activity e vira classe consultável, devolvendo `PermissionStepMachine.PermissionState`; a Activity passa a usá-la; nenhuma mudança de comportamento no onboarding.

**B2 — Nome do colaborador persistido no vínculo**
Modify `auth/TokenStorage.kt`, `auth/AuthManager.kt` · Test `.../auth/TokenStorageTest.kt`, `.../auth/AuthManagerTest.kt`
Aceite: o `nomeCompleto` que já volta em `validarIdentificador` é gravado no sucesso de `vincular()` e limpo em `clear()`; **nenhuma chamada de API nova**.

**B3 — `AgentStatusPresenter` monta as 10 linhas**
Modify `ui/AgentStatusPresenter.kt` · Test `.../ui/AgentStatusPresenterTest.kt`
Aceite: função pura que recebe telemetria + permissões + bateria + vínculo + fila e devolve a estrutura de 4 seções da §5.3; **nenhum código de enum aparece em texto de UI** — cada `AgentDiagnosticCode` mapeia para uma frase em português; "há {n} min" formatado de forma estável e testável (relógio injetado).

**B4 — Estado de bateria e vínculo mascarado para a Tela 4**
Modify `auth/TokenStorage.kt`, `ui/AgentStatusPresenter.kt` · Create `auth/IdentificadorMasker.kt` · Test `.../auth/IdentificadorMaskerTest.kt`
Aceite (**AJ-3**): o identificador é gravado **já mascarado** no sucesso do vínculo — `joao.silva@empresa.com` → `joao.***@empresa.com`, matrícula `123456` → `12***6`; o identificador cru **nunca** fica em repouso; `BatteryOptimizationInspector` (de A8) é exposto ao presenter como 5ª linha do Bloco 3. Testes cobrem e-mail, matrícula numérica, string curta (≤4 chars) e vazia.

### BLOCO D — Jornada v2 / UI (🟡 design fechado; naming pendente de @Steve/Marcos)

**D0 — Design tokens**
Modify `res/values/colors.xml`, `res/values/strings.xml` · Create `res/values/dimens.xml`, `res/values/type.xml`, `res/drawable/ic_*.xml`
Aceite: as 9 cores de estado da §2.1 do @Groot existem como recursos; escalas de tipografia e espaçamento viram `dimens`/`style`; `color.text.primary` corrigido de `#0F172A` para `#1E293B`; **AJ-4** — os ~8 ícones Material Symbols exportados como `VectorDrawable`; **P-1** — o nome do produto isolado em **um único** recurso `app_name`, referenciado por todas as telas e pela notificação, para a troca ser de uma linha. Sem teste unitário (recursos); valida no build.

**D1 — Tela 1 no Refino 1 "Sem cartão"**
Modify `res/layout/activity_permission_dispatch.xml`, `ui/PermissionDispatchActivity.kt`
Aceite: `FrameLayout` + `layout_gravity` trocados por `ConstraintLayout` + `layout_constraintVertical_bias="0.5"`; sem card, sem chip de empresa (a empresa vai para o subtítulo); ícone `44dp` solto; 2 clusters com respiro desigual (8-10 dp interno / 40 dp entre); rodapé de versão ancorado no root, fora do bloco centralizado. **Testar com teclado aberto** (`adjustResize`) — é o caso que quebra o layout antigo.

**D2 — Tela 2: loading dentro do botão**
Modify `res/layout/activity_permission_dispatch.xml`, `ui/PermissionDispatchActivity.kt` · Create `res/drawable/ic_spinner_anim.xml`
Aceite: `progressBar` e `tvStatus` soltos **removidos**; `MaterialButton` com `app:icon` de `AnimatedVectorDrawable` + `iconGravity=textStart`, texto vira "Confirmando identidade…", botão `disabled` mantendo o indigo (não cinza); campo e checkbox em `alpha 0.5` + `enabled=false`; erro passa a ser inline no campo via `TextInputLayout.setError()` (o `showStatus()` de erro morre junto).

**D3 — Tela 3: bottom sheet de confirmação**
Modify `ui/PermissionDispatchActivity.kt` · Create `ui/ConfirmIdentityBottomSheet.kt`, `res/layout/sheet_confirm_identity.xml` · Test `.../ui/ConfirmIdentityBottomSheetTest.kt`
Aceite: `MaterialAlertDialogBuilder` cru substituído por `BottomSheetDialogFragment`, cantos superiores 20 dp, drag handle, avatar de iniciais 56 dp, "Sim, sou eu" **acima** de "Não, corrigir", `isCancelable = false` preservado. Teste cobre a extração de iniciais (nome simples, composto, um nome só, com espaços extras).

**D4 — Tela de permissões "Configurando proteção"**
Modify `ui/PermissionStepMachine.kt`, `ui/PermissionDispatchActivity.kt`, `res/layout/activity_permission_dispatch.xml` · Create `res/layout/view_permission_checklist.xml` · Test `.../ui/PermissionStepMachineTest.kt`
Aceite (**AJ-1 + AJ-2**): a tela é um **estado de view dentro da `PermissionDispatchActivity`** (`ViewFlipper`/`include` alternado) — **nenhuma `Activity` ou `Fragment` novo**; `executeStep` renderiza o checklist e só dispara a `Intent` no toque de "Abrir configurações"; nova função pura `PermissionStepMachine.checklist(perms, oemProblematico, sdkInt): List<StepStatus>`; **o denominador de "Passo N de M" é calculado** (M ∈ {3,4,5}); `NOTIFICATION` some do checklist em `sdkInt < 33`; a linha OEM **nunca** exibe `check_circle` — é neutra/"Recomendado".
- [ ] Teste: `sdkInt = 30`, OEM não problemático → 3 passos, sem NOTIFICATION
- [ ] Teste: `sdkInt = 34`, OEM não problemático → 4 passos
- [ ] Teste: `sdkInt = 34`, Samsung → 5 passos, OEM em estado neutro
- [ ] Teste: 2 permissões concedidas → índice do passo atual é 3
- [ ] Teste de regressão: `nextStep` continua com o comportamento de hoje — a máquina **não** muda

**D5 — Tela 4 completa (hero + 5 blocos)**
Modify `res/layout/activity_agent_status.xml`, `ui/AgentStatusActivity.kt`, `ui/AgentStatusPresenter.kt` · Create `ui/HeroStatusCard.kt`, `res/layout/view_permission_row.xml`
Aceite: hero com 3 estados (cor + ícone + título + subtítulo dinâmico + timestamp + ação condicional); Bloco 2 Dispositivo (empresa, identificador mascarado, versão — **badge de ambiente só quando `BuildConfig.AMBIENTE != "prod"`**, o que também corrige o vazamento atual); Bloco 3 Permissões com 4 permissões **+ otimização de bateria** (lacuna 2) e "Corrigir" inline; Bloco 4 Atividade colapsável; **Bloco 5 Privacidade** (lacuna 3, §7.4) — sujeito a P-3; ações no rodapé ("Verificar agora", "Exportar diagnóstico para o suporte", "Informações do aplicativo").
Mapeamento dos 4 estados da spec de 2026-08-13 para os 3 do @Groot, para @Sam não ter que adivinhar: `Funcionando corretamente` → **OK**; `Em andamento` → **OK** com subtítulo "Proteção iniciando…"; `Atenção` → **Atenção**; `Crítico` → **Erro**.
- [ ] Teste: cada `AgentDiagnosticCode` mapeia para um estado de hero e uma frase em português — **nenhum nome de enum vaza para a UI**
- [ ] Teste: badge de ambiente ausente quando `AMBIENTE == "prod"`

**D6 — Vocabulário e ações manuais**
Modify `res/values/strings.xml`, `service/NotificationHelper.kt`, `ui/AgentStatusActivity.kt`
Aceite: toda a tabela §1 do @Groot aplicada; **nenhuma ocorrência** de "Agent", "Monitor", "Monitoramento", "Observador", "Vigilância", "Rastreamento" em `strings.xml`; **nenhuma string alega detecção de vírus/malware/ameaça**; a notificação persistente usa a redação de §7.4 (sujeito a P-2); "Verificar agora" e "Tentar novamente" chamam `UploadCoordinator.request(...)` **com bypass do cooldown** (ver A4).
- [ ] Teste de guarda: varredura em `strings.xml` falha o build se qualquer termo banido aparecer — barato e impede regressão de vocabulário em qualquer PR futuro

### BLOCO C — Exportar diagnóstico (🟡 depende de aprovação Marcos; independente do @Groot)

**C1 — Empacotador com redação**
Create `log/DiagnosticoPackager.kt`, `log/LogRedactor.kt` · Test `.../log/DiagnosticoPackagerTest.kt`, `.../log/LogRedactorTest.kt`
Aceite: gera um `.zip` em `cacheDir/diagnostico/` com os N logs + `resumo.txt`; `LogRedactor` mascara `Authorization`, `Bearer <jwt>`, `chaveAtivacao`, cookies e refresh token; **CG-4** — teste com `Authorization: Bearer eyJhbGciOi...` plantado prova saída mascarada; `resumo.txt` não contém CPF/matrícula.

**C2 — Share sheet como ação primária**
Modify `service/EnviarDiagnosticoReceiver.kt`, `ui/AgentStatusActivity.kt`, `updater/src/main/res/xml/updater_file_paths.xml`, `res/values/strings.xml` · Create `log/DiagnosticoSharer.kt` · Test `.../log/DiagnosticoSharerTest.kt`
Aceite: botão "Compartilhar diagnóstico" abre `createChooser(ACTION_SEND)` com URI de FileProvider e `FLAG_GRANT_READ_URI_PERMISSION`; assunto pré-preenchido; **CG-3** — diálogo prévio declara o conteúdo do arquivo; o rótulo "Enviar diagnóstico" deixa de mentir.

**C3 — "Salvar em Downloads" real**
Modify `log/DiagnosticoExporter.kt` · Test `.../log/DiagnosticoExporterTest.kt`
Aceite: usa `MediaStore.Downloads` (API 29+) e `DIRECTORY_DOWNLOADS` + `MediaScannerConnection` (API 26-28); o arquivo aparece na pasta Downloads do app Arquivos; o feedback mostra `Downloads/ManagerAgent-diagnostico-AAAA-MM-DD.zip`, **nunca** caminho absoluto; o KDoc mentiroso de `DiagnosticoExporter.kt:12-14` é corrigido.

---

## 9. Testes obrigatórios

### 9.1 Unit — resumo

| # | Teste | Alvo | Coverage |
|---|---|---|---|
| A1-U1 | Batch serializa no envelope `{tipoEvento, dados}` | `EventBatchRequest` | branch 100% |
| A1-U2 | Campos de topo obrigatórios presentes e não-vazios | `EventBatchRequest` | branch 100% |
| A2-U1 | Resposta real do backend desserializa sem exceção | `EventBatchResponse` | branch 100% |
| A2-U2 | HTTP 208 → ACK, não release | `BatchSenderWorker` | branch 100% |
| A3-U1..U10 | Um por status HTTP; nenhum retorna `null` | `UploadFailureClassifier` | branch 100% |
| A3-U11..U16 | Um por exceção | `UploadFailureClassifier` | branch 100% |
| A4-U1 | Progressão de cooldown 60/120/300 s | `UploadCooldown` | branch 100% |
| A4-U2 | Sucesso zera cooldown | `UploadCooldown` | branch 100% |
| A4-U3 | **Regressão CR-4:** work finalizado não bloqueia nova `request()` | `UploadCoordinator` | branch 100% |
| A5-U1 | `clear()` cura o diagnóstico | `AgentDiagnosticStore` | branch 100% |
| A5-U2 | Contador diário vira na data | `DeliveryTelemetry` | branch 100% |
| A6-U1 | **Regressão CR-7:** reconciliação funciona em processo sem Service | `UsageStatsReconcileRunner` | branch 100% |
| A7-U1 | Reagendamento com intervalo novo tem efeito | `BatchScheduler` | branch 100% |
| A8-U1 | Falha de intent OEM registra diagnóstico | `PermissionDispatchActivity` | branch 100% |
| A9-U1..U3 | Watchdog: isento / não-isento / vivo | `ServiceWatchdog` | branch 100% |
| C1-U1 | **CG-4:** `Bearer <jwt>` sai mascarado do zip | `LogRedactor` | branch 100% |

### 9.2 E2E em aparelho físico (@Natasha — Samsung + 1 não-Samsung)

| # | Cenário | Passos | Esperado |
|---|---|---|---|
| **E-0** | 🔴 **PROVA DOS 60 s** — protocolo completo de §4.3.5 | 30 min de uso contínuo com a tela desbloqueada, depois as 2 consultas SQL | `total_heartbeats ≥ 28`, `gap_medio_s ≤ 65`, `gap_p95_s ≤ 75`, `gap_max_s ≤ 120`, `latencia_max_s ≤ 90`. **Evidência anexada:** saída bruta das consultas + gravação de tela + print da Tela 4. `gap_max_s ≥ 300` = blackout do CR-4 ainda vivo |
| E-1 | Entrega em 60 s (verificação rápida) | Vincular, desbloquear, aguardar 2 min | Registros do `MOBILE_ANDROID` em `batimentos` no banco de staging em ≤ ~2 min |
| E-2 | LOCK/UNLOCK imediato | Bloquear e desbloquear a tela | `eventos_sessao` recebe LOCK e UNLOCK sem esperar o tick |
| E-3 | Offline → online | Modo avião 10 min, coletar, religar | Fila drena sem perda e **sem duplicidade** (dedup via 208) |
| E-4 | **Regressão do bug** | Forçar 400 (apontar para endpoint inválido), esperar 5 min, corrigir | Tela mostra diagnóstico específico; **não** entra em blackout; volta a enviar em ≤ 1 tick após corrigir |
| E-5 | Samsung "colocar para dormir" | Ativar restrição agressiva, aguardar 30 min | Envio retoma em ≤ 15 min via WorkManager; tela sinaliza otimização de bateria ativa |
| E-6 | Processo morto | Force-stop, aguardar o worker periódico | Reconciliação de UsageStats **grava** eventos (prova CR-7 corrigido) |
| E-7 | Auto-update | Instalar 1.0.14 por cima de 1.0.13 | Service religa sem reboot; fila antiga da 1.0.13 drena sem perda |
| E-8 | Compartilhar diagnóstico | Botão → WhatsApp para si mesmo | Zip chega, abre, sem token/segredo, com `resumo.txt` legível |
| E-9 | Salvar em Downloads | Botão secundário → abrir app Arquivos | Arquivo visível em Downloads sem caçar pasta |
| E-10 | Jornada visual completa | Instalação limpa → Tela 1 → Tela 2 → Tela 3 → checklist → Tela 4, com teclado aberto na Tela 1 | Layout do Refino 1 centralizado e estável; "Passo N de M" com M correto para o aparelho; linha OEM sem check verde; nenhum termo banido visível |
| **E-11** | 🔴 **Degradação medida** — Samsung "colocar app em suspensão" ativado, 60 min | Mesmas consultas de E-0 | `gap_max_s ≤ 1000` e **zero perda** (soma de eventos bate com o esperado). **É este número que autoriza a frase "até 15 minutos" dita ao cliente** — se vier pior, a frase muda para o valor medido |
| E-12 | Ações manuais furam o cooldown | Forçar falha até entrar em cooldown, tocar "Tentar novamente" | Tentativa imediata (< 2 s), sem esperar o cooldown expirar |

---

## 10. Ordem de execução — entrega única (decisão do Marcos, 2026-08-14)

**Mudança de sequenciamento:** o @Sam **não** faz o Bloco A isolado. Pega A + B + C + D numa tanque só, com o design do @Groot já fechado. São **25 tasks** em 7 fases.

Isso não elimina a validação antecipada do bug — só a move para **dentro** da tanque. O `SELECT` no staging continua acontecendo cedo, no fim da Fase 2, porque ele valida o pipeline de dados e **não depende de uma única linha de UI**. Entrega única não pode virar validação única: descobrir no fim que o envelope ainda está errado, depois de 6 fases de trabalho, seria repetir exatamente o erro que criou este bug.

```
FASE 1 — Contrato e entrega            A1 → A2 → A3 → A4 → A4B → A5 → A6 → A7  (8 tasks)
                                                        ▲
                                       A4B é a task que faz os 60 s entregarem (§4.3)
FASE 2 — Aparelho                      A8 → A9                                (2 tasks)

  ═══ GATE 1 — VALIDAÇÃO DE ENTREGA + PROVA DOS 60 s (bloqueante, no meio da tanque) ═══
      Nada de UI daqui. Só compila, roda e prova que o dado chega — e em quanto tempo.

FASE 3 — Dados de apresentação         B1 ∥ B2 ∥ B3 → B4                      (4 tasks)
FASE 4 — Diagnóstico compartilhável    C1 → C2 → C3                           (3 tasks)
FASE 5 — Design tokens                 D0                                     (1 task)
FASE 6 — Telas                         D1 → D2 → D3 → D4 → D5 → D6            (6 tasks)

  ═══ GATE 2 — RELEASE ═══

FASE 7 — Validação final               A10                                    (1 task)
```

**Por que esta ordem e não outra:**
- Fases 1-2 são a correção do bug e não tocam em nenhum recurso visual. Ficam primeiro para que o Gate 1 aconteça o quanto antes.
- Fase 3 produz os **dados** que as telas consomem. Se viesse depois das telas, o @Sam montaria layout contra dados inexistentes.
- Fase 4 (export) é lógica pura; D5 só pendura o botão numa tela.
- **Fase 5 antes da Fase 6, sempre.** Cores, dimens, tipografia e ícones são pré-requisito de compilação de qualquer layout novo. Fazer telas antes dos tokens é retrabalho garantido.
- D1→D6 na ordem da jornada do usuário, que também é a ordem de dependência: D4 (checklist) precisa da `PermissionStepMachine.checklist()`, D5 precisa de B1-B4, D6 fecha o vocabulário em cima de tudo que já existe.

### 10.1 Gate 1 — o que precisa estar verde antes do `SELECT` no staging

Este é o gate que o Marcos pediu explicitamente. Ele acontece **no fim da Fase 2**, não no fim da tanque.

**Precondições — todas obrigatórias:**

1. **A1 a A9 implementadas e com teste verde.** Especificamente os testes de regressão que provam cada causa raiz morta:
   - A1-U1/U2 — envelope `{tipoEvento, dados}` + campos de topo (CR-1, CR-2)
   - A2-U1/U2 — resposta real desserializa; 208 → ACK (CR-5, CR-6)
   - A3-U1..U16 — nenhum status ou exceção retorna `null` (CR-3)
   - A4-U3 — work finalizado não bloqueia nova `request()` (CR-4)
   - **A4B — o ticker drena direto, sem `WorkManager` no caminho quente (elo 4 da §4.3.1)**
   - A6-U1 — reconciliação funciona em processo sem Service (CR-7)
2. **Suíte completa verde:** `./gradlew :app:testDebugUnitTest` sem regressão nova. As 9 falhas pré-existentes do handoff de 2026-08-13 (`UsageStatsCollectorTest` 3, `RefreshInterceptorTest` 2, `OEMIntentFactoryTest` 4) são o baseline tolerado — **exceto** `OEMIntentFactoryTest`, que A8 toca e portanto precisa ficar verde.
3. **Cobertura Jacoco** nas classes alteradas: linha ≥ 80%, branch ≥ 95%.
4. **`./gradlew :app:assembleStagingRelease`** gera APK assinado.
5. APK instalado em **Samsung físico**, vinculado a colaborador de teste.
6. **E-1 a E-7 executados** (§9.2) — os cenários de entrega, offline, regressão de blackout, morte de processo e auto-update.

**Então, as duas provas que fecham o gate:**

**Prova 1 — o dado chega.** `SELECT` em `batimentos` e `eventos_*` no banco de staging filtrando `dispositivo_tipo = 'MOBILE_ANDROID'`, confirmando linhas gravadas com timestamps dentro da janela do teste.

**Prova 2 — chega em 60 s.** Cenário **E-0** completo (protocolo de §4.3.5): 30 minutos de uso contínuo, as duas consultas SQL, e os 5 limites atendidos — `total_heartbeats ≥ 28`, `gap_medio_s ≤ 65`, `gap_p95_s ≤ 75`, `gap_max_s ≤ 120`, `latencia_max_s ≤ 90`. Evidência (saída bruta + gravação de tela + print da Tela 4) anexada, não apenas a afirmação de que passou.

**Prova 3 — o número da degradação.** Cenário **E-11**: mesma medição com a suspensão agressiva da Samsung ligada. O resultado **define** o número que vai na comunicação ao cliente. Se medir pior que 15 min, a frase muda para o valor medido — nunca o contrário.

**Se qualquer uma das três falhar, a tanque para.** Não se avança para a Fase 3. Um envelope ainda divergente — ou um `gap_max_s` de 300 s revelando que o blackout continua vivo — descoberto na Fase 6 custa seis fases de retrabalho, e é literalmente a falha de processo que deixou este bug vivo por 13 versões.

**O que NÃO é precondição do Gate 1:** nada de UI. Nenhuma task B, C ou D. A tela de status pode estar exatamente como está hoje, com "Fila local" e jargão. O Gate 1 pergunta duas coisas só: **o dado chega no banco, e em quanto tempo?**

### 10.2 Gate 2 — release

- Fases 3 a 6 completas, suíte verde, cobertura mantida.
- E-8 a E-12 (§9.2) executados.
- **P-1 resolvido** (nome do produto) — sem isso não se congela `app_name`.
- **P-2 resolvido** (redação da notificação, aprovada pelo Marcos como DPO).
- **P-3 resolvido** (placement do bloco de privacidade) ou fallback de rodapé aplicado.
- Revisão visual do @Groot sobre o app rodando, não sobre screenshot.
- Bump para `1.1.0` / `versionCode 15`.

### 10.3 Dependências externas ao @Sam

| Item | Dono | Quando trava |
|---|---|---|
| `audit_events.dados` aceita valor livre (A3) | @Shuri | Início da Fase 1 — confirmação de 5 minutos |
| Nome do produto (P-1) | @Steve + Marcos | Gate 2 |
| Redação da notificação (P-2) | Marcos (DPO) | Gate 2 |
| Placement do bloco de privacidade (P-3) | @Groot | Fase 6, task D5 |
| Ícone do launcher (P-4) | @Groot | Gate 2 (identidade visual) |
| Bateria vira etapa da jornada? (P-5) | @Groot | Fase 6, task D4 |

Nenhuma trava a Fase 1 nem o Gate 1.

---

## 11. Definition of Done

**Gate 1 — entrega de eventos e ciclo de 60 s (bloqueia o resto da tanque):**
- [ ] Testes de §9.1 das tasks A1-A9 passando; branch ≥ 95% nas classes alteradas
- [ ] Nenhuma regressão na suíte existente além das falhas pré-existentes do handoff de 2026-08-13 (`OEMIntentFactoryTest` **precisa** ficar verde — A8 toca nela)
- [ ] `:app:assembleStagingRelease` gera APK assinado
- [ ] E-1 a E-7 validados em Samsung físico, com evidência
- [ ] **Prova 1:** `SELECT` comprova `batimentos` e `eventos_*` gravados com `MOBILE_ANDROID`
- [ ] **Prova 2 (E-0):** os 5 limites de cadência atendidos — `total_heartbeats ≥ 28`, `gap_medio_s ≤ 65`, `gap_p95_s ≤ 75`, `gap_max_s ≤ 120`, `latencia_max_s ≤ 90` — com saída bruta das consultas, gravação de tela e print da Tela 4 anexados
- [ ] **Prova 3 (E-11):** número da degradação sob suspensão da Samsung **medido**, e a frase de comunicação ao cliente ajustada ao valor real
- [ ] `EventUploader` no caminho quente; **`WorkManager` fora do ciclo de 60 s** (A4B)
- [ ] Zero mudança em `manager-srv-events-node`
- [ ] @Shuri confirmou que `audit_events.dados` aceita valor livre

**Gate 2 — release:**
- [ ] B1-B4, C1-C3, D0-D6 com testes; `AgentStatusPresenter` é função pura com relógio injetado
- [ ] CG-1 a CG-4 satisfeitos; KDoc de `DiagnosticoExporter` deixa de chamar diretório privado de "público"
- [ ] **AJ-1** — a tela de permissões é estado de view dentro da `PermissionDispatchActivity`; **nenhuma `Activity`/`Fragment` novo** no fluxo
- [ ] **AJ-2** — "Passo N de M" com M calculado; OEM sem check verde; `NOTIFICATION` ausente em `sdkInt < 33`
- [ ] **AJ-3** — identificador gravado **já mascarado**; identificador cru nunca em repouso
- [ ] **AJ-4** — ícones exportados como `VectorDrawable` e validados pelo @Groot
- [ ] Teste de guarda de vocabulário verde: zero termos banidos em `strings.xml`; nenhuma string alega detecção de vírus/malware/ameaça
- [ ] Badge de ambiente ausente em build `prod`
- [ ] E-8 a E-12 validados em aparelho físico
- [ ] **P-1** (nome do produto), **P-2** (redação da notificação, aprovada pelo Marcos como DPO) e **P-3** (placement do bloco de privacidade) resolvidos
- [ ] Revisão visual do @Groot sobre o app rodando, não sobre screenshot
- [ ] Nenhum commit feito por @Sam — Marcos commita

---

## 12. Riscos e mitigações

| # | Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|---|
| R-1 | Ao corrigir o envelope, a fila acumulada drena de uma vez e duplica dado em produção | Média | Alto | A2 entrega dedup **no mesmo release** que A1. E-3 valida. Nunca soltar A1 sem A2. |
| R-2 | Há mais divergências de contrato além de CR-1/CR-2 que só aparecem com o payload já corrigido | Média | Médio | A10 exige verificação **no banco**, não só HTTP 202. `motivosIgnorados` da resposta 202 precisa ser logado — batch aceito com evento descartado é falso positivo. |
| R-3 | Aparelhos com a 1.0.13 já instalada ficam presos ao agendamento antigo | Alta | Médio | A7 (`UPDATE`). Aparelhos que nunca receberem update ficam presos de qualquer forma — por isso o rollout precisa de `obrigatoria=true` na release (padrão já usado na 1.5.1 do Windows). |
| R-4 | Samsung mata o service mesmo com tudo correto | Alta | Médio | Aceito e declarado: pior caso vira 15 min via WorkManager. A8 + A9 reduzem a frequência. A tela de status (#8) torna visível ao colaborador. **Não prometer 60 s absoluto ao cliente.** |
| R-5 | Watchdog dispara `ForegroundServiceStartNotAllowedException` em Android 12+ | Média | Médio | A9 só tenta religar quando o app está isento de otimização de bateria; caso contrário registra diagnóstico e desiste. Teste A9-U2 cobre. |
| R-6 | Bump futuro para `targetSdk 35` liga o timeout de 6 h/dia do FGS `dataSync` no Android 15 | Média | **Alto** | Hoje `targetSdk = 34`, então não incide. Como o APK é sideloaded (fora da Play Store), não há prazo obrigatório. **Fica registrado: subir para 35 exige spec própria** — `dataSync` deixa de servir para jornada de 8 h. |
| R-7 | `AgentEventSerializationTest` hoje assere o formato errado; mexer nele pode mascarar regressão | Baixa | Médio | A1 mantém os asserts de formato flat como testes de **persistência** e adiciona novos testes de **wire**. As duas camadas ficam cobertas separadamente. |
| R-8 | ~~Blocos B/C travados pelo @Groot~~ | — | — | **RESOLVIDO 2026-08-14** — design entregue e aprovado (§7). Restam P-1 a P-5, nenhum bloqueando o Gate 1. |
| R-10 | Nome do produto não decidido a tempo e o @Sam hardcoda strings espalhadas | Média | Médio | D0 exige que o nome viva num **único** recurso `app_name`, referenciado por telas e notificação. Troca de última hora vira uma linha. |
| R-11 | Permissão revogada depois do onboarding não é detectada — a Tela 4 mostra "concedida" para algo que o usuário desligou | Média | Médio | `PermissionInspector` (B1) é consultado a **cada `onResume`** da Tela 4, não cacheado. Pergunta 3 do @Groot ficou sem resposta visual; @Sam usa o mesmo estilo de linha "Corrigir" do checklist. |
| R-12 | **Os 60 s passam em E-0 no aparelho de teste e degradam em campo** (aparelho mais antigo, bucket pior, rede instável) | Média | Alto | E-0 roda em **Samsung + 1 não-Samsung** (§9.2). A telemetria de entrega (A5) fica no aparelho e no `resumo.txt` do diagnóstico, então qualquer degradação em campo é diagnosticável sem acesso remoto. A frase ao cliente usa o número medido em E-11, não o do melhor caso. |
| R-13 | Rede lenta faz um `drain()` ultrapassar 60 s e atropelar o ciclo seguinte | Média | Médio | Os timeouts do OkHttp já estão corretos e **não** são o problema: `ApiClient.kt:50-53` define 10 s connect + 30 s read/write, então uma requisição isolada não passa de ~40 s. O risco é **cumulativo**: `MAX_BATCHES_PER_RUN = 5` faz até 5 POSTs sequenciais no mesmo `drain()`, e 5 × 40 s = 200 s — três ciclos perdidos, com os ticks seguintes empilhando no `Mutex`. **Mitigação (aceite de A4B): orçamento de tempo de parede de 45 s por `drain()`** — atingido o orçamento, o drain encerra limpo (lease liberada) e o restante da fila sai no próximo tick. Isso mantém o ciclo de 60 s honesto sob rede ruim, em vez de deixá-lo derreter em silêncio. É a versão correta do W-5 do Windows, onde o timeout de 100 s é maior que o intervalo de 60 s e ninguém percebeu. |
| R-9 | A causa raiz encontrada indica que ninguém testou a integração de ponta a ponta | Alta | Alto | **Ação de processo, não de código:** nenhum APK Android vai a piloto sem o gate "evento visível no banco". Registrar no DoD de toda release Android daqui para frente. |

---

## 13. Achados colaterais no Agent Windows — FORA DE ESCOPO desta spec

A auditoria comparativa do Agent Windows V2 (feita para responder à pergunta de paridade) encontrou defeitos latentes **em produção no desktop**. Não são escopo do @Sam nem desta spec, mas não podem ficar sem registro. Recomendo spec própria e @Bucky como executor, **se e quando o Marcos priorizar**.

| # | Achado | Evidência | Severidade |
|---|---|---|---|
| W-1 | **Head-of-line blocking**: o SELECT de pendentes ordena por `session_id, windows_user, id` com `LIMIT`. Um evento rejeitado deterministicamente pelo backend (ex.: 400) volta a ser o primeiro da fila em todo ciclo e **os eventos atrás dele nunca drenam**. Não há contador de tentativas, dead-letter nem coluna `attempts` no schema | `Service/Storage/SqliteEventBuffer.cs:81-91,275-277` | **Alta** — é a versão Windows do mesmo tipo de falha que travou o Android |
| W-2 | **`events.db` sem teto**: `CleanupAsync` só apaga registros com `uploaded = 1`. Eventos pendentes nunca expiram nem são limitados. Backend fora do ar por semanas = crescimento ilimitado em disco. O legado tinha `MaxPendingEvents = 50_000` / `CleanupDays = 7`, **não portado para o V2** | `Service/Storage/SqliteEventBuffer.cs:361-385` vs. `Tray/Storage/SqliteEventBuffer.cs:470-471` | Média |
| W-3 | **Flush em LOCK/UNLOCK é código morto**: o sinal é emitido, ninguém consome no V2. O log afirma "disparando flush imediato" e nada acontece | `SessionWorker/Capture/SessionEventService.cs:99-104`; nenhum `WaitAsync` no SessionWorker | Média — log enganoso durante diagnóstico |
| W-4 | **Janela de perda de eventos no drain**: `DrainAll()` **deleta** as linhas do SQLite antes do envio pelo pipe; `PipeClient.SendAsync` descarta em silêncio se o pipe cair. Crash/kill entre o delete e o envio = perda definitiva | `SessionWorker/Pipe/AutonomousBuffer.cs:196-201`, `Pipe/PipeClient.cs:213-220` | Média |
| W-5 | **Timeout de HTTP (100 s, default) maior que o intervalo (60 s)**: com backend pendurado, o período efetivo vira ~160 s. `AddHttpClient<HttpEventUploader>()` não define timeout | `Service/Program.cs:152`, `Upload/UploadWorker.cs:72,85` | Baixa |
| W-6 | **Telemetria falsa na bandeja**: `LastUploadStatus` hardcoded `"OK"`, `BackendReachable` hardcoded `true`, `EventosPendentes` hardcoded `0`. Três valores que parecem diagnóstico e não são | `Service/Pipe/PipeMessageHandler.cs:183,185`, `SessionWorker/Capture/HeartbeatService.cs:42` | Média — **induz a erro exatamente durante um incidente** |
| W-7 | **`appsettings.json` decorativo**: `IntervaloUploadSegundos` não é bindado no Service nem no SessionWorker. Mudar o arquivo não muda nada; só recompilar muda | `Service/Program.cs` (sem `Configure<>`), `SessionWorker/Program.cs:91-94` | Baixa — mas é armadilha operacional |
| W-8 | **Estados de tray nunca atribuídos**: `Error`, `Paused` e `Updating` existem no enum e nunca são setados. O usuário do Windows **nunca vê "Erro"**, mesmo com o agente quebrado | `SessionWorker/Worker.cs:310` é o único `SetStatus` | Média |

W-1, W-2 e W-6 juntos descrevem, no Windows, o mesmo padrão que quebrou o Android: **falha permanente + ausência de sinal + fila sem teto**. Vale a pena o Marcos decidir se quer olhar isso antes de escalar a base de clientes.

## 14. Referências

- `.brain/tecnologia/specs/2026-08-13-agent-android-entrega-confiavel-health-design.md` — arquitetura de 3 camadas, fila com lease, allowlist de diagnóstico (**esta spec estende, não substitui**)
- `.brain/tecnologia/plans/2026-08-13-agent-android-entrega-confiavel-health.md` — Tasks 1-4, restrição "sem endpoint de health"
- `.brain/tecnologia/registro/2026-08-13-handoff-agent-android-sessao-ui.md` — pendências abertas: `goAsync()` síncrono no `SessionBroadcastReceiver`, 4 ajustes de UI não commitados, 9 testes falhando pré-existentes
- `.brain/tecnologia/specs/2026-08-08-agent-android-byod-mvp-design.md` — D1-D19; D7/D8 **revertidas** desde 2026-08-13
- `.brain/tecnologia/specs/2026-08-08-srv-events-node-suporte-agent-android.md` — contrato de ingestão
- `.brain/tecnologia/features/agent-android-jornada-v2/design-spec.md` + `mockups.html` — @Groot, layout fechado (Variante A + Refino 1 "Sem cartão"); vocabulário §1, tokens §2, telas §3-§8, componentes §9
- `.brain/_shared/lgpd-operacional.md` §1, §10 — o que os Agents fazem e nunca fazem; postura BYOD; notificação persistente como instrumento de transparência (base da §7.4)
- Memórias aplicadas: `feedback_agent_spec_obrigatoria`, `feedback_enum_sem_coalescencia_silenciosa`, `feedback_enum_novo_backend_antes`, `feedback_teste_branch_coverage`, `feedback_nao_commitar_execucao_subagent`, `feedback_subagent_sem_branch_ops`, `feedback_agent_windows_logs_sem_grafana`
