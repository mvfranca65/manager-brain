# Entrega — item 1 (@Shuri) e rollback 3a (@Bucky) (2026-08-24)

> **Para:** @Tony (review) | **Aprovador:** Marcos
> **Refinamentos:** `2026-08-24-refinamento-shuri-lote-e-sessao.md`,
> `2026-08-24-refinamento-bucky-rollback.md`, `2026-08-24-review-tony-desenho-rollback.md`
>
> **Estado: nada commitado, nada empurrado, nada testado em maquina.**

---

## Resumo

| Item | Dono | Estado |
|---|---|---|
| 1 — lote recusado por conteudo | @Shuri | **feito** |
| 3a — ligar o rollback ao backup | @Bucky | **feito** |
| 3b — teste E2E no CI | @Bucky + @Vision | **parcial** — ver secao propria |
| 4 — senha do staging | @Vision | nao e codigo |

**Testes.** `manager-srv-events-node`: 678 -> **695**. `manager-srv-agent`: 1010 -> **1041**.
Nenhum teste existente reescrito. Typecheck e lint limpos no Node; nenhum aviso novo de
compilacao no Agent (seguem os mesmos tres `CS1998` de antes).

---

## Item 1 — lote recusado por conteudo

### Causa 1: `.optional()` recusa `null` explicito

`windowsUsername`, `identificadorColaborador` e `instalacaoId` passaram de `.optional()` para
`.nullish()`.

**Medicao pedida no refinamento, feita:** os tres sao `string?` no `AgentEventBatchDto.cs` e o
`JsonOptions` do `HttpEventUploader` **nao tem** `DefaultIgnoreCondition` — os tres saem no corpo
como `"campo": null` quando vazios. Os tres tinham o mesmo defeito; os tres foram corrigidos.

### Causa 2: erro de item derrubava o lote

Os campos de lote foram extraidos para um objeto unico (`camposDeLote`), consumido por dois
schemas que **nao podem divergir**:

- `AgentEventBatchSchema` — contrato completo, itens inclusive. Inalterado no comportamento, e por
  isso os testes dele continuam passando sem uma linha mudada.
- `AgentEventBatchEnvelopeSchema` — o que o servico valida: campos de lote estritos, itens crus.

`validarEventos()` roda item a item e devolve `{ validos, ignorados }`, com o indice do lote
**original** preservado. O `dispatch` passou a usar esse indice em vez da posicao no array — sem
isso, os `motivosIgnorados` dos handlers apontariam para o evento errado sempre que houvesse um
buraco antes deles. Ha teste so para isso.

**Decisao do @Tony implementada:** lote com todos os itens invalidos responde **202**, nao 400.

### O que os testes provaram, e o que corrigiram em mim

17 testes novos. Desfiz as duas correcoes e **7 deles falharam** — a conferencia que o @Tony
cobra.

**Um teste meu estava errado, nao o codigo.** Escrevi um caso com
`{ tipoEvento: 'WindowActivityEvent', dados: { faltando: 'tudo' } }` esperando recusa. Passou:
`windowActivityPayload` tem **todos** os campos opcionais e o Zod descarta chave desconhecida.
O que de fato falha e valor com tipo errado. Corrigi o teste e deixei o motivo escrito nele —
a permissividade do payload por item e maior do que eu supunha, e isso reduz o tamanho real do
risco da causa 2.

**Paridade:** a divergencia com o Java (la o lote inteiro cai) **ainda nao foi registrada no
`manager-parity-runner`** — nao mexi no runner. Fica para a @Shuri fechar.

---

## 3a — rollback

### Os tres defeitos, corrigidos

**1. Endereco.** Os caminhos do rollback saem agora de `ConfigPaths`
(`PastaBackupAnterior`, `PastaBackupAnteriorEmConstrucao`, `ArquivoResultadoRollback`,
`ArquivoRunRollback`), num lugar so. O gerador do PS1 de update passou a consumi-los em vez de
recalcular com `Split-Path`. **Era a duplicacao que causava tudo.**

**2. Destino da restauracao.** O construtor de producao passou a receber
`ConfigPaths.PastaInstalacao` — a raiz, de onde o SCM roda — e nao `...\ManagerAgent\bin`.

**3. Nao mover a instalacao.** A troca de arquivos foi delegada a `run-rollback.ps1`, disparado
por WMI, mesmo caminho do Plan A do `UpdateApplier`. O orquestrador agora verifica, escreve o
script, marca a intencao e sai.

