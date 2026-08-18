# Especificação — Distribuição de Agents Windows e Android pelo Portal

**Status:** pronta para implementação  
**Data:** 2026-08-12  
**Owners:** @Tony (coordenação), @Shuri (Node), @Sam (Android), @Peter/Miles (Portal), @Natasha (QA), @Vision (infra/segredos)

## 1. Objetivo

Concluir a publicação e a distribuição autenticada de dois agentes pelo Portal:

- **Windows:** instalador ZIP personalizado por empresa, mantendo o fluxo atual.
- **Android:** APK-base publicado por versão, personalizado por empresa no download, re-assinado e entregue pelo mesmo fluxo do Portal.

O Portal Web nunca chama `srv-admin-node` diretamente. Todo tráfego do navegador é exclusivamente para `srv-portal-node`.

```text
fed-portal ── JWT do portal ──> srv-portal-node ── chave interna da empresa ──> srv-admin-node
                                                                        │
                                                Windows: ZIP + appsettings.json
                                                Android: APK + assets/config.json + assinatura
```

## 2. Escopo e decisões

### 2.1 Decisões fechadas

1. A aba **Agent** substitui o card macOS por **Android**.
2. A tela apresenta Windows e Android apenas quando existir release ativo para cada plataforma.
3. A rota canônica externa é `GET /api/instalador/download/:sistemaOperacional` no `srv-portal-node`.
4. A rota legada `GET /api/instalador/download` continua e equivale a `WINDOWS`.
5. O APK é **um artefato por empresa e versão**, não por colaborador. O colaborador é confirmado na primeira abertura do app.
6. O browser não recebe JWT, chave de ativação, URL de storage ou credencial de assinatura na URL. O download ocorre por `HttpClient` autenticado.
7. `versoes_agente.sistema_operacional` usa o vocabulário de release: `WINDOWS` e `ANDROID`. O vínculo do dispositivo usa o vocabulário canônico: `DESKTOP_WINDOWS` e `MOBILE_ANDROID`. Não misturar os dois contratos.

### 2.2 Fora de escopo

- macOS e iOS;
- distribuição por Google Play ou MDM;
- personalização por colaborador no binário;
- exposição pública de link personalizado;
- alteração dos serviços Java mortos.

## 3. Estado atual e lacunas confirmadas

| Capacidade | Windows | Android |
|---|---:|---:|
| Release por `sistemaOperacional` no admin-node | Sim | Metadado sim; artefato não |
| Upload de artefato correto | Sim, ZIP | Não, APK é rejeitado |
| Chave de storage segregada por plataforma | Não | Não |
| Verificação de update por SO | Sim | Consulta sim; URL/chave ainda incorreta |
| Download inicial pelo portal | Sim após rota parametrizada | Não |
| Configuração por empresa | Sim, ZIP | Não |
| Assinatura válida após personalização | N/A ZIP | Não existe |
| Confirmação do colaborador no primeiro uso | Sim | Implementada no app; requer testes integrados |

Lacunas técnicas que impedem Android hoje:

- `POST /api/admin/releases` exige nome e extensão `.zip`, com MIME fixo `application/zip`.
- A chave S3 atual não contém plataforma; uma versão Android pode colidir com Windows.
- A geração de URL de update calcula sempre a chave Windows.
- O download do admin escolhe a última versão sem filtrar plataforma e só reescreve `appsettings.json` de ZIP.
- O `srv-portal-node` bloqueia `ANDROID` antes de chamar upstream e a sua metadata só representa Windows/macOS.

## 4. Contratos de publicação

### 4.1 `POST /api/admin/releases` — `srv-admin-node`

Mantém `multipart/form-data`, `arquivo`, `versao`, `changelog?`, `obrigatoria?` e `sistemaOperacional`.

