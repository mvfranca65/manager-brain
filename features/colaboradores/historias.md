# Histórias — Colaboradores
> Domínio: @Steve | PO
> Última atualização: 2026-03-25

---

## Épico

**Como gestor, quero uma visão completa e centralizada de cada colaborador do meu time — dados cadastrais, histórico de atividade e indicadores individuais — para tomar decisões de gestão com precisão e sem depender de planilhas.**

---

## Mapa de cobertura da API

Endpoints já existentes em `ColaboradorController`:

| Endpoint | Método | Status |
|---|---|---|
| `POST /api/colaborador` | Criar | Implementado |
| `POST /api/colaborador/lote` | Criar em lote | Implementado |
| `GET /api/colaborador/{id}/detalhes` | Detalhar por ID | Implementado |
| `GET /api/colaborador/identificador/{id}/detalhes` | Detalhar por identificador | Implementado |
| `PUT /api/colaborador/{id}` | Atualizar | Implementado |
| `PATCH /api/colaborador/{id}/status` | Ativar/desativar | Implementado |
| `DELETE /api/colaborador/{id}` | Excluir | Implementado |
| `GET /api/colaborador/{id}/avatar` | Obter avatar | Implementado |
| `GET /api/colaborador/{id}/dispositivos` | Listar dispositivos | Implementado |
| `DELETE /api/colaborador/{id}/dispositivos/{agenteId}` | Desvincular dispositivo | Implementado |
| `GET /api/colaborador/{id}/status-online` | Status online | Implementado |
| `GET /api/colaborador/gestores` | Listar gestores | Implementado |
| `GET /api/colaborador/home/resumo` | Resumo home | Implementado |
| `GET /api/gerente/listar-colaboradores` | Listagem geral | Implementado |
| `GET /api/times/{timeId}/colaboradores` | Listagem por time | Implementado |

---

## Épico A — Tela Unificada de Colaboradores

**Motivação:** A jornada atual exige 3 cliques para chegar ao detalhe de um colaborador (selecionar departamento > selecionar time > selecionar colaborador). Unificar as duas primeiras telas elimina um passo inteiro.

**Spec completo:** `docs/superpowers/specs/2026-03-25-colaboradores-tela-unificada-design.md`

---

### US-C01: Selecionar time via dropdown

**Como** gestor/admin,
**Quero** selecionar um time em um dropdown com busca integrada,
**Para** acessar rapidamente os colaboradores sem navegar por uma tela separada de times.

**Criterios de aceite:**
- Dropdown lista apenas os times visiveis na hierarquia do usuario logado
- Dropdown suporta digitacao para filtrar times por nome
- Botao de limpar (X) volta ao estado vazio
- Sem time selecionado, nenhum dado e carregado — apenas mensagem "Selecione um time para visualizar os colaboradores"

---

### US-C02: Visualizar resumo do time selecionado

**Como** gestor/admin,
**Quero** ver um resumo compacto do time selecionado (quantidade de colaboradores, gestor, descricao),
**Para** ter contexto sobre o time sem precisar de uma tela dedicada.

**Criterios de aceite:**
- Faixa compacta com fundo indigo claro e borda lateral colorida
- Exibe: quantidade de colaboradores, nome do gestor principal, descricao do time
- Se o time nao tem colaboradores, exibe mensagem "Este time ainda nao possui colaboradores"

---

### US-C03: Filtrar e buscar colaboradores

**Como** gestor/admin,
**Quero** buscar colaboradores por nome, identificador ou cargo, e filtrar por status online,
**Para** encontrar rapidamente quem preciso.

**Criterios de aceite:**
- Input de busca com debounce de 300ms
- Filtros de status: Todos, Online, Ocioso, Offline — com contadores totais do time
- Contadores refletem o total do time (via API), nao apenas a pagina atual
- Paginacao funciona corretamente (16, 32, 64 por pagina)

---

### US-C04: Navegacao e deep linking

**Como** gestor/admin,
**Quero** que a URL reflita o time selecionado,
**Para** poder compartilhar links diretos e voltar ao mesmo estado apos navegar.

**Criterios de aceite:**
- URL: `/colaboradores?time=123`
- Acessar URL direta pre-seleciona o dropdown e carrega dados
- Voltar da tela de detalhe preserva o time selecionado
- Redirect de `/times` para `/colaboradores` funciona (inclui sub-rotas)
- Time invalido no param exibe toast "Time nao encontrado"

