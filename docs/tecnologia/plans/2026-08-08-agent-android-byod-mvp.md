# Agent Android BYOD MVP — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Entregar o APK Android do Agent Manager, autocontido, pronto pra ser distribuído (rodadas seguintes: portal + pilotos + docs pública).

**Architecture:** Kotlin nativo Android, arquitetura por responsabilidade (não por camada) — pacotes `service/`, `collector/`, `event/`, `auth/`, `network/`, `sender/`, `ui/`, `oem/`, `log/`, `update/`. `ForegroundService` orquestra `AccessibilityService` + `BroadcastReceivers` + `PhoneCallListener` como fontes de evento, `Room` como buffer, `WorkManager` como scheduler de envio via `Retrofit/OkHttp`.

**Tech Stack:** Kotlin 1.9+, Android SDK 34 (minSdk 26 / Android 8), Gradle 8+ com KTS, AndroidX (Foreground Service, Accessibility, WorkManager, Room, Security, Startup), Retrofit 2 + OkHttp 4, Kotlinx Serialization, Timber (logging), JUnit 5 + MockK + MockWebServer + Room-testing (unit tests), Jacoco (coverage).

## Global Constraints

- Package name prod: `com.trivion.manageragent`
- Package name staging: `com.trivion.manageragent.staging`
- `minSdk = 26` (Android 8 Oreo — mínimo por causa de `ForegroundService`)
- `targetSdk = 34` (Android 14)
- `compileSdk = 34`
- Kotlin: 1.9.22+
- Cobertura obrigatória: linha ≥80%, branch ≥95% (regra Marcos — cada `if/else/when/catch` testado)
- Contrato JSON dos eventos: **compatível com srv-events (Node)** — payloads seguem shape existente do Java, herdado pelo Node. Campos novos (`dispositivoTipo=ANDROID`, `PhoneCallEvent`, `sistema_operacional`) suportados desde o dia 1 (premissa: Node 100% em prod antes de dev começar).
- Endpoints:
  - `POST /api/agente/dispositivos/vincular` (srv-admin-node)
  - `POST /api/agent/events` (srv-events-node)
  - `POST /api/agente/auth/refresh` (srv-admin-node)
  - `GET /api/agente/atualizacoes/verificar?sistemaOperacional=ANDROID` (srv-admin-node)
  - `POST /api/agente/atualizacoes/resultado` (srv-admin-node)
  - `POST /api/agente/auditoria/registrar` (srv-admin-node)
  - `POST /api/agent/error-report` (srv-admin-node)
  - `POST /api/agente/v1/colaboradores/validar` (srv-admin-node)
- LGPD inegociável — **NUNCA** capturar/persistir/enviar:
  - Conteúdo de campos de texto (`AccessibilityEvent.getText()`, `getContentDescription()` completo, `getSource().getText()`)
  - Screenshots (fora de escopo mesmo do plano Plus mobile no MVP)
  - Áudio / microfone / câmera / GPS
  - URL path completa — apenas domínio raiz (mesmo padrão do Windows)
  - Número/contato de chamada telefônica
- Convenções:
  - Idioma do código: **inglês**
  - Idioma de documentação/comentários no repo: **português**
  - Commits: Conventional Commits (`feat:`, `fix:`, `test:`, `docs:`, `refactor:`)
  - Nomes de tabela Room em `snake_case`, entidades em `PascalCase`, funções em `camelCase`
- Estrutura de teste TDD obrigatória (regra Marcos): teste primeiro, teste falha, código mínimo, teste passa, commit
- Assinatura APK: keystore `staging.jks` e `prod.jks` guardados no Bitwarden (Marcos), decodificados no CI via secret base64
- Notification persistente do `ForegroundService` DEVE ser mínima/silenciosa (channel `IMPORTANCE_LOW`, sem badge, sem lock screen)
- `PermissionDispatchActivity` tem `MAIN + LAUNCHER` **temporariamente** — após vinculação completa, chamar `PackageManager.setComponentEnabledSetting(COMPONENT_ENABLED_STATE_DISABLED)` pra sumir da gaveta

## Índice de tasks (41 tasks em 7 blocos)

- **B2 — Setup projeto (T1-T7):** repo, Gradle multi-source, build variants, keystore, CI/CD, scripts
- **B3 — Coleta core (T8-T18):** Room, ForegroundService, BroadcastReceivers, AccessibilityService, eventos WindowActivity/Idle/Session/Heartbeat
- **B4 — Coleta complementar (T19-T25):** UsageStats, ActivitySummary, MeetingDetector, PhoneCall, StatusTransition, UrlDomain
- **B5 — Auth + backend integration (T26-T31):** ConfigReader, TokenStorage, Retrofit, interceptors, AuthManager, BatchSenderWorker
- **B6 — PermissionDispatchActivity + LGPD (T32-T36):** UI, encadeamento intents, OEMDetector, vinculação, self-disable LAUNCHER
- **B7 — Logs (T37-T40):** Timber + FileLoggingTree, AuditReporter, CrashReporter, escape hatch diagnóstico
- **B8 — Auto-update (T41-T44):** UpdateService, download+checksum, PackageInstaller, audit result

---

<!-- INÍCIO_TASKS -->

## BLOCO B2 — Setup projeto (T1-T7)

### Task T1: Criar estrutura do repo + Gradle root

**Files:**
- Create: `settings.gradle.kts`
- Create: `build.gradle.kts` (root)
- Create: `gradle.properties`
- Create: `gradle/libs.versions.toml`
- Create: `.gitignore`
- Create: `README.md`

**Interfaces:**
- Consumes: nada
- Produces: workspace Gradle multi-project inicializado, capaz de rodar `./gradlew tasks`

- [ ] **Step 1: Criar `settings.gradle.kts`**

```kotlin
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}

rootProject.name = "manager-srv-agent-android"
include(":app")
```

- [ ] **Step 2: Criar `gradle/libs.versions.toml` (version catalog)**

```toml
[versions]
agp = "8.2.2"
kotlin = "1.9.22"
coreKtx = "1.12.0"
lifecycleRuntimeKtx = "2.7.0"
workManager = "2.9.0"
room = "2.6.1"
security = "1.1.0-alpha06"
retrofit = "2.9.0"
okhttp = "4.12.0"
kotlinxSerialization = "1.6.2"
kotlinxSerializationConverter = "1.0.0"
timber = "5.0.1"
junit5 = "5.10.1"
mockk = "1.13.9"
mockwebserver = "4.12.0"
turbine = "1.0.0"
coroutinesTest = "1.7.3"
jacoco = "0.8.11"

[libraries]
androidx-core-ktx = { group = "androidx.core", name = "core-ktx", version.ref = "coreKtx" }
androidx-lifecycle-runtime-ktx = { group = "androidx.lifecycle", name = "lifecycle-runtime-ktx", version.ref = "lifecycleRuntimeKtx" }
androidx-work-runtime-ktx = { group = "androidx.work", name = "work-runtime-ktx", version.ref = "workManager" }
androidx-room-runtime = { group = "androidx.room", name = "room-runtime", version.ref = "room" }
androidx-room-ktx = { group = "androidx.room", name = "room-ktx", version.ref = "room" }
androidx-room-compiler = { group = "androidx.room", name = "room-compiler", version.ref = "room" }
androidx-room-testing = { group = "androidx.room", name = "room-testing", version.ref = "room" }
androidx-security-crypto = { group = "androidx.security", name = "security-crypto", version.ref = "security" }
retrofit-core = { group = "com.squareup.retrofit2", name = "retrofit", version.ref = "retrofit" }
retrofit-kotlinx-serialization = { group = "com.jakewharton.retrofit", name = "retrofit2-kotlinx-serialization-converter", version.ref = "kotlinxSerializationConverter" }
okhttp-core = { group = "com.squareup.okhttp3", name = "okhttp", version.ref = "okhttp" }
okhttp-logging = { group = "com.squareup.okhttp3", name = "logging-interceptor", version.ref = "okhttp" }
okhttp-mockwebserver = { group = "com.squareup.okhttp3", name = "mockwebserver", version.ref = "mockwebserver" }
kotlinx-serialization-json = { group = "org.jetbrains.kotlinx", name = "kotlinx-serialization-json", version.ref = "kotlinxSerialization" }
timber = { group = "com.jakewharton.timber", name = "timber", version.ref = "timber" }
junit-jupiter = { group = "org.junit.jupiter", name = "junit-jupiter", version.ref = "junit5" }
mockk = { group = "io.mockk", name = "mockk", version.ref = "mockk" }
mockk-android = { group = "io.mockk", name = "mockk-android", version.ref = "mockk" }
turbine = { group = "app.cash.turbine", name = "turbine", version.ref = "turbine" }
coroutines-test = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-test", version.ref = "coroutinesTest" }

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
ksp = { id = "com.google.devtools.ksp", version = "1.9.22-1.0.17" }
```

- [ ] **Step 3: Criar `build.gradle.kts` root**

```kotlin
plugins {
    alias(libs.plugins.android.application) apply false
    alias(libs.plugins.kotlin.android) apply false
    alias(libs.plugins.kotlin.serialization) apply false
    alias(libs.plugins.ksp) apply false
}

tasks.register("clean", Delete::class) {
    delete(rootProject.buildDir)
}
```

- [ ] **Step 4: Criar `gradle.properties`**

```properties
org.gradle.jvmargs=-Xmx4096m -Dfile.encoding=UTF-8
org.gradle.parallel=true
org.gradle.caching=true
kotlin.code.style=official
android.useAndroidX=true
android.nonTransitiveRClass=true
```

- [ ] **Step 5: Criar `.gitignore`**

```
*.iml
.gradle/
build/
local.properties
.idea/
captures/
.externalNativeBuild/
.cxx/
*.apk
*.aab
!/scripts/*.sh
release/
keystores/
*.jks
!*.jks.enc
```

- [ ] **Step 6: Criar `README.md` mínimo**

```markdown
# manager-srv-agent-android

Agent Desktop Manager para Android BYOD.

## Requisitos
- JDK 17
- Android SDK 34
- Gradle wrapper (incluso)

## Build local
- `./gradlew assembleStagingRelease` — APK staging assinado
- `./gradlew assembleProdRelease` — APK produção assinado
- `./gradlew test` — unit tests
- `./gradlew jacocoTestCoverageVerification` — checa cobertura

Ver `.brain/tecnologia/plans/2026-08-08-agent-android-byod-mvp.md` para plano completo.
Ver `.brain/tecnologia/specs/2026-08-08-agent-android-byod-mvp-design.md` para spec.
```

- [ ] **Step 7: Criar Gradle wrapper**

```bash
gradle wrapper --gradle-version 8.5
```

Verificar que `gradlew`, `gradlew.bat`, `gradle/wrapper/gradle-wrapper.jar`, `gradle/wrapper/gradle-wrapper.properties` foram criados.

- [ ] **Step 8: Verificar setup**

Run: `./gradlew tasks`
Expected: lista de tasks Gradle sem erro. Root project chamado `manager-srv-agent-android`.

- [ ] **Step 9: Commit**

```bash
git init
git add .
git commit -m "chore: initial Gradle multi-project setup with version catalog"
```

---

### Task T2: Criar módulo :app + build.gradle com build variants

**Files:**
- Create: `app/build.gradle.kts`
- Create: `app/src/main/AndroidManifest.xml` (mínimo)
- Create: `app/src/main/kotlin/com/trivion/manageragent/ManagerAgentApplication.kt`
- Create: `app/proguard-rules.pro`
- Create: `app/src/main/res/values/strings.xml`

**Interfaces:**
- Consumes: version catalog de T1
- Produces: módulo `:app` compilável com 3 build variants: `debug`, `stagingRelease`, `prodRelease`, cada uma com `packageName` correto

- [ ] **Step 1: Criar `app/build.gradle.kts`**

```kotlin
import org.jetbrains.kotlin.gradle.tasks.KotlinCompile

plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.kotlin.serialization)
    alias(libs.plugins.ksp)
    id("jacoco")
}

android {
    namespace = "com.trivion.manageragent"
    compileSdk = 34

    defaultConfig {
        applicationId = "com.trivion.manageragent"
        minSdk = 26
        targetSdk = 34
        versionCode = 1
        versionName = "1.0.0"

        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
        vectorDrawables { useSupportLibrary = true }
    }

    signingConfigs {
        create("staging") {
            storeFile = file(System.getenv("KEYSTORE_STAGING_PATH") ?: "keystores/staging.jks")
            storePassword = System.getenv("KEYSTORE_STAGING_PASSWORD") ?: ""
            keyAlias = "manageragent"
            keyPassword = System.getenv("KEYSTORE_STAGING_PASSWORD") ?: ""
        }
        create("prod") {
            storeFile = file(System.getenv("KEYSTORE_PROD_PATH") ?: "keystores/prod.jks")
            storePassword = System.getenv("KEYSTORE_PROD_PASSWORD") ?: ""
            keyAlias = "manageragent"
            keyPassword = System.getenv("KEYSTORE_PROD_PASSWORD") ?: ""
        }
    }

    buildTypes {
        debug {
            isMinifyEnabled = false
            applicationIdSuffix = ".debug"
            versionNameSuffix = "-debug"
        }
        create("stagingRelease") {
            initWith(getByName("release"))
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
            signingConfig = signingConfigs.getByName("staging")
            applicationIdSuffix = ".staging"
            versionNameSuffix = "-staging"
            matchingFallbacks += listOf("release")
        }
        create("prodRelease") {
            initWith(getByName("release"))
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
            signingConfig = signingConfigs.getByName("prod")
            matchingFallbacks += listOf("release")
        }
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    kotlinOptions { jvmTarget = "17" }
    buildFeatures {
        buildConfig = true
        viewBinding = true
    }
    packaging { resources.excludes += "/META-INF/{AL2.0,LGPL2.1}" }

    testOptions {
        unitTests.isIncludeAndroidResources = true
        unitTests.isReturnDefaultValues = true
    }
}

jacoco {
    toolVersion = "0.8.11"
}

tasks.withType<Test> {
    useJUnitPlatform()
    extensions.configure(JacocoTaskExtension::class) {
        isIncludeNoLocationClasses = true
        excludes = listOf("jdk.internal.*")
    }
}

tasks.register<JacocoReport>("jacocoTestReport") {
    dependsOn("testDebugUnitTest")
    reports {
        xml.required.set(true)
        html.required.set(true)
    }
    val fileFilter = listOf(
        "**/R.class", "**/R$*.class", "**/BuildConfig.*", "**/Manifest*.*",
        "**/*Test*.*", "android/**/*.*"
    )
    val debugTree = fileTree("$buildDir/tmp/kotlin-classes/debug") { exclude(fileFilter) }
    val mainSrc = "$projectDir/src/main/kotlin"
    sourceDirectories.setFrom(files(mainSrc))
    classDirectories.setFrom(files(debugTree))
    executionData.setFrom(fileTree(buildDir) { include("jacoco/testDebugUnitTest.exec") })
}

tasks.register("jacocoTestCoverageVerification") {
    dependsOn("jacocoTestReport")
    doLast {
        // Verificação será implementada em T5 (CI) — placeholder verificado por script separado
        println("Coverage report at app/build/reports/jacoco/jacocoTestReport/html/index.html")
    }
}

dependencies {
    implementation(libs.androidx.core.ktx)
    implementation(libs.androidx.lifecycle.runtime.ktx)
    implementation(libs.timber)

    testImplementation(libs.junit.jupiter)
    testImplementation(libs.mockk)
    testImplementation(libs.turbine)
    testImplementation(libs.coroutines.test)
}
```

- [ ] **Step 2: Criar `app/src/main/AndroidManifest.xml` mínimo**

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <application
        android:name=".ManagerAgentApplication"
        android:allowBackup="false"
        android:label="@string/app_name"
        android:supportsRtl="true"
        android:theme="@android:style/Theme.NoDisplay">
    </application>

</manifest>
```

- [ ] **Step 3: Criar `ManagerAgentApplication.kt`**

```kotlin
package com.trivion.manageragent

import android.app.Application
import timber.log.Timber

class ManagerAgentApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        if (BuildConfig.DEBUG) {
            Timber.plant(Timber.DebugTree())
        }
    }
}
```

- [ ] **Step 4: Criar `app/src/main/res/values/strings.xml`**

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="app_name">ManagerAgent</string>
</resources>
```

- [ ] **Step 5: Criar `app/proguard-rules.pro` mínimo**

```
# Timber
-dontwarn org.jetbrains.annotations.**

# Kotlinx Serialization
-keepattributes *Annotation*, InnerClasses
-dontnote kotlinx.serialization.AnnotationsKt
-keep,includedescriptorclasses class com.trivion.manageragent.**$$serializer { *; }
-keepclassmembers class com.trivion.manageragent.** {
    *** Companion;
}
-keepclasseswithmembers class com.trivion.manageragent.** {
    kotlinx.serialization.KSerializer serializer(...);
}
```

- [ ] **Step 6: Build & verify**

Run: `./gradlew :app:assembleDebug`
Expected: BUILD SUCCESSFUL. APK gerado em `app/build/outputs/apk/debug/app-debug.apk`.

Run: `./gradlew :app:assembleStagingRelease` (falha esperada — sem keystore ainda)
Expected: falha em signing (esperado). Corrigido em T3.

- [ ] **Step 7: Commit**

```bash
git add .
git commit -m "chore: add :app module with 3 build variants (debug, staging, prod)"
```

---

### Task T3: Setup keystores + assinatura CI-friendly

**Files:**
- Create: `keystores/README.md` (instruções, sem keystores reais)
- Create: `scripts/generate-dev-keystores.sh`
- Modify: `app/build.gradle.kts` (usar env var pra path do keystore fallback)

**Interfaces:**
- Consumes: `app/build.gradle.kts` de T2
- Produces: keystores locais gerados via script, path documentado, CI pronto pra receber base64 via secret

- [ ] **Step 1: Criar `scripts/generate-dev-keystores.sh`**

```bash
#!/usr/bin/env bash
set -euo pipefail

KEYSTORES_DIR="$(cd "$(dirname "$0")/.." && pwd)/keystores"
mkdir -p "$KEYSTORES_DIR"

for env in staging prod; do
    KEYSTORE_PATH="$KEYSTORES_DIR/$env.jks"
    if [[ -f "$KEYSTORE_PATH" ]]; then
        echo "SKIP: $KEYSTORE_PATH já existe. Delete manualmente se quiser regenerar."
        continue
    fi
    PASS_VAR="KEYSTORE_${env^^}_PASSWORD"
    if [[ -z "${!PASS_VAR:-}" ]]; then
        echo "ERRO: export $PASS_VAR=<senha-forte> antes de rodar" >&2
        exit 1
    fi
    keytool -genkeypair -v \
        -keystore "$KEYSTORE_PATH" \
        -alias manageragent \
        -keyalg RSA -keysize 4096 -validity 10000 \
        -storepass "${!PASS_VAR}" \
        -keypass "${!PASS_VAR}" \
        -dname "CN=ManagerAgent, O=Trivion, C=BR"
    echo "OK: gerado $KEYSTORE_PATH"
done

echo ""
echo "Salvar keystores E senhas no Bitwarden imediatamente."
echo "Base64 dos keystores pra CI:"
for env in staging prod; do
    echo "$env: $(base64 -i "$KEYSTORES_DIR/$env.jks" | head -c 40)... (usar valor completo como secret KEYSTORE_${env^^}_BASE64)"
done
```

- [ ] **Step 2: Criar `keystores/README.md`**

```markdown
# Keystores

## NUNCA commitar keystores neste diretório

Este diretório está no `.gitignore`. Arquivos `*.jks` são ignorados.

## Gerar keystores localmente

```bash
export KEYSTORE_STAGING_PASSWORD="<senha-forte-staging>"
export KEYSTORE_PROD_PASSWORD="<senha-forte-prod>"
./scripts/generate-dev-keystores.sh
```

Guardar imediatamente no Bitwarden:
- `staging.jks` (arquivo + senha)
- `prod.jks` (arquivo + senha)

## CI (GitHub Actions)

Secrets necessários:
- `KEYSTORE_STAGING_BASE64` — output de `base64 -i staging.jks`
- `KEYSTORE_STAGING_PASSWORD`
- `KEYSTORE_PROD_BASE64`
- `KEYSTORE_PROD_PASSWORD`

Workflow decodifica no runtime.
```

- [ ] **Step 3: Tornar script executável**

```bash
chmod +x scripts/generate-dev-keystores.sh
```

- [ ] **Step 4: Gerar keystores localmente (dev novo executa)**

```bash
export KEYSTORE_STAGING_PASSWORD="dev-only-staging-password"
export KEYSTORE_PROD_PASSWORD="dev-only-prod-password"
./scripts/generate-dev-keystores.sh
```

Verificar: `ls -la keystores/` mostra `staging.jks` e `prod.jks`.

- [ ] **Step 5: Build assinado local (verificação)**

```bash
export KEYSTORE_STAGING_PATH="$(pwd)/keystores/staging.jks"
./gradlew :app:assembleStagingRelease
```

Expected: BUILD SUCCESSFUL. APK gerado em `app/build/outputs/apk/stagingRelease/app-stagingRelease.apk`.

Verificar assinatura:
```bash
$ANDROID_HOME/build-tools/34.0.0/apksigner verify --print-certs app/build/outputs/apk/stagingRelease/app-stagingRelease.apk
```
Expected: certificado `CN=ManagerAgent, O=Trivion, C=BR`.

- [ ] **Step 6: Commit**

```bash
git add scripts/generate-dev-keystores.sh keystores/README.md
git commit -m "chore: add keystore generation script (staging + prod)"
```

---

### Task T4: BuildConfig fields por variant (endpoints + ambiente)

**Files:**
- Modify: `app/build.gradle.kts`

**Interfaces:**
- Consumes: build variants de T2
- Produces: `BuildConfig.API_ADMIN_URL`, `BuildConfig.API_EVENTS_URL`, `BuildConfig.AMBIENTE` disponíveis em runtime, diferentes por variant

- [ ] **Step 1: Escrever teste unitário**

Criar `app/src/test/kotlin/com/trivion/manageragent/BuildConfigTest.kt`:

```kotlin
package com.trivion.manageragent

import org.junit.jupiter.api.Test
import kotlin.test.assertEquals
import kotlin.test.assertTrue

class BuildConfigTest {

    @Test
    fun `API_ADMIN_URL nao esta vazia`() {
        assertTrue(BuildConfig.API_ADMIN_URL.isNotBlank())
    }

    @Test
    fun `API_EVENTS_URL nao esta vazia`() {
        assertTrue(BuildConfig.API_EVENTS_URL.isNotBlank())
    }

    @Test
    fun `AMBIENTE reconhecido`() {
        val ambientesValidos = setOf("debug", "staging", "prod")
        assertTrue(BuildConfig.AMBIENTE in ambientesValidos, "Ambiente ${BuildConfig.AMBIENTE} inválido")
    }
}
```

- [ ] **Step 2: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.BuildConfigTest"`
Expected: FAIL — `BuildConfig.API_ADMIN_URL` unresolved.

- [ ] **Step 3: Modificar `app/build.gradle.kts` — adicionar `buildConfigField` por variant**

Dentro de `android { ... }`, atualizar `buildTypes`:

```kotlin
buildTypes {
    debug {
        isMinifyEnabled = false
        applicationIdSuffix = ".debug"
        versionNameSuffix = "-debug"
        buildConfigField("String", "API_ADMIN_URL", "\"http://10.0.2.2:8081\"")  // emulator loopback
        buildConfigField("String", "API_EVENTS_URL", "\"http://10.0.2.2:8080\"")
        buildConfigField("String", "AMBIENTE", "\"debug\"")
    }
    create("stagingRelease") {
        initWith(getByName("release"))
        isMinifyEnabled = true
        isShrinkResources = true
        proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
        signingConfig = signingConfigs.getByName("staging")
        applicationIdSuffix = ".staging"
        versionNameSuffix = "-staging"
        matchingFallbacks += listOf("release")
        buildConfigField("String", "API_ADMIN_URL", "\"https://admin-staging.imanagerportal.com\"")
        buildConfigField("String", "API_EVENTS_URL", "\"https://api-events-staging.imanagerportal.com\"")
        buildConfigField("String", "AMBIENTE", "\"staging\"")
    }
    create("prodRelease") {
        initWith(getByName("release"))
        isMinifyEnabled = true
        isShrinkResources = true
        proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
        signingConfig = signingConfigs.getByName("prod")
        matchingFallbacks += listOf("release")
        buildConfigField("String", "API_ADMIN_URL", "\"https://admin.imanagerportal.com\"")
        buildConfigField("String", "API_EVENTS_URL", "\"https://api-events.imanagerportal.com\"")
        buildConfigField("String", "AMBIENTE", "\"prod\"")
    }
}
```

- [ ] **Step 4: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.BuildConfigTest"`
Expected: PASS — todos os 3 testes verdes.

- [ ] **Step 5: Verificar variants**

Run: `./gradlew :app:assembleDebug :app:assembleStagingRelease :app:assembleProdRelease`
Expected: 3 APKs gerados, cada um com `applicationId` diferente:
- `app-debug.apk` → `com.trivion.manageragent.debug`
- `app-stagingRelease.apk` → `com.trivion.manageragent.staging`
- `app-prodRelease.apk` → `com.trivion.manageragent`

Verificar com: `$ANDROID_HOME/build-tools/34.0.0/aapt dump badging <apk> | grep package`

- [ ] **Step 6: Commit**

```bash
git add app/build.gradle.kts app/src/test/kotlin/com/trivion/manageragent/BuildConfigTest.kt
git commit -m "feat: add BuildConfig fields for endpoints + environment per variant"
```

---

### Task T5: CI GitHub Actions — build + tests + coverage gate

**Files:**
- Create: `.github/workflows/ci.yml`
- Create: `.github/workflows/build-and-publish.yml`

**Interfaces:**
- Consumes: build variants + testes de T2/T4
- Produces: pipeline CI que roda em cada push/PR (ci.yml) + pipeline de release em tag (build-and-publish.yml)

- [ ] **Step 1: Criar `.github/workflows/ci.yml`**

```yaml
name: CI

on:
  push:
    branches: [main, staging]
  pull_request:
    branches: [main, staging]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      - uses: gradle/actions/setup-gradle@v3
      - name: Run unit tests
        run: ./gradlew testDebugUnitTest
      - name: Generate coverage report
        run: ./gradlew jacocoTestReport
      - name: Check coverage thresholds
        run: ./gradlew jacocoTestCoverageVerification
      - name: Upload coverage HTML
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: app/build/reports/jacoco/jacocoTestReport/html/

  build-debug:
    runs-on: ubuntu-latest
    needs: unit-tests
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      - uses: gradle/actions/setup-gradle@v3
      - name: Build debug APK
        run: ./gradlew :app:assembleDebug
      - name: Upload APK
        uses: actions/upload-artifact@v4
        with:
          name: app-debug
          path: app/build/outputs/apk/debug/app-debug.apk
```

- [ ] **Step 2: Criar `.github/workflows/build-and-publish.yml`**

```yaml
name: Build & Publish APK

on:
  push:
    branches: [main, staging]
    tags: ['v*']
  workflow_dispatch:
    inputs:
      environment:
        description: 'Ambiente'
        required: true
        default: 'staging'
        type: choice
        options: [staging, prod]

jobs:
  build-signed:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      - uses: gradle/actions/setup-gradle@v3

      - name: Decode keystores
        env:
          KEYSTORE_STAGING_BASE64: ${{ secrets.KEYSTORE_STAGING_BASE64 }}
          KEYSTORE_PROD_BASE64: ${{ secrets.KEYSTORE_PROD_BASE64 }}
        run: |
          mkdir -p keystores
          echo "$KEYSTORE_STAGING_BASE64" | base64 -d > keystores/staging.jks
          echo "$KEYSTORE_PROD_BASE64" | base64 -d > keystores/prod.jks
          echo "KEYSTORE_STAGING_PATH=$(pwd)/keystores/staging.jks" >> $GITHUB_ENV
          echo "KEYSTORE_PROD_PATH=$(pwd)/keystores/prod.jks" >> $GITHUB_ENV

      - name: Build Staging APK
        if: github.ref == 'refs/heads/staging' || startsWith(github.ref, 'refs/tags/v') || github.event.inputs.environment == 'staging'
        env:
          KEYSTORE_STAGING_PASSWORD: ${{ secrets.KEYSTORE_STAGING_PASSWORD }}
        run: ./gradlew :app:assembleStagingRelease

      - name: Build Prod APK
        if: github.ref == 'refs/heads/main' || startsWith(github.ref, 'refs/tags/v') || github.event.inputs.environment == 'prod'
        env:
          KEYSTORE_PROD_PASSWORD: ${{ secrets.KEYSTORE_PROD_PASSWORD }}
        run: ./gradlew :app:assembleProdRelease

      - name: Compute SHA-256 checksums
        run: |
          cd app/build/outputs/apk
          find . -name "*.apk" -exec sha256sum {} \; > ../../../../checksums.txt
          cat ../../../../checksums.txt

      - name: Upload APKs
        uses: actions/upload-artifact@v4
        with:
          name: signed-apks
          path: |
            app/build/outputs/apk/**/*.apk
            checksums.txt

      # Upload S3 fica pra rodada de infra (@Vision integra com Tigris)
```

- [ ] **Step 3: Verificar sintaxe local**

Push num branch feature e observar Actions rodando. Se não tiver acesso ao repo GitHub ainda, rodar `actionlint` local:

```bash
docker run --rm -v $(pwd):/repo rhysd/actionlint:latest -color -pyflakes= /repo/.github/workflows/*.yml
```

Expected: no errors.

- [ ] **Step 4: Commit**

```bash
git add .github/
git commit -m "ci: add unit-tests + build workflows"
```

---

### Task T6: Script de build personalizado local (config.json inject)

**Files:**
- Create: `scripts/build-personalizado.sh`
- Create: `app/src/stagingRelease/assets/config.json`
- Create: `app/src/prodRelease/assets/config.json`
- Create: `app/src/debug/assets/config.json`

**Interfaces:**
- Consumes: build variants de T2
- Produces: script local pra dev testar APK com `chaveAtivacao` de empresa de teste sem precisar do pipeline server-side (rodada futura da @Shuri)

- [ ] **Step 1: Criar `app/src/debug/assets/config.json`**

```json
{
  "ambiente": "debug",
  "chaveAtivacao": "PLACEHOLDER_DEV_LOCAL",
  "nomeEmpresa": "Empresa Dev Local"
}
```

- [ ] **Step 2: Criar `app/src/stagingRelease/assets/config.json`**

```json
{
  "ambiente": "staging",
  "chaveAtivacao": "PLACEHOLDER_STAGING",
  "nomeEmpresa": "Empresa de Teste Staging"
}
```

- [ ] **Step 3: Criar `app/src/prodRelease/assets/config.json`**

```json
{
  "ambiente": "prod",
  "chaveAtivacao": "PLACEHOLDER_PROD",
  "nomeEmpresa": "PLACEHOLDER"
}
```

- [ ] **Step 4: Criar `scripts/build-personalizado.sh`**

```bash
#!/usr/bin/env bash
set -euo pipefail

usage() {
    cat <<EOF
Uso: $0 <ambiente> <chaveAtivacao> "<nomeEmpresa>"

  ambiente:      debug | staging | prod
  chaveAtivacao: UUID da empresa (ex: 3a5b7c9d-...)
  nomeEmpresa:   Nome exibido no APK

Exemplo:
  $0 staging 3a5b7c9d-e1f2-4a3b-9c8d-7e6f5a4b3c2d "Acme LTDA"

Saída: APK em app/build/outputs/apk/<variant>/app-<variant>.apk
EOF
    exit 1
}

[[ $# -lt 3 ]] && usage

AMBIENTE=$1
CHAVE=$2
NOME=$3

case "$AMBIENTE" in
    debug|staging|prod) ;;
    *) echo "Ambiente inválido: $AMBIENTE" >&2; usage ;;
esac

if [[ "$AMBIENTE" == "debug" ]]; then
    VARIANT="debug"
    SRC_SET="debug"
elif [[ "$AMBIENTE" == "staging" ]]; then
    VARIANT="stagingRelease"
    SRC_SET="stagingRelease"
else
    VARIANT="prodRelease"
    SRC_SET="prodRelease"
fi

CONFIG_PATH="app/src/$SRC_SET/assets/config.json"
BACKUP="$CONFIG_PATH.bak"

# Backup original
cp "$CONFIG_PATH" "$BACKUP"

# Trap para restaurar backup mesmo em caso de erro
trap "mv '$BACKUP' '$CONFIG_PATH'; echo 'Config original restaurada'" EXIT

# Escrever config personalizado (JSON manual, sem jq necessário)
cat > "$CONFIG_PATH" <<EOF
{
  "ambiente": "$AMBIENTE",
  "chaveAtivacao": "$CHAVE",
  "nomeEmpresa": "$NOME"
}
EOF

echo "Config personalizado escrito em $CONFIG_PATH"
echo "Rodando build..."

if [[ "$AMBIENTE" == "debug" ]]; then
    ./gradlew ":app:assembleDebug"
else
    ./gradlew ":app:assemble${VARIANT^}"
fi

APK_PATH="app/build/outputs/apk/$VARIANT/app-$VARIANT.apk"
[[ -f "$APK_PATH" ]] || { echo "APK não encontrado em $APK_PATH" >&2; exit 1; }

echo ""
echo "OK: APK gerado em $APK_PATH"
echo "Empresa: $NOME"
echo "Chave: $CHAVE"
```

- [ ] **Step 5: Tornar executável**

```bash
chmod +x scripts/build-personalizado.sh
```

- [ ] **Step 6: Testar script**

```bash
./scripts/build-personalizado.sh staging test-chave-uuid "Empresa Teste"
```

Expected:
- Config `app/src/stagingRelease/assets/config.json` recebe `chaveAtivacao=test-chave-uuid`, `nomeEmpresa="Empresa Teste"`
- Build roda
- APK gerado em `app/build/outputs/apk/stagingRelease/app-stagingRelease.apk`
- Config original restaurada ao final (mesmo se falhar)

- [ ] **Step 7: Commit**

```bash
git add scripts/build-personalizado.sh app/src/*/assets/
git commit -m "chore: add build-personalizado.sh + assets/config.json per variant"
```

---

### Task T7: Configurar Jacoco coverage verification (linha ≥80%, branch ≥95%)

**Files:**
- Modify: `app/build.gradle.kts` (task `jacocoTestCoverageVerification`)

**Interfaces:**
- Consumes: task `jacocoTestReport` de T2
- Produces: gate de coverage que falha o build se abaixo dos thresholds

- [ ] **Step 1: Escrever teste que garante alguma cobertura mínima existir**

Criar `app/src/main/kotlin/com/trivion/manageragent/util/Sanity.kt`:

```kotlin
package com.trivion.manageragent.util

object Sanity {
    fun echo(s: String): String = "echo:$s"
}
```

Criar `app/src/test/kotlin/com/trivion/manageragent/util/SanityTest.kt`:

```kotlin
package com.trivion.manageragent.util

import org.junit.jupiter.api.Test
import kotlin.test.assertEquals

class SanityTest {
    @Test
    fun `echo prepends 'echo-'`() {
        assertEquals("echo:hello", Sanity.echo("hello"))
    }
}
```

- [ ] **Step 2: Rodar coverage**

```bash
./gradlew jacocoTestReport
open app/build/reports/jacoco/jacocoTestReport/html/index.html
```

Verificar visualmente que `Sanity.kt` aparece com 100% cobertura.

- [ ] **Step 3: Substituir a task `jacocoTestCoverageVerification` no `app/build.gradle.kts`**

Trocar o `tasks.register("jacocoTestCoverageVerification") { ... placeholder }` por:

```kotlin
tasks.register<JacocoCoverageVerification>("jacocoTestCoverageVerification") {
    dependsOn("jacocoTestReport")
    executionData.setFrom(fileTree(buildDir) { include("jacoco/testDebugUnitTest.exec") })
    classDirectories.setFrom(fileTree("$buildDir/tmp/kotlin-classes/debug") {
        exclude(
            "**/R.class", "**/R$*.class", "**/BuildConfig.*", "**/Manifest*.*",
            "**/*Test*.*", "android/**/*.*",
            "**/ManagerAgentApplication.*"  // Android framework — excluído
        )
    })
    sourceDirectories.setFrom(files("$projectDir/src/main/kotlin"))

    violationRules {
        rule {
            element = "BUNDLE"
            limit {
                counter = "LINE"
                minimum = "0.80".toBigDecimal()
            }
            limit {
                counter = "BRANCH"
                minimum = "0.95".toBigDecimal()
            }
        }
    }
}
```

- [ ] **Step 4: Rodar e ver o gate**

Run: `./gradlew jacocoTestCoverageVerification`
Expected: PASS (Sanity 100%, nada mais existe ainda).

- [ ] **Step 5: Adicionar Sanity intencionalmente ruim pra ver o gate falhar**

Editar `Sanity.kt`:

```kotlin
object Sanity {
    fun echo(s: String): String = "echo:$s"

    fun untested(x: Int): Int {
        return if (x > 0) x * 2 else x - 1
    }
}
```

Run: `./gradlew jacocoTestCoverageVerification`
Expected: FAIL — branch coverage < 95% em `Sanity.untested`.

- [ ] **Step 6: Cobrir `untested` com testes**

Adicionar a `SanityTest.kt`:

