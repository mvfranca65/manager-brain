> **STATUS:** ATIVA — aguardando execução por @Shuri
> **DATA:** 2026-08-08
> **DONO ARQUITETURAL:** @Tony
> **EXECUTOR:** @Shuri (dona técnica do srv-admin-node)
> **DEPENDÊNCIA:** habilitador da spec `.brain/tecnologia/specs/2026-08-08-agent-android-byod-mvp-design.md`
> **PREMISSA:** deve estar 100% em produção **antes** do desenvolvimento do Agent Android começar (regra Marcos 2026-08-08)

# srv-admin-node — Suporte Agent Android BYOD

## 1. Contexto

O Agent Desktop Manager está sendo estendido pra Android BYOD (celular pessoal do colaborador). Todo o código do Android está desenhado pra usar **os endpoints do srv-admin que já existem no Java** — o srv-admin-node (em migração pela @Shuri) precisa nascer com esses endpoints já estendidos pra aceitar:

- Novos campos que identificam origem Android (`dispositivoTipo=ANDROID`, SO rico com fabricante+modelo)
- Novos tipos de audit event (`AGENT_INSTALADO_ANDROID`, `ACCESSIBILITY_ATIVADA`, `OEM_PROBLEMATICO_DETECTADO`, etc.)
- Novo tipo de crash (`ANR` — Android specific)
- Filtro de versão por sistema operacional (Android tem seu próprio APK, Windows seu próprio EXE)
- Header `X-Agent-Revoked: true` em respostas 403 quando gestor revogou o Agent Mobile

Não há **novos endpoints** — só **estender contratos existentes**. Alterações de código isoladas e cirúrgicas.

## 2. Escopo

### 2.1 Dentro do escopo

- Extensão de 7 endpoints REST (todos já existem no Java atual)
- 2 migrations Drizzle (colunas em `agentes` e `versoes_agente`)
- 1 novo enum TS: `TipoDispositivo`
- 11 novos audit events + 1 novo tipo de crash suportados
- Header `X-Agent-Revoked` na resposta 403
- Testes de contrato (parity com Java quando aplicável) + testes unitários novos

### 2.2 Fora do escopo (rodadas futuras)

- Portal com botão "Baixar APK Android" (rodada futura Marcos)
- Pipeline server-side de re-empacotamento de APK personalizado (rodada futura)
- Endpoint `POST /api/agente/logs/upload` (rodada futura)
- `GET /api/agente/config` estendido com `enviarLogsAte`/`logLevel` dinâmico (rodada futura)
- Alterações no fluxo de emissão de Device JWT (mantém idêntico ao Java atual)

## 3. Alterações de schema (Drizzle migrations)

### 3.1 Migration 1 — `agentes.dispositivo_tipo`

```sql
ALTER TABLE agentes
  ADD COLUMN dispositivo_tipo VARCHAR(20) NOT NULL DEFAULT 'WINDOWS';

ALTER TABLE agentes
  ADD CONSTRAINT chk_agentes_dispositivo_tipo
  CHECK (dispositivo_tipo IN ('WINDOWS', 'MACOS', 'ANDROID', 'IOS'));

CREATE INDEX idx_agentes_dispositivo_tipo
  ON agentes(empresa_id, dispositivo_tipo)
  WHERE desvinculado_em IS NULL;
```

Rationale: `DEFAULT 'WINDOWS'` preserva compat com Agents Windows existentes (que continuarão a existir e não conhecem esse campo). Novos vínculos vindos do Android setam `ANDROID`. Index parcial cobre queries "quantos agentes Android ativos por empresa" comuns na Home API.

### 3.2 Migration 2 — `versoes_agente.sistema_operacional`

```sql
ALTER TABLE versoes_agente
  ADD COLUMN sistema_operacional VARCHAR(20) NOT NULL DEFAULT 'WINDOWS';

ALTER TABLE versoes_agente
  ADD CONSTRAINT chk_versoes_agente_sistema_operacional
  CHECK (sistema_operacional IN ('WINDOWS', 'MACOS', 'ANDROID', 'IOS'));

-- Drop UNIQUE antigo (se existir) e recria com sistema_operacional
-- Confirmar constraint atual antes de dropar
ALTER TABLE versoes_agente
  DROP CONSTRAINT IF EXISTS versoes_agente_versao_key;

ALTER TABLE versoes_agente
  ADD CONSTRAINT uq_versoes_agente_versao_so
  UNIQUE (versao, sistema_operacional);
```

