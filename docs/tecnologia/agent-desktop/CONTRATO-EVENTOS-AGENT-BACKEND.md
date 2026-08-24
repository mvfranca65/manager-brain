> **STATUS:** ATIVO
> **DATA:** 2026-08-24
> **DONO:** @Bucky (Agent Desktop) — secoes de backend revisadas com @Shuri
> **PARA:** quem for criar ou alterar API que consome estes eventos

# Contrato de eventos — Agent Desktop -> backend

Documento de referencia. Toda afirmacao aqui foi conferida no codigo e cita `arquivo:linha`.
O que nao foi conferido esta marcado **NAO VERIFICADO** — nao ha afirmacao sem prova neste
documento.

**Prefixos de caminho usados nas citacoes:**

| prefixo | repositorio |
|---|---|
| `AGENT/` | `manager-srv-agent` (C#/.NET 8 — Windows) |
| `EVENTS/` | `manager-srv-events-node` (recebe os eventos) |
| `ADMIN/` | `manager-srv-admin-node` (vinculo, token, versao) |
| `BRAIN/` | `manager-brain` |

**Versao do Agent analisada:** 1.5.16 (`AGENT/src/ManagerAgent.Service/ManagerAgent.Service.csproj:8`,
`AGENT/src/ManagerAgent.SessionWorker/ManagerAgent.SessionWorker.csproj:11`).

**Qual codigo realmente executa.** Existem duas arquiteturas no repositorio do Agent. A que roda
hoje e a v2: `ManagerAgent.Service.exe` (servico, sessao 0) + `ManagerAgent.SessionWorker.exe`
(um por sessao interativa). O `ManagerAgent.Tray` e a v1 legada: nao entra no pacote
(`AGENT/scripts/build/build-pacote-e2e.ps1:92-93` lista apenas Service, SessionWorker, Watchdog e
Configurator) e o instalador da v2 mata o processo dela
(`AGENT/src/ManagerAgent.Installer/MigrationV1ToV2.cs:89`). **Tudo neste documento descreve o
caminho v2.** Se voce encontrar `ManagerAgent.Tray/...` em alguma citacao de outro documento,
esta olhando codigo que nao executa.

---

## 1. O que voce precisa saber em 2 minutos

| pergunta | resposta curta | onde conferir |
|---|---|---|
| Endereco | `POST {baseUrlEvents}/api/agent/events` | `AGENT/src/ManagerAgent.Service/Upload/HttpEventUploader.cs:113`; `EVENTS/src/modules/ingestion/ingestion.controller.ts:152,159` + `EVENTS/src/main.ts:71` |
| Producao | `https://api-events.imanagerportal.com` | `AGENT/src/ManagerAgent.Service/appsettings.json:9` |
| Autenticacao | `Authorization: Bearer <Device JWT>` (HS256) | `AGENT/.../HttpEventUploader.cs:198-199`; `EVENTS/src/common/auth/device-jwt.guard.ts:34-35` |
| Validade do Device JWT | 2 horas (default 7200 s) | `ADMIN/src/common/config/env.schema.ts:85-89` |
| Validade do refresh token | 7 dias | `ADMIN/src/common/config/env.schema.ts:92-96` |
| Tamanho do lote | 100 eventos (Agent) / 1..1000 (backend) | `AGENT/.../UploadWorker.cs:29`; `EVENTS/src/modules/ingestion/dto/agent-event-batch.dto.ts:40-41` |
| Cadencia | 60 s, com envio imediato em fronteira de sessao | `AGENT/.../UploadWorker.cs:28`, `:239`, `:257-274` |
| Resposta normal | **202 Accepted** | `EVENTS/src/modules/ingestion/ingestion.controller.ts:163` |
| Lote repetido | **208 Already Reported** (cache de 10 min, por processo) | `EVENTS/.../ingestion.controller.ts:149,291-293`; `EVENTS/src/modules/ingestion/services/dedup.service.ts:51-52` |
| Erro de LOTE | **400** `PAYLOAD_INVALIDO` | `EVENTS/src/modules/ingestion/ingestion.service.ts:127-129` |
| Erro de ITEM | **nunca 400** — vai para `motivosIgnorados` no 202 | `EVENTS/.../dto/agent-event-batch.dto.ts:443-444`, `:467-489` |
| Rate limit | 60 lotes/minuto por `instalacaoId`; `Retry-After: 60` | `EVENTS/src/common/config/env.schema.ts:51-55`; `EVENTS/src/common/errors/global-exception.filter.ts:88-91` |
| Tipos de evento | 11 no enum do backend; o Agent Windows emite **7** | `EVENTS/.../dto/agent-event-batch.dto.ts:62-74`; ver secao 3 |
| Quem calcula ATIVO/PAUSA/AUSENTE | **o Agent**, desde a 1.5.12. O backend so grava. | `AGENT/src/ManagerAgent.SessionWorker/Capture/UserStatusManager.cs:129-134`; ver secao 4 |
| Chaves UNIQUE | so 3 tabelas tem; `eventos_sessao` **nao tem** | ver secao 5 |
| Fuso | ISO 8601 **com offset obrigatorio**; colunas `timestamptz` | `EVENTS/.../dto/agent-event-batch.dto.ts:206-209` |

### As cinco coisas que mais derrubam quem escreve API nova

1. **`identificadorColaborador`, `instalacaoId` e `windowsUsername` chegam como `null` explicito,
   nao ausentes.** O envelope do Agent nao omite nulos
   (`AGENT/.../HttpEventUploader.cs:29-34` — sem `DefaultIgnoreCondition`). O backend so passou a
   aceitar isso em 24/08, trocando `.optional()` por `.nullish()`
   (`EVENTS/.../dto/agent-event-batch.dto.ts:399-414`, justificativa em `:391-397`: 93 recusas
   HTTP 400 medidas em 21/08, sempre em `windowsUsername`). Dentro de `dados`, ao contrario, nulo
   e **omitido** (`AGENT/src/ManagerAgent.SessionWorker/Pipe/PipeEventBuffer.cs:18-22`).
2. **Um item invalido nao derruba mais o lote, mas some em silencio.** Ele vira `motivosIgnorados`
   no 202 — e **o Agent nao le o corpo da resposta em sucesso**
   (`AGENT/.../HttpEventUploader.cs:221-227` retorna sem ler), marcando tudo como enviado
   (`AGENT/.../UploadWorker.cs:353-355`). Ninguem do lado do Agent sabe que perdeu o evento.
3. **Ha perda de dado por teto de tentativas.** Lote recusado por conteudo 5 vezes e **descartado**
   (`AGENT/.../UploadWorker.cs:42`, `:396-421`). Ate 100 eventos por vez. Ver secao 1.6.
4. **`eventos_sessao` nao tem chave UNIQUE.** LOGOUT em duplicata e possivel e conhecido — o
   proprio Agent registra a brecha em `AGENT/src/ManagerAgent.Service/ManagerAgentService.cs:317-327`.
5. **O campo de status pode mentir.** Ver a caixa de destaque na secao 4.

---

## 2. Transporte

### 2.1 Endereco e metodo

| item | valor | citacao |
|---|---|---|
| Metodo | `POST` | `EVENTS/src/modules/ingestion/ingestion.controller.ts:159` |
| Caminho | `/api/agent/events` | `@Controller('agent')` `:152` + `@Post('events')` `:159` + `setGlobalPrefix('api')` em `EVENTS/src/main.ts:71` |
| Montagem no Agent | `{config.BaseUrlEvents.TrimEnd('/')}/api/agent/events` | `AGENT/.../HttpEventUploader.cs:112-113` |
| `Content-Type` | `application/json`, UTF-8 | `AGENT/.../HttpEventUploader.cs:195` |
| Base URL de producao | `https://api-events.imanagerportal.com` | `AGENT/src/ManagerAgent.Service/appsettings.json:9` |
| Onde a URL e guardada | `C:\ProgramData\ManagerAgent\config.json`, campo `baseUrlEvents` | `AGENT/src/ManagerAgent.Shared/Config/ConfigPaths.cs:8-9,22-33,57`; `AGENT/src/ManagerAgent.Shared/Config/ConfigLocal.cs:11-12` |

### 2.2 Cabecalhos que o Agent envia

| header | quando | citacao |
|---|---|---|
| `Authorization: Bearer <token>` | sempre que houver token | `AGENT/.../HttpEventUploader.cs:198-199` |
| `X-Installation-Id` | quando `config.InstalacaoId` nao e vazio | `AGENT/.../HttpEventUploader.cs:201-202` |
| `X-Client-Dedup-Batch` | quando o hash do lote nao e vazio | `AGENT/.../HttpEventUploader.cs:204-205` |

**`X-Installation-Id` e ignorado pelo backend, de proposito** — foi removido por ser falsificavel
(`EVENTS/src/modules/ingestion/ingestion.service.ts:61-69`). A identidade vem so das claims do JWT.
O Agent continua enviando; e ruido inofensivo, mas **nao construa API que dependa dele**.

**O Agent NAO envia `X-Agent-Version`.** Ver a consequencia na secao 9.3.

### 2.3 Autenticacao — Device JWT

**Como nasce.** Vinculacao em dois passos, ambos contra o `ADMIN`, autenticados pela chave de
ativacao da empresa:

| passo | endpoint | citacao |
|---|---|---|
| 1. valida colaborador | `POST {baseUrlAdmin}/api/agente/colaboradores/validar` | `AGENT/src/ManagerAgent.Service/Linking/AgentLinkService.cs:129` |
| 2. vincula dispositivo | `POST {baseUrlAdmin}/api/agente/dispositivos/vincular` | `AGENT/.../AgentLinkService.cs:162`; `ADMIN/src/modules/agent-vinculacao/agent-vinculacao.controller.ts:94,155` |

Corpo do passo 2 (`AGENT/.../AgentLinkService.cs:163-177`): `identificadorColaborador`,
`instalacaoId`, `nomeMaquina`, `usuarioSessao` (fixo `"SYSTEM"`), `descricaoSo`, `versaoAgente`,
`ip`, `hardwareFingerprint` (SHA-256; so o hash sai da maquina — `:154-159`),
`dispositivoTipo` (fixo `"DESKTOP_WINDOWS"` — `:176`).

Resposta (`ADMIN/src/modules/agent-vinculacao/dto/vincular-agente.response.dto.ts:25,37,45,54,63,73,83,93`):
`statusVinculo` (`VINCULADO`|`JA_VINCULADO`), `colaboradorId`, `agenteId`, `instalacaoId`,
`ultimoHeartbeatEm`, **`deviceToken`**, **`refreshToken`**. Nao ha `expiresIn`.
O Agent guarda token e refresh em `AGENT/.../AgentLinkService.cs:204-211`.

**Onde e guardado.** `C:\ProgramData\ManagerAgent\config.json`, protegidos por DPAPI LocalMachine
nos campos `deviceTokenDpapi` / `refreshTokenDpapi`
(`AGENT/src/ManagerAgent.Shared/Config/ConfigLocal.cs:42-53`). Os campos em texto plano
`deviceToken`/`refreshToken` existem so para config anterior a v1.3.0 (`:55-67`).

**Claims** (`ADMIN/src/common/auth/device-jwt.issuer.ts:78-89`):
`sub` (agenteId, **string**), `tenant_id` (empresaId, numero), `usuario_id` (numero),
`instalacao_id` (string), `cnpj` (string), `iss` = `manager-srv-admin`, `iat`, `exp`.
Assinatura HS256, segredo `MANAGER_DEVICE_JWT_SECRET` (`:55`), minimo 32 chars
(`ADMIN/src/common/config/env.schema.ts:80-82`).

O `EVENTS` valida HS256 apenas (`EVENTS/src/common/auth/device-jwt.verifier.ts:35,84`),
`clockTolerance: 0` (`:85`), e exige `sub`, `tenant_id`, `usuario_id`, `instalacao_id`
(`:111-141`). **Nao valida `iss`, `nbf` nem `kid`** — declarado em `:6-8,86`.

**Validade e renovacao.**

| item | valor | citacao |
|---|---|---|
| TTL do Device JWT | 7200 s (2 h), env `MANAGER_DEVICE_JWT_EXPIRATION_SECONDS` | `ADMIN/src/common/config/env.schema.ts:85-89` |
| TTL do refresh token | 7 dias, env `MANAGER_DEVICE_REFRESH_TOKEN_EXPIRATION_DAYS` | `ADMIN/src/common/config/env.schema.ts:92-96` |
| Endpoint de renovacao | `POST {baseUrlAdmin}/api/agente/auth/refresh`, corpo `{refreshToken}` | `AGENT/src/ManagerAgent.Shared/Auth/TokenManager.cs:168,170`; `ADMIN/src/modules/agent-auth/agent-auth.controller.ts:72,76` |
| Quando o Agent renova | quando faltam **menos de 1 minuto** para expirar | `AGENT/.../TokenManager.cs:205-218` (`ValidTo < UtcNow.AddMinutes(1)`) |
| Coordenacao entre processos | mutex nomeado `Global\ManagerAgent.TokenRefresh`, timeout 30 s | `AGENT/.../TokenManager.cs:36-37,90-145` |
| Rotacao | refresh token e de uso unico (revogado no ato) | `ADMIN/src/modules/agent-auth/agent-auth.repository.ts:137-145` |
| Rate limit do refresh | 20 req / 60 s **por IP** | `ADMIN/src/modules/agent-auth/agent-auth.controller.ts:68-69,78-82` |

O refresh e `@Public` (`ADMIN/.../agent-auth.controller.ts:77`) — o proprio refresh token e a
credencial. **Ele nunca devolve 403 e nunca emite `X-Agent-Revoked`**; um agente revogado ainda
consegue token novo ali e so e barrado no proximo endpoint protegido
(`ADMIN/src/common/auth/revocation-check.interceptor.ts:76-78`;
`ADMIN/src/modules/agent-auth/agent-auth.repository.ts:206-219`).

### 2.4 Tamanho do lote e cadencia

| item | valor | citacao |
|---|---|---|
| Eventos por lote (Agent) | 100 | `AGENT/.../UploadWorker.cs:29` |
| Lotes por ciclo | `max(10, sessoesAtivas x 2)`, teto 50 | `AGENT/.../UploadWorker.cs:30-31,430-441` |
| Intervalo do ciclo | 60 s | `AGENT/.../UploadWorker.cs:28` |
| Envio imediato | LOCK, UNLOCK, LOGIN, LOGOUT furam os 60 s | `AGENT/.../UploadWorker.cs:239,257-274`; `AGENT/src/ManagerAgent.Shared/Session/SessionEventTypes.cs:30-31,40` |
| Minimo/maximo aceito pelo backend | 1 a 1000 eventos | `EVENTS/.../dto/agent-event-batch.dto.ts:40-41,416-420` |
| Agrupamento | um lote por par (sessao Windows, usuario) | `AGENT/src/ManagerAgent.Service/Storage/SqliteEventBuffer.cs:411-458` |
| Buffer do SessionWorker | 10.000 itens em memoria; acima disso descarta o mais antigo | `AGENT/src/ManagerAgent.SessionWorker/Pipe/AutonomousBuffer.cs:17,185-186` |
| Dreno worker -> servico | a cada 10 s | `AGENT/src/ManagerAgent.SessionWorker/Worker.cs:96,418-419` |
| Limite do pipe | 10 MB por mensagem | `AGENT/src/ManagerAgent.Shared/Pipe/PipeProtocol.cs:12,33-35` |

**Nao ha limite de bytes no corpo HTTP configurado em nenhum dos dois lados** — NAO VERIFICADO se
o proxy/Fly.io impoe um; nao ha nada no codigo dos repositorios.

### 2.5 O que acontece quando falha

O Agent classifica o resultado em tres
(`AGENT/src/ManagerAgent.Service/Upload/ResultadoEnvio.cs:74-90`):

| resultado | quando | o que o Agent faz |
|---|---|---|
| `Ok` | 2xx, ou 208 | marca enviado e apaga do buffer (`UploadWorker.cs:353-355`) |
| `FalhaTransitoria` | sem rede, 5xx, timeout, **408**, **429**, 401 sem refresh, 403 com `X-Agent-Revoked` | volta para a fila, **sem teto e sem contador** (`UploadWorker.cs:360-365`) |
| `FalhaPermanente` | qualquer outro 4xx (400, 404, 412, 422...) | conta uma tentativa; no teto, **descarta** |

A classificacao esta em `AGENT/.../ResultadoEnvio.cs:109-122`: 401, 403, 408 e 429 sao
explicitamente excluidos de "falha de conteudo".

Dentro de uma unica chamada ha no maximo **2 tentativas** (`AGENT/.../HttpEventUploader.cs:22`),
e a segunda so existe para reexecutar apos um refresh disparado por 401 (`:229-248`).
Nao ha backoff: o retry real e o proximo ciclo de 60 s.

Sem rede o ciclo nem comeca (`AGENT/.../UploadWorker.cs:294-299`).

### 2.6 O que e descartado — teto do A-64 (PERDA DE DADO)

> **Depois de 5 recusas por conteudo, o lote inteiro sai da fila sem ter sido enviado.**
> Ate 100 eventos por vez, definitivamente perdidos.

- Teto: `MaxFalhasPermanentesPorLote = 5` (`AGENT/.../UploadWorker.cs:42`). Com o ciclo de 60 s,
  isso da **cinco minutos**.
- A contagem e **por linha**, nao por lote, e sobrevive a reboot — coluna `tentativas_falha`
  no SQLite local (`AGENT/.../SqliteEventBuffer.cs:91,519-564`).
- No teto, as linhas recebem `uploaded = 2` ("descartado") e um `ultimo_erro`
  (`AGENT/.../SqliteEventBuffer.cs:509,570-603`), e somem na faxina de 24 h
  (`:617-641`).
- E registrado **um** `LogError` no momento do descarte, nao um por ciclo
  (`AGENT/.../UploadWorker.cs:413-418`).
- Falha passageira **nao** consome o teto (`AGENT/.../UploadWorker.cs:358-365`).

**Por que isso importa para quem escreve API:** um 400 novo introduzido pelo backend — um campo
que passou a ser obrigatorio, um `max()` reduzido — apaga dados da frota inteira em cinco
minutos, em silencio. **Nunca aperte validacao de lote sem antes conferir o que a frota manda.**

---

## 3. O envelope do lote

Serializado por `AGENT/.../HttpEventUploader.cs:169` com as opcoes de `:29-34`
(camelCase, sem indentacao, **sem `DefaultIgnoreCondition`**). Montado em `:152-167`.

Validado por `AgentEventBatchEnvelopeSchema`
(`EVENTS/.../dto/agent-event-batch.dto.ts:446-449`), cujos campos vem de `camposDeLote`
(`:399-414`).

| campo | tipo | obrigatorio | aceita `null`? | limite | o Agent manda | citacao (Agent / backend) |
|---|---|---|---|---|---|---|
| `maquinaId` | string | **sim** | nao | 1..255 | `Environment.MachineName` | `HttpEventUploader.cs:154` / `dto:400` |
| `identificadorColaborador` | string | nao | **sim (`.nullish()`)** | 1..64 | `config.IdentificadorColaborador` — **`null` quando vazio** | `HttpEventUploader.cs:155`, `AgentEventBatchDto.cs:15-16` / `dto:401` |
| `instalacaoId` | string | nao | **sim (`.nullish()`)** | 1..64 | `config.InstalacaoId` — **`null` quando vazio** | `HttpEventUploader.cs:156`, `AgentEventBatchDto.cs:18-19` / `dto:402` |
| `versaoAgente` | string | **sim** | nao | 1..50 | versao do assembly, **4 partes** (`1.5.16.0`) | `HttpEventUploader.cs:140,158` / `dto:403` |
| `descricaoSo` | string | **sim** | nao | 1..255 | `RuntimeInformation.OSDescription` | `HttpEventUploader.cs:141,159` / `dto:404` |
| `dispositivoTipo` | enum | **sim** | nao | 7 valores | **`"WINDOWS"`** (legado) | `HttpEventUploader.cs:144,160`, `DispositivoTipoDetector.cs:18,34-35` / `dto:405-411` |
| `enviadoEm` | ISO 8601 c/ offset | **sim** | nao | — | `DateTimeOffset.UtcNow` | `HttpEventUploader.cs:161` / `dto:412` |
| `windowsUsername` | string | nao | **sim (`.nullish()`)** | 0..255 | usuario Windows do lote — **`null` quando vazio** | `HttpEventUploader.cs:157`, `UploadWorker.cs:347` / `dto:413` |
| `eventos` | array | **sim** | nao | 1..1000 | ate 100 | `UploadWorker.cs:29` / `dto:416-420,448` |

Cada item de `eventos` tem exatamente dois campos
(`AGENT/src/ManagerAgent.Service/Upload/AgentEventBatchDto.cs:48-55`):

| campo | tipo | citacao |
|---|---|---|
| `tipoEvento` | string | `AgentEventBatchDto.cs:50-51` |
| `dados` | objeto | `AgentEventBatchDto.cs:53-54` |

### 3.1 Os tres campos que chegam como `null` explicito

`AgentEventBatchDto.IdentificadorColaborador`, `.InstalacaoId` e `.WindowsUsername` sao `string?`
(`AGENT/.../AgentEventBatchDto.cs:16,19,22`). O serializador nao omite nulos
(`AGENT/.../HttpEventUploader.cs:29-34`). Logo o corpo carrega literalmente
`"windowsUsername": null`.

Foi isso que derrubou lotes ate 24/08: `.optional()` do Zod aceita a chave **ausente** e recusa
`null` **explicito**. A correcao trocou os tres para `.nullish()`
(`EVENTS/.../dto/agent-event-batch.dto.ts:401,402,413`, justificativa em `:391-397`).

**Regra para API nova: todo campo opcional de lote precisa ser `.nullish()`, nunca `.optional()`.**

**Dentro de `dados` a regra e a oposta:** o buffer do worker serializa com
`JsonIgnoreCondition.WhenWritingNull` (`AGENT/src/ManagerAgent.SessionWorker/Pipe/PipeEventBuffer.cs:18-22`),
entao um evento em andamento chega **sem a chave** `finalizadoEm`. Os schemas de item usam
`.optional()` e isso esta correto para eles.

### 3.2 Aliases historicos aceitos

**Nao existe nenhum alias de NOME DE CAMPO.** O comentario em
`EVENTS/.../dto/agent-event-batch.dto.ts:12-14` afirma que o controller normaliza
`machineId -> maquinaId` antes do parse. **Isso nao existe no codigo**: o controller passa
`@Body() body: unknown` direto (`EVENTS/src/modules/ingestion/ingestion.controller.ts:266-289`) e
o servico faz `safeParse` no corpo cru (`EVENTS/.../ingestion.service.ts:119`). Um agente que
mandar `machineId` recebe **400 PAYLOAD_INVALIDO**.

Os aliases que existem sao de **valor**, dentro dos schemas:

| dominio | alias -> canonico | citacao |
|---|---|---|
| SessionEvent | `LOGON -> LOGIN`, `LOGOFF -> LOGOUT` | `dto:101-102,105-106` |
| UserStatus | `ONLINE -> ATIVO`, `OFFLINE -> AUSENTE`, `IDLE -> AUSENTE`, `INATIVO -> AUSENTE` | `dto:129-132,135-138` |
| padraoDigitacao | `NORMAL`/`HUMAN`/`DIGITACAO_NORMAL -> NATURAL` | `dto:168-171,189-190` |
| | `ROBO -> ROBOTICO` | `dto:170,191` |
| | `PADRAO_TECLA_REPETIDA_SUSPEITO`/`BURST_MESMA_TECLA -> REPETITIVO` | `dto:172-173,192-193` |
| | `NAVEGACAO_APENAS -> IRREGULAR` | `dto:174,194` |
| | `MOUSE_APENAS -> INDEFINIDO` | `dto:175,195` |
| | `CONTINUOUS -> STEADY` | `dto:176,196` |
| cliques | `cliquesMouseEsq/Dir/Meio -> cliquesEsquerdo/Direito/Meio` | `dto:273-275`; `activity-summary.handler.ts:128-130` |
| intervalo | `iniciadoEm/finalizadoEm -> intervaloInicio/intervaloFim` | `activity-summary.handler.ts:96-97` |
| tipo de sessao | `tipoEvento` e `tipoTransicao` — qualquer um serve | `dto:249-250`; `session.handler.ts:58` |
| tipoEvento | `InputActivitySummary` e `InputSummaryEvent` = `ActivitySummaryEvent` | `handler-registry.ts:78-80` |
| dispositivo | `WINDOWS -> DESKTOP_WINDOWS`, `MACOS -> DESKTOP_MACOS`, `ANDROID -> MOBILE_ANDROID`, `IOS -> IOS` | `normalizar-dispositivo.ts:74-82` |

Todos os normalizadores de valor fazem `trim().toUpperCase()` antes
(`dto:92,121,157`).

---

## 4. Status do colaborador — ATIVO / PAUSA / AUSENTE

### 4.1 Quem calcula: o Agent

**Desde a versao 1.5.12.** O arquivo `UserStatusManager.cs` foi reintroduzido no commit `c680cd2`,
cujo `csproj` marca `<Version>1.5.12</Version>` (verificado com
`git log --diff-filter=A -- src/ManagerAgent.SessionWorker/Capture/UserStatusManager.cs` e
`git show c680cd2:src/ManagerAgent.SessionWorker/ManagerAgent.SessionWorker.csproj`).

> **Divergencia de documentacao:** o comentario do proprio codigo diz "v1.5.11: a regra voltou"
> (`AGENT/src/ManagerAgent.SessionWorker/Capture/DailyBoundaryWorker.cs:13,16`). O commit que
> trouxe o arquivo de volta carrega `<Version>1.5.12</Version>`. Use **1.5.12** como corte.

**Historico que importa (achado A-58).** Entre a 1.5.0 e a 1.5.11 o Agent Windows parou de emitir
`StatusTransitionEvent`, apostando que o backend derivaria o status. **O backend nunca derivou —
ele so grava o que recebe.** Resultado: nenhuma transicao de status em maquina Windows entre
2026-05-17 e a volta da regra
(`AGENT/src/ManagerAgent.SessionWorker/Capture/Models/UserStatus.cs:6-8`;
`AGENT/.../DailyBoundaryWorker.cs:11-14`).

**Nao existe caminho legado no backend.** Nao ha nenhum codigo em `EVENTS` que derive status a
partir de ociosidade ou de resumo de input: o `StatusTransitionHandler` faz um `INSERT` simples e
nada mais (`EVENTS/src/modules/ingestion/handlers/status-transition.handler.ts:34-55`). Se o Agent
nao mandar, a tabela fica vazia.

### 4.2 Limiares e cadencia

| item | valor efetivo | citacao |
|---|---|---|
| ATIVO -> PAUSA | silencio >= **5 minutos** | `AGENT/.../UserStatusManager.cs:50-51`; `AGENT/src/ManagerAgent.SessionWorker/appsettings.json:25` |
| PAUSA -> AUSENTE | silencio >= **15 minutos** | `AGENT/.../UserStatusManager.cs:52-53`; `appsettings.json:26` |
| Reavaliacao | a cada **30 s** | `AGENT/src/ManagerAgent.SessionWorker/Worker.cs:151-153`; `appsettings.json:27` |
| Fonte do silencio | `GetLastInputInfo` do Windows | `AGENT/.../Worker.cs:86-87`; `AGENT/src/ManagerAgent.SessionWorker/Capture/Infrastructure/WindowsIdleDetector.cs` |
| Config invertida (`ausente <= pausa`) | ignorada; volta para 5 e 15 com WARN | `AGENT/.../UserStatusManager.cs:58-66` |
| Config zerada/ausente | vira o default, nunca zero | `AGENT/.../UserStatusManager.cs:50-53` |

### 4.3 Transicoes possiveis

Classificacao pura (`AGENT/.../UserStatusManager.cs:129-134`):

```
silencio >= 15min -> AUSENTE
silencio >=  5min -> PAUSA
                  -> ATIVO
```

| de | para | quando | citacao |
|---|---|---|---|
| ATIVO | PAUSA | 5 min de silencio | `:132` |
| ATIVO | **AUSENTE** | salto direto, quando a avaliacao perdeu a janela da PAUSA (maquina saturada, worker relancado) — **e correto** | `:123-127,131` |
| PAUSA | AUSENTE | 15 min de silencio | `:131` |
| PAUSA | ATIVO | qualquer interacao | `:133` |
| AUSENTE | ATIVO | qualquer interacao | `:133` |
| AUSENTE | PAUSA | **impossivel** — o silencio so cresce | `:129-134` |

**Duas situacoes em que NAO ha evento:**

1. **Primeira avaliacao apos o worker subir.** O status inicial e adotado a partir do silencio
   real, **sem emitir transicao** (`:79-88`). E de proposito: `statusAnterior` e obrigatorio no
   schema e `NOT NULL` na tabela (`EVENTS/src/common/db/schema/eventos.schema.ts:142`), entao
   toda transicao emitida ja tem de onde veio.
2. **Fronteira de sessao** (LOCK, UNLOCK, LOGIN, LOGOUT). O status volta a ATIVO **sem emitir**
   (`AGENT/.../UserStatusManager.cs:108-121`;
   `AGENT/src/ManagerAgent.SessionWorker/Capture/SessionEventService.cs:104-107`). O periodo
   bloqueado ja esta descrito em `eventos_janela` e `eventos_ociosidade`, com mais precisao — e a
   tabela de transicoes **nao tem trava contra duplicata**.

A avaliacao fica **dentro** do portao de pausa do laco principal
(`AGENT/.../Worker.cs:200-221`): com a captura suspensa nao se emite transicao
(justificativa em `AGENT/.../Worker.cs:331-334`).

### 4.4 O que o backend recebe e faz

Payload: ver secao 5 da tabela de eventos (`StatusTransitionEvent`).
O handler faz **INSERT puro, sem `ON CONFLICT`**
(`EVENTS/src/modules/ingestion/handlers/status-transition.handler.ts:50-55`), nunca rejeita item
(`:31,57`), e nao trunca `motivo` (`:45` — coerente com `varchar(255)` e o `max(255)` do Zod).

Tabela `eventos_transicao_status` (`EVENTS/src/common/db/schema/eventos.schema.ts:139-154`).
`status_anterior` e `status_novo` sao `varchar(32) NOT NULL` (`:142-143`).
`agente_id` e `NOT NULL` aqui — diferente de `eventos_janela`/`ociosidade`/`sessao`, onde e
nulavel (`:150` vs `:70,102,122`).

---

> ## AVISO — o campo de status pode mentir, e nao ha como o backend perceber
>
> **O que acontece hoje.** A conversao do enum interno para o texto que vai no evento tem um caso
> curinga:
>
> ```csharp
> // AGENT/src/ManagerAgent.SessionWorker/Capture/UserStatusManager.cs:145-151
> internal static string Nome(UserStatus status) => status switch
> {
>     UserStatus.Ativo   => "ATIVO",     // :147
>     UserStatus.Pausa   => "PAUSA",     // :148
>     UserStatus.Ausente => "AUSENTE",   // :149
>     _                  => "ATIVO"      // :150  <-- aqui
> };
> ```
>
> **O que o backend recebe.** A string `"ATIVO"`, byte a byte identica a de um ATIVO de verdade.
> Ela passa pelo `userStatusNormalizer` sem marca nenhuma
> (`EVENTS/.../dto/agent-event-batch.dto.ts:125-128`) e e gravada em
> `eventos_transicao_status.status_novo` como qualquer outro ATIVO
> (`EVENTS/.../handlers/status-transition.handler.ts:43`).
>
> **O que o backend NAO tem como saber.** Nada. Nao ha campo de confianca, nao ha motivo
> distinguivel — o `motivo` gerado nesse caminho e o mesmo texto do ATIVO legitimo
> (`AGENT/.../UserStatusManager.cs:136-139`). Uma API que somar tempo em ATIVO para calcular
> produtividade contaria como trabalho um estado que o Agent **nao soube classificar**.
>
> **Por que isto e diferente dos outros defeitos de 24/08.** Os outros perdem dado ou ficam em
> silencio. Este **afirma**. Um dado ausente e visivel; um dado errado com cara de certo, nao.
>
> ### Isto e latente hoje, nao ativo
>
> Conferido linha a linha, nesta versao:
>
> | | |
> |---|---|
> | Valores no enum `UserStatus` | **3** — `Ativo` (`AGENT/src/ManagerAgent.SessionWorker/Capture/Models/UserStatus.cs:17`), `Pausa` (`:20`), `Ausente` (`:23`) |
> | Valores tratados no `switch` | **os 3** — `:147`, `:148`, `:149` |
> | Curinga alcancavel hoje? | **nao** |
>
> **Todos os tres valores estao tratados.** O curinga e inalcancavel na 1.5.16, e o defeito e
> **latente**. Ele vira ativo no dia em que alguem acrescentar um quarto valor ao enum — e o
> agravante e que o proprio guarda-corpo do repositorio nao avisaria: `CS8509` esta promovido a
> erro de compilacao (`AGENT/Directory.Build.props:22`), mas **um `switch` com ramo `_` nunca
> emite CS8509**. O curinga desliga exatamente a rede que existe para pegar isso.
>
> ### Caso irmao, mesma forma, impacto menor
>
> ```csharp
> // AGENT/src/ManagerAgent.SessionWorker/Capture/RenderConfianca.cs:30-37
> public static string ToApiString(this RenderConfianca c) => c switch
> {
>     RenderConfianca.Ausente               => "AUSENTE",                 // :32
>     RenderConfianca.ProcessoNativoReuniao => "PROCESSO_NATIVO_REUNIAO", // :33
>     RenderConfianca.BrowserUrlReuniao     => "BROWSER_URL_REUNIAO",     // :34
>     RenderConfianca.Incerto               => "INCERTO",                 // :35
>     _                                     => "INCERTO"                  // :36  <-- aqui
> };
> ```
>
> Enum com **4** valores (`RenderConfianca.cs:13,17,20,24`), **os 4 tratados** — tambem latente.
> A diferenca que importa: aqui o rotulo do curinga e `INCERTO`, que o backend ja trata como "nao
> sei" (`EVENTS/.../dto/agent-event-batch.dto.ts:316,329`). **A solucao certa ja existe e ja e
> usada dentro do proprio produto** — o que falta e aplica-la ao status.
>
> Estes sao os **dois unicos** curingas que devolvem literal de string em todo o caminho de
> captura, Shared e Service (`grep -rn '_ => "'` em
> `src/ManagerAgent.SessionWorker`, `src/ManagerAgent.Shared`, `src/ManagerAgent.Service`).
>
> Registrado tambem como defeito aberto na secao 11 (D-4). **A correcao nao esta decidida** — o
> caminho natural seria o curinga registrar e degradar para um rotulo visivel, como o
> `RenderConfianca` faz, mas isso e tarefa, nao detalhe de documento.

---

## 5. Cada tipo de evento

O enum do backend tem **11 valores** (`EVENTS/.../dto/agent-event-batch.dto.ts:62-74`).
O Agent Windows 1.5.16 emite **7** deles.

| # | `tipoEvento` | o Agent Windows emite? | tabela | citacao da emissao |
|---|---|---|---|---|
| 1 | `WindowActivityEvent` | **sim** | `eventos_janela` | `AGENT/.../WindowActivityService.cs:164,181,205,246` |
| 2 | `IdleEvent` | **sim** | `eventos_ociosidade` | `AGENT/src/ManagerAgent.SessionWorker/Pipe/PipeEventBuffer.cs:55-59` |
| 3 | `HeartbeatEvent` | **sim** | `batimentos` | `AGENT/.../HeartbeatService.cs:134` |
| 4 | `SessionEvent` | **sim** | `eventos_sessao` | `AGENT/.../SessionEventService.cs:112`; `AGENT/src/ManagerAgent.Service/ManagerAgentService.cs:367-376` |
| 5 | `ActivitySummaryEvent` | **NAO** | `resumos_atividade_input` | — |
| 6 | `InputActivitySummary` | **NAO** | `resumos_atividade_input` | — |
| 7 | `InputSummaryEvent` | **sim** (e este que sai) | `resumos_atividade_input` | `AGENT/.../Worker.cs:325` |
| 8 | `StatusTransitionEvent` | **sim** | `eventos_transicao_status` | `AGENT/.../Worker.cs:349-350` |
| 9 | `MeetingEvent` | **sim** | `eventos_reuniao` | `AGENT/.../Worker.cs:407` |
| 10 | `MeetingSnapshotEvent` | **sim** | `eventos_reuniao_snapshot` | `AGENT/.../MeetingSnapshotWorker.cs:124` |
| 11 | `PhoneCallEvent` | **NAO** (so Android/iOS) | `eventos_chamada_telefonica` | — |

> **Divergencia 1.** O Agent Windows **nunca** manda `ActivitySummaryEvent`. Ele manda o nome
> legado `InputSummaryEvent` (`AGENT/.../Worker.cs:325`). Os tres caem no mesmo handler e na mesma
> tabela (`EVENTS/.../handler-registry.ts:78-80`), mas quem filtrar por `ActivitySummaryEvent`
> nao vera nenhum evento de Windows.

**Ordem de despacho no backend** (`EVENTS/.../handler-registry.ts:44-56`), sequencial, cada handler
aguardado antes do proximo (`EVENTS/.../ingestion.service.ts:341-356`):
Heartbeat, WindowActivity, Idle, Session, ActivitySummary, InputActivitySummary, InputSummaryEvent,
StatusTransition, Meeting, MeetingSnapshot, PhoneCall.

**Regra geral do backend para timestamp faltando:** quando o timestamp do payload esta ausente ou
nao parseia, o handler substitui por `ctx.enviadoEmFallback`, que e **o `enviadoEm` do envelope**
(hora do Agent, nao do servidor) — `EVENTS/.../ingestion.service.ts:237`;
`EVENTS/.../event-handler.interface.ts:28`. A unica excecao e `ActivitySummary`, que rejeita.

---

### 5.1 `WindowActivityEvent`

**Quando o Agent emite** (`AGENT/src/ManagerAgent.SessionWorker/Capture/WindowActivityService.cs:11-29`):

| momento | `finalizadoEm` | citacao |
|---|---|---|
| janela nova aguenta o debounce de **1500 ms** | ausente (evento ABERTO) | `:71,141-143,167-182` |
| a cada **60 s** enquanto segue em foco (snapshot) | ausente; `iniciadoEm` **nunca muda** | `:70,191-206` |
| troca de janela | preenchido com o instante em que a nova ganhou foco | `:152-165` |
| ociosidade, LOCK, encerramento | preenchido com o instante informado | `:208-261` |
| morte do worker sem despedida | preenchido pelo **Service**, com a hora do EOF do pipe | `AGENT/src/ManagerAgent.Service/Resilience/OrphanWindowEventCloser.cs:51-97` |

Filtros que impedem o evento de nascer (`:270-325`): processo vazio; prefixo em `ProcessosIgnorados`
(`ManagerAgent`, `unknown` — `Config/ManagerAgentUploadOptions.cs:130-134`); substring em
`TitulosIgnorados` (`:140-146`); **processo de shell E titulo placeholder** juntos (`:156-186`).
"Mesma janela" = mesmo processo **e** mesmo dominio; trocar de aba no browser conta como janela
nova (`:263-268`).

**Payload** (`AGENT/.../Capture/Models/EventoJanela.cs:19-30`, camelCase, nulos omitidos):

| campo | tipo | obrig. no backend | limite | o Agent manda? | citacao |
|---|---|---|---|---|---|
| `nomeProcesso` | string | nao (mas `min(1)` se vier) | 1..255 | **sim** | `EventoJanela.cs:21` / `dto:220` |
| `tituloJanela` | string | nao | 0..500 | **sim** | `EventoJanela.cs:23` / `dto:221` |
| `iniciadoEm` | ISO c/ offset | nao | — | **sim** | `EventoJanela.cs:25` / `dto:222` |
| `finalizadoEm` | ISO c/ offset | nao | — | so no fechamento | `EventoJanela.cs:27` / `dto:223` |
| `statusUsuario` | enum | nao | — | **NAO** | `dto:224` |
| `urlDominio` | string (`null` -> `undefined`) | nao | 0..255 | so em browser | `EventoJanela.cs:29` / `dto:225,213-217` |
| `ipOrigem` | string | nao | 0..255 | **NAO** | `dto:226` |

**O que o backend faz** (`EVENTS/src/modules/ingestion/handlers/window-activity.handler.ts`):
colapsa duplicatas do mesmo lote em memoria (`:69-110,163`) — obrigatorio, senao o Postgres devolve
erro 21000 (`:160-161`) — e faz **`onConflictDoUpdate`** em
`(agente_id, nome_processo, iniciado_em)` (`:174-175`), com
`setWhere: finalizado_em IS NULL` (`:185`): **uma vez fechada, a linha e imutavel**.
Campos mutaveis usam `COALESCE(excluded, atual)` (`:177-182`); `urlDominio` usa
`COALESCE(NULLIF(excluded,''), atual)` (`:179`); `dispositivoTipo` e sempre sobrescrito (`:183`).
`nomeProcesso` ausente vira `'DESCONHECIDO'` (`:57,128,138`).
**Nunca rejeita item** (`:124,190`).

Tabela `eventos_janela` (`EVENTS/src/common/db/schema/eventos.schema.ts:68-86`).
Atencao: `titulo_janela` e `varchar(255)` na coluna (`:75`) enquanto o Zod aceita **500**
(`dto:221`) — ver defeito D-6.

---

### 5.2 `IdleEvent`

**Quando o Agent emite** (`AGENT/src/ManagerAgent.SessionWorker/Capture/IdleMonitorService.cs`):
limiar de ociosidade **60 s** (`:61`; `appsettings.json:18`), verificado a cada **5 s**
(`:62`; `appsettings.json:19`).

| momento | `motivo` emitido | `finalizadoEm` | citacao |
|---|---|---|---|
| ociosidade detectada (abertura) | `IdleStarted` | ausente | `:156,192` |
| input volta | `IdleTimeout` | preenchido | `:162` |
| camera/microfone ativos durante a ociosidade | `MediaActivity` | preenchido | `:117` |
| buraco de wall-clock (sleep/hibernate/hang) | `WallClockGap` / `WallClockGapBoundary` | preenchido | `:247,250` |
| flush externo (LOCK, corte diario, encerramento) | `ManualFlush` | preenchido | `:320` |

Ao abrir, o Agent **fecha a janela em andamento antes** e suspende a captura de janela, para os
dois blocos ficarem encostados (`:174-190`). Nao ha snapshot periodico durante a ausencia (`:170-172`).
Ociosidade com menos de **60 s** e descartada **apenas se nada foi enviado ainda**; se a abertura
ja subiu, o fechamento e obrigatorio (`:34,342-349`).

> **Divergencia 2.** O comentario `AGENT/.../Storage/IdleEventRecord.cs:214` lista os valores
> `"IdleTimeout" (default), "WallClockGap", "SystemSuspend", "ManualFlush"`. O codigo emite
> **`IdleStarted`, `IdleTimeout`, `MediaActivity`, `WallClockGap`, `WallClockGapBoundary` e
> `ManualFlush`** — `SystemSuspend` **nunca** e usado como motivo de ociosidade; ele e motivo de
> `SessionEvent LOCK` (`AGENT/.../PowerModeMonitor.cs:142,157`).

**Payload** (`AGENT/.../Capture/Storage/IdleEventRecord.cs:178-219`):

| campo | tipo | obrig. no backend | limite | citacao |
|---|---|---|---|---|
| `iniciadoEm` | ISO c/ offset | **sim** | — | `IdleEventRecord.cs:191-192` / `dto:230` |
| `finalizadoEm` | ISO c/ offset | nao | — | `:202-203` / `dto:231` |
| `motivo` | string (`null` -> `undefined`) | nao | 0..50 | `:217-218` / `dto:233` |
| `statusUsuario` | enum | nao | — | **o Agent nao manda** / `dto:232` |
| `ipOrigem` | string | nao | 0..255 | **o Agent nao manda** / `dto:234` |

> **Divergencia 3 — campos que o Agent manda e o backend joga fora.** O `IdleEventRecord`
> serializa tambem `instalacaoId`, `machineId`, `userId`, `operatingSystemDescription`,
> `ipAddress`, `agentVersion`, `createdAtUtc` e **`duracaoSegundos`**
> (`AGENT/.../IdleEventRecord.cs:183-189,206-209`). Nenhum esta em `idlePayload`
> (`dto:229-235`), e `z.object` do Zod descarta chave desconhecida. **`duracaoSegundos` e
> calculado pelo Agent (`IdleMonitorService.cs:412`) e descartado** — a duracao no banco e sempre
> derivada de `finalizado_em - iniciado_em`.

**O que o backend faz** (`EVENTS/src/modules/ingestion/handlers/idle.handler.ts`):
colapsa por `agenteId::iniciadoEm` (`:70-72,84-93,141`), depois
**`onConflictDoUpdate`** em `(agente_id, iniciado_em)` (`:151-152`) com
`setWhere: finalizado_em IS NULL` (`:160`). `motivo` e **truncado em 50** (`:63,131`).
Nunca rejeita item (`:116,165`).
Tabela `eventos_ociosidade` (`EVENTS/.../eventos.schema.ts:100-115`), coluna `motivo varchar(50)` (`:113`).

---

### 5.3 `SessionEvent`

**Quando o Agent emite:**

| tipo interno | quando | `motivo` tipico | citacao |
|---|---|---|---|
| `LOGIN` | worker sobe numa sessao interativa | — | `AGENT/.../SessionMonitor.cs:102,127` |
| `LOGOUT` | desligamento do worker | `ServiceShutdown`, `SESSION_LOGOFF`, `MACHINE_SHUTDOWN`, `UPDATE`, `SERVICE_STOP`, `DUPLICATE_WORKER` | `AGENT/.../Worker.cs:610,615`; `AGENT/src/ManagerAgent.Shared/Pipe/ShutdownReasons.cs:16,34,41,52,66` |
| `LOGOUT` (emitido pelo **Service**) | o worker morreu sem se despedir | `OsLogoff` (default) | `AGENT/src/ManagerAgent.Service/ManagerAgentService.cs:282,367-380` |
| `LOCK` | tela bloqueada; ou suspensao | `SystemSuspend` | `AGENT/.../LockScreenDetector.cs:195`; `AGENT/.../PowerModeMonitor.cs:142,157` |
| `UNLOCK` | tela liberada; ou retorno da suspensao | `SystemResume` | `AGENT/.../LockScreenDetector.cs:218`; `AGENT/.../PowerModeMonitor.cs:225` |
| `SESSAO_INTERROMPIDA` | no startup, quando a sessao anterior nao fechou limpo | `NoCleanShutdownFlag` | `AGENT/.../SessionMonitor.cs:236`; regra em `AGENT/src/ManagerAgent.Shared/Runtime/SessaoInterrompidaDecider.cs:135-164` |

`LOCK`, `UNLOCK`, `LOGIN` e `LOGOUT` sao **fronteira de sessao** e disparam envio imediato nas duas
pontas (`AGENT/src/ManagerAgent.Shared/Session/SessionEventTypes.cs:30-31,40`;
`AGENT/.../SessionEventService.cs:124-128`; `AGENT/.../Pipe/PipeMessageHandler.cs:397-400`).
`SESSAO_INTERROMPIDA` fica de fora de proposito (`SessionEventTypes.cs:17`).

**Payload** (`AGENT/.../Capture/Models/EventoSessao.cs:120-133`):

| campo | tipo | obrig. no backend | limite | citacao |
|---|---|---|---|---|
| `tipoEvento` | enum | um dos dois e obrigatorio | — | `EventoSessao.cs:122-123` / `dto:250` |
| `tipoTransicao` | enum | alternativa ao anterior | — | — / `dto:249` |
| `ocorreuEm` | ISO c/ offset | **sim** | — | `EventoSessao.cs:125-126` / `dto:251` |
| `motivo` | string (`null` -> `undefined`) | nao | 0..50 | `EventoSessao.cs:131-132` / `dto:252` |
| `ipOrigem` | string | nao | 0..255 | **o Agent nao manda** / `dto:253` |

Valores aceitos: `LOGIN`, `LOGOUT`, `LOCK`, `UNLOCK`, `SESSAO_INTERROMPIDA`, mais os aliases
`LOGON` e `LOGOFF` (`dto:96-102`). O Agent usa os canonicos
(`AGENT/src/ManagerAgent.Shared/Session/SessionEventTypes.cs:21-28`).

**O que o backend faz** (`EVENTS/src/modules/ingestion/handlers/session.handler.ts`):
**INSERT puro, sem `ON CONFLICT`** (`:95-100`).
**Rejeita o item** quando `tipoTransicao` e `tipoEvento` estao ambos ausentes, com motivo
`"SessionEvent sem tipoTransicao/tipoEvento"` (`:58-66`). `motivo` truncado em 50 (`:27,68`).
Emite um sinal `SINAL_EVENT_SESSION_SAVED` por linha (`:109-122`).
Tabela `eventos_sessao` (`EVENTS/.../eventos.schema.ts:120-134`), coluna `tipo_evento varchar(32) NOT NULL` (`:129`).

---

### 5.4 `InputSummaryEvent` (= `ActivitySummaryEvent`)

**Quando o Agent emite:** a cada **180 s** em producao
(`AGENT/.../Worker.cs:148-149,312-326`; `AGENT/src/ManagerAgent.SessionWorker/appsettings.json:21`
— o **default do codigo e 60 s**, o appsettings manda 180).

**Payload** (`AGENT/.../Capture/Models/ResumoAtividadeEntrada.cs:196-263`):

| campo | tipo | limite backend | o Agent manda? | citacao |
|---|---|---|---|---|
| `iniciadoEm` | ISO | — | **sim** | `ResumoAtividadeEntrada.cs:198-199` / `dto:263` |
| `finalizadoEm` | ISO | — | **sim** | `:201-202` / `dto:264` |
| `intervaloInicio` / `intervaloFim` | ISO | — | **NAO** (usa os dois de cima) | `dto:265-266`; fallback em `activity-summary.handler.ts:96-97` |
| `teclasPressionadas` | int >= 0, default 0 | — | **sim** | `:210-211` / `dto:267` |
| `teclasDistintas` | int >= 0, default 0 | — | **sim** | `:216-217` / `dto:268` |
| `cliquesMouse` | int >= 0, default 0 | — | **NAO** | `dto:269` |
| `cliquesEsquerdo` / `cliquesDireito` / `cliquesMeio` | int >= 0 | — | **sim** | `:204-208,213-214` / `dto:270-272` |
| `scrollsVertical` | int >= 0 | — | **sim** | `:222-223` / `dto:276` |
| `movimentosMouse` | int >= 0, default 0 | — | **NAO** | `dto:277` |
| `distanciaMousePx` | int >= 0 | — | **sim** | `:219-220` / `dto:278` |
| `padraoDigitacao` | enum | — | **sim** | `:225-226` / `dto:279` |
| `posicaoClickBucketsTop5` | array de `{bucketX,bucketY,count}` | `unknown` (nao valida) | **sim** | `:231-232,253-263` / `dto:281` |
| `deltaTCliquesAvgMs` / `deltaTCliquesStddevMs` | int >= 0 | — | **sim** | `:234-238` / `dto:282-283` |
| `teclaDominanteVk` | int | — | **sim** | `:240-241` / `dto:284` |
| `teclaDominanteSharePct` | numero 0..100 | — | **sim** | `:243-244` / `dto:285` |
| `teclasNavegacao` | int >= 0 | — | **sim** | `:246-247` / `dto:286` |
| `mouseJigglerSuspeito` | bool | — | so quando `true` | `:249-250` / `dto:287` |
| `ipOrigem` | string | 0..255 | **NAO** | `dto:288` |

> **Divergencia 4 — tres colunas que nunca recebem dado real do Windows.**
> `cliquesMouse` e `movimentosMouse` sao `NOT NULL` no banco
> (`EVENTS/.../eventos.schema.ts:248,251`) e tem `.default(0)` no Zod (`dto:269,277`). O
> `ResumoAtividadeEntrada` **nao tem esses campos** — logo as duas colunas ficam **sempre 0** em
> maquina Windows. Os cliques reais estao em `cliques_esquerdo`/`_direito`/`_meio`.

> **Divergencia 5 — `distanciaMousePx` nao e distancia.** O valor e
> `movimentosMouse * 100` (`AGENT/.../InputActivityTracker.cs:435`) — uma contagem de eventos de
> movimento multiplicada por uma constante, nao pixels percorridos. **Nao use este campo como
> distancia em nenhum calculo.**

> **Divergencia 6 — `padraoDigitacao`: o Agent so emite alias.** Os valores gerados sao
> `NORMAL`, `BURST_MESMA_TECLA`, `NAVEGACAO_APENAS` e `MOUSE_APENAS`
> (`AGENT/.../InputActivityTracker.cs:403,407,411,417,421`). Nenhum e canonico. Depois da
> normalizacao (`dto:189-196`) o banco recebe `NATURAL`, `REPETITIVO`, `IRREGULAR` e `INDEFINIDO`.
> Os canonicos `ROBOTICO`, `BURST` e `STEADY` **nunca aparecem vindos de Windows.**
> Regras: `>=30 teclas e tecla dominante >=60%` -> `BURST_MESMA_TECLA`;
> `>=10 teclas e >=90% de navegacao` -> `NAVEGACAO_APENAS`; sem tecla mas com mouse ->
> `MOUSE_APENAS`; resto (inclusive zero input) -> `NORMAL` (`:398-422`).

**O que o backend faz** (`EVENTS/src/modules/ingestion/handlers/activity-summary.handler.ts`):
atende os tres nomes (`:51-55`), **INSERT puro** (`:167-172`).
**Rejeita o item** quando `intervaloInicio` **ou** `intervaloFim` nao resolve, com motivo
`"ActivitySummary sem intervaloInicio/intervaloFim"` e `tipoEvento` **fixo em
`'ActivitySummaryEvent'`** mesmo que tenha vindo com outro nome (`:96-105`).
E o **unico** handler sem `enviadoEmFallback`. `padraoDigitacao` ausente vira `'INDEFINIDO'`
(`:134`); `teclasNavegacao ?? 0` (`:140`); `mouseJigglerSuspeito ?? false` (`:141`).
Tabela `resumos_atividade_input` (`EVENTS/.../eventos.schema.ts:242-273`), com CHECK de
`padrao_digitacao` em `EVENTS/drizzle/migrations/20260812000000_canonical_event_constraints.sql:69-70`.

---

### 5.5 `HeartbeatEvent`

**Quando o Agent emite:** a cada **120 s** em producao
(`AGENT/.../HeartbeatService.cs:113,118-137`; `AGENT/src/ManagerAgent.SessionWorker/appsettings.json:20`
— o default do codigo e 60 s).

**Payload** (`AGENT/.../Capture/Models/EventoSinalVida.cs:136-145`):

| campo | tipo | obrig. backend | o Agent manda? | citacao |
|---|---|---|---|---|
| `enviadoEm` | ISO c/ offset | **sim** | **sim** | `EventoSinalVida.cs:140` / `dto:238` |
| `eventosPendentes` | int >= 0, default 0 | nao | **sim** | `EventoSinalVida.cs:138` / `dto:239` |
| `acessibilidadeAtiva` | bool | nao | **NAO** | `dto:245` |
| `ipOrigem` | string 0..255 | nao | **NAO** | `dto:240` |
| `quantidadeMonitores` | int | — | manda, **backend descarta** | `EventoSinalVida.cs:142` |
| `redeDisponivel` | bool | — | manda, **backend descarta** | `EventoSinalVida.cs:144` |

`eventosPendentes` = buffer autonomo do worker + backlog do Service informado no ultimo PONG
(`AGENT/.../Worker.cs:177-178`).

> **Divergencia 7.** `acessibilidadeAtiva` existe no contrato (`dto:241-245`) e o backend o
> denormaliza em `agentes.acessibilidade_ativa`
> (`EVENTS/.../ingestion.service.ts:386-388,409-419`). **O Agent Windows nunca envia o campo**
> (`grep -rn "acessibilidadeAtiva" src/` no repositorio do Agent nao retorna nada). A coluna
> nunca e atualizada a partir de Windows.

**O que o backend faz:** `HeartbeatHandler` faz **INSERT puro** em `batimentos` (`:64-69`),
usando `versaoAgente` e `descricaoSo` **do envelope, nao do item** (`:56-57`).
Nunca rejeita item (`:45,71`).

> **Ponto que muda desenho de API:** `agentes.ultimo_heartbeat_em` e atualizado em **todo lote**,
> tenha ele `HeartbeatEvent` ou nao — a chamada esta fora do dispatch, em
> `EVENTS/.../ingestion.service.ts:181-189,367-398`. Nao trate "ultimo heartbeat" como prova de
> que houve um `HeartbeatEvent`.

---

### 5.6 `StatusTransitionEvent`

Ver secao 4. Payload (`AGENT/.../Capture/Models/EventoTransicaoStatus.cs:164-191`):

| campo | tipo | obrig. backend | limite | citacao |
|---|---|---|---|---|
| `statusAnterior` | enum | **sim** | — | `EventoTransicaoStatus.cs:174-175` / `dto:292` |
| `statusNovo` | enum | **sim** | — | `:178-179` / `dto:293` |
| `transicaoEm` | ISO c/ offset | **sim** | — | `:182-183` / `dto:294` |
| `motivo` | string (`null` -> `undefined`) | nao | 0..255 | `:189-190` / `dto:295` |
| `ipOrigem` | string | nao | 0..255 | **o Agent nao manda** / `dto:296` |

`motivo` gerado pelo Agent (`AGENT/.../UserStatusManager.cs:136-139`):
`"Atividade do colaborador"` quando volta a ATIVO; `"Silencio de entrada (X -> Y)"` nos demais.

---

### 5.7 `MeetingEvent`

**Quando o Agent emite** (`AGENT/.../MeetingDetector.cs`):

| momento | `finalizadoEm` | citacao |
|---|---|---|
| reuniao confirmada apos **1 minuto** de deteccao continua | ausente | `:265,325-348`; `Config/ManagerAgentUploadOptions.cs:100` |
| reuniao encerrada apos **2 minutos** sem sinal | preenchido | `:266,378-396`; `ManagerAgentUploadOptions.cs:106` |

Nos dois eventos `IniciadoEm = _detectedAt` (`:342,389`) — e o que reconcilia os dois no upsert.

**Payload** (`AGENT/.../Capture/Models/EventoReuniao.cs:35-68`):

| campo | tipo | obrig. backend | limite | citacao |
|---|---|---|---|---|
| `aplicativo` | string | nao (desde 18/08) | 0..100 | `EventoReuniao.cs:37-38` / `dto:307` |
| `tituloReuniao` | string | nao | 0..512 | `:40-41` / `dto:308` |
| `iniciadoEm` | ISO c/ offset | **sim** | — | `:43-44` / `dto:309` |
| `finalizadoEm` | ISO c/ offset | nao | — | `:46-47` / `dto:310` |
| `cameraAtiva` | bool | nao | — | `:49-50` / `dto:311` |
| `microfoneAtivo` | bool | nao | — | `:52-53` / `dto:312` |
| `participantesDetectados` | int >= 0 | nao | — | `:55-56` / `dto:313` |
| `renderAtivo` | bool | nao | — | `:61-62` / `dto:314` |
| `renderConfianca` | enum de 4 | nao | — | `:66-67` / `dto:315-317` |
| `ipOrigem` | string | nao | 0..255 | **o Agent nao manda** / `dto:318` |

**O que o backend faz** (`EVENTS/src/modules/ingestion/handlers/meeting.handler.ts`):
colapsa em memoria com chave `(agenteId, aplicativo, iniciadoEm)` usando `JSON.stringify` no
aplicativo, para casar com o `NULLS NOT DISTINCT` do indice (`:141-147`, justificativa `:128-139`);
**`onConflictDoUpdate`** em `(agente_id, aplicativo, iniciado_em)` (`:287-288`) com
`setWhere: finalizado_em IS NULL AND empresa_id = excluded.empresa_id` (`:300`) — **o unico handler
com guarda de tenant no `setWhere`** (`:273-277`).
`cameraAtiva`, `microfoneAtivo`, `renderAtivo` e `dispositivoTipo` sao sobrescritos sem COALESCE
(`:295-298`); o resto usa COALESCE (`:290-294`).
Nunca rejeita item — `aplicativo` em branco vira `null` e a linha e gravada (`:223-234,348`).
O pareamento do sinal emitido e **por chave, nao por indice**, porque `setWhere` pode derrubar
linhas (`:314-327`).
Tabela `eventos_reuniao` (`EVENTS/.../eventos.schema.ts:170-193`); `aplicativo` virou nulavel em
`EVENTS/drizzle/migrations/20260818000000_eventos_reuniao_aplicativo_nullable.sql`.

**Reconciliacao por igualdade exata, sem janela de tolerancia** (`:58-66`).

---

### 5.8 `MeetingSnapshotEvent`

**Quando o Agent emite:** a cada **60 s** enquanto a reuniao esta confirmada; o primeiro sai no
ato (GUID novo) e o tick interno e de 5 s
(`AGENT/.../MeetingSnapshotWorker.cs:30,34,89-126`).

**Payload** (`AGENT/.../Capture/Models/EventoReuniaoSnapshot.cs:86-115`):

| campo | tipo | obrig. backend | limite | citacao |
|---|---|---|---|---|
| `reuniaoInstalacaoId` | string (GUID) | **sim** | 1..36 | `EventoReuniaoSnapshot.cs:90-91` / `dto:322` |
| `aplicativo` | string | nao | 0..100 | `:93-94` / `dto:323` |
| `capturadoEm` | ISO c/ offset | **sim** | — | `:96-97` / `dto:324` |
| `cameraAtiva` | bool, default false | nao | — | `:99-100` / `dto:325` |
| `microfoneAtivo` | bool, default false | nao | — | `:102-103` / `dto:326` |
| `renderAtivo` | bool, default false | nao | — | `:105-106` / `dto:327` |
| `renderConfianca` | enum de 4 | nao | — | `:110-111` / `dto:328-330` |
| `participantesDetectados` | int >= 0 | nao | — | `:113-114` / `dto:331` |
| `ipOrigem` | string | nao | 0..255 | **o Agent nao manda** / `dto:332` |

O GUID nasce quando a reuniao entra em `Confirmed`
(`AGENT/.../MeetingDetector.cs:332`) e e limpo no encerramento (`:398`).

> **Divergencia 8 — o snapshot nao se liga ao evento pai por chave nenhuma.**
> `reuniaoInstalacaoId` existe **so** no snapshot. O `EventoReuniao`
> (`AGENT/.../Capture/Models/EventoReuniao.cs:35-68`) nao tem o campo, `meetingPayload`
> (`dto:306-319`) nao o aceita, e `eventos_reuniao` nao tem a coluna
> (`EVENTS/.../eventos.schema.ts:170-193`). Para correlacionar snapshot com reuniao hoje so resta
> `(agente_id, aplicativo)` mais a janela de tempo. **Quem for construir a API de "reuniao
> fantasma" precisa saber disso antes de comecar.**

**O que o backend faz:** **INSERT puro, sem dedup nenhum** — todo snapshot vira linha nova
(`EVENTS/src/modules/ingestion/handlers/meeting-snapshot.handler.ts:49-54`). Nunca rejeita
(`:27,56`). Nao grava `dispositivo_tipo` — a coluna nao existe nessa tabela
(`EVENTS/.../eventos.schema.ts:198-216`).

---

### 5.9 `PhoneCallEvent` — o Agent Windows nunca emite

Payload (`dto:335-339`): `tipoTransicao` (`CALL_START`|`CALL_END`, obrigatorio), `ocorreuEm`
(obrigatorio), `ipOrigem`.

**Se um lote com `dispositivoTipo` de desktop trouxer um `PhoneCallEvent`, TODOS os itens desse
tipo sao rejeitados** para `motivosIgnorados`, com o motivo
`"PhoneCallEvent so aceito para dispositivos moveis (MOBILE_ANDROID); dispositivoTipo=..."`
(`EVENTS/src/modules/ingestion/handlers/phone-call.handler.ts:42-49`). O texto cita so
`MOBILE_ANDROID` embora `IOS` tambem passe na guarda (`normalizar-dispositivo.ts:103-105`).

Ha ainda uma guarda de **lote** anterior: lote com `dispositivoTipo=IOS` que contenha qualquer
outro tipo de evento recebe **400 `DISPOSITIVO_TIPO_INVALIDO`**
(`EVENTS/.../ingestion.service.ts:205-224`).

Tabela `eventos_chamada_telefonica` (`EVENTS/.../eventos.schema.ts:310-318`) — sem coluna de numero
ou duracao, por LGPD (`:306-309`).

---

### 5.10 `ActivitySummaryEvent` e `InputActivitySummary`

Mesmo shape e mesmo handler do 5.4. O Agent Windows nao emite nenhum dos dois.

---

## 6. Duplicatas e idempotencia

### 6.1 As tres camadas de dedup

| camada | onde | escopo | citacao |
|---|---|---|---|
| Lote repetido (208) | header `X-Client-Dedup-Batch` | **por processo**, cache LRU de 10.000, TTL **10 min** | `EVENTS/src/modules/ingestion/services/dedup.service.ts:51-52,119-138` |
| Lote no pipe | `received_batches` no SQLite do Agent | por maquina, faxina de 1 h | `AGENT/.../SqliteEventBuffer.cs:99-102,631-632` |
| Linha no banco | chaves UNIQUE + `ON CONFLICT DO UPDATE` | por empresa/agente | ver 6.3 |

**O hash do lote** e SHA-256 do concat **ordenado** das chaves por evento
(`AGENT/src/ManagerAgent.Shared/Runtime/EventDedupKey.cs:72-82`); a chave por evento e
SHA-256 de `tipoEvento|dadosJson|capturedAtIso`
(`:62-66`), calculada na insercao no buffer (`AGENT/.../SqliteEventBuffer.cs:276`).

O backend: header ausente ou em branco -> sem dedup (`dedup.service.ts:120-122`); hash fora de
`/^[0-9a-f]{64}$/` -> **processa normalmente, nao rejeita** (`:55,124-126`); chave de cache =
`instalacaoId:hash` (`:127`). Duplicata resolvida devolve **208 com o corpo da resposta original**
(`ingestion.service.ts:151-162`; status em `ingestion.controller.ts:149,291-293`). Duplicata
concorrente (`IN_FLIGHT`) espera a original e tambem recebe 208 (`:155-162`).

> **Limitacao declarada no proprio codigo:** o cache e por processo. Com mais de uma replica atras
> do balanceador, o 208 nao funciona (`dedup.service.ts:36-42`).

O Agent trata 208 como sucesso (`AGENT/.../HttpEventUploader.cs:213-219`).

### 6.2 Chaves UNIQUE que existem hoje

**Nenhuma esta declarada no schema TypeScript.** Todos os `pgTable` em
`EVENTS/src/common/db/schema/eventos.schema.ts` sao a forma de 2 argumentos, sem callback de
indice (o proprio arquivo diz isso em `:65-66,96-98,166-168`). As chaves vivem **so nas migrations
SQL**.

| tabela | indice | colunas | `NULLS NOT DISTINCT`? | citacao |
|---|---|---|---|---|
| `eventos_janela` | `uq_eventos_janela_agente_processo_inicio` | `(agente_id, nome_processo, iniciado_em)` | **nao** | `EVENTS/drizzle/migrations/20260815000000_uq_eventos_janela_sessao.sql:72-73` |
| `eventos_ociosidade` | `uq_eventos_ociosidade_agente_inicio` | `(agente_id, iniciado_em)` | **nao** | `EVENTS/drizzle/migrations/20260815010000_uq_eventos_ociosidade_sessao.sql:92-93` |
| `eventos_reuniao` | `uq_eventos_reuniao_agente_app_inicio` | `(agente_id, aplicativo, iniciado_em)` | **SIM** | `EVENTS/drizzle/migrations/20260821000000_uq_eventos_reuniao_sessao.sql:180-181` |
| `eventos_sessao` | — | — | — | **nao existe** |
| `eventos_transicao_status` | — | — | — | **nao existe** |
| `eventos_reuniao_snapshot` | — | — | — | **nao existe** |
| `batimentos` | — | — | — | **nao existe** |
| `resumos_atividade_input` | — | — | — | **nao existe** |
| `eventos_chamada_telefonica` | — | — | — | **nao existe** |

Os tres indices sao **totais**, sem `WHERE` — nao sao parciais
(`20260815000000:12-14`). Nenhum usa `CREATE INDEX CONCURRENTLY`, porque o migrator roda cada
arquivo em transacao (`20260815000000:41-45`, `20260821000000:80,96`).

> Um `uq_eventos_ociosidade_agente_periodo` foi citado por anos no cabecalho do handler de
> ociosidade e **nunca existiu no banco** (`EVENTS/.../handlers/idle.handler.ts:26-33`;
> `20260815010000_uq_eventos_ociosidade_sessao.sql:24-32`). O `onConflictDoNothing` antigo nunca
> teve conflito para descartar.

### 6.3 O que faz dois eventos serem "o mesmo"

| tipo | chave | consequencia |
|---|---|---|
| janela | agente + processo + `iniciadoEm` **exato** | snapshot e fechamento reconciliam com a abertura |
| ociosidade | agente + `iniciadoEm` **exato** | idem |
| reuniao | agente + aplicativo + `iniciadoEm` **exato** | idem |
| resto | nada | toda emissao vira linha nova |

**Igualdade exata de timestamp, sem tolerancia** (`EVENTS/.../meeting.handler.ts:58-66`). Como o
`iniciadoEm` vem do Agent, **um ajuste de relogio entre a abertura e o fechamento produz duas
linhas** — exatamente a duplicacao que os indices existem para evitar.

### 6.4 Reconciliacao abertura/fechamento — `ON CONFLICT ... DO UPDATE`

Os tres handlers com upsert seguem a mesma forma:

1. **Colapso em memoria** do que veio no mesmo lote — obrigatorio, senao o Postgres devolve
   erro 21000 por atualizar a mesma chave duas vezes no mesmo comando
   (`window-activity.handler.ts:160-161`).
   Regra do merge: **`finalizadoEm` primeiro nao-nulo vence**; demais campos mutaveis, ultimo
   nao-nulo vence; `criadoEm` mantem o primeiro
   (`window-activity.handler.ts:91-97`; `idle.handler.ts:87-91`; `meeting.handler.ts:183-191`).
2. **`onConflictDoUpdate`** com `COALESCE(excluded.X, atual.X)` nos campos nulaveis.
3. **`setWhere: finalizado_em IS NULL`** — **uma vez fechada, a linha e imutavel**
   (`window-activity.handler.ts:185`; `idle.handler.ts:160`; `meeting.handler.ts:300`).

> **Armadilha de contagem.** Linha barrada pelo `setWhere` **nao volta no `RETURNING`**
> (`EVENTS/.../handler-utils.ts:26-31`). Por isso `inseridos` pode ser menor que o numero de
> eventos enviados (`window-activity.handler.ts:190`), e por isso `meeting.handler.ts` pareia por
> chave e nao por indice (`:314-327`). **Nao escreva API que case resposta com pedido por
> posicao.**

### 6.5 O caso `NULLS NOT DISTINCT` (24/08)

Em 18/08 `eventos_reuniao.aplicativo` deixou de ser `NOT NULL`
(`EVENTS/drizzle/migrations/20260818000000_eventos_reuniao_aplicativo_nullable.sql`) para que uma
reuniao sem aplicativo detectado nao derrubasse o evento.

Isso quebrou o upsert em silencio: **o default do Postgres em UNIQUE e `NULLS DISTINCT`** — duas
linhas com `aplicativo = NULL` nunca colidem, e o `ON CONFLICT` nunca acharia a linha aberta da
reuniao que a propria 18/08 tinha passado a salvar. Cada fechamento viraria uma segunda linha
(`20260821000000_uq_eventos_reuniao_sessao.sql:99-123`).

A correcao de 24/08 tem tres partes, e as tres importam para quem replicar o padrao:

1. `CREATE UNIQUE INDEX ... NULLS NOT DISTINCT` (Postgres 15+; o projeto roda 16) —
   `20260821000000:180-181`.
2. Um **`DROP INDEX IF EXISTS` defensivo antes** (`:174`): `CREATE UNIQUE INDEX IF NOT EXISTS`
   sozinho nao recria um indice que ja existe sem a propriedade (`:167-173`).
3. O colapso em memoria do handler precisou usar `JSON.stringify(aplicativo)` para que `NULL`
   vire `null` e um aplicativo literalmente chamado `"null"` continue distinto — as duas pontas
   **tem de concordar** (`meeting.handler.ts:128-147`).

`NULLS NOT DISTINCT` aparece **uma unica vez** em todo o SQL de producao.

> **Para API nova:** `ON CONFLICT (a, b, c)` infere o indice pela lista de colunas, e a
> propriedade `NULLS NOT DISTINCT` vem junto (`20260821000000:122-123`). Se alguma coluna da sua
> chave for nulavel e voce nao declarar `NULLS NOT DISTINCT`, **o upsert vira insert e ninguem
> percebe.**

### 6.6 Onde ainda ha colisao possivel

| caso | por que | citacao |
|---|---|---|
| **LOGOUT em duplicata** | `eventos_sessao` nao tem UNIQUE. O Agent tem duas guardas locais (despedida registrada; consulta ao buffer numa janela de 2 min), mas se o pipe estiver caido o LOGOUT do worker sobe depois, de outro processo, possivelmente apos reboot | `AGENT/src/ManagerAgent.Service/ManagerAgentService.cs:286-315,317-327`; `AGENT/.../SqliteEventBuffer.cs:302-345` |
| Transicao de status duplicada | tabela sem trava; e por isso que o Agent nao emite na fronteira de sessao nem no corte diario | `AGENT/.../IUserStatusManager.cs:162-168`; `AGENT/.../DailyBoundaryWorker.cs:16-19` |
| Snapshot de reuniao repetido | INSERT puro | `EVENTS/.../meeting-snapshot.handler.ts:49-54` |
| Heartbeat / resumo repetido | INSERT puro | `heartbeat.handler.ts:64-69`; `activity-summary.handler.ts:167-172` |
| Lote reenviado apos 10 min | cache de dedup expirou | `dedup.service.ts:52` |
| Lote reenviado para outra replica | cache e por processo | `dedup.service.ts:36-42` |

---

## 7. Tempo

### 7.1 Formato

| item | regra | citacao |
|---|---|---|
| Formato exigido | ISO 8601 **com offset obrigatorio** | `EVENTS/.../dto/agent-event-batch.dto.ts:206-209` (`z.iso.datetime({offset:true})`) |
| Mensagem de erro | `"timestamp deve ser ISO 8601 com offset"` | `dto:208` |
| Colunas | **`timestamptz`** em todas as tabelas | `EVENTS/src/common/db/schema/eventos.schema.ts` — nenhum `withTimezone:false` no repo |
| Precisao | `precision: 6` nas colunas antigas; sem precisao explicita nas novas | ex. `:72` vs `:204` |

**Nao ha configuracao de fuso em lugar nenhum** do `EVENTS`: nem no `Dockerfile`, nem no
`docker-compose.yml`, nem no cliente `postgres.js`
(`EVENTS/src/common/db/db.client.ts:48-54` — so `prepare`, `ssl`, `max`, timeouts). A correcao
depende inteiramente de `timestamptz` + o offset que o Agent manda.

### 7.2 Qual relogio cada carimbo usa

| carimbo | relogio | citacao |
|---|---|---|
| `enviadoEm` (envelope) | `DateTimeOffset.UtcNow` — **UTC, offset +00:00** | `AGENT/.../HttpEventUploader.cs:161` |
| Janela, heartbeat, resumo, reuniao, status | `DateTimeOffset.Now` — **hora local com offset** (ex. `-03:00`) | `AGENT/.../Worker.cs:202` (o `now` do laco) |
| Ociosidade | `DateTimeOffset.UtcNow` | `AGENT/.../IdleMonitorService.cs:95,413` |
| Snapshot de reuniao | `DateTimeOffset.UtcNow` | `AGENT/.../MeetingSnapshotWorker.cs:62` |
| `LOGOUT` emitido pelo Service | hora do EOF do pipe, ou `UtcNow` | `AGENT/src/ManagerAgent.Service/ManagerAgentService.cs:299` |
| Fechamento de janela orfa | hora do EOF do pipe | `AGENT/.../OrphanWindowEventCloser.cs:51,80` |
| `criado_em` (todas as tabelas) | **`new Date()` do Node**, um por invocacao de handler | ex. `window-activity.handler.ts:136,154` |
| `agentes.ultimo_heartbeat_em` | `new Date()` do Node | `EVENTS/.../ingestion.service.ts:373,382` |
| Timestamp faltando no payload | **`enviadoEm` do envelope** (hora do Agent) | `EVENTS/.../ingestion.service.ts:237`; `event-handler.interface.ts:28` |

> **Consequencia pratica:** dentro do mesmo lote convivem carimbos com offset `-03:00` (janela) e
> `+00:00` (ociosidade). Como todos carregam offset e todas as colunas sao `timestamptz`, o
> instante e o mesmo — mas **nunca compare a string** e nunca assuma que o offset e uniforme.

### 7.3 Armadilhas ja medidas

**a) Fim anterior ao inicio (achado A-30, reaberto).**
A ociosidade e datada **retroativamente** na ultima interacao do usuario, mas so e detectada quando
cruza o limiar de 60 s. Uma janela que ganha foco dentro desse intervalo silencioso — worker
reiniciando, aplicativo roubando foco — abre legitimamente e, ate 59 s depois, recebe um fim
anterior ao proprio inicio.

