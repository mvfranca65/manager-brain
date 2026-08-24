# O recall e uma porta de mao unica (2026-08-24)

> **Achado:** @Tony, conferindo o contrato das rotas antes do primeiro recall de teste
> **Registro e verificacao no codigo:** @Shuri | **Repo:** `manager-srv-admin-node`, branch `staging`
> **Estado:** achado registrado. **Nada foi implementado** — a solucao e decisao do @Tony.
>
> Nao confundir com a Decisao 1 do mesmo dia (`sistemaOperacional` obrigatorio no
> `revogar-versao`), que foi implementada e commitada. Sao coisas separadas.

---

## O achado em uma frase

**Existe `revogar-versao` e nao existe nada que desfaca.** Quem revogar a versao errada nao tem
botao de volta.

---

## O que existe hoje

As rotas que mexem em versoes do Agent sao estas cinco, e so estas:

| Rota | O que faz | Desfaz revogacao? |
|---|---|---|
| `POST /api/admin/releases` | publica uma versao nova | nao |
| `GET /api/admin/releases` | lista as versoes | nao (so le) |
| `DELETE /api/admin/releases/:versao` | desativa (`ativa = false`) | **nao** — nem encosta em `revogada` |
| `POST /api/admin/fleet/pausar-versao` | para a entrega (`pausada = true`) | nao |
| `POST /api/admin/fleet/revogar-versao` | **revoga** (`revogada = true` **e** `pausada = true`) | e ela que causa |

Conferido no codigo, nao na documentacao: em todo o `manager-srv-admin-node`, a coluna
`versoes_agente.revogada` e escrita em **um unico lugar** — `FleetHealthRepository.revogarVersao` —
e sempre com o valor `true`. Nao existe caminho de codigo, em nenhum endpoint, que grave
`revogada = false`.

O `DELETE /:versao` engana quem le rapido. O texto dele no Swagger diz "da pra reativar depois se
precisar", mas ele so escreve `ativa = false`, e **tambem nao ha endpoint que reative**. E outra
porta de mao unica, menor, na mesma familia.

### Por que isso nao apareceu antes

Porque foi decidido de proposito, e esta escrito assim no proprio endpoint:

> "Desfazer uma revogacao precisa ser feito por dentro do banco, de proposito — e uma decisao
> consciente, igual ao bloqueio."

O raciocinio original era razoavel: revogar e uma acao seria, desfazer tambem deveria ser. **O que
nao foi considerado e que revogar e uma acao de emergencia e desfazer e uma acao de conserto.** Sao
situacoes diferentes. Quem revoga esta com um incidente na mao; quem precisa desfazer esta com
*dois* — o incidente original e o erro que acabou de cometer. Exigir a mesma cerimonia dos dois e
exigir mais calma justamente de quem tem menos.

---

## Por que e um problema operacional

O recall e disparado sob pressao, no meio de um incidente, e o erro mais provavel nao e tecnico: e
digitar a versao errada, ou escolher o sistema errado. Nesse momento, o operador descobre que:

1. **A acao ja pegou.** Revogar marca `pausada` junto, no mesmo comando. A versao para de ser
   entregue na hora.
2. **A frota comeca a voltar sozinha.** Cada maquina que faz check-in — ate 6 horas — recebe a
   ordem de voltar para a instalacao anterior. Nao ha como chamar de volta as que ja voltaram.
3. **Chamar o endpoint de novo nao ajuda.** Ele e idempotente: a segunda chamada e no-op. Nao existe
   parametro, flag ou variante que reverta.
4. **O unico caminho documentado e `UPDATE` direto no banco.**

E o ponto 4 e onde trava de vez: **o acesso ao banco de staging e uma pendencia em aberto** — item 4
da linha de corte (`registro/2026-08-21-linha-de-corte-producao.md`), senha ainda nao trocada, e o
item mudou de natureza em 24/08 e espera decisao do Marcos. Ou seja, hoje o unico botao de volta que
existe esta atras de uma porta que tambem esta fechada.

Vale dizer o que **nao** e o problema: o recall funciona, e o dano de um recall errado nao e perda
de dados. As maquinas voltam para uma versao que ja rodava nelas. O problema e de **controle** — a
acao nao tem freio de re, e quem a dispara nao consegue corrigir o proprio erro sem depender de
terceiros e de uma pendencia aberta.

---

## Os dois contornos que existem hoje

### Contorno A — publicar uma versao nova

Publica-se uma versao acima da revogada e a frota passa a receber essa.

