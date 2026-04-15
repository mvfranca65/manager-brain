# Spec: Migracao do Agent Layer para tabela usuarios

> Dependencia: Refatoracao CRUD Colaboradores (concluida)
> Decisor: @Tony (TL)
> Atualizado em: 2026-04-02

---

## Objetivo

Eliminar as tabelas `colaboradores` e `vinculo_colaborador_time` migrando todas as referencias nos 3 backends (srv-portal, srv-admin, srv-events) para usar a tabela unificada `usuarios`.

## Estado atual

| Tabela | srv-portal | srv-admin | srv-events |
|--------|-----------|-----------|------------|
| `colaboradores` | Espelho (write) | Entity + validacao agente | Entity + ingestao eventos |
| `vinculo_colaborador_time` | Espelho (write) | Nao usa | Nao usa |
| `agentes.colaborador_id` | Queries de heartbeat | Vinculacao de agente | Ingestao de eventos |

## Plano

### Fase 1: Migrar `agentes.colaborador_id` → `agentes.usuario_id`

A coluna `agentes.colaborador_id` e a FK central que conecta o agent desktop ao colaborador monitorado. Precisa de:
1. Adicionar coluna `usuario_id` na tabela `agentes`
2. Backfill: popular `usuario_id` a partir de `colaboradores.identificador` → `usuarios.identificador`
3. Atualizar os 3 backends para usar `usuario_id`
4. Dropar `colaborador_id` apos validacao

### Fase 2: Migrar srv-events

O srv-events usa `colaboradores` para resolver o colaborador na ingestao de eventos:
- `ColaboradoresRepository.findByCnpjEmpresaAndIdentificador()` → trocar para `UsuarioRepository`
- `EventoAgente.colaborador_id` → trocar para `EventoAgente.usuario_id`
- `Agente.colaborador_id` → ja migrado na Fase 1

### Fase 3: Migrar srv-admin

O srv-admin usa `colaboradores` para validar e vincular agentes:
- `AgenteVinculacaoController` → validar/vincular usando `usuarios`
- `ColaboradorConfigController` → config por identificador usando `usuarios`
- `Agente.colaborador_id` → ja migrado na Fase 1

### Fase 4: Eliminar tabela `colaboradores` no srv-portal

Remover:
- `Colaboradores.java` entity
- `ColaboradorRepository.java`
- Mirror writes no `ColaboradorService`
- `VinculoColaboradorTime` entity e repository
- `VinculoColaboradorDepartamento` entity e repository (se ainda existir)

### Fase 5: DROP tabelas

```sql
DROP TABLE IF EXISTS vinculo_colaborador_time CASCADE;
DROP TABLE IF EXISTS colaboradores CASCADE;
```

## Impacto no agent desktop

**Nenhum.** O agent envia `identificador` (string) para se vincular. O backend resolve `identificador` → `usuarios.id` em vez de `identificador` → `colaboradores.id`. O agent nao precisa ser alterado.

## Tabelas de eventos

As tabelas de eventos (`eventos_atividade`, `eventos_ociosidade`, `eventos_janela`, `eventos_sessao`, `eventos_transicao_status`) tem `colaborador_id`. Essas colunas precisam ser migradas para `usuario_id` tambem, mas o schema e gerenciado pelo Hibernate `ddl-auto: update` — ao mudar a entity no srv-events, o Hibernate cria a nova coluna automaticamente.