| sistemaOperacional | Nome obrigatório | Extensão | Content-Type | Chave do objeto base |
|---|---|---|---|---|
| `WINDOWS` | `ManagerAgent-Installer-v{versao}.zip` | `.zip` | `application/zip` | `releases/WINDOWS/{versao}/ManagerAgent-Installer-v{versao}.zip` |
| `ANDROID` | `ManagerAgent-Android-v{versao}.apk` | `.apk` | `application/vnd.android.package-archive` | `releases/ANDROID/{versao}/ManagerAgent-Android-v{versao}.apk` |

Regras:

1. Validar o nome, extensão e MIME esperado pelo SO antes do hash/upload.
2. Unicidade continua `(versao, sistema_operacional)`.
3. Persistir `url_download`, checksum e tamanho do **artefato-base** publicado.
4. `url_download` não é retornada ao fed-portal como link público de instalação. É informação administrativa/auto-update.
5. A listagem e inativação devem receber `sistemaOperacional` e usar exatamente o mesmo vocabulário de release.

## 5. Download inicial personalizado

### 5.1 Contrato Portal

```http
GET /api/instalador/download/WINDOWS
GET /api/instalador/download/ANDROID
Authorization: Bearer <portal-jwt>
```

O `srv-portal-node` deve:

1. validar o JWT do Portal;
2. resolver `empresaId` pelo contexto tenant;
3. recuperar a chave de ativação da empresa no banco;
4. chamar internamente o `srv-admin-node`, sem redirecionar o browser;
5. fazer streaming da resposta;
6. emitir `INSTALLER_DOWNLOADED` somente após upstream aceitar a solicitação, com `empresaId`, `usuarioId`, IP, User-Agent e `sistemaOperacional`;
7. retornar `Content-Type`, `Content-Disposition` e `Content-Length` do upstream.

Erros estáveis:

| Situação | Status | Código |
|---|---:|---|
| JWT ausente/inválido | 401 | padrão de autenticação |
| empresa sem chave | 404 | `MANAGER.INSTALADOR.CHAVE_AUSENTE` |
| SO inválido | 400 | `MANAGER.INSTALADOR.SISTEMA_OPERACIONAL_INVALIDO` |
| sem release ativo para o SO | 404 | `MANAGER.INSTALADOR.RELEASE_NAO_DISPONIVEL` |
| falha do admin/worker/storage | 502 | `MANAGER.INSTALADOR.UPSTREAM_FALHOU` |

### 5.2 Contrato interno admin-node

Adicionar endpoint literal antes do endpoint público por chave:

```http
GET /api/agente/atualizacoes/instalador/download/:sistemaOperacional
X-Ativacao-Key: <chave-da-empresa>
```

Compatibilidade: `GET /api/agente/atualizacoes/instalador/download` continua Windows. Aceitar `Authorization` legado no admin-node enquanto o portal migra para `X-Ativacao-Key`; novos chamados internos usam somente `X-Ativacao-Key`.

O endpoint deve buscar a última versão **ativa e não pausada** para o SO solicitado. O caminho público existente `/instalador/:chaveEmpresa` não deve ser usado pelo Portal nem receber Android neste escopo, porque mantém chave na URL.

### 5.3 Windows

Manter a implementação atual, tornando todas as consultas/chaves explicitamente `WINDOWS`:

- obter o ZIP-base da chave S3 Windows;
- reescrever `appsettings.json` com chave de ativação e endpoints;
- stream do ZIP com `Content-Type: application/octet-stream` e filename `ManagerAgent-Installer-{Empresa}.zip`;
- sem carregar o arquivo inteiro em memória.

### 5.4 Android

O APK-base contém `assets/config.json` com placeholders. No download:

1. buscar o APK-base ativo Android no storage;
2. criar diretório temporário exclusivo com permissões restritas;
3. substituir somente `assets/config.json` por JSON validado:

```json
{
  "ambiente": "prod|staging",
  "chaveAtivacao": "<chave da empresa>",
  "nomeEmpresa": "<nome da empresa>"
}
```

