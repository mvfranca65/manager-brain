# Design — Tours Contextuais
> Domínio: @Steve | PO
> Status: Documento de design — NÃO implementado
> Criado em: 2026-03-31
> Leitura obrigatória para: @Tony (TL), @Peter (dev), @Miles (dev), @Natasha (QA)

---

## Visão geral

O tour linear atual (16 passos percorrendo tudo de uma vez) é substituído por **8 tours independentes por seção**, cada um disparado contextualmente com base no estado real dos dados do usuário.

O objetivo é conduzir o gestor naturalmente pelas seções do produto à medida que ele as acessa pela primeira vez — sem forçar um caminho único e sem cansá-lo com uma jornada longa de onboarding.

**Princípios do novo sistema:**
- Máximo de 5–8 steps por tour
- Cada tour tem seu próprio ciclo de vida e flag de conclusão
- Progressão baseada em dados reais (times existem? colaboradores cadastrados?)
- Repetição sempre disponível via ícone `?` no header da página e em Configurações

---

## Tabela de flags do usuário

Todas as flags são armazenadas por usuário no banco de dados (tabela `user_tour_flags`).

| Flag | Tipo | Default | Quando muda para `true` |
|------|------|---------|------------------------|
| `tour_home_concluido` | boolean | false | Ao concluir o último step do tour-home |
| `tour_org_estrutura_concluido` | boolean | false | Ao concluir ou pular o tour-org-estrutura |
| `tour_org_times_concluido` | boolean | false | Ao concluir ou pular o tour-org-times |
| `tour_org_hierarquia_concluido` | boolean | false | Ao concluir ou pular o tour-org-hierarquia |
| `tour_colaboradores_concluido` | boolean | false | Ao concluir ou pular o tour-colaboradores |
| `tour_colaboradores_com_dados_concluido` | boolean | false | Ao concluir o tour-colaboradores quando já há dados reais |
| `tour_colaborador_detalhe_concluido` | boolean | false | Ao concluir ou pular o tour-colaborador-detalhe |
| `tour_relatorios_concluido` | boolean | false | Ao concluir ou pular o tour-relatorios |
| `tour_perfil_concluido` | boolean | false | Ao concluir ou pular o tour-perfil |
| `primeiro_login` | boolean | true | Vira `false` após o tour-home concluir |
| `menu_org_pulsando` | boolean | false | Vira `true` quando tour-home termina; `false` quando usuário clica em Organização |

---

## Fluxo de decisão geral (pseudocódigo)

```
ao entrar em /home:
  se primeiro_login == true:
    disparar tour-home (sem opção de pular)

ao entrar em /organizacao (qualquer tab):
  se tour_org_estrutura_concluido == false:
    navegar para tab Departamentos (se não estiver nela)
    disparar tour-org-estrutura

ao entrar em /organizacao com tab Times ativa:
  se tour_org_times_concluido == false:
    verificar se existe pelo menos 1 time cadastrado
    disparar tour-org-times (adapta steps conforme existência de times)

ao entrar em /organizacao com tab Gestão & Hierarquia ativa:
  se tour_org_hierarquia_concluido == false:
    disparar tour-org-hierarquia

ao entrar em /colaboradores:
  se tour_colaboradores_concluido == false:
    disparar tour-colaboradores (step 1 sempre aparece)
    verificar se existe pelo menos 1 time com colaboradores
    se não existir: tour para no step 1 e seta flag parcial
    se existir: tour completo

  senão, se tour_colaboradores_com_dados_concluido == false E colaboradores existem:
    disparar tour-colaboradores a partir do step 2 (agora com dados)

ao entrar em /colaboradores/membro/:id:
  se tour_colaborador_detalhe_concluido == false:
    disparar tour-colaborador-detalhe

ao entrar em /relatorios:
  se tour_relatorios_concluido == false:
    disparar tour-relatorios

ao entrar em /configuracoes/perfil:
  se tour_perfil_concluido == false:
    disparar tour-perfil
```

---

## Efeito "pulsando" no menu (Organização)

### Qual elemento recebe a animação

O item de menu **Organização** (`[data-tour="menu-organizacao"]`) recebe a classe CSS `menu-item--pulsando` enquanto a flag `menu_org_pulsando` for `true`.

### CSS da animação

```scss
@keyframes pulse-ring {
  0% {
    box-shadow: 0 0 0 0 rgba(99, 102, 241, 0.5); // azul-índigo com 50% opacidade
  }
  70% {
    box-shadow: 0 0 0 8px rgba(99, 102, 241, 0);
  }
  100% {
    box-shadow: 0 0 0 0 rgba(99, 102, 241, 0);
  }
}

.menu-item--pulsando {
  animation: pulse-ring 1.6s ease-out infinite;
  border-radius: 8px;
  position: relative;

  // Badge de ponto vermelho opcional (alternativa ao glow)
  &::after {
    content: '';
    position: absolute;
    top: 6px;
    right: 6px;
    width: 7px;
    height: 7px;
    background: #ef4444;
    border-radius: 50%;
    border: 1.5px solid white;
  }
}
```

### Quando desaparece

- Quando o usuário clica no menu Organização → `menu_org_pulsando` vira `false` → classe `menu-item--pulsando` é removida imediatamente.
- Se o usuário navegar diretamente para `/organizacao` por qualquer outro meio, a flag também é zerada.

---

## Os 8 tours

---

### 1. tour-home

**Descrição:** Boas-vindas ao Manager. Apresenta a estrutura da tela inicial e direciona o gestor para o próximo passo: configurar a organização.

**Condição de disparo:**
- Automático ao carregar `/home` quando `primeiro_login == true`
- Sem botão "Pular" em nenhum step — o tour é obrigatório uma única vez
- Flag controlada: `tour_home_concluido`, `primeiro_login`, `menu_org_pulsando`

**Steps:**

#### Step 1
```
element:     [data-tour="home-greeting"]
route:       /home
title:       👋 Bem-vindo ao Manager
description: Este é seu painel de comando. Aqui você tem uma visão rápida do que está
             acontecendo com seu time agora mesmo — <strong>quem está online, quem está
             ausente e quantos colaboradores estão monitorados</strong>.
side:        bottom
action:      nenhuma
```

#### Step 2
```
element:     [data-tour="home-stats"]
route:       /home
title:       📊 Seus números em tempo real
description: Esses cartões mostram o <strong>pulso do seu time agora</strong>. Total de
             colaboradores monitorados, quem está online e quem está ausente. Quando o
             agente estiver instalado nas máquinas, os dados aparecem aqui automaticamente.
side:        bottom
action:      nenhuma
```

