> **STATUS:** ATIVO
> **DATA:** 2026-06-11
> **DONO:** @Natasha (QA)
> **REVISADO POR:** @Tony

# Plano de Testes Regressivos - Manager Agent V2

**Versao:** 2.0.0
**Data:** 2026-06-11
**Objetivo:** Validar todas as funcionalidades do Manager Agent V2 antes de release

> ⚠️ **Nota sobre AthenaAgent.Tray:** A V1 do Agent rodava com o processo `AthenaAgent.Tray`. Esse nome aparece neste plano nos casos de migração V1→V2 (R11.x) e nas verificações de limpeza pós-migração — é intencional, representa o legado. Para a V2 (atual), o processo é `ManagerAgent.Tray`.

---

## Indice

1. [Preparacao do Ambiente](#preparacao-do-ambiente)
2. [Testes de Instalacao](#testes-de-instalacao)
3. [Testes de Interface (System Tray)](#testes-de-interface-system-tray)
4. [Testes de Captura de Dados](#testes-de-captura-de-dados)
5. [Testes de Scripts Diagnosticos](#testes-de-scripts-diagnosticos)
6. [Testes de Performance](#testes-de-performance)
7. [Testes de Erro e Recuperacao](#testes-de-erro-e-recuperacao)
8. [Testes de Ciclo de Vida do Service](#testes-de-ciclo-de-vida-do-service)
9. [Testes Multi-Sessao](#testes-multi-sessao)
10. [Testes de Named Pipe](#testes-de-named-pipe)
11. [Testes de Auto-Update](#testes-de-auto-update)
12. [Testes de Migracao V1 para V2](#testes-de-migracao-v1-para-v2)
13. [Testes de AutonomousBuffer](#testes-de-autonomousbuffer)
14. [Testes de Worker Watchdog](#testes-de-worker-watchdog)
15. [Testes de Desinstalacao](#testes-de-desinstalacao)
16. [Relatorio Final](#relatorio-final)

---

## Preparacao do Ambiente

### Pre-requisitos

- [ ] Windows 10 ou 11 (64-bit)
- [ ] .NET 8.0 Runtime instalado
- [ ] PowerShell 5.1 ou superior
- [ ] Execution Policy configurada: `RemoteSigned` ou `Unrestricted`
- [ ] Conta de administrador local disponivel
- [ ] Conexao com internet (para testes de upload e auto-update)
- [ ] Para testes multi-sessao: ao menos dois usuarios locais configurados

### Setup Inicial

```powershell
# 1. Verificar versao do Windows
winver

# 2. Verificar .NET Runtime
dotnet --list-runtimes
# Esperado: Microsoft.NETCore.App 8.x.x ou superior

# 3. Verificar PowerShell
$PSVersionTable.PSVersion
# Esperado: 5.1 ou superior

# 4. Configurar Execution Policy
Get-ExecutionPolicy -Scope CurrentUser
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned -Force

# 5. Preparar pasta de trabalho
New-Item -ItemType Directory -Path "C:\Temp\ManagerAgent-Tests" -Force
```

### Preparacao do Instalador

```powershell
# Copiar instalador para pasta de testes
Copy-Item "ManagerAgent-Setup.exe" "C:\Temp\ManagerAgent-Tests\"

# Verificar hash do instalador
Get-FileHash "C:\Temp\ManagerAgent-Tests\ManagerAgent-Setup.exe" -Algorithm SHA256
```

---

## Testes de Instalacao

### Teste 1.1: Instalacao Limpa (Primeira Vez)

**Objetivo:** Validar instalacao em sistema sem Manager Agent

**Pre-condicao:** Sistema sem Manager Agent instalado anteriormente

**Procedimento:**

```powershell
# 1. Confirmar que nao existe instalacao previa
Get-Service ManagerAgent -ErrorAction SilentlyContinue
# Esperado: Vazio

Test-Path "C:\Program Files\ManagerAgent"
# Esperado: False

# 2. Executar instalador como administrador
Start-Process "C:\Temp\ManagerAgent-Tests\ManagerAgent-Setup.exe" -Verb RunAs

# 3. Seguir assistente Inno Setup
```

**Validacoes:**

- [ ] **R1.1.1:** Instalador inicia sem erros
- [ ] **R1.1.2:** Pasta `C:\Program Files\ManagerAgent` criada
- [ ] **R1.1.3:** `ManagerAgent.Service.exe` presente
- [ ] **R1.1.4:** `ManagerAgent.SessionWorker.exe` presente
- [ ] **R1.1.5:** `ManagerAgent.Configurator.exe` presente
- [ ] **R1.1.6:** Pasta `scripts\` com 5 scripts PowerShell presentes
- [ ] **R1.1.7:** Pasta `C:\ProgramData\ManagerAgent` criada
- [ ] **R1.1.8:** `config.json` criado com campos obrigatorios
- [ ] **R1.1.9:** `buffer.db` criado (SQLite WAL)
- [ ] **R1.1.10:** Pasta `logs\` criada
- [ ] **R1.1.11:** Windows Service "ManagerAgent" registrado
- [ ] **R1.1.12:** Service configurado para iniciar automaticamente
- [ ] **R1.1.13:** SCM recovery configurado (1s, 5s, 30s)
- [ ] **R1.1.14:** Service inicia automaticamente apos instalacao
- [ ] **R1.1.15:** SessionWorker iniciado pelo Service (icone aparece na bandeja)

**Comandos de Validacao:**

```powershell
# Verificar Service
Get-Service ManagerAgent | Select-Object Name, Status, StartType

# Verificar processos
Get-Process -Name "ManagerAgent.Service" -ErrorAction SilentlyContinue
Get-Process -Name "ManagerAgent.SessionWorker" -ErrorAction SilentlyContinue

# Verificar config.json
Get-Content "C:\ProgramData\ManagerAgent\config.json" | ConvertFrom-Json

# Verificar instalacaoId gerado (campo no config.json)
$config = Get-Content "C:\ProgramData\ManagerAgent\config.json" | ConvertFrom-Json
$config.instalacaoId
# Esperado: GUID valido (formato xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx)

# Verificar DeviceToken encriptado (nao deve estar em plaintext)
$config.deviceToken
# Esperado: string base64 longa (DPAPI encrypted), nao um token JWT raw

# Verificar pasta de scripts
Get-ChildItem "C:\Program Files\ManagerAgent\scripts"
# Esperado: 5 arquivos .ps1
```

---

### Teste 1.2: Instalacao com Configuracao Customizada

**Objetivo:** Validar instalacao com parametros personalizados via Configurador

**Procedimento:**

1. Executar instalacao limpa
2. Abrir `ManagerAgent.Configurator.exe`
3. Preencher `baseUrlAdmin`, `baseUrlEvents`, `chaveAtivacaoEmpresa` e `identificadorColaborador`
4. Salvar e reiniciar Service

**Validacoes:**

- [ ] **R1.2.1:** Configurador salva corretamente em `config.json`
- [ ] **R1.2.2:** `config.json` contem todos os campos esperados
- [ ] **R1.2.3:** Service reinicia apos salvar configuracao
- [ ] **R1.2.4:** SessionWorker usa a nova configuracao

---

### Teste 1.3: Deteccao de Instalacao Existente

**Objetivo:** Validar comportamento ao reinstalar sobre versao existente

**Pre-condicao:** Manager Agent V2 ja instalado

**Procedimento:**

```powershell
# Tentar instalar novamente com mesmo instalador
Start-Process "C:\Temp\ManagerAgent-Tests\ManagerAgent-Setup.exe" -Verb RunAs
```

**Validacoes:**

- [ ] **R1.3.1:** Instalador detecta instalacao existente
- [ ] **R1.3.2:** Oferece opcao de sobrescrever ou cancelar
- [ ] **R1.3.3:** Se sobrescrever: config.json existente e preservado
- [ ] **R1.3.4:** Se sobrescrever: instalacaoId nao e regenerado
- [ ] **R1.3.5:** Service reinicia apos upgrade

---

## Testes de Interface (System Tray)

### Teste 2.1: Icone na Bandeja

**Objetivo:** Validar aparencia e comportamento do icone gerenciado pelo SessionWorker

**Pre-condicao:** Service rodando e SessionWorker iniciado

**Procedimento:**

```powershell
Get-Process -Name "ManagerAgent.SessionWorker"
```

**Validacoes:**

- [ ] **R2.1.1:** Icone aparece na bandeja do sistema (canto inferior direito)
- [ ] **R2.1.2:** Icone customizado do iManager exibido corretamente
- [ ] **R2.1.3:** Tooltip exibe "iManager - vX.Y.Z"
- [ ] **R2.1.4:** Icone nao pisca ou fica instavel
- [ ] **R2.1.5:** Icone nao e removido ao reiniciar o Explorer

---

### Teste 2.2: Estrutura do Menu (Clique Direito)

**Objetivo:** Validar estrutura correta do menu V2

**Procedimento:**

1. Clique direito no icone na bandeja
2. Observar menu que aparece

**Validacoes:**

- [ ] **R2.2.1:** Menu abre sem delay perceptivel (< 500ms)
- [ ] **R2.2.2:** Cabecalho exibe "iManager - vX.Y.Z" em negrito (desabilitado como item)
- [ ] **R2.2.3:** Separador apos cabecalho
- [ ] **R2.2.4:** Item "Ferramentas" com seta (submenu)
- [ ] **R2.2.5:** Separador antes de "Sobre"
- [ ] **R2.2.6:** Item "Sobre" visivel e clicavel
- [ ] **R2.2.7:** Nenhum item "AthenaAgent" ou referencia a versao anterior

---

### Teste 2.3: Submenu Ferramentas

**Objetivo:** Validar submenu de ferramentas V2

**Procedimento:**

1. Clique direito no icone
2. Passar mouse sobre "Ferramentas"
3. Observar submenu

**Validacoes:**

- [ ] **R2.3.1:** Submenu abre ao passar mouse
- [ ] **R2.3.2:** Item "Verificacao de Saude" visivel
- [ ] **R2.3.3:** Item "Logs em Tempo Real" visivel
- [ ] **R2.3.4:** Item "Exportar Diagnostico" visivel
- [ ] **R2.3.5:** Separador antes do item destrutivo
- [ ] **R2.3.6:** Item "Limpar Dados e Reiniciar" visivel
- [ ] **R2.3.7:** Todos os itens clicaveis

---

### Teste 2.4: Acao - Verificacao de Saude (via menu)

**Objetivo:** Validar execucao do health-check via menu

**Procedimento:**

1. Clique direito no icone
2. Ferramentas → Verificacao de Saude

**Validacoes:**

- [ ] **R2.4.1:** Janela PowerShell abre
- [ ] **R2.4.2:** Script `health-check.ps1` executa
- [ ] **R2.4.3:** Score >= 80% em instalacao normal
- [ ] **R2.4.4:** Verifica Service, SessionWorker e Named Pipe
- [ ] **R2.4.5:** Exibe resumo ao final

---

### Teste 2.5: Acao - Limpar Dados e Reiniciar

**Objetivo:** Validar dialogo de confirmacao obrigatoria antes de acao destrutiva

**Procedimento:**

1. Clique direito no icone
2. Ferramentas → Limpar Dados e Reiniciar

**Validacoes:**

- [ ] **R2.5.1:** Dialogo de confirmacao aparece ANTES de qualquer acao
- [ ] **R2.5.2:** Dialogo descreve claramente o que sera feito
- [ ] **R2.5.3:** Aviso explicito que a acao nao pode ser desfeita
- [ ] **R2.5.4:** Botao "Nao" e o padrao (selecionado por Enter)
- [ ] **R2.5.5:** Clicar "Nao": operacao cancelada, Service continua rodando
- [ ] **R2.5.6:** Clicar "Sim": buffer.db removido, Service reiniciado, SessionWorker relancado

---

### Teste 2.6: Acao - Sobre

**Objetivo:** Validar janela "Sobre" do agente

**Procedimento:**

1. Clique direito no icone
2. Clicar em "Sobre"

**Validacoes:**

- [ ] **R2.6.1:** Janela de informacoes abre
- [ ] **R2.6.2:** Exibe versao atual (formato X.Y.Z)
- [ ] **R2.6.3:** Exibe instalacaoId
- [ ] **R2.6.4:** Exibe identificador do colaborador configurado
- [ ] **R2.6.5:** Botao OK fecha a janela

---

## Testes de Captura de Dados

### Teste 3.1: Captura de Janela Ativa

**Objetivo:** Validar captura correta via SessionWorker

**Preparacao:**

```powershell
# Limpar buffer para teste limpo
Stop-Service ManagerAgent
Remove-Item "C:\ProgramData\ManagerAgent\buffer.db" -Force
Start-Service ManagerAgent
Start-Sleep -Seconds 5
```

**Procedimento:**

1. Abrir Google Chrome e navegar para GitHub.com — aguardar 10 segundos
2. Abrir Notepad — aguardar 10 segundos
3. Abrir Visual Studio Code e um arquivo .cs — aguardar 10 segundos

**Validacoes:**

```powershell
# Verificar eventos no buffer
$eventos = sqlite3 "C:\ProgramData\ManagerAgent\buffer.db" `
    "SELECT json FROM eventos WHERE tipo='janela_ativa' ORDER BY timestamp DESC LIMIT 10"
$eventos | ForEach-Object { $_ | ConvertFrom-Json }
```

- [ ] **R3.1.1:** Evento com processo "chrome.exe" capturado
- [ ] **R3.1.2:** Titulo contem "GitHub"
- [ ] **R3.1.3:** Evento com processo "notepad.exe" capturado
- [ ] **R3.1.4:** Evento com processo "Code.exe" capturado
- [ ] **R3.1.5:** Titulo do VS Code contem nome do arquivo
- [ ] **R3.1.6:** Timestamp em formato ISO 8601
- [ ] **R3.1.7:** `instalacaoId` presente e valido (GUID)
- [ ] **R3.1.8:** Campo `usuario` contem o nome de usuario Windows
- [ ] **R3.1.9:** Campo `idle` = false durante uso ativo
- [ ] **R3.1.10:** Nenhum conteudo digitado, screenshot ou senha capturada

---

### Teste 3.2: Deteccao de Idle

**Objetivo:** Validar deteccao de inatividade do usuario pelo SessionWorker

**Procedimento:**

1. Garantir que ha atividade normal (mover mouse, digitar)
2. Parar completamente de interagir (nao tocar mouse nem teclado)
3. Aguardar 2 minutos
4. Retomar atividade

**Validacoes:**

```powershell
# Verificar logs do worker
Get-Content "C:\ProgramData\ManagerAgent\logs\worker-$(Get-Date -Format yyyyMMdd).log" |
    Select-String "idle" | Select-Object -Last 10
```

- [ ] **R3.2.1:** Log indica transicao para idle apos tempo configurado
- [ ] **R3.2.2:** Eventos de janela ativa param durante idle
- [ ] **R3.2.3:** Log indica retorno de idle ao resumir atividade
- [ ] **R3.2.4:** Eventos de janela ativa voltam apos retorno

---

### Teste 3.3: Eventos de Sessao

**Objetivo:** Validar captura de SessionLock, SessionUnlock, SessionLogoff e SessionLogon

**Procedimento:**

1. Pressionar Win+L para bloquear — aguardar 5 segundos
2. Desbloquear com senha

**Validacoes:**

```powershell
$eventos = sqlite3 "C:\ProgramData\ManagerAgent\buffer.db" `
    "SELECT json FROM eventos WHERE tipo='sessao' ORDER BY timestamp DESC LIMIT 5"
$eventos | ForEach-Object { $_ | ConvertFrom-Json }
```

- [ ] **R3.3.1:** Evento "SessionLock" capturado pelo SessionWorker
- [ ] **R3.3.2:** Evento "SessionUnlock" capturado
- [ ] **R3.3.3:** Timestamps em ordem cronologica correta
- [ ] **R3.3.4:** Campos `usuario` e `maquina` preenchidos

---

### Teste 3.4: Heartbeats

**Objetivo:** Validar envio periodico de heartbeats com uptime correto

**Procedimento:**

```powershell
# Aguardar 3 minutos e verificar
Start-Sleep -Seconds 180
$heartbeats = sqlite3 "C:\ProgramData\ManagerAgent\buffer.db" `
    "SELECT json FROM eventos WHERE tipo='heartbeat' ORDER BY timestamp DESC LIMIT 5"
$heartbeats | ForEach-Object {
    $hb = $_ | ConvertFrom-Json
    Write-Host "Timestamp: $($hb.timestamp), Uptime: $($hb.uptime)s"
}
```

**Validacoes:**

- [ ] **R3.4.1:** Pelo menos 3 heartbeats gerados em 3 minutos
- [ ] **R3.4.2:** Uptime aumenta de forma consistente a cada heartbeat
- [ ] **R3.4.3:** Campo `versaoAgente` presente e correto
- [ ] **R3.4.4:** Campo `eventosPendentes` indica quantidade atual no buffer

---

### Teste 3.5: Persistencia no Buffer SQLite WAL

**Objetivo:** Validar armazenamento local com WAL mode

**Procedimento:**

```powershell
# Verificar modo WAL
sqlite3 "C:\ProgramData\ManagerAgent\buffer.db" "PRAGMA journal_mode;"
# Esperado: wal

sqlite3 "C:\ProgramData\ManagerAgent\buffer.db" ".schema"
sqlite3 "C:\ProgramData\ManagerAgent\buffer.db" "SELECT COUNT(*) FROM eventos"
sqlite3 "C:\ProgramData\ManagerAgent\buffer.db" `
    "SELECT tipo, enviado, timestamp FROM eventos ORDER BY timestamp DESC LIMIT 10"
```

**Validacoes:**

- [ ] **R3.5.1:** Arquivo `buffer.db` existe
- [ ] **R3.5.2:** Modo WAL ativo (journal_mode = wal)
- [ ] **R3.5.3:** Tabela `eventos` com colunas corretas
- [ ] **R3.5.4:** Eventos sao inseridos pelo SessionWorker
- [ ] **R3.5.5:** Eventos sao lidos e enviados pelo Service
- [ ] **R3.5.6:** Campo `enviado` = 1 apos upload com sucesso
- [ ] **R3.5.7:** Banco nao corrompe sob carga normal

---

## Testes de Scripts Diagnosticos

### Teste 4.1: Health Check

**Objetivo:** Validar script de verificacao completa para V2

**Procedimento:**

```powershell
cd "C:\Program Files\ManagerAgent\scripts"
.\health-check.ps1
```

**Validacoes:**

- [ ] **R4.1.1:** Script executa sem erros
- [ ] **R4.1.2:** Verifica Windows Service "ManagerAgent" → [OK]
- [ ] **R4.1.3:** Verifica processo `ManagerAgent.Service` → [OK]
- [ ] **R4.1.4:** Verifica processo `ManagerAgent.SessionWorker` → [OK]
- [ ] **R4.1.5:** Verifica arquivos de instalacao → [OK]
- [ ] **R4.1.6:** Verifica `config.json` com campos obrigatorios → [OK]
- [ ] **R4.1.7:** Verifica buffer SQLite WAL → [OK]
- [ ] **R4.1.8:** Verifica logs de servico e worker → [OK]
- [ ] **R4.1.9:** Testa conectividade HTTPS com APIs → [OK ou WARN]
- [ ] **R4.1.10:** Verifica Named Pipe ativo → [OK]
- [ ] **R4.1.11:** Score >= 80% em instalacao normal
- [ ] **R4.1.12:** Exibe resumo com recomendacoes ao final
- [ ] **R4.1.13:** Nenhuma referencia a AthenaAgent.Tray ou processos V1

---

### Teste 4.2: Monitorar Logs

**Objetivo:** Validar monitoramento simultaneo de Service e Worker

**Procedimento:**

```powershell
.\monitorar-logs.ps1
```

**Validacoes:**

- [ ] **R4.2.1:** Exibe logs de `service-YYYYMMDD.log` e `worker-YYYYMMDD.log`
- [ ] **R4.2.2:** Cores aplicadas: vermelho (ERR), amarelo (WRN), verde (INF)
- [ ] **R4.2.3:** Atualiza em tempo real
- [ ] **R4.2.4:** Filtro `-Filter Error` mostra apenas erros
- [ ] **R4.2.5:** Filtro `-Source Service` mostra apenas logs do servico
- [ ] **R4.2.6:** Filtro `-Source Worker` mostra apenas logs do worker
- [ ] **R4.2.7:** Ctrl+C encerra script graciosamente

---

### Teste 4.3: Coletar Diagnostico

**Objetivo:** Validar coleta de ZIP com informacoes V2

**Procedimento:**

```powershell
.\coletar-diagnostico.ps1
```

**Validacoes:**

- [ ] **R4.3.1:** Script executa sem erros
- [ ] **R4.3.2:** Coleta status do Windows Service
- [ ] **R4.3.3:** Coleta informacoes dos dois processos (Service e SessionWorker)
- [ ] **R4.3.4:** config.json incluido com DeviceToken mascarado
- [ ] **R4.3.5:** Logs de servico e worker incluidos (ultimos 3 de cada)
- [ ] **R4.3.6:** Informacoes do buffer SQLite incluidas
- [ ] **R4.3.7:** Resultado do health-check incluido
- [ ] **R4.3.8:** ZIP criado: `diagnostico-manager-agent-YYYYMMDD-HHMMSS.zip`
- [ ] **R4.3.9:** Explorer abre na pasta do ZIP gerado
- [ ] **R4.3.10:** ZIP pode ser extraido sem erros

---

### Teste 4.4: Limpar/Reset com Confirmacao

**Objetivo:** Validar confirmacao obrigatoria e execucao correta

**Procedimento:**

```powershell
.\limpar-reset.ps1
```

**Validacoes (fluxo "Nao"):**

- [ ] **R4.4.1:** Dialogo de confirmacao aparece antes de qualquer acao
- [ ] **R4.4.2:** Descricao clara das acoes que serao executadas
- [ ] **R4.4.3:** Aviso que a acao nao pode ser desfeita
- [ ] **R4.4.4:** Clicar "Nao": operacao cancelada, Service continua rodando
- [ ] **R4.4.5:** buffer.db nao e modificado apos cancelamento

**Validacoes (fluxo "Sim"):**

- [ ] **R4.4.6:** Service e parado
- [ ] **R4.4.7:** SessionWorker e encerrado
- [ ] **R4.4.8:** buffer.db e removido
- [ ] **R4.4.9:** Logs preservados (padrao)
- [ ] **R4.4.10:** config.json preservado (padrao)
- [ ] **R4.4.11:** Service e reiniciado
- [ ] **R4.4.12:** SessionWorker e relancado pelo Service
- [ ] **R4.4.13:** Icone volta a aparecer na bandeja

```powershell
# Verificar buffer vazio apos reset
sqlite3 "C:\ProgramData\ManagerAgent\buffer.db" "SELECT COUNT(*) FROM eventos"
# Esperado: 0 ou numero muito baixo
```

---

### Teste 4.5: Monitorar Performance

**Objetivo:** Validar monitoramento dos dois processos V2

**Procedimento:**

```powershell
.\monitorar-performance.ps1 -DurationSeconds 120
```

**Validacoes:**

- [ ] **R4.5.1:** Monitora `ManagerAgent.Service` e `ManagerAgent.SessionWorker`
- [ ] **R4.5.2:** Exibe CPU, memoria, threads e handles para cada processo
- [ ] **R4.5.3:** Valores dentro do esperado (Service < 100 MB, Worker < 100 MB)
- [ ] **R4.5.4:** Alertas emitidos se valores anormais
- [ ] **R4.5.5:** Resumo estatistico ao final

---

## Testes de Performance

### Teste 5.1: Consumo de Memoria

**Objetivo:** Validar que os dois processos nao consomem memoria excessiva

**Procedimento:**

```powershell
$inicio = Get-Date
while ((Get-Date) -lt $inicio.AddMinutes(10)) {
    $service = Get-Process -Name "ManagerAgent.Service" -ErrorAction SilentlyContinue
    $worker  = Get-Process -Name "ManagerAgent.SessionWorker" -ErrorAction SilentlyContinue
    $mSvc  = if ($service) { [math]::Round($service.WorkingSet64 / 1MB, 1) } else { "N/A" }
    $mWrk  = if ($worker)  { [math]::Round($worker.WorkingSet64  / 1MB, 1) } else { "N/A" }
    Write-Host "$(Get-Date -Format 'HH:mm:ss') - Service: $mSvc MB | Worker: $mWrk MB"
    Start-Sleep -Seconds 30
}
```

**Validacoes:**

- [ ] **R5.1.1:** Service: memoria inicial <= 100 MB
- [ ] **R5.1.2:** Worker: memoria inicial <= 100 MB
- [ ] **R5.1.3:** Nenhum dos dois cresce continuamente (sem memory leak)
- [ ] **R5.1.4:** Memoria se estabiliza apos inicializacao completa

---

### Teste 5.2: Consumo de CPU

**Objetivo:** Validar que os processos nao consomem CPU excessivamente em idle

**Validacoes:**

- [ ] **R5.2.1:** Service: CPU% no Task Manager < 2% em idle
- [ ] **R5.2.2:** Worker: CPU% no Task Manager < 3% em idle
- [ ] **R5.2.3:** Durante captura ativa: CPU combinado < 10%
- [ ] **R5.2.4:** Nenhum pico constante de CPU

---

### Teste 5.3: Tempo de Inicializacao

**Objetivo:** Validar que Service e Worker iniciam rapidamente

**Procedimento:**

```powershell
$inicio = Get-Date
Start-Service ManagerAgent
# Aguardar icone na bandeja aparecer (manual)
Read-Host "Pressione Enter quando o icone aparecer na bandeja"
$fim = Get-Date
Write-Host "Tempo total: $(($fim - $inicio).TotalSeconds) segundos"
```

**Validacoes:**

- [ ] **R5.3.1:** Service inicia em <= 5 segundos
- [ ] **R5.3.2:** SessionWorker inicia em <= 10 segundos apos Service
- [ ] **R5.3.3:** Icone aparece na bandeja em <= 15 segundos apos `Start-Service`

---

### Teste 5.4: Crescimento do Buffer

**Objetivo:** Validar que buffer.db nao cresce indefinidamente

**Validacoes:**

- [ ] **R5.4.1:** Buffer nao excede 50 MB em uso normal
- [ ] **R5.4.2:** Eventos antigos (> 7 dias) sao removidos automaticamente
- [ ] **R5.4.3:** Apos upload bem-sucedido, eventos sao marcados como enviados
- [ ] **R5.4.4:** VACUUM executado periodicamente (banco compactado)

---

## Testes de Erro e Recuperacao

### Teste 6.1: Servidor Offline

**Objetivo:** Validar comportamento quando backend esta inacessivel

**Procedimento:**

```powershell
# Modificar config.json com URL invalida
$config = Get-Content "C:\ProgramData\ManagerAgent\config.json" | ConvertFrom-Json
$config.baseUrlEvents = "https://servidor-inexistente-12345.com"
$config | ConvertTo-Json | Out-File "C:\ProgramData\ManagerAgent\config.json"
Restart-Service ManagerAgent
Start-Sleep -Seconds 120
```

**Validacoes:**

- [ ] **R6.1.1:** Service continua rodando (nao trava nem para)
- [ ] **R6.1.2:** SessionWorker continua ativo (icone permanece na bandeja)
- [ ] **R6.1.3:** Captura de janelas continua funcionando
- [ ] **R6.1.4:** Eventos armazenados no buffer local
- [ ] **R6.1.5:** Logs indicam falha de upload (nao erro critico)
- [ ] **R6.1.6:** Retry e tentado periodicamente pelo Service
- [ ] **R6.1.7:** Nenhum crash ou exception nao tratada

```powershell
# Restaurar config apos teste
# Editar config.json com URL correta e reiniciar Service
Restart-Service ManagerAgent
```

---

### Teste 6.2: Configuracao Invalida

**Objetivo:** Validar comportamento com config.json malformado

**Procedimento:**

```powershell
# Backup config
Copy-Item "C:\ProgramData\ManagerAgent\config.json" `
          "C:\ProgramData\ManagerAgent\config.json.bak"

# Criar config invalido
"{ json invalido " | Out-File "C:\ProgramData\ManagerAgent\config.json"

Stop-Service ManagerAgent -ErrorAction SilentlyContinue
Start-Service ManagerAgent
Start-Sleep -Seconds 10
```

**Validacoes:**

- [ ] **R6.2.1:** Service detecta config invalido
- [ ] **R6.2.2:** Log de servico indica erro de configuracao
- [ ] **R6.2.3:** Service nao trava indefinidamente (encerra ou usa valores padrao)
- [ ] **R6.2.4:** Mensagem de erro clara nos logs

```powershell
# Restaurar config
Copy-Item "C:\ProgramData\ManagerAgent\config.json.bak" `
          "C:\ProgramData\ManagerAgent\config.json" -Force
Start-Service ManagerAgent
```

---

### Teste 6.3: Buffer Corrompido

**Objetivo:** Validar recuperacao quando buffer.db esta corrompido

**Procedimento:**

```powershell
Stop-Service ManagerAgent
"dados corrompidos" | Out-File "C:\ProgramData\ManagerAgent\buffer.db" -Encoding ASCII -NoNewline
Start-Service ManagerAgent
Start-Sleep -Seconds 10
```

**Validacoes:**

- [ ] **R6.3.1:** Service detecta banco corrompido
- [ ] **R6.3.2:** Log indica problema com banco
- [ ] **R6.3.3:** Service recria banco automaticamente
- [ ] **R6.3.4:** Service e SessionWorker funcionam normalmente apos recriar banco

---

## Testes de Ciclo de Vida do Service

### Teste 7.1: Inicializacao no Boot

**Objetivo:** Validar que o Service inicia automaticamente com o Windows

**Procedimento:**

1. Reiniciar o computador
2. Fazer login
3. Aguardar 30 segundos

**Validacoes:**

```powershell
Get-Service ManagerAgent
# Esperado: Status = Running

Get-Process -Name "ManagerAgent.Service"
Get-Process -Name "ManagerAgent.SessionWorker"
# Esperado: ambos rodando
```

- [ ] **R7.1.1:** Service inicia automaticamente no boot (sem acao do usuario)
- [ ] **R7.1.2:** SessionWorker lancado automaticamente pelo Service apos login
- [ ] **R7.1.3:** Icone aparece na bandeja em <= 30 segundos apos login

---

### Teste 7.2: Parada Graceful do Service

**Objetivo:** Validar encerramento graceful com flush do buffer

**Procedimento:**

```powershell
# Verificar eventos pendentes antes
$antes = sqlite3 "C:\ProgramData\ManagerAgent\buffer.db" `
    "SELECT COUNT(*) FROM eventos WHERE enviado=0"
Write-Host "Eventos pendentes antes: $antes"

Stop-Service ManagerAgent
Start-Sleep -Seconds 5

# Verificar logs de encerramento
Get-Content "C:\ProgramData\ManagerAgent\logs\service-$(Get-Date -Format yyyyMMdd).log" -Tail 20
```

**Validacoes:**

- [ ] **R7.2.1:** Service para sem timeout do SCM (< 30 segundos)
- [ ] **R7.2.2:** SessionWorker e encerrado pelo Service antes de parar
- [ ] **R7.2.3:** Icone desaparece da bandeja apos Service parar
- [ ] **R7.2.4:** Log indica shutdown graceful (sem crash)
- [ ] **R7.2.5:** Buffer nao e corrompido no shutdown

---

### Teste 7.3: Auto-Restart pelo SCM apos Crash

**Objetivo:** Validar que o SCM reinicia o Service apos falha inesperada

**Procedimento:**

```powershell
# Simular crash forcando kill do processo
$proc = Get-Process -Name "ManagerAgent.Service"
Stop-Process -Id $proc.Id -Force
Write-Host "Processo morto. Aguardando restart pelo SCM..."

# SCM deve reiniciar em 1 segundo (primeira falha)
Start-Sleep -Seconds 5
Get-Service ManagerAgent
Get-Process -Name "ManagerAgent.Service" -ErrorAction SilentlyContinue
```

**Validacoes:**

- [ ] **R7.3.1:** SCM detecta que o Service parou inesperadamente
- [ ] **R7.3.2:** Service reinicia em <= 5 segundos (primeira falha: 1s, segunda: 5s)
- [ ] **R7.3.3:** Apos reinicio, SessionWorker e relancado automaticamente
- [ ] **R7.3.4:** Log indica reinicio apos falha
- [ ] **R7.3.5:** Buffer nao e perdido (WAL mode protege contra crash)

---

### Teste 7.4: Reinicio Manual via SCM

**Objetivo:** Validar reinicio limpo via Gerenciador de Servicos

**Procedimento:**

```powershell
Restart-Service ManagerAgent
Start-Sleep -Seconds 15
```

**Validacoes:**

- [ ] **R7.4.1:** Service reinicia sem erros
- [ ] **R7.4.2:** SessionWorker relancado apos reinicio do Service
- [ ] **R7.4.3:** Icone volta a aparecer na bandeja
- [ ] **R7.4.4:** Captura de dados retoma normalmente

---

## Testes Multi-Sessao

### Teste 8.1: Dois Usuarios Simultaneos

**Objetivo:** Validar que cada sessao de usuario tem seu proprio SessionWorker

**Pre-requisito:** Dois usuarios locais configurados no Windows

**Procedimento:**

1. Fazer login com Usuario A (sessao 1)
2. Usar "Switch User" ou RDP para criar sessao do Usuario B (sessao 2)
3. Ambos logados simultaneamente

**Validacoes:**

```powershell
# Verificar multiplos processos SessionWorker
Get-Process -Name "ManagerAgent.SessionWorker" | Select-Object Id, SessionId, UserName
# Esperado: um processo por sessao ativa
```

- [ ] **R8.1.1:** Dois processos `ManagerAgent.SessionWorker` rodando (um por sessao)
- [ ] **R8.1.2:** Named Pipes distintos para cada sessao (`\\.\pipe\ManagerAgent_{sessionId}`)
- [ ] **R8.1.3:** Eventos de janela ativa capturados independentemente por sessao
- [ ] **R8.1.4:** Icone na bandeja visivel para cada usuario em sua sessao
- [ ] **R8.1.5:** Buffer compartilhado (buffer.db) recebe eventos de ambas as sessoes

---

### Teste 8.2: Logoff de Um Usuario (Outro Continua)

**Objetivo:** Validar que encerrar uma sessao nao afeta a outra

**Procedimento:**

1. Dois usuarios logados simultaneamente (Teste 8.1)
2. Usuario B faz logoff

**Validacoes:**

- [ ] **R8.2.1:** SessionWorker da sessao B e encerrado corretamente
- [ ] **R8.2.2:** SessionWorker da sessao A permanece ativo
- [ ] **R8.2.3:** Service permanece ativo
- [ ] **R8.2.4:** Captura de dados continua normalmente para sessao A

---

## Testes de Named Pipe

### Teste 9.1: Comunicacao Normal Service-Worker

**Objetivo:** Validar fluxo de eventos do Worker para o Service via Named Pipe

**Procedimento:**

```powershell
# Monitorar logs de Named Pipe
.\monitorar-logs.ps1 | Select-String "pipe|Pipe|PIPE"
# Em outra janela: alternar janelas para gerar eventos
```

**Validacoes:**

- [ ] **R9.1.1:** Log indica "Pipe connected" apos SessionWorker iniciar
- [ ] **R9.1.2:** Eventos enviados pelo Worker chegam ao Service via pipe
- [ ] **R9.1.3:** Service grava eventos no buffer.db (recebidos via pipe)
- [ ] **R9.1.4:** Latencia de entrega via pipe e imperceptivel (< 1 segundo)

---

### Teste 9.2: Named Pipe - Reconexao apos Interrupcao

**Objetivo:** Validar reconexao automatica quando o pipe e interrompido

**Procedimento:**

```powershell
# Reiniciar apenas o Service (pipe sera encerrado e recriado)
Restart-Service ManagerAgent
Start-Sleep -Seconds 15
```

**Validacoes:**

- [ ] **R9.2.1:** SessionWorker detecta que o pipe foi encerrado
- [ ] **R9.2.2:** SessionWorker entra em modo AutonomousBuffer (bufferiza localmente)
- [ ] **R9.2.3:** Apos Service reiniciar, SessionWorker reconecta ao novo pipe
- [ ] **R9.2.4:** Eventos bufferizados localmente sao enviados apos reconexao
- [ ] **R9.2.5:** Nenhum evento perdido durante a interrupcao

---

### Teste 9.3: ACL do Named Pipe

**Objetivo:** Validar que o Named Pipe aceita apenas o usuario da sessao e SYSTEM

**Validacoes:**

- [ ] **R9.3.1:** Pipe aceita conexoes do usuario da sessao correspondente
- [ ] **R9.3.2:** Pipe aceita conexoes do SYSTEM (Service)
- [ ] **R9.3.3:** Tentativa de conexao de outro usuario e rejeitada
- [ ] **R9.3.4:** Log nao exibe conteudo de eventos (apenas metadados de conexao)

---

## Testes de Auto-Update

### Teste 10.1: Update Plan A - Script PS1 via schtasks

**Objetivo:** Validar fluxo de atualizacao preferencial via schtask PowerShell

**Pre-requisito:** Servidor de atualizacoes configurado com nova versao disponivel

**Procedimento:**

1. Verificar que ha uma versao mais nova disponivel no servidor
2. Aguardar ou acionar verificacao de update
3. Monitorar logs de update

```powershell
Get-Content "C:\ProgramData\ManagerAgent\logs\service-$(Get-Date -Format yyyyMMdd).log" |
    Select-String "update|Update|UPDATE|schtask" | Select-Object -Last 20
```

**Validacoes:**

- [ ] **R10.1.1:** Service detecta nova versao disponivel
- [ ] **R10.1.2:** Download do novo pacote para `C:\ProgramData\ManagerAgent\updates\`
- [ ] **R10.1.3:** schtask criado para executar script PS1 de update
- [ ] **R10.1.4:** Service e SessionWorker encerrados antes da atualizacao
- [ ] **R10.1.5:** Novos binarios copiados corretamente
- [ ] **R10.1.6:** Service reinicia com a nova versao
- [ ] **R10.1.7:** Versao nova confirmada apos restart

```powershell
(Get-Item "C:\Program Files\ManagerAgent\ManagerAgent.Service.exe").VersionInfo.FileVersion
```

---

### Teste 10.2: Update Plan B - Fallback via BAT

**Objetivo:** Validar fallback para BAT via schtasks quando PS1 falha

**Procedimento:**

Simular falha do Plan A (bloquear execucao de PS1 via Execution Policy) e verificar se o Plan B (BAT) e acionado automaticamente.

**Validacoes:**

- [ ] **R10.2.1:** Plan A falha (detectado no log)
- [ ] **R10.2.2:** Service aciona automaticamente o Plan B
- [ ] **R10.2.3:** schtask com script BAT criado
- [ ] **R10.2.4:** Update completa com sucesso via BAT
- [ ] **R10.2.5:** Service reinicia com nova versao

---

### Teste 10.3: Update Plan C - In-Process Rename

**Objetivo:** Validar fallback final quando schtasks nao esta disponivel

**Procedimento:**

Simular falha dos Plans A e B e verificar se o Plan C (rename in-process) e acionado.

**Validacoes:**

- [ ] **R10.3.1:** Plans A e B falham (detectados no log)
- [ ] **R10.3.2:** Service aciona Plan C (in-process rename)
- [ ] **R10.3.3:** Binarios renomeados corretamente
- [ ] **R10.3.4:** Novo Service inicia
- [ ] **R10.3.5:** Log indica qual plan foi utilizado com sucesso

---

### Teste 10.4: Falha Total de Update

**Objetivo:** Validar que o Service continua operacional se todos os plans de update falham

**Validacoes:**

- [ ] **R10.4.1:** Todos os 3 plans falham (simulado)
- [ ] **R10.4.2:** Service continua rodando com versao atual
- [ ] **R10.4.3:** Log indica falha em todos os plans com detalhes
- [ ] **R10.4.4:** Captura de dados continua funcionando normalmente

---

## Testes de Migracao V1 para V2

### Teste 11.1: Upgrade de Instalacao V1 Existente

**Objetivo:** Validar que instalacoes V1 fazem upgrade corretamente para V2

**Pre-requisito:** Maquina com Manager Agent V1 instalado (com `AthenaAgent.Tray.exe`)

**Procedimento:**

```powershell
# Verificar estado V1 antes
Get-Process -Name "AthenaAgent.Tray" -ErrorAction SilentlyContinue
Test-Path "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup\Manager Agent.lnk"

# Executar instalador V2
Start-Process "ManagerAgent-Setup.exe" -Verb RunAs
```

**Validacoes:**

- [ ] **R11.1.1:** Instalador V2 detecta instalacao V1 existente
- [ ] **R11.1.2:** Processo `AthenaAgent.Tray` e encerrado antes da migracao
- [ ] **R11.1.3:** Atalho de Startup da V1 e removido
- [ ] **R11.1.4:** config.json da V1 e migrado para o formato V2
- [ ] **R11.1.5:** `instalacaoId` da V1 e preservado (nao regera novo GUID)
- [ ] **R11.1.6:** Windows Service V2 e registrado
- [ ] **R11.1.7:** Service V2 inicia e lanca SessionWorker
- [ ] **R11.1.8:** Nenhum processo `AthenaAgent.Tray` restante

```powershell
# Validacao pos-migracao
Get-Process -Name "AthenaAgent.Tray" -ErrorAction SilentlyContinue
# Esperado: Vazio

Get-Service ManagerAgent
# Esperado: Running

Get-Process -Name "ManagerAgent.Service", "ManagerAgent.SessionWorker"
# Esperado: ambos rodando
```

---

### Teste 11.2: Preservacao de Dados na Migracao

**Objetivo:** Validar que dados existentes (buffer, config) sao preservados

**Validacoes:**

- [ ] **R11.2.1:** buffer.db da V1 e aproveitado ou migrado sem perda de dados
- [ ] **R11.2.2:** Historico de eventos nao e perdido na migracao
- [ ] **R11.2.3:** Parametros de configuracao (chave, identificador) migrados corretamente
- [ ] **R11.2.4:** DeviceToken re-encriptado com DPAPI se necessario

---

### Teste 11.3: Rollback em Caso de Falha na Migracao

**Objetivo:** Validar que instalacao V1 e preservada se a migracao V2 falhar

**Validacoes:**

- [ ] **R11.3.1:** Se migracao falhar, instalador restaura estado V1
- [ ] **R11.3.2:** Processo `AthenaAgent.Tray` volta a rodar
- [ ] **R11.3.3:** config.json V1 nao e corrompido
- [ ] **R11.3.4:** Log indica motivo da falha da migracao

---

## Testes de AutonomousBuffer

### Teste 12.1: Bufferizacao Local quando Pipe esta Indisponivel

**Objetivo:** Validar que SessionWorker bufferiza eventos localmente quando o pipe esta down

**Procedimento:**

```powershell
# Parar apenas o Service (Worker continua rodando)
Stop-Service ManagerAgent -NoWait
Start-Sleep -Seconds 2

# Worker esta rodando sem pipe por 30 segundos
# Gerar eventos: alternar janelas ativas
Start-Sleep -Seconds 30

# Verificar logs do Worker
Get-Content "C:\ProgramData\ManagerAgent\logs\worker-$(Get-Date -Format yyyyMMdd).log" -Tail 20 |
    Select-String "buffer|autonomous|pipe"
```

**Validacoes:**

- [ ] **R12.1.1:** Worker detecta que o pipe esta indisponivel
- [ ] **R12.1.2:** Worker entra em modo AutonomousBuffer (log indica transicao)
- [ ] **R12.1.3:** Worker continua capturando eventos durante modo autonomo
- [ ] **R12.1.4:** Eventos bufferizados localmente pelo Worker (nao perdidos)

---

### Teste 12.2: Flush do Buffer Autonomo apos Reconexao

**Objetivo:** Validar que eventos bufferizados sao entregues ao Service apos reconexao

**Procedimento:**

```powershell
# Continuar do Teste 12.1: reiniciar o Service
Start-Service ManagerAgent
Start-Sleep -Seconds 15

# Verificar que eventos chegaram ao buffer.db
sqlite3 "C:\ProgramData\ManagerAgent\buffer.db" `
    "SELECT COUNT(*) FROM eventos WHERE enviado=0"
```

**Validacoes:**

- [ ] **R12.2.1:** Worker detecta que o pipe foi restabelecido
- [ ] **R12.2.2:** Worker envia eventos bufferizados ao Service via pipe
- [ ] **R12.2.3:** Service grava eventos no buffer.db
- [ ] **R12.2.4:** Nenhum evento duplicado no buffer.db
- [ ] **R12.2.5:** Log indica saida do modo AutonomousBuffer

---

### Teste 12.3: Limite do Buffer Autonomo

**Objetivo:** Validar comportamento quando buffer autonomo atinge limite de capacidade

**Validacoes:**

- [ ] **R12.3.1:** Worker nao consome memoria excessiva durante modo autonomo longo
- [ ] **R12.3.2:** Se buffer atingir limite configurado, eventos mais antigos sao descartados
- [ ] **R12.3.3:** Log indica descarte de eventos com contagem

---

## Testes de Worker Watchdog

### Teste 13.1: Deteccao de Worker Morto

**Objetivo:** Validar que o Service detecta quando um SessionWorker para de responder

**Procedimento:**

```powershell
# Forcar kill do SessionWorker (simular crash)
$worker = Get-Process -Name "ManagerAgent.SessionWorker"
Stop-Process -Id $worker.Id -Force
Write-Host "Worker morto. Aguardando watchdog do Service..."
Start-Sleep -Seconds 15

# Verificar se Service relancou o Worker
Get-Process -Name "ManagerAgent.SessionWorker" -ErrorAction SilentlyContinue
```

**Validacoes:**

- [ ] **R13.1.1:** Service detecta que o Worker nao esta mais respondendo
- [ ] **R13.1.2:** Log do Service indica deteccao de worker morto
- [ ] **R13.1.3:** Service relanca automaticamente o SessionWorker
- [ ] **R13.1.4:** Novo Worker aparece em <= 15 segundos
- [ ] **R13.1.5:** Icone volta a aparecer na bandeja do usuario
- [ ] **R13.1.6:** Captura de dados retoma apos relancamento

---

### Teste 13.2: Multiplos Crashes do Worker

**Objetivo:** Validar comportamento com crashes repetidos do Worker

**Procedimento:**

```powershell
# Simular 3 crashes consecutivos com intervalo de 10 segundos
for ($i = 1; $i -le 3; $i++) {
    $worker = Get-Process -Name "ManagerAgent.SessionWorker" -ErrorAction SilentlyContinue
    if ($worker) {
        Stop-Process -Id $worker.Id -Force
        Write-Host "Crash $i simulado"
    }
    Start-Sleep -Seconds 10
}
```

**Validacoes:**

- [ ] **R13.2.1:** Service relanca Worker apos cada crash
- [ ] **R13.2.2:** Log registra cada crash e relancamento
- [ ] **R13.2.3:** Apos multiplos crashes, Service nao para de tentar relancar
- [ ] **R13.2.4:** Service permanece estavel durante crashes repetidos do Worker

---

### Teste 13.3: Transicao de Meia-Noite

**Objetivo:** Validar que a transicao diaria de meia-noite e tratada corretamente

**Validacoes:**

- [ ] **R13.3.1:** Novo arquivo de log criado ao virar o dia (`service-YYYYMMDD.log`, `worker-YYYYMMDD.log`)
- [ ] **R13.3.2:** Log do dia anterior e fechado corretamente
- [ ] **R13.3.3:** Sessoes ativas de captura nao sao interrompidas pela transicao
- [ ] **R13.3.4:** Eventos do novo dia tem timestamps corretos

---

## Testes de Desinstalacao

### Teste 14.1: Desinstalacao Completa via Inno Setup

**Objetivo:** Validar remocao completa do agente V2

**Procedimento:**

```powershell
# Via Painel de Controle ou Configuracoes → Aplicativos
# Ou via executavel de desinstalacao do Inno Setup
```

**Validacoes:**

- [ ] **R14.1.1:** Windows Service "ManagerAgent" parado antes da remocao
- [ ] **R14.1.2:** Processos Service e SessionWorker encerrados
- [ ] **R14.1.3:** Windows Service desregistrado do SCM
- [ ] **R14.1.4:** Pasta `C:\Program Files\ManagerAgent` removida
- [ ] **R14.1.5:** Opcao de preservar dados do usuario (config.json, buffer.db) oferecida
- [ ] **R14.1.6:** Apos reboot, Service nao inicia automaticamente
- [ ] **R14.1.7:** Nenhum processo ManagerAgent.* restante

```powershell
# Verificacao final
Get-Service ManagerAgent -ErrorAction SilentlyContinue
# Esperado: Vazio

Test-Path "C:\Program Files\ManagerAgent"
# Esperado: False

Get-Process -Name "ManagerAgent.*" -ErrorAction SilentlyContinue
# Esperado: Vazio
```

---

### Teste 14.2: Reinstalacao Apos Desinstalacao

**Objetivo:** Validar que reinstalacao funciona apos desinstalacao completa

**Procedimento:**

1. Executar desinstalacao completa (Teste 14.1)
2. Reinstalar com o mesmo instalador (Teste 1.1)

**Validacoes:**

- [ ] **R14.2.1:** Instalacao funciona sem erros ou conflitos
- [ ] **R14.2.2:** Novo `instalacaoId` gerado (UUID diferente do anterior)
- [ ] **R14.2.3:** Service registrado e iniciado corretamente
- [ ] **R14.2.4:** Agente funciona normalmente apos reinstalacao

---

## Relatorio Final

### Template de Relatorio

```
RELATORIO DE TESTES REGRESSIVOS - Manager Agent V2

Data: [DATA]
Testador: [NOME]
Versao Testada: 2.x.x
Sistema: Windows [VERSAO]

RESUMO
Total de Testes: X
Testes Passaram: Y (Z%)
Testes Falharam: W

RESULTADOS POR CATEGORIA

1. Instalacao
   Executados: N | Passou: N | Falhou: N

2. Interface (System Tray)
   Executados: N | Passou: N | Falhou: N

3. Captura de Dados
   Executados: N | Passou: N | Falhou: N

4. Scripts Diagnosticos
   Executados: N | Passou: N | Falhou: N

5. Performance
   Executados: N | Passou: N | Falhou: N

6. Erro e Recuperacao
   Executados: N | Passou: N | Falhou: N

7. Ciclo de Vida do Service
   Executados: N | Passou: N | Falhou: N

8. Multi-Sessao
   Executados: N | Passou: N | Falhou: N

9. Named Pipe
   Executados: N | Passou: N | Falhou: N

10. Auto-Update
    Executados: N | Passou: N | Falhou: N

11. Migracao V1 para V2
    Executados: N | Passou: N | Falhou: N

12. AutonomousBuffer
    Executados: N | Passou: N | Falhou: N

13. Worker Watchdog
    Executados: N | Passou: N | Falhou: N

14. Desinstalacao
    Executados: N | Passou: N | Falhou: N

BUGS ENCONTRADOS

Bug #1: [Titulo]
  Severidade: Critico / Alto / Medio / Baixo
  Passos para reproduzir:
    1. ...
    2. ...
  Resultado esperado: ...
  Resultado obtido: ...
  Log/Screenshot: [Anexar]

APROVACAO
[ ] APROVADO para release
[ ] APROVADO COM RESSALVAS (bugs nao-criticos)
[ ] REPROVADO (bugs criticos impedem release)

Assinatura: ________________
Data: ________________
```

---

## Metricas de Sucesso

### Criterios de Aprovacao

| Categoria | Taxa Minima de Sucesso |
|-----------|------------------------|
| Instalacao | 100% |
| Interface System Tray | 95% |
| Captura de Dados | 90% |
| Scripts Diagnosticos | 90% |
| Performance | 100% |
| Erro e Recuperacao | 80% |
| Ciclo de Vida do Service | 100% |
| Multi-Sessao | 90% |
| Named Pipe | 95% |
| Auto-Update | 80% |
| Migracao V1 para V2 | 100% |
| AutonomousBuffer | 90% |
| Worker Watchdog | 95% |
| Desinstalacao | 100% |

**Taxa Global:** >= 90%

### Bugs Criticos (Bloqueiam Release)

- Service nao inicia ou para inesperadamente sem reiniciar
- SessionWorker nao e lancado pelo Service
- Named Pipe nao estabelece comunicacao
- Perda de dados no buffer durante shutdown normal
- Migracao V1 para V2 corrompe config.json ou perde instalacaoId
- Consumo excessivo de recursos (> 300 MB RAM combinado ou > 20% CPU combinado)
- Impossibilidade de desinstalar

### Bugs Nao-Criticos (Nao Bloqueiam)

- Bugs cosmeticos no menu da bandeja
- Mensagens de log pouco descritivas (mas funcionais)
- Delay > 15 segundos para icone aparecer (porem funcional)
- Scripts diagnosticos com pequenas inconsistencias de exibicao

---

## Smoke Test (15 minutos)

Use este checklist para validacao rapida antes de testes completos:

```
SMOKE TEST - MANAGER AGENT V2

1. Instalacao
   [ ] Instalar via ManagerAgent-Setup.exe
   [ ] Service "ManagerAgent" rodando
   [ ] Icone aparece na bandeja

2. Interface
   [ ] Clique direito → Menu abre com estrutura correta
   [ ] Cabecalho exibe "iManager - vX.Y.Z"
   [ ] Submenu Ferramentas abre com 4 itens

3. Captura
   [ ] Alternar 3 janelas diferentes
   [ ] Verificar eventos no buffer.db

4. Health Check
   [ ] Executar via menu: Ferramentas → Verificacao de Saude
   [ ] Score >= 80%
   [ ] Verifica Service, Worker e Named Pipe

5. Performance
   [ ] Service < 100 MB RAM
   [ ] Worker < 100 MB RAM
   [ ] CPU combinado < 10%

6. Watchdog
   [ ] Forcar kill do SessionWorker
   [ ] Worker relancado automaticamente em <= 15 segundos

7. Desinstalacao
   [ ] Desinstalar via Painel de Controle
   [ ] Service desregistrado
   [ ] Processos encerrados

Se todos passarem: APROVADO para testes completos
```

---

---

## Scripts PowerShell Uteis (Referencia Rapida para QA)

> Esta secao consolida o SCRIPTS-DIAGNOSTICO.md (absorvido em 2026-06-11).

### Localizacao

```
C:\Program Files\ManagerAgent\scripts\
├── health-check.ps1
├── monitorar-logs.ps1
├── coletar-diagnostico.ps1
├── limpar-reset.ps1
└── monitorar-performance.ps1
```

Execucao: `cd "C:\Program Files\ManagerAgent\scripts"` + `.\nome-do-script.ps1`

### Scripts disponiveis

**`health-check.ps1`** — Status geral do agente (service + worker + config + conectividade + SQLite + named pipe). Retorna score 0-100% com status SAUDAVEL / ATENCAO / CRITICO. Usar como primeiro passo em qualquer investigacao.

```powershell
.\health-check.ps1
```

**`monitorar-logs.ps1`** — Monitor em tempo real com cores (ERR vermelho, WRN amarelo, INF ciano). Monitora service e worker simultaneamente.

```powershell
.\monitorar-logs.ps1                        # todos os logs
.\monitorar-logs.ps1 -Filter Error          # apenas erros
.\monitorar-logs.ps1 -Filter Warning        # apenas avisos
.\monitorar-logs.ps1 -TailLines 100         # iniciar com 100 linhas de historico
.\monitorar-logs.ps1 -Source Service        # apenas servico
.\monitorar-logs.ps1 -Source Worker         # apenas worker
```

**`coletar-diagnostico.ps1`** — Coleta completa para suporte: info do sistema, status do service, processos, config (dados sensiveis mascarados), logs recentes, status SQLite, named pipe, resultado do health-check, testes de conectividade. Output: ZIP em `%TEMP%`.

```powershell
.\coletar-diagnostico.ps1
```

**`limpar-reset.ps1`** — Reset do agente sem desinstalar (para comportamento estranho ou buffer corrompido). Para o service, remove `buffer.db`, opcionalmente remove config/logs, reinicia o service.

```powershell
.\limpar-reset.ps1                          # limpar tudo (pede confirmacao)
.\limpar-reset.ps1 -KeepConfig              # manter configuracao (apenas limpa buffer)
.\limpar-reset.ps1 -KeepLogs                # manter logs
.\limpar-reset.ps1 -KeepConfig -KeepLogs    # manter ambos
.\limpar-reset.ps1 -Force                   # sem confirmacao interativa
```

ATENCAO: eventos no buffer SQLite serao perdidos permanentemente.

**`monitorar-performance.ps1`** — Monitor de CPU, memoria, threads e handles. Detecta memory leak (crescimento continuo). Alerta fora do esperado.

```powershell
.\monitorar-performance.ps1                              # 60 segundos (padrao)
.\monitorar-performance.ps1 -DurationSeconds 300         # 5 minutos
.\monitorar-performance.ps1 -DurationSeconds 600 -IntervalSeconds 10  # 10 min, medicoes a cada 10s
```

Valores esperados V2:

| Processo | Memoria | Threads | CPU (idle) |
|----------|---------|---------|------------|
| ManagerAgent.Service | < 100 MB | < 30 | < 2% |
| ManagerAgent.SessionWorker | < 100 MB | < 30 | < 3% |

### Quick Reference

| Situacao | Script | Comando |
|----------|--------|---------|
| Status geral | health-check | `.\health-check.ps1` |
| Ver logs ao vivo | monitorar-logs | `.\monitorar-logs.ps1` |
| Ver so erros | monitorar-logs | `.\monitorar-logs.ps1 -Filter Error` |
| Coletar para suporte | coletar-diagnostico | `.\coletar-diagnostico.ps1` |
| Reset sem perder config | limpar-reset | `.\limpar-reset.ps1 -KeepConfig` |
| Monitor performance 5min | monitorar-performance | `.\monitorar-performance.ps1 -DurationSeconds 300` |

### Fluxo de troubleshooting recomendado

**"O agente nao esta funcionando":**
1. `health-check.ps1` — score < 80%? Analisar problemas listados
2. `monitorar-logs.ps1 -Filter Error` — identificar componente com falha
3. Verificar Named Pipe nos logs (buscar "Pipe connected")
4. `coletar-diagnostico.ps1` + enviar ZIP para analise
5. `monitorar-performance.ps1 -DurationSeconds 300` — verificar consumo

**Service nao inicia:**
```powershell
Get-Service ManagerAgent
Get-Content "C:\ProgramData\ManagerAgent\logs\service-$(Get-Date -Format yyyyMMdd).log" -Tail 50
Start-Service ManagerAgent
Get-EventLog -LogName System -Source "Service Control Manager" -Newest 10
```

**SessionWorker nao aparece no tray:**
```powershell
Get-Service ManagerAgent
Get-Process -Name "ManagerAgent.SessionWorker"
Get-Content "C:\ProgramData\ManagerAgent\logs\service-$(Get-Date -Format yyyyMMdd).log" -Tail 100 |
    Select-String "SessionWorker|worker|launch"
```

**Problemas com Execution Policy:**
```powershell
Get-ExecutionPolicy -Scope CurrentUser
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned -Force
```

---

**Ultima atualizacao:** 2026-06-11
