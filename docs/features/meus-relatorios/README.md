# Feature: Meus Relatórios (Visão Colaborador)
> Rota: `/meus-relatorios` (menu aponta para `/relatorios`)
#feature #portal
> Perfil: COLABORADOR
> Status: MVP1 funcional

## O que é

Permite que o colaborador veja seus próprios relatórios semanais de desempenho. Mesma tela de relatórios do gestor, mas filtrada para mostrar apenas os dados do próprio colaborador.

## O que tem na tela

### Navegador de semanas
- Lista de semanas disponíveis com seleção
- Navega entre semanas sem reload

### Relatório individual
- Score IA + sinais (desempenho, cansaço, tendência)
- Diagnóstico com ponto crítico
- Alertas com severidade
- Foco da semana com ações
- Destaques positivos
- Retrospectiva (comparação com semana anterior)
- Mensagem ao gestor
- Detalhes técnicos (pilares + indicadores)

### Estados
- **Carregando** — skeleton loader
- **Vazio** — "Nenhum relatório disponível"
- **Erro** — mensagem de falha

## Dados consumidos
- `GET /api/v1/relatorios/semanas?timeId=` — semanas disponíveis
- `GET /api/v1/relatorios/colaborador?colaboradorId=&semana=` — relatório individual

## Observação
- A rota `/meus-relatorios` existe no router mas o menu COLABORADOR aponta para `/relatorios`
- O componente `MeusRelatoriosPageComponent` usa `app-semana-navigator` + `app-relatorio-individual`
