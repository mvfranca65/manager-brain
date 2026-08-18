> **STATUS:** REVISADA — execução autorizada por Marcos em 2026-08-13
> **DONO ARQUITETURAL:** @Tony
> **EXECUTORES:** @Sam (Android), @Vision (Grafana), @Groot (UX), @Natasha (QA)

# Agent Android — Entrega Confiável e Observabilidade por Eventos

## Objetivo

Dar ao Android a mesma regra do Agent Windows: coletar apenas com sessão ativa, enviar em lote a cada 60 segundos, registrar bloqueio/desbloqueio e preservar eventos em fila local. A operação é observada pelos **eventos e heartbeats já existentes**; não haverá endpoint ou ping de health adicional.

## Decisões

| Situação | Comportamento |
|---|---|
| Sessão desbloqueada | Foreground service gera heartbeat e solicita upload a cada 60 s. |
| `LOCK` | Persiste `SessionEvent(LOCK)`, solicita upload imediato e pausa collectors/ticker. |
| `UNLOCK` | Persiste `SessionEvent(UNLOCK)`, solicita upload imediato e retoma collectors/ticker. |
| Sem rede | Mantém a Room; tenta novamente com backoff quando houver rede e sessão ativa. |
| Processo interrompido | WorkManager de 15 min permanece como recuperação durável. |
| Vários gatilhos | Um coordinator único e leases atômicos evitam duplicidade. |
| Estado no Portal/Grafana | Derivado de `batimentos`, `eventos_sessao`, ingestão e auditorias já existentes. |
| Erro local | Diagnóstico estruturado, mascarado e exportável; nunca payload, segredo ou stack trace bruto. |

O intervalo de 60 s é melhor esforço enquanto o foreground service está ativo. Doze, economia de bateria, OEM agressivo, processo encerrado ou ausência de rede podem atrasar; a garantia é entrega eventual sem perda e atraso observável.

## Arquitetura

```text
Collectors / SessionBroadcastReceiver
              │ persistência transacional
              ▼
         Room EventQueue ── lease ──► UploadCoordinator
              ▲                           │
 LOCK / UNLOCK ┴──────────────────────────┤ unique OneTimeWork (rede)
 ForegroundService ticker (60 s, UNLOCKED) ┤
 WorkManager (15 min, recuperação) ───────┘
                                          ▼
                         POST /api/agent/events (Device JWT existente)
                                          ▼
                         batimentos + eventos_sessao + eventos_* existentes
                                          ▼
                     Portal / Grafana / Loki / audit_events existentes
```

### Fila segura

Cada evento tem `PENDING` ou `IN_FLIGHT`, `leaseId`, `leaseUntil` e tentativas. Uma transação reserva IDs pendentes (ou lease vencida); ACK remove somente IDs da lease correspondente; falha devolve os mesmos IDs a pendente. O worker drena até cinco batches de 100 eventos e agenda continuação se necessário.

Não usar `ExistingWorkPolicy.REPLACE`: um upload aceito pelo servidor não pode ser cancelado antes do ACK local.

### Heartbeat e observabilidade

`HeartbeatEvent` existente passa a ser produzido pelo ticker de 60 s em sessão desbloqueada. O servidor já persiste o heartbeat, que é o sinal operacional canônico: último contato, versão/instalação conforme contrato atual e eventos pendentes.

Grafana deve ler apenas dados existentes e sanitizados:

- idade do último heartbeat por dispositivo;
- quantidade de agentes sem heartbeat na janela por plataforma;
- `LOCK`/`UNLOCK` e período sem atividade enquanto bloqueado;
- taxa de respostas 4xx/5xx e latência do `srv-events-node` via Prometheus/Loki;
- auditorias existentes de falha de token, fila limitada, serviço reiniciado e permissão ausente;
- distribuição de versões Android.

Alertas iniciais: Android sem heartbeat por mais de 5 min após último `UNLOCK`; ingestão 5xx; fila local atingindo limite (audit); token revogado/falha de refresh (audit); permissão essencial ausente (audit). Limiares são configuráveis por plataforma.

Falhas sem conectividade não chegam imediatamente ao servidor por definição. Elas ficam no diagnóstico local e serão auditadas na primeira conexão autenticada seguinte; a ausência de heartbeat já sinaliza o problema no Grafana.

## Experiência do app

### Confirmação e vínculo

Modal Material com estados pronto, preparando e falha. O diálogo de identidade confirma o colaborador antes das telas nativas de permissão. Após a última permissão, um modal não cancelável de vínculo fica aberto durante a chamada, com retry seguro; não volta visualmente para o formulário.

### Tela Saúde do Agent

Após vínculo, manter o ícone na gaveta e abrir uma tela local que mostra: versão/ambiente, serviço, sessão, permissões, fila, última tentativa/sucesso local e diagnóstico categorizado. Estados: “Funcionando corretamente”, “Em andamento”, “Atenção” e “Crítico”. Ações: atualizar diagnóstico, exportar diagnóstico e abrir informações do aplicativo.

O app não declara “enviado” antes de receber ACK. Sem rede, mostra claramente “Aguardando conexão”.

### Segurança e LGPD

- Remover a captura/exibição staging de corpo HTTP e stack trace bruto.
- Categorias permitidas: `NETWORK_DNS`, `NETWORK_TLS`, `HTTP_401`, `HTTP_429`, `HTTP_5XX`, `TOKEN_REFRESH_FAILED`, `QUEUE_LIMIT_REACHED`, `PERMISSION_MISSING`.
- Logs/auditoria/exportação mascaram `Authorization`, activation key, cookies, tokens e identificadores.
- Nunca capturar conteúdo de outros apps, screenshots, áudio, câmera, GPS, payload completo de eventos ou corpo HTTP.

## Entregas

| Repositório | Entrega |
|---|---|
| `manager-srv-agent-android` | Queue lease, coordinator, ticker/sessão, heartbeat 60 s, auditorias de transição, diagnóstico/tela/modal/ícones seguros. |
| `manager-srv-events-node` | Nenhum endpoint novo; manter ingestão atual e métricas/logs já existentes. |
| `manager-srv-portal-node` / `manager-fed-portal` | Nenhuma API nova nesta rodada; Portal usa sinais já existentes quando a tela/dispositivo correspondente existir. |
| `manager-srv-monitoring` | Dashboard e alertas sobre eventos, batimentos, auditorias e métricas existentes. |

## Aceitação em staging

1. Com tela desbloqueada e rede, há heartbeat/event batch em aproximadamente 1 min.
2. `LOCK` chega sem esperar o próximo ciclo; não há ticker recorrente enquanto bloqueado.
3. `UNLOCK` chega imediatamente e o ciclo retoma.
4. Offline preserva a fila; ao reconectar, envia sem duplicar IDs.
5. Remover permissão, induzir 401 ou exceder fila gera auditoria/diagnóstico categorizado, sem segredos.
6. Grafana mostra atraso de heartbeat e falhas de ingestão; tela local mostra a condição correspondente.