---

### US-C05: Responsividade

**Como** gestor/admin,
**Quero** usar a tela em desktop, tablet e celular,
**Para** acessar os colaboradores de qualquer dispositivo.

**Criterios de aceite:**
- Desktop (1024px+): dropdown 400px, cards 2 colunas, filtros inline
- Tablet (768-1023px): dropdown 100%, cards 2 colunas, filtros inline
- Mobile (375-767px): dropdown 100%, cards 1 coluna, filtros em scroll horizontal

---

### US-C06: Seguranca — validacao hierarquica na API

**Como** sistema,
**Quero** que o endpoint de colaboradores valide se o usuario tem acesso hierarquico ao time solicitado,
**Para** impedir que um gestor veja colaboradores de times fora da sua hierarquia.

**Criterios de aceite:**
- Admin acessa qualquer time da empresa
- Gestor acessa apenas times abaixo dele na hierarquia
- Tentativa de acessar time fora da hierarquia retorna 403
- Tentativa de acessar time de outro tenant retorna 401/403

---

## Épico B — CRUD completo de colaboradores

---

### US-COL-01 — Listagem de colaboradores

**Como** gestor ou admin, **quero** visualizar todos os colaboradores da minha cadeia em uma lista paginada com busca e filtros, **para** localizar rapidamente quem preciso e ter uma visão do status do time.

**Prioridade:** P1
**Status:** Implementado (frontend + backend)

**Critérios de aceitação:**
- [ ] A lista exibe: avatar, nome completo, cargo, departamento, time(s), status online (Online / Ausente / Offline), status de cadastro (ATIVO / INATIVO)
- [ ] A busca filtra por nome, identificador e e-mail corporativo (debounce 400ms)
- [ ] Paginação com tamanho configurável (padrão 20 itens)
- [ ] Filtro por gestor direto (dropdown pré-populado com gestores visíveis ao usuário logado)
- [ ] Filtro por time e por status de cadastro (ATIVO / INATIVO / Todos)
- [ ] Colaboradores inativos exibidos com badge visual distinto
- [ ] ADMIN vê todos os colaboradores do tenant; MANAGER e VIEWER veem apenas sua cadeia hierárquica
- [ ] VIEWER: botões de ação (editar, desativar) ocultos
- [ ] Estado vazio com mensagem amigável quando não há resultados

---

### US-COL-02 — Cadastro de novo colaborador

**Como** admin, **quero** cadastrar um novo colaborador preenchendo um formulário completo, **para** registrá-lo na plataforma e iniciar o monitoramento assim que o agente for instalado.

**Prioridade:** P1
**Status:** Implementado (frontend `ColaboradorFormModalComponent` + backend `POST /api/colaborador`)

**Critérios de aceitação:**
- [ ] Campos obrigatórios: nome, sobrenome, identificador (CPF ou matrícula), cargo, departamento
- [ ] Campos opcionais: e-mail corporativo, e-mail pessoal, telefone, data de nascimento, data de contratação, estado civil, gênero, jornada de trabalho (horas/dia, padrão 8h), time(s)
- [ ] Identificador é único por tenant — erro amigável se duplicado
- [ ] Avatar opcional — upload de JPG/PNG; backend converte para WEBP
- [ ] Vinculação a um ou mais times via multiselect; vinculação a gestor direto via dropdown com busca
- [ ] Departamento obrigatório
- [ ] Ao salvar: colaborador aparece na listagem com status ATIVO
- [ ] Apenas ADMIN pode cadastrar — botão oculto para MANAGER e VIEWER
- [ ] Colaborador pertence exclusivamente ao tenant do usuário logado (multitenancy por CNPJ)
- [ ] Erro de validação exibido campo a campo (não somente toast genérico)

---

### US-COL-03 — Edição de colaborador

**Como** admin, **quero** editar os dados de um colaborador existente, **para** manter o cadastro atualizado após mudanças de cargo, departamento, time ou informações pessoais.

**Prioridade:** P1
**Status:** Implementado (frontend `ColaboradorFormModalComponent` + backend `PUT /api/colaborador/{id}`)

