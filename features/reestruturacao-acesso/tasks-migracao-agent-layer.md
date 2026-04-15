# Tasks: Migracao do Agent Layer

> Spec: `spec-migracao-agent-layer.md`
> Atualizado em: 2026-04-02

---

## Fase 1: Migrar agentes.colaborador_id (@Thor)

| # | Task | Esforco | Descricao |
|---|------|---------|-----------|
| AL-01 | Adicionar usuario_id na entity Agente (srv-portal) | P | Adicionar campo, backfill via SQL, atualizar queries de heartbeat |
| AL-02 | Adicionar usuario_id na entity Agente (srv-admin) | P | Mesmo campo, atualizar vinculacao |
| AL-03 | Adicionar usuario_id na entity Agente (srv-events) | P | Mesmo campo, atualizar ingestao |

## Fase 2: Migrar srv-events (@Vision)

| # | Task | Esforco | Descricao |
|---|------|---------|-----------|
| AL-04 | Criar entity Usuario no srv-events | M | Entity minima (id, empresa_id, cnpj, identificador, nome, ativo, monitorado). Repository com findByCnpjEmpresaAndIdentificador. |
| AL-05 | Migrar EventoAgente.colaborador_id → usuario_id | M | Atualizar entity, queries nativas, AgentEventIngestionService |
| AL-06 | Remover Colaboradores entity do srv-events | P | Deletar entity, repository, imports |

## Fase 3: Migrar srv-admin (@Vision)

| # | Task | Esforco | Descricao |
|---|------|---------|-----------|
| AL-07 | Criar entity Usuario no srv-admin (se nao existir) | P | Verificar se ja existe (foi criada em sprint anterior). Se nao, criar entity minima. |
| AL-08 | Migrar AgenteVinculacaoController para usar usuarios | M | validar() e vincular() buscam em usuarios em vez de colaboradores |
| AL-09 | Migrar ColaboradorConfigController para usar usuarios | P | Config por identificador busca em usuarios |
| AL-10 | Remover Colaborador entity do srv-admin | P | Deletar entity, repository, imports |

## Fase 4: Eliminar colaboradores no srv-portal (@Thor)

| # | Task | Esforco | Descricao |
|---|------|---------|-----------|
| AL-11 | Remover mirror writes em ColaboradorService | M | Remover criacao/atualizacao/exclusao em tabela colaboradores. Remover VinculoColaboradorTime/Departamento writes. |
| AL-12 | Deletar entities e repositories legados | P | Colaboradores.java, ColaboradorRepository.java, VinculoColaboradorTime.java, VinculoColaboradorTimeRepository.java, VinculoColaboradorDepartamento.java, VinculoColaboradorDepartamentoRepository.java |

## Fase 5: DROP tabelas + testes (@Thor + @Natasha)

| # | Task | Esforco | Descricao |
|---|------|---------|-----------|
| AL-13 | SQL: DROP colaboradores e vinculo_colaborador_time | P | Script manual (Flyway nao roda automaticamente) |
| AL-14 | Compilar e testar os 3 backends | M | mvn compile + mvn test em cada. Corrigir falhas. |
| AL-15 | @Natasha: Teste completo end-to-end | G | Criar gestor (srv-admin), primeiro acesso, criar colaborador, hierarquia, vincular agente, ingestao de eventos, editar, excluir |

---

## Ordem de execucao

```
AL-01 + AL-02 + AL-03 (paralelo — adicionar usuario_id em Agente nos 3 backends)
    ↓
AL-04 → AL-05 → AL-06 (srv-events migrado)
AL-07 → AL-08 → AL-09 → AL-10 (srv-admin migrado, paralelo com srv-events)
    ↓
AL-11 → AL-12 (srv-portal limpo)
    ↓
AL-13 → AL-14 → AL-15 (drop + testes)
```

## Atribuicao

| Dev | Tasks | Total |
|-----|-------|-------|
| @Thor | AL-01, AL-02, AL-03, AL-11, AL-12, AL-13, AL-14 | 7 |
| @Vision | AL-04, AL-05, AL-06, AL-07, AL-08, AL-09, AL-10 | 7 |
| @Natasha | AL-15 | 1 |
| **Total** | | **15 tasks** |