4. executar `zipalign` no APK modificado;
5. executar `apksigner sign` com a chave de assinatura de release;
6. executar `apksigner verify --verbose --print-certs` e falhar sem enviar bytes se a verificação falhar;
7. opcionalmente armazenar em cache privado por `(empresaId, versao, hashConfig, ambiente)` com TTL/versionamento; nunca reutilizar artefato entre empresas;
8. stream do APK com `Content-Type: application/vnd.android.package-archive` e filename `ManagerAgent-Android-{Empresa}-v{versao}.apk`.

Modificar um APK invalida a assinatura; por isso `zipalign` e `apksigner` são obrigatórios. Nesta fase, o `srv-admin-node` executa essas ferramentas no seu container, com keystore montado como segredo protegido e sem jamais expor seus caminhos, alias ou senhas. A extração para worker isolado permanece uma evolução operacional, sem alterar o contrato.

## 6. Segurança e infraestrutura da assinatura Android

1. Keystore, alias e senhas ficam em Secret Manager/KMS/volume protegido montado somente no `srv-admin-node`; nunca em Git, banco, logs, frontend ou respostas HTTP.
2. O processo aceita somente APK-base identificado por versão Android já publicada e configuração derivada do `empresaId`; não aceita caminho, shell argument ou JSON arbitrário do cliente.
4. Arquivos temporários são removidos em `finally`, inclusive em erro/cancelamento. Logs não podem conter chave de ativação.
5. A mesma chave de assinatura deve ser usada em todas as versões distribuídas por sideload, ou Android recusará atualização sobre instalação existente.
6. O APK personalizado é entregue apenas pelo proxy autenticado do Portal. Não gerar URL S3 pública, presigned URL para browser ou link com chave/token.

## 7. Portal Web

Na aba **Agent**:

- trocar a coluna macOS por Android;
- título: `Android`;
- requisito: `Android 8.0 ou superior` (minSdk 26);
- detalhe: tamanho e SHA-256 do APK-base publicado;
- botão chama `InstaladorApiService.download('ANDROID')` por `HttpClient` e salva Blob;
- não usar `window.open`; o interceptor injeta `Authorization: Bearer`;
- o card fica desabilitado quando a metadata Android estiver ausente;
- remover labels, tooltips, links e testes que tratam macOS como segundo artefato publicado.

`GET /api/instalador/versao-vigente` deve evoluir para retornar dados independentes:

```json
{
  "windows": { "versao": "1.5.1", "publicadoEm": "...", "tamanhoBytes": 0, "checksumSha256": "...", "disponivel": true },
  "android": { "versao": "1.0.0", "publicadoEm": "...", "tamanhoBytes": 0, "checksumSha256": "...", "disponivel": true }
}
```

Não retornar URL de download direto para nenhum SO. A resposta pode manter temporariamente o shape Windows atual como compatibilidade, mas novos consumidores usam o objeto por plataforma.

## 8. Primeiro uso e vínculo Android

Após instalar o APK personalizado:

1. `ConfigReader` lê empresa e chave do `assets/config.json`;
2. usuário informa matrícula ou e-mail;
3. app chama `POST /api/agente/v1/colaboradores/validar` com `X-Ativacao-Key`;
4. API retorna `nomeCompleto`; app pergunta **“Você é {nomeCompleto}?”**;
5. somente em **“Sim, sou eu”** persiste o identificador, solicita permissões e chama `POST /api/agente/dispositivos/vincular`;
6. vínculo envia `dispositivoTipo=MOBILE_ANDROID`, recebe Device JWT + refresh token e inicia a coleta;
7. em “Não, corrigir”, falha de validação ou falha de vínculo, não inicia coleta e não preserva um vínculo indevido.

Correção obrigatória antes de E2E: alinhar o enum aceito pelo `srv-admin-node` para `MOBILE_ANDROID` no vínculo e confirmar o DTO real da validação (`colaboradorId` versus `usuarioId`) com teste de contrato, sem aliases silenciosos.

## 9. Plano de implementação

### Fase A — Admin-node releases e artefatos (@Shuri)

