# Reestruturação das Abas — Refinamento Técnico
> Domínio: @Tony | Tech Lead
> Data: 2026-04-06
> Status: refinamento aprovado — pronto para desenvolvimento

---

## Decisões de arquitetura

### 1. Novas APIs vs extensão das existentes
Decisão: **endpoints separados** para tendência e média do time.
Justificativa: o `resumo-periodo` já retorna KPIs médios. Adicionar tendência por dia e sessões de foco como campos nele tornaria o payload pesado e acoplaria concerns diferentes. Endpoints separados permitem carregar em paralelo no frontend via `forkJoin`.

### 2. Frase acionável — backend ou frontend?
Decisão: **frontend**, inicialmente.
Justificativa: a lógica de classificação (fragmentado/equilibrado/deep work) é puramente derivada dos dados já retornados. Não requer acesso ao banco. Frontend calcula em memória, sem roundtrip. Revisitar se a lógica escalar para ML/LLM no futuro.

### 3. Média do time — como calcular
Decisão: endpoint `/media-time` recebe `dataInicio` + `dataFim`. O backend identifica o time do colaborador via `relacionamentos_hierarquicos` (papel = MEMBRO), agrega os KPIs médios dos membros do mesmo time no período, e retorna os valores. Query JPQL com `GROUP BY` + `AVG` sobre os campos de resumo já calculados.

---

## Backend (manager-srv-portal)

### APIs novas a criar

#### 1. `POST /api/v2/eventos/{usuarioId}/tendencia-periodo`

**Responsável:** @Thor
**Descrição:** retorna os KPIs de cada dia do período, para renderizar os gráficos de barras de tendência de jornada e de foco.

**Request body:** `ResumoPeriodoColaboradorRequest` (reutiliza DTO existente)
```json
{ "dataInicio": "2026-03-30", "dataFim": "2026-04-05" }
```

**Response DTO a criar:** `TendenciaPeriodoResposta`
```java
// TendenciaPeriodoResposta.java
public class TendenciaPeriodoResposta {
    private Long colaboradorId;
    private LocalDate dataInicio;
    private LocalDate dataFim;
    private List<DiaTendenciaResposta> dias;
}

// DiaTendenciaResposta.java
public class DiaTendenciaResposta {
    private LocalDate data;
    private String diaSemana;          // "Seg", "Ter", ... para label no gráfico

    // Jornada do dia
    private Long jornadaMinutos;
    private Boolean jornadaExcedida;
    private Integer jornadaEsperadaMinutos;

    // Sessões de foco do dia (para gráfico de barras empilhadas)
    private List<SessaoFocoDiaResposta> sessoesFoco;
    private Integer totalSessoesFoco;
    private Long maiorSessaoFocoSegundos;
    private Long tempoTotalFocoSegundos;

    // Classificação do dia (calculada pelo backend)
    private ClassificacaoDiaFoco classificacao; // FRAGMENTADO | EQUILIBRADO | DEEP_WORK | SEM_DADOS
}

// SessaoFocoDiaResposta.java
public class SessaoFocoDiaResposta {
    private Long duracaoSegundos;
    private String faixa; // "RASO" (5-20min) | "PRODUTIVO" (20-40min) | "BOM_FOCO" (40-60min) | "DEEP_WORK" (60min+)
}
```

**Lógica de classificação por dia (backend):**
- `FRAGMENTADO`: totalSessoesFoco > 5 E maiorSessaoFocoSegundos < 1800 (30min)
- `EQUILIBRADO`: totalSessoesFoco entre 2-5 OU maiorSessaoFoco entre 1800-3600
- `DEEP_WORK`: alguma sessão > 3600 E totalSessoesFoco < 4
- `SEM_DADOS`: dia sem eventos

**Nota de implementação:** reutilizar a query de sessões de foco já existente no `TimelineColaboradorService`. Iterar dia a dia no range do período, executar a lógica de detecção de sessões para cada data, montar o DTO.

---

#### 2. `POST /api/v2/eventos/{usuarioId}/media-time`

**Responsável:** @Vision
**Descrição:** retorna os KPIs médios do time ao qual o colaborador pertence no mesmo período.

**Request body:** `ResumoPeriodoColaboradorRequest` (reutiliza)

**Response DTO a criar:** `MediaTimeResposta`
```java
// MediaTimeResposta.java
public class MediaTimeResposta {
    private Long colaboradorId;
    private LocalDate dataInicio;
    private LocalDate dataFim;
    private Integer membrosConsiderados;

    private Long jornadaMediaMinutos;
    private Long tempoAtivoMedioMinutos;
    private Long tempoPausaMedioMinutos;
    private Long tempoTotalFocoMedioSegundos;
    private Long maiorSessaoFocoMedioSegundos;
}
```

**Lógica de implementação:**
1. Identificar o time do colaborador: buscar `RelacionamentoHierarquico` onde `usuarioId` é membro (papel COLABORADOR/MEMBRO) e obter o `timeEquipeId`.
2. Buscar todos os membros ativos do mesmo time (excluindo o próprio colaborador).
3. Para cada membro, calcular o resumo do período usando a lógica existente em `TimelineColaboradorService.calcularResumoPeriodo`.
4. Agregar os resultados com `AVG` em memória (ou JPQL se performance exigir).
5. Retornar `membrosConsiderados = 0` e todos os campos nulos se o colaborador não pertencer a nenhum time — frontend trata ausência silenciosamente.

