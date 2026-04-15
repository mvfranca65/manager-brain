# Teste Regressivo — Agent Desktop (manager-srv-agent)
> @Natasha | QA
> Ultima atualizacao: 2026-03-27
> Executar antes de cada release

---

## Pre-requisitos

- [ ] Agent compilado sem erros (`dotnet build` ou build via IDE)
- [ ] Backend srv-events rodando e acessivel
- [ ] Backend srv-admin rodando e acessivel
- [ ] Agent configurado com `config.json` valido (chave de ativacao + identificador)
- [ ] Banco SQLite limpo (ou deletar `events.db` para testar do zero)

---

## 1. Startup e Inicializacao

| # | Teste | Como validar | Resultado |
|---|-------|-------------|-----------|
| 1.1 | Agent inicia sem erro | Abrir o agent, verificar que o icone aparece na bandeja do sistema. Sem crash. | [ ] |
| 1.2 | Log de startup | Verificar `agent-tray.log` — deve ter "Worker iniciado" sem exceptions | [ ] |
| 1.3 | Config carregado | Log deve mostrar identificador mascarado e URLs configuradas | [ ] |

---

## 2. Coleta de Janela Ativa

| # | Teste | Como validar | Resultado |
|---|-------|-------------|-----------|
| 2.1 | Detecta troca de janela | Abrir 2 apps (ex: Chrome e Bloco de Notas), alternar entre eles. Log deve mostrar "Evento de janela finalizado" e "Novo evento de janela iniciado" | [ ] |
| 2.2 | Nome do processo correto | No log, verificar que `nomeProcesso` corresponde ao app ativo (ex: "chrome", "notepad") | [ ] |
| 2.3 | Titulo da janela correto | No log, verificar que `tituloJanela` mostra o titulo real da janela | [ ] |
| 2.4 | Duracao calculada | Ficar 30+ segundos em um app, trocar. O evento finalizado deve ter `duracaoSegundos >= 30` | [ ] |

---

## 3. Deteccao de Ociosidade

| # | Teste | Como validar | Resultado |
|---|-------|-------------|-----------|
| 3.1 | Detecta idle | Nao mexer no mouse/teclado por 60+ segundos. Log deve mostrar transicao para "Ausente" | [ ] |
| 3.2 | Retorno de idle | Apos ficar idle, mexer o mouse. Log deve mostrar retorno para "Online" | [ ] |
| 3.3 | Evento de ociosidade persistido | Verificar no `events.db` (tabela Events) que existe um registro com EventType contendo idle/ociosidade | [ ] |

---

## 4. Eventos de Sessao

| # | Teste | Como validar | Resultado |
|---|-------|-------------|-----------|
| 4.1 | Detecta LOCK | Bloquear a tela (Win+L). Log deve mostrar evento LOCK | [ ] |
| 4.2 | Detecta UNLOCK | Desbloquear a tela. Log deve mostrar evento UNLOCK | [ ] |
| 4.3 | Flush no LOCK | Ao bloquear, verificar que o UploadWorker acorda e tenta enviar eventos pendentes | [ ] |

---

## 5. Heartbeat

| # | Teste | Como validar | Resultado |
|---|-------|-------------|-----------|
| 5.1 | Heartbeat logado | Aguardar 60 segundos. Log deve mostrar "Heartbeat AthenaAgent" | [ ] |
| 5.2 | Heartbeat no buffer | Verificar no `events.db` que existe registro com EventType = "HeartbeatEvent" | [ ] |
| 5.3 | Heartbeat enviado | Apos um ciclo de upload (5 min), verificar que HeartbeatEvent foi marcado como Uploaded = 1 | [ ] |

---

## 6. Buffer e Upload de Eventos

