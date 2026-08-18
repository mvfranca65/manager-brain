> **STATUS:** ATIVA — aguardando execução por @Shuri
> **DATA:** 2026-08-08
> **DONO ARQUITETURAL:** @Tony
> **EXECUTOR:** @Shuri (dona técnica do srv-events-node)
> **DEPENDÊNCIA:** habilitador da spec `.brain/tecnologia/specs/2026-08-08-agent-android-byod-mvp-design.md`
> **PREMISSA:** deve estar 100% em produção **antes** do desenvolvimento do Agent Android começar (regra Marcos 2026-08-08)
> **PAR:** spec complementar `.brain/tecnologia/specs/2026-08-08-srv-admin-node-suporte-agent-android.md` (deploy conjunto)

# srv-events-node — Suporte Agent Android BYOD

## 1. Contexto

O Agent Android (spec `2026-08-08-agent-android-byod-mvp-design.md`) envia eventos comportamentais pro srv-events, com paridade Windows + 1 tipo novo (`PhoneCallEvent` de chamada telefônica). O srv-events-node em migração pela @Shuri precisa nascer aceitando:

- Novo tipo de evento `PhoneCallEvent` (CALL_START/CALL_END, sem número por LGPD)
- Novo campo `dispositivoTipo=ANDROID` no batch e nos registros persistidos
- `WindowActivityEvent.urlDominio` continua opcional/vazio (já deve funcionar hoje — validar)
- Header `X-Agent-Revoked` em respostas 403 (idêntico ao srv-admin-node — spec par)
- Migration adicional: `dispositivo_tipo` denormalizado em tabelas `eventos_*`

Não há **novos endpoints** — só **estensão do endpoint único** `POST /api/agent/events`.

## 2. Escopo

### 2.1 Dentro do escopo

- Extensão do endpoint `POST /api/agent/events` (único endpoint de negócio do service)
- Extensão de 1 endpoint de health (`/actuator/health` mantido, sem mudança)
- 8 migrations Drizzle (uma coluna nova em cada tabela `eventos_*`)
- 1 novo tipo de evento `PhoneCallEvent`
- Header `X-Agent-Revoked` no middleware (compartilha lógica com srv-admin-node)
- Testes de contrato + parity + unit

### 2.2 Fora do escopo

- Novos endpoints (nenhum precisa nascer)
- Alterações no fluxo de validação Device JWT (mantém idêntico)
- Purge de eventos (mantém retenção 180 dias existente)
- Índices agregados por `dispositivo_tipo` — só coluna, análise de performance depois

## 3. Alterações de schema (Drizzle migrations)

Todas as tabelas de eventos ganham `dispositivo_tipo` denormalizado (default `WINDOWS`).

### 3.1 Migration 1 — `eventos_janela.dispositivo_tipo`

```sql
ALTER TABLE eventos_janela
  ADD COLUMN dispositivo_tipo VARCHAR(20) NOT NULL DEFAULT 'WINDOWS';

ALTER TABLE eventos_janela
  ADD CONSTRAINT chk_eventos_janela_dispositivo_tipo
  CHECK (dispositivo_tipo IN ('WINDOWS', 'MACOS', 'ANDROID', 'IOS'));
```

Rationale de denormalização: srv-ia lê `eventos_janela` diretamente pra montar timeline. Query `WHERE dispositivo_tipo=?` evita JOIN com `agentes` (que teria custo em queries de janela semanal com milhões de linhas).

### 3.2 Migrations 2-8 — mesmo padrão nas outras tabelas

