# Tasks: Reestruturacao de Acesso v2

> Contexto: DB ja recriado do zero. Tabela `usuarios` unificada ja existe. Sem migracao de dados.
> Ordem: Sprint 1 (Backend core) → Sprint 2 (Frontend core) → Sprint 3 (Visao colaborador + polish)
> Atualizado em: 2026-03-31

---

## Sprint 1: Backend — Perfis + Primeiro Acesso + Controle de Acesso

### Bloco A: Simplificacao de perfis (pre-requisito para tudo)

| # | Task | Responsavel | Esforco | Descricao |
|---|------|-------------|---------|-----------|
| RA-01 | Atualizar `PortalRole` enum | @Thor | P | Remover `ADMIN` e `VIEWER`. Manter `GESTOR` e `COLABORADOR`. |
| RA-02 | Atualizar `JwtTokenProvider` | @Thor | P | Claims: perfil sempre `GESTOR` ou `COLABORADOR`. Remover mapeamentos legados. |
| RA-03 | Atualizar `JwtAuthenticationFilter` | @Thor | P | Remover alias `ADMIN`/`VIEWER`/`MANAGER`. Aceitar apenas `GESTOR`/`COLABORADOR`. |
| RA-04 | Atualizar `PortalUserPrincipal` | @Thor | P | Record com perfil simplificado. |
| RA-05 | Simplificar `AccessService` | @Thor | M | `GESTOR` = acesso total (substitui antigo ADMIN+GESTOR). `COLABORADOR` = acesso limitado. Remover metodo `exigirAdmin()`. |
| RA-06 | Atualizar `entidades_hierarquicas.tipo_entidade` | @Thor | P | Valores: `GESTOR`, `COLABORADOR`, `TIME`. Remover `USUARIO_PORTAL` e `ADMIN`. |
| RA-07 | Atualizar DTOs de resposta | @Vision | M | `UsuarioPortalResponse`, `HierarquiaEntidadeResposta`: perfil = GESTOR ou COLABORADOR. |

### Bloco B: Primeiro acesso

| # | Task | Responsavel | Esforco | Descricao |
|---|------|-------------|---------|-----------|
| RA-08 | Endpoint `POST /auth/primeiro-acesso/validar` | @Vision | M | Valida identificador + email. Condicoes: usuario existe, acessa_portal=true, senha_hash IS NULL. Retorna { valido, nome }. |
| RA-09 | Endpoint `POST /auth/primeiro-acesso/criar-senha` | @Vision | M | Recebe identificador + email + novaSenha. Mesmas validacoes + BCrypt hash. Response 204. |
| RA-10 | Remover campo `senha` da criacao de usuario | @Vision | P | `CadastrarUsuarioPortalRequest`: remover campo senha. Service cria com `senha_hash = null`. |
| RA-11 | Atualizar login: diferenciar erros | @Thor | M | 3 cenarios: credenciais invalidas (401), sem acesso portal (403 + codigo `ACESSO_PORTAL_NEGADO`), sem senha/primeiro acesso (401 + codigo `PRIMEIRO_ACESSO_PENDENTE`). |

### Bloco C: Controle de acesso ao portal

| # | Task | Responsavel | Esforco | Descricao |
|---|------|-------------|---------|-----------|
| RA-12 | Endpoint `PATCH /api/v2/usuarios/{id}/acesso-portal` | @Vision | P | Toggle acessa_portal. Apenas GESTOR pode chamar. Response 204. |
| RA-13 | Adicionar `acessa_portal` na criacao de colaborador | @Vision | P | `ColaboradorCadastroRequest`: novo campo opcional (default false). `CadastrarColaboradorLoteRequest`: idem. |

---

## Sprint 2: Frontend — Perfis + Login + Primeiro Acesso + Controle

### Bloco D: Simplificacao de perfis no frontend

| # | Task | Responsavel | Esforco | Descricao |
|---|------|-------------|---------|-----------|
| RA-14 | Atualizar `AuthUser` e `auth.model.ts` | @Peter | P | `perfil: 'GESTOR' \| 'COLABORADOR'`. Remover referencias a ADMIN/VIEWER. |
| RA-15 | Atualizar `AuthService` | @Peter | P | Remover `isAdmin()`. `isGestor()` cobre tudo. Adicionar `isColaborador()`. |
| RA-16 | Remover `AdminGuard`, simplificar guards | @Peter | M | `GestorGuard` para rotas de gestao. `PortalAccessGuard` para tela de bloqueio. Novo `ColaboradorGuard` para rotas do colaborador. |
| RA-17 | Menu/sidebar dinamico por perfil | @Peter | M | Gestor: menu completo. Colaborador: Meu Painel, Meus Relatorios, Meu Perfil. |
| RA-18 | Modal criar usuario portal: remover senha e seletor perfil | @Miles | P | Remover campos senha + confirmar senha. Remover seletor Admin/Gestor. Tipo fixo = GESTOR. |
| RA-19 | Excel usuario portal: remover colunas Perfil e Senha | @Miles | P | Atualizar `ExcelUsuarioPortalService`: remover colunas G (Perfil) e J (Senha). |
| RA-20 | Badges na hierarquia: Gestor/Colaborador | @Miles | P | Substituir badges ADM/VIS por Gestor/Colaborador nos templates da hierarquia. |

### Bloco E: Login + Primeiro acesso + Tela de bloqueio

