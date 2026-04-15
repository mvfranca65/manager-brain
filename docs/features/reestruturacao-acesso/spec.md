# Spec: Reestruturacao de Acesso v2

> ADR: `adr/ADR-002-unificacao-usuarios.md`
> Tasks: `tasks.md`
> Atualizado em: 2026-03-31

---

## Contexto

A unificacao de `usuarios_portal` + `colaboradores` na tabela `usuarios` ja foi concluida.
O banco foi recriado do zero — nao ha dados legados para migrar.
Esta spec cobre as mudancas restantes: simplificacao de perfis, primeiro acesso e controle de acesso ao portal.

---

## 1. Modelo de perfis (2 perfis)

### Antes: 3 perfis (ADMIN, GESTOR/VIEWER, COLABORADOR)
### Agora: 2 perfis (GESTOR, COLABORADOR)

```
GESTOR — acesso total ao sistema
COLABORADOR — acesso ao perfil proprio + seus relatorios (quando autorizado)
```

### Hierarquia

```
Gestor (CEO/Dono)
  └── Gestor (Diretor)
        └── Gestor (Coordenador)
              └── Time "Desenvolvimento"
                    ├── Colaborador A
                    ├── Colaborador B
                    └── Colaborador C
```

- Gestor pode ter outros gestores abaixo
- Antes dos colaboradores, precisa existir um time (padrao atual mantido)
- O escopo de visibilidade e definido pela hierarquia, nao pelo perfil

### Mudancas no backend

- `PortalRole` enum: remover `ADMIN` e `VIEWER`, manter `GESTOR` e `COLABORADOR`
- `AccessService`: simplificar — `GESTOR` tem acesso total, `COLABORADOR` acesso limitado
- JWT claim `perfil`: sempre `GESTOR` ou `COLABORADOR`
- `entidades_hierarquicas.tipo_entidade`: valores `GESTOR`, `COLABORADOR`, `TIME`

### Mudancas no frontend

- `AuthUser.perfil`: apenas `'GESTOR' | 'COLABORADOR'`
- Remover `AdminGuard` — usar apenas `GestorGuard`
- Remover seletor Admin/Gestor da modal de criacao de usuario portal
- Remover coluna "Perfil" do Excel de importacao de usuarios
- Badges na hierarquia: apenas "Gestor" / "Colaborador"

---

## 2. Controle de acesso ao portal (flag `acessa_portal`)

### Regras

- Campo `acessa_portal` na tabela `usuarios` (boolean, NOT NULL)
- **Gestor:** default `true` ao criar
- **Colaborador:** default `false` ao criar
- Na criacao: campo select "Acessa o portal?" com opcoes Sim/Nao
- Na edicao: editavel tanto na hierarquia quanto no detalhe do colaborador
- Somente gestores podem alterar essa flag

### Tela de bloqueio

Quando um usuario tenta fazer login e `acessa_portal = false`:
- Backend retorna erro diferenciado (ex: HTTP 403 com codigo `ACESSO_PORTAL_NEGADO`)
- Frontend exibe pagina `/sem-acesso` com mensagem:
  - "Voce nao tem permissao para acessar o portal"
  - "Entre em contato com seu gestor para solicitar acesso"
- Nao redireciona para login — e uma pagina propria

### Endpoint de toggle

```
PATCH /api/v2/usuarios/{id}/acesso-portal
Authorization: Bearer {jwt} (perfil GESTOR)
Content-Type: application/json

{ "acessaPortal": true }

Response 204 No Content
Response 403 — perfil nao e GESTOR
Response 404 — usuario nao encontrado
```

---

## 3. Primeiro acesso (sem senha temporaria)

### Fluxo

```
Gestor cria usuario → SEM senha (senha_hash = null)
         ↓
Usuario abre o portal → tela de login
         ↓
Clica "Primeiro acesso?" → /primeiro-acesso
         ↓
Step 1: Informa identificador + email → backend valida
         ↓
Step 2: Cria senha (minimo 8 chars) + confirma
         ↓
Senha salva → redireciona para /login com mensagem de sucesso
```

### Endpoints

**Validar primeiro acesso:**
```
POST /auth/primeiro-acesso/validar
Content-Type: application/json

{ "identificador": "I123456", "email": "joao@empresa.com" }

Response 200 — { "valido": true, "nome": "Joao" }
Response 400 — usuario nao encontrado, ja tem senha, ou acessa_portal=false
```

Condicoes para sucesso:
- Usuario existe com esse identificador + email
- `acessa_portal = true`
- `senha_hash IS NULL` (nunca fez primeiro acesso)

**Criar senha:**
```
POST /auth/primeiro-acesso/criar-senha
Content-Type: application/json

{ "identificador": "I123456", "email": "joao@empresa.com", "senha": "MinhaS3nha!" }

Response 204 No Content
Response 400 — validacao falhou (mesmas condicoes acima)
```

