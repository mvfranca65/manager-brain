# ADR-001 (Agent Desktop) — Captura e envio exigem vínculo ativo com colaborador

> **Status:** Aceita — implementada pelo @Bucky, **aguardando validação em máquina**
> **Data:** 2026-08-20
> **Decisor:** Marcos | **Primeira implementação (reprovada):** @Tony | **Review e correção:** @Bucky
> **Domínio afetado:** Agent Desktop (Windows/macOS). O Agent Android herdaria a mesma regra.

---

## Situação desta ADR

Implementada e revisada. 736 testes verdes nas quatro suítes; nada commitado.

O caminho até aqui vale registro: a primeira implementação foi escrita pelo @Tony e
**reprovada** pelo @Bucky em review, com quatro bloqueadores — dois deles parando a captura
de máquina boa em campo, o oposto do que esta ADR promete. As 693 suítes verdes daquela
versão não significaram nada: os testes concordavam com o código errado, e o teste da defesa
mais importante exercitava um caminho que a implementação real nunca alcançava.

O @Bucky corrigiu os cinco achados e a ADR foi ajustada onde a implementação mostrou que a
decisão original estava imprecisa (ver "Revogação" e "Risco de rollout"). Detalhes em
`registro/2026-08-20-review-bucky-adr-001.md`.

**Falta:** rodar o N9 e o N8 em máquina real. Nada foi validado fora de teste unitário.

Fora de escopo: purga do buffer local após revogação definitiva — em aberto com o @Steve,
não bloqueia.

---

## Contexto

A regra foi enunciada pelo Marcos em 2026-08-20:

> "Só podemos capturar evento e enviar pra api se tiver vínculo com colaborador."

Ao conferir o código na 1.5.10 antes de escrever a tarefa, o estado atual é **mais permissivo
do que se imaginava**, em três pontos que se somam:

### 1. A captura não é condicionada a nada

O `SessionWorker` recebe `Linked` na mensagem WELCOME do Service, e o único uso é o ícone
da bandeja:

```csharp
// src/ManagerAgent.SessionWorker/Worker.cs:362
_trayIconManager.SetStatus(welcome.Linked ? TrayStatus.Monitoring : TrayStatus.WaitingLink);
```

O worker captura janela ativa, ociosidade, reunião e resumo de input independentemente do
valor. Um agent instalado que **nunca** conseguiu vincular — identificador errado, colaborador
inexistente, chave de ativação rejeitada — captura tudo e acumula em `events.db`
indefinidamente. O ícone mostra "aguardando vínculo" enquanto o disco enche de dado de uma
pessoa que o sistema não sabe quem é.

### 2. `IsLinked` é frouxo e irreversível

```csharp
// src/ManagerAgent.Service/Linking/AgentLinkService.cs:64
return !string.IsNullOrEmpty(config.DeviceToken) ||
       config.UltimaVinculacaoEm.HasValue;
```

Dois problemas:

- **É um OU, e não olha o colaborador.** `ColaboradorId` e `AgenteId` existem no config e
  não entram na conta. `UltimaVinculacaoEm` é gravado ao fim de `EnsureLinkedAsync` **mesmo
  quando a resposta não trouxe `colaboradorId`** (linha 208, fora de qualquer condicional).
  Basta uma tentativa chegar até ali para `IsLinked` virar `true` para sempre.
- **Não existe caminho de volta.** Nada no código escreve `UltimaVinculacaoEm = null` nem
  limpa o `DeviceToken`. Uma vez "vinculado", sempre vinculado.

### 3. Revogação não é reconhecida (A-53)

O `HttpEventUploader` trata 208, 2xx e 401. Qualquer outro status, inclusive **403**, cai no
ramo genérico `"Batch upload failed. Status={Status}"` e o `UploadWorker` retenta no ciclo
seguinte, para sempre. O header `X-Agent-Revoked` não é lido em lugar nenhum do código
Windows — `grep -rn "Revoked" src/` acha só um `is403` no `UpdateDownloader` da Tray.

O Agent Android já tem `RevocationInterceptor`, que para o serviço via broadcast. As duas
pontas divergem hoje.

---

## Decisão

**Captura e envio ficam condicionados a um vínculo ativo com colaborador.** Sem vínculo, o
agent não coleta e não envia — fica em espera, visível na bandeja, consumindo o mínimo.

### O que é "vínculo ativo"

Vínculo ativo exige, cumulativamente:

| Campo | Por quê |
|---|---|
| `ColaboradorId` preenchido | é o vínculo com a **pessoa**, que é o que a regra pede |
| `AgenteId` preenchido | identifica o dispositivo do lado do backend |
| `DeviceToken` presente | credencial de envio |
| `UltimaVinculacaoEm` preenchido | marca temporal da vinculação |
| ausência de marca de revogação | ver abaixo |

A definição atual (`DeviceToken` **ou** `UltimaVinculacaoEm`) é substituída por essa
conjunção. `UltimaVinculacaoEm` passa a ser gravado **somente** quando `colaboradorId` vier
na resposta; hoje é gravado de qualquer jeito.

### Onde o portão fica

Em **dois** lugares, de propósito:

1. **No Service, antes de lançar o SessionWorker.** Sem vínculo, o worker não sobe. É o
   portão que garante que nada é coletado — e o mais barato de auditar.
2. **No SessionWorker, ao receber WELCOME/estado com `Linked=false`.** Se o worker já estiver
   de pé quando o vínculo cair (revogação em runtime), ele pausa a captura e fecha os eventos
   em andamento, do mesmo jeito que já faz no bloqueio de tela.

Um portão só no upload **não atende a regra**: o dado já teria sido coletado e gravado em
disco. A regra é sobre capturar, não só sobre enviar.

### Revogação

Ao receber **403** com `X-Agent-Revoked: true`, ou ao receber `TokenRefreshFailedException`
(refresh token invalidado no servidor — o código já registra "cliente precisará re-vincular"):

1. Marcar o vínculo como revogado no config (campo novo `revogadoEm`).
2. Parar a captura em todas as sessões.
3. Parar de retentar upload — hoje o `UploadWorker` retenta em loop, o que é ruído e,
   com o vínculo desfeito, envio indevido.
4. Registrar auditoria e refletir na bandeja.
5. Continuar tentando **re-vincular** no ciclo normal: revogação pode ser temporária
   (colaborador reativado). Ao re-vincular com sucesso, `revogadoEm` é limpo e a captura volta.

### O que acontece com o que já está no buffer

**Eventos capturados enquanto o vínculo era válido são preservados e enviados quando o
vínculo voltar.** Eles foram coletados licitamente e pertencem ao histórico do colaborador;
descartá-los cria buraco no relatório sem ganho de privacidade.

**Eventos capturados sem vínculo não existem** — com o portão em pé, não são criados. Não há
o que descartar.

Fica **em aberto para o @Steve**: se a revogação for definitiva (colaborador desligado),
existe algum prazo após o qual o buffer local deve ser purgado? Hoje o buffer não tem teto de
retenção no Windows (o Android tem trim FIFO de 10k eventos ou 7 dias). Isso não bloqueia
esta ADR.

---

## Consequências

**A favor**

- Fecha a exposição de LGPD: nenhuma coleta sem vínculo identificado.
- Fecha o A-53 e alinha Windows com o `RevocationInterceptor` do Android.
- O buffer para de crescer sem teto em máquina que nunca vinculou.
- Máquina mal instalada fica evidente na bandeja em vez de silenciosamente coletando.

**Contra, e aceito**

- Uma indisponibilidade longa do `srv-admin` na primeira instalação atrasa o início da
  coleta. Aceito: o backend não vincular é exatamente o caso em que não se deve coletar.
  Instalação já vinculada não é afetada — o vínculo persiste no config.
- Um falso 403 do backend derruba a captura da máquina. Mitigação: só reage a 403 **com** o
  header `X-Agent-Revoked: true`, e a re-vinculação continua sendo tentada.
- Custo de teste maior: a suíte precisa cobrir o caminho "sem vínculo" que antes não existia.

---

## Alternativas descartadas

**Portão só no upload.** Não atende a regra: o dado seria coletado e gravado em disco, e a
exposição de LGPD é na coleta.

**Capturar e descartar depois.** Pior dos dois mundos — coleta acontece, e ainda perde-se
dado legítimo por engano.

**Confiar no `IsLinked` atual.** Ele é verdadeiro depois de qualquer tentativa que chegue à
linha 208 e nunca volta a ser falso. Não sustenta a regra.

---

## Aceite

**Testes automatizados** (verdes): `VinculoStatusTests` cobre os dois lados — o que a regra
bloqueia (nunca vinculou, token sem data, data sem token, revogado) e o que ela **não pode**
bloquear (config legado sem colaborador, config ilegível). `WorkerLauncherTests` cobre o
portão. `HttpEventUploaderTests` cobre a revogação, inclusive os dois casos em que ela
**não** deve disparar (403 sem o cabeçalho, cabeçalho com valor `false`).

**Cenários na máquina:**

- **N9 modo diagnóstico** (padrão, não altera nada) — responde antes de instalar se aquela
  máquina continuaria capturando com a regra nova. É a rede de proteção do rollout.
- **N9 `-SimularSemVinculo`** — remove o vínculo e prova que nada é capturado nem enviado,
  exigindo que o agent tenha tentado re-vincular (senão "não capturou" não prova nada).
- **N8 `-ExigirParadaNaRevogacao`** — passa a ser o modo padrão no runner.

## Risco de rollout, e como foi contido

O critério antigo era um OU; o novo é conjuntivo. Máquina que estivesse só com a metade
certa passaria a não capturar. Três defesas:

1. Ausência de `ColaboradorId` **não** bloqueia — só gera aviso em log.
2. Config ilegível **libera** a captura. Dúvida não é prova de ausência de vínculo.
3. O N9 em modo diagnóstico responde, máquina a máquina, antes de instalar.

---

## Relacionados

- A-53 (revogação não tratada no Windows) — fechado por esta ADR
- A-47 (título de janela carrega conteúdo digitado) — aceito como está pelo Marcos; independente
- Handoff `registro/2026-08-20-handoff-bucky-a52-e-vinculo.md`
