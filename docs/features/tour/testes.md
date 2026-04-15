# Testes — Tours Contextuais
> Domínio: @Natasha | QA
> Última atualização: 2026-03-31

---

## Cobertura esperada

| Camada | Abordagem |
|--------|-----------|
| Backend (TourController) | Testes de integração JUnit/MockMvc |
| Frontend (TourService) | Jasmine/Jest unitário |
| E2E | Validação manual por cenário (Cypress opcional fase 2) |

---

## Backend — TourControllerTest

### PATCH /api/v1/tour/flags

- [ ] Flag válida + JWT válido → 204, banco atualizado
- [ ] Flag válida com `valor: false` (reset para "Rever tour") → 204, banco atualizado
- [ ] Flag desconhecida (não existe no enum `TourFlag`) → 400 com mensagem descritiva
- [ ] Sem JWT → 401
- [ ] JWT de outro usuário não altera flags do usuário correto (isolamento por `usuarioPortalId`)
- [ ] Dois usuários da mesma empresa têm flags independentes

### Login — `tourFlags` no response

- [ ] Usuário novo (primeiro login): `primeiroLogin = true`, todas as flags `false`
- [ ] Usuário que já concluiu `tour-home`: `tourHomeConcluido = true`, `primeiroLogin = false`
- [ ] Usuário sem registro em `user_tour_flags` (existente antes da migration): recebe defaults sem erro

### Migration

- [ ] Todos os usuários existentes recebem registro na `user_tour_flags` com defaults corretos
- [ ] `primeiro_login = true` para usuários com `tour_concluido = false` (legado)
- [ ] `tour_home_concluido = true` para usuários com `tour_concluido = true` (legado)

---

## Frontend — TourService (unitário)

### deveDisparar()

- [ ] `primeiroLogin = true` → `tour-home` deve disparar em `/home`
- [ ] `primeiroLogin = false` → `tour-home` não dispara em `/home`
- [ ] `tourOrgEstruturaConcluido = false` → `tour-org-estrutura` dispara ao entrar em `/organizacao`
- [ ] `tourOrgEstruturaConcluido = true` → `tour-org-estrutura` não dispara
- [ ] `tourColaboradoresConcluido = false` + sem times → dispara variação A (steps 1-2)
- [ ] `tourColaboradoresConcluido = false` + com times + sem dados → dispara variação B (steps 1-3)
- [ ] `tourColaboradoresConcluido = false` + com dados → dispara variação C (steps 1-5)
- [ ] `tourColaboradoresConcluido = true` + `tourColaboradoresComDadosConcluido = false` + com dados → dispara a partir do step 2
- [ ] `tourColaboradorDetalheConcluido = false` + colaborador sem dados de semana → tour NÃO dispara

### encerrarTour()

- [ ] Ao encerrar `tour-home` → chama PATCH com `tour_home_concluido: true`, `primeiro_login: false`, `menu_org_pulsando: true`
- [ ] Ao pular qualquer tour (exceto tour-home) → flag do tour é setada como `true`
- [ ] Ao clicar "Rever" em Configurações → flag é resetada para `false` e PATCH é chamado

### Pulse no menu

- [ ] `menuOrgPulsando = true` → classe `menu-item--pulsando` adicionada no elemento `[data-tour="menu-organizacao"]`
- [ ] Clicar no menu Organização com flag `true` → PATCH `menu_org_pulsando: false` + classe removida
- [ ] Navegar diretamente para `/organizacao` → mesma remoção de pulse

---

## E2E — Validação manual por tour

### tour-home

- [ ] **Primeiro login real:** usuário novo acessa `/home` → tour inicia automaticamente no step 1, sem botão "Pular"
- [ ] **4 steps completos:** navegar por step 1 → 2 → 3 → 4, verificar textos corretos e elementos destacados
- [ ] **Concluir no step 4:** botão "Entendi" → menu Organização começa a pulsar (animação visível)
- [ ] **Reacesso após conclusão:** usuário fecha e reabre o portal → tour-home não dispara novamente
- [ ] **Rever via Configurações:** clicar "Rever" no tour-home → tour inicia novamente com botão "Pular" disponível

### tour-org-estrutura

- [ ] **Primeiro acesso a /organizacao:** tour inicia automaticamente na aba Departamentos
- [ ] **Navegação entre steps:** step 1 (tab-departamentos) → step 2 (tab-cargos) → step 3 (tab-departamentos)
- [ ] **Pular:** clicar "Pular" em qualquer step → flag setada, tour não reaparece no próximo acesso
- [ ] **Menu pulsando para de pulsar:** ao entrar em `/organizacao`, animação do menu é removida

### tour-org-times

