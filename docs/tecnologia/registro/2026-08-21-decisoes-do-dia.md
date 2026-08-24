# Decisoes de 2026-08-21

> **Registro:** @Tony | **Aprovador:** Marcos
> Origem: refinamentos do dia para @Bucky (Agent Windows) e @Shuri (backend Node).

---

## D1 — Saneamento das reunioes duplicadas: APROVADO

**Decisao do Marcos.** O criterio proposto pela @Shuri no passo 1 e 2 da migration
`20260821000000_uq_eventos_reuniao_sessao.sql` esta aprovado:

- dentro de cada grupo `(agente_id, aplicativo, iniciado_em)` com mais de uma linha, sobrevive
  a de **menor id**;
- a sobrevivente recebe o `finalizado_em` **mais tardio** do grupo;
- `titulo_reuniao`, `participantes_detectados` e `render_confianca` so sao preenchidos onde a
  sobrevivente estava nula;
- as demais linhas do grupo sao removidas;
- **nenhuma duracao e inventada** — grupo sem nenhum `finalizado_em` continua aberto e cai no
  criterio geral do passivo de eventos abertos, que segue **em aberto com o @Steve**.

**Por que precisou de decisao, e por que nao dava para adiar:** o handler corrigido depende da
UNIQUE `uq_eventos_reuniao_agente_app_inicio`, e essa UNIQUE **nao pode ser criada com
duplicata na tabela**. Sem aprovar a limpeza, a correcao do A-57 nao existe. Nao era um extra.

Mesmo criterio ja aprovado antes para `eventos_janela` (`20260815000000`) e
`eventos_ociosidade` (`20260815010000`).

**Estado:** codigo e migration escritos, **nao commitados, nao executados**. Deploy e do
@Vision.

---

## D2 — Quem emite o LOGOUT no logoff (A-62)

**Decisao do @Tony**, a partir de evidencia levantada pelos dois lados.

O caminho aprovado no refinamento continua: **o Service emite o LOGOUT**, porque e ele que
sobrevive ao logoff — o worker morre ~1s antes de o Service saber que houve logoff.

**A restricao nova:** o `eventos_sessao` **nao tem UNIQUE nenhuma** (varrido em todas as
migrations do `manager-srv-events-node`; existem UNIQUE apenas para `eventos_janela`,
`eventos_ociosidade` e agora `eventos_reuniao`). O `BatchDedupService` nao cobre este caso — so
pega reenvio de batch byte-a-byte identico, em 10 min, e so com o header
`X-Client-Dedup-Batch`. E os dois LOGOUT nasceriam em **buffers diferentes** (Service e
worker), entao o `client_dedup_key` local tambem nao os junta.

**Regra definida:** o Service so emite o LOGOUT **quando nao houve `GoodbyeMessage` daquela
sessao**. O worker emite quando da tempo; o Service cobre o buraco quando nao da. **Nunca os
dois.**

Em aberto para o @Bucky responder: o caso da `GoodbyeMessage` que chega **depois** de o Service
ja ter emitido.

---

## D3 — Proposta de ciclo de vida do SO: aprovada em partes

Proposta em `2026-08-21-proposta-bucky-ciclo-vida-so.md`. **Aprovado pelo Marcos executar os
passos 0, 1 e 2.**

| Passo | O que | Estado |
|---|---|---|
| 0 | `PipeServer.cs:196` remocao condicional + reativar os 3 arquivos de teste excluidos | autorizado |
| 1 | `SessionLifecycleTracker` + janela de confirmacao antes de relancar (A-61) | autorizado |
| 2 | Relogio unbiased no heartbeat, com fallback (A-56 na origem) | autorizado |
| 3 | Service emite LOGOUT + flag comparada por LogonId (A-62) | **desbloqueado por D2**, aguarda liberacao |
| 4 | Rejeicao ativa + `DUPLICATE_WORKER` (invariante "so um agent") | **retido** — maior risco, Marcos revisa o desenho antes |

Decisoes tecnicas tomadas dentro da proposta:
- `QueryUnbiasedInterruptTime` como correcao principal; `OnPowerEvent` fica fora desta rodada.
- Janela de confirmacao de ~5s em vez de "ensinar o watchdog" — a guarda ja existe e foi
  consultada 981ms antes de existir a informacao de que precisava.
