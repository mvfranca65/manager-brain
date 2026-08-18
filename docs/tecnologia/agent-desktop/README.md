> **STATUS:** ATIVO
> **DATA:** 2026-06-11
> **DONO:** @Tony (TL)
> **REVISADO POR:** @Tony

# Manager Agent — Documentacao

O Manager Agent e o componente desktop do Manager. Roda em background nas maquinas dos colaboradores e captura dados de atividade (janela ativa, ociosidade, sessoes) para envio ao backend.

---

## Versao Atual: V2.0.0

**Plataforma:** Windows 10/11 64-bit (macOS nao suportado na V2)
**Stack:** C# .NET 8, Windows Service (SCM), Named Pipes, SQLite WAL, Serilog, Inno Setup
**Repositorio:** `manager-srv-agent` (GitHub: `mvfranca65/manager-srv-agent`)

### Arquitetura em dois processos

| Processo | Executavel | Contexto | Responsabilidade |
|----------|-----------|----------|-----------------|
| **ManagerAgent.Service** | `ManagerAgent.Service.exe` | Session 0, SYSTEM | Servico Windows: gerencia sessoes, buffer SQLite, upload HTTP, auto-update |
| **ManagerAgent.SessionWorker** | `ManagerAgent.SessionWorker.exe` | Sessao interativa do usuario | Captura janela ativa, ociosidade, heartbeat, bandeja do sistema |

Comunicacao entre processos: Named Pipe `\\.\pipe\ManagerAgent_{sessionId}` com ACL restrita.

### O que o agent captura

- Janela ativa (processo + titulo) a cada 5 segundos
- Eventos de sessao (logon, logoff, lock, unlock, RDP)
- Ociosidade (sem input de teclado/mouse por 5+ minutos)
- Heartbeat a cada 60 segundos
- Deteccao de videochamada (uso de camera/microfone)

### O que o agent NAO faz (por decisao de produto — LGPD 2026-04-15)

- Nao captura tela (sem screenshots)
- Nao registra teclas digitadas (sem keylogger)
- Nao acessa camera, microfone ou audio
- Nao le conteudo de emails, mensagens ou arquivos

---

## Documentos

| Doc | Descricao | Audiencia |
|-----|-----------|-----------|
| `MANUAL-COMPLETO.md` | Referencia completa: visao geral, arquitetura, instalacao, configuracao, uso diario, troubleshooting, FAQ. Inclui secoes de build por empresa e geracao de release. | Dev + Suporte + TI |
| `GUIA-RELEASE.md` | Passo a passo operacional para criar e publicar nova versao. Inclui checklist pre-release. | Dev + @Vision |
| `PLANO-TESTES-REGRESSIVOS.md` | Roteiro completo de testes manuais antes de cada release. Inclui scripts PowerShell uteis para diagnostico durante os testes. | @Natasha + Dev |

---

## Instalacao rapida

```powershell
# Verificar se instalou corretamente
Get-Service ManagerAgent
Get-Process -Name "ManagerAgent.SessionWorker"

# Health check completo
& "C:\Program Files\ManagerAgent\scripts\health-check.ps1"
```

---

## Caminhos importantes (producao)

| Item | Caminho |
|------|---------|
| Binarios | `C:\Program Files\ManagerAgent\` |
| Configuracao | `C:\ProgramData\ManagerAgent\config.json` |
| Buffer SQLite | `C:\ProgramData\ManagerAgent\buffer.db` |
| Logs Service | `C:\ProgramData\ManagerAgent\logs\service-YYYYMMDD.log` |
| Logs Worker | `C:\ProgramData\ManagerAgent\logs\worker-YYYYMMDD.log` |
| Scripts diagnostico | `C:\Program Files\ManagerAgent\scripts\` |

---

## Backlog de evolucao (itens identificados, nao priorizados)

Os itens abaixo foram catalogados para melhoria futura. Itens marcados `[x]` ja foram implementados.

### Seguranca

- [x] DPAPI para `chaveAtivacaoEmpresa` no `config.json`
- [ ] Filtrar titulos de janela com dados sensiveis (blocklist de processos sensíveis, ex: `keepass`)
- [ ] Validacao de formato no `identificadorColaborador` (regex antes de salvar)
- [ ] Certificate pinning para endpoints do backend (protecao contra MITM em redes corporativas)

### Robustez

- [x] `ParseVersion` seguro com `int.TryParse` em `UpdateChecker.cs`
- [x] Timeout explícito de 30s nos `HttpClient`s
- [x] `CommandTimeout = 30` em todos os comandos SQLite
- [ ] Limite de tamanho/retencao no banco SQLite (job de delecao de eventos > 7 dias ou > 50 MB)
- [ ] Timeout configuravel para o loop do `AgentLinkService`

### Qualidade

- [ ] Projeto `ManagerAgent.Tests` com testes unitarios (xUnit + NSubstitute) — prioridade: `ConfigManager`, `UpdateChecker`, `SqliteEventBuffer`
- [ ] Interface `IActiveWindowProvider` para desacoplar P/Invoke e habilitar testes
- [ ] Centralizar constantes de timing em `appsettings.json` (timeouts hardcoded em `UpdateApplier.cs`, etc.)
- [ ] Extrair logica de negocio dos `BackgroundService` (`UploadWorker`, `UpdateCheckerWorker`)

### Funcionalidades futuras

- [ ] "Exportar diagnostico" no menu do tray (ZIP com logs + config sanitizado)
- [ ] Corrigir race condition no cache de IP local em `Worker.cs` (lock no `_cachedLocalIp`)

---

## V1 vs V2

| Aspecto | V1 (legado) | V2 (atual) |
|---------|------------|-----------|
| Processo principal | `AthenaAgent.Tray.exe` (usuario) | `ManagerAgent.Service.exe` (SYSTEM) + `ManagerAgent.SessionWorker.exe` (usuario) |
| Inicializacao | Atalho na pasta Startup do usuario | Windows Service gerenciado pelo SCM |
| Multi-sessao | Nao suportado | Suportado (um SessionWorker por sessao ativa) |
| Auto-update | Manual | Automatico (Plano A/B/C, verifica a cada 6h) |
| Buffer local | SQLite basico | SQLite WAL com timeout, retry e retencao 7 dias |
| Migracao V1→V2 | — | Automatica via `MigrationV1ToV2.cs` no instalador |

O codigo V1 (`ManagerAgent.Tray`) esta no repositorio apenas como referencia historica.
