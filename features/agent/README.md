# Feature: Agent Desktop (manager-srv-agent)
> @Tony | TL — decisoes tecnicas
> Status: Em auditoria e refatoracao

---

## Objetivo

Agente desktop (Windows/macOS) que coleta eventos de atividade do colaborador e envia em batch para o backend. Monitora: janelas ativas, idle, sessoes (lock/unlock/login/logout), heartbeat e input (teclado/mouse).

## Stack

- **Linguagem:** C# .NET 8.0+ (suporte .NET 10.0 Windows)
- **Tipo:** System Tray Application
- **Storage local:** SQLite (buffer de eventos)
- **Criptografia:** Windows DPAPI para chave de ativacao
- **Logging:** Serilog (rotacao diaria, 7 dias)

## Repositorio

`/Agente/manager-srv-agent/`

## Contexto detalhado

-> `tecnico.md` — decisoes tecnicas e auditoria
-> `adr/` — decisoes de arquitetura
