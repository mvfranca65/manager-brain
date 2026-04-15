# Pipeline Técnico — Relatórios Semanais IA
> Domínio: @Tony | TL
> Última atualização: 2026-04-09

---

## Visão geral

```
@Scheduled (seg 00h00)  ou  POST /admin/relatorio/disparar?semana=YYYY-Www
                │
                v
   OrquestradorRelatorioService
                │
    ┌───────────┼───────────────────────┐
    │           │                       │
    v           v                       v
  Empresa A   Empresa B   ...   Empresa N
    │
    v
  Para cada time ativo da empresa:
    │
    ├─► Para cada colaborador monitorado:
    │     │
    │     ├─ 1. ExtratorDadosService.extrairDadosSemanaV2()
    │     │      → eventos_janela + eventos_ociosidade → DadosColaboradorSemanaDTO
    │     │
    │     ├─ 2. CalculadoraPilaresService.calcular*()
    │     │      → 6 scores de pilar (0–100)
    │     │
    │     ├─ 3. Fator de confiabilidade
    │     │      → se diasComDados < 3: todos os scores × (dias/3)
    │     │
    │     ├─ 4. CalculadoraIndicadoresService.calcular*()
    │     │      → 6 indicadores psicológicos (0–100)
    │     │
    │     ├─ 5. LlmService.chamarLlm(individual-system.txt, dados)
    │     │      → RelatorioIndividualJSON (score_ia, diagnóstico, alertas, ações)
    │     │
    │     └─ 6. Persistir RelatorioIndividual (JSONB + scores)
    │            → save imediato por colaborador (resiliência)
    │
    └─► Após todos os individuais do time:
          │
          ├─ 7. Montar resumos compactos (~300-400 tokens cada, ~90% redução)
          │
          ├─ 8. LlmService.chamarLlm(time-system.txt, resumos)
          │      → RelatorioTimeJSON (score_ia_time, diagnóstico, plano gestor)
          │
          └─ 9. Persistir RelatorioTime (JSONB + score)
```

---

## Serviços envolvidos

| Serviço | Porta | Função | Autenticação |
|---------|-------|--------|--------------|
| `manager-srv-events` | 8080 | Ingestão de eventos do agent desktop | Device JWT |
| `manager-srv-admin` | 8081 | Gestão de tenants, API keys | Admin JWT |
| `manager-srv-portal` | 8082 | APIs REST do portal (relatórios, times, etc.) | JWT portal |
| `manager-srv-ia` | 8085 | Pipeline de geração de relatórios IA | API Key interna |

---

## Tabelas do banco de dados

### Tabelas de entrada (leitura)

| Tabela | Serviço escritor | Descrição |
|--------|------------------|-----------|
| `eventos_janela` | srv-events | Janela ativa: processo, título, início, fim |
| `eventos_ociosidade` | srv-events | Períodos de inatividade: início, fim |
| `usuarios` | srv-admin/portal | Cadastro com `jornada_horas`, `monitorado`, `ativo` |
| `times_empresa` | srv-portal | Times por empresa |

### Tabelas de saída (escrita)

| Tabela | Campos principais | Constraint |
|--------|-------------------|------------|
| `relatorio_individual` | `colaborador_id`, `time_id`, `empresa_id`, `semana_referencia`, `imanager_score`, `score_ia`, `relatorio_json` (JSONB), `pilares_json` (JSONB), `indicadores_json` (JSONB), `nome_colaborador`, `status`, `gerado_em` | UNIQUE(colaborador_id, semana_referencia) |
| `relatorio_time` | `time_id`, `empresa_id`, `semana_referencia`, `score_ia_time`, `relatorio_json` (JSONB), `status`, `gerado_em` | UNIQUE(time_id, semana_referencia) |
| `audit_events` | Tipo, nível, mensagem, tenant_id, evento JSON | Retenção 90 dias |

---

## Serviços do pipeline (srv-ia)

### ExtratorDadosService
- **Input:** `usuarioId` + `semanaReferencia` (formato `YYYY-Www`)
- **Output:** `DadosColaboradorSemanaDTO`
- Agrega `eventos_janela` e `eventos_ociosidade` em métricas diárias:
  - `activeMsPerDia` — milissegundos ativos por dia
  - `idleMsPerDia` — milissegundos ociosos por dia
  - `blocosFocoPorDia` — lista de durações de blocos contínuos (para análise de foco)
  - `appSwitchesFiltradosPorDia` — trocas de app (mudanças consecutivas de `nome_processo`)
  - `fdsAtividadeMs`, `fdsHorasAtivas`, `fdsAproveitamento` — métricas de fim de semana
