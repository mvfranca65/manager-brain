# Tarefa para @Bucky — R-03: obedecer o recall, lado Agent (2026-08-24)

> **De:** @Tony | **Aprovador:** Marcos
> **Para ser executado numa sessao nova.** Este documento e autossuficiente: leia so ele.
> **Nao commite. Nao faca push. Ao terminar, @Tony revisa.**
>
> **Repo desta tarefa: `manager-srv-agent`.**
>
> **PRE-REQUISITO: faca o R-01 antes** (`registro/2026-08-24-tarefa-bucky-R01-sticky.md`). Os dois
> mexem no mesmo metodo, e o R-01 e menor. Fazer o recall primeiro obriga a mexer duas vezes no
> mesmo ponto.

---

# 1. Contexto

## 1.1 Onde as coisas estao

Diretorio de trabalho: `C:\Users\NoisyTech\Documents\Manager\`

| Repo | O que e |
|---|---|
| `manager-srv-agent` | **o seu.** Agent Desktop Windows, C#/.NET 8 |
| `manager-srv-admin-node` | backend que ja implementou o outro lado — @Shuri |
| `manager-brain` | documentacao |

**Atencao:** o prompt de sessao manda ler `.brain/`. **Essa pasta nao existe** — o cerebro esta em
`manager-brain/docs/`.

## 1.2 O que e o recall, em uma frase

Uma versao vai para a frota, **sobe normalmente**, e so depois se descobre que esta errada — captura
dado errado, gasta bateria, trava depois de horas. Nada disso faz o Service deixar de subir, entao
o rollback automatico nunca percebe. O recall e o Marcos apertar um botao e a frota voltar sozinha.

## 1.3 O que ja existe

**Do lado do servidor: pronto e revisado.** @Shuri entregou em 24/08 (commit `257096f` no
`manager-srv-admin-node`). Review em `registro/2026-08-24-review-tony-R03-recall.md`.

**Do lado da maquina: tambem pronto.** Em 24/08 o rollback automatico foi consertado —
`bin.previous`, `run-rollback.ps1`, reconciliacao no boot. **Voce vai reusar isso inteiro.**

**Falta so ligar um no outro:** ler o campo novo da resposta e disparar o caminho que ja existe.

---

# 2. O contrato — copiado do handoff da @Shuri, secao 1

Fonte: `manager-srv-admin-node/HANDOFF-R03-RECALL.md`. **Se divergir de la, la ganha.**

## 2.1 A resposta do `GET /api/agente/atualizacoes/verificar`

Dois campos novos, **opcionais e aditivos**. Nada mais mudou.

```jsonc
// COM recall — a versao que este Agent roda foi revogada:
{
  "atualizacaoDisponivel": false,   // SEMPRE false quando ha recall
  "recallSolicitado": true,         // presente APENAS quando ha recall
  "recallMotivo": "Captura de tela saindo em branco na 1.5.13"  // pode faltar
}

