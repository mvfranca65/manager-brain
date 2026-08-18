> **STATUS:** Layout FECHADO — Variante A + Refino 1 "Sem cartão" + Telas 1, 2, 3 e 4 todas APROVADAS (Marcos, 2026-08-14: "gostei demais"). Único item em aberto: decisão de naming do @Steve/Marcos (§0).
> **DATA:** 2026-08-14 (atualizado no mesmo dia — rodada 3: Refino 1 aplicado também à Tela 2 e aos estados de erro/sem conexão da Tela 1)
> **DONO:** @Groot (UX/UI Designer)
> **PARA:** @Sam (implementação Kotlin), Marcos (decisão), @Steve (naming), @Tony (revisão arquitetural — telas novas de permissão)
> **ESCOPO:** Redesenho completo da jornada visual do Agent Mobile Android (BYOD) — mudança de identidade "agent/monitor" → "proteção/segurança"
> **REFERÊNCIA DE CÓDIGO:** `manager-srv-agent-android/app/src/main/kotlin/com/trivion/manageragent/ui/` (`PermissionDispatchActivity`, `AgentStatusActivity`, `VincularDialogFragment`, `PermissionStepMachine`) + `res/layout/*.xml` + `res/values/{colors,strings}.xml`

---

## 0. Mudança de identidade — decisão central

**Problema:** o app hoje se apresenta como "MANAGER · AGENT MOBILE" / "Status do Agent" — vocabulário de vigilância corporativa instalado num celular pessoal (BYOD). Marcos pediu que pareça um app de segurança/proteção, não um observador.

**Direção adotada em todo o redesenho:**
- Nunca usar "Agent", "Monitor", "Observador", "Monitoramento" na UI.
- Vocabulário de referência: apps de segurança pessoal (antivírus). Estado positivo dominante no topo ("Proteção ativa"), com um escudo como ícone-âncora em toda a jornada.
- Detalhes técnicos (fila local, sessão, ambiente staging/prod) saem da camada de linguagem principal e vão para uma seção secundária "Detalhes técnicos" — **mesmo padrão já usado no relatório semanal do Manager** (Score IA visível vs. Detalhes Técnicos escondidos). Reaproveitar esse idioma do produto em vez de inventar um novo.
- Cor de estado desacoplada do indigo de marca: verde = protegido, âmbar = atenção, vermelho = interrompido — como qualquer app de segurança. O indigo continua como cor de ação/marca (botões, links, ícone de app). **Desvio deliberado** do "só indigo" do portal — justificado porque o portal é uma ferramenta de gestão (não precisa de semáforo de segurança) e o app mobile precisa comunicar estado de proteção à primeira vista, sem o colaborador ter que ler texto.

### Naming — PROPOSTA, decisão final pendente de @Steve/Marcos

| Opção | Nome exibido | Rationale | Risco |
|---|---|---|---|
| **A (recomendada)** | **iManager Proteção** | Mantém o vínculo com "iManager", nome que o gestor já comunicou ao colaborador no convite de instalação (`_shared/produto.md` §1 — iManager é o nome comercial). Troca "Agent" por "Proteção", que é o que muda a percepção. Menor risco de confundir o colaborador que já ouviu falar de "iManager" pelo RH. | Ainda carrega "Manager" na raiz — se o colaborador pesquisar o nome, chega ao produto de gestão |
| **B (alternativa)** | **Trivion Shield** | Rompe completamente com "Manager", 100% linguagem de segurança, mais parecido com Malwarebytes/Norton. | Quebra a ponte com o que o colaborador já ouviu no onboarding — pode parecer um app diferente, gerando desconfiança ("por que tem um app que eu não conheço no meu celular?") |

**Recomendação do @Groot: Opção A.** Nos mockups abaixo, o nome usado é **"iManager Proteção"** como placeholder de trabalho — não é decisão final. `@Sam` não deve hardcodar strings de nome de produto sem confirmação de @Steve/Marcos.

**Pendência explícita:** nome final, se muda o ícone do launcher/label do pacote (`app_name` em `strings.xml`), e se afeta o texto que o RH/gestor usa para pedir a instalação (fora do escopo deste redesenho — comunicação/onboarding é @Steve).

---

## 1. Tabela de vocabulário — texto velho → texto novo

