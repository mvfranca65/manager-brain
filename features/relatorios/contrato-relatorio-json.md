# Contrato JSON — Relatórios Semanais IA
> Documento de referência para o output esperado da LLM em cada tipo de relatório.
> Mantido por @Jarvis. Última atualização: 2026-04-06

Este contrato define a estrutura **exata** que a LLM deve retornar. Qualquer campo fora deste contrato será ignorado pelo backend. Campos obrigatórios ausentes causarão falha na validação (`validarJsonSaida`).

---

## 1. RelatorioIndividualJSON

### Visão geral

Gerado por `RelatorioIndividualService` a partir de `InputLlmIndividualDTO`. O JSON completo é armazenado em `relatorio_individual.relatorio_json` (TEXT/JSONB). O backend extrai `score_ia.valor` separadamente para a coluna `score_ia`.

### Schema completo

```json
{
  "score_ia": {
    "valor": 78,
    "sinais": {
      "desempenho": "BOM",
      "cansaco": "ATENCAO",
      "tendencia": "ESTAVEL"
    },
    "justificativa": "Semana sólida com foco alto e consistência mantida. O único ruído é o burnout risk chegando em 55 — ainda moderado, mas vale monitorar."
  },

  "diagnostico": {
    "resumo": "Ana teve uma semana acima da média. Foco e consistência juntos — isso não é coincidência, é ritmo. O que chama atenção é a carga levemente elevada na quinta e sexta, que pode estar puxando o burnout risk para cima.",
    "ponto_critico": "Burnout risk moderado (55) pelo segundo mês consecutivo. Não é alarme ainda, mas é sinal para não ignorar.",
    "contexto": "Semana com 3 dias de trabalho intenso seguidos — padrão que historicamente antecede quedas de consistência na semana seguinte."
  },

  "destaques_positivos": [
    {
      "area": "foco",
      "titulo": "Foco acima do esperado para o período",
      "narrativa": "Com score de foco em 82, Ana ficou acima da média histórica dela em 12 pontos. Dias longos não quebraram a concentração — isso é resiliência real."
    },
    {
      "area": "consistencia",
      "titulo": "Quarta semana consecutiva de consistência alta",
      "narrativa": "Consistência em 79 e quatro semanas seguidas acima de 70. Esse é o tipo de padrão que separa desempenho real de resultado de semana boa."
    }
  ],

  "alertas": [
    {
      "area": "saude",
      "severidade": "MODERADO",
      "titulo": "Burnout risk subindo pela segunda semana",
      "narrativa": "O indicador saiu de 48 para 55 em duas semanas. Individualmente ainda está moderado, mas a tendência é o que importa aqui.",
      "se_nada_mudar": "Se o padrão de carga da quinta e sexta se repetir, há risco real de queda de desempenho na próxima semana."
    }
  ],

  "foco_desta_semana": {
    "objetivo_principal": "Proteger o ritmo de foco sem aumentar a carga horária",
    "acoes": [
      {
        "acao": "Converse com Ana sobre a distribuição de tarefas no final da semana",
        "racional": "A carga concentrada em quinta e sexta é o principal fator que puxa o burnout risk para cima.",
        "prioridade": "ALTA",
        "evidencia": "Qui: 9.2h ativas (habitual: 7h). Sex: 8.8h com foco caindo para 68."
      },
      {
        "acao": "Avalie se há dependências externas forçando o trabalho concentrado no fim da semana",
        "racional": "Pode ser um problema de processo, não de comportamento individual.",
        "prioridade": "MEDIA",
        "evidencia": "Burnout risk subiu de 48 para 55 em duas semanas — padrão recorrente no fim da semana."
      }
    ]
  },

  "retrospectiva": {
    "resumo_semana_anterior": "Na semana passada o score ficou em 71, com foco caindo para 68 e consistência em 74. A tendência era de possível recuperação.",
    "evolucao": "MELHOROU",
    "membros_que_cumpriram_acoes": ["Manteve disciplina de horário de início"],
    "membros_que_nao_cumpriram_acoes": [],
    "observacao": "A recuperação do foco confirmou o diagnóstico da semana anterior — era uma queda pontual, não uma tendência."
  },

  "mensagem_ao_gestor": "Ana está em bom ritmo — mas o burnout risk em trajetória de alta é o detalhe que você não quer ignorar. Uma conversa rápida sobre a carga do fim da semana pode prevenir uma queda que, neste ponto, ainda dá para evitar."
}
```

