# Linha de corte para producao — Agent Windows (2026-08-21)

> **Decisao:** @Tony | **Aprovador:** Marcos
> **Vale para:** o que segura a ida do Agent Windows para producao. Nada mais.
>
> **Documento autoritativo.** Achado novo nao entra aqui por ser grave; entra se cruzar o
> criterio da secao 1. Quem quiser incluir algo, argumenta contra o criterio, nao contra a lista.

---

## Por que este documento existe

Marcos, hoje: *"parece que toda hora aparece algo e nunca finalizamos isso"*.

Ele esta certo, e a causa nao e a quantidade de achados — e a **falta de uma linha definida**. Sem
criterio escrito, todo achado novo parece candidato a bloqueador, a lista so cresce e nao existe
estado final possivel. **A linha e responsabilidade do TL e nao tinha sido tracada.** Esta e ela.

---

## 1. Criterio

Segura producao **apenas** o que se encaixa em um destes:

| | Criterio |
|---|---|
| **A** | Perde ou corrompe dado do colaborador em maquina individual |
| **B** | Pode derrubar a frota sem caminho de volta |
| **C** | Expoe credencial ou dado pessoal |

**Nao segura:** ruido de log, divida de disco, comentario defasado, defeito que so aparece em
cenario fora do ICP, e qualquer coisa que o rollout em etapas (secao 4) consiga conter.

---

## 2. O que segura — cinco itens

| # | O que | Criterio | Dono | Feito quando | Estado |
|---|---|---|---|---|---|
| 1 | **Um item invalido derruba o lote inteiro no `IngestionService`** (`safeParse` no corpo todo) | A | @Shuri | O backend recusa item por item e aceita o resto do lote | codigo pronto e publicado; sem teste em maquina |
| ~~2~~ | ~~`eventos_sessao` sem UNIQUE~~ | — | — | **REMOVIDO em 2026-08-24 por decisao do Marcos** — ver secao 3 | — |
| 3a | **Ligar o rollback do auto-update ao backup que existe** | B | @Bucky | Instalacao real, versao que quebra no startup: o Service volta sozinho na versao anterior | **fechado em 24/08** — mecanismo exercitado em maquina real pelo recall |
| 3b | **Teste E2E de rollback rodando no CI** | B | @Bucky + @Vision | `e1-alt-rollback-crash.ps1` verde, contra a instalacao gerada pelo instalador de verdade | em aberto; falta decidir a maquina |
| 4 | **Senha do banco de staging trocada** | C | @Vision | Senha nova em uso, a antiga revogada | em aberto; a senha passou pelo chat duas vezes |
| **5** | **A telemetria do Watchdog nunca chega ao servidor** — o reporter procura o token em texto puro, que o `ConfigManager` apaga do disco ao criptografar; a requisicao sai sem autenticacao e volta 401 | **secao 4** | @Bucky | Um evento do Watchdog aceito pelo backend em maquina real, **e** falha de envio aparecendo como falha no log | **ACEITO em 2026-08-24 por decisao do Marcos**; em correcao |

> **Sobre o criterio do item 5.** Ele nao encaixa em A, B nem C — nao perde dado, nao derruba a
> frota, nao expoe credencial. Entrou por outro caminho: a **secao 4** deste documento diz que o
> que encerra a lista e o rollout em etapas, e o criterio de avanco do rollout e *"telemetria
> zero"*. Os eventos que nunca chegam sao exatamente os de seguranca de frota — volta automatica e
> modo de emergencia. Zero garantido por construcao nao e zero medido. **E o primeiro item que
> entra por proteger o mecanismo de encerramento, e nao por um dos tres criterios diretos.**
> Detalhe completo no bloco "Item 5" no fim deste documento.

### Notas de cada um

**1 — e o unico que piorou nesta rodada, por decisao minha.** O teto do A-64 trocou "fila travada
para sempre" por "descarta ate 100 eventos apos 5 tentativas". Foi a troca certa — a fila travada
queimava a janela de emergencia de todo desligamento —, mas ela converteu um travamento visivel em
**perda de dado silenciosa**. So o backend fecha isso. Enquanto nao fechar, a perda e real.