| String resource | Texto atual | Texto novo | Onde aparece |
|---|---|---|---|
| `permission_brand` | `Manager · Agent Mobile` | `IMANAGER · PROTEÇÃO` (eyebrow, uppercase, letterspacing) | Tela 1, topo do card |
| `permission_title` | `Configure seu acesso` | `Ativar proteção` | Tela 1, título |
| `permission_subtitle` | `Informe seus dados para vincular este aparelho à sua empresa.` | `Confirme sua identidade para ativar a proteção neste aparelho.` | Tela 1, subtítulo |
| `permission_empresa_prefix` | `Empresa: ` (bug: falta espaço na composição atual, verificar concatenação em `PermissionDispatchActivity.kt:139`) | `Empresa · ` (separador visual, não dois-pontos colado) | Chip da empresa |
| `permission_identificador_hint` | `Matrícula ou e-mail funcional` | `Matrícula ou e-mail funcional` (mantido — já é claro) | Campo de input |
| `permission_termos_prefix` + `permission_termos_link` | `Li e aceito os ` + `Termos de Uso` | `Li e aceito os ` + `Termos de Uso e Privacidade` | Checkbox |
| `permission_botao_continuar` | `Continuar` | `Ativar proteção` | Botão primário |
| `permission_validando` | `Validando identificador…` | `Confirmando identidade…` | Estado de loading (agora dentro do botão, ver §4) |
| `permission_confirmar_colaborador_titulo` | `Confirmar colaborador` | `Confirmar identidade` | Bottom sheet (era AlertDialog) |
| `permission_confirmar_colaborador_mensagem` | `Você é %1$s?` | `Você é %1$s?` (mantido) | Bottom sheet |
| `permission_confirmar_colaborador_sim` | `Sim, sou eu` | `Sim, sou eu` (mantido) | Botão primário do sheet |
| `permission_confirmar_colaborador_nao` | `Não, corrigir` | `Não, corrigir` (mantido) | Botão secundário do sheet |
| `permission_erro_id_nao_encontrado` | `Identificador não encontrado. Verifique com seu gestor.` | mantido, mas exibido **inline no campo** (erro do `TextInputLayout`), não como texto solto abaixo do botão | Tela 1, estado de erro |
| `permission_step_accessibility` | `Habilite o Serviço de Acessibilidade do ManagerAgent` | `Ative a permissão de Acessibilidade` (some a referência a "ManagerAgent" — ver §6, tela de checklist) | Tela de permissões (nova) |
| `permission_step_usage_stats` | `Permita acesso ao histórico de uso de apps` | `Permita ver quais apps você usa` | Tela de permissões |
| `permission_step_notification` | `Permita notificações do ManagerAgent` | `Permita notificações` | Tela de permissões |
| `permission_step_phone` | `Permita acesso ao estado do telefone` | `Permita identificar chamadas (sem número)` | Tela de permissões |
| `permission_step_oem` | `Configure a inicialização automática do ManagerAgent` | `Mantenha a proteção ativa ao reiniciar o aparelho` | Tela de permissões |
| `vincular_processing` | `Vinculando este aparelho` | `Ativando proteção` | Bottom sheet de vínculo |
| `vincular_processing_description` | `Mantenha esta tela aberta...` | `Mantenha esta tela aberta. Isso leva poucos segundos.` | idem |
| `vincular_failed` | `Não foi possível concluir o vínculo` | `Não conseguimos ativar a proteção` | idem, estado de falha |
| `status_title` | `Status do Agent` | *(some como título de texto — vira o hero de estado, ver §5)* | Tela 4 |
| `status_service_running` | `Serviço: em execução` | `Proteção em tempo real: ativa` (linguagem antivírus) — vai para seção "Atividade" | Tela 4 |
| `status_service_waiting` | `Serviço: aguardando inicialização` | `Proteção iniciando…` | Tela 4 |
| `status_session_unlocked` / `_locked` | `Sessão: ativa` / `Sessão: bloqueada` | `Conexão com a empresa: ativa` / `em espera` — renomeado porque "sessão" é jargão técnico sem sentido pro colaborador | Tela 4, "Detalhes técnicos" |
| `status_permissions_local` | `Permissões: verifique nas configurações do aparelho` | **substituído por checklist real** — cada permissão com status individual + ação "Corrigir" (ver §5) | Tela 4 |
| `status_queue` | `Fila local: %1$d eventos` | `Dados aguardando envio: %1$d` — vai para "Detalhes técnicos", não é mais frase solta | Tela 4, técnico |
| `status_refresh` | `Atualizar diagnóstico` | `Verificar agora` | Botão primário |
| `status_export` | `Exportar diagnóstico` | `Exportar diagnóstico para o suporte` | Botão secundário |
| *(sem string hoje)* | — | `Proteção ativa` / `Proteção parcial` / `Proteção interrompida` | Hero de estado, novo (ver §5) |
| *(sem string hoje)* | — | `Última verificação: há %1$s` | Hero, novo |
| *(sem string hoje)* | — | `Sem conexão com a internet` | Banner offline, novo (ver §7) |

**Termos banidos em qualquer texto novo:** "Agent", "Monitor", "Monitoramento", "Observador", "Vigilância", "Rastreamento". Termo preferido para a ação de fundo: **"proteção"**, **"verificação"**, **"sincronização"**.

---

## 2. Design tokens

### 2.1 Cores

