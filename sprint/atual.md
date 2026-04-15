# Sprint Atual
> Dominio: @Steve | PO + @Estranho | SM
> Atualizar a cada planning e daily

---

## Sprint: Refatoracao CRUD de Colaboradores
**Periodo:** 2026-04-02
**Objetivo:** Criar colaboradores direto na tabela `usuarios`, transacao unica, hierarquia funcional
**Status:** ✅ Implementação concluída — testes manuais restantes (RC-15, RC-16, RC-17)
**Spec:** `.brain/features/reestruturacao-acesso/spec-refatoracao-colaborador.md`
**Tasks:** `.brain/features/reestruturacao-acesso/tasks-refatoracao-colaborador.md`

### Bloco 1 — CRUD Backend (@Thor)
| # | Task | Status |
|---|------|--------|
| RC-01 | Refatorar POST /api/colaborador (criar) | ✅ |
| RC-02 | Refatorar POST /api/colaborador/lote | ✅ |
| RC-03 | Refatorar PUT /api/colaborador/{id} (editar) | ✅ |
| RC-04 | Refatorar DELETE /api/colaborador/{id} | ✅ |
| RC-05 | Refatorar GET /api/hierarquia | ✅ |
| RC-06 | Verificar PUT /api/hierarquia/mover e POST relacionamento | ✅ |

### Bloco 2 — Vinculos e limpeza (@Vision)
| # | Task | Status |
|---|------|--------|
| RC-07 | Migrar vinculo_colaborador_time para usuario_id | ✅ |
| RC-08 | Limpar ColaboradorService de paths legados | ✅ |
| RC-09 | Atualizar ColaboradoresV1Service | ✅ |

### Bloco 3 — Frontend (@Peter + @Miles)
| # | Task | Status |
|---|------|--------|
| RC-10 | Adaptar modal criar colaborador | ✅ (sem mudanças necessárias) |
| RC-11 | Adaptar modal editar colaborador (detalhe) | ✅ (sem mudanças necessárias) |
| RC-12 | Adaptar modal editar colaborador (hierarquia) | ✅ (sem mudanças necessárias) |
| RC-13 | Adaptar tela de hierarquia | ✅ (sem mudanças necessárias) |

### Bloco 4 — Testes (@Natasha)
| # | Task | Status |
|---|------|--------|
| RC-14 | Testar criar colaborador + hierarquia | ✅ Validado: Gestor > Time > Colaborador |
| RC-15 | Testar editar colaborador (mudar gestor/time) | ⏳ Pendente teste manual |
| RC-16 | Testar excluir colaborador | ⏳ Pendente teste manual |
| RC-17 | Testar criar em lote (Excel) | ⏳ Pendente teste manual |
| RC-18 | Testar hierarquia completa | ✅ Validado via browser |

---

## Sprints anteriores

### Sprint: Reestruturacao de Acesso — Sprint 1 (Backend)
**Periodo:** 2026-04-01 a 2026-04-07
**Objetivo:** Simplificar perfis para 2 (Gestor/Colaborador), implementar primeiro acesso e controle de acesso ao portal no backend
**Status:** ✅ CONCLUIDA
**Feature:** `.brain/features/reestruturacao-acesso/`
**Spec:** `.brain/features/reestruturacao-acesso/spec.md`
**Tasks completas:** `.brain/features/reestruturacao-acesso/tasks.md`

---

## Sprint anterior

> Sprint Relatorios Semanais por IA — Conclusao
> Periodo: 2026-03-25 a 2026-03-31
> Resultado: 33 tasks finalizadas | 130 testes | 3 builds verdes
> Retrospectiva: `.brain/registro/sessoes/2026-03-25-sprint-relatorios-colaboradores.md`

---

## Tasks da Sprint 1

### Bloco A — Simplificacao de perfis (@Thor)

| # | Task | Responsavel | Esforco | Status |
|---|------|-------------|---------|--------|
| RA-01 | Atualizar `PortalRole` enum (remover ADMIN/VIEWER) | @Thor | P | ✅ |
| RA-02 | Atualizar `JwtTokenProvider` (claims simplificados) | @Thor | P | ✅ |
| RA-03 | Atualizar `JwtAuthenticationFilter` (remover alias) | @Thor | P | ✅ |
| RA-04 | Atualizar `PortalUserPrincipal` | @Thor | P | ✅ |
| RA-05 | Simplificar `AccessService` (GESTOR=total, COLAB=limitado) | @Thor | M | ✅ |
| RA-06 | Atualizar `entidades_hierarquicas.tipo_entidade` | @Thor | P | ✅ |
| RA-07 | Atualizar DTOs de resposta | @Vision | M | ✅ |

### Bloco B — Primeiro acesso (@Vision)

