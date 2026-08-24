# Refinamento para @Bucky — A-63, A-64, A-65 (2026-08-21)

> **De:** @Tony | **Para:** @Bucky | **Aprovador:** Marcos
> **Origem:** bateria da 1.5.11 na `DESKTOP-VMSM6LE`, 21/08 entre 11:42 e 15:37.
> Registro completo, com todos os logs: `registro/2026-08-21-bateria-1.5.11.md`
>
> **Regras:** nao commite, nao faca push. Ao terminar, **@Tony revisa** — ordem do Marcos.

---

## Antes de comecar: o que NAO mexer

A 1.5.11 esta boa. Onze dos quinze testes de maquina passaram, incluindo os quatro mais dificeis
— logoff, reboot, desligamento e dormir 31 minutos. **Nao ha nada a corrigir nesses caminhos.**

**Nao encoste em:**

| Area | Por que |
|---|---|
| `SessaoInterrompidaDecider` | provado nos tres desligamentos de hoje, incluindo a reciclagem de SessionId |
| Janela de confirmacao de logoff (A-61) | `sessao 1 confirmou logoff dentro da janela de 5s. Nao relanca.` — funcionou no caso que a motivou |
| Relogio unbiased (A-56) | 31 minutos dormindo, nenhuma falsa morte de worker |
| Calculo de status (A-58) | 5min / 15min / volta imediata, medidos em uso real |
| Buffer por sessao (A-60) | dois usuarios simultaneos, zero falha de escrita |
| Formato da flag de shutdown limpo | sai com `logonId` correto |

O que segue sao **tres defeitos novos**, todos no caminho de **entrega de evento**. Nenhum no
ciclo de vida de sessao que voce reescreveu.

---

# A-63 — LOGOUT sem dono envenena a fila para sempre

**Severidade: ALTA.** Prioridade 1.

## Sintoma medido

93 recusas HTTP 400 hoje, de **dois** lotes distintos, ainda em laco agora:

```
Batch upload failed. Status=400, User=null, Attempt=1
{"codigo":"PAYLOAD_INVALIDO","issues":[{"path":"windowsUsername",
 "code":"invalid_type","message":"Invalid input: expected string, received null"}]}
```

`Attempt=1` em todas as 93 — o contador nunca avanca; cada ciclo trata como lote novo.

## A causa e uma corrida dentro do proprio Service

`ManagerAgentService.cs:319-321`, no caminho do LOGOUT emitido pelo Service:

```csharp
var usuario = _workerRegistry.Get(sessionId)?.WindowsUser;
if (string.IsNullOrWhiteSpace(usuario))
    usuario = _sessionMonitor.GetSessionUsername(sessionId) ?? string.Empty;
```

A intencao esta certa: registro primeiro, WTS como reserva. **O problema e quem chega antes.**

**Caso que deu certo (14:59:49, logoff):**

```
14:59:49.261  LOGOUT emitido pelo Service. Usuario=NoisyTech
14:59:49.312  Worker removed from registry. Session=1      <- removeu DEPOIS
```

**Caso que envenenou (15:21, troca de usuario):**

```
15:21:02.459  Worker removed from registry. Session=1      <- removeu ANTES
15:21:03.614  LOGOUT emitido pelo Service. Usuario=(desconhecido)
```

Quando o `WorkerWatchdog` trata a morte do worker **antes** de o handler de logoff rodar, o
registro ja nao tem a entrada. A reserva do WTS tambem falha, porque a sessao ja sumiu do
Windows. Sobra `string.Empty`, que vira `null` no payload.

### Por que a corrida aconteceu justamente ali

A fila de eventos de sessao ficou **37 segundos** parada tentando lancar o worker da sessao 2:

```
15:20:26.481  WTS session change: SessionLogoff, SessionId=1     <- chega
15:21:01.394  Plans A/B failed for session 2. Falling back to Task Scheduler.
15:21:03.599  Task Scheduler launch succeeded but worker process not found for session 2
15:21:03.600  Session event: SessionLogoff, SessionId=1          <- so agora e tratado
```

**Lancamento lento de uma sessao segura o tratamento do logoff de outra.** Numa troca de usuario
as duas coisas acontecem juntas — e o caso normal, nao a excecao.

## O que fazer

**1. Nao perder o nome.** O `WindowsUser` chega confiavel no CONNECT
(`PipeMessageHandler.cs:95`). Guarde-o por sessao **fora** do ciclo de vida da entrada do
registro, ou nao o apague quando o worker sai do registro. O logoff acontece depois da morte do
worker por definicao — depender de uma entrada viva e depender de algo que ja morreu.

