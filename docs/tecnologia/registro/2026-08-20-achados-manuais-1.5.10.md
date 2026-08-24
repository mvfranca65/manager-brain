# Achados dos testes manuais — 1.5.10 (2026-08-20)

> Executor: Marcos | Conferencia: @Tony | Binario: 1.5.10.0 compilado 20/08 10:49 (com ADR-001 e A-52)

## M1 — Bloqueio e desbloqueio: PASSOU

| Verificacao | Resultado |
|---|---|
| Quantidade de LOCK / UNLOCK | **1 e 1** (antes do A-33: 16 e 26 falsos) |
| Janela ativa fechada no bloqueio | sim, no instante exato do LOCK |
| LOGIN falso | nenhum |
| Ociosidade duplicada | nenhuma |

O aviso "Gap de wall-clock detectado" e comportamento correto: com a tela bloqueada o monitor
de ociosidade fica suspenso, e o agent avalia o intervalo cego. Os 45,8s foram **descartados**
porque o minimo para virar pausa e 60s. Confirmado no codigo (`MinIdleDurationSeconds = 60`).

## M5 — Suspender e acordar: PASSOU no essencial, com um achado

O objetivo principal foi cumprido: **os ~150s dormindo viraram pausa registrada, nao hora
trabalhada.** Confirmado na base — `eventos_ociosidade` com 150s cobrindo o periodo.

LOCK e UNLOCK corretos, um de cada. PowerMode Suspend e Resume detectados.

---

# A-56 (novo) — Suspensao gera falso "sessao interrompida" E deixa worker orfao

**Severidade: ALTA.** Nao perde dado, mas duplica processo e polui relatorio.

## O que acontece

```
22:44:40  suspensao. O worker para de pingar (a maquina dorme)
22:47:03  WorkerWatchdog: "dead worker detected. PID=20420, last ping was 174s ago"
22:47:04  o worker 20420 ACORDA e reconecta - ele nunca esteve morto
22:47:11  o Watchdog lanca o PID 23836 assim mesmo
22:47:12  23836 conecta e e registrado
22:47:13  20420 tenta reconectar -> "CONNECT rejected: PID mismatch"
```

Resultado: **dois SessionWorker vivos na mesma sessao.** O guard do Service funciona e so
aceita um no pipe — mas ninguem mata o outro.

Medido 6h depois: o orfao seguia vivo com **136MB e 410s de CPU acumulados**, e mostrava um
**segundo icone na bandeja** (foi assim que o Marcos notou). Encerrado manualmente.

E o worker novo, ao subir, nao encontra a flag de shutdown limpo e emite
**SESSAO_INTERROMPIDA (motivo NoCleanShutdownFlag)** — evento que significa crash ou queda de
energia. Suspender um notebook nao e nenhum dos dois.

## Por que acontece

`WorkerWatchdog.HeartbeatTimeout` e 90s fixos e **o watchdog nao sabe nada sobre suspensao** —
`grep` por PowerMode/Suspend/Resume no arquivo nao acha nada. Enquanto a maquina dorme, o
relogio corre e o ping nao chega; passados 90s, o worker vivo e declarado morto.

O `SessionWorker` tem `PowerModeMonitor` e trata Suspend/Resume corretamente. Quem nao trata
e o lado do Service.

## Impacto

- **Relatorio:** quem fecha o notebook varias vezes ao dia gera varios "sessao interrompida"
  falsos. O evento existe para sinalizar instabilidade real; assim ele perde o significado.
  Na base ha 86 ocorrencias em 3 dias so nesta maquina (parte vem dos testes, que matam o
  worker de proposito - mas o caminho da suspensao esta agora comprovado).
- **Recurso:** cada suspensao pode deixar um worker orfao de ~136MB. Varias suspensoes num
  dia acumulam. O teste R7 mede memoria POR PROCESSO e passaria de qualquer forma.
- **Captura:** ha uma janela sem captura entre a morte declarada e o worker novo.

## Relacao com achados anteriores

E o **A-21 reaberto por um caminho novo**. O @Bucky previu exatamente isto no review da
ADR-001: "a reconciliacao lanca um worker novo para uma sessao onde o worker antigo ainda
esta vivo". Ele falava do caminho da revogacao; este e o caminho da suspensao.

## Sugestao de correcao (decisao do @Bucky)

Duas frentes, e a primeira sozinha ja resolve o sintoma pior:

