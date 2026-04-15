# Fix: Poluição Visual — KPIs e Seções
> Autor: @Groot (UX/UI)
> Para: @Miles (implementação)
> Data: 2026-04-06
> Prioridade: Alta — feedback direto do fundador

---

## Contexto

Dois problemas visuais identificados na aba **Dia a Dia** (componente `tab-dia-dia`) e na aba **Resumo** (componente `tab-resumo`) do detalhe do colaborador:

1. Textos descritivos nos KPIs e cabeçalhos de seção sempre visíveis, truncando com "..." — poluição visual desnecessária.
2. Pergunta-guia "O que aconteceu hoje?" e "Qual o padrão desse colaborador?" estão genéricas, sem personalidade.

---

## MUDANÇA 1 — Remover textos descritivos visíveis dos KPI cards

### Arquivos afetados
- `tab-dia-dia.component.html`
- `tab-resumo.component.html`

### O que fazer

**Em `tab-dia-dia.component.html`** — remover o atributo `descricao` de todos os 4 `app-kpi-card`:

```html
<!-- REMOVER esta linha de cada kpi-card: -->
descricao="Quanto tempo durou o dia de trabalho. Acima do esperado pode indicar sobrecarga."
descricao="Tempo usando o computador. Compare com a jornada para ver o aproveitamento."
descricao="Pequenas pausas durante o dia. Pausas saudáveis são normais e esperadas."
descricao="Períodos longos sem atividade. Verifique se está dentro do esperado."
```

Os `tooltipTexto` de cada card DEVEM ser mantidos — eles aparecem só no hover do "?" e continuam acessíveis.

**Em `tab-resumo.component.html`** — idem, remover `descricao` dos 4 `app-kpi-card`:

```html
<!-- REMOVER estas linhas: -->
descricao="Média diária de horas trabalhadas no período. Se estiver consistentemente alta, pode indicar necessidade de redistribuir demandas."
descricao="Média de tempo produtivo por dia. Compare com a jornada para avaliar o aproveitamento."
descricao="Média de pausas por dia. Pouca pausa pode indicar pressão excessiva; muita pausa pode indicar desengajamento."
descricao="Maior bloco de trabalho contínuo. Sessões longas de foco indicam capacidade de concentração e ambiente sem interrupções."
```

> O `tooltipTexto` no card de "Foco médio" (`tab-resumo`) DEVE ser mantido.

### O que NÃO mexer no kpi-card.component
- Manter a prop `@Input() descricao` no componente — pode ser usada futuramente.
- Manter o bloco `<span *ngIf="descricao" class="kpi-card__descricao">` no template — simplesmente não vai renderizar quando `descricao` for vazio.
- Manter toda a classe `.kpi-card__descricao` no SCSS — não remover.

---

## MUDANÇA 2 — Remover textos descritivos das seções (cabeçalhos de card)

### Arquivos afetados
- `tab-dia-dia.component.html`
- `tab-resumo.component.html`

### Textos a remover em `tab-dia-dia.component.html`

**Seção Visualização Temporal** (linha ~123):
```html
<!-- REMOVER esta linha inteira: -->
<span class="secao-descricao">Veja como o dia foi distribuído. Blocos verdes são trabalho, amarelos são pausas e cinzas são ausências. Gaps frequentes podem indicar interrupções.</span>
```

Adicionar um "?" discreto ao lado do título, com tooltip explicativo:
```html
<!-- SUBSTITUIR: -->
<span class="timeline-header">Visualização Temporal</span>

<!-- POR: -->
<span class="timeline-header">
  Visualização Temporal
  <span
    class="secao-help"
    pTooltip="Blocos verdes = atividade, amarelos = pausa, cinza = ausência. Gaps frequentes podem indicar interrupções."
    tooltipStyleClass="menu-tooltip"
    tooltipPosition="top">
    ?
  </span>
</span>
```

**Seção Top 5 Apps** (linha ~202):
```html
<!-- REMOVER esta linha inteira: -->
<span class="secao-descricao">Em quais ferramentas o colaborador passou mais tempo. Útil para entender o tipo de trabalho realizado no dia.</span>
```

Adicionar "?" ao lado do título:
```html
<!-- SUBSTITUIR: -->
<span class="section-title">Top 5 Apps do dia</span>

<!-- POR: -->
<span class="section-title">
  Top 5 Apps do dia
  <span
    class="secao-help"
    pTooltip="Ferramentas com maior tempo de uso. Útil para entender o tipo de trabalho realizado no dia."
    tooltipStyleClass="menu-tooltip"
    tooltipPosition="top">
    ?
  </span>
</span>
```

