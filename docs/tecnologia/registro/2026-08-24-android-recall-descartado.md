# Android e recall — por que NAO vamos construir isto (2026-08-24)

> **De:** @Tony | **Decisao do Marcos, 24/08**
> **STATUS: NAO EXECUTAR.** Este documento nasceu como tarefa para o @Sam e virou registro.
>
> **Motivo, na palavra do Marcos:** *"no app teria que ser uma atualizacao com a versao anterior
> no caso ne? nunca voltar versao e sim lancar uma mais alta com o apk antigo... entao acho que
> nem vale mexer agora... isso eu ja consigo fazer na mao de certa forma."*
>
> **Ele esta certo, e a conclusao e mais forte do que parece:** no Android o "recall" e uma
> atualizacao normal — publicar um build com `versionCode` maior contendo o comportamento antigo.
> O `UpdateCheckWorker` do app **ja pergunta de 6 em 6 horas, ja baixa, ja valida SHA-256 e ja
> notifica**. Nao ha mecanismo novo a construir; haveria apenas um texto de notificacao diferente
> e um registro de auditoria. E como o colaborador precisa tocar de qualquer forma, nada se perde
> fazendo pelo processo de release de sempre.
>
> **Consequencia na API:** a coluna `versao_alvo`, que eu tinha especificado para servir ao app,
> **saiu** da tarefa da @Shuri. Ela nao servia nem ao Android nem ao Windows — era complexidade
> sem uso.
>
> **O que sobra de valioso aqui** e a investigacao da secao 1.3: por que o Android nao permite
> voltar de versao. Se um dia alguem propuser "rollback no app", a resposta esta la, medida no
> repo, e nao precisa ser redescoberta.

---

# Parte 1 — o contexto

## 1.1 O problema

Uma versao vai para a frota, sobe normalmente, passa pelo rollout, e so depois se descobre que
esta errada — captura dado errado, gasta bateria, trava depois de horas. Nada disso faz o app
deixar de abrir, entao nenhum mecanismo automatico percebe.

Hoje, no Android **e no Windows**, isso so se desfaz aparelho a aparelho, na mao.

O que existe hoje e um **freio, nao marcha a re**: `versoes_agente.pausada = true` impede a
entrega a quem ainda nao pegou. Quem ja atualizou fica onde esta.

## 1.2 Como resolvemos no Windows

O Agent Windows ganhou em 24/08 um mecanismo de restauracao local:

1. Antes de sobrescrever os binarios, o script de update **copia a instalacao inteira** para
   `C:\Program Files\bin.previous`.
2. Quando e preciso voltar, um script externo para os servicos, **copia o backup por cima** da
   instalacao e sobe de novo.
3. O resultado e reconciliado no proximo start.

Custo zero de rede, questao de segundos, sem participacao do usuario.

**Detalhe que vale para voce:** a troca e feita por um **processo externo**, e nao pelo proprio
Agent, porque nenhum processo substitui o proprio binario em execucao. E a mesma classe de
restricao que voce vai encontrar no Android, por outro motivo.

## 1.3 Por que isso NAO serve no Android — e a parte que muda tudo

Fui olhar o seu repo antes de escrever. O que encontrei:

- a instalacao do APK e por `ACTION_VIEW` (`ApkDownloadWorker.kt:102`), o fluxo padrao de instalar
  app de origem desconhecida, **com o colaborador tocando para confirmar** — como tem de ser em
  BYOD sem MDM;
- por esse caminho, o Android **recusa** instalar APK com `versionCode` menor que o instalado. Nao
  ha flag disponivel para app lateral sem privilegio de sistema;
- desinstalar antes **apagaria os dados do app** — o buffer Room com eventos ainda nao enviados, o
  vinculo, as permissoes concedidas. E o app se auto-remove da gaveta apos a configuracao, entao
  reonboarding em BYOD e um pesadelo. **Inaceitavel**;
