# ADR-001: Modelo de 3 estados de atividade

> Data: 2026-03-26
> Status: Aprovado
> Decisores: @Steve (PO), @Tony (TL), @Estranho (SM)

## Contexto

O modelo atual usa 4 estados (Ativo, Pausa, Sem atividade, Offline) para classificar a atividade do colaborador. A distincao entre "Sem atividade" (agente rodando sem uso) e "Offline" (PC desligado) e tecnica e nao agrega valor para o gestor. Ambos significam a mesma coisa na pratica: o colaborador nao esta trabalhando.

## Decisao

Simplificar para **3 estados**:

| Estado | Cor | Significado para o gestor | Regra tecnica |
|---|---|---|---|
| **Ativo** | Verde | Colaborador trabalhando | Janela ativa, interacao detectada |
| **Pausa** | Amarelo | Ausencia curta, esperada | LOCK, ociosidade <= 10 min, gaps de 10-30 min entre eventos |
| **Ausente** | Cinza | Nao esta trabalhando | Ociosidade > 10 min, sem heartbeat > 30 min, LOGOUT, PC desligado, gaps > 30 min |

### Regras complementares aprovadas

**Gap-filling entre eventos:**
- Gaps < 10 min entre eventos: ignorar (normal entre janelas)
- Gaps de 10-30 min: classificar como **Pausa**
- Gaps > 30 min: classificar como **Ausente**

**Jornada esperada vs real:**
- Mostrar a jornada **real** (span primeiro->ultimo evento)
- Quando exceder a jornada esperada (campo `jornadaHoras` do cadastro): indicador visual (valor em vermelho ou icone de alerta)
- Nao usar como teto — mostrar dados reais

### Formula de KPIs
- `Jornada = Ativo + Pausa + Ausente` (span completo)
- Os 3 percentuais somam 100%

## Consequencias

### Backend
- `mapearTipoBloco`: renomear OFFLINE -> AUSENTE
- `expandirBloco`: split de OCIOSIDADE > 10 min continua, mas a parte que era OFFLINE vira AUSENTE
- Novo: gap-filling entre blocos (gaps 10-30 min -> PAUSA, gaps > 30 min -> AUSENTE)
- Novo: flag `jornadaExcedida` na resposta quando jornada real > jornadaHoras
- Constantes: `PAUSA_OCIOSIDADE_MAX_SEGUNDOS` (10 min) e `OFFLINE_THRESHOLD_MINUTES` (30 min) mantem os mesmos valores

### Frontend
- Visao Geral: 3 KPIs (Ativo, Pausa, Ausente) em vez de 4
- Donut: 3 fatias em vez de 3+fundo
- Indicadores: 3 cards + 3 cores no grafico
- Legenda do grafico: 3 itens em vez de 4
- Indicador visual quando jornada excede o esperado

### O que NAO muda
- Thresholds numericos (10 min pausa, 30 min ausente)
- Regra de LOCK = Pausa
- Regra de LOGOUT = Ausente (antes era Offline)
- Filtro de janelas < 30s