O Agent **nao descarta** o evento (ele ja foi emitido como aberto; engolir o fechamento o deixaria
orfao para sempre — achado A-35). Ele fecha com **duracao ZERO** e loga WARN
(`AGENT/.../WindowActivityService.cs:216-239`). O mesmo clamp existe na troca de janela
(`:158-160`).

Ate a 1.5.5 isso virava um bloco de **1 ms** — valor fabricado com cara de dado real.

**Medicao:** o registro de regressivo de 18/08 aponta o evento `49253` com 1 ms as 14:48:26, com o
worker subindo (`BRAIN/docs/tecnologia/registro/2026-08-24-regressivo-definitivo-agent.md:659-671`;
placar em `:102,146`).
**NAO VERIFICADO:** a ocorrencia especifica de 24/08 "na troca de servico" citada no pedido. O que
esta registrado e a medicao de 18/08 acima; nao encontrei registro datado de 24/08 para este
sintoma. O caminho de codigo e o mesmo nos dois casos.

**Para quem consome:** blocos de `finalizado_em = iniciado_em` (duracao 0) existem e sao
legitimos — significam "nao houve atividade", nao "erro". Filtre-os; nao os trate como dado ruim.

**b) Ajuste de relogio parte o intervalo.** Ver 6.3.

**c) Buraco de wall-clock.** Se o processo dormiu (S3/S4) ou o SO travou, `GetLastInputInfo` e
`TickCount` nao veem o buraco. O Agent compara wall-clock entre ciclos e, se o delta passar de
`2 x intervalo + 30 s` (= 40 s na config atual), emite um `IdleEvent` sintetico cobrindo o periodo,
com motivo `WallClockGap` (`AGENT/.../IdleMonitorService.cs:217-253`).

