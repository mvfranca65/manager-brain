# Feature: Reestruturacao de Acesso v2
> Status: Em andamento
> Sprint: Dedicado
> Decisores: @Steve (PO), @Tony (TL)
> Atualizado em: 2026-04-02

---

## Objetivo

Simplificar o sistema de permissoes para 2 perfis (Gestor e Colaborador), implementar fluxo de primeiro acesso sem senha temporaria, e dar ao colaborador acesso limitado ao portal (perfil + relatorios).

## Status de implementacao

| Fase | Status |
|------|--------|
| Simplificacao de perfis (2 perfis) | ✅ Concluida |
| Primeiro acesso (sem senha temporaria) | ✅ Concluida |
| Flag acessa_portal | ✅ Concluida |
| Tela de bloqueio /sem-acesso | ✅ Concluida |
| Visao do colaborador (meu-painel, meus-relatorios) | ✅ Concluida |
| Eliminacao tabela usuarios_portal | ✅ Concluida |
| Alinhamento hierarquia (GESTOR/COLABORADOR) | ✅ Concluida |
| **Refatoracao CRUD de colaboradores** | 🔧 **Em andamento** |

## Decisoes-chave

1. **2 perfis apenas:** Gestor (acesso total) + Colaborador (acesso limitado)
2. **Sem senha temporaria:** Primeiro acesso via link no login (identificador + email → criar senha)
3. **Flag `acessa_portal`:** Select Sim/Nao na criacao, editavel na hierarquia e no detalhe
4. **Default sem acesso:** Colaboradores criados com `acessa_portal = false`
5. **Tela de bloqueio:** Pagina `/sem-acesso` para usuarios sem permissao
6. **Colaborador ve:** Perfil proprio + seus relatorios semanais
7. **DB limpo:** Tabela `usuarios_portal` eliminada. `usuarios` e a unica tabela de usuarios.
8. **CRUD unificado:** Colaboradores criados direto na tabela `usuarios` (decisao 2026-04-02)

## Documentacao

| Documento | Descricao |
|-----------|-----------|
| `spec.md` | Spec original (perfis, primeiro acesso, flag) |
| `spec-refatoracao-colaborador.md` | **Spec da refatoracao do CRUD de colaboradores** |
| `tasks.md` | Tasks originais (37 — concluidas) |
| `tasks-refatoracao-colaborador.md` | **Tasks da refatoracao (18 tasks)** |
| `testes.md` | Roteiro de testes (30 cenarios) |
| `resultados-testes.md` | Resultados da @Natasha |
| `adr/ADR-002-unificacao-usuarios.md` | ADR original |
| `migracao.md` | OBSOLETO |
