> **STATUS:** ATIVO
> **DATA:** 2026-08-04
> **DONO:** @Tony
> **REVISOR:** @Tony
> **COMPLEMENTA:** `GUIA-RELEASE.md` (passos operacionais de build/deploy)

# Regras de Release do Agent Desktop

Documento obrigatorio de precondicoes e regras inegociaveis antes de qualquer release do Agent. Diferente do `GUIA-RELEASE.md` (passos operacionais), este define as REGRAS.

---

## 1. Regras inegociaveis

### 1.1 Nao quebrar cliente atual em prod

Versao dominante em prod hoje (v1.3.13) precisa continuar funcionando durante rollout de qualquer nova versao. Nenhuma mudanca server-side pode tornar v1.3.x invalido — usar sempre:
- Campos NULL-able nas migrations
- `@JsonIgnoreProperties(ignoreUnknown=true)` nos DTOs (ja em prod)
- Aliases legados nos enums (via `fromString` custom)

### 1.2 Nao quebrar instalacao, iniciacao e auto-update

Regressao em qualquer um dos 3 fluxos = blocker de release. Cada mudanca precisa de teste de regressao correspondente (spec deve nomear os testes por ID).

### 1.3 Toda mudanca no Agent exige spec formal

Memoria `feedback_agent_spec_obrigatoria.md`. Sem excecao. Investigacao exaustiva antes de codigo. Aprovacao Marcos obrigatoria antes de qualquer PR.

### 1.4 Enum novo backend antes do Agent — REGRA DURA

Memoria `feedback_enum_novo_backend_antes.md`. Ordem obrigatoria:

1. **Backend Java** expande enum + expande CHECK constraint via migration Flyway
2. **Backend Java** deploy em prod (todos os pods rodando versao nova)
3. **Agent** somente entao comeca a enviar o valor novo

**Motivacao:** `TypingPattern.valueOf()`, `SessionEventType.valueOf()` etc caem em catch (do handler) e defaultam silenciosamente pra valor legado — dado corrompido sem trace. Ordem invertida = perda de dados durante dias entre release Agent e release backend.

**Excecao:** valores de enum que ja sao aceitos como alias legado (ex: `UserStatus.fromString` aceita `ONLINE→ATIVO`) — nesses o Agent pode enviar sem sync previa porque backend traduz.

**Aplicavel a:** todos os handlers do `AgentEventIngestionService` que usam `TypingPattern`, `SessionEventType`, e qualquer enum futuro (`RenderConfianca`, etc). Vale para srv-events, srv-admin, srv-portal-node (Shuri precisa espelhar quando estiver com endpoint migrado).

### 1.5 Logging estruturado obrigatorio

Cada operacao nova precisa de `_logger.LogInformation(...)` ou `_logger.LogWarning(...)` estruturado no padrao Serilog do time:
- Placeholders capitalizados (`{PropertyName}`)
- Mensagem em portugues
- Contexto minimo suficiente para diagnostico remoto sem repro

Exemplo:
```csharp
_logger.LogInformation("HardwareFingerprint computado. Hash={Hash} InstalacaoId={InstalacaoId}",
    hash, instalacaoId);
```

### 1.6 Nao commitar em execucao subagent

Subagents (Explore, general-purpose, claude generico) NAO commitam nem fazem push. Marcos aprova cada commit manualmente. Ver `feedback_nao_commitar_execucao_subagent.md`.

### 1.7 Codigo duplicado entre Service e SessionWorker

Mudanca em `IdleMonitorService`, `UserStatusManager`, `SqliteEventBuffer`, `HttpEventUploader`, `AgentLinkService` DEVE ser aplicada em AMBOS os projetos (Service + SessionWorker legacy Tray) ate MVP3 unificacao. Spec deve listar todos os arquivos alterados. Sem isso, drift silencioso.

---

## 2. Precondicoes por release

Antes de iniciar qualquer PR de codigo:

- [ ] Spec formal em `.brain/tecnologia/specs/YYYY-MM-DD-agent-<eixo>-design.md` aprovada por Marcos
- [ ] Investigacao do @Bucky no codigo real registrada (nao ancorar em documentacao potencialmente desatualizada)
- [ ] Impacto no backend Java catalogado por @Thor (novos campos, enum expansion, migrations)
- [ ] Timing com @Shuri (baseline Drizzle) validado — Node repo nao pode acumular drift
- [ ] Plano de testes @Natasha com cenarios E2E em VM Windows

---

## 3. Ordem de deploy end-to-end

Sequencia obrigatoria para qualquer release que introduz novos campos ou valores de enum:

1. **Backend Java** — migration Flyway + entity + enum expandido + handler
2. **Backend Java** deploy staging + validacao curl (payload novo + payload antigo, ambos devem funcionar)
3. **@Shuri** — resync Drizzle read-only (`pnpm db:introspect`) apos migration aplicada em staging
4. **Backend Java** deploy prod
5. **Agent** — implementacao + testes + build (segue `GUIA-RELEASE.md` operacional)
6. **Agent** publicado em `versoes_agente` como **opcional** (`obrigatoria=false`) inicialmente
7. **QA** cenarios E2E em VM Windows (Natasha)
8. **Marketing rollout** — se validar em 20% dos clientes por 7 dias sem incidente, `obrigatoria=true`
9. **srv-ia** (@Jarvis) — se aplicavel, prompt consome novos campos como contexto + rerodada 30/30 calibracao

Nao pular etapas. Nao paralelizar Agent v1.N.0 com backend se envolve enum novo.

---

## 4. Checklist de PR

### PR do backend Java (Thor)

- [ ] Migration Flyway nomeada `V##__descricao_snake_case.sql`
- [ ] Entity JPA atualizada com getters/setters + JPA annotations
- [ ] Handler no `AgentEventIngestionService` (srv-events) OU `AgenteVinculacaoService` (srv-admin)
- [ ] Repository query se aplicavel
- [ ] Enum Java expandido com `fromString` que aceita legados + defaulta seguro pra desconhecidos
- [ ] Testes unit cobrindo cada `case` novo do switch/handler (branch coverage ~100%)
- [ ] Teste que envia payload com campos novos + sem campos novos (ambos devem passar)
- [ ] Docs `.brain/_shared/banco-dados.md` atualizada
- [ ] Docs `.brain/tecnologia/services/srv-events/README.md` (ou srv-admin) atualizada

### PR do Agent C# (Bucky)

- [ ] `.csproj` bump versao (Watchdog, SessionWorker, Configurator, Tray, Service, Shared — TODOS)
- [ ] Serilog `_logger.LogInformation` em cada operacao nova
- [ ] Codigo espelhado em Service + SessionWorker se aplicavel (ate MVP3)
- [ ] Testes xUnit + NSubstitute com branch coverage ~100%
- [ ] Fallback gracioso pra dados corrompidos em SQLite local
- [ ] Nenhum novo I/O bloqueante no startup principal (fazer async ou lazy)
- [ ] `HANDOFF-YYYY-MM-DD.md` no repo Agent com notas de deploy
- [ ] Publicado em `versoes_agente` como opcional inicialmente

### PR do frontend (Peter)

