# Retomada dos testes manuais — onde paramos (2026-08-20, 23:35)

> Para a sessao que continuar depois do reboot/logoff. Escrito para ser lido do zero.

## Estado da maquina

Agent **1.5.10.0**, compilado 20/08 10:49 — **nao e o pacote publicado**: inclui a ADR-001
(regra de vinculo) e o A-52 (resumo de input a 180s). Servico e Watchdog de pe.

Nada commitado, em nenhum repo.

## Ja feito e aprovado

| Camada | Resultado |
|---|---|
| Regressivo local (com rate limit) | 22 passou, 0 falhou |
| Conferencia na base | 13 passou, 0 falhou |
| Cenarios N1..N9 | 7 de 7 |
| Testes de codigo (4 suites) | 753 verdes |

Manuais concluidos: **M1** (bloqueio), **M4** (reuniao), **M5** (suspender), **M6** (sem
internet), **M7** (status), **M9** (bandeja), **M10** (dois usuarios), **M11** (ciclo do
Service).

## Faltam tres, todos derrubam a sessao do terminal

### M3 — Reboot
1. Reiniciar o Windows, fazer login, esperar 30s.
2. Conferir: agent sobe sozinho; icone volta em ate 30s; **um unico LOGIN**;
   SESSAO_INTERROMPIDA aqui e **legitima** (diferente do A-56, onde vinha de suspensao);
   nenhum evento aberto de antes do reboot.

### M2 — Logoff e login
1. Logoff, login de novo, esperar ~1min.
2. Conferir: **um LOGOUT e um LOGIN**; o LOGOUT sobe rapido (A-36 fura o ciclo de 60s — se
   so aparecer no boot seguinte, e o A-38 regredindo); evento de janela aberto **fecha** no
   logoff (A-35); no log, "LOGIN suprimido - LogonId ja registrado" **nao** pode aparecer em
   login de verdade.

### M8 — Instalacao e desinstalacao
Por ultimo: destroi a maquina de teste. Aproveitar para medir o **A-49** — anotar o
`instalacaoId` antes e rodar depois:
`.\tests\e2e\scenarios\n6-instalacao-id-estavel.ps1 -IdEsperado <id-anterior>`

## Como conferir (o executor tem acesso ao banco)

Os scripts de consulta ficaram em `%TEMP%\claude\...\scratchpad\chk-*.js`. Precisam da
variavel `U` com a URL do Postgres de staging.

**A senha do banco passou pelo chat duas vezes hoje. Trocar.**

## Achados abertos desta rodada

| # | O que e | Dono |
|---|---|---|
| A-55 | caminho da URL chega ao banco pelo titulo da janela | decisao @Steve (como o A-47) |
| A-56 | suspensao gera falso "sessao interrompida" e deixa worker orfao | @Bucky |
| A-57 | uma reuniao vira duas linhas em `eventos_reuniao` | @Shuri (handoff pronto) |
| A-58 | ninguem gera ATIVO/PAUSA/AUSENTE desde maio | decisao: backend derivar (@Shuri) |
| A-59 | atividade de outro usuario vai para o colaborador vinculado | **decidido**: vinculo por maquina fica |

Ponto do Marcos a resolver: garantir **sempre um unico agent rodando**, por qualquer caminho.

## Pendente de decisao do Marcos

- Commitar o trabalho do dia (agent + brain).
- Os 9 cenarios de auto-update, que ele decidiu fazer manualmente.
- Publicar a 1.5.10 (o A-52, que bloqueava, esta corrigido).
