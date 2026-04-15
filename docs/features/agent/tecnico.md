# Tecnico — Agent Desktop
> Dominio: @Tony | TL
> Auditoria: 2026-03-26

---

## Auditoria de Coleta de Eventos (2026-03-26)

### Issues encontradas

#### CRITICOS (quebram funcionalidade ou seguranca)

| # | Issue | Arquivo | Status |
|---|-------|---------|--------|
| C1 | Heartbeat NAO enviado ao servidor — EventoSinalVida logado mas nunca adicionado ao buffer | `Worker.cs:435-450` | Planejado |
| C2 | Input hooks nunca liberados — SetWindowsHookEx sem Dispose no Worker | `InputActivityTracker.cs:46`, `Worker.cs:142` | Planejado |
| C3 | Contadores de input sem thread-safety — race condition em _keyPressCount/_mouseClickCount | `InputActivityTracker.cs:58,75,113` | Planejado |
| C4 | DPAPI fallback para plaintext — ChaveAtivacao salva sem criptografia se DPAPI falha | `ConfigManager.cs:138` | Planejado |

#### ALTOS (funcionalidade degradada)

| # | Issue | Arquivo | Status |
|---|-------|---------|--------|
| A1 | SessionMonitor nunca disposed — event listener leak em crash | `Worker.cs:134`, `SessionMonitor.cs:28` | Planejado |
| A2 | Falha silenciosa apos 3 retries — eventos ficam Uploaded=0 para sempre | `HttpEventUploader.cs:284-295` | Planejado |
| A3 | Auth header inconsistente — Bearer vs sem Bearer entre uploaders | `HttpEventUploader.cs:469` vs `ImmediateEventUploader.cs:111` | Planejado |
| A4 | Fila SQLite sem limite — sem max size nem cleanup automatico | `SqliteEventBuffer.cs` | Planejado |

#### MEDIOS (qualidade)

| # | Issue | Arquivo | Status |
|---|-------|---------|--------|
| M1 | Eventos de boundary diario nao enviados — TODO no DailyBoundaryWorker | `DailyBoundaryWorker.cs:60` | Planejado |
| M2 | MarkAsSentAsync ineficiente — UPDATE por evento em vez de batch | `SqliteEventBuffer.cs:240-261` | Planejado |
| M3 | Sem compressao de payload — JSON cru via HTTP | `HttpEventUploader.cs` | Backlog |

---

## Pontos positivos identificados

- Mascaramento de identificadores nos logs (MaskIdentifier)
- SQL parametrizado no SQLite (previne injection)
- Locks corretos no ConfigManager e AgentStateTracker
- Serilog com rotacao de arquivos
- Cache de IP por 5 minutos
- SQLite WAL mode + synchronous NORMAL para performance

---

## Auditoria de Auto-Update (2026-03-26)

### Issues encontradas

| # | Severidade | Issue | Status |
|---|-----------|-------|--------|
| U1 | CRITICAL | Sem code signing nos binarios — apenas checksum SHA-256 | Backlog |
| U2 | CRITICAL | Sem certificate pinning — MITM possivel em proxy TLS | Backlog |
| U3 | HIGH | Sem watchdog — crash pos-update nao detectado | Planejado |
| U4 | HIGH | Check apenas no startup — sem polling periodico | Planejado |
| U5 | HIGH | Auto-apply sem confirmacao — agente deve atualizar silenciosamente | Planejado |
| U6 | HIGH | Sem telemetria de update — backend nao sabe resultado | Planejado |
| U7 | MEDIUM | Pre-signed URL expira em 15 min — retry falha com 403 | Planejado |

### Pontos positivos do auto-update

- Checksum SHA-256 valida integridade do download
- Validacao do ZIP (verifica EXEs essenciais)
- Backup automatico com limite de 3 versoes
- Rollback via flags no proximo startup
- Fallback Task Scheduler → UAC elevation
- Retry com exponential backoff (3 tentativas)
- Preservacao de config files

---

## Evolucoes de Inteligencia (2026-03-27)

Decisao: NAO classificar apps como produtivo/improdutivo (ADR pendente).
Motivo: mesma app pode ser produtiva para um cargo e improdutiva para outro. A IA (iManager Score) ja analisa padroes sem julgamento por app individual.

### Itens aprovados

| # | Evolucao | Onde muda | Responsavel | Status |
|---|----------|-----------|-------------|--------|
| INT-2 | Extracao de dominio dos titulos de aba do navegador | Backend (parser) + Frontend (top apps) | @Vision + @Peter | Planejado |
| INT-3 | Tempo de foco continuo (deep work) — sessoes >= 5 min sem interrupcao | Backend (calculo) + Frontend (KPI) | @Thor + @Peter | Planejado |
| INT-4 | Horario de inicio/fim da jornada explicito nos DTOs | Backend (DTO) + Frontend (card) | @Vision + @Peter | Planejado |
| INT-5 | Numero de monitores no heartbeat (GetSystemMetrics) | Agent (P/Invoke) + Backend (armazenar) | @Thor | Planejado |
| INT-6 | Status de rede no heartbeat (NetworkInterface) | Agent + Backend | @Thor | Planejado |

---

## Melhorias de Instalacao e Deploy (2026-03-27)

### Itens aprovados

| # | Melhoria | O que resolve | Status |
|---|----------|---------------|--------|
| INST-3 | Instalador GUI via Inno Setup | UX intimidadora do CMD para o cliente | Planejado |
| INST-4 | Download do instalador via portal | Build manual com PS1 + colar UUID | Planejado |
| INST-1 | Flag --silent no instalador | Pre-requisito para deploy em massa | Planejado |
| INST-2 | Script de deploy em massa (CSV + WinRM) | 100 maquinas = 100 instalacoes manuais | Planejado |

### Ordem de execucao

1. INST-3 + INST-4 (paralelo) — instalador bonito + download via portal
2. INST-1 + INST-2 (sequencial) — silent mode + deploy em massa

### Arquitetura

**INST-3 (Inno Setup):** Script `.iss` que gera `.exe` com UI visual. Custom page para identificador. Encapsula a logica do `instalar.ps1`. Compilado no build via `ISCC.exe`.

**INST-4 (Download via portal):** O ZIP ja e enviado ao srv-admin na API de atualizacao. Novo endpoint `GET /api/agente/instalador/download` retorna o ZIP mais recente. Frontend: botao "Baixar instalador" na tela de Organizacao.

**INST-1 (Silent):** Inno Setup nativamente suporta `/SILENT`. Custom parameter `/identificador=` passado via linha de comando.

**INST-2 (Deploy em massa):** Script PS1 que le CSV (maquina,identificador), copia instalador via `\\maquina\C$\Temp\`, executa remotamente via `Invoke-Command -ComputerName`, gera relatorio.