- [ ] Testes E2E Playwright para cenarios de interface novos
- [ ] Design system respeitado (cards brancos, border #e5e7eb, radius 14px)
- [ ] Sem regressao em bundle size (verificar via `ng build --stats-json`)

---

## 5. Como identificar versao do Agent em prod

```sql
SELECT versao_agente, COUNT(*) AS total
FROM agentes
WHERE desvinculado_em IS NULL AND ultimo_heartbeat_em > NOW() - INTERVAL '7 days'
GROUP BY versao_agente
ORDER BY total DESC;
```

Se >5% dos agentes ativos ainda estao na versao anterior a v1.N-1.0, NAO publicar v1.N.0 como obrigatoria — pode quebrar clientes.

### 5.1 Rollout gradual formal (v2 pos-BAA — 2026-08-07)

Substitui a regra anterior "20%×7 dias". Vige a partir do BAA (spec `.brain/tecnologia/specs/2026-08-07-blindagem-auto-update-agent-design.md` + ADR-008 em `Backend/manager-srv-portal/docs/adr/ADR-008-rollout-gradual-agent.md`).

Toda nova versao publicada em `versoes_agente` DEVE seguir estas 5 etapas em ordem, com **telemetria zero** como criterio de avanco:

| Etapa | Config no banco | Duracao min | Criterio p/ avancar |
|---|---|---|---|
| **1. Canario** | `canary_only=true, rollout_percent=100, obrigatoria=false` + 1-3 empresas `canary=true` | 48h | 0 `UPDATE_STALE_RECOVERY` + 0 `UPDATE_ROLLBACK_TRIGGERED` + 0 SOS |
| **2. Opt-in 10%** | `canary_only=false, rollout_percent=10, obrigatoria=false` | 3 dias | Mesmo criterio |
| **3. Opt-in 50%** | `rollout_percent=50` | 3 dias | Mesmo criterio |
| **4. Opt-in 100%** | `rollout_percent=100, obrigatoria=false` | 7 dias | Mesmo criterio |
| **5. Obrigatorio** | `obrigatoria=true` | — | Telemetria zero + fleet >=95% na versao nova |

**Kill switch em qualquer etapa:**
```sql
UPDATE versoes_agente SET pausada = true WHERE versao = 'X.Y.Z';
```
No proximo ciclo de check (6h) de cada cliente, para de puxar. Reversivel via `pausada = false`.

### 5.2 Teste E2E de rollback obrigatorio (v2 pos-BAA — 2026-08-07)

Toda spec/PR que altere `UpdateApplier`, `UpdateCheckerWorker`, `Watchdog/**`, `Shared/Update/**` ou `Shared/Config/UpdateThresholds.cs` DEVE incluir teste E2E do rollback: **publicar v-nova que crasha no startup + validar que Watchdog reverte pra v-anterior automaticamente**.

Sem esse teste passando no CI, PR nao entra na `main`. Cenario referencia: `e1-alt-rollback-crash.ps1` (BAA Fase 4 Task 4.1).

### 5.3 CI obrigatorio no repo Agent (v2 pos-BAA — 2026-08-07)

- [ ] Workflow `tests-linux.yml` verde (ubuntu-latest, gratis unlimited em private)
- [ ] Workflow `unit-tests.yml` verde (windows-latest, roda apenas em PR pra main/staging pra economizar minutos)
- [ ] Branch protection rule na `main` e `staging` exige ambos como status checks
- [ ] Branch coverage >=95% em componentes criticos (`UpdateChecker*`, `UpdateApplier`, `Watchdog/*`, `Shared/Update/*`, `RollbackOrchestrator`, `WatchdogState`)

---

## 6. Historico de versoes

Timeline consolidada em memoria `project_agent_retomada_2026_08_04.md`. Marcos:

- v1.3.0 (jul/26) — BLOQUEANTES pre-1o cliente (DPAPI + SelfContained + Watchdog)
- v1.3.10 (jul/26) — WMI Plan A auto-update
- v1.3.13 (jul/26) — versao dominante em prod hoje
- v1.4.0 (jul/26) — LockScreenDetector + InstalacaoIdProvider + LOGIN dedup + status ocioso por duracao
- **v1.5.0** (planejada 2026-08+) — 5 eixos: Ociosidade + Input ampliado + Sessao + Reuniao/render + HardwareFingerprint

Specs desta wave em `.brain/tecnologia/specs/2026-08-04-agent-*-design.md` — ver INDICE.

---

## 7. Contato em caso de duvida

- Arquitetura: @Tony
- Codigo C# / Agent: @Bucky
- Backend Java: @Thor
- Backend Node (parcial — dominio portal): @Shuri
- QA: @Natasha
- Deploy: @Vision
- PO/decisoes de escopo: @Steve + Marcos (final)
