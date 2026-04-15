# Produto — Manager
> Domínio: @Steve | PO
#produto #negocio #icp
> Leitura obrigatória para: todos os agentes
> Última atualização: 2026-04-14

---

## O que é o Manager

Plataforma SaaS B2B de **gestão de produtividade e monitoramento de atividade** de colaboradores em estações de trabalho. Atende empresas com times remotos ou híbridos que precisam de visibilidade real sobre engajamento, foco e saúde do ritmo de trabalho — sem invadir privacidade.

**O problema central:** Gestores de times remotos/híbridos operam no escuro. Não sabem quem está engajado, quem está sobrecarregado silenciosamente, quem está desengajando, ou se o ritmo que estão pedindo é sustentável. O Manager transforma dados brutos de atividade em diagnósticos semanais com linguagem direta, ações concretas e accountability semana a semana — decisões baseadas em evidência, não em intuição.

---

## ICP — Cliente Ideal

- Empresa com **20–500 colaboradores**
- Times **remotos ou híbridos**
- Setores: tecnologia, contabilidade, atendimento, vendas internas, back-office financeiro
- Decisor: Gestor de RH, CEO, COO ou gerente de operações
- Já tentou Toggl/Clockify e não conseguiu adoção
- Precisa de visibilidade **sem vigilância** — quer dados, não câmeras

**Não é nosso cliente:** empresas puramente presenciais, cultura de vigilância extrema (keylogger/câmera), microempresas (<5 colaboradores)

---

## Diferenciais competitivos

1. **Relatório semanal por IA** — diagnóstico completo por colaborador e time, com linguagem de "colega experiente", não de dashboard
2. **Score IA** — nota única atribuída pela IA que interpreta os dados objetivos + tendência + sinais de risco. Fonte única de verdade
3. **Plano de ação com accountability** — cada relatório propõe ações concretas e na semana seguinte cobra se foram cumpridas
4. **Retrospectiva automática** — compara semana atual vs anterior, identifica melhoras e pioras, cobra ações pendentes
5. **6 indicadores psicológicos** — burnout risk, desengajamento, sobrecarga silenciosa, instabilidade de ritmo, fadiga de FDS, horas extras recorrentes
6. **5 pilares calculados objetivamente** — atividade, foco, consistência, saúde, fragmentação (+ anomalias no MVP2)
7. **Hierarquia organizacional nativa** — relatórios sobem na cadeia automaticamente
8. **Sem keylogger, sem câmera** — posicionamento ético. Monitora janela ativa e ociosidade, nunca conteúdo
9. **PDF profissional exportável** — relatórios individuais, do time e consolidado com visual estruturado

---

## Como funciona (fluxo resumido)

```
Agent Desktop (Win/macOS)
    → captura janela ativa + ociosidade em batch
    → envia para srv-events (Device JWT)

Pipeline semanal (srv-ia, toda segunda 00h):
    1. ExtratorDadosService  → agrega eventos da semana por colaborador
    2. CalculadoraPilares     → calcula 5 pilares (0-100)
    3. CalculadoraIndicadores → calcula 6 indicadores psicológicos (0-100)
    4. LlmService             → chama a IA para gerar diagnóstico, alertas, plano de ação
    5. Persiste               → JSONB no PostgreSQL

Portal (srv-portal + Angular):
    → gestor acessa visão do time, lista de membros, relatório individual
    → navega entre semanas, exporta PDF
```

---

## Score IA vs IManager Score

| | Score IA | IManager Score |
|---|---|---|
| **O que é** | Nota de 0-100 atribuída pela IA | Média ponderada dos pilares (fórmula) |
| **Fonte** | LLM interpreta pilares + tendência + contexto | Cálculo determinístico |
| **Exibição** | Score principal no portal e PDF | "Score Fórmula" nos detalhes técnicos |
| **Quando usar** | Sempre — é a fonte única de verdade | Backup silencioso; aparece se Score IA falhar |

**Decisão (2026-04-09):** O Score IA foi mais preciso em 3 dos 10 cenários críticos testados. A fórmula permite compensação excessiva entre pilares. Guardrail: se score_ia vier fora de 0-100 ou null, fallback para imanagerScore.

---

## Pilares — Scores Calculados (0-100)

| Pilar | Peso | O que mede | Como calcula |
|-------|------|-----------|--------------|
| Atividade | 25% | Horas produtivas vs jornada esperada | Ratio horas/dia vs contratado. Pico em ratio=1.0, penaliza overwork e underwork |
| Foco | 25% | Profundidade das sessões de trabalho | Duração média dos blocos + % de blocos deep (≥25min) |
| Consistência | 20% | Regularidade diária ao longo da semana | 100 - CV×120 (coeficiente de variação das horas/dia) |
| Saúde | 15% | Pausas adequadas e equilíbrio | Ratio de pausa vs atividade + jornada na faixa ideal |
| Fragmentação | 10% | Penaliza troca excessiva de apps | Switches/hora: ≤4=100, >20=próximo de 0 |
| Anomalias | 5% | Picos ou vales anormais | Hardcoded 100 no MVP1 — implementação real no MVP2 |