### Definição de campos

#### `score_ia` — obrigatório

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `valor` | integer (0–100) | sim | Score holístico gerado pela LLM. É o único número calculado pela IA. |
| `sinais.desempenho` | string | sim | `"BOM"`, `"REGULAR"` ou `"BAIXO"`. Avaliação geral do desempenho. |
| `sinais.cansaco` | string | sim | `"OK"`, `"ATENCAO"` ou `"CRITICO"`. Sinal de cansaço/esgotamento. |
| `sinais.tendencia` | string | sim | `"SUBINDO"`, `"ESTAVEL"` ou `"CAINDO"`. Trajetória das últimas semanas. |
| `justificativa` | string (máximo 2 frases) | sim | Explicação humana do que define esse número. Não pode ser genérica. |

**Regra de teto:** se qualquer indicador psicológico estiver em nível crítico (>80), `valor` não pode ultrapassar 60.

#### `diagnostico` — obrigatório

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `resumo` | string (3–5 frases) | sim | Narrativa principal da semana. Tom direto, com opinião. Deve citar dados reais. |
| `ponto_critico` | string (1–2 frases) | sim | O detalhe mais importante que o gestor não pode ignorar. Pode ser positivo ou negativo. |
| `contexto` | string (1–2 frases) | não | Padrão comportamental ou temporal que explica o resultado. Omitir se não houver contexto relevante. |

#### `destaques_positivos` — obrigatório (array vazio permitido se semana genuinamente ruim)

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `area` | string | sim | Área de origem: `"atividade"`, `"foco"`, `"consistencia"`, `"saude"`, `"ritmo"` ou `"geral"`. |
| `titulo` | string (máx. 80 chars) | sim | Título do destaque. Específico — não genérico. |
| `narrativa` | string (2–4 frases) | sim | Análise do destaque. Explica o que os dados significam, não apenas o que mostram. |

**Regra de equilíbrio:** se a semana foi genuinamente forte em todos os pilares, o alerta pode ser uma oportunidade de desenvolvimento — nunca invente um problema. Se foi genuinamente ruim, o destaque positivo pode ser um ponto de partida. O equilíbrio é sobre honestidade, não sobre simetria forçada.

#### `alertas` — obrigatório, mínimo 1 item

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `area` | string | sim | Área de origem (mesmos valores de `destaques_positivos`). |
| `severidade` | string | sim | `"BAIXO"`, `"MODERADO"` ou `"ALTO"`. Alinhado ao nível dos indicadores psicológicos quando relevante. |
| `titulo` | string (máx. 80 chars) | sim | Título do alerta. Específico — não genérico. |
| `narrativa` | string (2–4 frases) | sim | Análise do problema. Explica causa provável, não apenas sintoma. |
| `se_nada_mudar` | string (1–2 frases) | não | O que tende a acontecer se o alerta não for endereçado. Omitir se não houver previsão razoável. |

#### `foco_desta_semana` — obrigatório

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `objetivo_principal` | string (1 frase) | sim | Direção central para o gestor nessa semana. |
| `acoes` | array | sim | Mínimo 1, máximo 3 ações. |
| `acoes[].acao` | string | sim | Começa com verbo no imperativo. Ex: "Converse", "Avalie", "Reduza". |
| `acoes[].racional` | string (1–2 frases) | sim | Por que essa ação importa agora. |
| `acoes[].prioridade` | string | sim | `"ALTA"`, `"MEDIA"` ou `"BAIXA"`. |
| `acoes[].evidencia` | string | sim | Dados específicos (dias, horas, scores) que justificam esta ação. |

#### `retrospectiva` — condicional

