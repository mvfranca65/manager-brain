# Tecnico — Colaboradores
> Dominio: @Tony | TL
> Status: em refinamento

---

## Plano de implementacao

**Plano completo:** `docs/superpowers/plans/2026-03-25-colaboradores-tela-unificada.md`

---

## Decisoes tecnicas

### DT-01: Abordagem de refatoracao (frontend)

**Decisao:** Refatorar o `TimesDashboardPage` existente (Abordagem A) em vez de criar componente novo ou compor sub-componentes.

**Motivo:** Menor impacto no routing, reutiliza logica existente do `TimeEquipePage`, resulta em menos codigo no final.

### DT-02: Novo endpoint com validacao hierarquica (backend)

**Decisao:** Criar `GET /api/v1/times/:timeId/colaboradores` com validacao de acesso hierarquico via `verificarAcessoAoTime()`.

**Motivo:** O endpoint legado (`/api/times/:timeId/colaboradores`) so valida tenant (CNPJ), nao hierarquia. Um gestor poderia manipular o `?time=` e ver colaboradores de times fora da sua hierarquia.

### DT-03: Paginacao e filtro de status no banco

**Decisao:** Mover paginacao e calculo de status para query nativa PostgreSQL (CTE com CASE/WHEN no heartbeat).

**Motivo:** O codigo atual carrega TODOS os colaboradores em memoria, cria todos os DTOs, filtra e pagina in-memory. Com 500+ colaboradores isso desperdia heap e CPU. A query nativa faz tudo em uma ida ao banco.

### DT-04: URL com query param em vez de path param

**Decisao:** `/colaboradores?time=123` em vez de `/colaboradores/:timeId`.

**Motivo:** Permite estado "sem time selecionado" (URL sem param), facilita deep linking compartilhavel, e o dropdown funciona como filtro — nao como navegacao.

---

### DT-05: Refatoracao /resumo-periodo — queries de periodo inteiro (2026-03-26)

**Decisao:** Substituir loop dia-a-dia (N×7 queries) por 5 queries de periodo inteiro + groupBy em memoria.

**Motivo:** 30 dias = ~210 queries. Refatorado para 6 queries fixas. Ganho de ~40x. Indices em `(empresa_id, colaborador_id, iniciado_em)` garantem performance mesmo com 1 ano de dados. Limite de 90 dias ja protege.

**Status:** ✅ Implementado em 2026-03-26

### DT-06: Refatoracao /timeline/resumo — eliminacao de queries duplicadas (planejado)

**Decisao:** 3 melhorias na API de Indicadores:
1. Unificar `/timeline` + `/timeline/resumo` em 1 endpoint (KPIs + blocos + eventos paginados)
2. `resolverContexto` chamado 1x (hoje chamado 3x por request)
3. Paginacao de eventos detalhados no banco (LIMIT/OFFSET) em vez de in-memory

**Motivo:** Hoje o frontend faz forkJoin de 2 chamadas que buscam os mesmos dados. Total: 13 queries para 1 dia. Com unificacao: 5 queries. Blocos da Visualizacao Temporal nao podem ser paginados (grafico precisa de todos), mas eventos detalhados sim.

**Status:** ✅ Implementado em 2026-03-26

### DT-07: Modelo de 3 estados de atividade (2026-03-26)

**Decisao:** Simplificar de 4 estados (Ativo, Pausa, Sem atividade, Offline) para 3 (Ativo, Pausa, Ausente). Adicionar gap-filling entre blocos e indicador de jornada excedida.

**Motivo:** "Sem atividade" e "Offline" sao indistinguiveis para o gestor. Simplificar para 3 estados melhora a compreensao. Gap-filling resolve buracos nos KPIs onde tempo nao era contabilizado.

**ADR:** `adr/ADR-001-modelo-3-estados-atividade.md`

**Status:** 🟡 Planejado

### DT-08: Fix buildUsuarioPortalPayload sem timesIdsGestor (2026-03-26)

**Decisao:** Incluir `timesIdsGestor` no payload de criação individual de gestor.

**Motivo:** O campo nunca era enviado, causando: relação ADMINISTRADOR_TIME não criada, time não aparece no organograma, edição não mostra times pré-selecionados.

**Status:** ✅ Implementado em 2026-03-26

### DT-09: Refatoração APIs de Hierarquia — Performance (2026-03-26)

**Decisao:** 2 melhorias imediatas:
1. Eliminar N+1 em `processarNosTimes` — trocar loop individual por query batch `findByPai_IdIn`
2. `@Transactional(readOnly = true)` no `listar()` do HierarquiaService

**Motivo:** 10 gestores = 10 queries extras desnecessárias. Transação de escrita aberta para operação read-only.

**Status:** ✅ Implementado em 2026-03-26

---

## Tech debt identificado (backlog)

| Issue | Severidade | Arquivo |
|-------|------------|---------|
| N+1 em `buscarGestoresDoTime()` — 40 queries extras/pagina | CRITICAL | `TimeEquipeService.java:497-566` |
| BFS hierarquico com query por no — 20+ queries em orgs grandes | HIGH | `TimeEquipeService.java:211-271` |
| Lookup empresa por CNPJ sem cache (toda request) | MEDIUM | `TimesV1Controller.java:102-110` |
| `calcularStatus()` duplicado em 2 services | LOW | `TimeEquipeService` + `ColaboradorService` |
| Default de page size inconsistente (12, 20, 100) | LOW | Varios |
| `sincronizarTimesGestor` 2 overloads confusos — público refaz busca de entidade/dept | LOW | `UsuarioPortalService.java:263-274` |
| `processarNosTimes` mistura 2 responsabilidades (times de colab + times de gestor) | LOW | `HierarquiaService.java:195-238` |

---

## Stack envolvida

- **Frontend:** Angular 16 + PrimeNG (`p-dropdown` com filter, `p-paginator`, `p-skeleton`)
- **Backend:** Spring Boot + PostgreSQL (query nativa com CTE)
- **Endpoints:**
  - `GET /api/v1/times` — listar times do usuario (ja existe)
  - `GET /api/v1/times/:timeId/colaboradores` — novo, com auth hierarquica

## ADRs

→ `adr/` (criar conforme decisoes forem tomadas)