**Seção Eventos Detalhados** (linha ~266):
```html
<!-- REMOVER esta linha inteira: -->
<span class="secao-descricao">Tudo que aconteceu no dia, em ordem. Use para entender o contexto de uma pausa ou ausência específica.</span>
```

Adicionar "?" ao lado do título:
```html
<!-- SUBSTITUIR: -->
<span class="eventos-title">Eventos Detalhados</span>

<!-- POR: -->
<span class="eventos-title">
  Eventos Detalhados
  <span
    class="secao-help"
    pTooltip="Linha do tempo completa do dia, em ordem cronológica. Use para entender o contexto de pausas ou ausências específicas."
    tooltipStyleClass="menu-tooltip"
    tooltipPosition="top">
    ?
  </span>
</span>
```

### Textos a remover em `tab-resumo.component.html`

Remover todos os blocos `<div class="section-card__desc">...</div>` das seções abaixo (manter apenas o título e o "?" já existente onde aplicável):

**Tendência — Jornada por dia** (linha ~87):
```html
<!-- REMOVER: -->
<div class="section-card__desc">
  Evolução da jornada ao longo dos dias. Dias vermelhos excederam o esperado. Veja se é pontual ou um padrão.
</div>
```
Mover a informação para o tooltip do "?". Adicionar span `secao-help` ao lado de `section-card__title`:
```html
<!-- SUBSTITUIR o div do título por: -->
<div class="section-card__title">
  Tendência — Jornada por dia
  <span
    class="secao-help"
    pTooltip="Evolução da jornada ao longo dos dias. Dias vermelhos excederam o esperado. Veja se é pontual ou um padrão recorrente."
    tooltipStyleClass="menu-tooltip"
    tooltipPosition="top">
    ?
  </span>
</div>
```

**Top 5 apps (período)** (linha ~116):
```html
<!-- REMOVER: -->
<div class="section-card__desc">
  Ferramentas mais usadas no período. Mudanças bruscas podem indicar mudança de projeto ou responsabilidade.
</div>
```
Adicionar "?" ao lado do título com o mesmo texto no tooltip.

**Distribuição de Tempo** (linha ~162):
```html
<!-- REMOVER: -->
<div class="section-card__desc">
  Proporção entre trabalho, pausas e ausências no período. Uma distribuição saudável tem pausas regulares.
</div>
```
Adicionar "?" ao lado do título com o mesmo texto no tooltip.

**Tendência — Sessões de Foco por dia** (linha ~289):
```html
<!-- REMOVER: -->
<div class="section-card__desc">
  Cada barra mostra as sessões individuais de foco do dia. Cores indicam a profundidade: quanto mais escuro, mais longa a sessão.
</div>
```
O "?" já existe nessa seção (`section-card__tooltip-trigger`) — apenas adicionar essa descrição ao tooltip existente, concatenando:
```html
pTooltip="Uma sessão de foco é um período de trabalho contínuo sem interrupções maiores que 2 minutos. Só sessões acima de 5 minutos são contadas. Cores mais escuras = sessões mais longas."
```

---

## MUDANÇA 3 — Novo estilo para o "?" de seção (`.secao-help`)

### Arquivo afetado
- `tab-dia-dia.component.scss`
- `tab-resumo.component.scss`

Adicionar a classe `.secao-help` nos dois arquivos SCSS (pode ser idêntica em ambos):

```scss
/* ── Help icon de seção ──────────────────────────────────────────────────── */

.secao-help {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 15px;
  height: 15px;
  border-radius: 50%;
  background: #f1f5f9;
  color: #94a3b8;
  font-size: 9px;
  font-weight: 700;
  cursor: help;
  line-height: 1;
  vertical-align: middle;
  margin-left: 5px;
  transition: background 0.12s, color 0.12s;
  flex-shrink: 0;

  &:hover {
    background: #e2e8f0;
    color: #475569;
  }
}
```

Notar que esse estilo é intencionalmente menor (15px, font-size 9px) do que o `.kpi-card__help` (16px, font-size 10px) para ficar ainda mais discreto nos títulos de seção.

---

## MUDANÇA 4 — Pergunta-guia: novo texto e ícone

### Arquivo afetado
- `tab-dia-dia.component.html` — pergunta da aba Dia a Dia
- `tab-resumo.component.html` — pergunta da aba Resumo

### tab-dia-dia.component.html