```sql
-- eventos_ociosidade
ALTER TABLE eventos_ociosidade ADD COLUMN dispositivo_tipo VARCHAR(20) NOT NULL DEFAULT 'WINDOWS';
ALTER TABLE eventos_ociosidade ADD CONSTRAINT chk_eventos_ociosidade_dispositivo_tipo
  CHECK (dispositivo_tipo IN ('WINDOWS', 'MACOS', 'ANDROID', 'IOS'));

-- batimentos
ALTER TABLE batimentos ADD COLUMN dispositivo_tipo VARCHAR(20) NOT NULL DEFAULT 'WINDOWS';
ALTER TABLE batimentos ADD CONSTRAINT chk_batimentos_dispositivo_tipo
  CHECK (dispositivo_tipo IN ('WINDOWS', 'MACOS', 'ANDROID', 'IOS'));

-- resumos_atividade_input
ALTER TABLE resumos_atividade_input ADD COLUMN dispositivo_tipo VARCHAR(20) NOT NULL DEFAULT 'WINDOWS';
ALTER TABLE resumos_atividade_input ADD CONSTRAINT chk_resumos_atividade_input_dispositivo_tipo
  CHECK (dispositivo_tipo IN ('WINDOWS', 'MACOS', 'ANDROID', 'IOS'));

-- eventos_sessao
ALTER TABLE eventos_sessao ADD COLUMN dispositivo_tipo VARCHAR(20) NOT NULL DEFAULT 'WINDOWS';
ALTER TABLE eventos_sessao ADD CONSTRAINT chk_eventos_sessao_dispositivo_tipo
  CHECK (dispositivo_tipo IN ('WINDOWS', 'MACOS', 'ANDROID', 'IOS'));

-- eventos_reuniao
ALTER TABLE eventos_reuniao ADD COLUMN dispositivo_tipo VARCHAR(20) NOT NULL DEFAULT 'WINDOWS';
ALTER TABLE eventos_reuniao ADD CONSTRAINT chk_eventos_reuniao_dispositivo_tipo
  CHECK (dispositivo_tipo IN ('WINDOWS', 'MACOS', 'ANDROID', 'IOS'));

-- eventos_transicao_status
ALTER TABLE eventos_transicao_status ADD COLUMN dispositivo_tipo VARCHAR(20) NOT NULL DEFAULT 'WINDOWS';
ALTER TABLE eventos_transicao_status ADD CONSTRAINT chk_eventos_transicao_status_dispositivo_tipo
  CHECK (dispositivo_tipo IN ('WINDOWS', 'MACOS', 'ANDROID', 'IOS'));
```

### 3.3 Migration 8 (nova tabela) — `eventos_chamada_telefonica`

Nova tabela pra `PhoneCallEvent` (não cabe em `eventos_reuniao` porque tem semântica distinta — chamada é ponto-a-ponto sem "aplicativo"):

```sql
CREATE TABLE eventos_chamada_telefonica (
  id BIGSERIAL PRIMARY KEY,
  empresa_id BIGINT NOT NULL,
  agente_id BIGINT NOT NULL REFERENCES agentes(id),
  dispositivo_tipo VARCHAR(20) NOT NULL DEFAULT 'ANDROID',
  tipo_transicao VARCHAR(20) NOT NULL,
  ocorreu_em TIMESTAMPTZ NOT NULL,
  criado_em TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  CONSTRAINT chk_eventos_chamada_telefonica_tipo_transicao
    CHECK (tipo_transicao IN ('CALL_START', 'CALL_END')),
  CONSTRAINT chk_eventos_chamada_telefonica_dispositivo_tipo
    CHECK (dispositivo_tipo IN ('ANDROID', 'IOS'))
);

CREATE INDEX idx_eventos_chamada_telefonica_agente_ocorreu
  ON eventos_chamada_telefonica(agente_id, ocorreu_em DESC);

CREATE INDEX idx_eventos_chamada_telefonica_empresa_ocorreu
  ON eventos_chamada_telefonica(empresa_id, ocorreu_em DESC);
```

**Nota LGPD:** tabela **NÃO** tem coluna pra número/contato/duração_por_contato — só start/end como transição. Garantia arquitetural de que não vazamos dado sensível mesmo se APK "quiser" enviar.

### 3.4 Enum TS canônico

Reutilizar `TipoDispositivo` da spec srv-admin-node (mesma enum). Se srv-events-node tem tipos separados, criar cópia em `src/enums/tipo-dispositivo.enum.ts` idêntica.

## 4. Alterações no endpoint `POST /api/agent/events`

### 4.1 Estado atual (Java)

Aceita batch:

```json
{
  "maquinaId": "desktop-123",
  "identificadorColaborador": "I455676",
  "instalacaoId": "installation-456",
  "versaoAgente": "1.2.3",
  "descricaoSo": "Windows 11 Pro",
  "enviadoEm": "2024-03-01T12:00:00Z",
  "eventos": [
    { "tipoEvento": "WindowActivityEvent", "dados": {...} },
    ...
  ]
}
```

