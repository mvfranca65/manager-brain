# Testes — Colaboradores
> Dominio: @Natasha | QA
> Status: em refinamento

---

## Plano de testes: Tela Unificada de Colaboradores

### Backend — Endpoint `GET /api/v1/times/:timeId/colaboradores`

#### Autorizacao hierarquica
| Cenario | Esperado |
|---------|----------|
| Admin acessa qualquer time da empresa | 200 OK |
| Gestor acessa time abaixo dele na hierarquia | 200 OK |
| Gestor acessa time fora da sua hierarquia | 403 Forbidden |
| Viewer acessa time visivel | 200 OK |
| Acesso a time de outro tenant | 401/403 |
| Token invalido | 401 |

#### Paginacao e filtros (DB-level)
| Cenario | Esperado |
|---------|----------|
| Pagina 0, tamanho 16 | Retorna ate 16 colaboradores |
| Pagina 1 | Retorna proximo lote |
| Filtro por nome parcial | Retorna apenas matches |
| Filtro por identificador | Retorna apenas matches |
| Filtro por cargo | Retorna apenas matches |
| Status = ONLINE | Retorna apenas colaboradores com heartbeat < 10min |
| Status = AUSENTE | Retorna apenas colaboradores com heartbeat 10-30min |
| Status = OFFLINE | Retorna colaboradores sem heartbeat ou > 30min |
| Sem filtro de status | Retorna todos |
| Contadores de status | Refletem totais do time inteiro, nao da pagina |

#### Edge cases
| Cenario | Esperado |
|---------|----------|
| Time sem colaboradores | 200 com lista vazia e contadores zerados |
| Time com 500+ colaboradores | Resposta paginada sem OOM |
| Colaborador sem agente (sem heartbeat) | Status = OFFLINE |

---

### Frontend — Tela unificada

#### Funcional
| # | Criterio | Como testar |
|---|----------|-------------|
| 1 | Dropdown lista times do usuario | Login como gestor, verificar dropdown |
| 2 | Estado vazio sem time selecionado | Acessar `/colaboradores` sem param |
| 3 | Selecionar time carrega dados | Selecionar time, verificar faixa + cards |
| 4 | Busca com debounce 300ms | Digitar nome, verificar delay antes da request |
| 5 | Filtros de status com contadores | Clicar em cada filtro, verificar contadores |
| 6 | Paginacao | Navegar entre paginas |
| 7 | Deep linking | Acessar `/colaboradores?time=123` diretamente |
| 8 | Voltar do detalhe preserva time | Ir pro detalhe, voltar, verificar dropdown |
| 9 | Redirect `/times` → `/colaboradores` | Acessar URL antiga |
| 10 | Multitenancy | Gestor so ve times da sua empresa |
| 11 | Time sem colaboradores | Selecionar time vazio |
| 12 | Dropdown clear (X) | Limpar selecao |
| 13 | Time invalido no param | Acessar `?time=999999` |

#### Responsividade
| Breakpoint | Verificar |
|------------|-----------|
| Desktop (1024px+) | Dropdown 400px, cards 2 colunas, filtros inline |
| Tablet (768-1023px) | Dropdown 100%, cards 2 colunas |
| Mobile (375-767px) | Cards 1 coluna, filtros scroll horizontal |

#### Visual (@Groot)
| Item | Verificar |
|------|-----------|
| Faixa de resumo | Fundo `#EEF2FF`, borda esquerda `#6366F1` |
| Cards | Consistentes com design system |
| Empty states | Icone + texto em `#64748B` |
| Skeleton loading | Aparece durante carregamento |