**Acesso:** validar via `accessService.exigirAcessoAoUsuario(usuarioId)` (padrão v2).

---

### APIs existentes — modificações necessárias

#### `POST /api/v2/eventos/{usuarioId}/resumo-periodo`
**Sem alteração estrutural.** O `ResumoPeriodoColaboradorResponse` já contém `maiorSessaoFocoFormatada`, `totalSessoesFoco` e `tempoTotalFocoSegundos`. O campo `foco médio` da aba Resumo usa `maiorSessaoFocoFormatada` como valor principal.

**Adicionar campo opcional:**
```java
// Em ResumoPeriodoColaboradorResponse.java — adicionar:
private Long focoMedioSegundos;  // média da maior sessão de foco por dia (para KPI "Foco médio")
```
Atualmente `maiorSessaoFocoSegundos` retorna o pico do período inteiro. O KPI "Foco médio" da spec pede a média da maior sessão por dia. Adicionar cálculo no `TimelineColaboradorService.calcularResumoPeriodo`.

#### `POST /api/v2/eventos/{usuarioId}/timeline/completo` (a criar — equivalente v2)
Atualmente só existe `/api/eventos/{colaboradorId}/timeline/completo` (v1). Criar endpoint v2 com `accessService.exigirAcessoAoUsuario`.

**Responsável:** @Thor
**DTO:** sem alteração — reutiliza `TimelineCompletaResposta`.

---

### Novos DTOs — resumo

| DTO | Arquivo | Responsável |
|-----|---------|-------------|
| `TendenciaPeriodoResposta` | `dto/TendenciaPeriodoResposta.java` | @Thor |
| `DiaTendenciaResposta` | `dto/DiaTendenciaResposta.java` | @Thor |
| `SessaoFocoDiaResposta` | `dto/SessaoFocoDiaResposta.java` | @Thor |
| `ClassificacaoDiaFoco` | `dto/ClassificacaoDiaFoco.java` (enum) | @Thor |
| `MediaTimeResposta` | `dto/MediaTimeResposta.java` | @Vision |

---

## Frontend (manager-fed-portal)

### Componentes a criar

#### Aba "Dia a Dia"

| Componente | Arquivo | Responsável | Descrição |
|-----------|---------|-------------|-----------|
| `TabDiaDiaComponent` | `tab-dia-dia/tab-dia-dia.component.ts` | @Peter | Container principal da aba. Gerencia seletor de data e orquestra chamada ao `timeline/completo`. |
| `DiaDiaKpisComponent` | `tab-dia-dia/components/dia-dia-kpis.component.ts` | @Miles | 4 cards KPI do dia. Recebe `TimelineCompletaResponse`. |
| `FraseAcionavelDiaComponent` | `tab-dia-dia/components/frase-acionavel-dia.component.ts` | @Miles | Card com borda colorida. Lógica de geração da frase no próprio componente (TypeScript puro). |
| `DateNavComponent` | `tab-dia-dia/components/date-nav.component.ts` | @Miles | Botões "Dia anterior" / "Próximo dia" + seletor calendário (PrimeNG `p-calendar`). Emite `EventEmitter<Date>`. |
| `TopAppsDiaComponent` | `tab-dia-dia/components/top-apps-dia.component.ts` | @Miles | Lista de top 5 apps com barras proporcionais. Reutiliza estrutura do componente atual de top apps. |

**Nota:** `visao-geral-timeline.component.ts` e `visao-geral-eventos.component.ts` existentes serão **movidos** para dentro de `tab-dia-dia/components/` com renomeação:
- `visao-geral-timeline.component.ts` → `timeline-24h.component.ts`
- `visao-geral-eventos.component.ts` → `eventos-detalhados.component.ts`

#### Aba "Resumo"

| Componente | Arquivo | Responsável | Descrição |
|-----------|---------|-------------|-----------|
| `TabResumoComponent` | `tab-resumo/tab-resumo.component.ts` | @Peter | Container principal. Gerencia pills de período, chama `resumo-periodo`, `tendencia-periodo` e `media-time` em `forkJoin`. |
| `ResumoKpisComponent` | `tab-resumo/components/resumo-kpis.component.ts` | @Miles | 4 cards KPI médios. Inclui linha comparativa com o time abaixo de cada valor. |
| `PillPeriodoComponent` | `tab-resumo/components/pill-periodo.component.ts` | @Miles | Pills Hoje/7d/15d/30d + "Personalizado ▾" com `p-calendar` range. Emite `EventEmitter<{inicio: Date, fim: Date}>`. |
| `TendenciaJornadaComponent` | `tab-resumo/components/tendencia-jornada.component.ts` | @Peter | Gráfico de barras verticais por dia. Barras vermelhas quando `jornadaExcedida = true`. Usa **barras CSS puras** (sem PrimeNG Charts) para evitar dependência pesada. |
| `TendenciaFocoComponent` | `tab-resumo/components/tendencia-foco.component.ts` | @Peter | Gráfico de barras empilhadas por dia (sessões individuais). Cores por faixa. Frase acionável abaixo. Usa **barras CSS puras**. |
| `DonutDistribuicaoComponent` | `tab-resumo/components/donut-distribuicao.component.ts` | @Miles | Donut de distribuição Ativo/Pausa/Ausente com percentuais. Usa PrimeNG `p-chart` (Chart.js) — justificado pois o donut é complexo demais para CSS puro. |
| `TopAppsPeriodoComponent` | `tab-resumo/components/top-apps-periodo.component.ts` | @Miles | Top 5 apps do período. Estrutura idêntica ao `TopAppsDiaComponent` — considerar componente compartilhado `TopAppsListComponent`. |
| `FraseAcionavelFocoComponent` | `tab-resumo/components/frase-acionavel-foco.component.ts` | @Miles | Frase gerada a partir da classificação dos dias (recebe `DiaTendencia[]` já classificados pelo backend). |