Retorna 202 se aceito, 400 se payload inválido, 401 se JWT inválido, 404 se instalação não existe, 412 se dispositivo não vinculado, 500 se erro interno.

### 4.2 Alterações no Node

**Body — 1 campo novo obrigatório:**

- `dispositivoTipo: "WINDOWS" | "MACOS" | "ANDROID" | "IOS"` — obrigatório. Se ausente ou inválido: HTTP 400 `{ codigo: "DISPOSITIVO_TIPO_INVALIDO" }`

**Persistência — dispositivoTipo denormalizado:**

- Ao persistir cada evento, gravar `dispositivo_tipo` na coluna nova de cada tabela `eventos_*`
- Valor vem do body do batch (não do JOIN em `agentes` — evita 1 query extra por batch)
- **Validação de coerência:** se `body.dispositivoTipo != agentes.dispositivo_tipo` do agente resolvido pelo JWT → log WARN mas aceita (permite migração de dispositivo — cenário raro mas possível)

**Novo tipo de evento aceito:**

```json
{
  "tipoEvento": "PhoneCallEvent",
  "dados": {
    "tipoTransicao": "CALL_START" | "CALL_END",
    "ocorreuEm": "2026-08-08T14:00:00Z"
  }
}
```

Persistido em `eventos_chamada_telefonica`. **Não** aceita `PhoneCallEvent` de agente Windows/macOS (validação: `dispositivoTipo IN ('ANDROID', 'IOS')`) → se Windows tentar mandar, log WARN + ignora esse evento sem quebrar o batch.

**`WindowActivityEvent.urlDominio` opcional/vazio:**

- Já era opcional no Java. Confirmar que Node aceita `urlDominio: ""` sem quebrar validação Zod.
- Se Android manda vazio (porque `UrlDomainExtractor` não pegou), persiste string vazia — não NULL — pra manter compat com queries existentes.

### 4.3 DTOs Zod

```typescript
// src/agent-events/dto/agent-event.dto.ts

export const WindowActivityEventData = z.object({
  nomeProcesso: z.string().min(1).max(255),
  tituloJanela: z.string().max(500).default(''),
  iniciadoEm: z.string().datetime(),
  finalizadoEm: z.string().datetime(),
  statusUsuario: z.string().max(50).default('ATIVO'),
  urlDominio: z.string().max(255).default(''),
});

export const IdleEventData = z.object({
  iniciadoEm: z.string().datetime(),
  finalizadoEm: z.string().datetime(),
  statusUsuario: z.string().max(50).default('AUSENTE'),
});

export const HeartbeatEventData = z.object({
  enviadoEm: z.string().datetime(),
  eventosPendentes: z.number().int().nonnegative().default(0),
});

export const SessionEventData = z.object({
  tipoTransicao: z.enum(['LOCK', 'UNLOCK', 'LOGIN', 'LOGOUT']),
  ocorreuEm: z.string().datetime(),
});

export const ActivitySummaryEventData = z.object({
  iniciadoEm: z.string().datetime(),
  finalizadoEm: z.string().datetime(),
  teclasPressionadas: z.number().int().nonnegative(),
  cliquesMouseEsq: z.number().int().nonnegative(),
  cliquesMouseDir: z.number().int().nonnegative(),
  cliquesMouseMeio: z.number().int().nonnegative(),
  movimentosMouse: z.number().int().nonnegative(),
  scrollsVertical: z.number().int().nonnegative(),
  padraoDigitacao: z.string().max(50).default('NORMAL'),
});

export const MeetingEventData = z.object({
  iniciadoEm: z.string().datetime(),
  finalizadoEm: z.string().datetime(),
  aplicativo: z.string().min(1).max(100),  // "Zoom Android", "Teams", etc.
});

export const PhoneCallEventData = z.object({
  tipoTransicao: z.enum(['CALL_START', 'CALL_END']),
  ocorreuEm: z.string().datetime(),
});

export const StatusTransitionEventData = z.object({
  statusAnterior: z.string().max(50),
  statusNovo: z.string().max(50),
  transicaoEm: z.string().datetime(),
});

// Discriminated union pelo campo tipoEvento
export const AgentEventDto = z.discriminatedUnion('tipoEvento', [
  z.object({ tipoEvento: z.literal('WindowActivityEvent'), dados: WindowActivityEventData }),
  z.object({ tipoEvento: z.literal('IdleEvent'), dados: IdleEventData }),
  z.object({ tipoEvento: z.literal('HeartbeatEvent'), dados: HeartbeatEventData }),
  z.object({ tipoEvento: z.literal('SessionEvent'), dados: SessionEventData }),
  z.object({ tipoEvento: z.literal('ActivitySummaryEvent'), dados: ActivitySummaryEventData }),
  z.object({ tipoEvento: z.literal('MeetingEvent'), dados: MeetingEventData }),
  z.object({ tipoEvento: z.literal('PhoneCallEvent'), dados: PhoneCallEventData }),
  z.object({ tipoEvento: z.literal('StatusTransitionEvent'), dados: StatusTransitionEventData }),
]);

export const AgentEventBatchBodyDto = z.object({
  maquinaId: z.string().min(1).max(255),
  identificadorColaborador: z.string().min(1).max(64).optional(),  // opcional se JWT tem
  instalacaoId: z.string().uuid(),
  versaoAgente: z.string().min(1).max(50),
  descricaoSo: z.string().min(1).max(255),
  dispositivoTipo: z.enum(['WINDOWS', 'MACOS', 'ANDROID', 'IOS']),  // NOVO obrigatório
  enviadoEm: z.string().datetime(),
  eventos: z.array(AgentEventDto).max(1000),  // máximo 1000 eventos por batch
});
```