**Regras importantes:**
- A IA nunca calcula pilares — só interpreta os valores já calculados
- Fator de confiabilidade: se diasComDados < 3, todos os scores × (dias/3)
- diasComDados = 0 → status SEM_DADOS, LLM não é chamado

---

## Indicadores Psicológicos (0-100)

| Indicador | O que detecta | Classificação |
|-----------|--------------|---------------|
| Burnout Risk | Esforço excessivo + pausas insuficientes + continuidade em blocos longos | ≤30 BAIXO, ≤60 MODERADO, >60 ALTO |
| Desengajamento | Queda progressiva de atividade semana a semana (precisa de histórico) | Idem. INDETERMINADO sem histórico |
| Sobrecarga Silenciosa | Alta atividade + baixa fragmentação + baixa saúde (sem sinais clássicos de burnout) | Idem. Suprimido se burnout ≥60 |
| Instabilidade de Ritmo | Dias com horas muito diferentes (CV alto entre dias da semana) | Idem |
| Fadiga de Fim de Semana | Queda de desempenho seg→sex E/OU trabalho no fim de semana | Idem. FDS é sempre alerta, nunca mérito |
| Horas Extras Recorrentes | Padrão de múltiplas semanas com ratio > 1.15 | Idem. 25pts por semana em excesso |

**Regra do teto:** Se qualquer indicador psicológico > 80 (crítico), o Score IA não pode ultrapassar 60.

---

## Relatório Individual — O que contém

Gerado por semana, por colaborador. Estrutura do JSON:

| Seção | Conteúdo |
|-------|----------|
| **Score IA** | Valor 0-100 + sinais (desempenho, cansaço, tendência) + justificativa |
| **Diagnóstico** | Resumo (3 frases), ponto crítico, contexto temporal |
| **Alertas** | 1-2 alertas com severidade (ALTO/MODERADO/BAIXO), narrativa, "se nada mudar" |
| **Foco da Semana** | Objetivo principal + 1-3 ações imperativas com prioridade, racional e evidência |
| **Destaques Positivos** | O que foi bom (pode ser vazio em semanas genuinamente ruins) |
| **Retrospectiva** | Evolução vs semana anterior (MELHOROU/ESTÁVEL/PIOROU) + ações cumpridas/não cumpridas |
| **Mensagem ao Gestor** | 2 frases — tom de "parceiro direto", a frase que fica na cabeça |

**Linguagem:** direta, sem jargão técnico, nomes completos sempre, termos proibidos definidos nos prompts.

---

## Relatório do Time — O que contém

Gerado após todos os individuais da semana. Não resume os individuais — interpreta o coletivo.

| Seção | Conteúdo |
|-------|----------|
| **Score IA Time** | Média ponderada + interpretação coletiva |
| **Diagnóstico Time** | Resumo, dinâmica central, saúde coletiva, coesão de ritmo, distribuição de trabalho, padrão oculto |
| **Alertas Time** | SISTÊMICO (afeta vários) ou INDIVIDUAL (alguém precisa de atenção urgente) |
| **Plano Gestor** | 1-4 ações com dirigida_a (TIME/INDIVIDUAL), membros envolvidos, evidência |
| **Destaques** | Quem se destacou e por quê |
| **Retrospectiva Time** | Evolução coletiva + membros que responderam/regrediram |
| **Mensagem ao Gestor** | Fecha o relatório com urgência e direção |

---

## Modelo de negócio

- **SaaS B2B multitenancy** — cliente é a empresa, não o colaborador
- **Plano Básico:** monitoramento + portal + relatório semanal IA + PDF export
- **Plano Plus:** tudo + screenshots periódicos ("tela ao vivo")
- Precificação: a definir

---

## Componentes do sistema

| Componente | Descrição |
|-----------|-----------|
| **Agent Desktop** | App Windows/macOS que captura janela ativa + ociosidade em batch. Sem keylogger, sem câmera |
| **srv-events** (:8080) | Ingestão de eventos. Device JWT. Persiste no PostgreSQL |
| **srv-admin** (:8081) | Gestão de tenants, API keys, ativação de agentes |
| **srv-portal** (:8082) | Backend do portal. JWT. APIs de relatórios, hierarquia, PDF export |
| **srv-ia** (:8085) | Pipeline de geração de relatórios. Cálculos + LLM + persistência |
| **fed-portal** (:4200) | Frontend Angular 16 + PrimeNG. SPA com lazy loading |
| **PostgreSQL** | Banco compartilhado. Multitenancy por empresa_id/tenant_id |

---

## Estágio atual (abril 2026)

- **MVP1 em validação final**
- Pipeline de relatórios end-to-end funcionando (extração → cálculos → LLM → persistência → portal → PDF)
- 10 cenários de teste validados com 3 semanas de dados controlados
- Relatórios individuais, do time e consolidado com export PDF profissional
- Retrospectiva entre semanas com accountability
- Sem clientes pagantes ainda
- Empresa em estruturação (fundador solo: Marcos)

### Pendências para MVP2
- Pilar Anomalias (hoje hardcoded 100)
- Calibração do Score IA com feedback do gestor
- Dashboard de audit logs / observabilidade
- Email com template Thymeleaf (Task 11)
- Tratamento refinado de fim de semana
- Variação de linguagem entre relatórios gerados em sequência
