# Arquitetura — Manager
> Domínio: @Tony | TL  
#tech #arquitetura
> Leitura obrigatória para: @Tony, @Thor, @Vision, @Peter

---

## Visão geral dos serviços

```
┌─────────────────────────────────────────────────────┐
│                    MANAGER PLATFORM                  │
│                                                      │
│  [manager-srv-agent]     Desktop App (Win/macOS)     │
│         │ batch events                               │
│         ▼                                            │
│  [manager-srv-events]    Backend Ingestão            │
│         │ persiste                                   │
│         ▼                                            │
│  [PostgreSQL]            Banco multitenancy          │
│         ▲                                            │
│  [manager-srv-portal]    Backend Portal (Spring Boot)│
│         │ REST + JWT                                 │
│         ▼                                            │
│  [manager-fed-portal]    Frontend Angular 16         │
│                                                      │
│  [manager-srv-admin]     Gestão de tenants/API keys  │
└─────────────────────────────────────────────────────┘
```

---

## Serviços detalhados

### manager-srv-agent (Agent Desktop)
- Windows + macOS (suporte parcial)
- Captura: janela ativa (processo + título), idle time, eventos de sessão
- **Não captura:** teclado, áudio, vídeo, screenshots (exceto plano Plus)
- Envia eventos em **batch** para o backend de ingestão
- Valida API key na primeira execução (verifica status ACTIVE)

### manager-srv-events (Backend Ingestão)
- Recebe eventos do agent
- Persiste no PostgreSQL vinculado ao tenant correto

### manager-srv-admin (Backend Administrativo)
- Gestão de empresas (tenants): cadastro, validação
- Geração de chave de ativação (API key por empresa)
- O instalador do agent embute essa chave

### manager-srv-portal (Backend Portal)
- **Spring Boot (Java)** — REST API com JWT
- Regras de negócio: cálculo de scores, pilares, indicadores psicológicos
- Agendamentos via scheduler (geração de relatórios)
- Integração com LLM API
- Envio de emails

### manager-fed-portal (Frontend)
- **Angular 16 + PrimeNG + SCSS + RxJS**
- SPA com lazy loading, guards de autenticação e perfil
- Consome APIs do manager-srv-portal

---

## Banco de dados

- **PostgreSQL** — multitenancy por `tenant_id`
- Nenhum cliente vê dados de outro

---

## Deploy

- **Docker** + **Fly.io**
- Multi-stage build: Node builder + Nginx

---

## Design System (Frontend)

```scss
// Cards
border: 1px solid #e5e7eb;
border-radius: 14px;
background: white;

// Headers de card
color: #1e293b; // slate escuro
// com gradiente sutil

// Ações principais
// paleta azul-índigo

// Tipografia: limpa, B2B SaaS enterprise
```

---

## ADRs

→ Ver decisões específicas por feature em `.brain/features/*/adr/`
