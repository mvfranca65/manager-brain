# Testes Regressivos — Reestruturação de Acesso
> @Natasha | QA
> Executar antes e depois do cutover

---

## 1. Login e Autenticação

| # | Teste | Como validar | Resultado |
|---|-------|-------------|-----------|
| 1.1 | Login ADMIN | POST /auth/v2/login com admin. Deve retornar token com perfil=ADMIN | [ ] |
| 1.2 | Login GESTOR | POST /auth/v2/login com gestor. Deve retornar token com perfil=GESTOR | [ ] |
| 1.3 | Login COLABORADOR | POST /auth/v2/login com colaborador que tem acessa_portal=true. Deve funcionar | [ ] |
| 1.4 | Login COLABORADOR sem acesso | POST /auth/v2/login com colaborador acessa_portal=false. Deve retornar 403 | [ ] |
| 1.5 | Login com senha errada | Deve retornar 401 | [ ] |
| 1.6 | Token V2 aceito nas APIs | Usar token V2 para chamar GET /v2/me. Deve retornar dados do usuario | [ ] |

---

## 2. Permissões por Perfil

| # | Teste | Como validar | Resultado |
|---|-------|-------------|-----------|
| 2.1 | ADMIN vê todos os colaboradores | GET /api/v2/usuarios/{id}/detalhes com ID de qualquer usuario. Deve funcionar | [ ] |
| 2.2 | GESTOR vê sua cadeia | GET /api/v2/usuarios/{id}/detalhes com ID de subordinado. Deve funcionar | [ ] |
| 2.3 | GESTOR não vê fora da cadeia | GET /api/v2/usuarios/{id}/detalhes com ID fora da hierarquia. Deve retornar 403 | [ ] |
| 2.4 | COLABORADOR vê apenas si mesmo | GET /api/v2/usuarios/{id}/detalhes com proprio ID. Deve funcionar | [ ] |
| 2.5 | COLABORADOR não vê outros | GET /api/v2/usuarios/{id}/detalhes com ID de outro. Deve retornar 403 | [ ] |
| 2.6 | ADMIN acessa Organização | No portal, menu Organização deve aparecer | [ ] |
| 2.7 | GESTOR não acessa Organização | Menu Organização NÃO deve aparecer | [ ] |
| 2.8 | COLABORADOR vê Meu Painel | Menu Meu Painel deve aparecer | [ ] |

---

## 3. Tabela Unificada — Integridade

| # | Teste | Como validar | Resultado |
|---|-------|-------------|-----------|
| 3.1 | Todos os usuarios_portal migrados | SELECT count(*) FROM usuarios WHERE perfil IN ('ADMIN','GESTOR') deve = count de usuarios_portal | [ ] |
| 3.2 | Todos os colaboradores migrados | SELECT count(*) FROM usuarios WHERE perfil='COLABORADOR' deve = count de colaboradores | [ ] |
| 3.3 | Eventos com usuario_id | SELECT count(*) FROM eventos_atividade WHERE usuario_id IS NULL deve = 0 | [ ] |
| 3.4 | Agentes com usuario_id | SELECT count(*) FROM agentes WHERE usuario_id IS NULL deve = 0 | [ ] |
| 3.5 | Hierarquia com usuario_id | SELECT count(*) FROM entidades_hierarquicas WHERE usuario_id IS NULL AND time_id IS NULL deve = 0 | [ ] |
| 3.6 | Vinculos departamento migrados | count de vinculo_usuario_departamento >= count de vinculo_usuarios_departamentos + vinculo_colaboradores_departamentos | [ ] |
| 3.7 | Vinculos time migrados | count de vinculo_usuario_time = count de vinculo_colaborador_time | [ ] |

---

## 4. Funcionalidades Existentes (Regressão)

| # | Teste | Como validar | Resultado |
|---|-------|-------------|-----------|
| 4.1 | Home carrega stats | 4 cards (Total, Online, Ausente, Offline) com valores corretos | [ ] |
| 4.2 | Colaboradores: listar por time | Selecionar time, grid de cards aparece | [ ] |
| 4.3 | Detalhe: KPIs carregam | Visão Geral mostra KPIs com valores | [ ] |
| 4.4 | Detalhe: timeline carrega | Indicadores mostra gráfico temporal | [ ] |
| 4.5 | Relatórios: semanas disponíveis | Selecionar time, semanas carregam | [ ] |
| 4.6 | Organização: hierarquia carrega | Organograma mostra gestores e colaboradores | [ ] |
| 4.7 | Tour funciona | Clicar "Fazer tour" — 19 passos rodam | [ ] |
| 4.8 | Avatar carrega | Foto do colaborador aparece no card e detalhe | [ ] |

---

## 5. Agent

| # | Teste | Como validar | Resultado |
|---|-------|-------------|-----------|
| 5.1 | Vinculação funciona | Instalar agent com identificador. Deve vincular ao usuario na tabela unificada | [ ] |
| 5.2 | Eventos chegam | Agent envia eventos. Backend persiste com usuario_id | [ ] |
| 5.3 | Heartbeat funciona | Agent envia heartbeat. Backend recebe e atualiza | [ ] |
| 5.4 | Status online correto | Colaborador com agent ativo mostra "Online" no portal | [ ] |

---

## 6. Validação Pós-Cutover (RA-36)

| # | Teste | Como validar | Resultado |
|---|-------|-------------|-----------|
| 6.1 | Colunas legadas removidas | \d eventos_atividade — não deve ter colaborador_id | [ ] |
| 6.2 | Tabelas legadas existem | usuarios_portal e colaboradores devem existir (drop manual depois) | [ ] |
| 6.3 | Endpoints V1 ainda funcionam | GET /api/colaborador/{id}/detalhes ainda responde (compatibilidade) | [ ] |
| 6.4 | Endpoints V2 funcionam | GET /api/v2/usuarios/{id}/detalhes responde com dados corretos | [ ] |
| 6.5 | Login V1 ainda funciona | POST /auth/login ainda autentica admins (compatibilidade) | [ ] |
| 6.6 | Login V2 funciona | POST /auth/v2/login autentica qualquer perfil com acesso | [ ] |

---

## Checklist pós-teste

- [ ] Todos os testes acima passaram
- [ ] Nenhuma exceção nos logs dos 4 backends
- [ ] Agent funcionando normalmente
- [ ] Performance aceitável (tempo de resposta similar ao anterior)
- [ ] Dados consistentes entre V1 e V2 endpoints
