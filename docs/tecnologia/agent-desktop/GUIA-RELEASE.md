> **STATUS:** ATIVO
> **DATA:** 2026-06-11
> **DONO:** @Tony (TL)
> **REVISADO POR:** @Tony

# Guia de Release — Manager Agent

**Versao:** 2.0.0
**Data:** 2026-06-11
**Objetivo:** Passo a passo completo para criar e publicar uma nova versao do Manager Agent V2.

**Nota CI/CD:** A automacao de build via GitHub Actions esta documentada em `../../../specs/MVP2/ci-cd-github-actions.md`. Quando implementada, os passos de build manual serao substituidos pelo pipeline automatico (GitHub Actions + ghcr.io + webhook Coolify).

---

## Indice

1. [Pre-requisitos](#pre-requisitos)
2. [Bump de Versao](#1-bump-de-versao)
3. [Build do Pacote](#2-build-do-pacote)
4. [Verificar Checksums](#3-verificar-checksums)
5. [Testes Pre-Regressivos](#4-testes-pre-regressivos)
6. [Commit e Tag](#5-commit-e-tag)
7. [Deploy manager-srv-admin](#6-deploy-manager-srv-admin)
8. [Deploy manager-srv-events](#7-deploy-manager-srv-events)
9. [Upload do Instalador ao Servidor de Atualizacao](#8-upload-do-instalador-ao-servidor-de-atualizacao)
10. [Validacao Pos-Deploy](#9-validacao-pos-deploy)
11. [Distribuicao ao Cliente](#10-distribuicao-ao-cliente)

---

## Pre-requisitos

### Ferramentas

- [ ] .NET 8 SDK instalado (`dotnet --version`)
- [ ] PowerShell 5.1 ou superior
- [ ] Inno Setup 6 instalado (necessario para `build-pacote-instalacao.ps1`)
- [ ] `git` instalado e configurado
- [ ] Acesso ao servidor de deploy (Hetzner + Coolify ou equivalente)

### Acessos

- [ ] Acesso ao repositorio GitHub com permissao de push e criacao de tags
- [ ] Acesso ao painel de deploy de `manager-srv-admin` e `manager-srv-events`
- [ ] Chave de ativacao UUID da empresa-alvo (para embutir no instalador, se aplicavel)

---

## 1. Bump de Versao

Editar a propriedade `<Version>` nos quatro projetos:

```
src\ManagerAgent.Service\ManagerAgent.Service.csproj
src\ManagerAgent.SessionWorker\ManagerAgent.SessionWorker.csproj
src\ManagerAgent.Shared\ManagerAgent.Shared.csproj
src\ManagerAgent.Installer\ManagerAgent.Installer.csproj
```

Alterar para a nova versao seguindo [SemVer](https://semver.org/):

```xml
<Version>X.Y.Z</Version>
```

Regra de versionamento:
- **MAJOR** — quebras de compatibilidade com protocolo de pipe ou formato de config
- **MINOR** — novas funcionalidades retrocompativeis
- **PATCH** — correcoes de bugs

---

## 2. Build do Pacote

A partir do diretorio raiz do repositorio (`manager-srv-agent`):

```powershell
.\scripts\build\build-pacote-instalacao.ps1
```

O script realiza as seguintes etapas:

1. Publica `ManagerAgent.Service` em Release/win-x64 self-contained para `dist/service`
2. Publica `ManagerAgent.SessionWorker` em Release/win-x64 self-contained para `dist/worker`
3. Compila `ManagerAgent.Installer` (DLL auxiliar do Inno Setup)
4. Executa o script Inno Setup para gerar o instalador EXE
5. Gera o arquivo de checksum SHA-256 do EXE

Verificar que o build terminou sem erros:

```
Build: 0 errors, 0 warnings
```

### Build manual por componente

Caso necessario publicar individualmente:

```powershell
# Service
dotnet publish src/ManagerAgent.Service/ManagerAgent.Service.csproj `
    -c Release -r win-x64 --self-contained true -o dist/service

# SessionWorker
dotnet publish src/ManagerAgent.SessionWorker/ManagerAgent.SessionWorker.csproj `
    -c Release -r win-x64 --self-contained true -o dist/worker
```

---

## 3. Verificar Checksums

Confirmar que o arquivo de checksum foi gerado corretamente:

```powershell
Get-Content "dist\checksum-vX.Y.Z.txt"
# Deve exibir o hash SHA-256 do instalador EXE
```

Comparar manualmente com o EXE gerado para validar integridade:

```powershell
Get-FileHash "dist\ManagerAgent-Installer-vX.Y.Z.exe" -Algorithm SHA256
```

Os dois valores devem ser identicos.

---

## 4. Testes Pre-Regressivos

Antes de fazer o release, executar os testes automatizados:

```powershell
dotnet test tests/ManagerAgent.Service.Tests/ManagerAgent.Service.Tests.csproj
dotnet test tests/ManagerAgent.SessionWorker.Tests/ManagerAgent.SessionWorker.Tests.csproj
```

Todos os testes devem passar sem falhas.

Em seguida, consultar [PLANO-TESTES-REGRESSIVOS.md](PLANO-TESTES-REGRESSIVOS.md) para o roteiro completo de testes manuais em maquina limpa, incluindo:

- Instalacao limpa em VM ou maquina de teste Windows
- Verificacao que o Service e registrado e inicia automaticamente (Session 0)
- Verificacao que o SessionWorker e lancado pelo Service na sessao interativa
- Icone da bandeja aparece e menu responde conforme `menuVisivel`
- Vinculacao do agente ao backend de staging via `AgentLinkService`
- Captura de janelas ativas nos logs (`WindowActivityService`)
- Upload de eventos chegando ao backend (`UploadWorker`)
- Heartbeat registrado nos logs (`HeartbeatService`)
- Deteccao de ociosidade funcionando (`IdleMonitorService`)
- Comunicacao via Named Pipe (Service <-> SessionWorker) sem erros
- Fallback para `AutonomousBuffer` quando pipe nao disponivel
- Auto-atualizacao: simular endpoint retornando versao nova, verificar fluxo Plano A/B/C
- Desinstalacao limpa sem residuos em `C:\ProgramData\ManagerAgent`

---

## 5. Commit e Tag

```powershell
git add src/ManagerAgent.Service/ManagerAgent.Service.csproj
git add src/ManagerAgent.SessionWorker/ManagerAgent.SessionWorker.csproj
git add src/ManagerAgent.Shared/ManagerAgent.Shared.csproj
git add src/ManagerAgent.Installer/ManagerAgent.Installer.csproj
git commit -m "release: vX.Y.Z"
git tag vX.Y.Z
git push
git push --tags
```

Usar o mesmo numero de versao do `.csproj` no nome da tag.

---

## 6. Deploy manager-srv-admin

Acessar o painel Coolify (ou provedor de deploy configurado) e fazer deploy da imagem atualizada de `manager-srv-admin`.

Apos o deploy, verificar saude:

```
GET https://admin.imanagerportal.com/actuator/health
# Esperado: {"status":"UP"}
```

---

## 7. Deploy manager-srv-events

Acessar o painel e fazer deploy da imagem atualizada de `manager-srv-events`.

Verificar saude:

```
GET https://api-events.imanagerportal.com/actuator/health
# Esperado: {"status":"UP"}
```

---

## 8. Upload do Instalador ao Servidor de Atualizacao

O mecanismo de auto-atualizacao do agente funciona da seguinte forma:

- O `UpdateCheckerWorker` consulta `{baseUrlAdmin}/api/agente/atualizacoes/verificar` a cada 6 horas
- O endpoint retorna a versao mais recente disponivel e a URL de download do ZIP
- O `UpdateDownloader` baixa o ZIP e verifica o hash SHA-256
- O `UpdateApplier` aplica a atualizacao via:
  - **Plano A** — schtask com script PS1
  - **Plano B** — schtask com script BAT (fallback)
  - **Plano C** — rename in-process (fallback final)
- Arquivos de flag utilizados: `update-in-progress.flag`, `update-success.flag`, `update-failed.flag`

### Procedimento de upload

1. Empacotar o instalador EXE e o checksum em um ZIP de distribuicao (se aplicavel ao servidor)
2. Fazer upload para o endpoint de distribuicao configurado no `manager-srv-admin`
3. Atualizar o registro de versao no backend para que o endpoint `/api/agente/atualizacoes/verificar` retorne a nova versao

Verificar que o endpoint retorna a nova versao:

```
GET https://admin.imanagerportal.com/api/agente/versao
# Esperado: {"versao":"X.Y.Z","url":"https://..."}
```

---

## 9. Validacao Pos-Deploy

- [ ] Health check `manager-srv-admin` retorna `UP`
- [ ] Health check `manager-srv-events` retorna `UP`
- [ ] Endpoint de versao retorna a nova versao `X.Y.Z`
- [ ] Vincular um dispositivo de teste ao backend de producao e verificar:
  - [ ] Service inicia e registra no log: `[INF] [Service] [AgentLinkService] Dispositivo vinculado`
  - [ ] SessionWorker e lancado pelo Service e aparece na bandeja
  - [ ] Upload de eventos chega ao banco (verificar via painel admin)
  - [ ] Heartbeat registrado nos logs do worker
- [ ] Agentes existentes recebem a notificacao de nova versao em ate 6 horas
- [ ] Processo de auto-atualizacao concluido com sucesso (verificar `update-success.flag`)

---

## 10. Distribuicao ao Cliente

Para instalacao em novos dispositivos ou reinstalacao:

1. Enviar o instalador EXE ao cliente:
   ```
   dist\ManagerAgent-Installer-vX.Y.Z.exe
   ```

2. Enviar o checksum correspondente para que o cliente possa validar a integridade:
   ```
   dist\checksum-vX.Y.Z.txt
   ```

3. Orientar o cliente a executar o instalador como Administrador:
   ```
   Clicar com botao direito em ManagerAgent-Installer-vX.Y.Z.exe > Executar como Administrador
   ```

4. O instalador registra o Windows Service, copia os binarios e executa o `SelfTest` pos-instalacao.

5. Confirmar com o cliente que:
   - O icone da bandeja apareceu na sessao do usuario
   - O agente foi vinculado com sucesso (verificar logs em `C:\ProgramData\ManagerAgent\logs\`)

---

## Referencia: Caminhos Importantes

| Item | Caminho |
|------|---------|
| Configuracao | `C:\ProgramData\ManagerAgent\config.json` |
| Logs do Service | `C:\ProgramData\ManagerAgent\logs\service-YYYYMMDD.log` |
| Logs do Worker | `C:\ProgramData\ManagerAgent\logs\worker-YYYYMMDD.log` |
| Buffer SQLite | `C:\ProgramData\ManagerAgent\eventos.db` |
| Flag de atualizacao | `C:\ProgramData\ManagerAgent\update-*.flag` |
| Script de build | `scripts\build\build-pacote-instalacao.ps1` |

---

## Documentos Relacionados

- [PLANO-TESTES-REGRESSIVOS.md](PLANO-TESTES-REGRESSIVOS.md) — Roteiro detalhado de testes manuais
- [MANUAL-COMPLETO.md](MANUAL-COMPLETO.md) — Referencia completa do agente (inclui arquitetura, instalacao, dev)

---

## Checklist Pre-Release (antes de distribuir)

> Absorvido do CHECKLIST-PRODUCAO.md em 2026-06-11.

### 1. Codigo e Build

- [ ] Codigo compila sem erros: `dotnet build ManagerAgent.sln -c Release` → 0 erros, 0 avisos
- [ ] Todos os testes passam: `.\test-vinculacao.ps1` → todos [OK]
- [ ] Versao atualizada em `.csproj` (Service e SessionWorker): `<Version>X.Y.Z</Version>`
- [ ] Changelog atualizado no commit de release

### 2. Configuracao

- [ ] URLs de producao corretas em `ConfigPaths.cs`: `BaseUrlAdmin = "https://admin.imanagerportal.com"` e `BaseUrlEvents = "https://api-events.imanagerportal.com"`
- [ ] Chave de ativacao valida (testar com `.\test-vinculacao.ps1`)
- [ ] Endpoint de auto-update configurado e respondendo
- [ ] `config.json` com estrutura correta (campos: `baseUrlAdmin`, `baseUrlEvents`, `chaveAtivacaoEmpresa`, `identificadorColaborador`, `instalacaoId`, `deviceToken`, `menuVisivel`)

### 3. Testes Locais

- [ ] Servico inicia: `Start-Service ManagerAgent` + `Get-Service ManagerAgent` → Status `Running`
- [ ] SessionWorker iniciado pelo servico: `Get-Process -Name "ManagerAgent.SessionWorker"` → ativo
- [ ] Captura janelas reais (log: `[INF] Evento de janela iniciado: chrome`)
- [ ] Upload funciona (log: `[INF] Upload realizado com sucesso`)
- [ ] Heartbeats sendo enviados (log: `[INF] Heartbeat ManagerAgent`)
- [ ] Deteccao de idle funciona (deixar idle 5 min, verificar logs)
- [ ] Logs escritos em ambos os arquivos (`service-YYYYMMDD.log` e `worker-YYYYMMDD.log`)
- [ ] Logs rotacionam diariamente
- [ ] Comunicacao via named pipe sem erros nos logs

### 4. Pacote de Instalacao

- [ ] `.\build-pacote-instalacao.ps1` executa sem erros
- [ ] Instalador gerado: `instalador/ManagerAgent-Setup.exe` (~20-30 MB)
- [ ] Conteudo do instalador inclui todos os binarios e scripts de diagnostico

### 5. Teste de Instalacao (Maquina Limpa)

- [ ] Executar `ManagerAgent-Setup.exe` como Admin → instalacao completa sem erros
- [ ] Binarios em `C:\Program Files\ManagerAgent\` + dados em `C:\ProgramData\ManagerAgent\`
- [ ] Servico registrado: `Get-Service ManagerAgent` → `StartType: Automatic`, `Status: Running`
- [ ] Recuperacao SCM configurada (1s, 5s, 30s)
- [ ] `config.json` criado com `instalacaoId` GUID + `deviceToken` criptografado DPAPI
- [ ] SessionWorker iniciado automaticamente
- [ ] `test-vinculacao.ps1` → tudo OK
- [ ] Logs mostram captura real de janelas + upload funciona

### 6. Teste de Migracao V1 → V2

- [ ] Instalar sobre maquina com V1 (se aplicavel)
- [ ] `MigrationV1ToV2` executada automaticamente
- [ ] Dados preservados (`config.json`, `buffer.db`, logs)
- [ ] Nova versao inicia corretamente com mesma vinculacao (mesmo `instalacaoId`)
- [ ] Nenhuma referencia ao executavel antigo remanescente

### 7. Teste de Desinstalacao

- [ ] `"C:\Program Files\ManagerAgent\unins000.exe" /VERYSILENT` executa sem erros
- [ ] Servico parado e removido do SCM
- [ ] Processos encerrados (Service + SessionWorker)
- [ ] Binarios removidos de `C:\Program Files\ManagerAgent\`

### 8. Teste de Multi-sessao

- [ ] 2+ usuarios logados simultaneamente → um SessionWorker por usuario ativo
- [ ] Logs de cada usuario separados
- [ ] Captura de janelas independente por sessao

### 9. Teste de Auto-Update

- [ ] Agente verifica atualizacao a cada 6 horas (log: `[INF] Verificando atualizacoes`)
- [ ] Endpoint de atualizacao respondendo
- [ ] Mecanismo fallback (Plano A/B/C) configurado
- [ ] Atualizacao silenciosa sem interrupcao perceptivel

### 10. Seguranca

- [ ] Nenhuma senha ou credencial hardcoded
- [ ] HTTPS obrigatorio em todas as chamadas ao backend
- [ ] DeviceToken criptografado com DPAPI (verificar `config.json`)
- [ ] Named pipe com ACL restrita (usuario da sessao + SYSTEM apenas)
- [ ] Servico rodando como SYSTEM
- [ ] Logs nao expõem dados sensiveis
- [ ] Sem debug leftovers (`Console.WriteLine`, `#if DEBUG`)

### 11. Performance

- [ ] CPU do servico em idle < 1%: `Get-Process -Name "ManagerAgent.Service" | Select-Object CPU, WorkingSet`
- [ ] CPU do SessionWorker em idle < 1%
- [ ] RAM total (Service + SessionWorker) < 100 MB em idle
- [ ] Sem memory leak (rodar 1 hora, RAM constante)
- [ ] Buffer SQLite nao cresce indefinidamente apos uploads bem-sucedidos

### 12. Compatibilidade

- [ ] Funciona no Windows 10 (64-bit)
- [ ] Funciona no Windows 11 (64-bit)
- [ ] Funciona com UAC habilitado
- [ ] Funciona em conta de usuario padrao (SessionWorker)
- [ ] Funciona em multiplos monitores e com DPI scaling (125%, 150%)

### 13. Tratamento de Erros

- [ ] Falha de rede nao crasha o agente (servico permanece `Running`)
- [ ] API offline nao impede captura local (eventos salvos no buffer SQLite)
- [ ] Erro no upload retenta automaticamente com backoff
- [ ] Colaborador invalido: log `[ERR] Colaborador nao existe`
- [ ] Config invalido: log `[WRN] Configuracao incompleta`
- [ ] Falha do SessionWorker e detectada e o servico reinicia o worker

### 14. Final + Distribuicao

- [ ] Todos os arquivos no Git commitados e tag de versao criada: `git tag vX.Y.Z && git push --tags`
- [ ] Release notes escritas
- [ ] Instalador final: `instalador/ManagerAgent-Setup.exe`
- [ ] SHA256 calculado: `Get-FileHash instalador\ManagerAgent-Setup.exe -Algorithm SHA256`
- [ ] Arquivo enviado para distribuicao segura
- [ ] Suporte preparado para perguntas

### Red Flags — NAO distribuir se

- Codigo nao compila
- Teste de vinculacao falha
- Servico nao registra no SCM ou nao inicia automaticamente
- SessionWorker nao e lancado pelo servico
- Captura mostra "unknown" constantemente
- Upload nao funciona em teste
- RAM total > 200 MB sem razao
- Agente crasha ao desconectar internet
- Instalador falha em maquina limpa
- Desinstalador nao remove o servico
- Logs com erros constantes
- DeviceToken nao esta criptografado com DPAPI
- Named pipe sem ACL restrita

---

**Sign-Off**

Checklist completado por: ___________________________

Data: _______________ | Versao: _______________ | Assinatura: ___________________________

_Qualidade antes de velocidade. Melhor atrasar um dia do que distribuir com bugs._
