> **DATA:** 2026-08-18 | **AUTOR:** @Tony (refinamento tecnico) | **PEDIDO POR:** Marcos
> **ESTADO:** aguardando escopo do @Steve e desenho do @Groot. **Nao ha codigo escrito.**
> **SEQUENCIA:** entra depois que a 1.5.5 fechar. Abrir feature no meio de uma validacao foi como o A-32 nasceu.

# Refinamento — Tela de Diagnostico do Agent Windows

Pedido: uma tela de monitoramento no proprio Agent, aberta pelo menu da bandeja, no lugar de
depender de script e log para saber se o agente esta saudavel.

Este documento nao decide **se** a feature entra nem **como** ela vai parecer. Decide **como
construir**, se e quando entrar, e mostra o que ja existe — que e mais do que parece.

---

## 1. O que ja existe (e ninguem precisa reconstruir)

### Menu da bandeja
`TrayIconManager.CreateContextMenu` ja tem:

| Item | O que faz |
|---|---|
| Cabecalho `iManager - vX.Y.Z` | Versao, desabilitado |
| Ferramentas > Verificacao de Saude | `health-check.ps1` |
| Ferramentas > Logs em Tempo Real | `monitorar-logs.ps1` |
| Ferramentas > Exportar Diagnostico | `coletar-diagnostico.ps1` |
| Ferramentas > Limpar Dados e Reiniciar | `limpar-reset.ps1`, com confirmacao |
| Sobre | MessageBox com identificacao e status |

### O "Sobre" ja e 80% da tela pedida
`TrayIconManager.ShowAbout()` ja mostra versao, identificador do colaborador, conta Windows,
`instalacaoId` (adicionado pelo achado A-04) e status corrente. Falta ser **viva** em vez de
um retrato estatico, e ganhar os numeros de fila e envio.

### Os dados de saude ja trafegam pelo pipe
`PongMessage` (`ManagerAgent.Shared/Pipe/PipeMessages.cs`) ja carrega:

| Campo | Vira o que na tela |
|---|---|
| `backendReachable` | "Conexao com o servidor" |
| `bufferCount` | Parte de "Eventos pendentes" (backlog do Service) |
| `lastUploadStatus` | Base de "Ultimo envio" |

E o worker ja soma os dois lados da fila:
`_heartbeatService.SetPendingEventsProvider(() => _autonomousBuffer.PendingCount + _serviceBufferCount)`
(achado A-23).

**Conclusao:** a feature e majoritariamente **apresentacao**. Nao ha coleta nova a implementar,
o que muda bastante a estimativa e o risco.

---

## 2. O que falta tecnicamente

| # | Lacuna | Impacto |
|---|---|---|
| 1 | `PongMessage` nao traz **quando** foi o ultimo envio, so um status em texto | Sem isso nao da para escrever "ha 42s". Exige campo novo no contrato do pipe — **os dois lados** (Service emite, Worker consome) |
| 2 | O worker recebe o PONG mas nao guarda `backendReachable` / `lastUploadStatus` em lugar que o Tray enxergue | Precisa de um estado de saude compartilhado entre `Worker` e `TrayIconManager` |
| 3 | PING roda a cada 60s | Dado de ate 1 minuto de idade. Aceitavel para "ultimo envio", ruim para a sensacao de tela viva. Solucao: a janela pede um PING sob demanda ao abrir e a cada refresh |
| 4 | `TrayIconManager` hoje so sabe status, nao saude | A janela nao pode virar um segundo dono de estado — o dado tem de descer do Worker, nunca ser coletado pela tela |

**Regra que nao se negocia:** a tela **le** estado, nunca coleta. Se a janela consultar o SO ou o
banco por conta propria, viram duas fontes de verdade — que e exatamente o achado A-33, e custou
2 LOCK e 2 UNLOCK por ciclo mais 15s de tela de bloqueio contabilizada como trabalho.

---

## 3. Limites de LGPD (inegociaveis)

A tela e visivel para o **colaborador**. Isso a torna, ao mesmo tempo, uma oportunidade e um risco.

