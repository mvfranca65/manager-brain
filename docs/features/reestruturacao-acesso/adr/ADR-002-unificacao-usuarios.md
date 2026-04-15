# ADR-002: Unificacao de Usuarios e Colaboradores

> Data: 2026-03-27
> Status: Aprovado (planejamento)
> Decisores: @Steve (PO), @Tony (TL), Product Owner

## Contexto

O sistema possui duas entidades separadas para representar pessoas:
- `usuarios_portal` — gestores/admins que acessam o portal (nao monitorados)
- `colaboradores` — pessoas monitoradas pelo agent (nao acessam o portal)

Essa separacao gera: duplicacao de logica em todo o stack, complexidade na hierarquia (EntidadeHierarquica com 2 FKs), e impossibilita cenarios reais onde gestores devem ser monitorados ou colaboradores devem ver seus proprios dados.

## Decisao

Unificar ambas as tabelas em uma unica tabela `usuarios` com flags de controle.

### Novo modelo

```sql
CREATE TABLE usuarios (
    id                  BIGSERIAL PRIMARY KEY,
    empresa_id          BIGINT NOT NULL REFERENCES empresas(id),
    cnpj_empresa        VARCHAR(20) NOT NULL,
    identificador       VARCHAR(100) NOT NULL,
    nome                VARCHAR(100) NOT NULL,
    sobrenome           VARCHAR(120),
    email               VARCHAR(200),
    email_pessoal       VARCHAR(200),
    telefone            VARCHAR(30),
    senha_hash          VARCHAR(300),
    perfil              VARCHAR(20) NOT NULL, -- ADMIN, GESTOR, COLABORADOR
    acessa_portal       BOOLEAN NOT NULL DEFAULT false,
    monitorado          BOOLEAN NOT NULL DEFAULT false,
    ativo               BOOLEAN NOT NULL DEFAULT true,
    -- campos pessoais
    cargo               VARCHAR(120),
    genero              VARCHAR(20),
    estado_civil        VARCHAR(20),
    data_nascimento     DATE,
    data_contratacao    DATE,
    jornada_horas       INTEGER DEFAULT 8,
    -- avatar
    avatar_bytes        BYTEA,
    avatar_atualizado_em TIMESTAMPTZ,
    url_avatar          VARCHAR(500),
    -- agent
    ultima_maquina_id   VARCHAR(100),
    ultima_descricao_so VARCHAR(200),
    ultimo_ip           VARCHAR(50),
    tipo_identificador  VARCHAR(20),
    -- portal
    tour_concluido      BOOLEAN DEFAULT false,
    -- audit
    criado_em           TIMESTAMPTZ DEFAULT NOW(),
    atualizado_em       TIMESTAMPTZ,

    CONSTRAINT uk_usuario_empresa_identificador UNIQUE (empresa_id, identificador),
    CONSTRAINT uk_usuario_cnpj_email UNIQUE (cnpj_empresa, email)
);
```

### Perfis e flags

| Perfil | acessa_portal | monitorado | Descricao |
|--------|--------------|------------|-----------|
| ADMIN | true (sempre) | Opcional | Ve tudo. Pode ser monitorado se tiver agent. |
| GESTOR | Opcional | Opcional | Ve sua cadeia hierarquica. Admin controla o acesso. |
| COLABORADOR | Opcional | true (sempre) | Ve apenas seus proprios dados (read-only). Admin/Gestor controla acesso. |

### Regras de acesso ao portal

1. `acessa_portal = true` → usuario tem senha e pode fazer login
2. `acessa_portal = false` → sem senha, sem login, apenas entidade gerenciada
3. Se um GESTOR nao tem acesso, ninguem abaixo dele pode ter
4. COLABORADOR com acesso ve: dashboard pessoal, sua timeline, seus relatorios, seu perfil
5. COLABORADOR com acesso NAO ve: outros colaboradores, times, organizacao, dados de terceiros
6. ADMIN sempre tem acesso ao portal

### Simplificacao da hierarquia

```sql
-- ANTES: entidades_hierarquicas tinha 2 FKs opcionais
entidades_hierarquicas (usuario_portal_id, colaborador_id, time_id)

-- DEPOIS: 1 FK para usuario + 1 FK para time
entidades_hierarquicas (usuario_id, time_id)
```

Os `TipoRelacionamento` mudam:
- ADMIN_ADMIN, ADMIN_GESTOR, ADMIN_COLABORADOR, GESTOR_COLABORADOR, ADMIN_TIME, GESTOR_TIME, TIME_COLABORADOR

## Consequencias

### O que muda (resumo por area)

| Area | Impacto |
|------|---------|
| Banco de dados | Nova tabela `usuarios`, migracao de dados, drop das antigas |
| Entidades JPA | 1 entity `Usuario` em vez de 2 (em 4 projetos backend) |
| Repositorios | Unificar ColaboradorRepository + UsuarioPortalRepository |
| Hierarquia | EntidadeHierarquica com 1 FK (usuario_id) em vez de 2 |
| Autenticacao | Login por identificador — qualquer usuario com acessa_portal=true |
| JWT | Claims: id, cnpj, perfil, acessa_portal, monitorado |
| Agent | Sem mudanca — continua usando identificador para vincular |
| Frontend | Telas de CRUD unificadas, visao pessoal para colaborador |
| Permissoes | Middleware de permissao por perfil + flags em cada endpoint |

### Riscos

1. Migracao de dados — IDs mudam, FKs precisam ser atualizadas
2. 4 projetos backend compartilham o mesmo banco — todos precisam ser atualizados juntos
3. Downtime durante migracao (ou estrategia de migracao em fases)
4. Testes regressivos extensivos necessarios

### O que NAO muda

- Tabelas de eventos (apenas FK de colaborador_id vira usuario_id)
- Logica do agent desktop (continua usando identificador)
- Fluxo de coleta de eventos
- API de atualizacao do agent