**d) Ociosidade reaberta apos restart.** O inicio persistido e reaproveitado quando o recalculo cai
dentro de **5 s** dele; fora disso comeca uma nova (`AGENT/.../IdleMonitorService.cs:41,140-154`).
E o que evita duas linhas nao reconciliadas (achado A-48).

**e) Corte diario.** O `DailyBoundaryWorker` flusha ociosidade e janela as 23:59 **em hora local**
(`AGENT/.../DailyBoundaryWorker.cs:50-51`) para que nenhum bloco atravesse a fronteira do dia.

---

## 8. O contrato de erro

### 8.1 202 Accepted — o caso normal

Corpo (`EVENTS/.../dto/agent-event-batch.dto.ts:537-602`):

| campo | significado | citacao |
|---|---|---|
| `aceito` | bool | `dto:538-539` |
| `totalEventos` | **linhas efetivamente gravadas** | `dto:541-545`; **valor real** em `EVENTS/.../ingestion.service.ts:249` |
| `eventosJanela`, `eventosInatividade`, `eventosSessao`, `resumosEntrada`, `eventosHeartbeat`, `eventosTransicaoStatus`, `eventosReuniao`, `eventosReuniaoSnapshot`, `eventosChamadaTelefonica` | contadores por tipo | `dto:547-594`; `ingestion.service.ts:250-261` |
| `motivosIgnorados` | array de `{indice, tipoEvento, motivo}` | `dto:520-535,596-601` |