Rationale: Windows e Android compartilham `versoes_agente` mas cada um tem sua sequência independente de versões (1.4.5 Windows ≠ 1.4.5 Android). UNIQUE composto `(versao, sistema_operacional)` permite ambos.

### 3.3 Enum TS canônico

Criar `src/enums/tipo-dispositivo.enum.ts`:

```typescript
export const TipoDispositivo = {
  WINDOWS: 'WINDOWS',
  MACOS: 'MACOS',
  ANDROID: 'ANDROID',
  IOS: 'IOS',
} as const;

export type TipoDispositivo = typeof TipoDispositivo[keyof typeof TipoDispositivo];

export const TiposDispositivoAceitos = new Set<string>(Object.values(TipoDispositivo));
```

Nota alinhada com regra `feedback_enum_novo_backend_antes.md`: o valor `ANDROID` PRECISA estar no enum backend antes do Agent Android ser lançado, senão `TypingPattern.valueOf` (Java equivalente) cai em catch silencioso e corrompe dado.

## 4. Alterações por endpoint

### 4.1 `POST /api/agente/dispositivos/vincular`

**Estado atual (Java):** aceita `identificador`, `instalacaoId`, `maquinaId`, `nomeMaquina`, `versaoAgente`, `descricaoSo` via body + `X-Ativacao-Key` no header. Retorna `agenteId`, `usuarioId`, `deviceToken`, `refreshToken`.

**Alteração no Node:**

- **Body ganha campo obrigatório `dispositivoTipo`** — validar contra `TiposDispositivoAceitos`. Se ausente/inválido: HTTP 400 `{ codigo: "DISPOSITIVO_TIPO_INVALIDO", mensagem: "Campo dispositivoTipo obrigatório: WINDOWS, MACOS, ANDROID, IOS" }`
- **Persistir `dispositivo_tipo` em `agentes`** durante o INSERT/UPSERT
- **Nenhuma outra mudança de contrato** — mesma resposta, mesmo status codes, mesmo JWT emitido

**DTO Zod novo:**

```typescript
export const VincularAgenteBodyDto = z.object({
  identificador: z.string().min(1).max(64),
  instalacaoId: z.string().uuid(),
  maquinaId: z.string().min(1).max(255),
  nomeMaquina: z.string().min(1).max(255),
  versaoAgente: z.string().min(1).max(50),
  descricaoSo: z.string().min(1).max(255),
  dispositivoTipo: z.enum(['WINDOWS', 'MACOS', 'ANDROID', 'IOS']),
});
```

**Testes de aceitação:**

1. Vincular com `dispositivoTipo=ANDROID` cria row em `agentes` com `dispositivo_tipo=ANDROID`
2. Vincular sem `dispositivoTipo` → HTTP 400
3. Vincular com `dispositivoTipo="foo"` → HTTP 400 + código `DISPOSITIVO_TIPO_INVALIDO`
4. Vincular com `dispositivoTipo=WINDOWS` continua funcionando (Agent Windows existente)
5. Re-vincular mesmo `maquinaId + empresa_id` (após desvincular) atualiza `dispositivo_tipo` (não cria duplicado — respeita índice parcial `idx_agentes_empresa_maquina_ativo`)

**Contract test cross-service:** payload do Agent Android (fixture em `test/fixtures/vincular-android.json`) deve ser aceito 100%.

---

### 4.2 `GET /api/agente/atualizacoes/verificar`

**Estado atual (Java):** query opcional `versaoAtual`, retorna se há atualização disponível. Não filtra por SO — retorna a última versão ativa (assumindo Windows implicitamente).

**Alteração no Node:**

