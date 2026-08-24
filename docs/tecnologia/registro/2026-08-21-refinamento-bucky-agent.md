# Refinamento para @Bucky — Agent Windows (2026-08-21)

> **De:** @Tony | **Para:** @Bucky | **Aprovador:** Marcos
> **Origem:** testes manuais da bateria 1.5.10, executados na `DESKTOP-VMSM6LE` em 20/08
> **Binario testado:** 1.5.10.0 compilado 20/08 10:49 — inclui a ADR-001 e o A-52
>
> **Regras:** nao commite, nao faca push. As quatro suites tem que terminar verdes
> (hoje 753). `.ps1` em ASCII puro. Codigo em ingles, comentario em portugues.
> Ao terminar cada item, pare e reporte antes de seguir para o proximo.

---

## Ordem sugerida

1. **A-62** (logoff sem LOGOUT) — maior impacto em relatorio, e a correcao abre caminho para os outros
2. **A-56** (suspensao) — mesmo padrao, resolve o worker orfao
3. **A-61** (relance no shutdown) — cai junto com o A-56 se a correcao for a de fundo
4. **A-58** (status ATIVO/PAUSA/AUSENTE) — independente, e o de maior valor de produto
5. **A-60** (buffer do segundo usuario) — independente

---

# O padrao comum: A-56, A-61 e A-62

Antes dos itens, o diagnostico que os une, porque ele pode mudar como voce ataca os tres.

**O agent trata acoes normais do sistema operacional como falha.** Suspender, desligar e
sair da conta sao as tres coisas mais rotineiras que um usuario faz. Nas tres, hoje:

- o `WorkerWatchdog` declara o worker morto e tenta relancar
- a flag de shutdown limpo nao e marcada
- o proximo login emite **SESSAO_INTERROMPIDA** (motivo `NoCleanShutdownFlag`), evento que
  significa crash ou queda de energia

Na base de staging ha **86 SESSAO_INTERROMPIDA em 3 dias** so nesta maquina. O evento
perdeu o significado: se dispara na rotina, ninguem consegue usa-lo para detectar anomalia
de verdade.

Se voce enxergar uma correcao de fundo que resolva os tres — por exemplo, o Service passar a
reconhecer estados do sistema (suspensao, shutdown, logoff) e propagar isso para o Watchdog e
para a marca de shutdown limpo — traga a proposta antes de implementar item por item.
Prefiro discutir uma correcao estrutural a aprovar tres remendos.

---

# A-62 — logoff nunca gera LOGOUT

**Severidade: ALTA.** Nao existe registro de fim de expediente.

## O desenho esta certo; a ordem dos fatos, nao

O codigo ja preve tudo:

- `ManagerAgentService.HandleLogoff` (linha ~181) envia `ShutdownMessage` com
  `Reason = "SESSION_LOGOFF"`, `GracePeriodMs = 3000`
- `Worker.HandleShutdownAsync` responde certo: emite LOGOUT explicito via
  `_sessionMonitor.AddExternalEvent("LOGOUT", ...)`, drena para o buffer e marca a flag de
  shutdown limpo

## O que foi medido no logoff real (20/08, 23:44)

```
23:44:11.983  PipeServer: worker disconnected (EOF). Session=1     <- o Windows ja matou o worker
23:44:12.005  Evento de janela fechado por morte do worker          <- A-35 funcionou
23:44:12.015  WorkerWatchdog: attempting re-launch for Session=1    <- relanca no meio do logoff
23:44:12.964  WTS session change: Reason="SessionLogoff"            <- o Service SO AGORA sabe
23:44:12.975  Handling logoff for session 1                          -> manda ShutdownMessage
```

**O worker morreu 1 segundo antes de o Service saber do logoff.** A `ShutdownMessage` vai
para um processo que nao existe mais.

## Consequencias

1. Nunca ha LOGOUT. A jornada nao tem fechamento — so o LOGIN do dia seguinte.
2. Todo logoff vira SESSAO_INTERROMPIDA.
3. O A-36 (upload imediato de fronteira de sessao) fica sem efeito no logoff.

## O que precisa ser feito