#### Step 3
```
element:     [data-tour="home-quick-actions"]
route:       /home
title:       ⚡ Atalhos para as seções principais
description: Daqui você chega rápido em qualquer parte do Manager. <strong>Colaboradores</strong>
             para ver seu time, <strong>Relatórios</strong> para os diagnósticos semanais por IA
             e <strong>Organização</strong> para estruturar sua empresa.
side:        top
action:      nenhuma
```

#### Step 4
```
element:     [data-tour="menu-organizacao"]
route:       /home
title:       🏢 O primeiro passo é aqui
description: Antes de tudo, você precisa <strong>configurar sua organização</strong> — criar
             departamentos, cargos e times. Clique em <strong>Organização</strong> no menu
             para começar. Vamos guiar você por cada seção.
side:        right
action:      nenhuma (apenas aponta; o usuário clica quando quiser)
```

**Condição de saída:**
- Ao fechar o step 4 (botão "Entendi" ou "Concluir"), setar `tour_home_concluido = true`, `primeiro_login = false`, `menu_org_pulsando = true`.
- O menu Organização começa a pulsar.

---

### 2. tour-org-estrutura

**Descrição:** Apresenta as abas de Departamentos e Cargos — a base estrutural da organização.

**Condição de disparo:**
- Automático ao entrar em `/organizacao` quando `tour_org_estrutura_concluido == false`
- O sistema garante que a aba **Departamentos** está ativa antes de iniciar
- Botão "Pular" disponível em cada step
- Flags controladas: `tour_org_estrutura_concluido`

**Steps:**

#### Step 1
```
element:     [data-tour="tab-departamentos"]
route:       /organizacao
queryParams: tab=departamentos (forçar aba ativa)
title:       🏗️ Departamentos — a base da estrutura
description: Aqui você cria os <strong>departamentos da sua empresa</strong> — como TI,
             Comercial, Financeiro ou qualquer área que fizer sentido para você. Os
             colaboradores serão associados a um departamento depois.
side:        bottom
action:      nenhuma
```

#### Step 2
```
element:     [data-tour="tab-cargos"]
route:       /organizacao
title:       💼 Cargos — os papéis do seu time
description: Defina os <strong>cargos existentes na empresa</strong>, como Analista, Gerente
             ou Desenvolvedor. Isso permite filtrar relatórios e entender o desempenho por
             função — não apenas por pessoa.
side:        bottom
action:      nenhuma
```

#### Step 3
```
element:     [data-tour="tab-departamentos"]
route:       /organizacao
queryParams: tab=departamentos
title:       ✅ Crie seu primeiro departamento
description: Comece criando pelo menos <strong>um departamento</strong>. Você pode usar
             "Geral" para começar e refinar depois. Clique no botão <strong>"Novo
             departamento"</strong> dentro da aba para adicionar.
side:        bottom
action:      Crie um departamento (orientação, não bloqueante)
```

**Condição de saída:**
- Ao fechar o step 3 (botão "Entendi" ou "Pular"), setar `tour_org_estrutura_concluido = true`.
- O sistema não aguarda a criação do departamento para avançar — é uma orientação.

---

### 3. tour-org-times

**Descrição:** Apresenta a aba Times e pede para o gestor criar seu primeiro time antes de continuar.

**Condição de disparo:**
- Automático ao entrar em `/organizacao` com aba Times ativa, quando `tour_org_times_concluido == false`
- Ou disparado automaticamente após `tour_org_estrutura_concluido` virar `true` se o usuário ainda estiver na página
- Botão "Pular" disponível
- Flags controladas: `tour_org_times_concluido`

**Variações por estado de dados:**

| Cenário | Comportamento |
|---------|---------------|
| Nenhum time cadastrado | Tour completo — pede para criar o primeiro time |
| Já existe pelo menos 1 time | Tour encurtado — apresenta o card de time existente e explica as ações disponíveis |

**Steps (cenário: sem times):**

#### Step 1
```
element:     [data-tour="tab-times"]
route:       /organizacao
queryParams: tab=times
title:       👥 Times — o coração do monitoramento
description: <strong>Times são grupos de colaboradores</strong> que você vai monitorar
             juntos. Um time pode ser uma equipe, um projeto, um turno — o que fizer
             sentido para você. Tudo no Manager gira em torno dos times.
side:        bottom
action:      nenhuma
```

#### Step 2
```
element:     [data-tour="tab-times"]
route:       /organizacao
queryParams: tab=times
title:       🚀 Crie seu primeiro time agora
description: Você ainda não tem nenhum time cadastrado. Clique em <strong>"Novo
             time"</strong> para criar o primeiro. Sem times, não é possível monitorar
             colaboradores nem gerar relatórios.
side:        bottom
action:      Criar o primeiro time (orientação — o tour espera a ação ou aceita "Pular")
```

**Steps (cenário: já há times):**

#### Step 1
```
element:     [data-tour="tab-times"]
route:       /organizacao
queryParams: tab=times
title:       👥 Seus times estão aqui
description: Você já tem times criados. Aqui você pode <strong>editar, criar novos ou
             reorganizar</strong> os grupos de colaboradores da empresa. Cada time tem
             seu próprio painel de monitoramento e relatórios.
side:        bottom
action:      nenhuma
```

**Condição de saída:**
- Ao concluir ou pular qualquer variação, setar `tour_org_times_concluido = true`.

---

### 4. tour-org-hierarquia

**Descrição:** Apresenta o organograma, a ação de gestão de pessoas e o instalador do agente — o passo mais técnico do onboarding.

**Condição de disparo:**
- Automático ao entrar em `/organizacao` com aba Gestão & Hierarquia ativa, quando `tour_org_hierarquia_concluido == false`
- Visível apenas para usuários com perfil Admin (`ehAdmin == true`) — steps de botões admin só aparecem se o usuário tiver acesso
- Botão "Pular" disponível
- Flags controladas: `tour_org_hierarquia_concluido`

**Steps:**

#### Step 1
```
element:     [data-tour="tab-hierarquia"]
route:       /organizacao
queryParams: tab=hierarquia
title:       🌳 O organograma da sua empresa
description: Esta aba mostra <strong>como sua empresa está estruturada visualmente</strong>
             — quem responde a quem, por departamento. É aqui que você enxerga a cadeia
             de liderança de um jeito claro e organizado.
side:        bottom
action:      nenhuma
```

#### Step 2
```
element:     [data-tour="btn-gestao-pessoas"]
route:       /organizacao
queryParams: tab=hierarquia
title:       👤 Adicione pessoas à hierarquia
description: Com o botão <strong>"Novo usuário"</strong> você cadastra colaboradores e
             define o acesso deles ao portal. Cada pessoa precisa ser cadastrada aqui
             antes de aparecer nos relatórios e no monitoramento.
side:        left
action:      nenhuma (apenas apresenta o botão)
```