**2 — removido em 2026-08-24, por decisao do Marcos.** Ver secao 3.

**3a e 3b — regra nossa, do `REGRAS-RELEASE.md` secao 5.2**, e ela disparou porque **eu alterei o
`UpdateApplier`** (motivo `UPDATE` no lugar do texto livre). A regra existe exatamente para o caso
em que uma versao ruim se instala sozinha na frota inteira.

> **Atualizado em 2026-08-24.** Era um item so — "escrever o teste". Ao ir escrever, descobri que
> **o rollback nao funciona**: o `RollbackOrchestrator` procura o backup em
> `C:\Program Files\ManagerAgent\bin.previous`, que nao existe, enquanto o `UpdateApplier` grava em
> `C:\ProgramData\ManagerAgent\backups\pre-*`. Duas convencoes escritas em lados diferentes que
> nunca se encontraram. Hoje, versao que nao sobe = maquina sem captura ate reinstalar a mao.
> Detalhe e caminho de correcao: `registro/2026-08-24-rollback-nao-esta-ligado.md`.
>
> O 3b sem o 3a e teatro — o teste passaria no ambiente sintetico e continuaria mentindo.

**4 — passou pelo chat em 20/08.** Ja estava em aberto antes desta wave.

---

## 3. O que NAO segura — decidido, nao esquecido

| O que | Por que fica fora |
|---|---|
| **A-66** — `session-state-N.json` nao grava em sessao que trocou de dono | **So acontece em maquina compartilhada entre pessoas. Marcos confirmou hoje: o cliente e maquina individual.** Fora do ICP para este lancamento. Volta se vendermos para turno compartilhado (call center) |
| **Nove mensagens/comentarios defasados** (as tres do refinamento ja corrigidas, mais `WorkerLauncher.cs:361` e oito comentarios "backend deriva o status") | Nao perde dado. Varredura de texto numa rodada propria, depois que nao houver comportamento aberto |
| **Arquivo `.corrompido` nunca sai do disco** | Divida de disco. Prazo entra junto com o passivo de eventos abertos, com o @Steve |
| **Banco local sem teto de tamanho** (nao ha retencao no Agent — confirmado: nenhuma referencia a retencao no codigo) | Cresce ~1MB por dia por colaborador, e so enquanto o backend estiver fora do ar. O teto do A-64 ja tirou a causa que crescia para sempre |
| **Passivo de eventos abertos** (33 ociosidades, 27 janelas) | Dado historico, decisao de produto com o @Steve. Nao impede o proximo evento de entrar certo |
| **A-63 exercitado na maquina** | **Fechado.** O que inverteu a ordem em 15:21 foi um lancamento travado por 37s — nao se encena por vontade. Dois testes falham com a correcao desfeita, conferidos por @Tony e @Bucky separadamente. Isso basta |
| **`eventos_sessao` sem UNIQUE** *(era o item 2)* | **Saiu da lista em 2026-08-24, decisao do Marcos.** Eu tinha escrito que a trava no banco fechava a brecha do D5. **Errado, e conferido no codigo:** os dois LOGOUT daquele cenario carregam `ocorreuEm` diferentes — o do Service usa o instante do EOF do pipe, o do worker usaria o `UtcNow` dele. UNIQUE por igualdade exata nao junta os dois; fechar de verdade exigiria campo novo no contrato do Agent (`logon_id`) e deploy ordenado. Somado a isso, **a duplicata do D5 nunca foi observada** — as tres classes medidas (janela, ociosidade, reuniao) ja estao corrigidas. Protege so contra reenvio, que e classe real e menor. Volta com evidencia, se aparecer |

---

## 4. O que de fato encerra — o rollout em etapas

**A lista de achados nunca vai acabar. O que encerra e o rollout, nao a lista.**

Ja esta escrito em `agent-desktop/REGRAS-RELEASE.md` secao 5.1 e no ADR-008. Cinco etapas, com
telemetria zero como criterio de avanco:

