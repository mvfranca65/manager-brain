# Testes — Relatórios Semanais por IA
> Domínio: @Natasha | QA
> Última atualização: 2026-03-25

---

## Testes implementados: 76 (todos passando)

| Serviço | Classe | Testes |
|---------|--------|--------|
| manager-srv-ia | ExtratorDadosService | 2 |
| manager-srv-ia | CalculadoraPilaresService | 14 |
| manager-srv-ia | CalculadoraIndicadoresService | 13 |
| manager-srv-ia | LlmService | 4 |
| manager-srv-ia | RelatorioIndividualService | 1 |
| manager-srv-ia | RelatorioTimeService | 1 |
| manager-srv-ia | OrquestradorRelatorioService | 5 |
| manager-srv-portal | TimeServiceHierarquiaTest (Task 12) | 5 |
| manager-srv-portal | RelatorioControllerTest (Task 13) | 7 |
| manager-srv-portal | PdfExportTest (Task 14) | 5 |
| manager-srv-portal | MultitenaancyRelatorioTest (Task 20) | 19 |
| **Total** | | **76** |

---

## Testes implementados — Fase 2 (portal)

### Task 12 — TimeServiceHierarquiaTest ✅
- [x] Admin vê todos os times da empresa
- [x] Gestor vê apenas cadeia hierárquica abaixo dele
- [x] Gestor não vê times de outros departamentos passando IDs arbitrários
- [x] `departamentoId` filtra dentro do conjunto visível ao usuário
- [x] Sem `departamentoId` retorna todos os times visíveis

### Task 13 — RelatorioControllerTest ✅
- [x] `/semanas` retorna array vazio para time sem relatórios (nunca 404)
- [x] `/semanas` retorna semanas ordenadas do mais recente ao mais antigo
- [x] `/time` retorna 404 quando semana não tem relatório
- [x] `/colaborador` retorna 404 quando semana não tem relatório
- [x] Acesso a time fora da hierarquia retorna 403
- [x] Acesso a colaborador fora da hierarquia retorna 403
- [x] JWT inválido retorna 401

### Task 14 — PdfExportTest ✅
- [x] Tipo `TIME` gera PDF só do time
- [x] Tipo `INDIVIDUAL` exige `colaboradorId` (400 sem ele)
- [x] Tipo `CONSOLIDADO` inclui time + todos os individuais em ordem alfabética
- [x] Content-Type: `application/pdf`
- [x] Content-Disposition com filename correto

---

## Testes implementados — Fase 3 (Task 20)

### MultitenaancyRelatorioTest ✅ (19 testes)

**Bloco 1 — /semanas isolamento (3 testes)**
- [x] Empresa A só enxerga semanas do seu próprio time
- [x] Empresa A consultando time_id da Empresa B retorna lista vazia (sem dados vazados)
- [x] MANAGER da Empresa A consultando time da Empresa B recebe 403

**Bloco 2 — /time isolamento (3 testes)**
- [x] Empresa A vê seu próprio relatório de time corretamente
- [x] Empresa A consultando time_id da Empresa B recebe 404
- [x] Dados da Empresa B existentes no banco não aparecem para a Empresa A

**Bloco 3 — /colaborador isolamento (3 testes)**
- [x] Empresa A vê colaborador próprio corretamente
- [x] Empresa A consultando colaborador da Empresa B recebe 404
- [x] Colaborador da Empresa B encontrado mas pertence a time fora da hierarquia — 403

**Bloco 4 — export/pdf isolamento (3 testes)**
- [x] Export PDF tipo TIME para time da Empresa B retorna 404
- [x] Export PDF tipo INDIVIDUAL para colaborador da Empresa B retorna 404
- [x] Export PDF tipo CONSOLIDADO — buscarIndividuaisDoTime filtra por empresa_id

**Bloco 5 — parseSemana edge cases (4 testes)**
- [x] parseSemana retorna intervalo correto para 2025-W12
- [x] parseSemana retorna intervalo correto para 2025-W01 (primeira semana do ano)
- [x] parseSemana retorna intervalo correto para semana 52 (edge case final de ano)
- [x] parseSemana retorna 7 dias de intervalo sempre (invariante)

**Bloco 6 — Permissões por perfil (3 testes)**
- [x] ADMIN acessa qualquer time da mesma empresa sem verificar hierarquia
- [x] MANAGER sem entidade hierárquica recebe 403 ao acessar qualquer time
- [x] VIEWER sem entidade hierárquica recebe 403 ao acessar qualquer time
- [x] MANAGER com acesso ao time via hierarquia consulta semanas normalmente

---

## Testes pendentes — Fase 3 (frontend, Task 20)

### Validação manual / E2E visual

> Plano completo em: `.brain/features/relatorios/validacao-e2e.md`

- [ ] Fluxo completo: disparo manual → tabelas → email → portal
- [ ] Estado vazio (time sem relatórios): mensagem informativa, sem erro de console
- [ ] Primeira semana (sem retrospectiva): componente oculta seção retrospectiva
- [ ] Deeplink time `/relatorios?timeId=X`: pré-seleciona time correto
- [ ] Deeplink colaborador `/relatorios?timeId=X&colaboradorId=Y&semana=Z`: abre individual direto
- [ ] Navegação entre semanas: troca conteúdo sem recarregar página
- [ ] Export PDF dispara download no browser para os 3 tipos (TIME, INDIVIDUAL, CONSOLIDADO)
- [ ] Multitenancy visual: usuário de empresa A não vê dados de empresa B no portal

---

## Riscos identificados

- Rate limit da Anthropic API em times com muitos colaboradores (>20 — já tem log de aviso)
- Timeout em geração pode deixar relatório em estado inconsistente (verificar tratamento no Orquestrador)
- Prompt injection via título de janela malicioso no `evento_atividade`
- Semana ISO edge case: semana 53 em anos com 53 semanas — coberto no MultitenaancyRelatorioTest (W52/W01)
- JWT expirado durante navegação longa — portal deve tratar 401 e redirecionar para login
- Export CONSOLIDADO com muitos colaboradores pode demorar — StreamingResponseBody já implementado
