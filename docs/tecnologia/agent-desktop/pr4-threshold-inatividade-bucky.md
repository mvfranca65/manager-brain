> **STATUS:** PRONTO PARA INICIAR
> **DATA:** 2026-06-17
> **DONO:** @Bucky (Backend Dev C# / Agent Desktop)
> **COORDENA:** @Tony
> **DEPENDE DE:** PR 1 (srv-portal) e PR 2 (srv-events + V103) — ambos commitados localmente, aguardando push do Marcos

# PR 4 — Agent C# threshold inatividade (Bucky)

Este e o **PR 4** da spec `../../tecnologia/specs/2026-06-17-ajuste-thresholds-inatividade-design.md`. PR 1 (@Thor — srv-portal), PR 2 (@Thor — srv-events + V103 + 2 fixes srv-ia) e analise do @Jarvis ja estao concluidos. Falta o Agent C#.

---

## 1. O que mudar — visao geral

| Estado | Hoje | Depois |
|---|---|---|
| ATIVO (era "Online") | < 5 min sem atividade | < 5 min sem atividade |
| PAUSA (era "Ausente" no Agent) | 5 a 30 min sem atividade | **5 a 15 min sem atividade** |
| AUSENTE (era "Inativo" no Agent) | > 30 min sem atividade | **> 15 min sem atividade** |

Tres frentes:
1. **Threshold**: AUSENTE de 30min → 15min
2. **Renomeacao** de propriedades, enums e strings
3. **Strings serializadas** enviadas ao backend mudam de "Online"/"Ausente"/"Inativo" para "ATIVO"/"PAUSA"/"AUSENTE"

---

## 2. Repo e contexto critico — CODIGO DUPLICADO

**Caminho:** `/Users/Olimpo/Documents/Athena/Projetos/Agente/manager-srv-agent/`

**Existem ate 3 copias** do mesmo codigo:
- `src/ManagerAgent.Service/` (servico Windows legado)
- `src/ManagerAgent.SessionWorker/` (worker novo)
- `src/ManagerAgent.Tray/` (tray icon — confirmar se tem `appsettings.json` afetado)

**Comeca mapeando:**
```bash
cd /Users/Olimpo/Documents/Athena/Projetos/Agente/manager-srv-agent
find src -name "ManagerAgentUploadOptions.cs" -o -name "UserStatusManager.cs" -o -name "UserStatus.cs" -o -name "appsettings.json"
```

**Voce precisa tocar em TODAS as copias** — debito tecnico conhecido. Quando refatorarmos para projeto compartilhado (item MVP3), elimina-se a duplicacao.

---

## 3. Mapa de mudancas — arquivo por arquivo

### 3.1 `ManagerAgentUploadOptions.cs` (em todas as copias)

Hoje (linhas 75-84):
```csharp
/// <summary>
/// Tempo (minutos) sem atividade para transição do status Online → Ausente.
/// Padrão: 5.
/// </summary>
public int LimiteAusenteMinutos { get; set; } = 5;

/// <summary>
/// Tempo (minutos) sem atividade para transição do status Ausente → Inativo.
/// Padrão: 30.
/// </summary>
public int LimiteInativoMinutos { get; set; } = 30;
```

Depois:
```csharp
/// <summary>
/// Tempo (minutos) sem atividade para transição do status Ativo → Pausa.
/// Padrão: 5.
/// </summary>
public int LimitePausaMinutos { get; set; } = 5;

/// <summary>
/// Tempo (minutos) sem atividade para transição do status Pausa → Ausente.
/// Padrão: 15.
/// </summary>
public int LimiteAusenteMinutos { get; set; } = 15;
```

**Atencao:** `LimiteAusenteMinutos` MUDOU DE SEMANTICA — antes era a primeira transicao (5min); agora e a segunda (15min, antes era 30min `LimiteInativoMinutos`). Cuidado pra nao confundir.

### 3.2 `appsettings.json` (em todas as copias com bloco Capture)

Antes:
```json
"LimiteAusenteMinutos": 5,
"LimiteInativoMinutos": 30
```

Depois:
```json
"LimitePausaMinutos": 5,
"LimiteAusenteMinutos": 15
```

Localizacao tipica: `src/ManagerAgent.Service/appsettings.json`, `src/ManagerAgent.SessionWorker/appsettings.json`. Se houver `appsettings.Development.json` ou `appsettings.Production.json`, atualizar tambem.

### 3.3 `UserStatusManager.cs` (em todas as copias)

**Comentario de classe:**
```csharp
// ANTES
/// Transições: Online → Ausente (10min) → Inativo (30min total)

// DEPOIS
/// Transições: Ativo → Pausa (5min) → Ausente (15min total)
```

**Campos privados:**
```csharp
// ANTES
private readonly TimeSpan _ausenteThreshold;  // antigo 10/5min
private readonly TimeSpan _inativoThreshold;  // antigo 30min

// DEPOIS
private readonly TimeSpan _pausaThreshold;    // 5min
private readonly TimeSpan _ausenteThreshold;  // 15min — REUTILIZA O NOME, semantica nova
```

**Construtor:**
```csharp
// ANTES
_ausenteThreshold = TimeSpan.FromMinutes(opts.LimiteAusenteMinutos);  // 5
_inativoThreshold = TimeSpan.FromMinutes(opts.LimiteInativoMinutos);  // 30
_currentStatus = UserStatus.Online;

// DEPOIS
_pausaThreshold   = TimeSpan.FromMinutes(opts.LimitePausaMinutos);    // 5
_ausenteThreshold = TimeSpan.FromMinutes(opts.LimiteAusenteMinutos);  // 15
_currentStatus = UserStatus.Ativo;
```

**Metodos `RegisterActivity` e `CheckIdleTransition`:**
- Todas as referencias a `UserStatus.Online` → `UserStatus.Ativo`
- Todas as referencias a `UserStatus.Ausente` (status antigo, 5min) → `UserStatus.Pausa`
- Todas as referencias a `UserStatus.Inativo` (status antigo, 30min) → `UserStatus.Ausente`
- Mensagens de log: `"Idle timeout 30min"` → `"Idle timeout 15min"`; `"Idle timeout 10min"` → `"Idle timeout 5min"`

### 3.4 `UserStatus.cs` (enum — em todas as copias)

```csharp
// ANTES
public enum UserStatus { Online, Ausente, Inativo }

// DEPOIS
public enum UserStatus { Ativo, Pausa, Ausente }
```

### 3.5 `GetStatusString(UserStatus)` ou similar

Localizar onde o status e serializado para enviar ao backend. Atualizar mapeamento:
- `UserStatus.Ativo` → `"ATIVO"`
- `UserStatus.Pausa` → `"PAUSA"`
- `UserStatus.Ausente` → `"AUSENTE"`

(Uppercase. O backend `srv-events` ja aceita esses 3 valores diretamente — e tem aliases legados ONLINE→ATIVO, INATIVO→AUSENTE caso o Agent antigo envie algo no meio da transicao.)

---

## 4. Versionamento (csproj)

Bumpar versao no `*.csproj` (ex: 1.4.0 → 1.5.0). Encontrar com:
```bash
find . -name "*.csproj" -exec grep -l "Version" {} \;
```

Provavelmente em `ManagerAgent.Service.csproj`, `ManagerAgent.SessionWorker.csproj`, `ManagerAgent.Tray.csproj`.

**NAO publique o instalador no Tigris S3 nem registre em `versoes_agente` no banco** — isso depende de credenciais que o Marcos+@Vision controlam. Apenas bumpe o numero no codigo e avise no relatorio.

---

## 5. Testes

`dotnet test` precisa passar 100%. Tipicamente em `tests/ManagerAgent.Service.Tests/UserStatusManagerTests.cs` (ou equivalente).

Atualizar:
- Mocks de `IOptions<ManagerAgentUploadOptions>` para usar os novos nomes de propriedades.
- Assertions de transicoes para verificar `UserStatus.Pausa` (em 5min) e `UserStatus.Ausente` (em 15min).
- Strings esperadas em eventos de transicao serializados.

---

## 6. Regras absolutas

1. **NAO faca commit, NAO crie branch, NAO push.** Marcos comita e pusha apos validar.
2. **Limpeza:** `bin/`, `obj/`, `.vs/` NAO entram em `git status --short`. Se aparecerem, confirmar `.gitignore` e fazer `git rm --cached -r bin/ obj/ .vs/` se necessario.
3. NAO toque em outros services: srv-portal, srv-events, srv-admin, srv-ia, fed-portal estao fora do seu escopo.
4. Idioma do codigo = ingles, comentarios e Javadoc = portugues (regra `_shared/padroes.md`).
5. **Privacidade inegociavel** (`_shared/lgpd-operacional.md`): nada de keylog, screenshot, audio, camera, conteudo de arquivos, URL completa.

---

## 7. Relatorio que voce deve devolver

Quando terminar, devolva relatorio com:

1. **Mapa de copias** encontradas (caminhos dos arquivos duplicados que tocou)
2. **Lista de arquivos modificados** + 1 linha por arquivo
3. **Diff resumido do `appsettings.json`** em cada copia
4. **Output de `dotnet test`** (passou? falhou? quantos?)
5. **Versao antiga e nova** dos `.csproj` que voce bumpou
6. **`git status --short`** confirmando que `bin/obj/.vs` nao aparecem
7. **Decisoes arquiteturais proprias** que voce tomou (inconsistencia entre copias, codigo duplicado dificil de manter, etc.)
8. **Perguntas pendentes** se houver

NAO comite. So devolve o relatorio.

---

## 8. Coordenacao pos-PR4

Apos seu PR 4 estar pronto:
- @Tony (eu) revisa o codigo
- @Natasha valida QA (instalador + comportamento de status em maquina real)
- Voce ajusta testes ate 100% verde
- Marcos coordena com @Vision: publicacao no Tigris S3 + registro em `versoes_agente.obrigatoria=true` (sem clientes em prod, podemos marcar obrigatorio).

---

## 9. Estimativa

~3h pelos numeros da spec (consolidado de T4.1 a T4.5; T4.6 a T4.9 ficam com Marcos+@Vision).

---

## 10. Referencias

- Spec mae: `../../tecnologia/specs/2026-06-17-ajuste-thresholds-inatividade-design.md`
- PR 1 (Thor): srv-portal — commitado em `feature/threshold-inatividade-pr1` local
- PR 2 (Thor): srv-events + V103 + 2 fixes srv-ia — codigo no working tree, aguardando commit do Marcos
- Analise @Jarvis: secao 8 da spec mae confirma zero impacto no srv-ia
- Definicao do agent: `/Users/Olimpo/Documents/Athena/Projetos/.claude/agents/bucky.md`
