# Design Notes — Reestruturação das Abas
> Autor: @Groot (UX/UI Designer)
> Data: 2026-04-06
> Referência: spec.md (aprovada brainstorm 2026-04-06)

---

## Parecer Geral

O mockup está bem estruturado e alinhado com o design system do Manager. Os papéis das abas estão claramente diferenciados. As observações abaixo são ajustes de refinamento — nenhum é bloqueante para implementação.

---

## 1. Pergunta-guia no topo

### Avaliação: APROVADO com ajuste menor

A pergunta-guia como elemento visual de orientação é uma boa decisão de UX. Ela ancora o gestor no propósito da aba antes de processar os dados.

**Recomendação de estilo:**
```scss
.aba-question-guide {
  font-size: 18px;          // H3 — peso suficiente para ser percebida
  font-weight: 600;
  color: #1e293b;           // $header-color — tom de autoridade
  margin-bottom: 20px;
  line-height: 1.4;
}
```

**Atenção:** Não usar `font-size: 24px` (H2) — a pergunta é orientadora, não é o título da página. Não deve competir com o nome do colaborador no header superior. 18px com `font-weight: 600` é o equilíbrio certo.

**Espaçamento:** `margin-top: 0` se vier logo abaixo do seletor de data/período. `margin-bottom: 20px` antes dos KPIs.

---

## 2. Textos descritivos nos KPIs (Guia contextual B+A)

### Avaliação: APROVADO — spec correta, atenção na implementação

A spec define corretamente: `font-size: 12px`, `color: #94a3b8`, sem bold, uma linha apenas. Isso é adequado — é texto de suporte, não de destaque.

**O risco real é de poluição visual se mal posicionado.** Recomendações:

```scss
.kpi-description {
  font-size: 12px;
  color: #94a3b8;           // slate-400 — discreto mas legível
  font-weight: 400;
  line-height: 1.4;
  margin-top: 4px;          // colado ao valor do KPI, não flutuando
  display: block;
  white-space: nowrap;      // CRÍTICO: evitar quebra de linha no card
  overflow: hidden;
  text-overflow: ellipsis;
}
```

**Posição dentro do KPI card:**
```
┌─────────────────────────┐
│  Jornada Total           │  ← label, 12px #94a3b8
│  10h 21min               │  ← valor, 24px #1e293b bold
│  Do início ao fim do dia │  ← descrição, 12px #94a3b8
└─────────────────────────┘
```

O valor principal deve ser `font-size: 24px`, `font-weight: 700`. A descrição abaixo com 4px de gap. Sem essa hierarquia clara, o texto polui.

---

## 3. Frase acionável — card com borda lateral

### Avaliação: APROVADO com especificação de borda

A borda lateral colorida é o padrão de alert/insight do sistema — decisão correta. A cor da borda deve refletir o estado:

```scss
.actionable-insight {
  background: #f8fafc;           // fundo levemente off-white
  border: 1px solid #e5e7eb;    // $border-color padrão
  border-left: 4px solid;        // borda lateral de destaque
  border-radius: 14px;           // $border-radius do sistema
  padding: 12px 16px;
  margin: 16px 0;

  // Estados por tipo de insight:
  &.estado-alerta {
    border-left-color: #dc2626;  // vermelho — sobrecarga/excedido
  }
  &.estado-atencao {
    border-left-color: #f59e0b;  // amarelo — acima do habitual
  }
  &.estado-ok {
    border-left-color: #16a34a;  // verde — dentro do esperado
  }
  &.estado-neutro {
    border-left-color: #4f46e5;  // azul-índigo — informativo
  }

  .insight-text {
    font-size: 13px;
    color: #374151;              // slate-700 — legível mas não pesado
    font-weight: 400;
    line-height: 1.5;
  }
}
```

**Atenção:** `border-radius: 14px` com `border-left` mais espessa cria um efeito visual indesejado no canto superior esquerdo em alguns browsers. Solução: aplicar o `border-left` como `box-shadow` interno ou usar `border-left-width: 4px` com `border-radius` apenas nos cantos direitos via clip-path não — simplesmente testar no browser e ajustar se necessário. Na prática, `border-radius: 8px` funciona melhor para cards de alert com borda lateral.

**Revisão de border-radius:** Sugestão de usar `border-radius: 8px` neste componente específico (não 14px), pois cards de insight/alert com borda lateral têm melhor aparência com radius menor. O 14px fica bem para KPI cards e cards de conteúdo, mas não em banners de alerta.

---

## 4. Layout híbrido — Top Apps + Eventos lado a lado

### Avaliação: APROVADO — proporção 1:2 é correta

A proporção `flex: 1` (Top Apps) e `flex: 2` (Eventos) está bem dimensionada. O Top Apps é uma lista compacta de 5 itens; Eventos precisa de mais espaço para exibir timestamps e texto.

```scss
.hybrid-layout {
  display: flex;
  gap: 16px;
  align-items: flex-start;  // alinhar pelo topo, não esticar

  .top-apps-section {
    flex: 1;
    min-width: 0;            // evita overflow em flex
  }

  .events-section {
    flex: 2;
    min-width: 0;
  }

  // Responsivo — empilha em < 768px
  @media (max-width: 767px) {
    flex-direction: column;

    .top-apps-section,
    .events-section {
      flex: none;
      width: 100%;
    }
  }
}
```

**Ponto de atenção:** Em telas entre 768px e 1024px (tablets), a coluna de Top Apps pode ficar estreita demais com `flex: 1`. Considerar `min-width: 200px` para o Top Apps e `min-width: 320px` para Eventos, com breakpoint de empilhamento em 900px ao invés de 768px.

---

## 5. Gráficos de Tendência de Foco — barras empilhadas com cores por profundidade