| Token | Hex | Uso | Origem |
|---|---|---|---|
| `color.brand.indigo` | `#4F46E5` | Ações primárias, links, ícone do app, marca | Portal (mantido) |
| `color.brand.indigoDark` | `#3730A3` | Estado pressed de botões indigo | Portal (mantido) |
| `color.brand.indigoTint` | `#EEF2FF` | Fundo de chip, fundo de ícone secundário | Portal (mantido, já existe como `manager_chip_background`) |
| `color.state.safe` | `#16A34A` | Texto/ícone do estado "Proteção ativa" | **Novo** |
| `color.state.safeBg` | `#F0FDF4` | Fundo do hero card no estado seguro | **Novo** |
| `color.state.safeBorder` | `#BBF7D0` | Borda do hero card no estado seguro | **Novo** |
| `color.state.warning` | `#D97706` | Texto/ícone do estado "Proteção parcial" | **Novo** |
| `color.state.warningBg` | `#FFFBEB` | Fundo do hero card no estado de atenção | **Novo** |
| `color.state.warningBorder` | `#FDE68A` | Borda do hero card no estado de atenção | **Novo** |
| `color.state.danger` | `#DC2626` | Texto/ícone do estado "Proteção interrompida" | **Novo** |
| `color.state.dangerBg` | `#FEF2F2` | Fundo do hero card no estado de erro | **Novo** |
| `color.state.dangerBorder` | `#FECACA` | Borda do hero card no estado de erro | **Novo** |
| `color.bg.screen` | `#F8FAFC` | Fundo de tela | Portal (mantido, `manager_background`) |
| `color.bg.card` | `#FFFFFF` | Fundo de card | Portal (mantido) |
| `color.border.default` | `#E5E7EB` | Borda de cards secundários | Portal (mantido) |
| `color.text.primary` | `#1E293B` | Títulos — **alinhado ao header do portal** (`#1e293b`); código atual usa `#0F172A`, unificar nesta leva | Ajuste de paridade |
| `color.text.body` | `#334155` | Corpo de texto | Ajuste (levemente mais escuro que o `manager_muted` atual `#475569` para textos de leitura primária, ex. descrição do hero) |
| `color.text.muted` | `#64748B` | Texto secundário, legendas | Ajuste leve sobre `manager_muted` (`#475569`) para dar mais contraste hierárquico entre "corpo" e "muted" |
| `color.divider.subtle` | `#F1F5F9` | Fundo de linha de checklist / seção zebra | **Novo** |

### 2.2 Tipografia (Roboto — fonte padrão Android, sem custom font)

| Estilo | Tamanho (sp) | Peso | Uso |
|---|---|---|---|
| `type.eyebrow` | 11sp | 700, letter-spacing 1.5sp, uppercase | "IMANAGER · PROTEÇÃO" |
| `type.heroTitle` | 24sp | 700 | "Proteção ativa" / "Ativar proteção" |
| `type.screenTitle` | 20sp | 700 | Títulos de card secundário ("Permissões", "Dispositivo") |
| `type.body` | 15sp | 400, line-height 22sp | Textos de descrição |
| `type.bodyMuted` | 14sp | 400, line-height 20sp | Subtítulos, legendas |
| `type.label` | 13sp | 600 | Rótulos de linha ("Empresa", "Versão") |
| `type.button` | 16sp | 600 | Texto de botão |
| `type.caption` | 12sp | 400 | Versão do app, rodapé |

### 2.3 Espaçamento (dp) — grid base 4dp

`4, 8, 12, 16, 20, 24, 28, 32, 40`

- Padding de tela: `24dp` (mantido do código atual)
- Padding de card hero: `28dp`
- Padding de card secundário: `20dp`
- Gap entre seções verticais: `20dp`
- Gap entre linhas dentro de um card: `14dp`
- Altura mínima de toque (botões, linhas clicáveis): `48dp` (regra `mobile-design` — nunca abaixo de 44-48dp)

### 2.4 Raio de borda (dp)

| Elemento | Raio | Nota |
|---|---|---|
| Hero card (estado de proteção) | `20dp` | Maior que o portal (14px) — deliberado, para dar sensação mais suave/pessoal, coerente com apps de segurança consumer, não dashboard corporativo |
| Card secundário (Dispositivo, Permissões, Detalhes técnicos) | `16dp` | Próximo do portal, mantém parentesco visual |
| Botão primário/secundário | `14dp` | Alinhado ao portal (`border-radius: 14px`) |
| Chip | `999dp` (pill) | Mantido do código atual (`Chip` do Material Components) |
| Bottom sheet (Tela 3) | `20dp` só nos cantos superiores | Padrão Material bottom sheet |
| Input (TextInputLayout outlined) | `12dp` | Mantido do código atual |

### 2.5 Ícones

- Ícone-âncora: **escudo** (`shield`), estilo outline com preenchimento sutil no estado ativo. Usado no launcher (nova proposta de ícone — fora do escopo de asset final, mas a direção é: escudo indigo sobre fundo branco/claro, sem elementos de "olho" ou "lupa" que reforcem vigilância).
- Tamanho do ícone hero: `72dp` (badge circular)
- Tamanho de ícones de linha (checklist, permissões): `24dp`
- Biblioteca recomendada para @Sam: Material Symbols (`shield`, `check_circle`, `warning`, `error`, `sync`, `lock`, `visibility_off` — evitar `remove_red_eye`/`visibility` em qualquer contexto, ironia óbvia com vigilância)

---

## 3. Tela 1 — Refino 1 "Sem cartão": DECISÃO FINAL

