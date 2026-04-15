# Feature: Relatórios Semanais por IA
> Rota: `/relatorios`
#feature #relatorios #ia
> Perfil: GESTOR (visão time + individual) | COLABORADOR (apenas próprio)
> Status: MVP1 validado — pipeline end-to-end funcionando

**Esta é a feature de maior valor do produto.**

## O que é

Toda segunda-feira às 00h00, o sistema gera automaticamente um diagnóstico completo da semana anterior (seg→dom) por colaborador e por time. Disponível no portal e exportável em PDF.

## O que tem na tela

### Visão Time
- Score IA do time com sinais (desempenho, cansaço, tendência) + justificativa
- Diagnóstico: resumo, dinâmica central, saúde coletiva, distribuição de trabalho
- Lista de membros com Score IA + Score Fórmula + status
- Alertas (SISTÊMICO / INDIVIDUAL) com severidade e prazo
- Plano gestor com ações numeradas, prioridade, evidência
- Destaques positivos do time
- Retrospectiva (evolução vs semana anterior, membros que responderam/regrediram)
- Mensagem ao gestor

### Visão Colaborador (lista)
- Lista de membros com score arredondado
- Card "Sem dados" para colaboradores sem eventos

### Visão Individual
- Score IA + Score Fórmula com sinais
- Diagnóstico + ponto crítico
- Alertas com "se nada mudar"
- Foco da semana (objetivo + ações com evidência)
- Destaques positivos
- Retrospectiva (evolução, ações cumpridas/não cumpridas)
- Mensagem ao gestor
- Detalhes técnicos: 6 pilares (barras) + 6 indicadores psicológicos (badges)

### PDF Export (3 tipos)
- **TIME** — relatório do time com tabela de membros
- **INDIVIDUAL** — relatório de um colaborador com pilares e indicadores
- **CONSOLIDADO** — time + todos os individuais em ordem alfabética

### Navegação
- Seletor de time (dropdown)
- Navegador de semanas (prev/next)
- Deep links: `?timeId=X&semana=Y&colaboradorId=Z`

## Pipeline técnico
```
@Scheduled(seg 00h) ou POST /admin/relatorio/disparar
  → ExtratorDadosService (eventos_janela + ociosidade)
  → CalculadoraPilaresService (5 pilares)
  → CalculadoraIndicadoresService (6 indicadores psicológicos)
  → LlmService (Anthropic API → JSON estruturado)
  → Persistência (relatorio_individual + relatorio_time, JSONB)
```

## Validação
- 10 cenários de teste × 3 semanas (W12, W13, W14) validados
- Cenários cobrem: excelente, mediano, ruim, burnout, eficiência, dados insuficientes, recuperação, montanha-russa, multitask, queda abrupta
- Cálculos verificados com <3pts de diferença vs simulação
- Referência: `Backend/test-data/cenarios-3semanas.md`

## Dados consumidos
- `GET /api/v1/relatorios/semanas?timeId=` — semanas disponíveis
- `GET /api/v1/relatorios/time?timeId=&semana=` — relatório do time + membros
- `GET /api/v1/relatorios/colaborador?colaboradorId=&semana=` — relatório individual
- `GET /api/v1/relatorios/export/pdf?timeId=&semana=&tipo=&colaboradorId=` — PDF

## Sub-documentos
- `pipeline.md` — arquitetura técnica detalhada
- `contrato-relatorio-json.md` — schema JSON que a LLM retorna
- `historias.md` — tasks por fase
- `testes.md` — cobertura de testes
- `adr/ADR-001-biblioteca-pdf.md` — decisão OpenPDF

## Pendências MVP2
- Email com template Thymeleaf (Task 11)
- Calibração do Score IA com feedback do gestor
- Variação de linguagem entre relatórios
- Pilar Anomalias (hoje hardcoded 100)
