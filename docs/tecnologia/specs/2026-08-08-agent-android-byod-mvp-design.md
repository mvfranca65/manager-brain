> **STATUS:** APROVADA (brainstorm) — aguardando disparo de execução por Marcos
> **DATA:** 2026-08-08
> **DONO:** @Tony (arquitetura) + novo dev Android sênior (execução, a contratar)
> **CONSULTORIA:** @Bucky (Agent Windows como referência técnica)
> **REVISORES:** Marcos (aprovador final), @Steve (produto — rodada futura), @Shuri (backend Node — rodada futura), @Thor (contrato Java atual — referência), @Natasha (QA — rodada futura), @Groot (UX docs — rodada futura), @Mike + Raquel (jurídico LGPD — rodada futura)
> **REGRA APLICADA:** mudança no Agent exige spec formal antes de código (`feedback_agent_spec_obrigatoria.md`)

# Agent Mobile Android BYOD — MVP (só APK)

## Sumário

Spec do primeiro Agent Mobile do Manager, para **Android BYOD** (celular pessoal do colaborador, sem MDM), rodando 24/7 em background, coletando eventos comportamentais em **paridade com o Agent Windows atual** e enviando pro `srv-events` via Device JWT.

**Escopo desta spec:** APK Android completo, autocontido, pronto pra ser distribuído. Nada mais.

**Fora do escopo (rodadas futuras que Marcos vai planejar):** portal com botões de download, pipeline server-side de re-empacotamento de APK personalizado, docs pública em `docs.imanagerportal.com/mobile/`, testes cross-device com @Natasha, piloto interno, piloto cliente, alterações no backend Java (que está em migração pra Node — não pode ser mexido).

---

## 1. Contexto e problema

### 1.1 Situação hoje