---

### Componentes a modificar

#### `ColaboradorDetalheComponent`

**Arquivo:** `colaborador-detalhe.component.ts` / `.html`

Mudanças:
1. Renomear as abas no array `tabs`:
   ```typescript
   // Antes:
   { id: 'overview',    label: 'Visão Geral' },
   { id: 'indicadores', label: 'Indicadores' },
   // Depois:
   { id: 'dia-dia',  label: 'Dia a Dia' },
   { id: 'resumo',   label: 'Resumo' },
   ```
2. Substituir renderização condicional no template: `*ngIf="activeTab === 'dia-dia'"` e `*ngIf="activeTab === 'resumo'"`.
3. Remover lógica de KPIs, top apps, donut e timeline do componente pai — esses estados migram para `TabDiaDiaComponent` e `TabResumoComponent` (cada aba gerencia seu próprio estado e chamadas HTTP).
4. Manter `activeTab = 'dia-dia'` como default (era `'overview'`).

**Sem alteração de rota** — a rota `membro/:id` não muda. Não há query params de aba. Estado da aba é local ao componente.

#### `tab-visao-geral.component.ts` e `tab-timeline.component.ts` (aba Indicadores atual)
Esses componentes serão **descontinuados**. Lógica útil é extraída para os novos componentes de `tab-dia-dia` e `tab-resumo`. Não deletar imediatamente — manter comentados até validação em QA para facilitar rollback se necessário.

---

### Serviços

#### `ColaboradoresService` / `ColaboradoresApiService`
Adicionar métodos para as novas APIs:

```typescript
// Em colaboradores-api.service.ts (ou colaboradores.service.ts — verificar qual é o canônico)

tendenciaPeriodo(usuarioId: number, req: ResumoPeriodoRequest): Observable<TendenciaPeriodoResponse> {
  return this.http.post<TendenciaPeriodoResponse>(
    `/api/v2/eventos/${usuarioId}/tendencia-periodo`, req
  );
}

mediaTime(usuarioId: number, req: ResumoPeriodoRequest): Observable<MediaTimeResponse> {
  return this.http.post<MediaTimeResponse>(
    `/api/v2/eventos/${usuarioId}/media-time`, req
  );
}

timelineCompletoV2(usuarioId: number, filtro: TimelineFiltroRequest): Observable<TimelineCompletaResponse> {
  return this.http.post<TimelineCompletaResponse>(
    `/api/v2/eventos/${usuarioId}/timeline/completo`, filtro
  );
}
```

#### Modelos TypeScript a criar

```typescript
// tendencia-periodo.models.ts
export interface TendenciaPeriodoResponse {
  colaboradorId: number;
  dataInicio: string;
  dataFim: string;
  dias: DiaTendencia[];
}

export interface DiaTendencia {
  data: string;
  diaSemana: string;
  jornadaMinutos: number;
  jornadaExcedida: boolean;
  jornadaEsperadaMinutos: number;
  sessoesFoco: SessaoFoco[];
  totalSessoesFoco: number;
  maiorSessaoFocoSegundos: number;
  tempoTotalFocoSegundos: number;
  classificacao: 'FRAGMENTADO' | 'EQUILIBRADO' | 'DEEP_WORK' | 'SEM_DADOS';
}

export interface SessaoFoco {
  duracaoSegundos: number;
  faixa: 'RASO' | 'PRODUTIVO' | 'BOM_FOCO' | 'DEEP_WORK';
}

// media-time.models.ts
export interface MediaTimeResponse {
  colaboradorId: number;
  dataInicio: string;
  dataFim: string;
  membrosConsiderados: number;
  jornadaMediaMinutos: number | null;
  tempoAtivoMedioMinutos: number | null;
  tempoPausaMedioMinutos: number | null;
  tempoTotalFocoMedioSegundos: number | null;
  maiorSessaoFocoMedioSegundos: number | null;
}
```

---

### Gráficos de tendência — decisão de implementação

**Barras de tendência de jornada e foco: CSS puro (não PrimeNG Charts).**

Motivo: os dados são simples (valor por dia, cor por condição). CSS puro é:
- Sem dependência de Chart.js para um gráfico trivial
- Mais controlável para as cores condicionais (vermelho quando excedido)
- Mais leve (sem biblioteca externa carregando)
- Mais fácil de estilizar dentro do design system existente