#### Step 3
```
element:     [data-tour="btn-download-instalador"]
route:       /organizacao
queryParams: tab=hierarquia
title:       💻 Instale o agente nas máquinas
description: O Manager precisa de um <strong>agente instalado no computador de cada
             colaborador</strong> para coletar dados de atividade. Baixe o instalador
             aqui e distribua para seu time — é rápido e não precisa de TI especializado.
side:        left
action:      Baixar e instalar o agente (orientação crítica — o gestor precisa fazer isso)
```

#### Step 4
```
element:     [data-tour="tab-hierarquia"]
route:       /organizacao
queryParams: tab=hierarquia
title:       ✅ Estrutura pronta, agora o time
description: Com departamentos, cargos, times e pessoas configurados, você está pronto
             para <strong>monitorar seu time de verdade</strong>. Vá em
             <strong>Colaboradores</strong> no menu para ver o painel do seu time.
side:        bottom
action:      nenhuma
```

**Condição de saída:**
- Ao concluir ou pular, setar `tour_org_hierarquia_concluido = true`.

---

### 5. tour-colaboradores

**Descrição:** Apresenta a tela unificada de colaboradores. Adapta-se conforme a existência de dados reais.

**Condição de disparo:**
- Automático ao entrar em `/colaboradores` quando `tour_colaboradores_concluido == false`
- **OU** quando `tour_colaboradores_concluido == true` mas `tour_colaboradores_com_dados_concluido == false` E existem colaboradores com dados
- Botão "Pular" disponível
- Flags controladas: `tour_colaboradores_concluido`, `tour_colaboradores_com_dados_concluido`

**Variações por estado de dados:**

| Cenário | Steps exibidos | Flag setada ao final |
|---------|----------------|----------------------|
| Sem times ou colaboradores cadastrados | Apenas steps 1 e 2 | `tour_colaboradores_concluido = true` |
| Com times mas sem dados de atividade | Steps 1 a 3 | `tour_colaboradores_concluido = true` |
| Com times e dados de atividade | Steps 1 a 5 (completo) | `tour_colaboradores_com_dados_concluido = true` |

**Steps:**

#### Step 1
```
element:     [data-tour="times-header"]
route:       /colaboradores
title:       👥 Painel de Colaboradores
description: Aqui você acompanha <strong>todos os seus colaboradores em tempo real</strong>.
             Selecione um time no campo abaixo para ver quem está online, ausente ou
             offline — tudo em um único lugar.
side:        bottom
action:      nenhuma
```

#### Step 2 (aparece quando não há times ou colaboradores)
```
element:     [data-tour="times-select-card"]
route:       /colaboradores
title:       ⚠️ Nenhum time encontrado ainda
description: Para ver seus colaboradores aqui, você precisa primeiro <strong>criar um
             time em Organização</strong> e adicionar colaboradores a ele. Quando isso
             estiver feito, volte aqui e o painel estará populado.
side:        bottom
action:      nenhuma (tour para aqui neste cenário)
```

#### Step 2-alt (aparece quando há times — substitui o step 2 acima)
```
element:     [data-tour="times-select-card"]
route:       /colaboradores
title:       🔍 Selecione um time para começar
description: Use esse campo para <strong>escolher qual time você quer visualizar</strong>.
             Cada time tem seu próprio painel com status, scores e indicadores dos membros.
             Você pode alternar entre times a qualquer momento.
side:        bottom
action:      Selecionar um time no dropdown
```

#### Step 3 (aparece quando há times mas ainda sem dados de atividade)
```
element:     [data-tour="times-select-card"]
route:       /colaboradores
title:       ⏳ Aguardando dados do agente
description: O time está configurado, mas o <strong>agente ainda não enviou dados de
             atividade</strong>. Assim que os colaboradores estiverem com o agente
             instalado e ativo, os dados aparecem aqui automaticamente.
side:        bottom
action:      nenhuma (tour para aqui neste cenário)
```

#### Step 4 (aparece quando há dados reais — novo selector necessário)
```
element:     [data-tour="colaboradores-lista"]
route:       /colaboradores
title:       📋 Seu time em detalhes
description: Cada card de colaborador mostra o <strong>status atual, o IManager Score
             da semana e os indicadores de saúde</strong> como burnout e desengajamento.
             Clique em qualquer colaborador para ver o perfil completo dele.
side:        top
action:      nenhuma

NOVO SELECTOR NECESSÁRIO: adicionar data-tour="colaboradores-lista" no container
da lista de cards de colaboradores dentro de times-dashboard.page.html
```

#### Step 5 (aparece quando há dados reais)
```
element:     [data-tour="colaboradores-lista"]
route:       /colaboradores
title:       🎯 Clique num colaborador para mergulhar fundo
description: O painel individual mostra a <strong>linha do tempo de atividade, KPIs
             de produtividade e os indicadores psicológicos</strong> daquele colaborador.
             É onde você vai quando precisa entender o que está acontecendo com alguém
             do time.
side:        top
action:      Clique em um colaborador (orientação para avançar no fluxo natural)
```

**Condição de saída:**
- Cenário sem dados: `tour_colaboradores_concluido = true`
- Cenário com dados (completo): `tour_colaboradores_concluido = true` + `tour_colaboradores_com_dados_concluido = true`

---

### 6. tour-colaborador-detalhe

**Descrição:** Apresenta os KPIs, linha do tempo e indicadores psicológicos do perfil individual do colaborador.

**Condição de disparo:**
- Automático ao entrar em `/colaboradores/membro/:id` quando `tour_colaborador_detalhe_concluido == false`
- Pré-condição: deve existir pelo menos 1 semana de dados para o colaborador (senão o tour não dispara — não há o que mostrar)
- Botão "Pular" disponível
- Flags controladas: `tour_colaborador_detalhe_concluido`

**Steps:**

#### Step 1 (novo selector necessário)
```
element:     [data-tour="detalhe-score-card"]
route:       /colaboradores/membro/:id
title:       🏆 O IManager Score
description: Esse número resume a <strong>semana de trabalho do colaborador em 0 a
             100</strong>, calculado com base em 6 pilares como atividade, foco e
             consistência. A IA interpreta o score — você não precisa fazer contas.
side:        bottom
action:      nenhuma

NOVO SELECTOR NECESSÁRIO: adicionar data-tour="detalhe-score-card" no card de
score/KPIs principal dentro de colaborador-detalhe
```