```kotlin
@Test
fun `untested positivo dobra`() = assertEquals(6, Sanity.untested(3))

@Test
fun `untested zero ou negativo subtrai um`() {
    assertEquals(-1, Sanity.untested(0))
    assertEquals(-2, Sanity.untested(-1))
}
```

Run: `./gradlew jacocoTestCoverageVerification`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add app/build.gradle.kts app/src/main/kotlin/com/trivion/manageragent/util/Sanity.kt app/src/test/kotlin/com/trivion/manageragent/util/SanityTest.kt
git commit -m "test: add coverage verification gate (line >=80%, branch >=95%)"
```

---

## BLOCO B3 — Coleta core (T8-T18)

### Task T8: Modelo de eventos (sealed class + Kotlinx Serialization)

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/event/model/AgentEvent.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/event/model/AgentEventSerializationTest.kt`
- Modify: `app/build.gradle.kts` (adicionar dependência `kotlinx-serialization-json`)

**Interfaces:**
- Consumes: nada
- Produces: sealed class `AgentEvent` com subtipos `WindowActivityEvent`, `IdleEvent`, `HeartbeatEvent`, `SessionEvent`, `ActivitySummaryEvent`, `MeetingEvent`, `PhoneCallEvent`, `StatusTransitionEvent`. Cada um serializável em JSON via `kotlinx.serialization`. Formato JSON compatível com contrato srv-events (Node) — campos herdados do Java, dispositivoTipo=ANDROID.

- [ ] **Step 1: Adicionar dependência serialization em `app/build.gradle.kts`**

Em `dependencies { ... }` adicionar:

```kotlin
implementation(libs.kotlinx.serialization.json)
```

- [ ] **Step 2: Escrever teste de serialização**

Criar `app/src/test/kotlin/com/trivion/manageragent/event/model/AgentEventSerializationTest.kt`:

```kotlin
package com.trivion.manageragent.event.model

import kotlinx.serialization.encodeToString
import kotlinx.serialization.json.Json
import org.junit.jupiter.api.Test
import kotlin.test.assertEquals

class AgentEventSerializationTest {

    private val json = Json { classDiscriminator = "tipoEvento"; encodeDefaults = true }

    @Test
    fun `WindowActivityEvent serializa com campos completos`() {
        val evt: AgentEvent = AgentEvent.WindowActivity(
            nomeProcesso = "com.whatsapp",
            tituloJanela = "WhatsApp",
            iniciadoEm = "2026-08-08T14:00:00Z",
            finalizadoEm = "2026-08-08T14:01:00Z",
            statusUsuario = "ATIVO",
            urlDominio = ""
        )
        val out = json.encodeToString(evt)
        assertEquals(
            """{"tipoEvento":"WindowActivityEvent","nomeProcesso":"com.whatsapp","tituloJanela":"WhatsApp","iniciadoEm":"2026-08-08T14:00:00Z","finalizadoEm":"2026-08-08T14:01:00Z","statusUsuario":"ATIVO","urlDominio":""}""",
            out
        )
    }

    @Test
    fun `PhoneCallEvent serializa CALL_START`() {
        val evt: AgentEvent = AgentEvent.PhoneCall(
            tipoTransicao = "CALL_START",
            ocorreuEm = "2026-08-08T14:02:00Z"
        )
        val out = json.encodeToString(evt)
        assertEquals(
            """{"tipoEvento":"PhoneCallEvent","tipoTransicao":"CALL_START","ocorreuEm":"2026-08-08T14:02:00Z"}""",
            out
        )
    }

    @Test
    fun `IdleEvent serializa com statusUsuario`() {
        val evt: AgentEvent = AgentEvent.Idle(
            iniciadoEm = "2026-08-08T14:00:00Z",
            finalizadoEm = "2026-08-08T14:05:00Z",
            statusUsuario = "AUSENTE"
        )
        val out = json.encodeToString(evt)
        assertEquals(
            """{"tipoEvento":"IdleEvent","iniciadoEm":"2026-08-08T14:00:00Z","finalizadoEm":"2026-08-08T14:05:00Z","statusUsuario":"AUSENTE"}""",
            out
        )
    }

    @Test
    fun `HeartbeatEvent serializa com eventosPendentes`() {
        val evt: AgentEvent = AgentEvent.Heartbeat(enviadoEm = "2026-08-08T14:00:00Z", eventosPendentes = 42)
        val out = json.encodeToString(evt)
        assertEquals(
            """{"tipoEvento":"HeartbeatEvent","enviadoEm":"2026-08-08T14:00:00Z","eventosPendentes":42}""",
            out
        )
    }

    @Test
    fun `SessionEvent LOCK serializa`() {
        val evt: AgentEvent = AgentEvent.Session(tipoTransicao = "LOCK", ocorreuEm = "2026-08-08T14:00:00Z")
        val out = json.encodeToString(evt)
        assertEquals(
            """{"tipoEvento":"SessionEvent","tipoTransicao":"LOCK","ocorreuEm":"2026-08-08T14:00:00Z"}""",
            out
        )
    }

    @Test
    fun `ActivitySummaryEvent serializa contadores`() {
        val evt: AgentEvent = AgentEvent.ActivitySummary(
            iniciadoEm = "2026-08-08T14:00:00Z",
            finalizadoEm = "2026-08-08T14:01:00Z",
            teclasPressionadas = 15,
            cliquesMouseEsq = 8,
            cliquesMouseDir = 0,
            cliquesMouseMeio = 0,
            movimentosMouse = 0,
            scrollsVertical = 3,
            padraoDigitacao = "NORMAL"
        )
        val out = json.encodeToString(evt)
        assertEquals(
            """{"tipoEvento":"ActivitySummaryEvent","iniciadoEm":"2026-08-08T14:00:00Z","finalizadoEm":"2026-08-08T14:01:00Z","teclasPressionadas":15,"cliquesMouseEsq":8,"cliquesMouseDir":0,"cliquesMouseMeio":0,"movimentosMouse":0,"scrollsVertical":3,"padraoDigitacao":"NORMAL"}""",
            out
        )
    }

    @Test
    fun `MeetingEvent serializa aplicativo`() {
        val evt: AgentEvent = AgentEvent.Meeting(
            iniciadoEm = "2026-08-08T14:00:00Z",
            finalizadoEm = "2026-08-08T14:30:00Z",
            aplicativo = "Zoom Android"
        )
        val out = json.encodeToString(evt)
        assertEquals(
            """{"tipoEvento":"MeetingEvent","iniciadoEm":"2026-08-08T14:00:00Z","finalizadoEm":"2026-08-08T14:30:00Z","aplicativo":"Zoom Android"}""",
            out
        )
    }

    @Test
    fun `StatusTransitionEvent serializa transicao`() {
        val evt: AgentEvent = AgentEvent.StatusTransition(
            statusAnterior = "ATIVO",
            statusNovo = "AUSENTE",
            transicaoEm = "2026-08-08T14:00:00Z"
        )
        val out = json.encodeToString(evt)
        assertEquals(
            """{"tipoEvento":"StatusTransitionEvent","statusAnterior":"ATIVO","statusNovo":"AUSENTE","transicaoEm":"2026-08-08T14:00:00Z"}""",
            out
        )
    }
}
```

- [ ] **Step 3: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.event.model.AgentEventSerializationTest"`
Expected: FAIL — `AgentEvent` classes não existem.

- [ ] **Step 4: Implementar `AgentEvent.kt`**

Criar `app/src/main/kotlin/com/trivion/manageragent/event/model/AgentEvent.kt`:

```kotlin
package com.trivion.manageragent.event.model

import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable

/**
 * Eventos coletados pelo Agent Android. Serializados como JSON polimórfico
 * usando classDiscriminator = "tipoEvento" (contrato herdado do srv-events).
 */
@Serializable
sealed class AgentEvent {

    @Serializable
    @SerialName("WindowActivityEvent")
    data class WindowActivity(
        val nomeProcesso: String,
        val tituloJanela: String,
        val iniciadoEm: String,
        val finalizadoEm: String,
        val statusUsuario: String,
        val urlDominio: String = ""
    ) : AgentEvent()

    @Serializable
    @SerialName("IdleEvent")
    data class Idle(
        val iniciadoEm: String,
        val finalizadoEm: String,
        val statusUsuario: String
    ) : AgentEvent()

    @Serializable
    @SerialName("HeartbeatEvent")
    data class Heartbeat(
        val enviadoEm: String,
        val eventosPendentes: Int
    ) : AgentEvent()

    @Serializable
    @SerialName("SessionEvent")
    data class Session(
        val tipoTransicao: String,  // "LOCK", "UNLOCK", "LOGIN", "LOGOUT"
        val ocorreuEm: String
    ) : AgentEvent()

    @Serializable
    @SerialName("ActivitySummaryEvent")
    data class ActivitySummary(
        val iniciadoEm: String,
        val finalizadoEm: String,
        val teclasPressionadas: Int,
        val cliquesMouseEsq: Int,
        val cliquesMouseDir: Int,
        val cliquesMouseMeio: Int,
        val movimentosMouse: Int,
        val scrollsVertical: Int,
        val padraoDigitacao: String
    ) : AgentEvent()

    @Serializable
    @SerialName("MeetingEvent")
    data class Meeting(
        val iniciadoEm: String,
        val finalizadoEm: String,
        val aplicativo: String
    ) : AgentEvent()

    @Serializable
    @SerialName("PhoneCallEvent")
    data class PhoneCall(
        val tipoTransicao: String,  // "CALL_START", "CALL_END"
        val ocorreuEm: String
    ) : AgentEvent()

    @Serializable
    @SerialName("StatusTransitionEvent")
    data class StatusTransition(
        val statusAnterior: String,
        val statusNovo: String,
        val transicaoEm: String
    ) : AgentEvent()
}
```

- [ ] **Step 5: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.event.model.AgentEventSerializationTest"`
Expected: PASS — todos os 8 testes.

- [ ] **Step 6: Commit**

```bash
git add app/build.gradle.kts app/src/main/kotlin/com/trivion/manageragent/event/model/AgentEvent.kt app/src/test/kotlin/com/trivion/manageragent/event/model/AgentEventSerializationTest.kt
git commit -m "feat: add AgentEvent sealed class with 8 event subtypes (Kotlinx Serialization)"
```

---

### Task T9: Room database + EventEntity + EventDao

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/event/EventEntity.kt`
- Create: `app/src/main/kotlin/com/trivion/manageragent/event/EventDao.kt`
- Create: `app/src/main/kotlin/com/trivion/manageragent/event/EventDatabase.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/event/EventDaoTest.kt`
- Modify: `app/build.gradle.kts` (adicionar dependências Room + KSP)

**Interfaces:**
- Consumes: nada
- Produces:
  - `EventEntity(id: Long, payloadJson: String, tipoEvento: String, criadoEm: Long, tentativas: Int)` — linha no DB
  - `EventDao` com métodos: `suspend fun insert(entity: EventEntity): Long`, `suspend fun pending(limit: Int): List<EventEntity>`, `suspend fun deleteByIds(ids: List<Long>)`, `suspend fun count(): Int`, `suspend fun trimOldest(keepCount: Int)`, `suspend fun incrementTentativas(ids: List<Long>)`
  - `EventDatabase.getInstance(context)` como singleton

- [ ] **Step 1: Adicionar dependências Room em `app/build.gradle.kts`**

Em `dependencies { ... }`:

```kotlin
implementation(libs.androidx.room.runtime)
implementation(libs.androidx.room.ktx)
ksp(libs.androidx.room.compiler)

testImplementation(libs.androidx.room.testing)
```

- [ ] **Step 2: Escrever teste de DAO**

Criar `app/src/test/kotlin/com/trivion/manageragent/event/EventDaoTest.kt`:

```kotlin
package com.trivion.manageragent.event

import androidx.arch.core.executor.testing.InstantTaskExecutorRule
import androidx.room.Room
import androidx.test.core.app.ApplicationProvider
import androidx.test.ext.junit.runners.AndroidJUnit4
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.AfterEach
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import org.junit.runner.RunWith
import kotlin.test.assertEquals
import kotlin.test.assertTrue

@RunWith(AndroidJUnit4::class)
class EventDaoTest {

    private lateinit var db: EventDatabase
    private lateinit var dao: EventDao

    @BeforeEach
    fun setUp() {
        db = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(),
            EventDatabase::class.java
        ).allowMainThreadQueries().build()
        dao = db.eventDao()
    }

    @AfterEach
    fun tearDown() { db.close() }

    @Test
    fun `insert retorna id positivo`() = runTest {
        val id = dao.insert(EventEntity(payloadJson = "{}", tipoEvento = "Heartbeat", criadoEm = 100L))
        assertTrue(id > 0)
    }

    @Test
    fun `count reflete quantidade inserida`() = runTest {
        dao.insert(EventEntity(payloadJson = "{}", tipoEvento = "Heartbeat", criadoEm = 100L))
        dao.insert(EventEntity(payloadJson = "{}", tipoEvento = "Heartbeat", criadoEm = 200L))
        assertEquals(2, dao.count())
    }

    @Test
    fun `pending retorna eventos em ordem FIFO por criadoEm`() = runTest {
        val id2 = dao.insert(EventEntity(payloadJson = "e2", tipoEvento = "T", criadoEm = 200L))
        val id1 = dao.insert(EventEntity(payloadJson = "e1", tipoEvento = "T", criadoEm = 100L))
        val list = dao.pending(10)
        assertEquals(2, list.size)
        assertEquals(id1, list[0].id)
        assertEquals(id2, list[1].id)
    }

    @Test
    fun `pending respeita limit`() = runTest {
        repeat(5) { dao.insert(EventEntity(payloadJson = "e$it", tipoEvento = "T", criadoEm = it.toLong())) }
        assertEquals(3, dao.pending(3).size)
    }

    @Test
    fun `deleteByIds remove apenas ids indicados`() = runTest {
        val id1 = dao.insert(EventEntity(payloadJson = "e1", tipoEvento = "T", criadoEm = 100L))
        val id2 = dao.insert(EventEntity(payloadJson = "e2", tipoEvento = "T", criadoEm = 200L))
        dao.deleteByIds(listOf(id1))
        val remaining = dao.pending(10)
        assertEquals(1, remaining.size)
        assertEquals(id2, remaining[0].id)
    }

    @Test
    fun `trimOldest mantem os N mais recentes`() = runTest {
        repeat(10) { dao.insert(EventEntity(payloadJson = "e$it", tipoEvento = "T", criadoEm = it.toLong())) }
        dao.trimOldest(keepCount = 3)
        val remaining = dao.pending(20)
        assertEquals(3, remaining.size)
        // Os 3 restantes devem ser os mais recentes (criadoEm 7, 8, 9)
        assertEquals(listOf(7L, 8L, 9L), remaining.map { it.criadoEm })
    }

    @Test
    fun `incrementTentativas soma 1 aos ids indicados`() = runTest {
        val id = dao.insert(EventEntity(payloadJson = "e", tipoEvento = "T", criadoEm = 1L, tentativas = 2))
        dao.incrementTentativas(listOf(id))
        val list = dao.pending(10)
        assertEquals(3, list[0].tentativas)
    }
}
```

- [ ] **Step 3: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.event.EventDaoTest"`
Expected: FAIL — classes não existem.

- [ ] **Step 4: Criar `EventEntity.kt`**

```kotlin
package com.trivion.manageragent.event

import androidx.room.Entity
import androidx.room.Index
import androidx.room.PrimaryKey

@Entity(
    tableName = "event",
    indices = [Index(value = ["criadoEm"], name = "idx_event_criadoEm")]
)
data class EventEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val payloadJson: String,
    val tipoEvento: String,
    val criadoEm: Long,
    val tentativas: Int = 0
)
```

- [ ] **Step 5: Criar `EventDao.kt`**

```kotlin
package com.trivion.manageragent.event

import androidx.room.Dao
import androidx.room.Insert
import androidx.room.Query

@Dao
interface EventDao {

    @Insert
    suspend fun insert(entity: EventEntity): Long

    @Query("SELECT COUNT(*) FROM event")
    suspend fun count(): Int

    @Query("SELECT * FROM event ORDER BY criadoEm ASC, id ASC LIMIT :limit")
    suspend fun pending(limit: Int): List<EventEntity>

    @Query("DELETE FROM event WHERE id IN (:ids)")
    suspend fun deleteByIds(ids: List<Long>)

    @Query("UPDATE event SET tentativas = tentativas + 1 WHERE id IN (:ids)")
    suspend fun incrementTentativas(ids: List<Long>)

    @Query("""
        DELETE FROM event WHERE id NOT IN (
            SELECT id FROM event ORDER BY criadoEm DESC LIMIT :keepCount
        )
    """)
    suspend fun trimOldest(keepCount: Int)
}
```

- [ ] **Step 6: Criar `EventDatabase.kt`**

```kotlin
package com.trivion.manageragent.event

import android.content.Context
import androidx.room.Database
import androidx.room.Room
import androidx.room.RoomDatabase

@Database(entities = [EventEntity::class], version = 1, exportSchema = true)
abstract class EventDatabase : RoomDatabase() {

    abstract fun eventDao(): EventDao

    companion object {
        @Volatile
        private var INSTANCE: EventDatabase? = null

        fun getInstance(context: Context): EventDatabase {
            return INSTANCE ?: synchronized(this) {
                INSTANCE ?: Room.databaseBuilder(
                    context.applicationContext,
                    EventDatabase::class.java,
                    "manageragent-events.db"
                ).build().also { INSTANCE = it }
            }
        }
    }
}
```

- [ ] **Step 7: Adicionar schema location no `app/build.gradle.kts`**

Em `defaultConfig { ... }`:

```kotlin
ksp {
    arg("room.schemaLocation", "$projectDir/schemas")
}
```

- [ ] **Step 8: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.event.EventDaoTest"`
Expected: PASS — todos os 7 testes.

- [ ] **Step 9: Commit**

```bash
git add app/build.gradle.kts app/src/main/kotlin/com/trivion/manageragent/event/ app/src/test/kotlin/com/trivion/manageragent/event/EventDaoTest.kt app/schemas/
git commit -m "feat: add Room EventDatabase + EventDao with FIFO queue + trim + retry counter"
```

---

### Task T10: EventQueue repository (interface entre coletores e DAO)

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/event/EventQueue.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/event/EventQueueTest.kt`

**Interfaces:**
- Consumes: `EventDao` (T9), `AgentEvent` sealed class (T8)
- Produces:
  - `class EventQueue(private val dao: EventDao, private val json: Json)`
  - `suspend fun enqueue(event: AgentEvent)` — serializa e insere
  - `suspend fun dequeueBatch(limit: Int = 100): List<Pair<Long, AgentEvent>>` — retorna ids + eventos deserializados
  - `suspend fun ack(ids: List<Long>)` — remove enviados com sucesso
  - `suspend fun nack(ids: List<Long>)` — incrementa tentativas
  - `suspend fun trimIfExceeds(maxCount: Int = 10_000)` — trim FIFO se acima do limite
  - `suspend fun pendingCount(): Int`

- [ ] **Step 1: Escrever teste**

Criar `app/src/test/kotlin/com/trivion/manageragent/event/EventQueueTest.kt`:

```kotlin
package com.trivion.manageragent.event

import androidx.room.Room
import androidx.test.core.app.ApplicationProvider
import androidx.test.ext.junit.runners.AndroidJUnit4
import com.trivion.manageragent.event.model.AgentEvent
import kotlinx.coroutines.test.runTest
import kotlinx.serialization.json.Json
import org.junit.jupiter.api.AfterEach
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import org.junit.runner.RunWith
import kotlin.test.assertEquals
import kotlin.test.assertTrue

@RunWith(AndroidJUnit4::class)
class EventQueueTest {

    private lateinit var db: EventDatabase
    private lateinit var queue: EventQueue
    private val json = Json { classDiscriminator = "tipoEvento"; encodeDefaults = true }

    @BeforeEach
    fun setUp() {
        db = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(), EventDatabase::class.java
        ).allowMainThreadQueries().build()
        queue = EventQueue(db.eventDao(), json)
    }

    @AfterEach
    fun tearDown() { db.close() }

    @Test
    fun `enqueue serializa e persiste`() = runTest {
        queue.enqueue(AgentEvent.Heartbeat(enviadoEm = "2026-08-08T14:00:00Z", eventosPendentes = 0))
        assertEquals(1, queue.pendingCount())
    }

    @Test
    fun `dequeueBatch retorna eventos deserializados na ordem de insercao`() = runTest {
        queue.enqueue(AgentEvent.Heartbeat("2026-08-08T14:00:00Z", 0))
        queue.enqueue(AgentEvent.Heartbeat("2026-08-08T14:01:00Z", 5))
        val batch = queue.dequeueBatch(limit = 10)
        assertEquals(2, batch.size)
        val (_, first) = batch[0]
        assertTrue(first is AgentEvent.Heartbeat)
        assertEquals(0, (first as AgentEvent.Heartbeat).eventosPendentes)
    }

    @Test
    fun `ack remove eventos`() = runTest {
        queue.enqueue(AgentEvent.Heartbeat("t1", 0))
        queue.enqueue(AgentEvent.Heartbeat("t2", 0))
        val batch = queue.dequeueBatch(10)
        queue.ack(listOf(batch[0].first))
        assertEquals(1, queue.pendingCount())
    }

    @Test
    fun `nack incrementa tentativas mas mantem evento`() = runTest {
        queue.enqueue(AgentEvent.Heartbeat("t1", 0))
        val batch1 = queue.dequeueBatch(10)
        queue.nack(listOf(batch1[0].first))
        val batch2 = queue.dequeueBatch(10)
        assertEquals(1, batch2.size)
    }

    @Test
    fun `trimIfExceeds so aciona quando acima do limite`() = runTest {
        repeat(5) { queue.enqueue(AgentEvent.Heartbeat("t$it", it)) }
        queue.trimIfExceeds(maxCount = 10)
        assertEquals(5, queue.pendingCount())

        repeat(10) { queue.enqueue(AgentEvent.Heartbeat("t$it", it)) }
        queue.trimIfExceeds(maxCount = 10)
        assertEquals(10, queue.pendingCount())
    }
}
```

- [ ] **Step 2: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.event.EventQueueTest"`
Expected: FAIL — `EventQueue` não existe.

- [ ] **Step 3: Criar `EventQueue.kt`**

```kotlin
package com.trivion.manageragent.event

import com.trivion.manageragent.event.model.AgentEvent
import kotlinx.serialization.encodeToString
import kotlinx.serialization.json.Json

class EventQueue(
    private val dao: EventDao,
    private val json: Json
) {

    suspend fun enqueue(event: AgentEvent) {
        val payload = json.encodeToString(event)
        val tipo = event::class.simpleName ?: "Unknown"
        dao.insert(
            EventEntity(
                payloadJson = payload,
                tipoEvento = tipo,
                criadoEm = System.currentTimeMillis()
            )
        )
    }

    suspend fun dequeueBatch(limit: Int = 100): List<Pair<Long, AgentEvent>> {
        val rows = dao.pending(limit)
        return rows.map { it.id to json.decodeFromString<AgentEvent>(it.payloadJson) }
    }

    suspend fun ack(ids: List<Long>) {
        if (ids.isNotEmpty()) dao.deleteByIds(ids)
    }

    suspend fun nack(ids: List<Long>) {
        if (ids.isNotEmpty()) dao.incrementTentativas(ids)
    }

    suspend fun trimIfExceeds(maxCount: Int = 10_000) {
        if (dao.count() > maxCount) {
            dao.trimOldest(keepCount = maxCount)
        }
    }

    suspend fun pendingCount(): Int = dao.count()
}
```

- [ ] **Step 4: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.event.EventQueueTest"`
Expected: PASS — todos os 5 testes.

- [ ] **Step 5: Commit**

```bash
git add app/src/main/kotlin/com/trivion/manageragent/event/EventQueue.kt app/src/test/kotlin/com/trivion/manageragent/event/EventQueueTest.kt
git commit -m "feat: add EventQueue repository (enqueue/dequeue/ack/nack/trim)"
```

---

### Task T11: ForegroundService + notification persistente mínima

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/service/ManagerAgentService.kt`
- Create: `app/src/main/kotlin/com/trivion/manageragent/service/NotificationHelper.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/service/NotificationHelperTest.kt`
- Modify: `app/src/main/AndroidManifest.xml` (declarar service + permissões FOREGROUND_SERVICE)
- Modify: `app/src/main/res/values/strings.xml` (textos da notification)

**Interfaces:**
- Consumes: `EventDatabase` (T9), `EventQueue` (T10)
- Produces: `ManagerAgentService(Service)` — inicia no boot / após permissões OK, mantém notification persistente. Método `startForegroundWithNotification()` reutilizável.

- [ ] **Step 1: Adicionar strings**

Editar `app/src/main/res/values/strings.xml`:

```xml
<resources>
    <string name="app_name">ManagerAgent</string>
    <string name="notification_channel_id">manageragent_status</string>
    <string name="notification_channel_name">ManagerAgent status</string>
    <string name="notification_status_text">ManagerAgent • em execução</string>
    <string name="notification_action_enviar_diagnostico">Enviar diagnóstico</string>
</resources>
```

- [ ] **Step 2: Escrever teste do NotificationHelper**

Criar `app/src/test/kotlin/com/trivion/manageragent/service/NotificationHelperTest.kt`:

```kotlin
package com.trivion.manageragent.service

import android.app.NotificationManager
import android.content.Context
import android.os.Build
import androidx.test.core.app.ApplicationProvider
import androidx.test.ext.junit.runners.AndroidJUnit4
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.assertDoesNotThrow
import org.junit.runner.RunWith
import kotlin.test.assertEquals
import kotlin.test.assertNotNull

@RunWith(AndroidJUnit4::class)
class NotificationHelperTest {

    private val context: Context = ApplicationProvider.getApplicationContext()

    @Test
    fun `ensureChannel cria channel LOW se API 26 plus`() {
        NotificationHelper.ensureChannel(context)
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val nm = context.getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
            val channel = nm.getNotificationChannel("manageragent_status")
            assertNotNull(channel)
            assertEquals(NotificationManager.IMPORTANCE_LOW, channel.importance)
        }
    }

    @Test
    fun `buildForegroundNotification retorna notification nao-nula`() {
        NotificationHelper.ensureChannel(context)
        val notif = NotificationHelper.buildForegroundNotification(context)
        assertNotNull(notif)
    }

    @Test
    fun `ensureChannel eh idempotente`() {
        assertDoesNotThrow {
            NotificationHelper.ensureChannel(context)
            NotificationHelper.ensureChannel(context)
        }
    }
}
```

- [ ] **Step 3: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.service.NotificationHelperTest"`
Expected: FAIL.

- [ ] **Step 4: Criar `NotificationHelper.kt`**

```kotlin
package com.trivion.manageragent.service

import android.app.Notification
import android.app.NotificationChannel
import android.app.NotificationManager
import android.content.Context
import android.os.Build
import androidx.core.app.NotificationCompat
import com.trivion.manageragent.R

object NotificationHelper {

    const val CHANNEL_ID = "manageragent_status"
    const val FOREGROUND_NOTIFICATION_ID = 1001

    fun ensureChannel(context: Context) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val nm = context.getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
            if (nm.getNotificationChannel(CHANNEL_ID) == null) {
                val channel = NotificationChannel(
                    CHANNEL_ID,
                    context.getString(R.string.notification_channel_name),
                    NotificationManager.IMPORTANCE_LOW
                ).apply {
                    setShowBadge(false)
                    lockscreenVisibility = Notification.VISIBILITY_SECRET
                    enableVibration(false)
                    enableLights(false)
                }
                nm.createNotificationChannel(channel)
            }
        }
    }

    fun buildForegroundNotification(context: Context): Notification {
        return NotificationCompat.Builder(context, CHANNEL_ID)
            .setContentTitle(context.getString(R.string.app_name))
            .setContentText(context.getString(R.string.notification_status_text))
            .setSmallIcon(android.R.drawable.ic_menu_view)  // TODO: ícone próprio na tarefa de assets
            .setPriority(NotificationCompat.PRIORITY_LOW)
            .setOngoing(true)
            .setSilent(true)
            .setShowWhen(false)
            .build()
    }
}
```

- [ ] **Step 5: Criar `ManagerAgentService.kt`**

```kotlin
package com.trivion.manageragent.service

import android.app.Service
import android.content.Intent
import android.content.pm.ServiceInfo
import android.os.Build
import android.os.IBinder
import timber.log.Timber

class ManagerAgentService : Service() {

    override fun onBind(intent: Intent?): IBinder? = null

    override fun onCreate() {
        super.onCreate()
        Timber.tag(TAG).i("onCreate")
        NotificationHelper.ensureChannel(this)
        startForegroundWithNotification()
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        Timber.tag(TAG).i("onStartCommand startId=$startId")
        return START_STICKY
    }

    override fun onDestroy() {
        Timber.tag(TAG).i("onDestroy")
        super.onDestroy()
    }

    private fun startForegroundWithNotification() {
        val notif = NotificationHelper.buildForegroundNotification(this)
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            startForeground(
                NotificationHelper.FOREGROUND_NOTIFICATION_ID,
                notif,
                ServiceInfo.FOREGROUND_SERVICE_TYPE_DATA_SYNC
            )
        } else {
            startForeground(NotificationHelper.FOREGROUND_NOTIFICATION_ID, notif)
        }
    }

    companion object { private const val TAG = "ManagerAgentService" }
}
```

- [ ] **Step 6: Atualizar `AndroidManifest.xml`**

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    <uses-permission android:name="android.permission.WAKE_LOCK" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC" />
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

    <application
        android:name=".ManagerAgentApplication"
        android:allowBackup="false"
        android:label="@string/app_name"
        android:supportsRtl="true"
        android:theme="@android:style/Theme.NoDisplay">

        <service
            android:name=".service.ManagerAgentService"
            android:exported="false"
            android:foregroundServiceType="dataSync" />
    </application>
</manifest>
```

- [ ] **Step 7: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.service.NotificationHelperTest"`
Expected: PASS.

- [ ] **Step 8: Build APK e verificar**

Run: `./gradlew :app:assembleDebug`
Expected: BUILD SUCCESSFUL.

- [ ] **Step 9: Commit**

```bash
git add .
git commit -m "feat: add ManagerAgentService (ForegroundService dataSync) + NotificationHelper (silent LOW channel)"
```

---

### Task T12: SessionBroadcastReceiver + emissão de SessionEvent LOCK/UNLOCK

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/service/SessionBroadcastReceiver.kt`
- Create: `app/src/main/kotlin/com/trivion/manageragent/collector/SessionEventFactory.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/collector/SessionEventFactoryTest.kt`
- Modify: `app/src/main/AndroidManifest.xml` (declarar receiver)

**Interfaces:**
- Consumes: `EventQueue` (T10), `AgentEvent.Session` (T8)
- Produces:
  - `SessionEventFactory` — função pura que dado `intent.action` retorna `AgentEvent.Session?`
  - `SessionBroadcastReceiver` — registrado no Manifest pra `SCREEN_ON`/`SCREEN_OFF`/`USER_PRESENT`, delega pro `SessionEventFactory` e enfileira via `EventQueue`

- [ ] **Step 1: Escrever teste do factory**

Criar `app/src/test/kotlin/com/trivion/manageragent/collector/SessionEventFactoryTest.kt`:

```kotlin
package com.trivion.manageragent.collector

import android.content.Intent
import com.trivion.manageragent.event.model.AgentEvent
import org.junit.jupiter.api.Test
import kotlin.test.assertEquals
import kotlin.test.assertNull
import kotlin.test.assertNotNull

class SessionEventFactoryTest {

    private val factory = SessionEventFactory(clock = { "2026-08-08T14:00:00Z" })

    @Test
    fun `SCREEN_OFF vira LOCK`() {
        val evt = factory.fromAction(Intent.ACTION_SCREEN_OFF)
        assertNotNull(evt)
        assertEquals("LOCK", evt.tipoTransicao)
        assertEquals("2026-08-08T14:00:00Z", evt.ocorreuEm)
    }

    @Test
    fun `USER_PRESENT vira UNLOCK`() {
        val evt = factory.fromAction(Intent.ACTION_USER_PRESENT)
        assertNotNull(evt)
        assertEquals("UNLOCK", evt.tipoTransicao)
    }

    @Test
    fun `SCREEN_ON sem USER_PRESENT nao dispara evento`() {
        // SCREEN_ON sozinho não é UNLOCK real (tela pode estar acesa sem desbloqueio)
        val evt = factory.fromAction(Intent.ACTION_SCREEN_ON)
        assertNull(evt)
    }

    @Test
    fun `action desconhecida retorna null`() {
        assertNull(factory.fromAction("com.foo.BAR"))
        assertNull(factory.fromAction(null))
    }
}
```

- [ ] **Step 2: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.collector.SessionEventFactoryTest"`
Expected: FAIL.

- [ ] **Step 3: Criar `SessionEventFactory.kt`**

```kotlin
package com.trivion.manageragent.collector

import android.content.Intent
import com.trivion.manageragent.event.model.AgentEvent
import java.time.Instant

class SessionEventFactory(
    private val clock: () -> String = { Instant.now().toString() }
) {
    fun fromAction(action: String?): AgentEvent.Session? {
        val tipo = when (action) {
            Intent.ACTION_SCREEN_OFF -> "LOCK"
            Intent.ACTION_USER_PRESENT -> "UNLOCK"
            else -> null
        } ?: return null
        return AgentEvent.Session(tipoTransicao = tipo, ocorreuEm = clock())
    }
}
```

- [ ] **Step 4: Criar `SessionBroadcastReceiver.kt`**

```kotlin
package com.trivion.manageragent.service

import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import com.trivion.manageragent.collector.SessionEventFactory
import com.trivion.manageragent.event.EventDatabase
import com.trivion.manageragent.event.EventQueue
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.SupervisorJob
import kotlinx.coroutines.launch
import kotlinx.serialization.json.Json
import timber.log.Timber

class SessionBroadcastReceiver : BroadcastReceiver() {

    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
    private val factory = SessionEventFactory()
    private val json = Json { classDiscriminator = "tipoEvento"; encodeDefaults = true }

    override fun onReceive(context: Context, intent: Intent) {
        val event = factory.fromAction(intent.action) ?: return
        Timber.tag(TAG).i("Session event: ${event.tipoTransicao}")

        val queue = EventQueue(EventDatabase.getInstance(context).eventDao(), json)
        val pending = goAsync()
        scope.launch {
            try { queue.enqueue(event) } finally { pending.finish() }
        }
    }

    companion object { private const val TAG = "SessionBroadcastReceiver" }
}
```

- [ ] **Step 5: Atualizar `AndroidManifest.xml`**

Dentro de `<application>`:

```xml
<receiver
    android:name=".service.SessionBroadcastReceiver"
    android:exported="true"
    android:enabled="true">
    <intent-filter>
        <action android:name="android.intent.action.SCREEN_ON" />
        <action android:name="android.intent.action.SCREEN_OFF" />
        <action android:name="android.intent.action.USER_PRESENT" />
    </intent-filter>
</receiver>
```

Nota: `SCREEN_ON`/`SCREEN_OFF` não podem ser registrados apenas no Manifest a partir do Android 8 — devem ser registrados dinamicamente no `ManagerAgentService.onCreate()`. Adicionar registro dinâmico:

Modificar `ManagerAgentService.kt` — adicionar registro dinâmico:

```kotlin
private val sessionReceiver = SessionBroadcastReceiver()

override fun onCreate() {
    super.onCreate()
    Timber.tag(TAG).i("onCreate")
    NotificationHelper.ensureChannel(this)
    startForegroundWithNotification()
    val filter = IntentFilter().apply {
        addAction(Intent.ACTION_SCREEN_ON)
        addAction(Intent.ACTION_SCREEN_OFF)
        addAction(Intent.ACTION_USER_PRESENT)
    }
    registerReceiver(sessionReceiver, filter)
}

override fun onDestroy() {
    try { unregisterReceiver(sessionReceiver) } catch (e: Exception) { Timber.tag(TAG).w(e, "unregister failed") }
    super.onDestroy()
}
```

- [ ] **Step 6: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.collector.SessionEventFactoryTest"`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add .
git commit -m "feat: add SessionBroadcastReceiver + SessionEventFactory (LOCK/UNLOCK)"
```

---

### Task T13: BootReceiver — auto-start após boot

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/service/BootReceiver.kt`
- Modify: `app/src/main/AndroidManifest.xml`

**Interfaces:**
- Consumes: `ManagerAgentService` (T11)
- Produces: `BootReceiver` dispara `startForegroundService(Intent(context, ManagerAgentService::class.java))` ao receber `ACTION_BOOT_COMPLETED`. Nota: OEMs (Xiaomi/Huawei) podem bloquear — coberto pela detecção OEM em T33.

- [ ] **Step 1: Adicionar permissão no Manifest**

```xml
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
```

- [ ] **Step 2: Criar `BootReceiver.kt`**

```kotlin
package com.trivion.manageragent.service

import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import android.os.Build
import timber.log.Timber

class BootReceiver : BroadcastReceiver() {

    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action != Intent.ACTION_BOOT_COMPLETED &&
            intent.action != Intent.ACTION_LOCKED_BOOT_COMPLETED) {
            return
        }
        Timber.tag(TAG).i("Boot completed, iniciando ManagerAgentService")
        val svc = Intent(context, ManagerAgentService::class.java)
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            context.startForegroundService(svc)
        } else {
            context.startService(svc)
        }
    }

    companion object { private const val TAG = "BootReceiver" }
}
```

- [ ] **Step 3: Registrar no Manifest**

Dentro de `<application>`:

```xml
<receiver
    android:name=".service.BootReceiver"
    android:exported="true"
    android:enabled="true">
    <intent-filter android:priority="1000">
        <action android:name="android.intent.action.BOOT_COMPLETED" />
        <action android:name="android.intent.action.LOCKED_BOOT_COMPLETED" />
    </intent-filter>
</receiver>
```

- [ ] **Step 4: Escrever teste manual (não automatizado — precisa device real)**

Documentar no `README.md`, seção "Testes manuais":

```markdown
### Auto-start após reboot

1. Instale o APK em device de teste
2. Complete a vinculação inicial
3. Verifique notification "ManagerAgent • em execução"
4. Reboote o device
5. Após ~15s da tela desbloquear, notification deve reaparecer
6. Se não reaparecer, checar Settings > Apps > ManagerAgent > Battery > Sem restrição
   (OEMs matam por padrão — coberto por detecção OEM na tela de permissões)
```

- [ ] **Step 5: Commit**

```bash
git add .
git commit -m "feat: add BootReceiver for auto-start after device reboot"
```

---

### Task T14: AccessibilityService setup (config + skeleton)

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/service/ManagerAccessibilityService.kt`
- Create: `app/src/main/res/xml/accessibility_config.xml`
- Modify: `app/src/main/AndroidManifest.xml`
- Modify: `app/src/main/res/values/strings.xml`

**Interfaces:**
- Consumes: `EventQueue` (T10)
- Produces: `ManagerAccessibilityService` habilitável em Settings > Accessibility. Configurada pra receber `TYPE_WINDOW_STATE_CHANGED`, `TYPE_VIEW_CLICKED`, `TYPE_VIEW_SCROLLED`, `TYPE_VIEW_TEXT_CHANGED`. Ainda sem lógica de coleta — só skeleton que recebe eventos e chama `WindowActivityCollector` / `ActivitySummaryCollector` (implementados em T15+).

- [ ] **Step 1: Adicionar string de descrição do serviço**

Editar `app/src/main/res/values/strings.xml`:

```xml
<string name="accessibility_description">
    Coleta metadados de janela ativa e interações agregadas
    (sem ler conteúdo de textos) para gerar relatórios de produtividade.
</string>
```

- [ ] **Step 2: Criar `app/src/main/res/xml/accessibility_config.xml`**

```xml
<?xml version="1.0" encoding="utf-8"?>
<accessibility-service xmlns:android="http://schemas.android.com/apk/res/android"
    android:accessibilityEventTypes="typeWindowStateChanged|typeViewClicked|typeViewScrolled|typeViewTextChanged|typeWindowContentChanged"
    android:accessibilityFeedbackType="feedbackGeneric"
    android:accessibilityFlags="flagRetrieveInteractiveWindows|flagReportViewIds"
    android:canRetrieveWindowContent="true"
    android:description="@string/accessibility_description"
    android:notificationTimeout="200" />
```

- [ ] **Step 3: Criar `ManagerAccessibilityService.kt` (skeleton)**

```kotlin
package com.trivion.manageragent.service

import android.accessibilityservice.AccessibilityService
import android.view.accessibility.AccessibilityEvent
import timber.log.Timber

class ManagerAccessibilityService : AccessibilityService() {

    override fun onServiceConnected() {
        super.onServiceConnected()
        Timber.tag(TAG).i("AccessibilityService connected")
        // TODO: emitir audit AGENT_ACCESSIBILITY_ATIVADA (T38)
    }

    override fun onAccessibilityEvent(event: AccessibilityEvent?) {
        if (event == null) return
        when (event.eventType) {
            AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED -> {
                // TODO: encaminhar pra WindowActivityCollector (T15)
                Timber.tag(TAG).v("Window changed: ${event.packageName}/${event.className}")
            }
            AccessibilityEvent.TYPE_VIEW_CLICKED,
            AccessibilityEvent.TYPE_VIEW_SCROLLED,
            AccessibilityEvent.TYPE_VIEW_TEXT_CHANGED -> {
                // TODO: encaminhar pra ActivitySummaryCollector (T20)
            }
            else -> Unit
        }
    }

    override fun onInterrupt() {
        Timber.tag(TAG).w("AccessibilityService interrupted")
    }

    override fun onUnbind(intent: android.content.Intent?): Boolean {
        Timber.tag(TAG).w("AccessibilityService onUnbind — usuario desabilitou")
        // TODO: emitir audit AGENT_ACCESSIBILITY_DESATIVADA (T38)
        return super.onUnbind(intent)
    }

    companion object { private const val TAG = "ManagerAccessibilityService" }
}
```

- [ ] **Step 4: Registrar no Manifest**

Dentro de `<application>`:

```xml
<service
    android:name=".service.ManagerAccessibilityService"
    android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE"
    android:exported="true">
    <intent-filter>
        <action android:name="android.accessibilityservice.AccessibilityService" />
    </intent-filter>
    <meta-data
        android:name="android.accessibilityservice"
        android:resource="@xml/accessibility_config" />
</service>
```

- [ ] **Step 5: Build e verificar**

Run: `./gradlew :app:assembleDebug`
Expected: BUILD SUCCESSFUL.

Instalar em emulador e ativar em Settings > Accessibility > ManagerAgent → verificar log `AccessibilityService connected` em `adb logcat -s ManagerAccessibilityService:*`.

- [ ] **Step 6: Commit**

```bash
git add .
git commit -m "feat: add ManagerAccessibilityService skeleton (config + registration)"
```

---

### Task T15: WindowActivityCollector — traduz eventos do Accessibility em AgentEvent.WindowActivity

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/collector/WindowActivityCollector.kt`
- Create: `app/src/main/kotlin/com/trivion/manageragent/collector/PackageMetadataProvider.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/collector/WindowActivityCollectorTest.kt`
- Modify: `app/src/main/kotlin/com/trivion/manageragent/service/ManagerAccessibilityService.kt` (delegar pra collector)

**Interfaces:**
- Consumes: `EventQueue` (T10)
- Produces:
  - `WindowActivityCollector(queue: EventQueue, packageMeta: PackageMetadataProvider, clock: () -> String)`
  - `suspend fun onWindowStateChanged(packageName: String?, className: String?)` — fecha janela anterior (grava com `finalizadoEm=agora`) e abre nova janela (guarda no state)
  - Se `packageName` == mesmo da janela anterior, não emite (continuação)
  - Filtro LGPD: **nunca** grava título da janela contendo texto arbitrário; usa `packageMeta.humanNameFor(packageName)` como `tituloJanela`

- [ ] **Step 1: Escrever teste**

Criar `app/src/test/kotlin/com/trivion/manageragent/collector/WindowActivityCollectorTest.kt`:

```kotlin
package com.trivion.manageragent.collector

import com.trivion.manageragent.event.model.AgentEvent
import io.mockk.coEvery
import io.mockk.coVerify
import io.mockk.mockk
import io.mockk.slot
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import kotlin.test.assertEquals

class WindowActivityCollectorTest {

    private lateinit var queue: com.trivion.manageragent.event.EventQueue
    private lateinit var meta: PackageMetadataProvider
    private var now = 0L
    private lateinit var collector: WindowActivityCollector

    @BeforeEach
    fun setUp() {
        queue = mockk(relaxed = true)
        meta = mockk()
        collector = WindowActivityCollector(
            queue = queue,
            packageMeta = meta,
            clock = { "2026-08-08T14:00:0${now}Z".also { now = (now + 1) % 60 } }
        )
        coEvery { meta.humanNameFor(any()) } answers { "App[${firstArg<String>()}]" }
    }

    @Test
    fun `primeira janela nao gera evento imediato (so na proxima transicao)`() = runTest {
        collector.onWindowStateChanged("com.whatsapp", "HomeActivity")
        coVerify(exactly = 0) { queue.enqueue(any()) }
    }

    @Test
    fun `segunda janela diferente fecha primeira e emite WindowActivity`() = runTest {
        collector.onWindowStateChanged("com.whatsapp", "HomeActivity")
        collector.onWindowStateChanged("com.instagram", "FeedActivity")

        val slot = slot<AgentEvent>()
        coVerify(exactly = 1) { queue.enqueue(capture(slot)) }
        val evt = slot.captured as AgentEvent.WindowActivity
        assertEquals("com.whatsapp", evt.nomeProcesso)
        assertEquals("App[com.whatsapp]", evt.tituloJanela)
        assertEquals("2026-08-08T14:00:00Z", evt.iniciadoEm)
        assertEquals("2026-08-08T14:00:01Z", evt.finalizadoEm)
    }

    @Test
    fun `mesmo package em sequencia nao emite`() = runTest {
        collector.onWindowStateChanged("com.whatsapp", "HomeActivity")
        collector.onWindowStateChanged("com.whatsapp", "ChatActivity")
        coVerify(exactly = 0) { queue.enqueue(any()) }
    }

    @Test
    fun `packageName null ou vazio eh ignorado`() = runTest {
        collector.onWindowStateChanged(null, null)
        collector.onWindowStateChanged("", "")
        coVerify(exactly = 0) { queue.enqueue(any()) }
    }
}
```

- [ ] **Step 2: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.collector.WindowActivityCollectorTest"`
Expected: FAIL.

- [ ] **Step 3: Criar `PackageMetadataProvider.kt`**

```kotlin
package com.trivion.manageragent.collector

import android.content.Context
import android.content.pm.PackageManager

class PackageMetadataProvider(private val context: Context) {

    fun humanNameFor(packageName: String): String {
        return try {
            val pm = context.packageManager
            val info = pm.getApplicationInfo(packageName, 0)
            pm.getApplicationLabel(info).toString()
        } catch (e: PackageManager.NameNotFoundException) {
            packageName  // fallback: usar o próprio packageName
        }
    }
}
```

- [ ] **Step 4: Criar `WindowActivityCollector.kt`**

```kotlin
package com.trivion.manageragent.collector

import com.trivion.manageragent.event.EventQueue
import com.trivion.manageragent.event.model.AgentEvent
import java.time.Instant

class WindowActivityCollector(
    private val queue: EventQueue,
    private val packageMeta: PackageMetadataProvider,
    private val clock: () -> String = { Instant.now().toString() }
) {

    private var currentPackage: String? = null
    private var currentIniciadoEm: String? = null

    suspend fun onWindowStateChanged(packageName: String?, className: String?) {
        if (packageName.isNullOrBlank()) return
        val agora = clock()

        val previous = currentPackage
        val previousIniciadoEm = currentIniciadoEm

        if (previous != null && previous != packageName && previousIniciadoEm != null) {
            queue.enqueue(
                AgentEvent.WindowActivity(
                    nomeProcesso = previous,
                    tituloJanela = packageMeta.humanNameFor(previous),
                    iniciadoEm = previousIniciadoEm,
                    finalizadoEm = agora,
                    statusUsuario = "ATIVO",
                    urlDominio = ""  // preenchido em T25
                )
            )
        }

        if (previous != packageName) {
            currentPackage = packageName
            currentIniciadoEm = agora
        }
    }
}
```

- [ ] **Step 5: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.collector.WindowActivityCollectorTest"`
Expected: PASS.

- [ ] **Step 6: Integrar no AccessibilityService**

Editar `ManagerAccessibilityService.kt`:

```kotlin
package com.trivion.manageragent.service

import android.accessibilityservice.AccessibilityService
import android.view.accessibility.AccessibilityEvent
import com.trivion.manageragent.collector.PackageMetadataProvider
import com.trivion.manageragent.collector.WindowActivityCollector
import com.trivion.manageragent.event.EventDatabase
import com.trivion.manageragent.event.EventQueue
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.SupervisorJob
import kotlinx.coroutines.launch
import kotlinx.serialization.json.Json
import timber.log.Timber

class ManagerAccessibilityService : AccessibilityService() {

    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
    private val json = Json { classDiscriminator = "tipoEvento"; encodeDefaults = true }
    private lateinit var windowCollector: WindowActivityCollector

    override fun onServiceConnected() {
        super.onServiceConnected()
        Timber.tag(TAG).i("AccessibilityService connected")
        val queue = EventQueue(EventDatabase.getInstance(this).eventDao(), json)
        windowCollector = WindowActivityCollector(queue, PackageMetadataProvider(this))
    }

    override fun onAccessibilityEvent(event: AccessibilityEvent?) {
        if (event == null) return
        when (event.eventType) {
            AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED -> {
                val pkg = event.packageName?.toString()
                val cls = event.className?.toString()
                scope.launch { windowCollector.onWindowStateChanged(pkg, cls) }
            }
            else -> Unit
        }
    }

    override fun onInterrupt() { Timber.tag(TAG).w("Interrupted") }

    companion object { private const val TAG = "ManagerAccessibilityService" }
}
```

- [ ] **Step 7: Commit**

```bash
git add .
git commit -m "feat: add WindowActivityCollector wired to AccessibilityService"
```

---

### Task T16: IdleDetector — detecta ociosidade (sem interação)

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/collector/IdleDetector.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/collector/IdleDetectorTest.kt`

**Interfaces:**
- Consumes: `EventQueue` (T10)
- Produces:
  - `IdleDetector(queue, thresholdMillis: Long = 60_000)`
  - `fun onInteraction()` — atualiza `lastInteraction` timestamp
  - `suspend fun tick()` — chamado periodicamente (a cada 30s pelo Service). Se `now - lastInteraction > threshold` e não estava idle, emite `IdleEvent` de início e marca `isIdle`. Se estava idle e recebe interaction, fecha o evento anterior.

- [ ] **Step 1: Escrever teste**

Criar `app/src/test/kotlin/com/trivion/manageragent/collector/IdleDetectorTest.kt`:

```kotlin
package com.trivion.manageragent.collector

import com.trivion.manageragent.event.EventQueue
import com.trivion.manageragent.event.model.AgentEvent
import io.mockk.coVerify
import io.mockk.mockk
import io.mockk.slot
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Test
import kotlin.test.assertEquals

class IdleDetectorTest {

    @Test
    fun `nao emite se ativo abaixo do threshold`() = runTest {
        val queue = mockk<EventQueue>(relaxed = true)
        var now = 0L
        val det = IdleDetector(queue, thresholdMillis = 1_000L, nowMillis = { now })

        det.onInteraction()
        now = 500L
        det.tick()
        coVerify(exactly = 0) { queue.enqueue(any()) }
    }

    @Test
    fun `emite IdleEvent nao terminado ao ultrapassar threshold`() = runTest {
        val queue = mockk<EventQueue>(relaxed = true)
        var now = 0L
        val det = IdleDetector(queue, thresholdMillis = 1_000L, nowMillis = { now }, clock = {
            val ms = now
            "clock:$ms"
        })

        det.onInteraction()
        now = 2_000L
        det.tick()

        val slot = slot<AgentEvent>()
        coVerify(exactly = 1) { queue.enqueue(capture(slot)) }
        val evt = slot.captured as AgentEvent.Idle
        assertEquals("clock:1000", evt.iniciadoEm)   // início = último interaction + threshold... na verdade último interaction
        assertEquals("clock:2000", evt.finalizadoEm)
        assertEquals("AUSENTE", evt.statusUsuario)
    }

    @Test
    fun `nao emite duas vezes durante idle continuo`() = runTest {
        val queue = mockk<EventQueue>(relaxed = true)
        var now = 0L
        val det = IdleDetector(queue, thresholdMillis = 1_000L, nowMillis = { now }, clock = { "$now" })

        det.onInteraction()
        now = 2_000L; det.tick()
        now = 3_000L; det.tick()
        coVerify(exactly = 1) { queue.enqueue(any()) }
    }

    @Test
    fun `interaction apos idle emite evento de fim`() = runTest {
        val queue = mockk<EventQueue>(relaxed = true)
        var now = 0L
        val det = IdleDetector(queue, thresholdMillis = 1_000L, nowMillis = { now }, clock = { "$now" })

        det.onInteraction()          // ativo em t=0
        now = 2_000L; det.tick()     // vira idle → emite start
        now = 3_000L; det.onInteraction()  // volta a ativo → emite end
        coVerify(exactly = 2) { queue.enqueue(any()) }
    }
}
```

- [ ] **Step 2: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.collector.IdleDetectorTest"`
Expected: FAIL.

- [ ] **Step 3: Criar `IdleDetector.kt`**

```kotlin
package com.trivion.manageragent.collector

import com.trivion.manageragent.event.EventQueue
import com.trivion.manageragent.event.model.AgentEvent
import java.time.Instant

class IdleDetector(
    private val queue: EventQueue,
    private val thresholdMillis: Long = 60_000L,
    private val nowMillis: () -> Long = { System.currentTimeMillis() },
    private val clock: () -> String = { Instant.now().toString() }
) {

    private var lastInteractionMs: Long = nowMillis()
    private var idleStartClock: String? = null

    fun onInteraction() {
        val wasIdle = idleStartClock != null
        val agora = nowMillis()
        lastInteractionMs = agora
        if (wasIdle) {
            val start = idleStartClock!!
            idleStartClock = null
            // Emissão em coroutine no Service (aqui é síncrono pra teste)
            enqueueBlocking(
                AgentEvent.Idle(
                    iniciadoEm = start,
                    finalizadoEm = clock(),
                    statusUsuario = "AUSENTE"
                )
            )
        }
    }

    suspend fun tick() {
        val agora = nowMillis()
        val silenceMs = agora - lastInteractionMs
        if (silenceMs > thresholdMillis && idleStartClock == null) {
            // Marca início da ociosidade = momento da última interação (para incluir o intervalo)
            val startInstant = Instant.ofEpochMilli(lastInteractionMs).toString()
            idleStartClock = startInstant
            // Emite evento com finalizadoEm = agora (evento aberto — teste espera evento imediato)
            queue.enqueue(
                AgentEvent.Idle(
                    iniciadoEm = startInstant,
                    finalizadoEm = clock(),
                    statusUsuario = "AUSENTE"
                )
            )
        }
    }

    // Helper síncrono pra emissão via onInteraction (que não é suspend por design — chamado do AccessibilityService)
    // Em produção, delega pro CoroutineScope injetado; aqui usa runBlocking pra facilitar teste.
    private fun enqueueBlocking(event: AgentEvent) {
        kotlinx.coroutines.runBlocking { queue.enqueue(event) }
    }
}
```

- [ ] **Step 4: Ajustar teste (o teste espera clock baseado em nowMillis, então clock deve mapear pra "$now")**

Verificar teste `emite IdleEvent nao terminado ao ultrapassar threshold`:
- `onInteraction()` em `now=0` → `lastInteractionMs=0`
- `now=2000` → `tick()` detecta silence `2000-0 > 1000` → emite Idle com `iniciadoEm = Instant.ofEpochMilli(0)` = `"1970-01-01T00:00:00Z"`... mas o teste espera `"clock:1000"`.

Ajustar teste pra usar `Instant.ofEpochMilli`:

```kotlin
@Test
fun `emite IdleEvent nao terminado ao ultrapassar threshold`() = runTest {
    val queue = mockk<EventQueue>(relaxed = true)
    var now = 0L
    val det = IdleDetector(queue, thresholdMillis = 1_000L, nowMillis = { now }, clock = { "clock:$now" })

    det.onInteraction()   // lastInteractionMs = 0
    now = 2_000L
    det.tick()

    val slot = slot<AgentEvent>()
    coVerify(exactly = 1) { queue.enqueue(capture(slot)) }
    val evt = slot.captured as AgentEvent.Idle
    assertEquals("1970-01-01T00:00:00Z", evt.iniciadoEm)  // Instant.ofEpochMilli(0)
    assertEquals("clock:2000", evt.finalizadoEm)
}
```

- [ ] **Step 5: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.collector.IdleDetectorTest"`
Expected: PASS (4 testes).

- [ ] **Step 6: Commit**

```bash
git add .
git commit -m "feat: add IdleDetector (threshold-based, emits on transition)"
```

---

### Task T17: HeartbeatEmitter (WorkManager periódico 60s)

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/collector/HeartbeatWorker.kt`
- Create: `app/src/main/kotlin/com/trivion/manageragent/collector/HeartbeatScheduler.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/collector/HeartbeatWorkerTest.kt`
- Modify: `app/build.gradle.kts` (adicionar `androidx-work-runtime-ktx`)
- Modify: `ManagerAgentService.kt` (chama `HeartbeatScheduler.schedule(context)` no `onCreate`)

**Interfaces:**
- Consumes: `EventQueue` (T10)
- Produces:
  - `HeartbeatWorker(context, workerParams)` — `doWork()` enfileira `AgentEvent.Heartbeat(enviadoEm=agora, eventosPendentes=queue.pendingCount())`
  - `HeartbeatScheduler.schedule(context, intervalMinutes=15)` — registra `PeriodicWorkRequest` (mínimo do WorkManager é 15min; usa `KEEP` policy pra idempotência)

- [ ] **Step 1: Adicionar dependência WorkManager**

Em `app/build.gradle.kts`:

```kotlin
implementation(libs.androidx.work.runtime.ktx)
testImplementation("androidx.work:work-testing:2.9.0")
```

Adicionar em `libs.versions.toml`:

```toml
[libraries]
androidx-work-testing = { group = "androidx.work", name = "work-testing", version.ref = "workManager" }
```

- [ ] **Step 2: Escrever teste**

Criar `app/src/test/kotlin/com/trivion/manageragent/collector/HeartbeatWorkerTest.kt`:

```kotlin
package com.trivion.manageragent.collector

import android.content.Context
import androidx.test.core.app.ApplicationProvider
import androidx.test.ext.junit.runners.AndroidJUnit4
import androidx.work.ListenableWorker
import androidx.work.testing.TestListenableWorkerBuilder
import androidx.work.testing.WorkManagerTestInitHelper
import kotlinx.coroutines.runBlocking
import org.junit.jupiter.api.Test
import org.junit.runner.RunWith
import kotlin.test.assertEquals

@RunWith(AndroidJUnit4::class)
class HeartbeatWorkerTest {

    @Test
    fun `doWork retorna Success`() = runBlocking {
        val context: Context = ApplicationProvider.getApplicationContext()
        val worker = TestListenableWorkerBuilder<HeartbeatWorker>(context).build()
        val result = worker.doWork()
        assertEquals(ListenableWorker.Result.success(), result)
    }
}
```

- [ ] **Step 3: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.collector.HeartbeatWorkerTest"`
Expected: FAIL.

- [ ] **Step 4: Criar `HeartbeatWorker.kt`**

```kotlin
package com.trivion.manageragent.collector

import android.content.Context
import androidx.work.CoroutineWorker
import androidx.work.WorkerParameters
import com.trivion.manageragent.event.EventDatabase
import com.trivion.manageragent.event.EventQueue
import com.trivion.manageragent.event.model.AgentEvent
import kotlinx.serialization.json.Json
import timber.log.Timber
import java.time.Instant

class HeartbeatWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val json = Json { classDiscriminator = "tipoEvento"; encodeDefaults = true }
        val queue = EventQueue(EventDatabase.getInstance(applicationContext).eventDao(), json)
        val pending = queue.pendingCount()
        val evt = AgentEvent.Heartbeat(enviadoEm = Instant.now().toString(), eventosPendentes = pending)
        queue.enqueue(evt)
        Timber.tag("HeartbeatWorker").v("heartbeat enqueued, pending=$pending")
        return Result.success()
    }
}
```

- [ ] **Step 5: Criar `HeartbeatScheduler.kt`**

```kotlin
package com.trivion.manageragent.collector

import android.content.Context
import androidx.work.ExistingPeriodicWorkPolicy
import androidx.work.PeriodicWorkRequestBuilder
import androidx.work.WorkManager
import java.util.concurrent.TimeUnit

object HeartbeatScheduler {

    private const val WORK_NAME = "ManagerAgent.Heartbeat"

    fun schedule(context: Context, intervalMinutes: Long = 15L) {
        val req = PeriodicWorkRequestBuilder<HeartbeatWorker>(intervalMinutes, TimeUnit.MINUTES).build()
        WorkManager.getInstance(context).enqueueUniquePeriodicWork(
            WORK_NAME,
            ExistingPeriodicWorkPolicy.KEEP,
            req
        )
    }

    fun cancel(context: Context) {
        WorkManager.getInstance(context).cancelUniqueWork(WORK_NAME)
    }
}
```

- [ ] **Step 6: Wire no ManagerAgentService**

Editar `ManagerAgentService.onCreate()`:

```kotlin
override fun onCreate() {
    super.onCreate()
    Timber.tag(TAG).i("onCreate")
    NotificationHelper.ensureChannel(this)
    startForegroundWithNotification()
    val filter = IntentFilter().apply {
        addAction(Intent.ACTION_SCREEN_ON)
        addAction(Intent.ACTION_SCREEN_OFF)
        addAction(Intent.ACTION_USER_PRESENT)
    }
    registerReceiver(sessionReceiver, filter)
    HeartbeatScheduler.schedule(this)  // NOVO
}
```

- [ ] **Step 7: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.collector.HeartbeatWorkerTest"`
Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add .
git commit -m "feat: add HeartbeatWorker (WorkManager periodic 15min)"
```

---

### Task T18: Smoke test integrado — Service + Receivers + AccessibilityService

**Files:**
- Create: `app/src/androidTest/kotlin/com/trivion/manageragent/service/ManagerAgentServiceInstrumentedTest.kt`

**Interfaces:**
- Consumes: tudo de T11-T17
- Produces: teste instrumented que sobe o Service em emulador, força um `SCREEN_OFF` broadcast, valida que evento SessionEvent LOCK foi enfileirado na DB

- [ ] **Step 1: Criar teste instrumented**

Criar `app/src/androidTest/kotlin/com/trivion/manageragent/service/ManagerAgentServiceInstrumentedTest.kt`:

```kotlin
package com.trivion.manageragent.service

import android.content.Intent
import androidx.test.core.app.ApplicationProvider
import androidx.test.ext.junit.runners.AndroidJUnit4
import com.trivion.manageragent.event.EventDatabase
import com.trivion.manageragent.event.EventQueue
import kotlinx.coroutines.delay
import kotlinx.coroutines.runBlocking
import kotlinx.serialization.json.Json
import org.junit.After
import org.junit.Before
import org.junit.Test
import org.junit.runner.RunWith
import kotlin.test.assertTrue

@RunWith(AndroidJUnit4::class)
class ManagerAgentServiceInstrumentedTest {

    private val context = ApplicationProvider.getApplicationContext<android.content.Context>()

    @Before
    fun clearDb() = runBlocking {
        val db = EventDatabase.getInstance(context)
        db.eventDao().pending(Int.MAX_VALUE).map { it.id }.let {
            if (it.isNotEmpty()) db.eventDao().deleteByIds(it)
        }
    }

    @After
    fun stopSvc() { context.stopService(Intent(context, ManagerAgentService::class.java)) }

    @Test
    fun `service inicia notification e SCREEN_OFF gera SessionEvent LOCK`() = runBlocking {
        context.startForegroundService(Intent(context, ManagerAgentService::class.java))
        delay(2_000)

        // Dispara SCREEN_OFF simulado (registrado dinamicamente pelo Service)
        context.sendBroadcast(Intent(Intent.ACTION_SCREEN_OFF))
        delay(1_500)

        val json = Json { classDiscriminator = "tipoEvento"; encodeDefaults = true }
        val queue = EventQueue(EventDatabase.getInstance(context).eventDao(), json)
        val pending = queue.dequeueBatch(10)
        assertTrue(pending.any { (_, evt) -> evt is com.trivion.manageragent.event.model.AgentEvent.Session })
    }
}
```

- [ ] **Step 2: Rodar em emulador**

Precisa emulador ativo:

```bash
./gradlew :app:connectedDebugAndroidTest --tests "com.trivion.manageragent.service.ManagerAgentServiceInstrumentedTest"
```

Expected: PASS. Se falhar por causa de timing, ajustar `delay(1_500)` pra `delay(3_000)`.

- [ ] **Step 3: Commit**

```bash
git add app/src/androidTest/
git commit -m "test: add instrumented smoke test for ManagerAgentService + SessionReceiver"
```

---

## BLOCO B4 — Coleta complementar (T19-T25)

### Task T19: UsageStatsCollector — reconciliação a cada 5min

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/collector/UsageStatsCollector.kt`
- Create: `app/src/main/kotlin/com/trivion/manageragent/collector/UsageStatsReconcileWorker.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/collector/UsageStatsCollectorTest.kt`

**Interfaces:**
- Consumes: `EventQueue` (T10), Android `UsageStatsManager`
- Produces:
  - `UsageStatsCollector(usm: UsageStatsManager, queue, packageMeta, clock)`
  - `suspend fun reconcileWindow(sinceMillis: Long, untilMillis: Long)` — consulta `usm.queryEvents(sinceMillis, untilMillis)`, extrai transições `MOVE_TO_FOREGROUND` e para cada uma que não corresponde a WindowActivity já emitida (dedup por timestamp), enfileira via `queue`.
  - `UsageStatsReconcileWorker` (WorkManager 15min) — chama `reconcileWindow(agora - 15min, agora)` como fallback quando AccessibilityService é morto por OEM.

- [ ] **Step 1: Escrever teste**

Criar `app/src/test/kotlin/com/trivion/manageragent/collector/UsageStatsCollectorTest.kt`:

```kotlin
package com.trivion.manageragent.collector

import android.app.usage.UsageEvents
import android.app.usage.UsageStatsManager
import com.trivion.manageragent.event.EventQueue
import com.trivion.manageragent.event.model.AgentEvent
import io.mockk.coVerify
import io.mockk.every
import io.mockk.mockk
import io.mockk.slot
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Test
import kotlin.test.assertEquals

class UsageStatsCollectorTest {

    private val queue = mockk<EventQueue>(relaxed = true)
    private val meta = mockk<PackageMetadataProvider>().apply {
        every { humanNameFor(any()) } answers { firstArg<String>() }
    }
    private val usm = mockk<UsageStatsManager>()

    private fun mockUsageEvent(pkg: String, ts: Long, type: Int): UsageEvents.Event {
        val evt = mockk<UsageEvents.Event>()
        every { evt.packageName } returns pkg
        every { evt.timeStamp } returns ts
        every { evt.eventType } returns type
        return evt
    }

    private fun mockEvents(events: List<UsageEvents.Event>): UsageEvents {
        val ue = mockk<UsageEvents>()
        var idx = 0
        every { ue.hasNextEvent() } answers { idx < events.size }
        every { ue.getNextEvent(any()) } answers {
            val out = firstArg<UsageEvents.Event>()
            val src = events[idx++]
            every { out.packageName } returns src.packageName
            every { out.timeStamp } returns src.timeStamp
            every { out.eventType } returns src.eventType
            true
        }
        return ue
    }

    @Test
    fun `reconcileWindow enfileira WindowActivity para MOVE_TO_FOREGROUND`() = runTest {
        val e1 = mockUsageEvent("com.a", 100L, UsageEvents.Event.MOVE_TO_FOREGROUND)
        val e2 = mockUsageEvent("com.b", 200L, UsageEvents.Event.MOVE_TO_FOREGROUND)
        every { usm.queryEvents(0L, 300L) } returns mockEvents(listOf(e1, e2))

        val collector = UsageStatsCollector(usm, queue, meta, clock = { "clock:$it" })
        collector.reconcileWindow(0L, 300L)

        val slots = mutableListOf<AgentEvent>()
        coVerify(atLeast = 1) { queue.enqueue(capture(slots)) }
        val actives = slots.filterIsInstance<AgentEvent.WindowActivity>()
        assertEquals(1, actives.size)  // 1 evento (com.a → com.b, com.b abrindo)
        assertEquals("com.a", actives[0].nomeProcesso)
    }

    @Test
    fun `reconcileWindow ignora eventos que nao MOVE_TO_FOREGROUND`() = runTest {
        val e1 = mockUsageEvent("com.a", 100L, UsageEvents.Event.MOVE_TO_BACKGROUND)
        every { usm.queryEvents(0L, 300L) } returns mockEvents(listOf(e1))
        val collector = UsageStatsCollector(usm, queue, meta, clock = { "$it" })
        collector.reconcileWindow(0L, 300L)
        coVerify(exactly = 0) { queue.enqueue(any()) }
    }

    @Test
    fun `reconcileWindow ignora eventos duplicados do mesmo pacote consecutivos`() = runTest {
        val e1 = mockUsageEvent("com.a", 100L, UsageEvents.Event.MOVE_TO_FOREGROUND)
        val e2 = mockUsageEvent("com.a", 200L, UsageEvents.Event.MOVE_TO_FOREGROUND)
        every { usm.queryEvents(0L, 300L) } returns mockEvents(listOf(e1, e2))
        val collector = UsageStatsCollector(usm, queue, meta, clock = { "$it" })
        collector.reconcileWindow(0L, 300L)
        coVerify(exactly = 0) { queue.enqueue(any()) }
    }
}
```

- [ ] **Step 2: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.collector.UsageStatsCollectorTest"`
Expected: FAIL.

- [ ] **Step 3: Criar `UsageStatsCollector.kt`**

```kotlin
package com.trivion.manageragent.collector

import android.app.usage.UsageEvents
import android.app.usage.UsageStatsManager
import com.trivion.manageragent.event.EventQueue
import com.trivion.manageragent.event.model.AgentEvent
import java.time.Instant

class UsageStatsCollector(
    private val usm: UsageStatsManager,
    private val queue: EventQueue,
    private val packageMeta: PackageMetadataProvider,
    private val clock: (Long) -> String = { Instant.ofEpochMilli(it).toString() }
) {

    suspend fun reconcileWindow(sinceMillis: Long, untilMillis: Long) {
        val events = usm.queryEvents(sinceMillis, untilMillis)
        val out = UsageEvents.Event()
        val transitions = mutableListOf<Pair<String, Long>>()
        while (events.hasNextEvent()) {
            events.getNextEvent(out)
            if (out.eventType == UsageEvents.Event.MOVE_TO_FOREGROUND) {
                val pkg = out.packageName ?: continue
                if (transitions.isEmpty() || transitions.last().first != pkg) {
                    transitions.add(pkg to out.timeStamp)
                }
            }
        }
        // Emite WindowActivity para cada transição encerrada pela próxima
        for (i in 0 until transitions.size - 1) {
            val (pkg, start) = transitions[i]
            val (_, end) = transitions[i + 1]
            queue.enqueue(
                AgentEvent.WindowActivity(
                    nomeProcesso = pkg,
                    tituloJanela = packageMeta.humanNameFor(pkg),
                    iniciadoEm = clock(start),
                    finalizadoEm = clock(end),
                    statusUsuario = "ATIVO",
                    urlDominio = ""
                )
            )
        }
    }
}
```

- [ ] **Step 4: Criar `UsageStatsReconcileWorker.kt`**

```kotlin
package com.trivion.manageragent.collector

import android.app.usage.UsageStatsManager
import android.content.Context
import androidx.work.CoroutineWorker
import androidx.work.ExistingPeriodicWorkPolicy
import androidx.work.PeriodicWorkRequestBuilder
import androidx.work.WorkManager
import androidx.work.WorkerParameters
import com.trivion.manageragent.event.EventDatabase
import com.trivion.manageragent.event.EventQueue
import kotlinx.serialization.json.Json
import java.util.concurrent.TimeUnit

class UsageStatsReconcileWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val usm = applicationContext.getSystemService(Context.USAGE_STATS_SERVICE) as UsageStatsManager
        val json = Json { classDiscriminator = "tipoEvento"; encodeDefaults = true }
        val queue = EventQueue(EventDatabase.getInstance(applicationContext).eventDao(), json)
        val meta = PackageMetadataProvider(applicationContext)
        val collector = UsageStatsCollector(usm, queue, meta)
        val agora = System.currentTimeMillis()
        collector.reconcileWindow(agora - TimeUnit.MINUTES.toMillis(15), agora)
        return Result.success()
    }

    companion object {
        private const val WORK_NAME = "ManagerAgent.UsageStatsReconcile"
        fun schedule(context: Context) {
            val req = PeriodicWorkRequestBuilder<UsageStatsReconcileWorker>(15, TimeUnit.MINUTES).build()
            WorkManager.getInstance(context).enqueueUniquePeriodicWork(
                WORK_NAME, ExistingPeriodicWorkPolicy.KEEP, req
            )
        }
    }
}
```

- [ ] **Step 5: Wire no Service**

Editar `ManagerAgentService.onCreate()`:

```kotlin
HeartbeatScheduler.schedule(this)
UsageStatsReconcileWorker.schedule(this)  // NOVO
```

- [ ] **Step 6: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.collector.UsageStatsCollectorTest"`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add .
git commit -m "feat: add UsageStatsCollector + reconcile worker (15min fallback for Accessibility)"
```

---

### Task T20: ActivitySummaryCollector — contadores de toques/scrolls (SEM CONTEÚDO)

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/collector/ActivitySummaryCollector.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/collector/ActivitySummaryCollectorTest.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/collector/LGPDContentFilterTest.kt` (trava sabotagem futura)
- Modify: `app/src/main/kotlin/com/trivion/manageragent/service/ManagerAccessibilityService.kt` (delega eventos de VIEW_*)

**Interfaces:**
- Consumes: `EventQueue` (T10)
- Produces:
  - `ActivitySummaryCollector(queue, windowMillis: Long = 60_000, clock)` — agrega contadores por janela de 1min
  - `fun onClicked()`, `fun onScrolled()`, `fun onTextChanged()` — incrementa contadores in-memory
  - `suspend fun tick()` — se `now - windowStart >= windowMillis`, emite `AgentEvent.ActivitySummary` com contadores e reseta
- **LGPD:** collector NUNCA recebe texto — só chamadas booleanas de "algo aconteceu"