- **Query obrigatória `sistemaOperacional`** — validar contra `TiposDispositivoAceitos`. Se ausente: HTTP 400 `{ codigo: "SISTEMA_OPERACIONAL_OBRIGATORIO" }`
- **Query obrigatória `versaoAtual`** — string livre (compara semver-ish)
- **Backend filtra `versoes_agente` por `sistema_operacional`** — retorna a última versão ativa do SO indicado
- **Auth:** endpoint aceita tanto `X-Ativacao-Key` quanto Device JWT (compat Windows atual)

**Response:** mesmo shape atual:

```json
{
  "atualizacaoDisponivel": true,
  "versaoNova": "1.0.1",
  "urlDownload": "https://apk.imanager.trivion.com.br/updates/prod/1.0.1.apk",
  "checksumSha256": "abc123...",
  "obrigatoria": false
}
```

**DTO Zod:**

```typescript
export const VerificarAtualizacaoQueryDto = z.object({
  sistemaOperacional: z.enum(['WINDOWS', 'MACOS', 'ANDROID', 'IOS']),
  versaoAtual: z.string().min(1).max(50),
});
```

**Testes de aceitação:**

1. `?sistemaOperacional=ANDROID&versaoAtual=1.0.0` retorna a última versão Android ativa
2. `?sistemaOperacional=WINDOWS&versaoAtual=1.4.4` retorna a última versão Windows ativa
3. Sem `sistemaOperacional` → HTTP 400
4. `sistemaOperacional=ANDROID` mas nenhuma versão Android publicada → `{ atualizacaoDisponivel: false }`
5. `versaoAtual == versaoNova` → `{ atualizacaoDisponivel: false }`

---

### 4.3 `POST /api/agente/atualizacoes/resultado`

**Estado atual (Java):** aceita `{ versaoTentada, sucesso, motivoErro? }`. Auth Device JWT.

**Alteração no Node:**

- **Nenhuma mudança de contrato** — Android usa idêntico ao Windows.
- **Persistência:** logar em `audit_events` com `evento = 'UPDATE_RESULT'`, `detalhes = { versaoTentada, sucesso, motivoErro, sistemaOperacional }` — o `sistemaOperacional` sai do agente (JOIN em `agentes.dispositivo_tipo`).

**Testes de aceitação:**

1. POST com `{sucesso: true, versaoTentada: "1.0.1"}` de agente Android → audit event registrado com `sistemaOperacional=ANDROID`
2. POST com `{sucesso: false, motivoErro: "CHECKSUM_MISMATCH"}` → audit event registrado

---

### 4.4 `POST /api/agente/auditoria/registrar`

**Estado atual (Java):** aceita `{ evento, instalacaoId, agenteId, timestampUtc, dados }`. Fire-and-forget, Bearer Device JWT. Persiste em `audit_events`.

**Alteração no Node:**

- **Contrato inalterado.**
- **Aceitar 11 novos valores de `evento` sem rejeitar:** o backend não deve validar enum de `evento` — aceita string livre e persiste. Isso já é o comportamento atual e deve ser preservado.
- **Documentar (Swagger) os novos eventos Android-específicos** com `@ApiProperty` no exemplo:

Lista dos novos audit events Android (só documentação — backend não valida):

| Evento | Contexto |
|---|---|
| `AGENT_INSTALADO_ANDROID` | Primeira execução pós-install |
| `TERMOS_ACEITOS_ANDROID` | Colaborador aceitou termo individual na 1ª tela |
| `AGENT_VINCULADO_ANDROID` | Vinculação bem-sucedida |
| `AGENT_VINCULACAO_FALHOU_ANDROID` | Vinculação falhou (com `dados.motivo`) |
| `ACCESSIBILITY_ATIVADA` | Colaborador ativou AccessibilityService em Settings |
| `ACCESSIBILITY_DESATIVADA` | Colaborador desativou (perda de coleta iminente) |
| `USAGE_STATS_ATIVADA` | Colaborador ativou UsageStats permission |
| `USAGE_STATS_DESATIVADA` | Colaborador desativou |
| `OEM_PROBLEMATICO_DETECTADO` | Xiaomi/Huawei/Oppo detectado + status (`dados.fabricante`) |
| `SERVICE_MORTO_E_REINICIADO` | Watchdog reviveu ForegroundService |
| `AGENT_REVOGADO_ANDROID` | Última mensagem antes de parar coleta (backend retornou 403) |
| `UPDATE_INICIADO_ANDROID` | Download de atualização começou |
| `UPDATE_SUCESSO_ANDROID` | Atualização instalada com sucesso |
| `UPDATE_FALHOU_ANDROID` | Atualização falhou (com `dados.motivo`) |

