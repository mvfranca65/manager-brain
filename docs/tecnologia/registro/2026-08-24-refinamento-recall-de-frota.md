# Refinamento — recall de frota, e duas coisas que faltam antes dele (2026-08-24)

> **De:** @Tony | **Aprovador:** Marcos
> **Origem:** pergunta do Marcos — *"se eu quiser fazer rollback de uma implantacao, isso e
> possivel? Eu disparar algo que vai voltar a versao em todas as maquinas"*
>
> **Resposta curta: hoje nao.** E investigando o porque, achei dois defeitos que tornam o
> rollback **automatico** incompleto. Os dois vem antes do recall.

---

## O que existe hoje

| Mecanismo | O que faz | O que NAO faz |
|---|---|---|
| `versoes_agente.pausada = true` (kill switch, manual via `fleet-health`) | para de entregar a versao a quem ainda nao pegou | nao mexe em quem ja atualizou |
| `rollout_percent` | limita exposicao daqui para frente | idem |
| Rollback automatico (3a, feito hoje) | volta a versao anterior **quando o Service nao sobe 3x** | nao age se a versao sobe e esta errada de outro jeito |

**Republicar a versao antiga nao funciona.** `agent-update.service.ts` so oferece candidata que
passe em `isVersaoMaior(candidata, versaoAtual)`; maquina na 1.5.12 recebe "voce esta atualizado".
O Agent tem uma segunda trava pelo mesmo motivo (A-46). As duas estao certas — evitam laco de
reinstalacao —, e sao exatamente o que impede um downgrade.

---

# R-01 — o sticky nao e lido por ninguem

**Severidade: ALTA.** **Nao e item novo: e o 3a incompleto.** Sem isto, o rollback que acabamos de
consertar se desfaz sozinho.

## Evidencia

`StickyVersion` e `StickyUntil` sao **gravados** apos um rollback e lidos em **um lugar so**:

```
WatchdogState.cs:31,34            <- definicao
WatchdogService.cs:142            <- `stickyAtiva`, para NAO repetir rollback
WatchdogService.cs:304,305        <- escrita apos rollback bem-sucedido
```

O `UpdateCheckerWorker` **nao consulta sticky**. Ele so consulta `SosMode`
(`UpdateCheckerWorker.cs:213-220`). E o rollback bem-sucedido deixa `SosMode = false`.

## O que acontece hoje, em ordem

1. Maquina na 1.5.13 (quebrada). Falha 3x. Rollback devolve a 1.5.12. Sticky ate +24h.
2. Ate 6h depois, o `UpdateCheckerWorker` roda. `SosMode` e false, entao ele segue. Pergunta ao
   backend estando na 1.5.12; o backend oferece a 1.5.13 — que continua publicada, porque
   **a pausa e manual** (`fleet-health.service.ts`, `pausarVersao`) e ninguem apertou o botao.
3. Baixa, instala, quebra de novo.
4. Falha 3x outra vez. Agora o Watchdog ve `stickyAtiva` e **nao reverte**.

**Fim: a maquina fica quebrada ate a sticky vencer.** O sticky, que existe para proteger, e o que
prende. E o rollback automatico existe justamente para o caso em que ninguem esta olhando —
madrugada, fim de semana.

## O que fazer

O `UpdateCheckerWorker` passa a recusar instalar a versao registrada em `UltimaVersaoQuebrada`
enquanto `StickyUntil` estiver no futuro. Um `if`, no mesmo ponto em que ele ja consulta o estado
para `SosMode`.

**So isso.** Nao mexer no criterio de 3 falhas, na duracao da sticky, nem no `SosMode`.

## Aceite

- Estado com `UltimaVersaoQuebrada = X` e `StickyUntil` no futuro: o ciclo de update **nao**
  instala X, e registra por que pulou.
- Versao **diferente** de X continua sendo instalada normalmente — sticky nao congela a maquina,
  so barra a versao que quebrou.
- `StickyUntil` no passado: volta a instalar X (uma versao republicada corrigida com o mesmo
  numero e caso legitimo; e o operador tem o kill switch se nao for).
- Sem `WatchdogState` no disco (cliente novo): segue como hoje, sem bloquear.

**Dono: @Bucky.** So Agent, sem contrato novo, sem deploy ordenado.

---

# R-02 — o modo SOS e porta de mao unica

**Severidade: ALTA.** Item novo — proponho para a linha de corte, decisao do Marcos.

## Evidencia

`UpdateCheckerWorker.cs:213` diz que a maquina sai do SOS *"via header
`X-Manager-Sos-Recovery-Available` 3x consecutivos OU intervencao manual"*, e
`WatchdogState.SosRecoveryHeaderCount` existe para contar isso.

**Esse header nao existe em lugar nenhum.** Varri os tres repositorios:

```
manager-srv-agent        -> so os dois comentarios acima
manager-srv-admin-node   -> nenhuma ocorrencia
manager-srv-events-node  -> nenhuma ocorrencia
```

Ninguem envia, ninguem le, o contador nunca incrementa.

## Consequencia

Maquina que entra em SOS **para de se atualizar para sempre**. A unica saida documentada e
"intervencao manual", e manual aqui significa alguem editando `watchdog-state.json` **naquela
maquina** — nao ha comando remoto para limpar SOS.

Combinado com o R-01: a maquina que passa pelo laco acima acaba em SOS e nao volta mais sozinha.
O `fleet-health` ja lista quem esta em SOS — da para **ver**, nao da para **agir**.

## O que fazer

