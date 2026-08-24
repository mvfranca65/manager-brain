# Handoff para @Shuri — backend Node (2026-08-19)

> **De:** @Tony | **Para:** @Shuri
> **Origem:** regressivo automatizado do Agent Windows 1.5.7/1.5.8 contra staging
> **Evidencia completa:** `agent-desktop/ACHADOS-REGRESSIVO-2026-08-18.md`
> **Regra do Marcos:** nao mexemos mais no Java. O que estiver em servico Java migra ou
> espera; o que for Node e seu.

Tudo abaixo foi medido no banco de staging com SELECT, em transacao READ ONLY. Nenhuma
escrita foi feita.

---

## 1. A-49 — uma linha em `agentes` por instalacao, e o fingerprint nao e usado

**Severidade: ALTA.** Fragmenta o historico do colaborador entre varios ids, o que afeta
relatorio, nao so contagem de frota.

### O que foi medido

```sql
SELECT nome_maquina,
       COUNT(*)                             AS linhas,
       COUNT(DISTINCT instalacao_id)        AS instalacoes,
       COUNT(DISTINCT hardware_fingerprint) AS fingerprints
FROM agentes
WHERE nome_maquina = 'DESKTOP-VMSM6LE'
GROUP BY nome_maquina;
```

```
DESKTOP-VMSM6LE | 12 linhas | 12 instalacao_id distintos | 1 fingerprint distinto
```

Nenhuma das 12 tem `desvinculado_em` preenchido. As linhas cobrem de 03/07 a 18/08:

```
id=39 1.1.0.0   id=43 1.3.3.0   id=47 1.4.0.0   id=51 1.5.7.0 -> hoje 1.5.8.0
id=40 1.1.1.0   id=44 1.3.4.0   id=48 1.5.1.0
id=41 1.3.0.0   id=45 1.3.7.0   id=50 1.5.4.0
id=42 1.3.2.0   id=46 1.3.9.0
```

### O diagnostico

O `instalacao_id` muda a cada instalacao. O `hardware_fingerprint` — que existe exatamente
para reconhecer a mesma maquina — **e estavel nas 12 linhas** e nao esta sendo usado na
reconciliacao.

**Observacao importante que restringe o escopo:** a troca de binario in-place de 1.5.7 para
1.5.8 (feita hoje, preservando o `ProgramData`) **nao** criou linha nova — o id 51 foi
reaproveitado e so o `versao_agente` mudou. Ou seja, o gatilho nao e "toda versao nova"; e
alguma coisa que regenera o `instalacao_id`. Vale descobrir o que, porque a linha id=51
nasceu em 18/08 17:41, sem reinstalacao manual naquele momento.

### O que precisa ser decidido/feito

1. **Reconciliar por `hardware_fingerprint`** quando o `instalacao_id` for desconhecido mas
   o fingerprint ja existir para a empresa: atualizar a linha existente em vez de inserir.
   Cuidado: fingerprint pode colidir em VM clonada — combinar com `empresa_id` e, se
   preciso, `nome_maquina`.
2. **Marcar `desvinculado_em`** nas linhas antigas da mesma maquina ao reconhecer a nova.
   Hoje as 12 aparecem como ativas, o que infla qualquer contagem de frota.
3. **Investigar quem regenera o `instalacao_id`.** Se o gatilho for algo do Agent, volta
   para o @Bucky; se for o backend nao reconhecendo o id enviado, e seu.

Antes de mexer, alinhar com @Tony: mudar chave de reconciliacao de agente e decisao de
arquitetura, nao refactor local.

---

## 2. A-27 — `publicado_em` gravado no futuro

**Severidade: MEDIA.** Ja era conhecido do regressivo de 18/08 e **continua acontecendo**.

Medido em 2026-08-19, comparando com `now()` do proprio banco:

```
versao 1.0.27 -> publicado_em 2026-08-19T15:49:51Z  (+0.9h no futuro)
versao 1.0.26 -> publicado_em 2026-08-19T15:42:52Z  (+0.7h no futuro)
versao 1.0.23 -> publicado_em 2026-08-19T02:03:36Z  (passado, coerente)
versao 1.0.22 -> publicado_em 2026-08-19T01:21:22Z  (passado, coerente)
```

