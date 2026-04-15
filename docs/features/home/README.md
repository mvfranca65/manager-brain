# Feature: Home (Dashboard do Gestor)
> Rota: `/home`
#feature #portal
> Perfil: GESTOR
> Status: MVP1 funcional

## O que é

Dashboard principal do gestor. Primeira tela após login. Dá visão rápida do estado do time e atalhos para as áreas principais.

## O que tem na tela

### Stats (4 cards)
- **Total de colaboradores** — contagem geral
- **Online agora** — colaboradores com agent ativo (verde)
- **Ausente** — agent ativo mas sem atividade (amarelo)
- **Offline** — agent desligado (cinza)

### Barra de presença
- Progress bar mostrando % online: "12 de 50 colaboradores conectados"

### Ações rápidas (3 cards)
- **Gestão de Colaboradores** → `/colaboradores`
- **Relatórios Semanais** → `/relatorios`
- **Organização** → `/organizacao`

### Saudação personalizada
- "Bom dia, Marcos" + data atual
- Botão de tour contextual

## Dados consumidos
- `GET /api/v2/usuarios/stats` — contagens de status (total, online, ausente, offline)

## Pendências
- Nenhuma pendência MVP1