```html
<!-- SUBSTITUIR: -->
<div class="pergunta-guia">
  <i class="pi pi-sun pergunta-guia__icon"></i>
  <span class="pergunta-guia__texto">O que aconteceu hoje?</span>
</div>

<!-- POR: -->
<div class="pergunta-guia">
  <i class="pi pi-search pergunta-guia__icon"></i>
  <span class="pergunta-guia__texto">Analise com atenção: o que aconteceu hoje?</span>
</div>
```

### tab-resumo.component.html

```html
<!-- SUBSTITUIR: -->
<div class="pergunta-guia">
  <span class="pergunta-guia__texto">Qual o padrão desse colaborador?</span>
</div>

<!-- POR: -->
<div class="pergunta-guia">
  <i class="pi pi-lightbulb pergunta-guia__icon"></i>
  <span class="pergunta-guia__texto">Analise com atenção: qual o padrão desse colaborador?</span>
</div>
```

### CSS ajuste — `tab-resumo.component.scss`

A pergunta-guia no `tab-resumo` provavelmente não tem o estilo `.pergunta-guia__icon` definido (o componente original não tinha o ícone). Verificar se o SCSS do `tab-resumo` já contém `.pergunta-guia__icon` — se não tiver, adicionar:

```scss
/* ── Pergunta-guia ───────────────────────────────────────────────────────── */
/* (verificar se já existe antes de adicionar) */

.pergunta-guia {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
  background: #f8fafc;
  border-radius: 10px;
  padding: 14px 20px;
  border: 1px solid #e5e7eb;
}

.pergunta-guia__icon {
  font-size: 16px;
  color: #6366f1;  /* índigo — diferente do amarelo do dia-a-dia para distinguir as abas */
  flex-shrink: 0;
}

.pergunta-guia__texto {
  font-size: 15px;
  font-weight: 600;
  color: #1e293b;
}
```

No `tab-dia-dia.component.scss`, apenas atualizar a cor do ícone de `.pergunta-guia__icon` de `#f59e0b` (amarelo/sol) para `#6366f1` (índigo), já que o ícone muda de `pi-sun` para `pi-search`:

```scss
/* ALTERAR em tab-dia-dia.component.scss: */
.pergunta-guia__icon {
  font-size: 16px;
  color: #6366f1;  /* era #f59e0b */
  flex-shrink: 0;
}
```

---

## Resumo das mudanças por arquivo

| Arquivo | Ação |
|---------|------|
| `tab-dia-dia.component.html` | Remover `descricao=""` dos 4 kpi-cards; remover 3 `secao-descricao`; adicionar `secao-help` com tooltip nos 3 títulos de seção; atualizar pergunta-guia (ícone + texto) |
| `tab-dia-dia.component.scss` | Adicionar `.secao-help`; alterar cor de `.pergunta-guia__icon` para `#6366f1` |
| `tab-resumo.component.html` | Remover `descricao=""` dos 4 kpi-cards; remover 4 `section-card__desc`; adicionar `secao-help` em 3 títulos (Foco já tem "?"); atualizar tooltip do "?" de Foco; atualizar pergunta-guia (ícone + texto) |
| `tab-resumo.component.scss` | Adicionar `.secao-help`; adicionar/verificar `.pergunta-guia__icon` com cor `#6366f1` |
| `kpi-card.component.html` | Nenhuma alteração |
| `kpi-card.component.ts` | Nenhuma alteração |
| `kpi-card.component.scss` | Nenhuma alteração |

---

## O que NÃO mudar

- Os `tooltipTexto` dos kpi-cards em `tab-dia-dia` — MANTER TODOS. O "?" com tooltip é a nova forma de expor a explicação.
- O `tooltipTexto` do card "Foco médio" em `tab-resumo` — MANTER.
- O `section-card__tooltip-trigger` já existente na seção "Tendência — Sessões de Foco" — MANTER, só atualizar o texto do `pTooltip`.
- Os tooltips das `legend-item` na timeline e no donut — MANTER.
- Nenhum comportamento lógico / TypeScript é afetado.

---

## Notas de QA

Após implementar, verificar:
1. Nenhum texto truncado "..." aparece nos KPI cards.
2. O ícone "?" aparece em todos os títulos de seção que tiveram texto removido.
3. Hover no "?" abre tooltip com o texto correto.
4. A pergunta-guia renderiza o ícone correto e o novo texto nas duas abas.
5. O layout dos KPI cards não quebra por terem menos conteúdo (gap menor esperado).