- Flag de shutdown limpo passa a ser **comparada por `logonId`, nao apagada** — resolve junto a
  reciclagem de SessionId pelo Windows.
- Processo vivo **nao se relanca**.

---

## Divida registrada, sem dono nesta rodada

| # | O que | Origem |
|---|---|---|
| — | `eventos_sessao` e `eventos_transicao_status` sem UNIQUE e sem dedup real | @Shuri, hoje |
| — | Um item invalido derruba o batch inteiro no `IngestionService` (`safeParse` no corpo todo) | @Shuri, hoje |
| — | `statusAnterior` obrigatorio no Zod e NOT NULL — alinhar com o @Bucky antes do A-58 | @Shuri, hoje |
| — | `HeartbeatMonitor` do Watchdog externo tem o mesmo defeito de relogio do A-56 | @Bucky, hoje |
| — | Os 3 arquivos de teste em `<Compile Remove>` — os 753 verdes nao cobrem pipe nem watchdog | @Bucky, hoje |

O A-58 vai devolver volume a `eventos_transicao_status`. Os tres primeiros itens acima entram
no refinamento do A-58, nao antes.

---

## Ainda em aberto com o Marcos

- Os ~150 arquivos alterados e nao commitados no `manager-srv-agent`, misturando a rodada da
  1.5.10 com o que esta sendo escrito agora.
- A senha do banco de staging, que passou pelo chat em 20/08. **Trocar.**
- Passivo geral de eventos abertos (33 ociosidades, 27 janelas) — decisao com o @Steve.

---

## D4 — Entregas do @Bucky (fim do dia)

Testes do Agent Windows: **753 -> 935**, quatro suites verdes. **Nada commitado pelo @Bucky.**
Marcos commitou os 150 arquivos da 1.5.10 as 08:53 (`b1d227a`); o trabalho desta rodada
continua solto.

| Item | Estado |
|---|---|
| Passo 0 — `PipeServer` + reativar os 3 arquivos de teste | feito (+41 testes) |
| Passo 1 — `SessionLifecycleTracker` + janela de confirmacao (A-61) | feito (+16) |
| Passo 2 — relogio unbiased (A-56 na origem) | feito (+9) |
| Passo 4 — rejeicao ativa + `DUPLICATE_WORKER` (invariante do Marcos) | feito (+34) |
| A-58 — ATIVO/PAUSA/AUSENTE | feito (+34) |
| A-60 — buffer por sessao + varredura de orfaos | feito (+23) |
| Passo 3 — Service emite LOGOUT (A-62) | feito (+27). **Refinamento fechado.** |

**Pre-requisito antes de testar na maquina: bump para 1.5.11.** Sem isso o agent novo se declara
antigo e o encerramento do worker duplicado cai na via bruta (`Kill()` seco).

Decisoes tomadas dentro da rodada:
- **A-58 nao assume ATIVO no boot** — calcula o estado inicial pelo silencio real e adota **sem
  emitir**. Garante `statusAnterior` conhecido em toda transicao (exigencia do backend) e evita
  registrar um "voltou a trabalhar" que nunca aconteceu.
- **Campo `motivo` fica** na transicao de status. Conferido no `srv-events-node`: opcional no
  schema de entrada (255) e coluna existente. Paridade com o Android e de nomes e formato, nao
  de contagem de campos.
- **A-60 por buffer por sessao**, confirmado pela causa raiz real: a pasta usa a permissao
  padrao do Windows, em que quem cria o arquivo vira dono. Nao era permissao errada — era o
  comportamento normal. Por isso `session-state-N` sempre funcionou.
- **Varredura de buffers orfaos no Service** (so o Service consegue: o worker tem leitura no
  arquivo dos outros). So toca arquivo de sessao que o Windows nao lista mais e parado ha 10
  min. Entrega antes de apagar, sob o nome do dono original. Recolhe tambem o buffer da versao
  anterior — o nome muda no update, que e justamente quando o canal cai.

---

## Divida nova descoberta hoje (nao corrigida, de proposito)