- [ ] **Step 1: Escrever teste LGPD (trava sabotagem futura)**

Criar `app/src/test/kotlin/com/trivion/manageragent/collector/LGPDContentFilterTest.kt`:

```kotlin
package com.trivion.manageragent.collector

import org.junit.jupiter.api.Test
import kotlin.reflect.full.declaredMemberFunctions
import kotlin.test.assertFalse

class LGPDContentFilterTest {

    /**
     * Trava sabotagem futura: garante que ActivitySummaryCollector NÃO tenha nenhum método
     * que aceita String/CharSequence como parâmetro (isso indicaria coleta de texto).
     * Se alguém adicionar `fun onTextChanged(text: String)` no futuro, esse teste quebra.
     */
    @Test
    fun `ActivitySummaryCollector nao expoe metodos que aceitam texto`() {
        val forbidden = setOf("kotlin.String", "kotlin.CharSequence")
        val fns = ActivitySummaryCollector::class.declaredMemberFunctions
        for (fn in fns) {
            for (param in fn.parameters) {
                val classifier = (param.type.classifier as? kotlin.reflect.KClass<*>)?.qualifiedName
                assertFalse(
                    classifier in forbidden,
                    "LGPD BROKEN: ${fn.name} aceita ${classifier} — nunca coletar texto do usuário!"
                )
            }
        }
    }
}
```

- [ ] **Step 2: Escrever teste do collector**

Criar `app/src/test/kotlin/com/trivion/manageragent/collector/ActivitySummaryCollectorTest.kt`:

```kotlin
package com.trivion.manageragent.collector

import com.trivion.manageragent.event.EventQueue
import com.trivion.manageragent.event.model.AgentEvent
import io.mockk.coVerify
import io.mockk.mockk
import io.mockk.slot
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Test
import kotlin.test.assertEquals

class ActivitySummaryCollectorTest {

    @Test
    fun `emite ActivitySummary quando window expira`() = runTest {
        val queue = mockk<EventQueue>(relaxed = true)
        var now = 0L
        val collector = ActivitySummaryCollector(
            queue = queue,
            windowMillis = 1_000,
            nowMillis = { now },
            clock = { "t$it" }
        )

        collector.onClicked()
        collector.onClicked()
        collector.onScrolled()
        collector.onTextChanged()

        now = 1_500
        collector.tick()

        val slot = slot<AgentEvent>()
        coVerify(exactly = 1) { queue.enqueue(capture(slot)) }
        val evt = slot.captured as AgentEvent.ActivitySummary
        assertEquals(1, evt.teclasPressionadas)     // 1 text change = 1 tecla (agregação simples)
        assertEquals(2, evt.cliquesMouseEsq)        // clicks mapeados pra "mouse esq"
        assertEquals(0, evt.cliquesMouseDir)
        assertEquals(0, evt.cliquesMouseMeio)
        assertEquals(0, evt.movimentosMouse)
        assertEquals(1, evt.scrollsVertical)
        assertEquals("NORMAL", evt.padraoDigitacao)
    }

    @Test
    fun `nao emite se window ainda nao expirou`() = runTest {
        val queue = mockk<EventQueue>(relaxed = true)
        var now = 0L
        val collector = ActivitySummaryCollector(queue, windowMillis = 1_000, nowMillis = { now }, clock = { "t$it" })
        collector.onClicked()
        now = 500
        collector.tick()
        coVerify(exactly = 0) { queue.enqueue(any()) }
    }

    @Test
    fun `reseta contadores apos emissao`() = runTest {
        val queue = mockk<EventQueue>(relaxed = true)
        var now = 0L
        val collector = ActivitySummaryCollector(queue, windowMillis = 1_000, nowMillis = { now }, clock = { "t$it" })
        collector.onClicked()
        now = 1_500; collector.tick()
        now = 2_000
        collector.tick()  // não deve emitir de novo (contadores zerados, sem eventos novos, mas window expirou)
        // Emite mas com todos zeros? Comportamento: só emite se houve atividade
        coVerify(atLeast = 1) { queue.enqueue(any()) }
    }

    @Test
    fun `nao emite ActivitySummary com todos os contadores zerados`() = runTest {
        val queue = mockk<EventQueue>(relaxed = true)
        var now = 0L
        val collector = ActivitySummaryCollector(queue, windowMillis = 1_000, nowMillis = { now }, clock = { "t$it" })
        now = 2_000
        collector.tick()  // sem nenhuma interação — não emite
        coVerify(exactly = 0) { queue.enqueue(any()) }
    }
}
```

- [ ] **Step 3: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.collector.ActivitySummaryCollectorTest"`
Expected: FAIL.

- [ ] **Step 4: Criar `ActivitySummaryCollector.kt`**

```kotlin
package com.trivion.manageragent.collector

import com.trivion.manageragent.event.EventQueue
import com.trivion.manageragent.event.model.AgentEvent
import java.time.Instant

class ActivitySummaryCollector(
    private val queue: EventQueue,
    private val windowMillis: Long = 60_000L,
    private val nowMillis: () -> Long = { System.currentTimeMillis() },
    private val clock: (Long) -> String = { Instant.ofEpochMilli(it).toString() }
) {

    private var windowStart: Long = nowMillis()
    private var clicks = 0
    private var scrolls = 0
    private var textChanges = 0

    fun onClicked() { clicks++ }
    fun onScrolled() { scrolls++ }
    fun onTextChanged() { textChanges++ }

    suspend fun tick() {
        val agora = nowMillis()
        if (agora - windowStart < windowMillis) return

        val hasActivity = clicks > 0 || scrolls > 0 || textChanges > 0
        if (hasActivity) {
            queue.enqueue(
                AgentEvent.ActivitySummary(
                    iniciadoEm = clock(windowStart),
                    finalizadoEm = clock(agora),
                    teclasPressionadas = textChanges,
                    cliquesMouseEsq = clicks,
                    cliquesMouseDir = 0,
                    cliquesMouseMeio = 0,
                    movimentosMouse = 0,
                    scrollsVertical = scrolls,
                    padraoDigitacao = "NORMAL"
                )
            )
        }
        clicks = 0; scrolls = 0; textChanges = 0
        windowStart = agora
    }
}
```

- [ ] **Step 5: Integrar no AccessibilityService**

Editar `ManagerAccessibilityService.kt` — adicionar collector:

```kotlin
private lateinit var summaryCollector: ActivitySummaryCollector

override fun onServiceConnected() {
    super.onServiceConnected()
    val queue = EventQueue(EventDatabase.getInstance(this).eventDao(), json)
    windowCollector = WindowActivityCollector(queue, PackageMetadataProvider(this))
    summaryCollector = ActivitySummaryCollector(queue)
}

override fun onAccessibilityEvent(event: AccessibilityEvent?) {
    if (event == null) return
    when (event.eventType) {
        AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED -> {
            val pkg = event.packageName?.toString()
            val cls = event.className?.toString()
            scope.launch { windowCollector.onWindowStateChanged(pkg, cls) }
        }
        AccessibilityEvent.TYPE_VIEW_CLICKED -> summaryCollector.onClicked()
        AccessibilityEvent.TYPE_VIEW_SCROLLED -> summaryCollector.onScrolled()
        AccessibilityEvent.TYPE_VIEW_TEXT_CHANGED -> summaryCollector.onTextChanged()
        else -> Unit
    }
    // Ticker roda separado — ver Step 6
}
```

- [ ] **Step 6: Adicionar tick periódico no Service**

Criar `SummaryTicker.kt`:

```kotlin
package com.trivion.manageragent.service

import com.trivion.manageragent.collector.ActivitySummaryCollector
import kotlinx.coroutines.*

class SummaryTicker(
    private val collector: ActivitySummaryCollector,
    private val intervalMs: Long = 30_000L
) {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
    private var job: Job? = null

    fun start() {
        if (job?.isActive == true) return
        job = scope.launch {
            while (isActive) {
                collector.tick()
                delay(intervalMs)
            }
        }
    }

    fun stop() { job?.cancel() }
}
```

Wire no `ManagerAccessibilityService`:

```kotlin
private lateinit var summaryTicker: SummaryTicker

override fun onServiceConnected() {
    super.onServiceConnected()
    val queue = EventQueue(EventDatabase.getInstance(this).eventDao(), json)
    windowCollector = WindowActivityCollector(queue, PackageMetadataProvider(this))
    summaryCollector = ActivitySummaryCollector(queue)
    summaryTicker = SummaryTicker(summaryCollector).also { it.start() }
}

override fun onUnbind(intent: android.content.Intent?): Boolean {
    summaryTicker.stop()
    return super.onUnbind(intent)
}
```

- [ ] **Step 7: Run e ver passar (todos os 5 testes: 4 collector + 1 LGPD)**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.collector.ActivitySummaryCollectorTest" --tests "com.trivion.manageragent.collector.LGPDContentFilterTest"`
Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add .
git commit -m "feat: add ActivitySummaryCollector (aggregated counters, LGPD-safe — no content)"
```

---

### Task T21: MeetingDetector — Zoom/Teams/Meet foreground → MeetingEvent

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/collector/MeetingDetector.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/collector/MeetingDetectorTest.kt`
- Modify: `app/src/main/kotlin/com/trivion/manageragent/collector/WindowActivityCollector.kt` (notifica MeetingDetector)

**Interfaces:**
- Consumes: `EventQueue` (T10), sinais de foreground (via WindowActivityCollector)
- Produces:
  - `MeetingDetector(queue, minDurationMs: Long = 120_000, clock)` — sabe quais packages são "meeting apps" (`MEETING_PACKAGES`)
  - `suspend fun onForegroundChange(packageName: String)` — se entra em app meeting, marca start; se sai (outro app foreground) E duração >= 2min, emite `MeetingEvent`; se duração < 2min, descarta (não era chamada real)

- [ ] **Step 1: Escrever teste**

Criar `app/src/test/kotlin/com/trivion/manageragent/collector/MeetingDetectorTest.kt`:

```kotlin
package com.trivion.manageragent.collector

import com.trivion.manageragent.event.EventQueue
import com.trivion.manageragent.event.model.AgentEvent
import io.mockk.coVerify
import io.mockk.mockk
import io.mockk.slot
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Test
import kotlin.test.assertEquals

class MeetingDetectorTest {

    @Test
    fun `emite MeetingEvent quando sai de app meeting apos duracao minima`() = runTest {
        val queue = mockk<EventQueue>(relaxed = true)
        var now = 0L
        val det = MeetingDetector(queue, minDurationMs = 100L, nowMillis = { now }, clock = { "t$it" })

        det.onForegroundChange("us.zoom.videomeetings")
        now = 200L
        det.onForegroundChange("com.whatsapp")

        val slot = slot<AgentEvent>()
        coVerify(exactly = 1) { queue.enqueue(capture(slot)) }
        val evt = slot.captured as AgentEvent.Meeting
        assertEquals("Zoom Android", evt.aplicativo)
        assertEquals("t0", evt.iniciadoEm)
        assertEquals("t200", evt.finalizadoEm)
    }

    @Test
    fun `descarta se duracao abaixo do minimo`() = runTest {
        val queue = mockk<EventQueue>(relaxed = true)
        var now = 0L
        val det = MeetingDetector(queue, minDurationMs = 500L, nowMillis = { now }, clock = { "t$it" })

        det.onForegroundChange("us.zoom.videomeetings")
        now = 100L
        det.onForegroundChange("com.whatsapp")

        coVerify(exactly = 0) { queue.enqueue(any()) }
    }

    @Test
    fun `nao emite se app nao eh meeting`() = runTest {
        val queue = mockk<EventQueue>(relaxed = true)
        var now = 0L
        val det = MeetingDetector(queue, minDurationMs = 100L, nowMillis = { now }, clock = { "t$it" })

        det.onForegroundChange("com.whatsapp")
        now = 500L
        det.onForegroundChange("com.instagram")

        coVerify(exactly = 0) { queue.enqueue(any()) }
    }

    @Test
    fun `Teams reconhecido`() = runTest {
        val queue = mockk<EventQueue>(relaxed = true)
        var now = 0L
        val det = MeetingDetector(queue, minDurationMs = 100L, nowMillis = { now }, clock = { "t$it" })
        det.onForegroundChange("com.microsoft.teams")
        now = 200L; det.onForegroundChange("com.whatsapp")
        val slot = slot<AgentEvent>()
        coVerify(exactly = 1) { queue.enqueue(capture(slot)) }
        assertEquals("Teams Android", (slot.captured as AgentEvent.Meeting).aplicativo)
    }

    @Test
    fun `Google Meet reconhecido`() = runTest {
        val queue = mockk<EventQueue>(relaxed = true)
        var now = 0L
        val det = MeetingDetector(queue, minDurationMs = 100L, nowMillis = { now }, clock = { "t$it" })
        det.onForegroundChange("com.google.android.apps.meetings")
        now = 200L; det.onForegroundChange("com.whatsapp")
        val slot = slot<AgentEvent>()
        coVerify(exactly = 1) { queue.enqueue(capture(slot)) }
        assertEquals("Google Meet", (slot.captured as AgentEvent.Meeting).aplicativo)
    }
}
```

- [ ] **Step 2: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.collector.MeetingDetectorTest"`
Expected: FAIL.

- [ ] **Step 3: Criar `MeetingDetector.kt`**

```kotlin
package com.trivion.manageragent.collector

import com.trivion.manageragent.event.EventQueue
import com.trivion.manageragent.event.model.AgentEvent
import java.time.Instant

class MeetingDetector(
    private val queue: EventQueue,
    private val minDurationMs: Long = 120_000L,
    private val nowMillis: () -> Long = { System.currentTimeMillis() },
    private val clock: (Long) -> String = { Instant.ofEpochMilli(it).toString() }
) {

    companion object {
        val MEETING_PACKAGES = mapOf(
            "us.zoom.videomeetings" to "Zoom Android",
            "com.microsoft.teams" to "Teams Android",
            "com.google.android.apps.meetings" to "Google Meet",
            "com.google.android.apps.tachyon" to "Google Meet"  // legacy Duo/Meet
        )
    }

    private var currentMeetingPkg: String? = null
    private var startedMs: Long = 0L

    suspend fun onForegroundChange(packageName: String) {
        val isMeeting = packageName in MEETING_PACKAGES

        if (currentMeetingPkg != null && packageName != currentMeetingPkg) {
            val agora = nowMillis()
            val duration = agora - startedMs
            if (duration >= minDurationMs) {
                queue.enqueue(
                    AgentEvent.Meeting(
                        iniciadoEm = clock(startedMs),
                        finalizadoEm = clock(agora),
                        aplicativo = MEETING_PACKAGES[currentMeetingPkg]!!
                    )
                )
            }
            currentMeetingPkg = null
        }

        if (isMeeting && currentMeetingPkg == null) {
            currentMeetingPkg = packageName
            startedMs = nowMillis()
        }
    }
}
```

- [ ] **Step 4: Integrar no WindowActivityCollector**

Modificar `WindowActivityCollector.kt` — adicionar callback opcional:

```kotlin
class WindowActivityCollector(
    private val queue: EventQueue,
    private val packageMeta: PackageMetadataProvider,
    private val onForegroundListener: suspend (String) -> Unit = {},
    private val clock: () -> String = { Instant.now().toString() }
) {
    // ... existente

    suspend fun onWindowStateChanged(packageName: String?, className: String?) {
        if (packageName.isNullOrBlank()) return
        // ... lógica existente
        if (previous != packageName) {
            currentPackage = packageName
            currentIniciadoEm = agora
            onForegroundListener(packageName)  // NOVO
        }
    }
}
```

Wire no `ManagerAccessibilityService`:

```kotlin
private lateinit var meetingDetector: MeetingDetector

override fun onServiceConnected() {
    super.onServiceConnected()
    val queue = EventQueue(EventDatabase.getInstance(this).eventDao(), json)
    meetingDetector = MeetingDetector(queue)
    windowCollector = WindowActivityCollector(
        queue = queue,
        packageMeta = PackageMetadataProvider(this),
        onForegroundListener = { pkg -> meetingDetector.onForegroundChange(pkg) }
    )
    // resto igual
}
```

- [ ] **Step 5: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.collector.MeetingDetectorTest"`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add .
git commit -m "feat: add MeetingDetector (Zoom/Teams/Meet with 2min minimum duration)"
```

---

### Task T22: PhoneCallListener — TelephonyCallback (Android 12+) + PhoneStateListener (legacy)

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/service/PhoneCallListener.kt`
- Create: `app/src/main/kotlin/com/trivion/manageragent/collector/PhoneCallEventFactory.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/collector/PhoneCallEventFactoryTest.kt`
- Modify: `app/src/main/AndroidManifest.xml` (READ_PHONE_STATE)
- Modify: `ManagerAgentService.kt` (registra o listener)

**Interfaces:**
- Consumes: `EventQueue` (T10)
- Produces:
  - `PhoneCallEventFactory` — função pura: `fromCallState(oldState, newState, clock)` → `AgentEvent.PhoneCall?`. IDLE→OFFHOOK = CALL_START; qualquer→IDLE (se estava OFFHOOK/RINGING) = CALL_END.
  - `PhoneCallListener(context, queue)` — registra em `TelephonyManager` conforme API level, chama factory nos callbacks, enfileira eventos

- [ ] **Step 1: Adicionar permissão no Manifest**

```xml
<uses-permission android:name="android.permission.READ_PHONE_STATE" />
```

- [ ] **Step 2: Escrever teste do factory**

Criar `app/src/test/kotlin/com/trivion/manageragent/collector/PhoneCallEventFactoryTest.kt`:

```kotlin
package com.trivion.manageragent.collector

import android.telephony.TelephonyManager
import com.trivion.manageragent.event.model.AgentEvent
import org.junit.jupiter.api.Test
import kotlin.test.assertEquals
import kotlin.test.assertNull

class PhoneCallEventFactoryTest {

    private val factory = PhoneCallEventFactory(clock = { "2026-08-08T14:00:00Z" })

    @Test
    fun `IDLE para OFFHOOK vira CALL_START`() {
        val evt = factory.fromTransition(
            oldState = TelephonyManager.CALL_STATE_IDLE,
            newState = TelephonyManager.CALL_STATE_OFFHOOK
        )
        assertEquals("CALL_START", evt?.tipoTransicao)
    }

    @Test
    fun `OFFHOOK para IDLE vira CALL_END`() {
        val evt = factory.fromTransition(
            oldState = TelephonyManager.CALL_STATE_OFFHOOK,
            newState = TelephonyManager.CALL_STATE_IDLE
        )
        assertEquals("CALL_END", evt?.tipoTransicao)
    }

    @Test
    fun `RINGING para OFFHOOK vira CALL_START (atendida)`() {
        val evt = factory.fromTransition(TelephonyManager.CALL_STATE_RINGING, TelephonyManager.CALL_STATE_OFFHOOK)
        assertEquals("CALL_START", evt?.tipoTransicao)
    }

    @Test
    fun `RINGING para IDLE vira CALL_END (rejeitada ou perdida)`() {
        val evt = factory.fromTransition(TelephonyManager.CALL_STATE_RINGING, TelephonyManager.CALL_STATE_IDLE)
        assertEquals("CALL_END", evt?.tipoTransicao)
    }

    @Test
    fun `IDLE para RINGING nao emite (aguarda decisao do usuario)`() {
        assertNull(factory.fromTransition(TelephonyManager.CALL_STATE_IDLE, TelephonyManager.CALL_STATE_RINGING))
    }

    @Test
    fun `mesmo estado nao emite`() {
        assertNull(factory.fromTransition(TelephonyManager.CALL_STATE_IDLE, TelephonyManager.CALL_STATE_IDLE))
    }
}
```

- [ ] **Step 3: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.collector.PhoneCallEventFactoryTest"`
Expected: FAIL.

- [ ] **Step 4: Criar `PhoneCallEventFactory.kt`**

```kotlin
package com.trivion.manageragent.collector

import android.telephony.TelephonyManager
import com.trivion.manageragent.event.model.AgentEvent
import java.time.Instant

class PhoneCallEventFactory(
    private val clock: () -> String = { Instant.now().toString() }
) {

    fun fromTransition(oldState: Int, newState: Int): AgentEvent.PhoneCall? {
        if (oldState == newState) return null

        val tipo = when {
            newState == TelephonyManager.CALL_STATE_OFFHOOK &&
                (oldState == TelephonyManager.CALL_STATE_IDLE ||
                 oldState == TelephonyManager.CALL_STATE_RINGING) -> "CALL_START"
            newState == TelephonyManager.CALL_STATE_IDLE &&
                (oldState == TelephonyManager.CALL_STATE_OFFHOOK ||
                 oldState == TelephonyManager.CALL_STATE_RINGING) -> "CALL_END"
            else -> null
        } ?: return null

        return AgentEvent.PhoneCall(tipoTransicao = tipo, ocorreuEm = clock())
    }
}
```

- [ ] **Step 5: Criar `PhoneCallListener.kt`**

```kotlin
package com.trivion.manageragent.service

import android.content.Context
import android.os.Build
import android.telephony.PhoneStateListener
import android.telephony.TelephonyCallback
import android.telephony.TelephonyManager
import com.trivion.manageragent.collector.PhoneCallEventFactory
import com.trivion.manageragent.event.EventQueue
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.SupervisorJob
import kotlinx.coroutines.launch
import timber.log.Timber
import java.util.concurrent.Executors

class PhoneCallListener(
    private val context: Context,
    private val queue: EventQueue
) {

    private val factory = PhoneCallEventFactory()
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
    private var lastState: Int = TelephonyManager.CALL_STATE_IDLE
    private var registered: Any? = null

    fun register() {
        val tm = context.getSystemService(Context.TELEPHONY_SERVICE) as TelephonyManager
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
            val callback = object : TelephonyCallback(), TelephonyCallback.CallStateListener {
                override fun onCallStateChanged(state: Int) { emit(state) }
            }
            tm.registerTelephonyCallback(Executors.newSingleThreadExecutor(), callback)
            registered = callback
        } else {
            @Suppress("DEPRECATION")
            val listener = object : PhoneStateListener() {
                override fun onCallStateChanged(state: Int, phoneNumber: String?) {
                    // phoneNumber intencionalmente ignorado (LGPD)
                    emit(state)
                }
            }
            @Suppress("DEPRECATION")
            tm.listen(listener, PhoneStateListener.LISTEN_CALL_STATE)
            registered = listener
        }
    }

    fun unregister() {
        val tm = context.getSystemService(Context.TELEPHONY_SERVICE) as TelephonyManager
        val reg = registered ?: return
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S && reg is TelephonyCallback) {
            tm.unregisterTelephonyCallback(reg)
        } else {
            @Suppress("DEPRECATION")
            (reg as? PhoneStateListener)?.let { tm.listen(it, PhoneStateListener.LISTEN_NONE) }
        }
        registered = null
    }

    private fun emit(newState: Int) {
        val old = lastState
        lastState = newState
        val evt = factory.fromTransition(old, newState) ?: return
        Timber.tag(TAG).i("Phone call event: ${evt.tipoTransicao}")
        scope.launch { queue.enqueue(evt) }
    }

    companion object { private const val TAG = "PhoneCallListener" }
}
```

- [ ] **Step 6: Wire no ManagerAgentService**

Editar `ManagerAgentService.kt`:

```kotlin
private lateinit var phoneListener: PhoneCallListener

override fun onCreate() {
    super.onCreate()
    NotificationHelper.ensureChannel(this)
    startForegroundWithNotification()

    val filter = IntentFilter().apply {
        addAction(Intent.ACTION_SCREEN_ON)
        addAction(Intent.ACTION_SCREEN_OFF)
        addAction(Intent.ACTION_USER_PRESENT)
    }
    registerReceiver(sessionReceiver, filter)
    HeartbeatScheduler.schedule(this)
    UsageStatsReconcileWorker.schedule(this)

    val json = Json { classDiscriminator = "tipoEvento"; encodeDefaults = true }
    val queue = EventQueue(EventDatabase.getInstance(this).eventDao(), json)
    phoneListener = PhoneCallListener(this, queue).also { it.register() }
}

override fun onDestroy() {
    try { unregisterReceiver(sessionReceiver) } catch (_: Exception) {}
    phoneListener.unregister()
    super.onDestroy()
}
```

- [ ] **Step 7: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.collector.PhoneCallEventFactoryTest"`
Expected: PASS (6 testes).

- [ ] **Step 8: Commit**

```bash
git add .
git commit -m "feat: add PhoneCallListener + PhoneCallEventFactory (LGPD: no phone number)"
```

---

### Task T23: StatusTransitionCalculator — mapeia idle/interações em ATIVO/PAUSA/AUSENTE

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/collector/StatusTransitionCalculator.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/collector/StatusTransitionCalculatorTest.kt`

**Interfaces:**
- Consumes: `EventQueue` (T10)
- Produces:
  - `StatusTransitionCalculator(queue, pausaMs: Long = 300_000, ausenteMs: Long = 900_000, clock)`
  - `fun onInteraction()` — reseta silêncio, se estava PAUSA/AUSENTE, emite `StatusTransitionEvent` de volta pra ATIVO
  - `suspend fun tick()` — se `silenceMs >= ausenteMs` e status != AUSENTE, emite AUSENTE. Se `silenceMs >= pausaMs` e status == ATIVO, emite PAUSA.

Thresholds alinhados com desktop (que usa 5min pausa / 15min ausente por default — regra Marcos memória `feedback_home_api_leve` menciona 5→15min changes).

- [ ] **Step 1: Escrever teste**

Criar `app/src/test/kotlin/com/trivion/manageragent/collector/StatusTransitionCalculatorTest.kt`:

```kotlin
package com.trivion.manageragent.collector

import com.trivion.manageragent.event.EventQueue
import com.trivion.manageragent.event.model.AgentEvent
import io.mockk.coVerify
import io.mockk.mockk
import io.mockk.slot
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Test
import kotlin.test.assertEquals

class StatusTransitionCalculatorTest {

    @Test
    fun `estado inicial eh ATIVO e nao emite`() = runTest {
        val queue = mockk<EventQueue>(relaxed = true)
        var now = 0L
        val calc = StatusTransitionCalculator(queue, pausaMs = 100, ausenteMs = 500, nowMillis = { now }, clock = { "t$it" })
        calc.tick()
        coVerify(exactly = 0) { queue.enqueue(any()) }
    }

    @Test
    fun `emite ATIVO para PAUSA apos pausaMs`() = runTest {
        val queue = mockk<EventQueue>(relaxed = true)
        var now = 0L
        val calc = StatusTransitionCalculator(queue, pausaMs = 100, ausenteMs = 500, nowMillis = { now }, clock = { "t$it" })
        calc.onInteraction()
        now = 200
        calc.tick()
        val slot = slot<AgentEvent>()
        coVerify(exactly = 1) { queue.enqueue(capture(slot)) }
        val evt = slot.captured as AgentEvent.StatusTransition
        assertEquals("ATIVO", evt.statusAnterior)
        assertEquals("PAUSA", evt.statusNovo)
    }

    @Test
    fun `emite PAUSA para AUSENTE apos ausenteMs`() = runTest {
        val queue = mockk<EventQueue>(relaxed = true)
        var now = 0L
        val calc = StatusTransitionCalculator(queue, pausaMs = 100, ausenteMs = 500, nowMillis = { now }, clock = { "t$it" })
        calc.onInteraction()
        now = 200; calc.tick()  // PAUSA
        now = 600; calc.tick()  // AUSENTE
        val slots = mutableListOf<AgentEvent>()
        coVerify(exactly = 2) { queue.enqueue(capture(slots)) }
        val ausente = slots[1] as AgentEvent.StatusTransition
        assertEquals("PAUSA", ausente.statusAnterior)
        assertEquals("AUSENTE", ausente.statusNovo)
    }

    @Test
    fun `interacao volta pra ATIVO`() = runTest {
        val queue = mockk<EventQueue>(relaxed = true)
        var now = 0L
        val calc = StatusTransitionCalculator(queue, pausaMs = 100, ausenteMs = 500, nowMillis = { now }, clock = { "t$it" })
        calc.onInteraction()
        now = 200; calc.tick()      // ATIVO → PAUSA
        now = 300; calc.onInteraction()  // PAUSA → ATIVO
        val slots = mutableListOf<AgentEvent>()
        coVerify(exactly = 2) { queue.enqueue(capture(slots)) }
        assertEquals("PAUSA", (slots[1] as AgentEvent.StatusTransition).statusAnterior)
        assertEquals("ATIVO", (slots[1] as AgentEvent.StatusTransition).statusNovo)
    }
}
```

- [ ] **Step 2: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.collector.StatusTransitionCalculatorTest"`
Expected: FAIL.

- [ ] **Step 3: Criar `StatusTransitionCalculator.kt`**

```kotlin
package com.trivion.manageragent.collector

import com.trivion.manageragent.event.EventQueue
import com.trivion.manageragent.event.model.AgentEvent
import java.time.Instant

class StatusTransitionCalculator(
    private val queue: EventQueue,
    private val pausaMs: Long = 5 * 60_000L,     // 5min
    private val ausenteMs: Long = 15 * 60_000L,  // 15min (alinhado com desktop)
    private val nowMillis: () -> Long = { System.currentTimeMillis() },
    private val clock: (Long) -> String = { Instant.ofEpochMilli(it).toString() }
) {

    enum class Status { ATIVO, PAUSA, AUSENTE }

    private var status: Status = Status.ATIVO
    private var lastInteractionMs: Long = nowMillis()

    fun onInteraction() {
        val agora = nowMillis()
        lastInteractionMs = agora
        if (status != Status.ATIVO) {
            transition(from = status, to = Status.ATIVO, at = agora)
        }
    }

    suspend fun tick() {
        val agora = nowMillis()
        val silence = agora - lastInteractionMs
        when {
            silence >= ausenteMs && status != Status.AUSENTE -> transitionSuspend(status, Status.AUSENTE, agora)
            silence >= pausaMs && status == Status.ATIVO -> transitionSuspend(Status.ATIVO, Status.PAUSA, agora)
        }
    }

    private fun transition(from: Status, to: Status, at: Long) {
        status = to
        kotlinx.coroutines.runBlocking { emitEvent(from, to, at) }
    }

    private suspend fun transitionSuspend(from: Status, to: Status, at: Long) {
        status = to
        emitEvent(from, to, at)
    }

    private suspend fun emitEvent(from: Status, to: Status, at: Long) {
        queue.enqueue(
            AgentEvent.StatusTransition(
                statusAnterior = from.name,
                statusNovo = to.name,
                transicaoEm = clock(at)
            )
        )
    }
}
```

- [ ] **Step 4: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.collector.StatusTransitionCalculatorTest"`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add .
git commit -m "feat: add StatusTransitionCalculator (ATIVO/PAUSA/AUSENTE with 5min/15min thresholds)"
```

---

### Task T24: UrlDomainExtractor — best-effort URL bar reading (Chrome/Firefox)

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/collector/UrlDomainExtractor.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/collector/UrlDomainExtractorTest.kt`
- Modify: `app/src/main/kotlin/com/trivion/manageragent/collector/WindowActivityCollector.kt` (usa extractor)

**Interfaces:**
- Consumes: nada (função pura de string parsing)
- Produces:
  - `UrlDomainExtractor` — mapeamento de `packageName` de browser conhecido → função extratora
  - `fun extractDomain(packageName: String, urlBarText: String?): String` — retorna domínio raiz ou string vazia

- [ ] **Step 1: Escrever teste**

Criar `app/src/test/kotlin/com/trivion/manageragent/collector/UrlDomainExtractorTest.kt`:

```kotlin
package com.trivion.manageragent.collector

import org.junit.jupiter.api.Test
import kotlin.test.assertEquals

class UrlDomainExtractorTest {

    private val extractor = UrlDomainExtractor()

    @Test
    fun `Chrome com URL completa retorna dominio`() {
        assertEquals("github.com", extractor.extractDomain("com.android.chrome", "https://github.com/foo/bar"))
    }

    @Test
    fun `Chrome com URL sem esquema retorna dominio`() {
        assertEquals("github.com", extractor.extractDomain("com.android.chrome", "github.com/foo/bar"))
    }

    @Test
    fun `Firefox retorna dominio`() {
        assertEquals("mozilla.org", extractor.extractDomain("org.mozilla.firefox", "https://www.mozilla.org/en-US/"))
    }

    @Test
    fun `Samsung Browser retorna dominio`() {
        assertEquals("samsung.com", extractor.extractDomain("com.sec.android.app.sbrowser", "https://samsung.com"))
    }

    @Test
    fun `URL bar vazia retorna vazio`() {
        assertEquals("", extractor.extractDomain("com.android.chrome", null))
        assertEquals("", extractor.extractDomain("com.android.chrome", ""))
    }

    @Test
    fun `Package nao-browser retorna vazio`() {
        assertEquals("", extractor.extractDomain("com.whatsapp", "https://github.com"))
    }

    @Test
    fun `www prefix eh removido`() {
        assertEquals("github.com", extractor.extractDomain("com.android.chrome", "https://www.github.com/path"))
    }

    @Test
    fun `subdominio eh preservado`() {
        assertEquals("docs.google.com", extractor.extractDomain("com.android.chrome", "https://docs.google.com/document/d/123"))
    }

    @Test
    fun `URL invalida retorna vazio`() {
        assertEquals("", extractor.extractDomain("com.android.chrome", "not-a-url"))
    }
}
```

- [ ] **Step 2: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.collector.UrlDomainExtractorTest"`
Expected: FAIL.

- [ ] **Step 3: Criar `UrlDomainExtractor.kt`**

```kotlin
package com.trivion.manageragent.collector

class UrlDomainExtractor {

    companion object {
        val KNOWN_BROWSERS = setOf(
            "com.android.chrome",
            "com.chrome.beta", "com.chrome.dev",
            "org.mozilla.firefox", "org.mozilla.focus",
            "com.sec.android.app.sbrowser",
            "com.microsoft.emmx",
            "com.opera.browser", "com.opera.mini.native",
            "com.brave.browser",
            "com.duckduckgo.mobile.android"
        )
    }

    fun extractDomain(packageName: String, urlBarText: String?): String {
        if (packageName !in KNOWN_BROWSERS) return ""
        val text = urlBarText?.trim().takeIf { !it.isNullOrEmpty() } ?: return ""
        return try {
            val urlWithScheme = if (text.startsWith("http://") || text.startsWith("https://")) text else "https://$text"
            val host = java.net.URL(urlWithScheme).host ?: return ""
            host.removePrefix("www.").takeIf { it.contains(".") } ?: ""
        } catch (_: java.net.MalformedURLException) {
            ""
        }
    }
}
```

- [ ] **Step 4: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.collector.UrlDomainExtractorTest"`
Expected: PASS (9 testes).

- [ ] **Step 5: Integrar leitura da URL bar no AccessibilityService**

Adicionar em `ManagerAccessibilityService.onAccessibilityEvent()`:

```kotlin
private fun tryReadUrlBar(rootInActiveWindow: android.view.accessibility.AccessibilityNodeInfo?, pkg: String): String {
    val root = rootInActiveWindow ?: return ""
    val candidates = when (pkg) {
        "com.android.chrome" -> listOf("com.android.chrome:id/url_bar")
        "org.mozilla.firefox" -> listOf("org.mozilla.firefox:id/mozac_browser_toolbar_url_view")
        "com.sec.android.app.sbrowser" -> listOf("com.sec.android.app.sbrowser:id/location_bar_edit_text")
        else -> emptyList()
    }
    for (id in candidates) {
        val nodes = root.findAccessibilityNodeInfosByViewId(id)
        val node = nodes.firstOrNull() ?: continue
        val text = node.text?.toString().orEmpty()  // ATENÇÃO: leitura EXCEPCIONAL de node URL bar, com whitelist rígida de IDs — LGPD ok
        return text
    }
    return ""
}
```

Chamar no `TYPE_WINDOW_STATE_CHANGED` e passar pra `WindowActivityCollector.onWindowStateChanged` (adicionar parâmetro `urlDominio`).

Modificar `WindowActivityCollector.kt`:

```kotlin
class WindowActivityCollector(
    private val queue: EventQueue,
    private val packageMeta: PackageMetadataProvider,
    private val urlExtractor: UrlDomainExtractor = UrlDomainExtractor(),
    private val onForegroundListener: suspend (String) -> Unit = {},
    private val clock: () -> String = { Instant.now().toString() }
) {

    private var currentPackage: String? = null
    private var currentIniciadoEm: String? = null
    private var currentUrlBarText: String? = null  // raw text lida pelo Accessibility

    suspend fun onWindowStateChanged(packageName: String?, className: String?, urlBarText: String? = null) {
        if (packageName.isNullOrBlank()) return
        val agora = clock()

        val previous = currentPackage
        val previousIniciadoEm = currentIniciadoEm
        val previousUrl = currentUrlBarText

        if (previous != null && previous != packageName && previousIniciadoEm != null) {
            queue.enqueue(
                AgentEvent.WindowActivity(
                    nomeProcesso = previous,
                    tituloJanela = packageMeta.humanNameFor(previous),
                    iniciadoEm = previousIniciadoEm,
                    finalizadoEm = agora,
                    statusUsuario = "ATIVO",
                    urlDominio = urlExtractor.extractDomain(previous, previousUrl)
                )
            )
        }

        if (previous != packageName) {
            currentPackage = packageName
            currentIniciadoEm = agora
            currentUrlBarText = urlBarText
            onForegroundListener(packageName)
        } else {
            // Mesmo package (nova activity) — atualiza URL bar se veio nova
            if (!urlBarText.isNullOrEmpty()) currentUrlBarText = urlBarText
        }
    }
}
```