1. **Matar antes de relancar.** Se o watchdog decidiu que o worker morreu, o processo tem de
   ser encerrado antes do relancamento. Hoje ele so remove do registro.
2. **Ensinar o watchdog sobre suspensao.** O Service pode ouvir `SystemEvents.PowerModeChanged`
   e reiniciar a contagem do heartbeat no Resume, em vez de contar o tempo dormido. Isso
   evita o falso positivo na origem e, de quebra, o falso SESSAO_INTERROMPIDA.

## M9 — Icone e menu da bandeja: PASSOU (com o achado acima)

Menu correto: versao (iManager - v1.5.10), Ferramentas (Verificacao de Saude, Logs em Tempo
Real, Exportar Diagnostico, Limpar Dados e Reiniciar) e Sobre.

O segundo icone que apareceu **nao era do menu** — era o worker orfao do A-56.

---

## PONTO EM ABERTO — decisao do Marcos (2026-08-20)

> "Eliminar e sempre ter apenas um agent rodando."

Nao basta corrigir o caminho da suspensao. O Marcos quer a **garantia estrutural** de que
existe no maximo um SessionWorker por sessao, qualquer que seja o caminho.

Hoje a garantia e parcial: o Service **rejeita** a conexao do worker excedente
("CONNECT rejected: PID mismatch"), mas nao **encerra** o processo. Um worker rejeitado
continua vivo, consumindo memoria e mostrando icone, ate alguem matar na mao.

Caminhos conhecidos que produzem worker excedente:
- suspensao (A-56, comprovado nesta sessao)
- revogacao seguida de re-vinculacao (previsto pelo @Bucky no review da ADR-001)
- o proprio teste de reconciliacao da suite (A-32, corrigido hoje com RequiresIsolation)

A analisar quando entrar no roadmap: o Service deve encerrar o processo que rejeitar, e a
rejeicao precisa ser um caminho ativo, nao so uma recusa passiva. Envolve o WorkerWatchdog,
o PipeMessageHandler e o WorkerRegistry — dominio do @Bucky.

---

## M4 — Reuniao: o agent PASSOU, o backend nao

### O que o agent fez, e esta certo

| Verificacao | Resultado |
|---|---|
| Deteccao | Google Meet reconhecido; confirmada apos 1min de espera (evita aba aberta por engano) |
| Inicio | retroativo ao instante real da janela (22:55:26), nao ao da confirmacao |
| Encerramento | 22:02:30, apos 2min sem sinal. Duracao 7 min, correta |
| Snapshots | 6 registros, um por minuto, com o mesmo `reuniao_instalacao_id` |
| LGPD | so estado de dispositivo (`camera_ativa`, `microfone_ativo`, `render_ativo`). Nenhum conteudo |

Os dois ultimos snapshots saem com `render_ativo=false` e `render_confianca=AUSENTE`, depois
que a janela foi fechada — coerente com a espera de 2min antes de encerrar.

---

# A-57 (novo) — uma reuniao vira DUAS linhas em `eventos_reuniao`

**Severidade: ALTA** pelo efeito em relatorio. **Dono: @Shuri** (backend Node).

## Medido

```
id=3156  iniciado_em=01:55:26.171Z  finalizado_em=NULL              <- aberta para sempre
id=3157  iniciado_em=01:55:26.171Z  finalizado_em=02:02:30.508Z     <- a mesma reuniao
```

Mesmo `iniciado_em`, mesmo `titulo_reuniao`, mesmo agente. O evento de fim **criou linha
nova** em vez de atualizar a linha aberta.

## Por que isso ja deveria estar resolvido

E o mesmo defeito que o @Shuri ja corrigiu para as outras duas tabelas — os commits estao no
`manager-srv-events-node`:

- `fix: reconcilia snapshot em andamento de eventos_janela (upsert em vez de INSERT duplicado)`
- `fix: reconciliar snapshot de ausencia aberta em eventos_ociosidade`

**`eventos_reuniao` ficou de fora dessa correcao.** O agent manda certo: inicio e fim com o
mesmo `reuniaoInstalacaoId`. Quem nao reconcilia e o ingest.

## Impacto

- Toda reuniao deixa uma linha aberta permanente. Engorda o mesmo passivo que ja tem 33
  ociosidades e 27 janelas abertas — e que ja esta no handoff do @Shuri.
- Qualquer soma de duracao conta a reuniao duas vezes, ou trata a linha aberta como reuniao
  que nunca terminou.
- O pilar de Foco e a leitura de tempo em reuniao ficam inflados.