| # | O que | Gravidade |
|---|---|---|
| 1 | **As duas funcoes de limpeza do buffer no Windows sao conchas vazias.** O trim de 10k eventos / 7 dias e do Android, nao existe aqui — o banco cresce sem teto. O buffer por sessao multiplica por N um crescimento que ja era ilimitado. A varredura tampa o usuario que sumiu, **nao** o usuario ativo com canal dias fora do ar | ALTA |
| 2 | **Funcao morta no instalador que trancaria a pasta de dados so para o SYSTEM.** Se alguem liga-la achando que endurece seguranca, nenhum worker grava buffer e a captura offline morre em silencio na frota inteira | ALTA (latente) |
| 3 | `TryLaunchAsync_ComLaunchingTravado_SubstituiOWorker` — teste instavel, anterior a esta rodada. Nunca falhou nas 15 execucoes de hoje. Recomendacao do @Bucky, aceita: deixar quieto | BAIXA |
| 4 | Comentario falso em `DailyBoundaryWorker.cs` ("backend deriva status") — **corrigido hoje**. Era a unica documentacao da decisao, e sustentou o engano por tres meses | — |

---

## Testes que so a maquina prova — 15 itens, ordem sugerida

Do @Bucky. Os que enganam sao os de dormir, os de dois usuarios e o 14: quando falham, **nao ha
erro na tela** — some registro no banco.

1. Reboot | 2. Matar o worker a forca | 3. Dormir 3 min | 4. Dormir 30 min | 5. Logoff e login |
6. Desligar e ligar | 7. Troca rapida entre dois usuarios | 8. Parar 6 min (PAUSA), 16 (AUSENTE),
mexer (ATIVO) | 9. Lock/unlock nao gera transicao | 10. Noite parada gera **um** evento |
11. **Nenhum lote rejeitado** depois que a frota voltar a emitir | 12. Segundo usuario sem erro
de banco somente-leitura | 13. Servico derrubado com dois usuarios: cada um preserva os seus |
14. **Apos atualizar da 1.5.10: buffer antigo recolhido, eventos no banco** | 15. Usuario
removido: buffer dele sai do disco sozinho

**Olhar primeiro:** o 11 (derruba lote inteiro, leva janelas e reunioes junto) e o 14 (unico que
pode perder dado).

(Item 5 revisado — ver D5 no fim deste documento.)

---

## D5 — Passo 3 entregue: o refinamento do @Bucky esta fechado

Testes 908 -> **935**. Nada commitado.

- **O Service emite o LOGOUT** quando o Windows mata o worker antes de avisar, com o instante do
  EOF — nao o do processamento.
- **Guarda contra duplicata em duas camadas:** se houve `GoodbyeMessage`, o Service nao emite; e
  imediatamente antes de gravar, confere no proprio buffer local se ja existe um LOGOUT daquela
  sessao.
- **A flag de shutdown limpo passou a ser comparada por `logonId`, nao apagada.** Mata junto a
  quarta origem de alarme falso (o Windows recicla o numero da sessao entre pessoas diferentes).
- O `ShutdownMessage` continua existindo — worker vivo ainda fecha os eventos em andamento.

### Item 5 da lista da maquina (logoff e login) — revisado

Quatro verificacoes: sai **exatamente uma** saida, com o horario do logoff e **antes** do
fechamento da ultima janela; o login seguinte nao acusa interrupcao; a saida chega ao banco em
segundos, sem esperar o proximo boot; nenhum worker nasce entre a queda e o logoff.

Fazer em **duas variacoes**, que exercitam emissores diferentes:
- **logoff** — o Windows mata o worker primeiro, quem emite e o **Service**;
- **parar o Service com o usuario logado** — o worker esta vivo e se despede, quem emite e **ele**.

Nos dois casos: **uma** linha, nunca duas.

### Brecha conhecida, coberta por teste com esse nome

Se o pipe estiver fora do ar no instante do logoff, o LOGOUT do worker fica no buffer dele e pode
subir muito depois — as vezes so no proximo boot. Nesse instante o Service nao tem como ve-lo,
emite o dele, e a segunda copia chega mais tarde.

**Nenhuma verificacao do lado do agent fecha isso.** So a UNIQUE em `eventos_sessao`, que esta
na divida do backend. Se aparecerem duas linhas no teste do item 5, e esta brecha — a correcao
nao e no Agent.

---

## D6 — Conferencia antes de publicar (@Tony)

Duas coisas encontradas na maquina, antes de o Marcos gerar o pacote.

