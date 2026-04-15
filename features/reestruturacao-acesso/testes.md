# Testes: Reestruturacao de Acesso
> Responsavel: @Natasha
> Atualizado em: 2026-03-31

---

## RA-34 — Login + Primeiro Acesso + Bloqueio

### T34.1 — Login de gestor (caminho feliz)
**Pre-condicao:** Gestor criado com primeiro acesso ja realizado (tem senha)
1. Acessar /login
2. Informar identificador + senha corretos
3. **Esperado:** Redireciona para /home, menu completo visivel (Home, Colaboradores, Relatorios, Organizacao, Perfil)

### T34.2 — Login de colaborador com acesso (caminho feliz)
**Pre-condicao:** Colaborador com `acessa_portal=true` e senha ja criada
1. Acessar /login
2. Informar identificador + senha corretos
3. **Esperado:** Redireciona para /meu-painel, menu reduzido (Meu Painel, Meus Relatorios, Meu Perfil)

### T34.3 — Login com credenciais invalidas
1. Acessar /login
2. Informar identificador inexistente ou senha errada
3. **Esperado:** Mensagem de erro generica "Nao foi possivel entrar. Verifique usuario e senha." Permanece na pagina de login.

### T34.4 — Login de usuario sem acesso ao portal
**Pre-condicao:** Colaborador com `acessa_portal=false`
1. Acessar /login
2. Informar identificador + senha
3. **Esperado:** Redireciona para /sem-acesso. Pagina mostra "Acesso nao autorizado" + mensagem + botao "Voltar ao login"

### T34.5 — Login de usuario com primeiro acesso pendente
**Pre-condicao:** Gestor ou colaborador recem-criado (senha_hash = null)
1. Acessar /login
2. Informar identificador + qualquer senha
3. **Esperado:** Mensagem "Voce ainda nao criou sua senha. Use o link 'Primeiro acesso' abaixo." Link visivel na pagina.

### T34.6 — Primeiro acesso completo (caminho feliz)
**Pre-condicao:** Usuario recem-criado com acessa_portal=true e sem senha
1. Acessar /login
2. Clicar no link "Primeiro acesso? Crie sua senha"
3. **Esperado:** Redireciona para /primeiro-acesso. Step 1 visivel.
4. Informar identificador + email corporativo corretos
5. Clicar "Validar"
6. **Esperado:** Mostra "Ola, {nome}!" e avanca para Step 2
7. Informar nova senha (min 8 chars) + confirmar senha
8. Clicar "Criar senha"
9. **Esperado:** Toast de sucesso. Redireciona para /login apos ~2 segundos.
10. Fazer login com identificador + nova senha
11. **Esperado:** Login bem-sucedido

### T34.7 — Primeiro acesso com dados invalidos
1. Acessar /primeiro-acesso
2. Informar identificador correto + email errado
3. Clicar "Validar"
4. **Esperado:** Mensagem de erro inline "Dados invalidos ou primeiro acesso ja realizado"

### T34.8 — Primeiro acesso com usuario que ja tem senha
**Pre-condicao:** Usuario que ja completou o primeiro acesso
1. Acessar /primeiro-acesso
2. Informar identificador + email corretos
3. Clicar "Validar"
4. **Esperado:** Mensagem de erro (primeiro acesso ja realizado)

### T34.9 — Primeiro acesso com senhas diferentes
1. Acessar /primeiro-acesso
2. Validar identificador + email com sucesso (Step 1)
3. No Step 2, informar senha "12345678" e confirmar "87654321"
4. **Esperado:** Erro de validacao "As senhas nao coincidem". Botao desabilitado.

### T34.10 — Primeiro acesso com senha curta
1. No Step 2, informar senha "123" + confirmar "123"
2. **Esperado:** Erro de validacao sobre tamanho minimo (8 chars)

### T34.11 — Pagina /sem-acesso
1. Acessar /sem-acesso diretamente na URL
2. **Esperado:** Pagina exibida com icone, titulo "Acesso nao autorizado", mensagem e botao "Voltar ao login"
3. Clicar "Voltar ao login"
4. **Esperado:** Redireciona para /login