**Testes de aceitação:**

1. POST audit `evento=OEM_PROBLEMATICO_DETECTADO`, `dados={fabricante: "XIAOMI"}` → persiste em `audit_events`
2. POST audit `evento=ACCESSIBILITY_DESATIVADA` sem `dados` → persiste com `dados = {}`
3. POST audit sem `agenteId` no body → HTTP 400 (comportamento existente Java, manter)
4. POST audit com string `evento` não listada acima (`evento=EVENTO_INVENTADO`) → aceita e persiste (backend não valida enum)

---

### 4.5 `POST /api/agent/error-report`

**Estado atual (Java):** aceita `{ tipo, mensagem, stackTrace, versao, sistemaOperacional, maquina, colaboradorId, cnpj }`. 18 tipos existentes (FATAL_CRASH, UPDATE_CRASH, TOKEN_REFRESH_FAILED, etc.). Throttle 5min por tipo (side backend? confirmar). Sem Sentry.

**Alteração no Node:**

- **Contrato inalterado** — mesmo shape.
- **Adicionar `ANR` como tipo aceito** (Android-specific). Documentar no Swagger.
- **`sistemaOperacional` no body agora aceita string livre** (`"Android 13 (API 33)"` ou `"Windows 11 Pro"`). Não validar contra enum — dado histórico Windows tem strings variadas.
- **`cnpj` no body:** já extraído do JWT no Java atual? Confirmar. Se sim, ignorar body e usar JWT. Se não, deixar como está.

**Testes de aceitação:**

1. POST `tipo=ANR`, `mensagem="Main thread blocked 6000ms"` de agente Android → persiste em `audit_events` ou tabela dedicada (mesma tabela que Windows usa hoje)
2. POST `tipo=FATAL_CRASH` de agente Android com `sistemaOperacional="Android 14"` → persiste
3. Throttle 5min: 3 POSTs consecutivos do mesmo `tipo` do mesmo agente dentro de 5min → apenas 1 registrado (comportamento existente Java, manter)

---

### 4.6 `POST /api/agente/v1/colaboradores/validar`

**Estado atual (Java):** aceita `{ identificador, tipoIdentificador? }` via body + `X-Ativacao-Key` header. Retorna `{ existe, usuarioId?, nomeCompleto?, mensagem? }`.

**Alteração no Node:**

- **Nenhuma mudança** — Android usa idêntico ao Windows.
- **Confirmar** que a busca por identificador é case-insensitive (`ux_usuarios_empresa_identificador_ci` no schema).
- **Confirmar** que retorna `false` para colaborador sem `acessa_agent_mobile=true` (se essa flag existir — checar com @Steve na rodada futura). Por enquanto: retorna `true` se colaborador existe e está ativo.

**Testes de aceitação:**

1. Validar identificador existente → `{existe: true, usuarioId: N, nomeCompleto: "..."}`
2. Validar identificador inexistente → `{existe: false, mensagem: "..."}`
3. Chave de ativação inválida → HTTP 401

---

### 4.7 `POST /api/agente/auth/refresh`

**Estado atual (Java):** aceita `{ refreshToken }`, retorna `{ deviceToken, refreshToken }` (rotation). Detecção de theft: reutilização de token revogado invalida todos os tokens da instalação.

**Alteração no Node:**

- **Nenhuma mudança** — Android usa idêntico ao Windows.
- **Herdar UPDATE atômico** (regra da @Shuri no `jwt-contract.md` §6.4 — `UPDATE ... WHERE revoked_at IS NULL RETURNING`) pra evitar falso `TOKEN_THEFT_DETECTED` em race condition.

**Testes de aceitação:**

1. Refresh com token válido → novo par emitido, antigo revogado
2. Reutilização de token revogado → HTTP 401 + audit `TOKEN_THEFT_DETECTED` + revoga todos os tokens do agente
3. Refresh de agente Android funciona idêntico ao Windows

---