**Incluir apenas quando `relatorio_anterior` no input não for `null`.**
Se `relatorio_anterior` for `null`, o bloco `retrospectiva` deve ser completamente omitido do JSON.

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `resumo_semana_anterior` | string (2–3 frases) | sim | O que aconteceu na semana passada, em linguagem natural. |
| `evolucao` | string | sim | `"MELHOROU"`, `"ESTAVEL"` ou `"PIOROU"`. |
| `membros_que_cumpriram_acoes` | array de string | sim | Ações da semana anterior que foram cumpridas. Array vazio se nenhuma. |
| `membros_que_nao_cumpriram_acoes` | array de string | sim | Ações não cumpridas. Array vazio se todas foram cumpridas. |
| `observacao` | string (1–2 frases) | não | Interpretação da evolução. Omitir se não houver insight adicional. |

#### `mensagem_ao_gestor` — obrigatório

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| (raiz) | string (máx. 2 frases) | sim | Frase de encerramento. Tom de parceiro exigente. Deve ser a frase que fica na cabeça. |

---

## 2. RelatorioTimeJSON

### Visão geral

Gerado por `RelatorioTimeService` a partir de `InputLlmTimeDTO`. Recebe os relatórios individuais já gerados (`relatoriosIndividuais`) e dados agregados do time. O JSON é armazenado em `relatorio_time.relatorio_json`. O backend extrai `score_ia_time.valor` separadamente.

### Schema completo

```json
{
  "score_ia_time": {
    "valor": 72,
    "sinais": {
      "desempenho": "REGULAR",
      "cansaco": "ATENCAO",
      "tendencia": "ESTAVEL"
    },
    "justificativa": "Score médio razoável, mas três dos cinco membros com burnout risk acima de 50 ao mesmo tempo puxa o número para baixo."
  },

  "diagnostico_time": {
    "resumo": "O time entregou uma semana funcional, mas com sinais de pressão acumulada que os números individuais não capturam isoladamente. Quando três pessoas em um time de cinco chegam na sexta com carga acima do ideal, o problema é do time, não das pessoas.",
    "dinamica_central": "Carga mal distribuída com tendência a concentrar nas mesmas pessoas semana após semana.",
    "saude_coletiva": "MODERADA",
    "coesao_ritmo": "ASSINCRONA",
    "distribuicao_trabalho": "Ana e Carlos concentraram 60% das entregas críticas. Pedro e Mariana operam abaixo do potencial.",
    "padrao_oculto": "O time sincroniza nas segundas e terças, com forte assincronismo no fim da semana — isso cria gargalos de handoff."
  },

  "destaques_time": [
    {
      "titulo": "Consistência coletiva acima da média histórica",
      "narrativa": "Quatro dos cinco membros mantiveram consistência acima de 70 — o melhor resultado coletivo nos últimos dois meses. Isso é sinal de ritmo compartilhado, mesmo que frágil.",
      "membros_envolvidos": ["Ana", "Carlos", "Beatriz", "Luana"]
    }
  ],

  "alertas_time": [
    {
      "tipo": "SISTEMICO",
      "severidade": "ALTO",
      "titulo": "Pressão acumulada em múltiplos membros",
      "narrativa": "Três membros com burnout risk acima de 50 pelo segundo mês consecutivo. O time está operando perto do limite e a tendência não está revertendo sozinha.",
      "acao_requerida": true,
      "prazo_sugerido": "ESTA_SEMANA"
    },
    {
      "tipo": "INDIVIDUAL",
      "severidade": "MODERADO",
      "titulo": "Pedro em queda por duas semanas",
      "narrativa": "Queda de 15 pontos com padrão de foco fragmentado. Vale entender se é algo externo antes de agir.",
      "acao_requerida": true,
      "prazo_sugerido": "PROXIMA_SEMANA"
    }
  ],

  "plano_gestor": {
    "objetivo_semana": "Reequilibrar a carga e iniciar uma conversa coletiva sobre ritmo",
    "acoes": [
      {
        "acao": "Redistribua as entregas críticas da próxima semana — Ana e Carlos não podem concentrar mais do que 40% do volume",
        "dirigida_a": "TIME",
        "membros_envolvidos": ["Ana", "Carlos", "Pedro", "Mariana"],
        "racional": "A concentração de carga nas mesmas pessoas é o principal driver do burnout risk coletivo.",
        "prioridade": "ALTA",
        "evidencia": "Ana: 9.2h/dia média. Carlos: 8.8h/dia. Pedro e Mariana: abaixo de 6h/dia."
      },
      {
        "acao": "Agende um 1:1 com Pedro esta semana — queda de 15 pontos em duas semanas precisa de contexto",
        "dirigida_a": "INDIVIDUAL",
        "membros_envolvidos": ["Pedro"],
        "racional": "Antes de qualquer intervenção, entender a causa. Pode ser algo simples e resolvível.",
        "prioridade": "ALTA",
        "evidencia": "Score IA: 58 (era 73 há duas semanas). Foco caiu de 71 para 54."
      },
      {
        "acao": "Use Luana como referência na próxima sessão de alinhamento — compartilhe o padrão dela com o time",
        "dirigida_a": "TIME",
        "membros_envolvidos": ["Luana"],
        "racional": "Quem se destacou tem algo que o time pode aprender. Reconhecer isso em grupo cria incentivo.",
        "prioridade": "MEDIA",
        "evidencia": "Score 88, consistência acima de 70 pelo terceiro mês consecutivo."
      }
    ],
    "sem_acao_risco": "Se a carga permanecer concentrada, Ana e Carlos vão mostrar sinais de queda nas próximas duas semanas."
  },

  "retrospectiva_time": {
    "resumo_semana_anterior": "Na semana passada o score do time estava em 68, com distribuição de carga já desequilibrada e dois alertas sistêmicos ativos.",
    "evolucao_time": "LEVE_MELHORA",
    "membros_que_responderam": ["Ana", "Luana"],
    "membros_que_regrediram": ["Pedro"],
    "observacao": "A melhora do score médio está concentrada em Ana e Luana — o restante do time ficou estável ou piorou. Ainda não há recuperação coletiva."
  },

  "mensagem_ao_gestor": "O time tem capacidade — o problema é como a demanda está sendo distribuída. Agir nessa semana na redistribuição de carga não é microgerenciamento: é o que evita uma queda coletiva que seria muito mais difícil de reverter."
}
```

