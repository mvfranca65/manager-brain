# Histórias — Relatórios Semanais por IA
> Domínio: @Steve | PO
> Última atualização: 2026-03-25

---

## Épico
**Como gestor, quero receber toda segunda-feira um diagnóstico completo do meu time para tomar decisões baseadas em dados, não em intuição.**

---

## Status geral por fase

| Fase | Tasks | Concluídas | Pendentes |
|------|-------|-----------|-----------|
| Fase 1 — `manager-srv-ia` | 11 | 10 | 1 |
| Fase 2 — `manager-srv-portal` | 3 | 0 | 3 |
| Fase 3 — Frontend Angular | 6 | 0 | 6 |
| **Total** | **20** | **10** | **10** |

**Testes implementados:** 42 (todos passando)

---

## Fase 1 — manager-srv-ia

### ✅ Task 1 — Scaffold do projeto
Spring Boot 3.2.3, Java 17, Lombok, WebClient, Mail, Thymeleaf, PostgreSQL, JSONB

### ✅ Task 2 — Entidades JPA e repositórios
- `RelatorioIndividual` + `RelatorioTime` com JSONB
- Repositórios com queries por colaborador/time/semana

### ✅ Task 3 — DTOs de entrada e saída
- `InputLlmIndividualDTO`, `InputLlmTimeDTO`
- `RelatorioIndividualDTO` (com `getScoreIa()`), `RelatorioTimeDTO`

### ✅ Task 4 — ExtratorDadosService
- Query nativa em `evento_atividade`, parser semana ISO, agregação por dia
- 2 testes

### ✅ Task 5 — CalculadoraPilaresService
- 6 pilares com fórmulas exatas: Atividade, Foco, Consistência, Saúde, Fragmentação, Anomalias
- `calcularIManagerScore` — soma ponderada
- 14 testes

### ✅ Task 6 — CalculadoraIndicadoresService
- 6 indicadores: Burnout, Desengajamento, Sobrecarga Silenciosa, Instabilidade, Fadiga FDS, Horas Extras
- `INDETERMINADO` quando sem histórico
- 13 testes

### ✅ Task 7 — LlmService
- Suporte Anthropic + OpenAI, prompt caching condicional
- Retry 3× backoff 1s, timeout configurável
- `validarJsonSaida` — verifica `score_ia` obrigatório
- 4 testes

### ✅ Task 8 — RelatorioIndividualService + RelatorioTimeService
- Prompts em `classpath:prompts/*.txt`
- Re-tenta 1× em JSON inválido
- 2 testes

### ✅ Task 9 — OrquestradorRelatorioService
- N chamadas paralelas via `CompletableFuture.supplyAsync`
- Idempotência: pula colaboradores já gerados (quando `forcar=false`)
- Tolerância a falhas: erro em 1 colaborador não para os demais
- Log de aviso quando time > 20 membros
- 5 testes

### ✅ Task 10 — Scheduler + Endpoint manual
- `@Scheduled(cron)` toda segunda 00h00 fuso `America/Sao_Paulo`
- `@ConditionalOnProperty(relatorio.scheduler.enabled)`
- `AdminRelatorioController` — `@Profile("local")` — endpoints para disparo manual

### ⏳ Task 11 — EmailNotificacaoService + Template Thymeleaf
**PENDENTE**
- `EmailNotificacaoService`: `JavaMailSender` + `SpringTemplateEngine`
- Template `relatorio-semanal.html` (Thymeleaf)
- `enviarRelatorio`, `contarMembrosAtencao`

---

## Fase 2 — manager-srv-portal

### ⏳ Task 12 — GET /api/v1/times com filtro de hierarquia
**PENDENTE**
- `TimeController` — novo endpoint `/api/v1/times` (mantém `/api/times` por compatibilidade)
- `TimeService` — `listarTimesVisiveis(gestorId, filtro, page, size)` com subtree por JWT
- `TimeServiceHierarquiaTest`

### ⏳ Task 13 — APIs de relatório
**PENDENTE**
- `RelatorioController` — 3 endpoints: `/semanas`, `/time`, `/colaborador`
- `RelatorioService` — busca nas tabelas, verifica permissão via JWT
- `SemanaRelatorioDTO`, `RelatorioNaoEncontradoException`
- `RelatorioControllerTest`

### ⏳ Task 14 — Export PDF
**PENDENTE**
- `GET /api/v1/relatorios/export/pdf?timeId=&semana=&tipo=&colaboradorId=`
- Tipos: `TIME` | `INDIVIDUAL` | `CONSOLIDADO`
- Biblioteca: Flying Saucer / iText / JasperReports (a definir → ADR)

---

## Fase 3 — Frontend Angular

### ⏳ Task 15 — Interfaces TypeScript + RelatoriosApiService
**PENDENTE**
- `relatorio.model.ts` — interfaces completas
- `relatorios-api.service.ts` — `getSemanas`, `getRelatorioTime`, `getRelatorioColaborador`, `exportPdf`

### ⏳ Task 16 — Módulo lazy /relatorios
**PENDENTE**
- `relatorios.module.ts` + rota lazy em `app-routing.module.ts`

### ⏳ Task 17 — FiltroRelatorioComponent + SemanaNavigatorComponent
**PENDENTE**
- Seleção de time + semana
- Navegação anterior/próxima entre semanas disponíveis

### ⏳ Task 18 — RelatorioTimeComponent + RelatorioIndividualComponent
**PENDENTE**
- Score + delta + tendência
- Resumo executivo, ranking membros, dinâmicas coletivas
- Pilares (radar/barras), indicadores psicológicos
- Plano de ação, retrospectiva (condicional)
- `@Output() colaboradorClicado` → abre relatório individual

### ⏳ Task 19 — RelatoriosPageComponent + deeplinks
**PENDENTE**
- Página raiz que coordena todos os filhos
- Query params: `?timeId=&colaboradorId=&semana=`
- Deeplink de tela de time → `/relatorios?timeId=`
- Deeplink de tela de colaborador → `/relatorios?timeId=&colaboradorId=`

### ⏳ Task 20 — Validação end-to-end
**PENDENTE**
- Fluxo completo local com disparo manual
- Estado vazio (sem relatórios), primeira semana (sem retrospectiva)
- Export PDF para os 3 tipos