Estrutura de barra CSS:
```html
<div class="barra-container">
  <div class="barra" [style.height.%]="alturaRelativa(dia)"
       [class.excedida]="dia.jornadaExcedida">
  </div>
  <span class="label">{{ dia.diaSemana }}</span>
</div>
```

**Donut de distribuição: PrimeNG `p-chart` (Chart.js).**
Justificado pois donut com segmentos e legendas em CSS é impraticável sem reescrever o que já existe na lib.

---

### Frase acionável — lógica frontend

#### Frase do dia (aba Dia a Dia)
Função pura em `FraseAcionavelDiaComponent`, recebe `TimelineCompletaResponse`:

```typescript
gerarFraseDia(kpis: TimelineCompletaResponse): FraseAcionavel {
  const excesso = kpis.jornadaTotalSegundos - (kpis.jornadaEsperadaMinutos * 60);
  if (kpis.jornadaExcedida && excesso > 0) {
    const excessoFormatado = formatarDuracao(excesso);
    return { texto: `Jornada ${kpis.jornadaTotalFormatada} — ${excessoFormatado} acima do esperado. Verifique se há sobrecarga.`, cor: 'vermelho' };
  }
  // demais regras...
}
```

#### Frase de foco (aba Resumo)
Função em `FraseAcionavelFocoComponent`, recebe `DiaTendencia[]` (já classificados pelo backend):

```typescript
gerarFraseFoco(dias: DiaTendencia[]): string {
  const diasComDados = dias.filter(d => d.classificacao !== 'SEM_DADOS');
  const fragmentados = diasComDados.filter(d => d.classificacao === 'FRAGMENTADO').length;
  const deepWork = diasComDados.filter(d => d.classificacao === 'DEEP_WORK').length;
  // regras de texto conforme spec...
}
```

---

### Layout híbrido — responsividade

Implementar com Flexbox SCSS no padrão do design system:

```scss
// Padrão para seção inferior (Top Apps + Eventos ou Top Apps + Donut)
.layout-hibrido {
  display: flex;
  gap: 16px;

  .coluna-esquerda { flex: 1; }   // 1/3
  .coluna-direita  { flex: 2; }   // 2/3

  @media (max-width: 768px) {
    flex-direction: column;
  }
}
```

---

## Tarefas ordenadas por dependência

### FASE 1 — Backend: novos DTOs e endpoints

| # | Tarefa | Responsável | Depende de |
|---|--------|-------------|------------|
| BE-01 | Criar DTOs `DiaTendenciaResposta`, `SessaoFocoDiaResposta`, `TendenciaPeriodoResposta`, enum `ClassificacaoDiaFoco` | @Thor | — |
| BE-02 | Criar serviço `TendenciaPeriodoService` com lógica de cálculo por dia (reutilizando sessões de foco existentes) | @Thor | BE-01 |
| BE-03 | Criar endpoint `POST /api/v2/eventos/{usuarioId}/tendencia-periodo` em `EventosController` | @Thor | BE-02 |
| BE-04 | Criar DTO `MediaTimeResposta` | @Vision | — |
| BE-05 | Criar serviço `MediaTimeService` com lógica de agregação por time | @Vision | BE-04 |
| BE-06 | Criar endpoint `POST /api/v2/eventos/{usuarioId}/media-time` em `EventosController` | @Vision | BE-05 |
| BE-07 | Adicionar campo `focoMedioSegundos` em `ResumoPeriodoColaboradorResponse` + cálculo em `TimelineColaboradorService` | @Vision | — |
| BE-08 | Criar endpoint `POST /api/v2/eventos/{usuarioId}/timeline/completo` (v2 com `accessService`) | @Thor | — |

### FASE 2 — Frontend: serviços e modelos

| # | Tarefa | Responsável | Depende de |
|---|--------|-------------|------------|
| FE-01 | Criar modelos TypeScript `TendenciaPeriodoResponse`, `DiaTendencia`, `SessaoFoco`, `MediaTimeResponse` | @Miles | BE-01 (contrato) |
| FE-02 | Adicionar métodos `tendenciaPeriodo()`, `mediaTime()`, `timelineCompletoV2()` no service | @Miles | FE-01 |

### FASE 3 — Frontend: aba Dia a Dia

| # | Tarefa | Responsável | Depende de |
|---|--------|-------------|------------|
| FE-03 | Criar `DateNavComponent` (navegação de datas) | @Miles | — |
| FE-04 | Criar `DiaDiaKpisComponent` (4 cards KPI do dia) | @Miles | — |
| FE-05 | Criar `FraseAcionavelDiaComponent` com lógica de geração | @Miles | FE-04 |
| FE-06 | Mover e renomear `visao-geral-timeline` → `timeline-24h.component` | @Miles | — |
| FE-07 | Mover e renomear `visao-geral-eventos` → `eventos-detalhados.component` | @Miles | — |
| FE-08 | Criar `TopAppsDiaComponent` (ou extrair componente compartilhado) | @Miles | — |
| FE-09 | Criar `TabDiaDiaComponent` — orquestra FE-03..08, chama `timelineCompletoV2()` | @Peter | FE-02, FE-03..08 |

### FASE 4 — Frontend: aba Resumo