> **Divergencia 9.** A descricao Swagger de `totalEventos` diz "aceitas + ignoradas"
> (`dto:541-545`). O valor real e `dispatchResult.totalInseridos`
> (`EVENTS/.../ingestion.service.ts:249`) — **so o que foi gravado**. Nao use este campo para
> conferir se o lote chegou inteiro.

`resumosEntrada` soma os tres aliases de activity summary
(`EVENTS/.../ingestion.service.ts:253-256`).

### 8.2 `motivosIgnorados` — o significado exato de `indice`

**`indice` e sempre a posicao no array `eventos` do lote ORIGINAL**, comecando em 0. Nunca a
posicao numa lista filtrada. A cadeia inteira:

1. `validarEventos` itera `crus.entries()` sobre o array intocado e guarda a posicao tanto para o
   item valido quanto para o rejeitado (`dto:474-486`; doc em `:462-465`).
2. O item valido carrega a posicao em `EventoComIndice.indice` (`dto:455-458`).
3. O agrupamento por tipo usa `evento.indice`, **nao** a posicao no array agrupado
   (`EVENTS/.../ingestion.service.ts:334`, comentario em `:323-326`).
4. O handler recebe `originalIndices` e le `originalIndices[i] ?? i`
   (`session.handler.ts:57`; `activity-summary.handler.ts:95`; `phone-call.handler.ts:44`).

