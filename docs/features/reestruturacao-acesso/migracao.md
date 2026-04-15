# Plano de Migracao de Dados

> Estrategia: Migracao em fases com fallback
> Downtime estimado: 5-10 minutos (fase 3)

---

## Fase 1: Preparacao (sem downtime)

### 1.1 Criar tabela `usuarios` (nova)
```sql
CREATE TABLE usuarios ( ... ); -- conforme ADR-002
```

### 1.2 Migrar dados de `usuarios_portal`
```sql
INSERT INTO usuarios (
    empresa_id, cnpj_empresa, identificador, nome, sobrenome,
    email, senha_hash, perfil, acessa_portal, monitorado, ativo,
    cargo, telefone, data_nascimento, estado_civil, genero,
    email_pessoal, data_contratacao, avatar_bytes, avatar_atualizado_em,
    tour_concluido, criado_em
)
SELECT
    empresa_id, cnpj_empresa, identificador, nome, sobrenome,
    email, senha_hash,
    CASE perfil WHEN 'ADMIN' THEN 'ADMIN' ELSE 'GESTOR' END,
    true,     -- acessa_portal = true (todos os usuarios_portal acessam)
    false,    -- monitorado = false (nenhum era monitorado antes)
    ativo,
    cargo, telefone, data_nascimento, estado_civil, genero,
    email_pessoal, data_contratacao, avatar_bytes, avatar_atualizado_em,
    tour_concluido, criado_em
FROM usuarios_portal;
```

### 1.3 Migrar dados de `colaboradores`
```sql
INSERT INTO usuarios (
    empresa_id, cnpj_empresa, identificador, nome, sobrenome,
    email, perfil, acessa_portal, monitorado, ativo,
    cargo, telefone, data_nascimento, estado_civil, genero,
    email_pessoal, data_contratacao, avatar_bytes, url_avatar,
    tipo_identificador, ultima_maquina_id, ultima_descricao_so,
    ultimo_ip, jornada_horas, criado_em, atualizado_em
)
SELECT
    empresa_id, cnpj_empresa, identificador, nome, sobrenome,
    email, 'COLABORADOR',
    false,    -- acessa_portal = false (nenhum acessava antes)
    true,     -- monitorado = true (todos eram monitorados)
    ativo,
    cargo, telefone, data_nascimento, estado_civil, genero,
    email_pessoal, data_contratacao, avatar_bytes, url_avatar,
    tipo_identificador, ultima_maquina_id, ultima_descricao_so,
    ultimo_ip, jornada_horas, criado_em, atualizado_em
FROM colaboradores;
```

### 1.4 Criar tabela de mapeamento de IDs
```sql
CREATE TABLE _migracao_ids (
    tabela_origem VARCHAR(30),
    id_antigo BIGINT,
    id_novo BIGINT
);

-- Mapear IDs antigos para novos
INSERT INTO _migracao_ids
SELECT 'usuarios_portal', up.id, u.id
FROM usuarios_portal up
JOIN usuarios u ON u.empresa_id = up.empresa_id AND u.identificador = up.identificador AND u.perfil IN ('ADMIN', 'GESTOR');

INSERT INTO _migracao_ids
SELECT 'colaboradores', c.id, u.id
FROM colaboradores c
JOIN usuarios u ON u.empresa_id = c.empresa_id AND u.identificador = c.identificador AND u.perfil = 'COLABORADOR';
```

---

## Fase 2: Atualizar FKs (sem downtime, pode rodar em background)

### 2.1 Tabelas de eventos
```sql
ALTER TABLE eventos_atividade ADD COLUMN usuario_id BIGINT;
UPDATE eventos_atividade ea SET usuario_id = m.id_novo
FROM _migracao_ids m WHERE m.tabela_origem = 'colaboradores' AND m.id_antigo = ea.colaborador_id;

-- Repetir para: eventos_ociosidade, eventos_janela, eventos_sessao, eventos_transicao_status
```

### 2.2 Tabela agentes
```sql
ALTER TABLE agentes ADD COLUMN usuario_id BIGINT;
UPDATE agentes a SET usuario_id = m.id_novo
FROM _migracao_ids m WHERE m.tabela_origem = 'colaboradores' AND m.id_antigo = a.colaborador_id;
```

### 2.3 Tabela entidades_hierarquicas
```sql
ALTER TABLE entidades_hierarquicas ADD COLUMN usuario_id BIGINT;

UPDATE entidades_hierarquicas eh SET usuario_id = m.id_novo
FROM _migracao_ids m WHERE m.tabela_origem = 'usuarios_portal' AND m.id_antigo = eh.usuario_portal_id;

UPDATE entidades_hierarquicas eh SET usuario_id = m.id_novo
FROM _migracao_ids m WHERE m.tabela_origem = 'colaboradores' AND m.id_antigo = eh.colaborador_id;
```

### 2.4 Vinculos
```sql
-- Unificar vinculos de departamento
CREATE TABLE vinculo_usuario_departamento AS
SELECT m.id_novo as usuario_id, vud.departamento_id, vud.empresa_id, vud.criado_em
FROM vinculo_usuarios_departamentos vud
JOIN _migracao_ids m ON m.tabela_origem = 'usuarios_portal' AND m.id_antigo = vud.usuario_portal_id
UNION
SELECT m.id_novo as usuario_id, vcd.departamento_id, vcd.empresa_id, vcd.criado_em
FROM vinculo_colaboradores_departamentos vcd
JOIN _migracao_ids m ON m.tabela_origem = 'colaboradores' AND m.id_antigo = vcd.colaborador_id;

-- Vinculos de time
CREATE TABLE vinculo_usuario_time AS
SELECT m.id_novo as usuario_id, vct.time_id, vct.empresa_id, vct.criado_em
FROM vinculo_colaborador_time vct
JOIN _migracao_ids m ON m.tabela_origem = 'colaboradores' AND m.id_antigo = vct.colaborador_id;
```

---

## Fase 3: Cutover (downtime de 5-10 minutos)

### 3.1 Parar todos os servicos

### 3.2 Finalizar FKs
```sql
-- Adicionar constraints NOT NULL e FK nos novos campos
ALTER TABLE eventos_atividade ALTER COLUMN usuario_id SET NOT NULL;
ALTER TABLE eventos_atividade ADD FOREIGN KEY (usuario_id) REFERENCES usuarios(id);
ALTER TABLE eventos_atividade DROP COLUMN colaborador_id;
-- Repetir para todas as tabelas de eventos, agentes, entidades_hierarquicas

-- Drop tabelas antigas
ALTER TABLE entidades_hierarquicas DROP COLUMN usuario_portal_id;
ALTER TABLE entidades_hierarquicas DROP COLUMN colaborador_id;

-- Renomear tipo_entidade
UPDATE entidades_hierarquicas SET tipo_entidade = 'ADMIN' WHERE tipo_entidade = 'USUARIO_PORTAL';
```

### 3.3 Drop tabelas antigas
```sql
DROP TABLE vinculo_usuarios_departamentos;
DROP TABLE vinculo_colaboradores_departamentos;
DROP TABLE vinculo_colaborador_time;
DROP TABLE usuarios_portal;
DROP TABLE colaboradores;
DROP TABLE _migracao_ids;
```

### 3.4 Deploy dos 4 backends com codigo novo

### 3.5 Iniciar servicos e validar

---

## Rollback

Se algo der errado na Fase 3:
1. As tabelas antigas nao foram dropadas ainda (fazer ANTES de dropar)
2. Restaurar backup do banco (feito antes da Fase 3)
3. Redeploy com codigo antigo