### 4.8 Header `X-Agent-Revoked` (novo — em responses)

**Contexto:** quando o Marcos/gestor revoga o Agent Mobile do colaborador via portal (rodada futura), o backend precisa avisar o APK Android pra ele **parar de coletar** silenciosamente e ficar dormente.

**Novo comportamento:**

- **Endpoint afetado:** todos os endpoints autenticados por Device JWT (basicamente todos exceto vincular/validar).
- **Trigger:** se o agente estiver marcado como `revogado_em IS NOT NULL` na tabela `agentes` (nova coluna futura, mapeada abaixo), o middleware Node responde:
  - HTTP status **403**
  - Header **`X-Agent-Revoked: true`**
  - Body `{ codigo: "AGENT_REVOGADO", mensagem: "Este agente foi revogado. Coleta encerrada." }`

**Migration adicional (opcional MVP — pode ficar pra rodada futura de revogação self-service):**

```sql
ALTER TABLE agentes
  ADD COLUMN revogado_em TIMESTAMPTZ NULL;
```

Nota: nesta rodada, coluna e header ficam **implementados** mas o mecanismo que **seta** `revogado_em` (endpoint `POST /api/v1/agent-mobile/revogar` chamado pelo colaborador via portal) fica pra rodada futura. Vale implementar agora pra APK Android já reagir corretamente quando o feature de revogação existir.

**DTO Zod pra middleware:**

```typescript
// src/agent/middleware/revocation-check.middleware.ts
async function checkRevocation(agenteId: number, res: Response): Promise<boolean> {
  const agente = await db.select().from(agentes).where(eq(agentes.id, agenteId)).limit(1);
  if (agente[0]?.revogadoEm) {
    res.setHeader('X-Agent-Revoked', 'true');
    res.status(403).json({ codigo: 'AGENT_REVOGADO', mensagem: 'Este agente foi revogado. Coleta encerrada.' });
    return true;
  }
  return false;
}
```

**Testes de aceitação:**

1. Agente com `revogado_em=NULL` → requests passam normalmente
2. Agente com `revogado_em=NOW()` → HTTP 403 + `X-Agent-Revoked: true` em qualquer endpoint autenticado
3. Após 403, próximo request também retorna 403 (idempotente)

---

## 5. Fluxo end-to-end do Agent Android (referência)

```
1. APK dispara POST /api/agente/v1/colaboradores/validar (X-Ativacao-Key)
   → resposta {existe: true}
2. APK dispara POST /api/agente/dispositivos/vincular (X-Ativacao-Key)
   → body inclui dispositivoTipo=ANDROID
   → resposta {agenteId, usuarioId, deviceToken, refreshToken}
3. APK começa a coletar eventos
4. A cada 15min APK dispara POST /api/agent/events (srv-events-node, Device JWT)
5. A cada 6h APK dispara GET /api/agente/atualizacoes/verificar?sistemaOperacional=ANDROID&versaoAtual=...
6. Sob demanda APK dispara POST /api/agente/auditoria/registrar (eventos AGENT_INSTALADO_ANDROID, etc.)
7. Em crash APK dispara POST /api/agent/error-report (tipo ANR/FATAL_CRASH)
8. Quando Device JWT expira APK dispara POST /api/agente/auth/refresh
9. Após install de update APK dispara POST /api/agente/atualizacoes/resultado
10. Se gestor revoga (futuro) → próximo request retorna 403 + X-Agent-Revoked → APK para de coletar
```

## 6. Testes de contrato (parity + fixtures)

Adicionar em `test/fixtures/`:

- `vincular-android-request.json` — payload que APK Android envia
- `vincular-android-response.json` — resposta esperada
- `audit-android-events.json` — array com os 11 audit events Android + payloads exemplo
- `error-report-anr.json` — payload de ANR

Rodar contra srv-admin-node e validar contratos idênticos aos que o APK espera (documentados em `.brain/tecnologia/plans/2026-08-08-agent-android-byod-mvp.md` T29, T38, T39).

## 7. Ordem de implementação sugerida (para @Shuri)

