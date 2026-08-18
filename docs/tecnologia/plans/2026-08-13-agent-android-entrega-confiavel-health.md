# Agent Android — Eventos Confiáveis e Observabilidade Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Entregar sincronização Android a cada 60 s em sessão ativa, flush de lock/unlock sem duplicidade, tela de diagnóstico segura e observabilidade baseada nos eventos existentes.

**Architecture:** A Room é a fonte de verdade; um coordinator único transforma todos os gatilhos em uploads serializados por lease. O `POST /api/agent/events`, heartbeats, eventos de sessão, auditorias e métricas existentes são a única fonte remota de saúde; não há endpoint de health novo.

**Tech Stack:** Kotlin/AndroidX/Room/WorkManager/Retrofit, `srv-events-node` existente, auditoria existente, Grafana/PostgreSQL/Prometheus/Loki.

**Spec:** `.brain/tecnologia/specs/2026-08-13-agent-android-entrega-confiavel-health-design.md`

## Global Constraints

- Não criar, publicar ou chamar `POST /api/agent/health` nem tabela de health.
- `POST /api/agent/events` e eventos existentes são o sinal operacional canônico.
- Foreground service gera heartbeat e solicita upload a cada 60 s somente em sessão desbloqueada.
- `LOCK`/`UNLOCK` são persistidos antes de solicitar upload imediato; bloqueio pausa ticker e collectors.
- Fila usa lease atômica; ACK só remove IDs da lease correspondente; nunca usar `ExistingWorkPolicy.REPLACE`.
- Erros remotos usam auditoria/telemetria existentes e categorias allowlistadas; sem token, chave, payload, corpo HTTP, stack trace ou identificador.
- Não modificar os worktrees originais ou alterações pré-existentes do usuário.

---

## Task 1: Android — fila serializada por lease

**Status:** implementação concluída no commit local `ac692b0`; pendente revisão independente.

**Files:**
- Modify: `event/EventEntity.kt`, `event/EventDao.kt`, `event/EventDatabase.kt`, `event/EventQueue.kt`, `sender/BatchSenderWorker.kt`
- Create: `sender/UploadCoordinator.kt`
- Test: `EventQueueTest.kt`, `BatchSenderWorkerTest.kt`

**Acceptance:** duas reservas concorrentes não compartilham IDs; lease vencida é recuperada; ACK/NACK só afetam a lease; worker drena até cinco batches; dez solicitações produzem um trabalho único `ManagerAgent.UploadNow` com `KEEP` e rede obrigatória.

## Task 2: Android — sessão, heartbeat de 60 s e auditorias seguras

**Files:**
- Modify: `service/ManagerAgentService.kt`, `service/SessionBroadcastReceiver.kt`, `service/CollectorTicker.kt`, `sender/BatchScheduler.kt`, `collector/HeartbeatScheduler.kt`, `collector/HeartbeatWorker.kt`, `sender/BatchSenderWorker.kt`, `log/AuditReporter.kt`
- Create: `diagnostics/AgentDiagnosticStore.kt`, `diagnostics/AgentDiagnosticCode.kt`
- Test: `SessionBroadcastReceiverTest.kt`, `AgentDiagnosticStoreTest.kt`, `UploadCoordinatorTest.kt`

**Interfaces:**
- `AgentDiagnosticStore.record(code: AgentDiagnosticCode, at: Instant)` persists only an enum/time/count.
- `UploadCoordinator.request(reason: UploadReason): Unit` is called after a successful session-event enqueue.

- [ ] Write failing tests proving: lock enqueues then requests and stops ticker; unlock enqueues then requests and starts ticker; ticker requests only unlocked; failed upload records an allowlisted diagnostic and schedules retry.
- [ ] Run `./gradlew testStagingReleaseUnitTest --tests '*SessionBroadcastReceiverTest*' --tests '*AgentDiagnosticStoreTest*' --tests '*UploadCoordinatorTest*'` and verify RED.
- [ ] Implement: use `CollectorTicker(60_000L)` in the foreground service; each tick enqueues existing `HeartbeatEvent` then calls coordinator; remove competing heartbeat periodic work; retain 15-minute batch work only for recovery; map network/HTTP/token/queue/permission failures to the enum and report existing audit events only on state transitions.
- [ ] Run focused tests plus `:app:compileStagingReleaseKotlin`; commit `feat: sincronizar eventos Android por sessao`.