- **Funciona porque** a trava de 24h do Agent (o *sticky*) barra **uma versao especifica** — a que
  quebrou —, nao qualquer atualizacao. Versao diferente instala normalmente. Esta escrito na regra
  do R-01 (`registro/2026-08-24-tarefa-bucky-R01-sticky.md`): *"Sticky ativa e versao diferente da
  quebrada: instala normalmente."*
- **Custo:** exige um build, um numero de versao queimado e um ciclo de publicacao inteiro para
  desfazer um clique. E leva ate 6 horas para a frota reagir, mais o tempo de produzir o build.
- **Ressalva a confirmar com o @Bucky:** o comportamento acima e a *regra escrita* do R-01. Nao
  conferi na maquina se ja esta implementada e valendo. Esta base ja errou tres vezes lendo
  documento do Agent como se fosse codigo — antes de contar com este contorno num incidente real,
  conferir o `UpdateCheckerWorker` de verdade.

### Contorno B — `UPDATE` direto no banco

```sql
UPDATE versoes_agente
   SET revogada = false, revogada_motivo = NULL, pausada = false
 WHERE versao = '1.5.13' AND sistema_operacional = 'WINDOWS';
```

- **Funciona** e e imediato: o `verificar` volta a nao ver a versao como revogada no proximo
  check-in.
- **Custo:** depende do acesso ao banco de staging (item 4, em aberto), nao passa por auditoria
  nenhuma — nao gera evento, nao registra quem fez, nao registra por que —, e e escrito a mao sob
  pressao. Um `WHERE` incompleto aqui desrevoga mais do que se queria, e ninguem fica sabendo.
- E o caminho que hoje esta escrito como oficial. Na pratica ele e o caminho **menos** auditavel de
  todo o fluxo de recall, o que e o contrario do que a decisao original pretendia.

---

## Proposta para o @Tony decidir

Nao implementei nada. A proposta abaixo e uma sugestao de forma, para o @Tony aceitar, mudar ou
recusar.

**Um endpoint irmao do recall: `POST /api/admin/fleet/desrevogar-versao`.**

O desenho que me parece certo, e o porque de cada parte:

| Parte | Proposta | Por que |
|---|---|---|
| Corpo do pedido | `versao`, `sistemaOperacional`, `motivo` — os tres obrigatorios | mesmo contrato do `revogar-versao` depois da Decisao 1. Se escolher o alvo e proposital para revogar, e proposital para desrevogar |
| O que escreve | `revogada = false`, `revogada_motivo = NULL` | volta a versao ao estado anterior ao recall |
| E o `pausada`? | **nao mexer** — deixar pausada | e a parte que exige decisao. Ver abaixo |
| Auditoria | evento proprio (`VERSION_RECALL_REVERTED`), WARN, com autor, IP e motivo | o conserto tem de ser tao rastreavel quanto o estrago. Hoje o `UPDATE` no banco nao deixa rastro nenhum |
| Idempotencia | versao nao revogada -> no-op 200; par inexistente -> 404 | mesma semantica dos dois endpoints vizinhos, para nao inventar regra nova |

**O ponto que precisa da sua decisao — desrevogar deve despausar junto?**

Revogar marca as duas colunas juntas porque tem de marcar: revogar sem pausar cria laco infinito (a
maquina volta e o ciclo seguinte reinstala a revogada). **Desfazer nao tem essa obrigacao, e as duas
opcoes sao defensaveis:**

- **Deixar pausada** (minha inclinacao): desrevogar para de mandar a frota voltar, mas nao volta a
  distribuir a versao. Duas acoes conscientes, e a segunda ja existe. Erra para o lado seguro — se a
  versao *era* mesmo ruim e o recall foi certo, desrevogar por engano nao volta a espalhar o
  problema.
- **Despausar junto**: um botao so, "desfaz o recall inteiro". Mais simples para o operador, mas
  volta a distribuir a versao — e se o motivo do recall era real, isso reabre o incidente.

Ha ainda uma terceira via — deixar o campo explicito no corpo (`despausar: true|false`, obrigatorio)
— que empurra a escolha para quem chama. Mais honesta, mas mais uma decisao para tomar sob pressao.

**O que eu deixaria de fora desta rodada:** desfazer o `DELETE /:versao` (reativar release). E o
mesmo tipo de porta de mao unica, mas nao e acao de emergencia — desativar uma versao antiga nao
tem ninguem esperando do outro lado. Vale registrar e tratar depois.

---

## Enquanto nao houver decisao

Antes do primeiro recall de teste, vale que o roteiro manual da 1.5.13
(`registro/2026-08-24-roteiro-manual-1.5.13.md`) diga com todas as letras, no passo do recall:
**este passo nao tem volta pelo painel.** Quem executar precisa saber disso antes de apertar, nao
depois.