### 4.4 Response

Mesmo shape atual — HTTP 202 + body com contagens:

```json
{
  "totalRecebido": 42,
  "totalPersistido": 42,
  "totalIgnorado": 0,
  "motivosIgnorados": []
}
```

Se algum evento foi ignorado (ex: PhoneCall vindo de Windows), aparece em `motivosIgnorados`:

```json
{
  "totalRecebido": 42,
  "totalPersistido": 41,
  "totalIgnorado": 1,
  "motivosIgnorados": [
    { "indice": 15, "tipoEvento": "PhoneCallEvent", "motivo": "PhoneCallEvent so aceito para dispositivos moveis (ANDROID/IOS)" }
  ]
}
```

### 4.5 Códigos de retorno

Manter todos os códigos existentes do Java + adicionar:

| HTTP | Código | Significado (novo/existente) |
|---|---|---|
| 202 | — | Batch aceito e processado (existente) |
| 400 | `DISPOSITIVO_TIPO_INVALIDO` | `dispositivoTipo` ausente ou fora do enum (NOVO) |
| 400 | `PAYLOAD_INVALIDO` | Batch vazio, algum evento inválido, batch > 1000 eventos (existente) |
| 401 | `CHAVE_INVALIDA` | JWT ausente/inválido (existente) |
| 403 | `AGENT_REVOGADO` | Header `X-Agent-Revoked: true` — agente revogado (NOVO, ver §5) |
| 404 | `DISPOSITIVO_NAO_CADASTRADO` | `instalacaoId` não existe (existente) |
| 412 | `DISPOSITIVO_NAO_VINCULADO` | Sem `usuario_ref_id` (existente) |
| 500 | `ERRO_INTERNO` | Falha não tratada (existente) |

## 5. Header `X-Agent-Revoked` (compartilhado com srv-admin-node)

**Mesmo comportamento da spec par `2026-08-08-srv-admin-node-suporte-agent-android.md` §4.8:**

- Middleware Node checa `agentes.revogado_em` a cada request autenticado por Device JWT
- Se `revogado_em IS NOT NULL`: HTTP 403 + `X-Agent-Revoked: true` header + body `{ codigo: "AGENT_REVOGADO" }`
- APK detecta esse header e para de coletar

**Reuso:** extrair `RevocationCheckMiddleware` pra pacote compartilhado (`@manager/shared-middleware`) ou duplicar impl idêntica se preferir. Coordenar com @Shuri na decisão.

## 6. Testes de contrato + parity + unit

