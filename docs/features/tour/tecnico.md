# Técnico — Tours Contextuais
> Domínio: @Tony | TL
> Última atualização: 2026-03-31

---

## Stack envolvida

- **Frontend:** Angular 16 + driver.js (já instalado) + RxJS + SCSS
- **Backend:** Spring Boot (manager-srv-portal) + PostgreSQL
- **Comunicação:** REST — PATCH para persistir flags, GET para carregar estado inicial via login ou endpoint dedicado

---

## Arquitetura do novo sistema

```
UsuarioPortal (login) ──► AuthUser (JWT payload)
        │                      │
        │                 user_tour_flags (nova tabela ou campos)
        │
        ▼
TourFlagService (Angular)
  └─► carrega flags do AuthUser ao login
  └─► PATCH /api/v1/tour/flags ao concluir cada tour

TourDefinitions (mapa estático por tourId)
  └─► array de TourStep[] por seção
  └─► sem estado — só configuração

TourService (Angular — refatorado)
  └─► iniciarTour(tourId) — executa steps do TourDefinition
  └─► deveDisparar(tourId, flags, context) — decisão de disparo
  └─► encerrarTour(tourId) — salva flag via TourFlagService

RouteListeners (nos componentes de página)
  └─► ngOnInit → TourService.verificarEDisparar(tourId, context)
```

---

## Decisões técnicas

### DT-01: Tabela separada `user_tour_flags` em vez de colunas em `UsuarioPortal`

**Decisão:** Criar tabela `user_tour_flags` com colunas booleanas por flag, FK para `usuarios_portal`.

**Motivo:** `UsuarioPortal` já tem o campo `tour_concluido` legado (booleano único). Adicionar 10+ colunas na tabela principal é invasivo. Uma tabela separada permite evolução independente das flags sem migrations no core de usuário. A relação é 1:1 (um registro por usuário).

**Schema proposto:**
```sql
CREATE TABLE user_tour_flags (
    id                                    BIGSERIAL PRIMARY KEY,
    usuario_portal_id                     BIGINT NOT NULL UNIQUE REFERENCES usuarios_portal(id),
    tour_home_concluido                   BOOLEAN NOT NULL DEFAULT false,
    tour_org_estrutura_concluido          BOOLEAN NOT NULL DEFAULT false,
    tour_org_times_concluido              BOOLEAN NOT NULL DEFAULT false,
    tour_org_hierarquia_concluido         BOOLEAN NOT NULL DEFAULT false,
    tour_colaboradores_concluido          BOOLEAN NOT NULL DEFAULT false,
    tour_colaboradores_com_dados_concluido BOOLEAN NOT NULL DEFAULT false,
    tour_colaborador_detalhe_concluido    BOOLEAN NOT NULL DEFAULT false,
    tour_relatorios_concluido             BOOLEAN NOT NULL DEFAULT false,
    tour_perfil_concluido                 BOOLEAN NOT NULL DEFAULT false,
    primeiro_login                        BOOLEAN NOT NULL DEFAULT true,
    menu_org_pulsando                     BOOLEAN NOT NULL DEFAULT false,
    atualizado_em                         TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Migration:** Criar registros na `user_tour_flags` com defaults para todos os usuários existentes. O campo `tour_concluido` legado em `UsuarioPortal` permanece intacto para compatibilidade — não é removido neste ciclo.

---

### DT-02: Flags retornadas no payload de login (sem GET extra)

**Decisão:** Incluir o objeto `tourFlags` no `LoginResponse` (dentro de `AuthUser`).

**Motivo:** Evita uma requisição adicional no bootstrap da aplicação. O frontend já tem acesso às flags ao carregar a rota inicial. Se o usuário troca de flag em outra sessão, a divergência é irrelevante (cada flag só vai de `false → true` — nunca regride exceto via "Rever tour").

**Contrato de adição em `AuthUser` (frontend):**
```typescript
tourFlags?: {
  tourHomeConcluido: boolean;
  tourOrgEstruturaConcluido: boolean;
  tourOrgTimesConcluido: boolean;
  tourOrgHierarquiaConcluido: boolean;
  tourColaboradoresConcluido: boolean;
  tourColaboradoresComDadosConcluido: boolean;
  tourColaboradorDetalheConcluido: boolean;
  tourRelatoriosConcluido: boolean;
  tourPerfilConcluido: boolean;
  primeiroLogin: boolean;
  menuOrgPulsando: boolean;
}
```

---

### DT-03: Endpoint PATCH único para atualizar qualquer flag

**Decisão:** `PATCH /api/v1/tour/flags` com body `{ flag: string, valor: boolean }`.

**Motivo:** Endpoint simples e genérico. Alternativa seria um endpoint por flag (verboso) ou PUT do objeto inteiro (sobrescreve tudo, problemático com múltiplas abas). O PATCH parcial com campo nomeado é a abordagem mais segura.

**Contrato:**
```
PATCH /api/v1/tour/flags
Authorization: Bearer {jwt}
Content-Type: application/json