**O Service nao pode depender de avisar o worker — o Windows nao da esse tempo.**

Caminho recomendado: **o Service emite o LOGOUT.** Ele ja recebe `SessionLogoff` do WTS e ja
sabe agir sozinho nesse caminho — o "Evento de janela fechado por morte do worker
(WorkerDied)" prova que ele consegue fabricar evento sem o worker. O mesmo ponto pode:

- emitir o evento de sessao LOGOUT daquela sessao
- marcar a flag de shutdown limpo daquela sessao, para o proximo login nao acusar
  SESSAO_INTERROMPIDA

Alternativa que avaliei e **nao** recomendo: o worker assinar `SystemEvents.SessionEnding`.
Depende de o processo receber a mensagem a tempo, e o log mostra que o Windows nao coopera.

**Se voce discordar do caminho, diga antes de implementar.** Voce conhece o ciclo de vida
melhor que eu.

## Aceite

- Logoff real gera **um LOGOUT**, com o horario do logoff.
- O login seguinte **nao** emite SESSAO_INTERROMPIDA.
- O LOGOUT chega ao banco sem esperar o proximo boot (o A-36 volta a ter efeito).
- Teste automatizado cobrindo o caminho "Service recebe SessionLogoff com worker ja morto".

---

# A-56 — suspensao gera falso SESSAO_INTERROMPIDA e deixa worker orfao

**Severidade: ALTA.** Nao perde dado, mas duplica processo e polui relatorio.

## O que foi medido (20/08, suspensao de ~2min)

```
22:44:40  suspensao. O worker para de pingar (a maquina dorme)
22:47:03  WorkerWatchdog: "dead worker detected. PID=20420, last ping was 174s ago"
22:47:04  o worker 20420 ACORDA e reconecta - ele nunca esteve morto
22:47:11  o Watchdog lanca o PID 23836 assim mesmo
22:47:12  23836 conecta e e registrado
22:47:13  20420 tenta reconectar -> "CONNECT rejected: PID mismatch"
```

**Dois SessionWorker vivos na mesma sessao.** O guard do Service funciona e so aceita um no
pipe — mas ninguem mata o outro.

Medido 6h depois: o orfao seguia vivo com **136MB e 410s de CPU acumulados**, mostrando um
**segundo icone na bandeja**. Foi assim que o Marcos notou.

## Por que acontece

`WorkerWatchdog.HeartbeatTimeout` e 90s fixos e **o watchdog nao conhece suspensao** —
`grep` por PowerMode/Suspend/Resume no arquivo nao acha nada. Enquanto a maquina dorme, o
relogio corre e o ping nao chega.

O `SessionWorker` tem `PowerModeMonitor` e trata Suspend/Resume corretamente. Quem nao trata
e o lado do Service.

## O que precisa ser feito

Duas frentes. **A primeira sozinha ja resolve o pior sintoma:**

1. **Matar antes de relancar.** Se o watchdog decidiu que o worker morreu, o processo tem de
   ser encerrado antes do relancamento. Hoje ele so remove do registro
   (`Worker removed from registry`) e lanca outro.
2. **Ensinar o watchdog sobre suspensao.** O Service pode ouvir
   `SystemEvents.PowerModeChanged` e reiniciar a contagem do heartbeat no Resume, em vez de
   contar o tempo dormido. Isso evita o falso positivo na origem, e o falso
   SESSAO_INTERROMPIDA cai junto.

## Ponto explicito do Marcos

> "Eliminar e sempre ter apenas um agent rodando."

Nao basta corrigir o caminho da suspensao. Ele quer a **garantia estrutural** de que existe
no maximo um SessionWorker por sessao, por qualquer caminho.

Hoje a garantia e parcial: o Service **rejeita** a conexao do worker excedente
("CONNECT rejected: PID mismatch"), mas nao **encerra** o processo. Um worker rejeitado
continua vivo, consumindo memoria e mostrando icone, ate alguem matar na mao.

Caminhos conhecidos que produzem worker excedente:
- suspensao (este achado, comprovado)
- revogacao seguida de re-vinculacao (voce mesmo previu no review da ADR-001)
- o teste de reconciliacao da suite (A-32, ja contido com `RequiresIsolation`)