| # | Tarefa | Responsável | Depende de |
|---|--------|-------------|------------|
| FE-10 | Criar `PillPeriodoComponent` (pills + calendário custom) | @Miles | — |
| FE-11 | Criar `ResumoKpisComponent` com linha comparativa do time | @Miles | FE-01 |
| FE-12 | Criar `TendenciaJornadaComponent` (barras CSS por dia) | @Peter | FE-01 |
| FE-13 | Criar `TendenciaFocoComponent` (barras empilhadas CSS) | @Peter | FE-01 |
| FE-14 | Criar `DonutDistribuicaoComponent` (PrimeNG `p-chart`) | @Miles | — |
| FE-15 | Criar `TopAppsPeriodoComponent` | @Miles | FE-08 (componente base) |
| FE-16 | Criar `FraseAcionavelFocoComponent` | @Miles | FE-01 |
| FE-17 | Criar `TabResumoComponent` — orquestra FE-10..16, `forkJoin` nas 3 APIs | @Peter | FE-02, FE-10..16 |

### FASE 5 — Frontend: integração no detalhe do colaborador

| # | Tarefa | Responsável | Depende de |
|---|--------|-------------|------------|
| FE-18 | Atualizar `ColaboradorDetalheComponent`: renomear abas, trocar renderização, limpar estado legado | @Peter | FE-09, FE-17 |
| FE-19 | Declarar novos componentes no `ColaboradoresModule` | @Peter | FE-09, FE-17 |

### FASE 6 — QA e cleanup

| # | Tarefa | Responsável | Depende de |
|---|--------|-------------|------------|
| QA-01 | Validar Dia a Dia: KPIs do dia, timeline, eventos paginados, frase acionável | @Natasha | FE-18 |
| QA-02 | Validar Resumo: KPIs médios, comparativo time, tendência jornada, tendência foco, donut | @Natasha | FE-18 |
| QA-03 | Validar responsividade (< 768px): empilhamento das colunas | @Natasha | FE-18 |
| QA-04 | Remover componentes legados (`tab-visao-geral`, `tab-indicadores`) após aprovação | @Peter | QA-01, QA-02 |

---

## Pontos de atenção

1. **Multitenancy:** todos os endpoints v2 usam `accessService.exigirAcessoAoUsuario(usuarioId)` que já valida `tenant_id`. Não dispensar esse check.

2. **Performance — `media-time`:** se o time tiver muitos membros e o período for longo (30 dias), o cálculo síncrono pode ser lento. @Vision deve medir o tempo de resposta com dados reais e avaliar se é necessário cachear ou calcular assincronamente. Threshold aceitável: < 2s.

3. **Colaborador sem time:** endpoint `/media-time` retorna `membrosConsiderados: 0` e campos nulos. O `ResumoKpisComponent` deve esconder a linha comparativa nesses casos — não mostrar "Média do time: --".

4. **Fuso horário:** `LocalDate` no backend não tem fuso. Frontend envia datas no padrão ISO `yyyy-MM-dd`. Sem offset de timezone. Manter comportamento atual.

5. **Rollback seguro:** abas antigas (`tab-visao-geral`, `tab-indicadores`) ficam no código até `QA-04` ser aprovado. Renomear via toggle de aba no `ColaboradorDetalheComponent` é suficiente para esconder sem deletar.

---

## H10 — Distribuição de horários de foco (refinamento técnico)

> Responsável: @Tony | Data: 2026-04-06

### Contexto e diagnóstico

A H10 pede que, ao lado da frase acionável do `TendenciaFocoComponent`, apareça a distribuição percentual das sessões de foco por faixa horária (manhã / tarde / noite). O backend **já tem todos os dados necessários**: o `TendenciaPeriodoService` identifica, monta e devolve sessões de foco individuais com `duracaoSegundos` e `faixa`, e os eventos brutos carregados do banco via `EventoJanelaRepository` e `EventoOciosidadeRepository` contêm `iniciadoEm` como `Instant` (timestamp completo com fuso `America/Sao_Paulo`). A conversão `ev.getInicio().atZone(ZONA).toLocalDate()` já está implementada no service — trivial estender para `toLocalTime()` e classificar por faixa horária.

**Não é necessária nova query ao banco.** Os eventos já são carregados para o período inteiro (4 queries). A classificação por faixa horária é computada em memória sobre os mesmos blocos ATIVIDADE que já formam as sessões.

---

### Backend

#### 1. Novo DTO: `FaixaHorariaResposta`

Criar o arquivo `dto/FaixaHorariaResposta.java`:

```java
package com.olimpo.manager.dto;

import lombok.Builder;
import lombok.Getter;

@Getter
@Builder
public class FaixaHorariaResposta {
    private int totalSessoes;
    private int tempoTotalMinutos;
    private float percentual;
}
```

#### 2. Novo DTO: `DistribuicaoFocoPorFaixaResposta`

Criar o arquivo `dto/DistribuicaoFocoPorFaixaResposta.java`:

```java
package com.olimpo.manager.dto;

import lombok.Builder;
import lombok.Getter;

@Getter
@Builder
public class DistribuicaoFocoPorFaixaResposta {
    private FaixaHorariaResposta manha;   // 06:00–11:59
    private FaixaHorariaResposta tarde;   // 12:00–17:59
    private FaixaHorariaResposta noite;   // 18:00–05:59
}
```