**O array nao vem ordenado por `indice`.** Ele e montado como
`[...rejeitados pelo schema, ...rejeitados pelos handlers]`
(`EVENTS/.../ingestion.service.ts:264`) — os primeiros em ordem de lote, os segundos em ordem de
despacho (`:355`). Ordene voce mesmo se precisar.

`tipoEvento` do motivo e "melhor esforco": extraido do objeto cru, truncado em 64 chars, ou
`"(desconhecido)"` (`dto:495-501`). `motivo` sao as 3 primeiras issues do Zod, truncadas em 500
(`dto:504-510`).

**Todos os pontos que geram `motivosIgnorados`:**

| origem | condicao | motivo |
|---|---|---|
| schema do item | qualquer falha no `AgentEventItemSchema` (inclui `tipoEvento` desconhecido) | issues do Zod |
| `SessionHandler` | `tipoTransicao` e `tipoEvento` ambos ausentes | `"SessionEvent sem tipoTransicao/tipoEvento"` (`session.handler.ts:59-65`) |
| `ActivitySummaryHandler` | `intervaloInicio` **ou** `intervaloFim` nao resolve | `"ActivitySummary sem intervaloInicio/intervaloFim"` (`activity-summary.handler.ts:98-105`) |
| `PhoneCallHandler` | `dispositivoTipo` nao e movel | `"PhoneCallEvent so aceito para dispositivos moveis..."` (`phone-call.handler.ts:42-49`) |