**A rejeicao precisa virar um caminho ativo: quem e rejeitado por PID mismatch deve ser
encerrado, nao ignorado.**

## Aceite

- Suspender e acordar **nao** deixa worker orfao.
- Suspender e acordar **nao** gera SESSAO_INTERROMPIDA.
- Worker rejeitado por PID mismatch e encerrado.
- Teste automatizado para o cenario "worker declarado morto que volta a conectar".

---

# A-61 — o Watchdog relanca workers durante o desligamento

**Severidade: BAIXA.** Ruido, sem perda de dado.

Na sequencia do reboot de 20/08:

```
23:36:16  worker S1 morre (EOF)  -> Watchdog relanca PID 21664
23:36:21  SessionLogoff S1       -> morre de novo
23:36:21  worker S2 morre (EOF)  -> Watchdog relanca PID 21320
23:36:23  SessionLogoff S2       -> morre de novo
```

Dois processos nascem e morrem em segundos, enquanto a maquina esta desligando. O Watchdog
trata o EOF do pipe como morte a recuperar, sem saber que ha um shutdown em curso.

Sem consequencia pratica. **Provavelmente sai de graca junto com o A-56 ou o A-62**, se a
correcao for a de fundo. Se depois deles isto ainda acontecer, ai vale um guard proprio: nao
relancar quando o Service esta parando ou quando a sessao esta em logoff.

## Aceite

- Reboot e shutdown nao produzem lancamento de worker.

---

# A-58 — ATIVO/PAUSA/AUSENTE nao e gerado desde maio

**Severidade: ALTA — e funcionalidade de produto, visivel para o gestor.**
**Decisao do Marcos: comportamento igual ao Android.**

## O que foi medido

`eventos_transicao_status`, base de staging:

```
agente 29: 244 transicoes, ultima em 2026-05-15
agente 30: 204 transicoes, ultima em 2026-05-15
agente 31: 200 transicoes, ultima em 2026-05-17
```

**Nada depois de 17/05.** O agente 51 (a maquina de teste, ativa hoje) nao tem nenhuma.

## Por que este item e SEU, e nao do @Shuri

Cheguei a considerar que o backend deveria derivar. O Marcos decidiu **comportamento igual
ao Android**, e no Android quem calcula e emite e o proprio agent. Alem disso:

- O `srv-events-node` **ja tem** `StatusTransitionHandler`, que persiste o evento recebido.
  Nada precisa mudar no backend — ele grava assim que o evento chegar.
- O agent Windows parou de emitir na v1.5.0. Esta escrito em
  `DailyBoundaryWorker.cs:11`: *"v1.5.0: Agent nao emite mais StatusTransitionEvent (backend
  deriva status)"*. O backend nunca derivou.

## O codigo ja existiu no Windows, e foi apagado

Nao e escrever do zero. O commit **b45e181** ("feat: Novos eventos", 12/08/2026) removeu:

| Arquivo | Linhas |
|---|---|
| `Capture/UserStatusManager.cs` | 213 |
| `Capture/IUserStatusManager.cs` | 62 |
| `Capture/Models/UserStatus.cs` | 22 |
| `Capture/Models/EventoTransicaoStatus.cs` | 29 |

O cabecalho do arquivo removido dizia: *"Transicoes: Ativo -> Pausa (5min) -> Ausente (15min
total)"*, com os limites vindos de `opts.LimitePausaMinutos` (5) e `opts.LimiteAusenteMinutos`
(15) — que tambem sairam do `ManagerAgentUploadOptions`.

Recupere com `git show b45e181^:src/ManagerAgent.SessionWorker/Capture/UserStatusManager.cs`.

## O comportamento do Android, que e o alvo

`StatusTransitionCalculator.kt`:

| Regra | Valor |
|---|---|
| ATIVO -> PAUSA | 5 minutos sem interacao |
| -> AUSENTE | 15 minutos sem interacao |
| Volta a ATIVO | qualquer interacao, imediatamente |
| Frequencia de avaliacao | ~30s |