---

## RA-35 — CRUD com 2 Perfis

### T35.1 — Criar gestor (sem senha)
1. Logar como gestor
2. Ir em Organizacao > Hierarquia
3. Clicar "Novo usuario"
4. Na Central de Gestao, clicar "Criar Novo Gestor"
5. **Esperado:** Modal abre com titulo "Criar Novo Gestor". NAO deve haver campos de senha. NAO deve haver seletor de perfil (Admin/Gestor).
6. Preencher nome, sobrenome, email, departamento, cargo, identificador
7. Clicar criar
8. **Esperado:** Sucesso. Gestor aparece na hierarquia com badge "Gestor"

### T35.2 — Criar colaborador com acesso ao portal
1. Na Central de Gestao, aba "Colaboradores"
2. Clicar "Criar Novo Colaborador"
3. **Esperado:** Modal com campo "Acessa o portal?" (select Sim/Nao, default Nao)
4. Preencher dados + selecionar "Sim" em "Acessa o portal?"
5. Criar
6. **Esperado:** Colaborador criado com acessa_portal=true

### T35.3 — Criar colaborador sem acesso ao portal (default)
1. Criar colaborador sem alterar o campo "Acessa o portal?" (deixar default)
2. **Esperado:** Colaborador criado com acessa_portal=false

### T35.4 — Editar flag acessa_portal no detalhe do colaborador
**Pre-condicao:** Colaborador com acessa_portal=false
1. Ir em Colaboradores > clicar no colaborador > aba Detalhes
2. Localizar campo "Acessa o portal?" na secao "Dados na Empresa"
3. **Esperado:** Mostra "Nao" (select editavel para gestores)
4. Alterar para "Sim"
5. **Esperado:** Toast de sucesso. Valor salvo no backend.
6. Recarregar pagina
7. **Esperado:** Campo mostra "Sim"

### T35.5 — Editar flag acessa_portal na hierarquia
1. Ir em Organizacao > Hierarquia
2. Clicar para editar um colaborador no organograma
3. **Esperado:** Campo "Acessa o portal?" visivel no formulario de edicao
4. Alterar valor
5. **Esperado:** Valor salvo corretamente

### T35.6 — Gestores aninhados na hierarquia
1. Criar Gestor A (sem gestor acima — topo)
2. Criar Gestor B com Gestor A como responsavel
3. Criar Time sob Gestor B
4. Criar Colaborador no time
5. **Esperado:** Hierarquia exibida corretamente: A > B > Time > Colaborador. Todos com badges corretos.

### T35.7 — Verificar que nao existe mais referencia a "Administrador"
1. Navegar por todas as telas do sistema
2. **Esperado:** Nenhuma referencia a "Administrador", "Admin", "ADM", "Visualizador", "VIS" em labels, badges, tooltips ou mensagens. Tudo deve ser "Gestor" ou "Colaborador".

---

## RA-36 — Visao do Colaborador

### T36.1 — Meu Painel
**Pre-condicao:** Login como colaborador com acesso
1. Apos login, verificar que aterrissa em /meu-painel
2. **Esperado:** Saudacao "Ola, {nome}!", cards de score/status/KPIs
3. Verificar menu lateral
4. **Esperado:** Apenas "Meu Painel", "Meus Relatorios", "Meu Perfil" visiveis. Nada de Colaboradores, Organizacao, Home.

### T36.2 — Meus Relatorios
1. Clicar em "Meus Relatorios" no menu
2. **Esperado:** Pagina /meus-relatorios carrega. Mostra navegador de semanas.
3. Selecionar uma semana
4. **Esperado:** Relatorio individual do colaborador logado exibido (mesmo componente usado na visao do gestor)

### T36.3 — Meus Relatorios sem dados
**Pre-condicao:** Colaborador sem semanas de dados
1. Acessar /meus-relatorios
2. **Esperado:** Estado vazio com mensagem (nenhum relatorio disponivel)