**Critérios de aceitação:**
- [ ] Todos os campos do cadastro são editáveis, exceto o identificador (exibido em read-only)
- [ ] Avatar pode ser substituído por novo upload (JPG/PNG → WEBP)
- [ ] Troca de departamento, time(s) e gestor atualiza os vínculos hierárquicos
- [ ] Alteração de jornada de horas reflete nos cálculos de produtividade a partir da data de edição
- [ ] Apenas ADMIN pode editar
- [ ] Feedback de sucesso ou erro via toast após salvar
- [ ] Formulário abre em modal (padrão `ColaboradorFormModalComponent`)

---

### US-COL-04 — Ativação e desativação de colaborador

**Como** admin, **quero** ativar ou desativar um colaborador, **para** suspender o monitoramento de quem saiu da empresa sem excluir o histórico.

**Prioridade:** P1
**Status:** Implementado (backend `PATCH /api/colaborador/{id}/status`); frontend expõe ação na tela de detalhe

**Critérios de aceitação:**
- [ ] Ação disponível na tela de detalhe e na listagem (menu de contexto)
- [ ] Confirmação modal antes de desativar: "Isso suspende o monitoramento. Confirmar?"
- [ ] Colaborador desativado: badge INATIVO na listagem, agente para de registrar eventos
- [ ] Colaborador reativado retorna ao monitoramento normalmente sem perda de histórico
- [ ] Apenas ADMIN pode alterar status
- [ ] Operação refletida imediatamente na UI sem reload

---

### US-COL-05 — Exclusão permanente de colaborador

**Como** admin, **quero** excluir permanentemente um colaborador, **para** remover registros criados por engano ou por exigência da LGPD.

**Prioridade:** P2
**Status:** Implementado no backend (`DELETE /api/colaborador/{id}`); frontend precisa expor botão com confirmação

**Critérios de aceitação:**
- [ ] Ação disponível somente na tela de detalhe (não na listagem)
- [ ] Confirmação modal com texto explícito: "Esta ação é irreversível e remove todos os dados do colaborador."
- [ ] Após excluir: redirecionamento para a listagem
- [ ] Apenas ADMIN pode excluir
- [ ] API retorna 404 para acessos posteriores ao colaborador excluído

---

## Épico C — Tela de detalhe e atividade

---

### US-COL-06 — Visualização de detalhe do colaborador

**Como** gestor ou admin, **quero** acessar uma página de detalhe com todas as informações do colaborador, **para** ter uma visão completa antes de tomar decisões de feedback, alocação ou desligamento.

**Prioridade:** P1
**Status:** Implementado (frontend com 4 abas: Visão Geral, Indicadores, Timeline 360°, Detalhes do Colaborador)

**Critérios de aceitação:**
- [ ] Header com: avatar, nome, cargo, departamento, gestor direto, status online em tempo real (polling 60s)
- [ ] Aba "Visão Geral": KPIs do período selecionado, donut de tempo ativo/pausa/offline, top aplicativos
- [ ] Aba "Indicadores": scores e indicadores IA (ver US-COL-10)
- [ ] Aba "Timeline 360°": visualização gráfica e lista de eventos do dia selecionado (ver US-COL-07)
- [ ] Aba "Detalhes do Colaborador": dados cadastrais completos + dispositivos vinculados (ver US-COL-12)
- [ ] Rota: `/times/membro/:id`
- [ ] MANAGER e VIEWER acessam somente colaboradores da sua cadeia
- [ ] Deeplink funcional: outras telas linkam diretamente para esta página

---

### US-COL-07 — Timeline de atividades do colaborador

**Como** gestor, **quero** ver a timeline de atividades do colaborador para um dia específico, **para** entender como foi a jornada: início, pausas, aplicativos usados, horário de saída.

**Prioridade:** P1
**Status:** Implementado (frontend `TabTimelineComponent`; backend `TimelineColaboradorService`)

**Critérios de aceitação:**
- [ ] Barra grafica horizontal com segmentos por hora: Ativo (verde), Pausa (amarelo), Ausente (cinza)
- [ ] Seletor de data para navegar entre dias (padrao: hoje)
- [ ] Cards de resumo: Jornada total, Tempo ativo, Tempo em pausa, Tempo ausente
- [ ] Indicador visual quando jornada real excede a jornada esperada do cadastro
- [ ] Lista de eventos detalhados: inicio, fim, duracao, titulo (aplicativo ou tipo de evento)
- [ ] Paginacao na lista de eventos (padrao 10/pagina)
- [ ] Classificacao (3 estados — ADR-001):
  - ATIVO (janela ativa, interacao detectada)
  - PAUSA (LOCK, ociosidade <= 10min, gaps de 10-30min entre eventos)
  - AUSENTE (ociosidade > 10min, sem heartbeat > 30min, LOGOUT, gaps > 30min)