### O desenho aprovado, implementado

- Script escreve `rollback-result.json` com **cinco fatos** e nada mais. Ha teste checando que
  nomes de campo de estado (`stickyUntil`, `sosMode`, `watchdog-state`) **nao** aparecem no
  PowerShell — a decisao nao pode vazar para la.
- `WatchdogService.ReconciliarRollbackPendenteAsync` roda **antes** do primeiro ciclo e converte o
  resultado nos quatro `RollbackResult` de sempre. Nenhum saiu do contrato; `Dispatched` foi
  acrescentado e significa "ainda nao se sabe".
- `RollbackDispatchedAtUtc` gravado antes do disparo. Sem resultado apos **15 minutos**
  (a folga que o @Tony ajustou), vira `Error` e SOS — o script que nao roda deixou de ser silencio.
- Recovery do SCM desligado nos **dois** servicos antes da troca e restaurado no `finally`, com os
  valores medidos na maquina. Ha teste exigindo que a restauracao esteja **dentro** do `finally`.
- `WatchdogStateStore.Save` virou temp + move. Cinco testes, incluindo o que prova que um `.tmp`
  orfao de uma queda nao contamina o estado bom.

### O que os testes provaram

31 testes novos. Reintroduzi o defeito de endereco (constante voltando para `...\bin`) e **tres
falharam** na hora, apontando exatamente a divergencia de nivel de diretorio.

**Dois erros meus foram pegos pelos proprios testes**, e vale registrar: usei travessao em
comentarios dentro dos scripts PowerShell gerados, e existe guarda de ASCII para isso (achado
A-05: PowerShell 5.1 le `.ps1` sem BOM como ANSI e o parser quebra). A guarda funcionou nas tres
vezes em que eu errei.

---

## 3b — o que fiz e o que falta

**Feito, e e a parte que era codigo:** o cenario `e1-alt-rollback-crash.ps1` procurava
`bin.failed-*` **dentro** da instalacao. Com a correcao, a versao quebrada e preservada **ao lado**
dela — dentro seria pior, porque a restauracao copia por cima e a copia quebrada sobreviveria, e
porque o backup do proximo update levaria junto ~160MB de binario morto a cada rodada. Assercao
corrigida, e o cabecalho do cenario agora diz como o rollback assincrono funciona e avisa para
rodar contra a instalacao do instalador de verdade.

**NAO feito:** por em CI. Exige servico Windows, elevacao e reboot de servico — e decisao com o
@Vision sobre runner hospedado ou VM propria. **E o cenario continua sem nunca ter rodado.**

> **Isto e o que separa "funciona" de "esta provado".** A correcao do 3a esta coberta por teste
> unitario e pela conferencia de que o teste pega o defeito. Mas o rollback de verdade — servico
> parando, arquivos trocando, servico voltando — **nao foi exercitado nenhuma vez**. Foi
> exatamente essa distancia que escondeu o defeito original.

---

## Divida e pendencias desta rodada

| # | O que | Dono |
|---|---|---|
| 1 | Divergencia de paridade do item 1 nao registrada no `manager-parity-runner` | @Shuri |
| 2 | 3b em CI | @Vision + @Bucky |
| 3 | `bin.previous` e `bin.failed-*` moram em `C:\Program Files\` (fora da pasta do produto). Decisao adiada de proposito — mover e mexer no caminho de update, que funciona | @Tony |
| 4 | `bin.previous.building` orfa nunca e limpa. O comentario do PS1 que dizia o contrario **foi corrigido** nesta rodada; a limpeza em si, nao | @Bucky |
| 5 | Nono item da varredura de textos defasados continua aberto | @Tony |

---

## Arquivos

**`manager-srv-events-node`** — `dto/agent-event-batch.dto.ts`, `ingestion.service.ts`,
`test/unit/ingestion/ingestion.service.test.ts`.

**`manager-srv-agent`** — `Shared/Config/ConfigPaths.cs`, `Shared/State/WatchdogState.cs`,
`Shared/State/WatchdogStateStore.cs`, `Service/Update/UpdateApplier.cs`,
`Watchdog/Services/RollbackOrchestrator.cs`, `Watchdog/Services/WatchdogService.cs`,
`Watchdog/ManagerAgent.Watchdog.csproj`, mais os testes
`RollbackCaminhoRealTests.cs` e `ReconciliacaoRollbackTests.cs` (novos),
`WatchdogStateStoreTests.cs` e `e1-alt-rollback-crash.ps1` (acrescimos).