**1. O bump nao tinha sido feito.** Os sete `.csproj` ainda diziam `1.5.10`. O
`build-pacote-v2.ps1` le a versao do `ManagerAgent.Service.csproj`, entao o pacote sairia se
declarando igual ao que ja esta instalado. **Corrigido: 1.5.11 nos sete.**

**2. Defeito real que os 935 testes nao pegaram.**
`ManagerAgentService.cs` recebia `ICleanShutdownFlagStore` no construtor e **nunca atribuia ao
campo**. O compilador avisou (CS0649 + CS8618) e o aviso passou batido.

Efeito: `MarcarShutdownLimpoAsync` chamaria em cima de `null` a cada logoff, desligamento e
reboot. O `catch` em volta registra "nao-fatal" e segue — ou seja, **a flag de shutdown limpo
nunca seria gravada, em silencio**. Todo boot seguinte acusaria interrupcao. Isso derrubaria os
itens 1, 5 e 6 da lista da maquina, e o motivo nao apareceria na tela.

Exatamente o modo de falha que o @Bucky descreveu: nao ha erro, some registro.

**Corrigido:** uma linha de atribuicao no construtor. 935 testes verdes, sem os dois avisos.
**@Bucky confere na proxima rodada** — a duvida legitima e por que nenhum dos 27 testes do
Passo 3 instanciou o servico real.

Nada commitado.

---

## D7 — Linha de corte para producao: definida

**Decisao do Marcos, a pedido dele.** Motivo, com as palavras dele: *"parece que toda hora aparece
algo e nunca finalizamos isso"*.

A causa nao era a quantidade de achados — era nao existir criterio escrito do que segura producao.
Sem ele, todo achado novo parecia candidato a bloqueador e nao havia estado final possivel. **Era
falha minha como TL nao ter tracado a linha.**

Documento autoritativo: **`registro/2026-08-21-linha-de-corte-producao.md`**.

**Criterio.** Segura producao so o que: (A) perde ou corrompe dado do colaborador em maquina
individual; (B) pode derrubar a frota sem volta; (C) expoe credencial ou dado pessoal.

**Quatro itens seguram:**

| # | O que | Dono |
|---|---|---|
| 1 | Um item invalido derruba o lote inteiro no `IngestionService` | @Shuri |
| 2 | `eventos_sessao` sem UNIQUE | @Shuri |
| 3 | Teste E2E de rollback do auto-update (regra 5.2, disparada por eu ter alterado o `UpdateApplier`) | @Bucky + @Vision |
| 4 | Senha do banco de staging trocada | @Vision |

O item 1 **piorou nesta rodada por decisao minha**: o teto do A-64 trocou fila travada por
descarte de ate 100 eventos. Foi a troca certa, e converteu um travamento visivel em perda de dado
silenciosa. So o backend fecha.

**Ficam de fora, decididos e nao esquecidos:** A-66 (`session-state` de sessao que trocou de dono
— **o cliente e maquina individual**, confirmado pelo Marcos hoje), as nove mensagens e comentarios
defasados, o arquivo em quarentena que nao sai do disco, o banco local sem teto e o passivo de
eventos abertos.

**O que encerra nao e a lista acabar — e o rollout em etapas** (`REGRAS-RELEASE` 5.1 + ADR-008).
Canario 48h, 10%, 50%, 100%, obrigatoria, com pausa em qualquer etapa. E ele que permite ir para
producao com bug conhecido: o pior caso vira uma empresa afetada por 48h, com volta imediata.

**Regra de uso:** achado novo passa pelo criterio. Nao passou, vai para a proxima wave e **nao se
discute de novo**.

---

## D8 — A-63, A-64 e A-65 fechados

Entrega do @Bucky, review em `registro/2026-08-21-review-tony-a63-a64-a65.md` (aprovado, com o R1
que eu mesmo corrigi). Bateria em `registro/2026-08-21-bateria-1.5.12.md`.

Provado na maquina com a 1.5.12: o teto de tentativas (contador de `1/5` a `5/5`, dois lotes
descartados as 16:57, fila limpa as 16:59 — antes eram 93 recusas sempre em `Attempt=1`) e a
quarentena do buffer ilegivel (o aviso de 15 em 15s parou). O logoff das 17:02 saiu com **uma**
linha e `Usuario=NoisyTech`.