#### 3. Extensão do `TendenciaPeriodoResposta`

Adicionar o campo `distribuicaoFocoPorFaixa` no DTO existente `TendenciaPeriodoResposta.java`:

```java
// Em TendenciaPeriodoResposta.java — adicionar campo:
private DistribuicaoFocoPorFaixaResposta distribuicaoFocoPorFaixa;
```

**Decisão de localização:** o campo vai em `TendenciaPeriodoResposta` (não em `ResumoPeriodoColaboradorResponse`) porque a distribuição é derivada das sessões de foco por dia — dados já calculados pelo `TendenciaPeriodoService`. Adicionar no `resumo-periodo` obrigaria duplicar a lógica de sessões naquele service, que hoje só trabalha com KPIs agregados. O frontend já consome `tendencia-periodo` no `TabResumoComponent` via `forkJoin`, portanto nenhuma chamada adicional é necessária.

#### 4. Lógica de cálculo no `TendenciaPeriodoService`

O `TendenciaPeriodoService` já itera sobre `List<BlocoTemporalResposta>` filtrados por `ATIVIDADE` no método `calcularSessoesFocoIndividuais`. O horário de início de cada bloco está disponível como `LocalTime inicioBloco = LocalTime.parse(bloco.getInicio())`.

Adicionar método privado `calcularDistribuicaoPorFaixa` no `TendenciaPeriodoService`:

```java
private DistribuicaoFocoPorFaixaResposta calcularDistribuicaoPorFaixa(List<DiaTendenciaResposta> dias) {
    int sessManha = 0, sessTarde = 0, sessNoite = 0;
    long minManha = 0, minTarde = 0, minNoite = 0;

    for (DiaTendenciaResposta dia : dias) {
        // Necessário que DiaTendenciaResposta exponha os blocos de atividade com horário de início
        // Ver nota de implementação abaixo
        for (SessaoFocoComInicioInterno sessao : dia.getSessoesComInicio()) {
            LocalTime inicio = sessao.getHorarioInicio();
            long minutos = sessao.getDuracaoSegundos() / 60;
            if (inicio.getHour() >= 6 && inicio.getHour() < 12) {
                sessManha++; minManha += minutos;
            } else if (inicio.getHour() >= 12 && inicio.getHour() < 18) {
                sessTarde++; minTarde += minutos;
            } else {
                sessNoite++; minNoite += minutos;
            }
        }
    }

    int totalSessoes = sessManha + sessTarde + sessNoite;
    return DistribuicaoFocoPorFaixaResposta.builder()
            .manha(FaixaHorariaResposta.builder()
                    .totalSessoes(sessManha)
                    .tempoTotalMinutos((int) minManha)
                    .percentual(totalSessoes == 0 ? 0f : (sessManha * 100f) / totalSessoes)
                    .build())
            .tarde(FaixaHorariaResposta.builder()
                    .totalSessoes(sessTarde)
                    .tempoTotalMinutos((int) minTarde)
                    .percentual(totalSessoes == 0 ? 0f : (sessTarde * 100f) / totalSessoes)
                    .build())
            .noite(FaixaHorariaResposta.builder()
                    .totalSessoes(sessNoite)
                    .tempoTotalMinutos((int) minNoite)
                    .percentual(totalSessoes == 0 ? 0f : (sessNoite * 100f) / totalSessoes)
                    .build())
            .build();
}
```

**Nota de implementação — horário de início das sessões:**

O método `calcularSessoesFocoIndividuais` já lê `LocalTime inicioBloco = LocalTime.parse(bloco.getInicio())` para cada bloco ATIVIDADE. O ponto de início de cada sessão agregada corresponde ao `inicioBloco` do **primeiro bloco** da sessão (antes de entrar no loop de gap). @Thor deve:

1. Refatorar `calcularSessoesFocoIndividuais` para capturar e retornar o `horarioInicio` de cada sessão — ou criar uma estrutura interna `SessaoFocoComInicio` (não exposta na API) usada apenas para o cálculo da distribuição.
2. Passar essa lista estendida para `calcularDistribuicaoPorFaixa` antes de montar o `TendenciaPeriodoResposta`.
3. O `SessaoFocoDiaResposta` (DTO de API) **não precisa ser alterado** — o `horarioInicio` é usado apenas internamente para calcular a distribuição e não deve ser serializado.

**Formato de `bloco.getInicio()`:** os blocos usam `String` no formato `"HH:mm"` ou `"HH:mm:ss"` — verificar o formato exato em `BlocoTemporalResposta` antes de implementar o `LocalTime.parse`. Se necessário, usar `LocalTime.parse(bloco.getInicio(), DateTimeFormatter.ofPattern("HH:mm"))`.

#### 5. Integração no `TendenciaPeriodoService.calcular()`

No método `calcular()`, após montar a lista `dias`, invocar:

```java
DistribuicaoFocoPorFaixaResposta distribuicao = calcularDistribuicaoPorFaixa(dias);

return TendenciaPeriodoResposta.builder()
        .colaboradorId(usuarioId)
        .dataInicio(dataInicio)
        .dataFim(dataFim)
        .diasConsiderados(...)
        .dias(dias)
        .distribuicaoFocoPorFaixa(distribuicao)   // campo novo
        .build();
```

