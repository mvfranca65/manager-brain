# Retomada — reboot para instalar a 1.5.13 (2026-08-24)

> **Escrito por @Tony antes do reboot do Marcos.** Se a sessao se perder, comece por aqui.
> O Marcos ia retomar com `claude --continue`.

---

## O que fazer primeiro, ao voltar

1. **Confirmar que a 1.5.13 subiu de verdade.** E o cenario **C1** do roteiro
   (`registro/2026-08-24-roteiro-manual-1.5.13.md`). Os dois servicos no ar e a versao correta
   nos binarios. **Se a versao nao bater, pare tudo** — o resto do roteiro nao vale nada em cima
   de uma instalacao pela metade.

   ```powershell
   Get-Service ManagerAgent,ManagerAgentWatchdog | Select-Object Name,Status
   (Get-Item 'C:\Program Files\ManagerAgent\ManagerAgent.Service.exe').VersionInfo.ProductVersion
   (Get-Item 'C:\Program Files\ManagerAgent\ManagerAgent.Watchdog.exe').VersionInfo.ProductVersion
   ```

   **Confira os dois**, nao so o Service. O `GUIA-RELEASE` mandava bumpar quatro projetos quando
   sao sete, e o Watchdog era um dos esquecidos — e ele e quem compara versao para decidir
   rollback. O guia ja foi corrigido; a conferencia fica porque o pacote foi gerado hoje.

2. **Depois, os 7 cenarios seguros** do roteiro, na ordem que a @Natasha escreveu.

3. **A volta automatica de versao por ultimo**, fim do dia, de preferencia em maquina descartavel.
   Dois cenarios derrubam a maquina de proposito.

---

## Estado no momento do reboot

**Tudo publicado.** Quatro repos:

| Repo | Branch | Commit | Testes |
|---|---|---|---|
| `manager-srv-agent` | `staging` | `02b093c` | 1139 |
| `manager-srv-admin-node` | `staging` | `e86124d` | 1205 |
| `manager-srv-events-node` | `staging` | `dccd1a0` | 714 |
| `manager-brain` | `main` | `0515ecf` | — |

**Versao 1.5.13** nos sete `.csproj`. Pacote gerado pelo Marcos.

### Os quatro itens da linha de corte

| # | O que | Estado |
|---|---|---|
| 1 | lote nao cai inteiro | codigo pronto e publicado. **Nunca testado em maquina** |
| 3a | rollback ligado ao backup | codigo pronto e publicado. **Nunca testado em maquina** |
| 3b | teste E2E de rollback no CI | **em aberto.** Falta decidir a maquina |
| 4 | senha do staging | **em aberto.** Pendencia do Marcos |

### O que foi feito hoje, em uma linha cada

- **R-01** — a maquina parou de reinstalar a versao que a quebrou. Inclui `VersaoSemVer`, porque os
  dois lados guardam a versao em formatos diferentes.
- **R-02** — a versao quebrada passou a ser capturada antes de os binarios serem trocados. Sem
  isso a guarda do R-01 nascia inerte em toda a frota.
- **R-03** — recall de frota, os dois lados. Backend + Agent + nivel de alerta no painel.
- **A-57 + FIX B6** — a fusao no `events-node`: as duas correcoes de reuniao se anulavam
  (`NULL` na chave de unicidade). Resolvido com `NULLS NOT DISTINCT`.
- **Freio nos testes** — a suite parou de derrubar os servicos reais da maquina de quem a roda.
- **`GUIA-RELEASE` corrigido** — dizia quatro projetos, sao sete.

---

## Pendencias do Marcos, nenhuma resolvida

1. **Senha do staging.** Passou pelo chat duas vezes (20/08 e 24/08). Precisa trocar **e** dar o
   painel ao @Vision, senao repete.
2. **O staging e o mesmo projeto Supabase da producao?** Sem resposta. Se for, o item 4 vira
   criterio C (expoe dado de cliente) e deixa de ser higiene.
3. **Maquina para o 3b:** a do GitHub ou uma nossa. @Tony recomendou comecar pela do GitHub.
4. **Unificar os roteiros de teste.** Combinado: fica para depois dos testes de hoje. Hoje vale
   **so** o `2026-08-24-roteiro-manual-1.5.13.md`; os outros cinco sao de geracoes anteriores e
   nao conhecem rollback, recall nem as guardas.

---

## Avisos que custam caro se esquecidos

- **Nao rode os roteiros automatizados (`tests/e2e/`) no meio dos testes manuais.** Varios param e
  sobem servico de proposito e vao brigar com o que estiver sendo medido a mao.
- **Antes de mexer no rollback**, tenha `instalador\ManagerAgent-Setup-v1.5.12.exe` e o `v1.5.11`
  a mao. Se a volta falhar, o Agent entra em SOS e **para de capturar ate reinstalar a mao**.
- **A @Natasha mantem o bloqueio de producao** ate a volta automatica passar em maquina. @Tony
  sustenta.
- **Antes do deploy do `events-node`**, o @Vision precisa confirmar que o Postgres do staging e
  **15 ou mais novo**. A migration usa `NULLS NOT DISTINCT`, que nao existe antes disso. A
  justificativa escrita cita a versao local de desenvolvimento, nao a do Supabase real.

---

## A licao do dia, para nao se perder

Foram **cinco** defeitos da mesma familia, e nenhum foi achado por teste — todos por alguem abrindo
o codigo e conferindo o que o documento afirmava:

1. o rollback lia o backup um nivel de diretorio acima de onde ele e gravado;
2. o campo da versao quebrada nascia vazio em toda a frota;
3. a guarda do recall ficava numa camada que a resposta nunca alcanca;
4. o teste de rollback aceitava a pasta com nome `unknown` e passaria com o defeito;
5. o guia de release mandava bumpar quatro projetos de sete.

*O teste e o codigo concordam entre si, e os dois discordam do instalador de verdade.* E o
argumento do item **3b**: e o unico da lista que exercita a instalacao real.