**A corrida do A-63 nao se reproduziu no teste**, e isso e aceito: o que a inverteu em 15:21 foi um
lancamento travado por 37s, que nao se encena por vontade. O achado fecha pelos dois testes que
falham com a correcao desfeita, conferidos por @Tony e @Bucky separadamente.

1010 testes verdes. **Nada commitado.**

---

## D9 — 2026-08-24: item 2 sai da linha de corte; @Shuri e @Bucky soltos

**Decisao do Marcos.**

**Item 2 (`eventos_sessao` sem UNIQUE) sai da linha de corte.** Motivo: eu tinha escrito que a
trava no banco fechava a brecha do D5, e **conferindo o codigo do Agent, nao fecha** — os dois
LOGOUT daquele cenario carregam `ocorreuEm` diferentes (Service usa o EOF do pipe; worker usaria o
`UtcNow` dele), e UNIQUE por igualdade exata nao junta os dois. Somado a isso, a duplicata do D5
nunca foi observada. Fica para o futuro, volta com evidencia.

**@Shuri desenvolve o item 1** — `registro/2026-08-24-refinamento-shuri-lote-e-sessao.md`.
Causa medida: `windowsUsername` no schema Zod e `.optional()`, que aceita ausente e recusa `null`
explicito, que e o que o Agent manda. O resto do caminho ja trata null. Junto vai a regra "erro de
lote e 400, erro de item nunca e 400", usando o `motivosIgnorados` que ja existe.

**@Bucky desenvolve o rollback (3a + 3b)** —
`registro/2026-08-24-refinamento-bucky-rollback.md`.

**Correcao de um achado meu, no mesmo dia.** Publiquei primeiro que o `UpdateApplier` nao criava
`bin.previous`. **Errado** — cria, e o log da maquina prova. O defeito verdadeiro e mais estreito:
o script grava em `C:\Program Files\bin.previous` e o `RollbackOrchestrator` le em
`C:\Program Files\ManagerAgent\bin.previous`. **Um nivel de diretorio de diferenca**, porque a
constante em C# assume um layout (`.../ManagerAgent/bin`) que o instalador nunca produziu.