- [ ] Gap-filling: gaps entre blocos geram blocos sinteticos (10-30min = PAUSA, >30min = AUSENTE)
- [ ] Fuso horario: `America/Sao_Paulo`
- [ ] Eventos com duracao < 30s filtrados (ja implementado no backend)

---

### US-COL-08 — Indicadores individuais de período (KPIs)

**Como** gestor, **quero** ver os KPIs de produtividade de um colaborador para períodos configuráveis (hoje, 7d, 15d, 30d), **para** acompanhar tendências e identificar mudanças de comportamento.

**Prioridade:** P1
**Status:** Implementado (frontend `TabVisaoGeralComponent`; backend `ResumoPeriodoColaboradorResponse`)

**Critérios de aceitação:**
- [ ] Seletor de período: Hoje / Últimos 7 dias / Últimos 15 dias / Últimos 30 dias / Intervalo customizado
- [ ] KPIs: jornada media, tempo ativo medio, % ativo, % pausa, % ausente (3 estados — ADR-001)
- [ ] Donut chart com distribuicao visual ativo / pausa / ausente (3 fatias)
- [ ] Top 5 aplicativos mais usados: nome, percentual de uso, tempo absoluto
- [ ] Padrões de foco: sessão mais longa de atividade contínua, hora do dia com maior produtividade
- [ ] Dados calculados a partir de eventos reais do agente
- [ ] Para "Hoje": inclui dados até o momento atual

---

## Épico D — Gestão avançada (MVP2)

---

### US-COL-09 — Gestão de avatar

**Como** admin, **quero** fazer upload e substituir o avatar de um colaborador, **para** personalizar o perfil e facilitar a identificação visual.

**Prioridade:** P2
**Status:** Implementado (backend converte para WEBP; frontend aceita JPG/PNG no formulário)

**Critérios de aceitação:**
- [ ] Upload aceita JPG e PNG; tamanho máximo 5 MB; validação de mimetype e extensão no frontend
- [ ] Backend converte automaticamente para WEBP antes de persistir
- [ ] Preview imediato após seleção (antes de salvar)
- [ ] Avatar servido com `Cache-Control: max-age=86400, private`
- [ ] Exibido em listagem, detalhe, header e cards de time
- [ ] Sem avatar: placeholder com iniciais do nome em cor gerada por hash
- [ ] Apenas ADMIN pode alterar avatar

---

### US-COL-10 — Indicadores IA — iManager Score (integração com Relatórios)

**Como** gestor, **quero** ver na aba "Indicadores" do detalhe do colaborador o iManager Score e os indicadores psicológicos gerados pela IA, **para** ter uma leitura qualificada do estado do colaborador além dos dados brutos.

**Prioridade:** P1
**Dependência:** Conclusão da Task 13 do épico de Relatórios (`GET /api/v1/relatorios/colaborador`)
**Status:** Pendente

**Critérios de aceitação:**
- [ ] iManager Score com delta em relação à semana anterior (seta: subindo / caindo / estável)
- [ ] 6 pilares do score: Atividade, Foco, Consistência, Saúde, Fragmentação, Anomalias — em radar ou barras horizontais
- [ ] 6 indicadores psicológicos: Burnout, Desengajamento, Sobrecarga Silenciosa, Instabilidade, Fadiga FDS, Horas Extras — com badge de nível (VERDE / AMARELO / VERMELHO / INDETERMINADO)
- [ ] Resumo executivo em texto gerado pela IA
- [ ] Seletor de semana para navegar entre relatórios históricos disponíveis
- [ ] Botão "Ver relatório completo do time" → deeplink `/relatorios?timeId=&colaboradorId=&semana=`
- [ ] Estado vazio: "Relatório disponível toda segunda-feira"
- [ ] INDETERMINADO quando colaborador tem menos de 2 semanas de histórico

---

### US-COL-11 — Importação em lote via Excel

**Como** admin, **quero** importar uma planilha Excel com múltiplos colaboradores de uma vez, **para** agilizar o onboarding de empresas novas sem cadastro manual.

**Prioridade:** P2
**Status:** Implementado no backend (`POST /api/colaborador/lote`); modal `ModalUploadExcelColaboradorComponent` existe na feature de Hierarquia — precisa ser exposto na tela principal de colaboradores