| Etapa | Config | Minimo |
|---|---|---|
| 1. Canario | `canary_only=true`, 1-3 empresas | 48h |
| 2. Opt-in 10% | `rollout_percent=10` | 3 dias |
| 3. Opt-in 50% | `rollout_percent=50` | 3 dias |
| 4. Opt-in 100% | `rollout_percent=100, obrigatoria=false` | 7 dias |
| 5. Obrigatoria | `obrigatoria=true` | frota >=95% na versao nova |

Botao de pausa em qualquer etapa:
```sql
UPDATE versoes_agente SET pausada = true WHERE versao = 'X.Y.Z';
```

**E isto que permite ir para producao com bug conhecido.** Sem o rollout, nenhuma lista seria
suficiente, porque sempre aparece mais um. Com ele, o pior caso de um achado que escapou e uma
empresa afetada por 48h, com volta imediata.

---

## 5. Como usar

- Achado novo: passa pelo criterio da secao 1. Nao passou, vai para a proxima wave e **nao se
  discute de novo**.
- Este documento so muda com decisao do Marcos registrada aqui.
- Os quatro itens da secao 2 sao o estado final. Fechados eles + rollout na etapa 1, vai para
  producao.

---

## Estado em 2026-08-21

| | |
|---|---|
| Agent | 1.5.12 rodando na `DESKTOP-VMSM6LE`, fila de upload limpa, 1010 testes verdes |
| Codigo | **nao commitado, nao empurrado** |
| A-63 / A-64 / A-65 | fechados. A-64 provado em maquina; A-63 provado por teste |
| Itens da secao 2 | **quatro em aberto** (1 com a @Shuri; 3a e 3b com o @Bucky; 4 com o @Vision). O item 2 saiu em 24/08 |

**Referencias:** `registro/2026-08-21-refinamento-bucky-a63-a64-a65.md`,
`registro/2026-08-21-review-tony-a63-a64-a65.md`, `registro/2026-08-21-bateria-1.5.12.md`,
`registro/2026-08-21-decisoes-do-dia.md` (D7), `manager-srv-agent/HANDOFF-2026-08-21-A63-A64-A65.md`.

---

## Estado em 2026-08-24

> Registro de estado por @Tony. **A secao 1 (criterio) e a lista de itens da secao 2 nao mudaram
> aqui** — muda o que ja foi feito e o que ainda falta em cada um.

| Item | Dono | Estado ao fim do dia |
|---|---|---|
| 1 — lote nao cai inteiro | @Shuri | **codigo pronto e commitado.** Nao testado em maquina |
| 3a — rollback ligado ao backup | @Bucky | **codigo pronto e commitado.** Nao testado em maquina |
| 3b — teste E2E de rollback no CI | @Bucky + @Vision | **em aberto, nunca rodou.** Depende de decisao de maquina |
| 4 — senha do staging | @Vision / Marcos | **em aberto.** Ver nota abaixo |

### O que mudou de verdade hoje

**O 3a nao estava completo, e virou tres tarefas.** Ligar o rollback ao backup nao bastava:

- **R-01** — a maquina que voltava de uma versao quebrada **reinstalava a mesma versao** no ciclo
  seguinte. Guarda no `UpdateCheckerWorker`, com normalizador de versao: os dois lados guardam a
  versao em formatos diferentes (`1.5.12+hash` contra `1.5.12`) e a comparacao por texto simples
  nunca casaria. Feito. `manager-srv-agent/HANDOFF-R01-STICKY.md`.
- **R-02** — a guarda do R-01 nascia inerte: `WatchdogState.VersaoAtual` e escrito em **um unico
  lugar do produto**, dentro do proprio rollback, e esta vazio em toda maquina da frota. A versao
  quebrada passa a ser capturada antes de o script trocar os binarios. Destrava **cinco**
  consumidores, a guarda entre eles — a auditoria reportava `versaoQuebrada: null` e a pasta da
  versao quebrada saia como `bin.failed-unknown-*`. Feito.
  `manager-srv-agent/HANDOFF-R02-VERSAO-QUEBRADA.md`.