### 6.1 Fixtures (compartilhadas com spec srv-admin-node)

Adicionar em `test/fixtures/agent-events/`:

- `batch-android-window-activity.json` — batch Android com só WindowActivityEvent
- `batch-android-completo.json` — batch com todos os 8 tipos de evento + PhoneCallEvent
- `batch-android-phonecall-only.json` — só PhoneCallEvent (CALL_START + CALL_END)
- `batch-windows-legacy.json` — batch Windows sem `dispositivoTipo` no body (validar rejeição HTTP 400)
- `batch-windows-com-dispositivo-tipo.json` — batch Windows com `dispositivoTipo=WINDOWS` (validar aceitação)

### 6.2 Testes unitários mínimos

- `AgentEventsService.processarBatch()`:
  1. Batch sem `dispositivoTipo` → HTTP 400
  2. Batch com `dispositivoTipo=ANDROID` + eventos → persiste com `dispositivo_tipo=ANDROID` em cada tabela
  3. Batch com `dispositivoTipo=WINDOWS` + `PhoneCallEvent` → ignora esse evento (retorna em `motivosIgnorados`), aceita resto
  4. Batch com `dispositivoTipo=ANDROID` + `PhoneCallEvent CALL_START` → persiste em `eventos_chamada_telefonica`
  5. Batch com >1000 eventos → HTTP 400 `PAYLOAD_INVALIDO`
  6. Batch com `WindowActivityEvent.urlDominio=""` → persiste vazio (não NULL)
  7. Divergência `body.dispositivoTipo != agente.dispositivo_tipo` → log WARN + aceita (não rejeita)

- `RevocationCheckMiddleware`:
  1. Agente com `revogado_em=NULL` → next() chamado
  2. Agente com `revogado_em=NOW()` → 403 + `X-Agent-Revoked: true`

### 6.3 Parity runner (opcional — Node herda comportamento Java)

Se @Shuri usar parity runner (contra Java atual em staging):

- Enviar mesmo batch pra Java e Node
- Comparar respostas
- **Delta esperado:** Node valida `dispositivoTipo` obrigatório (400), Java não. Registrar como `intentionalDiff` com `sunsetAt = <data do desligamento do Java>`.

## 7. Ordem de implementação sugerida (para @Shuri)

1. **8 migrations Drizzle** (§3.1-§3.3) — testar em staging + prod
2. **Enum `TipoDispositivo`** (`src/enums/tipo-dispositivo.enum.ts` — mesmo da spec srv-admin-node)
3. **Zod DTOs** — atualizar `AgentEventBatchBodyDto` + adicionar `PhoneCallEventData`
4. **Repository `EventosRepository.persistirBatch()`** — passar `dispositivo_tipo` em cada INSERT
5. **Nova entity + repository `EventosChamadaTelefonicaRepository`**
6. **Service `AgentEventsService.processarBatch()`** — lógica de ignorar `PhoneCallEvent` de dispositivo não-mobile
7. **Middleware `RevocationCheckMiddleware`** — compartilhado com srv-admin-node
8. **Testes de contrato + unit + fixtures** — cobrir cada caso listado §6
9. **Swagger docs** — atualizar `@ApiOperation` do endpoint `POST /api/agent/events` com exemplos dos 8 tipos de evento (incluindo `PhoneCallEvent`) e `dispositivoTipo`

## 8. Definition of Done

- [ ] 8 migrations Drizzle aplicadas em staging + validadas
- [ ] `dispositivo_tipo` denormalizado gravado em cada tabela `eventos_*` corretamente
- [ ] `eventos_chamada_telefonica` criada, com CHECK constraint e índices
- [ ] Endpoint `POST /api/agent/events` aceita `dispositivoTipo` no body
- [ ] Endpoint aceita `PhoneCallEvent` como tipo novo
- [ ] Rejeição correta de `PhoneCallEvent` vindo de Windows (aceito no batch, ignora individualmente)
- [ ] Middleware `X-Agent-Revoked` funcional
- [ ] Cobertura linha ≥80% + branch ≥95% nos arquivos alterados
- [ ] Fixtures em `test/fixtures/agent-events/` cobrindo Windows + Android + phonecall
- [ ] Parity runner passa (ou `intentionalDiff` registrado se aplicável)
- [ ] Swagger atualizado com exemplos dos 8 tipos + `dispositivoTipo`
- [ ] Deploy staging + smoke test manual:
  - [ ] Agent Windows atual continua funcionando (regressão)
  - [ ] Batch teste com `dispositivoTipo=ANDROID` + PhoneCallEvent persiste corretamente
