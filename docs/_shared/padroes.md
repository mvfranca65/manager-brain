# Padrões — Manager
> Domínio: @Tony | TL  
#tech #padroes
> Leitura obrigatória para: todos os devs (@Peter, @Miles, @Thor, @Vision)

---

## Convenções gerais

- Idioma do código: **inglês**
- Idioma de documentação e comentários: **português**
- Commits: Conventional Commits (`feat:`, `fix:`, `docs:`, `refactor:`)

---

## Frontend (Angular 16)

- Componentes em **módulos lazy-loaded**
- Guards de autenticação e perfil obrigatórios em rotas protegidas
- Estilização via **SCSS** — sem inline styles
- Componentes de UI via **PrimeNG** — não recriar o que já existe na lib
- Estado reativo via **RxJS** — evitar subscribe aninhado, preferir operadores

### Design System obrigatório
```scss
// Nunca hardcodar cores fora das variáveis
$border-color: #e5e7eb;
$border-radius: 14px;
$header-color: #1e293b;
$primary: azul-índigo (ver tokens do design system)
```

---

## Backend (Spring Boot)

- Arquitetura em camadas: Controller → Service → Repository
- JWT para autenticação — nunca expor claims sensíveis
- Todo endpoint deve validar `tenant_id` — **multitenancy é inegociável**
- Queries complexas: preferir JPQL/Criteria API, evitar SQL nativo sem justificativa
- Agendamentos via `@Scheduled` — documentar horário e impacto

---

## Integração LLM

- **A IA nunca calcula scores** — só interpreta valores já calculados pelo backend
- Prompts em arquivos separados — nunca inline no código
- Outputs da LLM sempre tipados via schema JSON definido
- Chamadas paralelas por colaborador — nunca sequenciais

---

## Qualidade (@Natasha)

- Definition of Done: código revisado + testes + sem regressão + documentado
- Toda feature nova: casos de teste documentados antes do desenvolvimento
- Bugs críticos bloqueiam deploy imediatamente

---

## Processo

- Decisões técnicas relevantes → criar ADR na pasta da feature
- Impedimentos → registrar em `.brain/registro/impedimentos.md`
- Outputs de sessões importantes → registrar em `.brain/registro/sessoes/`