## Task 3: Android — vínculo, diagnóstico local e design de status

**Files:**
- Create: `ui/AgentStatusActivity.kt`, `ui/VincularDialogFragment.kt`, `res/layout/activity_agent_status.xml`, `res/layout/dialog_vincular.xml`, `res/drawable/ic_stat_manager_agent.xml`
- Modify: `ui/PermissionDispatchActivity.kt`, `service/NotificationHelper.kt`, `AndroidManifest.xml`, `res/layout/activity_permission_dispatch.xml`, `res/values/strings.xml`, `res/values/colors.xml`, `res/drawable/ic_launcher_foreground.xml`
- Remove/modify: raw body/stack diagnostic display paths in `network/ActivationNetworkDiagnostics.kt`, `auth/ValidationFailureDiagnostic.kt`
- Test: `PermissionDispatchActivityTest.kt`, `AgentStatusActivityTest.kt`, `LGPDContentFilterTest.kt`

**Acceptance:** confirmação usa modal Material pronto/processando/falha; modal de vínculo não fecha enquanto chamada está ativa; sucesso inicia serviço antes de abrir status; launcher permanece habilitado; status mostra serviço/sessão/permissões/fila/último diagnóstico sem segredos; notificação abre status; ícones seguem badge minimalista de 40–42 dp e notificação monocromática.

- [ ] Write failing UI/redaction tests for the stated acceptance cases.
- [ ] Run focused tests to verify current finish/launcher-disable and raw diagnostic behavior fail those assertions.
- [ ] Implement the dialog, status activity, notification action, resource design and diagnostic redaction.
- [ ] Run `./gradlew testStagingReleaseUnitTest :app:assembleStagingRelease`; inspect merged manifest and release APK; commit `feat: exibir diagnostico seguro do Agent Android`.

## Task 4: Grafana — observabilidade a partir de eventos existentes

**Files:**
- Create: `manager-srv-monitoring/grafana/dashboards/agent-android-operacao.json`
- Create: `.brain/tecnologia/testes/2026-08-13-agent-android-operacao-qa.md`

**Acceptance:** dashboard consulta apenas `batimentos`, `eventos_sessao`, `audit_events` e métricas/Loki existentes; não depende de tabela/API nova; não contém credencial; painéis e alertas distinguem sessão bloqueada de ausência inesperada de heartbeat.

- [ ] Write QA matrix for 60 s ativo, lock flush, sem ticker bloqueado, unlock flush, offline/reconexão, duplicidade, permissão removida, erro 401 e falha de ingestão.
- [ ] Create panels: idade do último heartbeat Android, ativos sem heartbeat após unlock, LOCK/UNLOCK, versão, 4xx/5xx/latência e auditorias categorizadas.
- [ ] Add alerts: ausência após unlock >5 min, ingestão 5xx, token/auditoria crítica, fila limitada e permissão ausente.
- [ ] Run `jq empty grafana/dashboards/agent-android-operacao.json && docker compose config`; commit `feat: observar operacao do Agent Android`.

## Final staging gate

- [ ] Regressão Android unitária e APK staging assinado passam.
- [ ] Samsung físico confirma 60 s ativo, LOCK/UNLOCK imediato e recuperação offline sem IDs duplicados.
- [ ] `srv-events-node` registra heartbeats/eventos de sessão normalmente; nenhuma rota/tabela health nova é deployada.
- [ ] Grafana mostra os sinais existentes e alertas com dados de staging.
- [ ] QA aprova antes de publicar a versão do APK.