- **R-03** — recall de frota: mandar de volta quem **ja tem** a versao ruim, distinto do `pausada`,
  que so impede novas entregas. Backend feito e commitado (@Shuri,
  `manager-srv-admin-node/HANDOFF-R03-RECALL.md`); lado Agent em desenho com o @Bucky.

**Terceira aparicao do mesmo padrao.** O rollback quebrado (manha), o `VersaoAtual` vazio (tarde) e
a assercao frouxa do `e1-alt` tem a mesma causa: *o teste e o codigo concordam entre si, e os dois
discordam do instalador de verdade.* Nao e coincidencia e nao se resolve achando defeito por
defeito — e o 3b que fecha, porque so ele exercita a instalacao real.

### Nota do item 4 — mudou de natureza, precisa de decisao do Marcos

A senha de staging **passou pelo chat outra vez em 24/08**. O item deixa de ser "trocar a senha":
enquanto a troca depender de alguem colar o valor em algum canto, isso se repete. Proposta de
redacao nova do "feito quando", **pendente de decisao do Marcos**:

> Senha nova em uso, a antiga revogada, **e o @Vision com acesso ao painel** — nenhuma troca futura
> passando por mensagem.

**Em aberto, sem resposta:** o staging e o mesmo projeto Supabase da producao? Se for, o item sobe
para o criterio **C** (expoe credencial ou dado pessoal) e deixa de ser higiene.

### Achado operacional que nao estava em nenhuma lista

Rodar a suite de testes do Agent **para os servicos de verdade da maquina que a roda** — os oito
testes de `RollbackCaminhoRealTests` disparam um script que para `ManagerAgent` e
`ManagerAgentWatchdog` pelo nome, ~17s por vez. So nao acontece sempre por uma corrida ganha por
acaso; em maquina mais lenta, acontece. **Toda bateria rodada na `DESKTOP-VMSM6LE` pode ter aberto
buraco de captura.** Freio aprovado por @Tony e em aplicacao pelo @Bucky.

**Nao muda a secao 1**: nao segura producao — nao afeta maquina de cliente, so a de quem roda a
suite. Fica registrado porque muda a leitura das baterias passadas, e isso e da @Natasha.

### Estado do codigo

**Commitado localmente em 24/08, autorizado pelo Marcos. Nada empurrado.**

| Repo | Commit | Testes |
|---|---|---|
| `manager-srv-agent` | `c680cd2` (branch `staging`) | 1010 -> **1092** |
| `manager-srv-admin-node` | `257096f` + merge `2910c9f` | 1105 -> **1199** (com o que veio do remoto) |

### Posicao da @Natasha

**Bloqueia a ida para producao.** A peca que protege a frota nunca foi executada uma unica vez em
maquina, e a regra 5.2 do `REGRAS-RELEASE` exige esse teste passando. @Tony sustenta o bloqueio.

**Referencias de 24/08:** `registro/2026-08-24-rollback-nao-esta-ligado.md`,
`registro/2026-08-24-entrega-item1-e-rollback.md`,
`registro/2026-08-24-refinamento-tony-R02-versao-quebrada.md`,
`registro/2026-08-24-tarefa-bucky-R01-sticky.md`,
`registro/2026-08-24-tarefa-shuri-R03-recall.md`,
`registro/2026-08-24-tarefa-bucky-R03-recall-agent.md`.

---

## 24/08, fim da tarde — o mecanismo rodou em maquina real

> Registro de @Tony. Marcos executou; @Tony acompanhou os logs ao vivo.
> Evidencia crua em `C:\Temp\ManagerAgent-Tests\2026-08-24\C11-recall.txt`.

**O recall de frota foi disparado de verdade e funcionou fim a fim.** E a primeira vez que a
maquinaria de restauracao roda desde que existe.

