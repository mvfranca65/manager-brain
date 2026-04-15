# Reestruturação das Abas — Detalhe do Colaborador

> Spec aprovada em brainstorm 2026-04-06
> Participantes: @Estranho (SM), @Steve (PO), Fundador

---

## Problema

As abas "Visão Geral" e "Indicadores" têm KPIs duplicados e papéis sobrepostos. O gestor precisa navegar entre abas pra ter a visão completa. A sobreposição confunde e quebra o fluxo de análise.

## Decisão

Reorganizar em **2 abas com papéis distintos**, eliminando redundância:

| Aba | Nome | Pergunta central | Propósito |
|-----|------|-------------------|-----------|
| 1 (default) | **Dia a Dia** | "O que aconteceu hoje?" | Visão detalhada de um dia específico |
| 2 | **Resumo** | "Qual o padrão desse colaborador?" | Visão agregada por período |
| 3 | Timeline 360° | — | Em breve (não alterada) |
| 4 | Detalhes | — | Dados cadastrais (não alterada) |

---

## Aba "Dia a Dia" (default)

### Navegação
- Botões "Dia anterior" / "Próximo dia"
- Seletor de data com calendário
- Default: hoje

### Componentes (ordem de cima pra baixo)

1. **Pergunta-guia** — texto visível: "O que aconteceu hoje?"
2. **KPIs do dia** — 4 cards em linha:
   - Jornada total (com alerta se excedida)
   - Tempo ativo (verde)
   - Tempo em pausa (amarelo)
   - Tempo ausente (cinza) — subtítulo "inclui almoço"
3. **Frase acionável do dia** — card com borda colorida, gerado automaticamente a partir dos KPIs. Exemplos:
   - "Jornada 10h 21min — 2h 21min acima do esperado. Verifique se há sobrecarga."
   - "Dia dentro do esperado. 87% de aproveitamento ativo."
   - "Tempo ausente acima do habitual — 1h 20min vs média de 45min."
4. **Visualização Temporal** — barra horizontal 24h com blocos coloridos (verde/amarelo/cinza)
5. **Seção inferior — layout híbrido (lado a lado):**
   - Esquerda (1/3): **Top 5 Apps do dia** — lista com barras proporcionais
   - Direita (2/3): **Eventos Detalhados** — lista paginada (10/30/50/100)

### Layout
- KPIs + Timeline: largura total
- Top Apps + Eventos: lado a lado (flex, 1:2)
- Responsivo: empilha em telas < 768px

---

## Aba "Resumo"

### Seletor de período
- Pills visíveis: Hoje / 7 dias / 15 dias / 30 dias
- Pill "Personalizado ▾": abre calendar picker para período custom (escondido por padrão)
- Default: 7 dias

### Componentes (ordem de cima pra baixo)

1. **Pergunta-guia** — texto visível: "Qual o padrão desse colaborador?"
2. **KPIs médios** — 4 cards em linha:
   - Jornada média (com alerta se excedida)
   - Tempo ativo médio (verde)
   - Tempo em pausa médio (amarelo)
   - Foco médio (verde) — subtítulo "maior sessão"
   - **Cada KPI inclui comparação com o time:** linha discreta abaixo do valor mostrando "Média do time: Xh Xmin". Indicador visual: acima/dentro/abaixo do padrão.
3. **Tendência — Jornada por dia** — gráfico de barras verticais, uma por dia do período. Barra vermelha quando excede jornada esperada (8h default).
4. **Seção inferior — layout lado a lado:**
   - Esquerda: **Top 5 Apps (período)** — lista com barras proporcionais
   - Direita: **Distribuição de Tempo (donut)** — Ativo/Pausa/Ausente com percentuais
5. **Tendência — Sessões de Foco por dia** — gráfico de barras verticais, uma por dia. Subtítulo: "X sessões no período · maior: Xh Xmin"

---

## Guia contextual (B + A)

### Texto descritivo por seção
Cada seção tem uma linha de texto descritivo abaixo do título:
- Fonte: 12px, cor `#94a3b8`, sem bold
- Sempre visível, sem interação necessária
- Uma linha apenas — objetivo é orientar, não explicar em detalhe

### Textos definidos

Cada texto segue o padrão: **o que é → como ler → quando agir**.
Linguagem simples, sem termos técnicos. O gestor deve entender sem precisar de treinamento.

#### Aba "Dia a Dia"

| Seção | Texto visível | Tooltip "?" (quando houver) |
|-------|--------------|----------------------------|
| Jornada | Quanto tempo durou o dia de trabalho, do início ao fim. Se estiver acima do esperado, pode indicar sobrecarga. | Calculado do primeiro ao último evento detectado. A jornada esperada é configurada no perfil do colaborador (padrão: 8h). |
| Tempo ativo | Quanto tempo o colaborador esteve usando o computador. Compare com a jornada para ver o aproveitamento do dia. | Inclui digitação, cliques e uso de aplicativos. Pausas menores que 5 minutos não interrompem o tempo ativo. |
| Tempo em pausa | Pequenas pausas durante o dia. Pausas saudáveis são normais e esperadas. | Detectado automaticamente quando não há atividade por mais de 5 minutos. Pausas acima de 30 minutos viram "ausente". |
| Tempo ausente | Períodos longos sem atividade, como almoço ou saída antecipada. Verifique se está dentro do esperado. | Inclui almoço, reuniões fora do computador e qualquer intervalo acima de 30 minutos. |
| Visualização Temporal | Veja como o dia foi distribuído. Blocos verdes são trabalho, amarelos são pausas e cinzas são ausências. Gaps frequentes podem indicar interrupções. | — |
| Top 5 Apps | Em quais ferramentas o colaborador passou mais tempo. Útil para entender o tipo de trabalho realizado no dia. | — |
| Eventos Detalhados | Tudo que aconteceu no dia, em ordem. Use para entender o contexto de uma pausa ou ausência específica. | — |