### Mudancas na criacao de usuarios

- Remover campo `senha` do `CadastrarUsuarioPortalRequest`
- Remover campos de senha da modal de criacao (frontend)
- Remover coluna "Senha" do Excel de importacao
- `senha_hash` salvo como `null` na criacao

### Mudancas no login

O login precisa diferenciar 3 cenarios de erro:
1. **Credenciais invalidas** — identificador nao existe ou senha errada → mensagem generica
2. **Sem acesso ao portal** — `acessa_portal = false` → redireciona para `/sem-acesso`
3. **Primeiro acesso pendente** — `senha_hash IS NULL` → mensagem "Use o link Primeiro Acesso"

### Pagina /primeiro-acesso (frontend)

- Rota publica (sem AuthGuard)
- Step 1: formulario com identificador + email + botao "Validar"
- Step 2 (apos validacao): campos nova senha + confirmar senha + botao "Criar senha"
- Ao concluir: toast de sucesso + redirect para /login
- Design: mesma linguagem visual da pagina de login

---

## 4. Matriz de acesso por perfil

| Recurso | GESTOR | COLABORADOR (com acesso) |
|---------|--------|--------------------------|
| Dashboard | Sim (sua cadeia) | Apenas o proprio |
| Colaboradores (lista) | Sua cadeia | Nao |
| Detalhe colaborador | Sua cadeia | Apenas o proprio |
| Relatorios (time) | Seus times | Nao |
| Relatorio individual | Sua cadeia | Apenas o proprio |
| Organizacao | Sim | Nao |
| Criar/editar usuarios | Sim | Nao |
| Perfil pessoal | Sim | Sim |
| Alterar senha | Sim | Sim |

### Menu por perfil

**Gestor:**
- Home (dashboard)
- Colaboradores
- Relatorios
- Organizacao
- Configuracoes / Perfil

**Colaborador (com acesso):**
- Meu Painel (dashboard pessoal)
- Meus Relatorios
- Meu Perfil

---

## 5. Frontend — visao do colaborador

### Meu Painel

Dashboard simplificado mostrando:
- IManager Score da semana
- Status atual (online/offline/ausente)
- KPIs pessoais resumidos

### Meus Relatorios

Lista dos relatorios semanais individuais do colaborador.
Mesmo componente `app-relatorio-individual` ja existente, apenas filtrado para o usuario logado.

### Meu Perfil

Mesmo componente de perfil existente. Colaborador pode editar:
- Foto
- Telefone
- Email pessoal
- Dados complementares (estado civil, genero, data nascimento)
- Senha

Nao pode editar: nome, sobrenome, email corporativo, cargo, departamento.

---

## 6. APIs impactadas

### manager-srv-portal

| Endpoint | Mudanca |
|---------|---------|
| POST /auth/login | Aceitar GESTOR e COLABORADOR. Diferenciar erros (sem acesso, sem senha, credenciais). |
| POST /auth/primeiro-acesso/validar | **Novo** |
| POST /auth/primeiro-acesso/criar-senha | **Novo** |
| PATCH /api/v2/usuarios/{id}/acesso-portal | **Novo** |
| POST /api/gerente | Remover campo senha. Perfil fixo = GESTOR |
| POST /api/colaborador | Adicionar campo acessa_portal (default false) |
| POST /api/colaborador/lote | Idem |
| GET /me | Funciona para ambos os perfis |
| AccessService | Simplificar: GESTOR = acesso total, COLABORADOR = acesso limitado |
| JwtTokenProvider | Claims: perfil = GESTOR ou COLABORADOR |

### manager-fed-portal

| Area | Mudanca |
|------|---------|
| Login page | Link "Primeiro acesso?". Tratar 3 tipos de erro. |
| /primeiro-acesso | **Nova pagina** (2 steps) |
| /sem-acesso | **Nova pagina** (tela de bloqueio) |
| Auth service | Remover isAdmin(). Simplificar para isGestor() / isColaborador(). |
| Guards | Remover AdminGuard. Criar ColaboradorGuard. |
| Menu/sidebar | Dinamico por perfil. |
| Modal criar usuario portal | Remover campos de senha e seletor de perfil. |
| Modal criar colaborador | Adicionar select "Acessa o portal?" (default Nao). |
| Excel usuario portal | Remover coluna Perfil e coluna Senha. |
| Excel colaborador | Adicionar coluna "Acessa portal?" (Sim/Nao). |
| Detalhe colaborador | Adicionar toggle/select "Acessa o portal?". |
| Hierarquia | Editar flag acessa_portal no no do colaborador. Badges: Gestor/Colaborador. |
| Rotas colaborador | /meu-painel, /meus-relatorios, /meu-perfil |