Os outros seis handlers **nunca** rejeitam item.

> **O QUE O AGENT FAZ COM ISSO: NADA.** Em resposta 2xx ele retorna sem ler o corpo
> (`AGENT/.../HttpEventUploader.cs:221-227`) e o `UploadWorker` marca todos os eventos como
> enviados (`AGENT/.../UploadWorker.cs:353-355`), removendo-os do buffer. **Evento rejeitado
> individualmente esta perdido e ninguem do lado do Agent fica sabendo.** Ver defeito D-3.

### 8.3 Quando vem 400

| codigo | quando | citacao |
|---|---|---|
| `PAYLOAD_INVALIDO` | falha no **envelope**: campo de lote faltando, tipo errado, `eventos` vazio ou > 1000, `enviadoEm` sem offset, `dispositivoTipo` fora do enum, ou **qualquer nome de campo desconhecido no lugar do canonico** | `EVENTS/.../ingestion.service.ts:119,127-129`; `detalhes.issues` em `:122-126` |
| `DISPOSITIVO_TIPO_INVALIDO` | **apenas** lote com `dispositivoTipo=IOS` contendo evento que nao seja `PhoneCallEvent` | `EVENTS/.../ingestion.service.ts:205-224` |

> **Divergencia 10.** `EVENTS/src/common/errors/error-codes.ts:29` e
> `dto:6-7` afirmam que `dispositivoTipo` ausente ou invalido devolve
> `DISPOSITIVO_TIPO_INVALIDO`. **Nao devolve** — cai no Zod e vira `PAYLOAD_INVALIDO`. O unico
> ponto que lanca aquele codigo e a guarda de composicao de lote IOS.