**Critérios de aceitação:**
- [ ] Template Excel disponível para download com colunas obrigatórias e opcionais documentadas
- [ ] Upload da planilha preenchida via modal
- [ ] Backend retorna: total enviado, criados com sucesso, falhas com descrição por linha
- [ ] Erros exibidos em tabela (linha, campo, motivo) para correção fácil
- [ ] Colaboradores com erro não bloqueiam os demais (tolerância a falhas)
- [ ] Sem suporte a avatar em importação em lote
- [ ] Apenas ADMIN pode importar

---

### US-COL-12 — Gestão de dispositivos vinculados

**Como** admin, **quero** visualizar e gerenciar os dispositivos associados a um colaborador, **para** identificar instalações duplicadas, dispositivos obsoletos ou problemas de comunicação com o agente.

**Prioridade:** P2
**Status:** Implementado (backend `GET` e `DELETE /dispositivos`; frontend `DispositivosColaboradorComponent` na aba Detalhes)

**Critérios de aceitação:**
- [ ] Lista de dispositivos: nome da máquina, SO, IP, versão do agente, data de instalação, último heartbeat
- [ ] Badge de status do agente: Ativo (< 10min), Sem comunicação (10–60min), Inativo (> 60min)
- [ ] Desvincular dispositivo com confirmação modal
- [ ] Após desvincular: dispositivo some da lista imediatamente
- [ ] Colaborador pode ter múltiplos dispositivos (notebook + desktop)
- [ ] Sem agentes: "Nenhum agente instalado. Baixe o instalador para começar o monitoramento."
- [ ] Apenas ADMIN pode desvincular dispositivos

---

## Regras transversais

### Multitenancy
- Todo colaborador pertence exclusivamente ao tenant identificado pelo CNPJ extraído do JWT
- Nenhuma operação atravessa tenants — validação obrigatória em todos os endpoints do `ColaboradorController`

### Controle de acesso por perfil

| Ação | ADMIN | MANAGER | VIEWER |
|------|-------|---------|--------|
| Listar colaboradores | Todos do tenant | Cadeia própria | Cadeia própria |
| Ver detalhe / Timeline / KPIs | Qualquer | Cadeia própria | Cadeia própria |
| Cadastrar / Editar | Sim | Não | Não |
| Ativar / Desativar | Sim | Não | Não |
| Excluir | Sim | Não | Não |
| Importar em lote | Sim | Não | Não |
| Gerenciar dispositivos | Sim | Não | Não |

### Observabilidade ética
- Sem keylogger, sem captura de tela, sem câmera
- Monitoramento baseado exclusivamente em: janela ativa (título do app), idle time, transições de sessão
- Esse posicionamento deve estar visível no onboarding do colaborador

---

## Backlog priorizado

### O que o gestor PRECISA para o MVP1 funcionar

| # | História | Prioridade | Status |
|---|----------|-----------|--------|
| 1 | US-COL-01 — Listagem com busca e filtros | P1 | Implementado |
| 2 | US-COL-02 — Cadastro de colaborador | P1 | Implementado |
| 3 | US-COL-03 — Edição de colaborador | P1 | Implementado |
| 4 | US-COL-04 — Ativar / Desativar | P1 | Implementado |
| 5 | US-COL-06 — Tela de detalhe (4 abas) | P1 | Implementado |
| 6 | US-COL-07 — Timeline de atividades | P1 | Implementado |
| 7 | US-COL-08 — KPIs de período | P1 | Implementado |
| 8 | US-C01 a US-C06 — Tela Unificada | P1 | Pendente |

### O que é diferencial competitivo (MVP1 avançado / MVP2 início)

| # | História | Prioridade | Dependência |
|---|----------|-----------|-------------|
| 9 | US-COL-10 — iManager Score (indicadores IA) | P1 | Relatórios Tasks 12–13 |
| 10 | US-COL-09 — Gestão de avatar | P2 | — |
| 11 | US-COL-12 — Gestão de dispositivos | P2 | — |

### O que pode esperar para MVP2

| # | História | Prioridade | Dependência |
|---|----------|-----------|-------------|
| 12 | US-COL-05 — Exclusão permanente (LGPD) | P2 | — |
| 13 | US-COL-11 — Importação em lote via Excel | P2 | — |