{ "flag": "tour_relatorios_concluido", "valor": true }

Response 204 No Content
Response 400 — flag desconhecida
Response 401 — JWT inválido
```

**Validação no backend:** lista de flags permitidas definida em enum `TourFlag`. Qualquer campo fora da lista retorna 400.

---

### DT-04: Estrutura TourDefinitions — mapa estático no frontend

**Decisão:** Arquivo `tour.definitions.ts` exporta `TOUR_DEFINITIONS: Record<TourId, TourStep[]>`.

**Motivo:** Separa completamente a configuração (o que mostrar) da orquestração (quando mostrar e como navegar). O `TourService` nunca mais precisa conhecer os steps — apenas executa o array que recebe.

**Estrutura:**
```typescript
// tour.definitions.ts
export type TourId =
  | 'tour-home'
  | 'tour-org-estrutura'
  | 'tour-org-times'
  | 'tour-org-hierarquia'
  | 'tour-colaboradores'
  | 'tour-colaborador-detalhe'
  | 'tour-relatorios'
  | 'tour-perfil';

export const TOUR_DEFINITIONS: Record<TourId, TourStep[]> = {
  'tour-home': [ /* 4 steps */ ],
  'tour-org-estrutura': [ /* 3 steps */ ],
  // ...
};
```

---

### DT-05: Lógica de disparo por contexto — método `deveDisparar()`

**Decisão:** Cada página chama `tourService.verificarEDisparar(tourId, context)` no `ngOnInit`. O `context` carrega dados relevantes (ex: `{ temTimes: boolean, temDados: boolean }`) para que o service selecione a variação correta de steps.

**Motivo:** Evita lógica de negócio espalhada pelos componentes. O componente não sabe qual variação disparar — apenas passa o contexto. A decisão vive no TourService.

**Interface:**
```typescript
interface TourContext {
  temTimes?: boolean;
  temColaboradores?: boolean;
  temDados?: boolean;        // dados de atividade existem?
  ehAdmin?: boolean;
}
```

---

### DT-06: Animação pulse CSS — classe `menu-item--pulsando`

**Decisão:** Implementar via CSS puro com `@keyframes pulse-ring` usando `box-shadow` animado. A classe é aplicada/removida pelo `TourService` com base na flag `menu_org_pulsando`.

**Arquivo-alvo:** `nav-shell.component.scss` (ou arquivo de estilos do componente de navegação lateral).

**CSS (conforme design do @Steve):**
```scss
@keyframes pulse-ring {
  0%   { box-shadow: 0 0 0 0 rgba(99, 102, 241, 0.5); }
  70%  { box-shadow: 0 0 0 8px rgba(99, 102, 241, 0); }
  100% { box-shadow: 0 0 0 0 rgba(99, 102, 241, 0); }
}