**Pode mostrar:** versao, identificador do colaborador, conta Windows, `instalacaoId`, status
(monitorando / pausado / aguardando vinculacao / erro), contadores agregados (eventos pendentes),
horario do ultimo envio, conectividade com o servidor.

**Nao pode mostrar:** titulo de janela, nome de processo capturado, dominio visitado, conteudo de
qualquer natureza. Nem o proprio historico de atividade do colaborador.

O motivo nao e so legal: o Agent nao e um visualizador de atividade. Mostrar o que foi capturado
transforma uma tela de suporte em um painel de vigilancia — e a tela pode estar aberta com alguem
olhando por cima do ombro, ou num compartilhamento de tela.

**A favor da feature:** transparencia sobre o que o agente esta fazendo reforca o posicionamento
declarado do produto ("visibilidade sem vigilancia"). Uma tela que diz o que o agente monitora e
que ele esta funcionando e argumento de venda, nao passivo.

---

## 4. Onde mora

`ManagerAgent.SessionWorker`, junto do Tray. Motivos:

- O loop de mensagens do WinForms ja roda ali (`TrayApplicationContext`).
- E o processo que roda na sessao do usuario — o Service e SYSTEM e nao tem desktop.
- O estado ja esta ali: status do tray, buffer autonomo, PONG do Service.

**Nao** criar projeto novo. **Nao** colocar no `ManagerAgent.Tray` — esse projeto e legado da v1,
nem e empacotado no v2 (o instalador so faz `taskkill` nele).

Cuidado herdado do achado A-29: qualquer janela nova precisa ser descartada no encerramento do
loop de mensagens, senao o processo nao sai sozinho no SHUTDOWN e volta a ser morto de fora.

---

## 5. Quebra de tarefas

| # | Tarefa | Dono | Depende de |
|---|---|---|---|
| D-01 | Definir escopo: quais campos, quem ve, quando abre | **@Steve** | — |
| D-02 | Desenho da janela (layout, textos, estados de erro) | **@Groot** | D-01 |
| D-03 | `PongMessage` ganha `lastUploadAt`; Service preenche | @Bucky | D-01 |
| D-04 | Estado de saude no Worker, alimentado pelo PONG e exposto ao Tray | @Bucky | D-03 |
| D-05 | PING sob demanda (abrir a janela forca atualizacao) | @Bucky | D-04 |
| D-06 | A janela em si, substituindo o MessageBox do "Sobre" | @Bucky | D-02, D-04 |
| D-07 | Descarte no encerramento do loop de mensagens (licao do A-29) | @Bucky | D-06 |
| D-08 | Testes: estado de saude, formatacao de "ha Xs", degradacao sem PONG | @Bucky | D-04 |
| D-09 | Teste de guarda LGPD: a janela nunca recebe titulo nem processo | @Bucky | D-06 |
| D-10 | Roteiro de teste manual (sem rede, sem Service, sem vinculacao) | @Natasha | D-06 |

**D-09 merece destaque.** Mesmo padrao do `LGPDContentFilterTest` do Agent Android: um teste que
trava a sabotagem futura, garantindo por reflexao que o modelo da tela nao aceita `string` de
titulo ou processo. Sem isso, basta alguem "melhorar" a tela um ano depois para virar painel de
vigilancia sem ninguem perceber.

---

## 6. O que eu recomendo

**Fazer, mas depois da 1.5.5.** A feature e barata (apresentacao de dado que ja existe), reduz
chamado de suporte e reforca o posicionamento do produto.

Duas ressalvas:

1. **Nao vira substituto do log.** Para depurar defeito como os desta rodada — evento aberto para
   sempre, LOCK duplicado, LOGIN falso — o que serve e o log e o banco. A tela responde
   "o agente esta funcionando?", nao "por que este evento saiu errado?". Se ela for vendida
   internamente como fim dos scripts de diagnostico, vai decepcionar.
2. **A pergunta que o @Steve precisa responder antes:** a tela e para o **colaborador** se
   tranquilizar, ou para o **suporte** diagnosticar remotamente? As duas respostas geram telas
   diferentes. Se for suporte, o `coletar-diagnostico.ps1` ja resolve e talvez o certo seja
   melhorar o que existe em vez de abrir janela nova.
