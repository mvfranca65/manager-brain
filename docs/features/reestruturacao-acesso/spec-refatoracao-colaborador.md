# Spec: Refatoracao do CRUD de Colaboradores

> Dependencia: Reestruturacao de Acesso v2 (concluida parcialmente)
> Decisor: @Tony (TL)
> Atualizado em: 2026-04-02

---

## Contexto

O fluxo atual de criacao/edicao de colaboradores faz dual-write entre as tabelas `colaboradores` (legada) e `usuarios` (unificada), com sync fragil que causa colaboradores orfaos e hierarquia quebrada. A API de hierarquia tambem depende de vinculos espalhados em multiplas tabelas.

### Problemas atuais

1. **Criar colaborador** salva em `colaboradores` + tenta criar `Usuario` + tenta vincular departamento + tenta vincular hierarquia — se qualquer passo falha, o colaborador fica orfao
2. **Editar colaborador** atualiza `colaboradores` mas nao propaga para `usuarios` de forma confiavel
3. **Excluir colaborador** precisa limpar 5+ tabelas (colaboradores, usuarios, entidades_hierarquicas, relacionamentos, vinculos)
4. **Obter hierarquia** depende de `vinculo_usuarios_departamentos.usuario_id` que nem sempre esta preenchido
5. **Vincular a gestor/time** usa IDs de tabelas diferentes (colaboradores.id vs usuarios.id) causando colisoes

### Decisao

Criar colaboradores **direto na tabela `usuarios`** como fonte primaria. A tabela `colaboradores` continua existindo para o agent monitoring (campo de eventos), mas nao e mais a fonte primaria para CRUD.

---

## APIs afetadas

### 1. POST /api/colaborador — Criar colaborador (REFATORAR)

**Fluxo atual (quebrado):**
```
1. Salva em colaboradores → OK
2. Cria Usuario → pode falhar silenciosamente
3. Vincula departamento (tabela legada) → OK
4. Vincula hierarquia → falha se Usuario nao existe
5. Vincula time → usa colaborador_id (tabela legada)
```

**Fluxo novo (transacao unica):**
```
1. Valida dados (empresa, departamento, gestor, identificador unico, email unico)
2. Cria Usuario (tabela usuarios) com perfil=COLABORADOR, monitorado=true
3. Cria VinculoUsuarioDepartamento (usuario_id)
4. Cria EntidadeHierarquica (usuario_id, tipo=COLABORADOR)
5. Cria RelacionamentoHierarquico:
   - Se gestorId informado: encontra entidade do gestor → GESTOR_COLABORADOR
   - Se timesIds informado: encontra entidade do time → TIME_COLABORADOR
6. Cria registro espelho em colaboradores (para agent monitoring)
7. Se qualquer passo falha: @Transactional faz rollback de tudo
8. Retorna resposta com dados completos
```

### 2. POST /api/colaborador/lote — Criar colaboradores em lote (REFATORAR)

Mesmo fluxo do item 1, mas para cada colaborador da lista. Erros individuais nao bloqueiam os demais — reporta falhas no response.

### 3. PUT /api/colaborador/{id} — Editar colaborador (REFATORAR)

**Pontos de edicao:**
- Detalhes do colaborador (tela de detalhe)
- Hierarquia (modal de edicao no organograma)

**Fluxo novo:**
```
1. Busca Usuario por ID
2. Atualiza campos basicos (nome, sobrenome, email, cargo, telefone, etc)
3. Se departamentoId mudou:
   - Remove vinculo antigo de VinculoUsuarioDepartamento
   - Cria novo vinculo
4. Se gestorId mudou:
   - Remove relacionamento GESTOR_COLABORADOR antigo
   - Cria novo relacionamento com novo gestor
5. Se timesIds mudou:
   - Sincroniza TIME_COLABORADOR (remove antigos, cria novos)
6. Propaga para tabela colaboradores (espelho para agent)
```

### 4. DELETE /api/colaborador/{id} — Excluir colaborador (REFATORAR)

**Fluxo novo (transacao unica):**
```
1. Busca Usuario por ID
2. Remove relacionamentos hierarquicos (pai e filho)
3. Remove entidade hierarquica
4. Remove vinculos de departamento
5. Remove vinculos de time
6. Remove registro de colaboradores (espelho)
7. Remove/desativa Usuario
```

### 5. GET /api/hierarquia — Obter hierarquia (REFATORAR)

**Problemas atuais:**
- `processarNosUnificados` depende de `vinculo_usuarios_departamentos.usuario_id` estar preenchido
- `processarNosTimes` usa `vinculoColaboradorTimeRepository` (tabela legada)
- `processarMembrosDosTimes` foi adicionado como patch mas depende de entidades com `usuario_id`

**Fluxo novo (simplificado):**
```
1. Buscar TODAS as entidades hierarquicas do departamento (via empresa_id + departamento vinculado)
2. Para cada entidade: montar no a partir de entidade.getUsuario() ou entidade.getTime()
3. Buscar TODOS os relacionamentos do departamento
4. Montar grafo (nos + relacoes)
```

A chave: a hierarquia depende APENAS de `entidades_hierarquicas` + `relacionamentos_hierarquicos`. Nao precisa consultar `vinculo_usuarios_departamentos` nem `vinculo_colaborador_time` para montar os nos — esses sao dados de apoio.

### 6. PUT /api/hierarquia/mover — Mover no na hierarquia (VERIFICAR)

Ja funciona via `entidades_hierarquicas` + `relacionamentos_hierarquicos`. Verificar se usa `usuario_id` corretamente.

### 7. POST /api/hierarquia/relacionamento — Criar relacionamento (VERIFICAR)

Idem.

---

## Tabelas envolvidas

| Tabela | Papel atual | Papel apos refatoracao |
|--------|-------------|----------------------|
| `usuarios` | Fonte primaria (parcial) | **Fonte primaria unica para CRUD** |
| `colaboradores` | Fonte primaria (legada) | Espelho para agent monitoring (write-only) |
| `entidades_hierarquicas` | Hierarquia (usa usuario_id) | Sem mudanca |
| `relacionamentos_hierarquicos` | Relacoes pai-filho | Sem mudanca |
| `vinculo_usuarios_departamentos` | Vinculo dept (usa usuario_id) | Sem mudanca |
| `vinculo_colaborador_time` | Vinculo time (usa colaborador_id) | **Migrar para usar usuario_id** |
| `vinculo_colaboradores_departamentos` | Legada (dropada) | Ja eliminada |

---

## Fora de escopo (requer migração do agent layer)

A tabela `colaboradores` NAO pode ser eliminada agora. É usada em 3 backends:
- **srv-portal:** Agente.colaborador_id, heartbeats, vinculos de time legados
- **srv-admin:** Agente.colaborador_id, validação de colaboradores, config de agente
- **srv-events:** Colaboradores entity, EventoAgente.colaborador_id, ingestão de eventos

Tarefas futuras (pós-agent migration):
- Eliminar tabela `colaboradores`
- Migrar tabelas de eventos (eventos_atividade, etc) de colaborador_id para usuario_id
- Migrar agentes.colaborador_id para agentes.usuario_id
- Refatorar agent desktop para usar usuario_id
- Eliminar srv-admin referências a Colaborador entity