### Avaliação: APROVADO com refinamento de paleta

A lógica de cores por profundidade de sessão está correta conceitualmente. Sugestão de refinamento para coerência com o design system (azul-índigo como primário):

```scss
// Paleta de foco — do mais raso ao mais profundo
$foco-raso:    #fef08a;   // amarelo — 5-20min (atenção fragmentada)
$foco-leve:    #a5b4fc;   // indigo-300 — 20-40min (produtivo)
$foco-bom:     #6366f1;   // indigo-500 — 40-60min (bom foco)
$foco-deep:    #3730a3;   // indigo-800 — 60min+ (deep work)
```

**Justificativa:** Usar a família indigo (em vez de azul genérico) mantém coerência com `$primary: #4f46e5`. O amarelo para sessões rasas cria contraste semântico claro (atenção vs. profundidade).

**Legenda obrigatória:** O gráfico PRECISA de legenda visual inline, abaixo do título. Sem ela, o gestor não consegue interpretar as cores.

```
● Raso (5-20min)  ● Produtivo (20-40min)  ● Bom foco (40-60min)  ● Deep work (60min+)
```

Fonte 11px, `color: #6b7280`, espaçamento `gap: 12px` entre os itens.

**Frase acionável abaixo do gráfico:** Manter em `font-size: 13px`, `color: #374151`. Separar do gráfico com `margin-top: 12px` e uma linha divisória `border-top: 1px solid #f1f5f9` (muito sutil).

---

## 6. Comparação com time nos KPIs do Resumo

### Avaliação: APROVADO — posição e estilo corretos

A linha de comparação abaixo do valor principal é o padrão mais limpo. Não competir com o valor principal é a prioridade.

**Especificação:**
```scss
.team-comparison {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-top: 6px;

  .team-label {
    font-size: 11px;
    color: #9ca3af;          // slate-400 — mais discreto que o texto de descrição
    font-weight: 400;
  }

  .comparison-indicator {
    font-size: 11px;
    font-weight: 600;

    &.acima {
      color: #16a34a;        // verde — acima do padrão (positivo para foco, negativo para ausência)
    }
    &.dentro {
      color: #9ca3af;        // neutro
    }
    &.abaixo {
      color: #dc2626;        // vermelho — abaixo do padrão
    }
  }
}
```

**Exemplo visual no card:**
```
┌─────────────────────────┐
│  Tempo Ativo Médio       │
│  6h 42min                │
│  Média do time: 5h 30min ↑ acima │
└─────────────────────────┘
```

**Atenção semântica:** "Acima" e "abaixo" têm valência diferente dependendo do KPI:
- Jornada alta → negativo (risco de sobrecarga)
- Tempo ativo alto → positivo
- Ausência alta → negativo
- Foco alto → positivo

O indicador visual (cor + seta) deve refletir essa semântica, não apenas o valor relativo. Isso é lógica de apresentação — o backend ou o frontend precisa saber a direção "desejável" de cada KPI para colorir corretamente.

---

## 7. Seletor "Personalizado ▾" discreto

### Avaliação: APROVADO — comportamento correto especificado

A spec define pills para Hoje/7dias/15dias/30dias e "Personalizado ▾" que abre calendar picker. O padrão de pills já existe no sistema, mas é importante garantir consistência visual:

```scss
.period-pills {
  display: flex;
  gap: 8px;
  align-items: center;
  flex-wrap: wrap;          // wrap em mobile

  .pill {
    padding: 6px 14px;
    border-radius: 20px;    // pill format
    font-size: 13px;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.15s ease;
    border: 1px solid #e5e7eb;
    background: white;
    color: #6b7280;

    &:hover {
      border-color: #4f46e5;
      color: #4f46e5;
    }

    &.active {
      background: #4f46e5;
      border-color: #4f46e5;
      color: white;
    }
  }

  // Pill "Personalizado" — visual idêntico mas com chevron
  .pill-custom {
    @extend .pill;

    &.has-custom-range {
      // Quando um range customizado está ativo, mostrar como "ativo"
      // mas com borda tracejada para indicar que é diferente das pills fixas
      border-style: dashed;
      background: #eef2ff;   // indigo-50
      color: #4f46e5;
      border-color: #4f46e5;
    }
  }
}
```

**O chevron "▾"** deve ser um ícone SVG pequeno (12px), não o caractere unicode, para melhor controle de alinhamento vertical.

**Quando personalizado está ativo:** Mostrar o range selecionado dentro ou ao lado da pill — ex: "12 Mar – 25 Mar ▾". Truncar se necessário com `max-width: 180px`.

---

## Resumo de Aprovações e Ajustes

| Item | Status | Prioridade do ajuste |
|------|--------|----------------------|
| Pergunta-guia | Aprovado | Garantir font-size 18px, não 24px |
| Textos descritivos KPIs | Aprovado | `white-space: nowrap` obrigatório |
| Frase acionável | Aprovado | Usar `border-radius: 8px` (não 14px) |
| Layout híbrido 1:2 | Aprovado | Revisar breakpoint em 900px |
| Barras foco empilhadas | Aprovado | Paleta indigo + legenda obrigatória |
| Comparação com time | Aprovado | Semântica de cor por KPI |
| Seletor Personalizado | Aprovado | Ícone SVG, estado "range ativo" |

---

## Nota final

O spec está maduro e implementável. Os ajustes acima são de refinamento visual — nenhum muda a estrutura ou o comportamento definido. Liberado para implementação com atenção aos pontos de CSS destacados acima.

**Próximo passo:** @Tony e time de frontend podem iniciar. Recomendo que o primeiro PR inclua os KPI cards com textos descritivos para validação visual antes de avançar para os gráficos.