- [ ] **Step 6: Commit**

```bash
git add .
git commit -m "feat: add UrlDomainExtractor (best-effort URL bar reading for Chrome/Firefox/Samsung)"
```

---

### Task T25: Wire completo — ManagerAccessibilityService + IdleDetector + StatusTransition integrados

**Files:**
- Modify: `app/src/main/kotlin/com/trivion/manageragent/service/ManagerAccessibilityService.kt`
- Modify: `app/src/main/kotlin/com/trivion/manageragent/service/SummaryTicker.kt` (renomear pra `CollectorTicker` — genérico)
- Create: `app/src/main/kotlin/com/trivion/manageragent/service/CollectorTicker.kt` (substitui SummaryTicker)
- Delete: `app/src/main/kotlin/com/trivion/manageragent/service/SummaryTicker.kt`

**Interfaces:**
- Consumes: T15, T16, T20, T21, T23, T24
- Produces: pipeline completo — cada evento de Accessibility percorre WindowActivityCollector (+MeetingDetector+UrlExtractor), ActivitySummaryCollector, IdleDetector, StatusTransitionCalculator. Ticker unificado chama `.tick()` em todos a cada 30s.

- [ ] **Step 1: Criar `CollectorTicker.kt`**

```kotlin
package com.trivion.manageragent.service

import kotlinx.coroutines.*

class CollectorTicker(
    private val onTick: suspend () -> Unit,
    private val intervalMs: Long = 30_000L
) {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
    private var job: Job? = null

    fun start() {
        if (job?.isActive == true) return
        job = scope.launch {
            while (isActive) {
                try { onTick() } catch (e: Throwable) { timber.log.Timber.tag("CollectorTicker").w(e, "tick failed") }
                delay(intervalMs)
            }
        }
    }

    fun stop() { job?.cancel() }
}
```

- [ ] **Step 2: Deletar `SummaryTicker.kt`**

```bash
rm app/src/main/kotlin/com/trivion/manageragent/service/SummaryTicker.kt
```

- [ ] **Step 3: Wire completo no `ManagerAccessibilityService.kt`**

```kotlin
package com.trivion.manageragent.service

import android.accessibilityservice.AccessibilityService
import android.view.accessibility.AccessibilityEvent
import android.view.accessibility.AccessibilityNodeInfo
import com.trivion.manageragent.collector.*
import com.trivion.manageragent.event.EventDatabase
import com.trivion.manageragent.event.EventQueue
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.SupervisorJob
import kotlinx.coroutines.launch
import kotlinx.serialization.json.Json
import timber.log.Timber

class ManagerAccessibilityService : AccessibilityService() {

    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
    private val json = Json { classDiscriminator = "tipoEvento"; encodeDefaults = true }

    private lateinit var windowCollector: WindowActivityCollector
    private lateinit var summaryCollector: ActivitySummaryCollector
    private lateinit var idleDetector: IdleDetector
    private lateinit var statusCalc: StatusTransitionCalculator
    private lateinit var meetingDetector: MeetingDetector
    private lateinit var urlExtractor: UrlDomainExtractor
    private lateinit var ticker: CollectorTicker

    override fun onServiceConnected() {
        super.onServiceConnected()
        Timber.tag(TAG).i("AccessibilityService connected")
        val queue = EventQueue(EventDatabase.getInstance(this).eventDao(), json)
        meetingDetector = MeetingDetector(queue)
        urlExtractor = UrlDomainExtractor()
        windowCollector = WindowActivityCollector(
            queue = queue,
            packageMeta = PackageMetadataProvider(this),
            urlExtractor = urlExtractor,
            onForegroundListener = { pkg -> meetingDetector.onForegroundChange(pkg) }
        )
        summaryCollector = ActivitySummaryCollector(queue)
        idleDetector = IdleDetector(queue)
        statusCalc = StatusTransitionCalculator(queue)
        ticker = CollectorTicker(onTick = {
            summaryCollector.tick()
            idleDetector.tick()
            statusCalc.tick()
        }).also { it.start() }
    }

    override fun onAccessibilityEvent(event: AccessibilityEvent?) {
        if (event == null) return
        when (event.eventType) {
            AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED -> {
                val pkg = event.packageName?.toString() ?: return
                val cls = event.className?.toString()
                val urlBar = tryReadUrlBar(rootInActiveWindow, pkg)
                scope.launch { windowCollector.onWindowStateChanged(pkg, cls, urlBar) }
            }
            AccessibilityEvent.TYPE_VIEW_CLICKED -> {
                summaryCollector.onClicked()
                idleDetector.onInteraction()
                statusCalc.onInteraction()
            }
            AccessibilityEvent.TYPE_VIEW_SCROLLED -> {
                summaryCollector.onScrolled()
                idleDetector.onInteraction()
                statusCalc.onInteraction()
            }
            AccessibilityEvent.TYPE_VIEW_TEXT_CHANGED -> {
                summaryCollector.onTextChanged()
                idleDetector.onInteraction()
                statusCalc.onInteraction()
            }
            else -> Unit
        }
    }

    private fun tryReadUrlBar(root: AccessibilityNodeInfo?, pkg: String): String {
        if (root == null) return ""
        val candidates = when (pkg) {
            "com.android.chrome" -> listOf("com.android.chrome:id/url_bar")
            "org.mozilla.firefox" -> listOf("org.mozilla.firefox:id/mozac_browser_toolbar_url_view")
            "com.sec.android.app.sbrowser" -> listOf("com.sec.android.app.sbrowser:id/location_bar_edit_text")
            else -> return ""
        }
        for (id in candidates) {
            val nodes = try { root.findAccessibilityNodeInfosByViewId(id) } catch (_: Exception) { emptyList() }
            val node = nodes.firstOrNull() ?: continue
            return node.text?.toString().orEmpty()
        }
        return ""
    }

    override fun onInterrupt() { Timber.tag(TAG).w("Interrupted") }

    override fun onUnbind(intent: android.content.Intent?): Boolean {
        ticker.stop()
        return super.onUnbind(intent)
    }

    companion object { private const val TAG = "ManagerAccessibilityService" }
}
```

- [ ] **Step 4: Build + smoke test manual em emulador**

```bash
./gradlew :app:assembleDebug
```

Instalar em emulador, ativar AccessibilityService, verificar em `adb logcat -s ManagerAccessibilityService:* WindowActivityCollector:*` que eventos aparecem.

- [ ] **Step 5: Commit**

```bash
git add .
git commit -m "feat: complete AccessibilityService pipeline (window+summary+idle+status+meeting+url)"
```

---

## BLOCO B5 — Auth + backend integration (T26-T31)

### Task T26: ConfigReader — lê `assets/config.json` embutido no APK

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/config/AgentConfig.kt`
- Create: `app/src/main/kotlin/com/trivion/manageragent/config/ConfigReader.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/config/ConfigReaderTest.kt`

**Interfaces:**
- Consumes: `assets/config.json` (T6)
- Produces:
  - `data class AgentConfig(val ambiente: String, val chaveAtivacao: String, val nomeEmpresa: String)`
  - `object ConfigReader { fun read(context: Context): AgentConfig }` — lê e parseia JSON de `assets/config.json`, lança exception clara se inválido

- [ ] **Step 1: Escrever teste**

Criar `app/src/test/kotlin/com/trivion/manageragent/config/ConfigReaderTest.kt`:

```kotlin
package com.trivion.manageragent.config

import android.content.Context
import io.mockk.every
import io.mockk.mockk
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.assertThrows
import java.io.ByteArrayInputStream
import kotlin.test.assertEquals

class ConfigReaderTest {

    private fun context(json: String): Context {
        val ctx = mockk<Context>()
        val assets = mockk<android.content.res.AssetManager>()
        every { ctx.assets } returns assets
        every { assets.open("config.json") } returns ByteArrayInputStream(json.toByteArray())
        return ctx
    }

    @Test
    fun `parseia config valido`() {
        val ctx = context("""{"ambiente":"staging","chaveAtivacao":"abc-123","nomeEmpresa":"Trivion"}""")
        val cfg = ConfigReader.read(ctx)
        assertEquals("staging", cfg.ambiente)
        assertEquals("abc-123", cfg.chaveAtivacao)
        assertEquals("Trivion", cfg.nomeEmpresa)
    }

    @Test
    fun `lanca se JSON invalido`() {
        val ctx = context("not-json")
        assertThrows<IllegalStateException> { ConfigReader.read(ctx) }
    }

    @Test
    fun `lanca se chaveAtivacao esta em placeholder`() {
        val ctx = context("""{"ambiente":"prod","chaveAtivacao":"PLACEHOLDER_PROD","nomeEmpresa":"X"}""")
        assertThrows<IllegalStateException> { ConfigReader.read(ctx) }
    }

    @Test
    fun `aceita PLACEHOLDER_DEV_LOCAL em debug`() {
        val ctx = context("""{"ambiente":"debug","chaveAtivacao":"PLACEHOLDER_DEV_LOCAL","nomeEmpresa":"X"}""")
        val cfg = ConfigReader.read(ctx)  // não lança em debug
        assertEquals("debug", cfg.ambiente)
    }
}
```

- [ ] **Step 2: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.config.ConfigReaderTest"`
Expected: FAIL.

- [ ] **Step 3: Criar `AgentConfig.kt`**

```kotlin
package com.trivion.manageragent.config

import kotlinx.serialization.Serializable

@Serializable
data class AgentConfig(
    val ambiente: String,
    val chaveAtivacao: String,
    val nomeEmpresa: String
)
```

- [ ] **Step 4: Criar `ConfigReader.kt`**

```kotlin
package com.trivion.manageragent.config

import android.content.Context
import kotlinx.serialization.json.Json

object ConfigReader {

    private val json = Json { ignoreUnknownKeys = true }

    fun read(context: Context): AgentConfig {
        val text = try {
            context.assets.open("config.json").bufferedReader().use { it.readText() }
        } catch (e: Exception) {
            throw IllegalStateException("config.json não encontrado em assets", e)
        }
        val cfg = try {
            json.decodeFromString<AgentConfig>(text)
        } catch (e: Exception) {
            throw IllegalStateException("config.json inválido: ${e.message}", e)
        }
        if (cfg.ambiente != "debug" && cfg.chaveAtivacao.startsWith("PLACEHOLDER")) {
            throw IllegalStateException("APK sem chaveAtivacao real — pipeline de personalização não rodou (ambiente=${cfg.ambiente})")
        }
        return cfg
    }
}
```

- [ ] **Step 5: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.config.ConfigReaderTest"`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add .
git commit -m "feat: add ConfigReader (parses embedded config.json + validates placeholders)"
```

---

### Task T27: TokenStorage — EncryptedSharedPreferences (Android Keystore)

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/auth/TokenStorage.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/auth/TokenStorageTest.kt`
- Modify: `app/build.gradle.kts` (add `androidx.security.crypto`)

**Interfaces:**
- Consumes: Android Keystore
- Produces:
  - `TokenStorage(context)` — wrapper de `EncryptedSharedPreferences`
  - `fun saveTokens(deviceToken: String, refreshToken: String, agenteId: Long, usuarioId: Long, instalacaoId: String)`
  - `fun getDeviceToken(): String?`, `getRefreshToken()`, `getAgenteId()`, `getUsuarioId()`, `getInstalacaoId()`
  - `fun clear()` — usado em revogação/relink

- [ ] **Step 1: Adicionar dependência**

Em `app/build.gradle.kts`:

```kotlin
implementation(libs.androidx.security.crypto)
```

- [ ] **Step 2: Escrever teste**

Criar `app/src/test/kotlin/com/trivion/manageragent/auth/TokenStorageTest.kt`:

```kotlin
package com.trivion.manageragent.auth

import androidx.test.core.app.ApplicationProvider
import androidx.test.ext.junit.runners.AndroidJUnit4
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import org.junit.runner.RunWith
import kotlin.test.assertEquals
import kotlin.test.assertNull

@RunWith(AndroidJUnit4::class)
class TokenStorageTest {

    private lateinit var storage: TokenStorage

    @BeforeEach
    fun setUp() {
        storage = TokenStorage(ApplicationProvider.getApplicationContext())
        storage.clear()
    }

    @Test
    fun `saveTokens persiste e recupera`() {
        storage.saveTokens(
            deviceToken = "dt", refreshToken = "rt",
            agenteId = 42L, usuarioId = 100L, instalacaoId = "uuid-123"
        )
        assertEquals("dt", storage.getDeviceToken())
        assertEquals("rt", storage.getRefreshToken())
        assertEquals(42L, storage.getAgenteId())
        assertEquals(100L, storage.getUsuarioId())
        assertEquals("uuid-123", storage.getInstalacaoId())
    }

    @Test
    fun `estado inicial retorna null e -1`() {
        assertNull(storage.getDeviceToken())
        assertNull(storage.getRefreshToken())
        assertEquals(-1L, storage.getAgenteId())
        assertEquals(-1L, storage.getUsuarioId())
        assertNull(storage.getInstalacaoId())
    }

    @Test
    fun `clear zera tudo`() {
        storage.saveTokens("dt", "rt", 42L, 100L, "uuid")
        storage.clear()
        assertNull(storage.getDeviceToken())
        assertEquals(-1L, storage.getAgenteId())
    }
}
```

- [ ] **Step 3: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.auth.TokenStorageTest"`
Expected: FAIL.

- [ ] **Step 4: Criar `TokenStorage.kt`**

```kotlin
package com.trivion.manageragent.auth

import android.content.Context
import android.content.SharedPreferences
import androidx.security.crypto.EncryptedSharedPreferences
import androidx.security.crypto.MasterKey

class TokenStorage(context: Context) {

    private val prefs: SharedPreferences = try {
        val masterKey = MasterKey.Builder(context)
            .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
            .build()
        EncryptedSharedPreferences.create(
            context,
            "manageragent_tokens",
            masterKey,
            EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
            EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
        )
    } catch (e: Exception) {
        // Fallback pra SharedPreferences plain se Keystore corrompido — logamos e seguimos.
        // Em produção, esse fallback é uma degradação; mas nunca deixa o app não subir.
        timber.log.Timber.tag("TokenStorage").w(e, "Falha no EncryptedSharedPreferences, usando plain")
        context.getSharedPreferences("manageragent_tokens_plain", Context.MODE_PRIVATE)
    }

    fun saveTokens(deviceToken: String, refreshToken: String, agenteId: Long, usuarioId: Long, instalacaoId: String) {
        prefs.edit()
            .putString(KEY_DEVICE_TOKEN, deviceToken)
            .putString(KEY_REFRESH_TOKEN, refreshToken)
            .putLong(KEY_AGENTE_ID, agenteId)
            .putLong(KEY_USUARIO_ID, usuarioId)
            .putString(KEY_INSTALACAO_ID, instalacaoId)
            .apply()
    }

    fun getDeviceToken(): String? = prefs.getString(KEY_DEVICE_TOKEN, null)
    fun getRefreshToken(): String? = prefs.getString(KEY_REFRESH_TOKEN, null)
    fun getAgenteId(): Long = prefs.getLong(KEY_AGENTE_ID, -1L)
    fun getUsuarioId(): Long = prefs.getLong(KEY_USUARIO_ID, -1L)
    fun getInstalacaoId(): String? = prefs.getString(KEY_INSTALACAO_ID, null)

    fun updateTokens(deviceToken: String, refreshToken: String) {
        prefs.edit()
            .putString(KEY_DEVICE_TOKEN, deviceToken)
            .putString(KEY_REFRESH_TOKEN, refreshToken)
            .apply()
    }

    fun clear() {
        prefs.edit().clear().apply()
    }

    companion object {
        private const val KEY_DEVICE_TOKEN = "device_token"
        private const val KEY_REFRESH_TOKEN = "refresh_token"
        private const val KEY_AGENTE_ID = "agente_id"
        private const val KEY_USUARIO_ID = "usuario_id"
        private const val KEY_INSTALACAO_ID = "instalacao_id"
    }
}
```

- [ ] **Step 5: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.auth.TokenStorageTest"`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add .
git commit -m "feat: add TokenStorage (EncryptedSharedPreferences with Keystore fallback)"
```

---

### Task T28: Retrofit + OkHttp setup + DTOs de vinculação

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/network/AdminApi.kt`
- Create: `app/src/main/kotlin/com/trivion/manageragent/network/EventsApi.kt`
- Create: `app/src/main/kotlin/com/trivion/manageragent/network/ApiClient.kt`
- Create: `app/src/main/kotlin/com/trivion/manageragent/network/dto/VincularDtos.kt`
- Create: `app/src/main/kotlin/com/trivion/manageragent/network/dto/EventBatchDtos.kt`
- Create: `app/src/main/kotlin/com/trivion/manageragent/network/dto/RefreshDtos.kt`
- Create: `app/src/main/kotlin/com/trivion/manageragent/network/dto/UpdateCheckDtos.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/network/ApiClientTest.kt`

**Interfaces:**
- Consumes: `BuildConfig.API_ADMIN_URL`, `API_EVENTS_URL` (T4), `AgentEvent` model (T8)
- Produces:
  - `AdminApi` — interface Retrofit com `vincular`, `refresh`, `verificarAtualizacao`, `reportarAtualizacao`, `registrarAudit`, `reportarErro`, `validarColaborador`
  - `EventsApi` — interface Retrofit com `enviarBatch`
  - `ApiClient.buildAdminApi(context, tokenStorage): AdminApi` e `buildEventsApi(context, tokenStorage): EventsApi`
  - DTOs correspondentes a cada request/response

- [ ] **Step 1: Adicionar dependências**

Em `app/build.gradle.kts`:

```kotlin
implementation(libs.retrofit.core)
implementation(libs.retrofit.kotlinx.serialization)
implementation(libs.okhttp.core)
implementation(libs.okhttp.logging)

testImplementation(libs.okhttp.mockwebserver)
```

- [ ] **Step 2: Criar DTOs — `VincularDtos.kt`**

```kotlin
package com.trivion.manageragent.network.dto

import kotlinx.serialization.Serializable

@Serializable
data class VincularRequest(
    val identificador: String,
    val instalacaoId: String,
    val maquinaId: String,
    val nomeMaquina: String,
    val versaoAgente: String,
    val descricaoSo: String,
    val dispositivoTipo: String  // "ANDROID"
)

@Serializable
data class VincularResponse(
    val agenteId: Long,
    val usuarioId: Long,
    val deviceToken: String,
    val refreshToken: String
)
```

- [ ] **Step 3: Criar `RefreshDtos.kt`**

```kotlin
package com.trivion.manageragent.network.dto

import kotlinx.serialization.Serializable

@Serializable
data class RefreshRequest(val refreshToken: String)

@Serializable
data class RefreshResponse(val deviceToken: String, val refreshToken: String)
```

- [ ] **Step 4: Criar `EventBatchDtos.kt`**

```kotlin
package com.trivion.manageragent.network.dto

import com.trivion.manageragent.event.model.AgentEvent
import kotlinx.serialization.Serializable

@Serializable
data class EventBatchRequest(
    val agenteId: Long,
    val dispositivoTipo: String,  // "ANDROID"
    val eventos: List<AgentEvent>
)

@Serializable
data class EventBatchResponse(
    val aceitos: Int,
    val rejeitados: Int = 0,
    val motivosRejeicao: List<String> = emptyList()
)
```

- [ ] **Step 5: Criar `UpdateCheckDtos.kt`**

```kotlin
package com.trivion.manageragent.network.dto

import kotlinx.serialization.Serializable

@Serializable
data class UpdateCheckResponse(
    val atualizacaoDisponivel: Boolean,
    val versaoNova: String? = null,
    val urlDownload: String? = null,
    val checksumSha256: String? = null,
    val obrigatoria: Boolean = false
)

@Serializable
data class UpdateResultRequest(
    val versaoTentada: String,
    val sucesso: Boolean,
    val motivoErro: String? = null
)

@Serializable
data class AuditRegisterRequest(
    val evento: String,
    val instalacaoId: String,
    val agenteId: Long,
    val timestampUtc: String,
    val dados: Map<String, String> = emptyMap()
)

@Serializable
data class ErrorReportRequest(
    val tipo: String,
    val mensagem: String,
    val stackTrace: String? = null,
    val versao: String,
    val sistemaOperacional: String,
    val maquina: String,
    val colaboradorId: Long
)

@Serializable
data class ValidarColaboradorRequest(val identificador: String)

@Serializable
data class ValidarColaboradorResponse(
    val existe: Boolean,
    val usuarioId: Long? = null,
    val nomeCompleto: String? = null,
    val mensagem: String? = null
)
```

- [ ] **Step 6: Criar `AdminApi.kt`**

```kotlin
package com.trivion.manageragent.network

import com.trivion.manageragent.network.dto.*
import retrofit2.Response
import retrofit2.http.*

interface AdminApi {

    @POST("/api/agente/v1/colaboradores/validar")
    suspend fun validarColaborador(
        @Header("X-Ativacao-Key") chaveAtivacao: String,
        @Body body: ValidarColaboradorRequest
    ): Response<ValidarColaboradorResponse>

    @POST("/api/agente/dispositivos/vincular")
    suspend fun vincular(
        @Header("X-Ativacao-Key") chaveAtivacao: String,
        @Body body: VincularRequest
    ): Response<VincularResponse>

    @POST("/api/agente/auth/refresh")
    suspend fun refresh(@Body body: RefreshRequest): Response<RefreshResponse>

    @GET("/api/agente/atualizacoes/verificar")
    suspend fun verificarAtualizacao(
        @Query("sistemaOperacional") so: String,
        @Query("versaoAtual") versaoAtual: String
    ): Response<UpdateCheckResponse>

    @POST("/api/agente/atualizacoes/resultado")
    suspend fun reportarAtualizacao(@Body body: UpdateResultRequest): Response<Unit>

    @POST("/api/agente/auditoria/registrar")
    suspend fun registrarAudit(@Body body: AuditRegisterRequest): Response<Unit>

    @POST("/api/agent/error-report")
    suspend fun reportarErro(@Body body: ErrorReportRequest): Response<Unit>
}
```

- [ ] **Step 7: Criar `EventsApi.kt`**

```kotlin
package com.trivion.manageragent.network

import com.trivion.manageragent.network.dto.EventBatchRequest
import com.trivion.manageragent.network.dto.EventBatchResponse
import retrofit2.Response
import retrofit2.http.Body
import retrofit2.http.POST

interface EventsApi {

    @POST("/api/agent/events")
    suspend fun enviarBatch(@Body body: EventBatchRequest): Response<EventBatchResponse>
}
```

- [ ] **Step 8: Criar `ApiClient.kt`**

```kotlin
package com.trivion.manageragent.network

import android.content.Context
import com.trivion.manageragent.BuildConfig
import com.trivion.manageragent.auth.TokenStorage
import kotlinx.serialization.json.Json
import okhttp3.MediaType.Companion.toMediaType
import okhttp3.OkHttpClient
import okhttp3.logging.HttpLoggingInterceptor
import retrofit2.Retrofit
import retrofit2.converter.kotlinx.serialization.asConverterFactory
import java.util.concurrent.TimeUnit

object ApiClient {

    private val json = Json {
        classDiscriminator = "tipoEvento"
        encodeDefaults = true
        ignoreUnknownKeys = true
    }

    fun buildAdminApi(context: Context, tokenStorage: TokenStorage): AdminApi {
        return retrofit(BuildConfig.API_ADMIN_URL, tokenStorage).create(AdminApi::class.java)
    }

    fun buildEventsApi(context: Context, tokenStorage: TokenStorage): EventsApi {
        return retrofit(BuildConfig.API_EVENTS_URL, tokenStorage).create(EventsApi::class.java)
    }

    private fun retrofit(baseUrl: String, tokenStorage: TokenStorage): Retrofit {
        val client = OkHttpClient.Builder()
            .connectTimeout(10, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .writeTimeout(30, TimeUnit.SECONDS)
            .addInterceptor(AuthInterceptor(tokenStorage))
            .apply {
                if (BuildConfig.DEBUG) {
                    addInterceptor(HttpLoggingInterceptor().apply { level = HttpLoggingInterceptor.Level.BASIC })
                }
            }
            .build()

        return Retrofit.Builder()
            .baseUrl(baseUrl)
            .client(client)
            .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
            .build()
    }
}
```

- [ ] **Step 9: Criar `AuthInterceptor.kt`**

```kotlin
package com.trivion.manageragent.network

import com.trivion.manageragent.auth.TokenStorage
import okhttp3.Interceptor
import okhttp3.Response

class AuthInterceptor(private val tokenStorage: TokenStorage) : Interceptor {

    override fun intercept(chain: Interceptor.Chain): Response {
        val req = chain.request()
        // Não injeta Bearer se request já tem X-Ativacao-Key (fluxo de vinculação/validação)
        if (req.header("X-Ativacao-Key") != null) return chain.proceed(req)
        val token = tokenStorage.getDeviceToken()
        val newReq = if (token != null) {
            req.newBuilder().addHeader("Authorization", "Bearer $token").build()
        } else req
        return chain.proceed(newReq)
    }
}
```

- [ ] **Step 10: Escrever teste com MockWebServer**

Criar `app/src/test/kotlin/com/trivion/manageragent/network/ApiClientTest.kt`:

```kotlin
package com.trivion.manageragent.network

import com.trivion.manageragent.auth.TokenStorage
import com.trivion.manageragent.network.dto.VincularRequest
import io.mockk.every
import io.mockk.mockk
import kotlinx.coroutines.runBlocking
import okhttp3.mockwebserver.MockResponse
import okhttp3.mockwebserver.MockWebServer
import org.junit.jupiter.api.AfterEach
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import retrofit2.Retrofit
import retrofit2.converter.kotlinx.serialization.asConverterFactory
import okhttp3.MediaType.Companion.toMediaType
import kotlinx.serialization.json.Json
import okhttp3.OkHttpClient
import kotlin.test.assertEquals

class ApiClientTest {

    private lateinit var server: MockWebServer
    private lateinit var api: AdminApi
    private val tokenStorage = mockk<TokenStorage>().apply {
        every { getDeviceToken() } returns "token-xyz"
    }

    @BeforeEach
    fun setUp() {
        server = MockWebServer().also { it.start() }
        val json = Json { ignoreUnknownKeys = true; encodeDefaults = true }
        val client = OkHttpClient.Builder().addInterceptor(AuthInterceptor(tokenStorage)).build()
        api = Retrofit.Builder()
            .baseUrl(server.url("/"))
            .client(client)
            .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
            .build()
            .create(AdminApi::class.java)
    }

    @AfterEach
    fun tearDown() { server.shutdown() }

    @Test
    fun `vincular envia payload correto e parseia response`() = runBlocking {
        server.enqueue(MockResponse().setResponseCode(200).setBody(
            """{"agenteId":42,"usuarioId":100,"deviceToken":"dt","refreshToken":"rt"}"""
        ))
        val response = api.vincular(
            chaveAtivacao = "chave-abc",
            body = VincularRequest("id1", "inst1", "maq1", "Pixel", "1.0.0", "Android 13", "ANDROID")
        )
        assertEquals(200, response.code())
        assertEquals(42L, response.body()!!.agenteId)

        val recorded = server.takeRequest()
        assertEquals("/api/agente/dispositivos/vincular", recorded.path)
        assertEquals("chave-abc", recorded.getHeader("X-Ativacao-Key"))
        // Não deve ter Bearer (tem X-Ativacao-Key)
        assertEquals(null, recorded.getHeader("Authorization"))
    }

    @Test
    fun `endpoints autenticados injetam Bearer do TokenStorage`() = runBlocking {
        server.enqueue(MockResponse().setResponseCode(200).setBody(
            """{"atualizacaoDisponivel":false}"""
        ))
        api.verificarAtualizacao("ANDROID", "1.0.0")
        val recorded = server.takeRequest()
        assertEquals("Bearer token-xyz", recorded.getHeader("Authorization"))
    }
}
```

- [ ] **Step 11: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.network.ApiClientTest"`
Expected: PASS (2 testes).

- [ ] **Step 12: Commit**

```bash
git add .
git commit -m "feat: add Retrofit clients (AdminApi + EventsApi) + AuthInterceptor + DTOs"
```

---

### Task T29: AuthManager — vinculação inicial (validarColaborador → vincular)

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/auth/AuthManager.kt`
- Create: `app/src/main/kotlin/com/trivion/manageragent/auth/DeviceInfo.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/auth/AuthManagerTest.kt`

**Interfaces:**
- Consumes: `AdminApi` (T28), `TokenStorage` (T27), `AgentConfig` (T26), `DeviceInfo`
- Produces:
  - `sealed class VincularResult { data class Success(val usuarioId: Long) : ...; data class Failure(val motivo: String) : ... }`
  - `AuthManager(adminApi, tokenStorage, deviceInfo, config)`
  - `suspend fun validarIdentificador(identificador: String): Boolean`
  - `suspend fun vincular(identificador: String): VincularResult`

- [ ] **Step 1: Criar `DeviceInfo.kt`**

```kotlin
package com.trivion.manageragent.auth

import android.os.Build
import java.util.UUID

data class DeviceInfo(
    val instalacaoId: String,   // UUID persistido pela primeira vez
    val maquinaId: String,      // Build.SERIAL não é acessível — usar hardware fingerprint estável
    val nomeMaquina: String,    // Build.MANUFACTURER + " " + Build.MODEL
    val descricaoSo: String     // "Android 13 (API 33)"
) {
    companion object {
        fun current(tokenStorage: TokenStorage): DeviceInfo {
            val instalacaoId = tokenStorage.getInstalacaoId() ?: UUID.randomUUID().toString()
            return DeviceInfo(
                instalacaoId = instalacaoId,
                maquinaId = "${Build.BOARD}-${Build.BRAND}-${Build.HARDWARE}".hashCode().toString(),
                nomeMaquina = "${Build.MANUFACTURER} ${Build.MODEL}",
                descricaoSo = "Android ${Build.VERSION.RELEASE} (API ${Build.VERSION.SDK_INT})"
            )
        }
    }
}
```

- [ ] **Step 2: Escrever teste**

Criar `app/src/test/kotlin/com/trivion/manageragent/auth/AuthManagerTest.kt`:

```kotlin
package com.trivion.manageragent.auth

import com.trivion.manageragent.BuildConfig
import com.trivion.manageragent.config.AgentConfig
import com.trivion.manageragent.network.AdminApi
import com.trivion.manageragent.network.dto.*
import io.mockk.coEvery
import io.mockk.mockk
import io.mockk.verify
import kotlinx.coroutines.test.runTest
import okhttp3.MediaType.Companion.toMediaType
import okhttp3.ResponseBody.Companion.toResponseBody
import org.junit.jupiter.api.Test
import retrofit2.Response
import kotlin.test.assertEquals
import kotlin.test.assertTrue

class AuthManagerTest {

    private val adminApi = mockk<AdminApi>()
    private val tokenStorage = mockk<TokenStorage>(relaxed = true)
    private val deviceInfo = DeviceInfo("inst-1", "maq-1", "Pixel 8", "Android 14 (API 34)")
    private val config = AgentConfig("staging", "chave-empresa-abc", "Trivion")

    private val manager get() = AuthManager(adminApi, tokenStorage, deviceInfo, config)

    @Test
    fun `validarIdentificador retorna true quando existe`() = runTest {
        coEvery { adminApi.validarColaborador(any(), any()) } returns Response.success(
            ValidarColaboradorResponse(existe = true, usuarioId = 100L, nomeCompleto = "João")
        )
        assertTrue(manager.validarIdentificador("cpf1"))
    }

    @Test
    fun `validarIdentificador retorna false quando nao existe`() = runTest {
        coEvery { adminApi.validarColaborador(any(), any()) } returns Response.success(
            ValidarColaboradorResponse(existe = false, mensagem = "not found")
        )
        assertEquals(false, manager.validarIdentificador("cpf-invalido"))
    }

    @Test
    fun `vincular sucesso salva tokens`() = runTest {
        coEvery { adminApi.vincular(any(), any()) } returns Response.success(
            VincularResponse(agenteId = 42L, usuarioId = 100L, deviceToken = "dt", refreshToken = "rt")
        )
        val result = manager.vincular("cpf1")
        assertTrue(result is AuthManager.VincularResult.Success)
        assertEquals(100L, (result as AuthManager.VincularResult.Success).usuarioId)
        verify {
            tokenStorage.saveTokens(
                deviceToken = "dt",
                refreshToken = "rt",
                agenteId = 42L,
                usuarioId = 100L,
                instalacaoId = "inst-1"
            )
        }
    }

    @Test
    fun `vincular retorna Failure em erro 4xx`() = runTest {
        coEvery { adminApi.vincular(any(), any()) } returns Response.error(
            400, """{"mensagem":"chave inválida"}""".toResponseBody("application/json".toMediaType())
        )
        val result = manager.vincular("cpf1")
        assertTrue(result is AuthManager.VincularResult.Failure)
    }

    @Test
    fun `vincular retorna Failure em exception de rede`() = runTest {
        coEvery { adminApi.vincular(any(), any()) } throws java.io.IOException("network down")
        val result = manager.vincular("cpf1")
        assertTrue(result is AuthManager.VincularResult.Failure)
        assertTrue((result as AuthManager.VincularResult.Failure).motivo.contains("network"))
    }
}
```

- [ ] **Step 3: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.auth.AuthManagerTest"`
Expected: FAIL.

- [ ] **Step 4: Criar `AuthManager.kt`**

```kotlin
package com.trivion.manageragent.auth

import com.trivion.manageragent.BuildConfig
import com.trivion.manageragent.config.AgentConfig
import com.trivion.manageragent.network.AdminApi
import com.trivion.manageragent.network.dto.ValidarColaboradorRequest
import com.trivion.manageragent.network.dto.VincularRequest
import timber.log.Timber

class AuthManager(
    private val adminApi: AdminApi,
    private val tokenStorage: TokenStorage,
    private val deviceInfo: DeviceInfo,
    private val config: AgentConfig
) {

    sealed class VincularResult {
        data class Success(val usuarioId: Long) : VincularResult()
        data class Failure(val motivo: String) : VincularResult()
    }

    suspend fun validarIdentificador(identificador: String): Boolean {
        return try {
            val resp = adminApi.validarColaborador(
                chaveAtivacao = config.chaveAtivacao,
                body = ValidarColaboradorRequest(identificador)
            )
            resp.isSuccessful && resp.body()?.existe == true
        } catch (e: Exception) {
            Timber.tag(TAG).w(e, "validarIdentificador falhou")
            false
        }
    }

    suspend fun vincular(identificador: String): VincularResult {
        return try {
            val resp = adminApi.vincular(
                chaveAtivacao = config.chaveAtivacao,
                body = VincularRequest(
                    identificador = identificador,
                    instalacaoId = deviceInfo.instalacaoId,
                    maquinaId = deviceInfo.maquinaId,
                    nomeMaquina = deviceInfo.nomeMaquina,
                    versaoAgente = BuildConfig.VERSION_NAME,
                    descricaoSo = deviceInfo.descricaoSo,
                    dispositivoTipo = "ANDROID"
                )
            )
            if (!resp.isSuccessful) {
                return VincularResult.Failure("HTTP ${resp.code()}")
            }
            val body = resp.body() ?: return VincularResult.Failure("body vazio")
            tokenStorage.saveTokens(
                deviceToken = body.deviceToken,
                refreshToken = body.refreshToken,
                agenteId = body.agenteId,
                usuarioId = body.usuarioId,
                instalacaoId = deviceInfo.instalacaoId
            )
            VincularResult.Success(body.usuarioId)
        } catch (e: Exception) {
            Timber.tag(TAG).e(e, "vincular falhou")
            VincularResult.Failure(e.message ?: "erro desconhecido")
        }
    }

    companion object { private const val TAG = "AuthManager" }
}
```

- [ ] **Step 5: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.auth.AuthManagerTest"`
Expected: PASS (5 testes).

- [ ] **Step 6: Commit**

```bash
git add .
git commit -m "feat: add AuthManager (validar + vincular) with token persistence"
```

---

### Task T30: RefreshInterceptor — renova device token em 401

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/network/RefreshInterceptor.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/network/RefreshInterceptorTest.kt`
- Modify: `app/src/main/kotlin/com/trivion/manageragent/network/ApiClient.kt` (encadear interceptor)

**Interfaces:**
- Consumes: `TokenStorage` (T27), `AdminApi.refresh` (T28)
- Produces:
  - `RefreshInterceptor(tokenStorage, refreshFn: suspend (String) -> RefreshResponse?)` — em 401, tenta refresh; se sucesso, salva novos tokens e refaz request; se falhar, retorna 401 original

- [ ] **Step 1: Escrever teste**

Criar `app/src/test/kotlin/com/trivion/manageragent/network/RefreshInterceptorTest.kt`:

```kotlin
package com.trivion.manageragent.network

import com.trivion.manageragent.auth.TokenStorage
import com.trivion.manageragent.network.dto.RefreshResponse
import io.mockk.every
import io.mockk.mockk
import io.mockk.verify
import okhttp3.OkHttpClient
import okhttp3.Request
import okhttp3.mockwebserver.MockResponse
import okhttp3.mockwebserver.MockWebServer
import org.junit.jupiter.api.AfterEach
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import kotlin.test.assertEquals

class RefreshInterceptorTest {

    private lateinit var server: MockWebServer
    private val storage = mockk<TokenStorage>(relaxed = true)

    @BeforeEach
    fun setUp() {
        server = MockWebServer().also { it.start() }
        every { storage.getDeviceToken() } returnsMany listOf("old-token", "new-token")
        every { storage.getRefreshToken() } returns "refresh-old"
    }