- [ ] Deploy prod aprovado por Marcos

## 9. Dependências e riscos

| Dep/Risco | Mitigação |
|---|---|
| Regressão Agent Windows (quebra ingest) | `dispositivo_tipo` com DEFAULT `WINDOWS` cobre agents legados; se Java continuar enviando sem o campo até desligar, aceitar durante coexistência com WARN |
| Batch grande com 1000+ eventos aumenta latência de INSERT | Manter limite `eventos.max(1000)` no Zod DTO; @Shuri considera batch INSERT (`INSERT INTO x VALUES (...), (...), (...)`) pra eficiência |
| `eventos_chamada_telefonica` cresce fora do controle (se OEMs mandam muitos start/end espúrios) | Adicionar ao purge existente (retenção 180 dias como outras `eventos_*`) — atualizar `PurgeEventosService` |
| Race condition: agente revogado durante batch in-flight | Middleware checa `revogado_em` **antes** do processamento — rejeita batch inteiro com 403 se revogado |
| `PhoneCallEvent` mudou spec futuramente (novos campos como duração_ms) | Zod DTO estende sem quebrar cliente; APK Android manda só os campos que conhece |

## 10. Impacto no srv-ia (fora do escopo desta spec — informar @Jarvis)

srv-ia lê `eventos_*` diretamente pra montar timeline e calcular Score IA. Com Android:

- `eventos_janela` da mesa Windows + celular Android para mesmo colaborador → timeline mista
- Pilar Foco calculado hoje com base em janela ativa no PC — Android confunde (uso de celular pesa como "não foco"?)
- Marcos decidiu: **Score IA fica calibrado só para dados PC no MVP** (regra spec §4.2 princípio 5)

**Ação recomendada pro srv-ia (rodada futura @Jarvis):**

Filtrar `WHERE dispositivo_tipo = 'WINDOWS'` no `ExtratorDadosService.buscarEventosDoPeriodo` até calibração v2. Simples, isola impacto, permite Android coletar sem quebrar Score IA existente.

Alternativa: flag configurável em `application.yml` — `srv-ia.incluir-dispositivos-mobile-no-score: false` (default) — permite ativação gradual quando calibração v2 estiver pronta.

Coordenar com @Jarvis quando spec srv-ia mobile calibração for aberta (rodada futura).

## 11. Referências cruzadas

- Spec principal Agent Android: `.brain/tecnologia/specs/2026-08-08-agent-android-byod-mvp-design.md`
- Plano de implementação Agent Android: `.brain/tecnologia/plans/2026-08-08-agent-android-byod-mvp.md`
- Spec par srv-admin-node: `.brain/tecnologia/specs/2026-08-08-srv-admin-node-suporte-agent-android.md`
- Schema atual `eventos_*`: `.brain/_shared/banco-dados.md` §"Eventos"
- srv-events Java (referência): `.brain/tecnologia/services/srv-events/README.md`
- Bugs em prod do Agent (contexto): `feedback_bugs_prod_agent_2026_08_04.md`

## 12. Fora do escopo — mapeamento para rodadas futuras

| Item | Rodada futura |
|---|---|
| Ajuste no srv-ia pra filtrar `WHERE dispositivo_tipo = 'WINDOWS'` até calibração mobile | Rodada srv-ia mobile |
| Purge de `eventos_chamada_telefonica` incluído no scheduler | Já entra no MVP se `PurgeEventosService` for genérico — coordenar com @Rocket |
| Índice agregado por `(dispositivo_tipo, empresa_id)` — análise de performance depois de dado real | Rodada performance |
| Portal com timeline mostrando origem 🖥/📱 | Rodada portal (@Peter/@Miles) |
| Endpoint `GET /api/agent/events/chamadas/{colaboradorId}` (leitura de chamadas telefônicas pro relatório) | Rodada relatórios mobile |