E nao e um defeito, sao tres: le no nivel errado; restauraria dentro de `bin\`, de onde nada roda;
e nao conseguiria mover a instalacao de qualquer forma, porque o `ManagerAgent.Watchdog.exe` que
executa o rollback mora dentro dela. O PS1 de update ja resolve esse mesmo obstaculo parando o
Watchdog antes de copiar — e o padrao a reusar.

Eu tinha lido so o lado C# e concluido cedo demais. Terceira vez que esta base morde por isso.

Estado da linha de corte apos hoje: **item 1** (@Shuri), **3a** e **3b** (@Bucky), **item 4**
(@Vision).

---

## D10 — 2026-08-24: tudo refinado e aprovado; desenvolvimento liberado

**Pedido do Marcos:** refinar tudo, aprovar, e so entao desenvolver — "garanta que tudo esta
funcionando e nao quebraremos nada".

**@Bucky trouxe o desenho que faltava** (`registro/2026-08-24-desenho-bucky-rollback-assincrono.md`)
e **@Tony aprovou** (`registro/2026-08-24-review-tony-desenho-rollback.md`).

**Desenho aprovado:** o script de rollback escreve um arquivo proprio de resultado, so com fatos;
o Watchdog volta (o script ja o religa, provado no update de 21/08), le, decide e apaga. A decisao
continua em C#; o script nao calcula nada.

**Buraco fechado antes de existir:** se o script nunca rodar, nao ha resultado e o Watchdog ficaria
contando falhas em silencio. Fecha com `rollbackDispatchedAtUtc` gravado antes de disparar +
folga de **15 minutos** (ajustada de 10 por mim: o Service leva 2min19s so para subir no boot, o
ciclo do Watchdog e 60s e o grace e 3min — errar para menos custa um SOS falso num cliente que
estava bem).

**Dois achados novos do @Bucky, os dois confirmados por mim:**

1. **O recovery do SCM esta ligado nos dois servicos**, e o update so desliga o do Service
   principal enquanto aplica `Stop-Process -Force` no Watchdog. E risco latente **no caminho de
   update de hoje**, nao so no rollback. Nunca mordeu por sorte. Entra no escopo do 3a, com
   restauracao obrigatoria no `finally` e com os valores medidos na maquina.
2. **O comentario do PS1 que diz que o Watchdog limpa o `bin.previous.building` e falso** — o
   cleaner so olha `C:\ProgramData`. Nono item da varredura de textos defasados.

**Um item que eu acrescentei ao escopo, e assumo:** `WatchdogStateStore.Save` usa
`File.WriteAllText`, sem temp+rename, com dois processos escrevendo. Queda de energia no meio
trunca o JSON e o boot seguinte cai no default — **`sosMode` volta a `false` e a maquina perde o
freio de auto-update justamente quando ja estava em apuros**. Criterio B da linha de corte. Como o
@Bucky ja vai abrir esse arquivo, entra junto, limitado ao `Save`.

**Correcao de premissa no desenho:** ele escreveu que o `watchdog-state.json` tem um dono so. Tem
dois processos escrevendo, com um serializador so. A conclusao dele nao muda — fica mais forte.

**Estado:** arvore em **1010 testes verdes**, conferida hoje antes de comecar. Item 1 (@Shuri) e
3a (@Bucky) liberados. 3b depende do 3a. Item 4 com o @Vision. **Nada commitado.**

---

## D11 — 2026-08-24: recall de frota refinado; R-01 e R-02 decididos

**Origem:** pergunta do Marcos — se da para disparar algo que volte a versao em todas as maquinas.

**Resposta: hoje nao da.** O kill switch (`versoes_agente.pausada`) so impede a entrega a quem
ainda nao pegou; quem ja atualizou fica. Republicar a versao antiga tambem nao resolve — o backend
so oferece candidata que passe em `isVersaoMaior`, e o Agent tem trava equivalente (A-46). As duas
estao certas: evitam laco de reinstalacao, e sao justamente o que impede um downgrade.

Refinamento completo em `registro/2026-08-24-refinamento-recall-de-frota.md`.

### R-01 — o sticky nao e lido por ninguem. **Entra como parte do 3a.**

`StickyVersion`/`StickyUntil` sao gravados apos um rollback e lidos so pelo proprio Watchdog, para
nao repetir rollback. O `UpdateCheckerWorker` nao os consulta.

Efeito: rollback devolve a versao boa, e ate 6h depois a maquina reinstala a versao quebrada —
porque a pausa da versao e **manual** e ninguem apertou o botao de madrugada. Quebra de novo, e ai
o sticky **impede** o segundo rollback. **A maquina fica quebrada ate a sticky vencer, 24h depois.**

**Decisao do Marcos:** nao e item novo na lista — e o 3a incompleto. Sem isso, o rollback
consertado hoje se desfaz sozinho.

Tarefa escrita para execucao em sessao separada:
`registro/2026-08-24-tarefa-bucky-R01-sticky.md`.

**Armadilha registrada ali, e e o que faria a correcao nao funcionar em silencio:** os dois lados
guardam a versao em formatos diferentes — `UltimaVersaoQuebrada` vem de `ProductVersion`
(`1.5.12+b1d227ac93...`, com hash do git) e a versao oferecida vem do backend como `1.5.12`.
Comparacao por string nunca casaria.

### R-02 — modo SOS e porta de mao unica. **FORA da linha de corte.**

O header `X-Manager-Sos-Recovery-Available`, que tiraria a maquina do SOS, **so existe em
comentario** — varri os tres repos, ninguem envia e ninguem le. Maquina que entra em SOS nunca
mais se atualiza sozinha.

**Eu tinha proposto como bloqueador e estava errado.** O Marcos perguntou se o rollback funciona
junto com o SOS, e a pergunta expos o erro: **os dois sao excludentes.** SOS so e ligado quando o
rollback **falhou**, e para chegar la as duas camadas de deteccao precisam acusar falha — ou seja,
o Service ja esta fora do ar. **Toda maquina em SOS ja parou de capturar e ja precisa de visita.**
O SOS nao piora nada.

Eu tinha somado o R-02 com o laco do R-01. Corrigido o R-01, o caminho para o SOS deixa de ser um
laco e passa a ser so maquina genuinamente quebrada.

Sobra um residuo pequeno: maquina consertada instalando por cima mantem o SOS e nunca mais se
atualiza, em silencio. O desinstalador ja apaga `ProgramData` inteiro, entao desinstalar e
instalar resolve. Anotar no procedimento de suporte. Proxima wave.

### R-03 — recall de frota. Nao segura producao.

Depende do R-01. Mecanismo aprovado: reusar o `bin.previous` e o `run-rollback.ps1` que ja existem
em cada maquina — uma versao para tras, sem download. Coluna nova `revogada` em `versoes_agente`,
separada de `pausada`, e a ordem chega pela resposta do `verificar`, sem canal novo.

**Latencia de ate 6 horas: aceita pelo Marcos.** Palavra dele: *"o importante e voltar a funcionar
sem precisar de atuacao manual de alguem"*.

**Limite que precisa estar visivel para quem dispara:** o recall so funciona em maquina que
entende o comando. Backend primeiro, Agent depois. **Ele protege as versoes lancadas depois de
existir, nunca a que estiver rodando quando for publicado.**

**Estado da linha de corte:** item 1 feito, 3a feito **menos o R-01**, 3b com o @Vision, item 4
com o @Vision.

---

## D12 — 2026-08-24: tarefas escritas para execucao em sessoes separadas

O Marcos vai passar cada uma para uma sessao propria. Os dois documentos sao **autossuficientes**:
carregam contexto, estado do repo, o defeito com linha de codigo, o que nao tocar, testes, aceite
e o que escrever no fim.

| Tarefa | Dono | Documento |
|---|---|---|
| **R-01** — sticky respeitado pelo `UpdateCheckerWorker` | @Bucky | `registro/2026-08-24-tarefa-bucky-R01-sticky.md` |
| **R-03** — recall de frota, lado backend | @Shuri | `registro/2026-08-24-tarefa-shuri-R03-recall.md` |

**Armadilha registrada nas duas, e e a mesma:** os dois lados guardam a versao em formatos
diferentes. `ProductVersion` traz `1.5.12+b1d227ac93...` com o hash do git colado; o backend manda
`1.5.12`. Comparacao por string nunca casaria, e a guarda ficaria no codigo sem nunca disparar.
Medido na maquina em 24/08. Sem esse aviso, as duas entregas passariam nos testes e nao
funcionariam.

**Regra dura que entrou no R-03:** revogar uma versao tem de **pausar junto, na mesma transacao**.
Revogar sem pausar faz a maquina voltar e reinstalar a revogada no ciclo seguinte.

**Limitacao que precisa aparecer para quem apertar o botao:** o recall so funciona em maquina cujo
Agent entende o comando. Backend primeiro, Agent depois. **Ele protege as versoes lancadas depois
de existir, nunca a que estiver rodando quando for publicado.**

**Cenarios de teste levantados** — `agent-desktop/TESTES-ROLLBACK-E-RECALL.md`, cobrindo rollback,
sticky e recall: 41 casos automatizados ja existentes, 16 a escrever, 6 cenarios E2E (**nenhum
jamais rodou**) e os testes de maquina 10.5 a 10.11 do plano regressivo.

---

## D13 — 2026-08-24: recall do app DESCARTADO; recall e so Windows

**Pedido inicial do Marcos:** que a mesma API servisse ao app, e um documento para o @Sam
desenvolver. **Investiguei o repo do Android e o proprio Marcos concluiu o contrario. Descartado.**

### O que a investigacao mostrou

No `manager-srv-agent-android`, a instalacao do APK e por `ACTION_VIEW`
(`ApkDownloadWorker.kt:102`), com o colaborador confirmando — como tem de ser em BYOD sem MDM. Por
esse caminho o Android **recusa APK com `versionCode` menor**. Desinstalar antes apagaria o buffer
de eventos, o vinculo e as permissoes, e o app se auto-remove da gaveta apos a configuracao. E nao
ha copia do APK anterior no aparelho.

**Ou seja: no app, "voltar" e publicar um build com `versionCode` maior contendo o comportamento
antigo.**

### A conclusao do Marcos, e ela e mais forte do que parece

Palavra dele: *"no app teria que ser uma atualizacao com a versao anterior no caso ne? nunca
voltar versao e sim lancar uma mais alta com o apk antigo... entao acho que nem vale mexer agora...
isso eu ja consigo fazer na mao de certa forma."*

**Esta certo.** Publicar a versao com o APK antigo **e uma atualizacao normal** — o
`UpdateCheckWorker` do app ja pergunta de 6 em 6 horas, ja baixa, ja valida SHA-256 e ja notifica.
Nao ha mecanismo novo a construir; haveria so um texto de notificacao diferente e um rastro de
auditoria. E como o colaborador precisa tocar de qualquer forma, nada se perde fazendo pelo
processo de release de sempre.

### O que isso desfez

Eu tinha redesenhado a API para servir aos dois: a resposta diria *"saia da X, va para a Y"*, com
uma coluna `versao_alvo`.

**Caiu junto.** Sem o Android na conta, o alvo explicito nao servia a ninguem — o Windows so
precisa de "volte", porque a copia anterior ja esta no disco dele. A tarefa da @Shuri voltou ao
desenho simples: **so a coluna `revogada`**, e a validacao passou a **recusar revogar versao de SO
Android** (400), deixando o escopo explicito no proprio codigo.

**Licao para mim:** generalizei a API para um caso que, ao ser olhado de perto, nao precisava dela.
O Marcos viu antes de eu escrever codigo.

### Documentos

| | |
|---|---|
| `registro/2026-08-24-tarefa-shuri-R03-recall.md` | tarefa, **so Windows**, desenho simples |
| `registro/2026-08-24-android-recall-descartado.md` | **NAO EXECUTAR.** Guarda a investigacao de por que o Android nao permite voltar de versao, para nao ser redescoberta |

---

## D14 — 2026-08-24: R-03 backend aprovado; lado Agent escrito com as tres guardas

**@Shuri entregou o R-03 backend** (commit `257096f` em `manager-srv-admin-node`, nao empurrado).
**Aprovado** — review em `registro/2026-08-24-review-tony-R03-recall.md`.

O Marcos conferiu pessoalmente os tres pontos duros (atomicidade, as tres grafias de versao, e a
ordem da checagem). Eu cobri o resto: refiz uma das mutacoes dela — removi `pausada: true` do
`set()` e tres testes quebraram, exatamente os relatados. Nenhum teste existente reescrito (2279
insercoes, 2 delecoes, ambas comentario). Nenhum `catch` engolindo falha. Typecheck limpo. Suite
estavel em tres execucoes.

**Coluna extra `revogada_motivo` aprovada pelo Marcos.** O furo era meu: mandei devolver o motivo a
maquina sem dizer onde guarda-lo. Ela viu e perguntou antes de o @Bucky implementar contra o
contrato.

**Ela me corrigiu de novo:** eu escrevi na tarefa que o teste de swagger falhava por ambiente
naquele repo. Ele esta **pulado**, nao falhando — carreguei do `manager-srv-events-node` sem
conferir. Terceira vez que descrevo um repo pelo outro.

### Achado meu no review — virou escopo do lado Agent

**O rollback automatico so dispara em maquina ja fora do ar. O recall dispara em maquina
saudavel.** As mesmas acoes tem consequencias diferentes, e o codigo de hoje trataria mal tres
casos:

| Guarda | O caso | O que aconteceria hoje |
|---|---|---|
| **A** | a versao do backup e a mesma da instalada (duas versoes ruins seguidas) | a cada 6h para os dois servicos, copia ~160MB, sobe, **e nao muda de versao**. Para sempre, com um `bin.failed-*` de ~160MB por volta |
| **B** | nao ha backup (primeira instalacao) | liga **modo SOS** numa maquina que esta capturando normalmente, tirando-a do auto-update sem necessidade |
| **C** | a maquina continua na versao revogada | recebe a ordem a cada ciclo, sem freio |

O `pausada=true` que o backend marca junto impede **reinstalar** a revogada. **Nao cobre nenhuma
das tres** — as tres sao do lado da maquina.

**Tarefa escrita:** `registro/2026-08-24-tarefa-bucky-R03-recall-agent.md`, com o contrato copiado
do handoff da @Shuri, as tres guardas e a restricao de arquitetura (quem le a resposta e o Service,
quem executa o rollback e o Watchdog — **processos diferentes**; o desenho vem antes do codigo).

**Ordem para o @Bucky: R-01 primeiro, recall depois.** Os dois mexem no mesmo metodo.

**Pendente:** rodar o parity runner de verdade — exige o lado Java de pe.

**Linha de corte: inalterada.** O recall e melhoria, nao bloqueador.