- nao existe copia do APK anterior guardada. Nao ha de onde restaurar.

**Conclusao: no app, "voltar" so pode ser "ir para frente".** Publicar um build com `versionCode`
MAIOR cujo comportamento e o da versao boa.

**Se voce chegar a outra conclusao, me procure antes de codificar** — este e o ponto em que a
tarefa inteira se apoia, e prefiro estar errado agora do que depois.

## 1.4 O que isso fez com a API — e por que ela voltou a ser simples

Cheguei a redesenhar a API para servir aos dois: a resposta deixaria de dizer *"volte uma versao"*
e passaria a dizer *"saia da X, va para a Y"*, com uma coluna `versao_alvo`.

**Isso caiu junto com a decisao.** Sem o Android na conta, o alvo explicito nao servia a ninguem —
o Windows so precisa de "volte", porque a copia anterior ja esta no disco dele. A tarefa da @Shuri
voltou ao desenho simples, com um campo booleano.

# Parte 2 — o que seria preciso, se um dia voltar

**Nao ha nada a fazer hoje.** O que segue e so o desenho que existia antes da decisao, guardado
para nao ser redescoberto do zero.

## 2.1 Nao ha push. O aparelho pergunta.

O servidor **nao alcanca** o aparelho — ele esta na rede do colaborador, atras de NAT e firewall.
Tudo parte do app: o `UpdateCheckWorker` pergunta de 6 em 6 horas, e qualquer ordem viaja de
carona nessa pergunta. As 6h nao sao lentidao do comando; sao o intervalo do app.

## 2.2 O que mudaria no app

Reusaria o que ja existe — download, validacao de SHA-256, notificacao e instalacao sao os mesmos.
Mudaria o gatilho e o texto:

- `UpdateAvailability` ganharia um caso alem de `None`/`Available`/`Error`, para o app distinguir
  "atualizacao normal" de "atualizacao necessaria" no texto e no log;
- a notificacao usaria o tratamento de obrigatoria que ja existe
  (`setFullScreenIntent`, `ApkDownloadWorker.kt:140`);
- auditoria registrando origem e alvo, pelo `UpdaterAuditBridge`.

**Ganho real sobre publicar a versao na mao: pequeno.** Texto de notificacao e rastro de auditoria.
O mecanismo de entrega seria exatamente o mesmo.

## 2.3 A verdade sobre "sem atuacao manual"

No Windows o recall e automatico de ponta a ponta.

**No Android nao pode ser.** BYOD sem MDM exige o toque do colaborador — nao e limitacao do nosso
codigo, e do Android. Isto vale hoje e valeria com o recall implementado: **e a razao pela qual
implementa-lo agrega tao pouco.**

---

# Parte 3 — o que NAO fazer, se isto voltar

| | Por que |
|---|---|
| **Nao tentar instalar `versionCode` menor** | o Android recusa. Se parecer funcionar em emulador com app debuggable, nao vale — em BYOD nao funciona |
| **Nao desinstalar para reinstalar** | apaga o buffer de eventos, o vinculo e as permissoes. Perda de dado do colaborador |
| **Nao guardar APK anterior no aparelho** | ~40MB por versao em aparelho pessoal, e nao resolve: o downgrade continua barrado |
| **Nao encurtar o intervalo de 6h** | decisao do Marcos |

---

# Anexo — leitura relacionada

| Documento | O que tem |
|---|---|
| `registro/2026-08-24-rollback-nao-esta-ligado.md` | como o rollback do Windows estava quebrado e por que ninguem viu |
| `registro/2026-08-24-refinamento-recall-de-frota.md` | o desenho do recall e as alternativas descartadas |
| `registro/2026-08-24-tarefa-shuri-R03-recall.md` | o lado backend, **so Windows** |
| `agent-desktop/TESTES-ROLLBACK-E-RECALL.md` | cenarios de teste da feature no Windows |
