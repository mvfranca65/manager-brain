# Review do @Tony — R-03: recall de frota, lado backend (2026-08-24)

> **De:** @Tony | **Autor da entrega:** @Shuri | **Aprovador:** Marcos
> **Entrega:** `manager-srv-admin-node/HANDOFF-R03-RECALL.md`, commit `257096f`
> **Tarefa:** `registro/2026-08-24-tarefa-shuri-R03-recall.md`

## Veredito: **APROVADO**

Nada aqui impede seguir. Os tres pontos duros da tarefa foram cumpridos, e o Marcos ja os conferiu
pessoalmente. Eu cobri o que ele nao tinha como cobrir.

---

## 1. O que eu conferi de fato, e nao apenas li

**Suite verde: 1199 testes, 183 pulados, zero vermelhos.** Rodei tres vezes seguidas — estavel.

**A discrepancia de numeros nao e defeito.** O handoff diz 1196 e eu medi 1199. A causa e o merge
de `origin/staging` que entrou junto (`2910c9f`): ele trouxe trabalho de outra pessoa em
`agent-auth`, com tres testes a mais. O numero da @Shuri estava certo no momento em que ela mediu.

**Um unico `UPDATE`, conferido na linha.** `fleet-health.repository.ts:409-412`:

```ts
.update(versoesAgente)
.set({ revogada: true, pausada: true, revogadaMotivo: motivo })
.where(and(alvo, eq(versoesAgente.revogada, false)))
```

Nao ha estado intermediario observavel. **Concordo com nao envolver em transacao explicita** — um
statement ja e atomico, e a transacao daria a impressao de haver duas escritas a coordenar. O
argumento dela esta certo.

**O guard e `revogada = false`, nao `pausada = false`.** Isto importa e ela viu sozinha: uma versao
ja pausada pelo kill switch precisa continuar podendo ser revogada. Com o guard errado, revogar
depois de pausar viraria no-op silencioso e o recall nunca sairia.

**A conferencia dela e real — refiz uma.** Removi `pausada: true` do `set()` e rodei:

```
× as duas colunas vao no MESMO set() — um unico UPDATE
× o set() nunca marca revogada sem pausada
× nenhum caminho de saida marca so uma das duas
Tests  3 failed | 150 passed
```

Tres falhas, exatamente as que ela relatou no experimento D. Restaurei e a arvore voltou limpa.

**Ordem da checagem confirmada.** `verificarRecall` na linha 162, `buscarCandidatasAtivasPorSO` na
166. Recall antes do laco.

**Nenhum teste existente reescrito.** O commit tem **2279 insercoes e 2 delecoes**, e as duas
delecoes sao linhas de comentario e de `CREATE TABLE` do setup de integracao — nenhuma assercao.

**Nenhum `catch` novo engolindo falha.** Sao tres: dois em `fleet-health.service.ts` (metrica
best-effort, `logger.error` + comentario dizendo que o audit ja saiu) e um em `normalizarVersao`,
sem log **de proposito** — quem chama tem o contexto e loga WARN. Conferi que loga mesmo:
`verificarRecall` linha 284.

**`tsc --noEmit` limpo.**

## 2. O que eu NAO consegui verificar

**O parity runner.** Ela registrou a divergencia no padrao dos vizinhos e diz que o runner nao
ficou vermelho, mas rodar exige o lado Java de pe. **Fica pendente de execucao real** — nao e
motivo para segurar, e alguem precisa rodar antes de ir a producao.

---

## 3. Decisao do Marcos: a coluna extra fica

`revogada_motivo TEXT`. **Aprovada pelo Marcos**, e a razao dele e a certa: sem a coluna, o
`recallMotivo` chegaria vazio na maquina e o post-mortem ficaria cego justamente na hora em que
alguem precisa saber por que a frota foi mandada voltar.

**O furo era meu.** A tarefa mandava devolver o motivo ao Agent e nao dizia de onde tira-lo — pedi
um campo sem especificar o lugar de guarda-lo. Ela viu, resolveu do jeito certo e **perguntou antes
de o @Bucky implementar contra o contrato**, que era exatamente a hora de perguntar.

---

## 4. Achado meu — o recall precisa de guardas que o rollback automatico nao precisa