## O que verificar junto

O `eventos_reuniao_snapshot` tem `reuniao_instalacao_id`, mas a tabela `eventos_reuniao`
aparentemente nao — vale conferir por qual chave o ingest tenta reconciliar. Se a coluna de
correlacao nao existir na tabela principal, a reconciliacao por upsert precisa dela.

---

## M6 — Sem internet: PASSOU, exemplar

Offline das 23:07:42 as 23:11:34. Ao voltar, subiu **29 eventos de uma vez**.

| Verificacao | Resultado |
|---|---|
| Continuou capturando offline | sim — 12 janelas registradas no periodo |
| Duplicatas apos reconexao | **nenhuma** |
| Janelas abertas indevidamente | nenhuma (so a em uso) |
| Horario dos eventos | o do acontecimento, nao o da entrega |

O ponto que o Marcos levantou merece registro, porque a intuicao era razoavel e o dado
mostrou o contrario. Ele fez um bloqueio/desbloqueio durante o offline e achou que tinha se
perdido, "porque deveria enviar direto e nao tinha rede". Metade certa:

```
23:10:06  LOCK    chegou no banco 149s depois
23:11:34  UNLOCK  chegou no banco  61s depois
```

LOCK e UNLOCK de fato disparam envio imediato — e ele falhou, sem rede. Mas o evento foi para
o buffer local e subiu na reconexao, com `ocorreu_em` preservado. Ficar sem internet atrasa a
entrega; nao distorce o relatorio.

## M11 — Ciclo de vida do Service: PASSOU

| Hora | Evento |
|---|---|
| 23:17:51 | parada graciosa, com **tentativa final de upload**: 10 eventos subiram antes de morrer |
| 23:18-23:22 | Watchdog detecta SCM=Stopped mas **nao aciona** — exige duas camadas |
| 23:23:07 | heartbeat atinge 405s (limite 360s): as duas camadas concordam |
| 23:23:12 | Service Running, worker de volta |

**Janela sem monitoramento: 5min21s.** E o preco deliberado da regra de duas camadas, que
evita recovery por falso positivo. Vale ter em mente: quem parar o Service tem ~6min ate o
agent voltar. Como parar exige elevacao, quem consegue fazer isso ja tem controle da maquina.

---

# A-58 (novo) — ninguem gera ATIVO/PAUSA/AUSENTE desde maio

**Severidade: ALTA — e funcionalidade de produto, visivel para o gestor.**

## O que foi medido

`eventos_transicao_status`, base de staging:

```
agente 29: 244 transicoes, ultima em 2026-05-15
agente 30: 204 transicoes, ultima em 2026-05-15
agente 31: 200 transicoes, ultima em 2026-05-17
agente 32: 180 transicoes, ultima em 2026-05-15
agente 36: 180 transicoes, ultima em 2026-05-15
```

**Nada depois de 17/05.** O agente 51 (a maquina de teste, ativa hoje) nao tem **nenhuma**.
E os horarios de maio sao todos redondos (19:41:00, 19:26:00), com cara de carga de teste, nao
de captura real.

## A causa: cada lado acha que o outro faz

- **Agent Windows:** parou de emitir na v1.5.0. Esta escrito no proprio codigo, em
  `DailyBoundaryWorker.cs:11` — *"v1.5.0: Agent nao emite mais StatusTransitionEvent (backend
  deriva status)"*. `grep` por StatusTransition no `src/` do agent so acha esse comentario.
- **Backend Node:** tem `StatusTransitionHandler`, que **persiste** o evento recebido do
  agent. Ele nao deriva nada — so grava o que chega.

Resultado: o Windows parou de mandar contando que o backend derivaria; o backend so sabe
gravar o que recebe. **Ninguem calcula.**

## O Android faz diferente

O Agent Android tem `StatusTransitionCalculator.kt` e **emite** o evento (thresholds 5min
pausa / 15min ausente). Ou seja, hoje o status funciona no Android e nao funciona no Windows —
com o mesmo backend.

## Impacto

O gestor ve ATIVO/PAUSA/AUSENTE no portal. Para maquina Windows esse dado nao existe desde
maio. Ou aparece vazio, ou aparece um valor derivado de outra fonte que precisa ser conferida.

## Precisa de decisao (nao e correcao obvia)

Duas saidas, e a escolha muda o dono:

1. **O backend deriva de verdade**, a partir de `eventos_ociosidade` e `eventos_janela` que
   ja chegam. Vantagem: uma regra so para Windows, Android e o que vier. Dono: @Shuri.
2. **O Windows volta a emitir**, como o Android faz. Vantagem: caminho ja existente e
   testado do outro lado. Desvantagem: duas implementacoes da mesma regra, que vao divergir.

Recomendo a **1**, pelo mesmo motivo que a IA nao calcula pilar: regra de negocio em um lugar
so. Mas e decisao de arquitetura com efeito em produto — @Steve precisa saber que o dado
esta faltando desde maio.

---

## M10 — Dois usuarios: separacao de sessao PASSOU, atribuicao NAO

### O que funciona

Um SessionWorker por sessao Windows, com log proprio (`session-worker-S2`). A troca de
usuario produziu a sequencia certa:

```
23:28:03  [NoisyTech]  LOCK     <- ao trocar
23:28:11  [Marcos   ]  LOGIN
23:29:46  [Marcos   ]  LOCK     <- ao voltar
23:29:57  [NoisyTech]  UNLOCK
```

E cada evento carrega o `windows_username` correto.

---

# A-59 (novo) — atividade de qualquer usuario da maquina vai para o mesmo colaborador

**Severidade: ALTA. Correcao de dado e LGPD.**

## O que foi medido

O usuario Windows "Marcos" abriu o Chrome e pesquisou no YouTube. Na base:

```
windows_username = Marcos
usuario_id       = I888777      <- o colaborador vinculado ao agent (do NoisyTech)
usuario_ref_id   = 3336
```

Tudo que o segundo usuario fez foi atribuido ao colaborador do primeiro.

## Por que acontece

O vinculo e **por maquina**, nao por usuario: `config.json` tem um unico
`identificadorColaborador`, definido na instalacao. O agent registra corretamente QUEM fez
(o `windows_username` vai em todo evento), mas na hora de dizer DE QUEM E o tempo, usa sempre
o mesmo colaborador.

## Impacto

Em qualquer maquina compartilhada — recepcao, chao de fabrica, turnos, quiosque — a atividade
de todos cai no relatorio de uma pessoa. Ela aparece trabalhando muito alem do que trabalhou,
e os demais aparecem sem dado nenhum. Os pilares e o Score IA dessa pessoa ficam sem sentido.

Do lado da LGPD: registra-se atividade de uma pessoa sob a identidade de outra.

## O que ja existe a favor

O dado para separar **ja esta no banco**: `windows_username` chega preenchido e correto em
todo evento. Falta o vinculo ser por usuario.

## Precisa de decisao do @Steve (nao e correcao tecnica obvia)

Tres caminhos, com efeitos bem diferentes:

1. **Vinculo por usuario Windows.** O primeiro login de cada usuario dispara uma vinculacao
   propria. Resolve de verdade, mas muda o modelo de vinculacao e o fluxo de instalacao.
2. **Uma maquina, um colaborador — assumido e documentado.** Mantem como esta e o produto
   declara que nao suporta maquina compartilhada. Barato, mas fecha um segmento inteiro de
   cliente (o ICP fala em atendimento e back-office, onde compartilhar maquina e comum).
3. **Descartar o que nao for do usuario vinculado.** O agent so envia eventos cujo
   `windows_username` bata com o do vinculo. Nao resolve o segundo usuario, mas para de
   sujar o relatorio do primeiro — e e o menor esforco com o maior ganho imediato.

Recomendo levar a **3** como contencao e a **1** como roadmap. Nao ha correcao segura que
nao passe por decisao de produto.

### DECISAO do Marcos (2026-08-20)

> "Hoje o vinculo sera por maquina mesmo, nada de usuario. Talvez uma evolucao pro futuro,
> mas hoje seguiremos assim."

**Caminho 2 escolhido.** Vinculo por maquina, uma maquina = um colaborador. O caminho 1
(vinculo por usuario) fica como evolucao futura, sem data.

Consequencia assumida: em maquina compartilhada, a atividade de qualquer usuario Windows
conta para o colaborador vinculado.

**Continua em aberto**, e e decisao diferente da que foi tomada: o que fazer com a atividade
de OUTROS usuarios na mesma maquina. Duas leituras, ambas compativeis com "vinculo por
maquina":

- **Como esta hoje:** tudo na maquina conta para o colaborador vinculado, seja quem for que
  esteja logado. Coerente com a premissa "esta maquina e desta pessoa".
