# QA Report — Reestruturação das Abas (Detalhe do Colaborador)

> Autora: @Natasha (QA Engineer)
> Data: 2026-04-06
> Branch: feature/pre-reestruturacao
> Baseado em: spec.md, historias.md, tecnico.md, design-notes.md

---

## Resultado Geral

**BLOQUEIO PARCIAL — deploy condicional**

O núcleo da feature está funcional. Backend compila, testes passam, frontend builda sem erros. Porém há 2 gaps de spec identificados nas histórias H2 e H3 que precisam ser endereçados antes do merge na main.

---

## 1. Backend — APIs compilam e testes passam

### 1.1 Build e testes
- **PASS** — `./mvnw test` encerrou com `BUILD SUCCESS`
- 121 testes rodados, 0 falhas, 0 erros, 1 skipped (ApplicationContext — esperado)

### 1.2 Endpoints em EventosController
- **PASS** — `POST /api/v2/eventos/{usuarioId}/tendencia-periodo` existe (linha 304)
- **PASS** — `POST /api/v2/eventos/{usuarioId}/media-time` existe (linha 269)
- Ambos validam via `accessService.exigirAcessoAoUsuario(usuarioId)` — multitenancy OK

### 1.3 DTOs
- **PASS** — `TendenciaPeriodoResposta` existe em `dto/TendenciaPeriodoResposta.java`
- **PASS** — `DiaTendenciaResposta` existe em `dto/DiaTendenciaResposta.java`
- **PASS** — `SessaoFocoDiaResposta` existe em `dto/SessaoFocoDiaResposta.java`
- **PASS** — `MediaTimeResposta` existe em `dto/MediaTimeResposta.java`

### 1.4 Services
- **PASS** — `TendenciaPeriodoService` existe
- **PASS** — `MediaTimeService` existe

### 1.5 StatusOnlineService
- **PASS** — não foram encontrados TODO/FIXME ou System.out no serviço

---

## 2. Frontend — Build limpo

### 2.1 Build Angular
- **PASS** — `npx ng build` encerra com `Build at: 2026-04-06T14:33:30.289Z` sem erros
- Warnings presentes são todos pré-existentes (optional chain em modal-criar-usuario-portal, colaborador-form-modal, e dependências CommonJS de exceljs/file-saver). Nenhum relacionado à feature nova.

### 2.2 Componentes novos — existência
- **PASS** — `tab-dia-dia` existe em `components/tab-dia-dia/`
- **PASS** — `tab-resumo` existe em `components/tab-resumo/`
- **PASS** — `kpi-card` existe em `components/kpi-card/`
- **PASS** — `frase-acionavel` existe em `components/frase-acionavel/`
- **PASS** — `pill-periodo` existe em `components/pill-periodo/`
- **PASS** — `tendencia-jornada` existe em `components/tendencia-jornada/`
- **PASS** — `tendencia-foco` existe em `components/tendencia-foco/`

### 2.3 Services
- **PASS** — `KpiComparacaoService` existe em `services/kpi-comparacao.service.ts`
- **PASS** — `ColaboradoresService` possui métodos `tendenciaPeriodo()` e `mediaTime()` apontando para os endpoints v2 corretos

### 2.4 Declarações no módulo
- **PASS** — `colaboradores.module.ts` declara: `TabDiaDiaComponent`, `TabResumoComponent`, `KpiCardComponent`, `PillPeriodoComponent`, `TendenciaJornadaComponent`, `TendenciaFocoComponent`, `FraseAcionavelComponent`

---

## 3. Integration — Tabs conectadas corretamente

### 3.1 colaborador-detalhe.component.html
- **PASS** — `*ngIf="activeTab === 'dia-a-dia'"` renderiza `<app-tab-dia-dia>`
- **PASS** — `*ngIf="activeTab === 'resumo'"` renderiza `<app-tab-resumo>`

### 3.2 colaborador-detalhe.component.ts
- **PASS** — `activeTab = 'dia-a-dia'` como default
- **PASS** — Array `tabs` configurado como:
  ```typescript
  { id: 'dia-a-dia', label: 'Dia a Dia' },
  { id: 'resumo',    label: 'Resumo' },
  { id: 'timeline',  label: 'Timeline 360°' },
  { id: 'dados',     label: 'Detalhes do Colaborador' }
  ```
- Ordem correta: Dia a Dia | Resumo | Timeline 360° | Detalhes

**Observação:** O `activeTab` usa `'dia-a-dia'` (com hífen simples), mas o template HTML usa `*ngIf="activeTab === 'dia-a-dia'"`. Consistente. Sem risco de mismatch.

