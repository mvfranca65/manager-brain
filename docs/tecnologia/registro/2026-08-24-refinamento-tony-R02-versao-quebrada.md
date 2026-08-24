# Refinamento @Tony — R-02: gravar qual versao quebrou, antes de trocar os arquivos (2026-08-24)

> **De:** @Tony | **Aprovador:** Marcos
> **Para:** @Bucky | **Para ser executado numa sessao nova.** Este documento e autossuficiente: leia so ele.
> **Nao commite. Nao faca push. Ao terminar, @Tony revisa.**
>
> **Continuacao direta do R-01.** O R-01 esta certo e fica como esta. Esta tarefa conserta o
> lado que **produz** o dado que a guarda do R-01 le.

---

# 1. Contexto

## 1.1 Onde as coisas estao

Diretorio de trabalho: `C:\Users\NoisyTech\Documents\Manager\`

| Repo | O que e |
|---|---|
| `manager-srv-agent` | **o seu.** Agent Desktop Windows, C#/.NET 8 |
| `manager-brain` | documentacao do time |

**Atencao:** o prompt de sessao manda ler arquivos em `.brain/`. **Essa pasta nao existe.** O
cerebro esta em `manager-brain/docs/`. Varios arquivos que o prompt lista como obrigatorios
(`banco-dados.md`, `seguranca.md`, `lgpd-operacional.md`, `tecnologia/services/`) nao existem em
repo nenhum. Nao perca tempo procurando.

Leitura util de verdade, se precisar: `manager-brain/docs/tecnologia/agent-desktop/REGRAS-RELEASE.md`.

## 1.2 Estado do repo hoje

| | |
|---|---|
| Ultimo commit | `b1d227a` |
| Nao commitado | 86 arquivos de 21/08 e 24/08 **+ o R-01 que voce acabou de entregar** |
| Versao nos 7 `.csproj` | **1.5.12** — nao bumpar |
| Testes | **1085 verdes** |

Rode a suite antes de comecar e confirme os 1085. Se nao bater, **pare e avise** — alguem mexeu.

```powershell
cd C:\Users\NoisyTech\Documents\Manager\manager-srv-agent
dotnet test ManagerAgent.sln --nologo -v q
```

## 1.3 O que voce entregou no R-01, e que continua valendo

A guarda no `UpdateCheckerWorker` que impede a maquina de reinstalar a versao que acabou de
quebra-la, durante as 24h de sticky. Ela esta **correta**. O `VersaoSemVer` esta correto, e a
prova por remocao (6 testes vermelhos sem a normalizacao) e a prova certa. **Nao mexa em nada
disso.**

---

# 2. O defeito — R-02

## 2.1 O que voce achou, e esta certo

`WatchdogState.VersaoAtual` e escrito em **um unico lugar em todo o produto**, e esse lugar e
dentro do proprio rollback. Conferi por fora, com grep em todo o `src/`:

```
src/ManagerAgent.Watchdog/Services/WatchdogService.cs:303   state.VersaoAtual = versaoRestaurada;
```

`WatchdogState.Default()` deixa o campo `null`. Nem o instalador, nem o `UpdateApplier`, nem o
Service carimbam ele. **Em toda maquina da frota hoje esse campo esta vazio.**

## 2.2 O estrago, que e maior do que so a sua guarda

O campo vazio nao quebra so o R-01. Ele tem **quatro** consumidores, e os quatro estao cegos hoje:

| Onde | Linha | O que acontece hoje |
|---|---|---|
| Evento de auditoria `UPDATE_ROLLBACK_TRIGGERED` | `RollbackOrchestrator.cs:122` | reporta `versaoQuebrada: null` — a telemetria nao sabe qual versao derrubou a maquina |
| Nome da pasta da versao quebrada | `RollbackOrchestrator.cs:149` | vira `bin.failed-unknown-<timestamp>` |
| Script de restauracao | `RollbackOrchestrator.cs:151` | recebe `null` |
| `UltimaVersaoQuebrada`, que alimenta a **sua guarda** | `WatchdogService.cs:302` | vira `null` e a guarda nao dispara |

Uma unica captura conserta os quatro. E por isso que a correcao vai onde vai.

## 2.3 Por que as duas saidas que voce propos nao funcionam

Voce fez certo em nao executar e trazer para mim. As duas gravariam a versao **boa**:

**(a) `state.VersaoAtual ?? LerVersaoInstalada()` dentro do `ConcluirRollbackAsync`.** Nao. O
rollback virou assincrono: quando esse metodo roda, o script **ja trocou os arquivos** e ja subiu
o servico. `LerVersaoInstalada()` ali devolve a versao **restaurada**. Voce gravaria "a versao boa
me quebrou" e barraria justamente a versao para a qual acabou de voltar — enquanto a quebrada
continuaria liberada. Ficaria com cara de resolvido e sem proteger nada.

**(b) o Service carimbar `VersaoAtual` no boot saudavel.** Tambem nao, para este fim. Se a versao
nova nunca sobe, ela nunca tem boot saudavel — o campo continuaria com a versao **anterior**, que
e a boa. Mesmo erro por outro caminho. (A ideia tem valor para o R-03 e para o campo passar a
significar o que o nome diz. Nao e desta rodada.)

---

# 3. O que fazer

## 3.1 A captura

Em `RollbackOrchestrator.TryRollbackAsync`, **no topo, logo depois do `_stateStore.Load()` e antes
do passo 1 (audit)**: ler a versao instalada e grava-la no estado. Nesse instante os binarios
quebrados ainda estao no disco — e o unico momento do fluxo em que a leitura devolve a versao
certa.

```csharp
var state = _stateStore.Load();