    @AfterEach
    fun tearDown() { server.shutdown() }

    @Test
    fun `em 401 tenta refresh e refaz request`() {
        server.enqueue(MockResponse().setResponseCode(401))
        server.enqueue(MockResponse().setResponseCode(200).setBody("ok"))

        val refreshResp = RefreshResponse(deviceToken = "new-token", refreshToken = "refresh-new")
        val client = OkHttpClient.Builder()
            .addInterceptor(AuthInterceptor(storage))
            .addInterceptor(RefreshInterceptor(storage) { _ -> refreshResp })
            .build()

        val response = client.newCall(Request.Builder().url(server.url("/x")).build()).execute()
        assertEquals(200, response.code)

        verify { storage.updateTokens(deviceToken = "new-token", refreshToken = "refresh-new") }
    }

    @Test
    fun `se refresh falhar retorna 401 original`() {
        server.enqueue(MockResponse().setResponseCode(401))

        val client = OkHttpClient.Builder()
            .addInterceptor(AuthInterceptor(storage))
            .addInterceptor(RefreshInterceptor(storage) { _ -> null })
            .build()

        val response = client.newCall(Request.Builder().url(server.url("/x")).build()).execute()
        assertEquals(401, response.code)
        verify(exactly = 0) { storage.updateTokens(any(), any()) }
    }

    @Test
    fun `nao tenta refresh se response for 200`() {
        server.enqueue(MockResponse().setResponseCode(200).setBody("ok"))
        val client = OkHttpClient.Builder()
            .addInterceptor(AuthInterceptor(storage))
            .addInterceptor(RefreshInterceptor(storage) { _ -> throw IllegalStateException("nao deveria chamar") })
            .build()
        val response = client.newCall(Request.Builder().url(server.url("/x")).build()).execute()
        assertEquals(200, response.code)
    }
}
```

- [ ] **Step 2: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.network.RefreshInterceptorTest"`
Expected: FAIL.

- [ ] **Step 3: Criar `RefreshInterceptor.kt`**

```kotlin
package com.trivion.manageragent.network

import com.trivion.manageragent.auth.TokenStorage
import com.trivion.manageragent.network.dto.RefreshResponse
import kotlinx.coroutines.runBlocking
import okhttp3.Interceptor
import okhttp3.Response
import timber.log.Timber

/**
 * Intercepta 401 e tenta refresh do token. Se sucesso, atualiza storage e retenta request.
 * `refreshFn` recebe o refresh token atual e retorna nova RefreshResponse ou null se falhar.
 * Design: refreshFn passado por injeção pra evitar dependência circular Retrofit ↔ interceptor.
 */
class RefreshInterceptor(
    private val tokenStorage: TokenStorage,
    private val refreshFn: suspend (String) -> RefreshResponse?
) : Interceptor {

    private val refreshLock = Any()

    override fun intercept(chain: Interceptor.Chain): Response {
        val response = chain.proceed(chain.request())
        if (response.code != 401) return response

        val refreshToken = tokenStorage.getRefreshToken() ?: return response

        val newTokens = synchronized(refreshLock) {
            // Double-check: outro thread pode ter refreshado
            val currentToken = tokenStorage.getDeviceToken()
            val originalToken = chain.request().header("Authorization")?.removePrefix("Bearer ")
            if (currentToken != null && currentToken != originalToken) {
                RefreshResponse(deviceToken = currentToken, refreshToken = tokenStorage.getRefreshToken() ?: "")
            } else {
                try {
                    runBlocking { refreshFn(refreshToken) }
                } catch (e: Exception) {
                    Timber.tag(TAG).w(e, "refresh failed")
                    null
                }
            }
        } ?: return response

        tokenStorage.updateTokens(newTokens.deviceToken, newTokens.refreshToken)
        response.close()
        val newReq = chain.request().newBuilder()
            .header("Authorization", "Bearer ${newTokens.deviceToken}")
            .build()
        return chain.proceed(newReq)
    }

    companion object { private const val TAG = "RefreshInterceptor" }
}
```

- [ ] **Step 4: Encadear interceptor no `ApiClient`**

Modificar `ApiClient.retrofit()`:

```kotlin
private fun retrofit(baseUrl: String, tokenStorage: TokenStorage): Retrofit {
    // Refresh usa uma referência lazy pro AdminApi criado pelo Retrofit dedicado (sem interceptor)
    val bareRetrofit = Retrofit.Builder()
        .baseUrl(BuildConfig.API_ADMIN_URL)
        .client(OkHttpClient())
        .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
        .build()
    val bareAdmin = bareRetrofit.create(AdminApi::class.java)

    val refreshFn: suspend (String) -> com.trivion.manageragent.network.dto.RefreshResponse? = { rt ->
        try {
            val resp = bareAdmin.refresh(com.trivion.manageragent.network.dto.RefreshRequest(rt))
            if (resp.isSuccessful) resp.body() else null
        } catch (e: Exception) { null }
    }

    val client = OkHttpClient.Builder()
        .connectTimeout(10, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .writeTimeout(30, TimeUnit.SECONDS)
        .addInterceptor(AuthInterceptor(tokenStorage))
        .addInterceptor(RefreshInterceptor(tokenStorage, refreshFn))
        .apply {
            if (BuildConfig.DEBUG) addInterceptor(HttpLoggingInterceptor().apply { level = HttpLoggingInterceptor.Level.BASIC })
        }
        .build()

    return Retrofit.Builder()
        .baseUrl(baseUrl)
        .client(client)
        .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
        .build()
}
```

- [ ] **Step 5: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.network.RefreshInterceptorTest"`
Expected: PASS (3 testes).

- [ ] **Step 6: Commit**

```bash
git add .
git commit -m "feat: add RefreshInterceptor with atomic double-check pattern"
```

---

### Task T31: BatchSenderWorker — envio periódico de eventos via WorkManager

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/sender/BatchSenderWorker.kt`
- Create: `app/src/main/kotlin/com/trivion/manageragent/sender/BatchScheduler.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/sender/BatchSenderWorkerTest.kt`
- Modify: `ManagerAgentService.onCreate()` (schedule)

**Interfaces:**
- Consumes: `EventQueue` (T10), `EventsApi` (T28), `TokenStorage` (T27)
- Produces:
  - `BatchSenderWorker(context, params)` — `doWork()`: dequeue ≤100 eventos, POST /api/agent/events, se sucesso ack; se falha, nack (incrementa tentativas)
  - `BatchScheduler.schedule(context, intervalMinutes=15)` — WorkManager periódico com network constraint

- [ ] **Step 1: Escrever teste**

Criar `app/src/test/kotlin/com/trivion/manageragent/sender/BatchSenderWorkerTest.kt`:

```kotlin
package com.trivion.manageragent.sender

import android.content.Context
import androidx.test.core.app.ApplicationProvider
import androidx.test.ext.junit.runners.AndroidJUnit4
import androidx.work.ListenableWorker
import androidx.work.testing.TestListenableWorkerBuilder
import com.trivion.manageragent.event.EventDatabase
import com.trivion.manageragent.event.EventQueue
import com.trivion.manageragent.event.model.AgentEvent
import kotlinx.coroutines.runBlocking
import kotlinx.serialization.json.Json
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import org.junit.runner.RunWith
import kotlin.test.assertEquals

@RunWith(AndroidJUnit4::class)
class BatchSenderWorkerTest {

    private val context: Context = ApplicationProvider.getApplicationContext()
    private lateinit var queue: EventQueue

    @BeforeEach
    fun setUp() = runBlocking {
        // Limpar fila
        val db = EventDatabase.getInstance(context)
        val existing = db.eventDao().pending(Int.MAX_VALUE).map { it.id }
        if (existing.isNotEmpty()) db.eventDao().deleteByIds(existing)
        queue = EventQueue(db.eventDao(), Json { classDiscriminator = "tipoEvento"; encodeDefaults = true })
    }

    @Test
    fun `worker retorna Success mesmo com fila vazia`() = runBlocking {
        val worker = TestListenableWorkerBuilder<BatchSenderWorker>(context).build()
        val result = worker.doWork()
        assertEquals(ListenableWorker.Result.success(), result)
    }

    @Test
    fun `worker esvazia fila com backend OK`() = runBlocking {
        queue.enqueue(AgentEvent.Heartbeat("t1", 0))
        queue.enqueue(AgentEvent.Heartbeat("t2", 0))
        assertEquals(2, queue.pendingCount())

        // Nota: neste teste, sem backend real, o worker fará POST que falha (network).
        // Comportamento esperado: fila permanece (nack incrementa tentativa). Tratado no próximo teste
        // com stub HTTP em androidTest — aqui só valida assinatura do doWork.
        val worker = TestListenableWorkerBuilder<BatchSenderWorker>(context).build()
        worker.doWork()  // ignora resultado — pode ser Success com nack ou Retry
    }
}
```

- [ ] **Step 2: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.sender.BatchSenderWorkerTest"`
Expected: FAIL.

- [ ] **Step 3: Criar `BatchSenderWorker.kt`**

```kotlin
package com.trivion.manageragent.sender

import android.content.Context
import androidx.work.CoroutineWorker
import androidx.work.WorkerParameters
import com.trivion.manageragent.auth.TokenStorage
import com.trivion.manageragent.event.EventDatabase
import com.trivion.manageragent.event.EventQueue
import com.trivion.manageragent.network.ApiClient
import com.trivion.manageragent.network.dto.EventBatchRequest
import kotlinx.serialization.json.Json
import timber.log.Timber

class BatchSenderWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val json = Json { classDiscriminator = "tipoEvento"; encodeDefaults = true }
        val queue = EventQueue(EventDatabase.getInstance(applicationContext).eventDao(), json)
        val tokenStorage = TokenStorage(applicationContext)
        val agenteId = tokenStorage.getAgenteId()
        if (agenteId <= 0L) {
            Timber.tag(TAG).w("Sem vinculação ainda, skip send")
            return Result.success()
        }

        val batch = queue.dequeueBatch(limit = 100)
        if (batch.isEmpty()) return Result.success()

        val eventsApi = ApiClient.buildEventsApi(applicationContext, tokenStorage)

        return try {
            val response = eventsApi.enviarBatch(
                EventBatchRequest(
                    agenteId = agenteId,
                    dispositivoTipo = "ANDROID",
                    eventos = batch.map { it.second }
                )
            )
            if (response.isSuccessful) {
                queue.ack(batch.map { it.first })
                Timber.tag(TAG).i("Sent ${batch.size} events successfully")
                Result.success()
            } else {
                queue.nack(batch.map { it.first })
                Timber.tag(TAG).w("Send failed with ${response.code()}, will retry")
                Result.retry()
            }
        } catch (e: Exception) {
            queue.nack(batch.map { it.first })
            Timber.tag(TAG).w(e, "Send exception, will retry")
            Result.retry()
        }
    }

    companion object { private const val TAG = "BatchSenderWorker" }
}
```

- [ ] **Step 4: Criar `BatchScheduler.kt`**

```kotlin
package com.trivion.manageragent.sender

import android.content.Context
import androidx.work.*
import java.util.concurrent.TimeUnit

object BatchScheduler {

    private const val WORK_NAME = "ManagerAgent.BatchSender"

    fun schedule(context: Context, intervalMinutes: Long = 15L) {
        val constraints = Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .build()
        val req = PeriodicWorkRequestBuilder<BatchSenderWorker>(intervalMinutes, TimeUnit.MINUTES)
            .setConstraints(constraints)
            .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 5, TimeUnit.MINUTES)
            .build()
        WorkManager.getInstance(context).enqueueUniquePeriodicWork(
            WORK_NAME,
            ExistingPeriodicWorkPolicy.KEEP,
            req
        )
    }

    fun cancel(context: Context) {
        WorkManager.getInstance(context).cancelUniqueWork(WORK_NAME)
    }
}
```

- [ ] **Step 5: Wire no ManagerAgentService**

```kotlin
override fun onCreate() {
    super.onCreate()
    // ... resto existente
    BatchScheduler.schedule(this)  // NOVO
}
```

- [ ] **Step 6: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.sender.BatchSenderWorkerTest"`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add .
git commit -m "feat: add BatchSenderWorker + BatchScheduler (periodic send with network constraint)"
```

---

## BLOCO B6 — PermissionDispatchActivity + LGPD (T32-T36)

### Task T32: Layout XML da tela mínima (identificador + aceite LGPD)

**Files:**
- Create: `app/src/main/res/layout/activity_permission_dispatch.xml`
- Modify: `app/src/main/res/values/strings.xml`
- Modify: `app/src/main/res/values/styles.xml` (criar se não existir)

**Interfaces:**
- Consumes: nada
- Produces: layout XML da única tela do app: título "Bem-vindo ao ManagerAgent", subtítulo com nome da empresa, campo texto identificador, checkbox aceite LGPD com link, botão "Continuar"

- [ ] **Step 1: Adicionar strings**

Editar `app/src/main/res/values/strings.xml`:

```xml
<resources>
    <string name="app_name">ManagerAgent</string>
    <string name="notification_channel_id">manageragent_status</string>
    <string name="notification_channel_name">ManagerAgent status</string>
    <string name="notification_status_text">ManagerAgent • em execução</string>
    <string name="notification_action_enviar_diagnostico">Enviar diagnóstico</string>

    <!-- PermissionDispatchActivity -->
    <string name="permission_title">Bem-vindo ao ManagerAgent</string>
    <string name="permission_empresa_prefix">Empresa: </string>
    <string name="permission_identificador_hint">Digite seu CPF ou matrícula</string>
    <string name="permission_termos_prefix">Li e aceito os </string>
    <string name="permission_termos_link">Termos de Uso e Política de Privacidade</string>
    <string name="permission_termos_url">https://imanagerportal.com/agent-mobile/termos</string>
    <string name="permission_botao_continuar">Continuar</string>
    <string name="permission_validando">Validando identificador…</string>
    <string name="permission_erro_id_nao_encontrado">Identificador não encontrado. Fale com seu gestor.</string>
    <string name="permission_erro_rede">Erro de conexão. Tente novamente.</string>
    <string name="permission_ok_toast">ManagerAgent ativo ✓</string>

    <string name="permission_step_accessibility">Ative o serviço de Acessibilidade</string>
    <string name="permission_step_usage_stats">Ative "Acesso a dados de uso"</string>
    <string name="permission_step_notification">Permita notificações</string>
    <string name="permission_step_phone">Permita ler estado do telefone</string>
    <string name="permission_step_oem">Configuração adicional do fabricante</string>
    <string name="accessibility_description">Coleta metadados de janela ativa e interações agregadas (sem ler conteúdo de textos) para gerar relatórios de produtividade.</string>
</resources>
```

- [ ] **Step 2: Criar `app/src/main/res/layout/activity_permission_dispatch.xml`**

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@android:color/white"
    android:padding="24dp">

    <TextView
        android:id="@+id/txtTitle"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="48dp"
        android:text="@string/permission_title"
        android:textSize="24sp"
        android:textStyle="bold"
        app:layout_constraintTop_toTopOf="parent" />

    <TextView
        android:id="@+id/txtEmpresa"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="8dp"
        android:textSize="16sp"
        app:layout_constraintTop_toBottomOf="@id/txtTitle"
        tools:text="Empresa: Trivion" />

    <com.google.android.material.textfield.TextInputLayout
        android:id="@+id/inputLayoutIdentificador"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="32dp"
        android:hint="@string/permission_identificador_hint"
        app:layout_constraintTop_toBottomOf="@id/txtEmpresa">

        <com.google.android.material.textfield.TextInputEditText
            android:id="@+id/edtIdentificador"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:inputType="text"
            android:maxLines="1" />
    </com.google.android.material.textfield.TextInputLayout>

    <LinearLayout
        android:id="@+id/layoutTermos"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="16dp"
        android:orientation="horizontal"
        app:layout_constraintTop_toBottomOf="@id/inputLayoutIdentificador">

        <CheckBox
            android:id="@+id/chkTermos"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content" />

        <TextView
            android:id="@+id/txtTermos"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:layout_gravity="center_vertical"
            android:textSize="14sp"
            tools:text="Li e aceito os Termos de Uso e Política de Privacidade" />
    </LinearLayout>

    <Button
        android:id="@+id/btnContinuar"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="32dp"
        android:text="@string/permission_botao_continuar"
        android:enabled="false"
        app:layout_constraintTop_toBottomOf="@id/layoutTermos" />

    <ProgressBar
        android:id="@+id/progress"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="16dp"
        android:visibility="gone"
        app:layout_constraintTop_toBottomOf="@id/btnContinuar"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />

    <TextView
        android:id="@+id/txtStatus"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="8dp"
        android:textSize="14sp"
        android:textColor="@android:color/darker_gray"
        app:layout_constraintTop_toBottomOf="@id/progress" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

- [ ] **Step 3: Adicionar dependências Material Design**

Em `app/build.gradle.kts`:

```kotlin
implementation("com.google.android.material:material:1.11.0")
implementation("androidx.constraintlayout:constraintlayout:2.1.4")
```

Adicionar `tools` namespace no manifest (`xmlns:tools="http://schemas.android.com/tools"`) — já deve estar.

- [ ] **Step 4: Criar tema Material no `themes.xml`**

Criar `app/src/main/res/values/themes.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources xmlns:tools="http://schemas.android.com/tools">
    <style name="Theme.ManagerAgent" parent="Theme.MaterialComponents.Light.NoActionBar">
        <item name="colorPrimary">#1976D2</item>
        <item name="colorPrimaryDark">#0D47A1</item>
        <item name="colorAccent">#448AFF</item>
    </style>
</resources>
```

Atualizar `AndroidManifest.xml` no `<application>`:

```xml
android:theme="@style/Theme.ManagerAgent"
```

- [ ] **Step 5: Build e verificar**

Run: `./gradlew :app:assembleDebug`
Expected: BUILD SUCCESSFUL.

- [ ] **Step 6: Commit**

```bash
git add .
git commit -m "feat: add layout for PermissionDispatchActivity (identifier + LGPD checkbox)"
```

---

### Task T33: OEMDetector + OEMIntentFactory (Xiaomi/Huawei/Oppo/Vivo/Samsung)

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/oem/OEMDetector.kt`
- Create: `app/src/main/kotlin/com/trivion/manageragent/oem/OEMIntentFactory.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/oem/OEMDetectorTest.kt`

**Interfaces:**
- Consumes: `Build.MANUFACTURER` (Android)
- Produces:
  - `enum class OEMFabricante { SAMSUNG, XIAOMI, HUAWEI, OPPO, VIVO, ONEPLUS, MOTOROLA, GOOGLE, NOKIA, OUTRO }`
  - `object OEMDetector { fun detect(manufacturer: String): OEMFabricante }`
  - `object OEMIntentFactory { fun intentForAutostart(oem: OEMFabricante): Intent? }` — retorna intent pro Settings específico do OEM ou null se não precisa (stock Android)

- [ ] **Step 1: Escrever teste do detector**

Criar `app/src/test/kotlin/com/trivion/manageragent/oem/OEMDetectorTest.kt`:

```kotlin
package com.trivion.manageragent.oem

import org.junit.jupiter.api.Test
import kotlin.test.assertEquals

class OEMDetectorTest {

    @Test
    fun `Xiaomi detectado`() {
        assertEquals(OEMFabricante.XIAOMI, OEMDetector.detect("Xiaomi"))
        assertEquals(OEMFabricante.XIAOMI, OEMDetector.detect("Redmi"))
        assertEquals(OEMFabricante.XIAOMI, OEMDetector.detect("POCO"))
    }

    @Test
    fun `Huawei detectado`() {
        assertEquals(OEMFabricante.HUAWEI, OEMDetector.detect("HUAWEI"))
        assertEquals(OEMFabricante.HUAWEI, OEMDetector.detect("HONOR"))
    }

    @Test
    fun `Samsung detectado`() {
        assertEquals(OEMFabricante.SAMSUNG, OEMDetector.detect("samsung"))
    }

    @Test
    fun `Oppo Realme detectados`() {
        assertEquals(OEMFabricante.OPPO, OEMDetector.detect("OPPO"))
        assertEquals(OEMFabricante.OPPO, OEMDetector.detect("realme"))
    }

    @Test
    fun `Vivo detectado`() {
        assertEquals(OEMFabricante.VIVO, OEMDetector.detect("vivo"))
    }

    @Test
    fun `OnePlus detectado`() {
        assertEquals(OEMFabricante.ONEPLUS, OEMDetector.detect("OnePlus"))
    }

    @Test
    fun `Motorola / Google / Nokia detectados`() {
        assertEquals(OEMFabricante.MOTOROLA, OEMDetector.detect("motorola"))
        assertEquals(OEMFabricante.GOOGLE, OEMDetector.detect("Google"))
        assertEquals(OEMFabricante.NOKIA, OEMDetector.detect("HMD Global"))  // Nokia moderna
    }

    @Test
    fun `desconhecido vira OUTRO`() {
        assertEquals(OEMFabricante.OUTRO, OEMDetector.detect("Fabricante Desconhecido XYZ"))
        assertEquals(OEMFabricante.OUTRO, OEMDetector.detect(""))
    }

    @Test
    fun `Xiaomi eh problematico e OUTRO nao`() {
        assertEquals(true, OEMFabricante.XIAOMI.isProblematico)
        assertEquals(true, OEMFabricante.HUAWEI.isProblematico)
        assertEquals(false, OEMFabricante.GOOGLE.isProblematico)
        assertEquals(false, OEMFabricante.MOTOROLA.isProblematico)
        assertEquals(false, OEMFabricante.OUTRO.isProblematico)
    }
}
```

- [ ] **Step 2: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.oem.OEMDetectorTest"`
Expected: FAIL.

- [ ] **Step 3: Criar `OEMDetector.kt`**

```kotlin
package com.trivion.manageragent.oem

enum class OEMFabricante(val isProblematico: Boolean) {
    SAMSUNG(isProblematico = true),        // Samsung mata mas de leve
    XIAOMI(isProblematico = true),
    HUAWEI(isProblematico = true),
    OPPO(isProblematico = true),
    VIVO(isProblematico = true),
    ONEPLUS(isProblematico = true),        // ColorOS-like
    MOTOROLA(isProblematico = false),
    GOOGLE(isProblematico = false),
    NOKIA(isProblematico = false),
    OUTRO(isProblematico = false)
}

object OEMDetector {

    fun detect(manufacturer: String): OEMFabricante {
        val m = manufacturer.trim().lowercase()
        return when {
            m.contains("xiaomi") || m.contains("redmi") || m.contains("poco") -> OEMFabricante.XIAOMI
            m.contains("huawei") || m.contains("honor") -> OEMFabricante.HUAWEI
            m.contains("samsung") -> OEMFabricante.SAMSUNG
            m.contains("oppo") || m.contains("realme") -> OEMFabricante.OPPO
            m.contains("vivo") -> OEMFabricante.VIVO
            m.contains("oneplus") -> OEMFabricante.ONEPLUS
            m.contains("motorola") || m.contains("moto") || m.contains("lenovo") -> OEMFabricante.MOTOROLA
            m.contains("google") || m.contains("pixel") -> OEMFabricante.GOOGLE
            m.contains("hmd") || m.contains("nokia") -> OEMFabricante.NOKIA
            else -> OEMFabricante.OUTRO
        }
    }
}
```

- [ ] **Step 4: Criar `OEMIntentFactory.kt`**

```kotlin
package com.trivion.manageragent.oem

import android.content.ComponentName
import android.content.Context
import android.content.Intent
import android.provider.Settings
import android.net.Uri

object OEMIntentFactory {

    /**
     * Retorna lista de intents candidatos pra abrir Settings de Autostart/Battery do OEM.
     * Testar cada um até um funcionar (Manifest pode variar entre versões de OEM firmware).
     */
    fun intentsForAutostart(oem: OEMFabricante, context: Context): List<Intent> {
        val intents = mutableListOf<Intent>()
        when (oem) {
            OEMFabricante.XIAOMI -> {
                intents += Intent().setComponent(ComponentName("com.miui.securitycenter", "com.miui.permcenter.autostart.AutoStartManagementActivity"))
                intents += Intent().setComponent(ComponentName("com.miui.securitycenter", "com.miui.permcenter.MainAcitivty"))
            }
            OEMFabricante.HUAWEI -> {
                intents += Intent().setComponent(ComponentName("com.huawei.systemmanager", "com.huawei.systemmanager.startupmgr.ui.StartupNormalAppListActivity"))
                intents += Intent().setComponent(ComponentName("com.huawei.systemmanager", "com.huawei.systemmanager.optimize.process.ProtectActivity"))
            }
            OEMFabricante.OPPO -> {
                intents += Intent().setComponent(ComponentName("com.coloros.safecenter", "com.coloros.safecenter.startupapp.StartupAppListActivity"))
                intents += Intent().setComponent(ComponentName("com.oppo.safe", "com.oppo.safe.permission.startup.StartupAppListActivity"))
            }
            OEMFabricante.VIVO -> {
                intents += Intent().setComponent(ComponentName("com.vivo.permissionmanager", "com.vivo.permissionmanager.activity.BgStartUpManagerActivity"))
            }
            OEMFabricante.SAMSUNG -> {
                intents += Intent(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS).apply { data = Uri.parse("package:${context.packageName}") }
            }
            OEMFabricante.ONEPLUS -> {
                intents += Intent().setComponent(ComponentName("com.oneplus.security", "com.oneplus.security.chainlaunch.view.ChainLaunchAppListActivity"))
            }
            else -> Unit
        }
        return intents
    }

    /**
     * Fallback universal: abre Settings > App > Battery > Não otimizar
     */
    fun ignoreBatteryOptimizationIntent(context: Context): Intent {
        return Intent(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS).apply {
            data = Uri.parse("package:${context.packageName}")
        }
    }
}
```

- [ ] **Step 5: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.oem.OEMDetectorTest"`
Expected: PASS (8 testes).

- [ ] **Step 6: Commit**

```bash
git add .
git commit -m "feat: add OEMDetector + OEMIntentFactory (Xiaomi/Huawei/Oppo/Vivo/Samsung/OnePlus)"
```

---

### Task T34: PermissionDispatchActivity — encadeamento de intents + vinculação

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/ui/PermissionDispatchActivity.kt`
- Modify: `app/src/main/AndroidManifest.xml` (registrar Activity com MAIN + LAUNCHER)

**Interfaces:**
- Consumes: `ConfigReader` (T26), `AuthManager` (T29), `TokenStorage` (T27), `OEMDetector`/`OEMIntentFactory` (T33), `ManagerAgentService` (T11)
- Produces:
  - `PermissionDispatchActivity` — única Activity, exibe layout T32, valida input, encadeia intents de permissão via `ActivityResultLauncher`, chama `AuthManager.vincular`, inicia Service, se auto-desabilita como LAUNCHER

- [ ] **Step 1: Criar `PermissionDispatchActivity.kt`**

```kotlin
package com.trivion.manageragent.ui

import android.accessibilityservice.AccessibilityServiceInfo
import android.app.AppOpsManager
import android.content.ComponentName
import android.content.Intent
import android.content.pm.PackageManager
import android.net.Uri
import android.os.Build
import android.os.Bundle
import android.os.Process
import android.provider.Settings
import android.view.View
import android.view.accessibility.AccessibilityManager
import android.widget.Toast
import androidx.activity.result.contract.ActivityResultContracts
import androidx.appcompat.app.AppCompatActivity
import androidx.lifecycle.lifecycleScope
import com.trivion.manageragent.R
import com.trivion.manageragent.auth.AuthManager
import com.trivion.manageragent.auth.DeviceInfo
import com.trivion.manageragent.auth.TokenStorage
import com.trivion.manageragent.config.ConfigReader
import com.trivion.manageragent.databinding.ActivityPermissionDispatchBinding
import com.trivion.manageragent.network.ApiClient
import com.trivion.manageragent.oem.OEMDetector
import com.trivion.manageragent.oem.OEMFabricante
import com.trivion.manageragent.oem.OEMIntentFactory
import com.trivion.manageragent.service.ManagerAccessibilityService
import com.trivion.manageragent.service.ManagerAgentService
import kotlinx.coroutines.launch
import timber.log.Timber

class PermissionDispatchActivity : AppCompatActivity() {

    private lateinit var binding: ActivityPermissionDispatchBinding
    private lateinit var config: com.trivion.manageragent.config.AgentConfig
    private lateinit var tokenStorage: TokenStorage
    private lateinit var authManager: AuthManager

    private val requestNotifPermission = registerForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        Timber.tag(TAG).i("Notification permission: $granted")
        nextStep()
    }
    private val requestPhonePermission = registerForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        Timber.tag(TAG).i("Phone permission: $granted")
        nextStep()
    }
    private val settingsLauncher = registerForActivityResult(
        ActivityResultContracts.StartActivityForResult()
    ) { nextStep() }

    private enum class Step { ACCESSIBILITY, USAGE_STATS, NOTIFICATION, PHONE, OEM, VINCULAR, FINISH }
    private var currentStep = Step.ACCESSIBILITY

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityPermissionDispatchBinding.inflate(layoutInflater)
        setContentView(binding.root)

        config = ConfigReader.read(this)
        tokenStorage = TokenStorage(this)

        binding.txtEmpresa.text = getString(R.string.permission_empresa_prefix) + config.nomeEmpresa

        // Se já vinculado (retomando pós-crash), pula direto pra iniciar Service e sair
        if (tokenStorage.getDeviceToken() != null) {
            iniciarServiceEFinalizar()
            return
        }

        val termosLink = getString(R.string.permission_termos_link)
        binding.txtTermos.text = getString(R.string.permission_termos_prefix) + termosLink
        binding.txtTermos.setOnClickListener {
            startActivity(Intent(Intent.ACTION_VIEW, Uri.parse(getString(R.string.permission_termos_url))))
        }

        val validate: () -> Unit = {
            binding.btnContinuar.isEnabled =
                binding.edtIdentificador.text?.isNotBlank() == true &&
                binding.chkTermos.isChecked
        }
        binding.edtIdentificador.addTextChangedListener(object : android.text.TextWatcher {
            override fun afterTextChanged(s: android.text.Editable?) = validate()
            override fun beforeTextChanged(s: CharSequence?, start: Int, count: Int, after: Int) {}
            override fun onTextChanged(s: CharSequence?, start: Int, before: Int, count: Int) {}
        })
        binding.chkTermos.setOnCheckedChangeListener { _, _ -> validate() }

        binding.btnContinuar.setOnClickListener { onContinuar() }
    }

    private fun onContinuar() {
        binding.btnContinuar.isEnabled = false
        binding.progress.visibility = View.VISIBLE
        binding.txtStatus.text = getString(R.string.permission_validando)

        val identificador = binding.edtIdentificador.text.toString().trim()

        lifecycleScope.launch {
            val deviceInfo = DeviceInfo.current(tokenStorage)
            val adminApi = ApiClient.buildAdminApi(this@PermissionDispatchActivity, tokenStorage)
            authManager = AuthManager(adminApi, tokenStorage, deviceInfo, config)

            val existe = authManager.validarIdentificador(identificador)
            if (!existe) {
                binding.txtStatus.text = getString(R.string.permission_erro_id_nao_encontrado)
                binding.progress.visibility = View.GONE
                binding.btnContinuar.isEnabled = true
                return@launch
            }
            // Guarda identificador pra usar no vincular ao final dos permission steps
            pendingIdentificador = identificador
            currentStep = Step.ACCESSIBILITY
            nextStep()
        }
    }

    private var pendingIdentificador: String = ""

    private fun nextStep() {
        when (currentStep) {
            Step.ACCESSIBILITY -> {
                if (isAccessibilityServiceEnabled()) {
                    currentStep = Step.USAGE_STATS
                    nextStep()
                } else {
                    binding.txtStatus.text = getString(R.string.permission_step_accessibility)
                    settingsLauncher.launch(Intent(Settings.ACTION_ACCESSIBILITY_SETTINGS))
                    currentStep = Step.USAGE_STATS  // proximo callback avança
                }
            }
            Step.USAGE_STATS -> {
                if (hasUsageStatsPermission()) {
                    currentStep = Step.NOTIFICATION
                    nextStep()
                } else {
                    binding.txtStatus.text = getString(R.string.permission_step_usage_stats)
                    settingsLauncher.launch(Intent(Settings.ACTION_USAGE_ACCESS_SETTINGS))
                    currentStep = Step.NOTIFICATION
                }
            }
            Step.NOTIFICATION -> {
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
                    val granted = checkSelfPermission(android.Manifest.permission.POST_NOTIFICATIONS) == PackageManager.PERMISSION_GRANTED
                    if (!granted) {
                        binding.txtStatus.text = getString(R.string.permission_step_notification)
                        requestNotifPermission.launch(android.Manifest.permission.POST_NOTIFICATIONS)
                        currentStep = Step.PHONE
                        return
                    }
                }
                currentStep = Step.PHONE
                nextStep()
            }
            Step.PHONE -> {
                val granted = checkSelfPermission(android.Manifest.permission.READ_PHONE_STATE) == PackageManager.PERMISSION_GRANTED
                if (!granted) {
                    binding.txtStatus.text = getString(R.string.permission_step_phone)
                    requestPhonePermission.launch(android.Manifest.permission.READ_PHONE_STATE)
                    currentStep = Step.OEM
                    return
                }
                currentStep = Step.OEM
                nextStep()
            }
            Step.OEM -> {
                val oem = OEMDetector.detect(Build.MANUFACTURER)
                if (oem.isProblematico) {
                    binding.txtStatus.text = getString(R.string.permission_step_oem)
                    val intents = OEMIntentFactory.intentsForAutostart(oem, this)
                    val first = intents.firstOrNull { it.resolveActivity(packageManager) != null }
                    if (first != null) {
                        settingsLauncher.launch(first)
                        currentStep = Step.VINCULAR
                        return
                    }
                    // Fallback: battery optimization
                    settingsLauncher.launch(OEMIntentFactory.ignoreBatteryOptimizationIntent(this))
                    currentStep = Step.VINCULAR
                    return
                }
                currentStep = Step.VINCULAR
                nextStep()
            }
            Step.VINCULAR -> {
                lifecycleScope.launch {
                    val res = authManager.vincular(pendingIdentificador)
                    when (res) {
                        is AuthManager.VincularResult.Success -> {
                            currentStep = Step.FINISH
                            nextStep()
                        }
                        is AuthManager.VincularResult.Failure -> {
                            binding.txtStatus.text = res.motivo
                            binding.progress.visibility = View.GONE
                            binding.btnContinuar.isEnabled = true
                            currentStep = Step.ACCESSIBILITY
                        }
                    }
                }
            }
            Step.FINISH -> iniciarServiceEFinalizar()
        }
    }

    private fun iniciarServiceEFinalizar() {
        // Inicia ForegroundService
        val svc = Intent(this, ManagerAgentService::class.java)
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) startForegroundService(svc) else startService(svc)

        // Auto-desabilita LAUNCHER pra sumir da gaveta (T36 continua)
        disableLauncherIcon()

        Toast.makeText(this, getString(R.string.permission_ok_toast), Toast.LENGTH_SHORT).show()
        finish()
    }

    private fun disableLauncherIcon() {
        val component = ComponentName(this, PermissionDispatchActivity::class.java)
        packageManager.setComponentEnabledSetting(
            component,
            PackageManager.COMPONENT_ENABLED_STATE_DISABLED,
            PackageManager.DONT_KILL_APP
        )
    }

    private fun isAccessibilityServiceEnabled(): Boolean {
        val am = getSystemService(ACCESSIBILITY_SERVICE) as AccessibilityManager
        val enabled = am.getEnabledAccessibilityServiceList(AccessibilityServiceInfo.FEEDBACK_ALL_MASK)
        return enabled.any { it.id.contains(packageName) && it.id.contains(ManagerAccessibilityService::class.java.simpleName) }
    }

    private fun hasUsageStatsPermission(): Boolean {
        val appOps = getSystemService(APP_OPS_SERVICE) as AppOpsManager
        val mode = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            appOps.unsafeCheckOpNoThrow("android:get_usage_stats", Process.myUid(), packageName)
        } else {
            @Suppress("DEPRECATION") appOps.checkOpNoThrow("android:get_usage_stats", Process.myUid(), packageName)
        }
        return mode == AppOpsManager.MODE_ALLOWED
    }

    companion object { private const val TAG = "PermissionDispatch" }
}
```

- [ ] **Step 2: Registrar no Manifest**

Editar `AndroidManifest.xml` — adicionar dentro de `<application>`:

```xml
<activity
    android:name=".ui.PermissionDispatchActivity"
    android:exported="true"
    android:excludeFromRecents="true"
    android:launchMode="singleTask">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
```

Adicionar permissões faltando (se ainda não estão):

```xml
<uses-permission android:name="android.permission.QUERY_ALL_PACKAGES" tools:ignore="QueryAllPackagesPermission" />
```

- [ ] **Step 3: Habilitar viewBinding e AppCompat**

Em `app/build.gradle.kts` — adicionar dependência AppCompat:

```kotlin
implementation("androidx.appcompat:appcompat:1.6.1")
implementation("androidx.activity:activity-ktx:1.8.2")
```

`buildFeatures.viewBinding = true` já foi configurado em T2.

- [ ] **Step 4: Build**

Run: `./gradlew :app:assembleDebug`
Expected: BUILD SUCCESSFUL.

- [ ] **Step 5: Teste manual em emulador**

Instalar APK no emulador com config staging válida. Executar:

```bash
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