O Manager tem Agent Desktop (C# .NET 8+, Windows/macOS) rodando na máquina do colaborador coletando janela ativa, ociosidade, sessão, input agregado, reuniões e status. Esses eventos alimentam o pipeline de Score IA que gera relatórios semanais.

Cenário problemático identificado por Marcos: **colaboradores que trabalham 50/50 PC + celular aparecem como "offline" no portal quando estão usando o celular**, mesmo estando ativamente trabalhando. Isso compromete a assertividade do relatório.

### 1.2 ICP do módulo mobile

Perfis que trabalham híbrido PC + celular no cotidiano:
- **Vendedor interno / SDR** — CRM no PC, prospecção via WhatsApp/LinkedIn no celular
- **Atendimento / Suporte** — ticket system no PC, WhatsApp Business e redes sociais no celular
- **Executivos / Gestores** — reuniões e aprovações no celular, decisões no PC

### 1.3 Objetivo desta rodada

Entregar o **APK Android do Agent** pronto pra ser distribuído. Portal, pilotos e integração operacional ficam pra rodadas separadas que Marcos planeja depois.

---

## 2. Escopo e não-escopo

### 2.1 Dentro do escopo desta spec

- Design completo do APK Android nativo (Kotlin)
- Todos os componentes internos, contratos de dados, permissões, fluxo de vinculação, auto-update
- Sistema de logs local + audit events + crash reports (equivalente Windows atual)
- Scripts locais de build + CI/CD do repo Android
- Mapeamento das alterações que o backend precisará no futuro (para consumo do @Shuri quando ela migrar rotas do Agent pra Node)

### 2.2 Fora do escopo (rodadas futuras)

| Item | Rodada futura owner |
|---|---|
| Portal (srv-portal-node) ganha aba do Agent com botões Android | @Shuri (rodada portal) |
| Pipeline server-side de re-empacotamento (`apksigner` + inject `config.json`) | @Shuri (rodada portal) |
| Endpoint `POST /api/agente/logs/upload` | Rodada backend Node |
| Endpoint `GET /api/agente/config` estendido com `enviarLogsAte` e `logLevel` dinâmico | Rodada backend Node |
| Documentação pública em `docs.imanagerportal.com/mobile/` (6 páginas) | Rodada docs |
| Testes cross-device automatizados (E2E Playwright + emulador) | Rodada QA (@Natasha) |
| Testes manuais em matriz de 6 devices físicos | Rodada QA (@Natasha) |
| Piloto interno com 3-5 colaboradores | Rodada piloto |
| Piloto cliente controlado | Rodada piloto |
| Score IA calibrado pra padrão mobile | Rodada srv-ia (@Jarvis) |
| Presença unificada PC+celular no portal | Rodada portal (@Shuri + @Peter) |
| Ícones 🖥/📱 no fed-portal | Rodada fed-portal (@Peter/@Miles) |
| iOS (mesmo BYOD limitado ou corporativo MDM) | Rodada futura pós-Android estabilizado |
| Migração eventual do Agent Windows pra enviar logs pro Loki | Débito compartilhado @Bucky (independente) |
| Config remota de log level (feature comum a Android + Windows) | Débito compartilhado |

### 2.3 Alterações necessárias no backend (só documentar, não fazer)

O backend (`srv-admin` Java + `srv-events` Java) está em processo de migração pra Node pela @Shuri. Regra dura de Marcos: **não mexer em Java durante a migração**. O `srv-admin-node` e `srv-events-node` vão nascer com o **mesmo contrato** do Java atual — usar Java como referência de shape.

Alterações que serão necessárias no backend (Node) para o Agent Android funcionar plenamente em produção comercial:

| Endpoint / tabela | Alteração | Motivo | Prioridade |
|---|---|---|---|
| `agentes` (tabela) | Adicionar coluna `dispositivo_tipo VARCHAR(20)` — enum {WINDOWS, MACOS, ANDROID, IOS}. Default `WINDOWS`. | Distinguir origem do agente | Alta |
| `eventos_*` (todas) | Adicionar `dispositivo_tipo` denormalizado (mesmo default) | Query eficiente por origem sem JOIN | Alta |
| `versoes_agente` (tabela) | Adicionar coluna `sistema_operacional VARCHAR(20)` — enum {WINDOWS, MACOS, ANDROID, IOS} | Distinguir versões por SO | Alta |
| `POST /api/agente/dispositivos/vincular` | Aceitar `dispositivoTipo=ANDROID` no body. Campo `sistemaOperacional` mais rico (versão Android + fabricante + modelo) | Vinculação de agente Android | Alta |
| `POST /api/agent/events` | Aceitar novo tipo de evento `PhoneCallEvent` (CALL_START/CALL_END, sem número/contato). Aceitar `WindowActivityEvent` com `urlDominio` vazio quando browser não expõe. | Novo tipo mobile-específico | Alta |
| `GET /api/agente/atualizacoes/verificar` | Aceitar query `sistemaOperacional=ANDROID`. Retornar `versoes_agente` filtrada por SO. Serve URL do APK. | Auto-update Android | Alta |
| Endpoints existentes `POST /api/agente/auditoria/registrar` e `POST /api/agent/error-report` | Nenhuma alteração de contrato — Android usa exatamente como Windows usa hoje | Auditoria + crash reports mobile | Alta (já compatível) |

**Rodada futura backend também:**

| Endpoint | O que precisa |
|---|---|
| `POST /api/agente/logs/upload` (novo) | Aceitar arquivo compactado (gzip) de logs + range temporal. Guardar em S3 `agent-logs/<tenant>/<agente>/`, retenção 30 dias. |
| `GET /api/agente/config` (existente) | Estender com campos `enviarLogsAte` (timestamp — Agent detecta e sobe logs sob demanda) e `logLevel` (dinâmico — TTL 1h) |

---

## 3. Decisões arquiteturais chave (brainstorm 2026-08-08)

Todas registradas via `AskUserQuestion` com Marcos:

| # | Decisão | Rationale |
|---|---|---|
| D1 | Paridade Windows: mesmos tipos de evento | Coerência de dados no backend, unificação futura no relatório |
| D2 | Android-first, iOS deprioridade | BYOD iOS Apple não permite tracking que precisamos. Fica pra futuro com MDM corporativo |
| D3 | BYOD (celular pessoal do colaborador) | Cliente-alvo tem colaboradores usando celular pessoal |
| D4 | Sideload (fora da Play Store) | Distribuição via link direto do portal. Sem review Google Play. Destrava Accessibility + Usage Stats sem risco de rejeição |
| D5 | Transparente + baixa visibilidade | Coerente com posicionamento ético ("sem vigilância, com dados"). Aceite LGPD claro, notification mínima, tutorial de permissões |
| D6 | APK único por empresa (chave embutida no `config.json`) | Simplifica pipeline de personalização — 1 APK por empresa vs 1 por colaborador |
| D7 | Identificador do colaborador vem em UI mínima na 1ª execução | Única tela do APK; sem UI depois |
| D8 | Ícone auto-remove da gaveta após 1ª execução | `PackageManager.setComponentEnabledSetting` na `PermissionDispatchActivity` após vinculação |
| D9 | Aceite LGPD duplo: institucional (gestor no download) + individual (colaborador na 1ª execução) | Cobre LGPD Art. 7º/8º — consentimento individual do titular |
| D10 | Logs iguais Windows atual: locais + audit events + crash reports. SEM Loki | Simplicidade + evita débito misturado. Loki entra em rodada futura pra Android+Windows juntos |
| D11 | Escape hatch de logs: notification action "Enviar diagnóstico" | Copia últimos logs do storage privado pra `Downloads/ManagerAgent/` público. Colaborador envia manualmente pro gestor. |
| D12 | Auto-update com clique do colaborador (impossível silencioso em BYOD) | Restrição Android. Notification → PackageInstaller → confirma |
| D13 | Package name: `com.trivion.manageragent` (prod) / `.staging` (staging) | Alinhado com razão social Trivion + build variants Gradle |
| D14 | Novo dev Android sênior contratado (não @Bucky aprende) | @Bucky mantém foco no Windows + Marcos e Harvey já decidiram contratação |
| D15 | Backend Java NÃO SERÁ TOCADO — só documentar alterações necessárias | Java em migração pra Node — regra dura. Node nasce com mesmo shape. |
| D16 | Ambientes staging + prod desde o dia 1 (igual desktop) | 2 build variants Gradle, 2 packageNames diferentes (instaláveis lado a lado) |
| D17 | Tentar coleta de `urlDominio` no browser (best-effort, vazio se falhar) | Paridade com Windows. Accessibility lê content-description da URL bar do Chrome. |
| D18 | `PhoneCallEvent` novo tipo pra chamada telefônica (CALL_START/CALL_END, sem número) | Cenário mobile relevante — SDR/atendimento usam celular pra chamar clientes. |
| D19 | `MeetingEvent` continua igual (Zoom/Teams/Meet) via detecção de app foreground | Best-effort — sabemos que app está aberto, não sabemos se está em chamada real. Aceitável. |

---

## 4. Arquitetura macro

### 4.1 Como o Android se encaixa no ecossistema Manager

```
[Colaborador]
    │
    ├── Windows/macOS ────►  ManagerAgent.exe (C# .NET 8) ──┐
    │                                                        │
    └── Android BYOD ─────►  ManagerAgent.apk (Kotlin) ─────┤
                              via config.json embutido:      │
                              chaveAtivacao empresa +        │
                              endpoints prod/staging          │
                                                             ▼
                                                    [manager-srv-events]
                                                    Ingesta batch
                                                    Device JWT (mesmo secret)
                                                    Contrato compatível com Java atual
                                                             │
                                                             ▼
                                                    PostgreSQL Supabase
                                                    eventos_* já existentes
                                                    (+ dispositivo_tipo — rodada futura)
```

### 4.2 Princípios inegociáveis desta arquitetura

1. **Mesmo backend, mesmo Device JWT, mesmo endpoint `/api/agent/events`** — não criamos ingestão paralela
2. **Um colaborador = uma identidade lógica, N dispositivos vinculados** — tabela `agentes` já suporta via `usuario_ref_id` + `maquina_id`
3. **Contrato compatível 100% com Java atual** — APK envia payloads que o Java existente aceita hoje (sem `dispositivoTipo` ou `PhoneCallEvent` novos até backend suportar; ver seção 6.4 fallback)
4. **Score IA fica sem calibração mobile no MVP** — Marcos planeja depois. Eventos mobile chegam no banco mas srv-ia ignora ou usa flag

---

## 5. Componentes internos do APK (15)

### 5.1 Lista completa

| # | Componente | Responsabilidade | Dependência principal |
|---|---|---|---|
| C1 | `ManagerAgentService` (ForegroundService, tipo `dataSync`) | Orquestra a coleta e envio. Notification persistente mínima obrigatória | AndroidX Foreground Service |
| C2 | `ManagerAccessibilityService` | Fonte primária de eventos: window changed + interações (toques, scrolls, digitações — SEM CONTEÚDO) | Android Accessibility API |
| C3 | `UsageStatsCollector` | Fonte de reconciliação: consulta `UsageStatsManager` a cada 5min pra suprir gaps do Accessibility (OEM matando service) | `PACKAGE_USAGE_STATS` permission |
| C4 | `SessionBroadcastReceiver` | Escuta `SCREEN_ON`/`SCREEN_OFF`/`USER_PRESENT`/`BOOT_COMPLETED` → mapeia pra `SessionEvent` LOCK/UNLOCK | BroadcastReceiver Android |
| C5 | `PhoneCallListener` (via `TelephonyManager.PhoneStateListener` ou `TelephonyCallback` Android 12+) | Detecta CALL_START/CALL_END → `PhoneCallEvent`. Sem número/contato. | `READ_PHONE_STATE` permission |
| C6 | `MeetingDetector` | Heurística: app foreground `IN (us.zoom.videomeetings, com.microsoft.teams, com.google.android.apps.meetings)` por >2min = MeetingEvent | AccessibilityService (usa dados de C2) |
| C7 | `UrlDomainExtractor` | Best-effort: lê content-description da URL bar do Chrome via Accessibility node (Chrome/Firefox/Samsung Browser). Fallback: vazio | AccessibilityService (usa dados de C2) |
| C8 | `EventQueue` (Room SQLite local) | Buffer resiliente de eventos pendentes de envio. Trim FIFO máx 10k eventos ou 7 dias | AndroidX Room |
| C9 | `BatchSender` (WorkManager job periódico 5min) | Envia batch acumulado pro `POST /api/agent/events`. Retry exponencial (5min→10min→20min→cap 1h) | WorkManager + OkHttp/Retrofit |
| C10 | `AuthManager` | Vinculação inicial + refresh token + storage seguro (EncryptedSharedPreferences) | AndroidX Security |
| C11 | `PermissionDispatchActivity` | Única Activity — 1 tela (identificador + aceite LGPD) → encadeia intents de permissão → vincula → inicia Service → auto-remove do LAUNCHER via `setComponentEnabledSetting` | Android Intents |
| C12 | `LoggingService` (Timber) | Logs locais estruturados, path privado do app, rotação diária, retenção 7 dias, formato texto estilo Serilog (paridade Windows) | Timber |
| C13 | `AuditReporter` | Fire-and-forget POST `/api/agente/auditoria/registrar` (endpoint existente, mesmo do Windows). Bearer DeviceToken | OkHttp |
| C14 | `CrashReporter` | `Thread.setDefaultUncaughtExceptionHandler` + ANR detector → POST `/api/agent/error-report` (endpoint existente). Throttle 5min por tipo | OkHttp |
| C15 | `UpdateService` | Verifica versão a cada 6h → baixa APK → valida SHA-256 → notification → PackageInstaller flow → audit resultado | WorkManager + PackageInstaller |

### 5.2 Anti-padrão explícito

- **NUNCA** ler `AccessibilityEvent.getText()`, `getContentDescription()` completo ou `getSource().getText()` — LGPD inegociável. Teste unitário `LGPDContentFilterTest` trava sabotagem futura.
- **NUNCA** capturar screenshot da tela — sem paridade com plano Plus do desktop no MVP mobile.
- **NUNCA** capturar áudio, microfone, câmera, GPS — fora do escopo do produto.
- **NUNCA** ler URL path completo — apenas domínio raiz (mesmo padrão Windows).

---

## 6. Mapeamento de eventos Windows → Android

### 6.1 Tipos suportados no MVP

| Tipo (Windows) | Android equivalente | Como coleta | Fidelidade | Contrato JSON |
|---|---|---|---|---|
| `WindowActivityEvent` | App em foreground (`packageName` + `className` da activity) + `urlDominio` best-effort | AccessibilityService `TYPE_WINDOW_STATE_CHANGED` + UrlDomainExtractor | Alta | Mesmo do Windows: `{nomeProcesso, tituloJanela, iniciadoEm, finalizadoEm, statusUsuario, urlDominio}` |
| `IdleEvent` | Sem interação com tela | AccessibilityService (ausência de eventos) + `PowerManager.isInteractive()` | Alta | Mesmo do Windows: `{iniciadoEm, finalizadoEm, statusUsuario}` |
| `HeartbeatEvent` | Idem | WorkManager periódico | Alta | Mesmo do Windows: `{enviadoEm, eventosPendentes}` |
| `SessionEvent` LOCK/UNLOCK | Tela bloqueada/desbloqueada | `ACTION_SCREEN_OFF`/`ACTION_USER_PRESENT` | Alta | Mesmo do Windows: `{tipoEvento, ocorreuEm}` |
| `SessionEvent` LOGIN/LOGOUT | Boot / desligar | `ACTION_BOOT_COMPLETED`/`ACTION_SHUTDOWN` | Média (Android reboota menos) | Idem |
| `ActivitySummaryEvent` | Toques, digitações (virtual), scrolls | AccessibilityService conta eventos por janela de 1min | Média (não distingue teclas físicas de virtuais) | Mesmo Windows: `{teclasPressionadas, cliquesMouseEsq/Dir/Meio, movimentosMouse, scrollsVertical, padrao_digitacao}`. Cliques/movimentos mouse ficam 0 no Android. |
| `MeetingEvent` (Zoom/Teams/Meet) | Detecção por app específico + duração mínima | MeetingDetector (heurística baseada em C2) | Média | Mesmo Windows: `{iniciadoEm, finalizadoEm, aplicativo}`. `aplicativo` = "Zoom Android"/"Teams Android"/"Google Meet". |
| `StatusTransitionEvent` | Idem, calculado localmente | Mesma lógica de thresholds do desktop | Alta | Mesmo Windows: `{statusAnterior, statusNovo, transicaoEm}` |

### 6.2 Novo tipo Android-específico (mapeado pra backend futuro)

| Tipo | Como coleta | Contrato JSON | Backlog backend |
|---|---|---|---|
| `PhoneCallEvent` | `PhoneStateListener.onCallStateChanged()` — CALL_STATE_OFFHOOK (start) e CALL_STATE_IDLE após offhook (end). SEM número. | `{tipoEvento: "CALL_START"\|"CALL_END", ocorreuEm}` | srv-events-node precisa aceitar novo tipo. Até lá, evento fica na fila local do APK ou é enviado como `MeetingEvent` com `aplicativo="PhoneCall"` como fallback compatível. |

### 6.3 Eventos NÃO coletados (fora de escopo)

- **Áudio/vídeo de reunião** — impossível sem gravar (LGPD proíbe)
- **Conteúdo de mensagens WhatsApp/Telegram/etc** — LGPD inegociável
- **Localização GPS** — não faz parte do produto
- **Screenshots** — sem paridade com plano Plus mobile no MVP
- **Wi-Fi conectado / rede** — não faz parte da coleta
- **URL path completa** — só domínio (mesmo padrão Windows)

### 6.4 Premissa de execução (2026-08-08, Marcos)

**Quando iniciarmos o desenvolvimento do APK, `srv-events-node` e `srv-admin-node` estarão 100% em produção com os campos novos.** Ou seja:

- `agentes.dispositivo_tipo`, `eventos_*.dispositivo_tipo` e `versoes_agente.sistema_operacional` já existirão como colunas
- `POST /api/agente/dispositivos/vincular` já aceitará `dispositivoTipo=ANDROID` e SO rico (fabricante + modelo + Android version)
- `POST /api/agent/events` já aceitará `PhoneCallEvent` como tipo próprio e `WindowActivityEvent.urlDominio` vazio
- `GET /api/agente/atualizacoes/verificar?sistemaOperacional=ANDROID` já retornará versões filtradas por SO

**Consequência pro APK:**
- APK envia payloads ricos **desde o dia 1 do desenvolvimento**
- Zero fallback / disfarce como Windows
- Zero condicional "se backend não aceita, faz X"
- Contrato Java atual serve apenas como **referência histórica de shape** — Node herdou e estendeu

**Responsabilidade da @Shuri (não desta spec):** garantir que rotas Node estejam prontas + campos novos deployados em prod antes do dia D do desenvolvimento Android começar. Coordenação de calendário com Marcos.

---

## 7. Onboarding e permissões (fluxo end-to-end)

### 7.1 Fluxo completo (rodada futura: portal envolvido — aqui só documenta)

```
GESTOR (portal — rodada futura):
  1. Vai na aba do Agent do portal (mesma tela do instalador Windows)
  2. Clica "Baixar Android (Prod)" [ou Staging]
  3. Modal com termos institucionais + botão "Aceito e baixar"
  4. Backend registra aceite institucional (tipo TERMOS_AGENT_MOBILE_INSTITUCIONAL, gestor_id, IP, timestamp)
  5. Pipeline de re-empacotamento retorna APK personalizado por empresa
     (chave empresa embutida no config.json)
  6. Gestor copia link ou baixa e envia por WhatsApp/email pro colaborador

COLABORADOR (celular):
  7. Clica link → Chrome baixa APK
  8. Habilita Fontes Desconhecidas (se 1ª vez)
  9. Instala APK
 10. Android auto-abre PermissionDispatchActivity (única Activity, LAUNCHER temporário):
     Tela mínima:
       "Bem-vindo ao ManagerAgent"
       "Empresa: {Nome vindo do config.json embutido}"
       [ Campo: Digite seu CPF ou matrícula ]
       [☐] Li e aceito os Termos de Uso e Política de Privacidade (link abre browser)
       [ Continuar ]
 11. Colaborador digita identificador + aceita termos + Continuar
 12. APK valida identificador contra backend
     • POST /api/agente/v1/colaboradores/validar (chave empresa + identificador)
     • Se OK: registra aceite individual (POST /api/agente/auditoria/registrar tipo TERMOS_ACEITOS_ANDROID)
     • Se erro (identificador não encontrado): mostra "Identificador não encontrado — fale com seu gestor"
 13. Encadeia intents de permissão (sequencial, com callback):
     a. Settings.ACTION_ACCESSIBILITY_SETTINGS → colaborador ativa toggle
     b. Settings.ACTION_USAGE_ACCESS_SETTINGS → colaborador ativa toggle
     c. Runtime permission POST_NOTIFICATIONS (Android 13+)
     d. Runtime permission READ_PHONE_STATE
     e. Se OEM problemático (Xiaomi/Huawei/Oppo/Vivo detectado): abre Settings do OEM (Autostart etc)
 14. Faz POST /api/agente/dispositivos/vincular
     • Header X-Ativacao-Key: {chave embutida}
     • Body: {identificador, instalacaoId, maquinaId, nomeMaquina, versaoAgente, descricaoSo}
     • Recebe deviceToken + refreshToken → guarda em EncryptedSharedPreferences
 15. Inicia ManagerAgentService (ForegroundService)
 16. PackageManager.setComponentEnabledSetting(
       ComponentName(this, PermissionDispatchActivity::class.java),
       COMPONENT_ENABLED_STATE_DISABLED,
       DONT_KILL_APP
     ) → Activity SE DESABILITA como componente. Ícone SOME da gaveta.
 17. Toast: "ManagerAgent ativo ✓" (único elemento visual pós-config)
 18. Activity.finish()

COLABORADOR vê a partir daqui:
  • Notification persistente mínima: "ManagerAgent" (silenciosa, baixa prioridade, sem badge, sem lock screen)
  • Ícone NÃO existe na gaveta
  • App NÃO abre por clique
  • Coleta rodando em background invisível
  • Se quiser desinstalar: Settings > Apps > ManagerAgent > Desinstalar (fluxo normal Android)
```

### 7.2 Permissões declaradas no AndroidManifest.xml

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.WAKE_LOCK" />
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
<uses-permission android:name="android.permission.PACKAGE_USAGE_STATS"
                 tools:ignore="ProtectedPermissions" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
<uses-permission android:name="android.permission.READ_PHONE_STATE" />
<uses-permission android:name="android.permission.REQUEST_INSTALL_PACKAGES" />
<uses-permission android:name="android.permission.QUERY_ALL_PACKAGES"
                 tools:ignore="QueryAllPackagesPermission" />

<application ...>
  <!-- Única Activity, LAUNCHER temporário -->
  <activity android:name=".PermissionDispatchActivity"
            android:exported="true"
            android:excludeFromRecents="true"
            android:noHistory="true"
            android:theme="@android:style/Theme.Material.Light.NoActionBar">
    <intent-filter>
      <action android:name="android.intent.action.MAIN" />
      <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
  </activity>

  <service android:name=".ManagerAgentService"
           android:foregroundServiceType="dataSync"
           android:exported="false" />

  <service android:name=".ManagerAccessibilityService"
           android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE"
           android:exported="true">
    <intent-filter>
      <action android:name="android.accessibilityservice.AccessibilityService" />
    </intent-filter>
    <meta-data android:name="android.accessibilityservice"
               android:resource="@xml/accessibility_config" />
  </service>

  <receiver android:name=".SessionBroadcastReceiver" android:exported="true">
    <intent-filter>
      <action android:name="android.intent.action.BOOT_COMPLETED" />
      <action android:name="android.intent.action.SCREEN_ON" />
      <action android:name="android.intent.action.SCREEN_OFF" />
      <action android:name="android.intent.action.USER_PRESENT" />
    </intent-filter>
  </receiver>
</application>
```

### 7.3 Detecção de OEM problemático

Cada fabricante Android modifica comportamento de background:

| Fabricante | Comportamento | Ação do APK |
|---|---|---|
| Samsung (One UI) | Moderadamente comportado | Solicita "Battery > Não otimizar" via Intent `ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` |
| Xiaomi/Redmi (MIUI) | Agressivo | Abre Settings específico MIUI (Autostart Management) via Intent com component `com.miui.securitycenter/.permission.PermMainActivity` |
| Huawei/Honor (EMUI/HarmonyOS) | Extremo | Abre `com.huawei.systemmanager/.optimize.process.ProtectActivity` |
| Oppo/Realme (ColorOS) | Similar Xiaomi | Abre `com.coloros.safecenter/.startupapp.StartupAppListActivity` |
| Vivo (Funtouch/OriginOS) | Similar Xiaomi | Abre `com.vivo.permissionmanager/.activity.BgStartUpManagerActivity` |
| Motorola / Nokia / Pixel (stock) | Comportado | Nenhuma ação extra |
| OnePlus (OxygenOS) | Moderado | Abre `com.oneplus.security/.chainlaunch.view.ChainLaunchAppListActivity` |

Component `OEMDetector` (utilitário do APK) usa `Build.MANUFACTURER` na 1ª execução e mapeia pra intent correto. Detalhes em código (implementação do dev novo).

---

## 8. Auto-update

BYOD Android **NÃO permite update silencioso** sem MDM/Device Owner. Sistema Android obrigatoriamente pede confirmação do usuário.

### 8.1 Fluxo

```
1. UpdateService (WorkManager job a cada 6h):
   • GET /api/agente/atualizacoes/verificar?sistema=ANDROID&versao=<atual>
     (query "sistema=ANDROID" — rodada futura backend Node; enquanto Java, sem query)
   • Backend responde: {
       atualizacao_disponivel: true,
       versao_nova: "1.2.0",
       url_download: "https://apk.imanager.trivion.com.br/updates/prod/1.2.0.apk",
       checksum_sha256: "abc123...",
       obrigatoria: false
     }

2. Se disponível:
   • Baixa APK pra /data/data/com.trivion.manageragent/files/updates/1.2.0.apk
   • Valida SHA-256 → se falhar, deleta e aborta (tenta em 24h)
   • Cria notification (baixa prioridade): "Atualização disponível — Toque pra instalar"

3. Colaborador clica na notification:
   • Intent ACTION_INSTALL_PACKAGE via PackageInstaller
   • Android mostra popup "Atualizar ManagerAgent?"
   • Colaborador confirma
   • Sistema atualiza
   • ForegroundService reinicia (se OEM permitir)
   • APK novo dispara POST /api/agente/auditoria/registrar (UPDATE_SUCESSO_ANDROID + versão)

4. Se colaborador ignora:
   • Notification persiste até update
   • Após 7 dias sem update: prioridade sobe pra alta
   • Se obrigatoria=true: prioridade alta desde dia 1 + heartbeat envia flag "versao_desatualizada"

5. Se update falha:
   • Deleta APK baixado
   • POST /api/agent/error-report (UPDATE_CRASH + motivo)
   • Retry em 24h
```

### 8.2 Diferenças críticas Windows vs Android (documentar expectativa)

| Aspecto | Windows | Android BYOD |
|---|---|---|
| Atualização silenciosa | Sim, invisível | **Não** — colaborador sempre clica |
| Reinício após update | Automático | Automático se OEM permitir (Samsung sim, Xiaomi às vezes não) |
| Auto-start no boot | Sim | Precisa liberar Auto-start no menu OEM |
| Ícone na gaveta pós-install | N/A | Removido após 1ª execução |
| Notification persistente | Não | **Obrigatória** (API do Android) |
| Colaborador pode matar processo | Difícil | Sim (Force Stop em Settings, Recent apps swipe) |
| Colaborador pode desinstalar | Depende de perms locais | Sim, sempre |

---

## 9. Sistema de logs

### 9.1 Estratégia: paridade com Windows atual

Baseado em investigação do @Bucky (Explore em 2026-08-08): Agent Windows atual **não envia logs pro Loki** — só logs locais (Serilog, `C:\ProgramData\ManagerAgent\logs\`, retenção 7 dias) + audit events + crash reports. Agent Android faz o mesmo padrão.

**Fora de escopo desta rodada:** integração com Loki (débito compartilhado com Windows). Config remota de log level (débito compartilhado).

### 9.2 Componente C12 — LoggingService (Timber)

- Biblioteca: `Timber` (padrão comunidade Kotlin, equivalente ao Serilog)
- Path: `/data/data/com.trivion.manageragent/files/logs/` (app-specific, isolado do usuário — equivalente ao `ProgramData` do Windows)
- Rotação: 1 arquivo por dia (`agent-2026-08-08.log`), retenção 7 dias, máx 5MB por arquivo (rotaciona pra `.log.1` se estourar)
- Níveis: TRACE / DEBUG / INFO / WARN / ERROR
- Debug builds: emite tudo pro LogCat + arquivo
- Release builds: só INFO+ no LogCat, TRACE+ em arquivo
- Formato texto estruturado (paridade Windows):
  ```
  {timestamp:yyyy-MM-dd HH:mm:ss.SSS zzz} [{level:3}] {tag}: {message}{newline}{exception}
  ```

### 9.3 Componente C13 — AuditReporter

- Endpoint: `POST /api/agente/auditoria/registrar` (existente, mesmo do Windows)
- Bearer token DeviceToken
- Fire-and-forget
- Payload:
  ```json
  {
    "evento": "ACCESSIBILITY_DESATIVADA",
    "instalacaoId": "uuid",
    "agenteId": "long",
    "timestampUtc": "2026-08-08T14:23:12Z",
    "dados": { "motivo": "user_disabled_in_settings" }
  }
  ```

**Tipos de audit event Android:**

| Evento | Quando |
|---|---|
| `AGENT_INSTALADO_ANDROID` | 1ª execução pós-install |
| `TERMOS_ACEITOS_ANDROID` | Colaborador aceita termo individual na PermissionDispatchActivity |
| `AGENT_VINCULADO_ANDROID` | Vinculação bem-sucedida |
| `AGENT_VINCULACAO_FALHOU_ANDROID` | Falha (identificador inválido, chave inválida, rede) — motivo detalhado |
| `ACCESSIBILITY_ATIVADA` / `ACCESSIBILITY_DESATIVADA` | Colaborador toggle em Settings |
| `USAGE_STATS_ATIVADA` / `USAGE_STATS_DESATIVADA` | Idem |
| `OEM_PROBLEMATICO_DETECTADO` | Build.MANUFACTURER = Xiaomi/Huawei/etc + status atual (autostart?) |
| `SERVICE_MORTO_E_REINICIADO` | Watchdog reviveu Service morto pelo OEM |
| `UPDATE_INICIADO_ANDROID` / `UPDATE_SUCESSO_ANDROID` / `UPDATE_FALHOU_ANDROID` | Ciclo de auto-update |
| `AGENT_REVOGADO_ANDROID` | Última mensagem antes de parar coleta (backend retornou 403) |
| `AGENT_DESINSTALACAO_INICIADA` | Colaborador tentou desinstalar (se pegarmos evento) |

### 9.4 Componente C14 — CrashReporter

- Endpoint: `POST /api/agent/error-report` (existente, mesmo do Windows)
- Bearer token DeviceToken
- Custom, sem Sentry/Firebase (alinha com posicionamento LGPD)
- Throttle 5min por tipo (mesmo padrão Windows)
- Captura via:
  - `Thread.setDefaultUncaughtExceptionHandler` (uncaught exceptions Kotlin/Java)
  - ANR detector (Android-specific: main thread não responde por 5s → dispatched como ANR)
  - OOM detector (via `Runtime.getRuntime().freeMemory()` monitorado no ForegroundService)
- Payload (mesmo do Windows):
  ```json
  {
    "tipo": "FATAL_CRASH" | "ANR" | "UPDATE_CRASH" | "OUT_OF_MEMORY" | "TOKEN_REFRESH_FAILED" | ...,
    "mensagem": "string",
    "stackTrace": "string (max 2000 chars)",
    "versao": "1.0.0",
    "sistemaOperacional": "Android 13 (API 33)",
    "maquina": "Xiaomi Redmi Note 12",
    "colaboradorId": "long",
    "cnpj": "string (extraído do JWT)"
  }
  ```

### 9.5 Escape hatch — action "Enviar diagnóstico" na notification

Sem endpoint de upload de logs (rodada futura), colaborador consegue enviar logs manualmente:

- Notification persistente do ForegroundService tem action "Enviar diagnóstico"
- Ao clicar: APK copia os últimos 3 arquivos de log (`logs/agent-*.log`) do storage privado pra `/storage/emulated/0/Download/ManagerAgent/logs/` (público)
- Colaborador vai em Downloads, encontra arquivos, envia pro gestor via WhatsApp/email
- Zero backend novo. Zero UI extra.

---

## 10. Distribuição sideload — infra mínima

### 10.1 Bucket S3 (Tigris — já usado pro instalador Windows)

Estrutura:
```
imanager-apks/
├── base/
│   ├── staging/
│   │   ├── manageragent-1.0.0-base.apk
│   │   ├── manageragent-1.0.0-base.apk.sha256
│   │   └── manageragent-latest -> manageragent-1.0.0-base.apk (symlink lógico)
│   └── prod/
│       ├── manageragent-1.0.0-base.apk
│       └── ...
├── personalized/  ← rodada futura (portal + pipeline @Shuri)
│   └── <TOKEN>/manageragent-<empresa>.apk (TTL 10min, auto-limpeza)
└── updates/  ← auto-update
    ├── staging/1.0.1.apk, 1.0.2.apk, ...
    └── prod/1.0.1.apk, 1.0.2.apk, ...
```

### 10.2 Domínio e CDN

- Domínio: `apk.imanager.trivion.com.br` (novo — aponta pro bucket via Cloudflare)
- CDN: Cloudflare em frente (reduz latência de download global)
- Setup: coordena com @Vision quando disparar rodada futura de portal

### 10.3 Keystore de assinatura

- 2 keystores: `staging.jks` e `prod.jks`
- Alias: `manageragent`
- Guardados em Bitwarden (cofre externo — reforça S-02 do MVP2)
- Backup local encriptado no computador do Marcos
- Rotação: anual (setembro-out)

### 10.4 Config embutida (`assets/config.json`)

APK base sai do CI com config genérica:
```json
{
  "ambiente": "staging",
  "endpointAdmin": "https://admin-staging.imanagerportal.com",
  "endpointEvents": "https://api-events-staging.imanagerportal.com",
  "chaveAtivacao": "PLACEHOLDER_PARA_PIPELINE_PERSONALIZAR",
  "nomeEmpresa": "PLACEHOLDER"
}
```

Prod build variant tem endpoints prod. Durante dev, dev usa `chaveAtivacao` de teste (empresa mock em staging).

Rodada futura: pipeline server-side substitui `chaveAtivacao` e `nomeEmpresa` por empresa real + re-assina.

---

## 11. Testing strategy (só unit tests inline no MVP)

### 11.1 Unit tests (Kotlin, JUnit + MockK) — obrigatório neste MVP

Coverage line ≥80%, branch ≥95% (regra Marcos).

| Camada | O que testa | Exemplos de classes |
|---|---|---|
| `EventCollector` | Cada tipo de evento montado corretamente | `WindowActivityEventBuilderTest`, `PhoneCallEventBuilderTest` |
| `AccessibilityFilter` | **Trava sabotagem LGPD** — nunca lê conteúdo | `LGPDContentFilterTest` |
| `EventQueue` (Room) | Insert, dedup, trim FIFO, retry ordering | `EventQueueDaoTest` (Room in-memory) |
| `BatchSender` (Retrofit) | Serialização, retry em falha, refresh em 401, re-vincula em 404 | `BatchSenderTest` (MockWebServer) |
| `AuthManager` | Vinculação, refresh, storage seguro | `AuthManagerTest` (mock EncryptedSharedPreferences) |
| `UpdateService` | Check versão, download, SHA-256, PackageInstaller | `UpdateServiceTest` |
| `OEMDetector` | Detecção + intent correto por fabricante | `OEMDetectorTest` (parametrizado) |
| `AuditReporter` | Payload correto, fire-and-forget | `AuditReporterTest` |
| `CrashReporter` | Throttle, payload | `CrashReporterTest` |
| `LoggingService` | Rotação, retenção, escape hatch | `LoggingServiceTest` |

### 11.2 Fora de escopo do MVP (rodada futura)

- E2E cross-device com @Natasha (Playwright + emulador Android)
- Testes manuais em matriz de 6 devices físicos (Samsung/Xiaomi/Moto/Pixel/OnePlus)
- Testes de longevidade 72h (bateria, memória, CPU)

Dev novo faz smoke test manual em pelo menos 1 emulador (Pixel API 34) + 1 device físico próprio pra sanity check antes de fechar cada bloco.

---

## 12. Scripts de build (pedido do Marcos)

### 12.1 Objetivo

Marcos pediu "script pra gerar apk também ou alguma forma mais fácil, igual temos do Windows". Gradle nativo já entrega isso — Android é bem servido de tooling. Precisamos só documentar.

### 12.2 Comandos locais (documentados no README)

Do repo `manager-srv-agent-android/`:

```bash
# Build APK debug (dev local)
./gradlew assembleDebug
# Saída: app/build/outputs/apk/debug/app-debug.apk

# Build APK staging release (assinado)
./gradlew assembleStagingRelease
# Saída: app/build/outputs/apk/stagingRelease/app-stagingRelease.apk

# Build APK prod release (assinado)
./gradlew assembleProdRelease
# Saída: app/build/outputs/apk/prodRelease/app-prodRelease.apk

# Rodar todos os unit tests
./gradlew test

# Rodar tests com relatório de coverage
./gradlew testDebugUnitTestCoverage
# Relatório: app/build/reports/jacoco/testDebugUnitTestCoverage/html/index.html

# Instalar debug no device conectado (adb)
./gradlew installDebug

# Ver logs em tempo real do device conectado
adb logcat -s ManagerAgent:*

# Copiar logs do device pro computador (pra debug)
adb shell "run-as com.trivion.manageragent.staging cat files/logs/agent-2026-08-08.log" > local-log.txt
```

### 12.3 CI/CD (GitHub Actions — coordena com @Vision)

Workflow `.github/workflows/build-and-publish.yml`:

```yaml
name: Build & Publish APK

on:
  push:
    branches: [main, staging]
    tags: ['v*']
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      - name: Cache Gradle
        uses: gradle/gradle-build-action@v3
      - name: Run unit tests
        run: ./gradlew test
      - name: Check coverage thresholds
        run: ./gradlew jacocoTestCoverageVerification
      - name: Decrypt keystore
        env:
          KEYSTORE_STAGING: ${{ secrets.KEYSTORE_STAGING_BASE64 }}
          KEYSTORE_PROD: ${{ secrets.KEYSTORE_PROD_BASE64 }}
        run: |
          echo "$KEYSTORE_STAGING" | base64 -d > keystore-staging.jks
          echo "$KEYSTORE_PROD" | base64 -d > keystore-prod.jks
      - name: Build staging APK
        if: github.ref == 'refs/heads/staging' || startsWith(github.ref, 'refs/tags/v')
        env:
          KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_STAGING_PASSWORD }}
        run: ./gradlew assembleStagingRelease
      - name: Build prod APK
        if: github.ref == 'refs/heads/main' || startsWith(github.ref, 'refs/tags/v')
        env:
          KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PROD_PASSWORD }}
        run: ./gradlew assembleProdRelease
      - name: Compute SHA-256
        run: sha256sum app/build/outputs/apk/*/app-*Release.apk > checksums.txt
      - name: Upload APK to S3 (Tigris)
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.TIGRIS_ACCESS_KEY }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.TIGRIS_SECRET_KEY }}
          AWS_ENDPOINT_URL_S3: https://fly.storage.tigris.dev
        run: |
          aws s3 cp app/build/outputs/apk/stagingRelease/app-stagingRelease.apk \
            s3://imanager-apks/base/staging/manageragent-${{ github.ref_name }}-base.apk
          # ... e assim por diante pra prod
      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: apks
          path: app/build/outputs/apk/*/app-*.apk