**2. Nunca montar lote sem dono.** Se apos as duas tentativas o usuario continuar vazio, **nao
chame `InsertEventsAsync`**. Logue **um** erro e siga. Um evento perdido e melhor que um lote
imortal que queima o flush de emergencia (ver A-64).

**3. Tratar logoff fora da fila que lanca worker.** Logoff tem prazo de validade; lancamento nao.

Faca 1 e 2 nesta rodada. **O 3 e mudanca de arquitetura da fila de eventos — traga o desenho
antes de escrever.**

## Aceite

- Troca rapida de usuario, com o worker da outra sessao demorando a subir: o LOGOUT sai com nome.
- Nenhum lote com `windowsUsername` vazio ou nulo chega a ser montado.
- Teste que reproduza **a ordem invertida** — registro removido antes do handler de logoff.

## Limpeza

Os dois lotes envenenados estao no `events.db` da maquina do Marcos. **Nao escreva rotina de
faxina para isso** — o teto do A-64 os resolve. Se atrapalhar o teste, apago a mao.

---

# A-64 — nada no Agent desiste nunca

**Severidade: ALTA.** Prioridade 2, e e o que protege o caminho critico.

## Duas ocorrencias, uma causa

**Lote recusado pelo backend.** 93 tentativas hoje. Sem teto, sem fila morta.

**Arquivo de buffer ilegivel.** `OrphanBufferSweeper.RecolherAsync`, ~linha 186:

```
13:05:10 [WRN] Buffer orfao autonomous-buffer-S77.db nao pode ser lido. Ficara para a proxima varredura.
13:05:25 [WRN] (identico)   13:05:40 [WRN] (identico)   13:05:55 [WRN] (identico)
```

A cada 15 segundos, para sempre. 5.760 avisos por dia.

O arquivo corrompido fui **eu** que plantei, para o item 15 da sua lista. O comportamento de
insistir sem fim independe de como o arquivo estragou — disco cheio, queda no meio da escrita,
truncamento no update.

## O que eleva isso de ruido a risco

No desligamento das 15:33:

```
15:33:21.698  "Tentativa final de upload antes de parar (teto 5s)."
15:33:21.710  Batch preparado. Events=1 User=null DedupHash=93926918...
15:33:26.711  "Tentativa final estourou o teto de 5s. Eventos seguem no buffer."
```

**Os 5 segundos de emergencia foram inteiros para um lote impossivel.** A fila real nao chegou a
ser tentada.

Hoje nao houve perda — os eventos reais tinham subido 100ms antes e o resto voltou no boot. Mas
**todo desligamento passa a queimar a janela de emergencia num lote que nunca sera aceito**, e
sao dois deles agora.

## O que fazer

**1. Teto de tentativas por lote.** Contador **persistido junto do lote** — hoje e `Attempt=1`
sempre, o que prova que nao ha estado nenhum. Estourado o teto: sai da fila, com **um** registro
de erro.

**2. Quarentena do arquivo ilegivel.** Apos N varreduras sem conseguir ler, renomeie com sufixo
(`.corrompido`) e pare de varrer. **Nao apague** — a regra que voce mesmo escreveu no codigo, de
so apagar apos entrega confirmada, continua valendo.

**3. Nenhum aviso identico se repete sem teto.** Vale como regra geral desta base.

**Nao mexa** na politica de retentativa de erro **transitorio** (rede caida, 5xx). O alvo e so a
falha **permanente**: 4xx por conteudo, e arquivo que nao abre.

## Aceite

- Lote recusado por conteudo sai da fila apos N tentativas, com um registro.
- Arquivo ilegivel vai para quarentena e para de ser varrido.
- Desligamento com lote envenenado na fila: a tentativa final alcanca os lotes validos.
- Nenhum aviso identico mais de N vezes.

---

# A-65 — sessao do segundo usuario fica "encerrada" enquanto ainda trabalha

**Severidade: ALTA** pelo efeito em relatorio. Prioridade 3.

## Medido

O que o banco viu da sessao do Marcos:

```
14:06:53  LOGIN
14:11:49  LOGOUT  (motivo SERVICE_STOP)
   ...    nada, ate o reboot das 15:09
```

O que aconteceu: ele **continuou logado e trabalhando das 14:12 as 15:09** — 57 minutos — com
eventos de janela chegando e subindo carimbados `User=Marcos`.

## Tres coisas, cada uma certa sozinha

1. **Service para -> worker se despede -> LOGOUT.** Mas o **logon do Windows continua vivo**. A
   pessoa nao saiu de lugar nenhum.