### T36.4 — Meu Perfil (campos editaveis)
1. Clicar em "Meu Perfil"
2. **Esperado:** Pagina de perfil carrega com dados do colaborador
3. Verificar secao "Dados na Empresa"
4. **Esperado:** Exibida como somente leitura (badge "Somente leitura" visivel)
5. Clicar "Editar perfil"
6. **Esperado:** Modal abre com campos editaveis: telefone, email pessoal, estado civil, genero, data nascimento. Campos bloqueados: nome, sobrenome, email corporativo, cargo, departamento.

### T36.5 — Meu Perfil (alterar senha)
1. Na pagina de perfil, secao "Seguranca"
2. Preencher senha atual + nova senha + confirmar
3. **Esperado:** Senha alterada com sucesso

### T36.6 — Tours ocultos para colaborador
1. Na pagina de perfil
2. **Esperado:** Secao "Tours e Onboarding" NAO aparece para colaboradores

### T36.7 — Colaborador nao acessa rotas de gestor
1. Logado como colaborador, tentar acessar /home diretamente na URL
2. **Esperado:** Redirecionado para /meu-painel (ou bloqueado pelo guard)
3. Tentar /colaboradores
4. **Esperado:** Redirecionado
5. Tentar /organizacao
6. **Esperado:** Redirecionado
7. Tentar /relatorios
8. **Esperado:** Redirecionado

---

## RA-37 — Excel sem Senha e sem Perfil

### T37.1 — Download template Excel de usuarios
1. Logar como gestor
2. Ir em Organizacao > Hierarquia > Novo usuario > Importar via Excel
3. Baixar o modelo Excel
4. **Esperado:** Planilha com 8 colunas: Identificador, Nome, Sobrenome, Email, Departamento, Cargo, Gestor responsavel, Times. NAO deve ter colunas "Perfil" e "Senha".

### T37.2 — Upload Excel de usuarios valido
1. Preencher planilha com dados validos (sem perfil, sem senha)
2. Fazer upload
3. **Esperado:** Validacao passa. Usuarios criados como GESTOR. Sem senha (primeiro acesso pendente).

### T37.3 — Download template Excel de colaboradores
1. Na Central de Gestao > aba Colaboradores > Importar via Excel
2. Baixar modelo
3. **Esperado:** Planilha com 16 colunas (A-P). Ultima coluna (P) deve ser "Acessa o portal?" com dropdown Sim/Nao.

### T37.4 — Upload Excel de colaboradores com acesso ao portal
1. Preencher planilha com colaboradores. Coluna "Acessa o portal?" = "Sim" para alguns
2. Fazer upload
3. **Esperado:** Colaboradores criados. Os marcados com "Sim" tem acessa_portal=true. Os demais (vazio ou "Nao") tem acessa_portal=false.

### T37.5 — Upload Excel de colaboradores sem coluna acessa_portal preenchida
1. Preencher planilha deixando coluna P vazia
2. Fazer upload
3. **Esperado:** Todos criados com acessa_portal=false (default)

---

## Checklist geral pos-testes

- [ ] Nenhuma referencia a "Administrador", "Admin", "ADM", "VIEWER", "VIS" em toda a UI
- [ ] Login diferencia corretamente os 3 tipos de erro
- [ ] Primeiro acesso funciona de ponta a ponta
- [ ] Tela /sem-acesso aparece para usuarios bloqueados
- [ ] Gestor ve menu completo, colaborador ve menu reduzido
- [ ] Colaborador so ve seus proprios dados (painel, relatorios, perfil)
- [ ] Colaborador nao consegue acessar rotas de gestor
- [ ] Perfil do colaborador mostra campos da empresa como somente leitura
- [ ] Excel de usuarios nao tem colunas Perfil e Senha
- [ ] Excel de colaboradores tem coluna "Acessa o portal?"
- [ ] Flag acessa_portal editavel no detalhe e na hierarquia
- [ ] Gestores aninhados funcionam na hierarquia