.menu-item--pulsando {
  animation: pulse-ring 1.6s ease-out infinite;
  border-radius: 8px;
  position: relative;
}
```

**Remoção:** Ao clicar no item de menu Organização ou ao navegar para `/organizacao` por qualquer rota, o `TourService` chama `PATCH /api/v1/tour/flags` com `{ flag: "menu_org_pulsando", valor: false }` e remove a classe do DOM.

---

### DT-07: Ícone `?` no header via `@Input() tourId` em `app-header`

**Decisão:** Adicionar `@Input() tourId?: TourId` ao `AppHeaderComponent`. Quando preenchido, renderiza um botão `pi-question-circle` no canto direito do header. Ao clicar, zera a flag do tour e chama `tourService.iniciarTour(tourId)`.

**Motivo:** Centraliza a UI de "repetir tour" no componente de header já existente. Cada página passa apenas o `tourId` — não precisa importar lógica de tour.

**Arquivo-alvo:** `app-header.component.ts` / `app-header.component.html`

---

### DT-08: Seção "Tours e Onboarding" em `/configuracoes/perfil`

**Decisão:** Nova seção abaixo do card de senha na página de perfil. Cada linha lista o nome amigável do tour e um botão "Rever tour →". O clique: (1) faz PATCH para zerar a flag, (2) navega para a rota-alvo do tour, (3) o tour dispara automaticamente via `ngOnInit` do destino.

**Arquivo-alvo:** `perfil.component.html` + `perfil.component.ts`

---

## Mudanças no backend

### Novos arquivos

| Arquivo | Tipo | Responsabilidade |
|---------|------|-----------------|
| `UserTourFlags.java` | Entity JPA | Mapeamento da tabela `user_tour_flags` |
| `UserTourFlagsRepository.java` | Repository | `findByUsuarioPortalId()` |
| `TourFlag.java` | Enum | Lista de flags válidas para validação do PATCH |
| `TourFlagsDTO.java` | DTO | Payload retornado no login e no GET |
| `TourController.java` | Controller | `PATCH /api/v1/tour/flags` |
| `TourService.java` | Service | Lógica de atualização e leitura de flags |
| `V{n}__create_user_tour_flags.sql` | Migration Flyway | DDL da tabela + insert dos registros existentes |

### Alterações em arquivos existentes

| Arquivo | Alteração |
|---------|-----------|
| `UsuarioPortalService.java` | Incluir `TourFlagsDTO` no build do `AuthUser` (no método que monta o response de login) |
| `AuthResponse.java` / `LoginResponseDTO.java` | Adicionar campo `tourFlags: TourFlagsDTO` |

---

## Mudanças no frontend

### Novos arquivos

| Arquivo | Localização | Responsabilidade |
|---------|-------------|-----------------|
| `tour.definitions.ts` | `src/app/shared/tour/` | TOUR_DEFINITIONS — steps de todos os tours |
| `tour-flag.service.ts` | `src/app/shared/tour/` | PATCH de flags para o backend, cache local |
| `tour.types.ts` | `src/app/shared/tour/` | Interfaces `TourId`, `TourContext`, `TourFlagsState` |

### Arquivos refatorados

| Arquivo | Mudança |
|---------|---------|
| `tour.service.ts` | Quebrar array STEPS monolítico; adicionar `iniciarTour(id, context)`, `verificarEDisparar(id, context, flags)`, `encerrarTour(id)` |
| `auth.model.ts` | Adicionar `tourFlags?: TourFlagsState` em `AuthUser` |
| `app-header.component.ts` | Adicionar `@Input() tourId?: TourId` + botão de repetição |

### Arquivos que recebem `data-tour` novos

| Selector novo | Arquivo-alvo | Localização provável |
|---------------|-------------|---------------------|
| `data-tour="colaboradores-lista"` | `times-dashboard.page.html` | Container da lista de cards de colaboradores |
| `data-tour="detalhe-score-card"` | Componente `colaborador-detalhe` | Card de IManager Score + KPIs |
| `data-tour="detalhe-timeline"` | Componente `colaborador-detalhe` | Gráfico/timeline de atividade |
| `data-tour="detalhe-indicadores"` | Componente `colaborador-detalhe` | Card de indicadores psicológicos |
| `data-tour="detalhe-relatorio-link"` | Componente `colaborador-detalhe` | Botão/link para relatórios individuais |
| `data-tour="relatorios-filtro-time"` | `relatorios-page.component.html` | Host do `app-filtro-relatorio` |
| `data-tour="relatorios-semana-nav"` | `relatorios-page.component.html` | Host do `app-semana-navigator` |
| `data-tour="relatorios-cards-area"` | `relatorios-page.component.html` | Container dos cards de relatório |
| `data-tour="configuracoes-tours"` | `perfil.component.html` | Nova seção "Tours e Onboarding" |

### Componentes de página que recebem lógica de disparo

| Componente | Tour disparado | Contexto necessário |
|------------|---------------|---------------------|
| `DashboardComponent` (`/home`) | `tour-home` | — (dispara se `primeiroLogin == true`) |
| `HierarquiaGestaoPageComponent` (`/organizacao`) | `tour-org-estrutura`, `tour-org-times`, `tour-org-hierarquia` | tab ativa na URL |
| `TimesDashboardPage` (`/colaboradores`) | `tour-colaboradores` | `temTimes`, `temDados` |
| `ColaboradorDetalheComponent` (`/colaboradores/membro/:id`) | `tour-colaborador-detalhe` | `temDados` (semana existente) |
| `RelatoriosPageComponent` (`/relatorios`) | `tour-relatorios` | `temRelatorios` |
| `PerfilComponent` (`/configuracoes/perfil`) | `tour-perfil` | — |

---

## Tasks de implementação

> Ordenadas por dependência. Backend deve estar pronto antes das tasks de integração frontend.

---

### T1 — [Backend] Tabela `user_tour_flags` + Entity + Migration
**Responsável:** @Vision
**Dependência:** nenhuma
**Entregáveis:**
- Migration Flyway `V{n}__create_user_tour_flags.sql` com DDL + INSERT de registros existentes (todos com defaults)
- `UserTourFlags.java` Entity
- `UserTourFlagsRepository.java`
- `TourFlag.java` Enum

---

### T2 — [Backend] Endpoint PATCH + GET de flags + inclusão no login
**Responsável:** @Vision
**Dependência:** T1
**Entregáveis:**
- `TourFlagsDTO.java`
- `TourService.java` (backend)
- `TourController.java` com `PATCH /api/v1/tour/flags` (validação via `TourFlag` enum)
- Alteração em `UsuarioPortalService` para incluir `tourFlags` no response de login
- Alteração em `AuthResponse` / `LoginResponseDTO`

---

### T3 — [Frontend] Refatorar TourService — quebrar STEPS monolítico
**Responsável:** @Peter
**Dependência:** T2 (contrato de `tourFlags` no AuthUser)
**Entregáveis:**
- `tour.types.ts` — interfaces `TourId`, `TourContext`, `TourFlagsState`
- `tour.definitions.ts` — TOUR_DEFINITIONS com os 8 tours (steps migrados + novos)
- `tour-flag.service.ts` — wraper HTTP para `PATCH /api/v1/tour/flags`, cache local no AuthUser
- `tour.service.ts` refatorado — métodos `iniciarTour(id, context)`, `verificarEDisparar(id, context)`, `encerrarTour(id)`
- `auth.model.ts` atualizado com `tourFlags?: TourFlagsState`

---

### T4 — [Frontend] Lógica de disparo por rota (auto-start baseado em flags + dados)
**Responsável:** @Peter
**Dependência:** T3
**Entregáveis:**
- Método `deveDisparar(tourId, context, flags)` no `TourService`
- Integração nos `ngOnInit` de: `DashboardComponent`, `HierarquiaGestaoPageComponent`, `TimesDashboardPage`, `ColaboradorDetalheComponent`, `RelatoriosPageComponent`, `PerfilComponent`
- Listener de rota para remoção do pulse quando usuário entra em `/organizacao`

---

### T5 — [Frontend] Steps do tour-home + animação pulse no menu
**Responsável:** @Peter
**Dependência:** T4
**Entregáveis:**
- Steps do `tour-home` em `tour.definitions.ts` (4 steps)
- CSS `@keyframes pulse-ring` + `.menu-item--pulsando` no arquivo de estilos do menu
- Aplicação/remoção da classe `menu-item--pulsando` via `TourService` baseada em `menuOrgPulsando`
- Condição de saída: ao fechar step 4 → flags `tour_home_concluido = true`, `primeiro_login = false`, `menu_org_pulsando = true`

---

### T6 — [Frontend] Steps do tour-org-estrutura + tour-org-times + tour-org-hierarquia
**Responsável:** @Peter
**Dependência:** T4
**Entregáveis:**
- Steps dos 3 tours de organização em `tour.definitions.ts`
- Variações de `tour-org-times`: cenário "sem times" (2 steps) e "já há times" (1 step)
- `tour-org-hierarquia` condicional para `ehAdmin` nos steps de botões admin

---

### T7 — [Frontend] Steps do tour-colaboradores (variações por estado)
**Responsável:** @Peter ou @Miles
**Dependência:** T4
**Entregáveis:**
- Steps completos do `tour-colaboradores` em `tour.definitions.ts`
- Variação A: sem times/colaboradores (steps 1-2)
- Variação B: com times, sem dados de atividade (steps 1-3)
- Variação C: com dados reais (steps 1-5, inclui `data-tour="colaboradores-lista"`)
- Lógica de segunda passagem: `tour_colaboradores_com_dados_concluido` quando há dados

---

### T8 — [Frontend] Steps do tour-colaborador-detalhe
**Responsável:** @Peter ou @Miles
**Dependência:** T4
**Entregáveis:**
- Steps do `tour-colaborador-detalhe` em `tour.definitions.ts` (4 steps)
- Adição dos 4 `data-tour` no template de `colaborador-detalhe`: `detalhe-score-card`, `detalhe-timeline`, `detalhe-indicadores`, `detalhe-relatorio-link`
- Pré-condição no disparo: tour não inicia se colaborador não tem dados de semana

---

### T9 — [Frontend] Steps do tour-relatorios
**Responsável:** @Peter ou @Miles
**Dependência:** T4
**Entregáveis:**
- Steps do `tour-relatorios` em `tour.definitions.ts` (3-4 steps)
- Adição dos `data-tour` em `relatorios-page.component.html`: `relatorios-filtro-time`, `relatorios-semana-nav`, `relatorios-cards-area`
- Variação: step 4 (`relatorios-cards-area`) só aparece quando há relatórios disponíveis

---

### T10 — [Frontend] Steps do tour-perfil + seção "Rever tours" nas Configurações
**Responsável:** @Miles
**Dependência:** T3, T4
**Entregáveis:**
- Steps do `tour-perfil` em `tour.definitions.ts` (3 steps)
- Nova seção "Tours e Onboarding" em `perfil.component.html` com lista dos 8 tours e botão "Rever tour →"
- Adição do `data-tour="configuracoes-tours"` na nova seção
- Lógica do botão "Rever": PATCH para zerar flag + navegar para rota-alvo

---

### T11 — [Frontend] Ícone `?` no header das páginas
**Responsável:** @Miles
**Dependência:** T3
**Entregáveis:**
- `@Input() tourId?: TourId` em `app-header.component`
- Botão `pi-question-circle` renderizado quando `tourId` está preenchido
- Tooltip "Rever o tour desta seção"
- Ao clicar: PATCH para zerar flag + `tourService.iniciarTour(tourId)`
- Atualização das 6 páginas com tour para passar `tourId` ao `app-header`

---

### T12 — [Frontend] Adicionar data-tour selectors novos nos componentes existentes
**Responsável:** @Miles
**Dependência:** T7, T8, T9, T10 (coordenado — pode ser feito junto)
**Entregáveis:**
- `data-tour="colaboradores-lista"` → `times-dashboard.page.html`
- `data-tour="detalhe-score-card"` → componente colaborador-detalhe
- `data-tour="detalhe-timeline"` → componente colaborador-detalhe
- `data-tour="detalhe-indicadores"` → componente colaborador-detalhe
- `data-tour="detalhe-relatorio-link"` → componente colaborador-detalhe
- `data-tour="relatorios-filtro-time"` → `relatorios-page.component.html`
- `data-tour="relatorios-semana-nav"` → `relatorios-page.component.html`
- `data-tour="relatorios-cards-area"` → `relatorios-page.component.html`
- `data-tour="configuracoes-tours"` → `perfil.component.html`

---

## Atribuição por dev

| Dev | Tasks |
|-----|-------|
| @Vision (BE) | T1, T2 |
| @Peter (FE Sr) | T3, T4, T5, T6, T7, T8 |
| @Miles (FE Pl) | T9, T10, T11, T12 |

---

## Ordem de execução recomendada

```
T1 → T2 (backend pronto)
         │
         ▼
        T3 → T4 (service core pronto)
                  │
         ┌────────┴────────────────────┐
         ▼                             ▼
    T5 → T6 → T7 → T8              T9 → T10 → T11 → T12
    (@Peter — tours core)          (@Miles — UI + tours finais)
```
