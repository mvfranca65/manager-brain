# Feature: Organização (Hierarquia + Times + Cargos + Departamentos)
> Rota: `/organizacao`
#feature #portal
> Perfil: GESTOR (admin para algumas ações)
> Status: MVP1 funcional

## O que é

Tela de gestão organizacional. 4 abas que cobrem toda a estrutura da empresa: hierarquia visual, times, cargos e departamentos.

## O que tem na tela

### Aba 1 — Gestão & Hierarquia
- **Filtros:** dropdown de departamento + busca por nome/identificador
- **Organograma:** árvore colapsável Gestor → Time → Colaborador
- **Ações admin:** "Baixar instalador" (split button) + "Nova pessoa"
- **Modais:** criar/editar/excluir usuário, gerenciar via upload Excel

### Aba 2 — Times
- Busca + botão criar time
- Tabela paginada: Time, Departamento, Colaboradores (contagem), Ações
- Modais: criar/editar/excluir time (nome, departamento, descrição)

### Aba 3 — Cargos
- Busca + botão criar cargo
- Tabela paginada: Cargo, Status (Ativo/Inativo), Colaboradores (contagem), Ações
- Modais: criar/editar/excluir cargo

### Aba 4 — Departamentos
- Busca + botão criar departamento
- Tabela paginada: Departamento, Status, Colaboradores, Ações
- Departamento "Geral" não pode ser excluído
- Modais: criar/editar/excluir departamento

## Quem usa
- Admin do tenant, RH, CEO/COO

## Dados consumidos
- `GET /api/v2/hierarquia` — árvore organizacional
- `GET/POST/PUT/DELETE /api/v2/times-empresa` — CRUD de times
- `GET/POST/PUT/DELETE /api/v2/cargos` — CRUD de cargos
- `GET/POST/PUT/DELETE /api/v2/departamentos` — CRUD de departamentos
- `GET/POST/PUT/DELETE /api/v2/usuarios` — CRUD de pessoas
- `POST /api/v2/usuarios/importar` — upload Excel