As duas mais recentes estao adiantadas; as antigas nao. O desvio nao e um offset fixo de 3h
como se suspeitou em 18/08, entao a hipotese "gravou horario local como UTC" nao explica
sozinha. Vale olhar o endpoint de publicacao e como o timestamp e montado.

**O que fazer:** gravar UTC explicitamente na publicacao e corrigir as linhas ja existentes.
Enquanto estiver no futuro, qualquer regra de rollout que compare `publicado_em` com `now()`
se comporta de forma imprevisivel.

---

## 3. Passivo de eventos abertos — precisa de decisao antes de script

**Severidade: ALTA** pelo efeito em relatorio.

Varredura de 7 dias, base inteira:

| Sintoma | Ocorrencias |
|---|---|
| `eventos_ociosidade` com `finalizado_em IS NULL` ha mais de 30min | **33** |
| `eventos_janela` com `finalizado_em IS NULL` ha mais de 30min | **27** |

Origem: achado A-48 (reinicio do worker durante ociosidade duplicava o evento e deixava a
primeira linha aberta) somado ao A-35. **O A-48 esta corrigido no Agent a partir da 1.5.8**,
comprovado em maquina: onde a 1.5.7 gerava duas linhas com 18ms de diferenca, a 1.5.8 gera
uma so.

Mas a correcao **nao fecha as linhas que ja existem**. Enquanto elas estiverem abertas, os
pilares Atividade e Saude leem tempo que nao aconteceu.

**O que precisa ser decidido (com @Tony e @Steve, nao unilateral):**

- Fechar retroativamente com que criterio? Sugestao: `finalizado_em = iniciado_em` para
  duplicatas evidentes (existe outra linha do mesmo agente iniciando a menos de 2s, ja
  fechada), e descartar as demais em vez de inventar duracao.
- Ou marcar como invalidas em vez de fechar, preservando o rastro.

Consulta que identifica os pares duplicados:

```sql
SELECT a.id AS id_orfa, b.id AS id_boa, a.agente_id,
       EXTRACT(EPOCH FROM (b.iniciado_em - a.iniciado_em)) * 1000 AS delta_ms
FROM eventos_ociosidade a
JOIN eventos_ociosidade b
  ON a.agente_id = b.agente_id
 AND a.id <> b.id
 AND b.iniciado_em BETWEEN a.iniciado_em - interval '2 seconds'
                       AND a.iniciado_em + interval '2 seconds'
WHERE a.finalizado_em IS NULL
  AND b.finalizado_em IS NOT NULL;
```

---

## 4. Contexto que talvez seja seu

Estes tres nao estao confirmados como Node, mas aparecem no caminho do Agent e valem
verificacao quando voce olhar o `srv-events-node`:

- **UNIQUE de `eventos_ociosidade` e `(agente_id, iniciado_em)`.** O A-48 mostrou que
  diferenca de milissegundos fura essa chave. A correcao do Agent resolve a causa, mas uma
  tolerancia no upsert (reconciliar por janela de 1s, por exemplo) seria defesa em
  profundidade. **Nao implementar sem alinhar com @Tony** — mexer em chave de reconciliacao
  tem risco de fundir eventos legitimos.
- **`eventos_sessao` nao tem UNIQUE** (registrado no briefing da 1.5.5 como A-33), entao
  duplicata ali entra no banco sem barreira.
- **A-47 (LGPD):** o titulo da janela pode carregar conteudo digitado — o Bloco de Notas do
  Win11 nomeia a aba nao-salva pela primeira linha do texto. Confirmado em
  `eventos_janela.titulo_janela` no staging. **A decisao e do @Steve**, mas se a saida for
  sanitizar no ingest em vez de no Agent, o trabalho cai no `srv-events-node`.

---

## O que NAO precisa de voce

O A-39, A-40, A-41, A-42, A-46 e A-48 sao todos do Agent Windows e ja estao corrigidos na
1.5.8, com teste. Nao ha mudanca de contrato: o payload que o `srv-events-node` recebe
continua identico.
