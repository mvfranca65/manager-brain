# Feature: Tours Contextuais
> @Steve | PO — negócio e histórias
> @Tony | TL — decisões técnicas
> Status: 🟡 Em desenvolvimento

---

## Objetivo

Substituir o tour linear único de 16 steps por **8 tours independentes por seção**, cada um disparado automaticamente com base no estado real dos dados do usuário.

O gestor é guiado de forma progressiva — sem onboarding forçado e sem jornada longa de entrada. Cada seção ensina a si mesma na primeira vez que é acessada.

---

## Quem usa

- Todo usuário com perfil `ADMIN` ou `GESTOR` ao acessar o portal pela primeira vez
- O tour-home é **obrigatório** uma única vez (sem botão pular)
- Os demais tours são opcionais — disparam automaticamente mas têm botão "Pular"
- Qualquer tour pode ser revisto a qualquer momento via ícone `?` no header ou via Configurações

---

## Princípios do sistema

- Máximo de 5–8 steps por tour
- Cada tour tem seu próprio ciclo de vida e flag de conclusão independente
- Progressão baseada em dados reais: times existem? colaboradores cadastrados? agente enviou dados?
- Flags persistidas no backend por usuário (tabela `user_tour_flags`)
- Repetição sempre disponível — nenhum tour some para sempre

---

## Os 8 tours

| ID | Nome amigável | Rota | Steps | Obrigatório |
|----|---------------|------|:-----:|:-----------:|
| `tour-home` | Visão geral do Manager | `/home` | 4 | Sim |
| `tour-org-estrutura` | Departamentos e Cargos | `/organizacao?tab=departamentos` | 3 | Não |
| `tour-org-times` | Criação de Times | `/organizacao?tab=times` | 1–2 | Não |
| `tour-org-hierarquia` | Organograma e Instalador | `/organizacao?tab=hierarquia` | 4 | Não |
| `tour-colaboradores` | Painel de Colaboradores | `/colaboradores` | 2–5 | Não |
| `tour-colaborador-detalhe` | Perfil do Colaborador | `/colaboradores/membro/:id` | 4 | Não |
| `tour-relatorios` | Relatórios Semanais por IA | `/relatorios` | 3–4 | Não |
| `tour-perfil` | Meu Perfil e Segurança | `/configuracoes/perfil` | 3 | Não |

---

## Contexto detalhado

→ `design-tours-contextuais.md` — design completo: todos os steps, variações por estado, flags, CSS do pulse (@Steve)
→ `tecnico.md` — decisões técnicas, contratos de API, estrutura do TourService refatorado (@Tony)
→ `testes.md` — cenários de teste por tour (@Natasha)