#### Step 2 (novo selector necessário)
```
element:     [data-tour="detalhe-timeline"]
route:       /colaboradores/membro/:id
title:       📅 A linha do tempo de atividade
description: Aqui você vê <strong>como o colaborador distribuiu sua atividade ao longo
             da semana</strong> — hora a hora, dia a dia. Padrões de pico, ociosidade e
             consistência ficam visíveis de um jeito que nenhum relatório tradicional
             mostra.
side:        top
action:      nenhuma

NOVO SELECTOR NECESSÁRIO: adicionar data-tour="detalhe-timeline" no componente de
timeline/gráfico de atividade dentro de colaborador-detalhe
```

#### Step 3 (novo selector necessário)
```
element:     [data-tour="detalhe-indicadores"]
route:       /colaboradores/membro/:id
title:       🧠 Indicadores de saúde e risco
description: Esta seção detecta sinais como <strong>risco de burnout, desengajamento
             silencioso e sobrecarga</strong> antes que virem problemas sérios. Não é
             vigilância — é cuidado baseado em dados objetivos, sem keylogger ou câmera.
side:        top
action:      nenhuma

NOVO SELECTOR NECESSÁRIO: adicionar data-tour="detalhe-indicadores" no card de
indicadores psicológicos dentro de colaborador-detalhe
```

#### Step 4 (novo selector necessário)
```
element:     [data-tour="detalhe-relatorio-link"]
route:       /colaboradores/membro/:id
title:       📄 Ver o relatório semanal completo
description: Quer ir além dos números? O <strong>relatório semanal por IA</strong> traz
             um diagnóstico em linguagem natural — o que aconteceu na semana, o que
             preocupa e o que o gestor pode fazer. Acesse em Relatórios no menu lateral.
side:        top
action:      nenhuma

NOVO SELECTOR NECESSÁRIO: adicionar data-tour="detalhe-relatorio-link" no botão ou
link que aponta para os relatórios dentro do detalhe do colaborador (se existir);
caso não exista ainda, o step aponta para o botão de voltar à lista ou elemento
contextual equivalente
```

**Condição de saída:**
- Ao concluir ou pular, setar `tour_colaborador_detalhe_concluido = true`.

---

### 7. tour-relatorios

**Descrição:** Apresenta a tela de relatórios semanais por IA — o diferencial central do produto.

**Condição de disparo:**
- Automático ao entrar em `/relatorios` quando `tour_relatorios_concluido == false`
- Botão "Pular" disponível
- Flags controladas: `tour_relatorios_concluido`

**Variações por estado de dados:**

| Cenário | Comportamento |
|---------|---------------|
| Sem relatórios disponíveis (sem dados ou sem semana completa) | Tour apresenta o header e explica o que o gestor vai encontrar aqui em breve |
| Com relatórios disponíveis | Tour completo mostrando filtros, navegação de semana e cards de relatório |

**Steps:**

#### Step 1
```
element:     [data-tour="relatorios-header"]
route:       /relatorios
title:       📊 Relatórios Semanais por IA
description: Esta é a seção mais poderosa do Manager. Toda semana, a <strong>IA gera
             um diagnóstico completo por colaborador e por time</strong> — o que aconteceu,
             quem precisa de atenção e o que você pode fazer como gestor.
side:        bottom
action:      nenhuma
```

#### Step 2 (novo selector necessário)
```
element:     [data-tour="relatorios-filtro-time"]
route:       /relatorios
title:       🔍 Filtre por time ou colaborador
description: Escolha <strong>qual time você quer analisar</strong>. Depois clique em
             qualquer colaborador do time para ver o relatório individual dele. O Manager
             organiza automaticamente os dados por hierarquia.
side:        bottom
action:      Selecionar um time (orientação)

NOVO SELECTOR NECESSÁRIO: adicionar data-tour="relatorios-filtro-time" no componente
app-filtro-relatorio dentro de relatorios-page.component.html
```

#### Step 3 (novo selector necessário)
```
element:     [data-tour="relatorios-semana-nav"]
route:       /relatorios
title:       📅 Navegue pelas semanas anteriores
description: Use as setas para <strong>acessar relatórios de semanas passadas</strong>.
             O Manager guarda o histórico completo para você comparar evolução e
             acompanhar tendências ao longo do tempo.
side:        bottom
action:      nenhuma

NOVO SELECTOR NECESSÁRIO: adicionar data-tour="relatorios-semana-nav" no componente
app-semana-navigator dentro de relatorios-page.component.html
```

#### Step 4 (aparece apenas quando há relatórios disponíveis — novo selector necessário)
```
element:     [data-tour="relatorios-cards-area"]
route:       /relatorios
title:       🤖 O diagnóstico completo da semana
description: Cada card traz o <strong>IManager Score, os 6 pilares de desempenho,
             os indicadores psicológicos e as recomendações da IA</strong> para aquele
             colaborador. É a visão que o gestor precisa para tomar decisões com
             confiança — não intuição.
side:        top
action:      nenhuma

NOVO SELECTOR NECESSÁRIO: adicionar data-tour="relatorios-cards-area" no container
principal dos cards de relatório dentro de relatorios-page.component.html
```

**Condição de saída:**
- Ao concluir ou pular, setar `tour_relatorios_concluido = true`.

---

### 8. tour-perfil

**Descrição:** Apresenta a seção de perfil — dados pessoais e segurança da conta do gestor.

**Condição de disparo:**
- Automático ao entrar em `/configuracoes/perfil` quando `tour_perfil_concluido == false`
- Botão "Pular" disponível
- Flags controladas: `tour_perfil_concluido`

**Steps:**

#### Step 1
```
element:     [data-tour="perfil-card"]
route:       /configuracoes/perfil
title:       👤 Suas informações pessoais
description: Aqui ficam os seus dados de perfil — <strong>nome, e-mail e informações
             da sua conta</strong>. Mantenha tudo atualizado para garantir que as
             notificações e relatórios cheguem corretamente.
side:        bottom
action:      nenhuma
```

#### Step 2
```
element:     [data-tour="perfil-senha"]
route:       /configuracoes/perfil
title:       🔐 Mantenha sua conta segura
description: Troque sua senha regularmente e use uma <strong>combinação forte com
             letras, números e símbolos</strong>. Sua conta dá acesso a dados sensíveis
             do seu time — segurança é inegociável.
side:        top
action:      nenhuma
```

#### Step 3 (novo selector necessário)
```
element:     [data-tour="configuracoes-tours"]
route:       /configuracoes/perfil
title:       🔁 Você pode rever qualquer tour
description: Em <strong>Configurações</strong>, na seção "Tours e Onboarding", você
             encontra a lista de todos os tours disponíveis e pode rever qualquer um
             deles a qualquer momento. O ícone <strong>?</strong> no topo de cada
             página também funciona para isso.
side:        top
action:      nenhuma

NOVO SELECTOR NECESSÁRIO: adicionar data-tour="configuracoes-tours" na seção de
tours dentro da tela de configurações (seção a ser criada — ver abaixo)
```

