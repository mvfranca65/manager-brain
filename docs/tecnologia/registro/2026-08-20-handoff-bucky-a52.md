# Handoff para @Bucky — A-52 antes de publicar a 1.5.10 (2026-08-20)

> **De:** @Tony | **Para:** @Bucky | **Aprovador:** Marcos
> **Bloqueia:** publicação da 1.5.10
> **Escopo:** uma correção. Nada além disto nesta rodada.

---

## O que é

A correção do A-50 ligou a vinculação da seção `Capture`, e o heartbeat passou a respeitar o
`appsettings` — comprovado: 120s configurados, 120s medidos, mediana de 396 intervalos.

Mas o A-50 tinha **duas** metades. O briefing dizia:

> "A frota mandava o dobro de batimentos e o triplo de resumos de input do que a config pedia."

O dobro de batimentos foi corrigido. **O triplo de resumos continua na 1.5.10**, porque o
consumidor nunca leu a opção:

```csharp
// src/ManagerAgent.SessionWorker/Worker.cs:55
private static readonly TimeSpan InputSummaryInterval = TimeSpan.FromSeconds(60);

// src/ManagerAgent.SessionWorker/Worker.cs:246
if (now - _lastInputSummaryAt < InputSummaryInterval) return;
```

O `appsettings` pede `IntervaloResumoEntradaSegundos: 180`. A propriedade existe, a
vinculação funciona, e ninguém a lê. O agent emite **3x mais** `InputSummaryEvent` do que a
configuração manda — 3x o volume em `resumos_atividade_input`, na frota inteira.

É a mesma família do A-50, causa diferente: lá era a seção que não chegava; aqui é o
consumidor que ignora o que chegou.

---

## Os outros dois da mesma família

Achados na mesma varredura. Decida com calma — **só o `IntervaloResumoEntradaSegundos`
bloqueia a publicação**, porque só ele tem divergência de valor observável.

| Campo | Constante que manda | JSON pede | Divergência |
|---|---|---|---|
| `IntervaloResumoEntradaSegundos` | `InputSummaryInterval = 60s` | 180s | **3x** — bloqueia |
| `IntervaloLoopWorkerSegundos` | `LoopDelay = 1s` | 1s | nenhuma hoje; muda o JSON e nada acontece |
| `IntervaloCheckHooksSegundos` | `HookCheckInterval = 30s` | 60s | 2x, mas o campo nem existe na classe de opções |

---

## O que precisa mudar

O `Worker` do SessionWorker **não recebe** `IOptions<ManagerAgentUploadOptions>` — não é só
trocar a constante por uma propriedade. A mudança é:

1. Injetar `IOptions<ManagerAgentUploadOptions>` no construtor do `Worker`
   (`src/ManagerAgent.SessionWorker/Worker.cs:61`).
2. Trocar o `static readonly` por campo de instância, com o mesmo padrão defensivo já usado
   em `HeartbeatService` e `IdleMonitorService`:
   ```csharp
   _inputSummaryInterval = TimeSpan.FromSeconds(
       opts.IntervaloResumoEntradaSegundos > 0 ? opts.IntervaloResumoEntradaSegundos : 60);
   ```
   O `> 0 ? : 60` importa: config zerada ou ausente não pode virar intervalo zero, que
   emitiria resumo a cada ciclo de 1s.
3. `services.AddHostedService<Worker>()` (Program.cs:175) resolve sozinho — não precisa mexer.
4. Ajustar `tests/ManagerAgent.SessionWorker.Tests/WorkerShutdownTests.cs`, o único ponto
   que faz `new Worker(...)` na mão.

Se decidir tratar os outros dois na mesma leva, `IntervaloCheckHooksSegundos` precisa
**primeiro** existir como propriedade em `ManagerAgentUploadOptions` — hoje é chave morta no
JSON.

---

## Aceite

**1. Teste unitário novo, no padrão do `CaptureConfigBindingTests`:** com
`IntervaloResumoEntradaSegundos = 180`, o worker não pode emitir dois `InputSummaryEvent` em
menos de 180s; com o campo ausente, mantém 60s.

**2. Remover a entrada da lista de dívida.** Em
`tests/ManagerAgent.Service.Tests/CaptureOptionsHaveConsumersTests.cs`, apague
`IntervaloResumoEntradaSegundos` de `SemConsumidorConhecido`. O teste
`Lista_de_chaves_sem_consumidor_nao_tem_entrada_obsoleta` **vai falhar** enquanto a entrada
estiver lá depois da correção — é ele que cobra a limpeza. Mesma coisa em
`AppSettingsKeysAreReadTests.MortasConhecidas` se você ligar o `IntervaloCheckHooksSegundos`.

**3. Verificação na base, depois de instalar:**
```powershell
node tests\e2e\regressivo\verificar-eventos-base.js
```
O item **B8** mede a cadência real de `resumos_atividade_input` contra o `appsettings`
instalado. Hoje ele acusa "medido ~60s, configurado 180s". Depois da correção tem que
fechar em ~180s. Precisa de ~3h de agent rodando para ter amostra.

**4. Suíte verde.** Hoje: 328 testes no `Service.Tests`, 0 falhas.

---

## Restrições

- Toque em `ManagerAgent.Service` **e** `ManagerAgent.SessionWorker` se o código estiver
  duplicado entre os dois — regra do seu papel, vale aqui.
- **Não commite nem faça push.** O Marcos valida e commita.
- Review comigo (@Tony) antes de fechar.

---

## O que NÃO entra nesta rodada

O **A-53** (agent não trata revogação: 403 + `X-Agent-Revoked` é ignorado, `UploadWorker`
retenta para sempre) foi levantado e **adiado pelo Marcos em 2026-08-20**. A análise está em
`agent-desktop/adr/ADR-001-captura-exige-vinculo-ativo.md`, com status Proposta/adiada, e o
comportamento atual está travado pelo cenário N8. Não implemente.