### Definição de campos

#### `score_ia_time` — obrigatório

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `valor` | integer (0–100) | sim | Score holístico do time gerado pela LLM. |
| `sinais.desempenho` | string | sim | `"BOM"`, `"REGULAR"` ou `"BAIXO"`. Avaliação geral do desempenho do time. |
| `sinais.cansaco` | string | sim | `"OK"`, `"ATENCAO"` ou `"CRITICO"`. Sinal de cansaço/esgotamento coletivo. |
| `sinais.tendencia` | string | sim | `"SUBINDO"`, `"ESTAVEL"` ou `"CAINDO"`. Trajetória coletiva das últimas semanas. |
| `justificativa` | string (máximo 2 frases) | sim | Explicação humana do que define esse número para o time. |

**Regra de teto:** se 50% ou mais dos membros tiverem algum indicador psicológico em nível crítico, `valor` não pode ultrapassar 60.

#### `diagnostico_time` — obrigatório

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `resumo` | string (3–5 frases) | sim | Narrativa principal do time nessa semana. Foco no coletivo, não nos indivíduos. |
| `dinamica_central` | string (1–2 frases) | sim | O padrão mais importante que explica o estado atual do time. |
| `saude_coletiva` | string | sim | `"BOA"`, `"MODERADA"`, `"CRITICA"`. Avaliação geral da saúde psicológica coletiva. |
| `coesao_ritmo` | string | sim | `"SINCRONA"`, `"ASSINCRONA"`, `"MISTA"`. Descreve como o time opera temporalmente. |
| `distribuicao_trabalho` | string (1–2 frases) | sim | Como o trabalho está distribuído entre os membros. |
| `padrao_oculto` | string (1–2 frases) | não | Padrão que só aparece olhando o time todo. Omitir se não houver padrão relevante. |

