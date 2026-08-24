# Tarefa para @Bucky — R-01: fazer o sticky ser respeitado (2026-08-24)

> **De:** @Tony | **Aprovador:** Marcos
> **Para ser executado numa sessao nova.** Este documento e autossuficiente: leia so ele.
> **Nao commite. Nao faca push. Ao terminar, @Tony revisa.**

---

# 1. Contexto — leia antes de abrir qualquer arquivo

## 1.1 Onde as coisas estao

Diretorio de trabalho: `C:\Users\NoisyTech\Documents\Manager\`

| Repo | O que e |
|---|---|
| `manager-srv-agent` | **o seu.** Agent Desktop Windows, C#/.NET 8 |
| `manager-brain` | documentacao do time |
| `manager-srv-events-node` | backend de ingestao (NestJS) — @Shuri |
| `manager-srv-admin-node` | backend de vinculo e updates (NestJS) — @Shuri |

**Atencao:** o prompt de sessao manda ler arquivos em `.brain/`. **Essa pasta nao existe.** O
cerebro esta em `manager-brain/docs/`. E varios arquivos que o prompt lista como obrigatorios
(`banco-dados.md`, `seguranca.md`, `lgpd-operacional.md`, `tecnologia/services/`) nao existem em
repo nenhum. Nao perca tempo procurando.

Leitura util de verdade: `manager-brain/docs/tecnologia/agent-desktop/REGRAS-RELEASE.md`.

## 1.2 Estado do repo `manager-srv-agent` hoje

| | |
|---|---|
| Ultimo commit | `b1d227a` |
| Arquivos nao commitados | **86** — trabalho de 21/08 e 24/08, tudo aprovado, nada empurrado |
| Versao nos 7 `.csproj` | **1.5.12** |
| Testes | **1041 verdes** (14 Tray + 62 Watchdog + 370 SessionWorker + 595 Service) |
| Maquina do Marcos | rodando 1.5.12 |

Rode a suite antes de comecar e confirme os 1041. Se nao bater, **pare e avise** — alguem mexeu.

```powershell
cd C:\Users\NoisyTech\Documents\Manager\manager-srv-agent
dotnet test ManagerAgent.sln --nologo -v q
```

## 1.3 O que aconteceu antes desta tarefa

Em 24/08 foi corrigido o **rollback automatico do auto-update** (item 3a da linha de corte de
producao). Ele nunca tinha funcionado: procurava o backup um nivel de diretorio acima do lugar em
que o script de update o grava.

O fluxo hoje, ja corrigido e testado:

1. Versao nova nao sobe. O Watchdog conta 3 falhas de startup.
2. Dispara `run-rollback.ps1`, que para os servicos, restaura `C:\Program Files\bin.previous` por
   cima da instalacao e sobe o Service de volta.
3. No boot seguinte, `WatchdogService.ReconciliarRollbackPendenteAsync` le
   `rollback-result.json` e conclui: sucesso vira **sticky de 24h**; falha vira **modo SOS**.

**Sticky** = "esta maquina acabou de voltar da versao X, que quebrou. Nao mexer por 24h."

---

# 2. O defeito — R-01

## 2.1 O que esta errado

**O sticky e gravado e nao e lido por quem precisa.**

```
WatchdogState.cs:31,34            definicao de StickyVersion / StickyUntil
WatchdogService.cs:142            le, so para NAO repetir rollback
WatchdogService.cs:304,305        escreve, apos rollback bem-sucedido
```

O `UpdateCheckerWorker` — quem baixa e instala update — **nao consulta sticky**. Ele so consulta
`SosMode` (`UpdateCheckerWorker.cs:211-228`). E rollback bem-sucedido deixa `SosMode = false`.

## 2.2 O que acontece na pratica

1. Maquina na 1.5.13 (quebrada). Falha 3x. Rollback devolve a 1.5.12. Sticky ate +24h.
2. Ate 6h depois o `UpdateCheckerWorker` roda. `SosMode` e false, entao segue. Pergunta ao backend
   estando na 1.5.12; o backend oferece a **1.5.13** — que continua publicada, porque a pausa da
   versao e **manual** e ninguem apertou o botao de madrugada.
3. Baixa 310MB, instala, quebra de novo.
4. Falha 3x outra vez. Agora `stickyAtiva` e verdadeiro e o Watchdog **nao reverte**.

**Resultado: a maquina fica quebrada ate a sticky vencer, 24h depois.** O sticky, que existe para
proteger, e o que prende.

Isto **nao e item novo** na linha de corte — e o item 3a incompleto. Decisao do Marcos em 24/08.

---

# 3. A armadilha — leia com atencao, e o ponto que faz a correcao funcionar ou nao

Os dois lados guardam a versao em **formatos diferentes**. Comparacao por string simples
**nunca casaria**, e a guarda ficaria ali sem nunca disparar. Em silencio.

Medido na maquina do Marcos agora:

```
FileVersionInfo.ProductVersion  ->  1.5.12+b1d227ac93fcdbec72ece38755e9bca2f034f8a9
FileVersionInfo.FileVersion     ->  1.5.12.0
ObterVersaoSemVer()             ->  1.5.12
```

- **`UltimaVersaoQuebrada`** e gravado por `RollbackOrchestrator.LerVersaoInstalada()`
  (`RollbackOrchestrator.cs:198-207`), que devolve `ProductVersion` — **com o hash do git colado**.
- **`info.NewVersion`** vem do backend, no formato de tres partes: `1.5.13`.

`"1.5.13" == "1.5.13+b1d227..."` e **falso**.

Nao ha helper reaproveitavel: o unico `ParseVersion` do repo esta em
`ManagerAgent.Tray/Services/UpdateChecker.cs:143`, e privado, e no projeto legado, e **lancaria
`FormatException`** com o hash colado (`int.TryParse("12+b1d227...")` falha).

## 3.1 O que fazer com isso

Criar um normalizador em `ManagerAgent.Shared` e usa-lo **nos dois lados da comparacao**.

Regra: pega so `major.minor.build`, ignorando o que vier depois de `+`, de `-` ou de uma quarta
parte. Entrada invalida ou vazia devolve `null` — e `null` **nunca casa com nada** (na duvida, a
guarda nao age; barrar update por engano e pior que deixar passar).

Casos que ele precisa tratar, e que valem virar teste:

| Entrada | Saida |
|---|---|
| `1.5.13` | `1.5.13` |
| `1.5.13.0` | `1.5.13` |
| `1.5.13+b1d227ac93` | `1.5.13` |
| `1.5.13-beta.1` | `1.5.13` |
| `1.5` | `null` |
| `""`, `null`, `"abc"` | `null` |

---

# 4. O que fazer

## 4.1 O normalizador

Novo arquivo em `src/ManagerAgent.Shared/` (sugestao: `Runtime/VersaoSemVer.cs`), publico e
estatico, com XML doc explicando **por que** existe — cite os tres formatos medidos acima.

## 4.2 A guarda no `UpdateCheckerWorker`

Arquivo: `src/ManagerAgent.Service/Update/UpdateCheckerWorker.cs`.

**Onde:** dentro de `CheckAndApplyAsync`, **depois** de a resposta do backend chegar e antes do
download. O ponto natural e junto da guarda do **A-46**
(`if (string.Equals(info.NewVersion, currentVersion, ...))`, hoje na linha ~301) — as duas
respondem a mesma pergunta: "esta versao oferecida deve mesmo ser instalada?".

**A regra:** se `StickyUntil` estiver no futuro **e** a versao oferecida normalizada for igual a
`UltimaVersaoQuebrada` normalizada, **nao instala**. Loga em `Warning`, dizendo qual versao foi
barrada e ate quando, e retorna.

**Detalhes que importam:**

- Ler o estado **uma vez** e reusar. O bloco das linhas 211-228 ja faz `_stateStore.Load()` dentro
  de um `try/catch` proprio, porque `watchdog-state.json` pode nao existir num cliente novo.
  **Mantenha esse comportamento:** falha ao ler estado nao pode bloquear update. Se preferir mover
  a leitura para uma variavel unica no comeco, tudo bem — mas o `catch` que deixa passar continua
  obrigatorio.
- `StickyUntil` e `DateTime?` em UTC. Compare com `DateTime.UtcNow`.
- Sticky vencida: **volta a instalar normalmente**, inclusive a versao que tinha quebrado. E
  deliberado — uma versao republicada corrigida com o mesmo numero e caso legitimo, e o operador
  tem o kill switch se nao for.
- Sticky ativa e versao **diferente** da quebrada: **instala normalmente**. O sticky barra uma
  versao, nao congela a maquina.

## 4.3 O que NAO mexer

| Area | Por que |
|---|---|
| `RollbackOrchestrator`, `run-rollback.ps1`, `ReconciliarRollbackPendenteAsync` | acabaram de ser feitos e revisados; a correcao e no consumidor do sticky, nao no produtor |
| Criterio de 3 falhas de startup e a janela de 24h da sticky | provados; mudar numero aqui e outra conversa |
| `SosMode` e o modo SOS | fora de escopo — foi tirado da linha de corte em 24/08 |
| `UpdateApplier` e o PS1 de update | funcionando, exercitado ao vivo em 21/08 |
| `WatchdogStateStore.Save` (escrita atomica) | feito em 24/08 |
| A guarda do A-46 | correta e por outro motivo. Some a sua ao lado, nao no lugar |
| `ManagerAgent.Tray` | projeto legado, so referencia historica |
| Bump de versao | **nao bumpar.** A 1.5.12 ainda nao terminou de ser validada |

---

# 5. Testes

## 5.1 O que escrever

Arquivo novo em `tests/ManagerAgent.Service.Tests/`.

**Do normalizador** — a tabela da secao 3.1 inteira, incluindo os casos que devolvem `null`.

**Da guarda**, no minimo:

1. Sticky ativa + versao oferecida **igual** a quebrada -> **nao instala**.
2. Sticky ativa + versao oferecida **igual a quebrada, mas com o hash colado** -> **nao instala**.
   *(este e o que prova que a armadilha da secao 3 foi tratada; sem ele o resto nao vale)*
3. Sticky ativa + versao **diferente** -> instala.
4. Sticky **vencida** + versao igual a quebrada -> instala.
5. Sem `WatchdogState` / estado ilegivel -> instala (nao bloquear cliente novo).
6. `UltimaVersaoQuebrada` nulo -> instala.

Se `CheckAndApplyAsync` nao for alcancavel por teste sem rede, **extraia a decisao para um metodo
puro** (algo como `DeveBarrarPorSticky(estado, versaoOferecida, agora)`) e teste esse metodo.
Extrair para testar e bem-vindo; **mudar comportamento junto, nao**.

## 5.2 A conferencia que eu vou cobrar

**Desfaca a sua correcao, rode os testes, confirme que falham, restaure.** Diga no relatorio
quantos falharam e quais. Sem isso eu nao aceito o "passou" — foi assim que o rollback quebrado
passou despercebido por meses, com teste verde.

## 5.3 Guardas da base que voces ja quebraram antes

- **ASCII em script PowerShell gerado.** Ha teste que falha se houver caractere nao-ASCII em
  qualquer `.ps1` versionado ou gerado (achado A-05 — PowerShell 5.1 le `.ps1` sem BOM como ANSI e
  o parser quebra). **Travessao (`—`) quebra.** Em 24/08 isso pegou tres erros meus. Se voce nao
  encostar em script, nao te afeta.
- **Avisos de compilacao.** Hoje ha exatamente tres `CS1998`, todos pre-existentes. **Nenhum aviso
  novo.** `CS0649`/`CS8618` sao especialmente serios: um deles escondeu um defeito em 21/08.

---

# 6. Aceite

- [ ] Estado com `UltimaVersaoQuebrada = X` e `StickyUntil` no futuro: o ciclo **nao** instala X e
      registra por que pulou.
- [ ] Vale tambem quando os formatos diferem (`1.5.13` vs `1.5.13+hash` vs `1.5.13.0`).
- [ ] Versao diferente de X continua sendo instalada.
- [ ] Sticky vencida: volta a instalar X.
- [ ] Sem estado no disco: instala, sem bloquear.
- [ ] **1041 testes continuam verdes**, mais os seus.
- [ ] Nenhum teste existente reescrito.
- [ ] Nenhum aviso novo de compilacao.
- [ ] Conferencia da secao 5.2 feita e relatada.

---

# 7. Ao terminar

Escreva `manager-srv-agent/HANDOFF-R01-STICKY.md` com:

- o que mudou, arquivo por arquivo;
- contagem de testes antes e depois;
- **o resultado da conferencia da secao 5.2** — quantos falharam com a correcao desfeita, e quais;
- qualquer coisa que voce tenha encontrado e **nao** corrigido, e por que;
- qualquer ponto em que este documento estava errado. **Isso e util, nao e critica** — em 21/08 e
  em 24/08 voce achou coisas que eu tinha escrito errado no refinamento, e nos dois casos voce
  tinha razao.

**Pare e chame o @Tony** se: a suite nao estiver em 1041 antes de voce comecar; a decisao nao
puder ser testada sem tocar em rede; ou se voce concluir que a guarda precisa mexer em algo da
secao 4.3.

**Nao commite. Nao faca push.**

---

# 8. Nao e desta rodada — para nao se perder

| Item | Estado |
|---|---|
| **R-02** — modo SOS e porta de mao unica (o header `X-Manager-Sos-Recovery-Available` so existe em comentario, ninguem envia nem le) | **fora da linha de corte**, decisao do Marcos em 24/08. Toda maquina que chega ao SOS ja esta com o Service fora do ar e ja precisa de visita — o SOS nao piora nada |
| **R-03** — recall de frota (mandar todas as maquinas voltarem de versao) | **e a sua proxima tarefa, e ja esta escrita:** `registro/2026-08-24-tarefa-bucky-R03-recall-agent.md`. O backend da @Shuri ja esta pronto e revisado. **Nao comece antes de terminar o R-01** — os dois mexem no mesmo metodo, e fazer o recall primeiro obriga a voltar la duas vezes |
| **3b** — teste E2E de rollback no CI | com o @Vision. O cenario `tests/e2e/scenarios/e1-alt-rollback-crash.ps1` nunca rodou |
| Varredura das mensagens e comentarios defasados (9 itens) | rodada propria, depois |

---

# 9. Cenarios de teste — documento irmao

Os casos automatizados desta tarefa (A1–A7) estao especificados na secao 5 acima **e** no
levantamento completo da feature:

**`manager-brain/docs/tecnologia/agent-desktop/TESTES-ROLLBACK-E-RECALL.md`**

Ele cobre as tres pecas juntas — rollback (feito), sticky (esta tarefa) e recall (futuro) —
separando o que ja existe, o que falta em unitario, o que falta em E2E e os testes de maquina
novos (10.5 a 10.11 do plano regressivo).

**O que interessa a voce agora:** secao 2.1 (os sete casos desta tarefa) e o **Teste 10.7**, que e
o teste de maquina que prova o R-01 — e que o Marcos ou a @Natasha vao rodar depois.