**Condição de saída:**
- Ao concluir ou pular, setar `tour_perfil_concluido = true`.

---

## Seção "Rever tours" nas Configurações

### Localização
Adicionar uma nova seção na tela `/configuracoes/perfil` (ou numa aba futura de Configurações gerais), abaixo do card de segurança.

### Lista de tours com nome amigável

| Nome amigável | Identificador interno | Rota para disparar |
|---------------|----------------------|--------------------|
| Visão geral do Manager | `tour-home` | `/home` |
| Departamentos e Cargos | `tour-org-estrutura` | `/organizacao?tab=departamentos` |
| Criação de Times | `tour-org-times` | `/organizacao?tab=times` |
| Organograma e Instalador | `tour-org-hierarquia` | `/organizacao?tab=hierarquia` |
| Painel de Colaboradores | `tour-colaboradores` | `/colaboradores` |
| Perfil do Colaborador | `tour-colaborador-detalhe` | `/colaboradores` (acessa um membro) |
| Relatórios Semanais por IA | `tour-relatorios` | `/relatorios` |
| Meu Perfil e Segurança | `tour-perfil` | `/configuracoes/perfil` |

### Comportamento do botão "Rever"
1. Ao clicar em "Rever", o sistema:
   - Zera a flag correspondente do tour (ex: `tour_relatorios_concluido = false`)
   - Navega para a rota correta do tour
   - O tour dispara automaticamente na rota de destino
2. O botão "Rever" não reinicia flags de outros tours — cada tour é independente.
3. Quando o tour está sendo revisto, o botão "Pular" está disponível (mesmo que o tour original fosse obrigatório, como o tour-home).

### Mockup da seção

```
┌────────────────────────────────────────────────────────┐
│ 🧭 Tours e Onboarding                                  │
│ Reveja qualquer seção do guia a qualquer momento.      │
├────────────────────────────────────────────────────────┤
│ Visão geral do Manager              [Rever tour →]     │
│ Departamentos e Cargos              [Rever tour →]     │
│ Criação de Times                    [Rever tour →]     │
│ Organograma e Instalador            [Rever tour →]     │
│ Painel de Colaboradores             [Rever tour →]     │
│ Perfil do Colaborador               [Rever tour →]     │
│ Relatórios Semanais por IA          [Rever tour →]     │
│ Meu Perfil e Segurança              [Rever tour →]     │
└────────────────────────────────────────────────────────┘
```

---

## Ícone `?` no header de cada página

### Comportamento
- Cada página que tem um tour associado exibe um ícone `?` (ou `pi-question-circle`) no canto superior direito do header da página.
- Ao clicar: zera a flag do tour correspondente e dispara o tour da página atual imediatamente (sem navegação).
- Tooltip ao hover: "Rever o tour desta seção"

### Onde adicionar
Adicionar ao componente `app-header` um `@Input() tourId: string` opcional. Quando preenchido, o ícone `?` aparece. O componente emite um evento que o `TourService` intercepta para disparar o tour correto.

```html
<!-- Exemplo de uso no template -->
<app-header
  titulo="Relatórios"
  subtitulo="Diagnósticos semanais por IA"
  tourId="tour-relatorios">
</app-header>
```

---

## Novos selectors `data-tour` necessários

Lista consolidada de todos os selectors que precisam ser **adicionados ao HTML** (não existem ainda):

| Selector | Arquivo | Contexto |
|----------|---------|---------|
| `data-tour="colaboradores-lista"` | `times-dashboard.page.html` | Container da lista de cards de colaboradores |
| `data-tour="detalhe-score-card"` | `colaborador-detalhe` (componente principal) | Card com IManager Score e KPIs |
| `data-tour="detalhe-timeline"` | `colaborador-detalhe` (componente principal) | Componente de gráfico/timeline de atividade |
| `data-tour="detalhe-indicadores"` | `colaborador-detalhe` (componente principal) | Card de indicadores psicológicos |
| `data-tour="detalhe-relatorio-link"` | `colaborador-detalhe` (componente principal) | Link/botão para relatórios individuais |
| `data-tour="relatorios-filtro-time"` | `relatorios-page.component.html` | Componente `app-filtro-relatorio` |
| `data-tour="relatorios-semana-nav"` | `relatorios-page.component.html` | Componente `app-semana-navigator` |
| `data-tour="relatorios-cards-area"` | `relatorios-page.component.html` | Container dos cards de relatório |
| `data-tour="configuracoes-tours"` | Seção nova em `configuracoes/perfil` | Seção "Tours e Onboarding" |

---

## Selectors existentes reutilizados

Lista dos selectors do tour atual que continuam sendo usados no novo sistema:

| Selector | Arquivo | Tour que usa |
|----------|---------|-------------|
| `data-tour="home-greeting"` | `dashboard.component.html` | tour-home (step 1) |
| `data-tour="home-stats"` | `dashboard.component.html` | tour-home (step 2) |
| `data-tour="home-quick-actions"` | `dashboard.component.html` | tour-home (step 3) |
| `data-tour="menu-organizacao"` | `menu.component.html` | tour-home (step 4) |
| `data-tour="tab-departamentos"` | `hierarquia-gestao-page.component.html` | tour-org-estrutura (steps 1, 3) |
| `data-tour="tab-cargos"` | `hierarquia-gestao-page.component.html` | tour-org-estrutura (step 2) |
| `data-tour="tab-times"` | `hierarquia-gestao-page.component.html` | tour-org-times (steps 1, 2) |
| `data-tour="tab-hierarquia"` | `hierarquia-gestao-page.component.html` | tour-org-hierarquia (steps 1, 4) |
| `data-tour="btn-gestao-pessoas"` | `aba-hierarquia.component.html` | tour-org-hierarquia (step 2) |
| `data-tour="btn-download-instalador"` | `aba-hierarquia.component.html` | tour-org-hierarquia (step 3) |
| `data-tour="times-header"` | `times-dashboard.page.html` | tour-colaboradores (step 1) |
| `data-tour="times-select-card"` | `times-dashboard.page.html` | tour-colaboradores (steps 2, 3) |
| `data-tour="relatorios-header"` | `relatorios-page.component.html` | tour-relatorios (step 1) |
| `data-tour="perfil-card"` | `perfil.component.html` | tour-perfil (step 1) |
| `data-tour="perfil-senha"` | `perfil.component.html` | tour-perfil (step 2) |