#### `destaques_time` — obrigatório, mínimo 1 item

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `titulo` | string (máx. 80 chars) | sim | Título do destaque coletivo. Específico. |
| `narrativa` | string (2–4 frases) | sim | O que esse destaque significa para o time como organismo. |
| `membros_envolvidos` | array de string | não | Membros que contribuíram para o destaque. Omitir se for algo difuso. |

#### `alertas_time` — obrigatório, mínimo 1 item

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `tipo` | string | sim | `"SISTEMICO"` (afeta o time) ou `"INDIVIDUAL"` (pessoa específica que o gestor precisa atender). |
| `area` | string | sim | Área de origem: `"atividade"`, `"foco"`, `"consistencia"`, `"saude"`, `"ritmo"` ou `"geral"`. |
| `severidade` | string | sim | `"BAIXO"`, `"MODERADO"` ou `"ALTO"`. |
| `titulo` | string (máx. 80 chars) | sim | Título do alerta. Específico. |
| `narrativa` | string (2–4 frases) | sim | Análise com causa provável e contexto. |
| `acao_requerida` | boolean | sim | Se requer ação do gestor. |
| `prazo_sugerido` | string | não | `"ESTA_SEMANA"` ou `"PROXIMA_SEMANA"`. Omitir se `acao_requerida` for `false`. |

#### `plano_gestor` — obrigatório

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `objetivo_semana` | string (1 frase) | sim | Direção central para o gestor nessa semana em relação ao time. |
| `acoes` | array | sim | Mínimo 1, máximo 4 ações. |
| `acoes[].acao` | string | sim | Começa com verbo no imperativo. |
| `acoes[].dirigida_a` | string | sim | `"TIME"` (ação coletiva) ou `"INDIVIDUAL"` (ação com pessoa específica). |
| `acoes[].membros_envolvidos` | array de string | sim | Quem está envolvido. Para ações de time, lista todos os membros afetados. |
| `acoes[].racional` | string (1–2 frases) | sim | Por que essa ação importa agora. |
| `acoes[].prioridade` | string | sim | `"ALTA"`, `"MEDIA"` ou `"BAIXA"`. |
| `acoes[].evidencia` | string | sim | Dados específicos (dias, horas, scores) que justificam esta ação. |
| `sem_acao_risco` | string (1–2 frases) | sim | O que acontece se o gestor não agir. Tom realista, não alarmista. |

#### `retrospectiva_time` — condicional

**Incluir apenas quando `relatorio_time_anterior` no input não for `null`.**
Se `relatorio_time_anterior` for `null`, o bloco `retrospectiva_time` deve ser completamente omitido do JSON.

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `resumo_semana_anterior` | string (2–3 frases) | sim | Estado do time na semana passada. |
| `evolucao_time` | string | sim | `"MELHOROU"`, `"LEVE_MELHORA"`, `"ESTAVEL"`, `"LEVE_PIORA"` ou `"PIOROU"`. |
| `membros_que_responderam` | array de string | sim | Membros que melhoraram em relação à semana anterior. Array vazio se nenhum. |
| `membros_que_regrediram` | array de string | sim | Membros que pioraram. Array vazio se nenhum. |
| `observacao` | string (1–2 frases) | não | Interpretação da evolução coletiva. |

#### `mensagem_ao_gestor` — obrigatório

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| (raiz) | string (máx. 2 frases) | sim | Encerramento do relatório. Tom de parceiro exigente. A frase que fica na cabeça. |

---

## Referências cruzadas

- Input individual: `InputLlmIndividualDTO` → `DadosSemanaDTO` → `PilarDTO` + `IndicadorPsicologicoDTO`
- Input time: `InputLlmTimeDTO` → `relatoriosIndividuais` (lista de RelatorioIndividualJSON já gerados)
- Prompts: `individual-system.txt`, `time-system.txt`
- Entidades de persistência: `RelatorioIndividual`, `RelatorioTime`
- Áreas possíveis: `atividade`, `foco`, `consistencia`, `saude`, `ritmo`, `geral`
- Indicadores psicológicos possíveis: `burnout_risk`, `desengajamento`, `sobrecarga`
- Níveis de indicador: `"BAIXO"`, `"MODERADO"`, `"ALTO"`, `"INDETERMINADO"`