// A versao quebrada so pode ser lida AQUI: o script do passo 5 troca os binarios, e a partir
// dai o disco tem a versao restaurada. Os quatro consumidores dependem desta linha.
var versaoQuebrada = LerVersaoInstalada();
if (versaoQuebrada is not null)
{
    state.VersaoAtual = versaoQuebrada;
}

var backupBinDir = BackupDir;
```

**A guarda do `is not null` nao e enfeite.** Se o executavel nao puder ser lido,
`LerVersaoInstalada()` devolve `null`; sem a guarda voce apagaria um valor bom que ja estivesse la
(o caso do segundo rollback em diante). Perder informacao e pior do que nao ganhar.

**Persistir.** O passo 4 ja faz `_stateStore.Save(state)` antes de disparar o script, e a captura
fica antes dele — entao ja e gravada. **Confirme isso lendo o codigo, nao presuma.** O script mata
este processo; o que nao estiver no arquivo se perde.

**Grave o valor cru, com hash.** Nao normalize na escrita. Quem compara e o seu `VersaoSemVer`, na
leitura. Normalizar aqui esconderia o formato real no log e na auditoria.

## 3.2 O que NAO mexer

- `UpdateCheckerWorker` e `VersaoSemVer` — sao o R-01, estao certos.
- `ConcluirRollbackAsync` (`WatchdogService.cs:295-315`) — a ordem dele fica como esta. Com a
  captura no lugar certo, `state.UltimaVersaoQuebrada = state.VersaoAtual` passa a valer sozinho.
- `UpdateApplier` e o script de update. Estao funcionando.
- `SosMode`, `StickyVersion`, o texto dos logs do bloco SOS.
- Nao bumpe a versao dos `.csproj`.

---

# 4. Testes

## 4.1 O que escrever

1. **O caso da frota inteira hoje:** estado com `VersaoAtual = null` e executavel instalado
   reportando a versao quebrada. Depois do disparo, `VersaoAtual` no arquivo salvo e a versao
   quebrada, e nao `null`.
2. **Ponta a ponta com a sua guarda:** partindo de `VersaoAtual = null`, dispara rollback, conclui
   com sucesso, e entao o `UpdateCheckerWorker` **recusa** a versao quebrada e **aceita** uma
   diferente. E o teste que prova que R-01 e R-02 se encaixam. Este e o mais importante dos cinco.
3. **Leitura falha nao apaga:** `LerVersaoInstalada()` devolvendo `null` com `VersaoAtual` ja
   preenchido, o valor antigo sobrevive.
4. **Com hash colado:** a versao capturada mantem o formato cru, e a comparacao pelo `VersaoSemVer`
   ainda casa.
5. **Auditoria e nome de pasta:** o evento `UPDATE_ROLLBACK_TRIGGERED` leva a versao, e a pasta sai
   `bin.failed-<versao>-<timestamp>`, sem `unknown`.

## 4.2 A armadilha que ja nos mordeu tres vezes — leia

Os cenarios em `tests/e2e/` **montam o `watchdog-state.json` eles mesmos**. Se algum deles semear
`VersaoAtual` com valor, o teste passa e a maquina real continua cega — que e exatamente como este
defeito, e o do rollback de manha, chegaram ate aqui: *o teste e o codigo concordam entre si, e os
dois discordam do instalador de verdade.*

**Confira o `e1-alt-rollback-crash.ps1` e os demais cenarios que escrevem esse arquivo. Se algum
semeia `VersaoAtual`, tire — o estado inicial tem que ser o que o instalador de verdade produz.**
Se tirar quebrar algum cenario, **nao conserte semeando de volta**: e sinal de defeito real.
Escreva o que achou e me chame.

---

# 5. Aceite

- [ ] Suite verde, partindo dos 1085, sem nenhum teste antigo alterado.
- [ ] Os 5 testes da secao 4.1 existem e passam.
- [ ] Prova por remocao: desfaca a captura da secao 3.1 e mostre quais testes ficam vermelhos. Se
      nenhum ficar, o teste nao esta provando nada.
- [ ] Cenarios e2e conferidos quanto a semeadura de `VersaoAtual`, com o resultado escrito.
- [ ] Nada commitado, nada empurrado, versao nao bumpada.

---

# 6. Ao terminar

Escreva `HANDOFF-R02-VERSAO-QUEBRADA.md` na raiz do `manager-srv-agent`, no mesmo formato do
`HANDOFF-R01-STICKY.md`: o que mudou arquivo por arquivo, os numeros de teste antes e depois, a
prova por remocao, o resultado da varredura dos cenarios e2e, e onde voce discordou deste
documento.

E devolva um resumo de no maximo 12 linhas, em portugues claro e sem jargao, que sera mostrado ao
Marcos.

---

# 7. O que continua fora desta rodada

- Carimbar `VersaoAtual` no boot saudavel, para o campo significar o que o nome diz (o R-03 vai
  querer isso).
- `StickyVersion`, que e escrita e nunca lida.
- Mover `bin.previous` para dentro da instalacao.
- O teste de maquina do rollback (item 3b) — depende de decisao de infra com o @Vision.