Fechar o que ja foi desenhado: o backend passa a responder o header quando a versao que quebrou
foi pausada ou substituida, e o Agent conta 3 consecutivos para sair do SOS. Ou, se o desenho de
header nao se sustentar, um campo na resposta do `verificar` com o mesmo efeito.

**Traga o desenho antes de escrever** — decidir entre header e campo de resposta e contrato, e o
`verificar` tem paridade byte-a-byte com o Java.

**Dono: @Bucky (Agent) + @Shuri (srv-admin-node).**

---

# R-03 — recall de frota (o que o Marcos pediu)

**Severidade: MEDIA.** Nao segura producao — **desde que o rollout gradual seja respeitado**. Com
canario de 48h e 10%, uma versao ruim alcanca poucas maquinas antes de ser pausada. O recall e o
que protege quando ela passa das etapas e chega a 100%.

**Depende do R-01.** Sem ele, a maquina reverte e reinstala a versao revogada no ciclo seguinte.

## Mecanismo: reverter local, nao baixar versao antiga

**Recomendo reusar o `bin.previous` e o `run-rollback.ps1`** que ja existem em cada maquina.

| | Reverter local | Baixar versao antiga |
|---|---|---|
| Custo por maquina | zero — o backup ja esta no disco | ~310MB |
| Alcance | **uma versao para tras** | qualquer versao publicada |
| Muda o `verificar`? | nao mexe na comparacao de versao | exige relaxar `isVersaoMaior` — paridade com o Java |
| Reusa o que foi feito hoje | sim, inteiro | nao |

O caso real e "a versao que acabamos de subir esta ruim, volta uma". Reverter local resolve isso e
nao paga nada. Baixar versao antiga fica para uma fase 2, **se** aparecer necessidade medida.

**Limite, e tem de estar visivel na tela de quem dispara:** o `bin.previous` guarda **uma** versao.
Se a frota ja andou duas vezes, o recall leva para a penultima, nao para a que se quer.

## Como a ordem chega ate a maquina

Pela resposta do `verificar`, que o Service ja consulta — **sem canal novo**.

**Latencia: ate 6 horas.** E o intervalo do `UpdateCheckerWorker`. Para um recall, e muito.
Duas saidas, e **quero a sua decisao, Marcos**:

- **aceitar as 6h** — mais simples, nada novo;
- **encurtar o intervalo** quando houver recall pendente. Ja existe
  `updateCheckIntervalSecondsOverride` no config (usado em teste), entao o mecanismo existe.

Nao vou pendurar isso no heartbeat de 60s do `srv-events`: aquele e o caminho quente da ingestao,
e misturar controle de frota com ingestao de evento acopla duas coisas que precisam falhar
separado.

## Lado do backend

Coluna nova em `versoes_agente`, **separada da `pausada`** — as duas dizem coisas diferentes:

| Campo | Significa |
|---|---|
| `pausada` | nao entregue a quem ainda nao tem |
| `revogada` *(nova)* | **quem ja tem, devolva** |

No `verificar`: se a versao **atual do agente** esta revogada, a resposta manda reverter em vez de
oferecer update. Campo novo, aditivo — Agent antigo ignora (`@JsonIgnoreProperties`).

**Registrar a divergencia no `manager-parity-runner`**, nao esconder: o Java nao tem este campo.

## Ordem de deploy — regra dura

`REGRAS-RELEASE` secao 3: backend primeiro, Agent depois.

**E a consequencia que precisa ser dita com todas as letras:** o recall so funciona em maquina que
entende o comando. A frota de hoje (1.5.12) vai ignorar. **O recall protege as versoes lancadas
DEPOIS de ele existir, nunca a que estiver rodando quando ele for publicado.**

## Seguranca e observabilidade

- Disparar recall e destrutivo em toda a frota. Endpoint autenticado, com motivo obrigatorio e
  audit — nao um `UPDATE` no banco a mao. O `fleet-health` ja tem o molde do `pausarVersao`.
- **Revogar deve pausar junto**, sempre. Revogar sem pausar manda a maquina voltar e o ciclo
  seguinte reinstala.
- Idempotente: maquina que ja esta na versao alvo nao faz nada.
- Sem `bin.previous` (primeiro update do cliente): nao ha para onde voltar. **Reporta e nao
  quebra** — e essa maquina precisa aparecer numa lista, senao o recall parece completo e nao esta.
- Acompanhar pelo que ja existe: `agentes.versao_agente` (a query da secao 5 do `REGRAS-RELEASE`)
  mostra a frota migrando de volta.

## Aceite

- Versao marcada como revogada + pausada: maquina naquela versao volta para a anterior sozinha,
  dentro do intervalo acordado.
- Maquina em outra versao ignora.
- Maquina sem backup reporta e continua funcionando na versao atual.
- Depois de voltar, **nao reinstala a versao revogada** (e o R-01 agindo).
- Agent anterior ao recall ignora o campo e nao quebra.
- Recall registrado em audit, com autor e motivo.

**Donos: @Shuri** (coluna, `verificar`, endpoint, paridade) **+ @Bucky** (obedecer o comando,
reusando `run-rollback.ps1`).

---

# Ordem

1. **R-01** — @Bucky. Pequeno, so Agent, e sem ele o 3a se desfaz sozinho.
2. **R-02** — desenho antes de codigo.
3. **R-03** — depois do R-01.

## O que peco ao Marcos

1. **R-01 entra como parte do 3a?** Minha leitura: sim — "o Service volta sozinho" inclui
   *continuar* na versao boa. Nao e item novo na lista.
2. **R-02 entra na linha de corte?** Minha leitura: sim, criterio B — maquina em SOS nao volta
   sem visita.
3. **R-03: 6 horas de latencia serve, ou encurta o intervalo?**