#### 6. Nenhuma alteração no `EventosController`

O endpoint `POST /api/v2/eventos/{usuarioId}/tendencia-periodo` já existe e já retorna `TendenciaPeriodoResposta`. Como o campo `distribuicaoFocoPorFaixa` é adicionado ao DTO, ele será serializado automaticamente por Jackson. **Nenhuma alteração no controller é necessária.**

**Responsável backend:** @Thor

---

### Frontend

#### 1. Novos modelos TypeScript

Em `tendencia-periodo.models.ts` (ou no próprio `tab-resumo.component.ts` onde as interfaces já estão declaradas), adicionar:

```typescript
// Adicionar em tab-resumo.component.ts (junto com DiaTendenciaApi, SessaoFocoApi, etc.)

export interface FaixaHoraria {
  totalSessoes: number;
  tempoTotalMinutos: number;
  percentual: number;
}

export interface DistribuicaoFocoPorFaixa {
  manha: FaixaHoraria;
  tarde: FaixaHoraria;
  noite: FaixaHoraria;
}
```

Estender `TendenciaPeriodoResponse` já declarada em `tab-resumo.component.ts`:

```typescript
export interface TendenciaPeriodoResponse {
  colaboradorId: number;
  dataInicio: string;
  dataFim: string;
  dias: DiaTendenciaApi[];
  distribuicaoFocoPorFaixa?: DistribuicaoFocoPorFaixa;  // campo novo — opcional por segurança
}
```

#### 2. Novos `@Input()` no `TendenciaFocoComponent`

Em `tendencia-foco.component.ts`, adicionar dois `@Input()`:

```typescript
@Input() distribuicao: DistribuicaoFocoPorFaixa | null = null;
// distribuicao: objeto com manha/tarde/noite recebido do TabResumoComponent
// Quando null (sem sessões no período), a seção não é exibida no template
```

O tipo `DistribuicaoFocoPorFaixa` deve ser importado de `tab-resumo.component.ts` ou extraído para um arquivo de modelos compartilhado se o tamanho do componente justificar.

#### 3. Propagação no `TabResumoComponent`

No `atualizarTendencias()` de `tab-resumo.component.ts`, capturar o campo:

```typescript
// Adicionar propriedade ao componente:
distribuicaoFoco: DistribuicaoFocoPorFaixa | null = null;

// Em atualizarTendencias():
this.distribuicaoFoco = tendencia.distribuicaoFocoPorFaixa ?? null;
```

No template `tab-resumo.component.html`, passar o input para o componente de tendência:

```html
<app-tendencia-foco
  [dias]="diasFoco"
  [totalSessoes]="totalSessoesFoco"
  [maiorSessaoFormatada]="maiorSessaoFormatada"
  [tempoTotalFocoFormatado]="tempoTotalFocoFormatado"
  [frase]="fraseFoco?.texto"
  [fraseTipo]="fraseFoco?.tipo"
  [distribuicao]="distribuicaoFoco">   <!-- novo binding -->
</app-tendencia-foco>
```

#### 4. Exibição no template `tendencia-foco.component.html`

Adicionar a seção abaixo da frase acionável existente, condicionada a `distribuicao` não nulo e `totalSessoes > 0`:

```html
<!-- Seção de distribuição por faixa horária — exibir somente quando há sessões -->
<div class="distribuicao-faixa" *ngIf="distribuicao && totalSessoes > 0">
  <div class="distribuicao-pills">

    <!-- Manhã -->
    <span class="pill"
          [class.pill--destaque]="faixaDominante === 'manha'">
      Manhã {{ percentualInteiro(distribuicao.manha.percentual) }}%
    </span>

    <!-- Tarde -->
    <span class="pill"
          [class.pill--destaque]="faixaDominante === 'tarde'">
      Tarde {{ percentualInteiro(distribuicao.tarde.percentual) }}%
    </span>

    <!-- Noite -->
    <span class="pill"
          [class.pill--destaque]="faixaDominante === 'noite'">
      Noite {{ percentualInteiro(distribuicao.noite.percentual) }}%
    </span>

  </div>
  <p class="distribuicao-frase">{{ fraseDistribuicao }}</p>
</div>
```

#### 5. Lógica TypeScript no `TendenciaFocoComponent`

Adicionar getters e método no componente:

```typescript
/** Faixa com maior percentual de sessões. */
get faixaDominante(): 'manha' | 'tarde' | 'noite' | null {
  if (!this.distribuicao) return null;
  const { manha, tarde, noite } = this.distribuicao;
  const max = Math.max(manha.percentual, tarde.percentual, noite.percentual);
  if (max === 0) return null;
  if (manha.percentual === max) return 'manha';
  if (tarde.percentual === max) return 'tarde';
  return 'noite';
}

/** Frase descritiva automática conforme regras da spec. */
get fraseDistribuicao(): string {
  if (!this.distribuicao) return '';
  const { manha, tarde, noite } = this.distribuicao;
  const max = Math.max(manha.percentual, tarde.percentual, noite.percentual);
  const min = Math.min(manha.percentual, tarde.percentual, noite.percentual);
  const diff = max - min;

  if (diff < 10) {
    return 'Foco distribuído de forma equilibrada ao longo do dia.';
  }

  const dominante = this.faixaDominante;
  const labels: Record<string, string> = { manha: 'manhã', tarde: 'tarde', noite: 'noite' };
  const pct = Math.round(max);
  return `Maior concentração de foco pela ${labels[dominante!]} (${pct}% das sessões).`;
}

/** Arredonda percentual para inteiro. */
percentualInteiro(valor: number): number {
  return Math.round(valor);
}
```

