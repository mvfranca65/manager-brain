> **STATUS:** ATIVO
> **DATA:** 2026-06-11
> **DONO:** @Tony (TL)
> **REVISADO POR:** @Tony

# Manual Completo - Manager Agent

**Versao:** 2.0.0
**Data:** 2026-06-11
**Sistema:** Windows 10/11 (64-bit)

---

## Indice

1. [Visao Geral](#visao-geral)
2. [Como Funciona](#como-funciona)
3. [O Que o Agente Captura](#o-que-o-agente-captura)
4. [Arquitetura](#arquitetura)
5. [Instalacao](#instalacao)
6. [Configuracao](#configuracao)
7. [Uso Diario](#uso-diario)
8. [Scripts Diagnosticos](#scripts-diagnosticos)
9. [Desenvolvimento](#desenvolvimento)
10. [Geracao de Nova Versao](#geracao-de-nova-versao)
11. [Desinstalacao](#desinstalacao)
12. [Troubleshooting](#troubleshooting)
13. [FAQ](#faq)

---

## Visao Geral

O **Manager Agent** e um sistema de monitoramento de atividades para Windows que coleta informacoes sobre o uso do computador para analise de produtividade. Na versao 2.0, a arquitetura foi completamente reescrita: em vez de um unico processo de bandeja do sistema rodando como usuario, o agente agora opera como um **Windows Service** gerenciado pelo SCM (Service Control Manager), com processos auxiliares dedicados por sessao de usuario.

Essa nova arquitetura torna o agente mais confiavel, suporta multiplas sessoes simultaneas (Terminal Server, Fast User Switching) e oferece atualizacoes automaticas sem intervencao humana.

### Caracteristicas Principais

- **Windows Service (SYSTEM)** - Servico gerenciado pelo SCM, inicia automaticamente com o Windows, independente de login
- **SessionWorker por usuario** - Processo leve lancado em cada sessao ativa, responsavel pela captura
- **Captura de Janelas Ativas** - Detecta aplicativo e titulo da janela em uso via Win32 API
- **Monitoramento de Sessao** - Rastreia logon, logoff, lock, unlock e conexoes remotas
- **Heartbeats** - Sinal de vida periodico enviado ao servidor com versao e estado
- **Deteccao de Idle** - Identifica quando o usuario esta ausente (mouse/teclado)
- **Deteccao de Reunioes** - Detecta videochamadas via uso de camera/microfone
- **Buffer Local SQLite** - Armazena eventos em modo WAL em `C:\ProgramData\ManagerAgent\buffer.db`
- **Named Pipes** - Comunicacao segura entre SessionWorker e Service via pipe nomeado
- **Atualizacao Automatica** - Sistema de auto-update com fallback Plano A/B/C
- **Watchdog** - Monitoramento de saude do SessionWorker com reinicio automatico
- **Logs Estruturados** - Serilog com rotacao diaria, retencao de 7 dias, 50 MB por arquivo
- **Menu Diagnostico** - Scripts PowerShell integrados para troubleshooting

### Quando Usar

- Monitoramento de produtividade corporativa
- Analise de uso de aplicacoes
- Rastreamento de tempo em projetos
- Auditoria de atividades de colaboradores
- Gestao de equipes remotas e ambientes de Terminal Server

### Quando NAO Usar

- Computadores pessoais sem consentimento do usuario
- Ambientes que requerem privacidade total
- Sistemas sem conectividade com servidor central

---

## Como Funciona

### Visao do Fluxo Operacional

```
+-------------------------------------------------------------+
|  1. INICIALIZACAO DO SERVICO                                |
|  O Windows inicia ManagerAgent.Service.exe como SYSTEM     |
|  na Sessao 0 (sem interface grafica)                        |
|  O SCM garante reinicio automatico em caso de falha        |
+-------------------------------------------------------------+
                            |
                            v
+-------------------------------------------------------------+
|  2. DETECCAO DE SESSAO DE USUARIO                          |
|  Service recebe notificacao WTS de nova sessao ativa       |
|  Lanca ManagerAgent.SessionWorker.exe na sessao do         |
|  usuario correspondente                                     |
+-------------------------------------------------------------+
                            |
                            v
+-------------------------------------------------------------+
|  3. COLETA DE DADOS (SessionWorker, por sessao)            |
|                                                             |
|  [CaptureWorker] - A cada 5 segundos:                      |
|    Captura janela ativa (processo + titulo)                 |
|    Detecta atividade do usuario (mouse/teclado)             |
|    Detecta idle (sem entrada por 5+ minutos)                |
|                                                             |
|  [SessionMonitor] - Eventos de sistema:                     |
|    Logon / Logoff                                           |
|    Lock / Unlock (Win+L)                                    |
|    Conexao / desconexao remota                              |
|                                                             |
|  [HeartbeatWorker] - A cada 60 segundos:                   |
|    Versao do agente, uptime, eventos pendentes             |
+-------------------------------------------------------------+
                            |
                            v
+-------------------------------------------------------------+
|  4. ENVIO VIA NAMED PIPE                                   |
|  SessionWorker envia eventos ao Service via pipe nomeado   |
|  Pipe: \\.\pipe\ManagerAgent_{sessionId}                   |
|  Mensagens JSON com envelope de tipo                        |
|  AutonomousBuffer mantem eventos se pipe estiver inativo   |
+-------------------------------------------------------------+
                            |
                            v
+-------------------------------------------------------------+
|  5. ARMAZENAMENTO LOCAL (Service)                          |
|  Service grava eventos no buffer SQLite (modo WAL)         |
|  Localizacao: C:\ProgramData\ManagerAgent\buffer.db        |
+-------------------------------------------------------------+
                            |
                            v
+-------------------------------------------------------------+
|  6. UPLOAD AO SERVIDOR (UploadWorker)                      |
|  A cada 30 segundos:                                        |
|    Le eventos pendentes do buffer                           |
|    Envia via HTTP POST para backend                         |
|    Marca eventos como enviados                              |
|    Remove eventos com mais de 7 dias                        |
+-------------------------------------------------------------+
                            |
                            v
+-------------------------------------------------------------+
|  7. AUTO-UPDATE (UpdateCheckerWorker)                      |
|  A cada 6 horas (com jitter):                              |
|    Verifica nova versao no servidor                         |
|    Baixa ZIP, valida SHA-256                                |
|    Aplica via Plano A (schtasks PS), B (schtasks BAT)      |
|    ou C (renomeio de arquivo em processo)                   |
+-------------------------------------------------------------+
```

### Tecnologias Utilizadas

| Tecnologia | Finalidade |
|------------|-----------|
| **.NET 8.0** | Framework principal |
| **Windows Service (SCM)** | Gerenciamento do ciclo de vida do servico |
| **Windows Forms** | Interface do System Tray (SessionWorker) |
| **SQLite (WAL)** | Buffer local de eventos |
| **Serilog** | Logging estruturado com enrichers |
| **Named Pipes** | Comunicacao IPC entre Service e SessionWorker |
| **DPAPI** | Criptografia de campos sensiveis no config.json |
| **WTS API** | Notificacoes de sessao Windows |
| **Win32 API (user32.dll)** | Captura de janela ativa e idle detection |
| **Microsoft.Extensions.Hosting** | Background workers |

---

## O Que o Agente Captura

### 1. Eventos de Janela Ativa

**Frequencia:** A cada 5 segundos (apenas se houver mudanca)

**Dados Capturados:**
```json
{
  "tipo": "janela_ativa",
  "timestamp": "2026-05-08T14:30:45.1234567Z",
  "instalacaoId": "abc123-def456-ghi789",
  "usuarioWindows": "joao.silva",
  "aplicativo": "chrome.exe",
  "titulo": "GitHub - Google Chrome",
  "duracao": 5000,
  "idle": false
}
```

**Exemplos:**
- Navegador: `chrome.exe` com titulo `"GitHub - Google Chrome"`
- Editor: `Code.exe` com titulo `"Worker.cs - Visual Studio Code"`
- Office: `WINWORD.EXE` com titulo `"Relatorio Mensal.docx - Word"`
- Comunicacao: `Teams.exe` com titulo `"Chat | Microsoft Teams"`

**O que NAO e capturado:**
- Conteudo digitado (sem keylogging)
- Screenshots da tela
- Conteudo de documentos ou formularios
- Senhas ou dados sensiveis

### 2. Eventos de Sessao

**Frequencia:** Ao ocorrer (eventos de sistema via WTS e SystemEvents)

**Tipos de Evento:**
- `SessionLogon` - Usuario fez login
- `SessionLogoff` - Usuario fez logout
- `SessionLock` - Usuario bloqueou a sessao (Win+L)
- `SessionUnlock` - Usuario desbloqueou a sessao
- `SessionRemoteConnect` - Conexao remota iniciada (RDP)
- `SessionRemoteDisconnect` - Conexao remota encerrada

**Dados Capturados:**
```json
{
  "tipo": "sessao",
  "timestamp": "2026-05-08T08:00:00.0000000Z",
  "instalacaoId": "abc123-def456-ghi789",
  "tipoEvento": "SessionLogon",
  "usuarioWindows": "joao.silva",
  "maquina": "DESKTOP-ABC123"
}
```

### 3. Heartbeats (Sinal de Vida)

**Frequencia:** A cada 60 segundos

**Dados Capturados:**
```json
{
  "tipo": "heartbeat",
  "timestamp": "2026-05-08T14:31:00.0000000Z",
  "instalacaoId": "abc123-def456-ghi789",
  "versaoAgente": "2.0.0",
  "uptime": 3600,
  "eventosPendentes": 12
}
```

### 4. Deteccao de Idle

**Logica:**
- Usuario considerado **IDLE** se nao houver atividade de mouse ou teclado por 5 ou mais minutos, ou se a sessao estiver bloqueada

**Comportamento:**
- Quando IDLE: para de gerar eventos de janela ativa
- Quando retorna: gera evento de retorno de atividade

### 5. Deteccao de Reunioes

O agente detecta video chamadas por meio do monitoramento do uso de camera e microfone pelo sistema operacional. Quando detectada, a informacao e incluida nos eventos de janela ativa para identificacao de blocos de reuniao.

---

## Arquitetura

### Projetos da Solucao

#### 1. ManagerAgent.Service

Servico Windows rodando como SYSTEM na Sessao 0. Responsavel por:

- Gerenciar o ciclo de vida das sessoes via notificacoes WTS (Windows Terminal Services)
- Manter o buffer SQLite em modo WAL em `C:\ProgramData\ManagerAgent\buffer.db`
- Fazer upload de eventos ao backend via HTTP em lotes
- Atuar como servidor de named pipe (`\\.\pipe\ManagerAgent_{sessionId}`) para receber eventos dos SessionWorkers
- Executar o sistema de auto-update com Plano A/B/C
- Monitorar a saude dos SessionWorkers via WorkerWatchdog
- Gerenciar a vinculacao do agente com o backend

**Background services:**
- `ManagerAgentService` - Nucleo do servico, gerencia sessoes
- `UploadWorker` - Envia eventos ao backend em lotes
- `WorkerWatchdog` - Monitora e reinicia SessionWorkers com falha
- `UpdateCheckerWorker` - Verifica e aplica novas versoes

#### 2. ManagerAgent.SessionWorker

Processo por usuario, lancado pelo Service em cada sessao ativa do Windows. Responsavel por:

- Capturar janelas ativas via Win32 API (`GetForegroundWindow`, `GetWindowText`)
- Detectar idle via `GetLastInputInfo`
- Monitorar eventos de sessao via `SystemEvents`
- Gerar heartbeats periodicos
- Exibir icone na bandeja do sistema com menu de ferramentas diagnosticas
- Enviar eventos ao Service via named pipe (cliente de pipe nomeado)
- Manter `AutonomousBuffer` local quando a conexao com o pipe estiver inativa
- Gerenciar transicoes de dia (midnight boundary worker)

#### 3. ManagerAgent.Shared

Biblioteca compartilhada entre Service e SessionWorker. Contem:

- `ConfigPaths` - Todos os caminhos do sistema de arquivos em `C:\ProgramData\ManagerAgent\`
- `ConfigManager` - Leitura e escrita de `config.json` com criptografia DPAPI para campos sensiveis (`DeviceToken`)
- DTOs do protocolo de pipe (`PipeMessage`, enum `PipeMessageType`)
- Enrichers do Serilog (`SourceEnricher` para prefixo "Service" ou "Worker" nos logs)
- `NativeMethods` - Declaracoes P/Invoke para APIs Win32

#### 4. ManagerAgent.Installer

Logica de instalacao, desinstalacao e migracao. Usado pelo instalador Inno Setup. Contem:

- `InstallActions` - Registro do servico no SCM, criacao de diretorios, permissoes
- `UninstallActions` - Remocao do servico, limpeza de arquivos
- `MigrationV1ToV2` - Migracao automatica do agente V1 (tray app) para a arquitetura V2

#### 5. ManagerAgent.Configurator

CLI para configuracao inicial do agente. Utilizado durante a instalacao ou para reconfigurar o dispositivo apos a instalacao.

#### 6. ManagerAgent.Tray (Legado)

Codigo V1 mantido para referencia historica. Nao e mais usado em producao. O icone da bandeja do sistema e agora responsabilidade do `ManagerAgent.SessionWorker`.

### Estrutura de Diretorios no Windows

```
C:\Program Files\ManagerAgent\          # Instalacao
|-- ManagerAgent.Service.exe            # Executavel do Windows Service
|-- ManagerAgent.SessionWorker.exe      # Executavel do worker por sessao
|-- ManagerAgent.Configurator.exe       # CLI de configuracao
|-- *.dll                               # Dependencias
+-- scripts\                            # Scripts PowerShell de diagnostico
    |-- health-check.ps1
    |-- monitorar-logs.ps1
    |-- coletar-diagnostico.ps1
    |-- limpar-reset.ps1
    +-- monitorar-performance.ps1

C:\ProgramData\ManagerAgent\            # Dados e configuracoes
|-- config.json                         # Configuracao do agente (campos criptografados via DPAPI)
|-- buffer.db                           # Buffer SQLite em modo WAL
|-- updates\                            # Pacotes de atualizacao baixados
+-- logs\                               # Arquivos de log
    |-- service-YYYYMMDD.log            # Logs do Service
    +-- worker-YYYYMMDD.log             # Logs do SessionWorker
```

### Fluxo de Dados

```
SessionWorker (Sessao 1) --+
SessionWorker (Sessao 2) --+-- Named Pipes --> Service (Sessao 0) --> HTTP POST --> Backend
SessionWorker (Sessao N) --+    msgs JSON       Buffer SQLite          Upload em lotes
```

### Protocolo de Named Pipe

- **Nome do pipe:** `ManagerAgent_{sessionId}`
- **Formato:** mensagens JSON com envelope de tipo: `{ "type": "EventBatch", "payload": {...} }`
- **Tipos de mensagem:** `EventBatch`, `Heartbeat`, `SessionEvent`, `Config`, `ConfigResponse`, `Shutdown`, `Error`
- **ACL:** restrita ao usuario da sessao correspondente e ao SYSTEM

### Sistema de Auto-Update

O `UpdateCheckerWorker` verifica novas versoes a cada 6 horas com jitter aleatorio para evitar sobrecarga no servidor. O processo de atualizacao funciona assim:

1. Baixa o pacote ZIP da nova versao
2. Verifica integridade via hash SHA-256
3. Aplica a atualizacao via fallback em tres niveis:
   - **Plano A:** Script PowerShell agendado via `schtasks` (mais confiavel, executa como SYSTEM)
   - **Plano B:** Script BAT agendado via `schtasks` (fallback se PowerShell estiver restrito)
   - **Plano C:** Renomeio de arquivos em processo (ultimo recurso, renomeia binarios em execucao para `.old`)

**Arquivos de estado:**
- `update-in-progress.flag` - Indica que uma atualizacao esta em andamento
- `update-success.flag` - Indica que a atualizacao foi bem-sucedida
- `update-failed.flag` - Indica que a atualizacao falhou

Um periodo de cooldown impede loops infinitos de reinicio em caso de falha consecutiva de atualizacoes.

---

## Instalacao

### Requisitos

- Windows 10 ou Windows 11 (64-bit)
- .NET 8.0 Runtime (incluido no instalador)
- Acesso de administrador para instalar o servico
- Conectividade de rede com o servidor backend

### Instalacao via Inno Setup (Recomendado)

1. **Execute o instalador:**
   ```
   ManagerAgent-Setup-2.0.0.exe
   ```

2. **Siga o assistente de instalacao:**
   - Aceite o contrato de licenca
   - Escolha o diretorio de instalacao (padrao: `C:\Program Files\ManagerAgent`)
   - Informe a URL do servidor e o identificador do colaborador
   - Confirme a instalacao

3. **O instalador ira automaticamente:**
   - Copiar os binarios para `C:\Program Files\ManagerAgent\`
   - Criar o diretorio de dados em `C:\ProgramData\ManagerAgent\`
   - Registrar `ManagerAgent.Service` no SCM com inicio automatico
   - Configurar recuperacao automatica do servico (1s, 5s, 30s)
   - Iniciar o servico imediatamente
   - Migrar dados de uma instalacao V1, se existente

4. **Verificar instalacao:**
   ```powershell
   # Verificar se o servico esta rodando
   Get-Service -Name "ManagerAgent"

   # Deve mostrar: Status=Running, StartType=Automatic
   ```

### Migracao da V1 para V2

Se houver uma instalacao V1 (tray app com atalho na pasta Startup), o instalador executa automaticamente o `MigrationV1ToV2`:

- Remove o atalho de inicializacao do usuario em `AppData\Roaming\...\Startup\`
- Encerra o processo da tray V1 se estiver em execucao
- Preserva `config.json` e `buffer.db` existentes
- Instala o servico V2

Nenhuma acao manual e necessaria para a migracao.

### Instalacao via CLI (Ambientes Corporativos)

Para deploy silencioso via GPO ou script de provisionamento:

```powershell
# Instalacao silenciosa
.\ManagerAgent-Setup-2.0.0.exe /VERYSILENT /SUPPRESSMSGBOXES /NORESTART

# Configurar apos instalacao
"C:\Program Files\ManagerAgent\ManagerAgent.Configurator.exe" `
  --baseUrlAdmin "https://admin.imanagerportal.com" `
  --baseUrlEvents "https://api-events.imanagerportal.com" `
  --chaveEmpresa "UUID-da-empresa" `
  --identificador "matricula-ou-email"

# Iniciar servico
Start-Service -Name "ManagerAgent"
```

---

## Configuracao

### Arquivo: `C:\ProgramData\ManagerAgent\config.json`

```json
{
  "baseUrlAdmin": "https://admin.imanagerportal.com",
  "baseUrlEvents": "https://api-events.imanagerportal.com",
  "chaveAtivacaoEmpresa": "UUID-da-empresa",
  "identificadorColaborador": "matricula-ou-email",
  "instalacaoId": "guid-gerado-automaticamente",
  "deviceToken": "encrypted-via-DPAPI",
  "menuVisivel": true
}
```

### Descricao dos Parametros

| Parametro | Tipo | Descricao |
|-----------|------|-----------|
| `baseUrlAdmin` | string | URL do servidor de administracao (obrigatorio) |
| `baseUrlEvents` | string | URL do servidor de eventos (obrigatorio) |
| `chaveAtivacaoEmpresa` | string | Chave UUID da empresa no Manager (obrigatorio) |
| `identificadorColaborador` | string | Matricula ou e-mail do colaborador (obrigatorio) |
| `instalacaoId` | string | GUID gerado automaticamente na primeira instalacao |
| `deviceToken` | string | Token do dispositivo criptografado via DPAPI (gerenciado automaticamente) |
| `menuVisivel` | bool | Define se o menu de ferramentas e exibido no tray (padrao: true) |

### Seguranca da Configuracao

O campo `deviceToken` e criptografado usando DPAPI (Data Protection API) do Windows em nivel de maquina. Isso significa:

- O token nao pode ser lido em texto claro por outros processos ou usuarios
- O token e descriptografado apenas pelo Service (rodando como SYSTEM)
- Em caso de reinstalacao em outra maquina, e necessaria uma nova vinculacao

### Reconfigurando o Agente

```powershell
# 1. Parar o servico
Stop-Service -Name "ManagerAgent"

# 2. Reconfigurar via CLI
"C:\Program Files\ManagerAgent\ManagerAgent.Configurator.exe" `
  --chaveEmpresa "novo-UUID" `
  --identificador "novo-identificador"

# 3. Reiniciar o servico
Start-Service -Name "ManagerAgent"
```

Alternativamente, edite `config.json` diretamente (como administrador) e reinicie o servico. Nao edite manualmente o campo `deviceToken`.

---

## Uso Diario

### Icone na Bandeja do Sistema

O icone do Manager Agent aparece na area de notificacao (bandeja do sistema) para cada usuario com sessao ativa. O icone reflete o estado atual do agente:

- **Normal** - Monitorando ativamente
- **Cinza/Pausado** - Sessao em segundo plano ou idle prolongado
- **Alerta** - Aguardando vinculacao ou erro

### Menu do System Tray (clique direito)

```
iManager - vX.Y.Z
-----------------------------------------
Ferramentas  >
    Verificacao de Saude
    Logs em Tempo Real
    Exportar Diagnostico
    ─────────────────────
    Limpar Dados e Reiniciar
-----------------------------------------
Sobre
```

### Dialogo "Sobre"

Ao clicar em "Sobre", e exibida uma janela com:

- Versao atual do agente
- Usuario Windows da sessao corrente
- Estado atual do agente

### Estados Possiveis

| Estado | Descricao |
|--------|-----------|
| **Monitorando** | Agente ativo e coletando dados normalmente |
| **Sessao em segundo plano** | Sessao nao esta em primeiro plano (outro usuario ativo) |
| **Aguardando vinculacao** | Agente instalado mas ainda nao vinculado ao backend |
| **Erro** | Falha de comunicacao ou configuracao incorreta |
| **Atualizando...** | Download e aplicacao de nova versao em andamento |

### Acoes Comuns

#### Verificar Status do Servico
```powershell
Get-Service -Name "ManagerAgent"
```

#### Reiniciar o Servico
```powershell
Restart-Service -Name "ManagerAgent"
```

#### Ver Logs em Tempo Real
Clique direito no icone da bandeja -> Ferramentas -> Logs em Tempo Real

Ou via PowerShell:
```powershell
Get-Content "C:\ProgramData\ManagerAgent\logs\service-$(Get-Date -Format 'yyyyMMdd').log" -Wait -Tail 50
```

---

## Scripts Diagnosticos

Os scripts estao em `C:\Program Files\ManagerAgent\scripts\` e podem ser acessados pelo menu da bandeja ou executados diretamente.

### 1. Verificacao de Saude (health-check.ps1)

**Localizacao:** `scripts\health-check.ps1`

**O que faz:**
- Verifica se o servico `ManagerAgent` esta rodando
- Verifica se o processo `ManagerAgent.SessionWorker` esta ativo na sessao corrente
- Valida o arquivo `config.json`
- Testa conectividade HTTP com os servidores configurados
- Verifica integridade do buffer SQLite
- Analisa logs recentes em busca de erros criticos
- Gera score de saude de 0 a 100%

**Como usar:**
```powershell
cd "C:\Program Files\ManagerAgent"
.\scripts\health-check.ps1
```

**Output esperado:**
```
========================================
MANAGER AGENT - HEALTH CHECK v2.0
========================================

[OK]  Servico ManagerAgent rodando (State=Running)
[OK]  SessionWorker ativo na sessao atual
[OK]  config.json valido e completo
[OK]  Conectividade com manager-srv-admin
[OK]  Conectividade com manager-srv-events
[OK]  Buffer SQLite funcional (modo WAL)
[OK]  Sem erros criticos nos ultimos logs

Score: 100% (7/7 verificacoes passaram)
```

### 2. Logs em Tempo Real (monitorar-logs.ps1)

**Localizacao:** `scripts\monitorar-logs.ps1`

**O que faz:**
- Monitora arquivos de log em tempo real com `Get-Content -Wait`
- Coloriza saida: vermelho para erros, amarelo para warnings, verde para informacoes
- Exibe logs do Service e do SessionWorker simultaneamente

**Como usar:**
```powershell
.\scripts\monitorar-logs.ps1
```

### 3. Exportar Diagnostico (coletar-diagnostico.ps1)

**Localizacao:** `scripts\coletar-diagnostico.ps1`

**O que faz:**
- Coleta informacoes do sistema (versao do Windows, .NET, hardware)
- Copia `config.json` com campos sensiveis mascarados
- Copia os ultimos 3 arquivos de log (service e worker)
- Executa health-check e inclui o resultado
- Testa conectividade e inclui o resultado
- Compacta tudo em um ZIP para envio ao suporte

**Como usar:**
```powershell
.\scripts\coletar-diagnostico.ps1
```

**Output:** `diagnostico-ManagerAgent-YYYYMMDD-HHmmss.zip` na area de trabalho do usuario

### 4. Limpar Dados e Reiniciar (limpar-reset.ps1)

**Localizacao:** `scripts\limpar-reset.ps1`

**O que faz:**
- Para o servico `ManagerAgent`
- Remove `buffer.db` (eventos pendentes nao enviados serao perdidos)
- Opcionalmente limpa os arquivos de log
- Opcionalmente remove `config.json` (requer reconfiguracao completa)
- Reinicia o servico

**ATENCAO:** Operacao destrutiva. Quando acionado pelo menu da bandeja, solicita confirmacao explicita antes de prosseguir.

**Como usar:**
```powershell
.\scripts\limpar-reset.ps1
```

### 5. Monitorar Performance (monitorar-performance.ps1)

**Localizacao:** `scripts\monitorar-performance.ps1`

**O que faz:**
- Monitora uso de CPU e memoria do `ManagerAgent.Service` e `ManagerAgent.SessionWorker`
- Exibe contagem de threads e handles
- Alerta quando o consumo excede limiares esperados
- Detecta possiveis vazamentos de memoria ao longo do tempo

**Como usar:**
```powershell
.\scripts\monitorar-performance.ps1
```

---

## Desenvolvimento

### Setup do Ambiente de Desenvolvimento

```powershell
# 1. Clonar repositorio
git clone https://github.com/mvfranca65/manager-srv-agent.git
cd manager-srv-agent

# 2. Restaurar dependencias
dotnet restore

# 3. Compilar solucao completa
dotnet build

# 4. Para desenvolver o SessionWorker (interface visual):
# Abrir ManagerAgent.sln no Visual Studio
# Definir ManagerAgent.SessionWorker como projeto de inicializacao
# Pressionar F5
```

### Estrutura de Codigo

```
src/
|-- ManagerAgent.Service/               # Windows Service (SYSTEM, Sessao 0)
|   |-- Program.cs                      # Entry point, registro no SCM
|   |-- ManagerAgentService.cs          # Nucleo: gerencia sessoes e workers
|   |-- Workers/
|   |   |-- UploadWorker.cs             # Upload de eventos ao backend
|   |   |-- WorkerWatchdog.cs           # Monitora saude dos SessionWorkers
|   |   +-- UpdateCheckerWorker.cs      # Verifica e aplica atualizacoes
|   |-- Pipes/
|   |   +-- PipeServer.cs               # Servidor de named pipe por sessao
|   |-- Linking/
|   |   +-- AgentLinkService.cs         # Vinculacao com o backend
|   +-- Update/
|       +-- UpdateApplier.cs            # Logica Plano A/B/C de atualizacao
|
|-- ManagerAgent.SessionWorker/         # Worker por sessao de usuario
|   |-- Program.cs                      # Entry point, lanca na sessao correta
|   |-- TrayIcon/
|   |   +-- TrayApplicationContext.cs   # Icone da bandeja e menu
|   |-- Workers/
|   |   |-- CaptureWorker.cs            # Captura janela ativa a cada 5s
|   |   |-- HeartbeatWorker.cs          # Gera heartbeats a cada 60s
|   |   +-- DailyBoundaryWorker.cs      # Gerencia transicoes de meia-noite
|   |-- Services/
|   |   |-- WindowsActiveWindowProvider.cs  # Win32 API para janela ativa
|   |   |-- SessionMonitor.cs               # Eventos de sessao
|   |   +-- InputActivityTracker.cs         # Deteccao de idle
|   |-- Pipes/
|   |   |-- PipeClient.cs               # Cliente de named pipe
|   |   +-- AutonomousBuffer.cs         # Buffer local quando pipe esta down
|   +-- Meetings/
|       +-- MeetingDetector.cs          # Deteccao de camera/microfone
|
|-- ManagerAgent.Shared/                # Biblioteca compartilhada
|   |-- Config/
|   |   |-- ConfigManager.cs            # Leitura/escrita de config.json + DPAPI
|   |   +-- ConfigPaths.cs              # Caminhos do sistema de arquivos
|   |-- Pipes/
|   |   |-- PipeMessage.cs              # DTO de mensagem do pipe
|   |   +-- PipeMessageType.cs          # Enum de tipos de mensagem
|   |-- Logging/
|   |   +-- SourceEnricher.cs           # Enricher Serilog (Service/Worker)
|   +-- Native/
|       +-- NativeMethods.cs            # P/Invoke Win32
|
|-- ManagerAgent.Installer/             # Logica de instalacao
|   |-- InstallActions.cs               # Registro de servico, permissoes
|   |-- UninstallActions.cs             # Remocao e limpeza
|   +-- MigrationV1ToV2.cs              # Migracao automatica do V1
|
|-- ManagerAgent.Configurator/          # CLI de configuracao
+-- ManagerAgent.Tray/                  # Legado V1 (apenas referencia)
```

### Compilacao

```powershell
# Debug
dotnet build -c Debug

# Release (todos os projetos)
dotnet build -c Release

# Publicar binarios para distribuicao
dotnet publish src/ManagerAgent.Service/ManagerAgent.Service.csproj `
  -c Release -r win-x64 --self-contained false

dotnet publish src/ManagerAgent.SessionWorker/ManagerAgent.SessionWorker.csproj `
  -c Release -r win-x64 --self-contained false
```

### Debugging

**Visual Studio:**
1. Abrir `ManagerAgent.sln`
2. Para testar o Service: selecionar `ManagerAgent.Service` como startup project, executar como Administrador
3. Para testar o SessionWorker: selecionar `ManagerAgent.SessionWorker` como startup project, F5

**Logs durante desenvolvimento:**
```powershell
# Service
Get-Content "C:\ProgramData\ManagerAgent\logs\service-$(Get-Date -Format 'yyyyMMdd').log" -Wait -Tail 50

# SessionWorker
Get-Content "C:\ProgramData\ManagerAgent\logs\worker-$(Get-Date -Format 'yyyyMMdd').log" -Wait -Tail 50
```

**Verificar estado do servico:**
```powershell
Get-Service -Name "ManagerAgent" | Select-Object Name, Status, StartType
sc.exe queryex ManagerAgent
```

### Convencoes de Log

O Serilog e configurado com:

- Rotacao diaria, retencao de 7 dias, limite de 50 MB por arquivo
- Template: `{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] [{Source}] [{Component}] {Message}{NewLine}{Exception}`
- O enricher `SourceEnricher` adiciona o campo `{Source}` com valor "Service" ou "Worker"
- Ambos os processos podem escrever nos mesmos arquivos de log de forma concorrente

---

## Geracao de Nova Versao

### Processo Completo

#### 1. Preparacao

```powershell
# 1.1. Atualizar versao em todos os .csproj relevantes
# ManagerAgent.Service.csproj:     <Version>2.1.0</Version>
# ManagerAgent.SessionWorker.csproj: <Version>2.1.0</Version>

# 1.2. Executar health check completo no ambiente de desenvolvimento
.\scripts\health-check.ps1
```

#### 2. Compilacao Release

```powershell
# Limpar builds anteriores
dotnet clean

# Compilar em Release
dotnet build -c Release

# Publicar Service
dotnet publish src/ManagerAgent.Service/ManagerAgent.Service.csproj `
  -c Release -r win-x64 --self-contained false -o build/service

# Publicar SessionWorker
dotnet publish src/ManagerAgent.SessionWorker/ManagerAgent.SessionWorker.csproj `
  -c Release -r win-x64 --self-contained false -o build/worker

# Publicar Configurator
dotnet publish src/ManagerAgent.Configurator/ManagerAgent.Configurator.csproj `
  -c Release -r win-x64 --self-contained false -o build/configurator
```

#### 3. Geracao do Pacote de Atualizacao

O sistema de auto-update espera um arquivo ZIP contendo os novos binarios e um arquivo `version.json` na raiz:

```json
{
  "version": "2.1.0",
  "sha256": "hash-do-zip",
  "releaseDate": "2026-05-08",
  "mandatory": false
}
```

```powershell
# Criar ZIP de atualizacao
$version = "2.1.0"
Compress-Archive -Path "build\*" -DestinationPath "releases\ManagerAgent-v$version-update.zip"

# Calcular SHA-256
$hash = (Get-FileHash "releases\ManagerAgent-v$version-update.zip" -Algorithm SHA256).Hash

# Atualizar version.json no servidor
# (publicar no endpoint verificado pelo UpdateCheckerWorker)
```

#### 4. Geracao do Instalador Inno Setup

```powershell
# Compilar o script Inno Setup
iscc installer\ManagerAgent.iss

# Output: installer\Output\ManagerAgent-Setup-2.1.0.exe
```

#### 5. Distribuicao

```powershell
# Calcular checksum do instalador
Get-FileHash "installer\Output\ManagerAgent-Setup-2.1.0.exe" -Algorithm SHA256

# Publicar no servidor de atualizacoes
# Publicar no canal de distribuicao da empresa
```

### Checklist de Build

- [ ] Versao atualizada em todos os `.csproj`
- [ ] Codigo compilando sem erros ou warnings
- [ ] Service publicado em Release
- [ ] SessionWorker publicado em Release
- [ ] ZIP de atualizacao gerado com `version.json` correto
- [ ] SHA-256 do ZIP calculado e publicado
- [ ] Instalador Inno Setup compilado
- [ ] Instalacao testada em maquina limpa (Windows 10 e 11)
- [ ] Migracao V1 para V2 testada
- [ ] Auto-update testado (Plano A pelo menos)
- [ ] Menu da bandeja funcional
- [ ] Scripts diagnosticos funcionando
- [ ] Logs sendo gerados corretamente (service e worker)
- [ ] Upload de eventos funcionando
- [ ] Vinculacao com backend funcionando

### Builds por Empresa

Cada empresa recebe um instalador com a `chaveAtivacaoEmpresa` embutida. Use o `ManagerAgent.Configurator` durante o processo de instalacao silenciosa para configurar a chave correta por empresa, ou gere instaladores separados via Inno Setup com a chave pre-definida em constantes de compilacao.

---

## Desinstalacao

### Metodo 1: Via Painel de Controle (Recomendado)

1. Abrir **Configuracoes** -> **Aplicativos** -> **Manager Agent**
2. Clicar em **Desinstalar**
3. O Inno Setup executara `UninstallActions` automaticamente

### Metodo 2: Script Manual

```powershell
# 1. Parar e remover o servico
Stop-Service -Name "ManagerAgent" -Force -ErrorAction SilentlyContinue
sc.exe delete ManagerAgent

# 2. Aguardar finalizacao dos processos
Get-Process -Name "ManagerAgent.SessionWorker" -ErrorAction SilentlyContinue |
  Stop-Process -Force

# 3. Remover binarios
Remove-Item "C:\Program Files\ManagerAgent" -Recurse -Force

# 4. Remover dados e configuracoes
Remove-Item "C:\ProgramData\ManagerAgent" -Recurse -Force

# 5. Verificar limpeza
Get-Service -Name "ManagerAgent" -ErrorAction SilentlyContinue
# Deve retornar vazio ou erro "nao encontrado"
```

### Desinstalacao Preservando Historico

```powershell
# Remover apenas binarios e servico
Stop-Service -Name "ManagerAgent" -Force
sc.exe delete ManagerAgent
Remove-Item "C:\Program Files\ManagerAgent" -Recurse -Force

# C:\ProgramData\ManagerAgent\ permanece intacto
# buffer.db e logs sao preservados para auditoria
```

---

## Troubleshooting

### Problema: Servico nao inicia

**Sintomas:**
- `Get-Service -Name "ManagerAgent"` mostra `Stopped` ou servico nao encontrado
- Nenhum processo SessionWorker na sessao do usuario

**Solucoes:**
```powershell
# 1. Verificar se o servico esta registrado
sc.exe query ManagerAgent

# 2. Tentar iniciar manualmente
Start-Service -Name "ManagerAgent"

# 3. Verificar logs de erro do servico
Get-EventLog -LogName Application -Source "ManagerAgent" -Newest 20

# 4. Verificar logs do agente
Get-Content "C:\ProgramData\ManagerAgent\logs\service-$(Get-Date -Format 'yyyyMMdd').log" -Tail 50

# 5. Verificar se .NET 8 esta instalado
dotnet --list-runtimes | Select-String "8\."

# 6. Reinstalar o servico
sc.exe create ManagerAgent binPath= "C:\Program Files\ManagerAgent\ManagerAgent.Service.exe" start= auto
Start-Service -Name "ManagerAgent"
```

### Problema: Eventos nao sendo enviados ao servidor

**Sintomas:**
- Buffer SQLite crescendo indefinidamente
- Logs mostram erros de upload

**Solucoes:**
```powershell
# 1. Testar conectividade com o servidor
Test-NetConnection -ComputerName "api-events.imanagerportal.com" -Port 443

# 2. Verificar config.json
Get-Content "C:\ProgramData\ManagerAgent\config.json"

# 3. Ver logs de upload
Select-String -Path "C:\ProgramData\ManagerAgent\logs\service-*.log" -Pattern "upload|UploadWorker"

# 4. Executar health check
.\scripts\health-check.ps1

# 5. Verificar se o deviceToken esta presente (campo nao vazio)
# Se vazio, vincular novamente via Configurator
"C:\Program Files\ManagerAgent\ManagerAgent.Configurator.exe" --re-link
```

### Problema: SessionWorker nao aparece na sessao do usuario

**Sintomas:**
- Servico esta rodando mas nenhum icone na bandeja do sistema
- Sem processo `ManagerAgent.SessionWorker` no Gerenciador de Tarefas

**Solucoes:**
```powershell
# 1. Verificar se o WorkerWatchdog esta ativo (logs do servico)
Select-String -Path "C:\ProgramData\ManagerAgent\logs\service-*.log" -Pattern "WorkerWatchdog|SessionWorker"

# 2. Verificar se o servico tem permissao para lancar processos em sessoes de usuario
# O servico precisa rodar como SYSTEM (nao como conta de usuario)
sc.exe qc ManagerAgent | Select-String "SERVICE_START_NAME"
# Deve mostrar: SERVICE_START_NAME : LocalSystem

# 3. Reiniciar o servico para forcara nova deteccao de sessao
Restart-Service -Name "ManagerAgent"

# 4. Verificar logs de erro de lancamento
Select-String -Path "C:\ProgramData\ManagerAgent\logs\service-*.log" -Pattern "Error|Launch|CreateProcess"
```

### Problema: Alto consumo de CPU ou Memoria

**Sintomas:**
- CPU do Service ou SessionWorker acima de 10% de forma continua
- Memoria acima de 200 MB

**Solucoes:**
```powershell
# 1. Monitorar performance
.\scripts\monitorar-performance.ps1

# 2. Verificar se ha loop de erros nos logs
Select-String -Path "C:\ProgramData\ManagerAgent\logs\*.log" -Pattern "Error|Exception" |
  Group-Object -Property Line | Sort-Object Count -Descending | Select-Object -First 10

# 3. Verificar tamanho do buffer
(Get-Item "C:\ProgramData\ManagerAgent\buffer.db").Length / 1MB

# 4. Se buffer muito grande, executar limpeza
.\scripts\limpar-reset.ps1

# 5. Reiniciar o servico
Restart-Service -Name "ManagerAgent"
```

### Problema: Auto-update falha repetidamente

**Sintomas:**
- Arquivo `update-failed.flag` presente em `C:\ProgramData\ManagerAgent\`
- Versao do agente nao atualiza

**Solucoes:**
```powershell
# 1. Verificar logs de atualizacao
Select-String -Path "C:\ProgramData\ManagerAgent\logs\service-*.log" -Pattern "Update|update"

# 2. Verificar arquivos de flag
Get-ChildItem "C:\ProgramData\ManagerAgent\" -Filter "*.flag"

# 3. Limpar flag de falha para permitir nova tentativa
Remove-Item "C:\ProgramData\ManagerAgent\update-failed.flag" -ErrorAction SilentlyContinue

# 4. Verificar se PowerShell esta disponivel (Plano A)
Get-Command powershell.exe

# 5. Verificar politica de execucao
Get-ExecutionPolicy -List

# 6. Para forcar atualizacao manual: baixar novo instalador e executar
.\ManagerAgent-Setup-2.x.x.exe /VERYSILENT
```

### Problema: Scripts diagnosticos nao executam via menu

**Sintomas:**
- Clicar em script no menu nao abre janela PowerShell

**Solucoes:**
```powershell
# 1. Verificar execution policy
Get-ExecutionPolicy -Scope CurrentUser

# 2. Configurar se necessario
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned -Force

# 3. Verificar se scripts existem
Get-ChildItem "C:\Program Files\ManagerAgent\scripts\"

# 4. Executar script diretamente para ver erro
powershell.exe -ExecutionPolicy Bypass -File "C:\Program Files\ManagerAgent\scripts\health-check.ps1"
```

---

## FAQ

### Geral

**O agente monitora tudo que eu faco no computador?**
Nao. Ele captura apenas o nome do aplicativo (processo) e o titulo da janela ativa. Nao captura conteudo digitado, conteudo de documentos, senhas ou realiza screenshots.

**Os dados ficam salvos no meu computador?**
Sim, temporariamente. Eventos ficam em buffer SQLite local ate serem enviados ao servidor. Apos o envio, sao mantidos por 7 dias antes de serem removidos automaticamente.

**Como sei se o agente esta funcionando?**
Observe o icone na bandeja do sistema. Se o icone nao aparecer, verifique o estado do servico com `Get-Service -Name "ManagerAgent"`. Para verificacao completa, execute `scripts\health-check.ps1`.

**Posso desabilitar temporariamente?**
Sim. Via PowerShell com permissao de administrador: `Stop-Service -Name "ManagerAgent"`. Para reativar: `Start-Service -Name "ManagerAgent"`. O servico retomara automaticamente no proximo boot.

**O agente funciona sem conexao com a internet?**
Sim. Eventos sao armazenados no buffer local e enviados quando a conectividade for restabelecida. O buffer suporta ate 7 dias de eventos offline.

### Tecnico

**Funciona em macOS ou Linux?**
Nao. A arquitetura V2 usa APIs especificas do Windows (WTS, SCM, DPAPI, Win32). Apenas Windows 10 e Windows 11 (64-bit) sao suportados.

**Suporta multiplos usuarios na mesma maquina?**
Sim. Essa e uma das principais melhorias da V2. O Service lanca um SessionWorker para cada sessao ativa, suportando Terminal Server (RDP), Fast User Switching e qualquer cenario multi-usuario do Windows.

**Como atualizar para uma nova versao?**
O agente se atualiza automaticamente via UpdateCheckerWorker. Para atualizacao manual, basta executar o novo instalador: `ManagerAgent-Setup-X.Y.Z.exe`. O instalador para o servico, atualiza os binarios e reinicia o servico.

**O que e o DPAPI e por que e usado?**
DPAPI (Data Protection API) e um servico criptografico do Windows que protege dados em nivel de maquina ou usuario sem necessidade de gerenciar chaves manualmente. O campo `deviceToken` e criptografado com DPAPI para que nao possa ser lido em texto claro, mesmo que alguem obtenha acesso ao arquivo `config.json`.

**Onde fica o codigo fonte?**
GitHub: [mvfranca65/manager-srv-agent](https://github.com/mvfranca65/manager-srv-agent)

**Quais portas de rede o agente usa?**
O agente se comunica exclusivamente via HTTPS (porta 443) com os servidores configurados. Nao abre portas de entrada.

**Como funciona a comunicacao interna entre Service e SessionWorker?**
Via Named Pipes do Windows. O Service cria um pipe nomeado `\\.\pipe\ManagerAgent_{sessionId}` com ACL restrita ao usuario da sessao e ao SYSTEM. O SessionWorker conecta a esse pipe para enviar eventos. Se o pipe estiver indisponivel, o SessionWorker usa o `AutonomousBuffer` local.

### Privacidade

**Quem tem acesso aos dados coletados?**
Apenas o servidor configurado nos campos `baseUrlAdmin` e `baseUrlEvents` do `config.json`.

**Os dados sao criptografados em transito?**
Sim. Toda comunicacao com o servidor e via HTTPS.

**O agente pode capturar senhas?**
Nao. O agente captura apenas nome do processo e titulo da janela. Nao ha keylogging, nao ha OCR, nao ha inspecao de conteudo de janelas.

**Posso auditar o codigo?**
Sim. O codigo e open source no GitHub. Voce pode compilar o agente a partir do codigo fonte e verificar exatamente o que e capturado.

---

## Instalacao Detalhada

> Esta secao consolida o GUIA-INSTALACAO.md (absorvido em 2026-06-11).

### Opcao 1: Instalacao individual (com interface grafica)

Use esta opcao para instalar em 1 maquina por vez com o assistente visual.

1. No portal Manager, va em **Organizacao** > clique no botao **Baixar instalador**
2. Execute `ManagerAgent-Setup.exe` como Administrador (UAC vai perguntar — clicar em Sim)
3. Siga o assistente:
   - Pasta de instalacao: manter padrao (`C:\Program Files\ManagerAgent`)
   - Identificador do colaborador: ex `i455676`, CPF, matricula ou email corporativo
4. O instalador realiza automaticamente: copia binarios, cria diretorio de dados, registra servico Windows com inicio automatico (conta SYSTEM), configura recuperacao automatica no SCM (1s, 5s, 30s), executa migracao V1→V2 se aplicavel, inicia o servico

Verificar instalacao:
```powershell
Get-Service ManagerAgent      # Status deve ser Running
Get-Process -Name "ManagerAgent.SessionWorker"
```

### Opcao 2: Instalacao silenciosa

```powershell
ManagerAgent-Setup.exe /VERYSILENT /SUPPRESSMSGBOXES /NORESTART /identificador=i455676

# Com log:
ManagerAgent-Setup.exe /VERYSILENT /SUPPRESSMSGBOXES /NORESTART /identificador=i455676 /LOG="C:\Temp\install-log.txt"
```

Parametros:

| Parametro | Obrigatorio | Descricao |
|-----------|-------------|-----------|
| `/VERYSILENT` | Sim | Instala sem janelas |
| `/SUPPRESSMSGBOXES` | Recomendado | Suprime dialogs de erro |
| `/NORESTART` | Recomendado | Nao reinicia a maquina |
| `/identificador=VALOR` | Sim (silencioso) | Identificador do colaborador |
| `/DIR="C:\Caminho"` | Nao | Pasta de instalacao customizada |
| `/LOG="C:\log.txt"` | Nao | Log da instalacao em arquivo |

Codigos de retorno: `0` = sucesso, `1` = falha geral, `2` = identificador nao informado, `3` = falha na validacao com o servidor.

### Opcao 3: Deploy em massa (multiplas maquinas)

Pre-requisitos: WinRM habilitado nas maquinas destino, credenciais de administrador do dominio, script `deploy-em-massa.ps1` (incluido no pacote), CSV com lista de maquinas.

```powershell
# Habilitar WinRM nas maquinas (ou via GPO)
winrm quickconfig -quiet

# CSV formato:
# maquina,identificador
# PC-JOAO,i455676
# PC-MARIA,i789012

# Executar deploy
.\deploy-em-massa.ps1 -CsvPath "colaboradores.csv" -InstaladorPath "ManagerAgent-Setup.exe"
```

O script testa ping, copia instalador via `\\maquina\C$\Temp\`, executa instalacao silenciosa via WinRM, limpa o instalador e gera relatorio CSV com status (SUCESSO/FALHA/ERRO/IGNORADO).

Troubleshooting deploy em massa:

| Problema | Causa provavel | Solucao |
|----------|---------------|---------|
| "Maquina inacessivel" | Maquina desligada | Verificar se esta ligada e conectada |
| "Acesso negado" | Credenciais invalidas | Usar conta de administrador do dominio |
| "WinRM nao pode completar" | WinRM nao habilitado | Rodar `winrm quickconfig` na destino |
| "Servico remoto nao respondeu" | Firewall bloqueando 5985/5986 | Liberar portas WinRM |

---

## Gerenciamento de Chave de Ativacao por Empresa

> Esta secao consolida o GUIA-CHAVE-EMPRESA.md (absorvido em 2026-06-11).
> Nota de legado: referencias a `AthenaAgent.Tray` nesse contexto referem-se a V1. Na V2 o processo e `ManagerAgent`.

Cada empresa recebe um pacote instalador com a `chaveAtivacaoEmpresa` embutida. O script de build solicita a chave, a embutte no `appsettings.json`, compila, e restaura o arquivo original (a chave nunca e commitada).

### Gerar build por empresa

```powershell
.\scripts\build\build-pacote-instalacao.ps1
# Quando solicitado, informar a chave da empresa
```

O script: faz backup do `appsettings.json`, configura a chave, compila os projetos, gera o ZIP, restaura o original.

Output: `instalador\ManagerAgent-Installer.zip` — distribuir APENAS para a empresa correspondente.

### Workflow para multiplas empresas

```powershell
# Empresa A
.\scripts\build\build-pacote-instalacao.ps1
# Informar: CHAVE_EMPRESA_A
# Renomear ZIP: ManagerAgent-EmpresaA-Installer.zip

# Empresa B
.\scripts\build\build-pacote-instalacao.ps1
# Informar: CHAVE_EMPRESA_B
# Renomear ZIP: ManagerAgent-EmpresaB-Installer.zip
```

Manter registro: `builds/EmpresaA/` + `builds/EmpresaB/` com o ZIP e a chave correspondente.

### Validar chave no pacote gerado

```powershell
Expand-Archive instalador\ManagerAgent-Installer.zip -DestinationPath C:\Temp\Test
$config = Get-Content C:\Temp\Test\bin\appsettings.json | ConvertFrom-Json
$config.ManagerAgent.Authorization
# Deve exibir a chave configurada
```

### Troubleshooting

| Problema | Solucao |
|----------|---------|
| "Chave de ativacao nao pode ser vazia!" | Execute novamente e informe a chave |
| "Falha ao atualizar appsettings.json" | Verificar se o JSON e valido: `Get-Content src\...\appsettings.json | ConvertFrom-Json` |
| Build falhou e `appsettings.json` ficou alterado | `Copy-Item src\...\appsettings.json.bak src\...\appsettings.json -Force` |

**Regras de seguranca:**
- NAO compartilhe chave de uma empresa com outra
- NAO commite o backup `.bak` no Git (ja esta no `.gitignore`)
- NAO distribua o mesmo ZIP para multiplas empresas
- SEMPRE documente qual chave foi usada em cada build

A chave de ativacao e gerenciada pelo servidor backend. Rotacao via `DELETE /api/admin/empresas/{id}/chave` (Admin JWT). A chave nao tem TTL automatico — valida indefinidamente ate ser revogada (GAP de seguranca — ver `_shared/seguranca.md`).

---

## Setup de Desenvolvimento

> Esta secao consolida o GUIA-DESENVOLVEDOR.md (absorvido em 2026-06-11).

### Pre-requisitos

- .NET SDK 8.0 — https://dotnet.microsoft.com/download
- Git — https://git-scm.com/downloads
- (Opcional) Visual Studio 2022 ou VS Code com extensao C#

```powershell
dotnet --version    # Deve mostrar 8.x ou superior
git --version
```

### Setup inicial

```powershell
git clone https://github.com/mvfranca65/manager-srv-agent.git
cd manager-srv-agent
dotnet restore ManagerAgent.sln
dotnet build ManagerAgent.sln -c Release

# Criar config.json de desenvolvimento
mkdir C:\ProgramData\ManagerAgent -Force
@"
{
  "baseUrlAdmin": "https://admin.imanagerportal.com",
  "baseUrlEvents": "https://api-events.imanagerportal.com",
  "chaveAtivacaoEmpresa": "SEU_UUID_AQUI",
  "identificadorColaborador": "SEU_ID_AQUI",
  "instalacaoId": "",
  "deviceToken": "",
  "menuVisivel": true
}
"@ | Out-File C:\ProgramData\ManagerAgent\config.json -Encoding UTF8
```

### Rodar em desenvolvimento

**Opcao 1 — SessionWorker direto (recomendado para captura de eventos):**

```powershell
dotnet run --project src/ManagerAgent.SessionWorker/ManagerAgent.SessionWorker.csproj
```

O icone aparece na bandeja. O SessionWorker usa `AutonomousBuffer` (SQLite local) quando nao ha pipe disponivel.

**Opcao 2 — Service + SessionWorker completo (requer privilegios de administrador):**

```powershell
dotnet publish src/ManagerAgent.Service/ManagerAgent.Service.csproj -c Release -r win-x64 --self-contained true -o dist/service
dotnet publish src/ManagerAgent.SessionWorker/ManagerAgent.SessionWorker.csproj -c Release -r win-x64 --self-contained true -o dist/worker

# PowerShell como Administrador
New-Service -Name "ManagerAgentService" -BinaryPathName "C:\caminho\dist\service\ManagerAgent.Service.exe" -StartupType Automatic
Start-Service ManagerAgentService
```

### Ver logs durante desenvolvimento

```powershell
# Logs em tempo real
Get-Content C:\ProgramData\ManagerAgent\logs\service-*.log -Wait -Tail 20
Get-Content C:\ProgramData\ManagerAgent\logs\worker-*.log -Wait -Tail 20

# Logs coloridos (erros em vermelho, warnings em amarelo)
Get-Content C:\ProgramData\ManagerAgent\logs\worker-*.log -Tail 50 |
  ForEach-Object {
    if ($_ -match '\[ERR\]') { Write-Host $_ -ForegroundColor Red }
    elseif ($_ -match '\[WRN\]') { Write-Host $_ -ForegroundColor Yellow }
    else { Write-Host $_ -ForegroundColor Cyan }
  }
```

Formato dos logs: `{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] [{Source}] [{Component}] {Message}{NewLine}{Exception}`

### Fluxo de desenvolvimento

```
1. Alterar codigo
2. Parar SessionWorker (clique direito > Sair) ou parar Service
3. dotnet build ManagerAgent.sln
4. dotnet run --project src/ManagerAgent.SessionWorker/...
5. Get-Content ...logs\worker-*.log -Wait -Tail 20
```

Nao ha hot reload (Windows Forms + Windows Service).

### Executar testes

```powershell
dotnet test tests/ManagerAgent.Service.Tests/ManagerAgent.Service.Tests.csproj
dotnet test tests/ManagerAgent.SessionWorker.Tests/ManagerAgent.SessionWorker.Tests.csproj
```

### Debug

**Visual Studio 2022:**
1. Abrir `ManagerAgent.sln`
2. Definir `ManagerAgent.SessionWorker` como startup project
3. F5

**VS Code:**
1. Abrir pasta raiz
2. F5 > `.NET: Launch ManagerAgent.SessionWorker`

### Troubleshooting desenvolvimento

| Problema | Solucao |
|----------|---------|
| Nao compila | `dotnet clean` -> `dotnet restore` -> `dotnet build` |
| Icone nao aparece | Verificar logs do worker no startup |
| Pipe nao conecta | Service pode nao estar rodando; usar AutonomousBuffer em dev (normal) |
| Config incompleta | Verificar `C:\ProgramData\ManagerAgent\config.json` |
| Upload falha | Checar conectividade com `baseUrlEvents`; ver logs do UploadWorker |
| SQLite error | Excluir `C:\ProgramData\ManagerAgent\eventos.db` e reiniciar |
| DPAPI error | Executar como o mesmo usuario que criou o token |

### Estrutura completa do projeto

```
src/
├── ManagerAgent.Service/              # Windows Service (Session 0, SYSTEM)
│   ├── Program.cs                     # Entry point, Serilog, DI
│   ├── ManagerAgentService.cs         # Hosted service principal
│   ├── SessionManagement/
│   │   ├── WorkerRegistry.cs          # State machine de sessoes
│   │   ├── WorkerLauncher.cs          # Lanca SessionWorker por sessao
│   │   └── WtsSessionMonitor.cs       # Notificacoes WTS
│   ├── Pipe/
│   │   ├── PipeServer.cs              # Servidor Named Pipe
│   │   └── PipeMessageHandler.cs      # Processa mensagens
│   ├── Storage/
│   │   └── ServiceSqliteEventBuffer.cs
│   ├── Upload/
│   │   ├── UploadWorker.cs
│   │   └── HttpEventUploader.cs
│   ├── Update/
│   │   ├── UpdateCheckerWorker.cs     # Verifica atualizacoes a cada 6h
│   │   ├── UpdateDownloader.cs        # Download + verificacao SHA-256
│   │   └── UpdateApplier.cs           # Plano A/B/C fallback
│   ├── Linking/
│   │   ├── AgentLinkService.cs
│   │   └── TokenManager.cs
│   ├── Diagnostics/
│   │   └── SelfTest.cs
│   ├── Resilience/
│   │   └── WorkerWatchdog.cs
│   └── Config/
│       ├── IConfigManager.cs
│       └── ConfigManager.cs           # Config criptografada com DPAPI
│
├── ManagerAgent.SessionWorker/        # Processo por usuario
│   ├── Program.cs
│   ├── Worker.cs                      # Loop principal de captura
│   ├── Capture/
│   │   ├── WindowActivityService.cs
│   │   ├── WindowsActiveWindowProvider.cs
│   │   ├── SessionMonitor.cs
│   │   ├── IdleMonitorService.cs
│   │   ├── HeartbeatService.cs
│   │   ├── InputActivityTracker.cs
│   │   ├── MeetingDetector.cs
│   │   ├── DailyBoundaryWorker.cs
│   │   └── UserStatusManager.cs
│   ├── Pipe/
│   │   ├── PipeClient.cs
│   │   ├── PipeEventBuffer.cs
│   │   ├── PipeErrorReporter.cs
│   │   └── AutonomousBuffer.cs        # Buffer SQLite fallback
│   └── Tray/
│       ├── TrayIconManager.cs
│       └── TrayApplicationContext.cs
│
├── ManagerAgent.Shared/               # Biblioteca compartilhada
│   ├── Config/
│   │   ├── ConfigPaths.cs
│   │   └── ConfigManager.cs
│   ├── Pipe/                          # DTOs do protocolo
│   ├── Logging/
│   │   └── SourceEnricher.cs
│   ├── Runtime/
│   └── NativeMethods.cs
│
├── ManagerAgent.Installer/            # DLL auxiliar Inno Setup
│   ├── InstallActions.cs
│   ├── UninstallActions.cs
│   └── MigrationV1ToV2.cs
│
└── ManagerAgent.Configurator/         # CLI de configuracao
    └── Program.cs
```

> `ManagerAgent.Tray` (V1) esta deprecado e sera removido em versao futura.

---

## Suporte

### Antes de Reportar um Problema

Execute o script de coleta de diagnostico e anexe o ZIP gerado:
```powershell
"C:\Program Files\ManagerAgent\scripts\coletar-diagnostico.ps1"
```

### Contato

- **GitHub Issues:** [mvfranca65/manager-srv-agent/issues](https://github.com/mvfranca65/manager-srv-agent/issues)
- **Wiki:** [GitHub Wiki](https://github.com/mvfranca65/manager-srv-agent/wiki)

---

**Versao do Documento:** 2.0.0
**Ultima Atualizacao:** 2026-06-11
**Licenca:** MIT