---

## 4. Spec compliance — Cross-reference com historias.md

### H1 — Reestruturação das abas
- **PASS** — "Visão Geral" renomeada para "Dia a Dia"
- **PASS** — "Indicadores" renomeada para "Resumo"
- **PASS** — "Dia a Dia" é aba default
- **PASS** — Abas "Timeline 360°" e "Detalhes" intactas
- **PASS** — Ordem correta das abas
- **PASS** — Nenhum KPI duplicado entre as abas

### H2 — Aba Dia a Dia: pergunta-guia, KPIs do dia, textos descritivos e tooltips
- **PASS** — Pergunta-guia "O que aconteceu hoje?" presente no topo
- **PASS** — 4 cards KPI em linha: Jornada total, Tempo ativo, Tempo em pausa, Tempo ausente
- **PASS** — "Tempo ausente" exibe subtítulo "inclui almoço"
- **PASS** — Botões "Dia anterior" / "Próximo dia" presentes
- **PASS** — Seletor de data com p-calendar presente
- **FAIL** — **Os KPI cards da aba Dia a Dia NÃO exibem textos descritivos por seção.** O template `tab-dia-dia.component.html` renderiza os cards com `*ngFor` sobre o array `cards: TimelineCard[]`, e o modelo `TimelineCard` não possui campo `descricao`. A spec exige linha de texto 12px / `#94a3b8` / sem bold em cada KPI (Jornada, Tempo ativo, Pausa, Ausente). Os textos estão definidos na spec mas não foram implementados nesta aba.
- **FAIL** — **Os KPI cards da aba Dia a Dia NÃO exibem ícone "?" com tooltip.** A spec define tooltips para: Jornada total, Tempo ativo, Tempo em pausa, Tempo ausente. O modelo `TimelineCard` não tem campo `tooltipTexto`. A aba Resumo usa corretamente `app-kpi-card` com `tooltipTexto` como @Input, mas a aba Dia a Dia usa cards inline sem esse suporte.

**O que precisa ser corrigido (H2):**
1. Adicionar campos `descricao` e `tooltipTexto` ao modelo `TimelineCard` (ou criar cards estáticos por tipo ao invés de `*ngFor` genérico).
2. Renderizar o texto descritivo em cada KPI card da aba Dia a Dia.
3. Renderizar o ícone "?" com `menu-tooltip` nos cards que têm tooltip definido na spec.

### H3 — Aba Dia a Dia: frase acionável automática
- **PASS** — Frase acionável presente em card com classe CSS de borda colorida (`.frase--sucesso`, `.frase--atencao`, `.frase--alerta`)
- **PASS** — Frase gerada automaticamente a partir dos KPIs do dia
- **PASS** — Quando dia dentro do esperado: informa percentual ativo ("Dia dentro do esperado. X% de aproveitamento ativo.")
- **FAIL** — **Quando jornada excedida, a frase NÃO menciona o excedente específico.** O spec define: "Jornada 10h 21min — **2h 21min acima do esperado**. Verifique se há sobrecarga." A implementação atual gera: "Jornada 10h 21min — **acima do esperado**. Verifique se há sobrecarga." — sem calcular e exibir o excesso em horas/min.
- **FAIL** — **Quando tempo ausente acima do habitual, a frase NÃO inclui a comparação com a média.** O spec define: "Tempo ausente acima do habitual — **1h 20min vs média de 45min**." A implementação atual gera apenas: "Tempo ausente acima do habitual." sem a comparação. Nota: isso requer dados históricos de média de ausência do colaborador, que a API atual (`timeline/completo`) pode não retornar. Necessário verificar com o @Tony se o campo `mediaAusenteMinutos` deve ser adicionado ao `TimelineCompletaResponse`.

**O que precisa ser corrigido (H3):**
1. No método `gerarFraseDia()` de `tab-dia-dia.component.ts`, calcular o excedente em segundos: `const excesso = jornada - (jornadaEsperadaMinutos * 60)` e formatar para incluir na frase.
2. Para a frase de "ausente acima do habitual": avaliar com @Tony se a API fornece a média histórica. Se sim, adicionar ao `TimelineCompletaResponse` e usar no cálculo. Se não, marcar como gap de API (H9 parcial).

