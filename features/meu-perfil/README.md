# Feature: Meu Perfil (Configurações)
> Rota: `/configuracoes/perfil`
#feature #portal
> Perfil: GESTOR + COLABORADOR
> Status: MVP1 funcional

## O que é

Tela de configurações pessoais do usuário. Edição de dados, troca de senha e gestão de tours.

## O que tem na tela

### Card 1 — Perfil
**Hero banner:** Avatar (foto ou iniciais) + nome + cargo + botão "Editar perfil" (abre modal)

**Dados Pessoais:**
- Nome completo, telefone, data de nascimento, estado civil, gênero, e-mail pessoal

**Dados na Empresa** (somente leitura para COLABORADOR):
- Identificador único, e-mail corporativo, data de contratação, cargo
- Departamento(s), perfil de acesso (badge), gestor responsável
- Times gerenciados (se GESTOR)

### Card 2 — Segurança
- Recomendações de senha (mín 8 chars, maiúscula, minúscula, número, especial)
- Formulário: senha atual → nova senha → confirmar
- Toggle mostrar/ocultar senha em cada campo

### Card 3 — Tours e Onboarding (apenas GESTOR)
- Lista de tours contextuais com status (Concluído / Pendente)
- Botão "Rever" para repetir qualquer tour

## Dados consumidos
- `GET /api/v2/usuarios/me` — dados do perfil
- `PUT /api/v2/usuarios/me` — atualizar perfil
- `PUT /api/v2/usuarios/me/senha` — trocar senha
- `GET /api/v2/usuarios/me/tours` — status dos tours