| # | Task | Responsavel | Esforco | Status |
|---|------|-------------|---------|--------|
| RA-08 | Endpoint POST /auth/primeiro-acesso/validar | @Vision | M | ✅ |
| RA-09 | Endpoint POST /auth/primeiro-acesso/criar-senha | @Vision | M | ✅ |
| RA-10 | Remover campo senha da criacao de usuario | @Vision | P | ✅ |
| RA-11 | Login: diferenciar 3 tipos de erro | @Thor | M | ✅ |

### Bloco C — Controle de acesso (@Vision)

| # | Task | Responsavel | Esforco | Status |
|---|------|-------------|---------|--------|
| RA-12 | Endpoint PATCH /api/v2/usuarios/{id}/acesso-portal | @Vision | P | ✅ |
| RA-13 | Adicionar acessa_portal na criacao de colaborador | @Vision | P | ✅ |

---

## Caminho critico

```
RA-01 → RA-02 → RA-03 → RA-04 → RA-05 → RA-06 (perfis simplificados)
                                        ↓
                                   RA-11 (login diferenciado)
                                        ↓
RA-08 → RA-09 → RA-10 (primeiro acesso — pode paralelizar com Bloco A)
RA-12 → RA-13 (controle acesso — independente)
```

## Atribuicao

| Dev | Tasks | Estimativa |
|-----|-------|-----------|
| @Thor | RA-01 a RA-06, RA-11 | ~15h |
| @Vision | RA-07 a RA-10, RA-12, RA-13 | ~12h |

---

---

## Sprint: Reestruturacao de Acesso — Sprint 2 (Frontend)
**Periodo:** 2026-03-31
**Objetivo:** Simplificar perfis no frontend, implementar primeiro acesso, tela de bloqueio e flag acessa_portal
**Status:** ✅ CONCLUIDA

### Bloco D — Perfis no frontend
| # | Task | Responsavel | Status |
|---|------|-------------|--------|
| RA-14 | Atualizar AuthUser e auth.model.ts | @Peter | ✅ |
| RA-15 | Atualizar AuthService | @Peter | ✅ |
| RA-16 | Remover AdminGuard, simplificar guards | @Peter | ✅ |
| RA-17 | Menu/sidebar dinamico por perfil | @Peter | ✅ |
| RA-18 | Modal criar usuario: remover senha e seletor perfil | @Miles | ✅ |
| RA-19 | Excel usuario portal: remover colunas Perfil e Senha | @Miles | ✅ |
| RA-20 | Badges na hierarquia: Gestor/Colaborador | @Miles | ✅ |

### Bloco E — Login + Primeiro acesso + Bloqueio
| # | Task | Responsavel | Status |
|---|------|-------------|--------|
| RA-21 | Pagina /primeiro-acesso | @Peter | ✅ |
| RA-22 | Link "Primeiro acesso?" no login | @Miles | ✅ |
| RA-23 | Pagina /sem-acesso (tela de bloqueio) | @Miles | ✅ |
| RA-24 | Tratar 3 tipos de erro no login | @Peter | ✅ |
| RA-25 | Rotas publicas /primeiro-acesso e /sem-acesso | @Peter | ✅ |

### Bloco F — Flag acessa_portal
| # | Task | Responsavel | Status |
|---|------|-------------|--------|
| RA-26 | Select "Acessa o portal?" na criacao de colaborador | @Miles | ✅ |
| RA-27 | Toggle acessa_portal no detalhe do colaborador | @Miles | ✅ |
| RA-28 | Toggle acessa_portal na hierarquia | @Miles | ✅ |
| RA-29 | Excel colaborador: coluna "Acessa portal?" | @Miles | ✅ |

---

## Sprint: Reestruturacao de Acesso — Sprint 3 (Visao colaborador + Testes)
**Periodo:** 2026-03-31
**Objetivo:** Paginas do colaborador e testes de regressao
**Status:** ✅ CONCLUIDA (Bloco G) / ⏳ Bloco H (testes)

### Bloco G — Paginas do colaborador
| # | Task | Responsavel | Status |
|---|------|-------------|--------|
| RA-30 | Pagina /meu-painel (dashboard pessoal) | @Peter | ✅ |
| RA-31 | Pagina /meus-relatorios | @Peter | ✅ |
| RA-32 | Adaptar perfil para colaborador | @Miles | ✅ |
| RA-33 | Rotas do colaborador em app-routing | @Peter | ✅ |

### Bloco H — Testes de regressao
| # | Task | Responsavel | Status |
|---|------|-------------|--------|
| RA-34 | Testes: login + primeiro acesso + bloqueio (11 cenarios) | @Natasha | ⏳ |
| RA-35 | Testes: CRUD com 2 perfis (7 cenarios) | @Natasha | ⏳ |
| RA-36 | Testes: visao do colaborador (7 cenarios) | @Natasha | ⏳ |
| RA-37 | Testes: Excel sem senha e sem perfil (5 cenarios) | @Natasha | ⏳ |

**Roteiro completo:** `.brain/features/reestruturacao-acesso/testes.md` — 30 cenarios + checklist final