| # | Task | Responsavel | Esforco | Descricao |
|---|------|-------------|---------|-----------|
| RA-21 | Pagina `/primeiro-acesso` | @Peter | G | Nova pagina standalone. Step 1: identificador + email → validar. Step 2: criar senha + confirmar. Redirect para /login. Design alinhado com login page. |
| RA-22 | Link "Primeiro acesso?" na pagina de login | @Miles | P | Adicionar link abaixo do botao de login. Rota para /primeiro-acesso. |
| RA-23 | Pagina `/sem-acesso` (tela de bloqueio) | @Miles | M | Nova pagina standalone. Mensagem: "Voce nao tem permissao para acessar o portal. Entre em contato com seu gestor." Botao voltar para login. |
| RA-24 | Tratar 3 tipos de erro no login | @Peter | M | Interceptar resposta do backend: credenciais invalidas → mensagem generica. Sem acesso → redirect `/sem-acesso`. Primeiro acesso pendente → mensagem com link para `/primeiro-acesso`. |
| RA-25 | Rota publica para /primeiro-acesso e /sem-acesso | @Peter | P | Adicionar em `app-routing.module.ts` sem AuthGuard. |

### Bloco F: Flag acessa_portal no frontend

| # | Task | Responsavel | Esforco | Descricao |
|---|------|-------------|---------|-----------|
| RA-26 | Select "Acessa o portal?" na modal de criacao de colaborador | @Miles | P | Campo select Sim/Nao no formulario. Default: Nao. Envia `acessaPortal` no payload. |
| RA-27 | Toggle acessa_portal no detalhe do colaborador | @Miles | M | Na secao "Informacoes na Empresa", select Sim/Nao editavel. Chama PATCH ao alterar. |
| RA-28 | Toggle acessa_portal na hierarquia (no do colaborador) | @Miles | M | No panel/popover de edicao do colaborador na arvore. Select Sim/Nao. Chama PATCH ao alterar. |
| RA-29 | Excel colaborador: adicionar coluna "Acessa portal?" | @Miles | P | Nova coluna com dropdown Sim/Nao. Default vazio = Nao. Mapear no upload. |

---

## Sprint 3: Visao do Colaborador + Testes

### Bloco G: Paginas do colaborador

| # | Task | Responsavel | Esforco | Descricao |
|---|------|-------------|---------|-----------|
| RA-30 | Pagina `/meu-painel` (dashboard pessoal) | @Peter | G | Dashboard simplificado: IManager Score, status atual, KPIs pessoais da semana. Reutilizar componentes existentes filtrados para o usuario logado. |
| RA-31 | Pagina `/meus-relatorios` | @Peter | M | Lista de relatorios semanais individuais. Reutilizar `app-relatorio-individual`. Filtrar para o usuario logado. |
| RA-32 | Adaptar `/meu-perfil` para colaborador | @Miles | M | Mesma pagina de perfil. Colaborador edita: foto, telefone, email pessoal, dados complementares, senha. Nao edita: nome, sobrenome, email corp, cargo, departamento. |
| RA-33 | Rotas do colaborador em `app-routing.module.ts` | @Peter | P | /meu-painel, /meus-relatorios. Guard: ColaboradorGuard. Redirect default para /meu-painel. |

### Bloco H: Testes de regressao

| # | Task | Responsavel | Esforco | Descricao |
|---|------|-------------|---------|-----------|
| RA-34 | Testes: login + primeiro acesso + bloqueio | @Natasha | G | Cenarios: login gestor OK, login colaborador OK, login sem acesso (tela bloqueio), primeiro acesso completo, primeiro acesso com email errado, tentativa com senha ja criada. |
| RA-35 | Testes: CRUD com 2 perfis | @Natasha | G | Criar gestor, criar colaborador com/sem acesso, editar flag acessa_portal, hierarquia com gestores aninhados. |
| RA-36 | Testes: visao do colaborador | @Natasha | M | Dashboard pessoal, relatorios pessoais, perfil editavel, campos bloqueados, menu correto. |
| RA-37 | Testes: Excel sem senha e sem perfil | @Natasha | M | Upload de usuarios sem coluna senha/perfil. Upload de colaboradores com coluna acessa_portal. |

---

## Resumo

| Sprint | Bloco | Tasks | Responsaveis |
|--------|-------|-------|-------------|
| Sprint 1 (Backend) | A: Perfis | RA-01 a RA-07 (7) | @Thor + @Vision |
| Sprint 1 (Backend) | B: Primeiro acesso | RA-08 a RA-11 (4) | @Thor + @Vision |
| Sprint 1 (Backend) | C: Controle acesso | RA-12 a RA-13 (2) | @Vision |
| Sprint 2 (Frontend) | D: Perfis | RA-14 a RA-20 (7) | @Peter + @Miles |
| Sprint 2 (Frontend) | E: Login/Primeiro acesso | RA-21 a RA-25 (5) | @Peter + @Miles |
| Sprint 2 (Frontend) | F: Flag acessa_portal | RA-26 a RA-29 (4) | @Miles |
| Sprint 3 (Colaborador) | G: Paginas | RA-30 a RA-33 (4) | @Peter + @Miles |
| Sprint 3 (Colaborador) | H: Testes | RA-34 a RA-37 (4) | @Natasha |
| **Total** | | **37 tasks** | **3 sprints** |

## Atribuicao por dev

| Dev | Tasks | Total |
|-----|-------|-------|
| @Thor | RA-01 a RA-06, RA-11 | 7 |
| @Vision | RA-07 a RA-10, RA-12, RA-13 | 6 |
| @Peter | RA-14 a RA-17, RA-21, RA-24, RA-25, RA-30, RA-31, RA-33 | 10 |
| @Miles | RA-18 a RA-20, RA-22, RA-23, RA-26 a RA-29, RA-32 | 10 |
| @Natasha | RA-34 a RA-37 | 4 |

## Esforcos

- P = Pequeno (1-2h)
- M = Medio (3-5h)
- G = Grande (6-10h)