**A regra que vale hoje: erro de LOTE e 400; erro de ITEM nunca e 400**
(`dto:443-444`). Isso mudou em 24/08, e mudou por causa do teto do A-64: da 1.5.12 em diante um
unico item invalido podia levar ate 99 eventos bons junto, em silencio (`dto:436-444`).

### 8.4 Os outros status

| status | codigo | quando | o Agent faz |
|---|---|---|---|
| **208** | — | lote ja processado (hash) | trata como sucesso (`HttpEventUploader.cs:213-219`) |
| **401** | `TOKEN_EXPIRADO` / `TOKEN_INVALIDO` / `CHAVE_INVALIDA` | JWT ruim, ausente ou fora do esquema Bearer | forca refresh e **retenta a mesma tentativa**; se o refresh falhar, `FalhaTransitoria` **e marca revogado** (`HttpEventUploader.cs:229-248`) |
| **403** | `EMPRESA_BLOQUEADA`, `USUARIO_INATIVO`, `DISPOSITIVO_DESVINCULADO`, `AGENT_REVOGADO` | guardas de negocio (`EVENTS/src/modules/ingestion/guards/device-guards.service.ts:48-78`); `AGENT_REVOGADO` e o unico que emite o header `X-Agent-Revoked: 'true'` (`EVENTS/src/common/auth/revocation-check.middleware.ts:39,167-169`, acionado pelo interceptor em `EVENTS/src/common/auth/revocation-check.interceptor.ts:23-34` e `EVENTS/.../ingestion.controller.ts:161`) | **so age se vier o header `X-Agent-Revoked: true`**; ai suspende a captura. 403 sem o header cai no ramo generico e vira `FalhaTransitoria` (`HttpEventUploader.cs:62-65,250-264`) |
| **404** | `DISPOSITIVO_NAO_CADASTRADO` | agente ou empresa nao existe (`EVENTS/.../services/empresa-resolver.service.ts:110-136`) | `FalhaPermanente` -> **descarta apos 5 ciclos** |
| **412** | `DISPOSITIVO_NAO_VINCULADO` | `usuario_ref_id` nulo (`empresa-resolver.service.ts:118-124`) | `FalhaPermanente` -> **descarta apos 5 ciclos** |
| **429** | `RATE_LIMITED` | > 60 lotes/min por `instalacaoId` | `FalhaTransitoria`, retenta para sempre (`ResultadoEnvio.cs:119`) |
| **5xx** | — | erro interno | `FalhaTransitoria` |

O `Retry-After: 60` e fixo e vem do filtro global, nao do controller
(`EVENTS/src/common/errors/global-exception.filter.ts:88-91`). **O Agent nao le esse header** —
ele simplesmente espera o proximo ciclo de 60 s.

**Corpo de erro** (`EVENTS/src/common/errors/base.exception.ts:30-58`):
`{ codigo, mensagem, detalhes, requestId? }`. `detalhes` sempre presente (`{}` quando vazio);
`requestId` vem do contexto de correlacao (`global-exception.filter.ts:77-86`).

> **404 e 412 sao permanentes para o Agent.** Uma maquina que perdeu o vinculo do lado do backend
> perde ate 100 eventos a cada cinco minutos, em vez de segurar a fila. Isso e deliberado
> (`ResultadoEnvio.cs:98-107`), mas quem administrar vinculos precisa saber.

---

## 9. Compatibilidade

### 9.1 Tipo de evento desconhecido

O `AgentEventItemSchema` e uma `discriminatedUnion` por `tipoEvento`
(`dto:346-376`). Tipo fora dos 11 **nao casa com nenhum branch** -> falha o parse do item ->
vai para `motivosIgnorados` com o nome truncado em 64 chars (`dto:481-486,495-501`).
**Nao derruba o lote.** O `handler-registry.get()` tambem devolve `undefined` para tipo
desconhecido (`handler-registry.ts:89-91`), mas na pratica o Zod barra antes.

### 9.2 Campo novo

| direcao | comportamento | citacao |
|---|---|---|
| Agent manda campo que o backend nao conhece **dentro de `dados`** | `z.object` do Zod **descarta** a chave; o evento e aceito | comportamento padrao do `z.object`; caso real em 5.2 (Divergencia 3) |
| Agent manda campo que o backend nao conhece **no envelope** | o envelope tambem e `z.object` sem `.strict()` (`dto:446-449`) — descartado, sem 400 | `dto:446-449` |
| Backend passa a exigir campo que o Agent antigo nao manda | **400 no lote inteiro** -> descarte apos 5 minutos | secao 2.6 |
| Backend acrescenta campo opcional | Agent antigo continua funcionando | ex. `acessibilidadeAtiva` em `dto:241-245` |
| Backend acrescenta campo na **resposta** | o Agent nao le a resposta em sucesso | `HttpEventUploader.cs:221-227` |

### 9.3 Versao velha de Agent contra backend novo

| item | situacao | citacao |
|---|---|---|
| Aliases de valor | **ainda obrigatorios** enquanto a 1.5.x nao dominar; sunset previsto para 100% da frota em 1.5.0+ | `dto:81-84` |
| `versaoAgente` que o Agent manda | `1.5.16.0` — **4 partes**, versao do assembly | `AGENT/.../HttpEventUploader.cs:140` |
| `versaoAgente` que o contrato de update usa | `1.5.16` — 3 partes | `AGENT/src/ManagerAgent.Shared/Runtime/VersaoSemVer.cs:22-24,41-55` |
| `dispositivoTipo` nos eventos | `WINDOWS` (legado), normalizado para `DESKTOP_WINDOWS` | `AGENT/src/ManagerAgent.Shared/Runtime/DispositivoTipoDetector.cs:18,34-35`; `EVENTS/.../normalizar-dispositivo.ts:74-75` |
| `dispositivoTipo` na vinculacao | `DESKTOP_WINDOWS` (canonico) | `AGENT/.../AgentLinkService.cs:176` |

> **Divergencia 11 — o mesmo Agent manda dois valores diferentes de `dispositivoTipo`.**
> `WINDOWS` para o `EVENTS`, `DESKTOP_WINDOWS` para o `ADMIN`. Os dois sao aceitos, mas API que
> comparar os dois campos sem normalizar vai errar.

> **Divergencia 12 — os cabecalhos de upgrade nunca sao emitidos para o Agent Windows.**
> `X-Manager-Upgrade-Available` e `X-Manager-Upgrade-Required` so sao calculados quando a
> requisicao traz `X-Agent-Version`; sem esse header o backend nem consulta o banco
> (`EVENTS/src/modules/ingestion/services/upgrade-header.service.ts:107-110`, aplicacao em
> `EVENTS/.../ingestion.controller.ts:302-308`). **O Agent envia apenas tres headers**
> (`AGENT/.../HttpEventUploader.cs:198-205`) e `X-Agent-Version` nao esta entre eles. Todo o
> mecanismo esta morto para Windows. Ver defeito D-2.

### 9.4 Backend novo contra Agent velho — e a ORDEM DE DEPLOY

> **BACKEND PRIMEIRO, AGENT DEPOIS. Sempre.**

A razao esta na assimetria de quem pode esperar:

- O **backend** e um lugar so, atualizado em minutos, e pode aceitar o formato velho e o novo ao
  mesmo tempo. Todos os aliases da secao 3.2 existem exatamente por isso.
- O **Agent** e a frota. Ele se atualiza sozinho, mas **a cada 6 horas**
  (`AGENT/src/ManagerAgent.Service/Update/UpdateCheckerWorker.cs:23`), com jitter inicial de ate
  30 minutos (`AGENT/src/ManagerAgent.Service/appsettings.json:20`), e so quando a maquina esta
  ligada. Uma maquina desligada uma semana volta na versao antiga.

Se o Agent for primeiro, ele manda um formato que o backend ainda nao aceita: **400 em todo lote**,
e o teto do A-64 apaga os eventos em cinco minutos (secao 2.6). Se o backend for primeiro, o pior
caso e uma coluna nova ficar nula ate a frota alcancar.

**Regra pratica:** todo campo novo nasce **opcional** no backend e so pode virar obrigatorio
quando a frota inteira o mandar. E a mesma regra ja escrita em `dto:46-48` para tipo de evento
novo, generalizada.

### 9.5 O canal de comando (recall)

Nao ha endpoint de comando. O unico canal backend->Agent e a **resposta do endpoint de
atualizacao**, `GET {baseUrlAdmin}/api/agente/atualizacoes/verificar?versaoAtual=...`
(`AGENT/.../UpdateCheckerWorker.cs:602`;
`ADMIN/src/modules/agent-update/agent-update.controller.ts:103,118`).

O recall vem como `recallSolicitado: true` **junto com `atualizacaoDisponivel: false`**
(`ADMIN/src/modules/agent-update/agent-update.service.ts:312-316`;
`AGENT/src/ManagerAgent.Service/Update/UpdateDownloader.cs:241-246`). Um agente que trate recall
depois de um `if (!temUpdate) return;` **nao trata recall nenhum** — era exatamente esse o defeito
corrigido em `AGENT/.../UpdateDownloader.cs:55-70`.

O backend nao diz para qual versao voltar: quem sabe e a maquina, que guarda uma unica versao
anterior (`ADMIN/src/modules/agent-update/dto/verificar-atualizacao.response.dto.ts:117-127`).

---

## 10. O que o Agent NUNCA manda (LGPD)

