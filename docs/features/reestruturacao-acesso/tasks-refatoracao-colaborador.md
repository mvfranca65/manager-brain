# Tasks: Refatoracao do CRUD de Colaboradores

> Spec: `spec-refatoracao-colaborador.md`
> Sprint: Dedicada
> Atualizado em: 2026-04-02

---

## Bloco 1: Backend — CRUD unificado (@Thor)

| # | Task | Esforco | Descricao |
|---|------|---------|-----------|
| RC-01 | Refatorar POST /api/colaborador (criar) | G | Transacao unica: cria Usuario + VinculoUsuarioDepartamento + EntidadeHierarquica + RelacionamentoHierarquico + espelho em colaboradores. Rollback se falhar. |
| RC-02 | Refatorar POST /api/colaborador/lote (criar em lote) | M | Mesmo fluxo do RC-01 para cada item. Erros individuais nao bloqueiam os demais. |
| RC-03 | Refatorar PUT /api/colaborador/{id} (editar) | G | Atualiza Usuario + propaga para colaboradores. Trata mudanca de departamento, gestor e times. |
| RC-04 | Refatorar DELETE /api/colaborador/{id} (excluir) | M | Transacao unica: limpa relacionamentos, entidades, vinculos, colaboradores, usuario. |
| RC-05 | Refatorar GET /api/hierarquia (obter hierarquia) | G | Simplificar: montar nos apenas de entidades_hierarquicas.usuario/time. Nao depender de vinculo_usuarios_departamentos para nos. |
| RC-06 | Verificar PUT /api/hierarquia/mover e POST /api/hierarquia/relacionamento | M | Garantir que usam usuario_id. Corrigir se necessario. |

## Bloco 2: Backend — Vinculos e limpeza (@Vision)

| # | Task | Esforco | Descricao |
|---|------|---------|-----------|
| RC-07 | Migrar vinculo_colaborador_time para usuario_id | M | Adicionar coluna usuario_id, backfill, atualizar VinculoColaboradorTime entity e queries. |
| RC-08 | Limpar ColaboradorService de paths legados | M | Remover dual-write desnecessario, remover referencias a gestorUsuarioPortalId, simplificar mapearGestor. |
| RC-09 | Atualizar ColaboradoresV1Service para usar fluxo unificado | M | Garantir que listarColaboradoresDoTime e listarUsuariosDoTime usam caminhos consistentes. |

## Bloco 3: Frontend — Adaptar chamadas (@Peter + @Miles)

| # | Task | Esforco | Descricao |
|---|------|---------|-----------|
| RC-10 | Adaptar modal criar colaborador para nova API | P | Se o contrato da API mudar (request/response), atualizar o frontend. |
| RC-11 | Adaptar modal editar colaborador (detalhe) | P | Garantir que mudanca de gestor/time envia os campos corretos. |
| RC-12 | Adaptar modal editar colaborador (hierarquia) | P | Idem para o modal de edicao no organograma. |
| RC-13 | Adaptar tela de hierarquia para novo response | P | Se o formato do response mudar, ajustar o mapper/display. |

## Bloco 4: Testes (@Natasha)

| # | Task | Esforco | Descricao |
|---|------|---------|-----------|
| RC-14 | Testar criar colaborador + verificar hierarquia | M | Criar colaborador, verificar que aparece na hierarquia com gestor e time corretos. |
| RC-15 | Testar editar colaborador (mudar gestor/time) | M | Editar gestor e time, verificar que hierarquia reflete a mudanca. |
| RC-16 | Testar excluir colaborador | P | Excluir, verificar que some da hierarquia sem erros de FK. |
| RC-17 | Testar criar colaborador em lote (Excel) | M | Importar via Excel, verificar que todos aparecem na hierarquia. |
| RC-18 | Testar hierarquia completa (Gestor > Time > Colaborador) | M | Montar hierarquia completa e verificar exibicao correta. |

---

## Ordem de execucao

```
RC-01 → RC-02 → RC-03 → RC-04 (CRUD sequencial — cada um depende do anterior como referencia)
    ↓
RC-05 (hierarquia — depende de RC-01 estar funcional para testar)
RC-06 (mover/relacionar — paralelo com RC-05)
    ↓
RC-07 → RC-08 → RC-09 (limpeza — apos CRUD funcional)
    ↓
RC-10 → RC-13 (frontend — apos APIs estaveis)
    ↓
RC-14 → RC-18 (testes — apos tudo)
```

## Atribuicao

| Dev | Tasks | Total |
|-----|-------|-------|
| @Thor | RC-01 a RC-06 | 6 |
| @Vision | RC-07 a RC-09 | 3 |
| @Peter + @Miles | RC-10 a RC-13 | 4 |
| @Natasha | RC-14 a RC-18 | 5 |
| **Total** | | **18 tasks** |

## Esforcos

- P = Pequeno (1-2h)
- M = Medio (3-5h)
- G = Grande (6-10h)