#### 6. SCSS no `tendencia-foco.component.scss`

Seguir o design system (border-radius `14px`, azul-índigo para destaque, fonte 13px, cor `#64748b`):

```scss
.distribuicao-faixa {
  margin-top: 12px;
}

.distribuicao-pills {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
  margin-bottom: 6px;
}

.pill {
  display: inline-flex;
  align-items: center;
  padding: 3px 10px;
  border-radius: 14px;
  font-size: 13px;
  color: #64748b;
  border: 1px solid #e5e7eb;
  background: #f8fafc;
  font-weight: 400;

  &--destaque {
    background: #4f46e5;   // azul-índigo
    color: #fff;
    border-color: #4f46e5;
    font-weight: 500;
  }
}

.distribuicao-frase {
  font-size: 13px;
  color: #64748b;
  margin: 0;
  line-height: 1.4;
}
```

**Responsável frontend:** @Miles

---

### Tarefas ordenadas por dependência

| # | Tarefa | Responsável | Depende de |
|---|--------|-------------|------------|
| H10-BE-01 | Criar DTOs `FaixaHorariaResposta` e `DistribuicaoFocoPorFaixaResposta` | @Thor | — |
| H10-BE-02 | Adicionar campo `distribuicaoFocoPorFaixa` em `TendenciaPeriodoResposta` | @Thor | H10-BE-01 |
| H10-BE-03 | Refatorar `calcularSessoesFocoIndividuais` em `TendenciaPeriodoService` para capturar `horarioInicio` de cada sessão (estrutura interna — não exposta na API) | @Thor | — |
| H10-BE-04 | Implementar método `calcularDistribuicaoPorFaixa` em `TendenciaPeriodoService` | @Thor | H10-BE-01, H10-BE-03 |
| H10-BE-05 | Integrar `calcularDistribuicaoPorFaixa` no método `calcular()` e popular o campo no `TendenciaPeriodoResposta` retornado | @Thor | H10-BE-02, H10-BE-04 |
| H10-FE-01 | Adicionar interfaces `FaixaHoraria` e `DistribuicaoFocoPorFaixa` e estender `TendenciaPeriodoResponse` com campo opcional em `tab-resumo.component.ts` | @Miles | H10-BE-02 (contrato) |
| H10-FE-02 | Adicionar `@Input() distribuicao` no `TendenciaFocoComponent` e implementar getters `faixaDominante`, `fraseDistribuicao` e método `percentualInteiro` | @Miles | H10-FE-01 |
| H10-FE-03 | Adicionar bloco HTML da seção de distribuição (pills + frase) no `tendencia-foco.component.html` | @Miles | H10-FE-02 |
| H10-FE-04 | Adicionar estilos SCSS das pills e frase no `tendencia-foco.component.scss` | @Miles | H10-FE-03 |
| H10-FE-05 | Propagar `distribuicaoFoco` no `TabResumoComponent`: capturar em `atualizarTendencias()` e bindar no template | @Miles | H10-FE-02 |
| H10-QA-01 | Validar exibição das pills com dados reais (percentuais arredondados, pill dominante destacada, frase correta) | @Natasha | H10-FE-05 |
| H10-QA-02 | Validar caso sem sessões no período (seção não exibida) | @Natasha | H10-FE-05 |
| H10-QA-03 | Validar caso equilibrado (diferença < 10pp → frase "distribuído de forma equilibrada") | @Natasha | H10-FE-05 |

---

### Pontos de atenção específicos da H10

1. **Formato de `bloco.getInicio()` em `BlocoTemporalResposta`:** verificar se o campo é `"HH:mm"` ou `"HH:mm:ss"` antes de usar `LocalTime.parse`. Usar `DateTimeFormatter` explícito se o formato variar entre blocos.

2. **Sessão que cruza meia-noite:** raro mas possível (noite → madrugada). A faixa é classificada pelo **horário de início** da sessão, conforme a spec. Sessões que iniciam às 23:50 são `noite` mesmo que terminem na madrugada.

3. **Rollback de campo opcional:** o campo `distribuicaoFocoPorFaixa` é `@Nullable` no DTO e `optional` na interface TypeScript (`?`). Se por algum motivo o campo não vier na resposta (versão de backend desatualizada), o frontend não quebra — `*ngIf="distribuicao && totalSessoes > 0"` garante isso.

4. **Percentual arredondado para exibição:** o backend retorna `float` com casas decimais. O frontend arredonda com `Math.round()` para exibição — não truncar. A soma dos três arredondamentos pode resultar em 99% ou 101% em casos limite; isso é aceitável e equivale ao comportamento padrão de UI do design system.
