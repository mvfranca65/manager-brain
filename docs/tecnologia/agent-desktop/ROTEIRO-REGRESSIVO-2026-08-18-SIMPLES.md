> **DATA:** 2026-08-18 | **VERSAO:** Agent Windows 1.5.x
> Versao tecnica com comandos: `ROTEIRO-REGRESSIVO-2026-08-18.md`

# Roteiro de Testes — Agent Windows (linguagem simples)

Marque OK ou FALHOU em cada linha. Se falhar, anote o que aconteceu.

**Antes de comecar, defina:** para qual ambiente o agent vai enviar os dados (producao, staging ou teste) e qual chave de empresa vai usar. Por padrao ele envia para **producao**.

---

## 1. Instalacao

| O que testar | Como fazer | Resultado esperado |
|---|---|---|
| Instalar do zero | Rodar o instalador como administrador | Instala sem erro, pede identificador e chave |
| Agent liga sozinho | Esperar terminar a instalacao | Icone do iManager aparece na bandeja (perto do relogio) em ate 15s |
| Vinculacao | Conferir no portal | O computador aparece vinculado ao colaborador |
| Reinstalar por cima | Rodar o instalador de novo | Avisa que ja existe instalacao e mantem a configuracao anterior |

## 2. Icone e menu

| O que testar | Como fazer | Resultado esperado |
|---|---|---|
| Icone estavel | Olhar a bandeja por alguns minutos | Icone fica fixo, nao pisca nem some |
| Menu abre | Clicar com botao direito no icone | Menu abre na hora, mostrando "iManager - v1.5.x" |
| Ferramentas | Abrir o submenu Ferramentas | Opcoes abrem e funcionam ao clicar |
| Verificacao de saude | Menu > Ferramentas > Verificacao de Saude | Abre uma janela e mostra resultado bom (80% ou mais) |
| Limpar dados | Menu > Ferramentas > Limpar Dados e Reiniciar | Pergunta antes de fazer; se responder Nao, nada acontece |
| Sobre | Menu > Sobre | Mostra a versao e o colaborador configurado |

## 3. Captura do trabalho

| O que testar | Como fazer | Resultado esperado |
|---|---|---|
| Programas usados | Abrir Chrome, Bloco de Notas e VS Code, uns 10s em cada | Os tres aparecem registrados, com o nome da janela |
| Bloqueio de tela | Apertar Win+L, esperar 5s, desbloquear | Registra que bloqueou e que desbloqueou |
| Ausencia | Ficar 6 minutos sem mexer no mouse e teclado | Registra que voce ficou ausente e que voltou |
| Sinal de vida | Deixar o computador ligado por 5 minutos | O agent envia sinal de vida a cada 2 minutos |
| Reuniao | Entrar numa reuniao (Teams, Meet ou Zoom) por 3 minutos | Registra inicio e fim da reuniao |
| Desligar/dormir | Colocar o computador para dormir e acordar | Registra o periodo dormindo, sem inventar horas trabalhadas |
| Sair da conta | Fazer logoff do Windows e voltar | Registra a saida e a volta corretamente |

## 4. Status do colaborador

| O que testar | Como fazer | Resultado esperado |
|---|---|---|
| Ativo | Trabalhar normalmente | Status fica ATIVO |
| Pausa | Parar de mexer por 5 a 10 minutos | Status vira PAUSA |
| Ausente | Continuar parado ate passar 15 minutos | Status vira AUSENTE |
| Voltar | Mexer no mouse | Volta para ATIVO na hora |

## 5. Privacidade (LGPD) — qualquer falha aqui e critica