- **Filtrar por `windows_username`:** o agent so envia o que for do usuario vinculado. Protege
  o relatorio de quem divide a maquina eventualmente — o tecnico que loga para dar suporte,
  o gestor que usa a estacao por 10 minutos.

O dado para filtrar ja existe: `windows_username` chega correto em todo evento. Nao urgente.

---

## M3 — Reboot: PASSOU no essencial, dois achados no caminho

| Verificacao | Resultado |
|---|---|
| Agent sobe sozinho apos o boot | sim, as 23:39:15 |
| Quantidade de LOGIN | **um so** — o dedup do A-34 funcionou |
| SESSAO_INTERROMPIDA | 1, e aqui e **legitima** (reboot nao tem shutdown limpo do worker) |
| Eventos de janela abertos de antes do reboot | nenhum |

**Demora para subir: 2min35s.** E por configuracao, nao defeito: o Service esta como
`DelayedAutoStart=True`, entao o Windows segura o inicio para nao concorrer com o boot. O
Watchdog sobe imediato (sem delay). Vale saber que existe essa janela de ~2,5min sem captura
depois de todo reboot.

O shutdown foi correto: tentou o upload final, estourou o teto de 5s e **deixou os 10 eventos
no buffer para o proximo start**, em vez de travar o desligamento da maquina.

---

# A-60 (novo) — o segundo usuario da maquina nao consegue gravar no buffer local

**Severidade: ALTA. Perda de evento.**

## O que foi medido

O worker da sessao 2 (usuario "Marcos") registrou **7 erros** entre 23:29:57 e 23:36:00:

```
[ERR] [SessionWorker] [S2] Error in capture loop iteration
Microsoft.Data.Sqlite.SqliteException: SQLite Error 8: 'attempt to write a readonly database'
```

A causa esta nas permissoes:

```
autonomous-buffer.db      dono: DESKTOP-VMSM6LE\NoisyTech
                          BUILTIN\Usuarios -> ReadAndExecute, Synchronize   (sem escrita)
```

