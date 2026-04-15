# Histórias de Usuário — Reestruturação das Abas (Detalhe do Colaborador)

> Autor: @Steve (PO)
> Data: 2026-04-06
> Baseado na spec aprovada em brainstorm 2026-04-06

---

## H1 — Reestruturação das abas

Como gestor, quero que as abas da tela de detalhe do colaborador sejam renomeadas e tenham papéis distintos, para que eu saiba exatamente o que vou encontrar em cada aba antes de clicar.

**Critérios de aceitação:**
- [ ] A aba "Visão Geral" é renomeada para "Dia a Dia"
- [ ] A aba "Indicadores" é renomeada para "Resumo"
- [ ] "Dia a Dia" é a aba default (aberta ao entrar na tela)
- [ ] As abas "Timeline 360°" e "Detalhes" permanecem sem alterações
- [ ] A ordem das abas é: Dia a Dia | Resumo | Timeline 360° | Detalhes
- [ ] Nenhum KPI ou componente aparece duplicado entre as duas abas

---

## H2 — Aba Dia a Dia: pergunta-guia, KPIs do dia, textos descritivos e tooltips

Como gestor, quero ver uma pergunta-guia, os KPIs do dia em cards e textos descritivos em cada seção, para que eu entenda o que estou lendo sem precisar de treinamento.

**Critérios de aceitação:**
- [ ] A pergunta-guia "O que aconteceu hoje?" aparece visível no topo da aba
- [ ] São exibidos 4 cards de KPI em linha: Jornada total, Tempo ativo, Tempo em pausa, Tempo ausente
- [ ] O card "Jornada total" exibe alerta visual quando o valor excede a jornada esperada
- [ ] O card "Tempo ativo" usa cor verde
- [ ] O card "Tempo em pausa" usa cor amarela
- [ ] O card "Tempo ausente" usa cor cinza e exibe subtítulo "inclui almoço"
- [ ] Cada seção da aba exibe uma linha de texto descritivo (fonte 12px, cor `#94a3b8`, sem bold, sem interação)
- [ ] Os textos descritivos seguem exatamente o conteúdo definido na spec (coluna "Texto visível")
- [ ] As seções com conteúdo de tooltip exibem ícone "?" ao lado do título, alinhado à direita
- [ ] O tooltip usa a classe `menu-tooltip` do design system
- [ ] Os tooltips exibem exatamente o conteúdo definido na spec (coluna "Tooltip ?")
- [ ] Seções sem tooltip não exibem o ícone "?"
- [ ] A navegação de datas exibe botões "Dia anterior" e "Próximo dia"
- [ ] Há um seletor de data com calendário
- [ ] O dia default ao abrir a aba é hoje

---

## H3 — Aba Dia a Dia: frase acionável automática do dia

Como gestor, quero ver uma frase acionável gerada automaticamente com base nos KPIs do dia, para que eu saiba se preciso agir e qual é o ponto de atenção sem precisar interpretar os números sozinho.

**Critérios de aceitação:**
- [ ] A frase acionável é exibida em um card com borda colorida, abaixo dos 4 KPIs
- [ ] A frase é gerada automaticamente com base nos KPIs do dia selecionado
- [ ] Quando a jornada excede o esperado, a frase menciona o excedente e sugere verificar sobrecarga (ex.: "Jornada 10h 21min — 2h 21min acima do esperado. Verifique se há sobrecarga.")
- [ ] Quando o dia está dentro do esperado, a frase informa o percentual de aproveitamento ativo (ex.: "Dia dentro do esperado. 87% de aproveitamento ativo.")
- [ ] Quando o tempo ausente excede a média habitual, a frase compara com a média (ex.: "Tempo ausente acima do habitual — 1h 20min vs média de 45min.")
- [ ] A cor da borda do card reflete o tipo de situação (alerta, positivo, neutro)
- [ ] A frase muda ao navegar entre datas

---

## H4 — Aba Dia a Dia: layout híbrido com Top Apps e Eventos lado a lado

Como gestor, quero ver o Top 5 Apps do dia e os Eventos Detalhados lado a lado na parte inferior da aba, para que eu tenha contexto de ferramentas e eventos sem precisar rolar entre seções separadas.

**Critérios de aceitação:**
- [ ] A seção inferior exibe dois painéis em layout flex lado a lado
- [ ] O painel esquerdo (1/3 da largura) exibe o Top 5 Apps do dia com barras proporcionais
- [ ] O painel direito (2/3 da largura) exibe a lista de Eventos Detalhados em ordem cronológica
- [ ] A lista de Eventos suporta paginação com opções de 10, 30, 50 e 100 itens por página
- [ ] Os KPIs e a Visualização Temporal (barra 24h) ocupam largura total acima dos dois painéis
- [ ] A barra de Visualização Temporal exibe blocos coloridos: verde (ativo), amarelo (pausa), cinza (ausente)
- [ ] Em telas com largura menor que 768px, os painéis empilham verticalmente (Top Apps acima, Eventos abaixo)
- [ ] Os textos descritivos de "Top 5 Apps" e "Eventos Detalhados" são exibidos conforme spec