> Selectors do tour atual que **não são reutilizados** no novo sistema:
> - `data-tour="menu-times"` — rota `/times` foi redirecionada para `/colaboradores`
> - `data-tour="menu-relatorios"` — não há step apontando para o menu de relatórios
> - `data-tour="menu-perfil"` — não há step apontando para o menu de perfil

---

## Resumo de steps por tour

| Tour | Steps mínimos | Steps máximos | Obrigatório |
|------|:---:|:---:|:---:|
| tour-home | 4 | 4 | Sim (sem pular) |
| tour-org-estrutura | 3 | 3 | Não |
| tour-org-times | 1 | 2 | Não |
| tour-org-hierarquia | 4 | 4 | Não |
| tour-colaboradores | 2 | 5 | Não |
| tour-colaborador-detalhe | 4 | 4 | Não |
| tour-relatorios | 3 | 4 | Não |
| tour-perfil | 3 | 3 | Não |

---

## Notas para implementação

1. **Ordem de tours sugerida para o desenvolvedor:**
   `TourService` → `TourFlagService` (HTTP) → guards/resolvers nas rotas → componente de popover → animação do menu

2. **A biblioteca de tour existente** (sea qual for a atual) pode manter sua API de steps — o que muda é a lógica de disparo e o controle de flags por backend.

3. **Flags no backend:** criar endpoint `PATCH /api/v1/tour-flags` que recebe `{ flagName: string, value: boolean }` e valida `tenant_id` + `user_id` — nunca misturar flags entre usuários do mesmo tenant.

4. **Verificação de dados reais:** o `TourService` deve consultar o backend antes de disparar tours adaptativos (ex: "tem times?", "tem colaboradores?"). Usar observables com `combineLatest` para não bloquear a UI.

5. **Acessibilidade:** todos os popovers do tour devem ter `role="dialog"`, `aria-label` descritivo e foco gerenciado para teclado.

---

## Personalização por Perfil de Acesso

> Status: **Estrutura preparada — MVP1 usa o fluxo ADMIN para todos.**
> Personalização real por perfil está planejada para uma iteração futura (pós-MVP1).

Os 3 perfis do sistema:
- **ADMIN** — configura empresa, vê tudo, acessa Organização
- **MANAGER** — acompanha times da sua hierarquia, sem acesso a Organização
- **VIEWER** — vê apenas seus próprios dados (perfil futuro, sem tela dedicada ainda)

---

### A. Matriz de visibilidade por perfil

| Tour | ADMIN | MANAGER | VIEWER | Diferenças |
|------|:-----:|:-------:|:------:|------------|
| tour-home | Sim | Sim | Sim | Step 4 muda completamente: ADMIN → Organização; MANAGER → Colaboradores; VIEWER → Perfil |
| tour-org-estrutura | Sim | Não | Não | MANAGER/VIEWER não têm acesso à rota `/organizacao` |
| tour-org-times | Sim | Não | Não | Idem — rota restrita |
| tour-org-hierarquia | Sim | Não | Não | Idem — rota restrita |
| tour-colaboradores | Sim | Sim | Não | MANAGER vê "seus times" em vez de "todos os times"; VIEWER não tem acesso à lista |
| tour-colaborador-detalhe | Sim | Sim | Sim (só o próprio) | VIEWER vê "sua produtividade" em vez de "do colaborador"; MANAGER vê time da hierarquia |
| tour-relatorios | Sim | Sim | Não | MANAGER vê relatórios dos seus times; VIEWER não tem acesso |
| tour-perfil | Sim | Sim | Sim | Mesmos steps para os 3 perfis |

**Resumo por perfil:**
- **ADMIN:** todos os 8 tours
- **MANAGER:** tour-home + tour-colaboradores + tour-colaborador-detalhe + tour-relatorios + tour-perfil (5 tours)
- **VIEWER:** tour-home + tour-colaborador-detalhe (o próprio) + tour-perfil (3 tours)

---

### B. Steps alternativos por perfil

Os tours abaixo têm variações de conteúdo por perfil. Os demais tours mantêm os mesmos steps independentemente de quem está logado.

---

#### tour-home — variante MANAGER

**Descrição:** Boas-vindas ao Manager para o gestor de time. Apresenta o painel e direciona para Colaboradores (não para Organização).

**Flags controladas:** `tour_home_concluido`, `primeiro_login`, `menu_colaboradores_pulsando`

##### Step 1
```
element:     [data-tour="home-greeting"]
route:       /home
title:       👋 Bem-vindo ao Manager
description: Este é seu painel de acompanhamento. Aqui você tem uma visão rápida do
             que está acontecendo com <strong>seus times agora mesmo</strong> — quem
             está online, quem está ausente e o resumo de produtividade.
side:        bottom
```

##### Step 2
```
element:     [data-tour="home-stats"]
route:       /home
title:       📊 O pulso dos seus times
description: Esses cartões mostram os <strong>indicadores em tempo real dos seus
             colaboradores</strong>. Total monitorado, online e ausente. Os dados
             aparecem automaticamente quando o agente estiver ativo nas máquinas.
side:        bottom
```

##### Step 3
```
element:     [data-tour="home-quick-actions"]
route:       /home
title:       ⚡ Suas seções principais
description: Daqui você chega rápido em <strong>Colaboradores</strong> para ver
             seus times, e em <strong>Relatórios</strong> para os diagnósticos
             semanais por IA. Esses são os seus dois painéis centrais.
side:        top
```

##### Step 4
```
element:     [data-tour="menu-colaboradores"]
route:       /home
title:       👥 Comece pelos seus colaboradores
description: O seu primeiro passo é ir em <strong>Colaboradores</strong> no menu.
             Lá você vê o status em tempo real dos seus times, acessa o perfil
             individual de cada um e acompanha os indicadores de saúde e produtividade.
side:        right
action:      nenhuma (apenas aponta; o usuário clica quando quiser)
```

**Condição de saída (MANAGER):** Ao fechar o step 4, setar `tour_home_concluido = true`, `primeiro_login = false`, `menu_colaboradores_pulsando = true`.

---

#### tour-home — variante VIEWER

**Descrição:** Boas-vindas ao Manager para o próprio colaborador. Apresenta o painel e direciona para o próprio perfil de produtividade.

**Flags controladas:** `tour_home_concluido`, `primeiro_login`

##### Step 1
```
element:     [data-tour="home-greeting"]
route:       /home
title:       👋 Bem-vindo ao Manager
description: Este é o seu painel pessoal. Aqui você acompanha
             <strong>sua própria produtividade</strong> — o que o sistema registra
             sobre seu trabalho, seus indicadores da semana e seu histórico.
side:        bottom
```