1. Centralizar `ReleasePlatformMetadata` (SO, extensão, MIME, filename, chave S3).
2. Aplicar metadados no controller, service, DTO/Swagger, upload e testes.
3. Atualizar `keyForRelease` e presign de auto-update para usar SO.
4. Garantir coexistência Windows/Android com mesma versão sem overwrite.

### Fase B — Download e assinatura (@Shuri + @Vision)

1. Criar lookup `buscarUltimaVersaoAtivaPorSO`.
2. Separar o personalizador Windows do gerador Android.
3. Personalizar, alinhar, assinar e verificar APK no container seguro do `srv-admin-node`.
4. Criar endpoint interno de download por SO e preservar endpoint Windows legado.
5. Propagar SO, MIME e filename para `srv-portal-node`.

### Fase C — Portal (@Peter/Miles + Shuri)

1. Expandir metadata por plataforma no `srv-portal-node`.
2. Substituir macOS por Android na aba Agent.
3. Manter downloads em Blob autenticado e mensagens de erro coerentes.

### Fase D — Agent Android (@Sam)

1. Consolidar teste de contrato da validação e do vínculo.
2. Cobrir UI de confirmação: aceitar, corrigir, nome ausente e falha de vínculo.
3. Validar APK personalizado, assinatura e atualização sobre versão anterior no emulador/device.

## 10. Critérios de aceite e testes ponta a ponta

### Windows

- publicar ZIP Windows válido;
- listar e verificar update Windows retorna URL/chave Windows;
- `GET /api/instalador/download/WINDOWS` com JWT retorna ZIP personalizado;
- rota sem SO retorna o mesmo comportamento Windows;
- empresa A e B recebem configurações diferentes;
- audit registra SO `WINDOWS`, usuário, tenant, IP e UA.

### Android

- publicar `ManagerAgent-Android-vX.Y.Z.apk` com `sistemaOperacional=ANDROID`;
- upload grava MIME e chave S3 Android; publicação da mesma versão Windows não é sobrescrita;
- verificar update Android retorna APK Android, checksum e tamanho Android;
- `GET /api/instalador/download/ANDROID` com JWT retorna APK com MIME e nome corretos;
- extração controlada em teste confirma `assets/config.json` da empresa correta;
- `apksigner verify` passa; APK instala em emulador/API 26+;
- usuário valida identificador, vê o nome, confirma explicitamente e recebe vínculo `MOBILE_ANDROID`;
- “Não, corrigir” não aciona permissões, vínculo, token ou coleta;
- empresa sem chave, release ausente, SO inválido e falha da assinatura retornam códigos estáveis sem vazar segredos;
- download Android registra audit com `sistemaOperacional=ANDROID`.

## 11. Arquivos/repositórios impactados

| Repositório | Responsabilidade |
|---|---|
| `Backend/manager-srv-admin-node` | publicação multi-artefato, lookup por SO, update URL, personalizador Windows/Android e assinatura |
| `Backend/manager-srv-portal-node` | autenticação portal, tenant/chave, proxy por SO, audit e metadata por plataforma |
| `Frontend/manager-fed-portal` | card Android, metadata, download autenticado Blob e testes |
| `Agente/manager-srv-agent-android` | leitura config, confirmação da identidade, vínculo e testes UI/contrato |
| Infraestrutura | imagem do admin com Android Build Tools, Secret Manager/KMS, mount de keystore, permissões storage e observabilidade |

## 12. Dependências e ordem de deploy

1. Infra do admin + segredos de assinatura Android.
2. Admin-node: modelo de release/keys por SO e upload APK.
3. Admin-node: download APK personalizado e verificações.
4. Portal-node: proxy Android e metadata por plataforma.
5. Fed-portal: trocar card macOS por Android, inicialmente feature-flag/disabled até release Android publicado.
6. APK Android: confirmação de colaborador e testes integrados.
7. Staging E2E; somente depois habilitar Android em produção.

Rollback: inativar apenas o release Android na API de releases; o Portal volta a desabilitar o card Android sem afetar Windows.