| nao envia | envia no lugar | onde isso e garantido |
|---|---|---|
| **Teclas digitadas** | apenas **contagens**: total, distintas, navegacao | `AGENT/.../InputActivityTracker.cs:273-281` — `RecordKey(vkCode)` incrementa um `Dictionary<int,int>` (`:49`). Nao ha nenhuma string acumulada em todo o arquivo. |
| **Tela / screenshot** | nada | nao existe captura de tela em nenhum ponto do repositorio |
| **Audio** | apenas dois booleanos (camera/microfone ativos) | `AGENT/.../MediaDeviceMonitor.cs:159-181` — le `AudioSessionState` via WASAPI; nunca abre buffer de audio |
| **Camera / imagem** | apenas o booleano | `AGENT/.../MediaDeviceMonitor.cs:186-200` — le a chave de consentimento do registro do Windows, nao o dispositivo |
| **URL completa** | apenas o **dominio** | `AGENT/src/ManagerAgent.SessionWorker/Capture/BrowserUrlParser.cs:13-15` — o regex `https?://([^/\s\)]+)` captura **so o host**; a partir da primeira `/` nada e capturado (`:67-68`) |
| **Conteudo de documento ou formulario** | apenas o **titulo da janela** | `AGENT/.../Capture/Models/EventoJanela.cs:23` |
| **Numero de telefone / duracao de chamada** | so `CALL_START`/`CALL_END` (e nem isso em Windows) | `EVENTS/src/common/db/schema/eventos.schema.ts:306-309` |
| **Identificadores de hardware individuais** | apenas o **hash SHA-256** | `AGENT/.../AgentLinkService.cs:154-159` — BIOS/MB/CPU/MAC/Volume ficam em `installation.db` local |

**Quando nao ha URL**, o parser cai numa tabela de titulos conhecidos que devolve tambem apenas um
dominio (`BrowserUrlParser.cs:17-58,70-75`) — ex.: titulo contendo "WhatsApp" vira
`web.whatsapp.com`. Nao ha caminho, query string nem fragmento em lugar nenhum.

**Ruido de shell nao vira evento**: area de trabalho, bandeja e menu iniciar sao descartados
quando o processo e de shell **e** o titulo e placeholder
(`AGENT/.../WindowActivityService.cs:289-298`;
`AGENT/.../Config/ManagerAgentUploadOptions.cs:156-186`).

### Dois campos que merecem leitura atenta antes de virarem API publica

Nenhum dos dois viola os limites acima, mas os dois sao mais especificos do que "contagem":

1. **`teclaDominanteVk`** — o codigo de virtual-key da tecla mais pressionada no intervalo, e o
   percentual dela (`AGENT/.../InputActivityTracker.cs:384-391,443-444`). Nao e conteudo — nao ha
   sequencia nem contexto —, mas **identifica uma tecla**. Coluna `tecla_dominante_vk integer`
   (`EVENTS/.../eventos.schema.ts:268`).
2. **`posicaoClickBucketsTop5`** — os cinco pontos mais clicados da tela, em blocos de **50x50
   pixels** (`AGENT/.../InputActivityTracker.cs:307,342-357`). Nao e a tela, e uma grade grossa de
   coordenadas. Coluna `posicao_click_buckets_top5 jsonb` (`EVENTS/.../eventos.schema.ts:265`),
   **sem validacao de shape** no Zod (`dto:281` usa `z.unknown()`).

Os dois existem para detectar automacao (auto-clicker, mouse jiggler) —
`AGENT/.../InputActivityTracker.cs:17`. Se forem expostos em API de gestor, e decisao de produto,
nao de engenharia.

---

## 11. Defeitos abertos que afetam quem consome o contrato

### 11.1 Ja conhecidos

| id | uma linha | citacao |
|---|---|---|
| **A-62** | `eventos_sessao` nao tem UNIQUE: LOGOUT em duplicata continua possivel quando o pipe cai e o evento do worker sobe depois, de outro processo. As guardas do Agent tornam o caso raro; so uma trava no banco o tornaria impossivel. | `AGENT/src/ManagerAgent.Service/ManagerAgentService.cs:317-327`; ausencia confirmada em `EVENTS/drizzle/migrations/` |
| **A-64** | Lote recusado por conteudo 5 vezes e **descartado** — ate 100 eventos perdidos por vez, com um unico log de erro. | `AGENT/.../UploadWorker.cs:42,396-421` |
| **Recall so alcanca quem ja entende o comando** | O recall e um campo novo na resposta de update; Agent anterior a **1.5.12** ignora o campo em silencio e segue o fluxo normal. Frota antiga nao pode ser revogada. | `AGENT/.../UpdateDownloader.cs:236-242`; `git show 39d20e3:.../ManagerAgent.Service.csproj` = `1.5.12`; limitacao declarada em `ADMIN/src/modules/fleet-health/fleet-health.controller.ts:342-350` |
| **`/health` nao informa a versao da aplicacao** | Os dois backends devolvem `version`, mas e o **SHA do git truncado em 7 chars** (ou a string `unknown`), nunca um semver. Nao da para conferir qual release esta no ar. | `EVENTS/src/modules/health/health.controller.ts:94-103`; `ADMIN/src/modules/health/health.controller.ts:55,62,74-80` |
| **Revogacao existe, mas ninguem a liga** | `agentes.revogado_em` e **so lido**, nunca escrito: `grep -rn "revogado_em\|revogadoEm" src/` nos dois backends devolve apenas leituras, e nao ha endpoint de revogacao em nenhuma das listas de rota. O recurso esta dormente. | `ADMIN/src/common/auth/revocation-check.interceptor.ts:89`; `ADMIN/src/common/db/schema/agentes.schema.ts:10,41-42`; `EVENTS/src/common/auth/revocation-check.middleware.ts:78-85`; `EVENTS/src/modules/ingestion/services/empresa-resolver.service.ts:48,92` |
| **Dedup de lote nao sobrevive a mais de uma replica** | O cache do 208 e por processo. | `EVENTS/.../services/dedup.service.ts:36-42` |

### 11.2 Achados novos desta leitura — NAO corrigidos

Registrados aqui porque a tarefa era ler, nao consertar.

**D-1 — Falha passageira no refresh de token revoga a maquina, e a revogacao nao se desfaz sozinha.
(GRAVE)**

Cadeia conferida:

1. `TokenManager.RefreshTokenAsync` lanca `TokenRefreshFailedException` em **qualquer** status
   nao-2xx do `/api/agente/auth/refresh` — `AGENT/src/ManagerAgent.Shared/Auth/TokenManager.cs:180-186`.
2. O `HttpEventUploader` captura essa excecao e chama `MarkRevokedOnRefreshFailure`, que marca o
   dispositivo como **revogado** sem olhar o status —
   `AGENT/.../HttpEventUploader.cs:78-81,121-131,240-247`.
3. O comentario XML logo acima afirma que "o `TokenManager` so lanca em 401/403 do endpoint de
   refresh" (`HttpEventUploader.cs:69-71`). **Isso e falso** — ver 1.
4. O `/refresh` do `ADMIN` devolve **429** ao passar de 20 requisicoes por minuto **por IP**
   (`ADMIN/src/modules/agent-auth/agent-auth.controller.ts:68-69,108-113`). Varias maquinas atras
   do mesmo NAT corporativo compartilham o IP. 5xx tambem chega ali.
5. `MarkRevoked` grava `revogadoEm` no `config.json`, que **sobrevive a reboot**
   (`AGENT/src/ManagerAgent.Service/Linking/LinkStatus.cs:188-208`;
   `AGENT/src/ManagerAgent.Shared/Config/ConfigLocal.cs:77-78`).
6. Com a marca, `CanCapture` vira `false` (`LinkStatus.cs:96-97`) e o `UploadWorker` **pausa a
   captura de todas as sessoes** a cada ciclo (`AGENT/.../UploadWorker.cs:104-107,115-163`).
7. `ClearRevocation` tem **um unico chamador**, dentro do caminho completo de vinculacao
   (`AGENT/.../AgentLinkService.cs:244`) — e esse caminho **nunca e alcancado**, porque
   `EnsureLinkedAsync` retorna `AlreadyLinked` logo no inicio quando ja ha token e data no config
   (`AGENT/.../AgentLinkService.cs:84-85`).

**Efeito:** um 429 ou um 5xx passageiro no endpoint de refresh para a captura da maquina **em
definitivo**, ate alguem re-vincular a mao. O envio continua (o `AlreadyLinked` nao interrompe o
ciclo), entao a maquina fica "viva" no painel enquanto nao produz mais nenhum evento.
**Para quem consome o contrato:** uma maquina pode ficar silenciosa por um erro de rede, sem
nenhum sinal do lado do backend.

**D-2 — Os cabecalhos de upgrade nunca chegam ao Agent Windows.**
O backend so calcula `X-Manager-Upgrade-Available` / `X-Manager-Upgrade-Required` quando a
requisicao traz `X-Agent-Version`
(`EVENTS/.../services/upgrade-header.service.ts:107-110`), e o Agent envia apenas `Authorization`,
`X-Installation-Id` e `X-Client-Dedup-Batch` (`AGENT/.../HttpEventUploader.cs:198-205`).
O mecanismo esta completo dos dois lados e desligado por um header que ninguem manda.

**D-3 — `motivosIgnorados` nao chega a lugar nenhum.**
O Agent nao le o corpo da resposta em 2xx (`AGENT/.../HttpEventUploader.cs:221-227`) e marca tudo
como enviado (`AGENT/.../UploadWorker.cs:353-355`). Evento rejeitado individualmente e apagado do
buffer sem registro local. O campo existe, e completo, e e escrito com cuidado — e nenhum
consumidor o le.

**D-4 — Curinga que decide "ATIVO".** Ver a caixa de destaque na secao 4.
Latente hoje (`UserStatus` tem 3 valores, os 3 tratados — `AGENT/src/ManagerAgent.SessionWorker/Capture/Models/UserStatus.cs:17,20,23` vs
`UserStatusManager.cs:147-149`), ativo no dia em que o enum crescer. Agravante: o curinga desliga
o `CS8509` promovido a erro em `AGENT/Directory.Build.props:22`, que e justamente a rede que
pegaria isso no build. Caso irmao, mesma forma, impacto menor:
`AGENT/.../RenderConfianca.cs:36` (`_ => "INCERTO"`), tambem latente, e cujo rotulo o backend ja
trata como "nao sei" — **a forma certa ja existe dentro do produto**.

**D-5 — O manual do Agent descreve um formato que nao existe.**
`BRAIN/docs/tecnologia/agent-desktop/MANUAL-COMPLETO.md:169-179,207-216` mostra payloads com os
campos `tipo`, `aplicativo`, `titulo`, `duracao`, `idle`, `usuarioWindows`, `maquina`, `uptime` —
**nenhum existe no formato real**. Os tipos de sessao listados em `:199-204`
(`SessionLogon`, `SessionLogoff`, `SessionLock`, `SessionUnlock`, `SessionRemoteConnect`,
`SessionRemoteDisconnect`) **nao sao aceitos por nenhum schema** (`dto:96-102`). A frequencia de
captura de janela em `:166` ("a cada 5 segundos") tambem nao bate: o laco roda a cada 1 s
(`appsettings.json:17`), com debounce de 1500 ms e snapshot de 60 s. Quem escrever API a partir
desse manual escreve API errada.

**D-6 — `tituloJanela`: o Zod aceita 500, a coluna tem 255.**
`dto:221` permite `max(500)`; `EVENTS/src/common/db/schema/eventos.schema.ts:75` declara
`varchar(255)`. O handler nao trunca (`window-activity.handler.ts:148`).
**NAO VERIFICADO** qual e o efeito real em Postgres — um titulo entre 256 e 500 chars deveria
gerar erro `22001` no INSERT, o que derrubaria o lote com 500; nao consegui confirmar sem tocar
no banco, e a regra da tarefa proibe encostar nos servicos. `eventos_reuniao.titulo_reuniao` nao
tem esse problema (`varchar(512)` em `:185` contra `max(512)` em `dto:308`).

**D-7 — `dispositivo_tipo` nao existe em todas as tabelas, ao contrario do que o schema afirma.**
`EVENTS/src/common/db/schema/eventos.schema.ts:22-26` diz que a coluna foi para toda tabela
`eventos_*`. `eventos_atividade` e `eventos_reuniao_snapshot` nao a tem; a migration cobre 7
tabelas (`EVENTS/drizzle/migrations/20260808000000_android_dispositivo_tipo.sql:26,37,48,59,70,81,92`).

---

## 12. O que ficou NAO VERIFICADO

| item | por que |
|---|---|
| Limite de bytes do corpo HTTP | Nao ha nada nos dois repositorios. Um limite de proxy/Fly.io e plausivel e nao pode ser conferido por leitura de codigo. |
| A ocorrencia de 24/08 de "fim anterior ao inicio na troca de servico" | O caminho de codigo esta confirmado (secao 7.3a). A medicao que encontrei no cerebro e de **18/08** (evento 49253, 1 ms). Nao ha registro datado de 24/08 para este sintoma. |
| Efeito real de `tituloJanela` entre 256 e 500 chars | Exige executar contra o banco; a tarefa proibe encostar nos servicos. Ver D-6. |
| Se a UNIQUE `uk_agente_empresa_instalacao` existe de fato em producao | Esta declarada em comentario (`ADMIN/src/common/db/schema/agentes.schema.ts:44-47`) e no baseline de teste (`EVENTS/test/fixtures/baseline.sql:1034`), mas nao ha `.unique()` no Drizzle nem migration nos repositorios lidos. |
| Comportamento do Agent Android | Fora do escopo. Este documento cobre Windows; `manager-srv-agent-android` nao foi lido. |