// SEM recall — identica a de hoje:
{ "atualizacaoDisponivel": false }
{ "atualizacaoDisponivel": true, "versaoNova": "...", /* ... */ }
```

## 2.2 As regras que sao contrato

1. **`atualizacaoDisponivel` vem `false` junto com o recall.** Se voce so olhar esse campo, vai
   ignorar o recall inteiro. **E o erro mais provavel desta tarefa** — ha teste exigindo o contrario.
2. **Nenhum campo de update acompanha o recall** — sem `versaoNova`, `urlDownload`,
   `checksumSha256`. Nao ha o que baixar.
3. **O backend NAO diz para qual versao voltar.** Quem sabe qual e a anterior e a propria maquina,
   que guarda exatamente uma em `bin.previous`.
4. **`recallMotivo` pode faltar** mesmo havendo recall. Trate como opcional de verdade. O uso
   previsto e **log local**, para o post-mortem na maquina do cliente.
5. **Campo desconhecido nao quebra Agent antigo.** A frota de hoje ignora e segue o fluxo normal.

## 2.3 O normalizador tem de casar com o do backend

O backend normaliza **os dois lados** antes de comparar, com esta regra:

> **tres primeiros componentes numericos, resto ignorado.** Menos de tres -> ilegivel.

| O Agent manda | Normaliza para |
|---|---|
| `1.5.13` | `1.5.13` |
| `1.5.13.0` | `1.5.13` |
| `1.5.13+hash` | `1.5.13` |

**Use o mesmo normalizador que voce criou no R-01.** Se as duas pontas discordarem do que "mesma
versao" significa, o recall passa a nao disparar em parte da frota, **em silencio** — que e
exatamente o defeito do A-28.

---

# 3. As tres guardas — o coracao desta tarefa

**Leia isto antes de escrever qualquer linha.**

O rollback automatico so dispara em maquina que **ja esta fora do ar**. O recall dispara em maquina
**saudavel**. As mesmas acoes tem consequencias diferentes, e o codigo de hoje trataria mal tres
casos.

## 3.1 Guarda A — nao reverter para a mesma versao

**O caso:** duas versoes ruins seguidas. A maquina volta da 1.5.13 para a 1.5.12, e a 1.5.12
tambem esta revogada. Ela pergunta de novo e recebe ordem de voltar de novo.

**O que aconteceria hoje:** o `RollbackOrchestrator` **nao compara versao antes de agir** — a unica
guarda e `Directory.Exists(backupBinDir)`. Entao, a cada 6 horas: para os dois servicos, copia
~160MB, sobe de novo, **e continua na mesma versao**. Para sempre, deixando um `bin.failed-*` de
~160MB a cada volta.

**O que fazer:** antes de disparar, ler a versao dentro do `bin.previous` e comparar (normalizada)
com a instalada. **Iguais -> nao dispara.** Registra em `Warning` dizendo que nao ha para onde
voltar, e segue funcionando na versao atual.

## 3.2 Guarda B — sem backup NAO e SOS

**O caso:** primeira instalacao do cliente, que nunca atualizou. Nao existe `bin.previous`.

**O que aconteceria hoje:** `BackupNotFound` -> o `WatchdogService` liga **modo SOS**. No caminho
automatico isso esta certo, porque a maquina ja estava morta. **No recall esta errado:** a maquina
esta capturando normalmente, e o SOS a tira do auto-update sem necessidade — inclusive da correcao
que vai consertar o problema que motivou o recall.

**O que fazer:** no caminho do recall, ausencia de backup **nao liga SOS**. Registra, reporta, e a
maquina continua trabalhando na versao atual.

## 3.3 Guarda C — freio de repeticao

**O caso:** enquanto a maquina continuar na versao revogada — porque a guarda A barrou, porque nao
havia backup, ou porque a restauracao falhou — ela recebe a ordem **a cada ciclo**.

**O que fazer:** nao tentar de novo em todo ciclo. O R-01 ja traz o mecanismo do lado da maquina
(sticky); decida se reusa aquilo ou se um carimbo proprio serve melhor, e **justifique a escolha**.

**Nao invente cadencia longa demais:** o recall existe para ser rapido. Se a primeira tentativa
falhou por algo transitorio (arquivo em uso), tentar de novo tem valor.

## 3.4 O que o backend ja faz por nos, e o que nao faz

O endpoint marca `pausada=true` junto com `revogada=true`, **no mesmo comando**. Isso impede a
maquina de **reinstalar** a versao revogada depois de voltar.

**Nao cobre nenhuma das tres guardas acima** — as tres sao do lado da maquina.

---

# 4. A restricao de arquitetura — traga o desenho antes do codigo

**Quem le a resposta e o Service. Quem executa o rollback e o Watchdog. Sao dois processos
diferentes.**

Conferido: `UpdateCheckerWorker` (Service) **nao enxerga** o `RollbackOrchestrator` (Watchdog).

O que existe hoje entre os dois: o `watchdog-state.json`, escrito pelos **dois** processos pelo
mesmo `WatchdogStateStore` — e o `UpdateCheckerWorker` **ja o le** (`linha 224`).

Ha pelo menos tres caminhos:

- **o Service marca "recall pendente" no estado** e o Watchdog age no ciclo dele (60s). Reusa o
  canal que ja existe, mantem a execucao do rollback num lugar so;
- **mover a geracao do script para o `Shared`** e os dois poderem disparar;
- **o Service dispara direto**, duplicando a logica do orquestrador. *(Eu desaconselho — foi
  duplicacao de caminho que causou o defeito original do rollback.)*

**Traga o desenho antes de escrever.** E a unica parte disto que e arquitetura.

Pontos que o desenho precisa responder:

- o audit do recall **nao pode** ser `UPDATE_ROLLBACK_TRIGGERED` — aquele significa "a versao nao
  subiu 3 vezes". Recall e outra coisa e precisa de nome proprio, senao a telemetria mistura os
  dois e o criterio de "telemetria zero" do rollout gradual passa a mentir;
- a reconciliacao no boot hoje mapeia falha para SOS. **A guarda B exige que o caminho do recall
  mapeie diferente.** Diga como distingue os dois na volta;
- o `recallMotivo` precisa chegar ao log local — e o que torna o post-mortem possivel na maquina do
  cliente.

---

# 5. O que NAO mexer

| Area | Por que |
|---|---|
| `run-rollback.ps1` e a troca de arquivos | feito e revisado em 24/08. Voce **reusa**, nao reescreve |
| Criterio de 3 falhas de startup, sticky de 24h, `RecoveryOrchestrator` | outro caminho, provado |
| `UpdateApplier` e o PS1 de update | funcionando, exercitado ao vivo em 21/08 |
| `WatchdogStateStore.Save` (escrita atomica) | feito em 24/08 |
| Guarda do A-46 e a guarda do sticky (R-01) | corretas e por outros motivos. Some a sua ao lado |
| `ManagerAgent.Tray` | legado |
| Bump de versao | **nao bumpar** |
| Qualquer coisa no `manager-srv-admin-node` | e da @Shuri. O contrato esta fechado |

---

# 6. Testes

## 6.1 Automatizados

| # | Cenario | Esperado |
|---|---|---|
| B1 | Resposta com `recallSolicitado: true` **e** `atualizacaoDisponivel: false` | recall e processado — **o teste do erro mais provavel** |
| B2 | Resposta normal de update | comportamento de hoje, inalterado |
| B3 | Resposta sem campos de recall (backend antigo) | comportamento de hoje, sem erro |
| B4 | Grafias `1.5.13`, `1.5.13.0`, `1.5.13+hash` | todas reconhecidas como a mesma |
| B5 | **Guarda A** — versao do `bin.previous` igual a instalada | **nao dispara**, registra, segue capturando |
| B6 | **Guarda B** — sem `bin.previous` | **nao liga SOS**, registra, segue capturando |
| B7 | **Guarda C** — recall repetido no ciclo seguinte, ainda na versao revogada | nao repete a cada ciclo, na cadencia que voce escolher |
| B8 | Recall com `recallMotivo` ausente | funciona igual; log diz que nao veio motivo |
| B9 | `recallMotivo` presente | aparece no log local |
| B10 | Audit do recall usa evento **proprio**, nao `UPDATE_ROLLBACK_TRIGGERED` | evento distinto |
| B11 | Recall dispara: caminho de restauracao e o mesmo do rollback automatico | reuso, nao caminho paralelo |

## 6.2 Manuais

Entram no `agent-desktop/TESTES-ROLLBACK-E-RECALL.md` como **Teste 10.11**, que ja esta esbocado
la. Acrescente os casos das tres guardas.

## 6.3 A conferencia que eu vou cobrar

**Desfaca cada guarda, uma de cada vez, rode, confirme que quebra, restaure.** Relate quantos
falharam e quais, guarda por guarda. Sem isso eu nao aceito o "passou".

## 6.4 Guardas da base

- **ASCII em script PowerShell gerado** — travessao quebra o parser do PowerShell 5.1 (A-05). Em
  24/08 isso pegou tres erros meus.
- **Nenhum aviso novo de compilacao.** Hoje ha exatamente tres `CS1998`, pre-existentes.
- **Rode a suite antes de comecar e anote o numero.** Se nao bater com o que o R-01 deixou, pare e
  avise.

---

# 7. Aceite

- [ ] Recall processado mesmo com `atualizacaoDisponivel: false`
- [ ] **Guarda A**: nao reverte para a mesma versao
- [ ] **Guarda B**: sem backup nao liga SOS, e a maquina segue capturando
- [ ] **Guarda C**: nao repete a ordem a cada ciclo
- [ ] Normalizacao casando com a do backend nas tres grafias
- [ ] Audit com evento proprio, distinto do rollback automatico
- [ ] `recallMotivo` no log local quando vier
- [ ] Backend antigo (sem os campos) nao quebra nada
- [ ] Restauracao reusa o caminho existente, sem duplicar
- [ ] Suite verde; nenhum teste existente reescrito; nenhum aviso novo
- [ ] Conferencia da secao 6.3 feita e relatada, guarda por guarda

---

# 8. Ao terminar

Escreva `manager-srv-agent/HANDOFF-R03-RECALL-AGENT.md` com:

- o que mudou, arquivo por arquivo;
- **o desenho que voce escolheu na secao 4, e por que** — inclusive os descartados;
- contagem de testes antes e depois;
- o resultado da conferencia da secao 6.3;
- o que encontrou e nao corrigiu, e por que;
- **qualquer ponto em que este documento estava errado.** Isso e util, nao e critica — nas ultimas
  rodadas voce e a @Shuri acharam varias coisas que eu tinha escrito errado, e em todas voces
  tinham razao.

**Pare e me chame** se: o desenho da secao 4 nao fechar; se alguma guarda exigir mexer em algo da
secao 5; ou se o contrato da @Shuri nao cobrir algo que voce precisa.

**Nao commite. Nao faca push.**

---

# 9. Depois disto

O recall so pode ser **ligado em producao** com o R-01 de pe e esta tarefa entregue. E vale a
limitacao que esta escrita no endpoint da @Shuri:

> **O recall so funciona em maquina cujo Agent entende o comando.** Ele protege as versoes
> lancadas depois de existir, nunca a que estiver rodando no dia em que for publicado.