---

## H5 — Aba Resumo: pergunta-guia e KPIs médios com comparação do time

Como gestor, quero ver os KPIs médios do colaborador no período com comparação em relação à média do time, para que eu entenda se o padrão de trabalho está dentro, acima ou abaixo do esperado para o grupo.

**Critérios de aceitação:**
- [ ] A pergunta-guia "Qual o padrão desse colaborador?" aparece visível no topo da aba
- [ ] São exibidos 4 cards de KPI médio: Jornada média, Tempo ativo médio, Tempo em pausa médio, Foco médio
- [ ] O card "Jornada média" exibe alerta visual quando a média excede a jornada esperada
- [ ] O card "Foco médio" usa cor verde e exibe subtítulo "maior sessão"
- [ ] Cada card exibe, abaixo do valor principal, uma linha discreta com "Média do time: Xh Xmin"
- [ ] Cada card exibe um indicador visual de posição: acima, dentro ou abaixo do padrão do time
- [ ] Os textos descritivos de cada seção seguem exatamente o conteúdo da spec
- [ ] O tooltip "?" aparece nos cards que possuem conteúdo de tooltip definido na spec

---

## H6 — Aba Resumo: gráfico de tendência de jornada por dia

Como gestor, quero ver a evolução da jornada do colaborador ao longo dos dias do período selecionado, para que eu identifique se dias com excesso de horas são pontuais ou um padrão recorrente.

**Critérios de aceitação:**
- [ ] O gráfico de tendência de jornada exibe barras verticais, uma por dia do período
- [ ] Barras cujo valor excede a jornada esperada (padrão: 8h) são exibidas em vermelho
- [ ] Barras dentro do esperado são exibidas na cor padrão (sem alerta)
- [ ] O gráfico atualiza ao mudar o período selecionado
- [ ] O texto descritivo da seção é exibido conforme spec: "Evolução da jornada ao longo dos dias. Dias vermelhos excederam o esperado. Veja se é pontual ou um padrão."

---

## H7 — Aba Resumo: gráfico de tendência de foco com barras empilhadas e frase acionável

Como gestor, quero ver as sessões de foco de cada dia do período em um gráfico de barras empilhadas por profundidade, acompanhado de uma frase acionável gerada automaticamente, para que eu entenda a qualidade de concentração do colaborador e saiba quando intervir.

**Critérios de aceitação:**
- [ ] O gráfico exibe barras verticais empilhadas, uma por dia, onde cada segmento representa uma sessão individual de foco
- [ ] As cores dos segmentos seguem a classificação por duração:
  - Amarelo claro: sessões de 5 a 20 minutos (foco raso)
  - Azul claro: sessões de 20 a 40 minutos (produtivo)
  - Azul médio: sessões de 40 a 60 minutos (bom foco)
  - Azul escuro: sessões acima de 60 minutos (deep work)
- [ ] O subtítulo do gráfico exibe "X sessões no período · maior: Xh Xmin"
- [ ] Abaixo do gráfico, uma frase acionável é gerada automaticamente com base no padrão do período
- [ ] A frase segue as regras de classificação por dia definidas na spec:
  - Fragmentado: mais de 5 sessões E maior sessão < 30min
  - Equilibrado: 2 a 5 sessões OU maior sessão entre 30 e 60min
  - Deep work: alguma sessão > 60min E menos de 4 sessões no dia
- [ ] Exemplos de frases geradas correspondem aos cenários da spec (fragmentado, equilibrado, queda, melhor semana)
- [ ] O tooltip "?" exibe: "Uma sessão de foco é um período de trabalho contínuo sem interrupções maiores que 2 minutos. Só sessões acima de 5 minutos são contadas."
- [ ] O gráfico atualiza ao mudar o período selecionado

---

## H8 — Aba Resumo: seletor de período com opção "Personalizado" discreta

Como gestor, quero selecionar o período de análise da aba Resumo usando pills visíveis e uma opção discreta de período personalizado, para que o fluxo padrão seja rápido e a flexibilidade esteja disponível quando necessário.