##### Step 2
```
element:     [data-tour="home-stats"]
route:       /home
title:       📊 Seus números da semana
description: Esses cartões mostram um resumo do <strong>que o Manager registrou
             sobre você</strong>. Os dados são coletados pelo agente instalado no
             seu computador — sem keylogger, sem câmera, só atividade de trabalho.
side:        bottom
```

##### Step 3
```
element:     [data-tour="home-quick-actions"]
route:       /home
title:       ⚡ O que você pode acessar
description: A partir daqui você acessa <strong>seu perfil de produtividade</strong>
             com sua linha do tempo, seu IManager Score e seus indicadores. Transparência
             total sobre o que o sistema sabe sobre você.
side:        top
```

##### Step 4
```
element:     [data-tour="menu-perfil"]
route:       /home
title:       👤 Veja sua produtividade
description: Clique em <strong>Perfil</strong> no menu para acessar seu painel
             individual — sua linha do tempo de atividade, seu score da semana e
             os indicadores que o sistema calcula para você.
side:        right
action:      nenhuma (apenas aponta; o usuário clica quando quiser)
```

**Condição de saída (VIEWER):** Ao fechar o step 4, setar `tour_home_concluido = true`, `primeiro_login = false`.

---

#### tour-colaboradores — variante MANAGER

**Descrição:** Apresenta o painel de colaboradores com foco em "seus times" — não todos os times da empresa, só os da hierarquia do gestor.

##### Step 1
```
element:     [data-tour="times-header"]
route:       /colaboradores
title:       👥 Seus Times
description: Aqui você acompanha <strong>os colaboradores dos seus times em tempo
             real</strong>. Selecione um time no campo abaixo para ver quem está
             online, ausente ou offline.
side:        bottom
```

##### Step 2 (sem times na hierarquia do MANAGER)
```
element:     [data-tour="times-select-card"]
route:       /colaboradores
title:       ⚠️ Nenhum time atribuído ainda
description: Você ainda não tem times vinculados à sua hierarquia. Fale com o
             <strong>administrador da empresa</strong> para que seus times sejam
             configurados — assim que isso acontecer, o painel estará pronto.
side:        bottom
```

##### Step 2-alt (com times na hierarquia do MANAGER)
```
element:     [data-tour="times-select-card"]
route:       /colaboradores
title:       🔍 Escolha qual time visualizar
description: Use esse campo para <strong>selecionar um dos seus times</strong>.
             Cada time tem seu próprio painel com status, scores e indicadores.
             Você alterna entre times a qualquer momento.
side:        bottom
```

##### Steps 3 a 5
Os mesmos do tour base (aguardando dados do agente, lista de cards, clique no colaborador) — sem alteração de conteúdo.

---

#### tour-colaborador-detalhe — variante VIEWER

**Descrição:** O VIEWER vê seu próprio perfil. A linguagem muda para "você" em vez de "o colaborador". Os selectors são os mesmos — a rota é `/colaboradores/membro/:id` onde `:id` é o próprio usuário.

##### Step 1
```
element:     [data-tour="detalhe-score-card"]
route:       /colaboradores/membro/:id
title:       🏆 Seu IManager Score
description: Esse número resume <strong>sua semana de trabalho em 0 a 100</strong>,
             calculado com base em 6 pilares como atividade, foco e consistência.
             A IA interpreta o score — você não precisa fazer contas.
side:        bottom
```

##### Step 2
```
element:     [data-tour="detalhe-timeline"]
route:       /colaboradores/membro/:id
title:       📅 Sua linha do tempo de atividade
description: Aqui você vê <strong>como você distribuiu sua atividade ao longo da
             semana</strong> — hora a hora, dia a dia. Você pode identificar seus
             próprios padrões de foco, pico de trabalho e momentos de baixa.
side:        top
```

##### Step 3
```
element:     [data-tour="detalhe-indicadores"]
route:       /colaboradores/membro/:id
title:       🧠 Seus indicadores de saúde
description: Esta seção mostra sinais como <strong>risco de burnout, desengajamento
             e sobrecarga</strong> com base nos seus padrões de atividade. É uma
             ferramenta de autocuidado — dados objetivos sobre como você está.
side:        top
```

##### Step 4
```
element:     [data-tour="detalhe-relatorio-link"]
route:       /colaboradores/membro/:id
title:       📄 Seu relatório semanal completo
description: Quer entender mais? O <strong>relatório semanal por IA</strong> traz
             um diagnóstico em linguagem natural sobre a sua semana — o que foi bem,
             o que merece atenção e como você pode se organizar melhor.
side:        top
```

---

### C. Estrutura técnica sugerida

A abordagem recomendada é **definição centralizada com overrides por perfil**, evitando duplicação de steps inteiros quando só o texto muda.

```typescript
type PortalRole = 'ADMIN' | 'MANAGER' | 'VIEWER';

interface TourStep {
  element: string;
  route: string;
  queryParams?: Record<string, string>;
  title: string;
  description: string;
  side: 'top' | 'bottom' | 'left' | 'right';
  action?: string;
}

interface TourDefinition {
  id: string;
  visibleTo: PortalRole[];           // quais perfis disparam este tour
  steps: TourStep[];                  // steps base (ADMIN ou default)
  overrides?: {
    MANAGER?: Partial<TourStep>[];    // sobrescreve campos dos steps por índice
    VIEWER?: Partial<TourStep>[];     // null num índice = usar step base; objeto = merge
  };
}
```

**Regra de merge:** `overrides.MANAGER[i]` faz um merge raso (`Object.assign`) sobre `steps[i]`. Se o índice não existir no override (ou for `null`), usa o step base. Se o override tiver `null` como valor explícito para um índice, o step é **omitido** para esse perfil.

**Exemplo de uso para tour-home:**