| Hora | O que |
|---|---|
| 14:43 | maquina chega na 1.5.14 pela **auto-atualizacao** (nao foi instalacao manual) |
| 14:55:05 | Marcos revoga a 1.5.14; a ordem chega na primeira consulta, com o motivo integro |
| 14:56:04 | Watchdog pega a marca — 59s, um ciclo de poll |
| 14:56:05 | script de restauracao disparado; servicos param |
| 14:56:47 | Service volta na 1.5.13 |
| 14:56:51 | reconciliacao: `SwapDone=true ServiceRunning=true Desfecho=Success` |
| 14:56:55 | nova consulta: **nao tenta reinstalar a 1.5.14** |

**42 segundos sem captura. Nunca entrou em SOS.**

### O que deixou de ser promessa e virou fato

| | |
|---|---|
| Restauracao (a peca do item 3a) | roda, em instalacao real |
| **R-02** | `ultimaVersaoQuebrada = 1.5.14+793e743…` — de manha esse campo nascia vazio |
| Nome da pasta da versao ruim | `bin.failed-1.5.14+793e743…`, e nao `bin.failed-unknown` |
| **R-01** | sticky de 24h gravado; a versao revogada nao foi reoferecida |
| Dono unico da restauracao | o Service so marcou; o Watchdog executou — o desenho aprovado se sustentou |
| Auto-update | duas vezes no dia, 1.5.12→1.5.13 e 1.5.13→1.5.14, com checksum conferido |

**O item 3a pode ser dado como fechado em maquina.** O 3b **nao** — ele e o teste automatizado, e
continua sem rodar. O que rodou hoje foi uma execucao manual, uma vez.

---

## Item 5 — **ACEITO em 2026-08-24 por decisao do Marcos**

> Recomendado por @Tony, aceito pelo Marcos no mesmo dia. Consta na lista da secao 2.
> A lista passa a ter **cinco** itens.

**A telemetria do Watchdog nunca chega ao servidor.** Descoberto porque o recall rodou de verdade:

```
14:56:05 [Watchdog] WatchdogAudit: reported (evento=UPDATE_RECALL_TRIGGERED, status=401)
```

**Causa, rastreada no codigo:** o `ConfigManager` criptografa o token da maquina e **apaga o texto
puro do disco** (`ConfigManager.cs:244` — `config.DeviceToken = null; // remove do disco`). O
`HttpWatchdogAuditReporter` abre o `config.json` na mao e procura **so** o campo em texto puro
(`HttpWatchdogAuditReporter.cs:117`). Nao acha, e entao **nem envia o cabecalho de autenticacao**.
O Watchdog nao tem uma linha de DPAPI e nao referencia o `ConfigManager`.

**Alcance:** nao e so o recall. Passam pelo mesmo reporter a volta automatica disparada, a volta
automatica falhou, o recall disparado/falhou/recusado e a limpeza de artefatos. **O canal inteiro
de telemetria do Watchdog esta morto, em silencio** — o codigo trata envio de audit como auxiliar
e segue. O canal do Service funciona, porque usa o leitor certo; por isso ninguem notou.

**Por que eu recomendo que segure.** Nao encaixa literalmente em A, B nem C: nao perde dado, nao
derruba a frota, nao expoe credencial. Mas a **secao 4** deste documento diz que o que encerra a
lista e o rollout em etapas, e o criterio de avanco dele e **"telemetria zero"**. Os eventos que
faltam sao exatamente os de seguranca de frota. Se eles nunca chegam, o zero e garantido por
construcao: a rede que torna aceitavel ir para producao com bug conhecido esta com o sensor
desligado.

| | |
|---|---|
| Dono | @Bucky (em correcao) |
| Feito quando | um evento do Watchdog e aceito pelo backend em maquina real, e falha de envio aparece como falha no log |
| Risco da correcao | o leitor compartilhado parece **escrever** ao carregar; usar sem cuidado poe Service e Watchdog gravando o mesmo arquivo sem trava |

**Aceito pelo Marcos em 24/08.** A lista passa a ter cinco itens.

### Setimo achado do mesmo padrao

Nenhum dos sete defeitos do dia foi encontrado por teste automatizado. Todos por alguem abrindo o
codigo, ou — este — por rodar o mecanismo de verdade uma unica vez. Os testes conferem que a
chamada **foi feita**; nenhum confere que ela **foi aceita**.