> **Fechado 2026-08-14 (rodada 3):** Marcos escolheu o **Refino 1 "Sem cartão"** — "gostei demais". É a decisão final de layout da Tela 1. O mesmo tratamento (sem cartão, centralização vertical real, 2 clusters com respiro desigual, rodapé de versão ancorado fora do fluxo) foi replicado na **Tela 2** (§4) e nos dois estados adicionais de erro/sem conexão da Tela 1 (§8.1-8.2). Telas 3 e 4 permanecem congeladas, sem alteração, desde a aprovação de 2026-08-14 (rodada 2).
>
> Histórico da decisão: rodada 1 comparou Variante A ("Shield Hero") vs. Variante B ("Split Panel") → A aprovada. Rodada 2 comparou Refino 1 ("Sem cartão") vs. Refino 2 ("Hero solto + superfície mínima") dentro da Variante A → Refino 1 aprovado. Rodada 3 (esta) estendeu o Refino 1 às demais telas de Tela 1 que ainda usavam o card antigo.

O diagnóstico por trás do feedback tem duas causas distintas, e os dois refinos abaixo atacam as duas:

1. **Densidade de conteúdo:** o card original (ver §3.3) empilhava 9 elementos num único container com borda (ícone, eyebrow, título, subtítulo, chip, campo, checkbox, botão, rodapé). Não é o *tamanho* do card que pesa — é a quantidade de elementos visuais distintos dentro dele.
2. **Centralização vertical real:** no mockup original, o bloco de conteúdo ficava ancorado perto do topo da tela (`padding-top` fixo), deixando espaço morto embaixo — por isso não parecia centralizado, mesmo com conteúdo relativamente compacto.

### 3.1 Refino 1 — "Sem cartão" (DECISÃO FINAL — escolhido pelo Marcos)

O card é dissolvido por completo: sem borda, sem sombra, sem cor de fundo diferenciada — o conteúdo senta direto no fundo da tela (`color.bg.screen`). Mudanças em relação ao card original:

- **Chip da empresa removido.** A informação da empresa entra dentro do próprio subtítulo: *"Confirme sua identidade para proteger este aparelho na iManager Tecnologia."* — isso elimina um componente inteiro (chip = borda + fundo + padding próprios) sem perder a informação.
- **Ícone reduzido** de `72dp` (badge circular com fundo) para `44dp` solto, sem badge/fundo circular — menos "objeto" visual, mais integrado ao texto.
- **Agrupamento em 2 clusters** com espaçamento desigual: dentro de cada cluster o espaçamento é apertado (`8-10dp`, elementos relacionados), *entre* os clusters o espaçamento é generoso (`40dp`) — cluster 1 = identidade (ícone + eyebrow + título + subtítulo), cluster 2 = ação (campo + checkbox + botão). Esse contraste de espaçamento é o que faz o olho ler "duas coisas simples", não "uma pilha de nove".
- **Centralização vertical real:** o bloco inteiro (os 2 clusters) fica dentro de um container `justify-content: center` que ocupa a altura útil da tela (descontando status bar). O rodapé de versão sai do fluxo do bloco e fica fixado a `22dp` do rodapé da tela (`position: absolute`, ancorado no container pai, não no bloco centralizado) — assim o texto de versão nunca "puxa" o centro do bloco pra cima.

**Este é o padrão canônico para toda a Tela 1 e a Tela 2 a partir de agora** — aplica-se a:
- Tela 1, estado padrão (§ acima)
- Tela 1, estado de erro de identificador — mesmo layout, campo em estado de erro nativo do `TextInputLayout` (ver §8.1)
- Tela 1, estado sem conexão — mesmo layout, banner de offline acima do bloco centralizado, botão desabilitado (ver §8.2)
- Tela 2, "Confirmando identidade" — mesmo layout, campo e checkbox em `alpha 0.5`, botão em estado de loading (ver §4)

### 3.2 Refino 2 — "Hero solto + superfície mínima no formulário" (DESCARTADO)

Mesma centralização vertical e o mesmo cluster de identidade solto no fundo (ícone `44dp`, eyebrow, título, subtítulo com empresa embutida). A diferença: o **grupo de ação** (campo + checkbox + botão) ganha uma superfície sutil — fundo branco puro (`#FFFFFF`) contra o fundo levemente acinzentado da tela (`#F8FAFC`), raio `16dp`, **sem borda, sem sombra**. O contraste de cor (branco vs. cinza muito claro) já é suficiente para o olho perceber "isso é um grupo", sem precisar de linha ou elevação.

Isso reduz o card de 9 elementos para 3 (campo, checkbox, botão) — a hero (ícone/título/subtítulo) deixa de fazer parte do "container" visualmente.

**Diferença prática entre os dois refinos:** Refino 1 é mais radical (zero superfície em toda a tela); Refino 2 mantém uma pista visual mínima de agrupamento em torno da ação. Ambos resolvem a densidade e a centralização.

**Decisão: Refino 1.** Marcos escolheu ao ver os dois lado a lado em `mockups.html` ("gostei demais"). O Refino 2 não é mais uma opção viva — fica registrado no HTML só como histórico da comparação, marcado como "descartado".