```

### 12.4 Script auxiliar `scripts/build-personalizado.sh` (dev local, sem pipeline server)

Enquanto o pipeline server-side não existir (rodada futura @Shuri), dev pode gerar APK personalizado localmente pra teste:

```bash
#!/bin/bash
# scripts/build-personalizado.sh
# Uso: ./scripts/build-personalizado.sh <ambiente> <chaveAtivacao> <nomeEmpresa>

AMBIENTE=$1
CHAVE_ATIVACAO=$2
NOME_EMPRESA=$3

# Substitui placeholders no config.json de assets
sed -i.bak \
  -e "s|\"chaveAtivacao\": \".*\"|\"chaveAtivacao\": \"$CHAVE_ATIVACAO\"|" \
  -e "s|\"nomeEmpresa\": \".*\"|\"nomeEmpresa\": \"$NOME_EMPRESA\"|" \
  app/src/${AMBIENTE}Release/assets/config.json

# Build assinado
./gradlew assemble${AMBIENTE^}Release

# Restaura config.json original
mv app/src/${AMBIENTE}Release/assets/config.json.bak app/src/${AMBIENTE}Release/assets/config.json

echo "APK gerado em app/build/outputs/apk/${AMBIENTE}Release/"
```

---

## 13. Riscos vivos

| # | Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|---|
| R1 | Fragmentação OEM (Xiaomi/Huawei matam Service) | Alta | Alto (dado some) | OEMDetector detecta cedo. Doc no MVP futura ensina liberação manual. Testes em Xiaomi Redmi cedo (não deixar pro fim do B3). |
| R2 | Colaborador não conclui as 4 permissões | Média | Colaborador fica "sem monitoramento" mas não sabe | Notification "Configuração pendente" persiste até completar. Backend detecta ausência de heartbeat pós-vinculação. |
| R3 | Colaborador desinstala | Média | Perda de coleta silenciosa | Backend detecta silêncio > 24h → alerta gestor (rodada futura portal) |
| R4 | Colaborador desativa Accessibility em Settings | Média | Perda de eventos primários | `AccessibilityService.onServiceDisconnected()` envia audit event `ACCESSIBILITY_DESATIVADA` como último ato. Backend alerta gestor. |
| R5 | Update precisa clique manual | Certeza | Fricção UX | Aceito por Marcos. Notification clara + intervalo respeitoso. |
| R6 | Keystore de assinatura vaza | Baixíssima | Catastrófico (todos APKs comprometidos) | Bitwarden (S-02 MVP2 vira crítico) + rotação anual + acesso restrito |
| R7 | Config estática limita testes multi-empresa | Certeza (por design) | Aceito no MVP | Pipeline server-side entra em rodada futura da @Shuri |
| R8 | Android nova versão quebra Accessibility/Usage Stats API | Baixa | Requer patch rápido | Testes em Android 10/12/13/14 (emulador) desde B3. CI valida em múltiplas API levels. |
| R9 | Backend Java atual não aceita `dispositivoTipo=ANDROID` | Certeza (não aceita hoje) | APK precisa se disfarçar como Windows temporariamente | Documentado como estratégia de compatibilidade (seção 6.4). Migra pra `ANDROID` real quando Node estiver pronto. |
| R10 | Consent LGPD colaborador vira bloqueante contratual | Alta | Sem colaborador aceitando = sem venda mobile | Aceite duplo já mapeado (institucional gestor + individual colaborador). @Steve + Raquel validam em rodada futura antes de venda comercial. |
| R11 | Bateria drenando no celular | Média | Colaborador reclama | Batch 5min (não real-time), Accessibility eficiente, WakeLock mínimo, benchmarks manuais 72h antes de release |
| R12 | Ícone auto-desabilitado quebra em Android futuro | Baixa | Ícone volta a aparecer | `setComponentEnabledSetting` é API estável desde Android 2.2. Testes em versões novas cobrem. |

---

## 14. Cronograma

### 14.1 Escopo desta rodada — só APK

| Bloco | Escopo | Estimativa | Owner |
|---|---|---|---|
| B2 | Setup projeto (repo, Gradle, build variants, keystore, CI/CD, scripts) | 1 sem | Dev novo + @Vision |
| B3 | Coleta core (Service, Accessibility, Broadcasts, Room, eventos: WindowActivity, Idle, Session, Heartbeat) | 4 sem | Dev novo |
| B4 | Coleta complementar (UsageStats, ActivitySummary, MeetingEvent, PhoneCallEvent, StatusTransition, urlDomain) | 3 sem | Dev novo |
| B5 | Auth + integração backend (vincular, refresh, storage seguro, batch sender, interceptors) | 2 sem | Dev novo |
| B6 | PermissionDispatchActivity + LGPD UI mínima + encadeamento intents + OEM detection + auto-desabilita LAUNCHER | 2 sem | Dev novo |
| B7 | Sistema de logs (Timber + AuditReporter + CrashReporter + ANR detector + escape hatch) | 1 sem | Dev novo |
| B8 | Auto-update (UpdateService + WorkManager + PackageInstaller + notification action) | 2 sem | Dev novo |
| B_test | Unit tests inline (≥80% linha, ≥95% branch) — paralelo aos blocos | ~2 sem esforço distribuído | Dev novo |

**Total desenvolvimento APK: ~15-16 semanas = ~3,5-4 meses** com 1 dev Android sênior dedicado full-time.

### 14.2 Caminho crítico

```
B2 (setup, 1 sem)
  └─► B3 (coleta core, 4 sem)
        └─► B4 (coleta compl, 3 sem)
              └─► B5 (auth, 2 sem)
                    └─► B6 (perm+LGPD, 2 sem)
                          └─► B7 (logs, 1 sem)
                                └─► B8 (update, 2 sem)
                                      │
                                      ▼
                                APK PRONTO
                                (rodadas futuras: portal, pipeline, docs, QA, piloto)