### H4 — Aba Dia a Dia: layout híbrido Top Apps + Eventos
- **PASS** — Layout `.layout-hibrido` com `.coluna-esquerda` e `.coluna-direita` implementado
- **PASS** — Top 5 Apps do dia à esquerda com barras proporcionais
- **PASS** — Eventos Detalhados à direita com paginação (opções 10, 30, 50, 100 via `pageSizeOptions`)
- **PASS** — KPIs e Timeline temporal acima dos dois painéis (largura total)
- **PASS** — Timeline exibe blocos: verde (atividade), amarelo (pausa), cinza (ausente)
- **PASS** — Textos descritivos presentes para "Top 5 Apps do dia" e "Eventos Detalhados"
- **OBSERVAÇÃO** — Responsividade mobile não foi verificada via execução (ambiente CLI). O SCSS usa `@media (max-width: 768px)` conforme spec — assumido correto, mas precisa de validação visual no browser.

### H5 — Aba Resumo: KPIs médios com comparação do time
- **PASS** — Pergunta-guia "Qual o padrão desse colaborador?" presente
- **PASS** — 4 cards KPI médio usando `app-kpi-card`: Jornada média, Tempo ativo médio, Pausa médio, Foco médio
- **PASS** — Jornada média exibe alerta quando excedida (`[alerta]="jornadaExcedida"`)
- **PASS** — Card "Foco médio" usa cor verde e subtítulo com contagem de sessões
- **PASS** — Comparação com time via `mediaTime` e `mediaTimeStatus` em cada card
- **PASS** — Textos descritivos em cada card usando o campo `descricao` do `app-kpi-card`
- **PASS** — Tooltip "?" no card "Foco médio" com texto correto da spec
- **PASS** — Quando `membrosConsiderados === 0`, comparação é ocultada silenciosamente (`this.comparacao = null`)
- **PASS** — Semântica de comparação correta por KPI implementada em `KpiComparacaoService` com comentários claros

### H6 — Aba Resumo: gráfico de tendência de jornada por dia
- **PASS** — Gráfico `app-tendencia-jornada` com barras verticais CSS, uma por dia
- **PASS** — Barras vermelhas quando `jornadaExcedida === true` (cor `#dc2626`)
- **PASS** — Barras verdes no estado padrão (cor `#22c55e`)
- **PASS** — Texto descritivo "Evolução da jornada ao longo dos dias. Dias vermelhos excederam o esperado. Veja se é pontual ou um padrão." presente
- **PASS** — Gráfico atualiza ao mudar período (vinculado ao `diasJornada` que é recalculado no `forkJoin`)

### H7 — Aba Resumo: gráfico tendência de foco com barras empilhadas e frase
- **PASS** — Gráfico `app-tendencia-foco` com barras empilhadas CSS por dia
- **PASS** — Paleta implementada conforme design-notes.md (família indigo): RASO `#fef08a`, PRODUTIVO `#a5b4fc`, BOM_FOCO `#6366f1`, DEEP_WORK `#3730a3`
- **PASS** — Legenda inline com as 4 faixas
- **PASS** — Subtítulo "X sessões no período · maior: Xh Xmin" via getter `subtituloGrafico`
- **PASS** — Tooltip "?" com texto correto da spec: "Uma sessão de foco é um período de trabalho contínuo sem interrupções maiores que 2 minutos. Só sessões acima de 5 minutos são contadas."
- **PASS** — Frase acionável de foco gerada automaticamente abaixo do gráfico
- **PASS** — Lógica de classificação: FRAGMENTADO, EQUILIBRADO, DEEP_WORK implementada em `gerarFraseFoco()` com frases exemplares corretas
- **PASS** — Gráfico atualiza ao mudar período

### H8 — Aba Resumo: seletor de período com "Personalizado" discreto
- **PASS** — Pills exibidas: Hoje | 7 dias | 15 dias | 30 dias
- **PASS** — Período default: `'7d'`
- **PASS** — Pill "Personalizado ▾" com ícone SVG (não caractere unicode) — implementado em `pill-periodo.component.html`
- **PASS** — Calendar picker oculto por padrão, abre ao clicar em "Personalizado"
- **PASS** — Estado visual "ativo" com classe CSS distinta para custom range
- **PASS** — Quando custom range ativo, exibe as datas no label da pill
- **PASS** — Todos os componentes da aba Resumo atualizam ao trocar o período

### H9 — APIs backend
- **PASS** — Endpoint `POST /api/v2/eventos/{usuarioId}/tendencia-periodo` retorna array com jornada e sessões de foco por dia
- **PASS** — Endpoint `POST /api/v2/eventos/{usuarioId}/media-time` retorna KPIs médios do time
- **PASS** — Ambos aceitam `dataInicio` / `dataFim` via `ResumoPeriodoColaboradorRequest`
- **PASS** — Dados de foco incluem campo `faixa` para classificação por duração
- **PASS** — Lógica de sessão de foco (>5min, sem interrupção >2min) é responsabilidade do `TendenciaPeriodoService`
- **PASS** — Contratos TypeScript documentados em `tab-resumo.component.ts` (`TendenciaPeriodoResponse`, `DiaTendenciaApi`, `SessaoFocoApi`, `MediaTimeResponse`)