### 3.3 Nota de implementação para @Sam — o bug de centralização

O XML atual (`activity_permission_dispatch.xml:15-18`) já declara `android:layout_gravity="center_vertical"` no `MaterialCardView` dentro do `FrameLayout`. Se em staging o card não pareceu centralizado, a causa provável não é a gravity em si, mas o **volume de conteúdo somado ao padding fixo** — um card com 9 elementos empilhados frequentemente já ocupa quase toda a altura útil da tela, tornando "centralizado" imperceptível na prática. Com o conteúdo reduzido pelos refinos acima, a centralização volta a ser visualmente óbvia. Ainda assim, ao implementar:

- Trocar o `FrameLayout` + `layout_gravity="center_vertical"` por um `ConstraintLayout` com o grupo de conteúdo ancorado por `app:layout_constraintTop_toTopOf` + `app:layout_constraintBottom_toTopOf` num guideline, ou por `app:layout_constraintVertical_bias="0.5"` — mais previsível do que gravity dentro de `NestedScrollView`/`FrameLayout` quando o teclado abre (o teclado reduz a altura útil; vale testar os dois refinos com o teclado aberto, já que o campo de input é o elemento que o dispara).
- O rodapé de versão (`tvAppVersion`) deixa de ser filho do grupo centralizado e passa a ser ancorado direto no `ConstraintLayout` raiz, `app:layout_constraintBottom_toBottomOf="parent"` com margem fixa — para não competir pelo centro vertical com o resto do conteúdo.

### 3.4 Arquivo histórico — comparação original A vs. B (não usar mais)

A primeira rodada deste redesenho comparou a Variante A ("Shield Hero", card único) com uma Variante B ("Split Panel", bloco de cor sólida no topo + formulário abaixo, mais institucional/MDM). Marcos aprovou a Variante A nessa comparação. A Variante B foi removida do `mockups.html` nesta rodada — não é mais uma opção viva, fica só registrada aqui para histórico da decisão.

---

## 4. Tela 2 — resolvendo o loader solto

**Problema original:** spinner ciano solto embaixo do botão "Continuar", desconectado da ação.

**Solução adotada:** o próprio botão primário assume estado de loading, dentro do layout sem cartão do Refino 1 (§3.1) — mesmos 2 clusters, mesma centralização vertical, mesmo rodapé de versão ancorado.
- Texto do botão muda de "Ativar proteção" → "Confirmando identidade…"
- Ícone de spinner (`16dp`, branco, mesma cor do texto do botão) aparece à **esquerda do texto**, dentro do botão
- Botão fica `disabled` (não clicável, mas mantém a cor indigo em opacidade 100% — não fica cinza, para não parecer "erro"; usar `alpha 0.9` no texto/ícone é suficiente para indicar estado ocupado)
- Campo de input e checkbox de termos ficam com `alpha 0.5` e `enabled=false` durante o loading — reforça visualmente "isso está em andamento, não mexa"
- Remove-se completamente o `ProgressBar` + `tvStatus` soltos abaixo do botão (`activity_permission_dispatch.xml:147-165`) — a função desse texto (erro/status) passa a viver **inline no campo** (ver §8.1, "erro de identificador")

Isso é o padrão nativo de qualquer botão de submit em Material Design (`CircularProgressIndicator` dentro do `MaterialButton` via `icon` + `iconGravity=start`), sem precisar de view extra nem cor fora da paleta (o ciano do estado atual não faz mais sentido — o spinner agora herda a cor branca do texto do botão).

**Atualizado 2026-08-14 (rodada 3):** esta tela, que já estava aprovada no formato com card, foi refeita no `mockups.html` sem cartão, para bater com a decisão final do Refino 1 — o card em si nunca fez parte do que Marcos aprovou como "excelente" nessa tela (o elogio era sobre a resolução do loader, não sobre o container). Nenhuma mudança de comportamento, só de container visual.

---

## 5. Tela 3 — substituindo o AlertDialog cru

**Problema original:** `MaterialAlertDialogBuilder` puro, sem estilização — foge do design system, título/botões no padrão cru do Android.