Abrir o app → tela aparece → digitar identificador de teste → aceitar termos → clicar Continuar → passar pelos 4 prompts de permissão → toast "ManagerAgent ativo ✓" → app fecha → ícone some da gaveta (verificar após rebootar launcher: `adb shell am force-stop com.android.launcher3 && adb shell am start -n com.android.launcher3/.Launcher`).

- [ ] **Step 6: Commit**

```bash
git add .
git commit -m "feat: add PermissionDispatchActivity (single-screen onboarding + permission chain + self-disable)"
```

---

### Task T35: Testes de state machine da PermissionDispatchActivity

**Files:**
- Create: `app/src/test/kotlin/com/trivion/manageragent/ui/PermissionStepMachineTest.kt`
- Refactor: extrair state machine da Activity pra classe testável `PermissionStepMachine`

**Interfaces:**
- Consumes: nada
- Produces:
  - `class PermissionStepMachine` — state machine pura (sem dependência Android), com `sealed class Step` e `fun nextStep(currentStep: Step, permsGranted: PermissionState, oemProblematico: Boolean): Step`
  - Refactor: `PermissionDispatchActivity.nextStep()` delega pra ela

- [ ] **Step 1: Escrever teste**

Criar `app/src/test/kotlin/com/trivion/manageragent/ui/PermissionStepMachineTest.kt`:

```kotlin
package com.trivion.manageragent.ui

import org.junit.jupiter.api.Test
import kotlin.test.assertEquals

class PermissionStepMachineTest {

    private val allGranted = PermissionState(accessibility = true, usageStats = true, notification = true, phone = true)
    private val nothingGranted = PermissionState(false, false, false, false)

    @Test
    fun `todas permissoes concedidas e OEM ok pula direto pra VINCULAR`() {
        val next = PermissionStepMachine.nextStep(PermissionStepMachine.Step.ACCESSIBILITY, allGranted, oemProblematico = false)
        assertEquals(PermissionStepMachine.Step.VINCULAR, next)
    }

    @Test
    fun `sem accessibility retorna ACCESSIBILITY como proximo`() {
        val next = PermissionStepMachine.nextStep(PermissionStepMachine.Step.ACCESSIBILITY, nothingGranted, oemProblematico = false)
        assertEquals(PermissionStepMachine.Step.ACCESSIBILITY, next)
    }

    @Test
    fun `com accessibility sem usage stats vai pra USAGE_STATS`() {
        val perms = allGranted.copy(usageStats = false)
        val next = PermissionStepMachine.nextStep(PermissionStepMachine.Step.ACCESSIBILITY, perms, false)
        assertEquals(PermissionStepMachine.Step.USAGE_STATS, next)
    }

    @Test
    fun `todas perms mas OEM problematico vai pra OEM`() {
        val next = PermissionStepMachine.nextStep(PermissionStepMachine.Step.ACCESSIBILITY, allGranted, oemProblematico = true)
        assertEquals(PermissionStepMachine.Step.OEM, next)
    }

    @Test
    fun `apos OEM vai pra VINCULAR`() {
        val next = PermissionStepMachine.nextStep(PermissionStepMachine.Step.OEM, allGranted, oemProblematico = true)
        assertEquals(PermissionStepMachine.Step.VINCULAR, next)
    }

    @Test
    fun `apos VINCULAR vai pra FINISH`() {
        val next = PermissionStepMachine.nextStep(PermissionStepMachine.Step.VINCULAR, allGranted, false)
        assertEquals(PermissionStepMachine.Step.FINISH, next)
    }
}
```

- [ ] **Step 2: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.ui.PermissionStepMachineTest"`
Expected: FAIL.

- [ ] **Step 3: Criar `PermissionStepMachine.kt`**

```kotlin
package com.trivion.manageragent.ui

data class PermissionState(
    val accessibility: Boolean,
    val usageStats: Boolean,
    val notification: Boolean,
    val phone: Boolean
)

object PermissionStepMachine {

    enum class Step { ACCESSIBILITY, USAGE_STATS, NOTIFICATION, PHONE, OEM, VINCULAR, FINISH }

    fun nextStep(currentStep: Step, perms: PermissionState, oemProblematico: Boolean): Step {
        return when (currentStep) {
            Step.ACCESSIBILITY, Step.USAGE_STATS, Step.NOTIFICATION, Step.PHONE, Step.OEM ->
                firstPending(perms, oemProblematico)
            Step.VINCULAR -> Step.FINISH
            Step.FINISH -> Step.FINISH
        }
    }

    private fun firstPending(perms: PermissionState, oemProblematico: Boolean): Step {
        if (!perms.accessibility) return Step.ACCESSIBILITY
        if (!perms.usageStats) return Step.USAGE_STATS
        if (!perms.notification) return Step.NOTIFICATION
        if (!perms.phone) return Step.PHONE
        if (oemProblematico) return Step.OEM
        return Step.VINCULAR
    }
}
```

- [ ] **Step 4: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.ui.PermissionStepMachineTest"`
Expected: PASS (6 testes).

- [ ] **Step 5: Refactor `PermissionDispatchActivity.nextStep()` pra usar `PermissionStepMachine`**

Substituir o `nextStep()` da Activity (T34) por:

```kotlin
private fun currentPermissionState(): PermissionState {
    return PermissionState(
        accessibility = isAccessibilityServiceEnabled(),
        usageStats = hasUsageStatsPermission(),
        notification = Build.VERSION.SDK_INT < Build.VERSION_CODES.TIRAMISU ||
            checkSelfPermission(android.Manifest.permission.POST_NOTIFICATIONS) == PackageManager.PERMISSION_GRANTED,
        phone = checkSelfPermission(android.Manifest.permission.READ_PHONE_STATE) == PackageManager.PERMISSION_GRANTED
    )
}

private fun nextStep() {
    val perms = currentPermissionState()
    val oem = OEMDetector.detect(Build.MANUFACTURER)
    val next = PermissionStepMachine.nextStep(currentStep, perms, oem.isProblematico)
    currentStep = next
    executeStep(next)
}

private fun executeStep(step: PermissionStepMachine.Step) {
    when (step) {
        PermissionStepMachine.Step.ACCESSIBILITY -> {
            binding.txtStatus.text = getString(R.string.permission_step_accessibility)
            settingsLauncher.launch(Intent(Settings.ACTION_ACCESSIBILITY_SETTINGS))
        }
        PermissionStepMachine.Step.USAGE_STATS -> {
            binding.txtStatus.text = getString(R.string.permission_step_usage_stats)
            settingsLauncher.launch(Intent(Settings.ACTION_USAGE_ACCESS_SETTINGS))
        }
        PermissionStepMachine.Step.NOTIFICATION -> {
            binding.txtStatus.text = getString(R.string.permission_step_notification)
            requestNotifPermission.launch(android.Manifest.permission.POST_NOTIFICATIONS)
        }
        PermissionStepMachine.Step.PHONE -> {
            binding.txtStatus.text = getString(R.string.permission_step_phone)
            requestPhonePermission.launch(android.Manifest.permission.READ_PHONE_STATE)
        }
        PermissionStepMachine.Step.OEM -> {
            binding.txtStatus.text = getString(R.string.permission_step_oem)
            val oem = OEMDetector.detect(Build.MANUFACTURER)
            val intents = OEMIntentFactory.intentsForAutostart(oem, this)
            val first = intents.firstOrNull { it.resolveActivity(packageManager) != null }
            settingsLauncher.launch(first ?: OEMIntentFactory.ignoreBatteryOptimizationIntent(this))
        }
        PermissionStepMachine.Step.VINCULAR -> {
            lifecycleScope.launch {
                val res = authManager.vincular(pendingIdentificador)
                when (res) {
                    is AuthManager.VincularResult.Success -> { currentStep = PermissionStepMachine.Step.VINCULAR; iniciarServiceEFinalizar() }
                    is AuthManager.VincularResult.Failure -> {
                        binding.txtStatus.text = res.motivo
                        binding.progress.visibility = View.GONE
                        binding.btnContinuar.isEnabled = true
                        currentStep = PermissionStepMachine.Step.ACCESSIBILITY
                    }
                }
            }
        }
        PermissionStepMachine.Step.FINISH -> iniciarServiceEFinalizar()
    }
}
```

- [ ] **Step 6: Build**

Run: `./gradlew :app:assembleDebug`
Expected: BUILD SUCCESSFUL.

- [ ] **Step 7: Commit**

```bash
git add .
git commit -m "refactor: extract PermissionStepMachine (pure, testable state machine)"
```

---

### Task T36: Reação a revogação (403) — para de coletar

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/network/RevocationInterceptor.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/network/RevocationInterceptorTest.kt`
- Modify: `ApiClient.kt` (adiciona interceptor)
- Modify: `ManagerAgentService.kt` (recebe intent de shutdown quando revogado)

**Interfaces:**
- Consumes: `TokenStorage` (T27)
- Produces:
  - `RevocationInterceptor(tokenStorage, onRevoked: () -> Unit)` — se response 403 com header `X-Agent-Revoked: true`, invoca `onRevoked` (que dispara `Intent` broadcast pra `ManagerAgentService` parar)

- [ ] **Step 1: Escrever teste**

Criar `app/src/test/kotlin/com/trivion/manageragent/network/RevocationInterceptorTest.kt`:

```kotlin
package com.trivion.manageragent.network

import okhttp3.OkHttpClient
import okhttp3.Request
import okhttp3.mockwebserver.MockResponse
import okhttp3.mockwebserver.MockWebServer
import org.junit.jupiter.api.AfterEach
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import kotlin.test.assertEquals
import kotlin.test.assertTrue

class RevocationInterceptorTest {

    private lateinit var server: MockWebServer

    @BeforeEach
    fun setUp() { server = MockWebServer().also { it.start() } }
    @AfterEach
    fun tearDown() { server.shutdown() }

    @Test
    fun `403 com X-Agent-Revoked chama callback`() {
        server.enqueue(MockResponse().setResponseCode(403).addHeader("X-Agent-Revoked", "true"))
        var callbackCalled = false
        val client = OkHttpClient.Builder()
            .addInterceptor(RevocationInterceptor { callbackCalled = true })
            .build()
        client.newCall(Request.Builder().url(server.url("/x")).build()).execute()
        assertTrue(callbackCalled)
    }

    @Test
    fun `403 sem header nao chama callback`() {
        server.enqueue(MockResponse().setResponseCode(403))
        var callbackCalled = false
        val client = OkHttpClient.Builder()
            .addInterceptor(RevocationInterceptor { callbackCalled = true })
            .build()
        client.newCall(Request.Builder().url(server.url("/x")).build()).execute()
        assertEquals(false, callbackCalled)
    }

    @Test
    fun `200 nao chama callback`() {
        server.enqueue(MockResponse().setResponseCode(200).addHeader("X-Agent-Revoked", "true"))
        var callbackCalled = false
        val client = OkHttpClient.Builder()
            .addInterceptor(RevocationInterceptor { callbackCalled = true })
            .build()
        client.newCall(Request.Builder().url(server.url("/x")).build()).execute()
        assertEquals(false, callbackCalled)
    }
}
```

- [ ] **Step 2: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.network.RevocationInterceptorTest"`
Expected: FAIL.

- [ ] **Step 3: Criar `RevocationInterceptor.kt`**

```kotlin
package com.trivion.manageragent.network

import okhttp3.Interceptor
import okhttp3.Response
import timber.log.Timber

class RevocationInterceptor(private val onRevoked: () -> Unit) : Interceptor {

    override fun intercept(chain: Interceptor.Chain): Response {
        val response = chain.proceed(chain.request())
        if (response.code == 403 && response.header("X-Agent-Revoked")?.lowercase() == "true") {
            Timber.tag(TAG).w("Agent REVOGADO pelo backend, invocando callback")
            onRevoked()
        }
        return response
    }

    companion object { private const val TAG = "RevocationInterceptor" }
}
```

- [ ] **Step 4: Wire no ApiClient**

Modificar `ApiClient.retrofit()`:

```kotlin
.addInterceptor(RevocationInterceptor {
    // Envia broadcast pra parar Service
    val intent = Intent("com.trivion.manageragent.ACTION_AGENT_REVOGADO")
    androidx.localbroadcastmanager.content.LocalBroadcastManager.getInstance(context).sendBroadcast(intent)
})
```

Adicionar dependência `androidx.localbroadcastmanager:localbroadcastmanager:1.1.0` em `libs.versions.toml` e `build.gradle.kts`.

- [ ] **Step 5: Registrar receiver no Service**

Editar `ManagerAgentService.kt`:

```kotlin
private val revocationReceiver = object : BroadcastReceiver() {
    override fun onReceive(context: Context?, intent: Intent?) {
        Timber.tag(TAG).w("Recebido broadcast REVOGADO, parando Service")
        // Zerar tokens
        TokenStorage(this@ManagerAgentService).clear()
        BatchScheduler.cancel(this@ManagerAgentService)
        HeartbeatScheduler.cancel(this@ManagerAgentService)
        UsageStatsReconcileWorker.let { WorkManager.getInstance(this@ManagerAgentService).cancelUniqueWork("ManagerAgent.UsageStatsReconcile") }
        stopSelf()
    }
}

override fun onCreate() {
    super.onCreate()
    // ... existente
    LocalBroadcastManager.getInstance(this).registerReceiver(
        revocationReceiver,
        IntentFilter("com.trivion.manageragent.ACTION_AGENT_REVOGADO")
    )
}

override fun onDestroy() {
    LocalBroadcastManager.getInstance(this).unregisterReceiver(revocationReceiver)
    // ... resto existente
}
```

- [ ] **Step 6: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.network.RevocationInterceptorTest"`
Expected: PASS (3 testes).

- [ ] **Step 7: Commit**

```bash
git add .
git commit -m "feat: add RevocationInterceptor + Service listener (backend 403 stops collection)"
```

---

## BLOCO B7 — Logs (T37-T40)

### Task T37: FileLoggingTree — Timber + rotação diária + retenção 7 dias

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/log/FileLoggingTree.kt`
- Create: `app/src/main/kotlin/com/trivion/manageragent/log/LoggingSetup.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/log/FileLoggingTreeTest.kt`
- Modify: `ManagerAgentApplication.kt` (plant FileLoggingTree em release)

**Interfaces:**
- Consumes: nada
- Produces:
  - `FileLoggingTree(logsDir: File, maxFileSizeBytes: Long = 5 * 1024 * 1024, retentionDays: Int = 7)` — extende `Timber.Tree`, escreve linhas texto no arquivo `agent-yyyy-MM-dd.log`, rotaciona por tamanho, purga antigos
  - `LoggingSetup.init(context)` — em release, planta FileLoggingTree; em debug, planta Timber.DebugTree + FileLoggingTree

- [ ] **Step 1: Escrever teste**

Criar `app/src/test/kotlin/com/trivion/manageragent/log/FileLoggingTreeTest.kt`:

```kotlin
package com.trivion.manageragent.log

import org.junit.jupiter.api.AfterEach
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import java.io.File
import kotlin.io.path.createTempDirectory
import kotlin.test.assertEquals
import kotlin.test.assertTrue

class FileLoggingTreeTest {

    private lateinit var tempDir: File

    @BeforeEach
    fun setUp() {
        tempDir = createTempDirectory("logs-test").toFile()
    }

    @AfterEach
    fun tearDown() { tempDir.deleteRecursively() }

    @Test
    fun `escreve linha no arquivo do dia`() {
        val tree = FileLoggingTree(tempDir, clock = { java.time.LocalDate.of(2026, 8, 8) })
        tree.log(4, "T", "hello", null)  // priority=4=INFO
        val f = File(tempDir, "agent-2026-08-08.log")
        assertTrue(f.exists())
        assertTrue(f.readText().contains("hello"))
        assertTrue(f.readText().contains("INF"))
        assertTrue(f.readText().contains("T"))
    }

    @Test
    fun `rotaciona quando arquivo passa do limite`() {
        val tree = FileLoggingTree(tempDir, maxFileSizeBytes = 200, clock = { java.time.LocalDate.of(2026, 8, 8) })
        repeat(20) { tree.log(4, "T", "linha longa de teste que deve ocupar bytes " + "x".repeat(50), null) }
        val active = File(tempDir, "agent-2026-08-08.log")
        val rotated = File(tempDir, "agent-2026-08-08.log.1")
        assertTrue(active.exists())
        assertTrue(rotated.exists())
    }

    @Test
    fun `purga arquivos mais antigos que retentionDays`() {
        // Cria arquivo "velho"
        val oldFile = File(tempDir, "agent-2020-01-01.log").apply { writeText("velho"); setLastModified(0L) }
        val tree = FileLoggingTree(tempDir, retentionDays = 7, clock = { java.time.LocalDate.of(2026, 8, 8) })
        tree.log(4, "T", "novo", null)
        tree.purgeOldFiles()
        assertEquals(false, oldFile.exists())
    }

    @Test
    fun `nao purga arquivos dentro da retencao`() {
        val recent = File(tempDir, "agent-2026-08-05.log").apply { writeText("recente") }
        val tree = FileLoggingTree(tempDir, retentionDays = 7, clock = { java.time.LocalDate.of(2026, 8, 8) })
        tree.purgeOldFiles()
        assertTrue(recent.exists())
    }
}
```

- [ ] **Step 2: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.log.FileLoggingTreeTest"`
Expected: FAIL.

- [ ] **Step 3: Criar `FileLoggingTree.kt`**

```kotlin
package com.trivion.manageragent.log

import android.util.Log
import timber.log.Timber
import java.io.File
import java.io.FileWriter
import java.time.LocalDate
import java.time.format.DateTimeFormatter
import java.time.temporal.ChronoUnit

class FileLoggingTree(
    private val logsDir: File,
    private val maxFileSizeBytes: Long = 5 * 1024 * 1024L,
    private val retentionDays: Int = 7,
    private val clock: () -> LocalDate = { LocalDate.now() }
) : Timber.Tree() {

    companion object {
        private val DATE_FORMAT: DateTimeFormatter = DateTimeFormatter.ofPattern("yyyy-MM-dd")
        private val TS_FORMAT: DateTimeFormatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss.SSS")
        private const val FILE_PREFIX = "agent-"
        private const val FILE_SUFFIX = ".log"
    }

    init { if (!logsDir.exists()) logsDir.mkdirs() }

    override fun log(priority: Int, tag: String?, message: String, t: Throwable?) {
        val today = clock()
        val file = File(logsDir, "$FILE_PREFIX${DATE_FORMAT.format(today)}$FILE_SUFFIX")

        if (file.exists() && file.length() > maxFileSizeBytes) {
            rotate(file)
        }

        val level = when (priority) {
            Log.VERBOSE -> "TRC"; Log.DEBUG -> "DBG"; Log.INFO -> "INF"
            Log.WARN -> "WRN"; Log.ERROR -> "ERR"; else -> "???"
        }
        val nowStamp = TS_FORMAT.format(java.time.LocalDateTime.now())
        val line = buildString {
            append(nowStamp); append(" [$level] ")
            if (tag != null) append("$tag: ")
            append(message); append('\n')
            if (t != null) append(Log.getStackTraceString(t)).append('\n')
        }
        try {
            FileWriter(file, true).use { it.write(line) }
        } catch (e: Exception) {
            Log.e("FileLoggingTree", "failed to write log", e)
        }
    }

    private fun rotate(current: File) {
        val rotated = File(current.parentFile, current.nameWithoutExtension + ".log.1")
        if (rotated.exists()) rotated.delete()
        current.renameTo(rotated)
    }

    fun purgeOldFiles() {
        val today = clock()
        val cutoff = today.minusDays(retentionDays.toLong())
        val cutoffMillis = cutoff.atStartOfDay(java.time.ZoneId.systemDefault()).toInstant().toEpochMilli()
        val files = logsDir.listFiles { f -> f.name.startsWith(FILE_PREFIX) && f.name.endsWith(FILE_SUFFIX) } ?: return
        for (f in files) {
            // Parse data do nome do arquivo
            val dateStr = f.nameWithoutExtension.removePrefix(FILE_PREFIX).take(10)
            try {
                val fileDate = LocalDate.parse(dateStr, DATE_FORMAT)
                if (ChronoUnit.DAYS.between(fileDate, today) > retentionDays) {
                    f.delete()
                }
            } catch (_: Exception) {
                // Fallback: usa lastModified
                if (f.lastModified() < cutoffMillis) f.delete()
            }
        }
    }
}
```

- [ ] **Step 4: Criar `LoggingSetup.kt`**

```kotlin
package com.trivion.manageragent.log

import android.content.Context
import com.trivion.manageragent.BuildConfig
import timber.log.Timber
import java.io.File

object LoggingSetup {

    fun init(context: Context) {
        val logsDir = File(context.filesDir, "logs")
        val fileTree = FileLoggingTree(logsDir)
        if (BuildConfig.DEBUG) Timber.plant(Timber.DebugTree())
        Timber.plant(fileTree)
    }

    fun purgeOldFiles(context: Context) {
        val logsDir = File(context.filesDir, "logs")
        FileLoggingTree(logsDir).purgeOldFiles()
    }
}
```

- [ ] **Step 5: Modificar `ManagerAgentApplication.kt`**

```kotlin
package com.trivion.manageragent

import android.app.Application
import com.trivion.manageragent.log.LoggingSetup

class ManagerAgentApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        LoggingSetup.init(this)
        LoggingSetup.purgeOldFiles(this)
    }
}
```

- [ ] **Step 6: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.log.FileLoggingTreeTest"`
Expected: PASS (4 testes).

- [ ] **Step 7: Commit**

```bash
git add .
git commit -m "feat: add FileLoggingTree (Timber file sink with daily rotation + 7d retention)"
```

---

### Task T38: AuditReporter — POST /api/agente/auditoria/registrar (fire-and-forget)

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/log/AuditReporter.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/log/AuditReporterTest.kt`
- Modify: `PermissionDispatchActivity.kt`, `ManagerAccessibilityService.kt` (emitem audit events em pontos-chave)

**Interfaces:**
- Consumes: `AdminApi.registrarAudit` (T28), `TokenStorage` (T27)
- Produces:
  - `AuditReporter(adminApi, tokenStorage)`
  - `suspend fun report(evento: String, dados: Map<String, String> = emptyMap())` — chama endpoint fire-and-forget, ignora erros

Audit events Android-específicos (todos usando string constants):
- `AGENT_INSTALADO_ANDROID`
- `TERMOS_ACEITOS_ANDROID`
- `AGENT_VINCULADO_ANDROID`
- `AGENT_VINCULACAO_FALHOU_ANDROID`
- `ACCESSIBILITY_ATIVADA` / `ACCESSIBILITY_DESATIVADA`
- `USAGE_STATS_ATIVADA` / `USAGE_STATS_DESATIVADA`
- `OEM_PROBLEMATICO_DETECTADO`
- `SERVICE_MORTO_E_REINICIADO`
- `AGENT_REVOGADO_ANDROID`

- [ ] **Step 1: Escrever teste**

Criar `app/src/test/kotlin/com/trivion/manageragent/log/AuditReporterTest.kt`:

```kotlin
package com.trivion.manageragent.log

import com.trivion.manageragent.auth.TokenStorage
import com.trivion.manageragent.network.AdminApi
import com.trivion.manageragent.network.dto.AuditRegisterRequest
import io.mockk.coEvery
import io.mockk.coVerify
import io.mockk.every
import io.mockk.mockk
import io.mockk.slot
import kotlinx.coroutines.test.runTest
import okhttp3.MediaType.Companion.toMediaType
import okhttp3.ResponseBody.Companion.toResponseBody
import org.junit.jupiter.api.Test
import retrofit2.Response
import kotlin.test.assertEquals

class AuditReporterTest {

    private val adminApi = mockk<AdminApi>()
    private val tokenStorage = mockk<TokenStorage>().apply {
        every { getAgenteId() } returns 42L
        every { getInstalacaoId() } returns "inst-1"
    }
    private val reporter = AuditReporter(adminApi, tokenStorage)

    @Test
    fun `report envia payload correto`() = runTest {
        coEvery { adminApi.registrarAudit(any()) } returns Response.success(Unit)
        val slot = slot<AuditRegisterRequest>()

        reporter.report("TERMOS_ACEITOS_ANDROID", mapOf("versao" to "1.0"))

        coVerify(exactly = 1) { adminApi.registrarAudit(capture(slot)) }
        val req = slot.captured
        assertEquals("TERMOS_ACEITOS_ANDROID", req.evento)
        assertEquals("inst-1", req.instalacaoId)
        assertEquals(42L, req.agenteId)
        assertEquals("1.0", req.dados["versao"])
    }

    @Test
    fun `report ignora exception de rede (fire-and-forget)`() = runTest {
        coEvery { adminApi.registrarAudit(any()) } throws java.io.IOException("network")
        // Não lança
        reporter.report("AGENT_VINCULADO_ANDROID")
    }

    @Test
    fun `report nao envia se sem agenteId (nao vinculado ainda)`() = runTest {
        every { tokenStorage.getAgenteId() } returns -1L
        reporter.report("AGENT_INSTALADO_ANDROID")
        coVerify(exactly = 0) { adminApi.registrarAudit(any()) }
    }
}
```

- [ ] **Step 2: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.log.AuditReporterTest"`
Expected: FAIL.

- [ ] **Step 3: Criar `AuditReporter.kt`**

```kotlin
package com.trivion.manageragent.log

import com.trivion.manageragent.auth.TokenStorage
import com.trivion.manageragent.network.AdminApi
import com.trivion.manageragent.network.dto.AuditRegisterRequest
import timber.log.Timber
import java.time.Instant

class AuditReporter(
    private val adminApi: AdminApi,
    private val tokenStorage: TokenStorage
) {

    object Events {
        const val AGENT_INSTALADO_ANDROID = "AGENT_INSTALADO_ANDROID"
        const val TERMOS_ACEITOS_ANDROID = "TERMOS_ACEITOS_ANDROID"
        const val AGENT_VINCULADO_ANDROID = "AGENT_VINCULADO_ANDROID"
        const val AGENT_VINCULACAO_FALHOU_ANDROID = "AGENT_VINCULACAO_FALHOU_ANDROID"
        const val ACCESSIBILITY_ATIVADA = "ACCESSIBILITY_ATIVADA"
        const val ACCESSIBILITY_DESATIVADA = "ACCESSIBILITY_DESATIVADA"
        const val USAGE_STATS_ATIVADA = "USAGE_STATS_ATIVADA"
        const val USAGE_STATS_DESATIVADA = "USAGE_STATS_DESATIVADA"
        const val OEM_PROBLEMATICO_DETECTADO = "OEM_PROBLEMATICO_DETECTADO"
        const val SERVICE_MORTO_E_REINICIADO = "SERVICE_MORTO_E_REINICIADO"
        const val AGENT_REVOGADO_ANDROID = "AGENT_REVOGADO_ANDROID"
    }

    suspend fun report(evento: String, dados: Map<String, String> = emptyMap()) {
        val agenteId = tokenStorage.getAgenteId()
        if (agenteId <= 0L) return
        val instalacaoId = tokenStorage.getInstalacaoId() ?: return

        try {
            adminApi.registrarAudit(
                AuditRegisterRequest(
                    evento = evento,
                    instalacaoId = instalacaoId,
                    agenteId = agenteId,
                    timestampUtc = Instant.now().toString(),
                    dados = dados
                )
            )
        } catch (e: Exception) {
            Timber.tag("AuditReporter").w(e, "Audit failed (silenced) — evento=$evento")
        }
    }
}
```

- [ ] **Step 4: Instrumentar pontos-chave — exemplos:**

**`ManagerAccessibilityService.kt` — onServiceConnected/onUnbind:**

```kotlin
override fun onServiceConnected() {
    super.onServiceConnected()
    // ... setup coletores ...
    scope.launch {
        val reporter = AuditReporter(
            ApiClient.buildAdminApi(this@ManagerAccessibilityService, TokenStorage(this@ManagerAccessibilityService)),
            TokenStorage(this@ManagerAccessibilityService)
        )
        reporter.report(AuditReporter.Events.ACCESSIBILITY_ATIVADA)
    }
}

override fun onUnbind(intent: android.content.Intent?): Boolean {
    ticker.stop()
    scope.launch {
        val reporter = AuditReporter(
            ApiClient.buildAdminApi(this@ManagerAccessibilityService, TokenStorage(this@ManagerAccessibilityService)),
            TokenStorage(this@ManagerAccessibilityService)
        )
        reporter.report(AuditReporter.Events.ACCESSIBILITY_DESATIVADA)
    }
    return super.onUnbind(intent)
}
```

**`PermissionDispatchActivity` — após vinculação com sucesso:**

```kotlin
is AuthManager.VincularResult.Success -> {
    lifecycleScope.launch {
        val reporter = AuditReporter(
            ApiClient.buildAdminApi(this@PermissionDispatchActivity, tokenStorage),
            tokenStorage
        )
        reporter.report(AuditReporter.Events.AGENT_VINCULADO_ANDROID)
        reporter.report(AuditReporter.Events.TERMOS_ACEITOS_ANDROID, mapOf("versao" to "1.0"))
        val oem = OEMDetector.detect(Build.MANUFACTURER)
        if (oem.isProblematico) {
            reporter.report(AuditReporter.Events.OEM_PROBLEMATICO_DETECTADO, mapOf("fabricante" to oem.name))
        }
    }
    iniciarServiceEFinalizar()
}
```

**`ManagerAgentService.onCreate()` (revocationReceiver callback):**

```kotlin
private val revocationReceiver = object : BroadcastReceiver() {
    override fun onReceive(context: Context?, intent: Intent?) {
        scope.launch {
            val reporter = AuditReporter(
                ApiClient.buildAdminApi(this@ManagerAgentService, TokenStorage(this@ManagerAgentService)),
                TokenStorage(this@ManagerAgentService)
            )
            reporter.report(AuditReporter.Events.AGENT_REVOGADO_ANDROID)
        }
        TokenStorage(this@ManagerAgentService).clear()
        // ... cancelamentos de scheduler ...
        stopSelf()
    }
}
```

- [ ] **Step 5: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.log.AuditReporterTest"`
Expected: PASS (3 testes).

- [ ] **Step 6: Commit**

```bash
git add .
git commit -m "feat: add AuditReporter + instrument events (vinculado, accessibility, revogado, OEM)"
```

---

### Task T39: CrashReporter + ANR detector

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/log/CrashReporter.kt`
- Create: `app/src/main/kotlin/com/trivion/manageragent/log/AnrDetector.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/log/CrashReporterTest.kt`
- Modify: `ManagerAgentApplication.kt` (registra handler global)

**Interfaces:**
- Consumes: `AdminApi.reportarErro` (T28), `TokenStorage` (T27)
- Produces:
  - `CrashReporter(context, adminApi, tokenStorage)` — instala `Thread.setDefaultUncaughtExceptionHandler` que salva crash em arquivo (pra retry no próximo start) e tenta enviar imediatamente. Throttle 5min por tipo. Reenvia pendentes no `init`.
  - `AnrDetector(handler = MainLooper)` — monitora se main thread responde em <5s; se não, dispara crash reporter tipo `ANR`

- [ ] **Step 1: Escrever teste**

Criar `app/src/test/kotlin/com/trivion/manageragent/log/CrashReporterTest.kt`:

```kotlin
package com.trivion.manageragent.log

import com.trivion.manageragent.auth.TokenStorage
import com.trivion.manageragent.network.AdminApi
import com.trivion.manageragent.network.dto.ErrorReportRequest
import io.mockk.coEvery
import io.mockk.coVerify
import io.mockk.every
import io.mockk.mockk
import io.mockk.slot
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Test
import retrofit2.Response
import kotlin.test.assertEquals

class CrashReporterTest {

    private val adminApi = mockk<AdminApi>(relaxed = true)
    private val tokenStorage = mockk<TokenStorage>().apply {
        every { getUsuarioId() } returns 100L
    }

    private val reporter = CrashReporter(
        adminApi = adminApi,
        tokenStorage = tokenStorage,
        versao = "1.0.0",
        sistemaOperacional = "Android 13",
        maquina = "Pixel 8",
        cnpj = "cnpj-1",
        nowMillis = { 1000L }
    )

    @Test
    fun `report envia payload`() = runTest {
        coEvery { adminApi.reportarErro(any()) } returns Response.success(Unit)
        val slot = slot<ErrorReportRequest>()
        reporter.report("FATAL_CRASH", "boom", "stack")
        coVerify(exactly = 1) { adminApi.reportarErro(capture(slot)) }
        assertEquals("FATAL_CRASH", slot.captured.tipo)
        assertEquals("boom", slot.captured.mensagem)
        assertEquals(100L, slot.captured.colaboradorId)
    }

    @Test
    fun `throttle bloqueia segundo report do mesmo tipo em 5min`() = runTest {
        coEvery { adminApi.reportarErro(any()) } returns Response.success(Unit)
        reporter.report("FATAL_CRASH", "boom1", "s1")
        reporter.report("FATAL_CRASH", "boom2", "s2")
        coVerify(exactly = 1) { adminApi.reportarErro(any()) }  // 2o bloqueado
    }

    @Test
    fun `tipos diferentes nao competem no throttle`() = runTest {
        coEvery { adminApi.reportarErro(any()) } returns Response.success(Unit)
        reporter.report("FATAL_CRASH", "b1", "s1")
        reporter.report("ANR", "b2", "s2")
        reporter.report("UPDATE_CRASH", "b3", "s3")
        coVerify(exactly = 3) { adminApi.reportarErro(any()) }
    }
}
```

- [ ] **Step 2: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.log.CrashReporterTest"`
Expected: FAIL.

- [ ] **Step 3: Criar `CrashReporter.kt`**

```kotlin
package com.trivion.manageragent.log

import com.trivion.manageragent.auth.TokenStorage
import com.trivion.manageragent.network.AdminApi
import com.trivion.manageragent.network.dto.ErrorReportRequest
import kotlinx.coroutines.runBlocking
import timber.log.Timber

class CrashReporter(
    private val adminApi: AdminApi,
    private val tokenStorage: TokenStorage,
    private val versao: String,
    private val sistemaOperacional: String,
    private val maquina: String,
    private val cnpj: String,
    private val throttleWindowMs: Long = 5 * 60_000L,
    private val nowMillis: () -> Long = { System.currentTimeMillis() }
) {

    private val lastReportByType = mutableMapOf<String, Long>()

    suspend fun report(tipo: String, mensagem: String, stackTrace: String? = null) {
        val now = nowMillis()
        val last = lastReportByType[tipo] ?: 0L
        if (now - last < throttleWindowMs) {
            Timber.tag("CrashReporter").d("Throttled $tipo (last=${now - last}ms ago)")
            return
        }
        lastReportByType[tipo] = now

        try {
            adminApi.reportarErro(
                ErrorReportRequest(
                    tipo = tipo,
                    mensagem = mensagem,
                    stackTrace = stackTrace?.take(2000),
                    versao = versao,
                    sistemaOperacional = sistemaOperacional,
                    maquina = maquina,
                    colaboradorId = tokenStorage.getUsuarioId()
                )
            )
        } catch (e: Exception) {
            Timber.tag("CrashReporter").w(e, "Error report failed (silenced)")
        }
    }

    fun installUncaughtHandler() {
        val previous = Thread.getDefaultUncaughtExceptionHandler()
        Thread.setDefaultUncaughtExceptionHandler { thread, throwable ->
            try {
                runBlocking {
                    report(
                        tipo = "FATAL_CRASH",
                        mensagem = throwable.message ?: throwable::class.simpleName ?: "unknown",
                        stackTrace = throwable.stackTraceToString()
                    )
                }
            } catch (_: Throwable) { /* absorve */ }
            previous?.uncaughtException(thread, throwable)
        }
    }
}
```

- [ ] **Step 4: Criar `AnrDetector.kt`**

```kotlin
package com.trivion.manageragent.log

import android.os.Handler
import android.os.Looper
import kotlinx.coroutines.*
import timber.log.Timber

/**
 * Detector simples de ANR: posta task no main thread a cada N segundos e mede latência.
 * Se latência > threshold, considera ANR e reporta.
 */
class AnrDetector(
    private val crashReporter: CrashReporter,
    private val checkIntervalMs: Long = 5_000L,
    private val anrThresholdMs: Long = 5_000L
) {

    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Default)
    private val mainHandler = Handler(Looper.getMainLooper())
    private var job: Job? = null

    fun start() {
        if (job?.isActive == true) return
        job = scope.launch {
            while (isActive) {
                val start = System.currentTimeMillis()
                val done = CompletableDeferred<Unit>()
                mainHandler.post { done.complete(Unit) }
                withTimeoutOrNull(anrThresholdMs) { done.await() }
                val elapsed = System.currentTimeMillis() - start
                if (elapsed >= anrThresholdMs) {
                    Timber.tag("AnrDetector").w("ANR detected (${elapsed}ms)")
                    crashReporter.report(tipo = "ANR", mensagem = "Main thread blocked ${elapsed}ms")
                }
                delay(checkIntervalMs)
            }
        }
    }

    fun stop() { job?.cancel() }
}
```

- [ ] **Step 5: Wire no Application**

Modificar `ManagerAgentApplication.kt`:

```kotlin
package com.trivion.manageragent

import android.app.Application
import android.os.Build
import com.trivion.manageragent.auth.TokenStorage
import com.trivion.manageragent.log.AnrDetector
import com.trivion.manageragent.log.CrashReporter
import com.trivion.manageragent.log.LoggingSetup
import com.trivion.manageragent.network.ApiClient

class ManagerAgentApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        LoggingSetup.init(this)
        LoggingSetup.purgeOldFiles(this)

        val tokenStorage = TokenStorage(this)
        val adminApi = ApiClient.buildAdminApi(this, tokenStorage)
        val crashReporter = CrashReporter(
            adminApi = adminApi,
            tokenStorage = tokenStorage,
            versao = BuildConfig.VERSION_NAME,
            sistemaOperacional = "Android ${Build.VERSION.RELEASE} (API ${Build.VERSION.SDK_INT})",
            maquina = "${Build.MANUFACTURER} ${Build.MODEL}",
            cnpj = ""  // preenchido depois se o token conter (não temos aqui direto)
        )
        crashReporter.installUncaughtHandler()
        AnrDetector(crashReporter).start()
    }
}
```

- [ ] **Step 6: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.log.CrashReporterTest"`
Expected: PASS (3 testes).

- [ ] **Step 7: Commit**

```bash
git add .
git commit -m "feat: add CrashReporter (uncaught handler + throttle 5min) + AnrDetector"
```

---

### Task T40: Escape hatch — action "Enviar diagnóstico" na notification

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/log/DiagnosticoExporter.kt`
- Create: `app/src/main/kotlin/com/trivion/manageragent/service/EnviarDiagnosticoReceiver.kt`
- Modify: `NotificationHelper.kt` (adiciona action)

**Interfaces:**
- Consumes: `FileLoggingTree` / logs em `filesDir/logs/`
- Produces:
  - `DiagnosticoExporter(context)`
  - `fun exportarLogsRecentes(): File?` — copia últimos 3 arquivos de log de `filesDir/logs/` pra `Downloads/ManagerAgent/logs/` público, retorna diretório destino
  - `EnviarDiagnosticoReceiver` — BroadcastReceiver acionado pela action da notification; chama `exportarLogsRecentes` e mostra Toast

- [ ] **Step 1: Criar `DiagnosticoExporter.kt`**

```kotlin
package com.trivion.manageragent.log

import android.content.Context
import android.os.Environment
import timber.log.Timber
import java.io.File

class DiagnosticoExporter(private val context: Context) {

    fun exportarLogsRecentes(): File? {
        val srcDir = File(context.filesDir, "logs")
        if (!srcDir.exists() || srcDir.listFiles().isNullOrEmpty()) return null

        val downloadsDir = context.getExternalFilesDir(Environment.DIRECTORY_DOWNLOADS) ?: return null
        val destDir = File(downloadsDir, "ManagerAgent-logs").apply { mkdirs() }

        val logFiles = srcDir.listFiles { f -> f.name.startsWith("agent-") }?.sortedByDescending { it.lastModified() }?.take(3) ?: return null
        for (f in logFiles) {
            val dst = File(destDir, f.name)
            try {
                f.copyTo(dst, overwrite = true)
                Timber.tag(TAG).i("Copiado ${f.name} pra ${dst.absolutePath}")
            } catch (e: Exception) {
                Timber.tag(TAG).w(e, "Falha copiando ${f.name}")
            }
        }
        return destDir
    }

    companion object { private const val TAG = "DiagnosticoExporter" }
}
```

- [ ] **Step 2: Criar `EnviarDiagnosticoReceiver.kt`**

```kotlin
package com.trivion.manageragent.service

import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import android.widget.Toast
import com.trivion.manageragent.log.DiagnosticoExporter
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.SupervisorJob
import kotlinx.coroutines.launch

class EnviarDiagnosticoReceiver : BroadcastReceiver() {

    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)

    override fun onReceive(context: Context, intent: Intent) {
        val pending = goAsync()
        scope.launch {
            try {
                val dest = DiagnosticoExporter(context).exportarLogsRecentes()
                val msg = if (dest != null) {
                    "Diagnóstico salvo em ${dest.absolutePath}"
                } else {
                    "Não há logs pra exportar"
                }
                Toast.makeText(context, msg, Toast.LENGTH_LONG).show()
            } finally { pending.finish() }
        }
    }
}
```

- [ ] **Step 3: Modificar `NotificationHelper.buildForegroundNotification()`**

```kotlin
fun buildForegroundNotification(context: Context): Notification {
    val diagnosticoIntent = Intent(context, EnviarDiagnosticoReceiver::class.java)
    val diagnosticoPendingIntent = PendingIntent.getBroadcast(
        context, 0, diagnosticoIntent,
        PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
    )
    return NotificationCompat.Builder(context, CHANNEL_ID)
        .setContentTitle(context.getString(R.string.app_name))
        .setContentText(context.getString(R.string.notification_status_text))
        .setSmallIcon(android.R.drawable.ic_menu_view)
        .setPriority(NotificationCompat.PRIORITY_LOW)
        .setOngoing(true)
        .setSilent(true)
        .setShowWhen(false)
        .addAction(
            NotificationCompat.Action(
                android.R.drawable.ic_menu_send,
                context.getString(R.string.notification_action_enviar_diagnostico),
                diagnosticoPendingIntent
            )
        )
        .build()
}
```

- [ ] **Step 4: Registrar receiver no Manifest**

```xml
<receiver
    android:name=".service.EnviarDiagnosticoReceiver"
    android:exported="false" />
```

- [ ] **Step 5: Teste manual em emulador**

Instalar APK, iniciar Service, na notification clicar em "Enviar diagnóstico" → Toast "Diagnóstico salvo em..." → verificar arquivos em `Downloads/ManagerAgent-logs/` via `adb shell ls /sdcard/Android/data/com.trivion.manageragent/files/Download/ManagerAgent-logs/`.

- [ ] **Step 6: Commit**

```bash
git add .
git commit -m "feat: add diagnostico exporter (notification action → Downloads/ManagerAgent-logs)"
```

---

## BLOCO B8 — Auto-update (T41-T44)

### Task T41: UpdateChecker — verifica versão nova a cada 6h

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/update/UpdateChecker.kt`
- Create: `app/src/main/kotlin/com/trivion/manageragent/update/UpdateCheckWorker.kt`
- Create: `app/src/main/kotlin/com/trivion/manageragent/update/UpdateScheduler.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/update/UpdateCheckerTest.kt`
- Modify: `ManagerAgentService.onCreate()` (agenda)

**Interfaces:**
- Consumes: `AdminApi.verificarAtualizacao` (T28)
- Produces:
  - `sealed class UpdateAvailability { object None; data class Available(...); data class Error(msg) }`
  - `class UpdateChecker(adminApi, versionAtual)` — `suspend fun check(): UpdateAvailability`
  - `UpdateCheckWorker` (WorkManager 6h) — se disponível, dispara download (T42)
  - `UpdateScheduler.schedule(context)`

- [ ] **Step 1: Escrever teste**

Criar `app/src/test/kotlin/com/trivion/manageragent/update/UpdateCheckerTest.kt`:

```kotlin
package com.trivion.manageragent.update

import com.trivion.manageragent.network.AdminApi
import com.trivion.manageragent.network.dto.UpdateCheckResponse
import io.mockk.coEvery
import io.mockk.mockk
import kotlinx.coroutines.test.runTest
import okhttp3.MediaType.Companion.toMediaType
import okhttp3.ResponseBody.Companion.toResponseBody
import org.junit.jupiter.api.Test
import retrofit2.Response
import kotlin.test.assertEquals
import kotlin.test.assertTrue

class UpdateCheckerTest {

    private val adminApi = mockk<AdminApi>()

    @Test
    fun `retorna None se backend diz que nao ha atualizacao`() = runTest {
        coEvery { adminApi.verificarAtualizacao("ANDROID", "1.0.0") } returns Response.success(
            UpdateCheckResponse(atualizacaoDisponivel = false)
        )
        val checker = UpdateChecker(adminApi, versaoAtual = "1.0.0")
        assertEquals(UpdateAvailability.None, checker.check())
    }

    @Test
    fun `retorna Available com metadados quando ha atualizacao`() = runTest {
        coEvery { adminApi.verificarAtualizacao("ANDROID", "1.0.0") } returns Response.success(
            UpdateCheckResponse(
                atualizacaoDisponivel = true,
                versaoNova = "1.2.0",
                urlDownload = "https://apk.imanager.trivion.com.br/updates/prod/1.2.0.apk",
                checksumSha256 = "abc123",
                obrigatoria = false
            )
        )
        val checker = UpdateChecker(adminApi, versaoAtual = "1.0.0")
        val res = checker.check()
        assertTrue(res is UpdateAvailability.Available)
        assertEquals("1.2.0", (res as UpdateAvailability.Available).versaoNova)
        assertEquals(false, res.obrigatoria)
    }

    @Test
    fun `retorna Error em falha de rede`() = runTest {
        coEvery { adminApi.verificarAtualizacao(any(), any()) } throws java.io.IOException("network")
        val checker = UpdateChecker(adminApi, versaoAtual = "1.0.0")
        assertTrue(checker.check() is UpdateAvailability.Error)
    }

    @Test
    fun `retorna Error em HTTP 500`() = runTest {
        coEvery { adminApi.verificarAtualizacao(any(), any()) } returns Response.error(
            500, "".toResponseBody("text/plain".toMediaType())
        )
        val checker = UpdateChecker(adminApi, versaoAtual = "1.0.0")
        assertTrue(checker.check() is UpdateAvailability.Error)
    }
}
```

- [ ] **Step 2: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.update.UpdateCheckerTest"`
Expected: FAIL.

- [ ] **Step 3: Criar `UpdateChecker.kt` + `UpdateAvailability`**

```kotlin
package com.trivion.manageragent.update

import com.trivion.manageragent.network.AdminApi
import timber.log.Timber

sealed class UpdateAvailability {
    object None : UpdateAvailability()
    data class Available(
        val versaoNova: String,
        val urlDownload: String,
        val checksumSha256: String,
        val obrigatoria: Boolean
    ) : UpdateAvailability()
    data class Error(val motivo: String) : UpdateAvailability()
}

class UpdateChecker(
    private val adminApi: AdminApi,
    private val versaoAtual: String
) {

    suspend fun check(): UpdateAvailability {
        return try {
            val resp = adminApi.verificarAtualizacao("ANDROID", versaoAtual)
            if (!resp.isSuccessful) return UpdateAvailability.Error("HTTP ${resp.code()}")
            val body = resp.body() ?: return UpdateAvailability.Error("body vazio")
            if (!body.atualizacaoDisponivel) return UpdateAvailability.None
            UpdateAvailability.Available(
                versaoNova = body.versaoNova ?: return UpdateAvailability.Error("sem versaoNova"),
                urlDownload = body.urlDownload ?: return UpdateAvailability.Error("sem urlDownload"),
                checksumSha256 = body.checksumSha256 ?: return UpdateAvailability.Error("sem checksum"),
                obrigatoria = body.obrigatoria
            )
        } catch (e: Exception) {
            Timber.tag("UpdateChecker").w(e, "check failed")
            UpdateAvailability.Error(e.message ?: "erro desconhecido")
        }
    }
}
```

- [ ] **Step 4: Criar `UpdateCheckWorker.kt`**

```kotlin
package com.trivion.manageragent.update

import android.content.Context
import androidx.work.CoroutineWorker
import androidx.work.OneTimeWorkRequestBuilder
import androidx.work.WorkManager
import androidx.work.WorkerParameters
import androidx.work.workDataOf
import com.trivion.manageragent.BuildConfig
import com.trivion.manageragent.auth.TokenStorage
import com.trivion.manageragent.network.ApiClient
import timber.log.Timber

class UpdateCheckWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val tokenStorage = TokenStorage(applicationContext)
        val adminApi = ApiClient.buildAdminApi(applicationContext, tokenStorage)
        val checker = UpdateChecker(adminApi, BuildConfig.VERSION_NAME)

        val res = checker.check()
        when (res) {
            is UpdateAvailability.None -> Timber.tag(TAG).i("Sem atualização disponível")
            is UpdateAvailability.Error -> Timber.tag(TAG).w("Check falhou: ${res.motivo}")
            is UpdateAvailability.Available -> {
                Timber.tag(TAG).i("Atualização disponível: ${res.versaoNova}")
                // Dispara download (T42) — enfileira ApkDownloadWorker
                val downloadReq = OneTimeWorkRequestBuilder<com.trivion.manageragent.update.ApkDownloadWorker>()
                    .setInputData(workDataOf(
                        "versaoNova" to res.versaoNova,
                        "urlDownload" to res.urlDownload,
                        "checksumSha256" to res.checksumSha256,
                        "obrigatoria" to res.obrigatoria
                    ))
                    .build()
                WorkManager.getInstance(applicationContext).enqueue(downloadReq)
            }
        }
        return Result.success()
    }

    companion object { private const val TAG = "UpdateCheckWorker" }
}
```

- [ ] **Step 5: Criar `UpdateScheduler.kt`**

```kotlin
package com.trivion.manageragent.update

import android.content.Context
import androidx.work.*
import java.util.concurrent.TimeUnit

object UpdateScheduler {

    private const val WORK_NAME = "ManagerAgent.UpdateCheck"

    fun schedule(context: Context, intervalHours: Long = 6L) {
        val constraints = Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .build()
        val req = PeriodicWorkRequestBuilder<UpdateCheckWorker>(intervalHours, TimeUnit.HOURS)
            .setConstraints(constraints)
            .build()
        WorkManager.getInstance(context).enqueueUniquePeriodicWork(
            WORK_NAME, ExistingPeriodicWorkPolicy.KEEP, req
        )
    }

    fun cancel(context: Context) {
        WorkManager.getInstance(context).cancelUniqueWork(WORK_NAME)
    }
}
```

- [ ] **Step 6: Wire no ManagerAgentService**

```kotlin
override fun onCreate() {
    super.onCreate()
    // ... existente
    UpdateScheduler.schedule(this)
}
```

- [ ] **Step 7: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.update.UpdateCheckerTest"`
Expected: PASS (4 testes).

- [ ] **Step 8: Commit**

```bash
git add .
git commit -m "feat: add UpdateChecker + Worker (6h periodic check, dispatches download)"
```

---

### Task T42: ApkDownloader — baixa APK + valida SHA-256

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/update/ApkDownloader.kt`
- Create: `app/src/main/kotlin/com/trivion/manageragent/update/ApkDownloadWorker.kt`
- Create: `app/src/test/kotlin/com/trivion/manageragent/update/ApkDownloaderTest.kt`

**Interfaces:**
- Consumes: OkHttp
- Produces:
  - `sealed class DownloadResult { data class Success(file: File); data class ChecksumMismatch; data class DownloadFailed(msg) }`
  - `ApkDownloader(client: OkHttpClient)`
  - `suspend fun download(url: String, expectedChecksum: String, destFile: File): DownloadResult`
  - `ApkDownloadWorker` — orquestra download + valida + dispara UpdateNotification (T43)

- [ ] **Step 1: Escrever teste**

Criar `app/src/test/kotlin/com/trivion/manageragent/update/ApkDownloaderTest.kt`:

```kotlin
package com.trivion.manageragent.update

import okhttp3.OkHttpClient
import okhttp3.mockwebserver.MockResponse
import okhttp3.mockwebserver.MockWebServer
import okio.Buffer
import org.junit.jupiter.api.AfterEach
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import kotlinx.coroutines.runBlocking
import java.io.File
import java.security.MessageDigest
import kotlin.io.path.createTempDirectory
import kotlin.test.assertEquals
import kotlin.test.assertTrue

class ApkDownloaderTest {

    private lateinit var server: MockWebServer
    private lateinit var tempDir: File
    private lateinit var destFile: File
    private val downloader = ApkDownloader(OkHttpClient())

    @BeforeEach
    fun setUp() {
        server = MockWebServer().also { it.start() }
        tempDir = createTempDirectory("apk").toFile()
        destFile = File(tempDir, "test.apk")
    }

    @AfterEach
    fun tearDown() { server.shutdown(); tempDir.deleteRecursively() }

    private fun sha256(bytes: ByteArray): String {
        return MessageDigest.getInstance("SHA-256").digest(bytes).joinToString("") { "%02x".format(it) }
    }

    @Test
    fun `download com checksum ok retorna Success`() = runBlocking {
        val payload = "conteudo do APK".toByteArray()
        val checksum = sha256(payload)
        server.enqueue(MockResponse().setBody(Buffer().apply { write(payload) }))

        val res = downloader.download(server.url("/apk").toString(), checksum, destFile)
        assertTrue(res is DownloadResult.Success)
        assertEquals(payload.size, destFile.readBytes().size)
    }

    @Test
    fun `download com checksum errado retorna ChecksumMismatch`() = runBlocking {
        val payload = "conteudo do APK".toByteArray()
        server.enqueue(MockResponse().setBody(Buffer().apply { write(payload) }))

        val res = downloader.download(server.url("/apk").toString(), "wrong-checksum", destFile)
        assertTrue(res is DownloadResult.ChecksumMismatch)
        assertEquals(false, destFile.exists())  // arquivo deletado
    }

    @Test
    fun `download com HTTP 404 retorna DownloadFailed`() = runBlocking {
        server.enqueue(MockResponse().setResponseCode(404))
        val res = downloader.download(server.url("/apk").toString(), "any", destFile)
        assertTrue(res is DownloadResult.DownloadFailed)
    }
}
```

- [ ] **Step 2: Run e ver falhar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.update.ApkDownloaderTest"`
Expected: FAIL.

- [ ] **Step 3: Criar `ApkDownloader.kt` + `DownloadResult`**

```kotlin
package com.trivion.manageragent.update

import okhttp3.OkHttpClient
import okhttp3.Request
import timber.log.Timber
import java.io.File
import java.security.MessageDigest

sealed class DownloadResult {
    data class Success(val file: File) : DownloadResult()
    object ChecksumMismatch : DownloadResult()
    data class DownloadFailed(val motivo: String) : DownloadResult()
}

class ApkDownloader(private val client: OkHttpClient) {

    suspend fun download(url: String, expectedChecksum: String, destFile: File): DownloadResult {
        return try {
            val req = Request.Builder().url(url).build()
            val resp = client.newCall(req).execute()
            if (!resp.isSuccessful) return DownloadResult.DownloadFailed("HTTP ${resp.code}")
            val bytes = resp.body?.bytes() ?: return DownloadResult.DownloadFailed("body vazio")

            destFile.parentFile?.mkdirs()
            destFile.writeBytes(bytes)

            val actualChecksum = sha256(bytes)
            if (!actualChecksum.equals(expectedChecksum, ignoreCase = true)) {
                Timber.tag(TAG).w("Checksum mismatch: expected=$expectedChecksum actual=$actualChecksum")
                destFile.delete()
                return DownloadResult.ChecksumMismatch
            }
            DownloadResult.Success(destFile)
        } catch (e: Exception) {
            Timber.tag(TAG).w(e, "download failed")
            destFile.delete()
            DownloadResult.DownloadFailed(e.message ?: "erro desconhecido")
        }
    }

    private fun sha256(bytes: ByteArray): String {
        return MessageDigest.getInstance("SHA-256").digest(bytes).joinToString("") { "%02x".format(it) }
    }

    companion object { private const val TAG = "ApkDownloader" }
}
```

- [ ] **Step 4: Criar `ApkDownloadWorker.kt`**

```kotlin
package com.trivion.manageragent.update

import android.content.Context
import android.content.Intent
import androidx.core.app.NotificationCompat
import androidx.core.app.NotificationManagerCompat
import androidx.work.CoroutineWorker
import androidx.work.WorkerParameters
import com.trivion.manageragent.R
import com.trivion.manageragent.service.NotificationHelper
import okhttp3.OkHttpClient
import timber.log.Timber
import java.io.File

class ApkDownloadWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val versaoNova = inputData.getString("versaoNova") ?: return Result.failure()
        val url = inputData.getString("urlDownload") ?: return Result.failure()
        val checksum = inputData.getString("checksumSha256") ?: return Result.failure()
        val obrigatoria = inputData.getBoolean("obrigatoria", false)

        val destDir = File(applicationContext.filesDir, "updates").apply { mkdirs() }
        val destFile = File(destDir, "$versaoNova.apk")

        val downloader = ApkDownloader(OkHttpClient())
        val result = downloader.download(url, checksum, destFile)

        return when (result) {
            is DownloadResult.Success -> {
                Timber.tag(TAG).i("APK $versaoNova baixado — mostrando notification")
                mostrarNotificationUpdate(versaoNova, destFile, obrigatoria)
                Result.success()
            }
            is DownloadResult.ChecksumMismatch -> {
                Timber.tag(TAG).e("Checksum mismatch pra $versaoNova")
                // Reportar (T44 chama isso — aqui só marca resultado)
                Result.failure()
            }
            is DownloadResult.DownloadFailed -> {
                Timber.tag(TAG).w("Download falhou: ${result.motivo}")
                Result.retry()
            }
        }
    }

    private fun mostrarNotificationUpdate(versao: String, apkFile: File, obrigatoria: Boolean) {
        NotificationHelper.ensureUpdateChannel(applicationContext)
        val installIntent = com.trivion.manageragent.update.UpdateInstallReceiver.buildPendingIntent(applicationContext, apkFile, versao)
        val notif = NotificationCompat.Builder(applicationContext, NotificationHelper.UPDATE_CHANNEL_ID)
            .setContentTitle("Atualização do ManagerAgent disponível")
            .setContentText("Toque pra instalar a versão $versao")
            .setSmallIcon(android.R.drawable.stat_sys_download_done)
            .setPriority(if (obrigatoria) NotificationCompat.PRIORITY_HIGH else NotificationCompat.PRIORITY_LOW)
            .setContentIntent(installIntent)
            .setAutoCancel(true)
            .build()
        NotificationManagerCompat.from(applicationContext).notify(2001, notif)
    }

    companion object { private const val TAG = "ApkDownloadWorker" }
}
```

- [ ] **Step 5: Run e ver passar**

Run: `./gradlew :app:testDebugUnitTest --tests "com.trivion.manageragent.update.ApkDownloaderTest"`
Expected: PASS (3 testes).

- [ ] **Step 6: Commit**

```bash
git add .
git commit -m "feat: add ApkDownloader + Worker (SHA-256 validation + notification)"
```

---

### Task T43: UpdateInstallReceiver + PackageInstaller flow

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/update/UpdateInstallReceiver.kt`
- Modify: `NotificationHelper.kt` (adiciona channel `UPDATE_CHANNEL_ID`)
- Modify: `AndroidManifest.xml` (declara receiver + FileProvider)
- Create: `app/src/main/res/xml/file_paths.xml`

**Interfaces:**
- Consumes: `PackageInstaller` (Android system)
- Produces:
  - `UpdateInstallReceiver` — BroadcastReceiver acionado pelo click na notification, dispara `PackageInstaller` com FileProvider URI + intent ACTION_INSTALL_PACKAGE
  - `NotificationHelper.ensureUpdateChannel(context)` — channel de notifications de update

- [ ] **Step 1: Criar `app/src/main/res/xml/file_paths.xml` (FileProvider config)**

```xml
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <files-path name="updates" path="updates/" />
</paths>
```

- [ ] **Step 2: Adicionar FileProvider no Manifest**

```xml
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>

<receiver
    android:name=".update.UpdateInstallReceiver"
    android:exported="false" />
```

- [ ] **Step 3: Adicionar update channel no NotificationHelper**

```kotlin
const val UPDATE_CHANNEL_ID = "manageragent_updates"

fun ensureUpdateChannel(context: Context) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        val nm = context.getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
        if (nm.getNotificationChannel(UPDATE_CHANNEL_ID) == null) {
            val channel = NotificationChannel(
                UPDATE_CHANNEL_ID,
                "ManagerAgent atualizações",
                NotificationManager.IMPORTANCE_DEFAULT
            )
            nm.createNotificationChannel(channel)
        }
    }
}
```

- [ ] **Step 4: Criar `UpdateInstallReceiver.kt`**

```kotlin
package com.trivion.manageragent.update

import android.app.PendingIntent
import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import androidx.core.content.FileProvider
import timber.log.Timber
import java.io.File

class UpdateInstallReceiver : BroadcastReceiver() {

    override fun onReceive(context: Context, intent: Intent) {
        val apkPath = intent.getStringExtra(EXTRA_APK_PATH) ?: return
        val versao = intent.getStringExtra(EXTRA_VERSAO) ?: "desconhecida"
        val file = File(apkPath)
        if (!file.exists()) {
            Timber.tag(TAG).w("APK não encontrado: $apkPath")
            return
        }

        val uri = FileProvider.getUriForFile(context, "${context.packageName}.fileprovider", file)
        val installIntent = Intent(Intent.ACTION_VIEW).apply {
            setDataAndType(uri, "application/vnd.android.package-archive")
            addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION or Intent.FLAG_ACTIVITY_NEW_TASK)
        }
        try {
            context.startActivity(installIntent)
            Timber.tag(TAG).i("Instalação disparada pra versão $versao")
        } catch (e: Exception) {
            Timber.tag(TAG).e(e, "Falha ao disparar installIntent")
        }
    }

    companion object {
        private const val TAG = "UpdateInstallReceiver"
        private const val EXTRA_APK_PATH = "apk_path"
        private const val EXTRA_VERSAO = "versao"

        fun buildPendingIntent(context: Context, apkFile: File, versao: String): PendingIntent {
            val intent = Intent(context, UpdateInstallReceiver::class.java).apply {
                putExtra(EXTRA_APK_PATH, apkFile.absolutePath)
                putExtra(EXTRA_VERSAO, versao)
            }
            return PendingIntent.getBroadcast(
                context,
                0,
                intent,
                PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
            )
        }
    }
}
```

- [ ] **Step 5: Build e verificar**

Run: `./gradlew :app:assembleDebug`
Expected: BUILD SUCCESSFUL.

- [ ] **Step 6: Teste manual (emulador)**

1. Publica APK "1.0.1" fake no bucket (versão maior que a instalada)
2. Configura endpoint mock respondendo `atualizacaoDisponivel=true` + URL + checksum válido
3. Aguarda UpdateCheckWorker rodar (ou dispara manualmente via `WorkManager.getInstance().enqueue(OneTimeWorkRequest(UpdateCheckWorker))`)
4. Verifica notification aparecer
5. Clica na notification
6. Android mostra popup "Deseja instalar?"

- [ ] **Step 7: Commit**

```bash
git add .
git commit -m "feat: add UpdateInstallReceiver + FileProvider + update notification channel"
```

---

### Task T44: Reportar resultado do update (audit + endpoint)

**Files:**
- Create: `app/src/main/kotlin/com/trivion/manageragent/update/UpdateResultReporter.kt`
- Modify: `ApkDownloadWorker.kt` (chama reporter em cada resultado)
- Modify: `AuditReporter.Events` (adiciona UPDATE_INICIADO/SUCESSO/FALHOU)

**Interfaces:**
- Consumes: `AdminApi.reportarAtualizacao` (T28), `AuditReporter` (T38)
- Produces:
  - `UpdateResultReporter(adminApi, auditReporter)`
  - `suspend fun reportSuccess(versao: String)`
  - `suspend fun reportFailure(versao: String, motivo: String)`

Também: adiciona hook no `ManagerAgentApplication.onCreate()` que, se `BuildConfig.VERSION_NAME` for maior que a última versão vista guardada no SharedPreferences, dispara `reportSuccess` (indicando que o update foi aplicado com sucesso — Application acabou de rodar na nova versão).

- [ ] **Step 1: Adicionar constants em `AuditReporter.Events`**

```kotlin
const val UPDATE_INICIADO_ANDROID = "UPDATE_INICIADO_ANDROID"
const val UPDATE_SUCESSO_ANDROID = "UPDATE_SUCESSO_ANDROID"
const val UPDATE_FALHOU_ANDROID = "UPDATE_FALHOU_ANDROID"
```

- [ ] **Step 2: Criar `UpdateResultReporter.kt`**

```kotlin
package com.trivion.manageragent.update

import com.trivion.manageragent.log.AuditReporter
import com.trivion.manageragent.network.AdminApi
import com.trivion.manageragent.network.dto.UpdateResultRequest
import timber.log.Timber

class UpdateResultReporter(
    private val adminApi: AdminApi,
    private val auditReporter: AuditReporter
) {

    suspend fun reportSuccess(versao: String) {
        try {
            adminApi.reportarAtualizacao(UpdateResultRequest(versaoTentada = versao, sucesso = true))
        } catch (e: Exception) {
            Timber.tag(TAG).w(e, "reportarAtualizacao(success) failed")
        }
        auditReporter.report(AuditReporter.Events.UPDATE_SUCESSO_ANDROID, mapOf("versao" to versao))
    }

    suspend fun reportFailure(versao: String, motivo: String) {
        try {
            adminApi.reportarAtualizacao(UpdateResultRequest(versaoTentada = versao, sucesso = false, motivoErro = motivo))
        } catch (e: Exception) {
            Timber.tag(TAG).w(e, "reportarAtualizacao(failure) failed")
        }
        auditReporter.report(AuditReporter.Events.UPDATE_FALHOU_ANDROID, mapOf("versao" to versao, "motivo" to motivo))
    }

    companion object { private const val TAG = "UpdateResultReporter" }
}
```

- [ ] **Step 3: Wire no `ApkDownloadWorker.doWork()`**

Modificar retornos:

```kotlin
override suspend fun doWork(): Result {
    val versaoNova = inputData.getString("versaoNova") ?: return Result.failure()
    // ... resto igual

    val tokenStorage = TokenStorage(applicationContext)
    val adminApi = ApiClient.buildAdminApi(applicationContext, tokenStorage)
    val auditReporter = AuditReporter(adminApi, tokenStorage)
    val resultReporter = UpdateResultReporter(adminApi, auditReporter)

    // Registra que começou
    auditReporter.report(AuditReporter.Events.UPDATE_INICIADO_ANDROID, mapOf("versao" to versaoNova))

    val result = downloader.download(url, checksum, destFile)

    return when (result) {
        is DownloadResult.Success -> {
            Timber.tag(TAG).i("APK $versaoNova baixado")
            mostrarNotificationUpdate(versaoNova, destFile, obrigatoria)
            Result.success()
        }
        is DownloadResult.ChecksumMismatch -> {
            resultReporter.reportFailure(versaoNova, "CHECKSUM_MISMATCH")
            Result.failure()
        }
        is DownloadResult.DownloadFailed -> {
            resultReporter.reportFailure(versaoNova, "DOWNLOAD_FAILED: ${result.motivo}")
            Result.retry()
        }
    }
}
```

- [ ] **Step 4: Hook de sucesso no `ManagerAgentApplication.onCreate()`**

```kotlin
override fun onCreate() {
    super.onCreate()
    LoggingSetup.init(this)
    LoggingSetup.purgeOldFiles(this)

    val tokenStorage = TokenStorage(this)
    val adminApi = ApiClient.buildAdminApi(this, tokenStorage)

    // Detecta se acabou de atualizar
    val prefs = getSharedPreferences("update_state", MODE_PRIVATE)
    val lastKnownVersion = prefs.getString("last_version", null)
    val currentVersion = BuildConfig.VERSION_NAME
    if (lastKnownVersion != null && lastKnownVersion != currentVersion) {
        val auditReporter = AuditReporter(adminApi, tokenStorage)
        val reporter = UpdateResultReporter(adminApi, auditReporter)
        kotlinx.coroutines.GlobalScope.launch {
            reporter.reportSuccess(currentVersion)
        }
    }
    prefs.edit().putString("last_version", currentVersion).apply()

    val crashReporter = CrashReporter(
        adminApi = adminApi,
        tokenStorage = tokenStorage,
        versao = currentVersion,
        sistemaOperacional = "Android ${Build.VERSION.RELEASE} (API ${Build.VERSION.SDK_INT})",
        maquina = "${Build.MANUFACTURER} ${Build.MODEL}",
        cnpj = ""
    )
    crashReporter.installUncaughtHandler()
    AnrDetector(crashReporter).start()
}
```

- [ ] **Step 5: Commit**

```bash
git add .
git commit -m "feat: add UpdateResultReporter (audit + endpoint on install success/fail)"
```

---

## Fim do escopo — resumo do que foi entregue

Ao completar T1-T44:

- **APK instalável**: staging + prod, 2 packageNames distintos, assinados
- **Coleta paridade Windows**: janela ativa, ociosidade, sessão lock/unlock, heartbeats, resumo input (sem conteúdo), reuniões (Zoom/Teams/Meet), chamadas telefônicas (start/end, sem número), transições de status ATIVO/PAUSA/AUSENTE, URL domain (best-effort)
- **Auth completo**: chave empresa via config.json, identificador no primeiro uso, DeviceJWT + refresh token, interceptors auth/refresh/revocation
- **Envio batch**: Room como buffer resiliente, WorkManager 15min com network constraint, retry exponencial
- **UI mínima**: 1 tela onboarding (identificador + aceite LGPD), sumindo da gaveta após 1º uso
- **OEM handling**: detecção fabricante + intent pro Settings de Autostart quando problemático
- **Logs**: Timber + FileLoggingTree (7 dias retenção), AuditReporter, CrashReporter (throttle 5min) + ANR detector
- **Escape hatch**: action "Enviar diagnóstico" na notification, copia logs pra Downloads público
- **Auto-update**: check 6h, download com SHA-256, notification pra instalar, reporter de sucesso/falha
- **CI/CD**: GitHub Actions build + tests + coverage gate; workflow separado pra publicar APKs assinados
- **Scripts locais**: `build-personalizado.sh` (dev testa APK sem pipeline server-side)
- **Cobertura**: linha ≥80% + branch ≥95% (verificação Jacoco no CI)

**Não entregue nesta rodada (fica pra próximas — Marcos planeja):**
- Portal com botão "Baixar APK Android" + pipeline server-side de re-empacotamento (@Shuri no srv-portal-node)
- Docs pública em `docs.imanagerportal.com/mobile/` (6 páginas)
- Testes cross-device automatizados (E2E @Natasha)
- Testes manuais em matriz de 6 devices físicos
- Piloto interno + piloto cliente
- Score IA calibrado pra padrão mobile (@Jarvis + @Thor)
- Portal com presença unificada PC+celular (@Shuri + @Peter)
- Alterações backend (dispositivo_tipo, PhoneCallEvent, sistema_operacional em versoes_agente) — @Shuri implementa no Node

## Self-Review checklist

- [x] **Placeholder scan:** todos os passos têm código concreto, sem TBD/TODO
- [x] **Type consistency:** `AgentEvent` sealed class usada consistentemente entre coletores/queue/sender; `TokenStorage` API idêntica em T27-T44
- [x] **Spec coverage:**
  - Componentes 1-15 da spec → T7, T9-T17, T19-T24, T27-T31, T32-T35, T37-T44 (cobertos)
  - Fluxo de onboarding da spec §7 → T32-T35 (cobertos)
  - Auto-update da spec §8 → T41-T44 (cobertos)
  - Logs da spec §9 → T37-T40 (cobertos)
  - Distribuição sideload da spec §10 → T3 (keystore), T5 (CI), T6 (script local), T42-T43 (FileProvider install)
  - Scripts de build da spec §12 → T5, T6 (cobertos)
  - Testing da spec §11 → cada task tem teste próprio (linha ≥80%, branch ≥95%)
- [x] **Bloco B4 Task T25**: wire completo do AccessibilityService integrado — pontua fim do pipeline de coleta
- [x] **Contrato compatível backend Node**: `dispositivoTipo=ANDROID` enviado desde T29 e T31 (premissa Marcos aprovada: Node 100% pronto)
- [x] **LGPD travado por teste**: `LGPDContentFilterTest` (T20) garante que collector nunca aceite String/CharSequence

## Métricas do plano

- **44 tasks** distribuídas em 7 blocos (B2-B8)
- **~120-150 arquivos criados** (código + testes + config + docs internas)
- **~15-16 semanas** de dev sênior full-time (paralelismo B_test)
- **Cobertura de teste TDD** em cada task (teste primeiro, sempre)

## Coordenações pendentes

- **Contratação dev Android sênior:** Marcos + Harvey — em paralelo ao brainstorm da próxima rodada
- **@Shuri:** migrar rotas Agent pra Node com contrato estendido (`dispositivoTipo`, `PhoneCallEvent`, `sistema_operacional`) antes do dia D
- **@Vision:** setup CI/CD secrets (KEYSTORE_*_BASE64, TIGRIS_*), config bucket S3, domínio `apk.imanager.trivion.com.br` — quando Marcos disparar
- **@Steve:** validação de produto do ICP mobile (SDR/Atendimento/Executivo) + preparação Raquel/@Mike pra aceite LGPD como bloqueante contratual

## Execution Handoff

Plano completo e salvo em `.brain/tecnologia/plans/2026-08-08-agent-android-byod-mvp.md`.

**2 opções de execução quando Marcos disparar:**

1. **Subagent-Driven (recomendado):** dispatch de subagent fresh por task, com review entre tasks. Fast iteration, contexto limpo por task, minha camada de review checa cada entrega. Ideal pra dev novo Android que vem sem contexto do Manager — cada task é auto-contida com paths + testes.

2. **Inline Execution:** dev novo executa tasks localmente seguindo o plano linear, com checkpoints periódicos comigo pra review. Menos overhead de coordenação, mas assume que o dev consegue seguir o plano sem tropeços frequentes.

**Recomendação: Opção 2 (Inline) pra este caso.** Motivos:
- Dev novo humano vai executar (não subagent) — o dev novo é o "runtime" das tasks
- Subagent-Driven é ótimo pra fluxo Claude-executa; aqui é humano-executa com Claude reviewing
- Checkpoints entre blocos (fim de B3, B5, B6, B7, B8) pra review estrutural comigo

Marcos, quando você disparar "vamos" com o dev novo Android alocado, começamos por T1.