B_test (~2 sem esforço) roda em paralelo aos blocos
```

### 14.3 Fora do escopo desta spec (rodadas futuras)

- Contratação do dev Android sênior (Marcos + Harvey — em paralelo)
- Rodada portal (@Shuri: aba do Agent + pipeline re-empacotamento)
- Rodada docs (docs.imanagerportal.com/mobile/)
- Rodada QA (@Natasha: E2E + matriz devices físicos)
- Rodada pilotos (interno + cliente)

---

## 15. Deps mínimas pra dev começar

Quando Marcos disparar "vamos começar", isso é o que precisa estar em mãos:

1. **Repo git criado** — pref. `Backend/manager-srv-agent-android/` no monorepo OU repo separado (padrão dos outros)
2. **Keystore de assinatura** — 2 keystores (staging + prod) guardados no Bitwarden. Se não existir, dev cria primeiro dia.
3. **Empresa de teste em staging** — 1 empresa mock ativa no srv-admin staging com `chaveAtivacao` conhecida
4. **Colaborador de teste em staging** — 1 usuário mock naquela empresa (identificador conhecido)
5. **Endpoint staging acessível** — srv-admin + srv-events staging respondendo (pra dev integrar em B5)
6. **Dev Android alocado** — contratação/onboard concluídos
7. **Acesso ao GitHub org** — novo repo + permissão de push
8. **Bucket S3 (Tigris) criado** — `imanager-apks/` com policy adequada
9. **DNS `apk.imanager.trivion.com.br`** — aponta pro bucket via Cloudflare (rodada futura pode postergar)

Coordenação: quando disparar, @Vision configura #7-9 em paralelo ao onboard do dev.

---

## 16. Referências

### Documentação interna Manager
- `_shared/produto.md` — visão geral do produto Manager/iManager
- `_shared/arquitetura.md` — stack e serviços
- `_shared/banco-dados.md` — schema (agentes, eventos_*, versoes_agente, aceite_termos, audit_events)
- `_shared/lgpd-operacional.md` — limites de coleta inegociáveis
- `_shared/padroes.md` — convenções de código + regra de teste por branch
- `tecnologia/services/srv-events/README.md` — contrato de ingestão (referência do shape que o APK precisa enviar)
- `tecnologia/services/srv-admin/README.md` — contrato de vinculação + auto-update + auditoria
- `tecnologia/autenticacao/autenticacao-sistema.md` — 3 domínios JWT (Domínio 1 = Agent ↔ Backend)
- `tecnologia/autenticacao/jwt-contract.md` — contrato JWT byte-por-byte
- `tecnologia/agent-desktop/` — Agent Desktop atual (referência técnica pra logs, auditoria, crash reports)

### Investigação do @Bucky (2026-08-08)
- Serilog config: `Agente/manager-srv-agent/src/ManagerAgent.Tray/Program.cs:38-51` e `src/ManagerAgent.Service/Program.cs:53-67`
- Audit endpoint: `Agente/manager-srv-agent/src/ManagerAgent.Shared/Audit/HttpAgentAuditReporter.cs:26`
- Error report endpoint: `Agente/manager-srv-agent/src/ManagerAgent.Tray/Services/ErrorReportService.cs:122`

### Regras aplicáveis
- `feedback_agent_spec_obrigatoria.md` — muda no Agent = spec obrigatória antes de código
- `feedback_enum_novo_backend_antes.md` — nunca introduzir enum no Agent antes do backend suportar (aplicado em seção 6.4)
- `feedback_teste_branch_coverage.md` — cobertura branch ≥95%
- `feedback_nao_commitar_execucao_subagent.md` — subagents não commitam; Marcos commita
- Regra "Java não muda durante migração Node" (Marcos 2026-07-21)

### Rodadas futuras que dependem desta spec (a Marcos disparar)
- Rodada backend Node (@Shuri): rotas Agent migradas pra Node com contrato compatível + campos novos (`dispositivoTipo`, `PhoneCallEvent`)
- Rodada portal (@Shuri + @Peter): aba Agent com botões Android + pipeline server-side de re-empacotamento
- Rodada docs (dev novo + @Groot + @Steve + jurídico): `docs.imanagerportal.com/mobile/` com 6 páginas
- Rodada QA (@Natasha): E2E cross-device + matriz de devices físicos
- Rodada pilotos (Marcos + @Steve): interno depois cliente controlado

---

## 17. Roadmap iOS (sinalização, não escopo)

iOS BYOD tem restrições fortes da Apple que praticamente inviabilizam paridade Windows sem MDM corporativo (Managed Device via Apple Business Manager):

- Screen Time API (`FamilyControls`) — restrito a apps de controle parental, review Apple duro
- Background modes — pra tarefas específicas (audio, VoIP, location), não app usage
- App usage tracking BYOD — praticamente inviável
- MDM iOS BYOD — supervised mode requer enrollment via Apple Business Manager

**Quando entrar iOS (rodada muito futura):**
- Só com MDM corporativo (empresa dá celular) — não BYOD puro
- Ou modo "executivo" com celular corporativo aprovando MDM voluntário
- Escopo reduzido: só presença + timing (não conseguirá app-por-app como Android)

Comunicar isso no material comercial: **"Android only na v1; iOS em roadmap com escopo diferente sob condições especiais."**