- Timezone: `America/Sao_Paulo`
- Semana ISO: Janeiro 4 sempre pertence à semana 1
- Enriquece com dados do usuário: nome completo, jornada_horas, empresa_id

### CalculadoraPilaresService
- **Input:** Métricas brutas + `hEsperadaDia`
- **Output:** 6 scores de pilar (0–100) + IManager Score composto
- Cálculos puros, sem estado, sem IO
- Fórmulas detalhadas em [formulas.md](formulas.md)

### CalculadoraIndicadoresService
- **Input:** Métricas da semana atual + até 3 semanas históricas (via `RelatorioIndividualRepository`)
- **Output:** 6 indicadores psicológicos (0–100) com classificação BAIXO/MODERADO/ALTO
- Fórmulas detalhadas em [formulas.md](formulas.md)

### LlmService
- **Provider:** Anthropic API (Claude) — suporte parcial OpenAI
- **Model:** Configurável via `llm.model`
- **Configuração:** `max-tokens=4096`, `temperature=0.4`, `timeout=60s` (individual) / override para time
- **Prompt caching:** Quando `llm.cache-enabled=true`, envia header `anthropic-beta: prompt-caching-2024-07-31` e `cache_control: {type: ephemeral}` no bloco system
- **Retry:** 3 tentativas com backoff exponencial (base 2s)
- **Validação:** JSON de saída validado por campos obrigatórios; markdown code fences removidos automaticamente
- **Auditoria:** Cada chamada (sucesso ou erro) registrada via `AuditService`

### OrquestradorRelatorioService
- Coordena todo o fluxo acima
- **Resiliência:**
  - Save imediato por colaborador — falha de um não impede os demais
  - Idempotência — verifica flag `forcar` antes de sobrescrever
  - Delete + flush + save dentro de `@Transactional`
  - `diasComDados == 0` → status `SEM_DADOS`, pula LLM
- **Resumo para time:** `montarResumoIndividual()` extrai ~300-400 tokens por membro (scoreIa, pilares, ponto_critico, resumo truncado 300 chars, títulos de alertas)
- **Alerta:** `AVISO_TIME_GRANDE = 20` membros (risco de estouro de tokens)
- **Deep work:** Bloco >= 25 min conta como foco profundo
- **Pausa dinâmica:** `ratioPausaMin = 0.10 × min(1.0, ratioAtividade/0.75)` — escala com carga efetiva

---

## APIs de consumo (srv-portal)

| Endpoint | Método | Descrição |
|----------|--------|-----------|
| `/api/v1/relatorios/semanas?timeId=` | GET | Lista semanas disponíveis (nunca 404, pode retornar `[]`) |
| `/api/v1/relatorios/time?timeId=&semana=` | GET | Relatório do time + lista de membros com scores |
| `/api/v1/relatorios/colaborador?colaboradorId=&semana=` | GET | Relatório individual com pilares e indicadores |
| `/api/v1/relatorios/export/pdf?timeId=&semana=&tipo=&colaboradorId=` | GET | Export PDF (tipo: TIME, INDIVIDUAL, CONSOLIDADO) |

**Controle de acesso:** JWT extraído em toda request. Perfil GESTOR tem acesso livre; demais perfis verificam hierarquia via `TimeEquipeService`.

**Cross-schema:** `RelatorioService` resolve `empresa_id` por CNPJ e `time_id` por nome entre as tabelas do srv-ia e srv-portal via native queries.

---

## Deploy e infraestrutura

- **Container:** Docker multi-stage (Java builder + JRE runtime)
- **Hosting:** Fly.io
- **Banco:** PostgreSQL compartilhado com multitenancy por `tenant_id`/`empresa_id`
- **Scheduler:** `@Scheduled(cron)` no srv-ia — segunda-feira 00h00 UTC-3
- **Email:** `EmailNotificacaoService` (Task 11 — pendente). Template Thymeleaf, SMTP configurável, fire-and-forget