**Critérios de aceitação:**
- [ ] As pills de período são exibidas como: Hoje | 7 dias | 15 dias | 30 dias
- [ ] O período default ao abrir a aba é "7 dias"
- [ ] A pill "Personalizado ▾" abre um calendar picker para seleção de intervalo customizado
- [ ] A pill "Personalizado" está visível mas discreta, sem chamar mais atenção que as outras opções
- [ ] O calendar picker fica oculto por padrão e aparece apenas ao clicar em "Personalizado"
- [ ] Todos os componentes da aba Resumo atualizam ao trocar o período
- [ ] A pill do período ativo exibe estado visual selecionado

---

## H9 — APIs backend: tendência por dia, média do time e dados de foco detalhados

Como sistema, preciso de endpoints que retornem a tendência de jornada por dia, as sessões de foco por dia e a média do time no período, para que o frontend da aba Resumo tenha todos os dados necessários sem múltiplas chamadas desnecessárias.

**Critérios de aceitação:**
- [ ] Existe um endpoint (ou extensão do `resumo-periodo` existente) que retorna array com jornada/ativo/pausa/ausente por dia do período
- [ ] Existe um endpoint (ou campo `tendenciaPorDia`) que retorna, por dia, a quantidade de sessões de foco, a duração individual de cada sessão e o tempo total de foco
- [ ] Existe um endpoint que retorna os KPIs médios do time do colaborador no mesmo período (jornada média, ativo médio, pausa média, foco médio)
- [ ] Os endpoints aceitam parâmetros de ID do colaborador e intervalo de datas (data início / data fim)
- [ ] Os dados de foco retornam somente sessões acima de 5 minutos
- [ ] A sessão de foco é calculada como período de trabalho contínuo sem interrupções superiores a 2 minutos
- [ ] A resposta é consistente independentemente de o dado vir como endpoint separado ou extensão do `POST /api/eventos/{id}/resumo-periodo`
- [ ] Os contratos de resposta são documentados antes da implementação frontend começar

---

## H10 — Aba Resumo: distribuição de horários de foco por faixa horária

Como gestor, quero ver em quais faixas horárias o colaborador costuma ter mais sessões de foco no período selecionado, para que eu entenda o padrão de concentração dele ao longo do dia e possa proteger esses horários de reuniões ou interrupções.

**Critérios de aceitação:**

### Backend (API)

- [ ] O endpoint `POST /api/eventos/{id}/tendencia-periodo` (ou extensão do `resumo-periodo`) passa a retornar um campo adicional `distribuicaoFocoPorFaixa` no payload de resposta
- [ ] O campo `distribuicaoFocoPorFaixa` é um objeto com três chaves fixas: `manha`, `tarde`, `noite`
- [ ] As faixas horárias são definidas como:
  - `manha`: sessões iniciadas entre 06:00 e 11:59
  - `tarde`: sessões iniciadas entre 12:00 e 17:59
  - `noite`: sessões iniciadas entre 18:00 e 05:59
- [ ] Cada faixa retorna: `totalSessoes` (int), `tempoTotalMinutos` (int), `percentual` (float 0–100)
- [ ] O percentual é calculado sobre o total de sessões do período (soma dos três = 100%)
- [ ] Somente sessões de foco acima de 5 minutos são contabilizadas (mesmo critério da H9)
- [ ] Quando não há sessões no período, todas as faixas retornam zero
- [ ] Formato esperado da resposta:
```json
{
  "distribuicaoFocoPorFaixa": {
    "manha":  { "totalSessoes": 18, "tempoTotalMinutos": 420, "percentual": 52.9 },
    "tarde":  { "totalSessoes": 12, "tempoTotalMinutos": 240, "percentual": 35.3 },
    "noite":  { "totalSessoes":  4, "tempoTotalMinutos":  80, "percentual": 11.8 }
  }
}
```

### Frontend (UI)

- [ ] A distribuição é exibida dentro do card "Tendência — Sessões de Foco" existente na aba Resumo, abaixo da frase acionável
- [ ] A exibição é uma linha simples com 3 pills/badges lado a lado: "Manhã X%", "Tarde X%", "Noite X%"
- [ ] A pill da faixa com maior percentual recebe destaque visual (cor de fundo preenchida azul-índigo; as demais ficam com fundo neutro/outline)
- [ ] Abaixo das pills, uma frase descritiva automática indica o padrão dominante (ex.: "Maior concentração de foco pela manhã (53% das sessões)")
- [ ] Quando todas as faixas têm percentual próximo (diferença < 10pp entre a maior e a menor), a frase exibe: "Foco distribuído de forma equilibrada ao longo do dia"
- [ ] Quando não há sessões no período, a seção de distribuição não é exibida
- [ ] Os percentuais exibidos são arredondados para inteiro (sem casas decimais)
- [ ] A seção atualiza ao trocar o período selecionado (pills de 7/15/30 dias ou personalizado)
- [ ] A linha de distribuição usa fonte 13px, cor `#64748b`, e as pills seguem o border-radius padrão do design system (14px)