---

## 5. Code Quality

### Frontend
- **PASS** — Nenhum `TODO` / `FIXME` nos novos componentes (`/components/*.ts`)
- **PASS** — Nenhum `console.log` nos novos componentes
- **PASS** — Nenhuma string hardcoded que deveria ser configurável (jornada esperada 480min é passada como `@Input` no `tendencia-jornada`, não hardcoded no template)
- **PASS** — Null safety: uso extensivo de `?? 0`, `?? '--'`, `catchError(() => of(null))`, `if (!colaboradorId) return`

### Backend
- **PASS** — Nenhum `TODO` / `FIXME` nos novos services (`TendenciaPeriodoService`, `MediaTimeService`)
- **PASS** — Nenhum `System.out.print` nos novos services
- **OBSERVAÇÃO** — Existe um `// TODO(MINOR-3)` em `UsuarioPortalAvatarUrlBuilder.java`, mas é pré-existente e não relacionado a esta feature

---

## Resumo Executivo

| ID | Item | Status | Severidade |
|----|------|--------|------------|
| BE-1 | Backend compila + testes passam (121/121) | PASS | — |
| BE-2 | Endpoint POST tendencia-periodo existe | PASS | — |
| BE-3 | Endpoint POST media-time existe | PASS | — |
| BE-4 | DTOs TendenciaPeriodoResposta, DiaTendenciaResposta, SessaoFocoDiaResposta, MediaTimeResposta | PASS | — |
| BE-5 | Services TendenciaPeriodoService, MediaTimeService | PASS | — |
| FE-1 | Build Angular limpo (sem erros) | PASS | — |
| FE-2 | Componentes tab-dia-dia, tab-resumo, kpi-card, frase-acionavel, pill-periodo, tendencia-jornada, tendencia-foco | PASS | — |
| FE-3 | KpiComparacaoService existe | PASS | — |
| FE-4 | Métodos tendenciaPeriodo(), mediaTime() no service | PASS | — |
| FE-5 | Declarações corretas no colaboradores.module.ts | PASS | — |
| INT-1 | Abas 'dia-a-dia' e 'resumo' renderizam componentes corretos | PASS | — |
| INT-2 | Tabs renomeadas corretamente, default correto | PASS | — |
| H1 | Abas renomeadas, ordem correta, default correto | PASS | — |
| H2 | Textos descritivos nos KPI cards do Dia a Dia | **FAIL** | MÉDIO |
| H2 | Tooltips "?" nos KPI cards do Dia a Dia | **FAIL** | MÉDIO |
| H3 | Frase inclui excedente específico (Xh Xmin acima do esperado) | **FAIL** | BAIXO |
| H3 | Frase ausente inclui comparação com média histórica | **FAIL** | BAIXO |
| H4 | Layout híbrido Top Apps + Eventos | PASS | — |
| H5 | KPIs médios com comparação do time | PASS | — |
| H6 | Gráfico tendência jornada | PASS | — |
| H7 | Gráfico tendência foco (barras empilhadas + frase) | PASS | — |
| H8 | Seletor período com Personalizado discreto | PASS | — |
| H9 | APIs backend novas | PASS | — |
| QC-1 | Sem TODO/FIXME no código novo | PASS | — |
| QC-2 | Sem console.log | PASS | — |
| QC-3 | Null safety em caminhos críticos | PASS | — |

---

## Falhas Detalhadas — O que fazer

### FAIL-1: H2 — Textos descritivos e tooltips ausentes nos KPI cards da aba Dia a Dia (MÉDIO)

**Problema:** O componente `tab-dia-dia.component.html` renderiza KPI cards via `*ngFor` sobre `cards: TimelineCard[]`. O modelo `TimelineCard` (em `timeline.models.ts`) não possui campos `descricao` ou `tooltipTexto`. Em consequência, nenhum texto descritivo nem ícone "?" com tooltip aparece nos 4 cards de KPI da aba Dia a Dia.

**A aba Resumo faz corretamente** via `app-kpi-card` com `@Input() descricao` e `@Input() tooltipTexto` — mas a aba Dia a Dia não usa esse componente.

**Fix sugerido (@Miles):**
Opção A (preferida): Converter os KPI cards da aba Dia a Dia para usar `app-kpi-card` com cards estáticos em vez de `*ngFor` genérico. Isso permite passar os textos descritivos e tooltips corretos conforme spec.