```typescript
const tourHome: TourDefinition = {
  id: 'tour-home',
  visibleTo: ['ADMIN', 'MANAGER', 'VIEWER'],
  steps: [
    // steps base = versão ADMIN (steps 1 a 4 conforme documento base)
    { element: '[data-tour="home-greeting"]', title: '👋 Bem-vindo ao Manager', ... },
    { element: '[data-tour="home-stats"]',    title: '📊 Seus números em tempo real', ... },
    { element: '[data-tour="home-quick-actions"]', title: '⚡ Atalhos para as seções principais', ... },
    { element: '[data-tour="menu-organizacao"]',   title: '🏢 O primeiro passo é aqui', ... },
  ],
  overrides: {
    MANAGER: [
      { description: 'Este é seu painel de acompanhamento. Aqui você tem uma visão rápida do que está acontecendo com seus times agora mesmo...' },
      { title: '📊 O pulso dos seus times', description: 'Esses cartões mostram os indicadores em tempo real dos seus colaboradores...' },
      { description: 'Daqui você chega rápido em Colaboradores para ver seus times, e em Relatórios para os diagnósticos semanais por IA...' },
      { element: '[data-tour="menu-colaboradores"]', title: '👥 Comece pelos seus colaboradores', description: 'O seu primeiro passo é ir em Colaboradores no menu...' },
    ],
    VIEWER: [
      { description: 'Este é o seu painel pessoal. Aqui você acompanha sua própria produtividade...' },
      { title: '📊 Seus números da semana', description: 'Esses cartões mostram um resumo do que o Manager registrou sobre você...' },
      { description: 'A partir daqui você acessa seu perfil de produtividade com sua linha do tempo...' },
      { element: '[data-tour="menu-perfil"]', title: '👤 Veja sua produtividade', description: 'Clique em Perfil no menu para acessar seu painel individual...' },
    ],
  },
};
```

**Por que não criar 3 arquivos separados de steps:** a maioria dos steps muda apenas o texto de `description` ou `title`. Manter tudo em um único objeto com overrides facilita manutenção — se o selector mudar, muda em um lugar só.

**Quando criar um tour completamente separado:** se mais de 60% dos steps de um perfil forem diferentes (estrutura diferente, não só texto), crie um `TourDefinition` próprio com `id: 'tour-home-viewer'` e registre-o no `TourRegistry` como alternativo para aquele perfil.

---

### D. Fluxo de decisão atualizado (pseudocódigo com perfil)

```
ao entrar em /home:
  perfil = usuario.perfil                          // 'ADMIN' | 'MANAGER' | 'VIEWER'
  se primeiro_login == true:
    steps = resolver_steps('tour-home', perfil)    // aplica overrides conforme perfil
    disparar tour-home com steps resolvidos (sem opção de pular)
    ao concluir:
      setar tour_home_concluido = true
      setar primeiro_login = false
      se perfil == 'ADMIN':    setar menu_org_pulsando = true
      se perfil == 'MANAGER':  setar menu_colaboradores_pulsando = true
      // VIEWER: nenhum item pulsa

ao entrar em /organizacao (qualquer tab):
  se perfil != 'ADMIN':
    redirecionar para /home (acesso negado — não exibir tour)
  senão:
    fluxo normal dos tours org (conforme documento base)

ao entrar em /colaboradores:
  se perfil == 'VIEWER':
    redirecionar para /colaboradores/membro/:id-proprio (ou /home)
  senão:
    steps = resolver_steps('tour-colaboradores', perfil)  // ADMIN = base, MANAGER = override
    fluxo normal de disparo conforme flags

ao entrar em /colaboradores/membro/:id:
  se perfil == 'VIEWER' E id != usuario.id:
    redirecionar para /colaboradores/membro/:id-proprio  (VIEWER só vê o próprio)
  steps = resolver_steps('tour-colaborador-detalhe', perfil)
  fluxo normal conforme flags

ao entrar em /relatorios:
  se perfil == 'VIEWER':
    redirecionar para /home (acesso negado)
  senão:
    fluxo normal do tour-relatorios (sem variação de steps por perfil)

ao entrar em /configuracoes/perfil:
  fluxo normal do tour-perfil (sem variação por perfil)

// Função auxiliar
resolver_steps(tourId, perfil):
  tour = TourRegistry.get(tourId)
  se perfil == 'ADMIN' OU tour.overrides[perfil] == undefined:
    retornar tour.steps
  retornar tour.steps.map((step, i) =>
    tour.overrides[perfil][i] == null
      ? OMITIR_STEP
      : Object.assign({}, step, tour.overrides[perfil][i] ?? {})
  )
```

---

### E. Menu pulsando por perfil

A animação de "menu pulsando" é o CTA visual pós-tour-home — direciona para onde o perfil deve ir primeiro.

| Perfil | Item que pulsa | Flag controlada | Para de pulsar quando |
|--------|---------------|-----------------|----------------------|
| ADMIN | Organização (`[data-tour="menu-organizacao"]`) | `menu_org_pulsando` | Usuário clica em Organização |
| MANAGER | Colaboradores (`[data-tour="menu-colaboradores"]`) | `menu_colaboradores_pulsando` | Usuário clica em Colaboradores |
| VIEWER | Nenhum item pulsa | — | — |

> **VIEWER sem pulsação:** o VIEWER já é direcionado pelo step 4 do tour-home para Perfil. Como não há ação urgente de configuração, não pulsar é a escolha certa — evita ansiedade sem propósito.

**Nova flag necessária no banco:**

| Flag | Tipo | Default | Quando muda para `true` | Quando volta para `false` |
|------|------|---------|------------------------|--------------------------|
| `menu_colaboradores_pulsando` | boolean | false | Após tour-home concluído por MANAGER | Quando MANAGER clica em Colaboradores |

Adicionar à tabela de flags da seção "Tabela de flags do usuário".

**Seletor necessário (novo):**

| Selector | Arquivo | Contexto |
|----------|---------|---------|
| `data-tour="menu-colaboradores"` | `menu.component.html` | Item "Colaboradores" no menu lateral — usado no tour-home variante MANAGER |

---

### F. Seção "Rever tours" nas Configurações — filtrada por perfil

A lista de tours exibida na seção "Tours e Onboarding" em `/configuracoes/perfil` deve ser filtrada pelo perfil do usuário logado — o MANAGER não precisa ver botões de tours de Organização que ele nunca acessará.

| Tour (nome amigável) | ADMIN | MANAGER | VIEWER |
|----------------------|:-----:|:-------:|:------:|
| Visão geral do Manager | Sim | Sim | Sim |
| Departamentos e Cargos | Sim | Não | Não |
| Criação de Times | Sim | Não | Não |
| Organograma e Instalador | Sim | Não | Não |
| Painel de Colaboradores | Sim | Sim | Não |
| Perfil do Colaborador | Sim | Sim | Sim |
| Relatórios Semanais por IA | Sim | Sim | Não |
| Meu Perfil e Segurança | Sim | Sim | Sim |

---

> **MVP1 — O que entra agora:** todos os usuários recebem o fluxo ADMIN (steps base, menu Organização pulsando). A estrutura de `TourDefinition` com `visibleTo` e `overrides` já pode ser implementada no `TourService`, mas os overrides de MANAGER e VIEWER ficam vazios até a iteração de personalização.
>
> **Pós-MVP1 — O que vem depois:** popular os overrides de MANAGER e VIEWER, implementar a lógica de `resolver_steps` por perfil, criar a flag `menu_colaboradores_pulsando` e filtrar a seção "Rever tours" por perfil.