**Nao e defeito desta entrega.** E do lado Agent, que ainda nao existe. Registro agora porque
precisa entrar no escopo do @Bucky **antes** de ele comecar.

**A diferenca de fundo:** o rollback automatico so dispara em maquina que **ja esta fora do ar**.
O recall dispara em maquina **saudavel**. As mesmas acoes tem consequencias diferentes.

Tres casos que o codigo de hoje trataria mal:

**a) A versao do backup e a mesma da instalada.** Se a anterior tambem for revogada — duas versoes
ruins seguidas —, a maquina volta para uma versao que tambem manda voltar. O
`RollbackOrchestrator` **nao compara versao antes de agir**: a unica guarda e
`Directory.Exists(backupDir)`. Resultado: a cada 6h ela para os dois servicos, copia ~160MB, sobe
de novo e nao muda de versao. **Para sempre**, acumulando um `bin.failed-*` de ~160MB por volta.

**b) Nao ha backup.** No caminho automatico isso vira SOS, e esta certo — a maquina ja estava
morta. **No recall estaria errado:** a maquina esta funcionando, e liga-la em SOS a tira do
auto-update sem necessidade. O certo e nao fazer nada e reportar.

**c) Repeticao.** Enquanto a maquina continuar na versao revogada — porque falhou, ou porque nao
tinha backup — ela vai receber a ordem a cada ciclo. Precisa de freio, como o sticky faz do outro
lado.

**O que o backend ja faz por nos:** o `pausada=true` que vai junto impede a maquina de *reinstalar*
a revogada. Ele nao cobre nenhum dos tres casos acima, que sao todos do lado da maquina.

---

## 5. O que ela encontrou e nao corrigiu — minha posicao

| # | Achado | Posicao |
|---|---|---|
| 6.1 | O teste de swagger **nao falha, esta pulado** — eu escrevi errado na tarefa | **Ela tem razao.** Carreguei isso do `manager-srv-events-node`, onde o teste falha mesmo, e nao conferi neste repo. Terceira vez que descrevo um repo pelo outro |
| 6.4 | Vocabulario de SO divergente: Agent que mandar `DESKTOP_WINDOWS` casa com **zero** linhas — nao acha update **nem** recall, em silencio | **Anterior ao R-03 e real.** Nao morde hoje porque os Agents mandam o vocabulario legado. Vira armadilha no dia em que alguem "modernizar" o Agent: o auto-update para de funcionar sem erro nenhum. **Tarefa propria**, e ela acertou em nao tocar |
| 6.5 | `agent-update.repository.ts` sem teste unitario (19%) | De acordo. Ela cobriu o metodo novo; o resto e lacuna anterior |
| 6.6 | `sistemaOperacional` opcional no endpoint | **Boa decisao.** Sem ele, revogar uma versao mandaria Windows e Android voltarem juntos. Mantem o alcance do `pausar-versao` |
| 6.6 | Metrica reusando `kill_switch_applied_total` com label novo | De acordo — mesma familia, separavel no Grafana |
| 6.6 | Sem `.parity.ts` para o endpoint novo | **Nao precisa.** Nao ha contraparte Java |
| 6.3 | Um erro de lint pre-existente sumiu sozinho | Conferido o raciocinio dela sobre nao haver vazamento: `mapRowToDto` escolhe campo a campo. Correto |

---

## 6. Observacao de processo

**A entrega esta commitada** (`257096f` + merge `2910c9f`), nao empurrada. A tarefa dizia para nao
commitar. Como o commit e do Marcos apos validar — e ele validou os tres pontos duros antes —,
isso esta dentro da regra dele. **Registro so para a linha do tempo ficar honesta**, nao como
reparo.

O merge de `origin/staging` trouxe trabalho de terceiros (`agent-auth`, `release-admin`) para a
mesma arvore. Nao conflita com o R-03, mas quem for revisar aquilo precisa saber que veio junto.

---

## 7. Estado

- **R-03 backend: pronto.** Contrato fechado no handoff, secao 1 — e contra ele que o @Bucky
  implementa.
- **Nao ligar em producao antes do R-01.** Continua valendo.
- **Pendente:** rodar o parity runner de verdade.
- **Entra no escopo do lado Agent:** as tres guardas da secao 4.
- Linha de corte: **inalterada.** O recall e melhoria, nao bloqueador.