O evento e um **marcador pontual** (`statusAnterior`, `statusNovo`, `transicaoEm`), sem par
abertura/fechamento — nao existe linha que fique orfa com campo nulo. O comentario do arquivo
Android confirma: *"Thresholds alinhados com o desktop Windows Agent (5min pausa / 15min
ausente)"*. O Android e um port do codigo que voce vai restaurar.

Detalhe que vale copiar: o Android trata `resetSessionBoundary()` (lock, troca de sessao)
voltando a ATIVO **sem emitir transicao**, e documenta o porque. Vale o mesmo criterio aqui.

## O que precisa ser feito

1. Restaurar o `UserStatusManager` e os modelos, adaptados ao codigo atual do SessionWorker
   (o arquivo e de antes de varias mudancas — nao aplique o diff cego).
2. Devolver `LimitePausaMinutos` e `LimiteAusenteMinutos` ao `ManagerAgentUploadOptions`, com
   o padrao defensivo `> 0 ? valor : default` que ja e usado no `HeartbeatService`.
   **Atencao:** se voce adicionar as chaves ao `appsettings`, os testes
   `AppSettingsKeysAreReadTests` e `CaptureOptionsHaveConsumersTests` vao cobrar consumidor
   real — o que e exatamente o desejado aqui.
3. Ligar ao `IdleMonitorService`, que era quem o alimentava (`IUserStatusManager _statusManager`).
4. Emitir pelo caminho normal de eventos, para o `StatusTransitionHandler` do backend receber.
5. Conferir a paridade de contrato com o Android — mesmos nomes de status e formato de
   `transicaoEm`. Se divergir, alinhe com o @Sam.

## Aceite

- Uma maquina Windows parada 6 minutos gera PAUSA; parada 16, gera AUSENTE; ao voltar a
  mexer, gera ATIVO.
- Os eventos chegam em `eventos_transicao_status` com `agente_id` correto.
- Testes unitarios cobrindo cada transicao e a volta imediata a ATIVO.
- Contrato identico ao do Android.

---

# A-60 — o segundo usuario da maquina nao consegue gravar no buffer local

**Severidade: ALTA. Perda de evento.**
**Decisao do Marcos: "podemos evoluir e melhorar isso."**

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

A **pasta** `data\` concede Write ao grupo Usuarios, mas os **arquivos** criados pelo primeiro
usuario ficam com apenas ReadAndExecute para os demais. O SQLite precisa modificar o arquivo
existente e criar `-wal`/`-shm`; sem isso, abre em somente-leitura e falha.

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

Duas saidas:

1. **Buffer por sessao** (`autonomous-buffer-S1.db`, `-S2.db`), no mesmo padrao que o
   `session-state` ja usa. Resolve a disputa e a permissao de uma vez.
2. **ACL explicita** nos arquivos de dados no momento da criacao, concedendo Modify aos
   usuarios interativos. Ha precedente no codigo: o `HeartbeatWriter` aplica ACL propria
   ("Heartbeat ACL aplicada. Sids=[SYSTEM]").

**Prefiro a 1**: nao depende de acertar ACL, e dois workers escrevendo no mesmo SQLite e
disputa que nao precisa existir. Mas a decisao e sua — se enxergar problema em multiplicar
arquivo de buffer (retencao, drenagem, limpeza na desinstalacao), traga.

## Aceite

- Um segundo usuario Windows usa a maquina sem nenhum [ERR] de banco somente-leitura.
- Com o pipe derrubado, os eventos do segundo usuario sao preservados e sobem depois.
- Teste cobrindo escrita concorrente ou isolamento por sessao, conforme o caminho escolhido.

---

## Contexto que voce vai precisar

- Registro completo dos testes manuais: `registro/2026-08-20-achados-manuais-1.5.10.md`
- Estado da bateria e o que ficou pendente: `registro/2026-08-20-bateria-1.5.10.md`
- ADR do vinculo (implementada por voce hoje): `agent-desktop/adr/ADR-001-captura-exige-vinculo-ativo.md`

**Nao entram nesta leva:** A-55 (URL no titulo — Marcos aceitou como esta, igual ao A-47) e
A-59 (vinculo por maquina — decidido, fica assim).