- [ ] **Cenário sem times:** acessar aba Times sem times cadastrados → step 1 + step 2 com mensagem "Crie seu primeiro time"
- [ ] **Cenário com times:** acessar aba Times com times cadastrados → apenas step 1 encurtado
- [ ] **Sequência após tour-org-estrutura:** ao concluir tour-org-estrutura ainda na página, tour-org-times inicia automaticamente

### tour-org-hierarquia

- [ ] **Acesso à aba Hierarquia:** tour inicia com 4 steps
- [ ] **Steps de botões admin:** `btn-gestao-pessoas` e `btn-download-instalador` aparecem apenas para perfil Admin
- [ ] **Pular no step 3:** tour conclui sem bloquear fluxo

### tour-colaboradores

- [ ] **Sem times:** steps 1 e 2 com mensagem orientativa, sem loop ou erro
- [ ] **Com times, sem dados:** steps 1 a 3, step 3 explica que agente ainda não enviou dados
- [ ] **Com dados:** steps 1 a 5 completos, step 4 e 5 apontam para `colaboradores-lista`
- [ ] **Segunda passagem (com dados, após conclusão parcial):** variação com dados dispara a partir do step 2
- [ ] **Selector `data-tour="colaboradores-lista"` presente no HTML:** tour não pula o step por elemento faltando

### tour-colaborador-detalhe

- [ ] **Colaborador sem dados de semana:** tour NÃO dispara (pré-condição não atendida)
- [ ] **Colaborador com dados:** 4 steps em sequência — score-card, timeline, indicadores, relatorio-link
- [ ] **Todos os 4 selectors presentes no HTML:** sem steps pulados por elemento não encontrado
- [ ] **Pular no step 2:** tour encerra, flag setada

### tour-relatorios

- [ ] **Sem relatórios disponíveis:** steps 1 a 3 (sem step 4 do `relatorios-cards-area`)
- [ ] **Com relatórios disponíveis:** steps 1 a 4 completos
- [ ] **Selectors presentes:** `relatorios-filtro-time`, `relatorios-semana-nav`, `relatorios-cards-area`
- [ ] **Ícone `?` no header:** clicar → tour reinicia imediatamente mesmo com flag `true`

### tour-perfil

- [ ] **Primeiro acesso a /configuracoes/perfil:** 3 steps em sequência — perfil-card, perfil-senha, configuracoes-tours
- [ ] **Step 3 aponta para seção nova:** `configuracoes-tours` existe no DOM na tela de perfil
- [ ] **Pular no step 1:** flag setada, tour não reaparece

---

## E2E — Seção "Rever tours" nas Configurações

- [ ] Todos os 8 tours listados com nome amigável correto
- [ ] Clicar "Rever" em `tour-relatorios` → navega para `/relatorios` e tour dispara automaticamente
- [ ] Clicar "Rever" em `tour-colaborador-detalhe` → navega para `/colaboradores` (instrução para acessar um membro)
- [ ] Clicar "Rever" em `tour-home` → tour dispara com botão "Pular" disponível (não obrigatório na revisão)
- [ ] Dois cliques em "Rever" não duplicam instâncias de tour — instância anterior destruída antes de iniciar nova

---

## E2E — Ícone `?` no header

- [ ] Páginas com tour associado exibem ícone `pi-question-circle` no header
- [ ] Tooltip "Rever o tour desta seção" aparece no hover
- [ ] Páginas sem tour associado (ex: `/configuracoes/seguranca`) não exibem o ícone
- [ ] Clicar no ícone em `/relatorios` → `tour-relatorios` inicia imediatamente sem navegação
- [ ] Clicar no ícone em `/home` → `tour-home` inicia com botão "Pular" disponível

---

## Regressão — Comportamentos que não devem mudar

- [ ] Tour antigo (campo legado `tourConcluido` em `UsuarioPortal`) não interfere nas novas flags
- [ ] Login de usuário existente sem registro em `user_tour_flags` funciona normalmente (defaults aplicados)
- [ ] Fechar tour clicando no overlay ou no `X` ainda seta a flag como concluída (mesmo comportamento atual)
- [ ] Navegação entre rotas durante um tour em andamento não duplica instâncias do driver.js

---

## Riscos identificados

- **Elemento não encontrado no DOM:** se o `data-tour` selector não existir quando o step for disparado, o `TourService` atual pula para o próximo step. Verificar que todos os novos selectors estejam presentes antes de subir cada tour para produção.
- **Race condition no ngOnInit:** tour pode disparar antes do componente renderizar os elementos. A lógica de `agendarDestaque` com delay deve ser preservada e ajustada por contexto.
- **Múltiplas abas:** usuário com duas abas abertas pode ter flags dessincronizadas em memória. Baixo impacto — flags só avançam, nunca regridem em uso normal.
- **Tour-home obrigatório na revisão:** ao clicar "Rever" no tour-home, o botão "Pular" precisa estar disponível mesmo que o tour original fosse sem pular. Garantir que a variação "revisão" tenha `showButtons` incluindo `'close'`.
