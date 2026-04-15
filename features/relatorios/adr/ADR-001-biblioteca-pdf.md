# ADR-001: Biblioteca de geração de PDF
> Status: Aceita
> Data: 2026-03-25
> Decisor: @Tony | TL

## Contexto

O projeto Manager precisa gerar PDFs de relatórios semanais de produtividade analisados por IA. Existem três tipos de relatório: TIME (visão por equipe), INDIVIDUAL (por colaborador) e CONSOLIDADO (visão gerencial agregada). A geração ocorre no backend (Spring Boot 3.2.3, Java 17) e os PDFs são disponibilizados para download pelos gestores.

O relatório contém textos narrativos gerados por IA, tabelas de métricas, indicadores de produtividade e identidade visual do produto (cabeçalho, paleta de cores, tipografia). Portanto, a biblioteca escolhida precisa suportar layout programático com controle fino sobre posicionamento, fontes e cores.

A escolha da biblioteca tem impacto direto em licenciamento, custo operacional e viabilidade do modelo SaaS comercial do Manager.

## Decisão

Adotar **OpenPDF** (`com.github.librepdf:openpdf:2.0.x`) como biblioteca de geração de PDF no backend.

OpenPDF é um fork open-source do iText 4, mantido ativamente pela comunidade, com licença LGPL 2.1 / MPL 2.0. A API é programática (baixo nível), permite controle completo sobre layout, fontes, cores e estrutura de páginas, e é amplamente documentada — qualquer material de referência do iText 4 se aplica diretamente.

## Alternativas consideradas

### iText 7
- API moderna, documentação extensa, amplamente adotada no ecossistema Java.
- **Descartada:** licença AGPL v3. Em ambiente SaaS com código não publicado, exige a aquisição de licença comercial (custo significativo e incompatível com o estágio atual do produto). Inviável.

### Flying Saucer (xhtmlrenderer)
- Renderiza HTML/CSS para PDF, o que permitiria reaproveitar templates do frontend.
- **Descartada:** suporte restrito a CSS2 — sem flexbox, sem grid, sem variáveis CSS. Os relatórios do Manager exigem layouts que vão além dessas limitações. Manutenção mínima desde 2023, sem perspectiva de evolução.

### JasperReports
- Ferramenta madura e poderosa para relatórios tabulares e corporativos.
- **Descartada:** exige arquivos `.jrxml` e uma camada de design visual separada, com curva de aprendizado elevada. Overkill para o volume e a complexidade dos relatórios do Manager. Introduz acoplamento desnecessário ao projeto.

## Consequências

**Positivas:**
- Licença LGPL 2.1 permite uso em produto comercial sem obrigação de publicar o código-fonte.
- API familiar a qualquer desenvolvedor com experiência em iText 4; sem curva de aprendizado significativa.
- Controle programático total sobre layout, favorecendo a identidade visual do produto (cards, paleta azul-índigo, cabeçalho `#1e293b`).
- Pode ser combinado com Flying Saucer como renderizador HTML→PDF em casos futuros, se necessário.
- Dependência leve, sem arquivos de configuração externos.

**Negativas / riscos:**
- API de baixo nível exige mais código para layouts complexos em comparação a abordagens baseadas em templates HTML.
- Não há suporte nativo a templates declarativos; mudanças visuais exigem alteração de código Java.
- Projeto fork — embora ativo, depende da continuidade da comunidade mantenedora.

**Ação mitigadora:**
Encapsular toda a lógica de geração de PDF em uma camada de serviço isolada (`PdfReportService`), de forma que a substituição da biblioteca, se necessária no futuro, afete apenas essa camada.