| # | Teste | Como validar | Resultado |
|---|-------|-------------|-----------|
| 6.1 | Eventos persistidos no SQLite | Apos interagir por 1-2 minutos, verificar `events.db` tabela Events — deve ter registros com Uploaded = 0 | [ ] |
| 6.2 | Upload em batch | Aguardar 5 minutos (IntervaloUploadSegundos). Verificar log "Upload concluido" ou similar. Eventos devem mudar para Uploaded = 1 | [ ] |
| 6.3 | Retry em falha | Desligar o backend, aguardar um ciclo de upload. Log deve mostrar tentativa + retry com backoff. Religar o backend, proximo ciclo deve enviar com sucesso | [ ] |
| 6.4 | Cleanup de eventos velhos | Verificar que nao existem eventos com Uploaded = 0 e OccurredAt > 7 dias (rode o agent por um tempo ou insira dados de teste) | [ ] |

---

## 7. Transicao de Status

| # | Teste | Como validar | Resultado |
|---|-------|-------------|-----------|
| 7.1 | Online ao iniciar | Apos startup, status deve ser "Online" no log | [ ] |
| 7.2 | Ausente apos idle | Nao mexer por 10+ minutos. Status deve transicionar para "Ausente" | [ ] |
| 7.3 | Inativo apos 30 min | Nao mexer por 30+ minutos. Status deve transicionar para "Inativo" | [ ] |
| 7.4 | Retorno para Online | Mexer mouse/teclado apos idle. Deve voltar para "Online" | [ ] |

---

## 8. Corte Diario (DailyBoundary)

| # | Teste | Como validar | Resultado |
|---|-------|-------------|-----------|
| 8.1 | Evento de corte | Manter o agent rodando na virada do dia (23:59 → 00:00). Log deve mostrar "Corte diario" com eventos de encerramento e reinicio | [ ] |
| 8.2 | Eventos bufferizados | Verificar no `events.db` que os eventos StatusTransitionEvent do corte foram persistidos | [ ] |

---

## 9. Auto-Update

| # | Teste | Como validar | Resultado |
|---|-------|-------------|-----------|
| 9.1 | Check no startup | Ao iniciar, log deve mostrar verificacao de atualizacao (GET /api/agente/atualizacoes/verificar) | [ ] |
| 9.2 | Sem update disponivel | Se nao ha versao nova, log deve mostrar "Nenhuma atualizacao disponivel" e continuar normalmente | [ ] |
| 9.3 | Auto-apply silencioso | Publicar versao nova no srv-admin. Reiniciar o agent (ou aguardar polling 6h). Deve baixar, validar checksum, criar backup e aplicar sem pedir confirmacao | [ ] |
| 9.4 | Polling periodico | Manter o agent aberto por 6+ horas. Verificar no log que houve re-verificacao | [ ] |
| 9.5 | Rollback em falha | Simular falha: renomear um DLL essencial no ZIP antes de publicar. O agent deve detectar falha e restaurar backup no proximo startup | [ ] |
| 9.6 | Telemetria de resultado | Apos update (sucesso ou falha), verificar no log que houve POST para /api/agente/atualizacoes/resultado | [ ] |
| 9.7 | Watchdog | Simular crash pos-update: iniciar agent, matar o processo em menos de 60s. No proximo startup, verificar que o watchdog detectou e fez rollback | [ ] |

---

## 10. Resiliencia

| # | Teste | Como validar | Resultado |
|---|-------|-------------|-----------|
| 10.1 | Agent sem rede | Desconectar a internet. O agent deve continuar coletando eventos localmente no SQLite sem crash | [ ] |
| 10.2 | Reconexao | Reconectar a internet. No proximo ciclo de upload, eventos pendentes devem ser enviados | [ ] |
| 10.3 | Matar processo | Matar o processo via Task Manager. Ao reiniciar, deve funcionar normalmente sem perda de config | [ ] |
| 10.4 | Eventos nao duplicados | Verificar no backend que os eventos enviados nao estao duplicados (mesmo ID/timestamp) | [ ] |

---

## Checklist pos-teste

- [ ] Todos os testes acima passaram
- [ ] Nenhuma exception nao tratada nos logs
- [ ] Memoria estavel (agent nao cresce indefinidamente no Task Manager)
- [ ] CPU idle < 1% quando nao ha atividade
- [ ] SQLite `events.db` nao cresce indefinidamente (cleanup funciona)
- [ ] Backend recebeu todos os tipos de eventos: WindowActivity, Session, StatusTransition, Heartbeat, Idle