**Solução adotada:** `BottomSheetDialogFragment` (Material Components já está na dependência do projeto via `com.google.android.material`), consistente com o resto da coleção de componentes:
- Sobe do rodapé da tela, cantos superiores `20dp`, drag handle (`4dp × 32dp`, cinza claro) no topo — sinaliza "isso é dispensável/arrastável", mais amigável que um modal bloqueante centralizado
- Avatar circular com iniciais do nome (`56dp`, fundo `indigoTint`, texto indigo) — dá um toque pessoal/humano à confirmação
- Título "Confirmar identidade" (`type.screenTitle`)
- Texto "Você é **{Nome Completo}**?" (nome em negrito)
- Botão primário full-width "Sim, sou eu" (indigo, `14dp` de raio) — **no topo**, dentro da zona de alcance do polegar (regra Fitts's Law do `mobile-design`: ação mais provável primeiro e mais perto do polegar)
- Botão secundário abaixo, estilo texto (não outlined) "Não, corrigir" — ação menos destrutiva mas menos provável, visualmente mais leve
- `isCancelable = false` mantido (mesma regra de negócio atual — não pode ser dispensado tocando fora)

---

## 6. Tela de permissões (nova) — o que faltava

Hoje o encadeamento de permissões (`PermissionStepMachine`) manda o colaborador direto para telas do Android Settings, uma atrás da outra, só com uma linha de texto (`tvStatus`) mudando por trás. Sem tela própria, é desorientador — o colaborador não sabe quantos passos faltam nem por quê.

**Nova tela "Configurando proteção"** (mostrada **antes de cada `Intent` de Settings ser disparada**, substitui o `showStatus()` atual):

- Título fixo: "Configurando proteção" (`type.screenTitle`)
- Indicador de progresso: "Passo 2 de 4" + barra de progresso linear fina (`4dp` altura, indigo sobre trilho `color.divider.subtle`)
- Lista checklist vertical dos 4 passos canônicos (+ OEM condicional), cada linha com:
  - Ícone de estado à esquerda: `check_circle` verde (concluído) / círculo indigo preenchido com número (atual) / círculo outline cinza (pendente)
  - Texto do passo (vocabulário novo da tabela §1: "Ative a permissão de Acessibilidade" etc.)
  - Subtexto de uma linha explicando o "porquê" em linguagem de proteção, ex: *"Isso nos permite identificar quando você está protegido durante o uso do aparelho."*
- Botão primário "Abrir configurações" — dispara o `Intent` correspondente ao passo atual (mesma lógica de `executeStep`, só que agora com uma tela intermediária em vez de pular direto)
- Rodapé com link texto "Por que preciso disso?" → abre um modal curto explicando cada permissão em linguagem simples (não é blocker, é FAQ opcional)

Isso não muda a máquina de estados (`PermissionStepMachine.nextStep` continua igual) — muda só a apresentação: em vez de `showStatus(String)` alterar um `TextView`, a Activity passa a navegar para esta tela de checklist entre cada `Intent`. **Ajuste arquitetural — @Tony revisa antes de @Sam implementar**, porque isso adiciona uma tela real (hoje é zero-UI entre steps).

---

## 7. Tela 4 — hierarquia de verdade

### 7.1 Estrutura (3 blocos, ~10 linhas de informação total)

**Bloco 1 — Hero de estado (dominante, topo)**
Card grande, cor de fundo/borda conforme estado (§2.1), ícone de escudo `72dp` como badge:
1. Ícone de estado (escudo + check / escudo + ! / escudo + x)
2. Título do estado: "Proteção ativa" / "Proteção parcial" / "Proteção interrompida"
3. Subtítulo de uma linha com o resumo humano do que está acontecendo
4. "Última verificação: há 2 min" (ou "há 3 horas" no erro)
5. *(só nos estados atenção/erro)* botão de ação dentro do hero: "Corrigir agora" / "Tentar novamente"

**Bloco 2 — "Dispositivo" (card secundário, raio 16dp)**
6. Empresa vinculada (ex. "iManager Tecnologia")
7. Identificador vinculado (matrícula/e-mail, mascarado parcialmente: `joao.***@empresa.com`)
8. Versão do app (`iManager Proteção 1.0.13`) — ambiente (staging/prod) só aparece aqui em builds não-prod, como badge pequeno, nunca em prod

**Bloco 3 — "Permissões" (card secundário, raio 16dp)**
9. Checklist real das 4 permissões (Acessibilidade, Uso de apps, Notificações, Telefone), cada uma com ícone check/warning + ação inline "Corrigir" quando faltando — **substitui a frase vaga `status_permissions_local`**

**Bloco 4 — "Atividade" (card secundário, colapsável — equivalente aos "Detalhes Técnicos" do relatório semanal)**
10. Proteção em tempo real: ativa/parada · Conexão com a empresa: ativa/em espera · Dados aguardando envio: N · (link "Ver mais" se precisar expandir)

**Ações (rodapé, fora dos cards)**
- Botão primário: "Verificar agora"
- Botão secundário outlined: "Exportar diagnóstico para o suporte"
- Link texto: "Informações do aplicativo"

Isso soma exatamente ~10 linhas de informação (itens 1-10) organizadas em 4 blocos com hierarquia clara, deixando espaço para o que o @Tony ainda está definindo (novos dados do Agent Windows) — o Bloco 3 (Permissões) e o Bloco 4 (Atividade) são extensíveis: qualquer item novo entra como mais uma linha dentro do bloco certo, sem precisar redesenhar a tela.

### 7.2 Os 3 estados do hero

| Estado | Cor | Ícone | Título | Subtítulo | Ação |
|---|---|---|---|---|---|
| **OK** | `color.state.safe` / `safeBg` / `safeBorder` | escudo + check | "Proteção ativa" | "Tudo funcionando normalmente." | nenhuma (some) |
| **Atenção** | `color.state.warning` / `warningBg` / `warningBorder` | escudo + `!` | "Proteção parcial" | "1 permissão pendente. Sua atividade pode não ser registrada corretamente." | "Corrigir agora" → leva à tela de permissões (§6), já filtrada só no passo faltante |
| **Erro** | `color.state.danger` / `dangerBg` / `dangerBorder` | escudo + `x` | "Proteção interrompida" | "Não conseguimos enviar seus dados há 3 horas." | "Tentar novamente" → força uma tentativa de sincronização imediata |

---

## 8. Estados adicionais fora das 4 telas originais

### 8.1 Erro de matrícula inválida (Tela 1)

Em vez do texto solto `tvStatus` abaixo do botão (layout atual, `activity_permission_dispatch.xml:155-165`), o erro vira **estado nativo do `TextInputLayout`**, dentro do layout sem cartão do Refino 1 (§3.1):
- Borda do campo muda de `color.border.default` para `color.state.danger`
- Texto de erro (`errorText` do Material `TextInputLayout`) aparece embaixo do campo, cor `color.state.danger`, `12sp`
- Ícone de erro (`error` outline) aparece dentro do campo, lado direito
- Mensagem: reaproveita as strings já existentes (`permission_erro_id_nao_encontrado` etc. — a tabela §1 já mapeia as que mudam de tom)
- O botão volta ao estado normal (não fica em loading), habilitado para nova tentativa assim que o campo for editado
- **Atualizado 2026-08-14 (rodada 3):** telas de estado (`mockups.html` bloco 2) refeitas sem cartão, para bater com a decisão final do Refino 1 — mesma centralização vertical, mesmos 2 clusters, rodapé de versão ancorado.

### 8.2 Sem conexão

Dois pontos de ocorrência:
- **Tela 1 (antes de enviar):** banner fino no topo da tela, fora do bloco centralizado do Refino 1, fundo `color.state.warningBg`, texto `color.state.warning`, ícone de wifi cortado: "Sem conexão com a internet — verifique sua rede." O botão "Ativar proteção" fica desabilitado enquanto o banner estiver visível. O bloco de conteúdo abaixo do banner mantém a mesma centralização vertical do Refino 1 — o banner não desloca o centro do bloco, só reduz um pouco a área útil acima dele.
- **Tela 4 (app já vinculado):** mapeia direto para o hero de estado **"Erro"** (§7.2) — reaproveita o mesmo componente, sem criar um 4º estado. Subtítulo muda dinamicamente conforme a causa (sem internet vs. servidor fora vs. token expirado), mas a apresentação visual é a mesma. Tela 4 está congelada — não houve alteração nesta rodada.

### 8.3 Fluxo de permissões

Coberto integralmente em §6 — é o item que mais faltava (`PermissionStepMachine` já existe no código como máquina de estados pura, mas sem nenhuma tela própria por trás).

---

## 9. Especificação de componentes (para @Sam)

### 9.1 `HeroStatusCard` (novo componente)
- `MaterialCardView`, `cardCornerRadius=20dp`, `strokeWidth=1dp`, `cardElevation=0dp` (usa `strokeColor` do estado, não sombra, para não pesar visualmente)
- Padding interno `28dp`
- Ícone: `72dp`, centralizado, `ImageView` com `tint` do estado sobre um `background` circular (`indigoTint`/`safeBg`/etc conforme estado — ver nota abaixo)
- **Nota de contraste:** o círculo de fundo do ícone usa a cor `*Bg` do próprio estado (ex. `safeBg` no estado OK), não o `indigoTint` fixo — isso reforça a leitura de cor à distância
- Título: `type.heroTitle`, cor do estado (`safe`/`warning`/`danger`)
- Subtítulo: `type.body`, `color.text.body`
- Timestamp: `type.caption`, `color.text.muted`
- Botão de ação (quando aplicável): `MaterialButton`, full-width, `48dp` altura, cor de fundo = cor do estado (ex. botão âmbar no estado atenção)

### 9.2 `PermissionChecklistRow` (novo componente)
- Altura mínima `48dp` (touch target)
- Ícone de status `24dp` à esquerda
- Texto do passo `type.label` + subtexto `type.bodyMuted` (2 linhas, opcional)
- Ação inline "Corrigir" (`TextButton`, `type.label`, cor indigo) à direita, só visível quando status ≠ concluído
- Divider `1dp` `color.divider.subtle` entre linhas

### 9.3 `InlineFieldError` (ajuste ao componente existente)
- Usa `TextInputLayout.setError(String)` nativo do Material Components — **não é componente novo**, é uso correto do que já existe na dependência (`com.google.android.material.textfield.TextInputLayout`), hoje subutilizado porque o erro está sendo mostrado num `TextView` solto ao invés de no próprio campo

### 9.4 `LoadingButton` (ajuste ao botão existente)
- `MaterialButton` com `app:icon="@drawable/ic_spinner"` (ou `CircularProgressIndicator` sobreposto via `FrameLayout` se preferir animação nativa) + `app:iconGravity="textStart"` + troca de `text` via `setText()`
- Não é um componente novo de biblioteca — é um padrão de estado (`isLoading: Boolean`) aplicado ao `MaterialButton` já existente

### 9.5 `ConfirmIdentityBottomSheet` (substitui `VincularDialogFragment`'s uso de `MaterialAlertDialogBuilder` cru para a confirmação — o `VincularDialogFragment` de loading/retry continua como dialog, mas ganha estilo consistente com os cards)
- `BottomSheetDialogFragment`
- `shapeAppearance` custom: `cornerSizeTopLeft/TopRight = 20dp`, `cornerSizeBottomLeft/BottomRight = 0dp`
- Avatar: `ShapeableImageView` ou `TextView` circular com iniciais, `56dp`, `indigoTint` de fundo, texto indigo `20sp` bold

---

## 10. Acessibilidade (WCAG aplicado ao contexto mobile nativo)

- Contraste mínimo 4.5:1 verificado para todas as combinações texto/fundo dos 3 estados (verde/âmbar/vermelho sobre seus respectivos `*Bg` claros — todos ≥ 4.6:1 com os hex propostos)
- Nunca usar cor isoladamente para comunicar estado — todo estado tem ícone + texto redundante (escudo+check / escudo+! / escudo+x, nunca só a cor da borda)
- Todo alvo de toque ≥ `48dp` (checklist rows, botões, ícones de fechar/dispensar)
- `contentDescription` obrigatório em todos os ícones de estado (ex. "Proteção ativa", não "escudo verde")
- Bottom sheet e checklist de permissões devem ser navegáveis via TalkBack em ordem lógica (topo→baixo, ação de confirmar antes da de corrigir)
- Texto nunca abaixo de `12sp` (caption é o piso)

---

## 11. Resumo de desvios do design system do portal — e por quê

| Desvio | Portal | App mobile | Justificativa |
|---|---|---|---|
| Paleta de estado | Só indigo | Verde/âmbar/vermelho + indigo | App precisa de leitura de "seguro/atenção/erro" à primeira vista, como qualquer app de segurança pessoal. Portal é ferramenta de gestão, não precisa desse semáforo. |
| Raio do hero card | `14px` fixo | `20dp` no hero, `16dp` nos cards secundários, `14dp` nos botões | Suaviza a sensação "instalação corporativa" → mais parecido com apps de segurança consumer. Botões mantêm o raio do portal para não perder o parentesco de marca. |
| Cor de título | `#1e293b` | `#1E293B` (igual) | Sem desvio — na verdade é uma correção: o app hoje usa `#0F172A`, ajustado nesta leva para bater com o portal. |
| Ícone de app | N/A | Escudo (não usa o ícone atual do launcher) | Ver §2.5 — fora do escopo de asset final, mas a direção visual está definida aqui. |

---

## 12. Pendências explícitas

1. **Nome final do produto** — @Steve/Marcos decide entre "iManager Proteção" (recomendado) e "Trivion Shield", ou propõe um terceiro. Bloqueia: `app_name` em `strings.xml`, ícone do launcher, texto usado no convite de instalação do RH. **Único item de layout/naming ainda em aberto.**
2. ~~Qual refino de densidade da Tela 1~~ — **RESOLVIDO 2026-08-14.** Refino 1 "Sem cartão" é a decisão final (§3.1). Aplicado em toda a Tela 1 e na Tela 2.
3. **Ícone do launcher final** — este spec define a direção (escudo, indigo, sem "olho"/"lupa"), mas o asset final (adaptive icon 108dp, safe zone 66dp) não foi produzido — não é atividade de wireframe/HTML, precisa de arte final.
4. **Tela de permissões é uma tela nova** (§6) — muda a experiência hoje "zero-UI" do `PermissionStepMachine`. @Tony precisa revisar antes de @Sam implementar, porque adiciona uma `Activity`/`Fragment` nova ao fluxo que hoje não existe.
5. **Quais dados novos entram no Bloco 3/4 da Tela 4** — depende do que @Tony está definindo em paralelo com base no Agent Windows. A estrutura deste spec já comporta itens novos sem redesenho (blocos extensíveis), mas o conteúdo exato não foi definido aqui. Tela 4 está congelada quanto ao layout aprovado; isso só adiciona linhas dentro dos blocos já existentes.
6. **Bug do espaço no chip "Empresa:"** — mencionado por Marcos na tela atual; no código-fonte (`PermissionDispatchActivity.kt:139`) a string `permission_empresa_prefix` já termina com espaço (`"Empresa: "`), então o bug pode estar em outro ponto da concatenação ou em como o `Chip` do Material Components corta espaços nas bordas do texto. **Ficou moot com a decisão final do Refino 1** — o chip não existe mais em nenhuma tela; a informação da empresa vive no subtítulo em texto corrido. Nada a investigar.
7. ~~Réplica do refino escolhido nos estados de erro/sem conexão da Tela 1~~ — **RESOLVIDO 2026-08-14 (rodada 3).** Os dois estados adicionais (`mockups.html` bloco 2) e a Tela 2 (bloco 3) foram atualizados para o layout sem cartão do Refino 1 — centralização vertical real, 2 clusters, rodapé de versão ancorado.