2. **Service volta -> `LOGIN suprimido — LogonId ja registrado`.** Correto: e o mesmo logon.
3. **Reboot com a sessao 2 desconectada:** so chegou `PipeServer: worker disconnected (EOF).
   Session=2`. **Nenhum `SessionLogoff`, nenhum LOGOUT, nenhuma flag nova.** O tratamento do
   A-62 depende do `SessionLogoff`, que para a sessao do console veio e para a desconectada nao.

Somadas: **um LOGOUT que nunca recebe o LOGIN de volta.**

## Impacto

- Relatorio mostra a pessoa saindo as 14:11 e produzindo evento ate as 15:09.
- Toda soma de tempo logado erra para menos.
- Alimenta o passivo de eventos abertos (33 ociosidades, 27 janelas) — decisao com o @Steve.
- **Escala com a frota:** todo update do Agent para o Service em cada maquina. Quem tiver duas
  contas abertas ganha um par quebrado por update.

## Caminhos

**1 (recomendado). `SERVICE_STOP` nao emite LOGOUT.** Nao houve saida de pessoa. Se e preciso
marcar a interrupcao da captura, use um tipo proprio — nao reaproveite LOGOUT, que tem
significado de negocio no relatorio. Menor mudanca, menor risco.

**2. Se mantiver o LOGOUT**, o retorno tem de reabrir: suprimir LOGIN so quando **nao** houve
LOGOUT daquele logon. Espelha a regra de D2 no sentido inverso. **Mais arriscado** — mexe na
supressao de LOGIN, que hoje esta certa nos outros casos.

**3. No sinal do SCM, fechar todas as sessoes**, nao so a do console. O Service ja sabe quais
existem (`Found N active user sessions`). Complementa 1 ou 2; nao substitui.

**Faca 1 + 3.** Se achar que o 2 e necessario, me chame antes — nao mexa na supressao de LOGIN
por conta propria.

## Aceite

- Parar e subir o Service com duas contas logadas nao gera LOGOUT sem LOGIN correspondente.
- Desligar a maquina com uma conta desconectada fecha a sessao dela tambem.
- Nenhuma sessao aparece encerrada com evento posterior ao encerramento.
- Os casos que ja passam continuam passando: logoff real, reboot, desligamento.

---

# Quarto item, pequeno: tres mensagens que mentem

Nesta base isso nao e cosmetico. A divida 4 de D4 registra um comentario falso que sustentou um
engano por **tres meses**, e o A-58 ficou morto nesse tempo.

| Onde | Diz | Verdade |
|---|---|---|
| Worker, ao acordar | `Gap de wall-clock detectado **sem PowerMode event**` | Suspend e Resume foram detectados e logados 4s antes. Duas ocorrencias em dois testes |
| Service, em todo login | `[ERR] Failed to launch worker for session N`, duas vezes | Nao ha erro: a sessao ainda nao existia porque a pessoa estava digitando a senha. `ConsoleConnect` chega ao aparecer a tela de login; `SessionLogon` so quando ela entra — 110s de diferenca no teste |
| `run-regressivo-local.ps1`, item M7 | "desde a v1.5.0 quem deriva o status e o backend" | Deixou de valer hoje: o A-58 devolveu o calculo ao Agent |

Corrigir o texto, e rebaixar o segundo para informativo. **So texto e nivel de log — nao mude
comportamento junto.**

---

# Ordem e limites

1. **A-63** itens 1 e 2 — para de criar lote envenenado
2. **A-64** itens 1 e 2 — desarma os dois que ja existem e protege o desligamento
3. **A-65** caminhos 1 e 3
4. Mensagens de log

**Traga o desenho antes de escrever:** A-63 item 3 (fila de eventos) e A-65 caminho 2 (supressao
de LOGIN).

**Nao commite. Nao faca push.** Ao terminar, avise que o **@Tony revisa** — pedido do Marcos, com
a palavra dele: *"muito cuidado para nao quebrar nada ou mexer onde nao devia"*.

**O que o review vai cobrar**, para voce ja se preparar:

- os 935 testes verdes, e os novos cobrindo **a ordem invertida** do A-63 — nao so o caminho feliz
- nenhum teste existente reescrito para acomodar comportamento novo sem justificativa dentro do
  proprio teste
- os itens 1 a 9, 12 e 13 continuando a passar na maquina
- nenhum `catch` novo que engula falha em silencio — foi assim que o A-52 e o defeito de D6
  passaram despercebidos

**Pendencia conhecida, nao sua:** `eventos_sessao` nao tem UNIQUE. A brecha da `GoodbyeMessage`
atrasada (D5) continua aberta e so o banco fecha. Nao tente resolver no Agent.