#### Aba "Resumo"

| Seção | Texto visível | Tooltip "?" (quando houver) |
|-------|--------------|----------------------------|
| Jornada média | Média diária de horas trabalhadas no período. Se estiver consistentemente alta, pode indicar necessidade de redistribuir demandas. | — |
| Tempo ativo médio | Média de tempo produtivo por dia. Compare com a jornada para avaliar o aproveitamento. | — |
| Tempo em pausa médio | Média de pausas por dia. Pouca pausa pode indicar pressão excessiva; muita pausa pode indicar desengajamento. | — |
| Foco médio | Maior bloco de trabalho contínuo. Sessões longas de foco indicam capacidade de concentração e ambiente sem interrupções. | Uma sessão de foco é um período de trabalho contínuo sem interrupções maiores que 2 minutos. Só sessões acima de 5 minutos são contadas. |
| Tendência Jornada | Evolução da jornada ao longo dos dias. Dias vermelhos excederam o esperado. Veja se é pontual ou um padrão. | — |
| Distribuição de Tempo | Proporção entre trabalho, pausas e ausências no período. Uma distribuição saudável tem pausas regulares. | — |
| Top 5 Apps (período) | Ferramentas mais usadas no período. Mudanças bruscas podem indicar mudança de projeto ou responsabilidade. | — |
| Tendência Foco | Cada barra mostra as sessões individuais de foco do dia. Cores indicam a profundidade: quanto mais escuro, mais longa a sessão. | Uma sessão de foco é um período de trabalho contínuo sem interrupções maiores que 2 minutos. Só sessões acima de 5 minutos são contadas. |

### Gráfico de Tendência de Foco — Detalhes

**Formato:** Barras empilhadas por dia. Cada segmento = uma sessão individual de foco.

**Cores por duração da sessão:**
- 🟡 Amarelo claro: 5-20min (raso)
- 🔵 Azul claro: 20-40min (produtivo)
- 🔵 Azul médio: 40-60min (bom foco)
- 🔵 Azul escuro: 60min+ (deep work)

**Frase acionável:** Gerada automaticamente abaixo do gráfico com base no padrão do período.

Exemplos de frases:
- "3 de 5 dias com sessões fragmentadas — verifique a agenda de reuniões deste colaborador."
- "2 de 3 dias com foco equilibrado ou profundo — bom padrão de concentração."
- "Melhor semana de foco do mês — padrão estável."
- "Queda de foco nos últimos 3 dias — possível sobrecarga ou excesso de interrupções."

**Regras de classificação por dia (internas, usadas para gerar a frase):**
- **Fragmentado:** mais de 5 sessões E maior sessão < 30min
- **Equilibrado:** 2-5 sessões OU maior sessão entre 30-60min
- **Deep work:** alguma sessão > 60min E menos de 4 sessões no dia

### Ícone "?" com tooltip
- Aparece apenas nas seções que têm conteúdo na coluna "Tooltip" acima
- Tooltip usa classe `menu-tooltip` (design system existente)
- Posição: ao lado do título da seção, alinhado à direita

---

## O que muda vs estado atual

| Componente | Antes | Depois |
|-----------|-------|--------|
| Abas | "Visão Geral" + "Indicadores" | "Dia a Dia" + "Resumo" |
| KPIs | Duplicados entre abas | Cada aba tem seus KPIs sem overlap |
| Top Apps | Só na Visão Geral (período) | Dia a Dia (dia) + Resumo (período) |
| Donut | Na Visão Geral | Só no Resumo |
| Timeline temporal | Só nos Indicadores | Só no Dia a Dia |
| Eventos detalhados | Só nos Indicadores | Só no Dia a Dia |
| Tendência jornada | Não existia | Novo no Resumo |
| Tendência foco | Não existia | Novo no Resumo |
| Pergunta-guia | Não existia | Topo de cada aba |
| Textos descritivos | Não existia | Em cada seção |
| Subtítulo "inclui almoço" | Não existia | Card Tempo ausente |

---

## APIs necessárias

### Existentes (já funcionam)
- `POST /api/eventos/{id}/timeline/completo` — dados do dia (KPIs + blocos + eventos)
- `POST /api/eventos/{id}/resumo-periodo` — dados do período (KPIs médios + top apps)

### Novas necessárias
- **Tendência de jornada por dia** — array com jornada/ativo/pausa/ausente de cada dia do período
- **Tendência de foco por dia** — array com sessões de foco de cada dia (quantidade, duração individual, tempo total)
- **Média do time** — KPIs médios do time do colaborador no mesmo período (jornada, ativo, pausa, foco) para comparação
- **Frase acionável** — lógica backend que analisa os dados e gera texto de insight (pode ser calculada no frontend inicialmente)

> Decisão do @Tony: esses dados podem vir como extensão do `resumo-periodo` (campo `tendenciaPorDia`) ou endpoint separado.

---

## Restrições técnicas
- Angular 16 + PrimeNG
- Design system: cards brancos, border `#e5e7eb`, radius `14px`
- Gráficos de tendência: pode usar PrimeNG Charts (Chart.js) ou barras CSS puras
- Responsivo: layout híbrido empilha em mobile (< 768px)

---

## Fora de escopo
- Timeline 360° (permanece "em breve")
- Aba Detalhes (sem alterações)
- Relatórios IA (escopo separado)