1. **Migrations Drizzle** (schema-drift-report.md) — `dispositivo_tipo`, `sistema_operacional`, `revogado_em`
2. **Enum `TipoDispositivo`** (`src/enums/tipo-dispositivo.enum.ts`)
3. **Zod DTO extension** (`VincularAgenteBodyDto`, `VerificarAtualizacaoQueryDto`)
4. **Service `AgentesService.vincular`** — persistir `dispositivo_tipo`
5. **Service `VersoesAgenteService.buscarMaisRecente(so)`** — filtro por SO
6. **Middleware `RevocationCheckMiddleware`** — 403 + header
7. **Testes de contrato + parity + unit** — cobrir cada caso listado nas seções 4.1-4.8
8. **Swagger docs** — atualizar `@ApiOperation` de cada endpoint alterado com exemplos Android

## 8. Definition of Done

- [ ] Migrations aplicadas em staging + validadas
- [ ] 7 endpoints estendidos conforme seção 4
- [ ] Header `X-Agent-Revoked` implementado com middleware
- [ ] Enum `TipoDispositivo` + Zod DTOs criados
- [ ] Cobertura linha ≥80% + branch ≥95% nos arquivos alterados
- [ ] Fixtures em `test/fixtures/` cobrindo cenários Android
- [ ] Parity runner passa (contratos idênticos ao Java quando aplicável)
- [ ] Swagger atualizado com exemplos Android em cada endpoint
- [ ] Deploy staging + smoke test manual: vincular Agent Windows continua funcionando (regressão)
- [ ] Deploy prod aprovado por Marcos

## 9. Dependências e riscos

| Dep/Risco | Mitigação |
|---|---|
| Regressão Windows (Agent existente quebra) | Testes de contrato + parity runner mantidos; `dispositivo_tipo` tem DEFAULT `WINDOWS` |
| UNIQUE `(versao, sistema_operacional)` quebra se houver linha duplicada no legado | Query auditoria antes da migration: `SELECT versao, COUNT(*) FROM versoes_agente GROUP BY versao HAVING COUNT(*) > 1;` — se retornar linhas, resolver antes |
| Race condition no refresh token durante migração | Aplicar UPDATE atômico (`jwt-contract.md` §6.4) desde o dia 1 no Node |
| Enum `ANDROID` chegar no Agent antes do backend suportar | Ordem inegociável: srv-admin-node em PROD com essa spec ANTES do dev Android começar T29 (vinculação). Regra `feedback_enum_novo_backend_antes.md` aplicada. |

## 10. Referências cruzadas

- Spec principal Agent Android: `.brain/tecnologia/specs/2026-08-08-agent-android-byod-mvp-design.md`
- Plano de implementação Agent Android: `.brain/tecnologia/plans/2026-08-08-agent-android-byod-mvp.md`
- Contrato JWT (mantido inalterado): `.brain/tecnologia/autenticacao/jwt-contract.md`
- Schema banco atual: `.brain/_shared/banco-dados.md` (`agentes`, `versoes_agente`, `audit_events`)
- srv-admin Java atual (referência de shape): `.brain/tecnologia/services/srv-admin/README.md`
- Regra enum-antes-do-Agent: `feedback_enum_novo_backend_antes.md`
- Regra "Java não muda": Marcos 2026-07-21 — Node herda e estende, Java fica congelado

## 11. Fora do escopo — mapeamento para rodadas futuras

Documentar aqui pra @Shuri não implementar por engano:

| Item | Rodada futura |
|---|---|
| Endpoint `POST /api/v1/agent-mobile/gerar-apk` (portal-node — Gestor gera APK personalizado) | Rodada portal |
| Endpoint `POST /api/v1/agent-mobile/revogar` (colaborador revoga via portal) | Rodada portal (usa `revogado_em` já criada aqui) |
| Endpoint `POST /api/agente/logs/upload` (upload de logs sob demanda) | Rodada logs remotos |
| `GET /api/agente/config` estendido com `enviarLogsAte` e `logLevel` dinâmico | Rodada logs remotos |
| Pipeline server-side de re-empacotamento APK (`apksigner` + inject `config.json`) | Rodada portal |
| Página pública `/agent-mobile/<token>` mobile-friendly | Rodada portal |
| Push notification de update pra APK Android | Rodada notifications |