A **pasta** `data\` concede Write ao grupo Usuarios, mas os **arquivos** criados pelo
primeiro usuario ficam com apenas ReadAndExecute para os demais. O SQLite precisa modificar o
arquivo existente e criar `-wal`/`-shm`; sem isso, abre em somente-leitura e falha.

## Impacto

O buffer autonomo e a rede de seguranca do worker: e nele que os eventos ficam quando o pipe
com o Service esta indisponivel. Para o segundo usuario essa rede **nao existe** — o loop de
captura quebra com [ERR] em vez de guardar.

Enquanto o pipe estiver de pe, os eventos passam direto para o Service (que roda como SYSTEM
e grava sem problema). O prejuizo aparece exatamente quando mais importa: pipe caido, Service
reiniciando, update em andamento.

## Detalhe que aponta a correcao

`session-state-1.json` e `session-state-2.json` sao **por sessao** — cada usuario tem o seu, e
por isso funcionam. O `autonomous-buffer.db` e **unico e compartilhado**.

Duas saidas, ambas do @Bucky:
1. **Buffer por sessao** (`autonomous-buffer-S1.db`, `-S2.db`), no mesmo padrao que o
   `session-state` ja usa. Resolve a disputa e a permissao de uma vez.
2. **ACL explicita** nos arquivos de dados no momento da criacao, concedendo Modify aos
   usuarios interativos. Ja ha precedente no codigo: o `HeartbeatWriter` aplica ACL propria
   ("Heartbeat ACL aplicada. Sids=[SYSTEM]").

Prefiro a 1: nao depende de acertar ACL, e dois workers escrevendo no mesmo SQLite e disputa
que nao precisa existir.

---

# A-61 (menor) — o Watchdog relanca workers durante o desligamento da maquina

**Severidade: BAIXA.** Ruido, sem perda de dado.

Na sequencia do reboot:

```
23:36:16  worker S1 morre (EOF)  -> Watchdog relanca PID 21664
23:36:21  SessionLogoff S1       -> morre de novo
23:36:21  worker S2 morre (EOF)  -> Watchdog relanca PID 21320
23:36:23  SessionLogoff S2       -> morre de novo
```

Dois processos nascem e morrem em segundos, enquanto a maquina esta desligando. O Watchdog
nao sabe que ha um shutdown em curso e trata o EOF do pipe como morte a recuperar.

Sem consequencia pratica, mas e o mesmo padrao do A-56: **o Watchdog reage a sinais que nao
sao falha real** — suspensao la, desligamento aqui. Se o A-56 for corrigido ensinando o
Watchdog a reconhecer estados do sistema, este sai junto.

---

## M2 — Logoff e login: passou em duas verificacoes, REPROVOU na principal

| Verificacao | Resultado |
|---|---|
| Janela aberta fecha no logoff (A-35) | **PASSOU** — "Evento de janela fechado por morte do worker" |
| LOGIN apos o login | **PASSOU** — um so, dedup do A-34 funcionando |
| "LOGIN suprimido" indevido | nao apareceu |
| **LOGOUT** | **NENHUM** |
| LOGOUT subindo rapido (A-36) | nao verificavel: nao houve LOGOUT |

Detalhe util: o Windows trocou o SessionId do NoisyTech de 1 para 2 (a sessao 2 era do
Marcos). O agent comparou o LogonId anterior (Marcos) com o novo (NoisyTech), viu a mudanca
e emitiu LOGIN. Esse caminho esta correto.

---

# A-62 (novo) — logoff nunca gera LOGOUT: o Windows mata o worker antes do aviso chegar

**Severidade: ALTA.** Nao ha registro de fim de expediente.

## O desenho e correto; a ordem dos fatos, nao

O codigo preve tudo. `ManagerAgentService.HandleLogoff` manda `ShutdownMessage` com
`Reason = "SESSION_LOGOFF"`, e `Worker.HandleShutdownAsync` faz o certo em resposta: emite
LOGOUT explicito no SessionMonitor, drena para o buffer e marca a flag de shutdown limpo.

Medido no logoff real:

```
23:44:11.983  PipeServer: worker disconnected (EOF). Session=1     <- o Windows ja matou
23:44:12.005  Evento de janela fechado por morte do worker          <- A-35 funcionou
23:44:12.015  WorkerWatchdog: attempting re-launch for Session=1    <- relanca no meio do logoff
23:44:12.964  WTS session change: Reason="SessionLogoff"            <- o Service SO AGORA sabe
23:44:12.975  Handling logoff for session 1                          -> manda ShutdownMessage
```

**O worker morreu 1 segundo antes de o Service saber do logoff.** A `ShutdownMessage` e
enviada para um processo que nao existe mais.

## Consequencias

1. **Nunca ha LOGOUT.** Nao existe registro de fim de expediente. Para o relatorio, a jornada
   da pessoa nao tem fechamento — so o LOGIN do dia seguinte.
2. **Todo logoff vira SESSAO_INTERROMPIDA.** Como o `HandleShutdownAsync` nao roda, a flag de
   shutdown limpo nao e marcada, e o proximo login emite SESSAO_INTERROMPIDA com motivo
   `NoCleanShutdownFlag` — evento que significa crash ou queda de energia. Logoff normal e
   fim de expediente, nao anomalia.
3. O A-36 (upload imediato de fronteira de sessao) fica sem efeito no logoff: nao ha o que
   subir.

## Relacao com os outros achados

E o terceiro caso do mesmo padrao hoje:

| Achado | Gatilho | Resultado |
|---|---|---|
| A-56 | suspensao | falso SESSAO_INTERROMPIDA + worker orfao |
| A-61 | desligamento | Watchdog relanca workers durante o shutdown |
| A-62 | logoff | sem LOGOUT + falso SESSAO_INTERROMPIDA |

Nos tres, o agent trata um evento normal do sistema operacional como falha. O
SESSAO_INTERROMPIDA, que deveria ser sinal raro de anomalia, hoje dispara em suspensao,
logoff e reboot — ou seja, na rotina.

## Caminho de correcao (do @Bucky)

O Service nao pode depender de avisar o worker: o Windows nao da esse tempo. Duas saidas:

1. **O Service emite o LOGOUT.** Ele ja recebe `SessionLogoff` do WTS e ja sabe fechar evento
   de janela por morte do worker ("WorkerDied") — o mesmo caminho pode emitir o LOGOUT e
   marcar a flag de shutdown limpo daquela sessao. Nao depende do worker estar vivo.
2. **O worker antecipa.** Assinar `SystemEvents.SessionEnding`, que o Windows dispara ANTES
   de encerrar a sessao, e emitir LOGOUT ali. Depende de o processo receber a mensagem a
   tempo — menos confiavel que a opcao 1.

Recomendo a **1**. Quem sobrevive ao logoff e o Service; e nele que o registro de fim de
sessao deve nascer.
