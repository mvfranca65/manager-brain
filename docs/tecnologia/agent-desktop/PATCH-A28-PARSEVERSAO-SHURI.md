> **DATA:** 2026-08-18 | **PARA:** @Shuri (`manager-srv-admin-node`) | **DE:** @Bucky / @Tony
> **ORIGEM:** achado **A-28** do regressivo do Agent Windows
> **PRIORIDADE:** ALTA — desbloqueia o teste de auto-update da 1.5.4 em staging
> **STATUS:** patch escrito e pronto para revisao. Nao commitado — Marcos valida.

# Patch A-28 — `parseVersao` rejeita a versao que o Agent Windows envia

## O problema em uma frase

O backend responde `atualizacaoDisponivel: false` para **toda** versao que o Agent Windows pergunta,
porque o Agent envia uma versao de 4 componentes e o `parseVersao` exige exatamente 3.

## Como cheguei nisso

Publicamos a 1.5.4 em staging, reiniciamos o `ManagerAgent` para forcar a checagem, e o log do agent
trouxe:

```
UpdateCheckerWorker: checking for updates. CurrentVersion=1.5.3.0,
  Endpoint=https://staging-api-admin.imanagerportal.com/api/agente/atualizacoes/verificar
UpdateCheckerWorker: no update available. Agent is up to date.
```

Com a 1.5.4 `ativa=true`, `pausada=false`, `rollout_percent=100`, `canary_only=false`,
`sistema_operacional=WINDOWS`. Nenhum filtro deveria ter barrado.

Bati no endpoint real para isolar:

```bash
curl "$URL/api/agente/atualizacoes/verificar?versaoAtual=1.5.3.0"
# {"atualizacaoDisponivel":false}

curl "$URL/api/agente/atualizacoes/verificar?versaoAtual=1.5.3"
# {"atualizacaoDisponivel":true,"versaoNova":"1.5.4","obrigatoria":true,...}
```

A unica diferenca e o quarto componente.

## Causa

`src/modules/agent-update/agent-update.service.ts`:

```ts
function parseVersao(versao: string): [number, number, number] {
  const parts = versao.split('.');
  if (parts.length !== 3) {
    throw new Error(`Formato versao invalido: ${versao}`);   // <- "1.5.3.0" cai aqui
  }
  ...
}

function isVersaoMaior(versaoNova: string, versaoAtual: string): boolean {
  try { ... } catch {
    return false;   // <- excecao vira "sem atualizacao", em silencio
  }
}
```

O `Assembly.GetName().Version.ToString()` do .NET **sempre** devolve 4 componentes. O Agent Windows
nunca conseguiu passar por esse parser.

**O auto-update do Windows nunca funcionou.** Ficou mascarado porque todas as releases foram instaladas
manualmente — e a instalacao manual apaga o `ProgramData` e revincula, entao ninguem sentiu falta.

## As duas mudancas

### 1. `parseVersao` aceita 3 ou mais componentes

```diff
-  if (parts.length !== 3) {
-    throw new Error(`Formato versao invalido: ${versao}`);
+  if (parts.length < 3) {
+    throw new Error(`Formato versao invalido (esperado major.minor.patch): ${versao}`);
   }
```

O quarto em diante e ignorado — a comparacao SemVer so olha major/minor/patch. Menos de 3 continua
sendo erro: `"1.5"` e ambiguo demais para adivinhar.

### 2. A falha de parse deixa de ser silenciosa

`isVersaoMaior` ganha um parametro opcional `logger` e emite `WARN` no `catch`. O retorno continua
`false` (nao ha como comparar duas versoes se uma nao parseia), mas agora **aparece**.

Isto e tao importante quanto a correcao em si: **falha de parse virando "voce esta atualizado" e o pior
default possivel**. O agent nunca atualiza e nada no log denuncia. Com esse WARN, o defeito teria
aparecido no primeiro deploy em vez de passar meses despercebido.

## Nota de parity — precisa da sua decisao

O comentario do codigo diz:

> `Byte-a-byte com Java parseVersao linhas 356-363`
> `Java retorna false (linha 349-350). Idem Node.`

Ou seja: **o Java tem o mesmo defeito** e a paridade o replicou fielmente. Ao sincronizar, corrigir os
dois — nao restaurar o comportamento antigo em nome da paridade. Deixei isso escrito no proprio codigo
para o proximo que passar por ali.

Vale a pena varrer os outros pontos de parity procurando defeitos herdados do mesmo jeito.

## O que ficou de fora de proposito

O `Number.parseInt` continua lenient: `parseInt("3-beta")` devolve `3`, entao `1.2.3-beta` e tratado
como `1.2.3`. Comportamento herdado, **preservado**. Apertar isso transformaria versoes hoje aceitas em
"sem atualizacao" — exatamente a falha silenciosa que este patch existe para eliminar. Se quiser
prerelease de verdade, e mudanca coordenada com o Java, em separado.

## Testes

Adicionei 6 casos em `test/unit/modules/agent-update/agent-update.service.test.ts`:

| Teste | Garante |
|---|---|
| `versaoAtual` com 4 componentes enxerga atualizacao | o caso real que estava quebrado |
| 4 componentes na mesma versao NAO oferece atualizacao | `1.5.4.0 == 1.5.4`, nada a fazer |
| candidata com 4 componentes tambem e comparada | o formato vale dos dois lados |
| menos de 3 componentes continua invalido | nao afrouxamos demais |
| `parseVersao` ignora alem do terceiro | `1.5.3.0` e `1.5.3.0.7` -> `[1,5,3]` |
| falha de parse deixa rastro no log | a parte que evita o proximo silencio |

## Lado do Agent (ja feito, @Bucky)

`UpdateCheckerWorker.ObterVersaoSemVer()` passa a montar `Major.Minor.Build`, com 2 testes — um deles
guarda direta contra alguem voltar a usar `Assembly.GetName().Version.ToString()`.

**Mas isso nao substitui o patch do backend.** A maquina de teste roda a **1.5.3**, que continua
mandando 4 componentes. Quem faz a pergunta e o binario velho. Sem esta correcao, a unica saida da
1.5.3 e instalacao manual — que nao testa o auto-update e ainda destroi o `ProgramData`.

Alem disso, corrigir so o Agent deixaria o backend fragil para qualquer cliente antigo no parque.
As duas pontas.

## Arquivos

- `src/modules/agent-update/agent-update.service.ts`
- `test/unit/modules/agent-update/agent-update.service.test.ts`
