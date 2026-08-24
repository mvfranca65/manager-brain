# Review do @Bucky — ADR-001 (vinculo) — REPROVADO

> **Data:** 2026-08-20 | **Autor do codigo:** @Tony | **Revisor:** @Bucky
> **Situacao:** nao commitar. Correcoes com @Bucky.

Build limpo e 693 testes verdes. Isso nao provou nada: os testes concordam com o codigo
errado. Quatro bloqueadores.

## B1 — A protecao principal do rollout nao existe

A ADR promete: "config ilegivel libera a captura". O `ConfigManager.CarregarOuPadrao()`
**nunca lanca** — devolve um config vazio. Entao o `catch` do `VinculoStatus` e codigo morto,
e config corrompido/travado resulta em `PodeCapturar = false`: a captura **para**. Exatamente
o oposto do que foi escrito.

Pior: o teste `Config_ilegivel_libera_a_captura` mocka uma excecao que a implementacao real
nunca produz. A garantia mais importante do rollout esta coberta por um teste de caminho
inalcancavel.

## B2 — Falha de DPAPI para a maquina, e o diagnostico nao detecta

Se o blob DPAPI nao descriptografa (FIPS por GPO, reset in-place do Windows, imagem clonada,
ProgramData restaurado de backup), o `DeviceToken` fica nulo sem lancar excecao. Hoje a
maquina captura; depois do update, para em silencio.

O N9 em modo diagnostico so confere se o **campo existe** no disco, nao se ele descriptografa.
A maquina passa no diagnostico e quebra depois de instalar.

## B3 — Revogado e reativado fica pausado para sempre

A pausa e enviada, mas ninguem envia o retomar. E o caminho de unlock que poderia salvar tem
guarda de estado que nunca e satisfeita, porque o broadcast nao atualiza o registro do worker.
A ADR promete que a captura volta. Nao volta.

## B4 — Loop de relancamento a cada 15s

Worker pausado para de mandar ping; o watchdog o considera morto, remove do registro e tenta
relancar; o portao novo recusa; a recusa nao conta no rate-limit; a reconciliacao tenta de
novo a cada 15s, para sempre, com dois avisos por ciclo por sessao.

Falta a guarda simetrica no `WorkerWatchdog`, que ja existe para o `UpdateGate`.

## C1 — A pausa nao fecha o evento de janela em aberto

O caminho de bloqueio de tela faz o flush antes de pausar, e ate documenta o porque. A
revogacao pulou isso: fica bloco aberto no buffer e o buffer autonomo nunca dreana —
contradizendo o "eventos capturados com vinculo valido sao preservados".

## Clean code — os tres que importam

1. A regra esta escrita duas vezes (`VinculoStatus` e o fallback do `AgentLinkService`), e as
   duas ja divergem. Os testes de `IsLinked` exercitam a copia que producao nao executa.
2. `LaunchWorkerAsync_SemPortaoConfigurado_NaoBloqueia` nao testa o que o nome diz: para no
   guard anterior e nao distingue "sem portao" de "portao fechado".
3. O `HttpEventUploader` da Tray e uma copia que ficou sem tratamento de revogacao.

Mais: escrita descartada no caminho de `Retry`, flag de pausa marcada antes do await,
`TokenRefreshFailedException` nunca ligado a revogacao, e comentario com valor errado (60s
vs 180s).

## Certo, e vale manter

Ordem do 403 (depois do 401, antes do ramo generico); exigir o header alem do status, com os
dois testes negativos; `MarcarRevogado` idempotente; a ideia do N9 em modo diagnostico.

## Cadeia de dependencias: sem problema

@Bucky tracou o grafo: nao ha `new` em codigo de producao, tudo passa pelo registro central,
e o `PipeServer` chega de fato ao `UploadWorker`. O que incomoda e o parametro opcional: se
alguem remover o registro, o portao some sem erro de compilacao e sem teste vermelho.

---

# Correcoes do @Bucky — 2026-08-20

Os cinco achados foram corrigidos por ele. Conferi por fora: a solucao compila e sao **736
testes verdes** (Service 387, SessionWorker 304, Watchdog 31, Tray 14), +43 casos. O portao
de ASCII dos `.ps1` continua verde. Nada commitado.

## Como cada um foi fechado

- **B1** — o criterio passou a distinguir "config ausente" (bloqueia) de "config ilegivel"
  (libera). O teste agora corrompe arquivo de verdade em disco, com contraprova de que o
  caminho feliz nao passa pelo ramo de duvida.
- **B2** — blob DPAPI presente que nao descriptografa passa a contar como vinculo: a maquina
  vinculou, e credencial ilegivel e problema de envio, nao licenca para parar de capturar. O
  N9 agora tenta descriptografar de fato, em vez de so conferir se o campo existe.
- **B3** — a pausa passou a atualizar o estado no registro de workers; sem isso o caminho de
  desbloqueio nunca resgatava a sessao. So retoma o que ele mesmo pausou.
- **B4** — a raiz era outra, e mais antiga: o envio de sinal de vida estava dentro do bloco
  pausado, entao worker pausado era declarado morto. Ja valia para a pausa por desconexao,
  antes da ADR. Corrigido na raiz, mais a guarda simetrica no watchdog.
- **C1** — o fechamento dos eventos em aberto passou a acontecer antes de pausar, com teste
  travando a ORDEM.

## Decisoes dele, diferentes do que ele mesmo propos no review

1. A limpeza da revogacao ficou num lugar so, e nao inline como ele tinha sugerido.
2. Nomes novos em ingles, com uma excecao justificada: o campo gravado no arquivo de config
   fica em portugues, porque renomear quebraria o contrato do arquivo ja em disco.
3. A guarda do token continua olhando o config, e nao a resposta: numa re-vinculacao apos
   revogacao o backend pode nao reemitir token, e exigir token novo prenderia a maquina
   travada para sempre.
4. Extraiu duas costuras que nao estavam na lista, para conseguir testar envio com falha e
   provar que o registro de dependencias monta. Justificou: erro ali nao aparece em teste,
   aparece como servico que nao sobe na frota inteira.

## Fora de escopo, com motivo

- **Tray:** nao trata revogacao. O instalador v2 nem distribui a Tray — ele mata a versao
  legada. E codigo morto; o motivo esta registrado na propria classe.
- **Escrita concorrente de config** entre revogacao e renovacao de token: padrao
  pre-existente, exigiria mudanca no projeto compartilhado. Nao entrou.

## Achado colateral, confirmado por mim

`UploadWorkerTests.cs`, `WorkerWatchdogTests.cs` e `PipeMessageHandlerTests.cs` estao
**excluidos do build** (`<Compile Remove>` no csproj, linhas 21-23). Tres arquivos de teste
mortos: `UploadWorker` e `WorkerWatchdog` nao tinham cobertura compilada nenhuma. Alguem
precisa decidir se ressuscita ou apaga.

## Nao verificado

Ninguem subiu o servico nem rodou N8/N9 em maquina real — exige agent instalado e elevacao.
Fica para a bateria da 1.5.10.