Opção B: Adicionar campos `descricao?: string` e `tooltipTexto?: string` ao `TimelineCard` e popular no `ColaboradorDetalheComponent` ao criar os `defaultTimelineCards`.

Textos a implementar (da spec):
- Jornada total: descricao = "Quanto tempo durou o dia de trabalho, do início ao fim. Se estiver acima do esperado, pode indicar sobrecarga." | tooltip = "Calculado do primeiro ao último evento detectado. A jornada esperada é configurada no perfil do colaborador (padrão: 8h)."
- Tempo ativo: descricao = "Quanto tempo o colaborador esteve usando o computador. Compare com a jornada para ver o aproveitamento do dia." | tooltip = "Inclui digitação, cliques e uso de aplicativos. Pausas menores que 5 minutos não interrompem o tempo ativo."
- Tempo em pausa: descricao = "Pequenas pausas durante o dia. Pausas saudáveis são normais e esperadas." | tooltip = "Detectado automaticamente quando não há atividade por mais de 5 minutos. Pausas acima de 30 minutos viram 'ausente'."
- Tempo ausente: descricao = "Períodos longos sem atividade, como almoço ou saída antecipada. Verifique se está dentro do esperado." | tooltip = "Inclui almoço, reuniões fora do computador e qualquer intervalo acima de 30 minutos."

---

### FAIL-2: H3 — Frase acionável não menciona excedente específico (BAIXO)

**Problema:** Em `tab-dia-dia.component.ts`, método `gerarFraseDia()`, quando a jornada é excedida a frase gerada é:
`"Jornada 10h 21min — acima do esperado. Verifique se há sobrecarga."`

O spec exige:
`"Jornada 10h 21min — 2h 21min acima do esperado. Verifique se há sobrecarga."`

O componente já recebe `jornadaTotalSegundos` como `@Input`. O `@Input() jornadaExcedida = false` indica quando foi excedida, mas o componente não recebe a jornada esperada em segundos para calcular o excesso.

**Fix sugerido (@Miles):**
1. Adicionar `@Input() jornadaEsperadaMinutos = 480` ao `TabDiaDiaComponent`.
2. No método `gerarFraseDia()`:
   ```typescript
   const jornadaEsperadaSegundos = (this.jornadaEsperadaMinutos ?? 480) * 60;
   const excesso = jornada - jornadaEsperadaSegundos;
   const excessoFormatado = this.formatarSegundos(excesso);
   return {
     texto: `Jornada ${jornadaLabel} — ${excessoFormatado} acima do esperado. Verifique se há sobrecarga.`,
     tipo: 'atencao'
   };
   ```
3. Passar `[jornadaEsperadaMinutos]` do `ColaboradorDetalheComponent` para `app-tab-dia-dia`.

---

### FAIL-3: H3 — Frase "ausente acima do habitual" sem comparação com média (BAIXO / GAP DE API)

**Problema:** A frase gerada é "Tempo ausente acima do habitual." sem incluir "Xh Xmin vs média de Ymin".

**Análise:** A API `timeline/completo` não retorna a média histórica de ausência do colaborador. Isso é um gap de dados — a lógica de comparação existe na spec mas o backend não provê o dado necessário.

**Ação recomendada:**
- @Tony: avaliar se `TimelineCompletaResponse` deve incluir campo `mediaAusenteHabitualMinutos` (calculado via histórico dos últimos 30 dias, por exemplo).
- Até que o dado exista na API, a implementação atual ("Tempo ausente acima do habitual.") é aceitável como fallback.
- Registrar como dívida técnica (não bloqueia o deploy desta feature).

---

## Decisão de Deploy

| Cenário | Recomendação |
|---------|-------------|
| Deploy com FAIL-1 (textos/tooltips ausentes no Dia a Dia) | **NÃO aprovado** — a spec é explícita e o Resumo já implementou corretamente. Gap visível ao gestor. |
| Deploy com FAIL-2 (frase sem excedente específico) | **Aprovado condicionalmente** — o comportamento atual não é errado, apenas incompleto. Pode seguir como v1.1. |
| Deploy com FAIL-3 (frase ausente sem comparação) | **Aprovado** — requer dado de API que não existe. Registrar como dívida técnica. |

**Veredito final: BLOQUEADO até resolução de FAIL-1.**
FAIL-2 pode ser resolvido junto ou entrar como issue imediata pós-merge.
FAIL-3 fica em backlog aguardando decisão de @Tony sobre a API.