| O que testar | Como fazer | Resultado esperado |
|---|---|---|
| Nao grava o que voce digita | Digitar uma senha e um texto longo num email | Nada do que foi digitado aparece nos dados |
| Nao tira foto da tela | Usar o computador normalmente | Nenhuma imagem da tela e gerada ou enviada |
| Site sim, pagina nao | Navegar em varias paginas de um site | Registra so o site (ex: google.com), nunca a pagina completa |
| Titulo limpo | Abrir um email e uma conversa | Registra so o titulo da janela, nunca o conteudo |
| Diagnostico seguro | Menu > Ferramentas > Exportar Diagnostico | O arquivo gerado nao mostra senha nem chave da empresa |

## 6. Quando algo da errado

| O que testar | Como fazer | Resultado esperado |
|---|---|---|
| Sem internet | Desligar o Wi-Fi por 10 minutos e usar o computador | Continua registrando tudo; quando a internet volta, envia o acumulado sem repetir nada |
| Servidor fora do ar | Manter o computador ligado com o servidor indisponivel | O agent nao trava nem fecha; guarda os dados e tenta de novo |
| Agent fechado a forca | Fechar o agent pelo Gerenciador de Tarefas | Ele volta sozinho em ate 15 segundos |
| Computador reiniciado | Reiniciar o Windows | O agent sobe sozinho e o icone volta em ate 30 segundos |

## 7. Peso na maquina

| O que testar | Como fazer | Resultado esperado |
|---|---|---|
| Memoria | Abrir o Gerenciador de Tarefas e olhar o iManager | Cada parte do agent abaixo de 100 MB |
| Processador | Olhar com o computador parado | Praticamente 0% de uso |
| Nao pesa com o tempo | Conferir de novo depois de algumas horas | O consumo se mantem estavel, nao cresce sem parar |

## 8. Atualizacao automatica

| O que testar | Como fazer | Resultado esperado |
|---|---|---|
| Atualiza sozinho | Publicar uma versao nova e aguardar | O agent atualiza sem o colaborador precisar fazer nada |
| Continua funcionando | Conferir apos a atualizacao | Icone volta, versao nova aparece em Sobre, captura continua |
| Volta atras se der errado | Publicar uma versao com defeito proposital | O agent volta sozinho para a versao anterior e continua funcionando |

## 9. Mais de uma pessoa no mesmo PC

| O que testar | Como fazer | Resultado esperado |
|---|---|---|
| Dois usuarios logados | Trocar de usuario sem deslogar o primeiro | Cada um tem seu icone e seus dados registrados separadamente |
| Um sai, outro fica | Fazer logoff de um deles | O outro continua sendo monitorado normalmente |

## 10. Desinstalacao (deixar por ultimo)

| O que testar | Como fazer | Resultado esperado |
|---|---|---|
| Desinstalar | Painel de Controle > Aplicativos > desinstalar iManager | Sai por completo, icone desaparece, nada fica rodando |
| Desvincular | Conferir no portal | O computador some da lista de dispositivos vinculados |
| Instalar de novo | Rodar o instalador outra vez | Instala normalmente e volta a funcionar |

---

## Teste rapido (15 minutos)

Se quiser so uma validacao rapida antes da rodada completa:

1. Instalar e ver o icone aparecer
2. Abrir o menu e rodar a Verificacao de Saude (tem que dar 80% ou mais)
3. Abrir 3 programas diferentes e conferir se foram registrados
4. Fechar o agent a forca e ver ele voltar sozinho
5. Conferir memoria e processador no Gerenciador de Tarefas
6. Desinstalar e ver se sai limpo

Passou nos 6? Pode fazer a rodada completa.

---

## Como reportar um problema

Para cada falha, anote:

- **O que voce fez** (passo a passo)
- **O que esperava**
- **O que aconteceu**
- **Quando** (data e hora, para achar no log)
- Exporte o diagnostico: Menu > Ferramentas > Exportar Diagnostico

**Impede o lancamento:** agent nao liga, nao volta sozinho depois de fechado, perde dados, qualquer vazamento de privacidade, atualizacao que quebra sem voltar atras, ou nao conseguir desinstalar.
